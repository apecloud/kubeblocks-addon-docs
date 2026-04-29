# Addon 测试 Host 压力与污染累积指南

> **Audience**: addon test engineer / TL、host 运维、值班工程师
> **Status**: draft（cell-4 evidence pending；chapter 7 N=2 对账 pending）
> **Applies to**: 任何在共享单 host k3d 环境跑 addon 全量回归的场景；具体引擎实现细节放案例附录

## 先用白话理解这篇文档

### 这篇文档解决什么问题

你跑 addon 全量回归（smoke + chaos + 多个 suite）时，**k3d 集群可能突然"看起来崩了"**：`/readyz` 报错、CSI 容器 CrashLoop、kubelet 一堆 timeout。本能反应是"host 出问题了，重启 docker 或重建 k3d"。

本文档的核心结论是：**这通常不是 host 真崩，是控制面被测试残留累积压垮**。重启级 recovery 多数时候没必要——清掉测试 namespace 里残留的 cluster / Pod / PVC，控制面 1-2 分钟内自愈。

### 读完你能做什么决策

- **撞到 cascade 信号时**：不靠重启 docker 也能恢复（chapter 1 / chapter 5）
- **runner 设计阶段**：知道 cleanup gate 是 mission-critical 而不是测试卫生，决定要做到什么严密程度（chapter 4）
- **现场 freeze 时**：知道按什么顺序抓什么 evidence、用什么命令清污染（chapter 6）
- **规划 hot-fix 类操作时**：知道为什么"先清污染再 hot-fix"比"直接跑 hot-fix"更可靠（chapter 7）

### 为什么独立成篇

这篇文档围绕一个具体观察：**"清污染 → 自愈"是 host cascade 类信号的 first-line 恢复手段**。这条观察统一了 cleanup gate 设计、判定流程、hot-fix 价值归因——所以适合放在一篇里讲。其它独立主题（例如 narrow-scope force-delete、env gate hygiene）参见相邻 doc。

---

## 主体内容

本文沉淀多团队（Valkey + MariaDB）在共享单 host k3d 环境跑全量回归时观察到的 **host 层 cascade 现象**，以及如何用 **evidence-preserving cleanup** 作为 first-line 恢复手段。

不绑定单一引擎；具体引擎实现、samples、commit hash 等放在案例附录里。

**关键术语**（首次出现先解释，后续直接使用）：

- **cascade**：控制面/CSI/kubelet 多个组件反复 fail-restart-fail 形成的自反馈失效，下文 chapter 1 详述
- **reactive overload**：组件因 reactive 处理过多事件/对象，导致 probe fail → restart → 再被同样负载推垮，**这是 cascade 的根因机制**，不是 host 资源耗尽
- **bounded eventually**：测试 verify 时给一个有限重试窗口（次数 + 间隔 + 总超时），超过仍 fail 才记产品 fail；与"无限等"不同
- **evidence-preserving cleanup**：清污染前先把现场 evidence 落盘（freeze），让 cleanup 之后仍能审计 cascade 当时的状态；chapter 6 详述
- **cleanup gate**：runner 在两个 suite 之间显式等前一 suite 资源全部清干净再进下一 suite 的检查机制；chapter 4 详述
- **污染累积（pollution accumulation）**：runner cleanup 不严时，多个 suite 的残留 cluster / Pod / PVC 同时存在于 host 上，等效于持续叠加压力；chapter 2 evidence matrix 的 cell-2

## chapter 1：cascade 是控制面 reactive overload，不是 host 真崩

实际观察到的"cascade 信号"组合通常是：

- `/readyz` 间歇性 `InternalError`，body 含 `etcd-readiness failed: reason withheld` 等控制面失败项
- `csi-hostpathplugin-0` 等关键 DaemonSet/Pod 出现容器级 `CrashLoopBackOff` 或 `ContainerCreating` 风暴
- `kubelet` 反复 `FailedMount`（configmap cache sync timeout）、`probe failed`
- `coredns` readiness/liveness Warning
- `resource quota evaluation timed out`
- 节点 `Ready` 状态可能仍然 `True`（不是真 NotReady），但功能 degraded
- k3d 容器 docker stats CPU 飙升（曾观察到单容器 600%+）

**容易踩的误判**：把这套信号读成"host 崩了，需要 docker restart 或 k3d 重建"。

**更准确的理解**：这套信号是**控制面 reactive overload**——某个或几个组件持续 reactive 处理过多对象/事件/exec 调用，liveness/readiness probe 反复 fail 触发 restart，restart 后又被同样的负载继续推垮，形成自反馈。host 本身（kernel、磁盘、网络）通常没真崩。

**关键证据**：移除压力源（清掉测试 namespace 里的 polluted resources）后，**通常不需要 docker restart 也不需要 k3d 重建**，控制面会在 1-2 分钟内自愈：`/readyz` 回 ok、CSI plugin 停止 CrashLoop、k3d CPU 显著回落。这一现象在多个真实事件里重复观察到（参见 cell-2 / 附录 A），是本文档最核心的产物。

