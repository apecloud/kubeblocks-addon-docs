# Addon Probe 超时与软失败语义指南

本文面向 Addon 开发工程师与技术负责人，重点解决"K8s `livenessProbe` / `readinessProbe` 的快速二态语义" 与 "数据库控制面慢但合法响应" 之间的张力。它不是关于测试探针（参见 `addon-test-probe-classification-guide.md`），而是关于**addon 在 CmpD 里挂的探针脚本怎么写、参数怎么设，才不会在合法的慢窗口里把容器误杀**。

## 适用场景

- 用 SQL/CLI 客户端（sqlplus / mysql / redis-cli / dgmgrl 等）作为 livenessProbe / readinessProbe exec 的脚本
- 引擎在某些状态下控制面响应天然变慢（switchover / failover / 备份恢复 / DG broker reconfigure / 主备角色重建 / 大量 redo apply / startup 期 instance restart）
- 出现下列任一现象：
  - 容器 restarts 不断增长，但 DB 内部日志看不到崩溃证据
  - 多副本中只有"刚刚做完角色变更/拓扑变更"的那个 pod 被反复 kill
  - readiness 反复 flap，导致 KB lifecycle action 的 `RuntimeReady` / `ComponentReady` preCondition 永远不满足
  - smoke 单测可过，但跑组合（reconfigure_deep / chaos）时 fix 之前 PASS 的子用例突然 fail

## 核心原则

> 探针的两态信号要落到"产品语义"，不要落到"客户端 timeout"。
>
> rc!=0 是**信道层错**，不是产品层错。把信道层错当成 "DB 不可用" 是这个领域绝大多数 false positive 的根源。

具体做法：

1. **每一次外部调用都显式 `timeout N`**。不能让客户端阻塞超过你能控制的窗口。
2. **rc!=0 一律 `exit 0`**（transient / unknown），不计入失败计数。
3. **只有"客户端成功返回 + 输出确认 bad state"才 `exit 1`**。
4. **已知合法慢窗口用进程探针守门（pgrep）**：检测到正在跑的合法工作（init / DBCA / RMAN / 备份恢复 / setup_dg_*）时，直接 `exit 0`，连客户端都不调。
5. **probe 参数三件套要算给最慢的合法窗口留余量**：`initialDelaySeconds`、`periodSeconds`、`timeoutSeconds * failureThreshold`。

## 为什么必须显式分清"信道层错 vs 产品层错"

K8s probe 的 contract 是 binary：rc=0 = 健康，rc!=0 = 不健康。但 DB 控制面在合法状态下也可能：

- 客户端连接超过 `timeoutSeconds` 才返回 → 网络/sock 不是问题，但 client 端被 K8s SIGKILL 收掉
- 角色变更瞬间所有 client 拿到 `ORA-XXXX` / 拒绝连接 → 几秒后自然恢复，但这一瞬被 probe 取样
- 控制面 reconfigure 期间元数据查询 hold 住 → DB 业务面其实可读可写

如果直接 "rc!=0 → exit 1" / "stderr 含 ORA → exit 1"，上面三种**合法慢**全部被 probe 当成"DB 死了"。失败一旦累计到 `failureThreshold`：

- livenessProbe → 容器被 SIGKILL → restart → 再赶上下一个慢窗口 → 再 kill → **cascade**
- readinessProbe → endpoint 摘出 service → KB controller 看到 `Status.ReadyReplicas` 下降 → `RuntimeReady` 不满足 → memberJoin / hscale / reconfigure 的 lifecycleAction 永远不触发 → cluster 卡 Updating

这两条 cascade 是同一个根因不同表象。修法是同一套：把信道层错降级成 transient。

## 设计规约（写 probe 脚本的 7 条硬规则）

### 1. 客户端调用必须 `timeout N` 包装

不要相信 client 自己的 `--connect-timeout`。`timeout` 命令对 SIGKILL 守门，client 自带的超时不能完全保证。

