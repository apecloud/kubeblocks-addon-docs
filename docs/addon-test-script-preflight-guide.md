# Addon 测试脚本 Preflight 指南 — 把共享 client 状态当 protected invariant

> **Audience**: addon dev / test / TL，特别是在多 line 共享同一台 Mac / IDC bastion / shared vcluster 上跑 long-running 测试的人
> **Status**: draft v0.3 (2026-05-05)
> **Applies to**: 任何 KB addon 的测试 runner / soak / chaos / smoketest 工具链
> **Applies to KB version**: any（client-side discipline，与 KB / addon 版本解耦）
> **Affected by version skew**: 不受 KB 版本影响
> **Sibling doc**: `addon-test-environment-gate-hygiene-guide.md`（单 line / 单环境视角）— 本文是它的**跨 line / 多 tenant / shared mutable state** 补集

属于：方法论主题文档（不绑定单一引擎）。

## 0. 这篇要解决的问题（trigger-event-driven invariant verification framing）

> **One-line for non-tech readers**: 多个 addon line 共用一台机器跑测试时，机器上有些"全局开关"会被任一 line 改了影响所有 line（kube context、docker login、HTTP proxy 都是这种开关）。这篇讲怎么不让别人切错开关时把你的测试搞挂。

我们已经在 `addon-test-environment-gate-hygiene-guide.md` 里强调过"测试 runner 跑起来之前要把环境就绪当证据收一遍"——那是**单 line / 单环境**的 post-restart 视角。

这篇是它的**跨 line 版本**：当**多个 addon line 共享一台机器**（Mac local k3d / IDC bastion / shared Mac+vcluster）时，所有线都会读写一些**共享 mutable client-side state**：

- `~/.kube/config` 的 `current-context`
- `~/.docker/config.json` 的 `auths`
- `~/.helm/repositories.yaml` 的 repo cache
- shell 的 `KUBECONFIG` / `HTTP_PROXY` / `HTTPS_PROXY` env
- macOS 系统级代理设置
- `/tmp/kubeconfig-*.yaml` 等 stale kubeconfig 残留文件

**核心 doctrine（在共享 mutable state 系统下成立，单机隔离环境不必这么严）**：所有 multi-line 共享的 client config（kube/docker/helm/proxy）都按 **protected invariant** 对待：
- **写入路径必须显式 audit**（任何 `kubectl config use-context` / `helm repo add` / `docker login` 都构成 cross-line side effect，不是私有动作）
- **读取路径必须显式锁**（`--context` / `KUBECONFIG=` / `--kubeconfig=` / `unset HTTPS_PROXY`）
- **不锁就视为程序 bug，不是 process discipline 的失误**

这条 doctrine 的核心论点是：**共享 mutable state 必须程序锁，不能靠 process discipline**。如果只发"大家不要切 context" / "大家不要登错 docker registry" 这种通告，长跑里 100% 会被破坏，因为人会忘、脚本会写死、新 agent 不知道规约。**只有显式锁是稳的**。

<a name="doctrine-voice-commitment"></a>
**进一步推论（已有实战 N=2 双 incident 实战验证）**：

> **Voice commitment is not invariant; programmatic enforcement is.**

口头/文字承诺（"我以后不切 context 了"）不是 invariant，它最多是 process discipline——长跑里被破坏的概率近 100%。Invariant 必须是程序级别的强制：要么共享 state 被绕开（minified kubeconfig per shell），要么读取路径被显式锁（`--context=` / `KUBECONFIG=`），要么探测层有 fail-fast assert（`KUBE_STRICT=1`）。**任意承诺 → 实战仍违反 → 这恰好证明 doctrine 必要性**（见 §2 row 4 实证：21:57 +0800 在 doctrine 已成文 + 当事人 voice commitment 已下达 1h+ 后 仍发生第二次 context flip）。

### 三层防御（lock wording）

任意一层独立失效时另两层兜底，不依赖 process discipline：

1. **①程序锁** — 所有读取 client state 的命令必须 pin `--context=<intended>` / `KUBECONFIG=<file>` / `--kubeconfig=`。silent fall through to default = bug
2. **②fingerprint 区分** — 在 destructive op 入口处验证 cluster fingerprint（API server URL / TLS SAN / addon set / node 集合）与 intended target 一致；mismatch → fail
3. **③fail-fast assert** — 关键脚本入口加 `KUBE_STRICT=1` 模式，未设 `KUBE_CONTEXT` / current-context 与 intended 不符 → exit 99

