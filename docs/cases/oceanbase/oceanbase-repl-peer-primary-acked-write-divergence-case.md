# OceanBase repl 拓扑：transient peer-primary 接受 ack 写入后被 final primary 丢失（C09 acked-write divergence）

> **Audience**: OceanBase addon developer / tester；KubeBlocks roleProbe / sidecar (syncer / kbagent) 维护者
> **Status**: draft
> **Applies to**: OceanBase replication 拓扑（`oceanbase-rep-ocp-1.0.3`，2-pod primary + 1 standby + 4Gi request/pod）；C09 primary pod kill 期间持续唯一 id 写入的 chaos 场景
> **Applies to KB version**: KB v1.0.2 验证；KB roleProbe + 标签传播 + rw endpoint routing 行为跨版本演化待评估
> **Affected by version skew**: KB roleProbe + sidecar (syncer / kbagent) 实现演化

属于：[`addon-control-plane-election-guide.md`](../../addon-control-plane-election-guide.md) 的工程现场补充（roleProbe 健康口径 / role 标签 / rw service 端点选择 / peer-primary safety guard）。Trace v2 reproduction 段同时是 [`addon-evidence-discipline-guide.md`](../../addon-evidence-discipline-guide.md) 的 evidence-gap 闭环案例（埋点 → 复现 → 关闭 raw decision gap，不改 semantics）。

## Summary

C09 测试一条「primary kill recovery 期间数据持久化」契约：在 primary pod 被杀、replication 拓扑自然恢复期间，每一条客户端拿到 ack 的写入都必须出现在最终 primary 上。

原始 C09 N=1 与 trace-v2 reproduction 都违反了这条契约：

- 客户端 INSERT 返回 rc=0；
- 立即 readback 看到该行；
- 自然收敛后，最终 primary 上没有这条记录；
- 最终 standby 保留该记录，并伴随 `STANDBY LOG NOT MATCH / -4070 / Log conflicts / RAW_WRITE` 硬错误。

这是确定性的 acked-write divergence — 不是 commit-unknown，不是客户端误判。

repl 场景下当前接受的修复方向：syncer roleProbe 在自身被识别为 PRIMARY 时检查 peer，发现 peer 也是 semantic PRIMARY 就拒绝上报 PRIMARY。安全 trade-off：unsafe window 内出现短暂无 rw endpoint，客户端写入被拒绝/重试，避免 ack 后落到非最终 primary 链上。Guard v1 在 N=1 / N=5 / N=10 timeout-isolation 三轮验证下 `ack_missing=0`。

## Topology boundary

本案专用于 `repl` 拓扑：

- 2 个 OB pod（1 primary + 1 standby）；
- syncer / kbagent 的 roleProbe 驱动 `kubeblocks.io/role` 标签；
- KubeBlocks 的 rw service 通过 `kbselector + role label` 路由 primary endpoint。

westonnnn 已澄清：repl 是 dev/test 拓扑，生产用 dist。本案 evidence:

- C09 暴露的是真实的 roleProbe / routing safety 缺陷，对 repl 用户立即有意义；
- C09 fix validation 不能替代 dist 生产场景验证；
- dist D-4 等价场景（持续写入 + dist observer kill）目前在 Machine B 不可调度（生产 profile 需要 3 × 4Gi observer，2-node k3d 顶不下来），dist 验证状态为 `conditional / needs-production-capacity`。

### Out of scope

本案 evidence 不证明以下任何一项：

- OceanBase distribution 拓扑（`oceanbase-dist-ocp-1.0.x`）下同款 chaos 是否出现 acked-write divergence（dist OBserver 间是 Paxos quorum，故障语义跟 repl 不同；要 dist D-4 production-capacity 集群单独验）；
- 生产 D-4 部署 / RootService leader kill / RootService re-election / zone-级 failure；
- repl C10 / C05 deep-window / N=20 / 24h soak / multi-chaos overlay；
- 当前 guard 修复对 N>10 长程稳定性的保证（10 轮已稳定，更长 soak 待 production capacity）。

环境依赖：2-pod repl topology, 4Gi request/pod, k3d Machine B 双节点；这些参数不影响 acked-write divergence 是否出现的结论，只影响复现速度和资源占用。

## Reproduction conditions

测试入口：

