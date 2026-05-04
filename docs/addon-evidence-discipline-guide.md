# Addon Evidence Discipline — 不要把动机假设、N=1 样本、间接旁证升格为硬结论

> **Audience**: addon dev / test / TL
> **Status**: stable
> **Applies to**: any KB addon
> **Applies to KB version**: any (methodology, version-agnostic)

属于：方法论主题文档（不绑定单一引擎）。

## 1. 这篇要解决的问题

我们已经在 `addon-bounded-eventual-convergence-guide.md` 里强调过"对外部系统观测要 bounded retry，不要单次 snapshot 当结论"。这篇是它的内向版本：**对自己产出的结论也要做"bounded retry"**，而不是把第一个出现的解释、第一个 N=1 数据点、第一个间接旁证当作最终判断推给协作者或决策者。

最常踩的反模式有三类：

1. **Motivated narrative inflation**：脑子里有一个解释方向（比如"机器在退化""hot-fix 没修对""产品在变慢"），就用观察数据去 fit 这个 narrative，而不是反过来让数据决定 narrative。
2. **N=1 → "average / frequency / trend" inflation**：手上只有一个或两个数据点，描述时却用"平均""通常""每隔 X 分钟"这类需要分布建模才能用的词。
3. **Indirect evidence → systemic verdict inflation**：把一个间接旁证（自己 kubectl timeout、单次 grep miss、单次 tail empty）当成"已系统性证伪"。

这三类反例的共同后果都是：**协作者必须花一轮 challenge round 才能把你拉回到证据强度匹配的描述**，浪费决策时间，也降低你下一次结论的可信度。

## 2. 三条硬规则

### 规则 A — 描述的强度必须匹配证据的强度

**口径对应表**（任何外发结论都要先过这张表）：

| 证据强度 | 允许的描述形式 | 不允许的描述形式 |
|---|---|---|
| N=1 的现象 | "出现过一次"、"目前唯一一个观测样本" | "平均"、"通常"、"每隔" |
| N=2 但 variance 巨大 | "两个样本差距 N 个量级，分布未知" | "平均"、"会"、"倾向于" |
| 间接旁证（自己探测失败 / 副作用观察 / 推断） | "旁证、不升格"、"提示某个方向值得查" | "系统性证伪"、"已证明"、"客观上是 X" |
| 主因/单因 narrative | "目前最可能的因果是 X，仍有 Y 没排除" | "主因 X"、"就是 X" |
| 双因合流（A + B 一起触发） | "A 和 B 共同触发" | 二选一表述 |

如果你正在写 "平均 / 通常 / 每隔 / 是 X 导致 / 主因是" 之类的词，**先回头数样本**。N<3 而 variance 巨大时，这些词不能用。

> **Layer-aware 应用**：本规则的 layer 化版本（按 first-blocker 5 层拆分什么强度证据对应什么强度描述，如"环境层 N=1 即合规 vs 产品层 N=1 是 inflation"）见 [`addon-test-acceptance-and-first-blocker-guide.md`](addon-test-acceptance-and-first-blocker-guide.md) §Layer-aware evidence 强度对应表。底座规则（本节三规则）不变；layer-aware 应用是 single-source 在那一篇，本 doc 仅 anchor 反链。

### 规则 B — 区分"我观察到 A → A 是事实"与"A 可能是 X 的证据"

观察事实是直接的：pod RESTARTS=3、`/readyz` 返回 timeout、文件不存在。这些是硬数据。

但**你 kubectl 连续 timeout 不直接等于"集群已经 X"**。它可能是：
- 集群 API 真挂了（目标想要的解释）
- 你的 kubecontext 配错了
- 你的本机网络抖动
- 你的 agent 自己出口堵了
- 你打的请求恰好命中正在 rolling 的 controller

只有当你把"间接观察"和"其他可能解释"都列一遍并能逐个 rule out 时，旁证才能升格成硬证。否则就只是"指向某个方向值得查"。

