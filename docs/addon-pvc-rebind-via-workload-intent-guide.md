# Addon PVC Rebind via Workload Intent 指南

本文面向 Addon 开发与测试工程师，聚焦一个在 OpsRequest 控制器和 Workload 控制器同时存在的体系里几乎一定会遇到的问题：**当某个 OpsRequest 想把同一个 PVC 名字下的存储从一块旧 PV 改绑到一块预先准备好的新 PV，而 Workload 控制器又会按模板自动重建丢失的 PVC，两边会抢同一个 PVC 名字的所有权。**

正文写通用方法论，不绑定某一个引擎、不绑定某一类 Ops；引擎相关的现场、命令、案例放在附录。

## 先用白话理解这篇文档

KubeBlocks 的实例（Pod）的存储一般是 Workload 控制器用 `volumeClaimTemplates` 模板生成的。Workload 控制器有一个隐含合同：**只要这个实例还在 desired set 里，相对应的 PVC 就必须存在**。如果 PVC 被外部删了，Workload 控制器会立刻按模板再造一个 PVC（没有 `spec.volumeName`），让动态 provisioner 给它分配一块新的空 PV。

很多 Ops（最典型的是从备份重建实例 `RebuildInstance`）需要在某个时刻把这个 PVC 从原来的 PV 改绑到一块新 PV（备份恢复出来的 helper PV）。"改绑"在 Kubernetes 层只能通过"删除原 PVC + 创建一个 `spec.volumeName=helperPV` 的同名新 PVC"实现，因为 `spec.volumeName` 是 immutable 字段。

这就在两边之间制造了一个明显的所有权窗口：

- **OpsRequest 控制器**：删原 PVC → 想立刻创建新 PVC（`spec.volumeName=helperPV`）。
- **Workload 控制器**：发现 PVC 不见了 → 按模板创建 PVC（无 `spec.volumeName`）。
- **动态 provisioner**：看见 Pending PVC 没有 `spec.volumeName` → 给它分配一块新的空 PV。

如果 Workload 控制器和动态 provisioner 抢在 OpsRequest 控制器的"创建带 `spec.volumeName` 的新 PVC"动作之前完成，那么用户拿到的是一个 `Bound` 但绑错 PV 的 PVC：助手 PV（含恢复出来的备份数据）孤立在一边没人引用，新 PVC 绑的却是一块全新的空盘。**OpsRequest 可能仍然 `Succeed`（控制器看到 PVC `Bound`）但事实上数据没恢复，业务层会感觉到"备份失效"。**

更糟的是，如果在这条 race 的某个瞬间 OpsRequest 控制器走 `cli.Get(ObjectKey{Name: sourcePvc.Spec.VolumeName})` 而 `sourcePvc.Spec.VolumeName == ""`（PVC 被 Workload 控制器刚刚重建、动态 provisioner 还没分配 PV 的过渡态），调用会以 `PersistentVolume "" not found` 的形式 fatal 出来。

这篇文档解决的不是"避免某个 race"，而是 — **当一个 Ops 必须改绑 PVC 时，如何把 PVC 所有权完整、唯一地交给一个写者，让另一个控制器变成读者。**

## 适用场景

当你在做下面任一件事时，本文适用：

- 设计或验证一个新的 Ops 能力，且这个 Ops 需要修改某个 PVC 的 `spec.volumeName` 或重建 PVC（rebuild、backup-restore-into-place、PV migration、storage class change 等）。
- 现场观察到一个 Ops 报 `PersistentVolume "" not found`、`source pvc not found`、`pvc-protection` 持续不释放、或 PVC 最终绑到一块陌生 PV 上的现象。
- 写 ship-readiness 的 chaos 矩阵，需要补一类"PVC 所有权抢夺"的故障注入。

## 核心边界：唯一写者原则

任何一个 PVC 的 `spec.volumeName` 在生命周期里应该有且只有一个写者。在 KubeBlocks 体系里，这个写者只能是 Workload 控制器：它从模板生成 PVC、它会因为 PVC 缺失而再生 PVC，它对 PVC 的形状有最完整的视图。**OpsRequest 控制器不应该和它抢这个写者位置。**

