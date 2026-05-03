# Addon Ops Restart 排障分流指南

> **Audience**: addon dev / test / 排障工程师，遇到 OpsRequest restart 看起来卡住时
> **Status**: stable
> **Applies to**: 任何 KB addon（OpsRequest queue/execute 分层是 KB 通用控制流）
> **Applies to KB version**: KB 1.0.x / 1.1.x / main（OpsRequest queue 入口与执行体内部状态机契约跨 1.x 稳定）
> **Affected by version skew**: 不受 KB 版本影响 — 'queue 入口未放行' vs '执行体内部' 的两段式排障逻辑跨版本一致

本文面向 Addon 开发与测试工程师，聚焦一个常见但容易被混读的问题：当 restart / rolling restart / pod rolling update 看起来"卡住"时，先不要立刻进入 Pod 级进度分析，而要先判断这条 `OpsRequest` 是否已经真正进入执行体。

正文只写通用方法论，不绑定某一个引擎。引擎相关实现、现场、样本结论放在独立案例材料中。

## 先用白话理解这篇文档

可以把 restart 排障想成两道门：

1. **有没有进门**
- request 还在 `Pending`、没有 `START_TS`，更像还在门口排队
- 这时去问“里面是哪台 pod 卡住了”，问题就问早了

2. **进门后卡在哪**
- request 已经拿到 `START_TS`，才说明它真的开始执行
- 这时再看 pod、revision、gate、progressDetails 才有意义

还有一个特别容易混的点：

- `restart` 很多时候更像“下发一张更新通知”
- 真正决定“下一台打谁、为什么现在不继续”的，往往是 workload controller 的 update plan 和 gate

所以这篇文档想解决的不是“怎么解释所有 restart 现象”，而是先把问题问准：

- 这是**还没轮到执行**
- 还是**已经执行，但推进 gate 没放行**

## 适用场景

当你遇到以下任一现象时，这份文档适用：

- 新的 restart request 长时间停在 `Pending`
- condition / event 反复只有 `WaitForProgressing`
- 旧 request 还在 `Running`
- 团队已经开始讨论 `progressDetails`、某个 pod 为什么没继续推进，但这条新 request 还没有 `START_TS`
- 同一个 cluster 上前后两条 restart request 互相影响，容易把旧样本和新样本混在一起

## 核心边界

排这类问题时，先把问题分成两类：

1. `queue 入口未放行`
2. `restart 执行体内部`

最重要的分界点是：

- `status.startTimestamp`

如果一条新的 restart request 还没有 `START_TS`，它通常还不能算“有效执行样本”。这时不应先回答：

- 某个 pod 为什么没继续滚动
- `progressDetails` 为什么停在某个成员
- 某个参数修改是否已经影响本轮 restart

更准确的问题应该是：

- 这条 request 为什么还没拿到正式执行机会

## 先看是不是 `queue 入口未放行`

在进入 Pod 级分析前，先固定检查下面几项。

### 1. 这条 action 是否有队列语义

优先确认：

- 当前 action 是否按 cluster 串行执行，例如 `QueueByCluster`
- 控制器是否要求同一个 cluster 上前一条 request 完成后，后一条才能进入正式执行

如果答案是“是”，那么新的 request 停在 `Pending` 未必说明执行体出错，也可能只是还没轮到它。

### 2. 新 request 当前到底停在哪

固定收以下信息：

- `phase`
- `conditions`
- `events`
- `status.startTimestamp`
- `progress` / `progressDetails`

如果当前形状更像：

- `phase = Pending`
- `condition = WaitForProgressing`
- `status.startTimestamp` 为空

那么这条 request 更应先归到 `queue 入口未放行`，而不是 `restart 执行体内部`。

### 3. 同 cluster 是否还有旧 request 未结束

继续确认：

- 同一个 cluster 上是否还有旧 restart / ops request 处于 `Running`
- 旧 request 是否仍占着队列
- 旧 request 的 `progress` / `progressDetails` 是否显示它还没 complete

这一步的价值在于把主问题改写准确：

- 不再问“新 request 为什么不跑”
- 而是问“旧 request 为什么还没有 complete / dequeue”

### 4. 队列释放条件到底是什么

不要想当然地把下面这些状态当成 queue 已释放：

- 某个 pod `Available`
- `progress = 1/N`
- 某个 component `Status OK`

对有队列语义的 action，真正该核的是：

- queue 释放是否绑定在旧 request 进入 complete phase
- complete 后是否明确触发 dequeue

如果 queue 释放条件还没满足，那么新 request 的 `Pending / WaitForProgressing` 通常只是后继结果，不是当前主根因。