### 规则 C — Reach for "数据不足以下结论" 比 "推一个结论凑合用" 更专业

很多场景下，"我们目前 N=2，variance 巨大，要给 durability 一个数字需要先 reset 环境再跑 N≥3" 才是诚实的回答。

把这种诚实留给自己说出来，而不是等 reviewer 拦你。

实践上意味着：
- 给 reviewer / 决策者的 DM 里，永远附一段 **"数据强度自评"**：N、variance、有没有 controlled 实验、有没有 confounder
- 不确定时显式标 "**目前没有数据可以分辨 X 和 Y，需要 ZZZ 才能进一步验证**"
- 你的推荐方向不应该建立在你不严谨的 narrative 上 —— 应该建立在那些**不论 narrative 怎么改都仍然成立的事实**上

例：在本案例里，即便没法证明 "hot-fix 平均 15min 失效"，B + A1 推荐仍然成立，因为它建立在"variance 巨大 + 现状累积态没法靠再做 hot-fix 回到干净起点"这两条事实上，跟具体的 durability 分布无关。

## 3. 自检清单（外发结论前过一遍）

每条结论外发前自问：

1. **N 是几？** N=1 的话，我是不是用了 average/通常/每隔/趋势？
2. **variance 多大？** 如果 variance 跨数量级，分布建模需要的 N 比直觉大很多。
3. **直接还是间接？** 我说的"已证明"是建立在直接观察上，还是几步推理后？
4. **我有动机假设吗？** 我是不是已经有一个故事，正在用数据 fit 这个故事？
5. **如果这条结论错了，我的推荐还成立吗？** 如果不成立，说明推荐建立在脆弱基础上 — 重写。
6. **协作者最可能怎么挑战这条？** 如果挑战会让我立刻撤回，先撤再发。

## 4. 退路：当被挑战时怎么处理

被挑战是好事 — 说明协作者在做你应该自己做的 evidence discipline。处理路径：

1. **先承认**：用清晰的"撤回 / 修正 / 接受"开头，不要 hedge
2. **重述真严谨的版本**：N、variance、source quality 老老实实写出来
3. **检查其他依赖**：被撤回的 narrative 可能在你别处的话术里也用过（DM、thread、memory），一起更新
4. **固化反例**：把这次挑战写进 skills（就是这篇）或 work-log，避免下次重蹈

## 5. Harness 修复时机：dirty → clean 三段式 timeline

前面四节面向「自己产出的结论」的 evidence discipline。这一节面向另一类 case：**evidence pack 是用一个 harness（runner / sampler / asserter）跑出来的，跑完才发现 harness 本身有 bug**。这种情况下 evidence pack 的可信度取决于 bug 是在哪个时间点被发现的。

总规则一句话（**central rule**）：

> **pre-sample harness fix is free; post-sample / pre-promote fix requires reject-and-rerun; post-promote fix requires invalidate-and-re-promote.**

把 evidence cycle 按 sample / promotion 两条边界切三段：

### Phase 1 — 还没产出任何样本时发现 harness deficit（pre-sample）

允许直接修 harness、加 preflight gate、改 axis 谓词；evidence pack 还没产出，所以不存在"被污染的下游"。

操作要求：

- harness 修法 ship 在同一份 evidence cycle 的 commit / artifact 里，不分仓发
- 该 cycle 的 run summary 要 explicit 记 "harness 修了什么 → 为什么不污染本 cycle 评估"，避免下次回看时 reviewer 误以为 harness fix 是"在 evidence 之后偷偷发生"
- 不需要废任何之前已 promote 的 evidence

判定信号：runner 仍在 preflight / setup / dry-run 阶段，sampler 还没产出任何 tally / 样本行 / artifact。

