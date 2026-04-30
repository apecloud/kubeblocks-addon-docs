# Addon Probe 脚本 fork 与 zombie 进程指南

本文面向 Addon 开发与测试工程师，聚焦一个会被 5290 case 测试 0 product fail 都掩盖、但放到生产长寿 pod 上几小时就把集群打挂的问题：**addon probe / lifecycle 脚本里 fork 出去的后台子进程，会在容器内以 zombie 形式累积，最终撞 pod PID 上限**。

正文是 KubeBlocks 通用方法论，引擎相关现场放在案例附录。

## 先用白话理解这篇文档

写 addon 脚本时常见冲动：

- "这个 probe 顺便做点维护，比如修一下 cascade、清一下 stall。"
- "维护逻辑可能慢，干脆 `&` 扔到后台跑，别阻塞 probe 主路径。"

听起来很合理，但在 KubeBlocks 的 kbagent + 容器架构下会踩三个坑：

1. **kbagent 容器 PID 1 不 reap orphan child**。这意味着 probe 脚本退出后，留下的后台 subshell 会变 zombie 直到容器重启。
2. **probe 周期短**（roleProbe 默认 5s），**zombie 累积速率高**（实测 ~5-14 个/分钟）。
3. **K8s 默认 `pids.max=4096`**，**5-13 小时填满**。一旦填满，连 valkey-cli 都 fork 不出来，role probe 全失败，cluster phase 退化。

→ 真正的方法论是：**probe / lifecycle 脚本里禁止 fork 后台进程**。维护逻辑要么改成 sync（带 timeout 兜底），要么搬到本身有 reap 行为的 surface（独立 sidecar / 控制器 reconcile loop）。

### 何时本文方法论 apply

| 场景 | 关键决策 |
|---|---|
| 写新的 addon probe / action 脚本 | 设计上禁掉 `&` / `nohup` / `setsid` —— 不要 fork |
| 现有脚本里看到 `(... ) &` 或 `nohup` | 立刻 audit，确认是否会留 zombie |
| addon test 里 5000+ case 全过但生产长寿 pod 行为可疑 | 怀疑 zombie 累积，按本文工序量化 |
| 长 soak / 灾难演练时 pod 突然报 fork failed / cannot allocate | 大概率撞 pids.max，先看 zombie 数 |

### 读完你能做什么决策

- review addon PR 时，能看到 `&` 直接 block。
- 写新脚本时知道 sync + timeout 是默认形态。
- 看到 zombie 实测数据时，能判定"这是产品 bug" vs "这是 K8s default 紧"。

## 适用场景

当你的 addon 满足以下任一条件，本文适用：

- 提供 probe（roleProbe / availableProbe / readinessProbe / 自定义 probe）
- 提供 lifecycle action（postProvision / preStop / switchover / accountProvision 等）
- 脚本里有任何 `&` / `nohup` / `setsid -f` / `disown` 这类 detach 模式

## 核心边界：PID 1 reap 行为决定一切

理解 zombie 累积要先理解 Linux 的 reaping 模型：

1. 每个进程退出后变 zombie（占进程表 slot），等父进程 `wait()` 回收。
2. 如果父进程先死，孤儿被 reparent 到 PID 1。
3. **PID 1 必须显式 `wait()` 才能 reap zombie**。Linux 不自动回收。

容器场景的关键问题：**这个容器的 PID 1 是什么进程，它会不会 reap unrelated children**？

| 容器 PID 1 情形 | 是否 reap | 后果 |
|---|---|---|
| `tini` / `dumb-init` 等 init 包装 | ✅ 会 | 安全 |
| `shareProcessNamespace: true` + pause 容器 | ✅ pause 会 reap | 安全（但 PID namespace 共享有副作用） |
| Go binary 主程序（如 `kbagent`） | ⚠️ **不 reap unrelated children**（只 wait 它显式 fork 的） | **zombie 累积** |
| 业务进程（如 valkey-server / mariadbd） | ⚠️ 一般也不 reap unrelated children | **zombie 累积** |
| busybox `init` | ✅ 会 | 安全（但容器很少这样跑） |