## 如果已进入执行体，再看 `restart 执行体内部`

只有在新的 request 已经拿到 `START_TS`，并进入 `Validated` / `Restarting` / `Running` 之后，才进入执行体分析。

这时建议固定下面的检查顺序。

## 什么叫“冻结现场”

排 restart / rolling update 问题时，经常会说“先冻结现场”。这里的冻结，不是：

- 停止分析
- 停止取证
- 把集群直接停掉

更准确的意思是：

- 先把当前这份已经复现出问题的 live 样本保护住
- 不再用新的写操作把这份样本覆盖掉
- 让开发、测试、控制面分析都基于同一份证据基线说话

通常包括：

- 不再继续重跑同一路径测试
- 不主动清理、删 pod、删对象、重建 cluster
- 不同时再引入新的变量，例如换测试集、换版本、扩测试范围
- 先固定样本名、版本、镜像、对象快照、日志、revision、events、conditions

冻结现场的目的通常只有两个：

1. **保住根因证据**
- 避免后续操作把原始症状冲掉

2. **隔离变量**
- 避免把“旧现场的问题”与“新改动后的结果”混成一条

所以更准确的一句话是：

- **先停写、先保样本、先取证，再做下一次且仅一次的定向动作**

### 1. 用 `START_TS` 固定样本窗口

执行体分析必须围绕新 request 自己的 `START_TS` 展开，不要混读旧 baseline。

至少要固定：

- 这条 request 的 `OPS_REQ`
- `START_TS`
- 围绕 `START_TS` 的状态快照
- 同窗口 controller / workload / pod 日志

如果团队已经有固定模板，例如多个 UTC 取样点和 `+5m` 快照，继续沿用即可。

### 2. 再看 `progressDetails`

此时才适合分析：

- 哪些成员已经 `Succeed`
- 哪些成员仍是 `Pending`
- 哪些成员没有命中这次 restart 的有效进度条件

如果实现里进度统计依赖：

- pod 是否在 `START_TS` 之后被重建
- 是否命中特定 `podApplyOps`
- 是否满足某个 component phase / readiness 条件

那就要明确指出：是哪一个成员没满足哪一类条件，而不是笼统说“restart 没继续推进”。

### 3. 先分清“谁发起 restart”与“谁决定下一拍”

白话一点说：

- `restart` 控制器常常只是“按下开始键”
- workload controller 才更像“真正调度下一步的人”

如果现场表现成“只动了一台就不继续”，不要立刻理解成：

- restart 自己只想打一台

更常见的真实含义是：

- restart intent 已经发出去了
- 但 workload 那边的串行策略、ready gate、quota 或 revision 条件还没放下一拍

所以这里真正要找的是“谁没给下一张通行证”，不是只盯着“谁先发起了 restart”。

不要默认 `restart` / `OpsRequest` 控制器自己决定：

- 先打哪个 pod
- 下一拍还能不能继续
- 为什么会停在某一个成员之后

在很多实现里，`restart` 动作本身只负责推进 component 级 intent，例如写 annotation 或触发上层动作；真正决定：

- 串行顺序
- 下一拍的 update plan
- `Ready` / `Available` / `RoleReady` 一类 gate 是否放行

往往是 workload 控制器。

所以如果现场形状是“只更新了一个成员就不继续”，不要只盯 `restart` action 本身，还要继续核：

- workload controller 的 serial 策略
- update plan
- readiness / availability / role gate

否则很容易把“执行体内部的推进 gate 没放行”误判成“restart 自己只想打一台 pod”。

### 4. 不要按默认 workload 策略推断，要看 live object

白话一点说，不要拿“代码里常见默认值”当现场事实。

- 默认策略像产品说明书上的默认档位
- live workload object 才像车现在实际挂着的档位

排障时如果只按默认值推断，很容易把：

- 实际并行策略
- 实际 revision 分布
- 实际 gate 顺序

误讲成另一套根本没在现场生效的逻辑。

方向上知道“真正的推进与 gate 在 workload controller 一侧”还不够，后续还要继续确认：

- live workload object 的实际 update strategy
- pod management / update / upgrade policy
- `currentRevision` / `updateRevision`
- `updatedReplicas` / `currentReplicas`
- 每个 pod 当前命中的 revision

也就是说，不要因为代码里常见某条默认路径，就直接把现场收成：

- 默认 `Serial`
- 默认某个 update plan
- 默认某类 gate 顺序

