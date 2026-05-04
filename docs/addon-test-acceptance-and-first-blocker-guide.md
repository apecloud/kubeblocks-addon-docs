# Addon 测试验收与 First Blocker 分层指南

> **Audience**: addon dev / test / TL
> **Status**: stable
> **Applies to**: any KB addon
> **Applies to KB version**: any (test methodology, version-agnostic)

本文面向 Addon 开发者、测试工程师和技术负责人，重点总结如何定义测试成功语义、如何收 first blocker，以及如何避免把环境、测试口径和产品缺陷混写。

## 先用白话理解这篇文档

### 这篇文档解决什么问题

测试报失败之后争论最大的不是"修不修"，而是"到底是哪一层错了"。

**新读者常踩的两种相反陷阱**：

- 看到 runner 报 FAIL 立刻当成产品 fail —— 实际上 first blocker 经常落在**比产品更早的层**：环境就绪、镜像分发、CSI plugin 状态、runner harness、测试口径、控制面 contract 都可能先错
- 看到任何 FAIL 都先怀疑是测试本身的问题 —— 这把真实产品 bug 当噪音吞掉，留给客户撞

→ 真正的方法论是**先把"哪一层算成功"分清**，再按 first blocker 分层归因，**不要在没有分层的状态下争论**。

### 何时本文方法论 apply

| 场景 | 关键决策 |
|---|---|
| 跑 smoke / chaos / regression 拿到第一个 FAIL | 先按 first blocker 分层（环境 / runner / 口径 / 控制面 / 产品），不要立刻写 product fail |
| OpsRequest 报 `Succeed` 但下一步 case 报 FAIL | 区分动作层 vs 拓扑层 vs 数据层 vs route 层成功 |
| 单 case 定向复验 | 先校验验证入口坐标 / 样本绑定基线 / runner 重打包后是否还指向同一样本 |
| post-restart 后跑测试 | 默认怀疑 setup blocker 在环境层先失效（详见 `addon-test-environment-gate-hygiene-guide.md`）|
| 写 RCA / final review-summary | 标 first blocker 时必须打层级 tag，不只写"FAIL 数" |
| 看 runner exit 1 但 post-run 残留 0 | 检查是不是 observation-class fail（不写产品 fail）|

### 读完你能做什么决策

- **拿到 N 个 FAIL 时**：能在 5 分钟内按层级分类，知道哪些是 setup blocker / runner harness / observation gap / 真产品 fail
- **判产品 fail 时**：知道这条结论需要先排除哪 4 层 noise 才能下
- **写 review-summary 时**：能给每条 fail 标 layer tag（不只 PASS/FAIL/SKIP 计数）
- **判 OpsRequest Succeed 是否真成功**：知道"controller phase Succeed"和"runtime / route / 数据层 ok"是不同 contract

### 为什么独立成篇

跟 `addon-test-environment-gate-hygiene-guide.md`（开跑前不踩雷）+ `addon-test-probe-classification-guide.md`（跑中失败不误归到 DB）一起构成完整的"环境 → 探针 → 验收"三阶段方法论。本文聚焦**验收 / first blocker 分层**这一阶段：测试结果出来后如何不混层归因。

---

## 适用场景

当你在推进以下工作时，这份文档适用：

- smoke / chaos / regression / backup-restore 回归
- 单 case 定向复验
- first blocker 收口
- runner / harness / helper 调整
- 现场冻结、proof 留存、问题分层

## 先分清“哪一层算成功”

很多争论不是实现问题，而是成功语义没有分层。至少要分清以下几层：

- **动作层成功**：某个 action / OpsRequest 自身返回 `Succeed`
- **拓扑层成功**：primary / secondary、replication、role truth 已闭合
- **入口层成功**：service / endpoint / client-facing route 已收敛
- **runtime 层成功**：live config / runtime value 已变成预期值
- **最终用户语义成功**：对客户端真正可用，且满足产品 contract

如果不先说明“这次 case 到底验哪一层”，就会出现：

- `Ops Succeed` 了，但 cluster 还没稳定；
- DB 已切换成功，但 service route 还没收敛；
- ConfigMap 已更新，但 runtime 还是旧值；
- scene 最终自愈了，但中途 action 失败了；

然后团队对同一个 case 会同时得出“通过”和“失败”两种结论。

## `OpsRequest Succeed` 不等于“下一步已经可以开始”

在链式 case 里，不要把：

- `wait_ops()`

当成完整的稳定态等待。

真实情况里经常会出现：

- `OpsRequest.phase=Succeed`
- 但 `Cluster.phase` 仍是 `Updating` / `Abnormal`
- endpoint / role / runtime truth 还在收敛

所以凡是：

- `reconfigure -> restart`
- `ops -> next ops`
- `switchover -> kill`
- `chaos -> 验收`

这类链路，都建议固定拆成：

1. 先等动作层完成；
2. 再等 bounded eventual convergence；
3. 最后检查 case 真正关心的 truth。

## 如果 `OpsRequest` 长时间 `Running` / timeout，先按动作层 blocker 收

还有一种常见误判是：

- 大家看到 cluster / role / endpoint / data 后面最终还能收敛
- 就想把一个长时间 `Running`、最后 timeout 的 `OpsRequest` 回填成“其实问题不大”

这会把两类问题混写：

- **动作层**：这次 `OpsRequest` 自己到底有没有正常退出
- **后续 truth 层**：scene 后来是否最终自愈或收敛

如果当前直接信号是：

- `Reconfiguring` / `Switchover` / 其他 `OpsRequest` 长时间 `Running`
- runner 自己在固定窗口后 timeout
- 或 action / controller 一直没有把 phase 收到 `Succeed/Failed`

那第一优先级应先收成：

- **action-layer / control-plane execution blocker**

而不是先往：

- 数据面 truth 最终闭合
- endpoint 最后恢复
- runtime 后来正常

这些方向回退。

这时建议固定收这几类证据：

1. 当前 `OpsRequest` 打到哪个对象、组件、action
2. 对应 action / `reconfigure.exec` 是否真的开始执行
3. `stdout/stderr`、event、phase 最后停在哪一层
4. 当前卡在：
   - 动作根本没起
   - 动作起了但没退出
   - 动作退出了但控制面没收口
5. timeout budget / contract 是否与这类操作的真实收敛时间匹配

一句话说：

- **Ops 自己没退出前，先按动作层 blocker 收；不要拿后续 truth 回填成功。**

## mid-flight target death 后，Ops 必须有终态 contract

对 `Reconfiguring` / `Switchover` / `Restart` 这类 action，如果 action 执行过程中目标实例被 kill、重启或进入不可服务状态，不能只看后续集群是否最终恢复。更关键的问题是：

- 当前 `OpsRequest` 对 **mid-flight target death** 有没有明确终态 contract。

如果目标实例在 action 过程中死亡，后面又恢复成某种需要人工/显式修复的状态，例如：

- divergence pending
- rebuild pending
- resync pending
- 角色无法发布
- action target 不再满足原 action 前置条件

那 `OpsRequest` 不应该长期保持 `Running`。它至少应该收敛到下面两类之一：

1. **明确 Failed**
   - 当前 action 的原始目标已经无法按原 contract 完成；
   - 失败原因里应说明 target death / target state changed / pending marker 等直接证据；
   - 后续恢复交给新的显式修复动作。
2. **显式进入新的恢复路径**
   - 例如 rebuild / resync / re-init；
   - 并且这个新路径要有独立的进度、终态和 timeout contract；
   - 不能把原 action 无限挂在 `Running`。

这类问题不要写成普通的“恢复慢一点”：

- 如果 action target mid-flight 死亡；
- 且目标恢复后已经不是原 action 可以继续操作的状态；
- 那 first blocker 就是 **terminal contract 缺失**，而不是简单 acceptance timing。

排查时建议固定三段时间线：

1. action start / target kill / target recovered 的顺序；
2. target recovered 后的真实状态，例如 role、replication、pending marker、runtime truth；
3. `OpsRequest` 在这些状态变化后是否进入 `Succeed` / `Failed` / 显式 recovery path。

一句话：

- **mid-flight target death 后，Ops 可以失败，也可以切入显式恢复，但不能无限 Running。**

## setup blocker 要和目标 runtime 结论分开

还有一种常见误判，是在做最小定向样本时：

- 本来想补某条 runtime / reconfigure / switchover 证据；
- 结果样本还没进入目标检查点；
- 却先在 create / startup / template / config render 这类前置阶段失败；
- 最后又把这个 setup 问题混写成目标问题线的“新结论”。

这会把两类问题混成一条：

- **setup blocker**：样本本身没成功进入要观察的阶段
- **target conclusion**：你原本真正想验证的 runtime / action / truth 假设

如果当前直接信号是：

- `config/script template` 缺失
- 启动期 `exec format error`
- create / render / install / bootstrap 还没成功
- 还没跑到 `wait_ops(...)`、runtime 检查、角色切换检查
- OpsRequest create/apply 自身失败，或者 helper 只生成了本地 synthetic name、实际 API 对象不存在

那应先收成：

- **setup blocker**

而不是：

- 直接更新原问题线的 runtime 结论
- 或把 setup 问题写成 addon/data correctness 回退

更稳的做法是固定拆成两步：

1. 先把这次 setup 问题单独建线、单独保存 artifact
2. 再决定：
   - 现有 frozen scene 是否足够继续补原问题的关键证据
   - 如果不够，是否要起 **1 套最小补证样本**

一句话说：

- **样本还没跑到目标检查点前，先把 setup blocker 单独收，不要混进原本想证明的 runtime 结论。**

这里还要再补一层，适合现在这种“已经换了新 single-cut，但样本先被别的 setup-flow 问题截住”的场景：

- 当前这轮本来是为了验证某个**新的最小代码切点**
- 但 fresh sample 还没进入目标结果面
- 却先在 create / setup / switchover / scale / topology flatten 之类前置流转上卡住

这时更稳的口径不是：

- “这版新 cut 依然无效”
- 或“这就是这版代码的新结论”

而应该先收成：

- **当前 single-cut 尚未被有效命中**
- **本轮 first blocker 仍是更早的 setup-flow blocker**

只有在样本真正进入原本想观察的结果面之后，新的 cut 才有资格被收成“有效负结果”或“有效正结果”。

一句话说：

- **新 cut 没打到目标结果面前，先冻结成 setup-flow blocker，不要提前升级成代码结论。**

如果同一条共享流程总是被较早的流程段截住，而你真正要验证的是后面的诊断点，就应该把两条线拆开：

- **流程线**：继续按完整 case 验证 create / switchover / scale / upgrade 等端到端行为；
- **诊断线**：只把样本带到最小稳定前置状态，然后直接触发目标动作取证。

例如要验证 reconfigure 的 runtime truth，就不一定要复用一条会先经过 scale-in / switchover 的完整共享流程。更稳的做法是：

- 只把样本拉到干净 `Running`；
- 直接发最小 `Reconfiguring OpsRequest`；
- 只收目标参数、Ops、controller、pod/runtime 证据。

这类拆分不是绕过测试，而是避免无关的 setup-flow blocker 反复挡住 root-cause 取证。

一句话说：

- **端到端流程被前置 blocker 挡住时，把诊断线拆成最小目标动作，不要让完整流程继续绑架根因取证。**

## 单 case 定向验证时，先校验验证入口坐标

当团队为了排除共享流程干扰，把某个 case 单独拎出来验证时，不要默认新的入口脚本天然可信。

定向入口至少要先确认三组坐标一致：

1. live Kubernetes 里的 namespace / cluster 名称
2. artifact root 里的 sample 名称
3. runner 输出、OpsRequest 名称、日志目录里的样本标识

如果其中任一处还出现：

- `PLACEHOLDER_NS`
- `PLACEHOLDER_CL`
- 空样本名
- 与 live cluster 不一致的 artifact 目录

那这一轮应先收成：

- **validation-entry setup bug**

而不是继续往目标 case 的产品结论里写。

这类问题尤其容易发生在“只跑某个 case”的新脚本里。脚本可能成功创建了一部分 live 对象，也可能只是打印了占位符，但只要样本坐标不可追踪，后续所有 evidence 都会失去审计价值。

更稳的处理方式是：

- 先停在 setup / Running 前后，不继续发目标 OpsRequest；
- 核对 live namespace / cluster 是否是真实样本；
- 核对 artifact root 是否用同一个真实样本名；
- 只有这三者一致，才进入目标 case 的 action / runtime 检查；
- 如果 live 资源已经按占位符创建，先清理并把本轮作为入口脚本问题归档。

一句话说：

- **定向验证可以跳过不相关前置流程，但不能跳过“入口坐标可审计”这一步。**


### full-suite runner 重新打包时，必须保留样本绑定基线

full-suite runner 做 targeted cut 后重新打包时，不要只确认新 cut 的函数或断言存在，还要确认 runner 入口仍然消费外部传入的样本坐标。特别是从仓库源码、历史 runner、临时补丁目录之间拷贝脚本时，很容易把已经修过的包装基线覆盖回默认值。

至少要在启动前固定三组证据：

1. wrapper 已导出当前 sample，例如 `CLUSTER_NAME=<fresh-sample>`；
2. 被执行的 case 脚本实际使用 `CLUSTER_NAME` / `ROUND_ID`，而不是重新生成 `mdb-async-$$` 或其他临时名；
3. live cluster、OpsRequest、runner log、proof 目录里的样本名全部一致。

如果 wrapper 和 case 脚本的样本绑定不一致，这一轮应立即收成：

- **runner packaging / sample-binding setup bug**

而不是继续运行后把结果写成产品 runtime 结论。已经创建出的错误样本要单独保存 line-level 差异、pre/post cleanup tail 和 artifact-only 证明；随后用已接受 runner 基线只叠加目标 cut，再从 fresh sample 重启。

一句话说：

- **验证新 cut 前，先证明 runner 仍然绑定同一个 fresh sample；样本坐标漂移时，这轮没有产品结论资格。**

## validation-only gate 身份未固定前，不要先创建 live 样本

定向验证里经常会临时切一版 validation-only controller / addon / runner gate，用来打日志、surface 状态或验证某个边界。这类 gate 的目标不是最终产品语义，而是让证据能回答一个明确问题。

在这种情况下，不要因为镜像已经编过、脚本已经准备好，就立刻创建 live 样本。先固定 gate 身份，否则后续证据无法审计，也无法判断它到底来自哪一版代码。

至少要在创建样本前固定这些信息：

- validation artifact root
- controller / addon / runner image tag
- imageID 或 digest
- live controller pod / addon revision / runner revision
- generation / resourceVersion / observedGeneration
- 本轮 single-cut 的文件路径或 diff 标识

如果这些字段里还有空值、占位符或上一轮残留，例如：

- `artifact root:` 为空
- `image:` 为空
- `imageID:` 为空
- sample 目录还没有真实 namespace / cluster 名

那这一轮应先停在 **gate-stabilization / traceability blocker**，不要继续创建新的 live 样本。否则即使后面拿到 runtime、OpsRequest、controller 日志，也会多一个根本问题：

- 这些证据到底是不是这版 validation-only cut 产生的？

更稳的处理顺序是：

1. 先切 live gate；
2. 等 controller / addon rollout 稳定；
3. 固定 gate 身份和 artifact root；
4. 再起 clean sample；
5. 取证完成后按 artifact-only 口径清 live。

一句话说：

- **validation-only gate 是证据的坐标系；坐标系没固定前，不要先造样本。**

### `config/script template has no template specified` 先看 file template 合成，不先怪缺失的目标 ConfigMap

如果 setup blocker 的直接信号是：

- `config/script template has no template specified: <name>`

那排查顺序也要收紧，不要先被表面现象带偏。

更准确的判断是：

- controller 里真正报错的直接触发点，通常是 `SynthesizeComponent.FileTemplates[].Template == ""`
- 这类检查往往发生在 **file template object create/update 之前**
- 所以样本 namespace 里最终**没生成**目标 `ConfigMap`
  - 例如 `...-valkey-valkey-replication-config`
- 更常是 **后果**，不是最早的先因

这类问题更稳的拆法是：

1. 先固定首个报错对象和首次报错时间点
2. 再固定当时 controller 实际消费到的：
   - `Component.spec.compDef`
   - `Component.spec.configs[*]`
   - 目标 file template 名
   - live `ComponentDefinition` 里对应的 template / namespace 引用
3. 最后再判断问题更像：
   - `chart/render` 根本没写出来
   - `apply` 阶段没创建成功
   - 还是 `ComponentDefinition/Component -> SynthesizeComponent.FileTemplates` 这段 **cmpd-consume/component build** 消费链里把 `Template` 合成丢了

一句话说：

- **看到 `has no template specified` 时，先查 file template 的消费/合成链；缺失的目标 ConfigMap 往往只是后果。**

进一步往下切时，还要把两种 controller 消费路径分开：

- **name-match 根本没命中**
  - `Component.spec.configs[*]` 里的名字没有对应到目标 file template
  - 最后落到空的用户模板对象
- **命中了，但 externalManaged ConfigMap 还没就绪**
  - `Component.spec.configs[*]` 已经有目标 name
  - 但消费到 `SynthesizeComponent.FileTemplates` 这一拍，`ConfigMap` 仍是 `nil`
  - controller 会把 `Template` 主动清空
  - 然后 `precheck()` 再报 `has no template specified`

如果当前已经能同时看到：

- `Component.spec.configs[*].name` 已存在
- live `ComponentDefinition` 里的 template / namespace 引用也还在
- 但 controller 仍在 `SynthesizeComponent.FileTemplates` 这拍报 `Template == ""`

那更应优先怀疑：

- **controller component synthesize / consume 链里的 `externalManaged` lazy-provision / ConfigMap-not-ready 状态**

而不是继续先怀疑：

- `chart/render`
- `apply`
- 或“目标 ConfigMap 没生成”本身

一句话说：

- **`has no template specified` 再往下切时，要先区分是 `name-match` 没命中，还是 `externalManaged ConfigMap` 还没被 controller 消费到。**

如果再往下还要区分：

- **是同一版对象被 controller 读旧了**
- 还是 **关键字段本来就是后补到后续 generation 的**

那不要先泛化成“旧 cache”，先对三件事做时间对齐：

1. `metadata.generation`
2. `status.conditions[*].observedGeneration`
3. 关键字段（例如 `Component.spec.configs[*].configMap.name`）是在第几代对象上才出现

如果当前能看到：

- 首个报错发生时，`observedGeneration` 还停在旧值
- 同一对象后来升到更高 `generation`
- 而关键字段是在这个更高 `generation` 才补上

那更应优先判断为：

- **source 在首个 reconcile 时还没 ready / 还没进入可消费状态**

而不是：

