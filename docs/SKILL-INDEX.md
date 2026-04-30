# KubeBlocks Addon Skills 文档索引

这套文档面向 Addon 开发与测试工程师，采用适合 Claude Code / Cursor 消费的 skills 化组织方式。

组织原则固定如下：

- 通用方法论放在主题文档中，不绑定单一引擎。
- 引擎相关实现、命令、验证结果、排障现场放在案例材料中。
- 一个主题一篇文档，不把 reconfigure / switchover / TLS / backup 这类内容混写。
- 同一主题的后续增量，优先继续迭代原文件，不另开平行文档。

## 主题文档

这些文档描述可复用的方法论、检查项、验证口径和常见坑：

- [`docs/addon-reconfigure-guide.md`](addon-reconfigure-guide.md)
- [`docs/addon-switchover-guide.md`](addon-switchover-guide.md)
- [`docs/addon-test-acceptance-and-first-blocker-guide.md`](addon-test-acceptance-and-first-blocker-guide.md) — 测试成功语义、bounded eventual convergence、first blocker 分层、validation-only gate 身份固定、现场冻结和测试资产统计口径
- [`docs/addon-test-probe-classification-guide.md`](addon-test-probe-classification-guide.md) — 一次探针失败如何分到正确的层（`route_api` / `<client>_<channel>` / `empty_output` / `parse_empty` / `runtime_mismatch` / `real_*_mismatch`），以及写探针的 7 条硬规则
- [`docs/addon-test-environment-gate-hygiene-guide.md`](addon-test-environment-gate-hygiene-guide.md) — 测试 runner 跑起来之前的环境就绪逐项坐实清单（路由/控制面/CSI/镜像分发/staged anchors/fresh slot/capability），尤其 post-restart 场景禁止复用 pre-restart 事实
- [`docs/addon-bounded-eventual-convergence-guide.md`](addon-bounded-eventual-convergence-guide.md) — 对异步收敛系统的状态判定必须 bounded retry，不能单次 snapshot；适用于测试 helper 与 addon 业务代码两侧。含 5 个 MariaDB 真实反模式 + 通用模板 + 7 条硬规则
- [`docs/addon-tls-guide.md`](addon-tls-guide.md)
- [`docs/addon-control-plane-election-guide.md`](addon-control-plane-election-guide.md)
- [`docs/addon-ops-restart-troubleshooting-guide.md`](addon-ops-restart-troubleshooting-guide.md) — Ops / Restart 排障时，先分 `queue 入口未放行` vs `执行体内部`，并用“冻结失败样本 + clean restart”的两段式验证来闭合 restart 修复
- [`docs/addon-componentdefinition-upgrade-guide.md`](addon-componentdefinition-upgrade-guide.md) — `ComponentDefinition` 升级时如何识别 immutable spec blocker，以及何时走新名字 / 新版本而不是同名覆盖
- [`docs/addon-chart-vs-kb-schema-skew-diagnosis-guide.md`](addon-chart-vs-kb-schema-skew-diagnosis-guide.md) — chart `helm install` 报 `field not declared in schema` 时的判定 framework：先扫同仓 release 分支、再追字段绝对路径、再 cross-chart dry-run；区分「chart 局部 bug」/「整代代差」/「chart 跟 KB main 但 API 未发布」三种根因（reconfigure 接口的具体跨版本演进与写法迁移见 `addon-reconfigure-version-skew-guide.md`，TBD）
- [`docs/addon-bootstrap-role-publish-guide.md`](addon-bootstrap-role-publish-guide.md) — bootstrap 初期如何先收 role truth，再收 role label / service / endpoint 的 publish 链
- [`docs/addon-narrow-scope-force-delete-guide.md`](addon-narrow-scope-force-delete-guide.md) — 「pod 在 image pull 阶段被删」引发的 cluster ← component ← instanceset ← pod ← CmpD 多级 finalizer 死锁；正确的窄域 recovery 是 force-delete 那一个 stuck pod，不要 patch finalizer
- [`docs/addon-evidence-discipline-guide.md`](addon-evidence-discipline-guide.md) — 对自己产出的结论也做 bounded retry：N=1→"average" / 间接旁证→"系统性证伪" / 动机假设→narrative inflation 三类反模式 + 自检清单 + 7 条硬规则
- [`docs/addon-k3d-kubeconfig-loopback-fix-guide.md`](addon-k3d-kubeconfig-loopback-fix-guide.md) — k3d 默认把 kubeconfig server 写成 `https://0.0.0.0:<port>`，macOS / 部分 Linux 报 EOF；统一改 `127.0.0.1`（含一次性、脚本、集群创建时三种修法）
- [`docs/addon-k3d-image-import-multiarch-workaround-guide.md`](addon-k3d-image-import-multiarch-workaround-guide.md) — k3d 节点拉 docker.io 超时（host 拉得动）的 host-side `docker save` + 节点 `ctr import` 注入路径；同时绕开 `k3d image import` 在 multi-arch manifest 上的静默 bug
- [`docs/addon-k3d-backup-restore-prereqs-guide.md`](addon-k3d-backup-restore-prereqs-guide.md) — k3d 上跑 KB Backup/Restore 的两层环境前置：装 VolumeSnapshot CRD（让 dataprotection controller 起来）+ 建默认 BackupRepo（让 Backup CR 不再 NoDefaultBackupRepo），引擎无关
- [`docs/addon-test-runner-portability-guide.md`](addon-test-runner-portability-guide.md) — macOS bash 3.2 + `set -euo pipefail` 下 runner 的 7 个常见兼容坑（空数组、env-default 时机、单条 local 内互引用、`v\$parameter`、`local x=$(cmd)` 等）+ 自检清单
- [`docs/addon-probe-script-fork-and-zombie-guide.md`](addon-probe-script-fork-and-zombie-guide.md) — addon probe / lifecycle 脚本里 fork 后台子进程（`&` / nohup / setsid）会在 kbagent / business 容器内累积 zombie（实测 5-14/min，~5-13h 撞 K8s pids.max=4096）；正确做法是 sync + 远端 INFO timeout 包装；包含 valkey check-role.sh fix 验证（前 14/min → 后 0/min）+ 短寿测试 cluster 看不到此问题的 caveat

