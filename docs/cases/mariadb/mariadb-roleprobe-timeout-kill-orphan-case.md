# MariaDB roleProbe timeout-kill orphan zombie case

属于：[`addon-probe-script-fork-and-zombie-guide.md`](../../addon-probe-script-fork-and-zombie-guide.md) **Pattern B（隐式 timeout-kill orphan）** 的实证补充。配合 [`mariadb-fork-audit-pass-negative-case.md`](mariadb-fork-audit-pass-negative-case.md) 一起读：那篇是 Pattern A（显式 fork）audit pass，本篇是 Pattern B 的低速率 leak。

> **Audience**：addon dev 给新 probe / lifecycle 脚本设 `timeoutSeconds` 时；addon test 在 idle / soak / chaos 阶段做 zombie 验证时
> **Status**：stable
> **Applies to**：MariaDB addon `1.1.1-alpha.12`（cadence 3s + timeoutSeconds 1s 组合）；机制对所有 KB addon 适用
> **Applies to KB version**：any（kbagent timeout SIGKILL 行为跨 KB 版本不变）

## 这是什么

04-30 task #402 给 MariaDB addon `1.1.1-alpha.12`（roleProbe cadence 5s→3s 后第一版）做 idle cluster 长时间 zombie 验证。预期是按 [`mariadb-fork-audit-pass-negative-case.md`](mariadb-fork-audit-pass-negative-case.md) 的 audit 结论 4h 全 0 zombie。

实测：

| 采样点 | total zombies | per pod / container 分布 |
|---|---|---|
| baseline 5min | 0 | 0/0/0/0/0/0 |
| sample 30min | 0 | 0/0/0/0/0/0 |
| sample 1h | 0 | 0/0/0/0/0/0 |
| **sample 2h** | **1** | **pod-2 kbagent=1**, 其它全 0 |

2h redline 触发 freeze + reverse investigation，机制坐实为 [`addon-probe-script-fork-and-zombie-guide.md`](../../addon-probe-script-fork-and-zombie-guide.md) **Pattern B — 隐式 timeout-kill orphan**：probe 1s timeout 偶发被 SHOW SLAVE STATUS 越界，kbagent SIGKILL 脚本主进程，pipeline 中的 grep 子进程被 reparent 到 kbagent (PID 1, Go 不 reap) → zombie。

leak rate ≈ 1/2400 probes ≈ **0.04%**。远低于 Pattern A 的 valkey check-role.sh `5-14/min` storm，但**机制同源**。

按主文档 4D audit cell：

| 维度 | 值 |
|---|---|
| context (W) | **kbagent-scheduled** (roleProbe via cmpd.yaml) |
| source (X) | **B** — 隐式 timeout-kill orphan |
| frequency (Y) | **high** — cadence 3s 周期触发 |
| reaper (Z) | **non-reaper** — kbagent (Go binary) PID 1 不 reap unrelated children |

落 **(kbagent, B, high, non-reaper) → ❌ LEAK** cell。跟 valkey check-role.sh:216 (kbagent, A, high, non-reaper) 同 cell，rate 不同 source 不同 cell 相同，需要 must-fix。

## 现场坐实证据链

### 1. zombie 直接命中

`kubectl exec -c kbagent <pod-2> -- sh -c '...zombie scan...'` 在 2h 采样点的输出：

```
pid=1     ppid=0  state=S  comm=kbagent     ← PID 1, Go binary, 不 reap
pid=34850 ppid=1  state=Z  comm=grep        ← zombie, parent = kbagent
pid=57881 ppid=0  state=S  comm=sh          ← 当前 sampler，非凶手
```

PID 34850 ppid=1 → reparent 到 kbagent。其它 sampler PIDs（5min/30min/1h: 2591/14670/29196）全部正常退出未留尸 → 排除 sampler 自身污染。

### 2. /proc/34850 完整状态确认

```
Name=grep
State=Z
PPid=1
cmdline=（空，符合 zombie 已退出特征）
```

### 3. kbagent log smoking gun

pod-2 kbagent 日志精确时间点：

```
2026-04-30T12:30:40Z roleProbe code=-1 output="secondary" message="timedOut"
2026-04-30T12:30:42Z roleProbe code=0  output="secondary"   ← 下一周期恢复
```

