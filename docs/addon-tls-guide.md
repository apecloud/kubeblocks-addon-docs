# Addon TLS 开发与验证指南

本文面向 KubeBlocks Addon 开发者，聚焦 TLS 能力的设计、验证与排障。通用正文只描述 Addon TLS 的共性问题；引擎、协议、命令行工具和拓扑细节只放在案例附录中。

## 先用白话理解这篇文档

### 这篇文档解决什么问题

实现 TLS 时最常踩的坑不是"加密配错了"，而是**"看起来通了，但安全边界其实没建立"**。

新读者通常用一条简单口径验证 TLS：客户端能用 TLS 连上 + 收到正常返回 → 通过。但这条口径**会漏 4 类关键安全问题**：

1. **明文端口仍开着** —— 客户端用 TLS 连上同时，攻击者也可以用明文连上，整个加密形同虚设
2. **内部组件之间没走 TLS** —— primary/secondary 间复制流是明文，sidecar 拉指标是明文，TLS 只是"对客户端可选项"而不是"系统级"
3. **证书来源不固定** —— 用 self-signed dev cert 跑通一次就当 OK，没验证证书 chain / 来源 / 信任配置
4. **TLS 与其他能力组合时退化** —— reconfigure / backup / restore / switchover 走 TLS 路径时可能失败或 silently 退回明文

→ 真正的方法论是：**TLS 不是单点能力，而是产品级安全边界**。验证时必须分别验加密链路通 + 明文链路被拒 + 内部链路被加密 + 跨能力组合时不退化，缺一不能算 TLS 通过。

### 何时本文方法论 apply

| 场景 | 关键决策 |
|---|---|
| 设计新 addon TLS 能力 | 先回答 6 个产品语义问题（强制 vs 可选 / 明文是否禁 / 内部通信是否走 TLS / 证书来源 / 轮换是否重启 / 失败如何暴露） |
| 写 TLS 测试 case | 加密 + 明文拒 + 内部加密 + 跨能力 4 个独立验证点，不能只验"客户端能连" |
| TLS 报"连接失败" | 按排障流程 5 步走（声明 / 证书 / 启动参数 / 连接行为 / 状态收敛），不要直接归到 DB 故障 |
| Pod Ready 但 TLS 连接 fail | 怀疑 socket / 端口 / 证书路径 / 鉴权 4 条，不要默认 DB bug |
| Sidecar / probe 撞 "TLS 工具不可用" | 大概率 sidecar 镜像没装 OpenSSL / TLS 客户端，跟 sidecar 镜像 ABI 主题相关 |
| TLS + reconfigure / backup / switchover 组合 | 显式测组合路径，不假设单 TLS case 通过就组合也通过 |

### 读完你能做什么决策

- **设计阶段**：能 5 分钟内列出 6 个产品语义问题各自的答案，避免实现期反复重写
- **写 TLS case 时**：能区分"加密 OK"和"安全边界 OK"两件事
- **撞 TLS fail 时**：能按 5 步排障流程定位，不在 DB / 配置 / 存储层瞎找
- **review TLS PR 时**：能识别"只验加密通"的浅 case 并 block

### 为什么独立成篇

TLS 的核心问题是**安全边界**，不是 reload 路径或幂等语义。把 TLS 跟 reconfigure / switchover / backup 等主题分开成独立一篇，避免新读者在 reconfigure 文档里漏掉 TLS 安全维度。

具体引擎实现（Valkey TLS / Redis TLS / MariaDB TLS / 各引擎 sidecar 镜像 ABI 限制）只放在案例附录里，正文是引擎无关的方法论。

---

## 适用场景

当 Addon 需要支持以下能力时，这份文档适用：

- 创建启用 TLS 的集群。
- 在集群内部组件之间使用加密连接。
- 对外暴露 TLS 访问入口。
- 禁止明文连接，或同时兼容 TLS 与明文连接。
- 将 TLS 与 reconfigure、backup / restore、switchover、upgrade 等能力组合验证。