**KubeBlocks 默认拓扑**：每个 addon pod 至少有 valkey/mariadb/oracle 业务容器 + kbagent sidecar 容器。这两个容器的 PID 1 都不是 init reaper。

## Audit checklist：fork 频率 × PID 1 reaper 行为

光找 `&` / nohup / setsid 这种 fork 关键字不够 —— 不是所有 fork 都会 leak。真正决定 leak 的是两个维度的 cross-product：

| | reaper PID 1（tini / dumb-init / pause / busybox init） | non-reaper PID 1（kbagent Go binary / valkey-server / mariadbd / 业务进程） |
|---|---|---|
| **低频 fork**（容器寿命内 1 次：start.sh 起 polling loop / entrypoint exec） | ✅ 安全 | ⚠️ 1 个长寿子进程，不堆积，但若该子进程 crash → zombie 1 个 |
| **高频 fork**（probe / action 周期触发：每 N 秒一次） | ✅ 安全 | ❌ **leak**：每周期 1 zombie，~5-13h 撞 `pids.max=4096` |

audit 时按这两个维度逐条 attribute 每个 fork pattern：

1. **找 fork**：`grep -rn '&[[:space:]]*$\|nohup\|setsid\|disown' addons/<engine>/scripts/ addons/<engine>/templates/`
2. **对每条命中**，回答：
   - **频率**：这个 fork 在容器寿命内执行几次？entrypoint 一次 vs probe 周期？
   - **reaper**：这个 fork 出去的子进程最终被 reparent 给哪个 PID 1？该 PID 1 是 init reaper 还是业务进程？
3. **判定**：
   - "高频 + non-reaper" → **必修**（valkey check-role.sh 命中这一格）
   - "低频 + non-reaper" → 可接受，但记录在案，**禁止后续把此 pattern 改成高频**
   - 任何 "reaper" 行 → 安全
4. **写到 audit 报告**：每条 fork 都列出"频率分类 + reaper 容器 + 判定"，留 paper trail。建议结构：

   ```
   | script:line                     | freq      | reaper container | verdict                 |
   |---------------------------------|-----------|------------------|-------------------------|
   | scripts/check-role.sh:216       | high      | kbagent (Go)     | LEAK — must fix         |
   | scripts/galera-start.sh:89      | low (1x)  | mariadbd (db)    | acceptable — flag only  |
   | templates/cmpd-semisync.yaml:280| low (1x)  | mariadbd (db)    | acceptable — flag only  |
   ```

   只有 verdict 列出现 `LEAK` 才是必修；`acceptable — flag only` 这一行要在后续 review 里继续监控，禁止把它升级成 high-freq。

### 容易混淆的细节