**边界守则（容易越界）**：如果 setup / preflight 已经在 cluster / disk / KB 控制面上留下被后续 sample 复用的状态（例如已经创建了 cluster、PV、SC，或者已经写入 controller 用来排序的 annotation），harness 修法可能会切掉那条 state 链，那就**不再是 Phase 1**，按 Phase 2 走。换句话说：Phase 1 的边界不是「sample tally 行数为 0」，而是「harness state 改完，前面已经发生的所有副作用仍然能被原 sample assumption 安全使用」。这两件事不一样，被混淆时往往会把本应 reject 的 pack 错放到 Phase 1 自由修。

### Phase 2 — 已经跑完 / 已经有 sample，但还没 promote 到对外 canonical claim 时发现 harness deficit

reject 当前 pack（不要 promote），修 harness，rerun。**不要在原 pack 上做事后补丁来"挽救"已跑出来的 sample** — sample 的 provenance 是它跑出来时的 harness state，事后修 harness 改不了那一份 sample 的来源。

操作要求：

- 当前 pack 在 run summary / artifact 里明确 mark `pre_acceptance / not counted`，不是悄悄丢弃
- harness 修法 ship 在同一仓 commit，rerun 用修后的 harness 跑同 axis
- 必须有一段 "为什么这个 pack 不可信 → 修了什么 → rerun 怎么验证" 的 dirty→clean trajectory 记录在 run summary 里。即使后来 clean rerun 的 tally 看上去跟 dirty pack 几乎一样，dirty pack 的存在和 reject 理由也必须留下来；reviewer 后回看时能直接看到这条 trajectory，不需要从 chat 历史里复原
- 如果 harness deficit 只影响某些 axis（比如 single-axis 调试路径专用的 silent diff），这个边界要在 summary 里写清楚，避免误把所有 axis 一起废
- rerun 要在跟原 pack 相同的环境（image SHA、cluster 版本、storage class）下做，不在 rerun 阶段再混入其它环境变更

判定信号：sample 已经存在但还没出现在 PR body / external report / canonical claim 里。

### Phase 3 — 已经 promote 到对外 canonical claim 时才发现 harness deficit

最严苛的一段。不能简单"修 harness 然后 rerun"了事，要按下面顺序：

1. **明确 invalidate scope**：先列出当前 canonical claim 里有哪些数字 / 论断是依赖这个 harness 的，逐项标记。harness deficit 不一定让所有依赖 evidence 都失效，但**只有在能 evidence-by-evidence 排除污染时**才能保留某条；不能排除的全部 invalidate。

   **artifacted audit 标准**（pollution exclusion 必须可被独立验证，不能依赖 chat 解释或主观判断）：可接受的排除证据形式包括 — (a) 原 artifact 中的 raw run log / run summary 显示该 deficit 触发条件未被命中（例如对应 axis 配置在该 run 没有走 zero-count 路径）；(b) 受影响的 code path 在当时的 harness state 下不会被执行（git blame / runner config 可证）；(c) 即使 deficit 触发，原 sample 的 acceptance invariant 已被独立 assertion 覆盖（例如另一个 grep predicate / 另一条 sampler 列）；(d) 重新解析原始 artifact 得到与原 promote 数字相同的结论。**不能从 artifact 重建结论的，一律 invalidate；以"我记得当时...""根据 chat 上 X 说...""按理推断..."为依据的排除一律不接受**
2. **公开撤回受影响的论断**：在原 promote 渠道（PR body / report / DM）显式撤回对应数字，不在静默状态下修改。撤回文字要包含 "harness deficit X，发现于 Y 时间，影响 Z 论断"
3. **rerun 边界覆盖原承诺强度**：rerun 必须达到原 promote 时的 N、覆盖维度、acceptance gate；**不能用"修了 harness、跑了一次过了"代替原 multi-N evidence**
4. **rerun completion 后 re-promote**，并把 dirty→clean 三段式 trajectory（原 evidence 强度 / harness deficit / invalidate scope / rerun result）加入新版 canonical claim 的 footnote 段，让外部 reviewer 不需要 dig 历史就能验证 trust recovery

