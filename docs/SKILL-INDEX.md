# KubeBlocks Addon Skills 文档索引

这套文档面向 Addon 开发与测试工程师，采用适合 Claude Code / Cursor 消费的 skills 化组织方式。

组织原则固定如下：

- 通用方法论放在主题文档中，不绑定单一引擎。
- 引擎相关实现、命令、验证结果、排障现场放在案例材料中。
- 一个主题一篇文档，不把 reconfigure / switchover / TLS / backup 这类内容混写。
- 同一主题的后续增量，优先继续迭代原文件，不另开平行文档。

## 按场景找文档

3-月-future-team 接手新 KubeBlocks Addon 时，按当前面对的工作场景进入相应文档。每篇方法论 doc 主要覆盖一个场景；少数跨场景的会在多个分组里出现并打 *(also relevant)* 标。

### 1. 设计 / 开发新 addon

调试 ComponentDefinition / 配置 lifecycle action / 决定 bootstrap 与角色发布逻辑：

- [`addon-bootstrap-role-publish-guide.md`](addon-bootstrap-role-publish-guide.md) — bootstrap 初期先收 role truth、再发 role label / service / endpoint 的链路
- [`addon-control-plane-election-guide.md`](addon-control-plane-election-guide.md) — 选主职责收口到单一 control-plane，避免多脚本并行抢拍板权
- [`addon-reconfigure-guide.md`](addon-reconfigure-guide.md) — 动态 vs 静态参数、`reconfigure.exec` vs legacy `reloadAction`、live-apply 路径排障
- [`addon-switchover-guide.md`](addon-switchover-guide.md) — switchover / failover 幂等性，重入下的 candidate 与角色收敛
- [`addon-tls-guide.md`](addon-tls-guide.md) — TLS 产品语义、明文拒绝正向验证、与 reconfigure / backup / switchover 组合验证
- [`addon-componentdefinition-upgrade-guide.md`](addon-componentdefinition-upgrade-guide.md) — `ComponentDefinition` 升级时识别 immutable spec blocker、何时走新名字 vs 同名覆盖

### 2. 写新 smoke / chaos 测试

设计 helper / runner / 验收口径，把第一次撞 bug 的责任落对层：

- [`addon-test-acceptance-and-first-blocker-guide.md`](addon-test-acceptance-and-first-blocker-guide.md) — 成功语义分层、first blocker 分层、validation-only gate 身份固定、现场冻结
- [`addon-test-probe-classification-guide.md`](addon-test-probe-classification-guide.md) — 探针失败分到 `route_api` / `<client>_<channel>` / `empty_output` / `parse_empty` / `runtime_mismatch` / `real_*_mismatch` 等正确层
- [`addon-bounded-eventual-convergence-guide.md`](addon-bounded-eventual-convergence-guide.md) — 异步收敛系统的状态判定必须 bounded retry，禁止单次 snapshot 当结论 *(also relevant in: 设计 / 开发新 addon — addon 启动 / rejoin / reconfigure 后的判定面)*
- [`addon-evidence-discipline-guide.md`](addon-evidence-discipline-guide.md) — 对自己产出的结论也做 bounded retry：N=1→"average" / 间接旁证→"系统性证伪" / 动机假设→narrative inflation 三类反模式
- [`addon-design-contract-review-during-xp-guide.md`](addon-design-contract-review-during-xp-guide.md) — XP 模式 review 阶段的 "design-contract challenge" checklist：8 类常见设计契约级缺陷（静默 fallback / 非空字段未强制 / 同 commit state 不连续 / sentinel 值传错误 / 条件清理状态枚举不穷尽 / NotFound 短路写入 / terminating vs absent 不区分 / 运算符优先级陷阱），每类含 review 模式 + 修法

### 3. 环境 ready 前 / 环境层撞坑

测试 runner 跑起来之前 / 测试机重启后 / 镜像 / 路由 / CSI / Backup 前置：

