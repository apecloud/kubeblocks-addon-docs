# Oracle 12c post-switchover probe cascade kill：DBCA 杀容器 + 慢控制面 flap readiness

> **Audience**: addon dev / test
> **Status**: stable
> **Applies to**: Oracle 12c addon on KubeBlocks（方法论可复用，参见 [`addon-probe-timeout-and-soft-failure-guide.md`](../../addon-probe-timeout-and-soft-failure-guide.md)）
> **Applies to KB version**: 验证于 KB 1.0.3-beta.5（reconfigure_deep Run 1 / Run 2 / Run 3，2026-04-29 ~ 2026-04-30）
> **Affected by version skew**: probe 调度方式语义稳定（kbagent timeoutSeconds SIGKILL vs kubelet livenessProbe SIGTERM grace period）跨 KB 版本一致；初始 delay / period 字段位置可能随 ComponentDefinition schema 演进调整，遇到时按 [`addon-chart-vs-kb-schema-skew-diagnosis-guide.md`](../../addon-chart-vs-kb-schema-skew-diagnosis-guide.md) 流程对位

属 [`addon-probe-timeout-and-soft-failure-guide.md`](../../addon-probe-timeout-and-soft-failure-guide.md) 工程现场补充。

## 现场

- 时间：2026-04-29 ~ 2026-04-30 reconfigure_deep Run 1 / Run 2 / Run 3
- Cluster：`ora-rcd-47018`（Run 1，FAIL）/ `ora-rcd-53242`（Run 2，partial）/ `ora-rcd-60842`（Run 3，full PASS）
- 失败现象：
  - Run 1：cluster 进 Updating 死循环，`pod 3/4 Running` `oracle 容器 ready=false restarts=1`，无法继续
  - 关联现场：T21e（switchover→HScale 1→3）尾部 KB 1s requeue 55min+，oracle-0/1 (post-switchover) 反复 `kubectl describe` 看到 `Liveness probe failed: command timed out` / `Readiness probe failed: Data Guard status check failed`
- 失败链：probe 误杀 + readiness flap → InstanceSet `ReadyReplicas=1<3` → KB `RuntimeReady=false` → memberJoin 永远不被调用 → oracle-2 永远 "not joined" → 1s requeue 死循环

## 三类 root cause（同一类反模式的三个表现）

### Bug #11 — post-switchover ordinal hardcoding

不属于本 case 范畴（`post-switchover-hscale-primary-ordinal-hardcoding-case.md` TBD，单独 PR 跟进）。但**它是 Bug #12/#13 的前置触发**：T21e 测试链（switchover ord-0→ord-1 + HScale 1→3）只有在 ordinal 修对之后才能跑到「post-switchover 慢窗口」这一步。

### Bug #12 — DBCA 跑期间被 liveness kill

DBCA 总耗时 ~21min。期间 Oracle instance 会做多次「短停 + 改 spfile + 重启」的 reconfig 步骤（典型 90s 窗口）。原 cmpd：

```yaml
livenessProbe:
  initialDelaySeconds: 600   # 10min — DBCA 跑到一半就开始查
  timeoutSeconds: 10
  periodSeconds: 30
  failureThreshold: 3
  exec: { command: ["/scripts/liveness.sh"] }
```

原 `liveness.sh`：

```bash
status=$(sqlplus -S / as sysdba <<EOF
  select database_status from v\$instance;
  exit
EOF
)
[ "$status" = "ACTIVE" ] || [ "$status" = "MOUNTED" ] || exit 1
```

DBCA 中段，sqlplus 在 instance 短停的瞬间报 `ORA-1034: ORACLE not available` → status 空 → exit 1 → SIGTERM → exit 137 → 容器 restart → control files 状态被破坏 → death spiral。

### Bug #13 — post-switchover 慢控制面 flap readiness

Oracle 在 switchover、RMAN active duplicate、DG broker reconfigure 期间，`v$database` / `v$instance` / `dgmgrl` 控制面查询天然慢（实测 5-30s）。原 readiness 用 `timeoutSeconds: 10` + sqlplus 不带 timeout，慢窗口里被 SIGKILL → readiness fail → endpoint 摘出 → KB 看到 `ReadyReplicas` 下降。

## 5 件 Evidence Dig（撤销错误假设）

Oracle TL 第一次读代码后假设根因是「broker self-check 循环依赖」，基于纯代码推理。测试 owner 的 5 件 evidence dig 推翻这个假设：

1. **CmpD memberJoin lifecycleAction 定义** — `preCondition: RuntimeReady` 要求所有 replica 的 runtime ready
2. **Component.Status 没有 `membersStatus` 字段** — 说明 KB 还没进入 memberJoin 流程
3. **kbagent oracle-2 日志** — `roleProbe code=0 output="secondary"` 在 16:23:43Z 成功，但**之后无任何 memberJoin 执行记录**
4. **member_join.sh 逻辑无问题** — 脚本本身实现正确
5. **oracle-0 / oracle-1 状态**：
   - oracle-0 (post-switchover standby) `oracle 容器 ready=false restarts=3`，events 反复 `Liveness probe failed: command timed out`
   - oracle-1 (post-switchover primary) `ready=false restarts=0`，events 反复 `Readiness probe failed`