- 脚本：`kubeblocks-tests/oceanbase/tests/chaos-writes-during-primary-kill.sh`
- chaos 动作：在持续唯一 id 写入的同时 delete 当前 primary pod；
- 不手动调用 `/tools/syncerctl getrole`；
- 不手动注入 roleProbe event；
- 不手动 patch role-label；
- `KEEP_ON_FAIL=1` on failure，cluster 留供事后取证。

freshness 与 divergence 分类口径：

- `ack_success`：客户端 INSERT rc=0；
- `immediate_readback`：刚 ack 的 row 立即 readback 命中；
- `ack_missing`：rc=0 写入在 final primary 上不存在 — 这是数据丢失断言；
- `client_error` / `confirmed_failed` / `unknown_after_client_error`：分开计数，不直接归为数据丢失。

成功 = `ack_missing=0`；命中 divergence = `ack_missing≥1`，整个 case fail。

## 排查路径

### 第一现场：原始 C09 N=1 — divergence 命中但缺 raw decision

Evidence root：

- `work/ob-test-supplement/20260503-2123-c09-writes-during-primary-kill-n1`（Mia archived bundle）
- root summary: `c09-root-evidence-summary.md`
- final-freeze archive sha256: `e3c0b833312ad1e0fd3f81b9e290b9263772b0573d574627227822a030dbc484`

Result：

```
PASS 15 / FAIL 1 / SKIP 0
attempts_total=60
ack_success=56
ack_missing=1
client_error=4
unknown_after_client_error=0
confirmed_failed=4
```

Missing write 现场：

- attempt `47`
- target pod: `ob-c09-writechaos-n1-oceanbase-0`
- 观察到的 K8s role label：`primary`
- 客户端 rc：`0`
- immediate readback：`1`

Final divergence：

- final primary pod-1 不含 id `47`；
- final standby pod-0 保留 `47 attempt-47`；
- pod-0 tenant：`STANDBY NORMAL`；
- pod-0 报 `STANDBY LOG NOT MATCH` + `-4070 Log conflicts` + `RAW_WRITE`；
- pod-1 tenant：`PRIMARY NORMAL` + `APPEND`。

Role-label 时序（KB controller 视角）：

- `13:29:38.351Z`: KB controller 接受 pod-0 `primary` event；
- `13:29:38.385Z`: KB controller 移除 pod-1 primary label；
- `13:29:38Z-13:29:39Z`: attempt 47 写到 pod-0，被 ack；
- `13:29:39.093Z`: KB controller 接受 pod-1 `primary` event；
- `13:29:39.120Z`: KB controller 移除 pod-0 primary label。

第一次跑剩下的 evidence gap：

- syncer roleProbe 的 raw SQL decision 当时没记录；
- failing 窗口前后的 endpoint history 没被 watcher 捕获。

### 第二现场：trace v2 reproduction — 关闭 raw decision gap

Trace v2 在 syncer 里加 raw decision logging（不改 semantics），并加了 200ms 粒度的 pod-label / endpoint watcher。

Evidence root：

- `work/ob-test-supplement/20260503-2235-c09-trace-v2-repro-n1/evidence/trace-v2-c09-acked-write-divergence-20260503-224553`
- archive sha256: `8729ddacb228a3580e894af09b8ecde7f9c91f1d660eba24c799449e4b66549f`
- summary sha256: `d1b232b2492195aa262fa4d6246f3c8a74bc085849a002424692524d9120efb2`

Result：

```
PASS 15 / FAIL 1 / SKIP 0
attempts_total=60
ack_success=57
ack_missing=1
client_error=3
unknown_after_client_error=0
confirmed_failed=3
```

Missing write 现场：

- attempt `44`；
- insert window: `14:42:09.906Z` ~ `14:42:10.014Z`；
- immediate readback: `14:42:10.145Z`；
- target pod: `ob-c09-trace-v2-n1-oceanbase-0`；
- target pod label: `primary`；
- endpoint membership: `rw`；
- `target_in_rw_endpoint=true`；
- rw endpoint resourceVersion at select: `776763`；
- 客户端 rc：`0`；immediate readback：`1`。

Controller + endpoint 时序：

- `14:42:09.589Z`: controller 处理 pod-1 `primary` event；
- `14:42:09.666Z`: controller 处理 pod-0 `primary` event；
- `14:42:09.723Z`: controller 移除 pod-1 primary label；
- `14:42:09.950Z`: rw endpoint 指向 pod-0；
- `14:42:11.451Z`: controller 处理 pod-0 `standby` event；
- `14:42:11.563Z`: controller 处理 pod-1 `primary` event；
- `14:42:11.727Z`: rw endpoint 指向 pod-1。

