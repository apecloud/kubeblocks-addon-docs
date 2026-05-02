# Addon 测试环境 k3d Host Precheck 指南

> **Audience**: addon dev / test / 本机 ops
> **Status**: stable
> **Applies to**: 任何在 macOS / Linux 单机用 k3d 跑 addon 测试的工作流
> **Applies to KB version**: any（host 层工具，与 KB 版本解耦）
> **Affected by version skew**: no（k3d v5 label 变更已在工具层处理）

本文面向在 k3d 上跑 KubeBlocks addon 测试的开发、测试、本机 ops 工程师。新建 / 复用 k3d 集群跑 smoke / chaos 之前，**主机层面的 k3d 集群健康必须先快速 snapshot 一遍，再启 runner**。本文给出 host precheck 的工具、三档输出、已知坑、runner 集成 pattern，以及跨主机 ops profile 的写法。

## 先用白话理解这篇文档

最容易被误判的一句话：

- 测试 T1 / 创建集群 / kubectl 不通 → 第一反应往往是「addon 没装好」「KB controller 挂了」
- 但根因经常落在 host 层：另一个集群把 CPU / MEM 吃满 / API 不通 / kubeconfig 0.0.0.0 / k3d label 变更没识别
- precheck 工具在 runner 之前跑 5s，把这些 host-level 故障一次 snapshot 出来，先**让 first blocker 不要落在 host 层**

把"host 健康"当成 runner 的前置 gate，在装 KB / 装 addon 之前就过一遍。

## 适用场景

- 跑任何 smoke / chaos / regression 之前
- 主机重启 / Docker Desktop 重启 / k3d 集群被动恢复之后
- 有多个 k3d 集群同主机共存（kbe-dev / kbe-smoke / oracle-test 这种）
- runner 在 T1 / 创建集群 / kubectl get ns 阶段就 timeout / EOF / context deadline
- 多人共享同一台测试机，要快速判定「是不是别人的集群把 host 吃满了」

## 工具：scripts/k3d-precheck.sh