任意两层 fail，第三层兜底。三层互相不替代。

### Trigger-event-driven，不是 always-check

老式的 preflight 是"每次跑测试前 check 一遍 X / Y / Z"——但工程实践里这变成 ritualized noise，跑久了大家就跳过。

正确的姿势是 **trigger-event-driven invariant verification**：把 preflight 绑到**触发事件**上，不是绑到时间窗口上：

| 触发事件 | 必须验证的 invariant |
|---|---|
| 进入 destructive op（delete / cleanup / use-context / login / mount） | §1.1 context preflight |
| 切换工作 namespace / 多 ns 操作 | §1.2 namespace preflight |
| 多 cluster 共存 / cross-cluster query | §1.3 cluster fingerprint preflight |
| 网络拓扑切换（local k3d ↔ remote IDC ↔ corp VPN ↔ vcluster） | §1.4 client-network preflight |

Invariant 不变 → 不需要重复验证。Invariant 可能因为某个事件而变 → 在事件**之后**立刻验证一次。

## 0.5 跨 line 实证证据池（N=3 incident + MySQL adoption/fingerprint evidence）

本文 doctrine 不是单 line 经验外推，而是 **2026-05-04 当天 3 条 line 各自独立踩到同 family incident + MySQL line 提供 adoption + fingerprint 维度证据**之后归纳成的：

| Line | 性质 | 锚点 | 同 family 体现 | mitigation 模式 |
|---|---|---|---|---|
| **Valkey** | incident | Valkey boot-up self-catch incident，shared `~/.kube/config` 残留 stale context 导致 boot-up self-catch | 共享 client state 残留污染 → addon helm install 落到错 cluster | ①程序锁（每脚本 `--context=`）+ ②fingerprint（boot-up server URL 探测） |
| **SQL Server** | incident | r2 writer context-flip artifact 双窗口 408 条 client_artifact entries（[`cases/methodology/r2-writer-context-flip-2026-05-04-case.md`](cases/methodology/r2-writer-context-flip-2026-05-04-case.md)） | 67 处 kubectl 不 pin context → default context 被 Oracle line 切走 → writer 读到空 ns | ①kube() wrapper + ②minified kubeconfig file + ③`KUBE_STRICT=1` 三层防御 |
| **Oracle** | incident | Run 7 chaos.sh 跑进 `k3d-mssql-kb103b5`（[`cases/methodology/run7-context-inheritance-2026-05-04-case.md`](cases/methodology/run7-context-inheritance-2026-05-04-case.md)），10 个 cluster-scoped 残留 + 24min cleanup wall-clock | chaos.sh 启动不锁 context → 继承 prior line 的 default | preflight guard（Oracle line commit `1c0b6a9`）+ KUBECONFIG strict + minified pattern（Oracle line commit `c0a4ffd`） |
| **MySQL** | adoption + fingerprint evidence | MySQL line 报告 idc bastion 共享 `~/.kube/config` 同 family 风险；提供 KubeBlocks CRD count = 28 invariant 实证 + KB image multi-source via `ParametersDefinition.toolsSetup.toolConfigs[].image` 实证 | runner 启动不验 fingerprint → addon install/upgrade 路径 silent 落到错 cluster | 同 SQL Server 三层防御模式（adoption） |

3 条 line 在不到 24h 内各自独立踩到同一 family incident + 第 4 条 line 提供 fingerprint 维度证据，说明这不是个例 doctrine。**N=3 incident + adoption/fingerprint evidence** 把 §1.4 cornerstone 从 "self-evident in SQL Server line" 升级为 "cross-line invariant"。

## 1. 四类 preflight category

> **Why these four**: 我们观察到 cross-line incident 集中在四类 trigger 上——destructive op 入口（context 漂）、ns 切换（误删跨 line 资源）、multi-cluster co-host（cluster fingerprint 漂）、network topology transition（client-side proxy 拦截 / TLS SAN 不对）。每一类都至少有 N=2 实战 incident 支撑（不是凭空设计）。

### 1.1 Context preflight — destructive op entry trigger

**Invariant**: 当前 shell / 脚本 / 工具的 kubectl 默认 context 与"intended target cluster"一致。