- controller 在同一版对象上只是吃了旧 cache

一句话说：

- **对 controller consume 时点问题，先看关键字段是不是后补到后续 generation；如果是，就先收成“source not ready on first reconcile”。**

对 `externalManaged` config 再往下压一层时，还要区分：

- source 本身是不是一开始就存在
- 还是 source 依赖 **另一条 controller 写路径** 先把对象生成出来，再把引用回写进 `Component.spec.configs[*]`

如果链路更像这样：

1. `ComponentParameter` 一侧先 render / create 出目标 config `ConfigMap`
2. 另一侧再由 `ParameterTemplateExtension` 之类的 controller 把
   - `Component.spec.configs[*].configMap.name`
   - 回写到 `Component`
3. 但 `componentFileTemplateTransformer.precheck()` 更早执行

那更准确的归类应是：

- **controller write-path timing**

而不是：

- `chart/render` 没写出来
- `apply` 没创建成功
- 或 addon/chart 自己把 template 绑错了

这类场景里更要先固定：

1. 是哪个 controller 先生成了目标 config `ConfigMap`
2. 是哪个 controller 后补了 `Component.spec.configs[*].configMap.name`
3. 这两个动作为什么晚于首个 `component template precheck`

一句话说：

- **如果 `configMap.name` 依赖后续 controller 回写，就先按 controller write-path timing 收，不要先怪 chart/apply。**

对 KubeBlocks 里的 `externalManaged` config，一个常见具体链路是：

1. `ComponentParameterReconciler`
   - 先 render / create 出目标 component config `ConfigMap`
2. `ParameterTemplateExtensionReconciler`
   - 发现这个 `ConfigMap` 已存在后
   - 再把 `Component.spec.configs[*].configMap.name`
   - patch 回 `Component`
3. 但 `componentFileTemplateTransformer.precheck()`
   - 可能已经更早执行
   - 于是首个 reconcile 先看到 `ExternalManaged=true && ConfigMap=nil`
   - 把 `Template` 清空并报 `has no template specified`

这类场景里，样本 namespace 里缺目标 config `ConfigMap` 不是关键结论；
更关键的是：

- **生成 config `ConfigMap` 的 controller 写路径**
- 和 **把 `configMap.name` 回写进 `Component.spec.configs[*]` 的 controller 写路径**
- 之间存在先后时序

如果当前已经能把链路收成：

- `ComponentParameter -> 生成 config ConfigMap`
- `ParameterTemplateExtension -> 回写 Component.spec.configs[].configMap.name`
- 这一整条链晚于首个 `component template precheck`

那就应优先收成：

- **controller write-path timing / source not ready on first reconcile**

而不是继续泛化成：

- addon/chart 模板问题
- `apply` 失败
- 或“目标 ConfigMap 没生成”

这里还要再补一层：

- **不要只问 first bad reconcile 为什么失败，还要问后续为什么没在窗口内修回来**

如果分钟级时间线已经能看到：

1. 首个 `component template precheck` 失败先发生
2. `ComponentDrivenParameter -> ComponentParameter -> ParameterTemplateExtension`
   - 这条写路径后进入
3. 在当前捕获窗口里
   - 对象已经推进到后续 `generation`
   - `spec.configs[].configMap.name` 也已经出现
   - 但 component 侧的 `no template specified` 仍持续了多轮 reconcile

那更准确的归类就不只是：

- `controller write-path timing`

而应进一步收成：

- **controller write-path multi-reconcile lag**

也就是：

- 不是一次 precheck 太早
- 而是 component controller 在多轮 reconcile 中都持续 outrun 了 source-ready 链
- 在当前采样窗口里，后补字段也没有把旧失败立即修回来

这里还要避免另一个误判：

- **窗口内没修回来**，不等于 **永远不会修回来**

如果更后面的同一组件日志已经显示：

- 不再停在 `template precheck`
- 而是进入了更后面的创建 / post-provision / action 阶段

那更准确的判断应是：

- 这个 setup blocker 在当前采样窗口里被 **multi-reconcile lag** 放大了
- 但它更像一个**迟滞修回**的 controller 时序问题
- 不要再把它扩大成“永久卡死”或 addon/chart 根因

一句话说：

- **`multi-reconcile lag` 可以表现为窗口内持续失败、窗口后迟滞修回；别把“没立刻修回来”直接写成“永远修不回来”。**

一句话说：

- **如果后续 reconcile 已经发生多轮，但 source-ready 链仍持续晚于 component precheck，就按 controller write-path multi-reconcile lag 收，不要再回退成 addon/chart 问题。**

## setup blocker 自愈后，下一层失败要重新分层

还有一种容易混写的情况：

- 前一层 setup blocker 已经迟滞自愈；
- controller 已经越过 template / config / render precheck；
- 但马上又进入新的 post-provision / action 阶段失败。

这时不要继续把新失败写回上一层问题。

例如，当日志已经从：

- `config/script template has no template specified`

推进到：

- `post-provision action failed`
- `has no pods to running the post-provision action`
- action target 为空

就说明 first blocker 已经从 **file template 合成 / source-ready 时序**，转到了更后面的 **post-provision action target / pod readiness** 层。

这时建议重新固定三组事实：

1. **模板 / config 层是否已经越过**
   - 是否还在报 template precheck；
   - 目标 file template / config object 是否已经生成；
   - `Component.spec.configs[*].configMap.name` 是否已经进入 observed generation。
2. **pod / workload 层是否真正可执行**
   - workload 是否已经创建；
   - pods 是否存在；
   - pods 是否至少有一个 `Running` / Ready；
   - selector / component name / workload owner 是否能把 action target 选出来。
3. **post-provision action 的 target contract**
   - action 是否允许在无 Running pod 时重试；
   - 无 target 时应 `Failed`、等待、还是显式跳过；
   - timeout 和 retry budget 是否有清晰上限。

如果当前直接信号是 `has no pods to running the post-provision action`，更合理的第一判断不是“模板问题还没修好”，而是：

- **post-provision action 找不到可运行 target**

然后再继续分：

- pod 根本没创建；
- pod 存在但没到 `Running`；
- selector / owner / component 名称导致 action target 选不出来；
- action 在无 target 时缺少终态 / retry contract。

一句话说：

- **前一个 setup blocker 自愈后，新的 post-provision 失败要按新层级重收 first blocker，不要把阶段推进后的失败倒灌回旧结论。**

## post-provision retry 后仍未到 Ready，要先验 workload readiness

如果 post-provision action 已经从“无 Running pod”进入 retry，但样本仍然没有到达 pod Ready / runtime 检查点，不要直接把它写成 runtime truth 失败。

这时 first blocker 还在 **post-provision retry -> workload readiness -> runtime** 之间，需要继续分层：

1. **retry 是否真的发生**
   - controller 是否重新排队；
   - retry 的时间点、次数、间隔是否符合 contract；
   - retry 后是否重新计算 action target。
2. **workload / pod 是否进入可执行态**
   - workload 是否已创建；
   - pod 是否已创建；
   - pod 是否被 scheduler 接受；
   - init container / sidecar / main container 是否启动；
   - readiness probe 是否通过。
3. **runtime 检查是否已经具备前置条件**
   - 至少一个目标 pod `Running` / Ready；
   - action target selector 能选到目标 pod；
   - runtime 查询命令有真实执行对象；
   - runtime check 的失败不是因为“没有 pod 可查”。

如果 retry 后仍没有 Ready pod，优先固定：

- pod event / describe；
- ownerRef / selector / label；
- PVC / image / init container / readiness probe 状态；
- controller 对 retry 后 target set 的日志；
- 最后一条阻断 pod Ready 的直接原因。

只有当 pod 已经 Ready，且 runtime 查询确实打到了目标进程，才能继续判断 runtime value 是否正确。

一句话说：

- **post-provision retry 后没到 Ready，runtime 结论仍未成立；先把 workload readiness / action target 层收清。**

## 对 eventual convergence，要显式写进测试口径

很多 case 不是“立刻通过”或“永远失败”，而是：

- 动作触发后几秒内仍是旧状态；
- 再过一小段时间才进入闭合状态。

这时要明确决定：

- 这个 case 是否允许 bounded eventual convergence；
- 窗口多大；
- 在窗口内要等哪些信号；
- 超过窗口后才算真正失败。

如果不把这层写清楚，就会出现“验收点过早”：

- 产品逻辑其实没问题；
- 只是 secondary publish / replication resumed / service route 比断言慢一点；
- 最后被误记成 addon 或控制面回退。

## bounded eventual convergence 只适用于单调收敛

还有一种常见误判，是把“窗口内发生真实回退”也写成 eventual convergence。

这两类情况必须分开：

- **单调收敛**：
  - 信号虽然慢一点，但方向一直是朝闭合前进；
  - 例如 `0 secondary -> 1 secondary`、`endpoint=<none> -> endpoint=primary`，中间没有再退回去。
- **窗口内回归**：
  - 信号已经先收敛了一次；
  - 之后又发生反向漂移；
  - 例如 `1 primary / 1 secondary -> 0 primary / 2 secondary -> 1 primary / 1 secondary`；
  - 或 `endpoint=pod-x -> endpoint=<none> -> endpoint=pod-y`。

第二类不能再简单收成“只是慢一点”。

如果当前 case 是：

- `OpsRequest` 已经 `Succeed`；
- 最终 DB truth / replication / data 也能闭合；
- 但在 bounded window 里出现了：
  - role publish 回退
  - endpoint publish 短空窗
  - condition / phase 先好后坏再恢复

那应优先收成：

- **control-plane convergence regression**

而不是：

- acceptance timing 太紧
- 最终 truth 已经闭合所以可以直接放过

这时建议固定做一件事：

- 对齐 `roleProbe / controller / syncer / pod DB truth` 的时间线，先找第一次从“已收敛状态”回退的 earliest break。

这里还要再补一层语义区分：

- **允许的短空窗**：
  - 前后 primary 确实切到了不同 pod；
  - 中间只有一个很短的 publish gap；
  - 最终 truth 又重新闭合。
- **不允许的 publish regression**：
  - 前后 primary 其实还是同一个 pod；
  - 但中间出现了 `1P1S -> 0P2S -> 1P1S`、`endpoint=<none>` 这类 publish 回退；
  - 这不能再按 eventual convergence 放过。

也就是说，不能只看“最后有没有恢复”，还要看：

- publish gap 是不是由 **primary identity 真切换** 带来的短暂空窗；
- 还是 **同一 primary** 上发生了 control-plane publish 抖动。

一句话说：

- **bounded eventual convergence 只适用于单调收敛；窗口内反向漂移要按 regression 单独收。**

## primary-switch 后的短暂 `1P0S`，要和 publish regression 分开

还有一类容易被误判的窗口，不是：

- `0P2S`
- `endpoint=<none>`

而是：

- **新 primary 已经切出来**
- endpoint 也已经指向新 primary
- 但旧 primary 还在恢复成 secondary 的路上
- 中间出现一个很短的 `1P0S`

这类窗口如果不分语义，很容易被直接收成：

- secondary publish/recovery regression
- role publish 回退
- runtime/data 问题

更稳妥的判断是先对齐四件事：

1. initial primary 是谁
2. final primary 是谁
3. bounded window 内 endpoint 指向谁
4. secondary 恢复到 `1P1S`、replication 闭合、data 闭合各自在什么时间点发生

如果同时满足下面这些条件：

- final primary **不同于** initial primary
- endpoint 在窗口内已经稳定指向 **新 primary**
- `1P0S` 只是旧 primary 恢复 secondary 的短暂窗口
- 窗口内没有 true divergence / pending marker
- 在明确上界内回到 `1P1S`
- 后续 replication / data truth 继续闭合

那这类现象更适合先收成：

- **primary-switch secondary recovery gap**

它更像一种有上界的 acceptance timing / bounded eventual convergence 问题，而不是直接收成 publish regression。

但边界要收紧，不能泛化放行。以下情况仍应继续按失败收：

- final primary 和 initial primary 其实是同一个 pod
- endpoint 在窗口内出现空窗，或没有稳定指向新 primary
- `1P0S` 超过约定窗口仍不恢复
- 窗口内出现 divergence / pending / role 反复抖动
- replication / data truth 最终没有闭合

一句话说：

- **如果 primary 已真实切 pod、endpoint 已稳定指向新主、最终又在上界内闭合到 `1P1S`，短暂 `1P0S` 更像 secondary recovery gap，不要直接写成 publish regression。**

## primary-switch secondary recovery gap，也可能先暴露成 helper 过窄

真实现场里，这类 bounded `1P0S` 不一定先表现为：

- runtime/data 不闭合
- true divergence
- role publish 永久回退

它也可能先表现为：

- 某条 acceptance helper / assertion 语义过窄
- helper 只要见过一次 `1P1S`，后面任何非 `1P1S` 都直接判 regression
- helper 没有区分 “同一 primary 抖动” 和 “primary 已切换后旧主恢复 secondary 的短窗口”

如果这时 proof / log / DB truth 已经能同时证明：

- final primary **不同于** initial primary
- endpoint 已稳定指向 **新 primary**
- bounded window 之后回到 `1P1S`
- replication / data truth 最终闭合
- 没有 divergence / pending marker

那 earliest break 更应该先收成：

- **acceptance / helper 语义过窄**

而不是直接升级成：

- runtime/data regression
- addon 缺陷
- 控制面 publish 缺陷

更稳妥的下一跳是先补 helper 语义：

1. 记录 initial primary / final primary
2. 记录窗口内 endpoint 实际指向
3. 允许 “endpoint 已到新主、旧主仍在恢复 secondary” 的 bounded `1P0S`
4. 继续保留 fail 条件：同主抖动、endpoint 空窗、超窗不恢复、或 truth 不闭合

一句话说：

- **如果强 truth 已说明是 primary-switch 后的 bounded recovery gap，而 helper 仍把它一律收成 regression，先修 helper 口径，不要先把锅扣到 runtime。**

## post-terminal secondary recovery 需要 bounded wait

在 chaos + reconfigure 组合场景里，`OpsRequest` 终态和角色最终恢复可能不是同一时刻完成。尤其是 kill secondary during reconfigure 这类 case，如果历史契约已经把它拆成：

- action terminal：`OpsRequest` 到达 `Succeed` 或 `Failed`；
- post-terminal recovery：secondary role publish / replication / data truth 在上界内最终闭合；

那么测试侧不能只在 terminal 后固定 sleep 一小段时间，再做一次即时 role label 扫描。正确口径是显式等待 bounded post-terminal recovery，直到 secondary role、replication truth、data truth 闭合，或超过窗口后失败。

排查这类问题时先固定四条证据：

1. `OpsRequest` terminal 时间；
2. controller roleProbe / role-change event 时间；
3. runner 在 terminal 后实际等待多久、是否只是 immediate scan；
4. 是否存在 current-round divergence / pending marker，最终 replication/data truth 是否闭合。

如果证据显示角色发布是单调从 `no secondary` 到 `secondary`，没有 current-round divergence，最终 replication/data truth 闭合，而 runner 只因为等待窗口过短失败，那么 earliest break 应收在测试侧 acceptance boundary 过窄，不应升级成 addon/runtime/control-plane blocker。

这不是放宽质量门槛。以下情况仍必须 fail：非单调收敛、窗口内 role/endpoint 反向漂移、current-round divergence/pending 真命中、最终 replication/data 不闭合、或超过约定 bounded window 仍没有 secondary。

## helper / assertion 缺口，不应自动升级成 runtime blocker

在真实回归里，常见一种“假 blocker”：

- runner 少一个 helper；
- 某个断言没执行；
- 某条弱证据链断了；

但与此同时，已经有更强的独立证据证明 case 真值闭合，例如：

- cluster 最终 `Running`
- role / endpoint / replication 全闭合
- 关键写入在两边都可见
- row count / marker truth 闭合

这时不要机械地把“少了一个 helper / 断言”直接升级成产品 blocker。

更推荐的处理是：

- 先判断这个 helper 缺口影响的是不是 **唯一关键证据**
- 如果不是，而更强的独立证据已经足够闭合成功语义，就把它记成：
  - **runner / harness 证据缺口**
  - 而不是 **runtime blocker**
- case 本身可以按产品语义收成 pass，同时把 helper 缺口留成单独的测试债务或后续修补项

一句话说：

- **强 truth 优先于弱 helper**

但前提是你已经明确：

- 哪些证据是直接证明成功语义的；
- 哪些只是辅助断言。

如果缺的正好是唯一关键证据，那当然不能放过；但如果缺的是一条弱证据，而强 truth 已经闭合，就不要把工具缺口误写成产品问题。

## 对 kill sidecar / helper / agent 类 case，先证明注入真的命中

对下面这类 case，不要一上来就讨论 runtime truth：

- kill sidecar
- kill helper
- kill agent
- 任何依赖 `exec` 注入的单容器故障动作

先固定问一件事：

- **这次注入到底有没有真的命中目标容器？**

默认至少收三层证据：

1. **注入动作确实发出**
   - runner / harness 已执行对应 `kill` / `exec`
   - target pod / target container / target process 记录明确
2. **容器重启或等价生命周期信号确实变化**
   - `containerID` 变化
   - `restartCount` 增长
   - `startedAt` 变化
3. **pod event / container status 有对应 termination / restart 证据**
   - event 中能看到 kill / restart
   - container status 中能看到 terminated / waiting / lastState

如果这三层都没有立住，而 scene 最终又仍然闭合，例如：

- cluster `Running`
- roles / endpoint / replication 闭合
- data truth 闭合

那正确分类通常不是 MariaDB runtime blocker，而是：

- harness / injection miss
- evidence gap
- 注入证据链不足

这时不要先往：

- failover 逻辑错误
- replication 回退
- addon runtime 缺陷

这些方向回退。

更务实的做法是：

- 先把 first blocker 收在“注入未命中 / 无重启证据”
- 把 kill target / exec 方式 / 可观察重启证据链补全
- 只有在注入命中已经被证明后，再讨论 runtime truth 是否失败

一句话说：

- **先证明 injection hit，再谈 runtime truth**

## 如果 `kill` primitive 和 runtime 语义不一致，先换成 runtime-native 入口

还有一种常见误判是：

- `kubectl exec` 返回 `rc=0`
- 目标进程看起来也被命中了
- 但容器生命周期完全没有变化

例如：

- `restartCount` 不变
- `containerID` 不变
- `startedAt` 不变
- `lastState.terminated` 为空
- pod event 里也没有 kill / restart 信号

这时不要急着把问题写成：

- MariaDB runtime 没反应
- sidecar 自愈逻辑异常
- 测试对象没有真正被杀掉

更应该先问：

- **当前使用的 kill primitive，是否真的等价于容器 runtime 眼里的 lifecycle 事件？**

