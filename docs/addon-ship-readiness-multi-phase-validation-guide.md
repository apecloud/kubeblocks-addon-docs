# Addon Ship-Readiness 多阶段验证矩阵指南

> **Audience**: addon TL / test 工程师，需要决定 addon 何时可以发布
> **Status**: stable
> **Applies to**: any KB addon（ship-readiness gate 是引擎无关的发布决策方法论）
> **Applies to KB version**: any（验证矩阵框架不依赖具体 KB 版本；引擎特化 N 计数 / flake 阈值由 owner 在案例附录补）
> **Affected by version skew**: 不受 KB 版本影响 — 三段矩阵 + Wilson CI + 二段 ship 判定与 KB 版本演化解耦

本文面向 Addon 技术负责人 / 测试工程师，回答一个不太好定义的问题：**一个 addon 在什么时候能被标记为可以发布？** 不是"测试都跑过一次"，也不是"chaos 跑了一晚上"，而是有一个分阶段、可累积证据、可量化 flake 复现率的验证矩阵。

正文是引擎无关的方法论，引擎相关结果放在案例附录。

## 先用白话理解这篇文档

新人 ship 一个 addon 时，常用直觉是：

- 跑一遍 smoke：通过。
- 跑一晚上 chaos：通过。
- ship。

这套直觉问题在于：

1. **"通过一次"会把偶发 transient 当成稳定信号**。一次 PASS 不能把"这条路径稳定"和"这次 timing 正好"分开。
2. **chaos 跑得再多，如果不区分阶段，最后只能给"过/不过"二元结论**，没法回答 "R07 这条 case 大概多大概率会 fail"。
3. **product fail 与 known caveat 没分开统计**，盘子一大，很难判断哪一个真的需要 block ship。

→ 真正的方法论是：把 ship-readiness 验证拆成**三段独立的 phase**，每段有自己的目的；累积所有 phase 的样本一起算 flake 复现率；按"产品级 fail = 0" + "已知 caveat 已 documented" 的二段判定来决定 ship。

### 何时本文方法论 apply

| 场景 | 关键决策 |
|---|---|
| 准备 ship 一个新 addon 或一个 addon 大改动 | 跑完三段，再做最终决定 |
| 已经 chaos 跑了几晚上，但 R07 / 某条 case 偶发 fail | 用本文方法论扩 N，量化复现率到 CI |
| 团队问 "再跑几轮才稳" | 用 Wilson CI 计算 N=多少能 narrow CI 到目标 |
| ship 前评审会上同事问"这个 addon 真的可以发布了吗" | 直接按矩阵 + 复现率 + caveat 列表给定量回答 |

### 读完你能做什么决策

- 设计 ship-readiness 计划时，能 5 分钟内画出三段 phase 矩阵 + 每段轮次。
- 看到一次 transient fail 时，不会立刻 panic 也不会忽略 — 会把它进一步扩 N 量化。
- 写 ship recommendation 时，能给出 "ship / 暂缓 / 阻塞" 三类明确结论 + 数值依据。

## 适用场景

- 一个 addon 第一次准备发布。
- 一次大版本更新（新增 Ops 能力、控制器逻辑改动、引擎跨大版本）。
- 之前 ship 过、但被告诉"再跑几轮看看 stability"。
- 某条 case 反复 transient fail，要量化复现率给上下游。

## 三段验证矩阵的核心结构

| 阶段 | 目的 | 期望终态 |
|---|---|---|
| Phase 1 baseline | 验"全链路是否能跑通"，覆盖度优先 | `0 product fail`，known caveat 列出 |
| Phase 2 chaos × N | 暴露并发 / 故障 / 抖动场景下的产品级 bug | `0 product fail`，N 越大越好 |
| Phase 3 regression × N | 量化偶发 transient 的复现率 + Phase 2 没覆盖的 case | `0 product fail`，flake 复现率有 CI |

三段的样本是**累积**的 — 计算 flake 复现率时把所有 phase 的相关 round 合并算 N，不要按 phase 单独算。

### Phase 1：baseline

