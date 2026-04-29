# Addon ParametersDefinition cue/tpl 数值范围设法指南

本文面向 Addon 开发工程师，重点解决 ParametersDefinition（cue / tpl 等 schema）里数值参数的范围（lower bound / upper bound）应该按什么标准来设。错误设法的代价是：**用户可以提交一个通过 schema 但实际无法启动 DB 的值，引发 OpsRequest 卡死、cluster 卡 Updating 死循环、需要手工 fresh restart 的 incident**。

## 适用场景

- 新 addon 写第一份 ParametersDefinition（cue / tpl 或等价 schema）
- 老 addon 收 reconfigure case fail / cluster 卡 Running with reconfigure 类 OpsRequest 卡 Running
- 任何带数值类型（`int` / `float`）+ range constraint（`>=N` / `<=M`）的参数
- 跨版本升级 addon 时（不同引擎大版本对参数的 hard min/max 可能变化）

## 核心原则

> 参数 schema 的范围必须是 **practical_min / practical_max**，不是 **doc_hard_min / doc_hard_max**。
>
> 文档说「最小可设 6」≠「最小能启动 6」。Schema 的责任是把"会让 DB 起不来"的值挡在 ValidatePhase，而不是放进 spfile / config 让 reconfigure 卡死。

具体做法：

1. **practical_min / practical_max 必须经过启动验证**：在测试环境用边界值起一遍 DB，能正常进 OPEN 才算定值
2. **schema 越严越安全**：宁可拒绝一个理论合法但实际不启动的值，也不要放进去等用户挨打
3. **同时定 lower 和 upper bound**：很多参数只设了 `>=N` 没设 `<=M`，会被资源耗尽类的极端值打挂
4. **不能直接抄官方文档的最小值**：官方文档的 hard min 是"理论可设"，不是"实际可启动"——典型差距 1~2 个数量级

## 为什么 hard_min 不能直接用

数据库系统的参数文档通常分两类：

| 类别 | 含义 | 例子 |
|---|---|---|
| **hard min/max** | 引擎源代码 / spfile parser 接受的最小/最大值；超出会直接 ORA-XXX / EINVAL 拒绝写入 | Oracle `processes` 文档 hard min = 6 |
| **practical min/max** | 引擎能在这个值下完整启动 + 提供基础服务的实际最小/最大值；hard min 通过但 practical min 没过 = DB 起不来 | Oracle `processes` 实际启动需要 ~100（DBWR/SMON/PMON/LGWR/CKPT/ARCn + 至少几个 session） |

两者的差距是**引擎内部的"必备资源"**：

- 后台进程数（必须存在的 SMON / PMON / DBWR / 日志写 / 归档 等）
- 协议层最小 buffer / cache / pool（log_buffer / shared_pool 启动时分配的最小块）
- 系统会话（DBA 自查、统计、健康检查）
- 系统对象（system tablespace 元数据、SYSAUX、系统索引）

如果 schema 用 hard_min，用户提交一个 hard_min ≤ x < practical_min 的值后：

1. cue / paramdef ValidatePhase **通过**（满足 `>=hard_min`）
2. KB 写入 spfile / 配置文件
3. DB 重启 → 启动失败 → instance terminated
4. KB OpsRequest **永远等不到 Succeed**（DB 不重新进 OPEN，状态机永远不推进）
5. cluster 卡 Updating 死循环
6. 用户唯一可行恢复路径是 fresh restart（手工 sqlplus / kubectl exec rescue 极脆弱）

**这是一类自伤事故**：用户按文档合法地用了一个值，被你的 addon 害死。schema 没尽到守门责任。

## 设计规约（写 ParametersDefinition 时的 7 条硬规则）

### 1. 每一个 `int & >=N` 都要回答「N 是 hard 还是 practical」

review 自己的 cue / tpl，对每一处：

```cue
processes: int & >=6   // ← 这里的 6 是从哪来的？
```

如果是从官方文档抄过来的「最小值」，**必错**。改成实测过的 practical_min。

### 2. practical_min 必须有 reproducer 测试

定下 practical_min 之前，跑一遍：

```bash
# 用边界值创建 cluster
helm install ... --set spec.componentSpecs[0].configs.parameters.<param>=<practical_min>
# 或直接 kubectl apply 一个 cluster yaml 把值写死
kubectl wait --for=condition=Ready cluster/<name> --timeout=10m
# 这个 wait 不超时 = practical_min 通过
```

如果用 practical_min - 1 启动会失败，practical_min 启动成功，那就是这个值。**不要怕实测耗时**——一次 fresh cluster 启动 ~10min 远比一次生产事故便宜。

### 3. 同时定 upper bound

很多 cue 只写 lower：

```cue
processes: int & >=100   // 没定 upper bound
```

用户可以提交 `processes=10000000`，oom 杀引擎、kbagent / kubelet 资源耗尽。upper bound 该是：

- **OS / 容器内核硬限制**（fd / pid / mmap 限制）按容器规格反推
- **引擎本身的设计 max**（参数文档 hard max）
- 取较小者

### 4. 范围要按"测试矩阵的实际覆盖"反推，不是按"语言上限"