在某些运行时环境里，`kill -9 1` 命中的是进程，但未必能稳定映射成你期待的容器 lifecycle 证据链。  
如果测试语义真正要验证的是“sidecar 容器发生一次可观察的 stop / restart”，那就应该优先选：

- container runtime 原生的 stop primitive
- 或者能稳定触发 lifecycle 变化的等价入口

例如这轮经验里，`kill -9 1` 不是当前 k3s/containerd 下合适的 lifecycle 证明 primitive，更合理的入口是：

- node-side `crictl stop <container-id>`

这样做的原则是：

- 不改业务 truth 语义
- 只把注入入口切到与 runtime lifecycle 一致的 primitive
- 先把 lifecycle 证据链立住，再去验 no-spurious-failover / same pod still primary / replication / data truth

一句话说：

- **要验证 lifecycle，就用 lifecycle-native primitive**

## First Blocker 必须先分层

first blocker 至少要先分到下面几类之一：

### 1. 环境 / 资源门禁问题

例如：

- slot 不够
- live cluster 数量超限
- PV / PVC / nodeAffinity 冲突
- dataprotection 镜像、备份依赖、StorageClass 不可用

如果 case 明确要验证 PVC 扩容、备份恢复、跨节点调度这类依赖底层能力的路径，环境门禁必须先证明底层能力存在。

以 PVC 扩容为例，测试入口至少要固定：

- 本轮实际使用的 `StorageClass` 名称；
- `allowVolumeExpansion` 是否为 `true`；
- provisioner 是否支持当前测试要求的 resize 语义；
- PVC / PV / Pod 事件里是否已经出现底层存储拒绝。

如果默认 `StorageClass` 本身不支持扩容，那么 first blocker 应先收成：

- **执行入口 / 环境 StorageClass 不匹配**

而不是：

- addon runtime 失败；
- reconfigure / scale / role 逻辑失败；
- 数据库没有正确处理扩容。

一句话说：

- **涉及底层资源能力的 case，先验环境门禁；底层能力不存在时，不要把失败倒灌成 addon runtime 结论。**

### 2. runner / harness / helper 问题

例如：

- helper 缺失
- 注入动作没真正触发
- case 入口跑偏
- 断言写错
- 验收点过早

### 3. 测试口径 / 产品语义未对齐

例如：

- `OpsRequest Failed`，但 scene 最终自愈闭合
- mid-flight kill 后到底按 action EOF 还是按 post-failover cluster truth 判定
- service route 是否算 action critical path

### 4. 控制面 / contract 问题

例如：

- action timeout contract 和 addon 内部等待预算冲突
- action target 死亡后，Ops 是否应继续按 scene truth 收口
- validate gate 是否应该允许当前时序

### 5. Addon / syncer / live-apply 真实缺陷

例如：

- 角色切换逻辑错误
- 参数没有真正 apply 到 runtime
- GTID / strict-mode 处理错误
- follow 假成功、节点假 ready

先把 blocker 放进这五类之一，再决定谁来修、怎么修、要不要继续扩样本。

## 每层对应的 falsification step

分层只是判类的第一步。要把 first blocker 真正落到某一层，必须能跑出**命中那一层**的 falsification 命令——不能只是脑子里"觉得是某层"。下面给每层 2-3 条最常用的命令链 + 期望输出 + 一行解读。具体 7 项 environment gate 全文见 [`addon-test-environment-gate-hygiene-guide.md`](addon-test-environment-gate-hygiene-guide.md)，本节只列"识别哪一层先 fail"用的关键信号。

### 第 1 层（环境 / 资源门禁）falsification

- `kubectl --kubeconfig <X> --context <X> get crd | grep <controller-domain> | wc -l` —— 期望与本轮 KB 基线匹配（例如 KB 1.0.3-beta.5 = 28）。漂移即 KB CRD 装错或装漏，**不是 addon bug**
- `kubectl --kubeconfig <X> --context <X> -n <ns> get pod <p> -o jsonpath='{range .spec.containers[*]}{.name}={.image}{"\n"}{end}'` —— 每个容器（含 init / sidecar）逐项核对 imageID 是否落到本轮预期 registry / tag。**漂移即镜像分发未收敛**
- `kubectl --kubeconfig <X> --context <X> -n <ns> get pod <csi-pod> -o jsonpath='{.status.containerStatuses[*].restartCount}'` —— 期望 0；非 0 即 CSI / dataprotection / vcluster 控制面层异常，**不要继续在该样本下产品结论**
- 任一信号触发 → 现场 freeze 后切到 [`addon-test-environment-gate-hygiene-guide.md`](addon-test-environment-gate-hygiene-guide.md) 的 7 项 gate 全检；不要在 env 异常样本上推 runtime / 产品 narrative

### 第 2 层（runner / harness / helper）falsification

- `grep -nrE "<anchor1>|<anchor2>" <runner-root>` —— 期望本轮所有 anchor 都被命中。staged copy 漏 anchor 即 runner 没装齐
- `bash -n <runner.sh>` + `git -C <runner-root> diff --check` —— 期望全绿。语法错或 trailing whitespace 即 runner 自身先 fail
- runner 内 `kubectl ...` 调用必有 `--kubeconfig + --context` **双锁**（参考 §1.4 of [`addon-test-script-preflight-guide.md`](addon-test-script-preflight-guide.md) — TBD pending PR）；裸 `kubectl` 即 cross-line broadcast 共享 default context 风险
- 任一信号触发 → 是 runner 自身 bug，**不是 addon 的事**；先修 runner 再重跑

### 第 3 层（测试口径 / 产品语义）falsification

- 单次 snapshot 当 reconverged 判定 → 错。bounded retry 必须显式带 `--max-attempts $N --interval $MS --total-timeout $T`，事件时间戳用真实事件时间不用 sample 时间（参考 [`addon-bounded-eventual-convergence-guide.md`](addon-bounded-eventual-convergence-guide.md) 反模式 5）
- expected 值硬编码（`expected='3'`）→ 错。链下游 case 对前置 INSERT 数量的依赖必须用变量（如 `EXPECTED_ROWS_POST_T5`）传播，不写死
- `OpsRequest.phase=Succeed` 当 "下一步可以开始" → 错。还要等动作层 / 拓扑层 / 入口层 / runtime 层各自显式收敛证据再推进
- 任一信号触发 → 是测试口径 bug，**不是 addon runtime 缺陷**；改测试口径再重跑

### 第 4 层（控制面 / contract）falsification

- `kubectl --kubeconfig <X> --context <X> -n kb-system logs deploy/kubeblocks --since=10m | grep -E 'wait for the workloads|finalizer'` —— 反复出现即可能撞 finalizer 反向死锁，按 [`addon-narrow-scope-force-delete-guide.md`](addon-narrow-scope-force-delete-guide.md) 处方收敛
- `kubectl --kubeconfig <X> --context <X> -n <ns> get opsrequest <op> -o yaml` —— phase=Succeed 但 cluster.phase 仍 Updating / route 未收敛 / runtime config 未变 → contract 层与数据面层收敛不一致
- 1s 粒度 TSV 抓 `kbagent` `change-only emit` 与 KB controller `exclusive-role-removal` 时序（参考 [`addon-rolling-day2-role-label-race-guide.md`](addon-rolling-day2-role-label-race-guide.md)）—— 时序错位即三控制环 race
- 任一信号触发 → 是 KB controller / kbagent / contract 层 bug，**不是 addon 业务层**；归 control_plane 上报，不混进 product 结论

### 第 5 层（Addon / syncer / live-apply 产品）falsification

- 上面 4 层的 falsification step **逐层 rule out** 后才有资格判 product fail
- N≥2 同条件复现 + cross-cluster 确认（不同 vcluster 或不同 host k8s）—— 单 cluster N=1 不够下 product 结论
- 把内部 syncer / agent 日志、DB 内部状态（SHOW REPLICA STATUS / SHOW MASTER STATUS）、pod runtime 三方对照
- 复现条件能稳定收敛后，做 **minimal cut + clean restart** bisection 定位修复点

## Layer-aware evidence 强度对应表

把 [`addon-evidence-discipline-guide.md`](addon-evidence-discipline-guide.md) 三规则按 layer 应用，强度阈值不同。给 reviewer 一份 layer-aware mapping，避免"产品层 fail 用 N=1 描述"或"环境层每条都要求 N=2"两种相反的 over-/under-claim。

| Layer | N 要求 | Cross-cluster 要求 | Rule-out scope（下结论前必须排除） | 描述强度上限 |
|-------|--------|---------------------|------------------------------------|---------------|
| **环境 / 资源门禁** | **N=1 即可** | 不要求（env 层信号本身定位单环境） | 仅排除"这一具体环境的 setup 状态" | "本环境观测到 X" |
| **runner / harness / helper** | **N=1 即可**（anchor 缺失 / 语法错是二值信号） | 不要求 | 静态 check（`bash -n` + anchor grep）已能下结论 | "runner 自身 bug，已 N=1 锁定" |
| **测试口径 / 产品语义** | **N≥2 推荐**（如比较 single-shot vs bounded retry 两种口径下行为） | 不严格要求 | test-spec review + bounded retry 复跑 | "口径不匹配产品 contract" |
| **控制面 / contract** | **N≥2 跨复现** | **推荐**（同 cluster shape 重跑） | KB controller log + ops timeline + kbagent emit 时序三方对照 | "跨 N=2 复现，控制面层 race / contract 漂" |
| **Addon / 产品真实缺陷** | **N≥2 + cross-cluster 确认** | **必需**（不同 vcluster / 不同 host k8s 排除 env confounder） | 上述 4 层 falsification step **逐层 rule out** | "产品层 fail，已 N≥2 + cross-cluster 锁定" |

> **核心原则**：描述强度 ≤ 证据强度对应那一行的上限。如果你正在写"产品层是 X"但 N=1 + 没排除控制面层，先回退到 "目前最可能的因果是产品层 X，仍有控制面层 Y 没排除"（参考 [`addon-evidence-discipline-guide.md`](addon-evidence-discipline-guide.md) 规则 A 描述-证据强度对应表）。

## 跨线 case 反链表

下表汇总各 layer 的实战 case anchor，按引擎 / 主题分类。新读者 / 新加入的 line 可以直接去对应 case 看完整闭环；本表只作 **anchor**，不复制 case 正文。

| Layer | Line / 主题 | Case anchor（doc 路径或 sediment thread）|
|-------|-------------|------------------------------------------|
| **env** | MySQL · idc2 docker.io 拉不到 → ACR mirror | task #3 evidence `evidence-task3.tar.gz/step4-image/registry-policy.txt`（N=2：`kubeblocks-tools` + `mysql:8.0.35` 同根因，patch live `ComponentVersion/mysql` + `ParametersDefinition/mysql-{5.7,8.0}-pd`，chart 源码不动） |
| **env** | OceanBase · stale kubeconfig 残留 | `~/.kube/config-ob-vcluster`（旧失效）vs `~/.kube/config-ob-vcluster-raw`（有效）共存 → typo / autocomplete 打错集群 |
| **env** | All lines · Mac → idc NodePort 路由 | 缺 VPN / 私网路由（如 Mac D 缺 192.168.10.0/24 路由）→ TCP timeout；从 host k8s 内 runner 走 ClusterIP 不受影响 |
| **env** | k3d · post-restart CSI mount propagation | [`docs/cases/mariadb/post-restart-csi-mount-propagation-case.md`](cases/mariadb/post-restart-csi-mount-propagation-case.md)（kubelet `/var/lib/kubelet` 失去 rshared，PVC 永远 Pending）|
| **runner** | SQL Server · 360-条 client_failed | 多 line 共用 `~/.kube/config` + `kubectl config use-context` 切走 default context → writer 不带 `--context` 连错集群（参考 r2-writer-context-flip-incident.md，evidence partner James / John）|
| **runner** | MySQL · runner 镜像分发漏检 | task #2 + task #3 N=2：`kubeblocks-tools` 6 分钟拉不到（task #2）+ docker.io/apecloud/mysql 多版本拉不到（task #3）→ Step 4 image distribution gate 必须含 runner / probe / tools 镜像，不只 addon 引擎主镜像 |
| **runner** | macOS · bash 3.2 兼容坑 | [`addon-test-runner-portability-guide.md`](addon-test-runner-portability-guide.md)（空数组 / env-default 时机 / 单条 local 内互引用 / `v\$parameter` 等 7 个常见兼容坑）|
| **口径** | OceanBase · 测试链 row-count 耦合 | [`docs/cases/oceanbase/oceanbase-test-chain-row-count-case.md`](cases/oceanbase/oceanbase-test-chain-row-count-case.md)（T5 INSERT 后 T7 `expected='3'` hardcoded → chaos 实际成功 false-fail；修复用 `EXPECTED_ROWS_POST_T5` 变量）|
| **口径** | MariaDB · single-shot 误当 bounded retry | [`docs/cases/mariadb/cm4-bounded-window-helper-semantic-bug-case.md`](cases/mariadb/cm4-bounded-window-helper-semantic-bug-case.md)（`last_elapsed` 误当 reconverged 时间，watch_window=deadline 永远 false-fail）|
| **control_plane** | OceanBase · KB↔addon role-label race | [`docs/cases/oceanbase/oceanbase-repl-restart-role-label-race-case.md`](cases/oceanbase/oceanbase-repl-restart-role-label-race-case.md)（T10 / T8 双复现 KB exclusive-role-removal × kbagent change-only emit × addon HA auto-failover 三控制环时序）|
| **control_plane** | MySQL · KB controller 镜像渲染多源 | task #3 `evidence-task3.tar.gz/baseline/config-manager-rendering-anomaly/`（`config-manager` sidecar 来自 `ParametersDefinition.spec.reloadAction.shellTrigger.toolsSetup.toolConfigs[].image`，**不在 ComponentVersion 范围**；patch live PD 后才收敛）|
| **control_plane** | Cross-line · OpsDef finalizer 反向死锁 | [`addon-narrow-scope-force-delete-guide.md`](addon-narrow-scope-force-delete-guide.md)（cluster ← component ← instanceset ← pod ← kubelet pulling image 反向锁；窄域 force-delete stuck pod，禁 patch finalizer）|
| **product** | MariaDB · rejoin gate single-shot bug | [`docs/cases/mariadb/rejoin-gate-single-shot-vs-bounded-wait-case.md`](cases/mariadb/rejoin-gate-single-shot-vs-bounded-wait-case.md)（alpha.10 单次 slave_status snapshot 卡 `.replication-pending`；alpha.11 加 30s bounded retry）|
| **product** | Valkey · stale POD_FQDN_LIST | [`docs/cases/valkey/scale-out-targeted-switchover-stale-pod-list-case.md`](cases/valkey/scale-out-targeted-switchover-stale-pod-list-case.md)（scale-out 后 targeted switchover 漏 fresh candidate FQDN → wait_for_new_master 超时；修复用 `pod_fqdns_with_candidate()`）|
| **product** | OceanBase · CREATE STANDBY fetch log timeout | [`docs/cases/oceanbase/oceanbase-repl-create-standby-fetch-log-timeout-case.md`](cases/oceanbase/oceanbase-repl-create-standby-fetch-log-timeout-case.md)（bootstrap 忽略 ERROR 4765 假报 ready，user 租户实际卡 CREATING_STANDBY；修复加 SYS LS NORMAL wait gate + ERROR 重试）|

## 复杂 case 一律 fail-fast，不要混跑

对以下 case，建议默认 fail-fast：

- chaos
- switchover / failover
- backup / restore
- 多步骤 regression
- 带 kill / dual-kill / mid-flight kill 的 case

规则建议固定为：

- 只收当前 first blocker；
- 不继续跑下一个 case；
- 冻结现场；
- 保存 proof、artifact、关键对象和日志；
- 问题线收干净前，不把下一个 case 混进来。

否则很容易出现：

- 后续失败覆盖前一个失败；
- 多个问题线混在同一现场；
- 谁也说不清到底哪条修复解决了什么。

## 现场冻结要明确“哪些保留，哪些可清”

冻结现场不能只说“先别清理”，最好明确成三层：

- **必须保留**：source cluster、restore cluster、Backup、Pending Backup、proof、artifact
- **可受控清理**：阻塞后续验证 slot 的临时对象
- **禁止扩写**：不允许在同一 frozen scene 上继续跑后续 case

这样后续做门禁的人才能知道：

- 哪些对象是证据；
- 哪些对象只是卡位资源；
- 哪些对象可以最小清掉释放入口。

这里还要再补一层：

- **frozen scene 不是万能证据源**

冻结现场的价值，是：

- 保住失败当下还存在的对象、日志、proof；
- 让后续能继续回读当时的 live truth / artifact；
- 避免问题被清理动作二次污染。

但它也有明确边界：

- 如果当前缺的是**当时没有被保存下来的同一时间点快照**
- 或缺的是已经不存在的对象本体
- 例如某条 case 专属 cluster、某个 `OpsRequest.status` 细节、某一时刻每个 pod 本地文件值

那 frozen scene 本身就**不一定补得回来**。

这时最容易犯的错误是：

- 围着现有 frozen 现场反复“再看一遍”

如果当前团队还在保护一套**共享 baseline / prep-only 恢复包**，清理边界还要再收紧一层：

- 只清**已经明确点名可动**的旧样本 / 卡位资源；
- 不动共享 baseline、本轮仍在复用的 prep runner / checklist、proof、artifact；
- 不把“别的引擎 / 别的 case 允许清某个 blocker 样本”误解成“可以顺手清更多 live 资源”。

如果现场里还存在**仍在 Running / Updating、但团队还没把它正式降成 artifact-only** 的样本，默认答案应该是：

- **当前没有新增可直接清理的集群**

而不是模糊地说“可以继续清一点旧资源”。这时要继续回收，必须再补一条明确 source-of-truth：

- 哪几套样本已经脱离当前验证链；
- 是“全部都清”，还是只点名清其中几套。

如果人已经明确下了“继续清”“全部清掉”这类指令，也不要把它执行成模糊的大扫除。更稳妥的落地方式是：

- 把这次批准的**具体样本名单**逐一写清；
- 同时把**仍然受保护、禁止顺手删除**的对象也写清。

这样后面才能回头审计：这次是一次**有明确范围的人工批准回收**，而不是默认安全边界被悄悄放宽。

这条规则的目的，不是保守，而是避免把：

- 还在共享验证链上的控制面对象
- 后续复验还要依赖的 gate / proof 包
- 以及另一个问题线仍在使用的恢复入口

一起误删掉，最后把 cleanup 动作本身变成新的 blocker。

### 资源窗口紧时，默认把 live scene 降成 artifact-only