**触发事件**: 任何 destructive 操作前——`delete`、`cleanup`、`apply -f`、`patch`、`use-context`、`drain`、`exec` 进 db pod、`exec` 跑 sql。

**怎么 verify**:
```bash
# 1. lock 显式 context（首选）
kubectl --context=k3d-mssql-kb103b5 ...
# 2. lock KUBECONFIG（次选）
KUBECONFIG=~/.kube/config-mssql kubectl ...
# 3. assert before destructive op（兜底）
[ "$(kubectl config current-context)" = "k3d-mssql-kb103b5" ] || { echo "FATAL: wrong context" >&2; exit 99; }
```

**绝对禁止**:
- 在多 line 共享的 `~/.kube/config` 上跑 `kubectl config use-context`，然后假设它会保持
- 把 `kubectl config current-context` 当成"安全验证"——它是**最后一次有人 use-context 的结果**，不是**当前 cluster 真实绑定**

### 1.2 Namespace preflight — multi-ns workload switch trigger

**Invariant**: 命令的目标 namespace 与"intended workload scope"一致。

**触发事件**: 多 ns 操作、cleanup 跨 ns、`-A` / `--all-namespaces` 用法。

**怎么 verify**:
- 优先用 `-n <ns>`，不要依赖 `kubectl config set-context --current --namespace=`（这也是共享 mutable state）
- destructive 操作（delete / patch）禁用 `-A` 除非脚本顶部有 explicit invariant assert
- ns 名字含 line prefix（`mssql-` / `ora-` / `mysql-` / `valkey-`），cleanup 时 grep prefix 反查不要误删

### 1.3 Cluster fingerprint preflight — multi-cluster co-host trigger

**Invariant**: 操作目标 cluster 的 fingerprint（API server URL / TLS SAN / version + addon set + storage class set + controller image + CRD count）与 baseline 一致；addon 的 Helm chart bootstrap 路径未被 bypass。

**触发事件**: 在共享 Mac / IDC 上同时跑多个 cluster；cluster restart / re-bootstrap；切换 KB 版本。

**Fingerprint 维度（按 fail-fast 顺序）**:

> 落地建议：把这一段写进每个 line 自己的 `bootstrap-verify.sh` / `chaos-preflight.sh` 入口；前 2 项（API server + TLS SAN）几乎不会假，最值得做 fail-fast。

1. **API server URL** — `kubectl config view --raw -o jsonpath='{.clusters[?(@.name=="<ctx>")].cluster.server}'` 与 baseline 比对
2. **TLS SAN** — 通过 `openssl s_client -connect` 提取 SAN 列表，对比 intended cluster 的 cert SAN
3. **KubeBlocks CRD count = 28（KB 1.0.3-beta.5 invariant，N=2 cross-line 实证：SQL Server + MySQL）** — `kubectl get crd -o name | grep kubeblocks.io | wc -l` 应稳定 = 28（KB 1.0.3-beta.5 标准 set）。**注意必须按 group filter `kubeblocks.io`**——cluster 上还会有 VolumeSnapshot / cert-manager / vcluster 等其他 CRD，直接 `kubectl get crd | wc -l` 会高出 28 误判 false-pass / false-fail。<28 必有 KubeBlocks CRD 缺失，addon install 会 silent fail
4. **VolumeSnapshot CRD first blocker（N=3 cross-line 实证：Oracle / SQL Server / MySQL）** — 没有 `volumesnapshots.snapshot.storage.k8s.io` CRD 时，`kubeblocks-dataprotection` controller 的 cache sync / Backup CR reconcile 失败或重启，Backup CR 可能无 status；但 chart install 阶段不报错。所有 line 都把它列为 first blocker check
5. **Addon 集合** — `helm list -A` 与 line baseline 比对，不在 baseline 的 addon 标记为 cross-line 残留
6. **Node 集合** — `kubectl get nodes -o name` 与 baseline 比对（漂移可能是 vcluster→host 错 mount）

**架构 invariant 补丁**:

