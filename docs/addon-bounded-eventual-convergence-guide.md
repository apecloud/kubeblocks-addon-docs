# Addon Bounded Eventual Convergence 指南

> **Audience**: addon dev / test / TL
> **Status**: stable
> **Applies to**: any KB addon
> **Applies to KB version**: any (methodology, version-agnostic)

## 先用白话理解这篇文档

### 你之前可能这么想

写一段 helper 检查分布式状态，思路通常是：
- "我做完了 START SLAVE 这个动作，下一句直接读 slave_status，看 IO/SQL 是否 Yes/Yes 就行了"
- "我做完了 OpsRequest，立刻读 cluster role label 看哪个 pod 是 secondary"
- "我做完了 reconfigure，直接读 `SHOW GLOBAL VARIABLES` 看新值生效没"

逻辑像是过程式编程：动作 → 后果立刻可见 → 判定。

### 这种思路在 K8s + 数据库这类系统里几乎必然踩坑

K8s + 数据库 addon 里**几乎所有状态都是异步收敛的**：
- START SLAVE 是 client 命令；IO 线程要几秒去连 master，SQL 线程更晚开始 apply relay log
- OpsRequest controller 标记完成后，role label 还要等 reconcile 链走完才贴
- Reconfigure 后，runtime 配置可能要等 mariadbd refresh 几秒才生效
- 任何 pod 重建后，新 pod 的 role / endpoint / route 全部要等 K8s 控制面 publish 链一步步走完

也就是说**做完动作的瞬间读出来的字段，大概率是中间态**。中间态 → false negative → 测试 fail。这里的 fail 看起来像产品 bug，实际是 helper 在错误时刻读了状态。

### 正确 reframe

**对异步收敛系统做判定 = bounded retry within reasonable window**。不是单次读，是有限多次读 + 见到收敛态就退出 + 超时才 fail。这条规则在测试 helper 和 addon 业务代码（如 bootstrap / rejoin gate）两侧都适用。

### 什么时候应该 reach 这篇

- **写 helper 判定状态时**：看到 single shot read（一次读、立刻判定）就先停下来确认上游状态是同步还是异步收敛
- **判 first blocker 时**：看到 "complete" 类信号（`OpsRequest Succeed` / `Slave_IO=Yes`）后立刻报 fail，先确认 helper 是不是单次读
- **写 RCA / debug 时**：撞到看似间歇性 fail 时，第一个怀疑就是单次 snapshot 撞中间态
- **写 bootstrap / rejoin / reconfigure 类 addon 业务代码时**：任何"做完一个动作立刻判定后果"的地方都要 bounded retry

### 读完这篇你能做什么

- 识别 "对异步收敛系统做单次 snapshot" 这类反模式（5 个真实反模式表）
- 用通用模板写 bounded retry 函数
- 区分**应该重试 vs 应该立即 bail** 的状态分类（保护真实产品 mismatch 不被 retry 掩盖）
- 7 条硬规则做 self-check

### 为什么独立成篇

- **方法论级别**：不绑定特定 phase（reconfigure / switchover / chaos 都用得上），不绑定特定引擎
- **跨 helper / 业务代码两侧通用**：测试 helper 用同一套，addon bootstrap / rejoin gate 也用同一套
- 与 [`addon-evidence-discipline-guide.md`](addon-evidence-discipline-guide.md) 互补 —— evidence-discipline 是"对自己的结论 bounded retry"（向内），bounded-eventual-convergence 是"对外部状态 bounded retry"（向外），两套同源不同 subject

## 主体

本文面向 Addon 开发者、测试工程师和技术负责人，统一总结一个反复出现的根因模式：**对分布式异步收敛过程做单次 snapshot 判定**，几乎必然踩到中间态导致 false negative。

修复模板永远是同一套：**bounded retry within reasonable window**。

## 适用场景

- 测试 helper 检查复制 / 角色 / 数据 / 端点 / 配置参数等"会异步收敛"的状态
- Addon 启动 / 重启 / rejoin / reconfigure 后判断"是否就绪"
- Probe / Action 在 Pod 重建后第一时间运行
- 任何"一个动作完成后立即检查后果"的 helper

## 反模式（5 个真实例子，全部踩过）

下表来自 MariaDB addon 的实战，全部归同一类 root cause：

| 现象 | 错误判定方式 | 真实状态 | 修复 |
|---|---|---|---|
| CM4 primary-switch convergence helper 报 `reconverged after 61s deadline=60s` | awk 用 `last_elapsed`（最后一个采样的相对时间）作为 reconverged 时间 | 实际 25-49s 就 1P1S 稳定，但 watch 跑满 60s | 跟踪真实 `post_gap_reconverged_ts`，用它做 deadline 检查 |
| `assert_replication_ok` 报 `Slave_SQL_Running=''` | bounded loop 等 IO=Yes，SQL 是 loop 外**一次性**读 | rolling restart 后 IO 先来、SQL 后到，几秒内都会到 Yes/Yes | bounded loop 等 **IO 和 SQL 都 Yes** |
| `assert_data_consistent` 报 `no primary` | `primary_idx` 是单次读 | role label 短暂 flicker（pod 重建期间），几秒后稳定 | bounded retry 给 `primary_idx`/`secondary_idx` 30s |
| SQL 探针报 `classification=route_api` | helper 已自愈了 route，但探针不重试 | route 重建后已可用 | bounded retry on `route_api` 分类（其他分类不重试，避免掩盖真问题） |
| addon `finalize_replication_rejoin_ready_gate` 让 pod 卡 `.replication-pending` | START SLAVE 后**立刻**判定 slave_status，IO 还在 Connecting | 几秒内 Slave_IO/SQL 都会 Yes | bounded loop 30s 内 re-query slave_status，收敛就 mark_replication_ready |

