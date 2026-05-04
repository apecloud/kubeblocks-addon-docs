# Incident: r2 24h soak writer context-flip artifact (双窗口)

> **Date**: 2026-05-04
> **Affected**: SQL Server line 24h soak r2 writer 进程
> **Severity**: client-side instrumentation artifact（cluster 实际正常，bad_ack=0）
> **Doctrine sediment**: §1.4 cornerstone "voice commitment is not invariant; programmatic enforcement is" 实战自证

## 时间线

| ts (UTC) | ts (+0800) | event | source-of-truth |
|---|---|---|---|
| 12:48Z | 20:48 | Oracle Run 7 chaos 启动（已知 fired_at，详见 run7-context-inheritance-incident.md） | John self-report |
| 12:57Z | 20:57 | Run 7 跑进错误 cluster (k3d-mssql-kb103b5) → John kill chaos + execute `kubectl config use-context k3d-oracle-test` | John DM msg=f05be90d |
| 13:18:08Z | 21:18:08 | r2 writer (Mac k3d-mssql-kb103b5 sqlserver-soak-24h-r2) 第一次报 "no primary pod label found" id=4251 | Jerry observation msg=c708b702 |
| 13:18:08Z–13:53:06Z | 21:18–21:53 | **Window 1 artifact**: 360 条虚假 client_failed (id 4251–4610)，writer `pod_role_rows()` 用 default context = k3d-oracle-test，查 sqlserver-soak-24h-r2 ns 返回空 | Jerry incident file |
| 13:45:xxZ | 21:45 | Jerry 交叉对比 cluster 实际（mssql-2 primary, 4/4 Running, bad_ack=0）vs writer 视角（看不到 primary）→ 定位根因 | Jerry msg=c708b702 |
| 13:53:00Z | 21:53 | Jerry execute `kubectl config use-context k3d-mssql-kb103b5` 切回 default context | Jerry msg=beaba345 |
| 13:53:06Z | 21:53:06 | writer id=4611 第一条 client_rc0 恢复 | Jerry id observation |
| 13:53:46Z | 21:53:46 | health-sampler 60s 周期同步恢复 (Running/Running 3/3, primary=mssqlsoak24h2-mssql-2) | Jerry msg=beaba345 |
| 13:53:06Z–13:57:07Z | 21:53–21:57 | 干净窗口 32 条 client_rc0 (id 4611–4642) | Jerry table |
| 13:57:00Z | 21:57:00 | **Slock msg=9b55d55f time=** John 在 #oracle:d4ca44fb 自报 "kill chaos.sh + 切回 k3d-oracle-test"。命令文本 `kill 31093 31105 2>/dev/null; kubectl config use-context k3d-oracle-test` | John DM msg=26a492af + slock daemon timestamp (不可篡改) |
| 13:57:07Z | 21:57:07 | r2 writer id=4643 第二次报 "no primary pod label found"（writer poll 7s 后命中） | Jerry msg=ba367713 |
| 13:57:07Z–14:01:21Z | 21:57–22:01 | **Window 2 artifact**: 48 条虚假 client_failed (id 4643–4690) | Jerry msg=234246ee |
| 14:01:21Z | 22:01:21 | Jerry 二次切回 default context = k3d-mssql-kb103b5；writer id=4690 client_rc0 | Jerry msg=234246ee |
| 14:02:32Z | 22:02:32 | Jerry 起 context-guardian PID 83771 (10s loop force `use-context k3d-mssql-kb103b5`)。Guardian 0 次翻转日志（14:02:32Z 起干净） | Jerry msg=ff41db8e |
| ~14:03:xxZ | ~22:03 | Oracle 端 John 切 minified kubeconfig (`KUBECONFIG=/tmp/kubeconfig-oracle.yaml`)，完全绕开 ~/.kube/config default context；guardian + minified 双 stop-gap 生效，**永久不再 flip** | James DM msg=478b786d → 642b6f88 |
| 14:08Z | 22:08 | r3+ patch v2 (kube() wrapper + KUBE_STRICT + KUBECONFIG + minified file + trap 清理) review approved，等 Jerry commit | Jerry msg=ff41db8e + Tom approve msg=758b7ccd |

## 数据汇总

