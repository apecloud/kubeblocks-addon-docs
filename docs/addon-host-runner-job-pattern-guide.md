# Addon Host Runner Job Pattern 指南

> **Audience**: addon dev / test / TL
> **Status**: draft
> **Applies to**: any KB addon test runner that must run against IDC / remote vcluster environments
> **Applies to KB version**: any（runner placement / kubeconfig / artifact discipline，与 KB 版本解耦）
> **Affected by version skew**: no（但 runner 镜像里的 `kubectl` 与目标集群 minor version 需保持兼容）

## Abstract

本文面向需要把测试从 MacBook / bastion 迁到 IDC host k8s 的 addon 团队。目标是让测试脚本在 host k8s 里以 Kubernetes Job 运行，Mac 只负责提交 Job、观察日志和取回 artifact，不再承载测试执行链路，也不再依赖 port-forward。

本文只讲 host-runner Job 模式本身；kube context / proxy / shared client state 的 preflight 见 [`addon-test-script-preflight-guide.md`](addon-test-script-preflight-guide.md)，shell 兼容性见 [`addon-test-runner-portability-guide.md`](addon-test-runner-portability-guide.md)。

## 先用白话理解这篇文档

在 IDC / vcluster 上跑测试时，最容易踩的坑不是测试脚本本身，而是**测试进程还跑在 Mac 上**：

- Mac 睡眠 / 网络抖动 / VPN 断开会中断长跑测试。
- port-forward 是一个本地进程，不是集群内稳定基础设施。
- 多个 line 共用一台 Mac 时，`~/.kube/config`、proxy、临时 kubeconfig 很容易互相污染。

Host-runner Job 模式把这个链路反过来：

- **测试进程跑在 IDC host k8s 里**。
- **被测对象仍然跑在 vcluster 里**。
- **Mac 只提交 Job、看结果、取 artifact**。

这样 Mac 断网不会影响测试继续跑，测试 evidence 也天然留在集群侧。

## 适用场景

当满足任一条件时，建议用 host-runner Job：

- 测试目标在 IDC / idc2 / idc4 vcluster。
- 测试需要跑 30 分钟以上，或包含 N 轮并发 / soak / chaos。
- 当前脚本依赖 Mac 上的 port-forward 访问 vcluster API。
- 测试过程中需要收集大量 pod log / controller log / SQL output / artifact。
- 多个 addon line 共享 Mac 或 bastion，默认 kube context 有串用风险。

不适用场景：

- 单机本地 k3d 快速开发调试，且测试 1-2 分钟内完成。
- 需要本地 IDE 逐行调试的脚本开发阶段。
- 只做一次只读 `kubectl get` inspection，不是正式测试执行。

## 目标架构

```text
IDC host k8s
┌────────────────────────────────────────────┐
│ namespace: <addon>-runner                  │
│                                            │
│  Job: <addon>-test-runner                  │
│  ├─ Secret: vcluster kubeconfig            │
│  ├─ ConfigMap: test scripts                │
│  ├─ PVC: artifacts                         │
│  └─ runner image: kubectl + bash + tools   │
│                                            │
└───────────────────┬────────────────────────┘
                    │ kubeconfig points at vcluster API
                    ▼
vcluster under test
┌────────────────────────────────────────────┐
│ KubeBlocks controller + addon              │
│ test namespaces / clusters / pods          │
└────────────────────────────────────────────┘
```

关键边界：

- Runner 放在 **host k8s**，不要放进被测 vcluster。vcluster 是被测对象，runner 不应跟被测对象同生命周期。
- Runner 使用独立 namespace，例如 `<addon>-runner`。
- Runner 不读取 Mac 的 `~/.kube/config`。vcluster kubeconfig 必须通过 Secret 注入。
- Runner artifact 写 PVC，不写 Mac 本地路径。
- Mac 不再执行测试脚本，也不跑 port-forward。

## 先选 vcluster API 入口

Runner 必须能访问 vcluster API。优先级建议如下：

1. **host k8s 内部 ClusterIP Service**：如果 runner namespace 能访问 vcluster service DNS / ClusterIP，这是最稳的集群内路径。
2. **NodePort**：如果已有标准化 NodePort kubeconfig，或跨 namespace / DNS 不方便，runner 也可以访问 host node 的 NodePort。
3. **禁止 port-forward**：port-forward 是本地临时进程，不应进入正式 IDC runner 链路。