仓库路径：[`kubeblocks-tests/scripts/k3d-precheck.sh`](https://github.com/apecloud/kubeblocks-tests/blob/main/scripts/k3d-precheck.sh)

输出 host 上所有 k3d 集群的快照：

- API 可达性 (pass / fail) + latency
- CPU / MEM 水位（per cluster，docker stats 按 `k3d.cluster` label 聚合，单位 MiB）
- Stuck 检测：API 不通 + 容器已跑 > STUCK_THRESHOLD 才标 YES（默认 60min，避免把"刚起还在初始化"误判 stuck）

### 三档输出模式

```bash
./k3d-precheck.sh                      # 默认人类可读表格
./k3d-precheck.sh --json               # newline-delimited JSON（runner 消费）
./k3d-precheck.sh --quiet              # 只设 exit code（集成用法）
./k3d-precheck.sh --stuck-min 30       # 调整 stuck 阈值（默认 60）
./k3d-precheck.sh --timeout 5          # 调整 API 超时（默认 10s）
```

### 退出码语义

| Exit | 含义 | runner 应做什么 |
|------|------|----------------|
| 0 | 全部 cluster 健康 | 继续启 runner |
| 1 | ≥1 cluster API 不通或 stuck | 停 runner，输出 host snapshot 给操作人 |
| 2 | host 上没有 k3d 集群 | 决策：自动创建 vs 报错退出 |

## 已知坑

这三条都是 tooling 层的通用陷阱，单写一节是因为它们不绑定 addon、不绑定主机、未来工具迭代时会被反复踩到。

### 坑 1: `k3d kubeconfig get` 输出 `https://0.0.0.0:<port>` → TLS EOF

`k3d kubeconfig get <cluster>` 默认把 `serverHost` 写成 `0.0.0.0`，但 macOS / 部分 Linux 内核版本下 **`0.0.0.0` 不是合法的 client 连接目标**。在脚本里直接 pipe 给 kubectl 会 TLS EOF。

修法：脚本生成 per-cluster kubeconfig 时统一 sed 替换：

```bash
k3d kubeconfig get "$cluster" | sed 's|https://0\.0\.0\.0:|https://127.0.0.1:|g' > "$tmp"
```

详细原理见 [`addon-k3d-kubeconfig-loopback-fix-guide.md`](addon-k3d-kubeconfig-loopback-fix-guide.md)。

### 坑 2: k3d v5 label key 是 `k3d.cluster`（点）不是 `k3d-cluster`（连字符）

docker stats 按 cluster 聚合时，filter 必须用：

```bash
docker ps --filter "label=app=k3d" --filter "label=k3d.cluster=${cluster}"
```

写成 `k3d-cluster=` 会查不到任何容器，CPU/MEM 全部归零，precheck 漏报"另一个集群在吃满 host"。

这是 k3d v5 的 breaking change，文档里没强调过。任何依赖 k3d 容器 label 的工具都要复核这一点。

### 坑 3: macOS bash 3.2 没有 `mapfile` / `declare -A`

如果 precheck 写成 bash 脚本，BSD bash 3.2 跑不动。两个选择：

- 改用 zsh shebang（macOS 默认 shell 是 zsh，已自带）
- 用临时文件替代关联数组聚合 per-cluster 结果

本工具走 zsh + 临时文件方案。详细 portability 坑见 [`addon-test-runner-portability-guide.md`](addon-test-runner-portability-guide.md)。

## Runner 集成 pattern

### Pre-hook（推荐）

```bash
if ! scripts/k3d-precheck.sh --quiet; then
  echo "[runner] host k3d 不健康，停 runner — 跑一次 scripts/k3d-precheck.sh 看 snapshot"
  exit 1
fi
```

### Pre-hook + JSON snapshot 落盘（chaos / 长跑场景）

```bash
snapshot_path="evidence/run-$(date +%Y%m%d-%H%M%S)/host-precheck.json"
mkdir -p "$(dirname "$snapshot_path")"
scripts/k3d-precheck.sh --json > "$snapshot_path" || {
  echo "[runner] host 不健康 — snapshot 已保存到 $snapshot_path"
  exit 1
}
```

落盘 snapshot 的好处：runner 报"T1 timeout"时第一反应就是去 evidence 目录看 host 在那一刻的状态，避免把 host 故障误归为产品故障。

## 输出语义 → 行动

| API | STUCK | 含义 | 行动 |
|-----|-------|------|------|
| OK | no | 该 cluster 健康 | 继续 |
| FAIL | no | 刚起 / 暂时不可达 | 等几分钟再试，不是 stuck |
| FAIL | YES | 长时间 API 不通 → 已 stuck | k3d cluster delete + 重新创建 |
| OK | no, CPU% > 80 | host 资源吃紧 | 评估是否要停其他集群 |
| OK | no, MEM 接近 host 上限 | host 资源吃紧 | 同上 |

## 跨主机 ops profile

本节记录团队三台测试机的 k3d 环境基线，用于：
- 快速判断某台机器的资源余量
- 跨机协调测试时避免因"对方 Docker context 不同"看不到对方容器而误判

> **注意**：三台机器分别使用独立的 Docker daemon。在 Machine C 上运行 `docker ps` 看不到 Machine A/B 的容器，反之亦然——这不是故障，是隔离设计。跨机 precheck 需要 SSH 到目标机后运行脚本。

### Machine A — MacBook-Pro-7.local

| 属性 | 值 |
|---|---|
| 机型 | MacBook Pro |
| 主机名 | MacBook-Pro-7.local |
| 负责人 | Jason |
| Docker | Docker Desktop（context: `desktop-linux`） |
| Docker socket | Docker Desktop socket（非 `/var/run/docker.sock`） |
| Host total memory | 32 GiB |
| Docker VM memory | 15.61 GiB（Desktop 分配） |
| k3s version | v1.27.4+k3s1 |

**k3d 集群拓扑**：

| 集群 | 节点拓扑 | KubeBlocks | 用途 |
|---|---|---|---|
| `kb-local` | 1 server + 3 agents + LB | custom build (r08a-on-main) | Valkey chaos 测试主力环境 |
| `oracle-test` | 1 server + LB | — | 闲置，无 workload |

**典型内存水位**（Valkey 12h chaos 测试高负载中）：

```
k3d-kb-local-server-0：2.44 GiB / 15.61 GiB limit
k3d-kb-local-agent-1： 1.06 GiB / 15.61 GiB limit
k3d-kb-local-agent-0：  583 MiB / 15.61 GiB limit
k3d-kb-local-agent-2：  591 MiB / 15.61 GiB limit
k3d-kb-local-serverlb：  26 MiB
```

空闲状态下 server-0 通常低于 1.5 GiB，agent 节点各约 400 MiB。

### Machine B — caoweis-MacBook-Air.local

| 属性 | 值 |
|---|---|
| 机型 | MacBook Air |
| 主机名 | caoweis-MacBook-Air.local |
| 负责人 | Lucas |
| Docker | Docker Desktop |
| Docker socket | `/var/run/docker.sock` → symlink → `/Users/wei/.docker/run/docker.sock` |
| Host total memory | 16 GiB |
| k3d cluster nodes memory limits | server-0: 8 GiB / agent (kb-worker-1-0): 6 GiB |

> **Symlink 陷阱**：Machine B 上 `/var/run/docker.sock` 是 symlink 指向 Docker Desktop socket，不是传统 Linux daemon socket。其他机器若用 `/var/run/docker.sock` 硬编码路径 SSH 连过来，行为跟预期一致；但 `docker context` 若指向 `default` context，在 Machine B 上等同于 Docker Desktop。

**k3d 集群拓扑**：

| 集群 | 节点拓扑 | KubeBlocks | 用途 |
|---|---|---|---|
| `kb-local` | 1 server (8Gi) + 1 agent kb-worker-1-0 (6Gi) + LB | v1.0.0 chart / v1.0.3-beta.5 image + OceanBase addon v1.0.3 | OceanBase 测试主力环境 |

**典型内存水位**（OB 测试进行中）：

```
k3d-kb-local-server-0：  2.1 GiB / 7.75 GiB limit
k3d-kb-worker-1-0：    256.7 MiB / 6 GiB limit
k3d-kb-local-serverlb：  3.6 MiB / 7.75 GiB limit
```

### Machine C — weideMacBook-Air.local（黑色的 mac air）

| 属性 | 值 |
|---|---|
| 机型 | MacBook Air |
| 主机名 | weideMacBook-Air.local |
| 负责人 | Jeff |
| Docker | Docker Desktop（active context: `desktop-linux`） |
| Docker socket | `unix:///Users/wei/.docker/run/docker.sock` |
| Host total memory | 11.67 GiB |
| k3d version | v5.8.3 |
| k3s version | v1.33.6+k3s1 |

**k3d 集群拓扑**：

| 集群 | 节点拓扑 | KubeBlocks | 用途 / 备注 |
|---|---|---|---|
| `kbe-dev` | 1 server（control-plane）| v1.0.2 | addon 基线复测，当前供 SQL Server 线 A1 |
| `kbe-smoke` | 1 server + 1 agent | v1.0.3-beta.5 | KBE Enterprise 开发环境（`kb-cloud` namespace）；**不要未经授权 pause/stop** |
| `oracle-test` | 1 server + 1 agent | v1.0.0 | Oracle addon 专用；**不要碰** |

**典型资源水位**（2026-05-02 baseline，无 mssql / oracle pod 负载）：

```
 k3d Host Precheck — 2026-05-02T06:30:30Z
 Docker Host : unix:///Users/wei/.docker/run/docker.sock

CLUSTER                API    LATENCY   CPU%     MEM(MiB)   STUCK  NOTE
────────────────────────────────────────────────────────────────────────────────────────
kbe-dev                OK     61ms      6.0%     1801       no
kbe-smoke              OK     101ms     37.2%    4973       no
oracle-test            OK     78ms      27.6%    1566       no

 Result: ALL CLEAR — safe to start tests
```

三集群合计约 8.3 GiB，host 剩余 ~3.4 GiB（无大规模 addon 测试时）。最小内存机器（11.67 GiB），3 副本 AG 测试（6-8 GiB）需协调释放资源。

## 适用边界

- macOS arm64 / x86_64（已验证）
- Linux 主流发行版（k3d 本来就主要在 Linux 跑，没遇到过反例）
- k3d v5+（label key 是 `k3d.cluster`）
- 不替代 cluster 内部健康检查（pod / node ready / CSI / 镜像缓存等仍按 [`addon-test-environment-gate-hygiene-guide.md`](addon-test-environment-gate-hygiene-guide.md) 走）
- 不替代 addon 内部健康检查（probe / role / spfile / DG 链等仍按各自 guide 走）

## 沉淀给团队的检查清单

对照下面清单，确认你这台测试机的 host precheck 已经 land：

- [ ] `scripts/k3d-precheck.sh` 在 `kubeblocks-tests` 仓里 + executable
- [ ] 在自己机器上跑过一次 `./k3d-precheck.sh`，确认输出格式正常
- [ ] 在自己机器上跑过 `--json` + `--quiet` 模式，确认 exit code 符合预期
- [ ] 在 runner 里加 pre-hook（`--quiet` 或 `--json` 落盘）
- [ ] chaos / 长跑场景的 evidence 目录里能找到 `host-precheck.json` snapshot
- [ ] 跨主机 ops profile 那节已有自己机器的视角

## 7 条硬规则

1. host precheck 必须在 runner 创建集群 / 装 KB 之前跑，不能放后面。
2. precheck 失败必须**先停 runner**，不能因为"也许是暂时的"就继续 — 环境不健康跑出来的失败无法归类。
3. precheck 工具不允许依赖 KUBECONFIG 全局变量，必须 per-cluster 生成临时 kubeconfig（避免污染调用方）。
4. 任何 docker / k3d label 假设必须在工具内部隔离（例如 `k3d.cluster` 这种 v5 breaking change），不能扩散到调用方。
5. precheck 不允许做 destructive 动作（不许 stop / restart / delete），只读 snapshot。
6. STUCK 检测必须有时间下限（默认 60min），避免把"刚起"误判 stuck。
7. JSON 输出必须 stable（字段名稳定、顺序稳定），便于 runner / dashboard 长期消费。

## 反模式

| 反模式 | 后果 |
|--------|------|
| runner 直接调 `kubectl` 不过 precheck | host 故障被归类成 addon 故障，调试方向错 |
| precheck 失败时 runner 继续跑 | 第一波 evidence 全部污染，事后归因混乱 |
| precheck 工具自己装 destructive cleanup（自动 delete stuck cluster） | 误删别人的工作集群，权限 escalate 到 ops 层 |
| 用 docker stats 不带 cluster filter | 一个集群的水位被算到其他集群头上 |
| precheck JSON 字段不稳定 | dashboard / runner 长期集成会反复 break |

## 待补 (TBD)

- 多 host SSH 远端 precheck 模式（一台 ops 机扫所有测试机），等真有 cross-host runner 再做
- ARM64 vs x86_64 docker stats 行为差异 audit（目前都跑得通，无需特殊处理）
- 与 KB Enterprise dataprotection / backup 的 host-side 资源水位 baseline 对齐