- 跑一次完整的 smoke / e2e suite，覆盖所有声明的能力点。
- 期望：所有 case 能跑、有清晰的 PASS / FAIL / SKIP 计数、所有 product fail 都有 root cause。
- known caveat（环境限制 / 依赖外部 infra / capability 缺失）此阶段就要列出，写进后续报告，不要带到 Phase 2。

不要在 Phase 1 跑 1 round 就算过 — 至少 round 3-5 来过滤掉 setup 抖动。

### Phase 2：chaos × N

- 在 Phase 1 baseline 上加 chaos：kill master、删 replica、网络抖动、控制器 crash、storage 抽风等。
- 每一轮起 fresh cluster（不要复用），跑完即清。
- N 至少 10。

Phase 2 的目的不是"跑出 fail"，是"让 timing-sensitive 路径暴露 product fail"。如果 N=10 全 PASS，可以说 chaos 路径下的产品稳定性高；如果 N=10 出现一两次 transient，进入 Phase 3 量化。

### Phase 3：regression × N

- 在 Phase 1 + Phase 2 的基础上，针对 Phase 2 暴露的可疑 case 或希望量化的 case，单独跑 regression suite × N 轮。
- N 至少 10，配合历史 baseline 的 round 数累积，最好达到 N ≥ 15 总样本。
- 每轮也是 fresh cluster + 清零。

Phase 3 输出的是 flake 复现率 + 95% Wilson CI，配合 ship 阈值给出定量结论。

## flake 复现率计算与 CI

### 累积样本

把所有相关 round 加起来：

```
N = baseline_round_count + phase2_chaos_round_count + phase3_regression_round_count
fail_count = 该条 case 在所有上述 round 中 fail 的次数
rate = fail_count / N
```

不要按 phase 单独算 — 单独算时 N 太小，CI 太宽，结论不可信。

### 95% Wilson CI

用 Wilson score interval（不要用 normal approximation，N 小时 normal 会给出负数下界）。常用的查表近似：

| N | fail | rate | Wilson 95% CI |
|---|---|---|---|
| 5 | 1 | 20% | 1.0% - 62.4% |
| 10 | 1 | 10% | 1.8% - 40.4% |
| 15 | 1 | 6.67% | 1.2% - 29.8% |
| 20 | 1 | 5.0% | 0.9% - 23.6% |
| 30 | 1 | 3.33% | 0.6% - 16.7% |

观察：从 N=5 到 N=15，CI 上界从 62% 降到 30%；继续扩到 N=30，上界降到 17%。建议：

- **N=15 是一个性价比高的目标**：能把 1 fail 的 CI 上界压到 30% 以下。
- 如果 N=15 时 CI 上界 > ship 阈值，再扩到 N=30。
- 如果 N=10 时已经多次 fail（>2），不要继续盲扩 N，先 root cause。

### ship 阈值

阈值取决于具体 case 的影响：

| 影响 | 建议阈值 |
|---|---|
| 数据丢失 / 写失败 | 0%（任何 fail 都阻塞） |
| 服务不可用 > 1 min | < 5% |
| transient 短暂行为不一致（client retry 即恢复） | < 30% |
| 仅日志或事件层 noise | 不阻塞 |

**判定规则**：CI 上界 ≤ 阈值 → ship；CI 上界 > 阈值 → 继续 root cause 或扩 N；CI 完全在阈值之上 → block。

## ship 决策的二段判定

最终决定是否 ship 时，按这个顺序：

### 第一段：产品级 fail 必须为 0

把所有 phase 累计的 fail 分类：

- **产品 fail**：实际 addon 代码 / 控制器逻辑 / 业务路径错。**必须为 0** 才能 ship。
- **环境 fail**：CSI / kubelet / 网络等基础设施缺陷。如果可定位到非 addon scope，挂 caveat 不阻塞。
- **测试 fail**：测试脚本本身的 bug 或 timing 问题。修测试 / 挂 caveat。
- **transient fail**：偶发 timing-sensitive，需要 Phase 3 量化复现率。按阈值规则。

