# Addon Controller Crash Resilience 指南

> **Audience**: addon dev / test / TL，需要验证 addon 在 KubeBlocks 控制器中段失效后的恢复语义
> **Status**: stable
> **Applies to**: any KB addon（控制层 crash 是 K8s 通用属性，不绑定单引擎）
> **Applies to KB version**: any（控制器 crash-recover 语义在 KB 各版本下行为一致；本文方法论 version-agnostic）
> **Affected by version skew**: 不受 KB 版本影响 — 控制器 crash 后从 CR desired state 恢复是 K8s controller 通用契约

本文面向 Addon 开发与测试工程师，聚焦一个被频繁默认而很少被显式验证的属性：**当 KubeBlocks 控制器在一个 Ops 进行到一半时被 crash / SIGKILL，重新拉起后，相关 OpsRequest 与 Cluster 是否能继续推进到正确终态。**

正文只写通用方法论，不绑定某一个引擎或某一类 Ops。引擎相关的现场和案例放在附录。

## 先用白话理解这篇文档

控制器在生产里会因为各种原因被打断 — OOM、节点重启、滚动升级、被运维 SIGKILL。一个健康的 Addon 应该有这样的属性：

- 控制器**就算挂在 Ops 中间**，重新启动后也能从 CR 上的 desired state 接着干，不需要人工补救。

但这个属性有两个常见误判：

1. **"我的 Cluster 跑过 chaos 就算可靠了"** — chaos 通常打的是数据层 pod（kill master / 删 replica）。控制层 crash 是另一类故障，必须独立测。
2. **"重启后 Cluster 仍然是 Running 就 OK"** — 这只验了"一段时间后系统稳定下来"。它没验**那条 in-flight 的 OpsRequest 有没有正确收尾、有没有被静默跳过、有没有被双重执行**。

这篇文档解决的不是"如何让控制器不挂"，而是 — 假设它会挂，**如何按一组明确的口径验证 Addon 在控制器 crash 后仍然行为正确**。

## 适用场景

当你在做下面任一件事时，本文适用：

- 设计或验证一个新的 Ops 能力（restart / reconfigure / scale / switchover / backup / restore 等）。
- 准备 ship 一个 addon，需要回答"控制器挂了之后这个 addon 还能自愈吗"。
- 现场观察到控制器在 Ops 中段被重启过，需要判断 Ops 的最终结果是不是可信。
- 写 ship-readiness 的 chaos 矩阵，需要补一类"控制层故障"。

## 核心边界：控制层故障 vs 数据层故障

测试 chaos 时把这两类故障显式分开：

| 类型 | 打谁 | 失败的语义 |
|---|---|---|
| 数据层故障 | DB pod / replica / sidecar | 业务进程异常、数据可访问性、复制延迟 |
| 控制层故障 | KB controller / addon controller | desired state 推进中断、CR 状态机推进卡住 |

很多团队 chaos 主要测前者，把后者默认为"基础设施可靠"。但在生产里**控制器被 OOM / preempt / 滚动升级 SIGKILL 的频率远比 DB pod 主动挂的频率高**，把它一起测才完整。

## desired state 在 CR 上的几个隐含语义

Addon 控制器是否 crash-resilient，本质上是看下面几个问题的答案：

1. **OpsRequest 的 desired state 是否完整地写在 CR 上**：是不是控制器在内存里维护了一段进度信息，crash 之后丢掉就不可恢复？
2. **Cluster 的 desired state 是不是幂等的**：重新 reconcile 是从同一个 desired state 出发，还是依赖 in-memory 的中间结果？
3. **副作用是不是有去重 / idempotency key**：例如某个 Ops 触发了一次 backup job、一次 secret 更新、一次外部 webhook，controller 重启后会不会再触发一次？
4. **Ops 之间的并发是否依赖 in-memory lock**：如果是，crash 后 lock 丢失，可能允许两条本来不应共存的 Ops 同时跑。