## 先定义 TLS 产品语义

实现 TLS 前，建议先明确以下问题：

1. TLS 是可选能力还是强制能力？
2. 开启 TLS 后，明文端口是否仍然开放？
3. 组件之间的内部通信是否也必须走 TLS？
4. 客户端证书、服务端证书和 CA 的来源是什么？
5. 证书轮换后是否需要重启实例？
6. TLS 配置失败时，Cluster / OpsRequest 应如何暴露状态？

不要只验证“客户端能连上”。TLS 能力的关键是明确安全边界：哪些路径必须加密，哪些路径允许明文，哪些失败应被视为预期拒绝。

## 证书与 Secret 检查

TLS 问题经常不是业务进程本身的问题，而是证书、Secret、挂载路径或权限问题。

建议固定检查：

- Secret 是否存在。
- Secret 中是否包含预期 key。
- 证书是否未过期。
- 证书 SAN 是否覆盖实际访问域名或地址。
- Pod 中的挂载路径是否正确。
- 证书文件权限是否允许业务进程读取。
- 多个组件是否引用同一套 CA 或兼容的信任链。

如果证书来自外部系统，还应记录证书签发方式、更新方式和轮换后的生效路径。

## 连接路径要分开验证

TLS 验证应至少区分三类路径：

- 客户端到服务入口。
- 集群内部成员之间。
- 控制脚本、sidecar、agent 或运维工具到业务进程。

这些路径的执行上下文不同，可能使用不同的证书路径、主机名、端口和工具。不要因为其中一条路径成功，就推断所有 TLS 路径都正确。

## 明文拒绝也是正向验证

如果 Addon 的目标是 TLS-only，那么明文连接失败是必须验证的正向结果。

验证时应记录：

- 明文连接使用的地址和端口。
- 明文连接的失败表现。
- 失败是否发生在预期阶段。
- 是否存在绕过 TLS 的备用入口。

如果明文连接仍然成功，应明确这是产品预期、兼容模式，还是安全边界缺陷。

## 组合能力验证

TLS 通常会影响其他 Addon 能力，建议把组合验证作为独立用例，而不是混进 TLS 基础用例中。

常见组合包括：

- TLS + reconfigure：参数变更后，TLS 连接路径是否仍正常。
- TLS + backup / restore：备份和恢复工具是否使用正确证书。
- TLS + switchover：角色切换或拓扑变化后，客户端和内部连接是否恢复。
- TLS + upgrade：滚动升级过程中证书挂载和连接行为是否稳定。
- TLS + external access：外部入口的证书、域名和端口是否匹配。

组合验证的结论应单独记录，避免把基础 TLS 能力和其他能力的失败原因混在一起。

## 证书轮换：双层 Secret 模型下的 silent failure trap

KB 渲染 TLS 时，常见的 Secret 拓扑是双层结构：

- **Source secret**：用户提供（UserProvided issuer）或 issuer 直接生成，承载原始 cert / key / CA。
- **Derived secret**：KB controller 渲染后挂载给业务 pod 实际使用的 Secret。

新读者经常默认"轮换证书 = 改 source secret"。在双层模型下这个直觉会踩坑，且失败方式是 silent 的，需要专门设计验证口径。

### 三个容易踩的传播假设

1. **改 source secret 不一定 propagate 到 derived secret**：是否 propagate 取决于 issuer 模式与 controller 监听规则。UserProvided 模式下，source secret 通常被视作"一次性输入"，controller 不再 watch 它的更新。看到 derived secret 长时间不动，是设计上不会动，不是同步延迟。
2. **改 derived secret 会触发 mount sync，但不会重启业务进程**：kubelet 几十秒内把新文件写到 pod 文件系统，然而业务进程仍持有旧证书 in-memory。客户端短连接走新证书，长连接（包括 inter-pod 复制、sidecar 长连接）继续走旧证书，行为间歇性不一致。
3. **pod readiness 通过不代表证书已生效**：readiness probe 通常只验进程存活或最小响应，不重新握手。pod Ready + secret 已挂 ≠ 业务进程已加载新证书。

