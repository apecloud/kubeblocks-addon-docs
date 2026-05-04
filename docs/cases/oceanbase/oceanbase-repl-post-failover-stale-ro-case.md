# OceanBase repl 拓扑：(primary + 1 standby) 同步 kill 后 standby 仍被 ro service 选中并返回过期数据案例

> **Audience**: OceanBase addon developer / tester；KubeBlocks roleProbe / sidecar (syncer / kbagent) 维护者
> **Status**: draft
> **Applies to**: OceanBase replication 拓扑（`oceanbase-rep-ocp-1.0.3`，3-pod primary + 2-standby + 4Gi request/pod）；C03 同时 kill primary pod + 1 standby pod chaos 场景
> **Applies to KB version**: KB v1.0.2 验证；KB roleProbe + ro service 选择行为跨版本演化待评估
> **Affected by version skew**: KB roleProbe + sidecar (syncer / kbagent) 实现演化

属于：[`addon-control-plane-election-guide.md`](../../addon-control-plane-election-guide.md) 的工程现场补充（roleProbe 健康口径 / role 标签 / ro service 端点选择）。Trap T 段同时是 [`addon-evidence-discipline-guide.md`](../../addon-evidence-discipline-guide.md) 反模式实证（早期错误假设被 evidence 反证）。

## Summary

C03 测试一条「post-failover 数据可见性」契约：当 primary pod 和 1 个 standby pod 被同时杀掉、survivor 接管为 new primary、被杀的 pod 恢复后，**对外暴露的 ro service 必须只选中跟 new primary 数据一致的 standby**。

实证发现这条契约被破坏：

- KubeBlocks `Cluster.status.phase` 和 `Component.status.phase` 都回到 `Running`；
- 被杀恢复的 pod-0 / pod-1 都被打上 `kubeblocks.io/role=standby`；
- ro service `kbselector` 选中 pod-0 / pod-1，给业务暴露 ro endpoint；
- 但这两个 standby 跟 new primary 之间已经发生 OB tenant log 时间线分叉，本地 redo log 在同一个 LSN 上和 new primary 不一致；
- `V$OB_LS_LOG_RESTORE_STATUS` 报 `STANDBY LOG NOT MATCH / -4070 / Log conflicts, the standby with the same LSN is different from the log source when fetch log`；
- log restore 卡住，standby 本地数据停在 chaos 之前的最后共同 LSN（这里只剩 3 行 seed），primary 已经写了几十条新数据，**ro 客户端长时间反复读到老数据，且没有任何 K8s/KB 层面的告警**。

这是 silent stale-RO，不是 commit-unknown，也不是 acked-write 丢失（这是 C09 的故障形态）。在我们这次的 evidence 中，rw 端始终是 source of truth；问题是 ro 端在「KB/K8s 全绿」的视角下持续返回过期数据。

## Topology boundary

本案专用于 `oceanbase-rep-ocp-1.0.3` 这条 **repl** 拓扑：

- 3 个 OB pod，每个 pod 跑 1 个 OBserver；
- 1 个 primary tenant + 2 个 standby tenant（不是 zone-级 Paxos quorum across 3 observers）；
- standby tenant 通过 OB 的 `LOG_RESTORE_SOURCE` 从 primary tenant 异步拉日志（语义类似 PG physical replication / MySQL replica）；
- syncer/kbagent 的 roleProbe 驱动 `kubeblocks.io/role` 标签；
- KubeBlocks 的 `rw` service / `ro` service 通过 `kbselector` + role label 路由流量。

> ⚠️ 把这套拓扑当成「3 副本 Paxos / N=3 observer 集群 → kill 2 ⇒ no-quorum」是错误模型。这个拓扑里 OB 没有跨 observer 的 quorum；任何「N-2 quorum loss / 旧 quorum lease 接受写入」的论证都不适用，参见下一节里的反证证据。

westonnnn 已澄清：repl 是 dev/test 拓扑，生产用 dist。本案不涵盖 dist 拓扑同款故障是否会出现；那部分需要在 dist D-4 production-capacity 集群上重新验证。

### 不适用范围

本案 evidence 不证明以下任何一项：

