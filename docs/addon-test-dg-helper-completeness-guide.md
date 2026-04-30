# Addon Test Helper Completeness — 多步骤异步操作的 multi-gate 验证，避免中间状态伪造 false positive

> **Audience**: addon dev / test / TL
> **Status**: stable
> **Applies to**: any KB addon
> **Applies to KB version**: any (methodology, version-agnostic)
> **Affected by version skew**: no

属于：方法论主题文档（不绑定单一引擎）。Oracle/DG 具体案例见文末附录。

## 1. 这篇要解决的问题

轮询等待某个操作"成功"的 test helper，其质量上限由它对"成功"的定义决定。当被测操作是**多步骤、异步执行**的，系统往往会在中间某一步完成时就发出一个"看起来像最终成功"的信号——而此时整个操作远未结束。

只检查单一终态信号的 helper 会发生 false positive：在系统仍处于不完整中间状态时就返回成功。下游 assertion 随即在错误的系统状态上执行，产生虚假的测试失败（更糟糕的是：虚假的测试通过）。

具体的失败模式：你写了一个检查 `"SUCCESS"` 字符串的 helper。被测操作实际上由三个顺序的异步步骤组成。步骤 1 单独完成后就产生 `Configuration Status: SUCCESS`——但 standby 成员尚未加入。你的 helper 立即返回成功。下一条 assertion 查询 standby，拿到的是无效数据。

这篇文档解决的问题就是：**如何设计 multi-gate helper，让它在操作真正完整完成前不会提前返回**。

## 2. 为什么中间状态看起来像最终状态

多步骤异步操作的典型结构：

1. 多个子任务独立完成，无原子事务保证
2. 每个子任务完成时立即更新系统状态
3. "终态成功信号"由步骤 1（或某个早期步骤）写出，而不是最后一步

这使得 `STATUS = SUCCESS` 成为**必要条件，但不是充分条件**。

不同引擎中出现同一模式的例子（引擎无关的泛化规律）：

- **Redis Sentinel**：`SENTINEL sentinels mymaster` 返回的 sentinel 数量小于 quorum，但 `INFO` 命令已返回 `role:master`
- **Galera**：`wsrep_cluster_size=1`（bootstrap 阶段，单节点认为自己已形成集群）
- **MySQL 半同步复制**：`information_schema.PLUGINS` 显示插件已加载，但 `rpl_semi_sync_master_status = OFF`（尚无已连接的 slave）
- **etcd**：单节点 quorum 已达成，但 peer endpoints 还未 join

规律相同：**一个早期步骤产生了外观与最终状态相同或相似的信号**。Oracle/DG 的具体时序见文末附录。

## 3. Multi-gate helper 设计

Multi-gate helper 的工作原理：检查**多个独立条件**，所有条件必须同时满足，才宣告成功。

> **gate 必须串行 AND，不能短路 OR。**

每个 gate 必须顺序检查。任何一个 gate 失败，整个 poll cycle 重新开始——不允许跳过失败的 gate 向后推进。"所有 gate 通过"≠"gate 3 通过，即使 gate 2 失败了"。必须在**同一个 poll 迭代内**所有 gate 都通过，才返回成功。

为什么是串行 AND 而非 OR？因为中间状态可以通过早期 gate，而在后续 gate 失败。gate 1（SUCCESS 字符串）单独通过不够；只有全部 gate 同时通过，才能证明系统处于完整的最终状态。

Multi-gate helper 的结构：

```
poll loop:
  sample system state  （单次快照）
  gate 1: 检查条件 A  → 失败 → sleep, continue（重新开始 poll cycle）
  gate 2: 检查条件 B  → 失败 → sleep, continue
  gate 3: 检查条件 C  → 失败 → sleep, continue
  ...
  gate N: 检查条件 N  → 失败 → sleep, continue
  所有 gate 通过 → 返回成功
超时 → 返回失败
```

关键约束：gate 1 失败后，不进行 gate 2-N 的检查，直接 sleep 并重试；gate 2 失败后，不进行 gate 3-N 的检查，直接 sleep 并重试。中间不允许有"部分通过"的中间结果被保留到下一次迭代。