**注意**：这条结论的强度建立在 cleanup-only 路径已观察到 self-heal；但**没有控制实验** disprove "如果同时 hot-fix 会更快或更彻底" 这类备选解释。chapter 7 详细讨论这条 framing 的 epistemic 边界。日常运维仍按 chapter 3 recovery 路径从轻到重排序。

**实际 actionable 含义**：

- 第一次撞这套信号时，**不要立刻上 docker restart / k3d 重建**
- 先做 evidence-preserving cleanup（chapter 6）
- 清完观察 30s-2min，env 大概率自愈
- 自愈后再判断是否需要进一步动作（重启 / 修 mount propagation / 调资源）

## chapter 2：empirical evidence matrix

观察到 cascade 的 stress 模型有四种典型形态。下表用今天单次跨团队事件填了三种，最后一种留 placeholder：

| stress 模型 | 是否撞 cascade | 关键证据要点 |
|---|---|---|
| **cell-1**：dual-engine + 双污染（A 引擎 + B 引擎同时跑全量 + 残留集群同时存在） | ✅ 红线 | MariaDB #396 Round 2 / Round 2-replacement：T12 reconfigure 期间 `csi-hostpathplugin-0` RESTARTS 0→3 → 3/4 Error，sidecars ContainerCreating 风暴，host API 短时不稳，11443 listener fail；详见 [appendix C](#appendix-c-cell-1-mariadb-pollution) |
| **cell-2**：single-engine 多 suite + 污染累积（一个引擎跑 sequential 多 suite 但 runner cleanup 不严，多个 cluster 残留） | ✅ 红线 | Valkey #208 Round 2：CSI plugin 进 `3/4 CrashLoopBackOff`、`/readyz InternalError` + `etcd-readiness failed`、k3d CPU 峰值 1645%；清掉 polluted namespaces 后**没做 host-side recovery 就自愈**（CPU 1645%→210%，CSI 回 4/4 Running）；详见 [appendix A](#appendix-a-valkey-208-round-2) |
| **cell-3**：single-engine 干净 single-load（runner 有严格 cleanup gate，suite 间清干净 cluster + namespace + PVC/PV） | ❌ 不撞 | Valkey Round 3/4/5 sequential `--test all`：3 轮一致 0 product fail、chaos 143/0/0 三连绿、CSI plugin RESTARTS 全期 22→23（仅 +1 health-monitor）、`/readyz` 全程 ok；详见 [appendix B](#appendix-b-cell-3-valkey-solo) |
| **cell-4**：single-engine 干净 single-load（另一引擎，对照） | ❌ 不撞 | MariaDB cell-4 R1（sample `mdb-vc396r1-134638`，alpha.11 + 12 patches，KB 1.0.2）：T1-T11 / CM1-4 / R3 全过；T12 触发 `csi-hostpath-resizer-0` 单一 sidecar restart 0→1 触发 hard-freeze，但 plugin 4/4 Running RESTARTS=23 不变 / sidecar 不 storm / `/readyz` 5×ok / k3d CPU 256%（远低于 cell-2 1645%）；**cascade shape 完全 absent**；详见 [appendix D](#appendix-d-cell-4-mariadb-solo) |

**从矩阵推导出的方法论**（four-cell two-engine N=2 confirmation）：

- cell-2 红线 + cell-3 绿 → 区分点是 **runner cleanup gate**：cleanup gate 不严会让 sequential 退化成"等效多 suite 同时占资源"
- cell-3 绿 + cell-4 绿 (跨引擎双 N=2) → cascade **不是单引擎 workload 的固有属性**，是"压力如何累积"决定的
- cell-1 红 + cell-2 红 (同形态 cascade，跨引擎双 N=2) → dual-load + 污染累积**双锁定为充分条件**
- 同时：cell-2 单独已证明 dual-load 不是必要诱因（valkey 单边 + 污染累积就足够触发）

→ 因此：cascade trigger 模型 = **(dual-load 或 single-load) + 污染累积 / 反应过载**；干净 single-load 不论引擎都不撞。

## chapter 3：清污染就是 first-line recovery

Recovery 路径按从轻到重排序：

1. **evidence-preserving cleanup**（chapter 6）——大多数 cascade 第一线手段
2. **针对性组件 rollout**（local-path-provisioner / metrics-server / 单点 controller）——污染清掉但某个组件仍在异常状态时
3. **CSI hot-fix**（节点 `mount --make-rshared /` + 重建 csi-hostpath* pods）——只在 mount propagation 真坏时
4. **docker daemon restart**——更重，但保留 image cache
5. **k3d cluster 重建**——彻底但代价大；需要重新 import controller image / 重装 addon

历史上（包括之前 #376 hot-fix）习惯直接跳到 3-5 步。新结论：**先试 1**。不要把"重启级 recovery"当成默认选项。

判定流程见 chapter 5。

## chapter 4：cleanup gate 是 mission-critical，不只是测试卫生

**经验教训**：runner 在 suite 之间没有严格 cleanup gate 时，sequential 模式会**退化成等效"伪并行"压力**——前一 suite 的 cluster / Pod / PVC / PV / Backup CR 还没清完，后一 suite 已经在新建对象，host 上同时存在多组 Valkey/MariaDB 集群，压力叠加。

cleanup gate 的设计标准：

- 删 cluster CR 后**等待 cluster 真消失**（不是 fire-and-forget `--wait=false`）
- 等 Pod / PVC / PV / Backup CR / Ops 等关键 owner 资源清干净（用 owner-residue 检查，不依赖 namespace 删除）
- 清完后**显式删除并重建 namespace**，把非 labeled residue 也清掉
- 失败路径（fatal abort、SIGINT、timeout）要走**EXIT trap** 触发 cleanup，不能让 abort 跳过 cleanup
- final suite 后也要做整体 cleanup，不留空壳 namespace

可测试性：写一个 mock run（比如 `RUNNER_CLEANUP_GATE=true ./run-tests.sh -t missing-a,missing-b`），验证 namespace 在 suite 间确实被删除/重建，最终 NotFound。

## chapter 5：判定流程——撞到 cascade 信号时怎么办

### 重要前置：cascade 早期信号可以早于 CSI 观测层

**经验教训**（来自附录 C 的 mariadb #396 cleanup attempt 19:41 时间线）：
- runner 卡住 / OpsRequest timeout / port-forward listener fail 这类 **product-layer 信号** 通常**早于** CSI 容器级 CrashLoop / `/readyz` InternalError 这类 **observation-layer 信号**
- 也就是说：当你看到 runner 卡住但 `/readyz` 仍 ok 时，**不要因为 readyz 绿就否定 cascade 早期信号**
- 反向：如果你只盯 `/readyz`，cascade 可能已经在 product layer 走完一半你才反应过来

判定流程**先看 product-layer 信号**，再看 observation-layer：

### 30 秒判定流程

1. **看 product-layer 早期信号**（runner / OpsRequest / port-forward）：
   - runner 卡 `Waiting for cluster ... Running` 超出预期 wait budget？
   - OpsRequest 提交后控制面长时间无响应？
   - port-forward listener 起不来 / connection refused？
   - 如果有任何一条 → 不要等 CSI/`/readyz` 信号确认，直接进 step 2 freeze evidence
2. **看 namespace 里残留**：`kubectl get cluster,pod,pvc,pv -A | grep <product>`
   - 有大量残留 → **先做 evidence-preserving cleanup**（chapter 6），不要重启
   - 无残留 → 跳到 step 3
3. **看 observation-layer 信号是否 sustained**：连续 `kubectl get --raw=/readyz` 5-10 次
   - 间歇性 ok → 可能是瞬时 spike，再观察 1-2 min
   - 持续 fail → 跳到 step 4
4. **看具体哪一层动**：
   - 仅 CSI hostpath plugin → 优先 chapter 3 step 2-3 路径
   - apiserver / etcd 也在响 → 跳到 step 5
5. **决策树**：
   - 如果有产品 evidence 价值（live CrashLoop 现场是关键诊断证据）→ **freeze + 落盘 evidence**（chapter 6 给方法）
   - evidence 已落盘后再决定是否 docker restart / k3d 重建
   - 不要因为现象严重就跳过 evidence 落盘

### 信号时间窗的非对称性

把 product-layer 和 observation-layer 信号画成 timeline，会发现两条线**通常不同步**：

- product-layer 信号 = "我做的事情完不成"（runner / OpsRequest / 用户操作）
- observation-layer 信号 = "K8s 自己说自己有问题"（`/readyz` / pod CrashLoop / events）

cascade 早期，product-layer 已经在 fail 但 K8s 还认为自己 healthy；cascade 中后期才会同步。所以单看任一层都会误判：

- 只看 product-layer：可能把 transient 网络抖动也读成 cascade
- 只看 observation-layer：cascade 已经走了一半才反应过来

**正确做法**：两层一起看，product-layer 早期信号触发 freeze + 抓 observation-layer 帧来对账。

## chapter 6：evidence-preserving cleanup 的最小动作集

### 第一步：固定 evidence

- **freeze active runner**：如果 runner 还在跑而你要清它的 namespace，先 SIGSTOP runner 父进程（避免 EXIT trap 干扰现场），然后 evidence 落盘后用 `kill -9` 直接收尸（stopped 进程 SIGTERM 不响应，SIGKILL 直接生效，绕过 EXIT trap）
- **抓时间戳同帧 evidence**：
  - `docker stats --no-stream`（k3d 容器 CPU/MEM/PIDs）
  - `kubectl get nodes -o wide`（节点 conditions / transition time）
  - `kubectl get --raw=/readyz` x 多次（区分瞬时 vs sustained）
  - `kubectl get pod -n kube-system / -n default`（关键 pod RESTARTS、phase）
  - `kubectl describe pod <suspect-pod>`（events 链）
  - `kubectl get events -A --sort-by='.lastTimestamp' | tail -30`
  - 怀疑 CSI 时：`kubectl describe pod csi-hostpathplugin-0`、相关 sidecar logs tail
- 关键产品 namespace 也要冻结：`kubectl get cluster/pod/pvc/pv/ops/backup -n <ns> -o yaml > freeze.yaml` 在删除前

**`/tmp` vs 持久化**：freeze evidence 第一时间可以落 `/tmp` 抢现场（速度优先），但**在 cleanup 报告（pings 同事 / 写 thread / 贴 PR）发出前**，必须把 evidence 复制 / tar 到持久 artifacts 目录并附 sha256 / attachment。否则 `/tmp` 会被系统清理或 reboot 干掉，evidence chain 失效。

参考路径：
- 临时抢现场：`/tmp/<run-id>/freeze/freeze-<timestamp>/`
- 持久归档：`/Users/.../artifacts/<run-id>/freeze-and-cleanup-<timestamp>.tar.gz`（带 sha256）

### 第二步：cleanup 顺序（按 namespace 类型分支，**禁止统一 `delete ns` 起手**）

先判断 namespace 类型，分支处置：

#### 分支 A：独占 / 一次性 namespace

适用场景：runner 启动时 `kubectl create ns <test-ns-with-timestamp>`，里面所有资源都属本次测试，namespace 本身就是一次性的。

1. `kubectl delete ns <ns>`（不加 `--wait=false`，让它自然走 finalizer 流程）
2. 如果某个 PVC 卡 `kubernetes.io/pvc-protection` 且 namespace 已不在但 PVC 还能 list（**phantom namespace** 现象）：
   - `kubectl create ns <ns>`（补回空壳 namespace）
   - `kubectl patch pvc <name> -n <ns> --type=merge -p '{"metadata":{"finalizers":[]}}'`
   - 等 namespace 自然走完
3. cluster CR 卡 finalizer 时同理

#### 分支 B：共享 namespace / vcluster source namespace

适用场景：namespace（如 `mariadb-test` / `valkey-test`）由测试基础设施持久管理，里面同时存在**本次测试要清的资源** 和 **不能动的 pre-existing 基础设施**（BackupRepo、StorageClass 引用的 PVC、长存 Service/Endpoint 等）。

**禁止** `kubectl delete ns` —— 那会把基础设施一起干掉。

正确顺序：
1. **走产品控制面 API** 删上层资源：`kubectl delete cluster/<owner-name> -n <ns>`、`kubectl delete ops/<name> -n <ns>` 等
2. **按 owner / name prefix 等待 drain**：`kubectl get pod,pvc,svc,cm,secret -n <ns> -l app.kubernetes.io/instance=<owner-name>` 应在合理时间内为空
3. **不要 force-delete pod / PVC**（不用 `--grace-period=0 --force`）—— 会污染 finalizer 状态且不干净
4. **保留 pre-existing 资源**：BackupRepo PVC、长存 Service、命名空间级 ServiceAccount 等

#### 分支 C：边界情况

- vcluster + host 双层 namespace：通过 vcluster connect 走 source plane delete，host 侧 mirror pod 自动 drain（**不直接动 host pod**）
- 控制面 dataprotection / addon controller 自身 CrashLoopBackOff：可以 `kubectl delete pod -n kb-system <crashloop-pod>` 让 deployment 重新拉，这是允许的（pod 是 deployment owner 自动 recreate；不影响 cluster）

### 第三步：post-cleanup env frame（self-heal evidence，必须落盘）

cleanup 不只是删完资源就结束 —— 还要采一帧 env 状态作为 **self-heal evidence**，证明 cleanup 不仅清干净也让 host 控制面自愈。**这是 chapter 1 hero takeaway 的 evidence 来源，非可选项**：

- `kubectl --raw=/readyz` × 5 次（区分瞬时 vs 稳定 ok）
- CSI plugin / sidecar pod RESTARTS 数（应 frozen，cleanup 后不再涨）
- `docker stats --no-stream` k3d 容器 CPU 总和（应显著低于 cleanup 前峰值）
- 任何 route / listener 残留（如 vcluster port-forward 进程是否还在）
- source / host tail count：`kubectl get pod,pvc -A | wc -l`（确认没 leftover）

落到 freeze 包同目录的 `csi-frame-post-cleanup.txt`，跟 freeze 时点对照，构成完整 self-heal evidence chain。

### 第四步：明确 "不碰" 边界写进报告

cleanup 报告里**显式列出哪些资源刻意没动**，避免协作者误以为漏清：

- pre-existing BackupRepo / StorageClass PVC
- 长存 Service / Endpoint
- 其它 namespace 的资源（如 kb-system 自身组件）
- 共享 ConfigMap / Secret

这条 discipline 防止 cleanup 越界变成 "环境清场"。

### Final check

所有清完后：
- `kubectl get cluster,pod,pvc,pv -A -l <owner-label>` 应为空（owner-residue 检查，不依赖 namespace 是否消失）
- 已落盘的 freeze + post-cleanup tar 包带 sha256，可被任意协作者下载验证
- 报告 DM / thread 里给出**持久 artifact 路径**，不只是 `/tmp` 路径

## chapter 7：hot-fix 真正的价值 vs 表面价值

### 之前的 #376 hot-fix 模型

`#376 hot-fix` playbook 是：节点 `mount --make-rshared /` + 重建 `csi-hostpath*` pods。背后假设是 `mount --make-rshared` 修复了 mount propagation drift。durability variance（20.5h vs 15min）解释为 "rshared 配置可能漂回 private"。

### 这条解释存在的问题（confounded action）

hot-fix 实际上同时做了多件事：

1. **force-delete csi-hostpathplugin pods** → 重建 pod，重置 plugin 内存状态、重连 sidecars
2. **mount --make-rshared / on each k3d node** → 重设节点根目录 propagation
3. （副作用）：plugin 重建过程中 host-side 部分污染状态可能也被清掉（旧 mount 释放、旧 PVC 引用清理）

**关键问题**：上述三件事是同一个动作里**捆绑发生**的，任何 hot-fix 后的 durability 数据**无法 attribute 到具体哪一项的修复价值**。这是一个典型的 confounded action — 没有控制实验你不知道哪一项在起作用。

### 来自 cell-2 自愈观察的反向推断

cell-2 evidence（valkey 多 suite 污染累积 → cascade）有一个**关键反向观察**：cleanup（**仅清污染，不动 plugin、不动 mount**）后 env 完全自愈，CSI plugin RESTARTS 停止增长，`/readyz` 恢复 ok，k3d CPU 显著回落。

这说明 cleanup 单独就能让 cascade 退出，不需要 hot-fix 路径里那两项 active 修复。也就是说：

- 如果某次 hot-fix 看起来"修复了"问题，可能它真正起作用的部分是 **side-effect 清污染**，而不是 rshared / plugin reset
- durability variance 更可能反映的是 **hot-fix 时点 cumulative pollution 量级**：
  - 第一次 hot-fix（撑 20.5h）：环境本来比较干净，hot-fix 顺带的污染清理效果维持得久
  - 第二次 hot-fix（15min 内又撞）：mariadb discarded 集群 + valkey 累积污染都还在，hot-fix 重启 plugin 后被同样的 reactive overload 直接拽回 cascade

**这条解释的强度**：比"rshared 漂回"假设更具 explanatory power（cleanup → self-heal 是直接 evidence 支持），但**没有控制实验**严格验证 — 我们没有跑过"清污染 + 不做 hot-fix"vs"做 hot-fix + 不清污染"的对照。

### 严谨表述

- **可以说**：cleanup 单独足以让 cell-2 cascade 退出（cell-2 evidence 直接证）
- **可以说**：hot-fix 的 durability variance 不能只用"rshared 漂回"解释，至少存在另一种合理解释（cleanup 程度差异）
- **不能说**：hot-fix 路径里的 active 修复（rshared / plugin reset）完全没用 — 没有控制实验 disprove
- **要做控制实验才能拍**：跑一次只清污染不动 plugin / mount，看 plugin RESTARTS 是否长期 frozen；或反过来 plugin reset + mount fix 但不清污染，看 cascade 是否短时复发

### Operational 含义（即便没控制实验也可立即 actionable）

- 撞到 hostpath flap 形态信号时，**不要立刻跑 hot-fix**
- 先做 evidence-preserving cleanup（chapter 6）
- cleanup 后观察 30s-2min 是否自愈
- **只有 cleanup 后仍 sustained fail，才考虑 hot-fix**
- 如果决定要 hot-fix，**hot-fix 之前也要先清环境**，否则 hot-fix 的修复价值会被污染掩盖、durability 显著缩短
- hot-fix 后**采 frozen baseline 帧**（CSI RESTARTS、`/readyz`、CPU），跟 hot-fix 前对照，让这次 hot-fix 的 durability 可被独立观察

### N=2 完整对账（cell-4 跑完后回填，two-engine confirmation）

cell-4 mariadb single-load 跑完得到的 **干净 single-load no cascade** 结果（参见 [appendix D](#appendix-d-cell-4-mariadb-solo)），跟 cell-3 valkey single-load no cascade（参见 [appendix B](#appendix-b-cell-3-valkey-solo)）合并后，给 hot-fix variance 假设一个跨引擎双 N 的 reconciliation：

1. **cell-2 + cell-1 cascade 形态一致** → 不是单引擎 quirk，是"压力如何累积"决定的（dual-load 或 single-load 都可能触发污染累积压力 → cascade）
2. **cell-3 + cell-4 no-cascade 形态一致** → 不是 hot-fix 这种"魔法变量"造成 cell-3 的 no-cascade；干净 single-load 不论引擎都不撞
3. **因此**："hot-fix 修好了 cascade" 那个解释其实是 **confounded** —— 真正的因变量是 **dual-load + 污染累积**，hot-fix 顺带的清污染只是 collateral effect

这条 reconciliation 把 chapter 7 主结论从"假设强但无控制实验" 升级为"两侧独立 N=2 重复观察支持"。仍**不是**严格 controlled experiment（没跑 "hot-fix only no cleanup" vs "cleanup only no hot-fix" 的对照），但 explanatory power 已经显著超过"rshared 漂回"假设。

---

## 附录 A：Valkey #208 Round 2 cascade case material

（cell-2 evidence 详细，引用 `valkey-full-suite-baseline-round1-213606/freeze/freeze-20260429-002757/` 和 `cleanup-westons-clear-20260429-003751/`）

待 fill：详细 timeline、freeze 时刻 vs cleanup 后 env 帧对比、CSI plugin RESTARTS / `/readyz` / k3d CPU 三组数据点。

## 附录 B：Valkey Round 3/4/5 cell-3 case material

引用 `valkey-full-suite-baseline-round{3,4,5}-*/final/review-summary.md`。3 轮 sequential 一致 0 product fail，chaos 143/0/0 三连绿。

待 fill：3 轮汇总表、关键正向信号（cleanup gate / smoke verify eventually / chaos C04/C08/C12 watchpoint）。

<a name="appendix-c-cell-1-mariadb-pollution"></a>
## 附录 C：MariaDB cell-1 case material

> **验证环境**：KB 1.0.2 + chart `mariadb-1.1.1-alpha.11`（含 12 tests-only patches）。Cluster runtime: vcluster `mariadb-test` namespace 通过 11443 port-forward。Host: Wei's Mac (Jason 管)。


任务上下文：MariaDB Phase 2ff (#396) — 3 轮 sequential stress test 求 stability signal，前置 #389 一次干净 PASS=108 FAIL=0 SKIP=4。

### Round 1（`mdb-vc396r1-180329`，18:03 起）

完整跑完，PASS=102 FAIL=5 SKIP=4，分类 `role_flicker_terminal_assertion_window`（C6/C8 阶段，OpsRequest 关单后立即终态断言读到空，观测节奏 issue 非产品 fail）。计入有效失败轮，不属 cell-1。

### Round 2（`mdb-vc396r2-183556`，18:35 起，**19:13 判 `env_freeze_uncounted`**）

跑到 R3 scale-out 后撞 host k3d 控制面抖动：
- 11443 vcluster port-forward `connection refused`
- host `kubectl --raw=/readyz` `await headers timeout`
- 默认 ns `kubectl get pods | grep hostpath` TLS handshake timeout
- 新起 `kubectl port-forward 11443` 进程不进入 listener 状态

Jack 19:23 升级判定为 host API 不稳，runner 冻结。Jason 同时段独立观察到 `csi-hostpathplugin-0` 容器级 `hostpath` 与 `csi-external-health-monitor-controller` 都有 RESTART，同步 #376 hostpath flap 形态（liveness 9898 timeout/500）—— 双因合流（CSI 反复 flap 给 apiserver 加负载 → apiserver 短时不稳）。

整轮丢弃。残留 6 个 mariadb-vcluster pod（mariadb-{0,1,2} × 2 cluster）保留作 evidence 不主动 cleanup。

### Round 2-replacement（`mdb-vc396r2-192421`，19:24 起，**19:38 判 `env_freeze_uncounted`**）

走完整 7-layer env gate（CSI/11443/readyz 三向基线落定后才起），新 timestamp slot。
- T1-T5 + exporter + R4-smoke 全 PASS
- **CM4 期间** `csi-hostpathplugin-0` 撞 mariadb 同形态 hostpath flap：pod RESTARTS 0→3 over <30min（容器级 `hostpath` 和 `health-monitor` 都 restart）
- 此时 sidecars 仍 1/1，`/readyz` 仍 ok（**黄线**，未进 cascade）
- Runner 卡 `Waiting for cluster ... Running`，触发 freeze

整轮丢弃。又一个 cluster 残留。

### 19:38-19:41 cleanup 期间二次劣化

Jack 启动 cleanup 路径：
- 临时 11443 cleanup route 启动后 source API 仍 connection refused
- 立即随后 host CSI 进一步劣化：`csi-hostpathplugin-0` 4/4 RESTARTS=3 → **3/4 Error RESTARTS=5**，sidecars 进 ContainerCreating/Pending 风暴
- `/readyz` 仍 ok（这阶段 cascade 尚未完整成立）

Jack 停止 cleanup 保留 evidence。

### 跨阶段信号摘要（**mariadb 这一侧观察到的**）

| 阶段 | CSI plugin | host API | 备注 |
|---|---|---|---|
| Round 2（19:13）| 子容器 restart | `/readyz` await timeout、TLS handshake timeout、11443 listener fail | 双因合流 |
| Round 2-replacement (19:38) | 4/4 RESTARTS=3 | `/readyz` ok | 黄线 |
| Cleanup attempt (19:41) | **3/4 Error RESTARTS=5**, sidecars 风暴 | `/readyz` ok | cascade 已经成立但 readyz 滞后 |

### 关于"dual + 双污染"框架的客观说明

mariadb 在 19:13 / 19:38 两次冻结时**不是孤立运行**：westonnnn 04-29 00:11 框架解释里把当时整体 host load 称为 "valkey 3-parallel" 之后被精确化为"valkey 11 sub-suite sequential + cleanup gate 不到位 → 残留集群叠加 = 等效 dual + 污染累积"。具体 valkey 在 19:13 / 19:38 时的活跃度和残留量级**详见 valkey 团队（Alice）的 Round 1-2 timeline**，不在本附录复述。本附录只描述 mariadb 一侧观察。

cell-1 的 cascade 完整成立信号（CSI 3/4 Error + sidecars 风暴）首次出现是 19:41 cleanup 期间，**晚于** runner 冻结时点。这条对 chapter 5 判定流程有意义：runner 卡住 / OpsRequest timeout 这类 product-layer 信号可以**早于** CSI 观测层的 cascade 全套指标，所以判定流程不能只看 CSI 信号也要看 runner 行为。

### Evidence package

完整 freeze + post-cleanup evidence 持久化：
- 路径：`/Users/wei/.slock/agents/165cfb62-c3ae-4a63-a499-92e93b517569/artifacts/mdb-vc396r2-192421-task396-env-freeze-cleanup-0049.tar.gz`
- sha256: `1f6e61f5ae026e995642e2a7da3a4ba935a277f545dcdd1e74a403a552cb0f80`
- 包内含：`env-freeze.txt`（Round 2-replacement 冻结时态）、`csi-frame-pre-delete.txt` + `plugin-logs-pre-delete.log`（delete 前补抓的关键现场）、`mariadb-cleanup-delete.log`（cleanup trace）、`csi-frame-post-cleanup.txt` + `csi-frame-post-cleanup-followup.txt`（清完 env 状态 5/5 ok 三项指标）

辅助 anchor 帧：
- Jason `helen-bilateral-postcleanup-20260428T164459Z.txt`（valkey 已清 + mariadb 残留还在那个时点的 host 帧，作为 cleanup 自愈过程中点）
- Jason `preflight-mariadb-solo-20260429T032932Z.txt`（mariadb solo 起跑前 baseline）

<a name="appendix-d-cell-4-mariadb-solo"></a>
## 附录 D：cell-4 MariaDB single-load reference

> **验证环境**：KB 1.0.2 + chart `mariadb-1.1.1-alpha.11`（含 12 tests-only patches）。Host: Wei's Mac (Jason 管)。Oracle 团队跑在 Jeff's Mac，**不共享 host**。Sample: `mdb-vc396r1-134638`。
> **two-engine N=2 confirmation**：本 appendix 跟 [appendix B](#appendix-b-cell-3-valkey-solo) 一起证 "干净 single-load 不论引擎都不撞 cascade"。

### 起跑前置

mariadb cell-4 R1 起跑前的 host baseline（@Jason `preflight-mariadb-solo-20260429T032932Z.txt`）：
- `csi-hostpathplugin-0` 4/4 Running, RESTARTS=23 frozen（cleanup 后稳态）
- sidecars 1/1（health-monitor=19, hostpath=4, liveness/registrar=0），provisioner=1
- `/readyz` ok，host docker stats k3d 容器合计 ~187%（含 oracle-test 不同 cluster 的 docker stats，因为 westonnnn 12:47 修正：Oracle 不在我们 host，`oracle-test` cluster 是不同 Mac 上的，docker stats 表显示是因为 docker daemon socket 共享——Wei's Mac 实际只跑 `kb-local` cluster；这里数字不影响 cell-4 cleanliness 判断）

### Round 1 phase 结果

`bash mariadb/tests/async.sh` 单次完整 invocation，跑到 T12 触发 hard-freeze。最终 counts：**PASS=83 / FAIL=6 / SKIP=3 / WARN=4**。

| Phase | 结果 |
|---|---|
| T1-T5（init / exporter / R4 smoke） | ✅ PASS |
| T6 / T7（VScale）/ T8（Restart）/ T9 / T10 / T11 | ✅ PASS（T10 scale-out new pod 45s 内发布 secondary，120s deadline 内）|
| CM1 / CM2 / CM3 / CM4 reconfigure 矩阵 | ✅ PASS（CM4 30s 远内 90s deadline）|
| R3 scale-out | ✅ PASS（new pod 37s 内 secondary + 追上 95 rows）|
| chaos C1 / C2 / C3 | ✅ PASS |
| **chaos C4** rapid primary kills round-1 | ❌ FAIL — `bounded recovery observation remained indeterminate within 120s`，分类 **observation-class fail**：pod_count=2, primary=1, secondary=none, role_holes=22, route_holes=0, read_holes=0（数据健康，仅 role label 未在 bounded window 内 re-publish）|
| chaos C5 / C6 / C8 | ❌ FAIL — 同 observation-class（no-secondary / role-unpublished + route_holes=0/read_holes=0）|
| **T12** VolumeExpansion | ❌ FAIL — observation-class no-secondary，**同时**触发 `csi-hostpath-resizer-0` RESTARTS 0→1（"Lost connection to CSI driver, exiting"）→ Jack 按严格 gate-hygiene hard-freeze runner |
| C7 chaos / T13 LB / T14 version | SKIP — capability-aware（Chaos Mesh / LB / version-condition）|

### cell-4 question 答案：no host cascade ✅

freeze 时点（14:15:32）host 状态对照 cell-2 cascade 形态：

| 信号 | cell-2 cascade（valkey 多 suite 污染累积） | cell-4 (本轮 freeze 时刻) |
|---|---|---|
| `csi-hostpathplugin-0` | 3/4 CrashLoopBackOff | **4/4 Running**, RESTARTS=23 frozen |
| sidecars (provisioner / attacher / snapshotter / resizer / registrar) | ContainerCreating / Pending 风暴 | 1/1 Running，仅 resizer +1 单点 restart |
| `/readyz` | InternalError + `etcd-readiness failed` | **5×ok** |
| host k3d CPU peak | **1645%** | **256.75%** |

→ T12 csi-resizer 单一 restart **不是 cascade 形态**，是 VolumeExpansion 期间 sidecar reconnect class（log: "Lost connection to CSI driver, exiting"）；不构成 cell-4 question 否定证据。

### Hard-freeze 触发分类

按 chapter 6 严格 gate-hygiene 规则，**任何 RESTARTS+ 都算 drift → freeze**。本轮按规则触发 freeze，但 evidence 上 cascade shape absent。所以 cell-4 evidence 是：

> "干净 mariadb single-load 期间观察到一次 csi-resizer single restart drift 但**无 cascade 形态**；T1-T11 / CM1-4 / R3 / 早期 chaos 都过；chaos 后期出现的是 observation-class fails（route_holes/read_holes=0 → 数据健康，仅 role label 节奏问题），跟 cascade 主题独立。"

这条结论**不要求**整轮 PASS / 0 fail；只要求 host cascade shape 不出现。✅ 满足。

### Observation-class fails 是独立 framing layer（不属本 appendix scope）

C4/C5/C6/C8/T12 五个 phase 都撞 role-unpublished/no-secondary observation gap，比之前 Phase 2ff Round 1（仅 C6/C8）范围扩大。这个 pattern 是独立调研主题（"post-chaos role-publish observation gap"），跟 cell-4 cascade question 解耦：
- 这套 fail 的 route_holes/read_holes=0 → 数据路径都健康
- 没触发 host cascade signal
- 是 KB controller role label publish 节奏与测试 bounded-wait window 之间的契约问题，应该单开 follow-up 调研

### Evidence package

完整 freeze + post-stop evidence 持久化：
- freeze dir: `/tmp/mdb-vc396r1-134638-task396-round1-csi-resizer-redline-freeze-141532/`
- tar: `/Users/wei/.slock/agents/165cfb62-c3ae-4a63-a499-92e93b517569/artifacts/mdb-vc396r1-134638-task396-cell4-csi-resizer-freeze-141532.tar.gz`
- sha256: `3085a16456b0812fe94be6e0e25fbc682ad323bb08de85c76701a47cc06f999c`
- attachment: `32adba89-f8ce-4ed1-8a08-30b1e1ae5bf1`
- 包内含：runner state、source/host CSI / `/readyz` / docker stats freeze 帧、T12 resizer pod describe + 容器 log、phase 分类汇总、host mirror 资源 inventory（3 pods + 3 PVCs，未清理保留作 evidence）

### 跨引擎对照（two-engine N=2 confirmation）

cell-4 mariadb single-load no cascade ↔ [appendix B](#appendix-b-cell-3-valkey-solo) cell-3 valkey single-load no cascade。两个引擎独立验证 → cascade 不依赖具体引擎 workload，是 "(dual-load 或 single-load) + 污染累积/反应过载" 触发的。

## 附录 E：MariaDB hot-fix #376 N=2 variance 完整重写

待 cell-4 evidence 齐全后由 Helen 主笔。

## evidence-discipline cross-references

- 关于"valkey 3-parallel 笼统化" framing 错误（Helen evidence-discipline 反例集），收作 chapter 1-2 的方法论引证
- "stress 模型 = 残留累积 / 真并行 / 干净 single-suite"是质上不同现象的 framing 教训