- OceanBase distribution 拓扑（`oceanbase-dist-ocp-1.0.x`）下同款 chaos 是否出现 stale-RO（dist 跨 OBserver 跑 Paxos quorum，故障语义跟本案不同；要 dist D-4 production-capacity 集群再验）；
- 生产 D-4 部署 / RootService 故障域 / zone-级 failure；
- repl 拓扑在 N=4+ 副本下故障形态相同（当前只有 1 primary + 2 standby 一种 layout 实证）。

环境依赖：3-pod repl topology, 4Gi request/pod, k3d Machine B 双节点；这些参数不影响 stale-RO 是否出现的结论，只影响复现速度和资源占用。

## Reproduction conditions

测试入口：

- 脚本：`kubeblocks-tests/oceanbase/tests/chaos-kill-primary-standby-quorum-loss.sh`
- chaos 动作：在持续唯一 id 写入的同时，**同步** delete 当前 primary pod **和** 1 个 standby pod；
- 留 1 个 standby pod 作为 survivor；
- 写入完成后等 cluster 自然恢复到 Running，然后跑 freshness gate；
- `KEEP_ON_FAIL=1` on failure，cluster 留供事后取证。

freshness gate 分类口径：

- `seed_missing`：chaos 之前的 3 行 seed 数据在 standby 上不到位（说明 setup 阶段 standby 还没拉完日志）；
- `ack_success`：客户端 insert rc=0；
- `ack_missing`：rc=0 写入在 final primary 上找不到（C09 故障形态，不是本案）；
- `k8s_ready_lt2_*`：K8s `Pod Ready` 数量 < 2 的窗口内的 attempt（用于检验旧 `no_quorum` 假设）；
- `ground_truth_unavailable`：rw primary 端不能给出可被 numeric 化的真值（保守降级，不强行结论）；
- `log_restore_stale`：standby `V$OB_LS_LOG_RESTORE_STATUS` 出现 `STANDBY LOG NOT MATCH` 或 ERR_CODE=`-4070` —— 主路径的 stale 命中信号；
- `count_match_3of3`：rw / ro 两侧 row count 三次连续读必须完全相等。

成功 = 所有上述 gate 都通过；命中 stale = `ro_stale_detected ≥ 1`，gate fail，整个 case fail。

## 排查路径

### 第一现场：cluster Running 但 ro 数据落后于 rw

C03 N=1 rerun2 跑完后（root summary `work/ob-test-supplement/20260504-1320-kill-primary-standby-N1-rerun2-clean`，archive sha `97003741b922f93d2748a5a15426040aedadd9912f45b25e36a285dd4bebebd1`，summary sha `7d3fe167157c4ac6b23cfb116b032f743f332b784b32801c150e7fa64d318221`），cluster 自然恢复到 Running，KB 状态全绿。

addendum read-only 复现命令（`work/ob-test-supplement/20260504-1320-kill-primary-standby-N1-rerun2-clean/evidence/readonly-addendum-20260504-1338`，archive sha `4702e1b85355ad666aa6c974da6a2a32a7168387c7ce83221b3023e117f6a428`，summary sha `d9a2573ed11100972917a2f19647f39ac2d2f1388a96d65a469acc47c75fe46c`）：

| Service | Selector | Endpoints |
|---|---|---|
| `...-oceanbase-rw` | `kubeblocks.io/role=primary` | `pod-2` 一个端点 |
| `...-oceanbase-ro` | `kubeblocks.io/role=standby` | `pod-0`、`pod-1` 两个端点 |

12 次连续 readback：

| Service | Iterations | Result（cnt min_id max_id range_1012_1032 has_9001） |
|---|---:|---|
| `rw` | 12/12 | `52 1 9001 21 1` |
| `ro` | 12/12 | `3 1 3 0 0` |

rw 看到 chaos 期间 21 条 attempt + 控制写入 9001，ro 一直只能看到 3 条 seed。

### 第二现场：被杀恢复的 pod 内部租户视角

per-pod `ob_chaos.quorumloss_t` count + id range：