- 部署 cluster CR **必须走 `addons-cluster/<engine>` Helm chart**——chart 提供 addon entrypoint 期待的 Secret/Volume bootstrap（如 SQL Server 的 `<name>-certificates` Secret、自动 genCA dbm_certificate.{cer,pvk,pfx,crt,key,password}）。**手写 Cluster YAML / kubectl apply 会 silently 丢失这些 Secret，addon entrypoint 在运行时报 cryptic 错误（"cp: cannot stat '/certificates/dbm_certificate.*'"）**
- StorageClass / CSI provisioner 在 vcluster→host 同名传递场景下，必须先在 vcluster 内 mirror 一个 SC 指向 host CSI provisioner（vcluster syncer 会把 PVC 路由到 host CSI）
- chaos primitive：`kubectl delete pod` in vcluster = host pod kill（vcluster syncer 立刻同步），**不需要 chaos-mesh** for pod-kill 类 chaos（OceanBase line idc4 实证）
- **KB image 多源**: config-manager sidecar image 来源不只是 ComponentVersion，还可能是 **`ParametersDefinition.toolsSetup.toolConfigs[].image`**（MySQL line 实证：sidecar image 实际指向 `docker.io/apecloud/mysql:8.0.44`，retag 后是 ACR 内的 `apecloud/mysql:8.0.44`，sha256 e21043d3...796ea1）。candidate image distribution 通常走 ACR mirror 模式：source registry → ACR mirror → 各 idc bastion 拉取。如果 idc bastion 不能直连 source registry，每 line 都要 audit `ComponentVersion` 与 `ParametersDefinition.toolsSetup.toolConfigs[]` 两处 image 字段，确保都已 mirror 到 ACR（任一处遗漏就会 ImagePullBackOff）

### 1.4 Client-network preflight — network topology transition trigger

**Invariant**: client → API server 的网络路径、TLS 链、proxy 链、context 锁全部确定。

**触发事件**: 切换 local k3d / IDC bastion / corp VPN / vcluster API；macOS 启停代理 app；跨子网。

#### 1.4.a vcluster NodePort + TLS SAN

vcluster API 走 NodePort 时 SAN 通常**不**包含 nodeIP，要在 kubeconfig 里 `insecure-skip-tls-verify: true` 并 drop `certificate-authority-data`（helm 不接受命令行 `--insecure-skip-tls-verify` 覆盖 kubeconfig CA）。

接受 `insecure-skip-tls-verify` 的前提是**已经在 1.3 里验证过 fingerprint**——单点 server URL 不够，必须配合 cluster fingerprint preflight。

#### 1.4.b `~/.kube/config` current-context 风险（核心论点）

**`~/.kube/config` `current-context` 在 multi-tenant Mac 上是 shared mutable global state**——任何 line 跑 `kubectl config use-context` 都构成 cross-line side effect。

**前 line 留下的 default 就足以污染后续工作（不需要任何 explicit use-context 动作）**——这是 Run 7 incident 给出的更深一层 doctrine：cross-line invariant break **不需要** explicit `kubectl config use-context`，**继承**前一个 session 留下的 default 就足以让 chaos.sh / writer / cleanup 跑进错的 cluster。

所有读 default context 的工具（writer / health-checker / hourly-cross-confirm / fault-runner / cleanup）必须 pin `--context=` 或 `KUBECONFIG=` 显式路径。

#### 1.4.c Stop-gap pattern: per-shell minified kubeconfig file

当 multi-line 共享 `~/.kube/config` 不可避免（local Mac k3d / 远程 bastion 共享 home），各 line 用独立 minified-context 文件完全绕开 default context shared mutable state：

```bash
kubectl config view --raw --minify --context=k3d-mssql-kb103b5 --flatten > /tmp/kubeconfig-mssql.yaml
export KUBECONFIG=/tmp/kubeconfig-mssql.yaml
```

不需要改一行业务脚本（kubectl 自动用 `KUBECONFIG=` 指向的文件，里面只有一个 context）。这是 r3+ kube() wrapper 落地之前的 immediate stop-gap，也可以作为长期模式。

**注意 stale kubeconfig file 残留风险**：`/tmp/kubeconfig-*.yaml` / `~/.kube/config-*` 这种文件如果上次 session 没 cleanup，下次新 line 看到同名 / 近名文件可能误用。entry script 启动时要**强制重新 minify**（不能 reuse `if [ -f /tmp/kubeconfig-mssql.yaml ]`），end script 要 `trap rm` 清理。OceanBase line 实证：

> OB line 在 Machine D 上同时有 idc4 host kubeconfig、有效 vcluster kubeconfig `~/.kube/config-ob-vcluster-raw`，以及旧的失效文件 `~/.kube/config-ob-vcluster`。风险不是已确认的具体故障，而是 typo / 自动补全 / 默认路径误用旧 kubeconfig，导致命令打到错目标或访问失败。入口脚本必须显式指定正确 kubeconfig，并先探测 `kubectl --kubeconfig <intended> get ns`；不要复用不明来源的 kubeconfig 文件。

