# Addon Design-Contract Review During XP 指南

> **Audience**: addon dev / test 工程师（XP 模式下两个角色同一人）+ pair-programming reviewer
> **Status**: stable
> **Applies to**: any KB addon（设计契约级 review 方法论，不绑定单引擎或单 ops 类型）
> **Applies to KB version**: any（review 模式与 KB 版本解耦；引擎特化 8 类 blocker 实证在文末附录）
> **Affected by version skew**: 不受 KB 版本影响 — review checklist 跨 KB 版本通用

本文面向 Addon 开发与测试工程师（XP 模式下他们是同一个人），聚焦一个常被默认但很少显式验证的属性：**review 阶段必须做"设计契约级 challenge"，不只是 syntax / lint / 测试通过的检查；这种 review 抓到的问题会比 e2e 早一个数量级**。

正文写通用方法论，引擎相关的实证（具体 8 类 blocker 的代码现场）在文末附录。

## 先用白话理解这篇文档

很多团队 review 是"代码 diff 没明显问题就 LGTM"。这种 review 抓得到笔误、命名不当、缺 nil-check 这类表层 bug，但抓不到下面这类设计契约级缺陷：

- "这个函数处理一个不该出现的 case 时静默 fallback，恰好覆盖了我们要消除的 race"
- "两段代码对同一个 in-memory 状态分别做了 read 和 write，但中间有 commit 边界，状态不连续"
- "一段 valid 数据通过了我们写的 schema 校验，但语义上是 'empty intent'，让消费者降级成 'no intent'"

这些缺陷的特点是：

1. **Compile pass / 单元测试 happy path pass / 简单 e2e pass**。
2. **只在低频场景或异常路径触发**，常规测试很难命中。
3. **触发后的现象往往是"状态不一致"，不是"crash"，更不是 fatal error**，所以日志里不显眼。
4. **修复成本随发现位置呈指数增长**：design phase 修是 5 分钟，review 阶段修是 30 分钟，单测阶段修是 2 小时，e2e 复现 + 修是半天，production 发现修是几天 + 信任损失。

这篇文档解决的不是"如何减少 bug"，而是 — **当一个第二个人 review 你的代码时，他应该按什么 checklist 去 challenge 你的设计契约**，把缺陷锁在前两个阶段而不是后三个。

## 适用场景

当你在做下面任一件事时，本文适用：

- 用 XP 模式跟另一个人结对开发或交叉 review 一段非平凡逻辑（race fix、并发协调、跨控制器交接、状态机推进等）。
- 写 ship-readiness 审计前，希望自己的代码在 review 阶段先被一轮"contract challenge"过滤。
- 团队 onboarding，给新进的 addon 工程师一个具体的 review 标准，让他们少 LGTM、多 challenge。

## 核心原则：review 是一次"设计契约审计"

把一段代码看作一组对外承诺（contracts）。每次 review，第二个人不是在看 syntax，而是在 audit 这组契约：

1. **Identify**：这段代码隐含了哪些对外承诺？（例：参数非空、状态可重入、调用顺序、并发安全、失败模式）
2. **Stress test**：每个契约在反例输入、极端时序、异常状态下都成立吗？
3. **Document**：契约必须在代码注释或文档里显式写出，不是埋在测试里。

如果一段代码的契约本身没法被列出来，那 review 还没完。

## 8 类设计契约级 blocker（按出现频率从高到低）

下面这 8 类是从一次完整 race fix（KubeBlocks issue #10190）的 XP 实施周期里抓到的实例。它们并非 issue-specific，而是泛型的契约缺陷模式，适用于任何分布式控制器代码。

### 1. 静默 fallback 覆盖了你想消除的 race

代码里有一个 helper（如 `LookupSomething(...)`）默默把"解析失败"和"未找到"都返回成 `not found`。表面合理，但调用方的逻辑是"如果 not found 就走 fallback 路径"。如果 fallback 路径正是你想消除的 race，这个 helper 就把 race 重新引回来了。

**Review 模式**：每个 helper 都问 "失败模式有几种？分别怎么报告？调用方在每种失败模式下走的路径是 race-safe 的吗？"

**修法**：失败模式分开 — parse error 必须能 propagate，跟 "no entry" 要区分。

### 2. 契约字段的"非空"没有在写入时强制

设计文档里说 "field A is the only conflict guard, must be non-empty"。但代码里的 Merge / Add / Set 函数没有在写入时拒绝空值。结果上游可以写一条 "anonymous entry" 进来，下游守不住。

**Review 模式**：所有 "must be non-empty"、"must be unique"、"must match X" 的字段，写入路径都得有 reject 分支。

**修法**：在 Merge 入口加 `if entry.X == "" { return error }`，并加单测覆盖 reject 分支。

### 3. 同 commit 边界内 state 不连续被误用

控制器框架里常有这种模型：reconciler A 在 in-memory tree 上 `Delete(obj)`，reconciler B 在同一 commit 内 `Add(obj)` 同名对象，框架在 commit 时把这俩动作 fold 成一个 `Update`。如果 obj 上有 immutable 字段（PVC `spec.volumeName`、Pod `spec.nodeName` 等），这个 Update 会失败。

**Review 模式**：每段"删除 + 重建同名对象"逻辑都问 "这俩动作落在同一个 commit 还是分两个？框架的 plan 是基于 in-memory 中间态还是基于初始 vs 最终 snapshot？"

**修法**：删除分支必须 `RetryAfter / Commit-and-requeue`，不能 `Continue` 让后续 reconciler 同轮再 Add。

