# 案例：扩展测试强度（Expansion Strength）逼出真实 race —— 3 个 cross-engine canonical 数据点

> **Audience**: addon dev / TL / 测试工程师 在小样本验收已 PASS、被问"还要不要继续加测"时；写 ship-readiness 决策依据时；评估"维持现有测试集 vs 扩展正交维度"时
> **Status**: stable (案例本身已收口；3 个 finding 已经各自有 PR/contract fix)
> **Applies to**: 任意 addon（不绑定单一引擎）。该案例覆盖 Valkey + OceanBase + SQL Server 3 条线，反模式机制（small-sample-clean ≠ contract-correct）跨引擎跨场景适用。
> **Applies to KB version**: any（强度扩展是测试方法论层，跟 KB 版本无关）

属于：[`addon-test-baseline-standard-guide.md`](../../addon-test-baseline-standard-guide.md) 第 4.3 节 Expansion Strength Mandate (Rule 2 — normative) 的支撑案例。

## 背景

2026-05-01 → 2026-05-03 的 cross-addon 强度扩展波次中，3 条 addon 测试线在原始小样本验收**全 PASS / 全 clean** 的状态下，被强制按 Rule 2 沿正交维度扩展（density / concurrency / chaos overlay / restart window），结果在每条线都逼出 contract-level 的真实 race / 协议缺口。

如果停留在原始小样本验收口径，3 个 race 全部不会被发现，会带 latent bug 上船。

下文把 3 个 finding 并列展开，证明 doctrine "clean small-sample acceptance is insufficient evidence of contract correctness" 不是抽象规定，而是有硬 evidence 支撑的硬规则。

## Finding #1 —— Valkey RebuildInstance：49-sample clean → 4 个 contract gap

**Original baseline**: Valkey rebuild-ops Phase 6 acceptance 49 samples 全 PASS / 全 clean。按传统口径已经可以宣布 ready-to-ship。

**Extension axes added** (per Rule 2 mandate):
- Density: N=5 / N=10 / N=20 dense replay
- Concurrency: 同一 instance / claim 上并发 rebuild
- Chaos overlay: pod restart window during rebuild

**4 个 contract-level race / gap 浮出来**:

1. **Concurrent rebuild silent retry**：同一 instance/claim 并发 rebuild 不进 terminal 状态，进入 silent retry-until-cleared，违反 fatal contract。
2. **Failed OpsRequest 留 intent annotation 残骸**：失败的 OpsRequest 没清理 intent annotation，挡住后续同 instance/claim 的 rebuild。
3. **`getRestoredPV` fatal-on-bind-window**：tmp PVC bind window 期间 fatal（应当返回 NeedWaiting）；同时 `cleanupRebuildIntentOnFailure` 没对 InstanceSet hot annotation 做 retry-on-conflict。
4. **`getRestoredPV` label-discovery 静默失败**：tmp PVC cleanup 后 label-based discovery 静默失败；helper PV truth source 必须改用 intent annotation 上的 pvName，不能用 label 兜底。

**Closure**: PR #10191 follow-up commits `10dfc40af` / `2e129834a` / `514214ac9` / `b99890fbc`。每一条都加了 sentinel / typed-error contract + unit test pin contract + e2e final 240/0/0 across N=5+10+20 dense。

**Doctrine match**: 三层正交扩展（concurrent + dense + restart-window）每加一层逼出 1 个 contract gap，原始 49-sample clean 完全无能力暴露这 4 条。

## Finding #2 —— OceanBase C09：small-N clean → 700ms dual-primary acked-write divergence

**Original baseline**: OceanBase chaos test C09 (primary kill + role flip) 在小 N 下全 PASS，role 切换时间 SLA 内。

**Extension axes added** (per Rule 2 mandate):
- Density: N=10 / N=20 dense replay
- Chaos overlay: kill 配合 client-side acked-write 持续注入

**Race 浮出来**: ~700ms dual-primary 窗口期间，旧 primary 已经被 fence 但 acked-write 已返回 client；新 primary 接管后这一窗口的 acked-write 在 follower 侧出现 divergence。原始小样本既不密集到能撞窗口，也不带 acked-write 持续注入，看不到此协议缺口。

**Doctrine match**: 仅"reset → kill → 等切换 → assert role" 这套 small-N flow，无能力发现"切换瞬间 acked-write 协议正确性"这一独立维度。需要把 client-side acked-write 注入作为一根**新维度**叠上来。

## Finding #3 —— SQL Server CH50：small-N clean → commit-window commit-unknown ambiguity

**Original baseline**: SQL Server CH50 (commit-time failure injection) 在小 N 下未触发异常，应用层显示一致。

