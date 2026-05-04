# Addon IDC 镜像 registry / mirror / sideload 指南

> **Audience**: addon dev / test / 本机 ops
> **Status**: draft
> **Applies to**: 任何在 IDC 共享 k8s 上跑 vcluster 的 addon 测试 / 部署工作流
> **Applies to KB version**: any（image 拉取属基础设施层，与 KB 版本解耦）
> **Affected by version skew**: 不同 chart 版本可能引入新 image source（例如新增 ghcr.io 依赖），这是 chart consumer 的责任，本文档方法论本身不变

本文面向把 addon 测试 / 部署从本地 k3d-on-Mac 搬到 IDC 共享 k8s（idc / idc2 / idc4 / 未来其他 IDC）+ vcluster 模式后，需要在 IDC 内决策镜像拉取路径的开发、测试、ops 工程师。**核心结论：不要假设单一镜像代理覆盖所有 source；按 registry 来源分流，每条来源有独立策略**。

## 先用白话理解这篇文档

最容易踩的坑：在 IDC 内试图用一个 mirror（比如 `dockerproxy.net`）覆盖所有 image，发现：
- `docker.io` 上的 image 走 mirror 拉得很顺
- `ghcr.io` 上的 image 仍然 ImagePullBackOff（mirror 不对应 ghcr.io）
- 自家 ACR 上的 image 反而走 mirror 绕远路（应该直拉）
- 大 image（几 GB）拉过几次后 mirror 也开始限速

正确做法：先列完整 image inventory → 标 source registry → 每条 source 走它对应的最优路径（mirror / 直拉 / sideload）。

把 image 当成跨 registry source 的混合品，**用决策树而不是单一规则**。

## 适用场景

- 第一次在 IDC 内 helm install addon chart，发现部分 image 拉不下来
- vcluster bootstrap 阶段（k3s control plane / syncer / cert-manager / chaos-mesh）image source 各异
- chart 引入新依赖（typically ghcr.io 上的 new tool），需决策走 mirror 还是 sideload
- 跨 IDC 复用：同一份 image 在 idc / idc2 / idc4 各 cluster 的拉取策略
- chaos-mesh 进 vcluster 内自装时，ghcr.io 的 chaos-mesh image 拉不下来

## 1. Per-source decision matrix（核心决策树）

不同 image source 走不同路径。这张表是 cross-line 标准（按 idc / idc2 / idc4 实测交叉验证）：

| Image source | 推荐 path | 理由 |
|---|---|---|
| `docker.io/*` | dockerproxy.net mirror（fallback: docker.m.daocloud.io / dockerpull.com / docker.1panel.live） | 国内 mirror 链稳定，链路有备份 |
| `apecloud-registry.cn-zhangjiakou.cr.aliyuncs.com/*` | ACR 直拉 + `imagePullSecrets: [apecloud-registry-cred]` | 阿里云内网 IDC 可达，stable tag 实测 PASS |
| `ghcr.io/*` | sideload pipeline（无可用 public mirror） | IDC 出口对 ghcr.io 长期不稳，pull 容易卡 22min+ 后 ImagePullBackOff |
| `quay.io/*` | dockerproxy.net mirror（覆盖 quay.io 路径） | 实测可达 |
| `registry.k8s.io/*` | sideload pipeline | IDC 内不稳定或不可达 |
| 自建 dev / PR-built image | sideload pipeline + `imagePullPolicy: Never` | 不在任何 public registry，必须本地落地 |

**关键 invariant**：sideloaded image 必须显式设置 `imagePullPolicy: Never`，否则 kubelet `IfNotPresent` 默认有 race（image cache GC 后会重拉外网，触发 ImagePullBackOff）。这是 chart consumer 的责任，不是 sideload 工具的责任 —— Helen `tools/sideload-image.sh` 等 sideload script 只投递镜像，`imagePullPolicy` 由 chart consumer 通过 values override 改为 `Never`。

### 1.1 docker.io mirror 链 fallback 顺序