落地 pattern：文件名要能看出用途（哪个 host / 哪个 cluster / 哪条 line），并避免 `<base>` 与 `<base>-suffix` 这种视觉相似但语义不同的并存名字；不要复用不明来源的 kubeconfig 文件，启动前实际探测一次再跑业务。

**架构层消除 stale 风险（OceanBase line idc4 实证）**：把 runner 迁到 host k8s 集群里跑，kubeconfig 走 Secret 挂载（`kubernetes.io/service-account-token` 或自定义 Secret with ClusterIP target），不再依赖 host filesystem 上散落的 `~/.kube/config-*` 文件。Pod 重建即拿到新 token，stale 风险从"文件管理纪律问题"降级为"架构上不存在"。这是 §1.4.c 整段问题最强的 mitigation，但前提是有专门的 host k8s 可以跑 runner（不是所有 line 都满足）。

#### 1.4.d Mid-run 自保：context-guardian loop

长跑测试已经 fork 出 worker 进程后才发现 invariant 破坏（worker 用错的 default context 跑了一段时间），不能 kill worker 的情况下，起 guardian loop 锁回 default context：

```bash
nohup bash -c 'while true; do kubectl config use-context k3d-mssql-kb103b5 2>/dev/null; sleep 10; done' &
```

注意 guardian 只是绷带，不是治本——前提是其他 line 也已经绕开 default context（用 minified file），否则双方互相切死锁。

#### 1.4.e Fail-fast strict 模式

长跑 soak 期间，加 fail-fast strict 模式：`KUBE_STRICT=1` + `kube()` wrapper，未 set `KUBE_CONTEXT` 直接 exit 99。

跨 line 通告永远不够，**必须程序锁**——已有实战 N=2 双 incident 实证（详见 §2 row 4 / row 4b）。

#### 1.4.f HTTPS_PROXY interception case（Valkey line 提出，Mac local 高发）

macOS 上后台代理 app（Surge / ClashX / Proxifier）会把 HTTPS_PROXY=http://127.0.0.1:6666 注入 env，**且不 bypass IDC CIDR**——任何走 IDC NodePort 的 HTTPS handshake 都会被代理拦截，伪装成各种 cert / routing 错误：

- 假象 1：vcluster cert SAN 缺 nodeIP（实际 SAN 没问题，是 proxy MITM）
- 假象 2：kube-proxy iptables 没装好（实际 proxy 把 connect 转到 127.0.0.1）
- 假象 3：vcluster routing 不支持外部 NodePort（实际是 client-side拦截）

**Falsification**: `unset HTTPS_PROXY HTTP_PROXY ALL_PROXY https_proxy http_proxy all_proxy`，再 `curl -v https://<nodeIP>:<port>/`。立刻 OK = 100% 是 client proxy 拦截。

#### 1.4.g Fingerprint 直接复核（Valkey line 推荐 prose 模式）

> 不要在受污染的 default context 上 falsify default context；要用一个**独立 minified kubeconfig**（或显式 `--context=<intended>`）发起 fingerprint 探测。Falsify 的目标是确认"intended cluster 的实际 fingerprint"，不是"当前 default context 看到什么"。如果两者矛盾，结论永远是 default context 已漂，不是 intended cluster 异常。

具体步骤：
1. 取 intended cluster 的预期 fingerprint（API server URL + TLS SAN + CRD count）作 baseline
2. 用 `kubectl --context=<intended-explicit>` 或 `KUBECONFIG=<minified-file>` 直接探测
3. baseline = probe → cluster 正常，问题在 client 侧 default 漂
4. baseline ≠ probe → 真的是 cluster 异常，进 cluster-side 排查

## 2. 反模式附录（3-column row：现象 / 错误归因 / falsification step）

