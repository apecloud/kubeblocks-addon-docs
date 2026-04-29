# Addon Backup / Restore 在 k3d 上的两层环境前置指南

> **Audience**: addon dev / test
> **Status**: stable
> **Applies to**: any KB addon backup/restore on k3d / kind / vanilla local k8s（mysql / mongo / oracle / pg / valkey 通用）
> **Applies to KB version**: 1.0.x / 1.1.x release tags（`BackupRepo` schema + `pvc` StorageProvider 在两版本验证过；KB main / 1.2 待重测）
> **Affected by version skew**: `dataprotection.kubeblocks.io/v1alpha1` 这套 BackupRepo / BackupPolicy / Backup CR 在 KB 1.x 系列稳定；如未来跨 major 演化需重测 §2 的 yaml 是否仍兼容

本文面向准备在本地 k3d / kind / vanilla k8s 上跑 KubeBlocks addon backup-restore smoke 的开发与测试工程师。两层前置是引擎无关的（mongo / mysql / oracle / pg / valkey 都吃同一套）：先装 VolumeSnapshot CRD（不然 dataprotection controller 起不来），再建一个默认 BackupRepo（不然每个 Backup CR 都报 `NoDefaultBackupRepo`）。两层都过了之后，引擎相关的 backup actionSet / restore 行为才能被真正测到。

## 先用白话理解这篇文档

最容易让人误判的两步：

- **第一步**：dataprotection controller `CrashLoopBackOff` —— 看起来像 KB 装坏了，其实只是缺 `VolumeSnapshot` CRD。装 3 个 CRD 解决。
- **第二步**：controller 起来了，但每个 Backup CR 永远 Failed —— 看起来像 backup actionSet 写错了，其实只是没 BackupRepo。建一个最小 PVC-backed BackupRepo 就好。

**关键认知**：即使你的 backup method 不依赖 VolumeSnapshot（如 mongo `dump`、oracle `rman`、mysql `xtrabackup`、pg `basebackup` 这些 in-DB / file-level 路径），controller 也会因为缺 CRD 而起不来 → **任何 backup method 都被堵住**。

## 症状速查表

| 症状 | 在哪一层 | 修复章节 |
|---|---|---|
| `kubeblocks-dataprotection` Pod CrashLoopBackOff | Layer 1 | §1 |
| dataprotection logs: `failed to wait for backup caches to sync ... VolumeSnapshot` | Layer 1 | §1 |
| Backup CR 永远 Pending / Failed，dataprotection controller 是 Running | Layer 2 | §2 |
| dataprotection logs: `no default BackupRepo found` | Layer 2 | §2 |
| `kubectl get backuprepo` → `No resources found` | Layer 2 | §2 |

## §1 Layer 1: VolumeSnapshot CRD

### 根因

KB 的 `kubeblocks-dataprotection` controller 启动时无条件 watch `*v1.VolumeSnapshot`（CSI 标准 snapshot CRD）。任何 vanilla k8s（k3d / kind / minikube）默认都不装 `snapshot.storage.k8s.io` CRD group。controller 没法 sync cache → manager 启动失败 → CrashLoopBackOff。

### 修复（CRD-only，零破坏面）

```bash
SNAP_VER="v8.2.0"   # external-snapshotter 当前 stable
BASE="https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/${SNAP_VER}/client/config/crd"

kubectl apply -f "${BASE}/snapshot.storage.k8s.io_volumesnapshots.yaml"
kubectl apply -f "${BASE}/snapshot.storage.k8s.io_volumesnapshotcontents.yaml"
kubectl apply -f "${BASE}/snapshot.storage.k8s.io_volumesnapshotclasses.yaml"
```

**为什么只装 CRD 不装 snapshot-controller**：CRD 让 dataprotection 启动通过；只有当你的 backup method 真的要 snapshot 物理卷时才需要 snapshot-controller + 一个能做 snapshot 的 CSI driver（hostpath-driver / OpenEBS / 等）。in-DB / file-level backup 用不到。

### 验证

```bash
kubectl get crd | grep snapshot.storage.k8s.io   # 必须 3 条
kubectl -n kb-system get pod -l app.kubernetes.io/name=kubeblocks-dataprotection
# 60-90s 内 Running 0 restart
```

如果 controller 60s 后还在 Crash，先看新一轮 `--previous` log，可能 cache sync 还有别的依赖（罕见）。

### 回滚

```bash
kubectl delete crd volumesnapshots.snapshot.storage.k8s.io \
  volumesnapshotcontents.snapshot.storage.k8s.io \
  volumesnapshotclasses.snapshot.storage.k8s.io
```

退回原状（dataprotection 又开始 crash）。

## §2 Layer 2: 默认 BackupRepo

### 根因

dataprotection controller 起来后，每个 Backup CR reconcile 时都会找：

