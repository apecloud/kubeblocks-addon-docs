# Addon 测试环境 Gate 卫生指南

> **Audience**: addon test engineer / TL
> **Status**: stable
> **Applies to**: any KB addon
> **Applies to KB version**: any (k3d/vcluster/host layer，跟 KB 版本不耦合)

## 先用白话理解这篇文档

### 你之前可能这么想

测试 runner 起来跑 smoke。第一步 "T1 创建集群" 失败了。先反应是 "addon 有 bug 没启动好"，开始查 addon code / DB pod log。结果查了半天发现：
- PVC 永远 Pending
- 镜像 ImagePullBackOff
- vcluster `127.0.0.1:11443` listener 不在
- csi-hostpathplugin 跑着但 1/4 Ready

**addon code 完全无辜**，是测试环境层在异响。

### 这种思路的代价

排障时间从 "5 分钟看完 DB log" 拖成 "半天 trace controller 全栈"。**first blocker 落到了错误的层**。报告写"产品 fail"，实际是环境 fail；下次同样症状别人又踩同样的坑。

特别危险的场景：**机器/容器重启之后**。CSI mount propagation 漂回 private、kubelet 根路径权限改了、镜像缓存被 GC、port-forward 进程死了 —— 任何一条都让 T1 timeout 但 addon log 完全干净。

### 正确 reframe

**测试环境就绪本身要当成证据收一遍**，不是默认信任。在 runner 跑起来之前，每一层（路由/控制面/CSI/镜像分发/staged anchors/fresh slot/capability）都坐实一次，留下时间戳证据。

post-restart 场景里这条最关键：**禁止复用 pre-restart 的任何事实**。"昨天能跑就今天能跑" 是错觉。

### 什么时候应该 reach 这篇

- **跑任何 smoke / chaos / regression 之前**：环境就绪走一次 7-layer gate
- **测试机重启 / Docker 重启 / k3d 集群被动恢复之后**：所有 pre-restart 事实作废重收
- **看到 T1 timeout 但拿不到 DB 层证据**：先怀疑 gate，不怀疑产品
- **判 first blocker 时**：分清"产品 fail"vs"环境 fail"vs"测试 fail" 三层

### 读完这篇你能做什么

- 7-layer gate checklist 在 runner 启动前逐项坐实
- 写 gate 的 source-of-truth 文件结构（route_action / route_rc / image_sha / pod_image 等）
- post-restart 场景的 reseat 步骤（特别是 CSI 的 mount propagation rshared 重设）
- 把"环境就绪"作为 first-class testable artifact，不让它当 silent default

### 为什么独立成篇

- **gate 作用域只在测试 runner 启动前**：跟产品语义、first blocker 分层、stress profile 等是不同 phase 的 concern
- **跨引擎复用**：mariadb / valkey / oracle 的 gate 流程一致，不绑定单一引擎实现
- 与 [`addon-test-acceptance-and-first-blocker-guide.md`](addon-test-acceptance-and-first-blocker-guide.md) 互补 —— first-blocker guide 处理 runner **跑起来之后**的 first blocker 分层，gate-hygiene 处理 runner **跑起来之前**的就绪坐实，两者顺序衔接

## 主体

本文面向 Addon 测试工程师与技术负责人，重点解决"机器/容器重启之后、跑测试之前，测试环境本身能不能用"的问题。它不是关于产品 / addon 的成功语义，而是关于**让 first blocker 不要落在测试环境本身**。

## 适用场景

- 跑任何 smoke / chaos / regression 之前
- 测试机重启 / Docker 重启 / k3d / vcluster 集群被动恢复之后
- 看到测试在 T1 / 创建集群阶段就 timeout，但拿不到任何 DB 层证据
- PVC 永远 Pending、Pod 永远 Pending、初始化容器没起来、镜像 ImagePullBackOff

## 核心原则

> 在测试 runner 跑起来之前，**所有"环境就绪"必须当成证据收一遍**，不是默认信任。
> 否则 first blocker 会落在环境层，但表现得像产品层。

post-restart 场景里这一点最重要：**禁止复用 pre-restart 任何事实**（路由、镜像缓存、控制器 pod、存储类、socket、临时目录），全部重新坐实。

## Gate 要逐项坐实的清单（顺序）

按从下往上的依赖顺序检查，任何一项脏就停在那一层、出 source-of-truth、不进 runner：

### 1. API / 路由层
- 主路由（如 vcluster `127.0.0.1:11443`）listener 在线、port-forward 进程存活
- `kubectl get ns` / `kubectl get pod -n <controller-ns>` 在路由后面能成功
- 留下 `route-pre-run-final.txt` / route_action / route_rc / route_stderr

### 2. 控制器 / 控制面身份层
- 控制器 pod 名 / image / imageID / sha256 与期望基线一致
- 如果跑在 vcluster，要分清 vcluster controller 与 host controller，**两者要分别记录**
- host controller 通常只作"环境上下文"，不作 gate 项；但要落 imageID