Raw decision trace（关闭第一次 gap 的关键证据）：

- pod-0 在 failing 窗口里上报 semantic PRIMARY：
  - `tenantRole=PRIMARY`
  - `tenantStatus=NORMAL`
  - `restoreRows=0`
  - `totalLS=2`
  - `appendModeLS=2`
  - `logStatDetail=1:LEADER:APPEND:YES,1001:LEADER:APPEND:YES`
  - `decisionRole=PRIMARY`
  - `decisionReason=tenant_role_accepted`
- 同窗口里 pod-1 也上报 semantic PRIMARY + APPEND。

Final divergence：

- final primary pod-1 缺行 `44`；
- final standby pod-0 保留 `44 attempt-44`；
- pod-0 报 `STANDBY LOG NOT MATCH` + `-4070` + `RAW_WRITE`；
- pod-1 报 `APPEND`。

这条 evidence 关闭了第一次跑剩下的 gap：**KB controller 不是凭空发明 primary label，它忠实跟随 roleProbe event**；事件本身在该窗口同时把两个 pod 报成 semantic PRIMARY + APPEND。

## 根因

Evidence-backed boundary：

- KubeBlocks controller 按 roleProbe event 给 pod 打互斥的 role 标签；
- failing 窗口里，syncer 的 raw decision 在两个 pod 上都给出 semantic PRIMARY + APPEND；
- KB controller 不知道哪条 OB log 链最终会成为 primary 链；
- 给 transient peer-primary 发 PRIMARY 标签 → rw endpoint 路由进去 → 客户端写入会落在最终成为 standby conflict 链的副本上。

修复方向（已实装并验证）：

- 安全 fix 落在 syncer roleProbe，**在它发出 PRIMARY event 之前**做 peer 检查；
- 当本 pod 是 semantic PRIMARY、peer 也是 semantic PRIMARY 时，**拒绝上报 PRIMARY**，logs `decisionReason=primary_peer_conflict_rejected`；
- 优先选择「短暂没有 rw endpoint」而不是「ack 后落到非最终 primary 链」。

非 goal / 本案不证明：

- 不主张 OceanBase kernel bug（OB 在 redo log 分叉时给 `-4070 / LOG NOT MATCH` 是预期硬约束，跟 [post-failover stale-RO 案例](./oceanbase-repl-post-failover-stale-ro-case.md) 同一引擎口径）；
- 不主张 KB core 单独有 bug（controller 忠实执行 roleProbe event）；
- 不主张 dist 拓扑同款故障形态成立（dist 跨 OBserver 跑 Paxos quorum，故障语义不同）。

故障分级：

- **acked-write divergence (C09 类) = P0** — rw 自身丢已 ack 写入；
- 跟 [post-failover stale-RO（C03 类）= P1](./oceanbase-repl-post-failover-stale-ro-case.md) 不能合并叙事：C03 是 ro 端 silent stale（rw 仍是 source of truth），C09 是 rw 自身丢数据。两者修复优先级、trade-off 不同。

## 修复行为

Guard v1 行为（已实装在 syncer roleProbe，跟 C03 案例里描述的「未实装的 log-restore guard」是同形 sidecar 安全口径，但目标不同：C03 guard 防 standby stale，C09 guard 防 transient peer-primary）：

1. 当前 pod 先查自身 semantic role；
2. 如果不是 PRIMARY，行为不变；
3. 如果是 semantic PRIMARY，syncer 查 peer；
4. 如果任一 peer 也是 semantic PRIMARY，syncer **拒绝**上报 PRIMARY，logs `decisionReason=primary_peer_conflict_rejected`；
5. KB controller 在该 conflict window 内不路由 rw endpoint。

预期安全性 trade-off：

- 修复前：unsafe window 内可以接受写入，最终 primary 漏掉这条记录；
- 修复后：unsafe window 内可能没有 PRIMARY / 没有 rw endpoint；客户端写入被分类为 `confirmed_failed`；**所有 ack 写入必须出现在 final primary 上**。

## Fix validation

### Guard v1 N=1

Root: `work/ob-test-supplement/20260503-2315-c09-peer-primary-guard-v1-n1`

Result：

```
PASS 16 / FAIL 0 / SKIP 0
attempts_total=60
ack_success=57
ack_missing=0
client_error=3
confirmed_failed=3
unknown_after_client_error=0
```

Guard evidence：

