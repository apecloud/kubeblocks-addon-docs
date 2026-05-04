# Incident: Oracle Run 7 chaos.sh context inheritance（污染源 incident）

> **Date**: 2026-05-04
> **Affected**: Oracle line chaos.sh Run 7（fired 12:48Z），并诱发下游 SQL Server line r2 24h soak writer artifact（详见 `r2-writer-context-flip-incident.md`）
> **Severity**: cross-line side effect — Oracle helm install 落到错的 cluster (`k3d-mssql-kb103b5`)，10 个 cluster-scoped 残留资源 + 24min cleanup wall-clock
> **Doctrine sediment**: §1.4 cornerstone "context 从来没有被切过，是继承了错误的默认 context" — voice commitment 之外，**前 line 留下的 default 就足以污染后续工作**

## 时间线

| ts (UTC) | ts (+0800) | event | source-of-truth |
|---|---|---|---|
| (前置) | (前置) | SQL Server 工作收尾后 `~/.kube/config` default context = `k3d-mssql-kb103b5`，**没人切，但也没人重置** | Tom's prior session |
| 12:48Z | 20:48 | John 启动 Oracle Run 7 chaos.sh，**chaos.sh 没有 preflight guard** → 直接继承默认 context = `k3d-mssql-kb103b5` | John self-report DM msg=621cf16c |
| 12:48Z–12:57Z | 20:48–20:57 | chaos.sh 在 `k3d-mssql-kb103b5` 上 helm install Oracle addon、create namespace `oracle-test`、create Cluster CR `ora-ch-49381` | John log review |
| 12:57Z | 20:57 | John `kubectl get pods -n oracle-test` 看到 pod 一直 Creating → `kubectl config current-context` 返回 `k3d-mssql-kb103b5` 不是 `k3d-oracle-test` → **意识到跑进错误 cluster** | John #oracle:d4ca44fb 自报 (~20:57+0800) |
| 12:57Z | 20:57 | John 第一阶段 cleanup: `helm uninstall` + `kubectl delete cluster ora-ch-49381` + `kubectl delete namespace oracle-test`（**未做 use-context**，仍在错误 cluster 操作 ns 删除） | John self-report |
| 13:08Z | 21:08 | James 批准 John 清理 ComponentDefinition / OpsDefinition / ParamConfigRenderer 残留；John 执行 `kubectl config use-context k3d-oracle-test` 切回 → **此时 default context 又变了 → 触发下游 r2 writer artifact**（详见 r2 incident 时间线 13:18Z 第一窗口） | James DM + Jerry msg=c708b702 |
| 13:21Z | 21:21 | OpsDefinition finalizer 最终释放，exit=0；ns `oracle-test` 完全消失；cluster-scoped 残留全清 | John self-report |

**Cleanup wall-clock**: 24min（12:57Z → 13:21Z），bottleneck = OpsDefinition finalizer 等 Oracle DB graceful shutdown。

## Cluster-scoped 残留资源（10 个）

进入 `k3d-mssql-kb103b5` cluster 的 Oracle helm install 副作用：

| Kind | Name | 备注 |
|---|---|---|
| ComponentDefinition | `oracle-12c-1.0.3` | |
| ComponentDefinition | `oracle-19c-1.0.3` | |
| ComponentDefinition | `oracle-23ai-1.0.3` | |
| ComponentDefinition | `oracle-observer-12c-1.0.3` | |
| ComponentDefinition | `oracle-observer-19c-1.0.3` | |
| OpsDefinition | `oracle-19c-dg-standby-config` | finalizer stuck → exit=1 一次 → 等 Oracle graceful shutdown 后 exit=0 |
| OpsDefinition | `oracle-19c-dg-config` | finalizer stuck → exit=1 一次 → 等 Oracle graceful shutdown 后 exit=0 |
| ParamConfigRenderer | `oracle-12c-pcr-1.0.3` | |
| ParamConfigRenderer | `oracle-19c-pcr-1.0.3` | |
| ParamConfigRenderer | `oracle-23ai-pcr-1.0.3` | |

**Cleanup pattern**: ns 先 Terminating（13s），但 cluster-scoped CmpD/OpsDef/PCR 不被 ns delete 触发清理（架构上正确——它们是 cluster-wide 资源）。需要单独 helm uninstall 触发 `kubectl delete cmpd/opsdef/pcr -l app.kubernetes.io/instance=...`。OpsDefinition finalizer 设计为等其管理的 Cluster 内 Oracle DB graceful shutdown，shutdown 路径锁住 24min wall-clock。

## 真实归因（不是"切错 context"，是"继承错的 default"）

**Punchline (John 原话)**:
> "context 从来没有被切过，是继承了错误的默认 context。"

这条比经典的 "voice commitment is not invariant" 更深一层——cross-line side effect **不需要**任何 explicit `kubectl config use-context` 动作。**前一个 line 留下的 default 就足以污染后续工作**。

**双因合流**:

1. **共享 client state 污染**（external trigger）：前 line 留下 `~/.kube/config` 的 default context 没重置 = "上次有人 use-context 的结果" 长期持有
2. **Oracle chaos.sh 不锁 context**（self-defect）：chaos.sh 启动时不 verify `current-context == ORACLE_EXPECTED_CONTEXT`，silent fall through 到任何 default