`code=-1` + `message="timedOut"` 触发 SIGKILL。`output="secondary"` 出现在 timeout 事件里说明：脚本确实跑到 line 122 `echo -n "secondary"` 阶段（4 个 grep pass 了），但 kbagent 在 wait 上拿到 SIGKILL 退出码（-9），把整次 probe 标 -1。kbagent 同时把 stdout buffer 里 "secondary" 抽走作为 output。

下一周期（cadence 3s，2s 后）恢复正常，符合"timeout 是偶发，下次 SHOW SLAVE STATUS 又 <1s"模型。

### 4. cluster 角色分布与脚本路径对应

```
pod-0 = primary    (label kubeblocks.io/role=primary)
pod-1 = secondary  (label kubeblocks.io/role=secondary)
pod-2 = secondary  (label kubeblocks.io/role=secondary)  ← 唯一发现 zombie 的 pod
```

`replication-roleprobe.sh:check_role()` 路径分歧：

```bash
if [ -f "$(master_info_file)" ]; then
  # secondary 路径
  db_ready || { not_ready; return $?; }
  secondary_replication_ready || { not_ready; return $?; }
  echo -n "secondary"
else
  # primary 路径（trivial）
  echo -n "primary"
fi
```

`secondary_replication_ready()` 函数（line 89-100）：

```bash
secondary_replication_ready() {
  local slave_status
  if [ "${MARIADB_ROLEPROBE_SKIP_DB_READY:-}" = "true" ]; then
    return 0
  fi
  slave_status=$(query_slave_status)                # 命令替换：1 个 subshell
  [ -n "${slave_status}" ] || return 1
  printf '%s\n' "${slave_status}" | grep -q "Slave_IO_Running: Yes" || return 1   # pipeline 1
  printf '%s\n' "${slave_status}" | grep -q "Slave_SQL_Running: Yes" || return 1  # pipeline 2
  printf '%s\n' "${slave_status}" | grep -q "Last_IO_Errno: 0" || return 1        # pipeline 3
  printf '%s\n' "${slave_status}" | grep -q "Last_SQL_Errno: 0" || return 1       # pipeline 4
}
```

4 个 grep pipeline + 1 个命令替换 + `query_slave_status` 内部远端 mariadb 调用 → 任一段被 kbagent SIGKILL 截断都可能留 orphan grep。primary 路径只有 `echo -n "primary"`，0 child fork → primary 不踩本 case。

`pod-1` 和 `pod-2` 都是 secondary 都跑同一段，**只 pod-2 在 2h 内中招**——纯属时序方差（哪次 SHOW SLAVE STATUS 偶发 >1s）。如果继续跑到 4h / 8h，pod-1 也会有概率出现 1-2 个 zombie。

## 为什么 Pattern A audit 没发现

`mariadb-fork-audit-pass-negative-case.md` audit 命令：

```bash
grep -rEn '\&[[:space:]]*$|nohup|setsid|disown' \
  addons/mariadb/scripts/ addons/mariadb/templates/
```

`replication-roleprobe.sh` 0 命中（脚本里没有 `&` / nohup / setsid / disown）。**所以 Pattern A audit 通过**。

但 Pattern A audit 完全看不见 pipeline / 命令替换里**潜在的 timeout-kill orphan**。这是 [`addon-probe-script-fork-and-zombie-guide.md`](../../addon-probe-script-fork-and-zombie-guide.md) Pattern B 章节存在的原因。

## 修复方向（候选 patch）

按主文档推荐的优先级：

### Patch 1（最便宜，保护性）：放宽 `roleProbe.timeoutSeconds` 1s → 2s

`addons/mariadb/values.yaml`:

```yaml
roleProbe:
  periodSeconds: 3
  timeoutSeconds: 2   # was 1
```

cadence 3s + timeout 2s 仍留 1s pacing 余量，不堵 probe 主路径。给 SHOW SLAVE STATUS 在 mariadbd busy 时多 1s 完成机会，把 timeout 越界率从 ~0.04% 压到接近 0。

### Patch 2（结构性，零 fork）：去掉 grep pipeline，用 shell 内建 case