- [`addon-test-environment-gate-hygiene-guide.md`](addon-test-environment-gate-hygiene-guide.md) — 环境就绪逐项坐实清单（API 路由 / 控制器身份 / CSI / 镜像分发 / staged anchors / fresh slot / capability），post-restart 禁止复用 pre-restart 事实
- [`addon-k3d-kubeconfig-loopback-fix-guide.md`](addon-k3d-kubeconfig-loopback-fix-guide.md) — k3d 默认 kubeconfig server 写成 `0.0.0.0` 在 macOS 报 EOF；统一改 `127.0.0.1`
- [`addon-k3d-image-import-multiarch-workaround-guide.md`](addon-k3d-image-import-multiarch-workaround-guide.md) — k3d 节点拉 docker.io 超时 + `k3d image import` 在 multi-arch manifest 上的静默 bug；走 host `docker save` + 节点 `ctr import`
- [`addon-k3d-backup-restore-prereqs-guide.md`](addon-k3d-backup-restore-prereqs-guide.md) — k3d 跑 Backup/Restore 的两层前置：装 VolumeSnapshot CRD + 建默认 BackupRepo

### 4. 运行期 / 排障

集群已经跑起来后撞到的问题，分流到对应方法论：

- [`addon-ops-restart-troubleshooting-guide.md`](addon-ops-restart-troubleshooting-guide.md) — Restart / RollingUpdate "卡住"时先分 `queue 入口未放行` vs `执行体内部`，再用"冻结样本 + clean restart"两段式验证
- [`addon-chart-vs-kb-schema-skew-diagnosis-guide.md`](addon-chart-vs-kb-schema-skew-diagnosis-guide.md) — `helm install` 报 `field not declared in schema` 时区分「chart 局部 bug」/「整代代差」/「chart 跟 KB main 但 API 未发布」三种根因
- [`addon-narrow-scope-force-delete-guide.md`](addon-narrow-scope-force-delete-guide.md) — pod 在 image pull 阶段被删引发的五级 finalizer 死锁；force-delete 那一个 stuck pod，不要 patch finalizer
- [`addon-test-host-stress-and-pollution-accumulation-guide.md`](addon-test-host-stress-and-pollution-accumulation-guide.md) — 共享 host k3d 跑全量回归撞 cascade 信号时不要先 docker restart；清污染让控制面自愈是 first-line recovery

### 5. 改造 runner / 工具链

写测试基础设施 / shell helper / 跨平台 portability：

- [`addon-test-runner-portability-guide.md`](addon-test-runner-portability-guide.md) — macOS bash 3.2 + `set -euo pipefail` 下 7 个常见兼容坑（空数组、env-default 时机、`local x=$(cmd)` 等）

### 跨场景方法论（reusable framework）

下面两篇虽然分别出现在场景 2，但本质是 **跨场景方法论**，任何分布式异步系统的判定面或任何外发结论都适用，建议作为团队共同基线：

- [`addon-bounded-eventual-convergence-guide.md`](addon-bounded-eventual-convergence-guide.md) — 外向 bounded retry：对外部异步系统的观察要多次 sample
- [`addon-evidence-discipline-guide.md`](addon-evidence-discipline-guide.md) — 内向 evidence discipline：对自己产出的结论要多 round challenge / 自我 review

## 文档全列表（详细描述）