## 4. Gate 设计原则

### §4a — Unfakeable Feature 原则（所有其他原则的根基）

> **一个好的 gate 必须校验一个中间状态无法伪造的状态特征（observable invariant）。**

Observable invariant 是系统的一个属性，满足：

- 只有在产生它的具体步骤真正完成后，才会变为 true
- 任何早期步骤无法作为副作用产生它
- 可以直接观测（不依赖推理）

**什么是 unfakeable（observable invariant）**：

| 特征类型 | 示例 | 为什么 unfakeable |
|---|---|---|
| 成员计数 = 预期值 | `member_count >= expected_replicas` | 单 primary 无法产生 count=2 |
| 特定角色标签存在 | 输出中存在 `Physical standby database` 行 | CREATE CONFIGURATION 阶段 primary 不会产生这行 |
| 数值在阈值内 | `apply lag < 30s` | 刚加入的 standby 正在追赶 redo，lag 会很高 |
| 特定状态标志 | `Fast-Start Failover: ENABLED` | ENABLE FAST_START FAILOVER 命令执行后才出现 |

**什么是 fakeable（不能作为单一 gate）**：

| 信号 | 为什么 fakeable |
|---|---|
| `grep "SUCCESS"` | 中间状态（单 primary CREATE 完成）也返回 SUCCESS |
| exit code 0 | 工具内部 ENABLE 可能失败，但仍 exit 0 |
| "查询返回非空" | 任何时刻都可能非空 |
| `STATUS = OK` 等通用状态字符串 | 任意早期步骤均可写出这些字段 |

**判断方法**：问自己"中间状态（尚未完成 step N）能产生这个特征吗？"。如果能 → 不是 unfakeable feature，不能单独作为 gate。

### §4b — Gate 排序

- 越稳定、越快速可验证的 gate 放前面——快速排除明显未完成状态，减少重试代价
- 越容易被中间状态伪造的特征放后面——让它无法单独"通关"
- 不可伪造的计数 / 角色标签 / 数值类 gate 放最关键位置（通常是 gate 2/3/4）

注意：fakeable gate（如状态字符串检查）可以保留为 gate 1，但它是 necessary 而非 sufficient——后续必须有 unfakeable gate 补位。

### §4c — 向后兼容

新 gate 通过可选参数激活，默认行为不变。已有调用者无需修改。

Pattern：`wait_for_x cluster [timeout] [strict_param_1] [strict_param_2]`

- 无 strict params → gate 1 only（旧行为）
- 有 strict params → 全部 gate 激活

这使得已有测试用例无需感知 helper 的变化，新场景按需启用更严格的校验。

### §4d — 容错 gate

部分 gate 允许在取值失败时降级（skip 而非 fail）。适用于：

- 被测指标在操作早期可能不可查（SQL 层尚未就绪）
- 网络抖动导致单次查询失败不代表系统状态变化

**降级 gate 不能是 unfakeable feature gate**——关键验证不能被降级掉。降级时必须记录 skip 原因（便于事后 log 分析）。

## 5. 验证纪律：dry-run + fresh install 缺一不可

Fix 实现后，必须同时跑两种验证：

**1. Dry-run on existing cluster**（验证无 false-negative）

- 对一个已经处于完整健康状态的集群运行修改后的 helper
- 预期：helper 应该立即通过所有 gate，不应该因为新 gate 把健康集群判成 unhealthy
- 证明：fix 不破坏向后兼容，不会让已完成的集群变红

**2. Fresh install**（验证 fix 真正 blocks race window）

- 从零开始安装集群，在操作进行中触发 helper
- 预期：helper 在 race window（操作未完成时）应该 fail，在操作完成后应该 pass
- **黄金证据形式**：helper 在 race window 内记录 `gate1=fail elapsed=Xs`，然后在下一次 poll 所有 gate 通过

这是"fix 真的等了那 X 秒"的直接证据。只有 dry-run 而没有 fresh install，无法证明 fix 真正阻断了 race window——因为已完成的集群上 race window 根本不会出现。

## 6. 泛化适用性