| # | 现象 | 错误归因 | Falsification step |
|---|---|---|---|
| 1 | vcluster NodePort 31815 HTTPS handshake stall (Mac → IDC nodeIP) | "vcluster cert SAN 缺 nodeIP，要重签 CA" / "kube-proxy iptables 没装好" / "vcluster routing 不支持外部 NodePort" | `unset HTTPS_PROXY HTTP_PROXY ALL_PROXY` 后立刻 OK→ 根因是 Mac client-side proxy interception。Falsification 命令：`curl -v https://<nodeIP>:<port>/` 配合 `env \| grep -iE 'proxy'` 双向比对 |
| 2 | 跨 vcluster / 跨 cluster session 里反复出现"network 像断了又通" / "有时候 cert OK 有时候 cert 不 OK" | "网络抖动" / "cert 有时被刷新" | 检查所有 PROXY env var 在不同 shell 里的值 + macOS 系统代理 GUI 状态。这是 cognitive blind spot：长期 background 噪声 → 大脑当 invariant 不复检 |
| 3 | 同 Mac 上跑 chaos 测试，cleanup 后发现自己 cluster 里有别的 line 的 CmpD/OpsDef/PCR | "addon helm install 时多装了" | `kubectl --context=<intended> get cmpd,opsdef,paramconfigrenderer -A` **显式锁 context** 复核（不要在 default context 上查；default 可能已经被漂走），对比时间戳。根因往往是 chaos.sh 没 explicit `--context` / `KUBECONFIG`，default context 漂到别的 cluster，对方 addon install 落到自己 cluster |
| 4 | 长跑 24h soak writer 突然连续报 "no primary pod label found" 但 cluster 实际正常（primary pod 4/4 Running、bad_ack=0） | "writer 进程 stuck" / "kubectl probe 卡住" / "primary 没 publish role label" | `kubectl --context=<intended> get pods -L kubeblocks.io/role` vs writer 视角对比。根因：writer 用 default context（`~/.kube/config` 的 current-context），被另一 line 的 `kubectl config use-context` 切到错的 cluster——是 **invariant 被 cross-line side effect 破坏**，不是 writer bug。修复：writer pin `--context` + r3+ `kube()` wrapper + `KUBE_STRICT=1` + 短期 context-guardian loop |
| 4b | 同 #4 但发生在 doctrine 已成文 + 当事人 voice commitment 已下达 1h+ 后再次复发 | "上次提醒过了应该不会再犯" / "process discipline 应该够了" | 直接看 `slock daemon timestamp`（不可篡改）：第二次 fired_at 与 voice commitment time 之间 1h+ → 反证 voice commitment 不是 invariant。强制升级到 ①程序锁 + ②fingerprint + ③fail-fast 三层防御 |
| 4c | 同一台机器上有多个相近 kubeconfig 文件，runner / 手工命令可能因 typo 或自动补全选错文件（OceanBase Machine D 风险实证） | "vcluster API 不稳定" / "kubeconfig 过期" | 列出 host 上所有 kubeconfig 候选 + 检查它们的 server URL / context name + `kubectl --kubeconfig=<候选> get ns` 各跑一次对比。根因：host 上同时存在 idc4 host kubeconfig + 有效 vcluster kubeconfig + 旧失效 vcluster kubeconfig，typo / 自动补全 / 默认路径误用旧文件 → 命令打到错目标或访问失败。修复：入口脚本显式指定正确 kubeconfig + 启动前探测 + 文件名带 purpose（不复用不明来源的文件） |
| 5 | 测试启动后立刻报 cluster context 不对 / 找不到 namespace / addon CRD 缺失（boot-up 阶段） | "addon install 失败" / "cluster crash" | boot-up 阶段就跑 fingerprint preflight（API server URL + KubeBlocks CRD count + addon set），fail-fast → Valkey line boot-up self-catch 实证：boot-up self-catch latency = 30s，远小于 mid-soak detect (35min) 远小于 post-voice-commit (1h+)。**Reset-cycle latency gradient: 越早 catch cost 越小**，所以 preflight 必须在 boot-up entry 就跑，不要等长跑里 mid-run 出问题 |
| 6 | idc bastion 上 addon helm install 落到 wrong cluster（Run 7 family，N=3 incident cross-line） | "chaos.sh 选错 cluster" / "脚本写死了 cluster name" | chaos.sh / runner 入口加 `KUBECONFIG=` 显式锁 + `current-context == intended` assert + 11-line preflight guard（Oracle line commit `1c0b6a9` 模板）。根因：脚本启动**继承**前一 session 留下的 default context，silent fall through 到任何 default。voice commitment 不防 inheritance |
| 7 | candidate image 分发到 IDC 后 helm install 失败 "ImagePullBackOff"（MySQL line 实证 sha256 e21043d3...796ea1） | "镜像 push 失败" / "registry quota 满" | 1) `kubectl describe pod` 看真实 image ref；2) 对照 `ParametersDefinition.toolsSetup.toolConfigs[].image`（**这是 config-manager sidecar image 的另一来源，不只是 ComponentVersion**）；3) `crictl images` 在 node 上验。根因：image multi-source（ComponentVersion + ParametersDefinition.toolsSetup）任一处指向不在 ACR mirror 的 source = pull fail |