### 推荐轮换路径

```
patch derived secret → 等 mount sync 完成 → rolling restart pod → 业务进程重新加载 → 验证四个口径
```

只做前两步而省略 rolling restart，是这个 trap 的典型现场。

### 验证证书真的轮换的四个口径

固定按这四层独立验证，缺一层就把"轮换通过"的结论收紧到对应层级：

1. **derived secret 内容已变**：基于 `tls.crt` 字节比较，或解析 NotBefore / NotAfter / Issuer / Serial。
2. **pod 内挂载文件已更新**：在容器里读挂载路径下的 cert，验证内容与 derived secret 一致。
3. **业务进程持有的证书已更新**：通过业务管理命令、运行期 API 或重启日志确认进程已加载新证书。这是最容易被跳过的一层。
4. **内部组件之间已重新握手**：复制链路、sidecar 长连接、agent 通道是否都是用新证书。

只验证 #1 / #2 容易把 trap 当成"通过"。

### 不要做的几件事

- 只改 source secret 后等待，期望 derived 自动同步。
- patch derived 后立刻判断"已轮换"，不做 rolling restart。
- 只重启一个 pod 不覆盖整个拓扑（primary 漏了 / replica 漏了 / sidecar 漏了都属于未完成轮换）。
- mount sync 超时时直接重建 cluster — 这会丢数据状态，且不解决 kubelet 层的 sync 阻塞，应先看 kubelet event 是否报 SecretSyncFailure。

### 跨能力组合时的注意事项

- 轮换叠加 reconfigure / switchover / backup 时，应先轮换、再触发其他动作，避免组合路径里同时出现旧证书与新证书。
- 滚动重启时应配合既有的 switchover 或顺序 restart 能力，避免在重启窗口内造成读写不可用。

## 推荐排障流程

### 1. 确认 TLS 声明

- Addon 是否声明了 TLS 开关。
- 组件模板是否正确消费 TLS 配置。
- 最终渲染出的资源是否包含 TLS 相关字段。
- 集群中的实际 CR 是否与源码模板一致。

### 2. 确认证书材料

- Secret 是否存在。
- 证书、key、CA 是否完整。
- 挂载路径是否存在。
- 文件权限是否正确。
- 证书 SAN 与访问地址是否匹配。

### 3. 确认启动参数

- 业务进程是否读取了 TLS 参数。
- TLS 端口是否监听。
- 明文端口是否符合预期开启或关闭。
- 日志中是否有证书加载失败、权限错误或协议错误。

### 4. 确认连接行为

- TLS 客户端连接是否成功。
- 读写或最小业务请求是否成功。
- 内部成员通信是否正常。
- 明文连接是否按预期成功或失败。

### 5. 确认状态收敛

- Cluster 是否 Running。
- Pod 是否 Ready。
- 相关运维对象是否 Succeed。
- 重启、升级、切换或恢复后 TLS 行为是否保持一致。

## 建议固定收集的验证项

每次验证 TLS 行为时，建议固定回报以下内容：

1. TLS 开关配置。
2. Secret 名称和关键字段是否存在。
3. 证书有效期和 SAN 检查结果。
4. Pod 中证书挂载路径。
5. TLS 端口监听结果。
6. TLS 连接命令和返回结果。
7. 最小读写或业务请求结果。
8. 明文连接测试结果。
9. Cluster / Pod / OpsRequest 最终状态。
10. 相关日志或附件路径。

## 常见问题

### 证书存在但连接失败

优先检查：

- 客户端是否信任正确 CA。
- 客户端访问的域名是否在证书 SAN 中。
- 证书和 key 是否匹配。
- 服务端是否实际加载了最新证书。
- 客户端工具是否默认启用了 SNI 或主机名校验。

### Pod Ready 但 TLS 连接失败