### 第二段：known caveat 都已 documented

每条 caveat 必须有：

- 明确的触发条件。
- 影响评估（是否数据丢失 / 是否需要人工介入）。
- 处理路径（绕过方法 / 修复计划）。
- 写进 release notes。

只有"产品 fail = 0" + "caveat 全 document" 两段都满足，才发 ship recommendation。

## 验证报告固定收集的项

发 ship-readiness final report 时建议固定收：

1. 三段 phase 的轮次与 case 数（per round + 累计）。
2. 每段累计的 product fail / non-product fail 计数。
3. 重点 case 的复现率（fail / N）+ Wilson 95% CI。
4. CSI / 资源 drift 全程量化（每轮起止 + 累计）。
5. cluster cleanup residue（zero-residue 是 immediate-cleanup 的强证据）。
6. ship-blocker 列表 + 状态（开 / 关）。
7. known caveat 列表 + 处理路径。
8. ship recommendation：ship / 暂缓 / 阻塞 + 量化依据。
9. follow-up 项（doc 沉淀 / 上游 PR / 跨 addon 模板化）。

## 常见误判

### 把 N=5 的 0 fail 当稳定信号

N=5 全 PASS 时，**实际 95% CI 上界在 ~52%**（Rule of Three 的精确版）。说 "稳定" 时心里要有这个 CI。N 越小，"全 PASS" 也越无意义。

### 把 1/5 当 20% 复现率然后立即修控制器

1/5 = 20% 看上去高，但 95% CI 下界只有 ~1%；可能它的真实复现率就 < 5%，timing 偏一下就跑出来一个 fail。**先扩 N，不要在 N=5 时就开始 patch 代码** — 否则 patch 是基于一次偶发 timing 的猜测。

### Phase 2 全 PASS 就跳过 Phase 3

Phase 2 没暴露 fail 不代表所有 transient 都消失。Phase 3 的目的是显式量化 transient 复现率给 ship recommendation 用。即使 Phase 2 N=10 全 PASS，仍建议 Phase 3 跑 N=10 把累积 N 推到 ≥20。

### 把环境 fail 当产品 fail

环境层（CSI 抽风、kubelet stress、k3d 抖动）的 fail 不该阻塞 ship。但要写进 caveat 并提供绕过方法。混在产品 fail 里会过度阻塞 ship，也让团队失去信心。

### 跨多次 cycle 时丢失累积 N

如果第一次 ship-readiness 跑了 N=10，过了一段时间又跑了 N=10，**两批样本应该合并算 N=20**（前提是 addon 代码 + 控制器版本一致）。不要每次重新从 0 开始计 N — 那等于浪费历史证据。

## 与其他主题的关系

- 本文给出多阶段验证的矩阵；具体 fail 分类与 first blocker 分层见 [`docs/addon-test-acceptance-and-first-blocker-guide.md`](addon-test-acceptance-and-first-blocker-guide.md)。
- 单次探针 fail 落到正确层的方法见 [`docs/addon-test-probe-classification-guide.md`](addon-test-probe-classification-guide.md)。
- 控制层故障如何编排进 chaos 矩阵见 [`docs/addon-controller-crash-resilience-guide.md`](addon-controller-crash-resilience-guide.md)。
- 测试 runner 的环境前置见 [`docs/addon-test-environment-gate-hygiene-guide.md`](addon-test-environment-gate-hygiene-guide.md)。

## 案例附录：Valkey ship-readiness 三段验证

Valkey addon 在 KB 1.2 的 ship-readiness 走了完整三段验证。本附录保留具体数值，但不绑定具体 OpsRequest 名字。

### 三段成绩

| Phase | 内容 | 轮次 | 累计 case | product fail |
|---|---|---|---|---|
| Phase 1 | full-suite baseline | Round 3-7（5 round） | ~3700 | 0 |
| Phase 2 | chaos × 10 | R1-R10 | 1430 | 0 |
| Phase 3 | regression × 10 | R1-R10 | 1120 | 0 |

**三段累计 0 产品级故障。**