| Pod | KB role label | Count | min..max id | has 9001 |
|---|---|---:|---|---:|
| pod-0 | standby | 3 | 1..3 | 0 |
| pod-1 | standby | 3 | 1..3 | 0 |
| pod-2 | primary | 52 | 1..9001 | 1 |

pod-0 / pod-1 的 `V$OB_LS_LOG_RESTORE_STATUS` 都给出统一的硬信号：

```
SYNC_STATUS = STANDBY LOG NOT MATCH
ERR_CODE    = -4070
COMMENT     = Log conflicts, the standby with the same LSN is different
              from the log source when fetch log
```

pod-2 这个视图没有行（它是 primary）。

`-4070` 是 OB engine 给「跟 log source 在同一 LSN 处发生 redo log 分叉」的硬错误码。OB 不会自动从分叉里恢复，需要 reseed（备份 + 增量）。换句话说：**只要被杀恢复的 standby 起回来时本地已经有跟 new primary 时间线冲突的尾部日志，OB engine 就不会让它继续做 standby**，但 KubeBlocks 不知道这一点。

### 第三现场：roleProbe 给 `standby` 标签的判据不够强

KubeBlocks 这条 sidecar 链路：

- syncer / kbagent 在每个 OB pod 上跑 roleProbe；
- 当前 roleProbe 实现读 `oceanbase.DBA_OB_TENANTS` 的 `TENANT_ROLE / STATUS / SWITCHOVER_STATUS`：
  - `TENANT_ROLE=PRIMARY, STATUS=NORMAL` → 报 PRIMARY；
  - `TENANT_ROLE=STANDBY, STATUS=NORMAL` → 报 STANDBY；
- KB controller 拿到 role 事件后给 pod 打 `kubeblocks.io/role=primary|standby` 标签；
- ro service 用 `kubeblocks.io/role=standby` 选中端点。

被杀恢复的 pod-0 / pod-1：

- `DBA_OB_TENANTS`：`TENANT_ROLE=STANDBY, STATUS=NORMAL`；
- `V$OB_LS_LOG_RESTORE_STATUS`：`STANDBY LOG NOT MATCH / -4070`。

→ roleProbe **看到 STATUS=NORMAL 就报 STANDBY**，对 log restore 健康一无所知；KB 按 role 事件给 standby 标签；ro 服务把它接进 endpoint。最后业务从 ro 端连接拿到的是「3 行 seed」这种过期数据，而 KB 层任何视图都看不到异常。

这一节的关键不是 `-4070` 本身（这是 OB 行为），而是 **roleProbe 的健康口径没把 log restore 状态算进 standby 资格**。

### 第四现场：harness 旧断言为什么把它误判成 no-quorum

C03 脚本 v0 把 K8s pod ready 数量 < 2 当成「OceanBase quorum loss」的代理变量。在 chaos 期间确实出现 ready_count<2 窗口（pod-0 / pod-1 在重启），同窗口里 survivor pod-2 仍然接收并 commit 了写入。脚本把这一段标记 `no_quorum_acked_writes` 并准备失败。

但前两节证据已经显示：repl 拓扑下 OB 不跨 OBserver 跑 quorum，那些 ack 写入的最终归宿是 final primary，**不存在「破坏 quorum 后仍允许写入再丢失」这条故障**。脚本旧断言把测试设计问题当成 product bug。

修正后 v4 脚本 (`tests/chaos-kill-primary-standby-quorum-loss.sh` sha `ce1204ab4ee889277b51d580003bd64915f8badc09b605967094191bb166cd3b`)：

- rename `no_quorum`→`k8s_ready_lt2`、`degraded_ready_window`→`k8s_ready_ge2`，仅作为现场观测 facet；
- 移除「ready_count<2 期间 ack 写入即失败」断言；
- 新增 `check_ro_freshness` gate（rw 端 ground truth → 每个 standby 看 `log_restore_stale` + `count_match_3of3`，rw / ro 三次连读必须完全一致）；
- 新增 setup 阶段 wait-for-replication 块（确保 standby 已经拉完 seed 才进入 chaos，避免 setup race 把 ground truth 拖成 unavailable）。

## 根因