按文档主题依次列出，含完整 metadata 描述，作为详细 reference：

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
- [`docs/addon-probe-script-fork-and-zombie-guide.md`](addon-probe-script-fork-and-zombie-guide.md) — addon probe / lifecycle 脚本里 fork 后台子进程在 kbagent / business 容器内累积 zombie 的两类机制：**Pattern A（显式 fork：`&` / nohup / setsid）** ~5-14/min 撞 pids.max；**Pattern B（隐式 timeout-kill orphan：pipeline / `$(...)` 子进程在 kbagent SIGKILL 父脚本时 orphan）**~0.04% probes 低速率但同 cell。统一 4D audit checklist（execution context × fork source × frequency × reaper）；valkey check-role.sh fix 验证（前 14/min → 后 0/min）+ mariadb #402 Pattern B 实证 + OceanBase audit pass scope carve-out
- [`docs/addon-k3d-host-precheck-guide.md`](addon-k3d-host-precheck-guide.md) — 跑 smoke / chaos 之前先跑 host-level k3d precheck（API 可达性 + 延迟 + 集群 CPU/MEM 水位 + stuck 检测）。配套工具 `kubeblocks-tests/scripts/k3d-precheck.sh`（zsh，三档输出：表格 / JSON / quiet exit）；含 4 个 tooling / ownership 通用坑（k3d kubeconfig 0.0.0.0 / k3d v5 label 变更 `k3d.cluster` / macOS bash 3.2 限制 / 跨 team 借用 k3d 造成版本偏移）+ runner 集成 pre-hook + 三台测试机（Machine A/B/C）跨主机 ops profile baseline
- [`docs/addon-probe-timeout-and-soft-failure-guide.md`](addon-probe-timeout-and-soft-failure-guide.md) — addon CmpD 内部 livenessProbe / readinessProbe 脚本的"信道层错 vs 产品层错"分层：客户端 rc!=0 一律 transient → exit 0；只有客户端成功返回 + 输出确认 bad state 才 exit 1；合法慢窗口用 pgrep 守门；含 7 条硬规则 + 反模式表 + 自检清单
- [`docs/addon-paramdef-cue-range-validation-guide.md`](addon-paramdef-cue-range-validation-guide.md) — ParametersDefinition cue/tpl 数值参数范围必须用 practical_min/max（实测能启动），不是 doc_hard_min/max（理论可设）；schema 是 reconfigure 流程的守门；含 7 条硬规则 + 反模式表 + 自检清单 + boundary-1 验证流程
- [`docs/addon-test-runner-cadence-discipline-guide.md`](addon-test-runner-cadence-discipline-guide.md) — 长时 test runner 运行期间，cadence 是操作者（而非 runner）的义务：5min 默认是硬约束；沉默 > 间隔 = 状态未知；no-progress ping 有效；cadence 触发器必须与 runner 进程解耦；含 7 条硬规则 + 反模式表 + 自检清单 + Run 5 反面 / Run 6 正面双案例
- [`docs/addon-test-dg-helper-completeness-guide.md`](addon-test-dg-helper-completeness-guide.md) — 多步骤异步操作的 test helper 必须使用 multi-gate：单一状态字符串是 fakeable 的，必须补充 unfakeable observable invariant（成员计数 / 角色标签 / 指标阈值）；gate 串行 AND 不能短路 OR；fix 需 dry-run（无 false-negative）+ fresh install（真正 blocks race window）双验证；含 7 条硬规则 + 反模式表 + 自检清单 + Oracle DG Bug #26 案例
- [`docs/addon-controller-crash-resilience-guide.md`](addon-controller-crash-resilience-guide.md) — 控制器在 Ops 中段被 SIGKILL / OOM 后，OpsRequest 是否能继续推进至正确终态：控制层故障 vs 数据层故障的边界、desired state 在 CR 上的 4 个隐含语义、crash 中段触发 + 4 项终态验证（Ops Succeed / 副作用计数 / 步骤完整 / 状态机连贯）+ 4 个常见误判
- [`docs/addon-design-contract-review-during-xp-guide.md`](addon-design-contract-review-during-xp-guide.md) — XP 模式 review 阶段的 design-contract challenge checklist。8 类常见设计契约级缺陷（静默 fallback / 非空字段未强制 / 同 commit state 不连续 / sentinel 值传错误 / 条件清理状态枚举不穷尽 / NotFound 短路写入 / terminating vs absent 不区分 / 运算符优先级陷阱）每类含 review 模式 + 修法 + 反面（传统 Dev/Test split 漏掉的概率）；适合 onboarding + pre-commit + review checklist 三种用法
- [`docs/addon-ship-readiness-multi-phase-validation-guide.md`](addon-ship-readiness-multi-phase-validation-guide.md) — addon 何时算可以 ship 的三段矩阵（baseline / chaos × N / regression × N），累积 N、Wilson 95% CI、ship 阈值表（数据丢失 0% / 服务不可用 5% / transient 30%）、二段判定（产品 fail = 0 + caveat 全 document）+ 5 个常见误判

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
- [`docs/cases/mariadb/mariadb-fork-audit-pass-negative-case.md`](cases/mariadb/mariadb-fork-audit-pass-negative-case.md) — MariaDB addon 4 处 fork pattern audit pass 的 negative case：fork 都落在"低频 + non-reaper"格不堆积；给后续 reviewer 一份对照案例（不要见 fork 就 block，按 [`addon-probe-script-fork-and-zombie-guide.md`](addon-probe-script-fork-and-zombie-guide.md) 二维表 attribute）。⚠️ 仅对 Pattern A（显式 fork）有效；Pattern B（timeout-kill orphan）实证见 [`mariadb-roleprobe-timeout-kill-orphan-case.md`](cases/mariadb/mariadb-roleprobe-timeout-kill-orphan-case.md)
- [`docs/cases/mariadb/mariadb-roleprobe-timeout-kill-orphan-case.md`](cases/mariadb/mariadb-roleprobe-timeout-kill-orphan-case.md) — `replication-roleprobe.sh` secondary 路径 4 个 `printf | grep -q` pipeline + `roleProbe.timeoutSeconds=1` 偶发被 SHOW SLAVE STATUS 越界 → kbagent SIGKILL 脚本主进程 → grep 子进程 reparent 到 kbagent (PID 1, Go 不 reap) → zombie 累积。task #402 alpha.12 idle 2h 实测 leak rate ~0.04%（Pattern B 主文档章节的实证补充，不是 Pattern A 显式 fork）