```
dockerproxy.net      ← 主选（idc / idc2 实测稳，rancher/k3s pull 在 30 秒内）
docker.m.daocloud.io ← 一回退
dockerpull.com       ← 二回退
docker.1panel.live   ← 三回退
```

**vcluster k3s control-plane image override**（Valkey 在 idc2 实测：k3s image 29s 拉完）：

```yaml
# vcluster values override — k3s 镜像走 dockerproxy.net mirror
# Mirror chain fallback: dockerproxy.net (主) → docker.m.daocloud.io → dockerpull.com → docker.1panel.live
controlPlane:
  distro:
    k3s:
      image:
        registry: dockerproxy.net
        repository: rancher/k3s
        tag: v1.27.11-k3s1
```

**vcluster-pro image**（`ghcr.io/loft-sh/vcluster-pro:0.25.1`）：没有可用 public mirror —— 必须走 sideload pipeline（§3）。sideload 后 vcluster helm values 把 image 字段指向本地已 import 的 tag，并显式设置 `imagePullPolicy: Never`。

> **PLACEHOLDER（Bob2 fill after T04 round 2 evidence）**：sideload 后 vcluster values image override 完整示例 + syncer image 处理细节

### 1.2 ACR 直拉 + pull-secret 部署位置

`apecloud-registry-cred` 是 cross-line 标准化的 imagePullSecret 名。它需要部署到**两个位置**：

- **host k8s namespace**（runner pod 所在 ns，例如 `valkey-runner-host` / `oceanbase-runner-host`）：runner pod 拉 ACR runner image 用
- **vcluster 内 namespace**（workload 所在 ns，例如 vcluster 内的 `valkey-test`）：addon Cluster CR 创建的 Pod 拉 ACR addon image 用

> **OPEN QUESTION（PoC verify）**：vcluster syncer 是否 auto-sync `kubernetes.io/dockerconfigjson` secret 跨 host k8s ↔ vcluster 边界？预期 Oracle 4-image + SQL Server 7-image 同 ACR 拉取行为应一致。
>
> - 若 syncer auto-sync：host k8s 单 deploy 即可，§2.4 一行说明
> - 若 syncer NOT auto-sync：fallback = host secret + vcluster syncer 显式配置 `--sync=secrets,...` flag，或 vcluster 内单独 deploy 同 secret
> - 验证触发：James Phase 2 PoC vcluster bootstrap + Tom SQL Server 7-image cross-validation pod 同时拉

## 2. Practical order（实操顺序）

避免"先 sideload 大 image 后发现 mirror 也能拉"或"先用 mirror 拉小 image 后才发现大 image 必须 sideload"的反复。**正确顺序固定 5 步**：

1. **完整 image inventory** —— 列出 chart 默认 + addon 实际用到的所有 image（含 base image / init container / sidecar）
2. **标 source registry** —— 每条 image 标记其 source（`docker.io` / `ghcr.io` / `apecloud-registry.cn-zhangjiakou` / `quay.io` / `registry.k8s.io` / 自建）
3. **`docker.io` 走 mirror** —— chart values 覆盖 registry 字段为 `dockerproxy.net`
4. **非 `docker.io` 做最小镜像 dry-run sideload** —— 先 sideload 一个小 image（< 100MB）跑通整条 pipeline，验证 helper pod / chunked transfer / `ctr import` / `imagePullPolicy: Never` 链路无误
5. **批量 sideload 大 image** —— dry-run pass 后再投递大 image（OB enterprise / Oracle 主 DB image / mssql 等几 GB image）

这样能把"网络问题"和"测试产品问题"分开：dry-run sideload 失败 = 网络 / 工具链问题；批量 sideload 失败 = 个别 image 特定问题。

## 3. Sideload pipeline 4 步（canonical）

跨 line 标准化 4 步，已在 Valkey / MariaDB / SQL Server 三 line 实测：

1. **本地 export**：`docker pull --platform linux/amd64 <image>` + `docker save <image> -o image.tar`
   > 显式指定 `--platform linux/amd64`：Mac 默认 arm64，import 到 amd64 node 会被静默过滤（见 §4 Anti-patterns）
