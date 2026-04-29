# Addon Bootstrap Role Publish 指南

本文面向 Addon 开发与测试工程师，聚焦一个经常被混读的问题：当 primary service、pod role label、EndpointSlice 或 VIP reachability 表现异常时，问题不一定先出在 service 或 controller publish 链路，也可能更早出在 bootstrap 初期的 role truth 发布。

正文只写通用方法论，不绑定某一个引擎。

## 先用白话理解这篇文档

这篇文档最想讲清楚的一件事是：

- **谁真的是 primary**
- 和
- **系统把谁对外公布成 primary**

不是一回事。

可以把它想成开会：

- `role truth` 像是屋里真正拍板的人是谁
- `publish chain` 像是门口名牌、会场指示牌、前台广播是不是都写对了

如果屋里拍板的人一开始就认错了，后面门牌、广播、前台全错，其实都只是连带结果。

反过来，如果屋里已经认对了，但门牌还没换，那问题就不在“谁是 primary”这层，而在后面的发布链。

## 适用场景

当你遇到以下任一现象时，这份文档适用：

- primary service VIP 指向了错误成员
- pod role label 和数据库实际角色不一致
- EndpointSlice 与 role label / service selector 长时间脱钩
- failover / bootstrap 初期角色频繁翻转
- 团队开始争论 service selector / controller retry，但 role truth 其实还没稳定

## 先分两层 truth

排这类问题时，先把问题分成两层：

1. **role truth**
- 数据面谁真的是 primary / secondary

2. **publish chain**
- pod role label
- service selector
- EndpointSlice
- VIP reachability

如果 role truth 在最前面就错了，后面的 publish chain 很可能只是连带结果。

## 不要把“自举已成功”和“对外发布已闭环”混成一层

白话一点说：

- 数据库里真正的 primary 已经出来了
- 不等于 service / VIP 已经把流量带到它身上了

前者像“内部已经选出负责人”，后者像“公司通讯录、前台牌子、外部联系方式都改好了”。

内部负责人选对了，只能说明第一层已经成立；不能直接推导整条对外链也已经恢复。

很多现场里，这两层会被一句“功能还没跑通”混在一起，导致判断失真。

更稳妥的拆法是：

1. **自举 / 复制协同已成功**
- 实例已经起来
- bootstrap helper / replication coordinator 已完成关键初始化
- 数据面角色已经正确
- 手工 roleProbe 或等价本地探针已经能返回正确角色

2. **对外发布已闭环**
- pod role label 已落下
- service selector 已指到正确成员
- Endpoint / EndpointSlice 已收敛
- VIP / 对外访问路径已恢复

第一层成立，只能说明：

- 问题已经不在“自举根本跑不起来”这一层

但这还不能自动推出：

- 下游 publish chain 已经恢复
- 整条 async / failover / 对外服务链已经闭环

所以在汇报和收口时，应该明确区分问题层级：

- 如果被问的是“复制协同 / bootstrap 这层能不能跑”，就用数据面角色、自举日志、本地 probe 结果回答
- 如果被问的是“整条对外服务链是否已经恢复”，就必须继续看 label / selector / endpoint / VIP

不要因为第二层还没闭环，就回退成“第一层也还没跑起来”。

## 不要先把问题收成 service/VIP 现象

看到 service 或 VIP 异常时，不要直接收成：

- selector 配错了
- EndpointSlice 没刷新
- controller retry 丢了

更应该先回答：

- role truth 是不是已经稳定正确
- bootstrap 初期有没有把错误角色过早发布出去

## bootstrap 初期最危险的点：过早发布最终角色

当以下条件仍未收敛时：

- HA lease
- member join
- 复制关系
- datadir / ready marker

如果 roleProbe 仍然过早返回最终的 `primary / secondary`，就很容易把整条 publish 链一起带偏。

典型后果是：

- 实际 primary 被打成 secondary
- pod role label 跟着错误 role truth 走
- service selector 继续按错误 label 选
- EndpointSlice 也被带到错误成员

## 推荐的 bootstrap 发布口径

在 ready 之前，更稳妥的策略通常是：

- roleProbe 统一返回 `initializing`
- 不在 bootstrap 中间态发布最终角色
- 等本地状态、复制关系、member join 等关键条件都收敛后，再发布 `primary / secondary`

这条口径的价值在于：

- 先阻断错误 role truth 被过早对外扩散
- 再观察 publish chain 是否自然恢复

## 不要把 role truth gate 和 service-ready gate 混成一层

白话一点说，这里其实是在回答两个不同问题：