如果一个 Ops 需要让 Workload 控制器创建一个不同 `spec.volumeName` 的 PVC，唯一可持续的做法是：**通过一种 Workload 控制器能读、能识别、能消费的渠道告诉它"这一次创建请用这个 helper PV"，而不是 OpsRequest 自己跳过去帮它创建。**

我们把这个渠道称为 **Workload Intent（工作负载意图）**。它的核心契约：

1. **位置**：写在 Workload CR 的 annotation 上（不是 Cluster、不是 Component），这样 Workload 控制器自然会消费。
2. **内容**：一段结构化数据，标识哪个实例、哪个 claim template、用哪块 helper PV、以及发起这条 intent 的 OpsRequest 身份（用于并发冲突识别）。
3. **写者**：OpsRequest 控制器在一次 Ops 内写入和清理。
4. **读者**：Workload 控制器，在两个地方消费：
   - 创建 PVC 时把 `spec.volumeName` 设成 intent 里的 helper PV。
   - 调和环节里识别"PVC 当前绑的不是 helper PV"，主动释放（按 pod → PVC 的安全顺序），让 PVC 创建路径接手。

## 设计契约（无论实现引擎）

下面这组不变量决定了 Workload Intent 模式是否安全。引擎相关的实现是后面的事，先把契约定死。

### 不变量 1：Workload 控制器是 PVC `spec.volumeName` 的唯一写者

OpsRequest 控制器在整个 Ops 周期内，**任何一个时刻都不**直接 `cli.Create` 或 `cli.Update` 那个目标 PVC。它只写 helper PV 自己（reclaim policy、ClaimRef）和 Workload CR 的 annotation。

这条不变量是 race 消除的根。没有它，所有其它防护都是补丁。

### 不变量 2：每条 intent 携带 OpsRequest 身份

Intent 数据结构里至少含 `opsUID`（最稳定的身份标识）和 `opsName`（人读用）。两个用途：

- **写时拒绝**：如果某条 (instance, claim) 上已经存在 intent 且 `opsUID` 不匹配，新一条 OpsRequest 写入时被拒（"已被另一条 OpsRequest 占用"）。
- **清理时拒绝**：清理某条 intent 必须传入和 stored `opsUID` 完全相等的 `opsUID`，避免 OpsRequest A 误清 OpsRequest B 的 intent。

`opsUID` 不能为空 — 一条匿名 intent 等于失去并发保护，必须在写入时硬性拒绝空值。

### 不变量 3：可重入

OpsRequest 控制器的状态机必须能在任何步骤被打断后从 CR 上的可观测状态恢复。具体到 PVC 改绑流程：

- 是否已经把 helper PV 标成 `Retain`：从 PV 自身的 reclaim policy 直接读。
- 是否已经记录原 reclaim policy：从 helper PV 的 annotation 直接读。
- 是否已经清理 tmp PVC：直接 `cli.Get` tmp PVC 看是否存在。
- 是否已经写 intent：从 Workload CR 的 annotation 直接读。
- 是否已经达到 bind 终态：从 source PVC 的 `spec.volumeName` 和 `status.phase` 直接读。

**任何"在内存里维护 in-flight 步骤"的设计都不可重入**；控制器一旦 crash，状态丢了就没法回来。

### 不变量 4：原 reclaim policy 可被推断

OpsRequest 必须能把 helper PV 在 Ops 终态恢复到"用户原本想要的 reclaim policy"。但是原 source PV 在某些时刻可能已经不存在（Ops 进行中被释放掉），所以单纯依赖"读原 PV 的 reclaim policy"会在 race 窗口里 fatal。

实际能用的推断顺序（四优先级决策树）：

1. **P1**：source PVC 当前 `spec.volumeName` 指向的 PV 的 reclaim policy。这是用户当时让 KubeBlocks 给它的盘的 reclaim policy。如果该 PV 仍存在，直接读；如果 NotFound（不是其它错误），fall through。
2. **P2**：source PVC 上写的 `storageClassName` 对应 StorageClass 的 reclaim policy（如果 StorageClass 自身没显式设，按 K8s defaulting 视为 `Delete`）。
3. **P3**：tmp PVC 上写的 `storageClassName` 对应 StorageClass 的 reclaim policy（同样套用 K8s defaulting）。tmp PVC 在 OpsRequest 自己控制下，至少 ReclaimPolicy 来源是稳的。
4. **P4**：上面三步都拿不到 → 显式 fatal。**不要**默认成 `Delete`，那等于把"用户希望保留"和"用户希望删"两种意图静默合并。

