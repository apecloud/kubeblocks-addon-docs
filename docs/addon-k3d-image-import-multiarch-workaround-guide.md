# Addon 测试环境 k3d 节点镜像注入与 Multi-Arch 绕坑指南

> **Audience**: addon dev / test
> **Status**: stable
> **Applies to**: any KB addon test stack on k3d
> **Applies to KB version**: any (k3d / containerd 层，与 KB 版本无关)

本文面向在 k3d 上跑 KubeBlocks addon 测试的开发与测试工程师。当 k3d 节点拉 docker.io 镜像超时（host 自己拉得动），用 host pull + `docker save` + 节点 `ctr import` 直接绕开；同时不要信赖 `k3d image import` 的退出码，它在 multi-arch manifest 上有已知静默 bug。本文给出排查、修复、验证三步法和反 anti-pattern。

## 先用白话理解这篇文档

最常踩两个坑：

- **坑 A**：k3d 节点（容器内）拉不到 docker.io 镜像，但 host 拉得动。这不是镜像不存在，是 k3d 容器内 DNS / 出网链路不通。
- **坑 B**：`k3d image import` 看起来跑完了 / 提示 success，但 `crictl images` 看不到镜像。这是 multi-arch manifest digest mismatch 的已知 bug，**不要信它的退出码**。

正确做法是：host 拉到本地，`docker save` 成 tar，分发到每个节点，`ctr -n k8s.io images import`。整个流程绕开 k3d 内部 hub 链路 + 绕开 k3d image import 自己的 bug。

## 适用现象

任一 pod stuck 在 `ImagePullBackOff` / `ErrImagePull` / 无限 `ContainerCreating`，`describe` 里看到：

```text
Failed to pull image "X": ... dial tcp <hub-ip>:443: i/o timeout
```

或 `kubectl events` 看到 `helper-pod-create-pvc-*` / 任何 system pod 拉 `rancher/mirrored-*` / `library/*` 失败。

## 第一步先确认是「节点拉不动」还是「host 也拉不动」

```bash
docker pull <image>:<tag>
```

- host 拉得动 → 节点内网络问题（典型：DNS / 出网代理 / containerd registry config）→ 走下面的 host-side 注入。
- host 也拉不动 → host 网络问题，先修 host 网络再说。

下面只覆盖「host 拉得动、节点拉不动」的情形。

## 推荐路径：host save + 节点 ctr import（绕开 k3d image import 的 multi-arch bug）

`k3d image import` 内部会把镜像打包成 tar 然后挂载进节点 import，但它对 multi-arch manifest 的处理在某些版本（≤ 5.x）会出 `content digest ... not found` 的报错，最后还假装成功。**不要信赖 k3d image import 的退出码。** 直接走原始路径：

```bash
# 1. host 拉到本地
docker pull <image>:<tag>

# 2. 导出成 tar
docker save <image>:<tag> -o /tmp/img.tar

# 3. 复制到每个节点
for node in $(k3d node list --no-headers | awk '{print $1}'); do
  docker cp /tmp/img.tar "${node}:/tmp/img.tar"
  docker exec "${node}" ctr -n k8s.io images import /tmp/img.tar
done

# 4. 验证（k3s/k3d 用 crictl 列 containerd 镜像）
docker exec <node> crictl images | grep <image-base-name>
```

**关键点**：

- `ctr -n k8s.io` 必须带 `-n k8s.io`，因为 k3s 用这个 namespace；不带就 import 到默认 namespace，kubelet 看不到
- 每个节点都要 import；server-only / agent-only 都不够
- import 完不需要重启 kubelet / containerd

## 验证镜像生效

最直接的验证：删掉那个 stuck 的 pod / helper pod，让 controller 重新 reconcile。新一轮 pod 应该立即从 `Pending` 走到 `ContainerCreating` → `Running`。

```bash
kubectl -n <ns> delete pod <stuck-pod>
sleep 30
kubectl -n <ns> get pod
```

## 长期方案候选（不在本主题范围）

- 给 k3d 节点配国内 registry mirror（写 `registries.yaml` + `k3d cluster create --registry-config`）
- 起 docker registry proxy 容器，节点走 `/etc/containerd/config.toml` 重定向
- 测试镜像统一预置到企业镜像源，CI 镜像列表入仓

## 验证清单（做完上述操作后必查）

- `crictl images` 在所有节点都能看到目标镜像
- `kubectl events` 没有新的 ImagePull 失败事件
- 之前 stuck 的 pod / PVC 解开（PVC Bound, Pod Init 或 Running）

## 与其他文档的关系

- `addon-test-environment-gate-hygiene-guide.md` 第 4 项「镜像分发层」就是本文话题；本文给具体修法。
- `addon-k3d-kubeconfig-loopback-fix-guide.md` 处理同一类「k3d 默认配置在测试环境踩坑」的另一面（kubeconfig 寻址）。

## 案例附录

### 2026-04-29 oracle smoke T02：local-path provisioner 拉 busybox 卡死

- 镜像：`rancher/mirrored-library-busybox:1.36.1`
- 节点 events：`Head "https://registry-1.docker.io/v2/rancher/mirrored-library-busybox/manifests/1.36.1": dial tcp 108.160.172.1:443: i/o timeout`
- 现象：oracle cluster 创建后，所有 PVC 永远 Pending，oracle pod 永远 Pending（PVC 没绑就调度不动）
- 根因：local-path provisioner 用 helper pod 创建 PV，helper pod 拉不到镜像
- 修复：用上述标准路径，2 节点 import 完后 30s 内 PVC Bound、pod Init:0/4
- `k3d image import` 也试过：每节点报 `ctr: content digest ... not found`，最后假装 `Successfully imported 1 image(s)` 但 `crictl images` 看不到 — multi-arch image manifest digest mismatch 的已知坑