**不是 OB engine bug**：`-4070 / STANDBY LOG NOT MATCH` 是 OB engine 在 standby 跟 log source 同 LSN 处冲突时的预期硬约束。OB 不会自愈，需要 reseed；这是已经发布的 engine 行为，不能要求 engine 在 redo log 分叉之后自动同步。

**也不是 KubeBlocks core bug**：KubeBlocks controller 忠实地按 sidecar 上报的 role 事件给 pod 标签、组装 ro/rw endpoint。它不知道 OB 内部 log restore 是否健康。

**真因：syncer / kbagent 的 OceanBase roleProbe 健康口径不完整**。它给一个 standby 节点打 `STANDBY` 的判据只看到 `DBA_OB_TENANTS.TENANT_ROLE / STATUS`，没看到 `V$OB_LS_LOG_RESTORE_STATUS.SYNC_STATUS / ERR_CODE`。在 redo log 分叉的 standby 上，前者是 `STANDBY/NORMAL`，后者是 `STANDBY LOG NOT MATCH / -4070`。当前的 roleProbe 把这个不健康的 standby 当作正常 standby 上报，KB 按这个 role 事件把 pod 接进 ro service，把过期数据暴露给业务。

具体到对外故障面：

- ro service 选中过期 standby，业务读到过期数据；
- KB / K8s 层任何监控视图都看不到异常（`Cluster.phase=Running`、`Pod Ready 4/4`、`role=standby`）；
- 业务侧只能通过自己的 freshness 校验（rw vs ro 数据 diff）才发现问题；
- standby 在没有人手工介入的情况下永远不会自愈（OB 需要 reseed），缺陷会无限期持续；
- 严重度 P1：rw 端始终是 source of truth，不丢已 ack 写入；问题是 silent stale-RO 视图。低于 C09（C09 是 ack 后 lost / divergence，P0）。

## 修复方向

### 1. 测试侧（已实装，本案 land 标记 P2）

`kubeblocks-tests/oceanbase/tests/chaos-kill-primary-standby-quorum-loss.sh` v4 已经把 freshness gate 实装，验证了 gate 在两条路径上都能正确给出 stale 结论：

| 路径 | archive sha256 | gate 触发口径 | classification 关键 |
|---|---|---|---|
| ground_truth_unavailable | `449b7d3491fc1c82d6787ec0178f0e1de884b5426a573ef8e4a12f46e486dd2f` | rw 端不能给 numeric 真值（这次是 setup race 导致 new primary 没有表，rw 查回 `ERROR 1146 Table doesn't exist`） | `ack_success=0, client_error=60, seed_missing=3` |
| log_restore_stale 主路径 | `da5ef49b8b831e2665cab0be7b8853227fb5a25e54bf3a4ab5ce17751fcec6bc` | pod-0/pod-1 都报 `LOG NOT MATCH/-4070`，rw=62 / ro=0/3 count 不一致 | `ack_success=58, ack_missing=0, seed_missing=0, ro_stale_detected=4` |

setup race 导致的 ground_truth_unavailable 路径之后被第二个 fix 关掉：在 chaos 之前增加 wait-for-replication 块，确保每个 standby 已经把 seed 拉到 `count=3` 才进入 victim selection。

```bash
# tests/chaos-kill-primary-standby-quorum-loss.sh L364-384 (v4)
section "C03 setup: wait standby replication catch up before chaos"
seed_replication_timeout="${TIMEOUT_SEED_REPLICATION:-60}"
for idx in $standby_indices; do
  deadline=$((SECONDS + seed_replication_timeout))
  standby_seed_count=""
  while [ $SECONDS -lt $deadline ]; do
    standby_seed_count=$(ob_exec_tenant "$CLUSTER" "$COMPONENT" "$idx" "root@${TENANT_NAME}" \
      "SELECT COUNT(*) FROM ob_chaos.quorumloss_t WHERE id IN (1,2,3);" 2>/dev/null \
      | tail -1 | tr -d ' \r\n')
    if [ "$standby_seed_count" = "3" ]; then
      pass "C03 setup standby idx=$idx replicated seed (count=3)"
      break
    fi
    sleep 1
  done
  if [ "$standby_seed_count" != "3" ]; then
    fail "C03 setup standby idx=$idx did not replicate seed within ${seed_replication_timeout}s (last count='${standby_seed_count}')"
    collect_evidence "C03-setup-standby-replication-$idx" "$CLUSTER" || true
    finish_with_summary
  fi
done
```