**两条任意一条不存在 incident 都不会发生**——和 r2 incident 双因合流同 family 实证。

## Cascade pattern（Run 7 → r2 writer artifact）

Run 7 cleanup 自身又触发下一次 invariant break：

| ts | event | cascade |
|---|---|---|
| 12:48Z | Run 7 chaos.sh 跑进 `k3d-mssql-kb103b5` | Incident 1 fired |
| 12:57Z | John `helm uninstall + delete cluster + delete ns` (在错的 cluster 操作) | Incident 1 detected, partial mitigated |
| 13:08Z | John `use-context k3d-oracle-test` 切回，做 CmpD/OpsDef/PCR cleanup | **Incident 1 cleanup** action |
| 13:18Z | r2 writer 开始报 "no primary pod label found" | **Incident 2 fired**（writer 用 default context = k3d-oracle-test 查 sqlserver-soak-24h-r2 ns）|
| 13:53Z | r2 writer 第一窗口被识别 + Jerry 切回 `k3d-mssql-kb103b5` | Incident 2 first window mitigated |

**核心 lesson**：cleanup operation 自己又是新的 invariant 写动作。**修一个 incident 时如果不显式锁 context，就是在制造下一个 incident**。Run 7 cleanup 严格按"先切回正确 context 再 cleanup"做了，但**切回这个动作本身**改变了下游所有依赖 default context 的工具（r2 writer）的视角——从而 fired Incident 2。

正解 = cleanup 期间所有 kubectl 都 pin `--context=k3d-oracle-test`，**不动 default**，下游就不会被影响。

## Mitigation chain

**短期（Run 7 期间，已落地）**:
- helm uninstall + ns delete + CmpD/OpsDef/PCR 单独清理（24min wall-clock）

**中期（James commits today）**:
- `1c0b6a9` — `oracle/tests/chaos.sh` 加 11 行 preflight guard：检查 `ORACLE_EXPECTED_CONTEXT`（default `k3d-oracle-test`）vs `kubectl config current-context`；mismatch → exit 1
- `c0a4ffd` — `oracle/tests/chaos.sh` 加 KUBECONFIG strict check（拒绝 `~/.kube/config`）+ isolated minified KUBECONFIG pattern

**长期（doctrine 落地）**:
- §1.4 cornerstone：所有 multi-line 共享 client config 按 protected invariant 对待
- 三层防御：①程序锁（`--context=` / `KUBECONFIG=`）+ ②fingerprint 区分（server URL / TLS SAN）+ ③fail-fast assert（`KUBE_STRICT=1`），任意两层 fail 第三层兜底
- Oracle line 全 ban `kubectl config use-context`，只用 `KUBECONFIG=~/.kube/config-idc2` / `--context=` 显式锁（James msg=85dc5857）

## Doctrine derived

1. **"前 line 留下的 default 就足以污染后续工作"** — 不需要任何 explicit use-context 动作，inheritance 已经是 invariant 破坏的来源
2. **Cleanup 操作自身是新的 invariant 写动作** — 修 incident 不锁 context = 制造下一个 incident
3. **Cluster-scoped 残留 cleanup 比 ns Terminating 慢一档** — finalizer 等 DB engine graceful shutdown，wall-clock 决定于引擎类型（Oracle DG = 24min；MSSQL Always-On = 类似数量级）
4. **Reset-cycle latency 越早 cost 越小**：boot-up self-catch 30s（Valkey）<< mid-soak detect 35min（r2 W1）<< post-voice-commit 1h+（r2 W2）

## 案例引用方

- §1.1 destructive op trigger preflight：chaos.sh 启动 = 大型 destructive op 入口，必须 preflight guard
- §1.3 cluster fingerprint preflight：Oracle helm install 落到 `k3d-mssql-kb103b5` → 残留 10 个 cluster-scoped 资源是 cluster fingerprint 漂移的强信号
- §1.4 cornerstone "default context inheritance ≠ explicit switch"
- §1.4 mitigation pattern：preflight guard + strict KUBECONFIG check + minified kubeconfig（James `1c0b6a9` + `c0a4ffd` 实战 N=2）
- §2 row 3 falsification：`kubectl --context=<intended> get cmpd -A` 显式复核

## 附件

- `oracle/scripts/chaos/incident-log/run7-context-leak-20260504.md`（kubeblocks-tests repo, by John，本 appendix 的一手 source-of-truth）
- `oracle/tests/chaos.sh` commit `1c0b6a9` (kubeblocks-tests repo, by James, preflight guard 11 行)
- `oracle/tests/chaos.sh` commit `c0a4ffd` (kubeblocks-tests repo, by James, KUBECONFIG strict + minified)

## Author / evidence partners

- 主笔：Tom (SQL Server TL)
- Evidence partner: John (Oracle test) — Run 7 first-detect、5 数据点完整 timeline、cluster-scoped 残留 inventory、cleanup wall-clock
- Evidence partner: James (Oracle TL) — chaos.sh preflight 实施 (`1c0b6a9`)、KUBECONFIG strict + minified 实施 (`c0a4ffd`)、doctrine 升级 cross-line discussion