2. **分块**：`split -b 50m image.tar image.tar.part-`
   > 单块 ≤ 50MB（Bob2）或 ≤ 100MB（Helen），上限是 `kubectl exec` stdin 大流 EOF 截断阈值
3. **per-node helper pod 投递 + import**：每个 host node 起一个 `debian:12-slim` helper pod，`mount /run/containerd/containerd.sock` + `privileged: true` + `chroot`，把分块 `kubectl cp` 进去后 `cat image.tar.part-* > image.tar`，执行 `ctr -n k8s.io images import image.tar`
   > 不用 alpine：alpine 的 musl libc 与 `ctr` 二进制不兼容（同源教训：Bob2 Valkey runner pod 的 musl wrapper）
4. **post-import 校验（强制）**：同节点立即 `ctr -n k8s.io images list | grep '<repo>:<tag>'`，必须有命中
   > 不校验时，Mac arm64 → amd64 node 静默过滤场景下 import rc=0 但 image 实际不在，下游 Pod 拉时才发现 ImagePullBackOff

> **PLACEHOLDER（Bob2 fill）**：Valkey 实测 sideload helper pod manifest（含 hostPath / privileged / chroot 完整 spec）+ `ctr import` 命令链 + 跨 node 验证脚本

## 4. Anti-patterns（踩过的坑）

### 4.1 `kubectl exec` 大 stream EOF 截断
- 单 image tar 直接 `kubectl exec ... -- tar xf -` 投递，几百 MB 后 stdin stream 被截断，import 失败
- 解决：`split -b 50m`（Bob2 严格上限） / `split -b 100m`（Helen 实测上限），分块 cp 进去后再 cat 重组

### 4.2 Mac arm64 → amd64 node 静默过滤
- Mac 默认 `docker pull` 拉的是 arm64 manifest，`docker save` 出来的 tar 也只含 arm64 layer
- import 到 amd64 IDC node 时 `ctr import` 会"成功"（rc=0）但实际过滤掉所有 arm64-only image，amd64 manifest 没有 → 下游 Pod 拉时 ImagePullBackOff
- 解决：`docker pull --platform linux/amd64 <image>`，`docker manifest inspect` 验证 manifest 含 amd64 layer，`ctr images list | grep` post-import 强制后验

### 4.3 sideload 后忘改 `imagePullPolicy: Never`
- chart 默认 `imagePullPolicy: IfNotPresent`，sideload 后看似工作，但 kubelet image GC 触发后会重拉外网（特别是 ghcr.io 这种本来就是因为不可达才 sideload 的 source），race 条件下偶发 ImagePullBackOff
- 解决：sideloaded image 全部 chart values override 改 `imagePullPolicy: Never`；CI 如果有 image GC 测试，专门 cover 这条 race

### 4.4 alpine helper pod 的 musl 不兼容 `ctr`
- 临时起 alpine helper pod 跑 `ctr -n k8s.io images import` 失败，报 ELF interpreter `/lib/ld-musl-x86_64.so.1` 缺失或 segfault
- 根因：`ctr` 二进制是 glibc-linked，alpine 是 musl
- 解决：用 `debian:12-slim` 作 helper image base

### 4.5 假设单一 mirror 覆盖所有 source
- 看到 `dockerproxy.net` 拉 docker.io 顺，错误推断"配置一个 mirror 就行"，结果 ghcr.io / registry.k8s.io 仍然 fail
- 解决：本文档 §1 决策树，按 source 分流

## 5. Cross-engine reuse status

各 line 在自家 chart 上的 source 分布 + 实测路径选择。**两个 family**：

- **§1 dockerproxy.net mirror family**（chart 主体走 docker.io）：Valkey / MariaDB
- **§3 ACR direct family**（chart 主体走 apecloud-registry.cn-zhangjiakou）：SQL Server / Oracle / OceanBase / KBE