还有一种常见情况是：

- 测试环境资源已经明显吃紧；
- 团队同时还在跑别的验证线；
- 继续长期保留 live scene，本身就会反过来制造新的 scheduler / PVC / slot blocker。

这时更稳的默认口径应切成：

- **先保 artifact / logs / 关键对象快照**
- **再把 live scene 降成 artifact-only 并尽快清理**

也就是说，默认答案不再是“先把样本留着”，而是：

- 只要关键证据已经固化，live scene 就应该释放；
- 后续如果还要继续验证，优先用新样本重放；
- 只有在明确说明“为什么这套现场不可替代”时，才允许例外保留。

这里最好固定再补一句人工判断：

- **保留 live scene 需要额外理由；清成 artifact-only 才是默认路径。**

这类额外理由通常至少要满足其一：

1. 当前问题线必须复用同一现场的连续状态，重新起样本会破坏关键证据；
2. 已知缺失的关键证据只能从这套 still-live 对象里补；
3. 人已经明确点名“这套样本先不要清”。

如果以上理由都没有，就不要把“也许以后还用得上”当成长期占用资源的依据。

一句话说：

- **资源紧张时，live scene 默认 artifact-only；保留需要单独论证。**

一旦团队把执行口径正式切成这一步，后续现场管理也要同步收紧：

- 不能一边说“默认 artifact-only”，一边又让大多数 live scene 因为“暂时还没删”而被默认保留；
- 不能把 `Running` / `Updating` 自动等同于“这套必须继续保现场”；
- 也不能让“要不要删”在每一轮都重新回到模糊讨论。

更稳的落地方式是固定两张名单：

1. **默认要降成 artifact-only 的 live scene**
2. **被明确点名为例外、必须继续保 live 的样本**

这样后续清理、重测、窗口释放时，团队就能直接对照：

- 这套样本是不是已经在默认回收名单里；
- 还是已经被明确提升成“必须保现场”的例外。

否则最常见的后果不是“多保留了一点资源”，而是团队在同一条环境边界上反复摇摆，最后把 cleanup 本身变成新的 blocker。

### artifact-only cleanup 不能早于目标取证点

把 live scene 降成 artifact-only 是资源紧张时的默认口径，但这个 cleanup 仍然要有明确触发点。最容易出错的是在定向验证脚本里写一个全局 `trap cleanup EXIT`：

- 样本刚到 `Running`；
- 还没发目标 `OpsRequest`；
- pre-op 取证或拓扑探测先 timeout；
- `trap` 立即把 live scene 删除；
- 最后这轮既没有目标 case 结果，也失去了可继续补证的现场。

这类结果不能写成目标 case 的代码结论。它只能算：

- **validation-entry / runner cleanup bug**

更稳的做法是把 cleanup gate 分成两段：

1. **目标动作发出前**
   - 只允许保存 pre-op blocker；
   - 不要自动删除 live scene，除非样本坐标错误、资源创建错误，或人明确要求清理；
   - 如果只是 pre-op capture timeout，优先修 capture timeout 后沿同一样本继续。
2. **目标动作发出后**
   - 已经拿到目标 action / runtime / status 证据，或已经确认目标动作无法进入；
   - 再执行 artifact-only cleanup；
   - cleanup 结果也要写入 artifact，说明 live 是否已删除、namespace 是否还在 `Terminating`。

一句话说：

- **artifact-only 是证据收齐后的资源回收策略，不是 pre-op 脚本失败时的无条件删库按钮。**

### cleanup 后只能用 artifact 复盘，不要再用 live 查询补结论

当样本已经按 artifact-only 边界清理后，namespace / cluster / workload 可能已经不存在。这时后续复盘的 source-of-truth 要切换到 artifact：

- run-entry / runner log；
- OpsRequest / ComponentParameter / ConfigMap 的 yaml 快照；
- controller / kbagent / pod 日志；
- runtime truth / live hash / instance status 的提取文件；
- cleanup 记录，例如 namespace 是否已删除、是否还有 PVC / workload / rolebinding 尾巴。

不要在 cleanup 后继续用当前 `kubectl get` 的空结果去推断“当时没有问题”。`kubectl` 只能证明现场已经被删，不能替代当轮 artifact 里的运行时事实。

更稳的写法是：

- **live 已清理**：当前环境查询只用于确认资源释放；
- **结果复盘**：只引用 artifact 里冻结的 Ops / status / runtime / log。

一句话说：

- **清理后的 live 空结果不是测试结论；artifact 才是当轮结论的证据源。**

如果 cleanup 后还剩一个 `Active` namespace，但其中已经没有 cluster / workload / PVC / Ops / ComponentParameter，只剩默认 `kubernetes` finalizer，要单独记成 cleanup hygiene tail：

- 它不是“测试仍在运行”；
- 它也不是“case 还有新结论”；
- 它只说明资源回收链还留下一个空 namespace，需要由环境/清理 owner 决定是否继续处理。

不要因为这个空 namespace 把测试状态误报成 active，也不要反过来用它覆盖 artifact 里的失败/通过事实。

如果 namespace 里没有 cluster / workload / Ops / ComponentParameter，但还剩 PVC / backup repo 这类资源，也不要把它误报成 active test run。它应先被分类为 **PVC-only cleanup residue**：

- 它会继续占用存储或调度资源，所以要单独列入 cleanup 候选；
- 它不是当前 case 的运行状态，也不能证明测试仍在推进；
- 是否删除要由环境 / 资源 owner 或明确的人类指令确认，不能在产品线巡检里顺手清。

巡检时要同时看 `cluster/pods/ops/componentparameters` 和 `pvc`。只看控制面对象可能会漏掉 PVC-only 尾巴；只看 PVC 又可能把历史资源误判成当前测试。

### Namespace 对象先消失但 PVC 还挂在 API 时的最小 unblock 步骤

清理过程里会偶尔遇到一种少见但容易卡很久的现场：

- `kubectl get pvc -n <ns>` 还能列出 PVC（带 `kubernetes.io/pvc-protection` finalizer）；
- `kubectl get cluster/pods -n <ns>` 显示已经没有 workload；
- 但 `kubectl patch pvc <name> -n <ns> --type=merge -p '{"metadata":{"finalizers":[]}}'` 报 `namespaces "<ns>" not found`。

直接读：**Namespace 对象本身已从 API 索引中消失，但其下的 PVC 还挂在 API**，于是带 `-n <ns>` 的 patch 走不通。这种现场往往伴随着：

- CSI driver 之前抖动过（PVC 删除被 driver 提前卡住）；
- Namespace 删除流程在 PVC protection finalizer 被释放前异常推进；
- driver 恢复后，PV controller 也无法给一个"已不存在"的 namespace 名称做 reconcile。

排查/处理顺序（对环境 owner 是低风险操作，但仍要先看 PV）：

1. **先看 PV 是否真有底层卷**：
   - `kubectl get pv | grep <ns-name-fragment>` 看 ReclaimPolicy / Status / Claim 字段；
   - 如果 PV 仍 `Bound` 且有合法 volume handle，说明只是 finalizer 卡住，不是 underlying volume 丢；
   - 如果 PV 已 `Released` / 找不到，需要先确认产品/数据没有可恢复的预期。
2. **重新创建空壳 namespace**（最小 unblock）：
   - `kubectl create ns <ns>`；
   - 这一步把 namespace 对象补回，patch / delete 才能再次 resolve。
3. **逐个 patch PVC 去掉 finalizer**：
   - `kubectl patch pvc <name> -n <ns> --type=merge -p '{"metadata":{"finalizers":[]}}'`；
   - PV controller 会立刻把对应 PV 摘除并按 ReclaimPolicy 处理底层卷。
4. **复核**：
   - `kubectl get pvc -A | grep <ns-name-fragment>` 应为空；
   - `kubectl get pv | grep <pvc-uuid>` 应为空；
   - `kubectl get cluster -A | grep <ns-name-fragment>` 应为空。
5. **删空壳 namespace**：
   - `kubectl delete ns <ns>`；
   - 状态应快速进入 `NotFound`。

替代路径：用 `kubectl replace --raw` 直接 PUT 到 PVC 资源 URL 去 finalizer，不依赖 namespace 对象存在。但 raw URL 写法对 apiVersion / cluster-scoped 路径敏感，调试比 `create ns` 补洞贵。

排查口径上要先把这条与"PVC-only cleanup residue"分开：

- **PVC-only cleanup residue**：namespace 还在，只剩 PVC / backup repo；普通清理流程能处理；
- **phantom namespace + orphan PVC**：namespace 对象已不在 API，PVC 还挂在 API；普通 `-n <ns>` 操作必失败，需要补 namespace 或走 raw 路径。

一句话说：

- **`kubectl patch -n <ns>` 报 `namespaces not found` 但 `kubectl get pvc -n <ns>` 仍能列出资源时，先 `create ns` 把 namespace 对象补回，再清 PVC finalizer，最后删空壳 namespace；不要等 driver 自愈。**

### frozen scene 只保已有证据，缺口要用最小补证样本

有些现场虽然已经被冻结或 artifact-only，但它本来就不包含接下来要的关键证据。更稳的做法是先明确回答：

1. 当前 frozen 现场里，缺的证据能不能直接拿到
2. 如果不能，是不是应该立即起 **1 套最小补证样本**
3. 最小样本只抓缺的那组快照，不顺手扩别的 case

一句话说：

- **frozen scene 只保你已经留住的证据；缺同一时间点快照时，不要空等，直接起最小补证样本。**

## existing-datadir / GTID 分叉场景要 fail-closed

如果已有 datadir 的 rejoin 场景下发现：

- GTID divergence
- strict-mode conflict
- 无法证明自己可安全 follow

那就应该：

- 明确 fail-closed
- 不要 follow 假成功
- 不要节点假 ready
- 不要为了“看起来恢复了”继续往下跑

否则后面看到的“闭合”可能只是更大的数据一致性问题被推迟暴露。

这里还要再补一层：

- **fail-closed 不只是最终状态，也包括 startup ordering**

对 existing-datadir 的 restart / rejoin 场景，正确顺序应是：

1. wrapper 先完成 GTID 判定；
2. 把状态收敛成：
   - `.replication-ready`
   - 或 `.replication-divergence-pending`
3. 只有在 ready 分支下，后续 HA / syncer 才能继续：
   - follow success
   - `CHANGE MASTER ... slave_pos`
   - `secondary` publish

如果在 GTID 判定完成前，后续路径就已经先跑：

- follow success
- `CHANGE MASTER ... MASTER_USE_GTID=slave_pos`
- publish `secondary`

那即使最终表面上看像“节点起来了”，本质上仍是：

- **startup ordering leak**

这类问题不要收成新的 stop/start 家族问题，也不要只写成“replication resume 没恢复”。

更准确的收法是：

- 先按 existing-datadir / strict-divergence 家族继续收窄；
- 再明确是不是 **GTID 判定前的 follow/publish 抢跑**。

这里还要再把这类 failover/rejoin case 的最终验收语义单独写清：

- 如果已经确认这是 **true divergence hit**
- 并且产品语义已经拍板：`fail-closed pending` 是预期行为

那对应的 failover/rejoin case（例如 `C1`、`C2`）验收就不应再要求：

- 原来的失效 / 被替换节点自动回成 `secondary`
- replication 在同一条 `C1` 结果里重新闭合

更准确的拆法应是两步：

1. **原始 failover/rejoin case 本身**
   - 先按 true divergence hit + fail-closed pending 收口
   - 不把原节点没自动回 `secondary` 继续记成普通 replication-resume 缺口
2. **后续恢复能力**
   - 单独起一条 `rebuild OpsRequest` 验证
   - 再看原节点经 rebuild 后能否恢复成 `secondary`
   - replication / data 是否重新闭合

一句话说：

- **true divergence 命中后，原始 failover/rejoin case 先按 fail-closed pending 收；原节点后续能否回成 `secondary`，改由单独 rebuild 线验证。**

## divergence 判定要持久化 source-of-truth

如果一个 case 的验收结论依赖 GTID / divergence 判定，不要只保留最终 marker、最终 phase 或一句“命中 fail-closed”。

这类判定应该像一个可审计的决策一样持久化当时的输入和分支，至少包括：

- 当前节点身份、时间戳、component / pod 名称
- 当前节点的 local GTID / executed GTID / slave position
- 当时 active primary 的身份、primary GTID / position
- 当时的 slave status / replication status 快照
- 参与判定的 pending marker / 本地状态文件
- 判定命中的分支：ready、generic pending、divergence pending、rebuild required
- 具体判定代码分支或日志关键字

这样下一轮复验时才能区分：

- 确实命中了 true divergence
- 旧状态 / 缓存状态导致误判
- primary 身份切换期间读到了错误对象
- slave status 或本地 marker 不一致

这条规则不改变 fail-closed 语义。

它只要求 fail-closed 有可复核的判定账本，避免下一轮只能根据最终结果反推原因。

一句话说：

- **fail-closed 要保留判定账本；否则下一轮只能看结果猜分支。**

## divergence 判定不要只依赖 service VIP 单次取样

如果 divergence 判定需要读取 active primary 的 GTID / position，不要只记录：

- service 名称
- 一次通过 service VIP 读到的 GTID

service VIP 在 endpoint 切换、旧连接复用、DNS / iptables 延迟、客户端连接池复用时，可能短时间指向不符合当前预期的实例。

这时单次 VIP 取样容易把采样抖动误判成数据分叉。

更稳妥的取样方式是让 primary evidence 能回答：

- **这个 GTID 到底来自哪个 pod / 实例？**

建议至少补充：

- service VIP 当时解析 / 路由到的 endpoint pod；
- 数据库侧 `server_id` / hostname / pod identity；
- `Master_Host`、active primary、endpoint resolved pod 三者是否一致；
- 短 retry 中每次的 resolved endpoint、server_id、GTID；
- retry 内 primary identity / GTID 是否稳定。

如果短 retry 内看到 primary identity 或 GTID 来源不稳定，优先收成：

- **primary sampling instability**

而不是直接定性为 true divergence。

一句话说：

- **divergence evidence 不能只说“我从 service 读到了这个 GTID”；还要说明“这个 GTID 来自哪个实例”。**

## 动态参数 case 必须验 runtime truth

对 reconfigure / dynamic parameter 相关 case，不要只看：

- `ComponentParameter`
- `ConfigMap`
- `OpsRequest Succeed`
- `Cluster Running`

最终还要看：

- 业务进程 runtime 是否真变成新值

如果控制面对象都对，但 runtime 还是旧值，这更像：

- live-apply 路径没执行到位
- 参数注入 / 传值链路有问题
- reload 脚本“假成功”

这类问题应直接归到 live-apply 路径，而不是继续加长等待时间。

如果进一步出现：

- `OpsRequest` 已经 `Succeed`
- `ComponentParameter` 已经 `Finished`
- live hash / instance status 已经 `N/N target`
- 但逐 pod runtime 仍是 `N-1 / N` 或 `1 / N` split

那不能把这轮写成“参数修改通过”。更稳的回报方式是把它拆成两句话：

- **高层控制面已闭环**
- **runtime truth 仍未闭环**

然后继续往 per-pod apply / action source / config projection / runtime proof 方向收，不要让高层终态掩盖 runtime split。

反过来也一样：如果逐 pod runtime 已经全部到目标值、live hash / instance status 也已经 `N/N target`，但 `OpsRequest` 或 `ComponentParameter` 仍停在 `Running / Retry / reconfiguring`，也不能直接写成“case 已通过”。这说明 runtime 已闭环，但控制面的 terminal contract 还没闭环。

这类结果应拆成：

- **runtime truth 已闭环**
- **control-plane terminal contract 未闭环**

后续应该继续看 status/reconcile/condition 收口路径，而不是再回头猜 runtime apply 失败。

## 测试资产统计要固定三层口径

后续汇报“总共有多少 cases”时，建议固定区分：

### 当前主回归集合

例如当前这轮真正正在跑的 smoke + chaos。

### 主入口会跑的 suites

例如 `run-tests.sh -t all` 会包含哪些 suites。

### repo 全量测试资产

包括 standalone / 定向补验 / diag / repeat / stress 这类不一定在主入口里的脚本。

如果不分这三层，就会反复出现：

- “当前回归是 31 个”
- “repo 里明明不止 31 个”
- “all 会跑 14 个 suites”

## 模型强度按决策风险分配，不按角色平均分

Addon 项目组通常会同时存在三类工作：

- 技术判断与窗口控制
- 代码 / 测试口径的最小修改
- 样本执行、日志收集和 proof 闭环

如果可用的强模型资源有限，不建议按角色平均分，也不建议默认把最强模型给“正在跑测试的人”。更稳的原则是：

- **谁承担最高的错误决策代价，谁优先用最强模型。**

通常优先级是：

1. **TL / 技术负责人**
   - 负责 first blocker 定性、source-of-truth 取舍、共享 baseline 窗口、任务拆分；
   - 一次错误判断会让 Dev 和 Test 都浪费轮次；
   - 因此最适合长期使用最强模型。
2. **Dev / 实现负责人**
   - 在复杂代码路径、最小 patch / cut 设计、埋点方案和跨组件契约判断时，适合使用强模型；
   - 简单模板改动、机械整理、已定方案落地可以降级。
3. **Test / 验证负责人**
   - 日常 runner 执行、proof 整理、日志归档更依赖纪律和稳定工具；
   - 复杂多日志归因、chaos / 概率性问题、埋点设计时再临时升到强模型。

不要把“强模型”理解成所有关键角色都必须使用同一档模型。更稳的做法是按职责选择不同模型 profile：

- **TL / 技术负责人** 更需要强推理、长上下文、低幻觉和保守判断，用来读关键路径、判断 first blocker、守共享 baseline / live window，以及复核最小 cut 是否打偏。
- **Dev / 实现负责人** 更需要强编码、repo navigation、局部实现和快速迭代能力，用来把 TL 定下的 cut 落成 patch / 埋点 / tests-side helper。
- **Test / 验证负责人** 更需要稳定执行、日志整理和 proof 闭环能力，只有在测试口径本身复杂或需要独立多日志归因时才临时升档。

TL 和 Dev 都读代码不一定浪费；浪费发生在两者用同一深度、同一目的、同一时间重复读同一段代码。合理边界是：

- TL 读到能判断方向、边界和风险；
- Dev 沿 TL 拍板的最小 cut 深入实现细节；
- TL 最后抽样 review patch 和 proof 是否仍命中原验证目标。

一句话说：

- **最强模型优先给会改变整个项目方向的人；执行侧按问题复杂度临时升档。**

## 工作区里“已经改了”，不等于这轮测试“实际命中了改动路径”

排查测试失败时，经常会看到这种说法：