操作要求：

- run summary / canonical claim 的 footnote 必须保留 dirty→clean trajectory，**不能只显示 clean 后的最终 tally**。透明地承担这次 invalidate 比"看上去从来没出过事"更重要
- 如果 harness deficit 的影响范围跨过多个 promote 周期 / 多个 PR / 多个外发结论，每一个独立 promote 都要走自己的 invalidate / rerun / re-promote 流程
- 如果跑出来 rerun 不能复现原样本（即原 promote 的数字本来是 harness silent bug 的产物），也要老老实实更新 canonical claim 而不是"假装当时是另一种 N"

判定信号：canonical claim 已经被外部 reviewer / 协作者引用过，或者已经进入 PR body / report 等长期可见的位置。

### 怎么决定一条 harness deficit 落在哪个 phase

按 promotion event 切：

- 若 promote 还没发生 → Phase 1（如果 sample 也还没产出）或 Phase 2（如果 sample 已产出）
- 若 promote 已发生 → Phase 3，无论 deficit 看起来多小

**不要按 deficit 严重程度切 phase**。"我觉得这条 deficit 不会影响结论" 在 Phase 2/3 不是 free pass — 只有 evidence-by-evidence 的污染排除才算数，主观判断不算。

### 自检清单（cycle 内任何 harness 修改前过一遍）

1. 这个 cycle 当前在哪个 phase（pre-sample / pre-promote / post-promote）？
2. 这条 harness 修改是否会让原 sample 的 provenance 失效？
3. 如果是 Phase 2/3，dirty→clean trajectory 在哪里被记下来？
4. 如果是 Phase 3，invalidate scope 的列表 / 撤回文字 / rerun 计划全到位了吗？
5. summary / artifact 在没有任何额外 chat 历史的情况下，能不能让外部 reviewer 重建这条 timeline？

## 6. 与 `addon-bounded-eventual-convergence-guide.md` 的关系

外向 bounded retry 是"对外部异步系统的观察要多次 sample"。
内向 evidence discipline 是"对自己产出的结论要多 round challenge / 自我 review"。

二者一致：**对单次 snapshot 的不信任**。区别在 subject 不同 — 一个是被观察对象，一个是观察者本身。

## 7. 案例附录索引

具体反例放在 `docs/cases/<engine>/` 或 `docs/cases/methodology/` 下，作为这篇通用方法论的实证补充。当前已有：

- [`docs/cases/methodology/evidence-inflation-in-csi-durability-debate-case.md`](cases/methodology/evidence-inflation-in-csi-durability-debate-case.md) — 04-28 MariaDB #396 CSI durability 讨论中的 3 次 inflation 与 3 次撤回（动机 narrative / N=1→average / 间接旁证→系统性证伪）

Phase 2 dirty → clean trajectory 的实证（§5 配套案例）：

下面是同一个 evidence cycle（Valkey RebuildInstance race fix 验证，`apecloud/kubeblocks` PR #10191）里**三条互相独立**的 harness deficit，每一条都在 pre-promote 阶段被识别、按 Phase 2 处理（reject pack → 修 harness → rerun）。三条 deficit 之间没有因果链：

**Deficit A — O06 dirty pack 抓到 stale retained rebuild PV 残留**

- 触发：O06 axis N=5 跑完得到 `121/0/0`，但 namespace cleanup 阶段抓到上一轮 iteration 留下的 retained rebuild PV
- 处理：pack 标 `pre_acceptance / not counted`，加 O00 stale rebuild-PV preflight（每轮 iteration 起手 fail-fast 任何残留）+ per-rebuild pre/post source-PVC PV reclaim-policy equality gate
- rerun：clean N=5 重跑得到 `121/0/0` 才接受。dirty → clean trajectory 写入 PR body 第七层 evidence 附录

