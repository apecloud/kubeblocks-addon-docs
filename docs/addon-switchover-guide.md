# Addon Switchover 开发与幂等性指南

本文面向 KubeBlocks Addon 开发者，重点总结 switchover / failover 生命周期动作中的设计原则、幂等性语义、常见坑。引擎相关内容只放在案例附录中。

## 先用白话理解这篇文档

### 这篇文档解决什么问题

写 switchover action 时，最容易踩的不是"切换没成功"，而是"看起来切换成功，但 controller 重试一次后集群状态错了"。

新读者通常会把 switchover 当成一个原子操作来写：调用 → DB promote → 返回 success。但**真实运行中 switchover 不是原子的**：候选 pod 已进入新角色，旧 pod 还没退出旧角色，sentinel/coordinator 拓扑缓存还没刷新，controller 又来重试一次。

这时候 "再次调用" 的语义不是实现细节而是产品语义：
- 如果脚本第二次调用直接 fail（"我已经切了，再切就是错的"），controller 会把这条解读成产品 fail
- 如果第二次调用直接 silent success（"已经在那个状态了，啥也不做"），controller 也可能错过收敛信号、放过未到位的状态

→ 真正问题是：**先把 success / unknown / error 三态语义定清楚，再写代码**，否则 race / 重试 / 双调用都会把集群推到歪状态。

### 读完你能做什么决策

- **设计 switchover action 时**：先定义三态边界，再决定脚本返回什么（chapter "先定义成功语义" + 原则 1-2）
- **判 controller 双调用语义时**：用状态语义还是因果语义？不同选择决定整个错误处理树（chapter "两种常见语义"）
- **写收敛校验时**：什么时候依赖 DB 拓扑 truth、什么时候依赖外部协调器，不要混层（原则 3 + 原则 6）
- **写测试时**：必须覆盖"切换中途态"（candidate 已升 / 旧 master 未降的窗口），不能只测起始 + 终态（原则 4）
- **写脚本里 member list 遍历时**：知道容器静态 env（如 `VALKEY_POD_FQDN_LIST`）跨 scale 不刷新；fresh scale-out candidate 不在 list 里（原则 7，Valkey #2609 case）

### 为什么独立成篇

switchover 跟 reconfigure / TLS / backup 等主题完全独立。它的核心问题是**幂等语义**，不是 reload 路径或安全边界。把这一类经验集中放一篇，避免新读者在多个 doc 之间拼凑。

具体引擎实现（Valkey switchover.sh / Redis redis-switchover.sh / MySQL semi-sync 等）只放在案例附录里，正文是引擎无关的方法论。

---

## 适用场景

当 Addon 需要支持以下能力时，这份文档适用：

- 手动 switchover
- 指定候选节点 switchover
- 无候选节点的任意成员切换
- 协调器 / 仲裁组件驱动的 failover
- controller 重试、重入、双调用

## 幂等性为什么是 switchover 的核心问题

switchover 不是一个完全同步的单点动作。真实执行中通常存在：

- candidate 先进入目标角色，旧角色持有者随后才退出旧角色；
- 不同协调组件的拓扑缓存刷新存在延迟；
- controller 可能在第一次请求尚未完全收敛时再次触发 switchover；
- 脚本执行过程中，容器可能重启或连接中断。

因此，“再次调用同一 switchover 应该返回什么”不是实现细节，而是产品语义。

## 先定义成功语义，再写代码

在实现 switchover 前，建议先回答以下问题：

1. 如果目标 candidate 已经处于目标角色，第二次调用是否应返回 success？
2. 如果 candidate 已处于目标角色，但旧成员还未退出旧角色，是否应视为成功？
3. 如果 candidate 已处于目标角色，但并不是因为本次请求才切换成功，是否还能算成功？
4. 如果角色暂时探测不到，应重试、warning 继续，还是立即失败？

如果没有先定义这些语义，代码实现和 code review 很容易长期摇摆。

## 两种常见语义：状态语义 vs 因果语义

### 状态语义