- 两个 pod 都打出 `primary_peer_conflict_rejected`；
- rw endpoint 在 unsafe transition 期间变空；
- attempts 7-9 失败，原因 `no-primary-role-label`，分类 `confirmed_failed`；
- 所有 ack 写入都出现在 final primary。

### Guard v1 N=5

Root: `work/ob-test-supplement/20260503-2325-c09-peer-primary-guard-v1-n5`
Archive sha256: `7228e094cf893452387ac1e86bff91a5a5f8120fcc6e1617d00f7b937d57b3bd`

Result：

- 5/5 cycles passed；
- 全部 cycle `ack_missing=0`；
- N=5 累计 client-ack 写入 288；
- 没有外层 kbagent `roleProbe timeout` event。

N=5 后剩余：

- peer probe timeout isolation 没闭环，因为 N=5 没遇到 timeout / deadline peer error。

### Guard timeout-isolation v1 N=10

Root: `work/ob-test-supplement/20260504-0005-c09-peer-primary-guard-timeout-v1-n10`
Archive sha256: `9e54e7695b692915a4a3e5f0bb63abe9f4806d0a2fa9af52561c6205c6d1ef9a`

Result：

- 10/10 cycles rc=0；
- 累计 client-ack 写入 570；
- 累计 `ack_missing=0`；
- 累计 client_error=30，全部 `confirmed_failed`；
- `unknown_after_client_error=0`；
- image gate：20/20 OB pod 镜像匹配预期 timeout-image；
- 当前 cycle 外层 `roleProbe timeout`：0；
- peer 实际 timeout / deadline 条目：cycle04 出现 2 条。

Timeout isolation 结论：

- 那 2 条 peer timeout / deadline error **保留在 `peerPrimaryErrors` 里**；
- 没有把整个 kbagent roleProbe deadline 吃掉；
- 当前 cycle 集群没有外层 `Role probe timeout` event。

Harness caveat（值得记录的踩坑）：

- 第一次包装跑在 cycle01 之后停了，因为一次未限范围的 grep 命中了 47h 前保留的 `ob-vfy-tp` YAML 中残留的 `Role probe timeout` 字样；
- cycle01 本身是干净的，已包含进 N=10 总结；
- 最终 N=10 summary 把 timeout 检测限定在「当前 cycle cluster / pod / event / log 名字」范围内。
- 教训：grep evidence 检测一定要 scope 到当前测试的资源 name，否则会被 namespace 内其他历史制品污染。这条教训跟资源 hygiene（清理历史 cluster）配合更安全。

### 设计教训（actionable for 其他 addon 的 sidecar role 安全口径）

把 Guard v1 + N=10 timeout isolation 的两条核心契约抽象出来，对 PG / MySQL / MongoDB 等其他 addon 的 sidecar role probe 同样适用：

1. **Peer-aware 才能给出 PRIMARY 标签**：单 pod 自查 `tenantRole=PRIMARY` 不够；必须查 peer，发现 peer 也是 semantic PRIMARY 时拒绝上报。短暂无 rw endpoint > ack 写入落到错误链。trace 字段建议：`peerPrimaryCheck / peerPrimaryConflict / peerPrimaryErrors / decisionReason=primary_peer_conflict_rejected`。
2. **Peer 探测错误必须封装在内层，不能吃掉外层 deadline**：peer probe 的 timeout / deadline 错误必须保留在 `peerPrimaryErrors` 字段里，不能让它们 propagate 成 outer kbagent `roleProbe timeout` event — 否则 sidecar 自身被 KB 当成 unhealthy 反而触发 cascading 状态变更。

跨 addon 反 anti-pattern：「把 sidecar 单 pod 视角等同于全集群视角」是同一类陷阱（也跟 C03 stale-RO 案例里 roleProbe 没看 `V$OB_LS_LOG_RESTORE_STATUS` 是同源），都属于 [`addon-control-plane-election-guide.md`](../../addon-control-plane-election-guide.md) 的 control-plane election 真值口径领域。

## Residual boundaries

本案不证明：

- dist 拓扑 D-4 production 同款故障是否成立；
- RootService leader kill / re-election 安全性；
- zone-级 failure；
- repl C10 / C05 deep-window 变种；
- N=20 / 24h soak / multi-chaos overlay 长程稳定性。

dist 状态：

- D-4 production profile 需要 3 × 4Gi observer pod；
- Machine B 按 request math 最多调度 2 个 4Gi OB pod；
- 因此 D-4 状态为 `conditional / needs-production-capacity`，不是 accepted production evidence。

