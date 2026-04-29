# MariaDB CM1 dynamic reload 证据摘要

这份摘要把 `CM1 / dynamic reload` 已闭环案例里最关键的证据固定到 skills 仓库路径，避免后续跨人复盘时只依赖 `/tmp` 下的临时文件。

## 对应原始材料

- `/tmp/mariadb-min-cm1-recheck-20260421-180830.log`
- `/tmp/mariadb-min-cm1-recheck-evidence-20260421-180830.txt`
- `/tmp/mariadb-min-cm1-recheck-watcher-20260421-180830.tsv`

## 当前已固定的事实

### 1. 问题不是“数据库完全不支持动态生效”

这轮闭环针对的是 Addon 动态 reload 声明是否真实进入 live 对象，而不是重新判断 MariaDB 参数语义本身。

### 2. 首个实现停点在 `reloadAction` 没有进入 live `ParametersDefinition`

当前已固定的根因表述是：

- `reloadAction` 未落进 live `ParametersDefinition`；
- 控制面因此没有拿到完整的 dynamic reload 契约；
- 最终行为退化成 restart 语义。

### 3. 最小隔离验证链路已经通过

当前闭环不是靠整条长主线一起通过，而是靠这条最小链路单独收口：

- `T1 -> T2 -> CM1`

这条链路通过后，`CM1` 可以从“提醒/怀疑态”切到“已闭环案例”。

## 复盘时建议优先查看的字段

如果后续有人需要重新核对这条案例，建议优先对照以下内容：

1. live `ParametersDefinition` 中是否存在预期的 `reloadAction`
2. dynamic reload 契约是否完整进入控制面读取对象
3. 最小链路 `T1 -> T2 -> CM1` 的通过信号是否齐全
4. 当前问题线是否已经与 `CM4`、guard 线、`#13` 长线方案隔离开

## 边界

这份摘要不是新的问题分析文档，只服务于一件事：

- 给 [`docs/cases/mariadb/cm1-dynamic-reload-case.md`](../cm1-dynamic-reload-case.md) 提供稳定证据入口。

后续如果需要补更多现场摘录，应继续加在这份摘要里，而不是重新创建一份平行证据文档。