后续会从 inline 抽到 `lib/cluster.sh::wait_standby_replication`，给 C07 kill-standby-only 等其他用例复用。

测试侧只解决「我们看见这个 silent stale 现象」的问题，**不修 addon 本身**。它把 silent failure 变成 explicit failure，跟 ro 服务给业务的视角对齐。

### 2. addon 侧（P1，未实装）：roleProbe 把 `V$OB_LS_LOG_RESTORE_STATUS` 加到 STANDBY 健康判据

`apecloud/syncer/engines/oceanbase/get_replica_role.go` 的 `GetReplicaRole` 在判 STANDBY 之前增加：

- 查 `V$OB_LS_LOG_RESTORE_STATUS` 上每个数据 LS 的 `SYNC_STATUS / ERR_CODE`；
- 命中 `STANDBY LOG NOT MATCH` 或 `ERR_CODE` 落在 `-4070 / -4012 / -4013` 这一类「无法自愈的 log restore 错误」时，roleProbe **不报 STANDBY**；
- 选项 A：报 secondary-role-but-not-eligible（KB 不给 `kubeblocks.io/role=standby` 标签 → ro service 不选中 → 该 pod 被 KB 视为有问题，可触发 alert）；
- 选项 B：报 explicit unhealthy（`peerPrimaryErrors` 同款机制，KB 当作 unhealthy 处理）。

trace 字段建议：`logRestoreSyncStatus / logRestoreErrCode / decisionReason=standby_log_restore_not_match` 之类，参考 C09 guard fix 的 `peerPrimaryCheck / peerPrimaryConflict / peerPrimaryErrors / decisionReason=primary_peer_conflict_rejected` 模式。

预期安全性 trade-off：

- 修复前：被杀恢复的 standby 进 ro endpoint，业务 silent 拿到过期数据；
- 修复后：被杀恢复的 standby 不进 ro endpoint，ro 在重建期间可能没有可用 endpoint（业务 ro 读暂时不可用），但**绝不会返回过期数据**。
- 这跟 C09 guard 的逻辑是一致的：宁可短暂没有路径，不要把流量送到不一致的副本。

修复落地后须配套：

1. 单元测试 `engines/oceanbase/get_replica_role_test.go`：「current STANDBY rejected when log restore reports `-4070/LOG NOT MATCH`」、「current STANDBY accepted when log restore is clean」、`logRestoreSyncStatus / logRestoreErrCode` trace 字段验证；
2. C03 chaos 测试 N=10 在新 syncer 镜像上重跑，期望 `log_restore_stale FAIL` 永远不再触发（pod 根本不会被打 `standby`）；
3. C09 N=10 在新 syncer 镜像上回归一次，确认 peer-primary guard + log-restore guard 不会互相干扰；
4. 告警视角：KB cluster `Component.status` 应该把这种 "standby 起来了但 log restore 不健康" 变成 Component-level Warning，让运维侧能看见。

## Fix validation

### freshness gate 主路径正例

archive sha256: `da5ef49b8b831e2665cab0be7b8853227fb5a25e54bf3a4ab5ce17751fcec6bc`
root: `work/ob-test-supplement/20260504-1445-c03-stale-ro-gate-v2-n1`
chaos script: `tests/chaos-kill-primary-standby-quorum-loss.sh` sha `ce1204ab4ee889277b51d580003bd64915f8badc09b605967094191bb166cd3b`

ro-freshness-gate.tsv：

```
pod_idx  check               expected  actual                                                              verdict
0        log_restore_stale   (clean)   1002 1 10466615 ... STANDBY LOG NOT MATCH -4070 Log conflicts...   FAIL
0        count_match_3of3    62        0/3                                                                 FAIL
1        log_restore_stale   (clean)   1002 1 10466615 ... STANDBY LOG NOT MATCH -4070 Log conflicts...   FAIL
1        count_match_3of3    62        0/3                                                                 FAIL
```

classification-summary.txt：

