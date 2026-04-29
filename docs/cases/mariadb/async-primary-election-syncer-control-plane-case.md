# MariaDB async primary election 对齐 syncer/control-plane 案例材料

本文是 MariaDB 案例材料，不是通用开发方法论文档。

它记录 MariaDB async 当前 primary 选主问题为什么不能继续停留在 `bootstrap + roleProbe + watcher` 的拆分实现上，以及 `#13` 主线为什么要收敛到 `syncer/control-plane`。

这份材料只回答这条 MariaDB 问题线的 3 件事：

1. 这次为什么判断需要 control-plane 收口
2. 当前 MariaDB 案例里哪些旧路径要退出主决策面
3. 后续实现拆解时，这条案例有哪些固定边界

通用方法论已经上提到：

- `docs/addon-control-plane-election-guide.md`

这不是要重新开启方向讨论的草稿，而是当前主线已经确认后的案例化基线。更准确地说，它是：

- 设计输入；
- 已确认的实现基线约束；
- 后续任务化拆解时的边界对照。

目标是把问题边界、目标职责和待决问题收清楚，避免后续实现又回到“局部加 guard 修补”的路径。

## 当前结论

当前 `#13` 已固定的基线结论是：

- MariaDB async 的 primary 选主职责应收敛到 `syncer/control-plane`。
- 旧的 `startup self-elect / roleProbe / watcher` 并行决策路径不能继续作为长期主线。
- 已经暴露出的早启动、早曝光、早切换问题，不应继续靠 guard live patch 作为长期修法。
- `syncer` 应成为 primary 选举、最终确认、old primary fencing、endpoint/publication 切换的权威控制面。

当前长期实现主线是 `#13`，不是之前那条已停止的 guard live 验证线。

## 本案例为什么需要 control-plane 收口

当前实现的核心问题不是某一个 if 条件写错，而是 MariaDB async 这条问题线里，选主职责被拆散在多个阶段和多个组件中：

- 实例启动阶段可能过早自举并自认为 primary。
- `roleProbe` 对外暴露的角色可能早于整体拓扑稳定。
- watcher 或其他异步路径可能在拓扑尚未完全收敛前触发后续动作。
- failover / switchover 的控制边界没有被单点收口。

这在本案例里带来的风险是：

- startup self-elect 暴露 primary 过早；
- startup auto-switchover 触发过早；
- 角色判断依赖多个并行信号，容易出现先后不一致；
- 短时间内可能出现 dual-primary / role oscillation；
- endpoint/publication/fencing 的执行时机不稳定。

问题的本质仍然是“谁有权最终确认 primary 并对外发布”没有被单点定义，但这条通用判断已经放进主题文档；本案例只保留 MariaDB 线上观察到的具体表现。

## 为什么本案例不继续沿 guard 路线修补

这轮排查里，团队曾经拿到过一个 guard 候选修法，并做过 live 验证准备。但用户已经明确要求：

- 这版 guard live 验证线先停掉；
- 既然已经发现问题，就不要继续修修补补。

因此 guard 线现在只能保留为证据和背景材料，不能再作为当前主线。它可以帮助我们说明旧模型为什么容易出错，但不再代表实现方向。

## 本案例里 `syncer/control-plane` 要收什么职责

在 MariaDB async 这条问题线里，`syncer/control-plane` 成为主线后，至少要对以下职责负责：

- primary 选举与最终确认；
- old primary fencing；
- endpoint/public service 绑定与切换；
- failover / switchover 的时序收口；
- 对外角色发布的最终权威面。

换句话说，本案例里的其他探针、watcher、启动脚本可以继续存在，但它们不再拥有并行拍板权。

## 本案例里旧路径哪些该删、哪些该降级

当前建议的处理原则如下。

### 应删除或退出主决策面的路径

- startup 阶段直接 self-elect 并对外暴露 primary；
- 不经过统一仲裁就独立切 endpoint 的路径；
- watcher 基于局部观察结果直接宣布新 primary 的路径。

### 可以保留但应降级为输入信号的路径

- `roleProbe`：
  - 可继续作为实例当前角色观测信号；
  - 不应单独决定对外拓扑。
- watcher：
  - 可继续负责事件监听、状态汇总、触发候选检查；
  - 不应在 control-plane 外单独完成最终切换。
- 启动脚本/bootstrap：
  - 可继续负责实例初始化与本地就绪；
  - 不应承担长期 primary 权威发布职责。

## 本案例目标状态应该长什么样

实现完成后，MariaDB async 至少应满足以下边界：

- 任何时刻只有一个权威面负责确认 primary。
- 对外发布 primary 前，旧 primary 是否已退位或被 fencing，有统一判断。
- failover / switchover 的完成信号，不再依赖分散脚本各自返回 success。
- endpoint 切换与 primary 确认具有清晰先后关系。
- 启动期短时抖动不会被误写成新 primary 已稳定。

## 本案例与通用方法文档的边界

下面这些通用内容已经抽到主题文档，不再在本案例里重复展开：

- data plane / signal plane / control plane 的通用分层
- 什么时候需要把选主职责收口到 control-plane
- 通用迁移顺序
- 通用验收口径

## 本案例实现与验证时必须回答的问题

下面这些问题如果不提前答清，后续代码很容易重新长回分散决策。

1. `syncer` 依赖哪些输入信号确认 primary，最小信号集是什么？
2. 什么条件下允许发布新 primary endpoint？
3. 什么条件下必须先做 old primary fencing？
4. failover 与 switchover 是否共用同一套最终发布逻辑？
5. 启动期如果实例短暂自认 primary，control-plane 如何避免过早对外曝光？
6. 如果 `roleProbe` 与复制拓扑短时冲突，谁有优先级？
7. 如果 endpoint 已切但旧 primary 未退位，系统如何阻断继续对外扩散？

## 本案例推荐关注的验收点

这条 MariaDB 线后续做实现验证时，建议至少固定回收以下结果：

1. primary 最终是否由 `syncer` 统一确认
2. endpoint 在何时切换，以及切换前后旧 primary 的状态
3. old primary fencing 是否触发，触发条件是什么
4. failover / switchover 完成信号是否回收到单一控制面
5. 启动期是否还会出现 dual-primary 或角色抖动扩散到对外状态

## 与已停止 guard 线的边界

为了避免后续讨论再混回去，这里把边界再写死一次：

- guard 候选修法和 live 验证线：停止；
- guard 相关证据：保留为背景输入；
- 当前实现主线：`#13`；
- 当前架构方向：MariaDB async primary election 对齐 `syncer/control-plane`。

## 当前文档状态

这是第一版“设计输入 + 已确认实现基线约束”的 MariaDB 案例材料，适合用于：

- 开发侧继续拆任务；
- 项目侧追 `#13` 主线；
- 评审时统一讨论“职责是否已经收口到 control-plane”。

如果后续实现确认了更细的 lease / fencing / endpoint 方案，应在本文件基础上继续补第二版，而不是另开一份互相冲突的说明。