ClusterIP 路径理论上更稳，但本文的 reference implementation 走的是 NodePort；ClusterIP 路径需要第一个采用该模式的 addon 团队补齐实测 evidence 后再升级为实证规则。

无论选 ClusterIP 还是 NodePort，都要把最终 server URL 固化进 vcluster kubeconfig Secret，并在 probe Job 里验证：

```bash
kubectl --kubeconfig "$KCFG" --request-timeout=10s get --raw=/version
kubectl --kubeconfig "$KCFG" --request-timeout=10s get ns
```

## 基础资源

一个最小 host runner 需要：

- Namespace：隔离 runner 本身。
- ServiceAccount：不要复用 `default`。
- Role / RoleBinding：只给 runner namespace 内查看 Job / Pod / logs 的权限。
- Secret：挂 vcluster kubeconfig。
- ImagePullSecret：如果 runner image 来自 ACR / private registry，必须让 runner ServiceAccount 或 Job pod 能冷拉镜像；不要依赖节点 cache。
- ConfigMap：挂测试脚本。
- PVC：保存 artifact。
- Job：一次测试一次 Job。

下面只给环境差异最小的 Namespace / ServiceAccount / PVC 骨架；Secret / ConfigMap / Job 在后续章节单独示范。Role / RoleBinding 是否需要更多 host-side 观测权限，取决于 runner 是否要读取 host k8s 自身的 pod / job / log。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: addon-runner
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: addon-runner
  namespace: addon-runner
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: addon-runner-artifacts
  namespace: addon-runner
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

StorageClass 不要照抄别人的名字。不同 host cluster 可能使用不同 StorageClass；apply 前先查：

```bash
kubectl get storageclass
```

## kubeconfig Secret

把 vcluster kubeconfig 当敏感材料，通过 Secret 注入：

```bash
kubectl -n addon-runner create secret generic addon-runner-kubeconfig \
  --from-file=kubeconfig=/path/to/vcluster.kubeconfig \
  --dry-run=client -o yaml | kubectl apply -f -
```

脚本入口必须显式读取 Secret 挂载路径：

```bash
KCFG="${KCFG:-/etc/addon-runner/kubeconfig}"
[[ -r "$KCFG" ]] || { echo "FATAL: kubeconfig $KCFG missing" >&2; exit 1; }
```

不要让 runner 在 Secret 缺失时 fallback 到 Mac 本地路径。fallback 可以保留给本地调试，但正式 Job 必须通过 env 覆盖，并有 startup assertion。

## 脚本 ConfigMap

脚本通过 ConfigMap 注入：

```bash
kubectl -n addon-runner create configmap addon-runner-scripts \
  --from-file=run_matrix.sh=./run_matrix.sh \
  --dry-run=client -o yaml | kubectl apply -f -
```

脚本移入 Job 前必须做 Mac 依赖扫描：

```bash
grep -RInE 'port-forward|/Users/|~/|localhost:11[0-9]{3}|shasum -a 256' run_matrix.sh
bash -n run_matrix.sh
```

常见修复：

- `shasum -a 256` 改成 `sha256sum` 优先，必要时再 fallback 到 `shasum`。
- artifact 路径必须使用 `/artifacts/<run-id>`。
- `tar` 打包绝对路径时用 `-C "$(dirname "$OUT")" "$(basename "$OUT")"`，不要直接 `tar "$OUT"`。
- SQL client 尽量通过 `kubectl exec <db-pod> -- <client>` 在被测 pod 内执行，避免 runner 镜像再安装数据库客户端。

## Probe-first，不要直接跑全量

正式 Job 前先跑一个 probe Job，只验证基础设施：

- runner image 能拉起。
- `bash` / `kubectl` / `tar` / `sha256sum` 存在。
- kubeconfig Secret mount 正确。
- vcluster API 可达。
- artifact PVC 可写。

probe command 示例：

```bash
set -euxo pipefail
command -v bash
command -v kubectl
command -v tar
command -v sha256sum
[[ -r "$KCFG" ]]
kubectl --kubeconfig "$KCFG" --request-timeout=10s get --raw=/version
kubectl --kubeconfig "$KCFG" --request-timeout=10s get ns > /tmp/namespaces.txt
head -20 /tmp/namespaces.txt
touch /artifacts/probe-ok-$(date +%s)
echo PROBE_OK
```

