# Addon 测试 / 运维中的 Narrow-Scope Force-Delete 指南

> **Audience**: addon dev / test / ops
> **Status**: stable
> **Applies to**: any KB addon
> **Applies to KB version**: 1.0.x / 1.1.x release tags（finalizer chain 模式在两版本都验证过；KB main / 1.2 待重测）
> **Affected by version skew**: KB 1.x finalizer 链路结构（cluster ← component ← instanceset ← pod）；如果 1.2 finalizer 模型变化，本文判定流程仍适用，但回归动作清单需要重测

本文面向 Addon 测试与运维工程师。当某个 pod 在 image pull 阶段被 helm uninstall / kubectl delete cluster，会引发 instanceset → component → cluster → CmpD 的反向 finalizer 死锁。直接 patch finalizer 是危险的；**正确的窄域 recovery 是 force-delete 那一个 stuck pod，让 KB 控制器接管后续清理**。本文给安全前提、标准恢复流程、反 anti-pattern 三块。

## 先用白话理解这篇文档

最容易出问题的两步：

- **先错的那一步**：在 pod 还在 `Pulling image` 的时候 helm uninstall cluster / 删 chart-managed CmpD。kubelet 没法 cancel 进行中的 image pull → pod 永不死 → 上面四级 finalizer 全部锁死。
- **再错的那一步**：直接 `kubectl patch` 清 finalizer 强行解锁。看起来对象删掉了，其实 KB controller 失去清理 PVC / 更新 OwnerReference 的机会，留下孤儿资源。

正确做法是**只 force-delete 那一个 stuck pod**，不动任何 finalizer，剩下的让 KB 控制器自己级联清理。前提是先做几项安全核查：确认 pod 从未真正跑过任何容器、PVC 没被实际写入。这些条件不全满足就**不要 force-delete**。

## 适用现象

helm uninstall cluster 或 kubectl delete cluster 之后：

- `cluster` STATUS=`Deleting`，metadata 里仍有 `cluster.kubeblocks.io/finalizer`
- `component` STATUS=`Deleting`
- `instanceset` 还在
- `pod` STATUS=`Terminating`，**Container ID 为空**
- `kubectl describe pod` 里只有 `Pulling image ...` 这一条 event
- 如果之前删过 chart-managed CmpD，CmpD 也在 `Deleting` 状态，仍有 `componentdefinition.kubeblocks.io/finalizer`
- `kubectl -n kb-system logs deploy/kubeblocks` 高频出现 `wait for the workloads to be deleted`

整个反向阻塞链：**CmpD ← cluster ← component ← instanceset ← pod ← kubelet（卡在 image pull）**

## 为什么会卡住

1. `cluster` finalizer 等 component 被删
2. `component` controller 等 instanceset 被删
3. `instanceset` controller 等它管的 pod 被删
4. `pod` 已经有 `deletionTimestamp`，但 kubelet 还在 image pull，没法转入 cleanup（kubelet 不能 cancel 进行中的 layer pull，containerd 内部锁）
5. pod 永不死 → instanceset 永不清 → component 永不清 → cluster 永不清 → CmpD 永不清

这不是 KB 的 bug，是 kubelet + containerd 在「pulling 状态 + pod deletion」组合下的众所周知坑。

## 安全前提（force-delete 之前必查）

force-delete pod 是绕过 grace period 的破坏性操作，**只在以下条件全部满足时才安全**：

```bash
# A) Container ID 真的是空（pod 从未跑过任何容器）
kubectl -n <ns> get pod <pod> -o jsonpath='{.status.containerStatuses[*].containerID}{"\n"}'
kubectl -n <ns> get pod <pod> -o jsonpath='{.status.initContainerStatuses[*].containerID}{"\n"}'

# B) 节点 crictl 也确认没有任何相关容器
for node in $(k3d node list --no-headers | awk '{print $1}'); do
  docker exec "$node" crictl ps -a | grep <pod-name-prefix>
done

# C) 该 pod 是 stateful workload 但还没启动过任何写操作（DB 没初始化、PVC 是空的）
kubectl -n <ns> get pvc -o yaml | grep -E "phase|capacity"
```