| Engine line | 主导 path | docker.io path | ACR path | ghcr.io path | 大 image sideload | dev/PR image | Owner / Notes |
|---|---|---|---|---|---|---|---|
| Valkey | §1 mirror | dockerproxy.net 主体 | runner image (kubeblocks-tools / e2e-ginkgo) | chaos-mesh sideload (vcluster 内自装时) | 无（chart image 都不大）| n/a | Alice / Bob2 — reference impl |
| MariaDB | §1 mirror | dockerproxy.net 主体 | runner image | chaos-mesh sideload | 无 | n/a | **PLACEHOLDER（Helen / Jack fill）** |
| SQL Server | §3 ACR | n/a (chart 无 docker.io) | 7-image 全 ACR cn-zhangjiakou stable tag：`apecloud/mssql:{2022-CU19, 2022-CU14, 2025-latest, 2019-CU30, 2019-CU29}` (~1.4GB ea) + `apecloud/syncer:0.7.2` (~50MB) + `apecloud/prometheus-mssql-exporter:1.3.2` (~150MB)；总 ~7GB 全版本或 ~3GB 单 major | n/a 直接（chart 0 ghcr.io 引用，已 audit `templates/_helpers.tpl` + `cmpv.yaml`）；间接通过 vcluster-internal chaos-mesh Phase 2 自装时触发 | 无 chart-side 主体 sideload；chaos-mesh ghcr.io image 走 §1 决策树第 3 行（ghcr.io → sideload pipeline）| n/a 当前（CU 编号 + 版本号锁定，0 dev/PR-built image）| Tom / Jerry — idc2 ACR probe（busybox spider，2026-05-04）：HTTP 401 + `Docker-Distribution-Api-Version: registry/2.0` + `WWW-Authenticate: Bearer realm=https://dockerauth-cn-zhangjiakou.aliyuncs.com/auth` PASS；`apecloud/netshoot:latest` 已 cache 在 idc2 node1+node3 |
| Oracle | §3 ACR | n/a（chart 无 docker.io 主流依赖） | 4-image enterprise + runner toolchain（apecloud-registry.cn-zhangjiakou，含 oracle:12.2.0.1-ee 几 GB） | n/a 直接；间接 chaos-mesh + cert-manager + vcluster syncer | Oracle 主 DB image 几 GB（Phase 2 PoC 走 §3 sideload 路径） | n/a 当前 | James / John — chart 0 ghcr.io / docker.io 依赖（已 audit）|
| OceanBase | §3 ACR + 大 image sideload | n/a | OB enterprise image (~3GB) 走 ACR；blocker = westonnnn pending push/pull credentials | chaos-mesh sideload Phase 2 | OB enterprise ~3GB 必走 §3 sideload pipeline | n/a 当前 | Noah / Mia — idc4 |
| KBE | §3 ACR + sideload 4 dev images | n/a（chart 无 docker.io 依赖） | cr4w / redis / hook / mysql 等 stable image 走 ACR 直拉；apiserver / dms / openconsole 4 件套为本地 source build (`local/*`) 走 sideload | vcluster-pro:0.25.1 — 初次拉取极慢（实测 idc2 约 90min 最终成功，不可靠）→ 后续必须预先 sideload | 4 dev image (apiserver / dms / openconsole-default / openconsole-admin) 全走 sideload pipeline（amd64 rebuild + helper pod + ctr import）；5th image: KBE runner (debian:12-slim + node 18 + Playwright + chromium) | KBE chart 不支持 `--set images.registry=local`（单 anchor 破 cr4w/redis），必须 post-install `kubectl set image` patch 4 个 deployment + `imagePullPolicy: Never` | Ava / Ben — idc2，sideload pipeline 2026-05-04 实测 OK（helper pod: kb-helpers ns，3 nodes） |

> 各 line owner 在 PR review 阶段填具体 cell 里的实测数据点（image tag / pull 时间 / 实测命令）。

## 6. 跨 IDC 实测可达性

每 IDC 对各 mirror / source 的实测连通性 + 选择差异：