只要目标状态已达成，就可以返回 success。

例如：

- `candidate_role == "<target-role>"`

优点：

- 更容易支持幂等和重试；
- 更符合“重复执行同一目标动作不应失败”的直觉。

缺点：

- 可能误把“candidate 因其他原因已经处于目标角色”当成本次 switchover 成功。

### 因果语义

必须证明目标状态是由本次 switchover 导致，才返回 success。

优点：

- 语义更严格；
- 更能避免“状态看起来对，但不是这次请求完成”的误判。

缺点：

- 分布式系统里通常很难完全证明；
- 会让幂等处理复杂很多。

工程上通常采用折中策略：

- 基本接受状态语义；
- 但附加最小收敛校验，避免明显的错误状态被误判成 success。

## 不要把“未知状态”当成“确定错误”

在 switchover 过程中，经常会遇到短暂的状态不可观测，例如：

- candidate 角色探测超时；
- 网络抖动导致 role 查询失败；
- failover 刚开始，拓扑还在传播；
- 协调组件缓存尚未刷新。

脚本实现时，要明确区分：

- **确定错误**
- **暂时未知**
- **异步收敛中**

尤其不要写成：

- “只要不是 `<expected-role>` 就 abort”

因为空字符串、探测失败、瞬时未知都可能误落到“错误角色”分支。

## 最低限度的收敛校验

为了避免过宽松的误判，建议在幂等 success 之前至少确认：

- target candidate 已进入目标角色；
- 原角色持有者已经退出旧角色，或至少不再维持双主 / 双写 / 双 primary 等冲突状态；
- 集群没有明显角色冲突信号；
- 拓扑满足最小健康条件。

## Action 成功条件应优先基于 DB 拓扑 truth

switchover action 的 critical path 应聚焦“数据库拓扑是否已经切换成功”，不要把 client-facing Service / Endpoint 路由收敛直接放进 action 的阻塞成功条件里。

更推荐的分层是：

- action 内部用 pod / headless 地址直连验证 DB truth：
  - candidate 已成为目标角色，例如可写 primary；
  - 原 primary 已退出旧角色，并 follow candidate；
  - replication / role 状态满足本引擎定义的最小安全条件；
- primary Service / Endpoint / EndpointSlice 路由收敛作为 post-check、诊断信息或控制面 eventual convergence 条件；
- 如果产品确实要求 `Ops Succeed` 包含 client-facing Service readiness，应明确写成控制面 / Ops 层 contract，并配置足够的 timeout、日志采集和观测条件，不要让 addon action 在缺少 headroom 的情况下阻塞等待。

这样做的原因是：

- DB 拓扑切换和 Kubernetes Service 数据面收敛是两条不同链路；
- Service 路由可能受 EndpointSlice、kube-proxy、网络数据面或控制面发布延迟影响；
- 把 Service route 放在 action critical path，会把“DB 已切换成功”误判成“switchover action 失败”；
- 一旦外层 action timeout 小于或等于内层等待预算，还会导致 action 被截断，诊断日志也来不及输出。

因此，switchover action 的成功条件建议写成：

- **必须阻塞确认**：candidate primary、old primary follows candidate、无明显冲突拓扑；
- **非阻塞诊断 / post-check**：primary Service 是否已路由到 candidate、Endpoint 是否已发布、客户端入口是否在目标时间内最终收敛。

## 测试幂等性时必须覆盖“中途状态转换”

很多 race condition 不是“一开始就成功”或“永远失败”，而是：

- 第 1 次轮询看到旧状态；
- 第 2 次或第 N 次轮询才看到新状态。

因此，针对 switchover 的 UT / e2e 不应只覆盖：

- 立即成功；
- 持续失败直到超时；

还必须覆盖：

- **轮询过程中的中途转换**。

## Shell 测试中的 subshell 陷阱

如果 switchover 脚本通过：

```bash
value=$(get_role ...)
```

这种 `$()` 方式调用函数，那么 mock 里的 shell 变量可能跑在 subshell 中，导致：