这四个问题任何一个回答"否"，crash resilience 都是脆弱的，必须显式验证。

## 测试方法

测试控制器 crash 中段恢复，建议固定下面这套工序：

### 1. 选一个会跨多步的 Ops

不要选那种 "下发即终态" 的 Ops（比如纯 status patch 类的），选一个有以下特征的：

- 跨多个 reconcile 周期（例如滚动重启每台 pod 间隔几十秒）。
- 有副作用（创建 / 删除 pod、写 Secret、触发 backup）。
- 有 detectable 的中间状态（progressDetails / phase / conditions 在不同阶段不一样）。

这种 Ops 才能暴露 in-memory 状态依赖。

### 2. 在中段触发 controller crash

不要等 Ops 已经走完最后一步再 crash —— 那是 trivial case。要在 progressDetails 显示"还在做"的时候打。

常用做法：

- 起 Ops，poll 直到 `status.phase` 进入推进阶段。
- 直接 `kubectl delete pod -n kb-system kubeblocks-controller-... --grace-period=0 --force`，或者更激进地 SIGKILL controller 进程。
- 等 controller pod 重新 Running。

如果 Ops 跨度很短，可以人为放慢一步（比如调 reconcile interval / 加 sleep hook），保证 crash 时机落在中段。

### 3. 等收敛，但**不只看终态**

收敛后做四件事，不能只看 Cluster Running：

- **OpsRequest 必须是 Succeed**：phase=Succeed 且 endTimestamp 已写入。如果停在 Running / Pending，说明 controller 没接上。
- **副作用没有被双重执行**：检查涉及的 pod 数 / Secret 内容 / 外部 job 数量是否符合 desired state，而不是 desired state × 2。
- **没有遗漏步骤**：例如 rolling restart Ops，所有目标 pod 都应该已经被滚过一次，不能漏掉一台。
- **状态机历史没有断点**：conditions / events 应该形成一条连贯的 timeline，而不是有大段空白后突然跳到 Succeed。

### 4. 重复多次

controller crash resilience 受到时机敏感性影响很大。建议同一个 Ops 类型 crash 中段至少 N=3 次，且 crash 落在 Ops 不同阶段（前段 / 中段 / 末段），覆盖度才够。

## 验证口径（每次回报固定收集）

跑完一轮 controller crash chaos，固定收下面这些：

1. Ops 类型与 spec。
2. crash 触发时间点（相对 Ops 起始）。
3. crash 时 OpsRequest 的 phase 与 progressDetails 快照。
4. controller 重启耗时。
5. controller 重启后 OpsRequest 终态、endTimestamp。
6. 副作用计数（pod 数 / Secret 版本号 / job 数）与 desired 是否一致。
7. Cluster 终态（phase / pod readiness / role 拓扑）。
8. controller 日志中有没有"resume" 类的关键词，或者明显的 panic / restart trace。

## 常见误判

### 把"Cluster 还活着"当作通过

Cluster 在 controller 重启前已经在 Running，重启后大概率还在 Running，这个事实不证明 Ops 自身正确收尾。判断标准是 OpsRequest 的终态。

### 把"Ops 终态是 Failed"当作产品 bug

Ops 在 controller crash 后给出 Failed 是合理结果之一 — 只要 Cluster 能回到 Running 且没有副作用堆积，"Ops Failed + 系统自愈" 比 "Ops 卡 Running 不动" 更可接受。把 Failed 当 bug 容易导致团队改控制器去强行返回 Succeed，反而引入幂等性 bug。

### 把"重启后没看到副作用重复"当作幂等性证据

如果你只跑过一次 crash，刚好这次幂等性没踩中，不代表整体幂等。需要在不同阶段 crash 多次再下结论。

### 把控制器 OOM 与控制器主动重启混为一谈

主动重启走 graceful shutdown 路径，可能保存了一些状态；OOM / SIGKILL 不会。crash resilience 应该按最差的那条路径（OOM）来验证。

## 与其他主题的关系