## 案例材料

这些文档记录某一引擎、某一问题线或某一次闭环案例，只作为主题文档的案例补充，不替代通用方法论。

### MariaDB

- [`docs/cases/mariadb/async-primary-election-syncer-control-plane-case.md`](cases/mariadb/async-primary-election-syncer-control-plane-case.md)
- [`docs/cases/mariadb/cm1-dynamic-reload-case.md`](cases/mariadb/cm1-dynamic-reload-case.md)
- [`docs/cases/mariadb/evidence/cm1-dynamic-reload-evidence-summary.md`](cases/mariadb/evidence/cm1-dynamic-reload-evidence-summary.md)
- [`docs/cases/mariadb/post-restart-csi-mount-propagation-case.md`](cases/mariadb/post-restart-csi-mount-propagation-case.md) — post-restart 后 k3d 节点 `/var/lib/kubelet` 失去 rshared 传播，csi-hostpathplugin Bidirectional mountPropagation 无法创建容器，全套 CSI Crash → PVC 永远 Pending → T1 timeout（属"测试环境 Gate 卫生"案例）
- [`docs/cases/mariadb/cm4-bounded-window-helper-semantic-bug-case.md`](cases/mariadb/cm4-bounded-window-helper-semantic-bug-case.md) — bounded primary-switch convergence helper 用 `last_elapsed`（最后一个采样的相对时间）误当 reconverged 时间，导致 watch_window=deadline 时永远 false-fail；修复加 `post_gap_reconverged_ts/elapsed` 用真实事件时间做 deadline 检查（属"测试验收 / first blocker 分层"案例）
- [`docs/cases/mariadb/roleprobe-bash-shebang-on-busybox-kbagent-case.md`](cases/mariadb/roleprobe-bash-shebang-on-busybox-kbagent-case.md) — `replication-roleprobe.sh` 用 `#!/bin/bash` shebang，但 kbagent sidecar 镜像只有 busybox `/bin/sh`，直接 exec 触发 OCI ENOENT；alpha.10 把 shebang 改 `#!/bin/sh`（脚本 body 已 POSIX 兼容）
- [`docs/cases/mariadb/rejoin-gate-single-shot-vs-bounded-wait-case.md`](cases/mariadb/rejoin-gate-single-shot-vs-bounded-wait-case.md) — addon bootstrap `finalize_replication_rejoin_ready_gate` 单次 snapshot slave_status 把 pod 卡在 `.replication-pending`；alpha.11 加 30s bounded retry，slave 收敛就 mark_replication_ready
- [`docs/cases/mariadb/mariadb-fork-audit-pass-negative-case.md`](cases/mariadb/mariadb-fork-audit-pass-negative-case.md) — MariaDB addon 4 处 fork pattern audit pass 的 negative case：fork 都落在"低频 + non-reaper"格不堆积；给后续 reviewer 一份对照案例（不要见 fork 就 block，按 [`addon-probe-script-fork-and-zombie-guide.md`](addon-probe-script-fork-and-zombie-guide.md) 二维表 attribute）

### Valkey

- [`docs/cases/valkey/scale-out-targeted-switchover-stale-pod-list-case.md`](cases/valkey/scale-out-targeted-switchover-stale-pod-list-case.md) — scale-out 后 targeted switchover 因 old primary 容器静态 `VALKEY_POD_FQDN_LIST` 漏掉 fresh candidate 导致 `wait_for_new_master` 超时；修复用 `pod_fqdns_with_candidate()` 把 action 入参 candidate FQDN 并入遍历 list

### Oracle

- [`docs/cases/oracle/oracle-chart-vs-kb-schema-skew-multi-stage-case.md`](cases/oracle/oracle-chart-vs-kb-schema-skew-multi-stage-case.md) — Oracle 1.0.0-alpha.0 chart 在已发布 KB 装不上的三段反转：先误判「整代代差」、再误判「chart 字段路径错位」、最后查清是「chart 跟 KB main 上未发布 API（PR #10100 / #10109）」。同仓 `release-1.0` 分支才是答案。属 [`addon-chart-vs-kb-schema-skew-diagnosis-guide.md`](addon-chart-vs-kb-schema-skew-diagnosis-guide.md) 的工程现场补充

### Methodology

- [`docs/cases/methodology/evidence-inflation-in-csi-durability-debate-case.md`](cases/methodology/evidence-inflation-in-csi-durability-debate-case.md) — 04-28 MariaDB #396 CSI durability 讨论中的 3 次 inflation 与 3 次撤回（动机 narrative / N=1→average / 间接旁证→系统性证伪），属 [`addon-evidence-discipline-guide.md`](addon-evidence-discipline-guide.md) 实证补充

## 使用建议

- 先读主题文档，建立通用判断框架。
- 遇到具体引擎问题时，再进入对应案例材料。
- 如果后续新增 Backup / Restore / Upgrade / Failover 等材料，也按同样方式放进 `docs/cases/<engine>/`，不要直接堆到 `docs/` 根目录。