```bash
# Bad
out=$(sqlplus -S / as sysdba <<<"select status from v\$instance;")
rc=$?

# Good
out=$(timeout 25 sqlplus -S / as sysdba 2>/dev/null <<EOF
  set pages 0 head off feedback off
  select status from v\$instance;
  exit
EOF
)
rc=$?
```

`timeout N` 的 N 要明显小于 probe 的 `timeoutSeconds`（典型留 5s 余量），让 timeout 命令自己先返回 rc=124，而不是被 probe 杀掉。

### 2. rc!=0 默认是 transient

```bash
if [ $rc -ne 0 ]; then
  echo "probe: client unavailable (rc=$rc), treat as transient"
  exit 0
fi
```

这一行是这个 guide 的核心。**不要做"区分超时 vs 其它错误"的细分**——任何 client 端 rc!=0 在 probe 语义里都是 unknown，都是 transient。

### 3. exit 1 必须建立在 client 成功返回 + 输出明确 bad state

```bash
# 只有这条 path 触发 exit 1
case "$status" in
  ACTIVE|MOUNTED|OPEN) exit 0 ;;
  STARTED|NOMOUNT)     exit 0 ;;  # 启动中，合法
  *)                   echo "instance bad state: $status"; exit 1 ;;
esac
```

这意味着 probe 只在 "我能确定地说 DB 现在 hosed" 时 SIGKILL 容器；其余情况都给 DB 一次机会。

### 4. 已知合法慢窗口用 pgrep 守门，越早越好

如果引擎有"在这一段时间内 client 必然慢"的明确进程标记（init / DBCA / 引擎重启脚本 / 备份恢复工具），在脚本最前面 short-circuit：

```bash
if pgrep -f 'dbca' > /dev/null 2>&1; then
  exit 0
fi
if pgrep -f 'setup_dg_primary|setup_dg_secondary|orapwd' > /dev/null 2>&1; then
  exit 0
fi
```

注意：

- pgrep 模式要稳，不能太宽（否则永远 exit 0 = probe 失效）也不能太窄（否则漏窗口）。`-f` + 强特征字符串是常用做法
- pgrep 必须在 client 调用 **之前**，不能在之后；否则 client 已经在赶慢窗口里被 kill 一次了
- pgrep 守门的进程必须在 init / 角色切换脚本里**显式存在**（自己写的脚本要保留进程名，不要 `bash -c "..."` 把名字洗没）

### 5. probe 参数三件套要按"最慢合法窗口"反推

`failureThreshold` × `periodSeconds` 决定从"开始失败"到"被 kill"的总窗口。这个窗口必须 ≥ 最慢合法操作的时间。

| 参数 | 推荐设法 |
|---|---|
| `initialDelaySeconds` (liveness) | 算到最慢初始化完成（含 DBCA / DG 配置 / 系统账号注入），留 50% 余量。默认 30s 这个值 **必坑** |
| `initialDelaySeconds` (readiness) | 短即可（10-30s），目的是初始化期间 endpoint 不进 svc，不会误杀 |
| `periodSeconds` | 不要小于 `timeoutSeconds`，否则同一时刻多个 probe 进程并存 |
| `timeoutSeconds` | 留余量给客户端 `timeout` 命令（`timeoutSeconds = client_timeout + 5`） |
| `failureThreshold` | 至少 5。让一两次合法慢被吸收 |

### 6. 同一 readiness 不要拉外部依赖来判健康

readiness 决定的是"自身能不能接流量"，不是"集群是否健康"。所以：

- 不要在 readiness 里要求 DG broker SUCCESS / replication healthy / quorum 满足。这些是 component 级状态，应由 lifecycleAction 或 controller 判断
- 如果非要查（用于 standby 不接读流量等场景），所有控制面调用都要按规则 1+2 软失败。**`ORA-16532 (broker not configured)` / `Replication broken` 这种"控制面尚未就绪"语义要 exit 0，不是 exit 1**

### 7. 验证 fix 是否真的生效，唯一可信信号是 `restarts=0`