1. **现在能不能判断谁是 primary**
2. **现在能不能把流量正式引过去**

第一问更像“先认人”，第二问更像“再放行”。

如果把两问绑死在一起，就会出现：

- 其实已经能认出谁是 primary
- 但因为它还没完全 ready，对外放行条件没过
- 最终连“谁是 primary”这件事也一起回答不出来

这就是为什么文档一直强调：role truth gate 要尽量小，service-ready gate 可以更强。

排 bootstrap 问题时，要特别警惕一种混读：

- probe 想回答的是“当前谁更像 primary / secondary”
- 但实现里先卡了一个更强的 service-ready / socket-ready / fully-ready gate
- 结果不是 role truth 错了，而是 probe 根本回不出可消费结果

这类情况下，控制面后面看到的往往只是：

- probe failed
- code 统一落成错误
- label / service / EndpointSlice 全都没法继续发布

更稳妥的设计通常是把两层 gate 拆开：

1. **role truth gate**
- 只回答“当前有没有足够证据判断角色”
- 可以依赖本地、单调、低抖动的证据，例如 datadir 状态、marker、复制元数据、bootstrap 阶段状态文件

2. **service-ready gate**
- 回答“这个实例现在是否已经适合对外承接流量”
- 这层可以继续要求 socket ready、lease 收敛、member join 完整等更强条件

这样可以避免：

- 角色判定本来已经有足够本地证据
- 却因为 service-ready 还没过，整个 roleProbe 只能统一失败
- 最终把 publish chain 全部拖死

## 在真实执行器契约下，先保证“可稳定判角”，再做 best-effort 健康检查

很多 probe 在容器里手工执行看起来正常，但一旦放回真实 agent / executor 路径，就会因为环境、时序或依赖差异，被更重的检查项先卡死。

常见形状是：

- 角色其实已经能从本地持久化证据判出来
- 但脚本先去做 socket 连接、数据库 client 调用、强 ready 检查
- 这些检查在 bootstrap 窗口里抖动更大
- 最终 probe 整体返回失败，控制面只看到 failed event

更稳妥的 probe 设计通常是：

1. **先用本地、低抖动证据给出角色结果**
- marker
- 复制元数据
- datadir 内持久化状态
- 本地 bootstrap 阶段文件

2. **再把更强的检查降成 best-effort**
- socket 探活
- 数据库 client 调用
- 更强的 ready / service-ready 判断

也就是说：

- 只要已经有足够本地证据判断 `primary / secondary`
- probe 就应优先稳定返回可消费结果
- 不要让更重的健康检查把整个角色发布一起变成失败

这条经验尤其适用于：

- bootstrap 初期
- 真实执行器环境比手工 shell 更受限
- service / socket 尚未完全就绪，但角色事实已经存在

## 优先选低抖动、本地的角色证据

如果当前处于 bootstrap / 初始化窗口，更适合作为 role truth 输入的，通常是：

- 本地 datadir / marker
- 已持久化的复制元数据
- 启动阶段明确写下的本地状态文件

而不是优先依赖：

- 需要网络或 agent 执行链路才能拿到的远端状态
- 尚未稳定的租约或 DCS 输出
- 依赖 socket fully-ready 之后才成立的更强信号

原则上，role truth 应先用最小、最本地、最不抖动的证据收住；service-ready 再用更强条件收口。

## 当 role truth 依赖本地 marker 时，状态切换成功后要立即补齐 marker 合约

白话一点说，本地 marker 很像门上的状态牌。

- 里面的人已经换了角色
- 但门口牌子还写着旧状态

这时外面的人即使按牌子办事完全“合规”，结果也会继续错。

所以 marker 不是装饰品，而是 publish chain 会不会继续跑下去的一部分契约。

有一类 publish 问题，数据面真实角色已经切换成功，但本地 marker 仍停在旧状态，例如：

- promoted primary 仍残留 `pending`
- 应该表示“可发布角色”的 `ready` / `done` 标记没有补上
- roleProbe 继续把实例判成 `initializing`

这类情况下，后面的 publish chain 往往会一起表现成：

- pod role label 长时间不更新
- primary service 没有 endpoint
- EndpointSlice 继续为空，或者仍指向旧成员

问题的第一拍通常不在 publish controller，而在：

- **本地 marker 合约没有跟上数据面真实状态**

### 更稳妥的 marker 合约

如果 roleProbe 的可发布条件依赖本地 marker，那么 promote / follow / demote 或等价状态切换成功后，应明确做到：