### R07 transient 复现率收敛

R07（scale-in 时 master kill 与 hot backup 重叠的窄时间窗）在 Phase 1 Round 5 出现一次 transient read empty。

- 初始 N=5：1/5 = 20%（Wilson CI 1.0% - 62.4%，CI 太宽，不能下结论）
- 扩到 N=15：1/15 = 6.67%（Wilson CI 1.2% - 29.8%）

阈值取 30%（transient + client retry 即恢复，影响低）。最终 CI 上界 29.8% < 30%，**判定为 known caveat 不阻塞 ship**。

### 集群健康量化

跨 Phase 1+2+3 共 25 round 的 CSI restartCount delta = 0。chapter 6 immediate-cleanup 口径在 24h+ 实际 cluster create-tear 循环里 validate 通过。

### ship recommendation

满足 "产品 fail = 0" + "caveat 全 document"，发 ship。

### 测试 artifacts

- Phase 3 final root：`/Users/wei/.slock/agents/.../valkey-phase3-regression-x10-091219/final/review-summary.md`
- 验证环境：k3d `kb-local`，KubeBlocks 1.2，Valkey addon rev 57

## 案例附录：Valkey RebuildInstance race fix — 五层验证矩阵 (PR #10191)

apecloud/kubeblocks PR #10191 的 e2e 验证用了一个比"baseline / chaos / regression"三段更细的五层矩阵。这个矩阵的 specific 价值是它在 Phase 6 acceptance 之外又加了两层"扩展强度"专门用来逼出 contract 层的并发缺口；最终在四个 follow-up commit 里关闭了 4 类没在 baseline N=10 里浮出来的 race。本附录给具体数和命令。

### 五层结构和数

| 层 | 内容 | 样本数 | product fail |
|---|---|---|---|
| 独立 acceptance | 全新 cluster + 30 baseline keys + slave disruption + RebuildInstance + data verify + topology verify | 10/10 PASS | 0 |
| 独立扩展 | 同形 acceptance 再 4 trial 拉宽 flake-detection 样本 | 4/4 PASS | 0 |
| 同 cluster 密集（initial） | 5 + 10 + 20 三组 rotating-slave RebuildInstance ops，no recreate between ops | 35/35 PASS | 0 |
| controller-restart chaos gate | 在 intent annotation `present → absent` AND pod `Ready=False` 同时成立时刻 `kubectl rollout restart` kb-controller | 1/1 PASS | 0 |
| 同 cluster 密集（follow-up after 4 fixes） | 三组 rotating-slave RebuildInstance ops 重跑 + 并发 OpsRequest O02 端到端 | 35/35 PASS（最终 N=20 final tally `PASS 240 / FAIL 0 / SKIP 0`） | 0 |

**五层累计 84+ valid sample，全程 0 个 `PersistentVolume "" not found`、0 个 source PVC 最终绑到非 helper PV、0 个 cleanup conflict、0 个 stale intent residue。**

### 扩展强度兑现的 4 个 contract gap

baseline 49-sample（前三层）clean。但加了第四层 chaos gate 和第五层 dense follow-up 之后，逼出 4 类原本看不见的 contract 缺陷，每一个都直接 land 一个 fix commit：

1. 同 instance/claim 并发 RebuildInstance 不 terminal — fix `10dfc40af`（`ErrRebuildIntentOwnedByDifferentOps` sentinel + handler `intctrlutil.NewFatalError` wrap）。
2. 失败 OpsRequest 留 intent residue 挡住后续 rebuild — fix `2e129834a`（lenient `CleanupRebuildIntentByOpsUID` + `cleanupRebuildIntentOnFailure` hook）。
3. tmp PVC 还没 bind 时 `getRestoredPV` fatal、cleanup 撞 InstanceSet hot-annotation conflict 不 retry — fix `514214ac9`（`NewErrorf(ErrorTypeNeedWaiting,...)` + `retry.RetryOnConflict`）。
4. tmp PVC cleanup 之后 helperPV label discovery 沉默失败 — fix `b99890fbc`（`getOrDiscoverHelperPV` 把 intent annotation pvName 当 truth）。