每一个看起来像独立 bug，但全是同一类 — **对异步系统做同步式 snapshot 判定**。

## 通用方法论

### 何时这个模式适用

只要满足以下任意一条，单次 snapshot 都不够：

- 状态由多个独立组件（线程 / Pod / controller / Service）协同收敛
- 上一个动作（Ops、kubectl exec、START SLAVE、CHANGE MASTER）的副作用是异步生效
- 标签 / 端点 / VIP 由 controller reconcile 链发布，不是被动作直接写入
- 传输层（kubectl exec、port-forward、route）有自愈机制

实际上 K8s + 数据库 addon 这类项目里，**几乎所有"判定状态"的地方**都该用 bounded eventual convergence，单次 snapshot 是例外不是默认。

### 通用模板（伪代码）

```bash
check_state_with_bounded_eventual_convergence() {
  local timeout="${1:-30}"
  local sleep_interval="${2:-3}"
  local deadline=$((SECONDS + timeout))

  while [ "$SECONDS" -lt "$deadline" ]; do
    refresh_state                                    # ← 每次循环必须重新取 state
    if state_meets_success_criteria; then
      return 0                                       # ← 收敛就立刻退出，别浪费时间
    fi
    if state_is_definitively_broken; then
      break                                          # ← 已知坏态，提前退出别等 timeout
    fi
    sleep "$sleep_interval"
  done

  # 退出循环后再做一次 final check（cover 边界情况：刚好在最后一个 interval 收敛）
  refresh_state
  if state_meets_success_criteria; then
    return 0
  fi
  return 1
}
```

### 七条硬规则（写 helper / addon 都适用）

1. **永远不要对异步状态做单次 snapshot**。如果你写的是 `value=$(query); assert value == expected`，停下来想清楚 query 的值有没有可能"几秒后才稳定"。
2. **deadline 跟 watch_window 不能等价**。常见 bug 是 watch 跑 N 秒、deadline 也是 N 秒，结果 last sample 时间戳必然 ≈ N，单次比较必踩边界。
3. **观测面 fail 与产品 fail 必须分类**。bounded retry **只对 transient classifications** retry（`route_api`、`empty_output`、role-flicker、Connecting 等）。**不要**对真实 `*_mismatch` retry — 那等于把真 bug 盖掉。
4. **状态来源要每次 refresh**。loop 里如果只用第一次的 snapshot，bounded retry 等于死循环 + sleep。
5. **early break on definitively broken**。例如 GTID divergence、real_data_mismatch 等已确认坏态，立刻退出别等 timeout。
6. **timeout 选 P95 + safety margin**，不是平均值。常见 IO+SQL 收敛 5-15s，给 30s；rolling restart 角色收敛 30-50s，给 60-90s。给得太紧（边界 ±1s）会变成噪音 bug，给得太松（10×P95）会掩盖真慢化。
7. **真实事件时间戳 vs 采样时间戳要分开**。helper 报"reconverged at X"时，X 必须是事件首次发生的时间，不是 watch 文件最后一行的时间。命名混淆是反复出现的源 bug。

## 与其他主题文档的关系

- [`addon-test-acceptance-and-first-blocker-guide.md`](addon-test-acceptance-and-first-blocker-guide.md) 讲"测试要看到什么算成功 / 哪一层算 first blocker"。
- [`addon-test-probe-classification-guide.md`](addon-test-probe-classification-guide.md) 讲"探针失败时如何把信号落到正确的层"。
- [`addon-test-environment-gate-hygiene-guide.md`](addon-test-environment-gate-hygiene-guide.md) 讲"测试 runner 跑起来之前的环境就绪"。
- 本篇讲"如何对异步收敛过程做正确的判定"。

四篇是闭环：环境就绪 → 探针失败正确分层 → bounded eventual convergence 判定 → 测试验收语义。任何一层缺失，都会出现"测试在错的地方报错 / 真错被掩盖"。

## 案例附录

- MariaDB CM4 helper bug：[`docs/cases/mariadb/cm4-bounded-window-helper-semantic-bug-case.md`](cases/mariadb/cm4-bounded-window-helper-semantic-bug-case.md)
- MariaDB roleProbe shebang bug + alpha.10 修复：本指南案例 1（待补独立 case 文件）
- MariaDB rejoin gate bounded-wait fix（alpha.11）：本指南案例 2（待补独立 case 文件）
- MariaDB 6-patch 闭环连锁（CM4 + IO/SQL + role flicker + route_api + shebang + rejoin gate）：本指南正文表格

## 一句话总结

**对一个异步收敛系统的状态判定，必须 bounded retry 直到收敛 / 超时 / 已知坏态，不能单次 snapshot。** 这条规则适用于测试 helper 和 addon 业务代码，是 KubeBlocks addon 在分布式数据库场景下最常见的根因模式。