**Extension axes added** (per Rule 2 mandate):
- Density: N=10 dense replay
- Failure overlay: commit-window network partition / pod kill 精确注入

**Race 浮出来**: client 收到 commit-unknown（既未明确 ACK 也未明确 fail）的窗口下，server 侧 transaction outcome 实际已 commit，但 client 端如做"假定失败 + retry"将造成重复提交 / 计费类幂等性破坏。原始小样本不密集到能落进 commit window，看不到此 ambiguity。

**Doctrine match**: density × precise-failure-window 的乘积才能暴露 commit-window ambiguity；任一维度单独扩展都不够。

## 三个 finding 的共性

| 共性 | 体现 |
|---|---|
| 原始口径全 clean | Valkey 49/0/0、OceanBase small-N PASS、SQL Server small-N PASS。按传统 ship-ready 口径都可上船 |
| 扩展沿**正交维度** | 每条线增加的不是 N 重复，是新维度（concurrency / dense / chaos overlay / failure window / acked-write injection） |
| 浮出的 race 都是 **contract level** | 不是性能抖动 / 偶发抖动，是协议契约破坏（fatal vs NeedWaiting、acked-write divergence、commit ambiguity） |
| 修复都需要 contract change | 加 sentinel / typed error / source-of-truth 切换 / 协议补强；不是参数微调 |
| 闭环都依赖 dense replay 验证 | 修复后再回到 N=10/20 dense 下跑全绿，才允许 close |

## 给后续 addon 测试工程师的固化要求

按 `addon-test-baseline-standard-guide.md` 第 4.3 节 (Rule 2 — normative) 与 `addon-test-intensity-templates-guide.md` 5 模板，扩展强度是**强制要求**而非可选：

1. **不要把"small-sample 全 clean"当作 ship-readiness 终点**。它只是起点。
2. **沿正交维度扩展，不是 N 重复**。多扩一根维度（concurrency / density / failure overlay / 数据规模 / 24h soak），就多一次找到 contract gap 的机会。
3. **优先扩"目前完全没有覆盖"的维度**。已经覆盖了 N=10 dense？下一步是 concurrent。已经覆盖了 concurrent？下一步是 multi-chaos overlay。
4. **找到的 race 即使是窗口级 / 概率性，也必须按 contract gap 修复**。3 条 finding 全部最终走到 contract-level fix（sentinel / typed-error / 协议补强），不是"概率太低不修"。
5. **修复后必须回到 dense 验证**。Valkey 240/0/0 across N=5+10+20 才允许 close 是参考标准。

## 反模式：常见的"我不需要扩展"理由

| 反模式说法 | 反驳 |
|---|---|
| "原始 N 已经够了，我跑了 X 轮全 clean" | 3 个 finding 都满足"小样本全 clean"。clean ≠ contract correct |
| "扩展只是 N 重复，意义不大" | 错。Rule 2 mandate 要求沿**正交维度**扩展，不是 N 重复 |
| "暴露出来的概率很低，业务上影响不大" | Valkey 4 个 gap、OB 700ms、SQL Server commit window 都是低概率窗口，但都是 contract level，不修上船带 latent bug |
| "我们没人 / 时间 / 资源做扩展" | 这是排期问题不是是否要做的问题。`addon-test-intensity-templates-guide.md` 提供了 5 个 ready-to-use 模板降低落地成本 |
| "我跑了 24h soak 也没看到" | 24h soak 只是一根维度。需要 multi-chaos overlay / concurrent / data scale 等维度并行扩展 |

## Next expansion axes（Valkey line per TL 输入）

Valkey TL 已把下一波扩展轴登记在 `valkey/COVERAGE.md` 的 P0 backlog：

- multi-PVC instance rebuild（data + journal claim 同时）
- non-default ReclaimPolicy storage classes
- KB controller crash window 更早（before intent commit）

每加一根，都按 Rule 2 mandate 的"扩展即可能找到新 race"假设处理；如果跑出 0 个 finding，case 也要回写 doctrine（哪根维度该 retire / replace）。

## 索引

- 上层方法论文档：`addon-test-baseline-standard-guide.md` 第 4.3 节 Expansion Strength Mandate
- 强度模板手册：`addon-test-intensity-templates-guide.md`
- Valkey closure PR：`apecloud/kubeblocks` PR #10191 follow-up commits `10dfc40af` / `2e129834a` / `514214ac9` / `b99890fbc`
- 相关方法论案例：`evidence-inflation-in-csi-durability-debate-case.md`（小样本表态时不要做 narrative inflation）