这条决策树的关键属性：**确定性**。同一个 cluster 状态、同一条 OpsRequest 推它两次必须得到同一个答案，否则 OpsRequest 在 retry 时可能误把 `Retain` 改写成 `Delete`。

### 不变量 5：清理顺序原子

Ops 收尾必须按以下严格顺序：

1. 等到 source PVC `spec.volumeName == helperPV.Name` 且 `status.phase == Bound`。
2. 把 helper PV 的 reclaim policy 从 `Retain` 改回原值（用步 4 的 P1-P4 推断结果）。
3. **才**清理 Workload CR 上的 intent annotation 条目。

颠倒 2 和 3 会出现下面这种状态：intent 已清，但 helper PV 还是 `Retain`，且操作员/用户那边没有任何 ops-side 记录提醒"这块 PV 需要改回 reclaim policy"。helper PV 永久滞留。

## 实施方法（kubebuilderx 引擎相关，但口径通用）

下面这套实施口径是基于 KubeBlocks 的 kubebuilderx-style 多阶段 reconciler 链给出的，但核心思路（intent 写者 / intent 读者 / pod 创建 gate / 单 commit 安全）在任何控制器框架里都能映射。

### 1. annotation schema

在 Workload CR（如 InstanceSet）的 annotation 里挂一段 JSON：

```yaml
metadata:
  annotations:
    # 命名空间用 Ops 的 group，避免和 workload 自身的 annotation 冲突。
    operations.kubeblocks.io/rebuild-instance-pvc-overrides: |
      {
        "<instance-pod-name>": {
          "<volume-claim-template-name>": {
            "pvName":  "<helper-pv-name>",
            "opsUID":  "<ops-request-uid>",
            "opsName": "<ops-request-name>"
          }
        }
      }
```

外层按实例名分组，内层按 claim template 名分组。同一个实例可以有多个 claim（data + journal），同一个 OpsRequest 可以一次 cover 多条 claim。

### 2. Workload reconciler 链中的两个钩子

**钩子 A — PVC 创建路径（buildInstancePVCByTemplate 一类）**：

每次按 claim template 生成 PVC 之前，从 Workload CR 上 parse 这条 annotation。如果当前 (instance, claim) 命中一条 entry，把生成的 PVC `spec.volumeName` 设成 entry 里的 `pvName`，然后 `tree.Add` 提交。其它分支（无 entry / entry pvName 为空 / annotation 解析失败）必须**显式失败**（参见"fail-closed"小节），不要 fall back 到默认 PVC 创建路径。

**钩子 B — 拓扑收敛 reconciler**：

在 alignment reconciler 之前的位置加一个新的 reconciler。它的 PreCondition 是"Workload CR 上存在这条 annotation"。它的 Reconcile 走以下决策：

- 对每个 (instance, claim) entry：
  - 找出预期 PVC 名字（用 `instancetemplate.ComposePVCName` 一类的 helper）。
  - 如果 PVC 不存在 → 不处理（等下一个 reconciler 的 PVC 创建路径）。
  - 如果 PVC 存在且 `spec.volumeName == helperPVName` → 已收敛，跳过。
  - 如果 PVC 存在但 `spec.volumeName` 是别的（旧 PV 或动态 provisioner 给的空 PV）→ 进入释放序列：
    - 如果 pod 还在且没有 `DeletionTimestamp` → `tree.Delete(pod)`，标记"本轮发出过 release 动作"。
    - 如果 pod 还在但已经是 terminating → 不再 delete pod / PVC，标记"本轮等待 in-flight termination"。
    - 如果 pod 已经不存在但 PVC 还在 → `tree.Delete(pvc)`，同样标记。
    - 如果 PVC 已经是 terminating → 等着，标记。