- 工作区文件已经改了
- 本地脚本里已经有新的等待
- 某个 patch 明明就在当前分支里

但这还不能直接证明：

- 当前这轮测试进程实际跑到的，就是这份已修改文件

尤其在下面这些场景里，更容易误判：

- 测试仓库是本地未提交修改
- 当前分支还是 `main`
- 运行入口可能来自另一份工作区 / 另一台 runner / 另一套 artifact
- log 里没有把关键 helper / wait path 打印出来

对这类问题，建议把判断拆成两层：

1. **工作区事实**
   - 文件里现在写了什么
   - 本地分支是什么
   - 改动是否已提交 / 已推送
2. **执行事实**
   - 本轮 run log 里实际执行了什么
   - 关键 helper / wait path 是否真的被调用
   - artifact 里有没有能直接证明执行路径的片段

如果失败点刚好是：

- “本应先等 stable state / cluster running”
- “理论上已经补了 wait”
- 但最终日志仍表现得像没等就直接执行了下一步

那第一优先级不是继续猜 addon 问题，而是先把这件事钉死：

- **本轮到底有没有真的跑到 patched path**

更稳的做法是：

- 不只报“工作区版本是 patched”
- 还要从 run log / artifact 里截出关键片段
- 明确证明：
  - 是否命中了新的 helper / wait path
  - 还是根本没跑到这段逻辑

一句话说：

- **workspace state 不等于 execution path，run log 才是执行事实。**

## controller baseline 漂移时，先停下解释 case 行为

有些回归失败不是因为 case 自身暴露了产品问题，而是测试执行期间控制面 baseline 发生了漂移。

典型形状是：

- run 开始前确认过 controller image / commit；
- run 执行中 `Deployment` 又发生 rollout；
- 后续失败刚好落在 roleProbe、Ops terminal、reconfigure、scale/decommission 这类控制面强相关路径；
- 但当轮 artifact 里无法证明失败发生时跑的就是预期 controller binary。

这时不要急着解释业务 case 行为。第一优先级是把控制面 baseline 重新钉死：

1. **目标 baseline**
   - 目标 image tag；
   - image digest / pod `imageID`；
   - binary build info / git sha / patch commit；
   - 关键 patch 的 `git show --stat` 或等价可审计入口。
2. **live baseline**
   - 当前 `Deployment` image；
   - 当前 controller pod；
   - pod start time；
   - pod `imageID`；
   - `Deployment.generation` / `resourceVersion`。
3. **rollout 来源**
   - `Deployment` events；
   - ReplicaSet owner；
   - rollout history / change-cause；
   - 是否有 CI、手工命令、其他任务或脚本同时在写 controller deployment。

如果无法确认 rollout 来源，就要把来源写成 `unknown`，不要用推测补。

更重要的是：只要 drift 发生在目标 case 之前或执行过程中，就不能直接用这轮结果判断目标 patch 是否有效。应先恢复成单一、可审计 baseline，再做最小复验或 full-suite rerun。

一句话说：

- **controller baseline 不可审计时，先收 baseline drift；不要把漂移中的结果解释成 addon / roleProbe / Ops contract 结论。**

## 单 case 定向复验通过，不等于全量回归已经闭环

还有一种很常见的误判是：

- 某条最小 directed rerun / single-case verify 已经 PASS
- 团队就自然把它理解成“这条修复已经完全闭环”

但这只能证明：

- 在那一套最小前置条件下
- 你验证的那条局部假设成立

它**不能自动证明**：

- 放回完整 suite 后不会再次失败
- 不会被前序 case 的状态、对象残留、时序、角色切换、作用域或执行入口影响

更稳的口径应该固定分两层：

1. **local fix verified**
   - 最小定向复验通过
   - 说明当前修点至少对准了一个真实问题
2. **full-suite regression closed**
   - 放回完整回归后仍通过
   - 才能说明这条修复没有被 suite 顺序 / 状态耦合重新击穿

如果出现这种现象：

- 单 case 定向复验 PASS
- 但 full-suite rerun 又在同类 case 上 FAIL

那更合理的解读通常不是“前面的定向复验白做了”，而是：

- 这条 fix 只覆盖了局部问题
- full-suite 暴露了新的前置条件 / 顺序耦合 / 状态交互
- 需要把“局部修复是否有效”与“全量回归是否已闭环”分开汇报

这时建议固定这样写结论：

- **directed rerun 证明了什么**
- **full-suite 又暴露了什么**
- 两者是不是同一个问题，还是同一症状下的不同前提

不要把：

- “最小复验通过”

直接压缩成：

- “整条问题线已收口”

一句话说：

- **single-case pass 证明局部正确；full-suite pass 才证明回归闭环。**

这类口径混淆。

## 同一检查点下结果分裂时，不要先怪等待窗口

还有一种常见误判是：

- single-case 定向复验里，某条 runtime 检查通过；
- full-suite 里，在**同一个检查点**做同样的检查，却失败；
- 然后大家第一反应还是“是不是再多等一会儿就好了”。

如果当前已经能证明：

- 命中的就是同一版 live `cmpd` / `reconfigure.exec`
- runtime 检查发生在同一个时点
  - 例如都是 `wait_ops(reconf)` 返回后立刻检查
- 失败现象不是整体都旧，而是按 pod 分裂
  - 例如 `1 pod` 旧值、`2 pod` 新值

那下一优先级通常就不该再停留在：

- 等待窗口再放大一点
- 先猜版本/入口没命中

而应该先往：

- **state interaction / execution context**
- **pod-local file projection / config refresh timing**
- **live-apply 执行路径在不同 pod 上命中的时点差异**

这类方向收窄。

更稳的补证顺序是按 pod 对齐同一时间点：

1. 本地配置文件里的目标值
2. runtime live 值
3. action / reconfigure 的时间戳

一句话说：

- **同一检查点下 single-case 能过、full-suite 仍分裂时，先查状态交互和按 pod 时点差异，不要先把问题收成“多等一会儿”。**

## 定时巡检要区分“有效等待”和“无人推进”

项目运行到多条验证线并行时，定时巡检不能只问“有没有新消息”或“有没有测试进程”。更重要的是判断当前等待是否有明确 owner、明确 blocker、明确下一跳结果面。

也不要只依赖群消息或 task board。测试 runner 可能已经在本机或共享环境里启动，但还没来得及回报结果。巡检时至少要交叉核对：

- 最新群消息 / DM source-of-truth；
- task board 是否有新 owner 或 blocker；
- 本地 runner 进程、artifact 目录、log path 是否出现新样本；
- live cluster / Ops / ComponentParameter 是否已经进入目标结果面。

如果交叉核对发现的是**别的产品线 / 别的 owner 的 runner**，不要直接接管、清理或替它下最终结论。更稳的处理方式是：

- 只记录事实：runner、sample、Ops、当前 phase、artifact path；
- 明确它不属于当前产品线的可操作范围；
- 等该 owner 回正式 source-of-truth 后，再决定本产品线窗口是否释放。

这样可以避免 TL 在巡检时把“旁路观察到的现场”误当成自己可以处置的任务。

如果这个 runner 是在 owner 还没正式回报前被本地巡检发现的，还要把它标成 **local observation / provisional**。本地观察可以说明“测试确实在跑”，但不能替代 owner 的结论，也不能直接变成其他产品线的 release signal。

如果本地巡检随后看到同一 artifact 已经出现 terminal 结果，也仍然要保持这个边界。可以把结果报清楚，但要分成两层：

- **local artifact result**：例如 OpsRequest / ComponentParameter / live hash / runtime 是否已经同时闭合；
- **owner source-of-truth**：是否由该产品线 owner 接受为最终结论，是否释放共享 baseline / 其他产品线窗口。

不要因为本地 artifact 显示 `Succeed / Finished / runtime N/N target` 就代替 owner 宣布“问题已解决”或“其他线可以开跑”。更稳妥的说法是：

- 先引用 artifact 路径和四层闭合状态；
- 明确这是本地巡检看到的 provisional 结果；
- 等 owner 回正式 source-of-truth 后，再改变共享窗口。

这样既不会漏报真实进展，也不会把旁路巡检升级成越权放行。

如果 bounded recovery helper 的 ledger 里大部分 probe 都是 Kubernetes API / vcluster route read failed，例如 `127.0.0.1:<port> refused`，不能把最终 `primary_count=0/secondary=none` 或 `role snapshot read failed` 直接写成产品 runtime truth。它首先说明 observation path 不可判定，需要把 API route continuity 和 runtime truth 分开固定。

这类收口至少要同时保留：runner fail line、每轮 probe ledger、port-forward/lsof/route continuity、kubectl stderr、Cluster/Component/role/endpoint/DB truth、pod/controller/kbagent 日志、divergence marker/log。只有证明 API 路径稳定且成功观测到真实 non-closed state，才能升级为 runtime blocker；否则先收成 probe/route indeterminate 或测试观察边界。

一句话说：

- **bounded recovery 失败如果被 API refused 主导，先收 route/probe truth，不要把 indeterminate ledger 反写成产品状态。**

如果后续多个巡检周期都只有这份 provisional artifact、没有新的 owner 回复，也不要每轮都重复催促或把同一份 artifact 包装成“新进展”。更好的回报方式是：

- 固定上一次 owner-confirmation request 的时间；
- 固定当前没有 active runner / 新 artifact / 新 source-of-truth；
- 固定阻塞仍然是“等待 owner 接受或拒绝这份 artifact”；
- 只有出现新 runner、新 artifact、owner 回复、或等待明显超时，才升级处理。

这样可以减少噪声，同时保留项目没有被遗忘的可审计状态。

如果这种等待跨过多个巡检周期，要把 **artifact age** 和 **owner-confirmation age** 分开记录：

- artifact age：这份证据是什么时候生成 / 冻结的；
- owner-confirmation age：什么时候第一次请求 owner 判断它能否成为 source-of-truth；
- escalation point：超过团队约定时限后，是再次点名 owner，还是由 TL 提议降级为 artifact-only / 释放其他产品线窗口。

不要把“artifact 已经很老”误读成“owner 已经接受”，也不要把“owner 暂未回复”误读成“测试还在运行”。这两个时钟分开，才能同时避免误放行和误报 idle。

同时要避免每个巡检周期都重复催同一个 owner。更稳的节奏是：

- 第一次发现 provisional artifact 后，请 owner 做 source-of-truth 判断；
- 等待超过约定窗口后，做一次最小二次确认；
- 如果仍无回复，后续巡检先记录“二次确认已发、仍无 owner 回复”，不要继续刷屏；
- 只有出现新证据、owner 回复、或等待超过需要 human/TL 升级的阈值时，再升级处理。

这样既保留推进压力，也避免把巡检变成低信号重复 ping。

同样，不要把历史保留现场误报成“测试正在运行”。如果 live cluster / pod / old Ops 只是为了保留 proof、备份现场或受保护样本而存在，且没有当前 runner、没有新 Ops、没有对应 owner 正在等待它的结果，它应被报成：

- **protected / frozen live scene**

而不是：

- **active test run**

这条也适用于受保护现场里的 `Pending` pod。比如历史备份 pod 或保留 proof 的 workload 还在 `Pending`，但没有当前 task、runner、Ops 或 owner 正在等待它的结果时，它不是“测试正在跑”，也不是新的环境 blocker。更准确的分类是：

- **protected artifact / scene residue**

只有当这类 pod 被当前验证入口引用，或者 owner 明确说正在等它收敛时，才把它升级为 active blocker。

判断测试是否在跑，要看当前 task 绑定的 runner、fresh sample、正在推进的 Ops / log，而不是只看集群是否还活着。

进程检查也要防止误判。`ps | grep/rg` 可能命中历史搜索命令、IDE sandbox、旧 shell、或一次性 `kubectl get`，这些都不是 active runner。把进程判成“测试正在跑”前，至少要能同时对上：

- runner 脚本路径；
- sample / namespace / cluster 名；
- artifact 或 run log 正在生成 / 最近更新；
- live Ops / ComponentParameter / pod 状态属于这次样本。

如果只看到一个长期存在的 `grep`、`kubectl`、IDE helper 或无 sample 绑定的 shell，就只能报成“无有效 runner”，不能当成测试活动。

连续多轮巡检都没有新消息、新 runner、新 artifact 时，也不要为了“有动作”而制造新任务或重跑。更稳的汇报方式是固定三件事：

- 当前没有新的 source-of-truth；
- 当前等待仍然有明确原因和释放条件；
- 下一步只在释放条件满足后执行。

如果等待条件已经不明确，才需要升级成“无人推进 / 需要重新指派”的问题。

如果等待条件本身明确、但 source-of-truth 已经长时间没有刷新，也要把“等待年龄”报出来，并向对应 owner 要一次状态确认，而不是无限重复同一句等待中。例如：

- 最后一次权威结论是什么时间；
- 当前等待哪条结果；
- owner 是否还在执行，还是已经转成 artifact-only / env blocker / 可释放窗口；
- 如果 owner 没有下一步，是否可以释放被阻塞的其他产品线。

这类确认不是接管对方任务，而是防止“有效等待”慢慢退化成无人推进的隐性 idle。

建议把巡检结论拆成三类：

1. **测试正在运行**
   - 需要固定 runner / sample / 当前 case / log path；
   - 如果长时间无结果，要点名当前卡在 Ops、pod 调度、cluster phase，还是脚本等待。
2. **测试没有运行，但等待是有效的**
   - 需要有明确原因，例如共享 baseline 被占用、setup-flow blocker 未分线、static gate 未释放；
   - 需要有明确 owner 和下一条 source-of-truth，例如等待 Dev 的最小 cut、Test 的 first-apply 结果、或环境管理员的 cleanup 结果。
3. **测试没有运行，且没有明确 owner / 下一跳**
   - 这就是项目空转；
   - 应立即创建或认领任务，把下一跳变成可执行动作。

如果测试没有运行是因为 TL 明确冻结了 live 面，例如共享 baseline 未释放、只允许 prep-only、或等待其他产品线 owner 接受 source-of-truth，那么“没有 runner”本身就是符合边界的健康状态。这时巡检不要把它升级成 idle，也不要为了满足“测试在跑”而启动新 runner。更准确的回报是：

- 当前 live runner = none；
- 这是预期状态，因为 freeze / prep-only 边界仍有效；
- release condition 是什么；
- release 后第一个可执行入口是什么。

这样能把“按边界不跑”和“没人推进”区分开。

如果巡检时发现共享 controller / baseline 已经变化，但聊天里还没有 owner 的 source-of-truth，也不要把这当成默认释放信号。应先把它记成 **unannounced shared-baseline drift**：

- 固定 live image / imageID / pod / generation / Ready 状态；
- 说明这只是本地观察，不是 owner 接受的窗口结论；
- 冻结依赖该 baseline 的产品线，不要顺手启动 rerun；
- 向 baseline owner 要一次最小确认：这版 gate 属于哪条问题线、是否稳定、是否允许其他产品线进入 static gate。

否则最容易出现的错误是：看到 controller 已经切新镜像，就误以为“窗口已经释放”，但实际上它可能仍是另一条验证线的 validation-only gate。

如果下一轮巡检发现这版 image / imageID / generation 没有变化，但 owner 仍未确认，也不要把同一条 drift 当成新进展重复上报。更准确的写法是：

- baseline drift unchanged；
- owner confirmation pending since `<time>`；
- dependent rerun remains blocked；
- no extra ping unless image 再次变化、出现新 artifact、或等待超过需要升级的阈值。

如果这种状态连续持续多轮，巡检要把“等待年龄”显式写出来，并提前定义升级阈值。例如超过团队约定窗口仍没有 owner 回复时，升级为“需要 TL/human 决策是否继续等待、回滚、或切换独立环境”，而不是无限期静默等待。

当共享 baseline owner 或 human 明确说“其他团队可以开始”时，也不要把它理解成“立刻跑完整 live case”。更稳的释放方式是分两段：

1. **static-gate-only release**
   - 只读取当前 controller image / imageID / Ready / generation；
   - 检查 fresh slot、历史残留、runner/proof 路径；
   - 不创建新 workload，不发新的 Ops，不重启 controller。
2. **case execution release**
   - 只有 static gate 被 TL 接受后，才进入最小 case；
   - 优先只跑当前 blocker 的最小复验，不直接扩 full suite；
   - 证据收齐后按 artifact-only 清 live。

这能避免把“上游不再占用”误读成“本线所有前置条件都已干净”。尤其当 controller image 已经变成另一条 validation-only gate 时，第一步必须是 gate 回读，而不是直接启动 runner。

如果上游团队说“可以开始”，但同时还有一个样本正在当前 controller gate 上取证，那么下游只能做不会改变 controller 身份的工作：

- read-only 分析；
- static gate / fresh slot 检查；
- runner / artifact 准备；
- 不写 `kb-system/kubeblocks`；
- 不切 controller image；
- 不启动依赖另一个 controller baseline 的 live rerun。

等上游样本结束，或者双方明确 announce gate 切换窗口后，下游再接管 shared controller。不要让两个产品线在同一份 artifact 期间交替覆盖 controller 身份。

static gate 也要按字段拆开判定，不要只看其中一项通过就放行。例如 fresh slot 已经干净，只说明当前前缀下没有历史 cluster / pod / PVC / Ops tail；如果 controller image / imageID 仍然不是本轮 runner 绑定的 baseline，这轮 gate 仍然是不干净的。正确处理是：

- 把 `fresh slot clean` 和 `manager baseline drift` 分开回报；
- stop reason 写成 image / imageID drift，而不是环境残留；
- 不启动 runner，不发最小 case；
- 等 owner 明确接受当前 controller 作为本线 baseline，或明确允许切回本线绑定 baseline 后，再重新跑 static gate。

这样可以避免一个常见误判：资源前缀看起来干净，就把依赖旧 controller 语义的 runner 直接跑在另一条产品线的 controller gate 上。

static gate 结果也有时效性。只要共享 controller 后续又换了 image / imageID / generation，上一轮 gate 的结论就只能当历史证据，不能继续当当前执行许可。下一轮恢复前必须重新读取：

- 当前 deploy image；
- 当前 manager pod 和 imageID；
- generation / observedGeneration；
- Ready 状态；
- 本线 fresh slot。

如果 image / imageID 再次漂移，阻塞点要更新成“新的 shared-baseline drift”，并重新找 owner 确认。不要拿 30 分钟前的 static gate 结果去解释当前 live controller 下的 case 行为。

如果切目标 controller image 时出现 `ErrImagePull` / `ImagePullBackOff`，先不要把它解释成 controller 代码不可用，也不要直接换另一版 baseline。应把 image 可用性拆成三层：