如果 live object 显示的是另一套策略，例如并行更新策略、不同的 pod 管理策略或 revision 分布，那么后续分析必须跟着 live 实物走，而不是按默认值假设。

### 5. 把 controller 证据和 workload 证据放到同一窗口

执行体问题如果没有同窗口证据，很容易变成猜测。

至少应把以下信息放到同一时间窗口里看：

- OpsRequest 状态变化
- component / workload 状态变化
- controller reconcile 日志
- pod / instance / sidecar 关键日志

不要只拿单边日志就提前收根因。

### 6. 外部状态只能收窄时，要切到最小埋点

白话一点说，外部状态看到这里还解释不通时，就不要继续“隔着玻璃猜里面发生了什么”。

更稳妥的做法是：

- 只补最少的 debug 信息
- 直接打在真正拍板的 gate、plan、quota 判断点上

目标不是一次把所有日志都加满，而是回答最小的问题：

- 这一步为什么判成不能继续
- 是哪个条件把下一拍挡住了

如果你已经把问题收窄到某一层，例如：

- queue 已不是主问题
- 哪些 pod 命中了 progress、哪些没命中已经明确
- live strategy / revision / gate 也已经对齐

但仍然回答不了“为什么下一拍没继续放行”，这时不要继续只靠外部状态推断，应转去补最小 debug log / instrumentation。

优先埋以下几类点：

- update plan 选择结果：这一拍选中了哪些成员、跳过了哪些成员、reason 是什么
- gate 判定结果：`Ready` / `Available` / `RoleReady` 一类 gate 对每个成员的判定值
- `canBeUpdated` 或等价放行条件：为什么下一拍没有继续放行
- 进入 plan 时的关键输入快照：strategy / revision / per-pod state / current vs update revision

目标不是泛泛加日志，而是：

- 让“为什么没继续推进”第一次有结构化 reason
- 让下一轮复现时，能直接把 plan / gate / skip reason 收回来

## 小心“伪 drift”把更新配额吃掉

还有一类很容易被误判成 addon 问题或 gate 问题的 blocker，其实落在 controller 的比较语义上：

- 某个成员被反复判定成“还没对齐”
- 但现场又看不到真实的 spec 差异
- 也没有真正的 update 动作发生
- 后续成员却一直拿不到继续推进的机会

这类问题常见于：

- controller 用来判断是否需要更新的字段没有先做归一化
- 语义等价的空值形态被判成不同，例如 `nil` vs `""`
- 注解、默认值、render 结果与 live 对象在“表示形式”上不同，但在业务语义上其实等价

一个很常见的具体来源是：

- 某条 config 在 spec/API 里本来就是可选字段，例如 `ConfigHash *string`
- 上游没有显式给值时，spec 侧自然保持 `nil`
- 但 controller 在把它落到 pod annotation、状态快照或中间对象时，会用默认值把 `nil` 序列化成 `""`
- 如果后续比较不做归一化，就会把同一个“未设置 hash”语义误判成 drift

这类形状在 `externalManaged` config 上尤其容易出现，因为：

- 这类 config 往往不会沿默认模板渲染路径自动补 hash
- spec 侧允许合法地保持“没有 hash”
- 下游序列化时却常常会把它写成空字符串

典型后果是：

- 某个 pod 被反复选进 update plan
- `configsToUpdate != 0`，但实际没有可执行的更新动作
- 控制面内部仍把它记成 `allUpdated=false`、`updatingPods++` 或等价状态
- 结果把有限的 `unavailableQuota` / 并发更新配额吃掉
- 真正还没处理的后续成员被饿死，看起来像“滚动更新只打一台就不继续”

遇到这种形状时，建议固定检查：

1. controller 比较的到底是哪几个字段
2. 这些字段在 live spec、pod annotation、render 结果里是否只是空值形态不同
3. 这个成员是否被反复选中，但每轮都没有真实 update 动作
4. 配额统计是否发生在“判定有更新需求”这一步，而不是“真正开始更新”之后

更稳妥的修法通常是：

- 先把语义等价的空值做归一化比较
- 把“需要更新”和“已经占用配额”拆成更严格的两层

否则现场会表现成：

- 看起来像 addon 生成了不稳定输入
- 或像 workload gate 没放行

但 first blocker 其实是 controller 把“伪 drift”当成了真实 drift。

还要注意一点：

- **controller 缺陷不一定会在所有 addon / 所有拓扑里一起复现**

很多这类问题本身就是：

- topology-sensitive
- input-shape-sensitive
- quota-sensitive

也就是说，别的拓扑没撞上，只能说明它没有命中同一组触发条件；这不能单独拿来反证 controller 没问题。

