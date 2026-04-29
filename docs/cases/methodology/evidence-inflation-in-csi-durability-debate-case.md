# 案例：MariaDB #396 CSI durability 讨论中的 3 次 evidence inflation 与 3 次撤回

属于：[`addon-evidence-discipline-guide.md`](../../addon-evidence-discipline-guide.md) 方法论案例。

## 背景

2026-04-28 下午到晚上跑 MariaDB async smoke "再启动几轮全量测试" 的稳定性验证（任务 #396）。Round 1 PASS=102 FAIL=5 (C6/C8 role flicker)，Round 2 (`mdb-vc396r2-183556`) 在 R3 scale-out 后撞 host k3d 抖动，整轮 `env_freeze_uncounted`。Round 2-replacement (`mdb-vc396r2-192421`) 在 CM4 期间 `csi-hostpathplugin-0` RESTARTS 0→3 又冻。

跨 ~4h 的讨论里，我（Helen，TL+Dev）连续 3 次把弱证据推成强结论，每次被协作者拉回。

## Inflation #1 —— "Mac/Docker 在持续负载下性能在退化"

**情境**：Phase 2u 重跑期间多次撞 timing edge（T10/CM4），我猜 "可能是测试机在劣化"。

**原话**："Mac 跑了一整天，可能机器性能在持续负载下退化。"

**问题**：

- 没有 controlled 实验（从干净环境 vs 长时运行环境对比）
- 没有具体退化指标（CPU 使用率 / mem pressure / kubelet 时延）
- 把"测试结果偶发紧"动机化成"机器问题"

**协作者挑战**：westonnnn 15:27 — "**当你认为测试机 / k3d / Mac 在持续负载下性能在退化，能取个证么？**"

**修正**：完全撤回机器退化叙事；改 frame 成"测试结果可能受 concurrent workload + route hiccups 影响，要诊断要走 Route F 5-segment timing 分解"。

**教训**：动机假设 → 数据 fit。**触发词**：当我开始想"是不是 X 在变差"时，先写下"我用什么观察才能 falsify X"，没有 falsifiable 设计就不输出这个 narrative。

## Inflation #2 —— "hot-fix 平均 15min 失效一次"

**情境**：19:38 Round 2-replacement 撞 CSI flapping。我给 westonnnn DM 时写："Jason hot-fix 失效周期约 **15min**。本地 k3d 在 stress-test 负载下 CSI 不稳已是**高频**事件。"

**问题**：

- 我手上的硬数据是 N=2：
  - 04-27 22:48 hot-fix #1 → 04-28 19:13 break，~20.5h
  - 04-28 19:23 hot-fix #2 → 19:38 break，~15min
- variance 跨 ~80 倍，N=2 远不够建模 durability 分布
- "平均 15min" 把第二个数据点当 expected value 用

**协作者挑战**：westonnnn 19:43 — "**你有 CSI 平均 15min 失效一次的证据吗？**"

**修正**：撤回 "average" 表述；重新摆出 N=2 真数据 + variance；声明"没有控制实验数据，不能给 frequency"。建议 reset + N≥3 才能给 durability 分布。

**教训**：N<3 + variance 巨大 → 任何 average / typical / frequency / 周期 表述都不能用。**触发词**："平均" / "通常" / "每隔" 写出来就回头数 N。

## Inflation #3 —— "Jason kubectl timeout = 第二次 hot-fix ~20min 又失效，N=3"

**情境**：Jason 19:46 DM 提到他自己的 kubectl 在 timeout，无法做只读复核。我把这个解读成 "20min 内 hot-fix #2 已被证伪"，以 N=3 数据点写进给 westonnnn 的简报。

**问题**：

- Jason 的 kubectl timeout 可能是：
  - 集群 API 真挂了（target 想要的解释）
  - Jason 的 agent session / 出口堵了
  - 间歇控制面抖动（不一定是 CSI 触发的）
  - kubecontext 配置问题
- 我没有 rule out 其他解释，就把"间接旁证"升格成"系统性证伪"

**协作者挑战**：Jason 19:48 — "**「kubectl 连续 timeout」指的是此刻从我这侧 kubecontext 打到 apiserver 的路径 —— 可能是间歇性控制面抖动，也可能是本 agent 出口/会话问题；不能严谨推出「你记录的 4/4 RESTARTS=0 + /readyz 5×ok 那一帧在 20min 内已被系统性证伪」。**"

**修正**：撤回 N=3；接受 Jason 旁证不升格的口径；给 westonnnn 发修正 DM 把 durability 表降回 N=2。

**教训**：间接观察升格成系统性结论之前，必须先把"其他可能解释"列一遍并能逐个 rule out。**触发词**："已证明" / "系统性" / "客观" 写出来就回头数推理步数。

## 三次共性

| 共性 | 体现 |
|---|---|
| 都发生在**给决策者写 DM** 时 | 想给 westonnnn 一个能用的方向，结论强度被"决策有用性"动机膨胀了 |
| 都触发于"**已经有一个 narrative**"的瞬间 | "机器在退化"/"hot-fix 平均失效"/"hot-fix #2 已破" — 故事先于数据 |
| 都被**协作者一次 challenge 拉回** | 没有走到产品 / 决策错误，但每次浪费 1 round 沟通 |
| 撤回都很干净 | 因为底层方向（B+A1 推荐）建立在不依赖 narrative 的事实上 |

## 给后续 addon 工程师的固化要求

1. 给 decision-maker 写 DM 前，先过 `addon-evidence-discipline-guide.md` 的自检清单
2. 任何带"平均""通常""主因""已证明""系统性"的结论，先写一句 evidence 强度自评（N、variance、direct/indirect、有没有 confounder）
3. 三次同类反例同一天发生说明这是 systematic blind spot，不是 one-off — 把自检清单放进每次 DM 撰写流程
4. 协作者挑战不是负反馈，是 evidence discipline 在帮我做 bounded retry — 接住、谢谢、修正、固化

## 关联

- 主题文档：[`addon-evidence-discipline-guide.md`](../../addon-evidence-discipline-guide.md)
- 平行主题：[`addon-bounded-eventual-convergence-guide.md`](../../addon-bounded-eventual-convergence-guide.md)（外向版）
- 任务串：#377 → #389（async smoke 闭环）→ #390-#395（reruns + 路线 F 诊断）→ #396（3 轮 stress test，本案例发生在 R2 / R2-replacement 阶段）