- **任何一个 entry 标记了 release / 等待**，整个 reconciler 必须返回 `RetryAfter`，让 plan builder **只 commit 这一轮的 delete 动作**，下一轮 reconcile 才让 alignment 看到一个 clean missing-PVC 状态。

如果在同一个 commit 里既 `tree.Delete(pvc)` 又让 alignment `tree.Add(pvc)` 同名的新 PVC，kubebuilderx 的 plan builder 会把这俩动作 fold 成一个 `Update` vertex，撞 `spec.volumeName` 的 immutable 字段，整轮 reconcile fail。**这是这个模式里最容易踩的坑**，所以不变量记成"任何 release 都要 RetryAfter，不要 Continue"。

**钩子 C — pod 创建 gate（在 alignment reconciler 内部）**：

在 alignment 走到 `tree.Add(newPod)` 那一步之前，先调一个 `AllRebuildIntentClaimsBound(workload, instanceName, pvcByName, template)`：

- 如果 annotation 上没有这个 instance 的 entry → 返回 `(true, nil)`，alignment 正常创建 pod。
- 如果有 entry 且每个 claim 都已 `spec.volumeName == helperPVName && status.phase == Bound` → 返回 `(true, nil)`。
- 如果有 entry 但任何一个 claim 还没收敛到上面那个状态 → 返回 `(false, nil)`，alignment 跳过这个 instance 的 pod 创建（PVC 创建仍要走，让 binding 有机会推进）。
- 如果 annotation 解析失败或 entry 引用了一个 template 里没有的 claim → 返回 `(false, error)`，整轮 reconcile abort。

这个 gate 是多 PVC 实例的原子保障：一个 pod 不会带着"data 已绑 helper / journal 还在旧 PV"的混合状态启动。

### 3. OpsRequest 控制器侧的状态机

```
1. （备份恢复阶段）让 helper PV 通过 tmp PVC 拿到恢复后的数据 — 不在本文范围。
2. 把 helper PV 的 reclaim policy 改成 Retain，并把"原 reclaim policy"写到 helper PV 的 annotation。
   原 reclaim policy 用四优先级决策树推断（不变量 4）。
3. cleanup tmp PVC（让 helper PV 进入 Released 状态）。
4. 清理 helper PV 的 stale ClaimRef — 但条件清：
     - ClaimRef 仍指向 tmp PVC（按 namespace + name 比对）→ 清成 nil；
     - ClaimRef 已指向 source PVC → 不要动，binding 已经收敛；
     - ClaimRef 指向第三方对象 → 视为编程错误，fatal。
5. 在 Workload CR 上 merge intent annotation。Merge 函数硬性拒绝：
     - 空 opsUID（违反不变量 2）；
     - 空 pvName（违反 fail-closed）；
     - 已存在的 entry 与 incoming opsUID 不一致（并发冲突）。
6. 等 source PVC `spec.volumeName == helperPV.Name && status.phase == Bound`。
   等不到就返回 NeedWaiting，下一轮 OpsRequest 重新评估。
7. 把 helper PV reclaim policy 改回原值。
8. 清理 intent annotation 上对应的条目。Delete 函数硬性要求：
     - 调用方传入的 opsUID 非空；
     - stored opsUID 非空；
     - 两者完全相等。
   不允许通过空 opsUID 清空任何条目，避免误清。
```

注意 OpsRequest 控制器**不**主动 `BackgroundDeleteObject(targetPod)`。Pod 的删除在钩子 B 里由 Workload reconciler 自己干，OpsRequest 只写 intent + 等结果。

## fail-closed 总览

annotation 是这条 race fix 的核心入口，所以每个消费它的位置都必须 **失败时 abort 整轮 reconcile，绝不静默跳过**。具体：

