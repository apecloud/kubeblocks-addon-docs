# Addon ComponentDefinition 升级与 Immutable Spec 指南

本文面向 Addon 开发与测试工程师，聚焦一个很容易被误判的问题：Addon 升级失败时，表面看起来像 cluster 卡在 `Creating`、component 不可用，甚至像 runtime 没起来，但 first blocker 其实可能更早地卡在 live `ComponentDefinition` 自身。

正文只写通用方法论，不绑定某一个引擎。

## 先用白话理解这篇文档

这篇文档最想讲清楚的是：

- 有时候不是“数据库没起来”
- 而是“施工蓝图先坏了”

这里的 `ComponentDefinition` 更像施工蓝图。

- pod / runtime 是按蓝图往下施工
- 如果蓝图对象自己已经 `Unavailable`
- 后面现场再像“机器没开起来”，第一拍也未必在 runtime

所以排这类升级问题时，先看 live `ComponentDefinition`，本质上是在先确认：

- 是施工阶段出问题
- 还是图纸阶段就已经卡死

## 适用场景

当你遇到以下任一现象时，这份文档适用：

- addon upgrade 后，cluster 长时间停在 `Creating`
- cluster condition 里有 `ApplyResourcesSucceed`，但 component 仍是 `has no available condition`
- 失败窗口里没有 pod 真正拉起来
- controller 日志提示 `ComponentDefinition` 不可用
- 你怀疑某次同名升级改到了 immutable spec

## 先别看 runtime，先看 live `ComponentDefinition`

遇到这类失败时，不要先发散到：

- SQL / runtime 语义
- pod 启动脚本
- 复制拓扑

更应该先钉实：

- live `ComponentDefinition.status.phase`
- live `ComponentDefinition.status.message`
- controller 对这条 `ComponentDefinition` 的报错

如果已经出现：

- `status.phase = Unavailable`
- `status.message` 指向 immutable update 或其他定义对象错误

那说明 first blocker 比 runtime 更早。

## 不要把 `ApplyResourcesSucceed` 误读成“定义层已健康”

白话一点说：

- `ApplyResourcesSucceed` 更像“图纸已经送到了工地”
- 不等于“图纸已经被审核通过，而且现在可施工”

如果 `ComponentDefinition` 自己还是 `Unavailable`，那资源虽然 apply 进去了，真正的定义层健康并没有成立。

`ApplyResourcesSucceed` 只能说明资源 apply 成功，不等于定义对象已经可用。

如果同时出现：

- `ApplyResourcesSucceed`
- `component ... has no available condition`

那就要继续查：

- `ComponentDefinition` 本身是不是已经 `Unavailable`
- controller 是否在报定义对象不可用

## 识别 immutable blocker 的最小证据链

建议最少固定回以下几项：

1. live `ComponentDefinition` 的：
- `metadata.generation`
- `status.observedGeneration`
- `status.phase`
- `status.message`

2. controller 日志里的对应报错：
- 例如 immutable fields 不能更新
- 或引用的 `ComponentDefinition` 不可用

3. 失败窗口里是否真的没有 pod 拉起

这三类证据通常足够把问题前移到：

- 定义对象升级路径

而不是继续误收成 runtime 没起来。

## 真正触发 immutable 的路径要精确到字段

不要笼统记成：

- “这轮 CD 改了很多东西”
- “模板重构后 spec 变了”

更有用的收法是：

- 精确到哪一条 immutable path 被改了
- 判断它是不是这轮 same-name 升级的唯一实质 diff

推荐做法是：

- 用 pre-upgrade 或 pre-delete live snapshot
- 对比当前 rendered spec
- 把 API 默认值回填和真正的业务 diff 分开

这样可以把 blocker 准确收成：

- 哪一条字段变化触发了 immutable update

## 当前现场无人引用时，修复动作先二选一

如果你已经确认：

- live `ComponentDefinition` 因 immutable update 失败而 `Unavailable`
- 当前现场又没有 cluster / component / instanceset 继续引用它

那后续修复动作不应继续模糊成“再试一次 upgrade 看看”，而应直接二选一：