- 计数器不递增；
- 状态机模拟失效；
- 本来想模拟“第 N 次变成功”，结果永远停留在第 1 次。

更稳妥的测试写法是：

- 用临时文件保存轮询状态；
- `Before` 初始化；
- `After` 清理；
- mock 每次从文件读取并写回新的轮询计数。

## Valkey 案例：switchover 幂等性 bug

下面这个案例来自 Valkey Addon 的真实排查。

### 暴露出的第一个问题：候选角色预检逻辑与注释不符

在 `switchover_with_sentinel()` 中，代码注释表达的是：

- 如果 candidate 的角色无法确定，应 warning 并继续；

但实际逻辑是：

- 只要 `candidate_role != "slave"` 就直接 abort；

这导致：

- 当 `get_role` 因瞬态网络抖动返回空字符串时；
- 脚本把“未知状态”误当成“角色错误”；
- switchover 随机失败。

这条经验适用于所有 Addon：

- 对于分布式探测结果，**空值 / 失败 / timeout 不应默认等于 hard failure**。

### 暴露出的第二个问题：`candidate_role == "master"` 是否直接 return 0

团队围绕这个点做了多轮讨论。

一种简单做法是：

- 只要 candidate 已经是 master，直接返回 success。

但这会有边界问题：

- candidate 可能不是因为本次 switchover 才成为 master；
- 旧主可能仍短暂报告 master；
- 集群可能尚未完全收敛。

最终更稳妥的修复方式是：

1. 当发现 candidate 已经是 master 时，不立刻 return 0；
2. 轮询旧主 `KB_SWITCHOVER_CURRENT_FQDN` 的角色；
3. 如果旧主在窗口内变为 slave，则把这次调用视为幂等成功；
4. 如果旧主仍是 master，则按 split-brain 风险 abort；
5. 如果旧主状态始终无法确认，也按不能确认收敛 abort。

这使得脚本能够正确处理：

- “第二次调用进来时，第一次 switchover 已经基本完成”的场景；
- 同时不会把“candidate 已是 master 但旧主未退位”的状态误判为安全成功。

### UT 层面还暴露出的一个测试陷阱

在为“轮询中途 master -> slave”场景补单元测试时，团队发现：

- 由于脚本通过 `$()` 调用 `get_role`；
- 之前使用 shell 变量计数的 mock 方案在 subshell 中失效；
- 所以测试无法稳定模拟“第 N 次轮询才变成功”。

最后改为：

- 通过 `/tmp` 下的临时文件保存轮询次数；
- 每轮 mock 从文件读取、递增、再写回；
- 成功模拟出“中途转换”场景。

## 对其他 Addon 开发者可复用的原则

### 原则 1：先定成功语义，再谈幂等实现

**何时适用**：开始写新 switchover/failover action 脚本时；review PR 看到"判到 candidate 是 master 就 return 0"这类直接实现时。

不要直接写"看到 candidate 已是 master 就 return 0"。先明确：

- 你们接受哪种成功语义；
- 需要什么最低限度的拓扑收敛证明；
- 哪些重复调用应该成功，哪些应该报错。

### 原则 2：未知状态应与错误状态分开处理

**何时适用**：写错误处理树的时候；遇到 timeout / 短时连接断开 / 探测不到角色时；写 retry 逻辑时。

探测不到角色、短时 timeout、缓存未刷新，不等于角色非法。

### 原则 3：幂等 success 不能没有收敛保护

**何时适用**：写 "重复调用幂等 return 0" 这条 path 时；判第二次 / 第三次 controller retry 应该返回什么时。

重复调用可以成功，但 success 不应完全无条件。

### 原则 4：race condition 测试必须覆盖"中途切换"

**何时适用**：写 ShellSpec / 单元测试时；review 测试覆盖率时；某个 race fix 修完后想验证不引入回归时。

否则你可能修掉一种假失败，却引入另一种误判。

### 原则 5：shell 测试要警惕 subshell 语义