- **kbagent 镜像默认没装 `ps`**：用 `/proc/*/stat` 的 state 字段（`Z` = zombie）替代。
- **同一句 `&` 在不同容器里行为不同**：valkey 把 fork 写在 kbagent 跑的 probe 脚本里 → reaper 是 kbagent → leak；mariadb 把 fork 写在 mariadbd 容器的 entrypoint → reaper 是 mariadbd → 单次 fork 不堆积。**不要按"脚本里有 `&` 就一概 block"**，而要按 cross-product 决断。
- **看 cmpd.yaml `exec.container` 字段不够**：实测 kbagent 即便配了 `container: valkey`，仍可能在自己容器里跑这条脚本（kbagent 通过 HTTP probe API 调度，不是 nsenter）。**audit 时务必实测**，不要只读 yaml。最小验证命令：

  ```bash
  # 1) 找你想 audit 的 probe / action（这里以 roleProbe 为例）触发的脚本输出文件路径
  PROBE_TMP_FILE=/tmp/<engine>_role_maintenance.log    # 替换为你的 addon 实际写出文件

  # 2) 在 db 容器查这个文件是否存在（如果脚本真在 db 容器跑会有这个文件）
  kubectl exec <pod> -c <db_container_name> -- ls -la "${PROBE_TMP_FILE}" 2>&1

  # 3) 同时在 kbagent 容器查（如果脚本真在 kbagent 容器跑会有这个文件）
  kubectl exec <pod> -c kbagent -- ls -la "${PROBE_TMP_FILE}" 2>&1

  # 4) 哪个容器的 ls 找到了文件，脚本就在哪个容器跑。reaper PID 1 = 那个容器的 PID 1。
  kubectl exec <pod> -c <winning_container> -- cat /proc/1/comm
  ```

  也可以反向用"哪个容器有 zombie"做实测：
  ```bash
  for c in $(kubectl get pod <pod> -o jsonpath='{range .spec.containers[*]}{.name}{" "}{end}'); do
    echo "=== ${c} ==="
    kubectl exec <pod> -c "${c}" -- sh -c '
      for pid in $(ls /proc | grep -E "^[0-9]+$"); do
        [ -r /proc/$pid/stat ] || continue
        state=$(awk "{print \$3}" /proc/$pid/stat 2>/dev/null)
        [ "$state" = "Z" ] && echo "Z: $(cat /proc/$pid/comm 2>/dev/null)"
      done
    ' 2>/dev/null
  done
  ```
  哪个容器列出 zombie，fork 就在那个容器跑。

## 为什么 probe 脚本最容易踩

probe 脚本在哪个容器里跑、PID 1 是谁，决定 zombie 的归属：

- 直接看 cmpd.yaml 的 `roleProbe.exec.container` 字段说哪个容器。
- 但实测发现：**即便 cmpd.yaml 写 `container: valkey`，kbagent 还是会在自己的容器里跑这条脚本**（kbagent 通过 HTTP API 把 probe 调度给自己的 worker，不是 nsenter 进 valkey 容器）。
- 所以 fork 出去的 subshell 留在 **kbagent 容器**，被 reparent 到 **kbagent 进程**。kbagent 不 reap。zombie 累积。

不要假设"我的脚本写在 valkey 容器里就被 valkey-server reap"。要实际进 kbagent 容器看 PID 1 + zombie 数。

## 测试 / 验证工序

### 1. 起一个 minimal 集群，让 secondary pod 跑 probe 一段时间

至少一个 replica role（probe 在 replica 路径上 fork 才暴露问题；primary 路径通常 trivial 一行 echo 就退出，不 fork）。

### 2. 进 kbagent 容器读 /proc

`ps` 在 kbagent 镜像里通常没有，用 /proc 直接数：

```bash
kubectl exec <pod> -c kbagent -- sh -c '
zombies=0
for pid in $(ls /proc | grep -E "^[0-9]+$"); do
  [ -r /proc/$pid/stat ] || continue
  state=$(awk "{print \$3}" /proc/$pid/stat 2>/dev/null)
  [ "$state" = "Z" ] && zombies=$((zombies+1))
done
echo "zombies=${zombies}"
'
```

State 字段为 `Z` 即 zombie。也可以用 `ls /proc/*/comm` 顺手看 zombie 来源（比如 `check-role.sh` / `roleProbe.sh`）。

### 3. 至少做 N=2 时间点测量

不要单次 snapshot 下结论，至少：
- T0：pod 起来 ~3 min，记录 zombie 数
- T1：T0 后 ~3-5 min，再记录

如果 zombie 数线性增长且增速 > 0，确认 leak 存在。增速 = (T1 - T0) / 时间间隔。

### 4. 推算撞 pids.max 的时间

K8s 默认 `pids.max` ≈ 4096（节点级 / pod 级各一）。撞 limit 时间 = 4096 / 增速。生产里 5-13 小时常见。

## Fix 设计原则

### 原则 1：不要 fork