不要看脚本的 md5、不要看日志里的 sentinel print。fix 是否真的避免了 cascade，唯一硬证据是：

- 在跑完一整套含 switchover / reconfigure / restart / hscale 的 smoke 之后
- `kubectl get pod -n $NS` 看到所有引擎容器 `restarts=0`

如果 restarts > 0：

- 看 `kubectl describe pod` 的 events，确认是 liveness probe failed 还是 OOMKilled / Error / completion
- 如果是 liveness probe failed，回头复盘是脚本里哪一条 path 漏了软失败 / pgrep 没盖到哪个慢窗口

## 反模式（必须避免）

| 反模式 | 后果 |
|---|---|
| `sqlplus / as sysdba <<< "..."` 不带 timeout | client hang 直到被 K8s SIGKILL，restarts cascade |
| `[ -n "$err" ] && exit 1` （任何 stderr 内容都 fail） | banner / NLS warning / `ORA-16xxx (transient)` 全部命中，readiness flap |
| `periodSeconds=10` + `timeoutSeconds=10` | 同时刻有上一周期还没结束的 probe 在跑，host load spike |
| `initialDelaySeconds=30` 给一个会跑 DBCA 几分钟的引擎 | DBCA 跑一半就被 kill，初始化永远不完成 |
| 用 readiness 判 cluster-level 健康 | 单 pod 的 readiness 受集群其它节点影响，flap 不可避免 |
| pgrep 守门**之后**才 timeout sqlplus | 进程已经被 kill 过一次，pgrep 拿不到 → 没意义 |
| 探针失败时往 stderr 写 stack / 长输出 | Pod event log 被 spam，真问题被淹 |

## 与其它指南的关系

- `addon-test-probe-classification-guide.md` 讲**测试探针**失败如何分层。本篇讲**addon 内部 livenessProbe/readinessProbe 脚本**怎么写。两者方向不同
- `addon-bootstrap-role-publish-guide.md` 讲 bootstrap 期 role truth 与 publish 链。本篇与之互补：bootstrap 期 publish 链跑通的前提之一就是 readiness 不能在合法慢窗口里 flap
- `addon-bounded-eventual-convergence-guide.md` 讲对异步收敛系统判定要 bounded retry。本篇是它在 probe 脚本侧的具体落地——probe 本身就是一次单点 snapshot，软失败语义把"等收敛"这件事还原回去
- `addon-test-acceptance-and-first-blocker-guide.md` 里的 first blocker 分类，post-switchover/restart 后 oracle 容器被 liveness 反复 kill 是典型的 "addon-design first blocker"，不是产品 bug、不是测试 bug

## 自检清单（PR review 与 chart upgrade 前必过）

- [ ] 脚本里所有 `sqlplus / mysql / dgmgrl / redis-cli / mongosh / etcdctl` 都被 `timeout N` 包了
- [ ] 脚本第一段是 pgrep 守门（如有合法慢窗口）
- [ ] client rc!=0 path 是 `exit 0` + 一句 stderr 说明（"transient"），不是 exit 1
- [ ] exit 1 path 仅在 client 成功返回 + 输出明确 bad state 时触发
- [ ] CmpD 里 `livenessProbe.initialDelaySeconds` ≥ 最慢合法初始化时间 × 1.5
- [ ] CmpD 里 `failureThreshold ≥ 5`
- [ ] CmpD 里 `periodSeconds ≥ timeoutSeconds`
- [ ] readiness 没有"外部依赖"判定（broker SUCCESS / replication healthy / quorum 等）
- [ ] 跑过完整 smoke（含 switchover + reconfigure + restart + hscale）之后所有引擎容器 `restarts=0`

## 案例附录

具体引擎案例放在 `cases/` 目录，按"一案一篇"组织：

- `cases/oracle-12c-post-switchover-probe-cascade-kill.md`（pending — 等 reconfigure_deep Run 3 全 PASS 后落） — Oracle 12c DG broker / switchover / DBCA / RMAN active duplicate 四个慢窗口与本指南规则的映射