**何时适用**：写 ShellSpec mock 时；mock function 配合 `$(...)` / 管道使用时；想"用 shell 变量计数 mock 被调用次数"时。

涉及 `$()`、管道、子进程时，不要用普通 shell 变量模拟多轮状态机；用 `/tmp/<test-marker>.$$` 这种文件状态。

### 原则 6：不要把 Service 路由收敛放进 action critical path

**何时适用**：设计 switchover action 的成功条件时；考虑"是否要等 primary Service Endpoint 切到新 primary 才返回 success"时。

switchover action 应以 DB 拓扑切换成功为核心成功条件。primary Service / Endpoint 路由收敛可以作为 post-check、诊断或控制面最终一致性条件；除非产品 contract 明确要求，否则不要让 action 因 Service route 尚未收敛而失败。

### 原则 7：action 脚本里遍历的 member list 不能只信容器静态 env

**何时适用**：写遍历 `<ENGINE>_POD_FQDN_LIST` / `<ENGINE>_POD_NAME_LIST` 这类 member list 的脚本时；scale-out 后立即触发 targeted switchover 的场景；OpsRequest 报"未确认 new primary"但 sentinel / 协调器视角已切换时。

很多 addon 的 switchover / failover / replicate-priority / re-register 类脚本会遍历一个由控制面注入的 pod 成员列表，例如：

- `<ENGINE>_POD_FQDN_LIST`
- `<ENGINE>_POD_NAME_LIST`
- `SENTINEL_POD_FQDN_LIST`
- `CURRENT_SHARD_POD_FQDN_LIST`

这些变量大多通过 `componentVarRef.podFQDNs / podNames` 注入到容器 env，这意味着：

- 它们是在**容器启动那一刻**由控制面解析、写进 pod env 的；
- pod 自身的 env 在容器生命周期内**不会**自动随集群拓扑（例如 scale-out）刷新；
- 当 `replicas` 从 N 增加到 N+1，**已经存在的 pod**（包括 old primary）env 中的这份 list 仍是 N 项；
- 即便控制面在调用 action 时重新计算并下发了新 list，如果 addon 脚本仍直接读容器 env 而不接收 action 入参里的最新值，新增成员就会被脚本遗漏。

这在 scale-out 之后立即发起 targeted switchover 时最容易暴露：

- candidate 是 fresh scale-out 出来的 pod；
- old primary 的 action 脚本去“给所有成员设置 priority / 收集 role / 等待 new master”，遍历的还是旧 list，**根本看不到 candidate**；
- 即使 sentinel / 协调组件已经把 candidate 选成 new master，addon 自己的等待循环仍 timeout 报“未确认 new primary”。

通用结论：

- **任何对 member list 的遍历，都不能只信容器静态 env。**
- 至少要在脚本里再合并一次 action 入参 / 控制面下发的 candidate FQDN，例如：
  - `pod_fqdns_with_candidate "${expected_fqdn}"` 这类辅助函数，把入参里的 candidate union 进环境 list；
  - 在所有 priority bias、priority restore、role 探测、`wait_for_new_master` 这类遍历点统一使用 union 后的 list。
- 控制面侧也要保证 action 入参里能拿到当前 candidate FQDN，不能只依赖容器 env。

shellspec / unit test 必须显式覆盖这个场景：

- 入参 `expected_fqdn` 在 `<ENGINE>_POD_FQDN_LIST` 中**缺失**；
- 验证脚本能找到 candidate、能给 candidate 设置 priority、能确认 candidate 已成为 master。

排障时更稳的口径：

- 看到 switchover Ops `Failed`、Sentinel / 协调器视角已经完成切换、但 addon 自检仍在“某个时间窗口内未确认 new primary”，**先怀疑 stale env list**，而不是直接归到 sentinel / runtime 缺陷；
- 收证据时固定：old primary action 容器里 `env | grep POD_FQDN_LIST` 的实际值、scale-out 后 cluster `replicas` / pod count、controller 调用 action 时实际下发的 candidate FQDN、以及脚本遍历点的入参快照。