- registry 是否存在该 tag；
- 本机 Docker 是否已有该 image；
- k3d 节点 containerd 是否已导入该 image。

如果本机 Docker 有、节点 containerd 没有，最小下一跳是 **只导入 image 到 k3d 节点** 并用 `crictl images` 验证；不要顺手改 Deployment 或启动测试。导入完成后再重新走 controller 切换和 static gate。这样能把“image 分发问题”和“controller baseline 语义问题”分开。

这类 image-import blocker 也需要看任务所有权。`ErrImagePull` 已经被收成“需要导入 image”后，如果导入任务还停在 todo / 未认领，那么当前状态不是“测试等待中”，而是“执行 owner 未接手”。巡检时应明确写出：

- 当前唯一 blocker 是 image import；
- import 任务是否已 claim；
- controller 是否已恢复到可用 image；
- runner 是否仍未启动；
- 如果任务未认领，需要点名 owner claim 或说明不能处理。

这样能防止项目在“已经知道下一步是什么”的情况下空转。

一句话说：

- **没有测试在跑不一定是停滞；但没有 owner、没有 blocker 名称、没有下一跳结果面，就是停滞。**

## 建议固定回报的证据项

每次 first blocker 收口时，建议至少固定回报：

1. case 名和入口脚本
2. 成功语义属于哪一层
3. first blocker 属于哪一类
4. 冻结现场对象
5. artifact / proof 路径
6. 当前 cluster / endpoint / replication / data truth
7. 是否需要 fresh scene
8. 谁接下一步

## 一句话准则

- 先定义成功语义，再写断言。
- 先分层 first blocker，再决定修谁。
- 先冻结现场，再决定是否继续扩样本。
- 先看 runtime truth，再说参数已生效。

image import 完成后也不能直接把测试放开。它只解除“节点 containerd 缺 image”这一层 blocker，真正的执行许可还必须重新经过 controller 切换和 static gate。

推荐顺序是：

1. 固定当前 live controller 身份：image、manager pod、imageID、generation / observedGeneration、Ready。
2. 切到目标 baseline image。
3. 等 rollout 实际完成，确认 pod imageID 命中目标 digest。
4. 重新检查 fresh slot。
5. 只在 gate 全部干净并被 TL 接受后启动最小 case。

不要把 `crictl images` 里能看到目标 image 误写成“baseline 已生效”。image import 只证明节点能启动该 image；Deployment 是否已经跑在该 image 上，是另一条证据链。

当多个产品线共用同一个 controller gate 时，某一方完成 static gate 并开始占用窗口后，要显式同步两件事：

- 当前 controller 身份已经被本线接管用于哪个最小验证；
- 本线不会在验证期间继续切 controller，并会在验证结束或 first blocker 固定后通知释放窗口。

这能避免另一条产品线把同一个 controller 身份误判成 drift 后立即覆盖，也能让对方知道自己的 addon-only 验证应暂停在 entry gate，而不是继续发 OpsRequest。

本地开发镜像不只 controller 需要导入，init/sidecar 镜像也要按同样规则处理。若 pod 卡在 `Init:ImagePullBackOff`，先确认失败容器名和镜像：

- 如果失败的是业务容器，说明业务 runtime 还未进入验证面；
- 如果失败的是 init container，例如 `init-syncer`，则 first blocker 仍在 create/setup 层；
- `ComponentParameter Finished` 或 controller 里其它 warning 不能覆盖 kubelet 直接拉镜像失败这一层事实。

对于本地 tag，必须检查本机 Docker 与每个 k3d 节点 containerd 是否一致。某个节点有镜像不代表所有可调度节点都有镜像；多 agent 环境里应逐节点 `crictl images` 验证。导入完成后，重新跑 fresh gate / fresh sample，不复用已经因 ImagePullBackOff 失败的旧现场。

对于被多个 setup blocker 截断过的复验，最终 pass 收口时要把“前置 blocker 已解除”和“目标语义已通过”分开写清楚。例如：

- 前置 blocker：controller image / init image 是否已导入并在每个节点可见；
- gate truth：controller imageID、pod、generation、Ready 是否仍命中目标；
- test boundary：只跑了哪个最小 case，是否扩 full suite；
- target contract：Ops、ComponentParameter、runtime、DB/replication truth 是否闭合；
- cleanup：live 是否 artifact-only 清理，目标前缀是否无尾巴。

这样后续读 artifact 时不会把“环境补齐后 T1 能跑”误当成“CM4 通过”，也不会把“CM4-only pass”误当成“全量 async 已通过”。

一个 scoped rerun 通过后，runner 停止通常是健康状态，不是停滞。只有在团队明确创建了下一层验证任务（例如从 CM4-only 扩到 full async / smoke / chaos）之后，才应该要求新的 runner 出现。巡检时要把“当前 scoped task 已 Done、无 runner 正常”和“下一层验证尚未分配”分开写清楚，避免把目标已闭合后的空闲误报成测试中断。

共享 controller 的 identity 也要结合窗口所有权解释。某条产品线明确释放 controller 窗口后，后续 image / pod / imageID 漂移到另一条产品线的 gate，不应再反向判定为已完成任务的 blocker。只有当本线仍持有窗口、或准备启动新的本线验证时，controller drift 才会阻断 static gate。

没有 active task 时，不要为了“看起来在推进”而复用旧 runner 或旧 proof 继续跑。正确做法是先确认下一层验证 scope：是继续 full async、某个 smoke/chaos 子集，还是进入开发修复。scope 未被创建前，Jack/Helen 没有新任务是合理状态；一旦 scope 创建，再重新走 static gate、fresh scene、fail-fast proof。

连续多轮巡检都没有新 task / runner 时，汇报重点应该从“重复列出旧 pass 细节”转成“当前 delta 为空、下一步需要谁创建什么 scope”。旧结论只保留一句 source-of-truth，避免让接收者误以为同一条验证还在持续执行或有新的测试结果。

释放窗口后 controller 可能被切回产品默认镜像或另一条线的验证镜像。这个事实要记录为“当前环境状态”，但不能倒灌到已完成任务的结论里；如果后续要复用旧 proof 对应的 gate，必须重新切回目标 baseline、重新确认 imageID / pod / generation / Ready / fresh slot，不能说“之前过过 gate，所以现在还能跑”。

验收口径调整不要简单描述成“放宽标准”。更稳妥的写法是说明它把哪类正常过渡期从失败里剥离出来，并同时列出仍然失败的条件。例如 primary-switch 后出现短暂 `1P0S`，只有在 endpoint 已指向新主、final primary 已变化、恢复窗口有上界、最终 replication/data truth 闭合时才可接受；endpoint 空窗、endpoint 指错主、同主抖动、恢复超时、最终复制或数据不闭合仍然必须失败。这样能避免把“更精确的验收语义”误解成“降低质量门槛”。

使用 vcluster 隔离共享 controller 时，preflight 要先证明“新跑道”本身可用，不能直接把产品测试放进去。至少要固定：vcluster 版本、host context、virtual kubeconfig、vcluster pod/PVC 是否 Running/Bound、virtual DNS 是否可用、PVC 是否能通过目标 StorageClass 绑定、以及 KubeBlocks/addon/test 的安装入口。

还要特别防止 kubectl context 串线：同一个 preflight manifest 如果误打到宿主集群，会得到看似成功的 PVC/DNS 结果，但它不能证明 vcluster 可用。建议所有 vcluster 内操作都显式使用独立 kubeconfig，并在报告里同时回：宿主误落资源是否已清理、virtual preflight namespace 是否已清理、vcluster 本体是否保留给下一阶段。

vcluster 解决的是 controller 控制面隔离，不等于完全隔离底层资源。节点 CPU/内存、image 分发、host StorageClass/PVC、底层网络仍共享宿主 k3d。因此下一阶段 runner 仍要做自己的 static gate，并建议显式指定 `STORAGE_CLASS=local-path`，不要依赖 virtual cluster 里不存在的默认 StorageClass 对象。

preflight 通过不等于测试已经开始，也不等于产品验证通过。它只证明测试跑道具备进入下一阶段的最低条件。正确交接应该写清三件事：

- 当前没有 runner 是正常状态，直到下一阶段任务被重新认领；
- 下一阶段必须重新做自己的 static/install gate，不能复用 preflight 结论；
- dependent task 如果曾被提前创建但已释放 claim，要在 preflight 通过后重新认领，避免任务状态看起来在跑、实际无人执行。

这能避免把“环境准备完成”误报成“测试在运行”，也能防止 Phase 0 和 Phase 1 的 proof 混用。

vcluster full-suite runner 启动后，早期状态要按“runner / virtual control plane / product object”三层分开看。比如进程和日志已经在跑，只能证明 runner 已启动；controller pod Ready 只能证明 vcluster 控制面可用；Cluster 仍在 T1 创建等待时，还不能写成 async 结论。巡检应固定 runner 命令、日志路径、样本名、当前 case、controller identity、cluster/component/pod 现状，再明确“尚未到 first blocker / pass 面”。

巡检里判断“测试是否正在跑”不要只依赖聊天口头状态。至少交叉核对三层：本机 runner 进程是否仍在、runner log 的 mtime / tail 是否有新增、目标执行面的 Cluster/Component/Pod/OpsRequest 是否处在与当前 case 匹配的状态。若 runner 已通过前半段但正在某个 OpsRequest 或 rolling restart 中，汇报应写成“active at section X / no final result yet”，不能把已通过的前半段误报成 full pass，也不能把正常 Updating/Terminating 误报成 blocker。

vcluster 里的 KubeBlocks 安装还要把 controller RBAC 和附属 CRD 看成 install gate 的一部分。Controller `Ready 1/1` 不代表它有足够权限 reconcile 业务对象；如果 T1 只有 Cluster spec、没有 status / Component / Pod / PVC，并且 controller 日志出现 cluster-scope `forbidden`，first blocker 应收在 vcluster KubeBlocks RBAC/setup 层，而不是 MariaDB async 语义。

这类修复应保持最小边界：先用 `kubectl auth can-i --as=system:serviceaccount:<ns>:<sa>` 固定缺哪些资源权限，再只补 vcluster 内 ClusterRole / ClusterRoleBinding，重启 controller 后检查 forbidden 是否消失。若 dataprotection 这类附属 controller 因缺 snapshot CRD CrashLoop，也要作为 install gate 尾巴处理；否则后续 full async proof 会混入无关控制面噪音。

当 startup 层出现类似 `no db manager for engineType ... and workloadType ...` 的 panic 时，不要只根据错误字面先补 env。更稳妥的做法是用同一 image 做最小本地复现，分别验证已有 env、疑似缺失 env、以及候选 image 的行为。如果旧 image 即使补齐 env 仍稳定 panic，而候选 image 在相同空 env 下能越过 manager-init，那么 earliest break 应收在 image / binary contract，而不是 env 渲染。

同一轮里看到 controller warning 时也不要自动并入 first blocker。比如 pod-side startup crash 已经能单独解释 Cluster Failed，而 controller 只是提示 config template name 不一致，就应把二者分开：先用单一 image cut 验证 startup blocker，template warning 单独建线处理。这样可以避免一次 patch 同时改变 image、env、templateName，导致下一轮 proof 无法判断是哪一刀生效。

single-cut patch 落地后，重跑 runner 前还要先做一次 rendered/live gate。不要只看 git diff 或 Helm values 已改，就直接进入产品测试；应在目标执行面里确认 Helm release revision 已更新、live `ComponentDefinition` / rendered manifest 里的 image 或关键字段已经命中目标值、fresh slot 仍为空。只有 gate 命中后再启动 runner，否则应停在 gate，把问题收成“patch 未被执行面消费”，而不是把下一轮失败写成产品语义结论。

full async / chaos-like 场景里，如果前半段已经通过，后半段出现新的故障，不要把结果倒灌到已通过段。比如 `Kill primary during writes` 后 secondary 角色恢复了，但 `Slave_SQL_Running=No`，那 first blocker 应收在当前故障注入 section，而不是回并到 restart / switchover / static-param / image gate。proof 要围绕当前 section 固定：故障前写入窗口、被 kill 的 primary、恢复后的 primary/secondary、endpoint、完整 `SHOW SLAVE/REPLICA STATUS\G`、`Last_SQL_Errno` / `Last_SQL_Error`、relay/master position、两侧数据对比、数据库和 sidecar 日志。这样后续归因才能直接回答“当前故障注入造成了什么复制语义问题”，而不是重新争论前面已通过的门禁。

复制角色恢复不能只看“角色名已经发布”。如果旧 primary 重启后作为 secondary rejoin，ready gate 必须同时检查复制线程和错误码，例如 `Slave_IO_Running=Yes`、`Slave_SQL_Running=Yes`、`Last_IO_Errno=0`、`Last_SQL_Errno=0`。只要 `SHOW SLAVE STATUS` 非空就发布 secondary，可能会把 GTID strict mode 下已经坏复制的节点误标为恢复完成，后续 divergence/rebuild 分支也会被绕过。

secondary role publish 还必须区分“marker 存在”和“当前 runtime 可验证”。`.replication-ready`、`master.info`、旧 slave config 这类文件只能说明历史状态或启动路径，不代表当前本地 MariaDB 还可连接。如果本地 `127.0.0.1:3306` 不可连，或者当前 `SHOW SLAVE STATUS` 取不到，roleProbe 不应继续发布或保持 `secondary` ready。

更稳的分流是：

- 本地 DB 不可连：不发布 secondary，保持 not-ready / unknown，等待下一轮 probe；
- 本地 DB 可连但 `SHOW SLAVE STATUS` 取不到：不发布 secondary，不把 marker 当 hard proof；
- 本地 DB 可连且复制字段可读，但 IO/SQL 非 Yes 或 errno 非 0：进入既有 divergence / rejoin / rebuild 分支；
- 本地 DB 可连且复制字段健康：才发布 secondary。

一句话说：

- **roleProbe 发布 secondary 必须基于当前 DB 可连和当前复制 truth；marker 只能辅助判断，不能越过 runtime hard gate。**

如果数据库进程由外层 wrapper 启动，wrapper 的“存活”不能等价成数据库进程存活。尤其是 `docker-entrypoint.sh mariadbd &` 之后再轮询本地 SQL 的模式，如果只无限 `SELECT 1`，却不检查后台 `mariadbd` 是否已退出，就可能出现容器外层还活着、数据库端口长期 refused、sidecar/roleProbe 一直失败、controller 长期 role_not_ready 的假活状态。

更稳的 wrapper 结构是：保存数据库子进程 PID；等待 local DB ready 时每轮先 `kill -0 $PID` 检查子进程还在；如果子进程已退出，`wait $PID` 取真实 rc 并让 wrapper 非零退出，交给容器重启恢复。正常 termination guard 要单独保留，例如预期 SIGTERM/143 不应被误报成 crash loop。

一句话说：

- **DB startup wrapper 等待 ready 时必须同时 watch 子进程存活；发现 mysqld 先退出就 fail-fast，让容器重启，不要形成“wrapper 活、DB 死”的假活。**


实现这类 hard gate 时，还要固定 SQL 客户端输出形态。`SHOW SLAVE STATUS\G` 这类 parser 如果按 `Slave_IO_Running: Yes` 这种 labeled 文本匹配，就不能让同一路径经过 `-N -s` 之类参数压成 value-only 输出。否则 live DB truth 明明健康，roleProbe 也会因为 literal grep miss 而持续返回 initializing。

更稳的做法是二选一：

- 保留 labeled 输出，并让 parser 只处理 labeled 形态；
- 或明确支持 value-only 输出，并用字段顺序/结构化转换做单一 parser。

不要同时留下“看起来像 labeled parser、实际拿 value-only output”的混合路径。UT 至少覆盖 labeled healthy、当前 value-only/不兼容形态、empty output、unhealthy fields。

一句话说：

- **roleProbe 的 runtime hard gate 不只要语义正确，还要让 SQL 输出形态和 parser 形态一致。**

遇到 GTID out-of-order 这类严格模式错误时，最小修复不应先改 switchover、syncer image 或测试 helper。先判断服务路由和 failover wiring 是否已正确：新主、service selector、`Master_Host`、IO 线程都正确时，first break 往往在 rejoin startup ready gate。此时正确动作是保持 pending 或进入 divergence-pending，让 rebuild path 接管；不要发布一个假的 recovered secondary。

vcluster 样本清理要同时看 virtual 资源和 host translated 资源。host 侧 orphan pod/PVC 有时是因为 virtual Cluster/Component/Pod/PVC 仍存在，直接删 host translated pod/PVC 会被 syncer 再拉起。更稳的顺序是先删除 virtual owner，再验证 virtual 侧 `cluster/component/pod/pvc/service/opsrequest` 为空，最后验证 host namespace 里 translated pod/PVC 前缀为空；不要为了清尾巴误删 vcluster 本体、controller 或 addon release。

如果 addon patch 修改了 `ComponentDefinition` spec，要先判断该对象在目标 KubeBlocks 版本里是否支持原地更新。对于被 controller 判定为 immutable 的字段，同名 Helm upgrade 会让旧 `ComponentDefinition` 进入 `Unavailable`，随后新 Cluster 在 T1 create 直接卡在 referenced ComponentDefinition unavailable；这属于 install/create gate，不是产品运行时语义。

最小修复通常不是 clean reinstall。clean reinstall 只能绕过当前现场，不能防止下一次同名 upgrade 再撞 immutable。更稳的代码层 cut 是让 `ComponentDefinition` 名称随 chart/app 版本或显式 name suffix 变化，并确保 `ClusterDefinition` / component 引用同步指向新名。复验 gate 要分开报告旧对象历史状态和当前准入状态：旧 unavailable 只有在当前 `ClusterDefinition` 仍引用它时才是 blocker；如果新 `ComponentDefinition` Available 且当前引用已切新名，旧对象只能作为历史尾巴记录。

复验这类 versioning cut 时，不能只看 Helm release revision 增加。必须固定完整引用链：render 和 live 的 `ClusterDefinition` async `compDef`、新 `ComponentDefinition` 名称、PCR name / `componentDef` 都指向新的 versioned name；旧 unavailable `ComponentDefinition` 不再被新样本引用；新 `ComponentDefinition` 自身是 `Available`；同时旧语义内容也要在新对象里保留，例如 startup wrapper、roleProbe hard gate、divergence-pending 分支。只有新样本真正用新 `compDef` 创建出 pod，才算 create gate 被 runtime 消费。

一句话说：

- **ComponentDefinition immutable-field cut 的验收点是“新引用链 + 新 CD Available + 新样本实际创建”，不是 Helm revision 变了。**

