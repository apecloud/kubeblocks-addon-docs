# MariaDB fork audit pass — negative case

属于：[`addon-probe-script-fork-and-zombie-guide.md`](../../addon-probe-script-fork-and-zombie-guide.md) audit checklist 实证补充。

> **Audience**：addon dev 写新 probe / action 脚本时；做 cross-engine fork audit 时；review fork pattern PR 时
> **Status**：stable
> **Applies to**：MariaDB addon（其它引擎按同 framework 自查）
> **Applies to KB version**：any（kbagent / kubelet 容器模型跨 KB 版本不变）

## 这是什么

Valkey check-role.sh fork-from-probe zombie 累积事件爆出后，全 addon 仓做了一轮 cross-engine fork pattern audit。MariaDB addon **audit pass，但不是因为没有 fork pattern**（实际有 4 处 `&` 后台 fork），而是因为这 4 处 fork 全部落在 [audit checklist 2x2 表](../../addon-probe-script-fork-and-zombie-guide.md#audit-checklistfork-频率-pid-1-reaper-行为) 的 "**低频 fork + non-reaper**" 格——**有 fork、不堆积、可接受**。

本案例存在的目的是：

1. 给后续 reviewer 一个具体例子，看到 fork 不要直接 block，按二维表 attribute
2. 给 mariadb addon 自己一份长期记录，**禁止后续把这些 fork pattern 改成高频**（否则就跨格成 leak）
3. 作为 evidence-discipline 案例补充：audit 一开始 over-broad（"找 fork = 找 bug"）的反例

## Audit 背景

- **触发**：04-30 18:24 Alice (valkey TL) 通报 valkey check-role.sh 在 kbagent 容器内累积 zombie，~5-13h 撞 K8s `pids.max=4096`，role probe 全 fail。
- **Audit 命令**：

  ```bash
  grep -rEn '\&[[:space:]]*$|nohup|setsid|disown' \
    addons/mariadb/scripts/ addons/mariadb/templates/
  ```

- **结果**：scripts/ 全部 0 命中；templates/ 4 处命中，分布在 3 个文件。

## 4 处 fork pattern attribute

| # | 位置 | fork 内容 | 频率 | reaper PID 1 | 判定 |
|---|---|---|---|---|---|
| 1 | `addons/mariadb/scripts/galera-start.sh:89` | 后台 polling `wsrep_local_state` 直到 SYNCED | 容器寿命内**1 次**（start.sh 启动时 fork，后台 loop 持续 polling 直到容器结束） | `mariadbd`（start.sh 末尾 `exec docker-entrypoint.sh mariadbd`，PID 1 被替换为 mariadbd） | ✅ 低频 + non-reaper：1 个长寿子进程，不堆积 |
| 2 | `addons/mariadb/templates/cmpd-replication.yaml:639` | 后台启动 mariadbd 主进程（标准 entrypoint pattern：先后台启 mariadbd 再做 user/replication setup，最后 `wait $MARIADB_PID`） | 容器寿命内**1 次** | bash 脚本作为 PID 1（reap 子进程） | ✅ 单次 fork，bash PID 1 reap，不堆积 |
| 3 | `addons/mariadb/templates/cmpd-semisync.yaml:114` | 同上（cmpd-semisync 的 entrypoint pattern） | 容器寿命内**1 次** | bash PID 1 | ✅ 单次 fork，bash PID 1 reap |
| 4 | `addons/mariadb/templates/cmpd-semisync.yaml:280` | 后台 monitoring loop 检查 semi-sync TCP 是否 deadlock，发现 deadlock 则 toggle `rpl_semi_sync_master_enabled` | 容器寿命内**1 次**（monitoring loop fork 后持续运行） | bash PID 1 | ✅ 单次 fork，bash PID 1 reap |

**所有 4 处都是"容器寿命内 1 次 fork"**，不是 probe / action 类**周期性触发**的高频 fork。所以即便有 non-reaper 容器（galera-start.sh 的 fork 落在 mariadbd 容器，PID 1 被 exec 替换为 mariadbd 这个不专门 reap 的业务进程），**也只是留 1 个长寿子进程，不会在 5-13h 内堆积撞 `pids.max`**。

## 实测验证

`kubectl exec -c kbagent <mariadb-pod> -- sh -c '...'`（zombie 计数 snippet 详见 [主文档](../../addon-probe-script-fork-and-zombie-guide.md#audit-checklistfork-频率-pid-1-reaper-行为)）：mariadb addon 长 soak / chaos 测试期间 zombie 数始终 = 0。

跟 valkey 对照：

| 项 | Valkey check-role.sh（leak 现场） | MariaDB addon（audit pass） |
|---|---|---|
| fork 容器 | kbagent（Go binary，不 reap） | mariadbd 容器（bash 或 mariadbd PID 1） |
| fork 频率 | probe 周期触发（每 5s） | 容器寿命内 1 次 |
| 落 2x2 格 | 高频 + non-reaper → ❌ leak | 低频 + non-reaper → ✅ acceptable |
| zombie 累积速度 | ~5-14/min | 0 |
| 撞 pids.max 时间 | ~5-13h | 永不（只有 1 个长寿子进程）|

## 长期 commit（写给后续 mariadb 维护者）

如果未来要给 mariadb addon 加新的 probe / lifecycle / Action 脚本，**必须满足以下任一条件**才能引入新的 `&` / nohup / setsid pattern：

1. fork 是容器寿命内单次执行（如 entrypoint 启动 monitoring loop），且 reaper PID 1 是 bash / init / mariadbd 这类业务进程**且**该业务进程显式 reap 子进程
2. fork 在 reaper 容器内（如未来如果有 sidecar 用 tini / dumb-init / pause 当 PID 1）
3. **如果上述都不满足**：**禁止** fork，改用 sync 或远端 INFO 调用替代。参考 valkey check-role.sh fix 的 sync rewrite

特别警告：**不要把现有 4 处低频 fork 改成"每周期触发一次"模式**（例如把 galera-start.sh 的 wsrep polling loop 改成 probe 脚本里每 5s fork 一次）—— 那会立刻跨进 "高频 + non-reaper" leak 格。

## 给 reviewer 的检查表

review mariadb addon 含 fork pattern 的 PR 时，按以下顺序：

1. `grep -rEn '\&[[:space:]]*$|nohup|setsid|disown'` 找新增的 fork
2. 对每条新增 fork，attribute (频率, reaper)：
   - **频率**：是 entrypoint 一次 vs 周期性触发？
   - **reaper**：fork 出去的子进程最终被 reparent 给哪个 PID 1？是 init reaper（tini / dumb-init / pause）还是业务进程？
3. 落 [audit checklist 2x2 表](../../addon-probe-script-fork-and-zombie-guide.md#audit-checklistfork-频率-pid-1-reaper-行为) 哪一格：
   - 任何 reaper 行 → ✅
   - 低频 + non-reaper → ⚠️ acceptable，但本案例附录加 1 行（更新 4 处 fork pattern 表）
   - 高频 + non-reaper → ❌ block，要求改 sync 或换 reaper 容器

## 关联

- 主文档：[`addon-probe-script-fork-and-zombie-guide.md`](../../addon-probe-script-fork-and-zombie-guide.md) — fork-from-probe zombie accumulation 通用方法论 + audit checklist 二维表
- evidence-discipline 反例（待补，等本系列 PR 全 land 后单独 PR）：本次 audit 一开始用 "grep fork = 找 bug" 是 over-broad，正确做法是 fork × reaper 二维 attribute；这是 audit checklist 自身可能 over-fit / under-fit 的反例