做跨拓扑对照时，建议先回答一个更基础的问题：

- **这个拓扑到底有没有走到同一类输入链路**

例如先看：

- 是否存在 `spec.configs`
- pod 上是否存在对应的 `config-hash` 注解
- 是否真的有配置对齐 / reconfigure-only / quota accounting 这条控制面路径

如果这些输入链路本来就不存在，那么“它没复现”通常只说明：

- 它没有命中这类问题的第一拍触发形态

而不是说明：

- controller 在另一条已闭合问题上的归属判断有误

## restart blocker 的修复，最好做“两段式收口”

这条规则也可以用很直白的话理解：

- **第一轮**像是拿旧事故录像回放，确认原来的撞点确实被修掉了
- **第二轮**像是重新上路跑一次，确认不是只对旧录像有效，而是新现场也能正常通过

只做第一轮，最多说明：

- 旧 bad path 被切断了

两轮都做完，才更有资格说：

- 这条 restart 修复已经闭合

当一条 restart blocker 已经收窄到具体代码修复后，不要只做一轮验证就直接宣称闭合。更稳妥的做法通常是分成两段：

### 第一段：在冻结失败样本上证明旧 bad path 已被切断

这一段的目标不是“证明所有 restart 都恢复正常”，而是：

- 继续使用已经固定下来的失败样本
- 只改变一个验证变量，例如 controller patch、单个 helper patch 或单个脚本修复
- 证明旧现场里那条已知 bad path 不再出现

这一轮更像是在回答：

- 这次修复是否真的命中了之前已经取证清楚的 first blocker

### 第二段：再做一轮 clean restart，确认新现场也能通过

冻结样本验证通过后，还应再做一轮新的、干净的 restart 样本，确认：

- 新 request / 新样本本身可以走通
- controller / workload 日志里不再出现上一轮那条旧 bad path
- 修复不是只对“旧现场残留状态”有效

这一轮更像是在回答：

- 这次修复是否不仅切断了旧链路，也让 fresh restart 路径恢复正常

### 为什么这两段不能互相替代

只做第一段，常见风险是：

- 你证明了旧样本不再复现
- 但还没有证明 clean restart / fresh rollout 真的恢复

只做第二段，常见风险是：

- 新现场看起来通过了
- 但你没有直接证明原来那条已冻结的 first blocker 被切断

所以更稳妥的闭合口径通常是：

1. **冻结失败样本通过**：说明旧 bad path 被切断
2. **clean restart 通过**：说明 fresh restart 也恢复

两段都成立后，再把这条 restart blocker 收口成“已修复并通过定向验证”，会比只靠单轮验证更稳。

## 常见误判

### 1. 没有 `START_TS` 就开始分析 Pod 级停点

这是最常见的混读。没有 `START_TS` 的新 request，通常还不能回答“它在执行体内部卡在哪”。

### 2. 拿旧 baseline 去回答新实验

如果一条旧 request 还在 `Running`，它最多只能当 baseline / 对照材料，不能直接拿来回答新的 A/B 样本。

### 3. 把“新 request Pending”直接当根因

在有队列语义的 action 里，`Pending / WaitForProgressing` 很多时候只是表象。更前面的根因可能是：

- 旧 request 没有 complete
- 旧 request 没有 dequeue
- 所以后继 request 还没拿到执行机会

### 4. 没核代码里的 queue / progress 语义就提前下结论

这类问题经常要求把 live 现场和实现路径对一下。至少要确认：

- 何时入队
- 何时出队
- 何时写 `START_TS`
- 何时把某个成员记成有效 progress

否则就容易把“入口阻塞”误判成“执行中段卡住”。

## 建议固定回收的证据

每次排这类 restart 问题时，建议固定回以下内容：

1. 新 request 的 `phase`、`conditions`、`events`
2. 新 request 是否已有 `status.startTimestamp`
3. 同 cluster 旧 request 的 `phase`、`progress`、`progressDetails`
4. action 是否有 `QueueByCluster` 或同类串行语义
5. queue 释放 / dequeue 的实现条件或日志证据
6. 同窗口 controller / workload / pod 日志
7. 当前结论到底是 `queue 入口未放行` 还是 `restart 执行体内部`

## 最小判断口径

可以把这条规则收成一句：

- 没有 `START_TS`，先排 `queue 入口未放行`
- 有了 `START_TS`，再排 `restart 执行体内部`

## 相关主题

- [`docs/addon-componentdefinition-upgrade-guide.md`](addon-componentdefinition-upgrade-guide.md)