1. **成功路径再切 marker**
- 只有在角色切换或复制状态真正成功落定后
- 才清理旧的 `pending` / `initializing` 标记
- 再补齐表示“可发布”的 `ready` / `done` 标记

2. **失败路径不要提前改 marker**
- 如果 promote / follow / demote 本身失败
- 不要先把旧 marker 清掉
- 否则很容易把真实失败伪装成“状态未知”或“现场看似已恢复”

3. **允许按 DB 真值回补陈旧 marker**
- 如果实例实际上已经是 writable primary
- 但本地仍残留旧的 `pending`
- 那么恢复路径应允许按真实状态回补 marker，避免 roleProbe 长期卡在 `initializing`

一句话：

- **本地 marker 不是附属装饰，而是 role truth 可发布性的契约之一；它必须和真实状态切换一起收敛。**

## 不要让“陈旧 pending”把 role truth 一直压成 `initializing`

这条也可以用很直白的话理解：

- 数据面真值已经说明“人已经转正了”
- 但系统门口还贴着一张旧便签：`pending`

那后面所有看便签行事的模块，都会继续把它当成“还在处理中”。

所以很多看起来像 label、service、Endpoint 没刷新的问题，第一拍其实不是发布链慢，而是上游那张旧便签根本没撕掉。

当现场表现是：

- 数据库真值已经能证明某个成员是 primary
- 但 roleProbe 仍长期返回 `initializing`
- agent / label / primary endpoint 都不继续发布

除了检查 probe 脚本本身外，还应优先确认：

- 是否还残留旧的 `pending`
- 是否缺少与之对应的 `ready` / `done`
- 这些 marker 是否本应在 promote / follow 成功后被切换

这类问题的危险在于：

- 数据面真实角色已经对了
- 但 publish chain 仍被本地旧 marker 卡死

从现象上看像：

- roleProbe 写错了
- label controller 没发布
- service / endpoint controller 没收敛

但 first blocker 往往更早，落在：

- **状态切换成功后，本地 marker 没有同步收敛**

## 推荐的判断顺序

遇到 role publish 异常时，建议固定按下面顺序看：

1. 先证真 role truth
- 数据库实例实际角色是什么
- bootstrap 初期有没有角色翻转

2. 再看 pod role label
- label 是否跟 role truth 一致

3. 再看 service selector
- service 当前在选谁

4. 再看 EndpointSlice / VIP
- endpoint 是否与 selector 一致
- VIP reachability 是否因此异常

只有在 role truth 已稳定正确之后，如果 label / service / endpoint 仍长期错位，才继续拆 controller retry 或 publish 下游问题。

## 先核 probe 契约，再怀疑 publish/controller

如果现场表现是：

- role label 没发布
- Endpoints / EndpointSlice 没起来
- controller 里又能看到 probe event 失败

不要立刻收成：

- label controller 漏发布
- service publish 链断了
- 一定是上游 exec 语义有 bug

更应该先确认：

- 当前 roleProbe 是否真的符合**实际执行器**的契约
- 在真实执行路径下，probe 返回值、exit code、stdout/stderr 是否满足控制面预期

这一步尤其重要，因为很多问题并不是“controller 没发布”，而是：

- controller 正确忽略了 failed probe event
- 真正的上游原因在于 probe 本身没有按当前 executor 契约返回可消费结果

## `RoleProbeNotDone` 要先拆 role truth / probe contract / publish chain

当现场出现：

- cluster 长期 `Updating`
- `RoleProbeNotDone`
- secondary role 未发布
- service / endpoint 没选到预期实例

不要先把它写成普通的 secondary recovery timing，也不要直接归到下游 publish controller。

更稳妥的拆法是三层：

1. **role truth 是否已经成立**
   - 实例自身是否已经具备可发布角色的真实条件；
   - replication / read-only / 本地 marker / pending marker 是否闭合；
   - 如果 role truth 还没成立，问题仍在数据面或启动协同上游。
2. **roleProbe contract 是否成立**
   - probe 在真实执行路径下的 exit code、stdout/stderr、payload 是否满足控制面契约；
   - 是否因为 timeout、executor 环境、工作目录、container 选择、shell 差异导致控制面无法消费结果；
   - 如果 role truth 已成立但 probe 仍 NotDone，优先查这里。
3. **publish chain 是否正确消费 probe 结果**
   - pod role label 是否更新；
   - service selector / EndpointSlice 是否跟着 label 变化；
   - controller retry 是否拿到了可消费 probe result。