A + B 都为空，且 C 确认 PVC 还没被实际写入 → force-delete pod 零数据风险。

## 标准恢复流程

```bash
# 仅删那一个 stuck pod，不动任何 finalizer
kubectl -n <ns> delete pod <pod> --grace-period=0 --force

# 然后什么都不做，等 30s 内反向链自动级联：
# pod gone → instanceset gone → component gone → cluster gone → CmpD finalizer 释放 → CmpD gone

# 验证
kubectl get cluster -n <ns>
kubectl get componentdefinition <cmpd-name>  # 应该 NotFound
```

**不要做**：

- 不要 `kubectl patch ... -p '{"metadata":{"finalizers":null}}'` — 直接清 finalizer 会让 KB controller 失去机会做 PVC 清理 / OwnerReference 更新，留下孤儿资源
- 不要 `kubectl delete --force` 整批 cluster / component / instanceset — 同上

## 反 anti-pattern：清理顺序

「helm uninstall cluster + 立刻删 chart-managed CmpD」是触发本死锁的常见操作。**正确顺序**：

1. helm uninstall cluster
2. **等** cluster 真的被删完（`kubectl get cluster -n <ns>` 返回 NotFound）
3. 然后才能 helm uninstall addon / 删 keep-policy CmpD
4. 中间任何卡顿都先 read-only 取证，不要急于追加删除命令

如果非要并行清理，至少要保证：cluster 引用的所有 CmpD 在 cluster 完全消失之前都还存在（即使有 deletionTimestamp 也行，KB controller 还能用）。

## 边界声明

本文给的 force-delete 是 **narrow-scope 一次性 recovery**，**不是默认自服务**：

- 任何「不确定 pod 是否真没数据」的情况 → 不 force-delete，先取证
- 任何「pod 正在写 PVC」的状态 → 绝对不 force-delete
- 任何「不是测试环境」的现场 → 团队 owner 决策，不 unilateral 执行

## 验证清单

- force-delete 前确认了 Container ID 为空 + crictl 无容器
- force-delete 后 30s 内五级资源全部消失
- kb-system controller 日志里 `wait for the workloads to be deleted` 停止刷新
- PVC 也都进入 Released / Deleted（不是 Terminating 卡住）

## 与其他文档的关系

- `addon-test-environment-gate-hygiene-guide.md` 第 6 项「fresh slot / no-tail 层」要求测试前现场干净；本文是当上一轮 cleanup 没收口时如何 recover。
- `addon-test-host-stress-and-pollution-accumulation-guide.md` 讲 host 层污染累积；本文是其中一种 cleanup gate failure 的具体修法。
- `addon-k3d-image-import-multiarch-workaround-guide.md` 处理「image pull 不动」根因；本文处理「image pull 中途被 cancel」的善后。

## 案例附录

### 2026-04-29 oracle smoke：T02 cleanup 触发五级死锁

- 触发：测试工程师在 oracle pod 处于 `Pulling apecloud/oracle:12.2.0.1-ee`（~5GB image）状态时执行 `helm uninstall ora-smk-37821 + helm uninstall oracle + kubectl delete CmpD`
- 现象：cluster `Deleting` 仍有 finalizer / component `Deleting` / instanceset 仍在 / pod `Terminating` ContainerID 空 / CmpD `Deleting` 仍有 finalizer
- 取证：`crictl ps -a` 在 server-0 和 agent-0 都没有 oracle 容器 → pod 从未运行过 → 零数据风险
- 恢复：`kubectl -n oracle-test delete pod ora-smk-37821-oracle-0 --grace-period=0 --force` → 30s 内五级级联释放
- 教训：以后测试清理流程要 sequencing — 等 cluster 真的没了再动 CmpD