`uint32 max = 4294967295` 不是合理上限。一个 `processes` 设 4 亿就是用户输入错误，schema 要直接拒。可以用一个明显 conservative 的上限（比如 `processes <= 10000`）——配合 KB 后续支持 `description` / `validation_message` 字段时给出文档解释。

### 5. 跨版本升级要重新验证 practical 边界

不同引擎大版本：

- Oracle 12c → 23ai：后台进程数变化、最小 SGA 变化
- MySQL 5.7 → 8.0：performance_schema 默认开关变化
- PostgreSQL N → N+1：parallel worker 默认变化

**不要假设「老版本能跑就新版本也能」**。每一次大版本升级都要重做 practical_min / max 的实测。两个版本如果 schema 共用同一份 cue，要 split。

### 6. 不要把"业务推荐值"和"启动最小值"混为一谈

经常在 PR review 看到 `processes: int & >=200`，作者解释「200 是推荐值」。这不对——schema 是阻止"起不来"，不是推荐"最佳实践"。

| 字段 | 角色 |
|---|---|
| `processes >= practical_min (100)` | schema 守门，拒绝起不来的值 |
| `processes default = 200` | KB 默认值，给最佳实践 |
| `processes recommended = 200` (description) | 文档/UI 提示 |

三者职责不同，不要叠在 lower bound 上。

### 7. 验证 fix 是否真的生效，唯一可信信号是「用 boundary-1 测试值 → ValidatePhase Failed」

修了 cue lower bound 之后：

```bash
# 用 practical_min - 1 提交 reconfigure
kubectl apply -f - <<EOF
apiVersion: operations.kubeblocks.io/v1alpha1
kind: OpsRequest
spec:
  reconfigure:
    componentName: ...
    parameters: [{ key: <param>, value: "<practical_min - 1>" }]
EOF

# 期望：30s 内 OpsRequest.status.phase = Failed
# 期望：cluster.status.phase 保持 Running
# 期望：DB 内查询参数返回原值（未改）
```

三条任一不满足 → fix 没真生效。

## 反模式（必须避免）

| 反模式 | 后果 |
|---|---|
| 直接抄官方文档「parameter minimum」 | hard_min 通过 practical_min 失败的值漏过 schema → reconfigure 卡死 |
| 只设 lower，不设 upper | 极端大值打死 OS/容器 |
| 只在 cue/tpl 文件改，不重测 boundary | fix 是"看起来对"，实际未验证 |
| 把"业务推荐值"作为 schema lower bound | 用户合法的低规格场景被拒，UX 倒退 |
| 同时支持 12c/23c 用同一份 cue 共用 lower | 一个版本能起，另一个起不来 |
| 测试只跑「正确值」case | negative boundary case 漏掉 = schema 形同虚设 |
| 跨大版本 chart 升级不重做 boundary 验证 | regression 累积，等一次生产事故才暴露 |

## 与其它指南的关系

- `addon-negative-test-contract-verification-guide.md`（TBD，单独 PR 跟进）讲 negative test 在 contract 不存在时怎么处理。本篇讲**让 contract 真的存在**：schema 要把 practical 边界守住，不让无效值流到 reconfigure 流程
- [`addon-reconfigure-guide.md`](./addon-reconfigure-guide.md) 讲 reconfigure 流程整体。本篇是它的"输入侧守门"补充：schema 是 reconfigure 流程能否正确进入 ValidatePhase 失败 fast-path 的关键
- [`addon-bounded-eventual-convergence-guide.md`](./addon-bounded-eventual-convergence-guide.md) 讲收敛系统的判定。本篇间接相关：schema 没守住 → reconfigure 永远不收敛 → 测试 helper 的 bounded retry 也救不回来（boundless wait）

## 自检清单（PR review 与 chart upgrade 前必过）

- [ ] cue / tpl 里每一个 `int & >=N` / `int & <=M` 都标注了 N/M 是 hard 还是 practical
- [ ] hard 的全部改成 practical，并附实测的 cluster wait-for-ready 证据链接
- [ ] 每一个 lower bound 都配套 upper bound（同时设）
- [ ] 跨版本（12c/23c、5.7/8.0 等）的 cue 是 split 的，没有共用 lower bound 假设
- [ ] negative test 集合里有 `boundary - 1` case，验证 schema 在 ValidatePhase 拒
- [ ] reconfigure 流程里手工跑过 boundary - 1 验证：`OpsRequest.phase = Failed within 30s` ✓ + `cluster.phase = Running` 保持 ✓ + `DB query 参数返回原值` ✓

## 案例附录

具体引擎案例放在 `cases/` 目录：

- [`cases/oracle/oracle-12c-processes-cue-paramdef-range-case.md`](./cases/oracle/oracle-12c-processes-cue-paramdef-range-case.md) — Oracle 12c `processes: int & >=6` 太宽松（hard_min=6，practical_min=100）；用户提交 processes=10 → cue 通过 → spfile 写入 → ORA-603 / ORA-1092 → instance terminated → KB OpsRequest 卡 Running 25min+；fix `int & >=100` Run 3 验证生效（T22d ValidatePhase reject within 10s）