```
attempts_total=60
k8s_ready_lt2_attempts=26
k8s_ready_lt2_ack_success=24
ack_success=58
ack_missing=0
client_error=2
unknown_after_client_error=0
confirmed_failed=2
seed_missing=0
post_recovery_control_write_id=9001
ro_stale_detected=4
```

期望对照：

- exit_code=1 ✅
- ro_stale_detected ≥ 1 ✅
- `log_restore_stale FAIL` 含 `LOG NOT MATCH/-4070` × 两个 standby ✅
- `count_match_3of3 FAIL` rw=62 / ro=0/3 ✅
- 最终 fail line `C03 stale-secondary-detected: 4 standby check(s) failed` ✅
- `seed_missing=0`、`ack_missing=0`、`unknown_after_client_error=0` ✅（排除 C09 类回归）
- `pass "C03 setup standby idx=N replicated seed (count=3)"` × 两个 standby ✅

PASS 21 / FAIL 1 / SKIP 0，唯一 FAIL 是 stale-secondary-detected，跟 TDD 设计一致。

### freshness gate ground_truth_unavailable 路径正例

archive sha256: `449b7d3491fc1c82d6787ec0178f0e1de884b5426a573ef8e4a12f46e486dd2f`
root: `work/ob-test-supplement/20260504-1432-c03-stale-ro-gate-v1-n1`

这次跑里 setup race（chaos 来得太早，standby 还没拉到 seed，survivor 接管成 new primary 后没有表）触发了 `ground_truth_unavailable` 短路保守路径。

ro-freshness-gate.tsv：

```
pod_idx  check                         expected   actual                                                                          verdict
rw       ground_truth_unavailable      (numeric)  empty ERROR 1146 (42S02): Table 'ob_chaos.quorumloss_t' doesn't exist  rc=1   FAIL
```

classification-summary.txt：

```
attempts_total=60
k8s_ready_lt2_attempts=28
k8s_ready_lt2_ack_success=0
ack_success=0
ack_missing=0
client_error=60
unknown_after_client_error=0
confirmed_failed=60
seed_missing=3
post_recovery_control_write_id=9001
ro_stale_detected=1
```

`seed_missing=3, ack_success=0, client_error=60` 这一组数据是 setup race 的诊断信号，不是真实 stale 现场。Gate 仍然 fail（exit!=0），不强行得出 stale 结论；这是设计意图——不能用一个不可信的 ground truth 反推 standby 的状态。

后续 v4 fix 把 setup race 关掉之后就不再走这条路径。

### 设计教训（actionable for 其他 addon 的 ro freshness gate）

把 v1 / v2 两条 archive 同时保留在主线，是因为它们一起锁住 freshness gate 的**两路设计契约**——这条契约可被其他 addon（PG / MySQL / MongoDB）的 ro 路由检查复用：

1. **主路径（gate v2）**：rw 端给出 numeric ground truth → 每个 standby 健康（这里 OB 是 `V$OB_LS_LOG_RESTORE_STATUS` clean）→ rw/ro 三次连读 row count 完全相等。任何一项 fail 即 `stale-secondary-detected`，整个 case fail（exit!=0）。
2. **保守路径（gate v1，`ground_truth_unavailable`）**：rw 端无法返回 numeric 真值（被杀的 primary 还没起来 / 被 setup race 拖到没表 / 业务表不可达）→ gate **不强行得出 standby 状态结论**，但 **必须 fail（exit!=0）**，绝对不能 silent-pass。

未来给其他 addon 写 ro freshness gate 时直接拍这两条契约即可。常见 anti-pattern 是「ground truth 拿不到 → silent skip → 漏掉真 stale」，C03 v2 之前的 `check_ro_freshness` 早期版本也踩过同款坑（rw 查询失败时静默 pass），后来才补上严格的 numeric 校验 + `ground_truth_unavailable` 显式 fail 路径。


### 仍未覆盖