后续推荐节奏：

- 短期：只做 D-4 / RootService runner 静态设计；
- 实跑：等到 production-capacity dist 环境再把 D-4 / RootService validation 作为 production evidence 跑完。

## 命题（reuse）

- KB 层的 `kubeblocks.io/role=primary` 标签 + rw endpoint 选中 ≠ 该 pod 的写入会出现在最终 primary 上。这条等价关系只在 syncer roleProbe 已经做了 peer-primary 互斥检查的前提下成立。任何 sidecar 上报 PRIMARY 之前不查 peer 的实现，都会在 transition window 把 ack 写入路由到错误链。
- syncer / kbagent 的 roleProbe 给一个 pod 贴 `PRIMARY` 标签的健康判据必须强于 `DBA_OB_TENANTS.TENANT_ROLE / STATUS` 单 pod 自查：必须把 peer 状态算进 health gate（peer 也是 semantic PRIMARY 时拒绝上报）。
- peer probe 的 timeout / deadline 错误必须封装在内层数据结构（`peerPrimaryErrors`），不能让它们 propagate 成 outer kbagent `roleProbe timeout`。否则 sidecar 自身会被 KB 当成 unhealthy 触发额外状态变更，反而放大故障面。
- chaos / smoke 测试里的 evidence-gap 闭环原则：第一次跑发现关键 raw decision 缺失 → 第二次跑加埋点（不改 semantics）+ 200ms 粒度 watcher → 复现 → 关闭 gap。trace v2 是这条工作流的具体实例。
- grep evidence 检测必须 scope 到当前测试的资源名（cluster / pod / event / log name），不能直接全 namespace 全 yaml 匹配 — 否则历史保留制品里的关键词会污染当前 cycle 的判定。这条教训跟资源 hygiene（清理 47h+ 旧 cluster）配合可显著降低 harness false 阳性。
- 故障分级：**acked-write divergence (C09 类) = P0**（rw 自身丢已 ack 写入），**post-failover stale-RO (C03 类) = P1**（rw 仍是 source of truth，ro 端 silent stale）。两者修复优先级和 trade-off 不同；不能合并叙事。

## Source evidence index

原始 C09 故障：

- root summary: `work/ob-test-supplement/20260503-2123-c09-writes-during-primary-kill-n1/c09-root-evidence-summary.md`（Mia archived bundle）
- final-freeze archive: `work/ob-test-supplement/20260503-2123-c09-writes-during-primary-kill-n1/c09-final-freeze-before-cleanup-20260503-222501.tar.gz`（archive sha256 `e3c0b833312ad1e0fd3f81b9e290b9263772b0573d574627227822a030dbc484`）

Trace v2 reproduction（关闭 raw decision evidence gap）：

- root: `work/ob-test-supplement/20260503-2235-c09-trace-v2-repro-n1/evidence/trace-v2-c09-acked-write-divergence-20260503-224553`
- summary: `SUMMARY.md`
- archive: `trace-v2-c09-acked-write-divergence-20260503-224553.tar.gz`（sha256 `8729ddacb228a3580e894af09b8ecde7f9c91f1d660eba24c799449e4b66549f`，summary sha256 `d1b232b2492195aa262fa4d6246f3c8a74bc085849a002424692524d9120efb2`）

Guard 验证：

- N=1: `work/ob-test-supplement/20260503-2315-c09-peer-primary-guard-v1-n1/c09-guard-v1-summary.md`
- N=5: `work/ob-test-supplement/20260503-2325-c09-peer-primary-guard-v1-n5/c09-guard-v1-n5-summary.md`（archive sha256 `7228e094cf893452387ac1e86bff91a5a5f8120fcc6e1617d00f7b937d57b3bd`）
- N=10 timeout isolation: `work/ob-test-supplement/20260504-0005-c09-peer-primary-guard-timeout-v1-n10/c09-timeout-guard-v1-n10-summary.md`（archive sha256 `9e54e7695b692915a4a3e5f0bb63abe9f4806d0a2fa9af52561c6205c6d1ef9a`）

Stage 3 chaos triad 收尾交叉引用：

- `work/ob-test-supplement/20260504-stage3-chaos-triad-summary.md`（C03 / C04 / C07 / C09 cross-reference summary，sha256 `f9c0d17f6efcbbdb2afe115122cd240d63198b8b50d710aa14e30099909315dd`）
