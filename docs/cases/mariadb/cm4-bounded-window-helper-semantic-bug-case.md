# 案例：CM4 helper 把 "last sample 时间" 误当 "reconverged 时间"，导致两次跑都精确卡 61s/60s

属于：[`addon-test-acceptance-and-first-blocker-guide.md`](../../addon-test-acceptance-and-first-blocker-guide.md) 的 helper 实现陷阱案例。

## 背景

- 任务：MariaDB addon 全量 vcluster async smoke。
- 涉及 helper：`mariadb/lib/common.sh::assert_async_primary_converges_after_ops_allow_primary_switch_gap`。
- 现象：`#377` / `#378` 两次跑，CM4 (`back_log=200` 静态参数 reconfigure，需要 rolling restart) 都精确报 `reconverged after 61s, deadline=60s`，但 post-checks（`back_log=200`、`Slave_IO/SQL_Running=Yes`）紧接着 PASS。
- N=2 同模式且差值精确为 1s — 强信号是 helper bug，不是产品慢化或 polling jitter。

## 现场摘要

`#377` / `#378` watch ledger 的关键节点：

| 样本 | Ops Succeed (watch) | first 1P1S (post-Succeed) | helper 报 reconverged | last_elapsed |
|---|---|---|---|---|
| #377 | 23:04:24 | +25s (23:04:49) | 61s | ~58s wall |
| #378 | 23:22:52 | +44s (23:23:36) | 61s | 56s wall |

**post_gap reconverged 时间分布 25s vs 49s — 差异显著**，但 helper 总是输出"61s"。这说明 helper 报告的"61"不是真实 reconverged time。

## 根因

调用链拆开：

1. `watch_async_primary_window` 用 `observe_after_ops_seconds=60`，意味着从首次看到 OpsRequest `phase=Succeed` 起再观察 60 秒才停。
2. `assert_async_primary_converges_after_ops_allow_primary_switch_gap` 把 `ops_completion_epoch` 作为 awk 的 `start`，这个 epoch 是 controller 标记的 `OpsRequest.status.completionTimestamp`，**比 watch 第一次采样到 `Succeed` 早 ~3-5s**（控制器先标完成，watch 下一次轮询才采到）。
3. awk 处理：
   - 在 `start_epoch` 那一行，cluster 已经是 1P1S（primary 切换早就完成、pod-0 还活着）→ 设 `first_ok=t (start_epoch)`，`first_ok_elapsed=0`。
   - 紧接着的样本进入 1P0S（pod-0 进入 rolling restart 时 role label 临时清空）→ 触发 `regressed=1`，`regressed_ts=Ops Succeed time`。
   - 进入 `allowed_recovery_gap=1`（`one_primary_no_secondary` + `endpoint_matches_role` + `current_primary != initial_primary`）。
   - 25-49s 后 pod-0 secondary 重新建立 → 1P1S 恢复，`last_ok=1` 在每个后续采样更新 `last_ts/last_elapsed`。
4. END 分支判定：
   - `regressed=1` 进入 regressed 分支
   - 触发 `if (last_elapsed > deadline)` → FAIL
   - 错就错在：**`last_elapsed` 是"最后一个样本的相对时间"，由 watch loop 跑了多久决定，不是 reconverged 时间**。watch_window=60s ⇒ last_elapsed ≈ 60s，再加上 ops_completion_epoch 早 ~5s 的 buffer，每次都精确报 61s。

## 修复（tests-only，addon 代码不动）

`assert_async_primary_converges_after_ops_allow_primary_switch_gap` 增加 post-gap 真实 reconverged 跟踪：

```awk
# 在 converged 分支里
if (allowed_recovery_gap && post_gap_reconverged_ts == "") {
  post_gap_reconverged_ts = $1
  post_gap_reconverged_elapsed = elapsed
  post_gap_reconverged_primary = current_primary
}
```

END 把 allowed-recovery-gap 分支的 fail 检查从 `last_elapsed > deadline` 换成 `post_gap_reconverged_elapsed > deadline`：

```awk
if (allowed_recovery_gap) {
  if (post_gap_reconverged_ts == "") { fail "did not reconverge after allowed gap" }
  if (post_gap_reconverged_elapsed > deadline) { fail "reconverged after Xs, deadline=..." }
  if (gap_primary != post_gap_reconverged_primary) { fail "post-gap primary mismatch" }
  print success
} else {
  # 原 last_elapsed > deadline 检查保留（无 gap 的 publish-only 路径）
}
```

成功消息同时打印 `reconverged=<post_gap_ts>` 和 `last_observed=<last_ts>`，保留 watch 长尾的诊断价值。

## 验证

`#378` frozen `cm4-primary-window-full.log` 下：

- patched 版：`post_gap_elapsed=49s, deadline=60s` → **PASS**
- 反向（`deadline=30`）：仍正确 **FAIL**（`gap_last_elapsed=31s` 走 invalid_regression）

后续 `#379` (Phase 2o) 用 patched helper 跑 fresh full smoke 验证端到端。

## 沉淀（写给所有 bounded-eventual-convergence 类 helper）

通用教训，不绑定 MariaDB：

1. **bounded helper 的 deadline 检查必须用"事件时间"，不能用"采样时间"**。`last_sample_elapsed > deadline` 几乎一定是错的；应该是 `event_first_observed_elapsed > deadline`，event 是"完成 / 收敛 / 入终态"那一下。
2. **watch_window 长度不要等于 deadline**。如果两者相等，最后一个采样必然落在 deadline 边界附近，任何用 last_sample 时间做判定都会出 false-fail。建议 watch_window ≥ deadline + N（N 给采样间隔留余量），或者在 first_ok 后再观察一段稳定窗口就提前 break。
3. **start 时间用什么直接影响判定**。如果用 controller 的 completionTimestamp（早），等价于把"观察起点"前移；如果用 watch 自己第一次看到 Succeed（晚），等价于"观察起点"后移。两者在 5s 量级的差异，对紧 deadline 是致命的。bounded helper 设计时要明确选哪种语义并写在 doc / 函数注释里。
4. **N=2 不动 deadline 直接调宽**。看到两次精确卡同一个数值就要怀疑 helper 实现，不要直接把 deadline 调宽 — 那等于拿 band-aid 盖掉真正的实现 bug，并降低未来对真实慢化的灵敏度。
5. **frozen evidence + dry-run** 是最便宜的 helper 修复验证手段。把现场 watch_file 提取出来，剥离调度依赖直接喂 awk，能在不重跑 smoke 的前提下分钟级验证 patch 正确性。

## 关联

- patch commit：见 `mariadb/lib/common.sh` `git diff` 引入 `post_gap_reconverged_*`
- 任务串：#377 → #378 → #379（Phase 2m → 2n → 2o）
- 主题文档：`addon-test-acceptance-and-first-blocker-guide.md`、`addon-test-probe-classification-guide.md`、`addon-test-environment-gate-hygiene-guide.md`