- N=5 / N=10 多轮稳定性（gate v2 只跑 N=1，主路径必现，因此当前已足够锁住故障形态；syncer 修复完成后再跑 N=10 作回归）；
- C07 kill-standby-only N=1（不杀 primary，只杀单个 standby pod，预期标准是 standby 重建后 log restore 应该重新追上 primary，**不出现** `-4070`；脚本待写）；
- dist 拓扑 D-4 同款故障形态（dist 拓扑 OBserver 之间是 Paxos quorum，故障语义跟本案不同；需要 production-capacity 集群验证）；
- 在 syncer roleProbe 修复落地之前的 N=20 / 24h soak（修复前会一直 stale，soak 没意义）。

## Residual boundaries

本案不证明：

- OB 4.2 / 4.3 在 dist 拓扑下 (`oceanbase-dist-ocp-1.0.x`) 同样 chaos 不会出现 stale-RO；
- KubeBlocks 在其他 addon (PG / MySQL / MongoDB) 的 ro 路由上没有同款 silent stale 风险；
- repl 拓扑在 N=4+ 副本下故障形态相同（当前只有 1+2 一种 layout 实证）。

跨 addon 抽象（如「ro 服务的 freshness gate 必须做什么 / standby 健康判据应该至少包含什么」）暂不写入本案，保留给 `methodology/` 单独立篇。

## Trap T：「`ready_count < 2 ⇒ no_quorum` ⇒ 写入应被拒绝」假设被 evidence 反证

> 本节是 RCA 副线：记录 chaos 脚本 v0 的早期错误假设和反证 evidence。主线（现象 / 根因 / 修法）不依赖此节，跳读不丢主旋律。

C03 脚本最初版（v0）写了一条断言：「在 K8s `Pod Ready` 数量 < 2 期间任何 ack-success 写入都算 product bug（被理解为破坏 quorum 后仍接受写入）」。第一次跑出 finding 时按这条断言会把 evidence 标成「no-quorum acked-write divergence product bug」。

read-only addendum + gate v2 N=1 evidence 综合反证（见 archive `da5ef49b8b831e2665cab0be7b8853227fb5a25e54bf3a4ab5ce17751fcec6bc` 的 classification-summary）：

```
attempts_total=60
k8s_ready_lt2_attempts=26
k8s_ready_lt2_ack_success=24       ← 反证关键行
ack_success=58
ack_missing=0
post_recovery_control_write_id=9001
```

26 个 K8s `Pod Ready` 数量 < 2 的 attempt 中，survivor pod-2 接受了 24 次 ack-success，全部出现在 final primary 上，没有 ack_missing。结合 read-only addendum 在 final primary 上看到完整 chaos 期间写入 + 控制写入 9001：repl 拓扑下 OB 不跨 OBserver 跑 quorum，K8s pod ready 数量根本不是 OB 写入许可的代理变量。

修法已落地在 chaos 脚本 v4：

- `no_quorum`/`degraded_ready_window` 这种 label rename 成中性的 `k8s_ready_lt2`/`k8s_ready_ge2`，仅作为现场观测 facet，不再驱动失败断言；
- 真正的 product 失败信号必须从 OB 内部状态拿（`V$OB_LS_LOG_RESTORE_STATUS` / row count parity / final primary diff），而不是从 K8s pod ready 数量推。

可复用教训（Trap T 形式）：

- **凡是用 K8s 层信号（pod ready 数量 / endpoint 数量 / replica count）反推数据库内部 quorum/写许可的断言，都需要先验证这条因果链对当前数据库拓扑成立。** 不同数据库 / 同数据库不同拓扑（PG patroni vs streaming repl、OB repl vs dist、MySQL group replication vs source-replica）这条映射关系都不一样。
- 当 chaos 脚本里出现「K8s 层信号触发 product-bug 断言」这种逻辑时，要在脚本注释里显式写出**该数据库拓扑**下的 quorum / 写许可机制 reference，避免把 K8s 视角当成万能代理。

## 命题（reuse）