- 控制层故障的 chaos 如何编排进 ship-readiness 矩阵 — 见 [`docs/addon-test-acceptance-and-first-blocker-guide.md`](addon-test-acceptance-and-first-blocker-guide.md)。
- Cluster 在 chaos 后是否真的回到 Running、各层 readiness 怎么判 — 见 [`docs/addon-bounded-eventual-convergence-guide.md`](addon-bounded-eventual-convergence-guide.md)。
- OpsRequest restart 没进执行体之前的排障思路 — 见 [`docs/addon-ops-restart-troubleshooting-guide.md`](addon-ops-restart-troubleshooting-guide.md)。

## 案例附录：Valkey G4 — controller crash 中段的 OpsRequest 恢复

Valkey ship-readiness G4 验证的是 KubeBlocks 控制器在 Valkey OpsRequest 中段被 SIGKILL 后，OpsRequest 是否能继续推进至终态。本附录保留引擎相关术语和命令。

### 现场设置

- KB 控制器 image：`apecloud/kubeblocks:r08a-...`，KB rev 包含 PR #10182 + #10185。
- Valkey cluster：1 primary + 2 replicas，addon rev 57。
- 触发 Ops：先发起一个 rolling restart 的 OpsRequest（跨多个 reconcile 周期）。

### 操作步骤

1. `kubectl create` OpsRequest 并 poll，直到 `status.phase=Running`，且 progressDetails 上至少一台 pod 已开始 restart。
2. `kubectl -n kb-system delete pod kubeblocks-controller-... --grace-period=0 --force`，让 controller pod 立即被 SIGKILL，K8s 自动重新拉起新 pod。
3. 不做任何其他干预，等 controller pod 重新 Running。
4. 持续 poll OpsRequest 终态，直到 phase 变化或 timeout。

### 观察结果

- controller pod 在 ~30s 内重新 Running。
- OpsRequest 在 controller 恢复后约 1 个 reconcile 周期内继续推进。
- 终态：`phase=Succeed`，`endTimestamp` 已写入。
- 副作用计数：所有目标 pod 各被 rolling 一次（不多不少）。
- Cluster 终态 Running，1 master / 2 replicas，role 拓扑 healthy。

### 结论

- KB OpsRequest 的 desired state 写在 CR 上、reconcile 是幂等的，至少在本次 rolling restart 场景下满足 controller crash resilience。
- 这条结果不能直接外推到所有 Ops 类型；建议把 reconfigure / switchover / backup 等其它 Ops 类型也分别做一遍 G4 验证。

### 测试 artifacts

- 验证脚本：`tests/g4-controller-crash.sh`
- 验证环境：k3d `kb-local`，KubeBlocks `1.2`，Valkey addon rev 57

## 案例附录：Valkey RebuildInstance — cleanup-timing 窗口 controller restart

apecloud/kubeblocks PR #10191 的 e2e 验证里加了一个特定的 controller restart 场景：在 OpsRequest 已经清掉自己写在 InstanceSet 上的 rebuild intent annotation 之后、但新 target pod 还没 Ready 之前，主动重启 KB controller，验证 OpsRequest 不会因为内存里"in-flight 状态"丢了而把已收敛的 PVC 绑定改回去。本附录给具体的触发命令和 pass 条件。

### 现场设置

- KB 控制器 image：`apecloud/kubeblocks:rebuild-fix-20260503-2312`（PR #10191 18-commit branch 编译产物）。
- Valkey cluster：1 primary + 2 replicas，复用 `kubeblocks-tests/valkey/tests/rebuild.sh` baseline 数据 + targeted slave disruption。
- Trial 编号：第 10 个 trial 专门用来插入 chaos restart，前 9 个保持 baseline 形态。

### 触发 chaos restart 的 polling 循环

不依赖 controller 日志关键字 — 触发依据是 InstanceSet annotation 实时 transition `present → absent` AND target pod `Ready=False`：