1. `delete / recreate` 同名对象
2. 改走新名字 / 新版本对象

在拍定修复路径前，继续盲目重跑测试，通常只会重复撞同一条 blocker。

## 正式产品路径：不要依赖同名 in-place upgrade

这条也可以用很直白的话理解：

- 你不能一边继续用老门牌号住人
- 一边偷偷把房子的地基换掉

immutable spec 就属于这种“地基级别”的东西。

如果它变了，更稳妥的产品路径通常不是继续拿同名对象硬覆盖，而是：

- 用新名字 / 新版本起一份新定义
- 再把引用逐步迁过去

这样做虽然看起来更麻烦，但它换来的是：

- 现场语义更清楚
- 回滚和迁移边界也更稳

对正式产品路径，更稳妥的口径通常是：

- 凡是会碰 immutable spec 的 `ComponentDefinition` 变更
- 不再走同名 in-place upgrade
- 改成新名字 / 新版本承接

如果当前命名链路已经由版本驱动，最直接的做法通常是：

- bump 版本
- 让新 `ComponentDefinition` 名字跟着版本一起变化
- 让 helper 自动把相关引用切到新名字

这能避免：

- 继续复用旧名覆盖 immutable spec
- 在同名对象上反复撞不可用路径

## 旧对象的清理门槛

新版本发布时，建议先新增，不删旧对象。

只有在引用检查为 0 后，旧 `ComponentDefinition` 才进入清理。

至少要确认：

- 没有 `Cluster` 仍引用旧名
- 没有 live `Component` 仍挂在旧对象上
- 没有 `InstanceSet` 或其他衍生对象仍沿着旧定义链路运行

也就是说：

- 新版本靠新名字承接
- 存量切换走显式迁移
- 引用归零后再清旧对象

## 推荐的判断顺序

建议固定成这条顺序：

1. 先查 live `ComponentDefinition.status`
2. 再查 controller 对定义对象的报错
3. 再判断是否是 immutable path 被改
4. 再决定 `delete / recreate` 还是新名字 / 新版本
5. 只有定义对象恢复可用后，再继续进入 runtime 排障

## 建议固定回收的验证项

每次处理这类升级问题时，建议至少回以下内容：

1. live `ComponentDefinition` 的 `phase / message`
2. controller 对这条定义对象的关键报错
3. 是否有 pod 拉起
4. 触发 immutable 的精确字段 diff
5. 当前现场是否还有对象引用旧定义
6. 当前选择的 live 修复路径
7. 正式产品路径是否已经改成新名字 / 新版本

## Spec-Changing ComponentDefinition Requires a New Versioned Name

KubeBlocks ComponentDefinition contains immutable fields. For addon development, a Helm release revision advancing does not prove a changed ComponentDefinition is usable. If the chart keeps rendering the same `ComponentDefinition.metadata.name`, Kubernetes may accept parts of the update while the controller marks the ComponentDefinition `Unavailable` with `immutable fields can't be updated`. New clusters that still reference that name can then fail before any pod is created.

The safe cut for spec-changing ComponentDefinition changes is to publish a new versioned name and move every reference to that name in the same render:

- `ComponentDefinition.metadata.name`
- `ClusterDefinition` topology `compDef`
- `ParamConfigRenderer.metadata.name`
- `ParamConfigRenderer.spec.componentDef`
- any other component/version selectors used by new samples

Acceptance proof should include both render and live gates. Render must show the old name is absent from the new reference chain and the new name is used consistently. Live gate must show the new ComponentDefinition is `Available`; the old unavailable ComponentDefinition may remain in the cluster, but no new sample should reference it. Only after that should a fresh sample be started.

If a gate discovers `release revision N -> N+1` but the same ComponentDefinition name becomes `Unavailable|immutable fields can't be updated`, stop before runner start. This is an apply/versioning gate blocker, not a runtime test result. Do not write conclusions about product recovery, roleProbe, or replication until a fresh sample has actually consumed the new available ComponentDefinition.

一句话说：

- **ComponentDefinition spec 变更要换新的 versioned name 并整链引用切换；Helm revision 递增但同名 CD Unavailable 时必须停在 apply gate，不能启动样本。**