Pod Ready 只说明 readiness probe 通过，不一定说明 TLS 业务路径可用。

继续检查：

- readiness probe 是否绕过了 TLS。
- 业务端口与 probe 端口是否一致。
- TLS 端口是否真正监听。
- 业务日志是否出现 handshake 失败。

### TLS 工具在 sidecar 中不可用

排障脚本或 reload 脚本可能运行在 sidecar，而不是业务主容器。

需要确认：

- 当前容器是否有对应 CLI。
- CLI 依赖库是否满足。
- 证书是否挂载到当前容器。
- 当前容器能否访问目标地址和端口。

## 案例附录：Valkey R09 TLS

Valkey R09 验证的是 Valkey TLS-only 能力是否按预期工作。本附录保留引擎相关术语和命令，避免污染通用正文。

验证结果：

- 自动化回归结果为 R09 `13 PASS / 0 FAIL`。
- TLS 集群创建后进入 Running。
- 拓扑健康，包含 1 master / 2 replicas。
- `valkey-cli --tls PING` 返回 `PONG`。
- 通过 TLS 通道写入 20 个 key，并在 3 个 data pod 上校验通过，说明复制链路正常。
- 明文连接被拒绝，表现为 I/O error / connection reset，而不是返回 `PONG`。

结论：

- TLS-only 强制执行验证通过。
- 该能力可作为独立回归信号，不并入 reconfigure 主结论。
- 测试脚本：`tests/r09-tls.sh`。

相关交付记录：

- Feature issue: https://github.com/apecloud/kubeblocks-addons/issues/2591
- PR: https://github.com/apecloud/kubeblocks-addons/pull/2592
- 验证环境：EKS `ap-southeast-1`，KubeBlocks `1.0.2`

## 案例附录：Valkey G7 TLS 证书轮换

G7 验证的是 Valkey TLS 集群在生产中做证书轮换时的实际行为。本附录保留 KB 双层 Secret 的具体名字与命令，验证口径只放方法论描述。

### 现场设置

- TLS-only 集群（`tls: true`，UserProvided issuer，shared CA 配置正确）。
- 集群为 1 primary + 2 replicas，进入 Running 后可正常 TLS PING。
- 起始 cert NotAfter 与新 cert NotAfter 之间相差明显（用于验证 #1 口径）。

### 试错路径与发现

1. **方案 A：rotate source secret，等 derived 同步**。`kubectl patch` 把新 cert 写入 source secret 后 30 分钟，derived secret 内容未变。原因：UserProvided issuer 模式下 KB controller 不 watch source secret 的更新，derived 是从 source 渲染一次后稳定。这条路径在双层模型下设计上不会成立。
2. **方案 B：直接 patch derived secret，不重启 pod**。derived secret 内容更新后 ~20s，pod 内挂载文件即更新，可被 `openssl x509 -noout -dates` 解析为新 NotAfter。但 `valkey-cli --tls` 行为仍然混合 — 新建连接走新证书，已建连接（包括 inter-pod 复制流）继续走旧证书。这是 silent failure 的现场：四个口径里只有 #1 / #2 通过，#3 / #4 没过。
3. **方案 C：patch derived secret + rolling restart 全部 pod**。每个 pod 重启后业务进程重新加载证书，`info replication` 显示 master/replica 间复制链路恢复。四个口径全部通过。

### 结论

- 双层模型下，正确轮换路径是 **patch derived → 等 mount sync → rolling restart**，不能省 restart。
- 验证不能只看 derived secret + pod 文件，必须显式确认业务进程层与内部连接层。
- 这个现象与具体引擎无关，正文方法论已抽离。

### 测试 artifacts

- 验证脚本：`tests/g7-tls-rotation.sh`。
- 验证环境：k3d `kb-local`，KubeBlocks `1.2`，addon rev 57。

## 相关主题

- [`docs/addon-reconfigure-guide.md`](addon-reconfigure-guide.md)
- [`docs/addon-switchover-guide.md`](addon-switchover-guide.md)