probe / 短期 lifecycle action 都做 sync 调用：

```bash
# ❌ 反模式
( do_maintenance ) >> /tmp/log 2>&1 &

# ✅ 推荐
do_maintenance || true   # sync, 不 fork
```

如果担心 sync 太慢卡住 probe 主路径，用 `timeout` 命令包远端调用：

```bash
do_maintenance_with_bound() {
  if command -v timeout >/dev/null 2>&1; then
    timeout 2 some_remote_cmd
  else
    some_remote_cmd
  fi
}
```

`timeout 2` 兜住 worst case 2s；总 probe latency = 本地操作 (~50ms) + 远端 timeout 上限。把这个总值控制在 cmpd.yaml 的 `roleProbe.timeoutSeconds` 之下（默认 3s，留 1s headroom 比较稳）。

### 原则 2：维护逻辑搬到长寿命 surface

如果维护本身耗时长（>2s 不可接受）或要持续运行，搬到能正确 reap children 的地方：

- **kbagent 自定义 probe / action**：kbagent 进程长寿命，对自己 fork 出来的子进程会 reap。在 cmpd.yaml 里另起一个 `customProbes` / `actions` 而不是塞进 roleProbe。
- **独立 sidecar 容器**：sidecar entry 用 tini 包装，确保 PID 1 reap。
- **KB controller reconcile loop**：如果维护需要 cluster-wide 视角（比如 cascade 修复），controller 已经在 reconcile，加一个反应器即可。

### 原则 3：observability 不靠 /tmp 文件

把维护决策的 echo 走 stderr → kbagent 把 stderr 抓进 probe event → `kubectl describe pod` 可见。

```bash
# ❌ 反模式
( ... ) >> /tmp/maintenance.log 2>&1 &
# 容器销毁就丢，运维 grep 不到

# ✅ 推荐
do_maintenance >&2   # stderr，kbagent 抓走
```

如果非要写文件，写到 `/proc/1/fd/1` / `/proc/1/fd/2` 把数据塞回容器主进程的 stdout/stderr，让 `kubectl logs` 抓到。

## 何时可以例外

少数情况确实需要 detach（典型：addon 内部要起一个长期 background watcher 一类）。这种情况：

- 不要在 probe 脚本里做。改在容器启动脚本（postProvision / start.sh）里 fork 一次。
- fork 完用 `setsid -f` 或 `nohup` + 写 PID 文件，便于运维管理。
- 自己处理 reap：要么再 fork（双 fork pattern + setsid），要么由 wrapper 脚本 trap SIGCHLD `wait`。
- 仍要在 cmpd 加一个 `shareProcessNamespace: true` 或 sidecar tini，让长寿后台进程死掉时能被 reap。

不要把例外路径混进 probe 主路径。

## 与其他主题的关系

- 单条 probe 失败的归因分层见 [`docs/addon-test-probe-classification-guide.md`](addon-test-probe-classification-guide.md)。
- 控制器 crash 与 desired-state 收敛见 [`docs/addon-controller-crash-resilience-guide.md`](addon-controller-crash-resilience-guide.md)。
- bootstrap 期 role 发布 vs role probe 见 [`docs/addon-bootstrap-role-publish-guide.md`](addon-bootstrap-role-publish-guide.md)。
- ship-readiness 测试矩阵能否暴露这类长寿现象的 caveat 见 [`docs/addon-ship-readiness-multi-phase-validation-guide.md`](addon-ship-readiness-multi-phase-validation-guide.md)。

## 配套案例

- [`docs/cases/mariadb/mariadb-fork-audit-pass-negative-case.md`](cases/mariadb/mariadb-fork-audit-pass-negative-case.md) — MariaDB addon 4 处 fork pattern 都落"低频 + non-reaper"格 audit pass 的 negative case，作为本文 audit checklist 二维表的对照例子（不是所有 fork 都 leak）

## 案例附录：Valkey check-role.sh 的 zombie 漏洞

