# Addon 测试环境 k3d kubeconfig 0.0.0.0 → 127.0.0.1 修正指南

> **Audience**: addon dev / test
> **Status**: stable
> **Applies to**: any KB addon test stack on k3d (macOS / Linux)
> **Applies to KB version**: any (k3d / kubeconfig 层，与 KB 版本无关)

本文面向在 k3d 上跑 KubeBlocks addon 测试的开发与测试工程师。新建 k3d cluster 后 `kubectl` / `helm` 立刻报 `Unable to connect to the server: EOF`，根因不是集群没起来，而是 kubeconfig 把 `server:` 写成了 `https://0.0.0.0:<port>` —— macOS 与部分 Linux 内核版本下，`0.0.0.0` 不是合法的 client 连接目标。本文给一次性修正、脚本场景修正、集群创建时直接固定三种做法。

## 先用白话理解这篇文档

最容易被误判的一句话：

- 看到 `kubectl` 报 EOF，第一反应往往是「集群挂了」「权限错了」「KB 没装好」
- 但只要 kubeconfig 里 `server:` 写的是 `https://0.0.0.0:<port>`，根因就是 client 端寻址错，跟集群、KB、helm 都没关系
- 改成 `127.0.0.1` 立刻通。先做这一步再排其他

任何 k3d 测试集群的「**第一道 gate**」都先跑这条 sed —— 比装 KB / 装 addon 都早。

## 适用现象

新建 k3d cluster 后立刻报：

```text
Unable to connect to the server: EOF
```

或 helm 同样报 EOF。复看 `~/.kube/config` 当前 cluster 项：

```yaml
clusters:
- cluster:
    server: https://0.0.0.0:<random-port>
  name: k3d-<cluster-name>
```

## 原因

`k3d cluster create` 默认把 `serverHost` 写成 `0.0.0.0`（监听所有 interface 的 bind 写法）。但 macOS / 部分 Linux 内核版本下 **`0.0.0.0` 不是合法的 client 连接目标**（它只能用作 `bind` 地址，不能用作 `connect` 地址）。Go 的 net stack 上经常表现为握手前直接 EOF。

正确的 client 目标是 **`127.0.0.1`**（loopback）。

## 一次性修正（已有 kubeconfig）

```bash
cp ~/.kube/config ~/.kube/config.bak
sed -i '' 's|server: https://0.0.0.0:|server: https://127.0.0.1:|g' ~/.kube/config
# Linux 用 sed -i 's...' 不带 ''
```

## 临时 kubeconfig（脚本里调用 `k3d kubeconfig get`）

`k3d kubeconfig get <cluster>` 仍会输出 `0.0.0.0`，**不能在脚本里直接 pipe 给 kubectl**。包一层 sed：

```bash
k3d kubeconfig get <cluster> | sed 's|server: https://0.0.0.0:|server: https://127.0.0.1:|g' > /tmp/kubeconfig
export KUBECONFIG=/tmp/kubeconfig
```

或者用 `k3d kubeconfig merge --kubeconfig-merge-default` 合并到默认 kubeconfig 后再统一 sed。

## 集群创建时直接固定（更彻底）

`k3d cluster create` 加 `--api-port 127.0.0.1:<port>`：

```bash
k3d cluster create <name> --api-port 127.0.0.1:6550 --servers 1 --agents 1
```

这样 kubeconfig 里写出来的 server 就直接是 `https://127.0.0.1:6550`，不需要后处理。

## 适用边界

- macOS（已确认两台不同机器都中招）
- 部分 Linux distro（行为不一致，但加 `127.0.0.1` 永远更安全）
- 不影响 cluster 内 pod-to-pod 通信（那是 cluster network，与 host kubeconfig 无关）
- 不影响 ingress / nodeport 对外暴露端口

## 沉淀给团队的检查清单

每次新建 k3d cluster 后**做 KB 安装基线之前**先做这步：

```bash
grep -E '^\s+server: https://0\.0\.0\.0' ~/.kube/config && {
  cp ~/.kube/config ~/.kube/config.bak
  sed -i '' 's|server: https://0.0.0.0:|server: https://127.0.0.1:|g' ~/.kube/config
  echo "fixed kubeconfig server to 127.0.0.1"
}
kubectl cluster-info
```

`kubectl cluster-info` 不报 EOF 才进入下一步基线快照。

## 与其他文档的关系

- [`addon-test-environment-gate-hygiene-guide.md`](addon-test-environment-gate-hygiene-guide.md) 第 1 项「API / 路由层」就包含 kubeconfig listener；本文是其中最常见的一种 first blocker 的具体修法。
- [`addon-k3d-image-import-multiarch-workaround-guide.md`](addon-k3d-image-import-multiarch-workaround-guide.md) 处理同一类「k3d 默认配置在测试环境踩坑」的另一面（image 分发）。

## 案例附录

- 2026-04-29 Phase 0 oracle-test cluster：本机和另一台 Mac 都中过同样的坑，团队成员在装 KB 1.0.2 之前先修了 kubeconfig 才能用 helm。