```bash
INTENT_KEY="operations.kubeblocks.io/rebuild-instance-pvc-overrides"
RESTART_TRIGGERED=0
for i in $(seq 1 60); do
  intent=$(kubectl -n "$NS" get instanceset "$ITS" \
    -o jsonpath="{.metadata.annotations['operations\.kubeblocks\.io/rebuild-instance-pvc-overrides']}")
  ready=$(kubectl -n "$NS" get pod "$REBUILT_POD" \
    -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null || echo "")
  if [ -z "$intent" ] && [ "$ready" != "True" ]; then
    kubectl -n kb-system rollout restart deploy/kubeblocks
    RESTART_TRIGGERED=1
    break
  fi
  sleep 1
done
```

`RESTART_TRIGGERED=1` 后捕获 IS annotation snapshot 作为 before-restart 证据，再每 10s 抓一份 after-restart 直到 pod Ready：

```bash
kubectl -n "$NS" get instanceset "$ITS" -o yaml > "$ART/its-before-restart.yaml"
for i in 1 2 3 4 5 6 7 8 9; do
  sleep 10
  kubectl -n "$NS" get instanceset "$ITS" -o yaml > "$ART/its-after-restart-${i}.yaml"
  if kubectl -n "$NS" wait --for=condition=Ready pod/"$REBUILT_POD" --timeout=5s 2>/dev/null; then
    break
  fi
done
```

### 4 项终态验证

- OpsRequest 终态 `Succeed`（不是被 controller restart 卡在 Running）。
- 目标 source PVC `spec.volumeName` 等于 helper PV name（chaos restart 没让 binding 回退）。
- helper PV reclaim policy 等于推断出来的原值（不是被反复改写）。
- IS annotation 全周期没有 `operations.kubeblocks.io/rebuild-instance-pvc-overrides`（chaos restart 也没让 intent 复活）。

### 观察结果

- Trial 10 单次 PASS。`restartTriggered=1`，monitor 抓到 `intent_present=yes → no` + `pod_ready=False` 同时成立才打的 restart，restart 后 pod 在两轮 reconcile 内重新 Ready，前 9 个 trial 累计 0 个 race-related 错误，第 10 个 trial 维持同样的 0 错误。
- Phase 6 N=10 acceptance + 同 cluster 多轮 follow-up（N=5 / N=10 / N=20）一共 84+ 个 valid sample，全程 0 个 `PersistentVolume "" not found`、0 个 source PVC 最终绑到非 helper PV。

### 结论

- 一个有效的 controller crash resilience 测试不应该靠 controller 日志关键字判定 restart 时机 — 日志是观测信号，不是设计契约本身。靠 CR 上可观测的状态字段（这里是 InstanceSet annotation transition）作为触发依据，得到的证据是不可伪造的。
- "测试期间 controller 没挂 = OpsRequest desired state 在 CR 上完整" 是必要不充分。必须显式插入 chaos restart 才有办法回答"挂了能不能继续"。

### 测试 artifacts

- 触发脚本：`kubeblocks-tests/valkey/tests/rebuild-ops.sh` 的 O02-chaos-gate 段（commit `18817ec`）。
- 完整 evidence bundle：`artifacts/valkey-phase6-rebuild-fix-20260503-2023/trial-10-controller-restart/`。
- 关联 fix PR：apecloud/kubeblocks#10191（commit chain `10dfc40af` → `b99890fbc` 关闭 4 类 contract gap）。

## 相关主题

- [`docs/addon-bounded-eventual-convergence-guide.md`](addon-bounded-eventual-convergence-guide.md)
- [`docs/addon-ops-restart-troubleshooting-guide.md`](addon-ops-restart-troubleshooting-guide.md)
- [`docs/addon-test-acceptance-and-first-blocker-guide.md`](addon-test-acceptance-and-first-blocker-guide.md)
- [`docs/addon-pvc-rebind-via-workload-intent-guide.md`](addon-pvc-rebind-via-workload-intent-guide.md) — PR #10191 关闭的 contract 体系，本附录的设计 prerequisite。