Valkey addon 的 `check-role.sh`（roleProbe）在 secondary 分支末尾用 `( ... ) >>"${ROLE_MAINTENANCE_LOG_FILE}" 2>&1 &` fork 出 subshell 跑 cascade + stall self-heal。意图是"不阻塞 probe 主路径"。

### 实测现场

- 起 1 primary + 1 secondary cluster，pod 寿命约 12 min。
- secondary pod kbagent 容器 PID 1 = `kbagent` Go binary，不 reap unrelated children。
- 测量数据：

| 时间 | zombie 数 |
|---|---|
| T0 (pod ~3 min) | 28 |
| T1 (+3 min) | 42 |
| T2 (+1 min) | 56 |
| T3 (+8 min, after CM patch) | 153 |

增速 ~5-14 zombie/min。按 K8s `pids.max=4096` 估算，**5-13 小时撞顶**。

### Fix 路径

去掉 fork，改成 sync 直接调 cascade + stall：

```diff
-run_secondary_self_healing_async() {
-  if [ "${ROLE_MAINTENANCE_ASYNC}" = "false" ]; then
-    run_secondary_self_healing >>"${ROLE_MAINTENANCE_LOG_FILE}" 2>&1 || true
-    return 0
-  fi
-  ( ...mkdir lock + run_self_heal... ) >>"${ROLE_MAINTENANCE_LOG_FILE}" 2>&1 &
-}
+# 删除 async wrapper，主路径直接 sync 调 run_secondary_self_healing
```

主路径：

```diff
   "role:slave")
     printf %s "secondary"
-    run_secondary_self_healing_async
+    run_secondary_self_healing
     ;;
```

cascade 函数内的远端 INFO 调用本来就有 `info_replication_with_timeout` 包装，把 timeout 默认从 3s 收到 2s，确保 worst-case 总延迟 ≤ kbagent `roleProbe.timeoutSeconds=3`。

### Fix 验证

| 时间 | zombie 数 | 说明 |
|---|---|---|
| 18:02:49（fix mount 起效后即刻） | 153 | fix 前的存量 |
| 18:03:25 (+30s) | 153 | +0 |
| 18:03:56 (+1 min) | 153 | +0 |
| 18:04:27 (+1.5 min) | 153 | +0 |
| 18:04:58 (+2 min) | 153 | +0 |

post-fix 跨 ~2 min 共 4 个采样点，zombie 数稳定在 153 不增长。kbagent log 显示 probe 仍正常工作（`succeed to send the probe event ... output: secondary`）。

### 副作用：观测性恢复

老版本 cascade / stall 决策的 echo 都被 redirect 到 `/tmp/valkey_role_maintenance.log`（容器内 tmpfs），cluster 销毁就丢。新版本直接走 stderr，kbagent 抓进 probe event，`kubectl describe pod` 可见。

### ship-readiness 5290 case 没暴露的原因

ship-readiness 的每 round cluster 寿命 ~3-5 min，加上 immediate cleanup 立刻销毁。**zombie 累积速率虽然存在，但跨 round 不会传递**（容器销毁 PID 表清零），所以 5290 case 全过。

教训：**短寿测试 cluster 不能用来验证长寿生产 pod 的 PID 表行为**。这条 caveat 应该写进 ship-readiness 矩阵的"未覆盖维度"里。

### 测试 artifacts

- 验证脚本：临时 zombie-test cluster，执行步骤上面已列。
- 验证环境：k3d `kb-local`，KubeBlocks 1.2，Valkey addon rev 57

## 相关主题

- [`docs/addon-test-probe-classification-guide.md`](addon-test-probe-classification-guide.md)
- [`docs/addon-controller-crash-resilience-guide.md`](addon-controller-crash-resilience-guide.md)
- [`docs/addon-ship-readiness-multi-phase-validation-guide.md`](addon-ship-readiness-multi-phase-validation-guide.md)