```bash
secondary_replication_ready() {
  local slave_status
  if [ "${MARIADB_ROLEPROBE_SKIP_DB_READY:-}" = "true" ]; then
    return 0
  fi
  slave_status=$(query_slave_status)
  [ -n "${slave_status}" ] || return 1

  # 用 shell 内建匹配代替 pipeline + grep（0 子进程候选 orphan）
  case "${slave_status}" in
    *"Slave_IO_Running: Yes"*) ;;
    *) return 1 ;;
  esac
  case "${slave_status}" in
    *"Slave_SQL_Running: Yes"*) ;;
    *) return 1 ;;
  esac
  case "${slave_status}" in
    *"Last_IO_Errno: 0"*) ;;
    *) return 1 ;;
  esac
  case "${slave_status}" in
    *"Last_SQL_Errno: 0"*) ;;
    *) return 1 ;;
  esac
}
```

但 `query_slave_status` 命令替换里**还有 1 个远端 mariadb 调用**仍然可能被 SIGKILL 截断留 orphan。所以要配合 Patch 3。

### Patch 3（远端调用自带 timeout）

```bash
query_slave_status() {
  local mariadb_cli
  mariadb_cli=$(resolve_mariadb_cli) || return 1
  if command -v timeout >/dev/null 2>&1; then
    timeout 1 "${mariadb_cli}" "-u${MARIADB_ROOT_USER:-root}" \
      "${MARIADB_ROOT_PASSWORD:+-p${MARIADB_ROOT_PASSWORD}}" \
      -P3306 -h127.0.0.1 --connect-timeout=1 -e "SHOW SLAVE STATUS\\G" 2>/dev/null || true
  else
    "${mariadb_cli}" "-u${MARIADB_ROOT_USER:-root}" \
      "${MARIADB_ROOT_PASSWORD:+-p${MARIADB_ROOT_PASSWORD}}" \
      -P3306 -h127.0.0.1 --connect-timeout=1 -e "SHOW SLAVE STATUS\\G" 2>/dev/null || true
  fi
}
```

`timeout 1` 兜住远端调用 worst case，在 kbagent SIGKILL 之前 self-clean。

### 推荐组合

**Patch 1 + Patch 2** 一起做：放宽 timeout 给余量（防偶发越界）+ pipeline 改 case（即便仍越界 0 child orphan）。Patch 3 可选（双保险，但增加 `timeout` binary 依赖；busybox 自带 `timeout` 通常 OK）。

cadence 3s 不动（per #397 alpha.12 设计，`addon-test-host-stress-and-pollution-accumulation-guide.md` chapter 1 hero takeaway 已 ack）。

## 测试 artifacts

evidence package（持久化）：

| 阶段 | path（Jack 工作区） | sha256 |
|---|---|---|
| 2h freeze 第一包 | `…/mdb-zombie402-191623-task402-zombie-redline-sample_2h-211837.tar.gz` | `247d10cc...` |
| v2 补证（cmdline / kbagent log / live recheck）pre-cleanup | `…-v2-precleanup-212330.tar.gz` | `058e5fa6...` |
| cleanup 后 | `…-v2-cleaned-212430.tar.gz` | `abaa1f2d...` |
| final residual sweep | `…-final-residual-212536.tar.gz` | `4f8532ff...` |

cluster `mdb-zombie402-191623` 已按 [`addon-test-host-stress-and-pollution-accumulation-guide.md`](../../addon-test-host-stress-and-pollution-accumulation-guide.md) chapter 6 `preserve evidence ≠ preserve live cluster` 规则在 evidence freeze 后立即 cleanup。

## 关联

- 主文档：[`addon-probe-script-fork-and-zombie-guide.md`](../../addon-probe-script-fork-and-zombie-guide.md) Pattern B 章节
- 配合阅读：[`mariadb-fork-audit-pass-negative-case.md`](mariadb-fork-audit-pass-negative-case.md)（Pattern A audit pass）— 二者范围互补
- 测试环境累积压力主题：[`addon-test-host-stress-and-pollution-accumulation-guide.md`](../../addon-test-host-stress-and-pollution-accumulation-guide.md)（cluster cleanup 纪律部分）
- evidence-discipline：[`addon-evidence-discipline-guide.md`](../../addon-evidence-discipline-guide.md)（"audit framework 自身可能 under-fit" 反例）