### Valkey

- [`docs/cases/valkey/scale-out-targeted-switchover-stale-pod-list-case.md`](cases/valkey/scale-out-targeted-switchover-stale-pod-list-case.md) — scale-out 后 targeted switchover 因 old primary 容器静态 `VALKEY_POD_FQDN_LIST` 漏掉 fresh candidate 导致 `wait_for_new_master` 超时；修复用 `pod_fqdns_with_candidate()` 把 action 入参 candidate FQDN 并入遍历 list

### Oracle

- [`docs/cases/oracle/oracle-chart-vs-kb-schema-skew-multi-stage-case.md`](cases/oracle/oracle-chart-vs-kb-schema-skew-multi-stage-case.md) — Oracle 1.0.0-alpha.0 chart 在已发布 KB 装不上的三段反转：先误判「整代代差」、再误判「chart 字段路径错位」、最后查清是「chart 跟 KB main 上未发布 API（PR #10100 / #10109）」。同仓 `release-1.0` 分支才是答案。属 [`addon-chart-vs-kb-schema-skew-diagnosis-guide.md`](addon-chart-vs-kb-schema-skew-diagnosis-guide.md) 的工程现场补充
- [`docs/cases/oracle/oracle-12c-post-switchover-probe-cascade-kill.md`](cases/oracle/oracle-12c-post-switchover-probe-cascade-kill.md) — reconfigure_deep Run 1→3 闭环：Bug #12 (DBCA 跑期间 liveness 误杀，initialDelay=600 + 90s 重启窗口) + Bug #13 (post-switchover 慢控制面 flap readiness)；3 layer fix（cmpd probe 参数 + liveness.sh 软失败 + checkDBStatus.sh best-effort dgmgrl）；Run 3 全 PASS + RESTARTS=0 实证；属 [`addon-probe-timeout-and-soft-failure-guide.md`](addon-probe-timeout-and-soft-failure-guide.md) 工程现场补充
- [`docs/cases/oracle/oracle-12c-processes-cue-paramdef-range-case.md`](cases/oracle/oracle-12c-processes-cue-paramdef-range-case.md) — reconfigure_deep T22d FAIL：`processes: int & >=6` cue 太宽，10 通过 ValidatePhase → ORA-603/1092 → instance terminated → KB OpsRequest 卡 Running 25min+；fix `>=100` Run 3 验证生效（ValidatePhase reject within 10s）；属 [`addon-paramdef-cue-range-validation-guide.md`](addon-paramdef-cue-range-validation-guide.md) 工程现场补充

### Methodology

- [`docs/cases/methodology/evidence-inflation-in-csi-durability-debate-case.md`](cases/methodology/evidence-inflation-in-csi-durability-debate-case.md) — 04-28 MariaDB #396 CSI durability 讨论中的 3 次 inflation 与 3 次撤回（动机 narrative / N=1→average / 间接旁证→系统性证伪），属 [`addon-evidence-discipline-guide.md`](addon-evidence-discipline-guide.md) 实证补充

## 使用建议

- 如果不知道从哪进入：从「按场景找文档」选当前最贴的场景。
- 已有具体方法论问题：直接进对应主题文档（详细描述在「文档全列表」里）。
- 遇到具体引擎问题时：再进入对应案例材料，但**不要替代主题文档作为决策基线**。
- 如果后续新增 Backup / Restore / Upgrade / Failover 等材料，也按同样方式放进 `docs/cases/<engine>/`，不要直接堆到 `docs/` 根目录。
