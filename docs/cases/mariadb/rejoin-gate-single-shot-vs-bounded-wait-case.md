# 案例：addon `finalize_replication_rejoin_ready_gate` 单次 snapshot 把 .replication-pending 永久卡死

属于：[`addon-bounded-eventual-convergence-guide.md`](../../addon-bounded-eventual-convergence-guide.md) 引擎案例（addon 内业务代码侧的同一类反模式）。

## 背景

- 任务序列：alpha.10 已修 roleProbe shebang，#384 验证 T7 VScale 通过；#385 跑到 T8 Restart 时 OpsRequest 300s timeout。
- 现象：T8 Restart 后 pod-0（rolling restart 中先重建的那个）的 roleProbe 持续返回 "initializing" + exit 1；KB controller 因 `readyWithRole=false` gate 住整个 OpsRequest。

## 现场摘要（来自 #384 live diagnostic）

- pod-0 DB 健康 secondary：`Slave_IO_Running=Yes, Slave_SQL_Running=Yes, Seconds_Behind_Master=0, read_only=1, sync_binlog=10, back_log=200`。
- pod-0 数据目录：**`.replication-pending` 存在，`.replication-ready` 缺失**，`master.info` 存在。
- pod-0 roleProbe（`#!/bin/sh` shebang 已 alpha.10 修过）手动 exec：返回 `initializing` exit 1（因 `not_ready()` 路径触发，pending 存在）。

也就是说：roleProbe 脚本逻辑没问题（pending 存在 → not_ready）；DB 也正常工作；但**没有任何机制把 `.replication-pending` 翻成 `.replication-ready`**。

## 根因

bootstrap 脚本（cmpd-replication.yaml startup 段）在"我有 master.info / 是 secondary"分支会调用 `finalize_replication_rejoin_ready_gate`：

```bash
finalize_replication_rejoin_ready_gate() {
  ...
  rm -f "${BOOTSTRAP_PRIMARY_GUARD}"
  if slave_status_is_ready_for_rejoin "${slave_status}"; then    # ← 单次 check
    mark_replication_ready
    return 0
  fi
  if slave_status_has_gtid_out_of_order "${slave_status}"; then
    ...
  else
    mark_replication_pending                                     # ← fallback：留 pending
    echo "WARNING: replication rejoin not yet healthy..."
  fi
  return 1
}
```

调用方刚跑过 `START SLAVE`，紧接着读一次 `SLAVE STATUS` 喂给 gate。问题：

- `START SLAVE` 是异步 — IO 线程要花几秒去连 master，SQL 线程更晚一点开始 apply relay log。
- 单次 snapshot 大概率落在 `Slave_IO_Running=Connecting` 或 `Slave_SQL_Running=No` 的中间态。
- `slave_status_is_ready_for_rejoin` 返回 false → 走 `mark_replication_pending` → bootstrap 继续后续步骤后退出。
- 退出之后**没有任何 daemon / 周期任务**会再回头 check 复制是否好了把 pending 翻成 ready。
- mariadbd 进程跑起来后，slave 几秒到几十秒内就完全收敛（IO/SQL=Yes/Yes lag=0），但标志位 `.replication-pending` 永远在那里。
- roleProbe 每 5s 看一次：pending 存在 → `not_ready` → "initializing" + exit 1 → KB controller `readyWithRole=false` → gate Ops。

## 修复（alpha.11）

`finalize_replication_rejoin_ready_gate` 加 bounded eventual convergence loop：

```bash
finalize_replication_rejoin_ready_gate() {
  local slave_status="$1" success_message="$2"
  ...
  rm -f "${BOOTSTRAP_PRIMARY_GUARD}"

  # Bounded eventual convergence: re-query slave_status every 3s for up to 30s.
  # Stop early on detected GTID divergence (no point waiting on that).
  local rejoin_deadline=$((SECONDS + 30))
  while [ $SECONDS -lt $rejoin_deadline ]; do
    if slave_status_is_ready_for_rejoin "${slave_status}"; then
      mark_replication_ready
      echo "${success_message}"
      return 0
    fi
    if slave_status_has_gtid_out_of_order "${slave_status}"; then
      break
    fi
    sleep 3
    slave_status=$(query_slave_status_verbose)
  done

  # Final post-loop check（cover 边界：刚好在最后一次 sleep 后才收敛）
  if slave_status_is_ready_for_rejoin "${slave_status}"; then
    mark_replication_ready
    echo "${success_message}"
    return 0
  fi
  ... # 原有 mark_replication_pending / GTID divergence 逻辑
}
```

`Chart.yaml` alpha.10 → alpha.11；helm template 渲染验证 patch 出现在产物中。

## 验证

`#385` (Phase 2u) 在 alpha.11 下：
- T8 Restart **PASS**：rolling restart 后 pod-0 → secondary，role label 正确发布，OpsRequest Succeed。
- runner 继续往后 PASS T9 / T10 / T11。
- 第一次进入 chaos 阶段才再撞到下一类问题（C1 GTID divergence — 不属本案例）。

## 给后续 addon 工程师的固化要求

1. **任何 addon bootstrap 脚本里"做完一个动作立刻判定后果"的地方，必须 bounded eventual convergence**。这次卡 30s 是个合理默认；具体场景可调。
2. **`mark_replication_pending` 是兜底**，不应作为 happy path 的退出方式。
3. **不要假设异步动作的副作用立即可见**：`START SLAVE` 后 SHOW SLAVE STATUS 立刻返回的字段大概率是中间态。
4. **`fail_closed_for_gtid_divergence` 类的 early break 必须保留**，但发生在 bounded loop 内（不要等满 30s 才检测分歧）。
5. **roleProbe 是 read-only**，不更新数据目录哨兵。所以 bootstrap 退出时哨兵状态必须正确，否则永远卡住。

## 关联

- patch commit：`addons/mariadb/templates/cmpd-replication.yaml::finalize_replication_rejoin_ready_gate` + `Chart.yaml` 1.1.1-alpha.11
- 任务串：#384 (T8 first observed) → #385 (alpha.11 验证 T8 PASS)
- 主题文档：`addon-bounded-eventual-convergence-guide.md`、`addon-bootstrap-role-publish-guide.md`、`addon-test-acceptance-and-first-blocker-guide.md`