| 入口 | 失败模式 | 期望行为 |
|---|---|---|
| Workload CR Parse(annotation) | JSON 损坏 | 返回 error；buildPVC / 钩子 B / pod gate 都要把这个 error 一路冒到上层 |
| Workload CR Merge(intent) | 空 opsUID / 空 pvName | OpsRequest 写入失败 |
| Workload CR Merge(intent) | 已存在 entry 的 opsUID 与 incoming 不匹配 | OpsRequest 写入失败 |
| Workload CR Delete(intent) | 调用方传入 opsUID 为空 | 拒绝清理 |
| Workload CR Delete(intent) | stored opsUID 为空 | 拒绝清理（防御老数据） |
| Workload CR Delete(intent) | 调用方 opsUID 与 stored 不等 | 拒绝清理 |
| buildPVC 路径 | annotation 上有 entry 且 pvName 空 | abort 创建（不要静默 fallback 成普通 PVC） |
| 钩子 B 路径 | claim 名不在 instance template 的 VolumeClaimTemplates 里 | abort（编程错误，不要无限 gate pod） |
| Pod gate | annotation parse 失败 / claim 名不在 template | abort 整轮 reconcile |

任何一个静默跳过都等于把"这条 race 没生效"和"没有 race"混淆。

## 验证口径

针对这个模式，建议 acceptance 至少包含三类样本：

### 类 1 — 独立 baseline acceptance

按 N=10 跑独立轮次：每轮新建一个干净 cluster + 写入一份 baseline 数据 + 制造一个目标实例 disruption（删 pod + 删 PVC）+ 发起 Ops + 等终态 + 验证数据全恢复 + topology flat。

每轮额外采集（这是关键）：

- KubeBlocks 控制器日志中 `PersistentVolume "" not found` 出现次数（期望 0）。
- 目标 source PVC 最终 `spec.volumeName` 是否绑到 helper PV（用 PV 上的 ops-side label / annotation 校验）。
- helper PV 的 reclaim policy 是否恢复到推断出的原值。
- Workload CR 上 intent annotation 是否被清空。

### 类 2 — 同 cluster 密集场景

在同一个 cluster 上轮换目标实例连续发起多次 Ops（5 / 10 / 20 ops 一组），验证"前一次 Ops 残留状态"不会让后一次 flaky。每条 Ops 都跑同一组验证。

### 类 3 — 控制器重启 chaos gate

在 OpsRequest 进行到"intent annotation 已清 + 新 pod 还未 Ready"的窗口里，主动 `kubectl rollout restart` KubeBlocks 控制器，验证：

- 重启不会让 intent annotation 复活；
- source PVC 维持绑定 helper PV；
- helper PV reclaim policy 不被反复改写；
- 新 pod 最终 Ready，数据可读。

触发依据必须是**实时观测 Workload CR 上的 annotation transition `present → absent`** 并且新 pod `Ready=False`，**不**依赖控制器日志关键字 — annotation 状态是设计契约本身的语义，日志只是观测信号。

## 常见坑

- **以为同一 commit 里 delete + add 同名对象会变成 delete+create**：kubebuilderx plan builder 是 oldTree vs final desiredTree diff，同名 = update，撞 immutable 字段失败。释放动作必须独占一个 commit。
- **以为 annotation 没有 entry 就是"不需要 gate"**：parse 失败也会进 fallback 路径；要把 parse 失败和 "无 entry" 显式分开。
- **以为 helper PV 的 ClaimRef 总该清掉**：binding 收敛后 ClaimRef 会被 K8s PV binder 重新写成指向 source PVC；这时候清 ClaimRef 等于把刚收敛的 binding 撤销。
- **以为 source PVC 永远存在**：在 Workload reconciler 释放动作进行中，source PVC 短暂消失。所有读取 source PVC 的代码必须把 NotFound 当作合法过渡态，不阻塞 intent 写入。
- **以为 4-priority reclaim 推断可以默认成 Delete**：默认成 `Delete` 等于把"用户希望保留"和"用户希望删"两种意图静默合并；P4 必须 fatal。

## 案例附录

引擎相关的现场放在独立的 case 文件里。例如：

- `cases/valkey/valkey-rebuild-instance-pvc-ownership-race-case.md`：在 Valkey addon 上观察到的 `PersistentVolume "" not found` 现场和修复 evidence（49 ops e2e 验证）。

---

写作建议：当你的 addon 引入或修改一个会改变 PVC `spec.volumeName` 的 Ops 时，把这个文档当 checklist 走一遍：5 条不变量是否都满足，3 个钩子是否都接上，fail-closed 表格是否每条都覆盖。漏一条都可能在某个时机暴露成 production race。