- KB 层的 `Cluster.phase=Running` / `Pod Ready 4/4` / `kubeblocks.io/role=standby` 都不能等价于 OB 层的「standby 副本数据可服务」。standby 数据可服务由 `V$OB_LS_LOG_RESTORE_STATUS.SYNC_STATUS=NORMAL` 加 `ERR_CODE` 不在错误集 (`-4070/-4012/-4013/...`) 共同决定，必须独立校验。
- OceanBase repl 拓扑在「primary 和 1 个 standby 同步 kill」这种事件序列下，被杀恢复的 standby 起回来后会跟 new primary 在同一 LSN 处发生 redo log 分叉，OB engine 报 `-4070 / STANDBY LOG NOT MATCH` 是预期硬约束，**不会自愈**，必须 reseed。任何修复方案都不能假设 standby 自己等一会就好。
- syncer/kbagent 的 roleProbe 给一个 standby 节点贴 `STANDBY` 标签的健康判据必须强于 `DBA_OB_TENANTS.TENANT_ROLE/STATUS`：要把 `V$OB_LS_LOG_RESTORE_STATUS` 也算进 health gate，否则 KB 路由层会把 stale standby 当成正常 standby 接进 ro endpoint。
- chaos / smoke 测试里凡是依赖「primary 拉完日志、standby 同步完成」的 setup 步骤，必须显式 wait standby 数据可见（这里是 `count=3` seed），不能信「primary 写入 commit 即 standby 已可读」。setup race 会把后续 gate 的 ground truth 拖成不可信，导致 gate 走保守路径而错过真实信号。
- 在 OceanBase repl 拓扑里，**Kubernetes Pod Ready 数量** 不是 OB 写入许可的代理变量。`ready_count<2 ⇒ no_quorum / 拒绝写入` 是错误模型；OB 在这条拓扑里不跨 OBserver 跑 quorum。任何 chaos 测试用 K8s pod ready 数量驱动「写入应被拒绝」断言都是测试设计问题。
- 验证 ro 服务正确性的最小 freshness gate 应该至少包含：rw 端给 numeric ground truth → 每个 standby `V$OB_LS_LOG_RESTORE_STATUS` clean → rw / ro 三次连续 row count match。任何一项 fail 就算 stale-secondary-detected，gate 必须 fail。`ground_truth_unavailable` 走保守 fail（不强行结论）这条路径要保留。
- 故障分级：**post-failover stale-RO** = P1（`Cluster.phase=Running` + ro 数据落后 primary，rw 仍是 source of truth，无 ack 丢失），**acked-write divergence (C09 类)** = P0（rw 自身丢已 ack 写入）。两者修复优先级和 trade-off 不同；不能合并叙事。

## Source evidence index

C03 原始 chaos run（read-only addendum 落地为 reframed-finding）：

- chaos rerun2 root: `work/ob-test-supplement/20260504-1320-kill-primary-standby-N1-rerun2-clean`
- chaos rerun2 archive sha256: `97003741b922f93d2748a5a15426040aedadd9912f45b25e36a285dd4bebebd1`
- chaos rerun2 summary sha256: `7d3fe167157c4ac6b23cfb116b032f743f332b784b32801c150e7fa64d318221`
- read-only addendum root: `work/ob-test-supplement/20260504-1320-kill-primary-standby-N1-rerun2-clean/evidence/readonly-addendum-20260504-1338`
- read-only addendum archive sha256: `4702e1b85355ad666aa6c974da6a2a32a7168387c7ce83221b3023e117f6a428`
- read-only addendum summary sha256: `d9a2573ed11100972917a2f19647f39ac2d2f1388a96d65a469acc47c75fe46c`

freshness gate 验证：

- gate v1 N=1 (ground_truth_unavailable path): root `work/ob-test-supplement/20260504-1432-c03-stale-ro-gate-v1-n1`, archive sha256 `449b7d3491fc1c82d6787ec0178f0e1de884b5426a573ef8e4a12f46e486dd2f`
- gate v2 N=1 (log_restore_stale main path): root `work/ob-test-supplement/20260504-1445-c03-stale-ro-gate-v2-n1`, archive sha256 `da5ef49b8b831e2665cab0be7b8853227fb5a25e54bf3a4ab5ce17751fcec6bc`
- chaos script with v4 freshness gate + wait-for-replication: `kubeblocks-tests/oceanbase/tests/chaos-kill-primary-standby-quorum-loss.sh` sha `ce1204ab4ee889277b51d580003bd64915f8badc09b605967094191bb166cd3b`