因此，`RoleProbeNotDone` 的第一句结论不应是“publish 没发出去”，而应先回答：

- role truth 本身是不是已经成立；
- roleProbe 在真实执行器下是不是返回了可消费结果；
- publish chain 是否拿到了这个结果。

一句话说：

- **`RoleProbeNotDone` 是分层信号，不是根因；先证真 role truth，再核 probe contract，最后才拆 publish chain。**

## transient role gap 不能直接变成 terminal failure

如果 reconfigure / reload / switchover 这类动作已经证明：

- action 已执行成功
- runtime truth 已经变成预期值
- 只是 roleProbe 在某个短窗口里超时或拿不到稳定 role

不要把这类问题重新写成 action / runtime 失败。

更准确的 first blocker 是：

- **post-apply role/probe gating**

修复目标也不是“取消 roleProbe”，而是给 transient role gap 明确 gating 语义：

1. 短窗口内 role 为空、probe timeout、probe 暂时不可消费时，不要立刻把整个 Ops 判成 terminal failure。
2. 在 bounded window 内重试 / 等待 role truth 收敛。
3. 如果 role truth 在窗口内恢复，再让 Ops 继续按成功路径收口。
4. 如果超过窗口仍没有可消费 role truth，仍应 `Failed`，并保留 probe stdout/stderr、timeout、target pod 和执行器信息。

验收时要同时证明两件事：

- 不会因为一次 transient role gap 误失败；
- 也不会因为等 roleProbe 而无限挂住。

一句话说：

- **role-probe gating 修的是短暂 role 空窗导致的终态误判，不是取消 roleProbe contract。**

## 不要只看脚本本地执行结果，要看真实执行路径

排这类 probe/publish 问题时，不要只验证：

- 脚本在容器里手工执行是否看起来正常
- 逻辑分支本身是否合理

还要继续确认：

- probe 是通过谁执行的，例如 agent / sidecar / controller 代理
- 真实执行时用的是哪个 container、哪个 shell、哪个工作目录
- 返回给控制面的 code / message / payload 形状是什么

也就是说，应该优先回答：

- **probe 契约** 和 **真实执行器契约** 有没有对上

如果没对上，后面的 label / service / endpoint 不收敛，可能只是连带结果。

## 最小归属判断

当 probe 在真实执行路径下只返回失败形状，例如统一落到某种 `code=-1` 或等价错误，而 controller 又只是按规则忽略 failed event 时，建议把归属判断先收成二选一：

1. addon probe 需要适配当前执行契约
2. 确实存在上游 executor / exec 语义需要修正

在这一步收清前，不要过早把修复直接落到：

- label controller
- EndpointSlice controller
- service selector

因为这些组件可能只是在消费一个已经失败的 probe 结果。

## 不要把下游 publish 问题和上游 bootstrap 问题混成一条

实践里最常见的混读是：

- 一边还没证真 role truth
- 一边已经开始怀疑 service / controller / EndpointSlice

这会导致排查面越拆越散。

更稳妥的收法是：

- 先把 bootstrap 初期的角色发布条件收紧
- 再只做一次定向验证
- 看 role label、service selector、EndpointSlice 是否恢复对齐

## 定向验证优于继续跑整轮测试

当问题已经收窄到某一个 publish 阶段时，不要继续跑整轮 async / chaos。

更适合的做法是：

- 只跑对应阶段的定向验证
- 只回 role truth、label、selector、EndpointSlice、VIP 这几类实物

这样更容易回答：

- 新脚本 / 新逻辑是否真的进 live
- publish chain 哪一层先恢复
- 还有没有必要继续拆 controller retry

## 建议固定回收的证据

每次处理这类问题时，建议至少回以下内容：

1. 各成员在失败窗口里的实际数据库角色
2. roleProbe 在同窗口返回了什么
3. pod role label 的实际值
4. primary service 的 selector
5. EndpointSlice 当前实际后端
6. VIP reachability 结果
7. 是否仍处于 bootstrap / HA lease 未收敛窗口
8. probe 在真实执行路径下的返回 code / payload / error 形状
9. role truth 判定依赖的是本地低抖动证据，还是被更强的 service-ready gate 挡住了

## 最小经验收口

可以把这类问题的经验收成一句：

- **先证真 role truth，再拆 publish chain**
- **若 publish 链路没动作，再先核 probe 契约与真实执行器是否匹配**
- **不要让更强的 service-ready gate 把本来可判定的 role truth 一起拦死**

如果前一层还没收稳，后一层的 service / endpoint 现象通常不是首个根因。