→ **真因不是 broker 循环依赖，是 oracle-0/1 not Ready 阻止 RuntimeReady → memberJoin 永远不被调用**。"oracle-2 not joined" 只是 symptom。

**这一段是 [`addon-evidence-discipline-guide.md`](../../addon-evidence-discipline-guide.md) 的活案例**：基于代码推理的 N=0 假设要被实际现场 evidence 证伪。

## 3 Layer Fix

### Layer 1 — cmpd.yaml probe 参数

```yaml
livenessProbe:
  initialDelaySeconds: 1800   # 30min，覆盖 DBCA 全程 + 50% 余量
  timeoutSeconds: 30          # 给 client 端 timeout 25s 留 5s 余量
  periodSeconds: 60           # ≥ timeoutSeconds，防 probe 进程并存
  failureThreshold: 5         # 一两次合法慢被吸收
readinessProbe:
  timeoutSeconds: 30
  periodSeconds: 30
  failureThreshold: 5
```

### Layer 2 — liveness.sh 软失败语义

```bash
# pgrep 守门：DBCA / setup_dg / orapwd 进行中跳过
if pgrep -f 'dbca' > /dev/null 2>&1; then exit 0; fi
if pgrep -f 'setup_dg_primary|setup_dg_secondary|orapwd' > /dev/null 2>&1; then exit 0; fi

# client 调用必须 timeout 包装
status=$(timeout 25 sqlplus -S / as sysdba 2>/dev/null <<EOF
  set pages 0 head off feedback off
  select database_status from v\$instance;
  exit
EOF
)
rc=$?

# rc!=0 = transient
if [ $rc -ne 0 ]; then
  echo "liveness: sqlplus unavailable (transient), skip"
  exit 0
fi

# 只有明确 bad state 才 exit 1
case "$status" in
  ACTIVE|MOUNTED) exit 0 ;;
  *) echo "Instance down (status=$status), liveness failed"; exit 1 ;;
esac
```

### Layer 3 — checkDBStatus.sh 软失败语义

```bash
# sqlplus 改 timeout + 软失败
out=$(timeout 25 sqlplus -S / as sysdba 2>/dev/null <<EOF
  select open_mode from v\$database;
  exit
EOF
)
[ $? -ne 0 ] && exit 0     # transient
[ -z "$out" ] && exit 0    # transient

# dgmgrl 改 best-effort：DG 控制面慢不阻塞 readiness
timeout 25 dgmgrl /@ORCLCDB_${ORD} 2>/dev/null <<EOF >/tmp/dg
  show database "ORCLCDB_${ORD}";
EOF
dg_rc=$?
if [ $dg_rc -ne 0 ]; then
  exit 0   # dgmgrl 超时/不可用 → DB 自己 OPEN/MOUNTED 即过 readiness
fi
# ORA-16532 (broker not configured) 也 exit 0
grep -q "ORA-16532" /tmp/dg && exit 0

# 仅 dgmgrl 成功返回 + 非 SUCCESS 才 fail
grep -q "SUCCESS" /tmp/dg && exit 0
echo "DG broker status not SUCCESS"
exit 1
```

## 实证

**Run 3（reconfigure_deep，cluster `ora-rcd-60842`）**：

```
PASS: 10  FAIL: 0  SKIP: 0
T22a: standalone cluster ready             ✓
T22a-1: rendered config snapshot           ✓
T22b: open_cursors=180 + session_cached=70 ✓
T22c: unknown param rejected               ✓
T22d: processes=10 → Failed (Bug #16)      ✓
T22e: processes=222 + survives pod kill    ✓
T22f: session_cached_cursors=88            ✓
T22g: cluster deleted                      ✓
```

**容器层硬证据**（DBCA 全程 + reconfigure 全场景）：

```
NAME                     READY   STATUS    RESTARTS
ora-rcd-60842-oracle-0   4/4     Running   0
  config-manager  restartCount=0
  exporter        restartCount=0
  kbagent         restartCount=0
  oracle          restartCount=0
```

`restarts=0` 是这套 fix 的唯一可信生效信号（参见主指南规则 7）。

## 教训

1. **基于代码推理的假设必须被现场 evidence 证伪**，不能在没看 pod 状态/事件之前就下结论
2. **Probe 失败的 first blocker 一定要先排"信道层 vs 产品层"**，不要直接归因到 DB
3. **cmpd 改 probe 参数 + 改脚本必须同步**，否则 helm upgrade 把脚本回滚但参数留下，半边失效
4. **md5 不能单独作为 fix 验证依据**：kubectl jsonpath 输出与 ConfigMap 投影到 pod 内文件存在 trailing-newline 差异；要配合内容 grep（pgrep / timeout 行 sentinel）做内容证伪
5. **helm upgrade 在 ConfigMap/CmpD 上的 field-ownership conflict 是 silent skip**：保留 kubectl-patched 状态。这次恰好对我们有利（保住了 Bug #12/#13 fix），但反例（漏 patch → 拿到旧 helm template）必须警觉，新 cluster 启动前要 verify CmpD 与 scripts 实际值