本文方法论不绑定 Oracle/DG。以下三个特征同时满足的场景，均适用本文的 multi-gate 设计：

1. **操作分多个异步步骤**：每步独立完成，没有原子事务
2. **中间步骤产生局部完成信号**：该信号外观与最终信号相同或相似
3. **helper 依赖 poll + 信号比对**：轮询直到信号出现即返回

这覆盖了绝大多数分布式系统运维操作的验证场景：复制拓扑建立、集群成员变更、配置参数热加载、备份 / 恢复完成判定等。

## 7. 反模式表

| 反模式 | 描述 | 后果 | 正确做法 |
|---|---|---|---|
| 只检查状态字符串 | `grep "SUCCESS"` 即返回 | 中间状态伪造成功，下游 assertion 基于错误状态运行 | 识别 unfakeable feature，增加 member count / role label / metric gate |
| gate 短路 OR | 任意一个 gate 通过即认为成功 | 早期步骤完成的 gate 通过，掩盖了后续步骤未完成 | gate 串行 AND，所有 gate 在同一 poll 迭代内全部通过 |
| 只做 dry-run 验证 | 在已完成的集群上测试 helper fix，没有 fresh install | 无法证明 fix 在 race window 内真正阻断，可能仍有漏洞 | dry-run + fresh install 双验证；缺一不可 |
| 把 observable invariant 降级为 optional gate | 将 unfakeable feature（如 member count）设为可选 / 可 skip | race window 中关键验证被跳过，false-positive 回归 | unfakeable feature gate 不能被降级；只有取值失败的辅助 gate 可以降级 |

## 8. 自检清单（写 test helper 前过一遍）

写一个轮询等待操作完成的 test helper 之前，先自问：

1. **我是否画出了被测操作的完整步骤时序图？** 每步会产生哪些可观测信号？
2. **每个 gate 是否校验了一个 unfakeable feature？** 中间状态能产生这个特征吗？
3. **我的 gate 是串行 AND 吗？** 任何一个失败是否都会重试整个 poll cycle，而不是跳过继续？
4. **新 gate 是否通过可选参数激活？** 已有调用者是否需要修改？
5. **fix 后是否同时跑了 dry-run + fresh install？** 两种验证缺一不可。

## 9. 与其他方法论文档的关系

`addon-bounded-eventual-convergence-guide.md` 解决的是：对外部异步系统做单次 snapshot 判定会踩中间态，修复模板是 bounded retry。

本文解决的是：即使已经做了 bounded retry，**poll 的终止条件本身不够强**——只检查单一信号，而系统在中间状态也能发出该信号。Multi-gate 是对 bounded retry 的补强：不只是"retry 够多次"，还要"每次 retry 检查足够多的维度"。

两篇结合：**对异步收敛状态，既要有足够的等待时间（bounded retry），也要有足够强的终止条件（multi-gate）**。

## 10. 给后续 addon 工程师的固化要求

1. **先画被测操作的步骤时序图**，识别每个步骤会产生哪些可观测信号——这是 gate 设计的前提
2. **每个 gate 必须校验一个 unfakeable feature（observable invariant）**——中间状态无法产生的特征
3. **只校验字符串 match / exit code 0 的 helper 必须审查**是否存在能在中间状态伪造这些信号的场景
4. **gate 串行 AND，不能短路 OR**——所有 gate 必须在同一 poll 迭代内同时通过
5. **新 gate 通过可选参数激活**，不破坏已有调用者
6. **fix 后必须同时跑 dry-run（无 false-negative）+ fresh install（真正 blocks race window）**
7. **黄金证据形式**：fresh install 时 helper 在 race window 内记录 fail，下一 poll 通过——这是"fix 等了那些秒"的直接证据

## 11. 案例附录

### Case：Oracle Data Guard broker setup — Bug #26

#### setup_dg_broker.sh 三会话时序（精确时间戳，来源：chaos-replication-run5-FAIL-20260430.log）

