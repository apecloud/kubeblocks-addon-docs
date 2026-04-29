# Addon TLS 开发与验证指南

本文面向 KubeBlocks Addon 开发者，聚焦 TLS 能力的设计、验证与排障。通用正文只描述 Addon TLS 的共性问题；引擎、协议、命令行工具和拓扑细节只放在案例附录中。

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

## 相关主题

- [`docs/addon-reconfigure-guide.md`](addon-reconfigure-guide.md)
- [`docs/addon-switchover-guide.md`](addon-switchover-guide.md)