注意 `set -o pipefail` 下不要写：

```bash
kubectl get ns | head -20
```

`head` 读够后关闭 pipe，`kubectl` 可能收到 SIGPIPE，脚本退出码变成 141。先写临时文件再 `head` 更稳。

## Runner image 选择

Runner image 至少需要：

- bash 4+
- kubectl
- tar
- sha256sum
- 证书 / CA 能访问目标 API server

不要假设 `kubeblocks-tools` 一定有 bash。部分工具镜像是 alpine / distroless 或只包含最小工具集。更稳的做法是：

- 主容器使用已有 e2e / CI runner 镜像，提供 bash / tar / sha256sum。
- initContainer 从工具镜像复制 kubectl 到 shared `emptyDir`。
- 主容器把 shared path 加进 `PATH`。
- 如果 runner image 来自 ACR / private registry，必须在 Job pod 或 ServiceAccount 上配置 `imagePullSecrets`；不要假设节点 cache 永远命中。

示例：

```yaml
initContainers:
  - name: kubectl-seed
    image: <registry>/apecloud/kubeblocks-tools:1.0.0
    command: ["/bin/sh", "-c"]
    args:
      - |
        set -e
        mkdir -p /work/bin
        cp /usr/bin/kubectl /work/bin/kubectl
        chmod 0755 /work/bin/kubectl
    volumeMounts:
      - name: work
        mountPath: /work
containers:
  - name: runner
    image: <registry>/apecloud/e2e-ginkgo:main
    command: ["/bin/bash", "-lc"]
    args:
      - |
        set -euo pipefail
        export PATH=/work/bin:$PATH
        exec /scripts/run_matrix.sh
    volumeMounts:
      - name: work
        mountPath: /work
```

如果 kubectl 是 musl-linked binary，而主容器是 glibc/debian 系，可能还要复制 musl loader 并用 wrapper 启动。不要在 full run 才发现 ABI 问题；probe Job 必须覆盖 `kubectl --kubeconfig "$KCFG" get --raw=/version`。

Reference implementation: 2026-05-05 MariaDB idc-k8s host runner 使用 ACR `apecloud/e2e-ginkgo:main` 作为主容器，并用 `apecloud/kubeblocks-tools:1.0.0` initContainer 提供 `kubectl`，已通过 probe + 8 组 full-run 验证，run id `mariadb-alpha16-fresh-matrix-g97h7`，artifact sha256 `a7f7af208d4e778bb4c4c8dd702f4d01b244b16a1336d88163fbabc875265197`。

## Full-run Job

一个正式 Job 应该具备：

- `generateName`，让每轮 run 有唯一 Pod / Job 名。
- `backoffLimit: 0`，失败不要自动重跑污染现场。
- `activeDeadlineSeconds`，防止卡死。
- `ttlSecondsAfterFinished`，保留一段审计窗口。
- `RUN_ID` 从 Pod name 或 Job name 注入，artifact 路径唯一。

示例结构：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: addon-matrix-
  namespace: addon-runner
spec:
  backoffLimit: 0
  activeDeadlineSeconds: 7200
  ttlSecondsAfterFinished: 86400
  template:
    spec:
      restartPolicy: Never
      serviceAccountName: addon-runner
      containers:
        - name: runner
          image: <registry>/apecloud/e2e-ginkgo:main
          env:
            - name: KCFG
              value: /etc/addon-runner/kubeconfig
            - name: RUN_ID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: OUT
              value: /artifacts/$(RUN_ID)
          command: ["/bin/bash", "-lc"]
          args:
            - |
              set -euo pipefail
              export PATH=/work/bin:$PATH
              kubectl --kubeconfig "$KCFG" --request-timeout=10s get --raw=/version
              exec /scripts/run_matrix.sh