CM4 这类中段 case 里如果先跑 manager baseline preflight，preflight 本身失败就要先收成 gate/probe blocker，而不是继续解释后续参数语义。例如 expected imageID 有值但 actual 取到 `<empty>`，同时日志里出现 Kubernetes API `TLS handshake timeout`，说明当前证据链还不能区分是 controller 真的 drift、kubectl/API 临时不可达、还是取证命令失败。

这时正确动作是先冻结 gate/probe 证据：controller Deployment image、pod name、pod imageID、Ready、generation/observedGeneration、kubectl 命令 stderr、API 连通性、runner failed assertion。只有证明 actual `<empty>` 不是取证失败，才可以升级为 controller identity drift。不要把这类 preflight 失败倒灌成 CM4 参数、role 或 replication 语义问题。

早段 exporter `/metrics` 失败也要先按三层拆开，不要直接写成数据库 runtime blocker。runner 只能证明某次 HTTP 探测失败；是否是 exporter 进程、Service/Endpoint、pod 网络、kbagent/controller、还是 MariaDB 本体问题，需要用同一现场的 source-of-truth 固定。

建议最小证据面包括：失败 pod、exporter container state / restartCount、对 `:9104/metrics` 的具体探测命令和错误、pod events、exporter / kbagent / mariadb 日志、Service 和 headless Endpoint、controller reconcile 日志，以及 DB role / replication truth。如果 DB 是 Running、primary/secondary role 和复制都正常，应明确写出来，把 exporter 暴露层和 MariaDB 数据面分开。只有证明 metrics 失败由数据库不可用或角色/复制异常导致，才把 blocker 升级到 runtime 语义；否则先收在 exporter/metrics 暴露或测试探测层。

如果 exporter 探测依赖 `kubectl exec`，还要把控制面 route 和容器内 HTTP 结果分开。每个 pod 的 probe 至少保留 route preflight/repair action、`kubectl exec` rc/stderr、容器内 `wget`/HTTP rc/stderr、body preview 和 metric count，并分类为 `route_api`、`exporter_socket_http` 或 `body_count_invalid`。当 `127.0.0.1:<port>` 这类本地 API route 断开时，当前结果只能证明观察路径不可用；即使某个 pod 的 exporter 刚刚通过，也不能把另一个 pod 的 route failure 写成产品 exporter 不工作。

一句话说：

- **exporter `/metrics` fail 必须先排除 API/exec 观察路径；只有 exec 成功且 HTTP/body/count 自洽失败，才进入 exporter 暴露层结论。**

runner 打包也属于测试准入的一部分，尤其当本地 runner 会从不同目录拼装 `tests/` 和 `lib/` 时。不要只对入口脚本做 `bash -n` 就开始跑；如果某个 case 依赖本地增强 helper，还应在 runner 目录内显式验证 helper 是否存在，例如 `source <runner>/lib/common.sh && declare -F <helper>`。

如果运行中发现 helper 缺失或 runner lib 被错误覆盖，这一轮应收成 aborted-invalid-run，而不是产品 first blocker。报告要固定：缺失 helper 名、期望来源、错误覆盖来源、已经清理的样本前缀、以及哪些已通过段只能作为进度背景不能作为产品结论。随后必须恢复正确 runner lib，用 fresh sample 重走 gate，不能复用已经受污染的现场。

runner staged copy 的入口 header 也是准入面的一部分。即使 gate 里已经绑定了样本名，如果实际执行脚本仍保留旧的 `CLUSTER="mdb-async-$$"`、delete-on-exit trap、默认 namespace/kubeconfig 或旧 cleanup 行为，实际跑出来的 sample 就不是 gate 接受的 sample。这类错误必须收成 invalid runner-copy/header run，而不是产品 first blocker。

恢复后要把卫生证据和产品证据分开：旧错误 sample 的 virtual/host tail 必须为 0；corrected runner copy 要证明 `CLUSTER` 消费外部 `CLUSTER_NAME`，cleanup 只 preserve artifacts，不在 exit trap 里删除现场；corrected run 启动前 gate sample 仍是 fresh slot；最终 runner log、proof tarball、summary 文件名和正文都只引用 corrected sample。不要把 invalid sample 的通过段或失败段混进 canonical result。

一句话说：

- **full async 复验的最小准入不只是“源码正确”，还包括 staged runner copy 的样本绑定和 cleanup 语义正确；跑错样本只能作废重跑。**

当为了解决后段环境 gate 而切换 StorageClass 或其它底层跑道时，前段 case 的失败不能自动解释成原来的后段问题已解决或回归。必须先证明本轮样本的实际 PVC / Pod / rendered manifest 确实使用了新跑道，然后把新的失败按当前 section 收口。

例如 T12 因 `local-path` 不支持扩容而改用可扩容 StorageClass 后，如果 CM4 又失败，应先把 CM4 的 Ops / ComponentParameter / pod UID / restart / role label / endpoint / DB role / replication truth 固定完整；同时单独固定 PVC `storageClassName`，用于排除“跑道没切中”。不要把 CM4 的角色收敛窗口失败写成 T12 StorageClass 结论，也不要把后段环境 gate 的修复结果反推到前段产品语义。

如果 helper 宣称允许一个 bounded convergence window，例如 `60s`，它的观测证据必须覆盖同一个窗口。不要用 `sleep 20 + kill watcher` 这类截断账本去判 `60s` 的 eventual convergence；否则会把产品在 23-24s 内恢复的场景误判成失败。

更稳的实现方式有两种：一是 watcher 至少持续到完整 deadline 后再断言；二是断言函数自己 live-poll 到 deadline，并把每次采样写到账本。无论采用哪种，都要保留原失败条件：endpoint 未切新主、secondary 未重新 publish、pending marker 未清、replication/data truth 不闭合、或收敛过程出现超过窗口的非单调/回退，仍必须失败。

在长链路 full async 复验里，巡检应同时报告“当前跑到哪里”和“关键修复是否已经被现场消费”。例如 CM4 observation-window cut 后，如果 runner 日志出现 `elapsed=19s deadline=60s` 并通过，说明该测试侧 cut 已被 live rerun 消费；后续即使 C1/C2/T12 失败，也不能再把它归回 CM4 watcher 截断问题。

这种中途正向证据也要写清仍未 full pass：前半段通过只是锁定边界，不能提前释放 smoke 或合并结论。正确表述是“已闭合到 section X，当前运行在 section Y，无 final pass/first blocker yet”。

如果 targeted cut 在复验中已经被 ledger 明确消费，后续出现的新 fail 要重新按新 section 收口，不能把它倒灌回刚修的 helper。比如 scale-out 新 Pod bounded confirmation 已经记录到 `role not yet published` 的前几轮 indeterminate，以及最终 `new pod=secondary / DB reachable / Slave_IO/SQL=Yes/Yes / marker visible / elapsed<=deadline` 的 closed probe，那么 T10 helper 边界已经被本轮消费；之后 C3/C4/C6 再失败，都应按 chaos section 自己的 fault target、pre-fault baseline、post-fault role/endpoint/DB/replication/data truth 收证。

连续修多个 helper 后，下一轮 gate/巡检要把每个 helper 的消费证据分开报告，而不是只说“跑过了”。例如同一轮里 scale-out 新 Pod confirmation 在 T10 以 `probe[6] elapsed=30s/60s` 闭合，双 Pod kill recovery confirmation 在 C3 以 `probe[2] elapsed=7s/60s` 闭合；这说明两个不同切口都进入了实际 runner 且各自解决了对应等待边界。后续如果 C4 或更后段失败，归因时就不应再回到 T10/C3 helper 本身，除非新 proof 显示这些 ledger 没有真正被执行或字段不可信。

复验更新的推荐写法是：`fix A consumed: probe/fields/deadline; fix B consumed: probe/fields/deadline; current section: X; no full pass/new blocker yet`。这样团队能在长链路 rerun 中保持清晰的因果边界。

这类报告最好同时列出：被消费 cut 的 ledger 行、通过的 elapsed/deadline、新 first blocker 的 section 名和 fail line、以及无效/污染运行是否已从产品统计中剥离。这样团队能清楚地区分“修复生效后暴露了更后面的 blocker”和“修复没有真正进入运行面”。

一句话说：

- **targeted cut 被 live ledger 消费后，后续失败必须重新定界为新的 section-local blocker；不要用旧修复线解释新现场。**

同理，修复某个测试 helper 后，复验报告要给出这把 cut 被现场消费的具体 ledger 行，而不是只说“该 section 通过”。例如 topology-change replication helper 改成 bounded confirmation 后，应报告 `probe[N] class=closed`、关键字段 `Slave_IO/SQL=Yes/Yes`、endpoint target、elapsed/deadline。这样如果后续 C1/C2/C6 再失败，可以明确新 blocker 与刚修的 T11 helper 无关。

一句话说：

- **targeted helper cut 的复验通过，要带实际 ledger 消费证据；section pass 只是结论，不是证据。**

连续稳定性断言不能把“观测不到”当作“观测到产品回退”。如果 helper 要求 `2 consecutive closed probes`，每轮 probe 必须先分出三态：closed、real non-closed、indeterminate/read failed。只有成功观测到 real non-closed 时才应重置 consecutive counter；kubectl/API 抖动、stderr 被吞、role discovery 返回空等 indeterminate 状态应记录 rc/stderr/ledger，并作为诊断或最终不可判定失败处理，而不能伪装成 primary=0/secondary=none 的产品状态。

否则会出现 mixed-state：role 采样丢失，却夹带上一轮 replication/data 字段，导致 failure payload 不是一个自洽的 runtime scene。遇到这类 mixed-state，优先修测试观测面和 per-probe ledger，再决定是否需要产品侧 cut。

当测试 helper 被修成三态 probe 后，复验 gate 不能只看脚本语法通过。必须在 runner 源里固定三类证据：每轮 probe ledger 会落盘，状态分支明确区分 `closed / nonclosed / indeterminate`，且 `indeterminate` 不会重置连续成功计数。这样下一轮若仍失败，proof 才能判断是真实产品不稳定，还是观测链路不可判定。

如果 bounded recovery 窗口里出现 API route / control-plane access 抖动，最终失败口径必须把“观察链路不可判定”和“产品真实未恢复”分开。测试 helper 通过本地端口或 kubeconfig 读取集群状态时，这条路径本身就是被测系统之外的观察链；一旦 `kubectl get/exec`、API route、端口转发或 stderr 显示 refused/timeout/read failed，后续 role、endpoint、row count、replication 字段都可能只是观察洞，不是一个自洽的 runtime snapshot。

这类 helper 的每轮 probe 应至少保留：route/API rc、stderr、endpoint 读取 rc/stderr、role snapshot rc/stderr、DB exec rc/stderr、raw DB output、row-count rc/stderr，以及该轮被归类为 `closed`、`real non-closed` 还是 `indeterminate` 的理由。最终失败也要分口径：如果窗口被 route/read holes 主导，应报 observation remained indeterminate；只有在观察链路连续可用且成功观测到真实 non-closed、divergence，或无 probe holes 的 bounded-window exhaustion，才报 runtime convergence failure。

如果测试 kubeconfig 本身依赖本地 port-forward，例如 `127.0.0.1:11443` 指向 vcluster API，那么 route repair 不能继续使用这份已经断链的 kubeconfig。断链后再执行 `kubectl --kubeconfig <11443.yaml>` 只会继续 refused，无法自救。修复应有一条 host-side control path：用 host kubeconfig / host namespace 重新拉起 `kubectl port-forward svc/<vcluster> 11443:443`，并把 port-forward PID、LISTEN 检查、preflight rc/stderr、restart action 写入 ledger。

bounded helper 进入每轮业务 probe 前，可以先做 route preflight：如果 route 可用，记录 `route_action=ok`；如果不可用，尝试 host-side restart 并记录 `route_action=restarted` / `restart_failed`、rc/stderr；只有 route 恢复后才继续读取 role/endpoint/DB truth。route restart 失败时要收成 observation hole，而不是 runtime non-closure。

一句话说：

- **依赖 port-forward 的测试链路要有 host-side route repair；断掉的 11443 kubeconfig 不能用来修复 11443 自己。**

一句话说：

- **control-plane/API route 断链时，测试只能证明“看不清”，不能直接证明“产品没恢复”；runtime first blocker 必须建立在可用观察链路上的自洽 truth。**


Scale-in / scale-out 之后的复制确认也不要只做一次裸 `SHOW SLAVE STATUS` 字段断言。拓扑变更刚完成时，Kubernetes exec、endpoint 更新、sidecar 角色缓存和数据库输出都可能出现短暂不可判定；如果 helper 不保存 rc/stderr/raw 输出，`Slave_SQL_Running=''` 只能说明解析结果为空，不能自动等价成复制线程真实关闭。

Scale-out 之后尤其要区分两类成功面：OpsRequest `Succeed` / pod count 达标，通常只能说明新 Pod 创建流程完成；它不必然等价于“新 Pod 已发布 secondary、DB 本地可连、复制追平、能读到新写入”。测试如果在 Ops `Succeed` 后固定 `sleep 10` 就直连新 Pod 读数据，会把正常的 roleProbe/replication catch-up 窗口误报成 runtime replication bug。

更稳的做法是把新 Pod 可读性改成 bounded confirmation，并在每轮 probe 记录：Ops phase、pod count、新 Pod 名称、role snapshot、新 Pod 本地 DB exec rc/stderr、`SHOW SLAVE STATUS\G` raw/parsed、marker/divergence 状态、目标行是否可见。只有观测到新 Pod 已是 secondary、DB 可连、复制状态闭合且目标行可见，才算通过；如果成功观测到真实 non-closure 或 current-round divergence，则严格失败；如果只是 role/DB/read 不可判定，则单独记为 probe hole，不要折叠成 runtime truth。

一句话说：

- **scale-out 的 pod-created success 不是 new-secondary-ready；新 Pod 读写断言必须等 role/DB/replication/data 四层闭合。**

这类检查应改成 bounded confirmation：在 deadline 内反复确认 pod count、secondary role、primary endpoint、`SHOW SLAVE STATUS` raw/parsed 字段和必要的数据闭环。每轮 probe 至少记录：pod_count、secondary identity、endpoint target、exec rc/stderr、raw/preview、`Slave_IO_Running`、`Slave_SQL_Running`、`Seconds_Behind_Master`。

失败条件仍要严格：deadline 内没有任何可证实的 secondary + `Yes/Yes`，或者成功观测到真实 role/endpoint regression、真实 replication non-Yes、current-round divergence，就应该失败。但单次 exec/read hole、partial stdout、empty parse 这类 indeterminate 只能记 ledger，不能单独成为 runtime blocker。

一句话说：

- **拓扑变更后的 replication assertion 要从单点字段断言升级成带原始输出的 bounded confirmation；空读不是 runtime truth。**

当一个观察链路修复在复验中被明确消费后，后续失败要重新按当前 section 定界。例如 route lifecycle cut 的消费证据不是“脚本里有 helper”，而是 bounded ledger 每轮都记录 `route_action=steady` / `route_rc=0`，并且业务闭合仍然依赖 role、DB、replication、data truth。若 T10 在这种账本下闭合，说明之前的 11443 route-hole blocker 已经跨过；后续 C1/C4 再失败，不能再归因到 T10 route 观察洞。

同一轮里，post-fail preserved truth 只能帮助下一步归因，不能反写前一段 terminal judgment。比如 C1 runner terminal 是“primary recovered 后无 secondary”，后续补采看到 SQL thread `No`、`Last_SQL_Errno=1950`、`.replication-divergence-pending`，这说明现场存在值得开发归因的 rejoin/divergence 证据，但如果这些证据采集发生在 runner 越过 C1 之后，就必须标成 preserved evidence，不应把它改写成 C1 已通过或直接写成 C4 结论。

一句话说：

- **targeted cut 的消费要靠当轮 ledger 证明；后续 section 的 preserved truth 只用于归因，不能回写前段结论或跨段下 C4/#runtime 结论。**

When a long rerun consumes multiple prior cuts, keep a running ledger of each consumed cut before the next failure. A later blocker should be framed as “new section after consumed cuts,” not as an ambiguous rerun failure. For example, if route repair is exercised and self-heals within the same probe, then scale-out closes; C1 then closes with replication/data truth; C4 then closes after rapid primary kills with explicit replacement/roleProbe evidence. A later T12 volume expansion failure is a new T12 blocker. It should not reopen T10, C1, or C4 unless the T12 proof directly shows their ledger fields were invalid.

For volume expansion sections, collect the storage evidence separately from database health evidence:

- OpsRequest phase, conditions, events, and controller messages.
- PVC `spec.resources.requests.storage`, `status.capacity.storage`, conditions, annotations, and StorageClass.
- StorageClass provisioner and `allowVolumeExpansion`.
- CSI / resize controller events and pod restart/recreate evidence.
- Database role, endpoint, replication, and row-count truth at the T12 window.
- Exact runner assertion line and whether later resize completion is only preserved truth.

一句话说：

- **PVC resize failure is a storage/ops section blocker; database health staying green does not make the resize assertion pass, and later PVC convergence cannot be back-written into the T12 terminal line.**

For tests that run inside a virtual control plane, environment gates must be validated in the same control plane that the controller uses. A host-side StorageClass can provision translated PVCs, but an Ops controller running in the vcluster may still validate `StorageClass` objects through the vcluster source-plane API. If the vcluster has no same-name expandable StorageClass object, VolumeExpansion can fail at validation even though the host cluster has an expandable CSI class.

A safe test-environment cut is to preflight the source-plane StorageClass before creating the sample or before running the relevant section. If the source-plane class exists and `allowVolumeExpansion=true`, record a steady pass. If it is missing in a known host/vcluster setup, sync a minimal source-plane StorageClass only when the host has the same name and `allowVolumeExpansion=true`; otherwise fail or skip early as an environment precondition with a clear reason. Do not allow the run to proceed to a product T12 assertion when the prerequisite is absent.

一句话说：

- **storage features must be gated in the controller’s source plane, not inferred from host-plane objects alone.**

When a reconfigure operation is intentionally killed and the OpsRequest reaches a terminal `Failed` phase, post-terminal recovery can have multiple independent closure surfaces. Role recovery, endpoint selection, replication health, and data consistency can all close before the new configuration revision has finished applying to every pod. Do not assert parameter equality immediately after role/replication/data recovery unless the helper also proves revision/config convergence or polls the parameter itself to closure.

For post-terminal reconfigure helpers, record per-probe evidence that keeps these surfaces separate:

- route/action rc and stderr
- pod role snapshot and primary endpoint target
- component generation, updateRevision, podRevision, and updated/readyWithRole state when available
- replication raw/parsed truth and row counts
- target parameter value on each relevant pod
- classification as closed, real non-closed, or indeterminate