### 4. 用 sentinel 值（空字符串、零值）传递错误

类似 `func computeName(...) string { if invalidInput { return "" } }`。调用方写 `if name == "" { ... }` 是为了某种"空名"路径，但它实际还需要区分"caller intentionally asked for empty" vs "this helper failed silently"。两条路径混在一起 → 一种合法情况下的"空名"会触发错误处理，一种错误状态会被当合法情况处理。

**Review 模式**：所有返回 sentinel 值的函数，都改成 `(value, error)` 或 `(value, ok bool)`，让调用方显式区分。

**修法**：把 helper 签名改成 `(string, error)`，调用方 propagate error，原本走 sentinel 分支的代码改成检查 error。

### 5. 条件清理依赖"先验状态"，但状态会在 in-flight 重建中改变

代码 "如果 X 引用了 Y 就清掉 X" 的写法，假设 X 的引用只来自一个 known writer。如果另一个 actor 在中间也会写 X 的引用，再加上你的代码会被多次执行，第二次执行时 X 上的引用可能是另一个合法 writer 写进来的，你不应该清。

**Review 模式**：清理动作的前提条件 must 写明"在哪些状态下我清，哪些状态下我留"，状态枚举要穷尽，不要漏第三方 writer。

**修法**：清理条件细化成枚举 — `nameRef pointing at A → clear; pointing at B → leave; pointing at anything else → fatal`。

### 6. NotFound 短路掉了"应该写入的副作用"

代码 `if cli.Get(X) returns NotFound, return err` 的写法，把 NotFound 当成 fatal 错误。但有些场景里 X 缺失是合法过渡态（被另一个控制器 in-flight 重建），代码该做的事不是退出，而是 **先写入意图**（让 X 重建时被改成正确形态），然后 wait。

**Review 模式**：每个 `cli.Get` 后的错误处理都问 "NotFound 在这条路径上是 fatal 还是过渡态？"

**修法**：NotFound 分支单独处理 — 设 sentinel `obj = nil`，下游逻辑允许 nil 进入 wait 路径。

### 7. 释放序列不区分"对象 still terminating" vs "对象 已 absent"

代码看 `if obj.DeletionTimestamp != nil, skip`，假设对象已经走完 termination。但在很多 release 序列里 "正在 terminating 但还没消失" 跟 "已经 absent" 是不同语义：

- terminating：对象还在，引用 / 锁 / mount 还在；
- absent：对象彻底消失，引用 / 锁 / mount 全部释放。

如果释放序列依赖 "absent" 而不是 "terminating"，那 terminating 时不能假装它已 absent 而提前进入下一步。

**Review 模式**：每个 `DeletionTimestamp != nil` 的分支都问 "我接下来的动作依赖对象彻底 absent，还是 terminating 就够？"

**修法**：terminating 分支单独 RetryAfter，等下一轮真正 absent 再推进。

### 8. 编译期成立但运行期 NPE 的运算符优先级陷阱

类似 Go 里的 `if a != nil && a.x != b || a.y != c {}`。`&&` 比 `||` 紧，所以这个表达式是 `(a != nil && a.x != b) || a.y != c`，当 `a == nil` 时仍然会 evaluate `a.y` 并 NPE。同 family 的还有：`!cond1 && cond2 || cond3`、`a == nil || a.method() == ...` 中的 short-circuit 误用。

**Review 模式**：所有混合 `&&` 和 `||` 的 boolean 表达式都强制加括号；nil-check 后的 `||` 分支要单独验证不会误 deref。

**修法**：括号显式化 + 加单测覆盖 nil 输入。

## 反面：传统 Dev/Test split 漏掉这 8 类的概率

"开发写代码、测试只跑 case" 的工作模式下，上面 8 类大部分都不会被抓到：

- 类 1、2、4 是 helper 内部契约缺陷，只看 e2e 测试不会触发，因为 happy path 不需要 fallback。
- 类 3 是框架语义陷阱，只看代码 diff 不读框架 commit 模型不会发现。
- 类 5、6、7 都是 race 路径上的状态枚举不穷尽，只在低概率时序下触发，e2e 跑 N=10 都未必命中一次。
- 类 8 在 syntax check 通过但运行期偶尔 NPE，没人 review 表达式优先级时会漏掉。

如果走 Dev/Test split，这 8 类都会带到 e2e 才暴露，然后开始 evidence collection、root cause 排查、修复、重测的循环。每个循环 30 分钟到几小时不等。XP 模式把这些循环全压在 review 阶段，每条 5-15 分钟修完。

## 推荐落地

1. **Pre-commit checklist**：每段 PR 先按上面 8 类自查一遍。任何一类不能立刻给 "不适用" 答案的，写明为什么并加测试。
2. **Review checklist**：第二个人 review 时按 8 类逐项 challenge。"我没看出问题" 不算 review pass；要能说出"类 1：我已经确认 X helper 的失败模式都 propagate"。
3. **Test 同步**：每修一类，加一个测试 pin 修复后的契约（包括 negative path：把 invalid 输入扔进去看是否 reject）。

每类 blocker 的具体 fix + 单测对应在源码里的位置和说明，可以在引擎相关的 case file 里展开。

## 案例附录

引擎相关的 8 类 blocker 真实代码现场放在独立的 case file 里：

- 一个完整周期里 8 类 blocker 全部出现的实证记录可以参考 KubeBlocks issue #10190 / PR #10191 的 commit 流（每个 fix commit 的 message 已经显式注明属于哪一类 blocker），作为新人 onboarding 时的 demonstrative 材料。