### 3. 存储 / CSI 就绪层（最容易被忽视）
- StorageClass：name、provisioner、`allowVolumeExpansion`、reclaimPolicy、bindingMode
- **CSI 控制套件**：provisioner / attacher / resizer / snapshotter / plugin（如 csi-hostpath-*）
  - 每个 pod 必须 Running 且全部容器 Ready
  - **不能允许 CrashLoopBackOff / Error / 1/N Ready**，哪怕 StorageClass 看起来没问题
- 如果是 hostpath / nodelocal 类驱动，还要看 csi.sock 是否存在
- post-restart 之后必须明确再核：mount propagation、宿主路径权限、kubelet 根路径

### 4. 镜像分发层
- 检查每一类必需镜像是否存在于 **每一个** k3d node / 每一个 worker（不是只看一台）
- 包含但不限于：addon 引擎主镜像、syncer / sidecar、init 容器、reloader、controller
- post-restart 后 k3d image cache 不可信，建议每次显式 `k3d image import`

### 5. 测试 runner / 锚点 staged 层
- runner 源码与 staged copy 都要含有当前一轮所有锚点（`grep -nE "anchor1|anchor2|..."`）
- 关键 helper（route-aware exec、bounded recovery、capability-aware skip 等）必须实际嵌进路径上
- `bash -n` / `git diff --check` 静态绿

### 6. fresh slot / no-tail 层
- 上一次运行的 sample 资源已彻底清理（cluster / component / pvc / pod / svc）
- 测试目录下没有 stale 现场（旧 log、旧 evidence、旧 lock）
- 留下 `source_tail_count=0` / `host_tail_count=0` 之类的零证据

### 7. 环境能力声明层
- 是否有 LoadBalancer 控制器？是否有外部 DNS？是否有 monitoring sidecar？
- 显式标 `has_lb_support` / `has_external_dns` 等 capability flag
- 后续动作（如 T13 expose）按 capability 走 fail-fast skip / 走完整路径

## 为什么必须包含 CSI 这一层

`StorageClass exists + allowVolumeExpansion=true` **不等于** PVC 能 bind。
真正等的链是：

```
PVC Pending
  → external-provisioner watching PVC
    → calls CreateVolume via csi.sock
      → csi-hostpathplugin (or similar) handles
```

任何一环（provisioner pod、plugin pod、socket、host mount propagation）出问题，PVC 就一直 Pending；scheduler 报"unbound immediate PersistentVolumeClaims"，pod 永远 Pending，测试在 T1 timeout。

如果 gate 只检查 StorageClass 而不检查 CSI 套件健康，**这种 first blocker 一定会被误归到产品层**。

## 一次干净的 post-restart gate 长什么样

伪代码 / 动作清单：

```
post_restart_gate():
  1. 确认 vcluster / k3d / Docker 都在线
  2. probe_api_route() 直到稳定 → 留 listener context
  3. assert vcluster_controller image == expected baseline
  4. assert addon live chain == expected version, 旧版本 not referenced
  5. for sc in storage_classes:
       assert sc fields == expected
  6. assert_csi_pods_all_ready(namespace, names)   # ← 关键，单独一道
  7. for img in required_images:
       for node in k3d_nodes:
         assert image_present(node, img)
  8. grep_runner_anchors(staged_copy, expected_anchors)
  9. bash -n + git diff --check
 10. assert no stale fresh slot, no tail in source/host
 11. record capability flags (lb / dns / monitoring)
 12. only then → start runner
```

每一步都要落具体的证据文件路径（`/tmp/<gate-dir>/<item>.txt`），方便 first blocker 复盘。

## 如果 gate 哪一步脏：回退动作

不要"修一下试试看"，按层回退：

| 脏的层 | 默认动作 |
|---|---|
| 路由 | 重启 port-forward / 检查 listener / 必要时换端口 |
| 控制器身份 | 重新 `helm upgrade` / 校对镜像 sha256 |
| CSI 套件 | 排查 mount propagation / kubelet 根路径 / 必要时 `mount --make-rshared` 或重建 k3d |
| 镜像分发 | `k3d image import` 全部 worker，或重建集群 |
| 锚点 staged | 重新 `git checkout` / `make stage` / 重新拷贝 helper |
| fresh slot | 主动 cleanup，不靠"反正会被新 sample 覆盖" |
| capability 声明 | 显式记录而不是默认 true/false |

## 与其他文档的关系

- [`addon-test-acceptance-and-first-blocker-guide.md`](addon-test-acceptance-and-first-blocker-guide.md) 讲"测试要看到什么算成功 / 哪一层算 first blocker"。
- [`addon-test-probe-classification-guide.md`](addon-test-probe-classification-guide.md) 讲"探针失败如何分到正确的层"。
- 本文讲"测试还没开始之前，环境就绪要逐项坐实，避免 first blocker 落到环境却被误归产品层"。

三者串起来：本文负责"开跑前不踩雷" → 探针分类负责"跑中失败不误归" → 测试验收负责"跑完结论不混层"。

## 案例附录

- MariaDB：[`docs/cases/mariadb/post-restart-csi-mount-propagation-case.md`](cases/mariadb/post-restart-csi-mount-propagation-case.md) — post-restart 后 k3d 节点 `/var/lib/kubelet` 失去 rshared 传播，csi-hostpathplugin Bidirectional mountPropagation 无法创建容器，全套 CSI Crash → PVC 永远 Pending → T1 timeout