A mixed-revision window is not automatically an addon parameter propagation failure. If controller logs still show `allUpdated=false`, pod recreate/submitting update, pod terminating, or updated-but-role-not-ready while one pod has the new parameter and another still has the old value, the test boundary is too early unless the bounded parameter-convergence deadline has actually expired.

The pass condition should use the values captured by the same successful probe that closed the boundary. Avoid doing a new immediate read after the helper has already decided to pass: that can race with a fresh transition and makes the proof harder to audit. Conversely, if role/endpoint/replication/data are closed but parameter values are not closed yet, keep probing until either the parameter values match the requested target on all relevant pods, the deadline expires, or a terminal non-converged condition is visible.

一句话说：

- **post-terminal recovery must close the same surface it asserts; role/replication/data closure alone is not proof that reconfigure parameter values are already consistent.**

For rapid primary-kill sections, the entry state is part of the test contract. Do not choose the kill target from a single `primary_idx` / role-label read that suppresses `kubectl` stderr. A single empty result can mean an API route hole, a read failure, a transient role-label publication window, or a real no-primary state. These must be separated before the test performs the destructive kill.

Use a bounded entry confirmation before each kill round: route probe, role snapshot, pod readiness/UID/deletion state, and a closed condition such as route-healthy `1P1S` with ready pods. If route/read holes persist, classify the section as observation indeterminate. If route is healthy and no primary remains through the deadline, classify it as real no-primary. Only after the entry primary is confirmed should the helper delete that pod and start the recovery ledger.

For each recovery probe after a kill, keep enough context to prove whether the fault actually happened and whether recovery closed: killed pod UID before/current, UID change, pod phase/ready/deleting, role snapshot, endpoint target, DB local role, replication raw/parsed truth, row counts, and closed/nonclosed/indeterminate classification. This prevents a helper observation hole from being mistaken for a runtime rapid-kill bug.

一句话说：

- **chaos helpers must prove a route-healthy pre-fault baseline before injecting the fault; otherwise the result is an observation/setup blocker, not a product chaos result.**

Early smoke checks often look simple, but they still need observable failure classes. A row-count assertion like “expected >= 3, got empty string” or a read-only assertion like `read_only=<unset>` is not enough to decide whether the database lost data or a secondary is writable. Empty output can come from an API route failure, `kubectl exec` failure, database socket/query failure, SQL returning no rows, or a real value mismatch. Treat these as different outcomes.

For SQL smoke probes, capture and report at least:

- outer `kubectl exec` rc and stderr
- inner database client rc and stderr, if the command wraps the client
- SQL stdout preview and parsed value
- route probe/action context when stderr matches API route failure
- classification such as `route_api`, `db_socket_query`, `empty_output`, or real mismatch

Only classify as product data/read-only mismatch after the SQL probe itself succeeded and returned a concrete value. If the probe cannot prove that, the first blocker is test observability or control-plane access, not data loss or split-brain.

一句话说：

- **empty SQL output is not product truth; smoke probes must preserve rc/stderr/stdout before turning an empty value into a database conclusion.**


Database role probes should not assume that every MySQL-family engine exposes the same system variables. A helper query like `SELECT @@global.read_only, @@global.super_read_only` may be valid for one engine/version and fail with “unknown system variable” on another. That failure proves a helper compatibility/read path issue unless the probe also has a successful, engine-supported source of local-role truth.

Prefer the smallest portable role signal for the engine under test, such as `@@global.read_only` for MariaDB primary/secondary local role, or explicitly capability-detect optional variables before using them. Preserve the query text, rc, stderr, stdout preview, parsed value, and classification. If an optional variable is unsupported, classify that as unsupported/query compatibility and either fall back to the supported signal or fail as observation-indeterminate; do not convert it into no-primary, role mismatch, or replication failure.

Real local-role mismatch remains strict: once the supported query succeeds, primary must report the expected writable value and secondary must report the expected read-only value. Only then can the test make a product role-truth conclusion.

一句话说：

- **helper SQL must be engine-compatible; unsupported variables are observation compatibility holes, not product role truth.**

When a helper parses role labels into internal indices, preserve both the raw label snapshot and the derived fields used by downstream checks. A raw snapshot such as `pod-0|secondary` / `pod-1|primary` can be correct while the helper still emits `primary=none secondary=none` if it forgets to derive pod index from pod name before assigning internal variables. That is a parser/index bug, not proof that role publication failed.

For bounded convergence helpers, add a defensive parse-consistency class. If the raw role snapshot contains published roles or `primary_count>0`, but the parsed primary/secondary index fields are empty, fail as helper parse inconsistency and include the raw snapshot, derived indices, counts, and parse note. Do not continue to endpoint, replication, row, revision, or config truth until the parser has produced coherent identities; otherwise downstream `<none>` or `na` fields become misleading product evidence.

一句话说：

- **raw role labels and parsed role indices are separate truth layers; published labels with empty parsed indices are a helper parser failure, not runtime no-secondary.**

Route preflight and the actual mutating API call are separate observation points. A helper can record `route_action=steady` on a lightweight read and still fail moments later when `kubectl apply` tries to download OpenAPI from the same API server. Do not let the preflight result overwrite the apply stderr. Preserve raw `kubectl apply` rc/stderr, OpenAPI download errors, route action/retry/listener context, manifest path, and intended object name.

For destructive sections, failed object creation must be terminal for that section unless a verified retry creates the object and returns a concrete name. Never pass an empty OpsRequest name into a waiter, and never perform a kill/delete using an empty pod index or empty pod name. Entry helpers should prove a route-healthy, non-empty target identity immediately before fault injection; otherwise classify as helper entry empty or route/API observation failure, not runtime no-primary.

一句话说：

- **route preflight is not apply truth; failed creates and empty destructive targets must stop as helper/observation failures before mutating the cluster.**

When a fail-fast runner continues after a terminal section failure, later green sections are spill, not proof that the earlier failure was benign. This is especially important for route/API observation failures: a route repair may succeed in a later helper, allowing later OpsRequests or reads to pass, while the first section still failed because its own probe could not observe product truth.

Freeze the first terminal section and package its own ledger before interpreting later output. For T4/R4-style smoke checks, collect the classified SQL probe context, route initial/retry rc and stderr, route snapshot/listener, role and endpoint snapshots after route restore, and current DB truth. Only promote to product data/role failure when the route/read path succeeded and returned concrete values in that section; otherwise keep it as route/API observation or helper probe failure.

一句话说：

- **later route recovery does not rewrite an earlier route-classified smoke failure; first-fail section evidence owns the boundary.**

Shared control-plane baselines are part of the test gate. If a host-side controller image changes between accepted reruns, a fresh sample must not be interpreted as product/runtime evidence until that baseline is either explicitly accepted or reverted. Even when the vcluster controller, addon live chain, route, and runner anchors all look healthy, an unapproved host controller change makes the run invalid-gate evidence only.

When this happens, stop any already-started runner before drawing section conclusions. Preserve before/after controller pod, image, imageID, deployment revision, change-cause, managedFields update time/manager, and whether vcluster/addon state stayed unchanged. Cleanup the sample and mark all early runner output as invalid-gate context, not pass/fail proof.

一句话说：

- **controller baseline drift invalidates rerun evidence until the shared baseline is explicitly accepted or restored.**

When a rerun is intentionally paused at gate level, do not create churn tasks just to keep activity visible. The correct progress signal is the explicit blocked state: no active runner, no active agent task, the exact shared baseline decision needed, and the evidence that prevents safe continuation. This avoids burning fresh samples under an invalid baseline and keeps future pass/fail claims auditable.

A paused gate report should include the blocking invariant, the last valid baseline, the current conflicting baseline, who has been notified, and the condition that unblocks testing. Only after that condition is met should a new rerun task be created.

一句话说：

- **gate-level pause is a valid project state when the next action is an explicit baseline decision, not another sample.**

For long-running validation lines, repeated status checks during a gate-level pause should verify negative state as carefully as positive progress. Record that no runner process exists, no active task is queued, and no fresh sample should be created until the named decision is resolved. This prevents well-intentioned reruns from bypassing a shared-baseline hold.

The status update should name the exact unblock choices rather than saying “waiting”: accept the current baseline as shared, restore the previous baseline, or explicitly create a new task to validate the new baseline. Without one of those actions, the correct state is blocked and idle by design.

一句话说：

- **during a baseline hold, “no active runner/task” is the expected safe state; progress resumes only after an explicit unblock decision.**

When a validation line is blocked on an external decision, repeated automation should avoid re-running the same expensive probes unless the decision inputs changed. A lightweight heartbeat is enough: task board empty, no runner process, no new channel messages, and the same explicit unblock condition. This keeps the system quiet while preserving accountability.

If the unblock condition is a shared baseline decision, do not silently reinterpret it as approval just because time passed. Require a new message or task that explicitly accepts the baseline, restores it, or scopes a separate validation of the new baseline.

一句话说：

- **time passing is not baseline approval; repeated catchups should verify the hold, not erode it.**

A baseline hold should name the decision owner or escalation path as soon as possible. If repeated catchups find no runner, no active task, and no new messages, the technical state is stable, but the project still needs an explicit owner to accept, revert, or separately validate the new baseline. Without that owner, creating more test tasks only hides the real blocker.

In status updates, separate the technical invariant from the decision gap: “safe to not run” is technical; “who accepts the new controller baseline” is coordination. This keeps engineering evidence clean while making the next human action visible.

一句话说：

- **when evidence is stable but blocked, escalate the baseline decision owner instead of creating more reruns.**

Running addon validation inside a vcluster does not remove host-side invariants from the gate. The product objects and addon controller logic may be inside the vcluster, but the host still carries the vcluster pod, local API port-forward, translated resources, storage backend, and parts of the observation path. A host controller change is therefore not automatically a product failure, but it is a changed environment variable that must be accepted or reverted before comparing results with previous reruns.

Gate reports should state this distinction explicitly: vcluster controller/addon versions define the product-under-test baseline, while host controller/storage/route components define the test-environment baseline. Both matter for reproducible conclusions, but they answer different questions.

一句话说：

- **vcluster isolates product objects, not every environment variable; host-side baseline drift still invalidates cross-run comparisons until accepted.**

When a validation line remains blocked across multiple reminders, keep the status language stable. Repeating the same invariant is useful if it is precise: no runner, no active task, no new evidence, and the same unblock choices. Avoid introducing new hypotheses or new terminology unless the evidence changed.

This makes it easier for humans to decide: the technical state is stable, and the only remaining variable is the explicit environment/baseline decision.

一句话说：

- **stable blocked evidence should produce stable status updates; do not invent new work while waiting for a baseline decision.**

Image pull failures before the database container starts are gate or image-distribution failures, not database runtime failures. A live `ErrImagePull` glance is only a candidate; wait for the runner's own terminal boundary, then correlate it with pod events and node image state before classifying.

For a T1 image-distribution boundary, preserve the init/container image name, pod phase and init status, Kubernetes pull events with reason/message, scheduled node, rendered/live addon image source, imageID absence, and image presence across the relevant k3d nodes. Also state whether any DB container ever started. If no database container started, do not draw conclusions about roleProbe, replication, data, or MariaDB runtime.

一句话说：

- **pre-runtime ImagePullBackOff is an image distribution gate failure; prove image absence/pull events and avoid MariaDB runtime conclusions.**

SQL probe output mode must match the parser contract. If a helper runs a multi-field diagnostic such as `SHOW SLAVE STATUS\G` but invokes the client with flags that suppress labels or reshape output, a parser that expects named fields can turn a successful query into empty parsed values. That is a probe/parser failure unless a concrete parsed value proves a real runtime mismatch.

For replication-lag or role probes, preserve both raw stdout preview and parsed fields. Include query text, client flags, kubectl rc/stderr, database client rc/stderr, route context, and current truth from an independently compatible read. Classify incompatible or empty parsed output separately from real lag, replication, or data mismatch.

一句话说：

- **SQL output shape is part of probe truth; do not treat parser-empty fields as runtime failure when raw query output succeeded.**

Gate reports are evidence, not formatting placeholders. If a report loses critical values such as sample name, image tag, route action, controller imageID, artifact paths, or tail counts, reject the gate and do not start the sample. Empty fields make the run unauditable even if the operator believes the checks passed.

Prefer a plain `key=value` block for gate facts when values contain punctuation, shell metacharacters, or markdown-sensitive text. This prevents command substitution, markdown stripping, or template rendering from erasing the exact fields needed for later attribution.

一句话说：

- **a gate with blank critical fields is not a gate; require literal non-empty values before starting a sample.**

Optional capability skips must be separated from operation plumbing failures. If a preflight says a capability such as LoadBalancer is unavailable, skip the external outcome assertion for that capability, but still classify any preceding OpsRequest failure on its own evidence. A timeout while enabling expose is not the same as “LB IP skipped”; it may indicate controller, Service, route/API, validation, or environment plumbing that must be captured at the section boundary.

For expose or similar optional-capability sections, preserve the preflight capability result, OpsRequest create/wait rc and last observed phase/conditions, generated manifest or request parameters, Service/Endpoint state, events, controller logs around the timeout, route context, and whether the runner attempted disable/cleanup after the failure. Do not let a later skipped external check hide an earlier enable/disable OpsRequest timeout.

一句话说：

- **capability unavailable can skip the final external assertion, but an OpsRequest timeout before that is still its own first blocker and needs section-local proof.**

## 存储驱动 / CSI 层不稳定时，先按 environment-layer setup blocker 收

如果 fresh sample 在最早的 PVC binding 阶段就停下来，常见的直接信号有：

- 所有数据 / sentinel / 协调组件 pod 长期 `Pending`
- pod events 是 `pod has unbound immediate PersistentVolumeClaims`
- PVC events 出现 `waiting for a volume to be created ... <provisioner>` 或 `driver name <provisioner> not found`
- node 侧 kubelet 报 `path is not a shared mount`
- CSI 控制面的 plugin / attacher / provisioner pod 反复进入 `ContainerCreating` / `CreateContainerError`

这种现场不应该被写成：

- “addon 启动失败”
- “addon 的 setup 流程有 bug”
- “某个产品参数没生效”

更准确的归类是：

- **environment-layer setup blocker**（存储驱动 / 节点 mount / CSI 控制面自身的可用性问题）

排查口径要分两层：

1. **是否还能用同一 storage class 继续测试**
   - 看 CSI 相关 pod 是否处于稳定的 `Running`，PVC binding latency 是否仍在合理范围。
   - 如果 plugin 反复重建、底层 mount 报错，就不要继续在该 storage class 上下产品结论。
2. **是否有可用的备用 storage class**
   - k3d / dev 集群里通常会有一个最简单的 `local-path` 之类的本地驱动。
   - 只要测试目标不是 PVC 扩容、跨节点 attach 这类强依赖具体 CSI 能力的 case，备用 storage class 就足以推进 setup / scale-out / role / switchover / reconfigure 这类主线验证。

这一轮的处理顺序固定为：

1. 先把当前 sample 的所有 `Pending` PVC、pod events、CSI 控制面状态、node kubelet 报错收成 `setup-blocker/` artifact；
2. 不在这套 sample 上继续推进任何 product-level OpsRequest；
3. 清掉残留的 namespace / cluster；
4. 切到可用的 storage class（例如显式 `STORAGE_CLASS=local-path`）重起 fresh sample；
5. 接受标准、验证边界保持不变，不要因为换了 storage class 就放宽产品语义。

一句话说：

- **PVC 在最早阶段就 unbound，先按 environment-layer setup blocker 收，不要写成 addon / 产品语义结论。**

## 何时选择本地 storage class，何时坚持可扩容 storage class

测试入口选 storage class 的标准应该和测试目标绑定，不要按惯例统一选一个。

更稳的口径是：

- **smoke / chaos / failover / switchover / reconfigure / scale-out 这类主线**
  - 默认用本地 storage class（例如 `local-path`），优先消除底层 CSI flake 干扰
  - 主线本身不依赖 PVC 扩容、跨节点 attach、快照这类能力
- **明确依赖底层存储能力的 case**
  - PVC 扩容 / restore / backup repo / 快照恢复 / 跨节点调度
  - 必须使用支持目标能力的 storage class，并在入口前先证明该 storage class 当前可用（CSI 控制面所有 pod `Running`、有最近成功 binding 的对照样本）
- **混跑 case**
  - 同一轮跑既要主线又要 PVC 扩容时，先把可扩容 storage class 的环境门禁过一次，再决定该轮统一用可扩容 storage class 还是分两轮跑

如果在跑“主线 + 单测 PVC 扩容”混跑时，可扩容 storage class 出现 driver-level flake，更稳的处理是：

1. 把出错样本归 environment-layer setup blocker，不混入主线产品结论；
2. 主线那部分用本地 storage class 单独再跑一轮；
3. PVC 扩容那部分单独等 CSI 修好后再跑，不要把可扩容 storage class flake 倒灌成 addon 缺陷。

一句话说：

- **storage class 不是惯例选项，是测试目标的一部分；底层 flake 出现时，按目标拆轮，不要让一类 case 的环境问题污染另一类的产品结论。**

## artifact root 必须放在持久目录，不要落在系统级 `/tmp`

测试样本的 artifact root（包含 raw events、controller log tail、kbagent log、pod desc、Ops yaml、final summary）必须落在持久目录，不要使用任何会被系统周期性清理的位置。

实际遇到过的失败模式是：

- root 设在系统 `/tmp/<run-id>/`
- 在 switchover / reconfigure action window 还没结束时，OS 已经清掉 `/tmp` 下的子目录
- 留下来的 live 现场虽然还能看 OpsRequest `Succeed`，但关键 action-window 原始行已不可恢复
- 这一轮事后只能按 artifact retention loss 归档，不能下产品结论

更稳的做法是：

1. **固定 root 放在持久目录**
   - 例如 agent 自己的 `~/.../artifacts/<task>-<sample>/`
   - 或仓库级 `artifacts/<task>-<sample>/`
   - 不要使用 `/tmp/<sample>` 或任何挂在 tmpfs / 周期性 GC 的目录
2. **runner 启动时 echo 一次 root 路径**，并在 final summary 顶部再写一次，方便 review 时复核
3. **action-window 期间持续轮询关键日志快照**
   - 例如 `/tmp/switchover.log` 这类 in-pod 输出，要在 action-window 内持续 cp 出来
   - 即使最终一份被覆盖，轮询快照仍可以拼接出原始行序列
4. **如果发现 artifact 已经被清**
   - 立即把这一轮归 `non-conclusive / harness-retention-loss`
   - 重起一轮 fresh sample，root 切到持久目录，再下结论

一句话说：

- **artifact root 是证据的住所；落在系统级 `/tmp` 等于把证据放在会被清理的桌面，下一轮就丢。**
