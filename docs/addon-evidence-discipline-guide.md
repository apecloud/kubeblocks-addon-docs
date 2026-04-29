# Addon Evidence Discipline — 不要把动机假设、N=1 样本、间接旁证升格为硬结论

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

## 5. 与 `addon-bounded-eventual-convergence-guide.md` 的关系

外向 bounded retry 是"对外部异步系统的观察要多次 sample"。
内向 evidence discipline 是"对自己产出的结论要多 round challenge / 自我 review"。

二者一致：**对单次 snapshot 的不信任**。区别在 subject 不同 — 一个是被观察对象，一个是观察者本身。

## 6. 案例附录索引

具体反例放在 `docs/cases/<engine>/` 或 `docs/cases/methodology/` 下，作为这篇通用方法论的实证补充。当前已有：

- `docs/cases/methodology/evidence-inflation-in-csi-durability-debate-case.md`（待写：04-28 MariaDB Round 2-replacement debate 中的 3 次推过头与 3 次撤回）

## 7. 给后续 addon 工程师的固化要求

1. 任何含有"平均""通常""每隔""主因"的结论 → 先数 N
2. 任何含有"已证明""系统性""客观地" 的结论 → 先确认是直接观察还是推理
3. 任何含有"机器在退化""产品在变慢""hot-fix 修不对" 的结论 → 先排除"我有动机假设"
4. 给决策者的 DM / thread 必须附 evidence 强度自评
5. 被挑战时先承认再重述，不 hedge
6. 反例固化进 skills，不要重复踩
7. **诚实说"数据不足"比推一个能用的故事更专业**