| 阶段 | 区间 | id 范围 | 条数 | 性质 |
|------|------|---------|------|------|
| Window 1 artifact | 13:18:08Z–13:53:06Z | 4251–4610 | 360 | 虚假 client_failed |
| 干净 1 | 13:53:06Z–13:57:07Z | 4611–4642 | 32 | client_rc0 真实成功 |
| Window 2 artifact | 13:57:07Z–14:01:21Z | 4643–4690 | 48 | 虚假 client_failed |
| Guardian 起后 | 14:02:32Z–end | — | 0 翻转 | 持续守护 |
| **合计虚假** | — | — | **408** | 不计入真实写失败 |

## 真实集群状态（全程健康）

- mssql-0/1/2 全程 4/4 Running
- mssqlsoak24h2-mssql-2 = primary（一次都没漂）
- bad_ack_readback = 0（writer 视角是 artifact，但 ack/readback 自洽性测得是 0 错误）

## 根因（双因合流）

1. **客户端 invariant 破坏**（external trigger）：multi-line 共享 `~/.kube/config` 上的 default `current-context` 被 Oracle line 切换 → SQL Server writer 读到错的 cluster
2. **客户端 instrumentation 不锁**（self-defect）：writer `pod_role_rows()` 等 67 处 kubectl 调用全部不 pin `--context` / `KUBECONFIG`，silent fall through 到 default → 把 cluster invariant 漂移误判成 cluster 故障

**两条任意一条不存在 incident 都不会发生**——这是经典的双因合流场景，和 `addon-evidence-discipline-guide.md` 规则强一致（不二选一表述）。

## Mitigation chain

**短期（r2 期间）**：
- guardian loop（force default context = k3d-mssql-kb103b5 每 10s）
- Oracle line 切 minified kubeconfig（不再触 default context）
- 双 stop-gap 互补，互不打架

**中期（r3 启动前）**：
- patch v1：kube() wrapper + KUBE_STRICT 加到 lib/common.sh + soak.sh + chaos.sh 启动 export
- patch v2：minified kubeconfig file 模式 + cleanup trap

**长期（r3+ 落地）**：
- 67 处 kubectl→kube() 全替换
- runner 启动前 cross-confirm `grep "kubectl " lib/ tests/` 应剩 0 处
- 三层防御（minified file + KUBE_STRICT + kube() wrapper）任意两条失效时还有第三条

## Doctrine derived

1. **Voice commitment is not invariant; programmatic enforcement is.**（21:57 第二次 flip 是在 17:30 doctrine 成文 + 21:54 当事人自报承诺 1h+ 后发生——voice commitment 失败实证）
2. **共享 mutable client state 必须按 protected invariant 对待**（kube/docker/helm config 全适用）
3. **Stop-gap pattern**: per-shell minified kubeconfig file = `kubectl config view --raw --minify --context=<x> --flatten > /tmp/...; export KUBECONFIG=...` 完全绕开共享 default context
4. **Mid-run 自保**: 已 fork worker 不能重启时，guardian loop 是绷带（前提是其他 line 也已绕开 default context）
5. **三层防御**: 任意一层失效时另两层兜底，不依赖 process discipline

## 误报 vs 真错的甄别口径（引用进 §2 row 4）

> 长跑 24h soak writer 突然连续报 "no primary pod label found" 但 cluster 实际正常 → 不要立刻判定 writer bug 或 cluster 故障。先用 **N+1 explicit context query 复核**：`kubectl --context=<intended> get pods -L kubeblocks.io/role`。如果 explicit 查正常 / writer 查异常 → 100% 是 client-side context 漂移，不是 cluster 故障。

## 案例引用方

- §2 row 4: writer "no primary pod label found" 误归因 → falsification 用 `--context=<intended>` 显式复核
- §1.4 cornerstone "voice commitment is not invariant"
- §1.4 cornerstone "stop-gap: minified kubeconfig + per-shell KUBECONFIG"
- §1.4 cornerstone "mid-run guardian loop"
- §1.4 cornerstone "三层防御"

## 附件

- `incidents/r2-context-flip-client-artifact.md` (sqlserver-tests repo, by Jerry)
- `incidents/kubectl-audit.md` (sqlserver-tests repo, by Jerry, 67 处 audit)
- `incidents/kube-wrapper-patch.diff` (sqlserver-tests repo, by Jerry, r3 prep)

## Author / evidence partners

- 主笔：Tom (SQL Server TL)
- Evidence partner: Jerry (SQL Server test) — writer artifact 数据、67 处 audit、patch 实现
- Evidence partner: John (Oracle test) — 双 fired_at self-report 时间戳、command 取证
- Evidence partner: James (Oracle TL) — minified kubeconfig stop-gap pattern 提出、doctrine 升级 "voice commitment is not invariant"
