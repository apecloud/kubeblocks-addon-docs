# MariaDB CM1 dynamic reload 已闭环案例

> **Audience**: addon dev / test 在 MariaDB reconfigure 或 `ParametersDefinition` / `reloadAction` 设计场景
> **Status**: stable (case closed)
> **Applies to**: MariaDB addon — `reloadAction` 落 live `ParametersDefinition` 路径；通用 reconfigure 经验回主题文档 [`addon-reconfigure-guide.md`](../../addon-reconfigure-guide.md)
> **Applies to KB version**: KB 1.0.x verified（ParametersDefinition schema 跨 1.0 / 1.1 稳定；KB main reconfigure API 重构见 [`addon-reconfigure-guide.md`](../../addon-reconfigure-guide.md)）

本文是 MariaDB 案例材料，不是通用 reconfigure 方法论文档。

它记录一个已经闭环的最小案例：`CM1 / dynamic reload`。目标不是复述整条项目历史，而是给 Addon 开发者留下一个可复用的“症状 -> 根因 -> 修复点 -> 最小验证口径”样板。通用 reconfigure 经验应继续沉淀在 [`docs/addon-reconfigure-guide.md`](../../addon-reconfigure-guide.md)。

## 结论先行

`CM1` 这一轮的根因已经收敛为：

- `reloadAction` 没有落进 live `ParametersDefinition`；
- 导致本应按 dynamic reload 生效的参数修改，退化成了 restart 语义。

这条已经完成最小隔离验证，当前可视为已闭环。

## 症状

外部现象不是“参数完全不生效”，而是：

- 预期应该走动态生效路径；
- 实际行为却退化成重启或需要重启才能达成目标；
- 这说明问题不一定在数据库引擎本身，而更可能在 Addon 如何把 dynamic reload 能力声明给控制面。

这类问题的危险点在于，它很容易被误记成：

- “数据库不支持热加载”；
- “测试环境偶发抖动”；
- “重启也能生效，所以先不追”。

但对 Addon 开发来说，这三种判断都不够准确。关键是要先确认 dynamic reload 的能力声明是否真的进入了 live 对象。

## 根因

这轮最关键的实现停点是：

- 源码层面虽然在推进 dynamic reload；
- 但 live `ParametersDefinition` 里没有拿到预期的 `reloadAction`；
- 控制面因此没有得到“参数变更后应该执行 reload”的明确信号；
- 最终动态参数退化成 restart 路径。

所以根因不应表述成笼统的“reload 没执行”，而应更准确地写成：

- `reloadAction` 未落进 live `ParametersDefinition`，导致动态参数能力声明不完整。

## 修复点

这一类问题的修复，不是先去改测试脚本或重新解释数据库行为，而是先修 Addon 声明面：

- 确保 `reloadAction` 出现在正确层级；
- 确保渲染结果和 live `ParametersDefinition` 中都存在该字段；
- 确保控制面最终拿到的是完整的 dynamic reload 描述，而不是半成品。

这轮闭环说明了一条很重要的经验：

- 只改源码 YAML 不够；
- 必须看到 live 对象里的最终字段；
- 否则你修到的可能只是“看起来像支持动态 reload”，不是“系统真的知道怎么做 dynamic reload”。

## 最小验证链路

这条案例最终不是靠大而全的整轮回归收口，而是靠一条最小隔离验证链路收住：

- `T1 -> T2 -> CM1`

这条最小链路的价值在于：

- 先把问题压缩到最小闭环；
- 避免后续 `CM4 static / rolling restart` 之类的新问题把 `CM1` 结论重新搅混；
- 让“这条到底闭没闭环”可以被单独回答。

## 当前已固定的验证结果

当前已经拿到的验证结论是：

- `CM1 / dynamic reload` 已通过；
- 根因已经固定到 `reloadAction` 没落进 live `ParametersDefinition`；
- 最小隔离验证 `T1 -> T2 -> CM1` 已通过。

## 已记录证据

这条闭环案例对应的证据文件已经固定为：

- `/tmp/mariadb-min-cm1-recheck-20260421-180830.log`
- `/tmp/mariadb-min-cm1-recheck-evidence-20260421-180830.txt`
- `/tmp/mariadb-min-cm1-recheck-watcher-20260421-180830.tsv`
- 稳定归档摘要：[`docs/cases/mariadb/evidence/cm1-dynamic-reload-evidence-summary.md`](evidence/cm1-dynamic-reload-evidence-summary.md)

这些材料应足以支撑以下三件事：

- 问题当时的外部现象；
- live 对象与控制面的关键证据；
- 最小回归链路已经通过的结论。

## 这条案例给其他 Addon 的可复用经验

### 1. 不要只看源码声明，要看 live 对象

对动态 reload 这类能力，最终是否生效取决于控制面实际读到了什么，不取决于源码里“本来想表达什么”。

### 2. `dynamicParameters` 与 `reloadAction` 是一组契约

只看到参数被归类为动态，不代表动态 reload 就完整了。还必须确认 reload 动作本身进入了 live 配置。

### 3. 闭环时优先收最小链路

如果一条主线上已经混入多个问题点，先收最小隔离验证，再讨论更后面的场景，效率更高，结论也更稳。

### 4. 不要把“退化成 restart”轻描淡写

对用户来说也许“重启后参数生效了”，但对 Addon 能力定义来说，这仍然是 dynamic reload 没实现到位。

## 与其他主线的边界

这条文档只收 `CM1 / dynamic reload` 已闭环案例，不覆盖：

- 已停止的 MariaDB guard live 验证线；
- `#13` 的 async primary election / syncer-control-plane 长线方案；
- `CM4 static / rolling restart` 后续暴露出的更早停点。

这样做的目的是把“已闭环案例”和“仍在推进中的架构主线”分开，避免后续阅读时把不同问题线混成一条。

## 当前文档状态

这是第一版闭环案例短文档，适合作为：

- 开发者复盘 dynamic reload 声明问题时的索引；
- 后续类似问题的对照样板；
- 项目侧标记“这条已经闭环，不再回到提醒态”的依据。