```
2026-04-30T11:31:20Z  Session 1: CREATE CONFIGURATION + ENABLE CONFIGURATION
                       → SHOW CONFIGURATION: Configuration Status: SUCCESS
                         (仅 ORCLCDB_0 Primary，无 standby，FSFO DISABLED)

2026-04-30T11:31:28Z  Session 2: ADD DATABASE ORCLCDB_1   (8 秒后)

2026-04-30T11:31:50Z  Session 3: ENABLE FAST_START FAILOVER  (再 22 秒后)
```

Race window：session 1 结束后 → session 2 开始前（约 8 秒）。

#### Bug #26 触发（Run 5 log，第 74–91 行）

```
# log line 74:
[INFO] Waiting for Data Guard broker SUCCESS on ora-ch-81949 ...
# log line 75-87: old helper (gate 1 only) — sees single-node SUCCESS, returns immediately
[INFO] Data Guard configuration shows SUCCESS
  Members:
  ORCLCDB_0 - Primary database
Fast-Start Failover: DISABLED
Configuration Status:
SUCCESS   (status updated 7 seconds ago)
[PASS] C00: DG configuration SUCCESS (DG broker SUCCESS)
# log line 91: standby not yet a member → SQL returns text instead of count
[FAIL] C00: baseline row exists (DG sync verified) (pod=1)
       (expected='1', got='SELECT COUNT(*) FROM john_chaos_rows WHERE id=1 AND val=''chaos''')
```

（来源：`oracle/evidence/chaos-replication-run5-FAIL-20260430.log`，第 74–91 行）

旧 helper 在 session 1 完成后的 8 秒 race window 内拿到 `SUCCESS` 字符串，立即返回。此时 standby 尚未加入 DG 配置，下一步对 standby pod 的 SQL 查询拿到的是原始 SQL 文本而非行数，导致 assertion 失败。

#### Fix — 5-gate wait_for_dg_healthy（commit ab7cb76）

来源：`kubeblocks-tests/oracle/lib/common.sh`，第 391–504 行

| Gate | 条件 | Observable Invariant? | 说明 |
|---|---|---|---|
| 1 | `Configuration Status: SUCCESS` | ❌ fakeable（单 primary 也返回） | Necessary but not sufficient；保留为快速前置检查 |
| 2 | member count ≥ expected_replicas | ✅ | standby 未加入时 count=1，无法伪造 count=2 |
| 3 | `Physical standby database` 存在 | ✅ | CREATE CONFIGURATION 阶段 primary 不产生这行 |
| 4 | `Fast-Start Failover: ENABLED` | ✅ | session 3 执行前不出现 |
| 5 | apply lag < 30s on each standby | ✅ | 追赶中 lag 会高，刚加入时无法满足 |

Gate 1 是 necessary 但 not sufficient（fakeable）；Gates 2–5 是 unfakeable features。全部串行 AND 通过才返回成功。

#### 黄金证据（Run 6 fresh install，第 74–99 行）

```
# log line 74:
[INFO] Waiting for Data Guard broker SUCCESS on ora-ch-562 (expected_replicas=2, mode=replication) ...
# log line 75: fix blocks the race window — gate 1 fail at 33s
[INFO] [wait_for_dg_healthy] elapsed=33s gate1=fail last_output=Connected to "ORCLCDB_0"
        Configuration - ora-ch-562
          Members:
          ORCLCDB_0 - Primary
# (next poll) all 5 gates pass:
[INFO] Data Guard configuration shows SUCCESS
  Members:
  ORCLCDB_0 - Primary database
    ORCLCDB_1 - (*) Physical standby database
Fast-Start Failover: ENABLED
Configuration Status:
SUCCESS   (status updated 3 seconds ago)
[PASS] C00: DG configuration SUCCESS (DG broker SUCCESS)
[PASS] C00: baseline row exists (DG sync verified) (pod=0)
[PASS] C00: baseline row exists (DG sync verified) (pod=1)
```

（来源：`oracle/evidence/chaos-replication-run6-PASS-20260430.log`，第 74–99 行）

Fix 在 race window 内（elapsed=33s）记录了 `gate1=fail`，然后在下一次 poll 所有 5 个 gate 通过。这是"fix 真的等了那 33 秒"的直接证据。