在 49-sample baseline 已 clean 的前提下，第四+五层每多 push 一档 strength（chaos / 并发 / 同-cluster 高密度），就抓出一类没浮出来的 contract gap。这也是本文 ship 决策"二段判定"的直接反例：满足"产品 fail = 0"还不够，**必须把"扩展强度"作为一个独立的 normative 维度**，否则 baseline-only ship 等于把 race 遗留到生产。

### Trial-loop wrapper（可复现）

```bash
NS=valkey-rebuild-race-r1
ART=artifacts/rebuild-race-fix-r1
mkdir -p "$ART"
for i in 1 2 3 4 5 6 7 8 9 10; do
  echo "=== Trial $i ==="
  ./tests/rebuild.sh "$NS" "$CLUSTER-$i" 2>&1 | tee "$ART/trial-$i.log"
  rc=${PIPESTATUS[0]}
  if [ $rc -ne 0 ]; then
    echo "FAIL on trial $i — stopping for forensics, namespace preserved"
    break
  fi
done
```

第 10 个 trial 内嵌 controller-restart chaos gate（触发依据是 IS annotation transition + pod Ready=False，命令见 `addon-controller-crash-resilience-guide.md` 案例附录）。

### Forensics grep（验收硬条件）

```bash
grep -E 'PersistentVolume "" not found|owned by ops|empty HelperPVName' \
  "$ART"/trial-*.log "$ART"/controller.log 2>/dev/null | wc -l
# 预期 0
```

```bash
kubectl -n "$NS" get pvc -o yaml | yq '.items[] | {name: .metadata.name, vol: .spec.volumeName}'
# 每个 source PVC 的 volumeName 都应该指向 helper PV（带 rebuild-from / rebuild-tmp-pvc 标记），而不是 default-provisioned 名字
```

```bash
kubectl get pv -o yaml | yq '.items[]
  | select(.metadata.labels."operations.kubeblocks.io/rebuild-tmp-pvc")
  | {name: .metadata.name, claimRef: .spec.claimRef, reclaim: .spec.persistentVolumeReclaimPolicy}'
# helper PV reclaim policy 必须等于推断出来的原值（这次 lane = local-path 默认 Delete）
```

### 测试 artifacts

- 完整 e2e harness：`apecloud/kubeblocks-tests/valkey/tests/rebuild-ops.sh`（commit `18817ec`），含 O01-O04 strength lane。
- 完整 evidence bundle：`artifacts/valkey-phase6-rebuild-fix-20260503-2023/`（baseline + chaos gate）+ `artifacts/valkey-rebuild-ops-20260503-2312-n20/`（follow-up 240/0/0 final tally）。
- 关联 fix PR：apecloud/kubeblocks#10191（关闭 issue #10190 的 12 类 contract gap，4 类是这五层矩阵的最后两层逼出来的）。
- 关联 self-audit：`apecloud/kubeblocks-tests/valkey/COVERAGE.md`（同 commit `18817ec`），列了 P0/P1/P2 扩展 backlog。

## 相关主题

- [`docs/addon-test-acceptance-and-first-blocker-guide.md`](addon-test-acceptance-and-first-blocker-guide.md)
- [`docs/addon-test-probe-classification-guide.md`](addon-test-probe-classification-guide.md)
- [`docs/addon-controller-crash-resilience-guide.md`](addon-controller-crash-resilience-guide.md)
- [`docs/addon-pvc-rebind-via-workload-intent-guide.md`](addon-pvc-rebind-via-workload-intent-guide.md) — 第四+五层 chaos / dense-follow-up 触发的 contract 体系本身。
- [`docs/addon-design-contract-review-during-xp-guide.md`](addon-design-contract-review-during-xp-guide.md) — XP review-阶段抓到的另外 8 类 contract gap，与本附录 4 类合起来是同一个 fix 周期里的 12 类总样本。
- [`docs/addon-evidence-discipline-guide.md`](addon-evidence-discipline-guide.md)
