# 案例：Valkey scale-out 后 targeted switchover 因 stale POD_FQDN_LIST 误判

主题文档：[`docs/addon-switchover-guide.md`](../../addon-switchover-guide.md)，原则 7 "action 脚本里遍历的 member list 不能只信容器静态 env"。本文只记录 Valkey addon 的具体表现、修复实现、shellspec 覆盖范围，引擎无关结论请回主题文档。

## 现场信号

- 集群初始 `replicas=3`（`valkey-0/1/2`），随后 `replicas=4` scale-out 出 `valkey-3`，sentinel 三路均 `known-slave` 收齐 `valkey-3`。
- targeted switchover 指定 `KB_SWITCHOVER_CANDIDATE_FQDN=valkey-3.<headless>...`。
- post 状态：sentinel 视图 + 实际拓扑均显示 `valkey-3` 已成为 master，replica-priority 已恢复，无 split-brain。
- OpsRequest 仍 `Failed`，原因是 addon 自检最后落到 `WARNING: could not confirm new primary within 300s`。

## 直接根因

`addons/valkey/scripts/switchover.sh` 中以下遍历点都基于容器启动时 `componentVarRef.podFQDNs` 注入的静态 `VALKEY_POD_FQDN_LIST`：

- `set_priorities_with_candidate_bias()` — 给所有成员设置 `replica-priority`（candidate 设 1，其余设 100）。
- `restore_priorities()` — 还原 `replica-priority`。
- `wait_for_new_master()` — 在 `300s` 窗口内轮询 `INFO replication` 找 `role:master`。
- 若干 `check_replication_status()` 类辅助点。

old primary（`valkey-0`）启动时 `replicas=3`，env 中的 list 只有 `valkey-0/1/2`。scale-out 之后 old primary 容器 env **不会**自动刷新；switchover action 在 old primary 内执行时遍历的仍是旧 3 项 list，根本不会去 `valkey-3` 上 probe。即使 sentinel 已经把 `valkey-3` 选成 new master，`wait_for_new_master()` 在 3 个旧成员上各看了 300s `role:slave`，最终超时报失败。

## 修复实现

引入辅助函数 `pod_fqdns_with_candidate()`，把 action 入参里的 `KB_SWITCHOVER_CANDIDATE_FQDN`（参数变量名 `expected_fqdn` / `candidate_fqdn`）union 进来：

```bash
pod_fqdns_with_candidate() {
  local candidate_fqdn="${1}"
  local result="${VALKEY_POD_FQDN_LIST:-}"
  if is_empty "${candidate_fqdn}"; then
    echo "${result}"
    return 0
  fi
  local candidate_pod="${candidate_fqdn%%.*}"
  IFS=',' read -ra pod_fqdns <<< "${result}"
  for fqdn in "${pod_fqdns[@]}"; do
    [ "${fqdn%%.*}" = "${candidate_pod}" ] && echo "${result}" && return 0
  done
  if is_empty "${result}"; then
    echo "${candidate_fqdn}"
  else
    echo "${result},${candidate_fqdn}"
  fi
}
```

并把所有遍历点替换为：

```bash
IFS=',' read -ra pod_fqdns <<< "$(pod_fqdns_with_candidate "${expected_fqdn}")"
```

实际 patch 覆盖了 `wait_for_new_master`、priority bias / restore、`check_*` 辅助等所有依赖 `VALKEY_POD_FQDN_LIST` 的位置。

## shellspec 覆盖

新增用例需要至少覆盖：

- `VALKEY_POD_FQDN_LIST` 缺 `expected_fqdn` 时，`wait_for_new_master(expected_fqdn=...)` 能命中 candidate 并返回 success；
- `VALKEY_POD_FQDN_LIST` 已经包含 `expected_fqdn` 时，行为不应该重复或漂移；
- `expected_fqdn` 为空（无候选切换）时，仍按原 list 遍历，不引入空字符串成员；
- priority bias / restore 在含有 fresh candidate 的并集 list 上能正确赋值与还原。

`switchover.sh` 对应 spec 通过 `55 examples, 0 failures`，`bash -n` / `helm lint` 同步通过。

## 适用扩展

同主题在 Redis / Sentinel addon 复读源码后形态完全相同（`REDIS_POD_FQDN_LIST`、`SENTINEL_POD_FQDN_LIST`、`redis-switchover.sh` 中 `set_redis_priorities` / `recover_redis_priorities` / `check_redis_kernel_status` / `check_switchover_result`）。同一 fix 模式应叠加到 Redis addon，PR 时建议成对提交。

## 排障 checklist

排查 "switchover Ops Failed but topology converged" 时建议固定取证：

- old primary 容器内 `env | grep _POD_FQDN_LIST` 实际值（注意是 action 进程而不是 shell 当前 env）。
- 当前 cluster 的 `replicas` 与实际 pod 数量。
- controller 调用 action 时的入参（含 `KB_SWITCHOVER_CANDIDATE_FQDN`）。
- action 脚本遍历点的入参快照（最简单是在 entrypoint 加一行 `echo "[debug] iterate list = $(pod_fqdns_with_candidate ...)"` 类的临时埋点）。
- sentinel 的 `SENTINEL master <name>` 输出与 known-slave 列表，证明真实 master 已切到 candidate。
- post-settle 拓扑 `INFO replication` 是否单 master、priority 是否还原。

如果 5 项都对得上、唯独 `wait_for_new_master` 这条超时，几乎一定是 stale env list。