```

`RUN_ID` 要进入 namespace / cluster / artifact 命名，避免连续两次 Job 撞同名资源。

## Artifact retrieval

Job 完成后，很多镜像不能再被 `kubectl cp`，因为 `kubectl cp` 需要在容器里 exec `tar`。稳妥方案有两种：

1. **Job pod 仍可 exec**：直接 `kubectl cp <pod>:/artifacts ...`。
2. **Reader pod 挂同一 PVC**：Job 完成后起一个临时 reader pod，mount artifact PVC，再 `kubectl cp reader:/artifacts ...`。

Reader pod 模式示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: addon-artifact-reader
  namespace: addon-runner
spec:
  restartPolicy: Never
  containers:
    - name: reader
      image: <registry>/apecloud/e2e-ginkgo:main
      command: ["/bin/bash", "-lc", "sleep 3600"]
      volumeMounts:
        - name: artifacts
          mountPath: /artifacts
  volumes:
    - name: artifacts
      persistentVolumeClaim:
        claimName: addon-runner-artifacts
```

取回后必须记录：

```bash
sha256sum <artifact>.tar.gz
```

正式汇报至少包含：Job 名、Pod 名、artifact 文件名、sha256、测试总数、pass/fail 摘要、cleanup 状态。

## Cleanup 与审计窗口

建议策略：

- 测试命名空间 / cluster：成功后按测试策略清理；失败时先冻结证据再决定是否保留现场。
- Runner Job / Pod / PVC：保留 24h 审计窗口。
- Artifact PVC：不要频繁多 Job 并发写同一个 RWO PVC；一次只跑一个 full-run Job。

RWO PVC 是 feature，不是 bug：它能防止两个 Job 同时写 `/artifacts` 覆盖彼此。若需要多 Job 并发，必须改成每个 Job 独立 PVC 或接入对象存储。

## 验收清单

一次 host-runner 迁移不能只说"Job 跑起来了"，至少要证明：

- Runner Pod 在 host k8s，不在 Mac，也不在被测 vcluster。
- 脚本进程在 runner Pod 内执行。
- vcluster API 访问不依赖 port-forward。
- `KCFG` 来自 Secret，不来自 Mac home。
- artifact 写入 PVC，并有 sha256。
- probe Job 先通过，再跑 full-run Job。
- full-run Job 产出与旧 Mac runner 等价的业务验收结果。
- cleanup 状态明确：被测资源清理 / 保留现场 / runner audit window 各自说明。

这几条没有同时满足时，不要宣称"测试已经迁到 IDC"；最多只能说"被测集群在 IDC，但 runner 迁移未闭环"。

## 常见坑表

| 现象 | 常见原因 | 修复 |
|---|---|---|
| Job `ImagePullBackOff` | 误用 docker.io tag / IDC 不能直连外网 | 换 ACR mirror、用已缓存 tag、或提前 sideload |
| `/bin/bash: not found` | 工具镜像是 alpine / distroless / minimal | 换 e2e runner 镜像，或重写 POSIX shell；不要在 P0 迁移里临时大改脚本 |
| `kubectl: not found` | runner image 没 kubectl | initContainer 复制 kubectl 到 shared volume |
| `kubectl` 执行时报 loader missing | kubectl binary 与主容器 libc 不匹配 | 同时复制 loader，或使用同 libc 系 runner image |
| `exit 141` | `set -o pipefail` 下命令管道接 `head` 引发 SIGPIPE | 先写临时文件，再 `head` |
| Job 过了但 `kubectl cp` 失败 | completed container 无法 exec `tar` | 起 reader pod 挂同一 PVC |
| summary 里列解析错位 | `kubectl get` 表格列不稳定 | 用 go-template / jsonpath 读 label，不解析人类表格 |
| Mac 断网后测试失败 | 脚本仍在 Mac 或依赖 port-forward | 把测试执行迁到 Job；Mac 只做提交和取证 |

## 与其他文档的关系

- 跑前环境 gate：[`addon-test-environment-gate-hygiene-guide.md`](addon-test-environment-gate-hygiene-guide.md)
- kube context / proxy / shared client state：[`addon-test-script-preflight-guide.md`](addon-test-script-preflight-guide.md)
- bash / macOS / Linux portability：[`addon-test-runner-portability-guide.md`](addon-test-runner-portability-guide.md)
- 长跑 cadence：[`addon-test-runner-cadence-discipline-guide.md`](addon-test-runner-cadence-discipline-guide.md)
- 证据强度与结论边界：[`addon-evidence-discipline-guide.md`](addon-evidence-discipline-guide.md)
- Reference implementation 的 image / mirror / evidence pack 记录在 [`addon-idc-image-registry-mirror-guide.md`](addon-idc-image-registry-mirror-guide.md) §5 MariaDB 行。