1. cluster 引用的 BackupPolicy
2. BackupPolicy 引用的 BackupRepo（或者 cluster default BackupRepo）

如果环境里**完全没有 BackupRepo**，controller 直接报：

```
ERROR no default BackupRepo found
```

对应 Backup CR 直接 Failed。

vanilla k3d 没有任何 BackupRepo，必须手动建一个。

### 修复（最小依赖：StorageProvider `pvc` + local-path）

KB 自带的 `pvc` StorageProvider 是 Ready 的（不需要任何外部 CSI / S3 / NFS），可以直接 ref。结合 k3d 自带的 `local-path` storageclass，可以零外部依赖建出可用 BackupRepo：

```yaml
apiVersion: dataprotection.kubeblocks.io/v1alpha1
kind: BackupRepo
metadata:
  name: local-backup-repo
  annotations:
    dataprotection.kubeblocks.io/is-default-repo: "true"
spec:
  storageProviderRef: pvc
  accessMethod: Mount
  pvReclaimPolicy: Retain
  volumeCapacity: 20Gi
  config:
    storageClassName: local-path
```

**字段释义**：

- `storageProviderRef: pvc` — 用内建的 PVC provider，跳开所有外部 storage 依赖
- `accessMethod: Mount` — backup 数据通过 PVC 挂载方式读写（vs Tool 模式用 CLI 工具如 mc/aws cli）
- `pvReclaimPolicy: Retain` — backup PV 不随 BackupRepo 删除，避免误删
- `volumeCapacity: 20Gi` — 一次完整 dump 需要的容量。Oracle rman/expdp ~5-10 GB，留 buffer
- `config.storageClassName: local-path` — k3d 默认 sc，一定存在

`is-default-repo: "true"` annotation 让所有未显式 ref 的 BackupPolicy 自动用它。

### 验证

```bash
# BackupRepo 自身 Ready
kubectl get backuprepo local-backup-repo -o jsonpath='{.status.phase}{"\n"}'   # Ready

# 看 prepare PVC helper pod（一次性，跑完即清）
kubectl -n kb-system get pod -l app.kubernetes.io/managed-by=kubeblocks-dataprotection
```

### 注意

- **Failed 的 Backup CR 不会自动重试**。修了 BackupRepo 之后要把旧的 Backup CR 删掉重建，或者直接清整个 cluster 重跑 smoke 让 OpsRequest 重新拉 backup
- 如果你的 backup method 真的依赖 snapshot（比如 mysql VolumeSnapshot backup method），光建 BackupRepo 不够，还得装 snapshot-controller + 一个能做 snapshot 的 CSI driver。`pvc` provider 只解锁 file-level / dump 类 backup
- `Retain` reclaim policy 意味着每次跑 smoke 都会留 PV，多次跑要手动清 — 测试环境可以容忍

## 执行顺序

1. §1 装 CRD（任何 backup 测试都要）
2. §2 建 BackupRepo（任何 backup 测试都要）
3. 验证 dataprotection Running + BackupRepo Ready
4. 跑 addon backup-restore smoke

两层都过了之后，引擎相关的 backup actionSet / restore 行为才能开始被真正测试 — 此前的失败都是环境层。

## 反 anti-pattern

- **不要** 在 runner 里加 `if ! has_backup_repo; then skip T08`：在本地可控环境，环境 gap 应该补齐而不是绕过。绕过等于 smoke 不完整，把"未验证"伪装成"通过"
- **不要** 直接在 `BackupPolicy` 里 inline storage 配置：会绕开 KB BackupRepo abstraction，导致 controller 路径行为和生产不一致
- **不要** 用 `accessMethod: Tool` + 没装 mc/aws cli 的 image：报错很隐蔽，多绕一圈

## 与其他文档的关系

- `addon-test-environment-gate-hygiene-guide.md` 第 3 项「存储 / CSI 就绪层」与本文的 §1 互补：那篇讲 PVC 能不能 bind，本文讲 backup PV / VolumeSnapshot 能不能 work。
- `addon-test-acceptance-and-first-blocker-guide.md` 强调 first blocker 分层；本文 §1/§2 是典型的「first blocker 落到环境层但表现得像产品层」例子。

## 案例附录

### 2026-04-29 Oracle Run 5 → Run 6：连续撞两层

- Run 5 跑到 T08 时 dataprotection Pod CrashLoopBackOff（46 次重启）→ §1 fix → controller 起来
- Controller 起来后立刻报 `no default BackupRepo found`，rman + expdp 两个 Backup CR 直接 Failed → §2 fix
- 修完 BackupRepo 后清掉 Run 5 cluster，Run 6 让 T08 在干净状态从头跑，T08 backup→restore 全程 PASS
- 教训：两层 fix 应该一开始就预备好；只解 §1 不解 §2 等于半完工