**Deficit B — O02 contract violation 被 record 为 FAIL 但 suite 不 fail-stop**

- 触发：O07 v3 N=1 全轴跑出来 `PASS 82 / FAIL 0 / SKIP 0` 之前的某一次 N=1 跑里，O02 出现一例 both-Succeed 样本。harness 把它 record 成 FAIL 但没 fail-stop，让 suite 继续往下跑
- 这是 **harness discipline 缺陷**：contract 违反不应 silent passthrough。改成 fail-fatal
- 处理：当前 pack 标 `pre_acceptance / not counted`；同 image 跑 O02-only clean rebaseline 一次 — 得到 `concurrent-a=Failed, concurrent-b=Succeed` PASS，contract 守住；audit 之前已 promote 的 dense N=5/10/20 + Retain + O06 artifact，所有的 O02 终态都是 exactly-one-success（artifacted audit 通过 §5 Phase 3 的 (a) 标准），判定 prior tally 不被这条 deficit 污染
- 结论：B 是 **harness fail-stop 缺陷**，不是 product regression；当时那一次 both-Succeed 是 one-shot 样本，不可重现

**Deficit C — BSD `seq 1 0` 让 zero-count loop 跑了两次**

- 触发：在 B 的 O02-only focused rerun 阶段，把其它轴关掉用 `REBUILD_OPS_DENSE_N=0` 等，跑完发现 O02 自己 PASS 但其后的 O03 axis 仍然失败 — O03 本应被 N=0 跳过
- 根因：runner 用 `for i in $(seq 1 $N)` 跑 axis loop。macOS BSD `seq 1 0` 输出 `1 0`，loop 跑两次（i=1, i=0）；O03 体没有二次 `[ "$N" -gt 0 ]` 守卫，于是 silent stray iteration（详见 [`addon-test-runner-portability-guide.md`](addon-test-runner-portability-guide.md) 坑 7）
- 处理：所有 `for i in $(seq 1 $N)` 加外层 guard 或转 C-style for；rerun 覆盖该 axis；同 cycle 内 audit 之前已 promote 的 dense / Retain / O06 artifact，全部是全轴模式，从未走 zero-count 调试路径，artifacted audit 判定不被污染
- 这是 **portability 缺陷**，跟 A、B 都没有因果关系：C 不是 B 的根因（B 那次 both-Succeed 是 full-axis N=1 跑出来的，O02 在 runner 里是 unconditional 不受 N=0 控制），也不是 A 的根因（A 是 PV cleanup 残留，跟 axis 计数无关）

三条 deficit 同 cycle 内被发现、修复、独立 rerun 验证，最终 ship 在 `kubeblocks-tests@a2c96cf`（A）+ `kubeblocks-tests@2fdcda5`（B + C）。**两个 commit 不构成 dependency 链**，只是 cycle 节奏上 A 先于 B 先于 C 暴露而已；任意一条单独发生都按 Phase 2 处理就够。

## 8. 给后续 addon 工程师的固化要求

1. 任何含有"平均""通常""每隔""主因"的结论 → 先数 N
2. 任何含有"已证明""系统性""客观地" 的结论 → 先确认是直接观察还是推理
3. 任何含有"机器在退化""产品在变慢""hot-fix 修不对" 的结论 → 先排除"我有动机假设"
4. 给决策者的 DM / thread 必须附 evidence 强度自评
5. 被挑战时先承认再重述，不 hedge
6. 反例固化进 skills，不要重复踩
7. **诚实说"数据不足"比推一个能用的故事更专业**
8. **harness 修法跟 evidence cycle 的 sample / promotion 边界对齐**：pre-sample 可直接修；post-sample / pre-promote 必须 reject 当前 pack、记录 dirty→clean trajectory、rerun 后才能 promote；post-promote 必须按 §5 Phase 3 走 invalidate / 撤回 / rerun / re-promote，不能事后偷偷修 harness。