## 案例

- [`cases/methodology/run7-context-inheritance-2026-05-04-case.md`](cases/methodology/run7-context-inheritance-2026-05-04-case.md) — Oracle Run 7 context leak（fired_at 12:48Z / detected_at 12:57Z / mitigated_at 13:21Z）；时间线、10 个残留 CmpD/OpsDef/PCR 清单、cleanup 24min wall-clock、Oracle line 11-line preflight commit `1c0b6a9` + KUBECONFIG strict + minified `c0a4ffd`、cascade pattern (Run 7 → r2 W1)
- [`cases/methodology/r2-writer-context-flip-2026-05-04-case.md`](cases/methodology/r2-writer-context-flip-2026-05-04-case.md) — r2 24h soak writer artifact window（fired_at 12:57Z / detected_at 13:18Z / mitigated_at 13:53Z 第一窗口；fired_at 13:57:07Z / mitigated_at via guardian 第二窗口）；408 条 client_failed 双窗口 id range；SQL Server line 67-call kubectl audit + `kube()` wrapper fix plan；context-guardian 10s loop 紧急绷带；voice-commitment-not-invariant 实证

## Cross-references

- `addon-test-environment-gate-hygiene-guide.md` — 单 line / 单环境 post-restart 视角。本文是它的 cross-line 补集
- `addon-evidence-discipline-guide.md` — 三规则：N≥2 案例 / 描述强度匹配证据强度 / 不二选一表述。本文 N=3 incident + MySQL adoption/fingerprint evidence pool 自身按这套规则跑
- `addon-github-submission-discipline-guide.md` — Doctrine B：commit no AI co-author trailer。本文 PR 的 commit hygiene 引用这个

## Author / evidence partners

- 主笔：Tom (SQL Server line)
- Evidence partner: James (Oracle line) — §1.1 + §1.4 cornerstone 共建；Run 7 incident 一手数据；context-flip-r2 二次重发 cross-line investigation；commits `1c0b6a9` + `c0a4ffd` 双 mitigation 实施
- Evidence partner: John (Oracle line) — Run 7 first-detect；context-flip 双 incident self-report timeline；shell history 取证；5 数据点 inventory
- Evidence partner: Jerry (SQL Server test) — r2 writer artifact 数据；67 处 kubectl audit；patch v1/v2 实现；context-guardian
- Evidence partner: Alice (Valkey line) — fingerprint pattern prose block；HTTPS_PROXY interception case；Valkey boot-up self-catch incident relay
- Evidence partner: William (MySQL line) — N=3+adoption evidence pool elevation；§2 row 3 falsification with `--context=<intended>`；KubeBlocks CRD count = 28 N=2 invariant；MySQL bastion 同 family 风险报告
- Evidence partner: Henry (MySQL line) — KB image multi-source via ParametersDefinition.toolsSetup.toolConfigs evidence (sha256 e21043d3...796ea1)；candidate image distribution ACR mirror pattern
- Evidence partner: Mia (OceanBase line) — Machine D 多 kubeconfig 共存 + typo / autocomplete 误用旧文件风险案例（via cross-line forward + Mia 复核）
- Evidence partner: Allen (cross-line review) — filename + cross-ref placement；§1.3 fingerprint 维度排序建议；OceanBase case relay
- Evidence partner: Noah (OceanBase line) — vcluster→host pod kill primitive idc4 实证；chaos-mesh 不必要的 N=2 confirmation；架构层消除 stale kubeconfig 风险（Secret-mounted + ClusterIP）
- Evidence partner: Bob2 (Valkey test) — boot-up self-catch incident；reset-cycle latency gradient lower-bound 数据点

(see `addon-evidence-discipline-guide.md` 三规则——这篇 doc 自己也按那个规则跑：N=3 cross-line incident + MySQL adoption/fingerprint evidence，描述强度匹配证据强度。)