| IDC | docker.io via dockerproxy.net | apecloud-registry.cn-zhangjiakou ACR | ghcr.io 直拉 | sideload pipeline 已就位 |
|---|---|---|---|---|
| idc | ✅ k3s pull 30s | ✅ stable tag PASS | ❌ 不稳 | ✅ Helen + Bob2 已 paved |
| idc2 | ✅ rancher/k3s 1.4s（实测）| ✅ HTTP 401 + registry/2.0 PASS（idc2 node1+node3 已 cache `apecloud/netshoot:latest`）| ⚠️ vcluster-pro 初次拉取约 90min 最终成功（极慢，不可靠；实测 2026-05-04）→ **后续必须预先 sideload** | ✅ 3-node helper pod (`kb-helpers` ns，debian:12-slim + privileged + hostPath /run/containerd/containerd.sock) 实测 OK（2026-05-04 KBE team 验证）|
| idc4 | **PLACEHOLDER（Mia / Noah fill）** | **PLACEHOLDER（Mia / Noah fill）** | **PLACEHOLDER** | **PLACEHOLDER** |

## 7. Pull-secret 标准化

跨 line 标准化的 imagePullSecret 配置：

- **Secret 名**：`apecloud-registry-cred`（cross-line 一致）
- **类型**：`kubernetes.io/dockerconfigjson`
- **部署位置**：host k8s 的 runner namespace + vcluster 内 workload namespace（参见 §1.2 OPEN QUESTION）
- **凭据来源**：westonnnn 通过受控渠道下发（不在文档里直贴）
- **Pod spec 引用**：`imagePullSecrets: [{name: apecloud-registry-cred}]`，所有引用 ACR image 的 Pod 必须显式引用

> **PLACEHOLDER（Tom / James fill after PoC）**：vcluster syncer 跨层 secret sync 实测结论 + 双层 deploy 标准化 manifest

## 8. 相关文档

- 上游 methodology / decision tree：[`addon-idc-vcluster-migration-checklist-guide.md`](addon-idc-vcluster-migration-checklist-guide.md) (TBD, Alice)
- 下游 bootstrap recipe：[`addon-vanilla-vcluster-bootstrap-guide.md`](addon-vanilla-vcluster-bootstrap-guide.md) (TBD, James + Noah co-author)
- 单机本地 image import（k3d 场景，非 IDC）：[`addon-k3d-image-import-multiarch-workaround-guide.md`](addon-k3d-image-import-multiarch-workaround-guide.md)

三篇 doc 互引 topology：

```
addon-idc-vcluster-migration-checklist-guide.md (Alice)
  ├─ §pre-flight: image inventory + sideload 决策 → cite 本文
  └─ §runner placement decision tree → cite addon-vanilla-vcluster-bootstrap-guide.md §2.5

addon-vanilla-vcluster-bootstrap-guide.md (James + Noah)
  ├─ §2.4 image registry mirror 接入 → cite 本文 §1 决策树（不重抄方法论）
  └─ §2.5 chaos-mesh placement → cite addon-idc-vcluster-migration-checklist-guide.md

本文档 (Alice + Bob2 co-owner，Allen 起骨架)
  ├─ §5 Cross-engine reuse status table 各 line 行 → 各 line owner fill
  ├─ §6 跨 IDC 实测可达性 → 各 IDC owner fill
  └─ §1.2 / §7 OPEN QUESTION → Phase 2 PoC verify (James + Tom 共 sign)
```

## 9. 文档完成度（draft 状态）

本骨架已落地 §1 决策树 + §2 实操顺序 + §3 sideload pipeline 4 步 + §4 反模式 + §5/§6 cross-line 矩阵框架 + §7 pull-secret 标准化框架。**待 fill**：

- §1.1 vcluster helm chart values 完整 image override 示例（Bob2 / Alice）
- §3 sideload helper pod 完整 manifest（Bob2）
- §5 各 line 行（Helen / Tom / James / Mia&Noah / Ben）
- §6 idc4 行（Mia / Noah）
- §7 vcluster syncer secret sync 实测结论（Tom + James 共 sign，PoC verify）

当所有 PLACEHOLDER 收敛 + OPEN QUESTION 闭环后，本文档从 `draft` 升 `stable`。
