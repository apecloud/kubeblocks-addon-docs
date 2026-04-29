# Addon Reconfigure 开发与排障指南

本文面向 KubeBlocks Addon 开发者，例如 MariaDB、PostgreSQL、Redis、Valkey 等 Addon 的维护者。

目标是说明 Addon 在实现参数变更能力时，如何区分动态参数和静态参数，如何声明 `dynamicParameters` 与对应的 live-apply 执行路径，以及当 reconfigure 行为不符合预期时如何排查。

## 适用场景

当 Addon 支持通过 OpsRequest 或 `kbcli cluster configure` 修改配置参数时，开发者需要回答两个问题：

1. 这个参数是否可以在业务进程运行时动态生效？
2. 如果可以动态生效，Addon 是否已经正确告诉 KubeBlocks 如何执行 reload？

不要把“数据库本身支持运行时修改某个参数”等同于“Addon 已经实现动态热加载”。KubeBlocks 需要从 Addon 的配置声明中知道哪些参数可以动态修改，以及修改后该执行什么动作。

## 动态参数与静态参数

Addon 开发者通常需要把参数分成两类：

- `dynamicParameters`：修改后可以通过 reload、SQL、CLI、HTTP API 或其他机制在运行中的实例上生效，理论上不需要重启 Pod。
- `staticParameters`：修改后需要重启实例，或者不允许在线变更。

如果一个参数没有被正确放进 `dynamicParameters`，KubeBlocks 不会自动推断它可以动态生效。即使数据库本身支持动态修改，也可能退化成 rolling restart。

## 先区分：legacy `reloadAction` 与当前优先的 `reconfigure.exec`

这份文档里提到的“live-apply 执行路径”，历史上至少有两种常见实现：

- **legacy 路径**：`ParametersDefinition` / 旧配置契约里的 `reloadAction`、`shellTrigger`
- **当前优先路径**：`ComponentDefinition.spec.configs[].reconfigure.exec`

需要明确：

- **新开发或当前主路径，优先看 `reconfigure.exec`，不要再把 `reloadAction` 当成默认推荐答案**
- 如果你维护的是旧 Addon、旧资源，或者在排查历史问题线，仍然可能需要检查 live 资源里是否还在使用 legacy `reloadAction`

也就是说，本文后面凡是讲“reload 执行路径”，默认应理解为：

- 当前优先先看 `reconfigure.exec`
- legacy 场景再补查 `reloadAction`

而不是统一推荐继续新增 `reloadAction`

如果在当前控制器上验证旧 addon，遇到类似下面的状态：

- `OpsRequest=Failed`
- `ComponentParameter=FailedAndPause`
- errMessage 包含 `unsupported legacy reloadAction ... legacy-config-manager-required is not enabled`

这不是“动态参数 runtime 没生效”的同一类结论，而是更早一层的执行路径不兼容。它说明这轮验证根本没有进入真实 reload / kbagent action 结果面。此时不能把 runtime 未变化写成 false success，也不能把这个 addon 当成已验证通过的动态 reconfigure 参考；应先把执行路径迁到当前 `ComponentDefinition.spec.configs[].reconfigure.exec`，或明确这是旧兼容模式下的验证。

升级到新 `ComponentDefinition` 路径后，也要确认配置模板 provisioning 链路仍然完整。特别是 `externalManaged: true` 的 config，如果没有对应的外部配置渲染器 / 参数控制器把模板注入到 Component，可能在创建阶段直接卡住：

```text
config/script template has no template specified: <config-name>
```

这同样不是动态参数 runtime 结论，而是 setup 阶段的 addon 打包 / 模板绑定问题。遇到这类 blocker 时，应固定以下证据：

- 新 Helm release revision / chart version；
- 新 `ComponentDefinition.spec.configs[*].template` 是否存在；
- 对应 ConfigMap 是否存在；
- 对应 `ParamConfigRenderer` 或等价配置渲染绑定是否存在；
- Component / Cluster event 中的具体 `config/script template has no template specified` 行。

只有样本能先过干净 `Running`，再发最小 reconfigure OpsRequest，才算进入真实 R08a 结果面。

旧版本组合也可能在进入 reconfigure 前被初始化脚本挡住。例如 Redis 1.0.2 在 addon 升级 / 回滚后的复用环境里，如果 Helm release 已回到 1.0.2，但带 keep 策略的 `ComponentDefinition` / `ComponentVersion` 仍残留新版本语义，data pod 可能停在 init 阶段：

```text
init-dbctl: cp: can't stat '/config': No such file or directory
```

这仍然不是参数热加载结论，也不能直接写成“KB 1.0.2 + Redis addon 1.0.2 不可用”。它说明 data pod 没有完成启动，`CONFIG GET` / `CONFIG SET` 的 runtime 结果面不存在。处理口径与其他 setup blocker 一样：

- 先固定 controller image / imageID / Helm chart version / addon release revision；
- 固定 live `ComponentDefinition` / `ComponentVersion` 的 chart label、init image、volumeMount、volume 配置，而不是只看 Helm release version；
- 固定 pod describe、init container 日志、Cluster / Component / InstanceSet 状态和 events；
- 如果怀疑 keep 资源污染，应对照目标 release 的 manifest 重新确认 live 对象是否真的回到目标版本；
- 不要把“没有进入 OpsRequest”写成动态参数通过或失败。

对 Redis 1.0.2 这类旧路径还要单独看 reload 机制。清理版本污染并恢复 1.0.2 manifest 后，Redis 的 `maxmemory-policy` 可以通过 legacy config-manager 把原始参数名和值作为脚本 argv 传给 `reload-parameter.sh`，例如执行：

```text
/opt/kb-tools/reload/redis-replication-config/reload-parameter.sh maxmemory-policy allkeys-lru
```

这类 argv 路径不会遇到 `maxmemory-policy` 不是合法 shell 变量名的问题。因此 Redis 1.0.2 的正例不能反推所有 `reconfigure.exec` / kbagent action 都能消费 raw key；它只说明在该 legacy config-manager 参数传递路径下，`maxmemory-policy` 可以被正确传给脚本并用 `CONFIG SET` 生效。

拆开看，Redis 1.0.2 的传参链路是：

- `ParametersDefinition.reloadAction.shellTrigger` 声明 reload 脚本，例如 `reload-parameter.sh`，并引用脚本 ConfigMap；
- 参数不在 static / restart-only 列表里时，控制器按动态参数处理；
- 控制器把 changed params 作为 key/value 字典交给 legacy config-manager，而不是要求 addon shell 直接读取同名环境变量；
- config-manager 最后以 argv 方式执行 reload 脚本，例如 `reload-parameter.sh maxmemory-policy allkeys-lru`；
- 脚本用 `$1` 作为参数名、`${@:2}` 作为参数值，再调用数据库命令，例如 `redis-cli CONFIG SET "$paramName" "$paramValue"`。

因此，Redis 1.0.2 的 `maxmemory-policy` 能生效，关键不是 “shell 能读取名为 `maxmemory-policy` 的环境变量”，而是这条 legacy config-manager 路径绕开了 shell 变量名限制，把参数名和值当作普通命令参数传入脚本。

Redis 1.1 的路径不同，不应沿用 Redis 1.0.2 的判断。Redis 1.1 的 chart 使用 `ComponentDefinition.spec.configs[].reconfigure.exec`，典型命令是：

```sh
env | cut -d= -f1 | grep -E '^[a-z0-9_.-][a-z0-9_.-]*$' | sort -u | while IFS= read -r param; do
  [ -n "${param}" ] || continue
  /scripts/reload-parameter.sh "${param}" "$(printenv "${param}")"
done
```

这表示 Redis 1.1 的脚本入口期望从 action 进程环境里枚举参数名，再把参数名和值作为 argv 传给 `reload-parameter.sh`。但在 KB 1.1 的 `reconfigure.exec` 路径里，InstanceSet 写入 action 的 `Parameters` 主要来自文件变化元数据：

```text
KB_CONFIG_FILES_CREATED
KB_CONFIG_FILES_REMOVED
KB_CONFIG_FILES_UPDATED
```

这些值会被 kbagent 合并为 action 进程环境变量；它们不是业务参数名本身。因此，除非控制器或 agent 另有明确参数注入契约，否则不能假设 `maxmemory-policy=allkeys-lru` 会天然出现在 Redis 1.1 reconfigure 脚本的 `env` 输出里。

检查 Redis 1.1 这类 `reconfigure.exec + env loop` 时，要把链路拆成两段：

- action request / InstanceSet config 里到底传了哪些 `Parameters`；
- kbagent 是否把这些 `Parameters` 原样注入为 action 进程环境；
- addon 脚本枚举环境时能否看到目标业务参数，例如 `maxmemory-policy`；
- `reload-parameter.sh` 是否真的执行了 `CONFIG SET maxmemory-policy allkeys-lru`。

如果样本还没过 setup 阶段，例如 Redis 1.1 出现 `config/script template has no template specified: redis-replication-config`，不能把它写成 `maxmemory-policy` 动态修改通过或失败；只能写成配置模板绑定 blocker。此时可以先用源码 / live `ComponentDefinition` 固定“设计上的传参路径”，但 runtime 结论必须等样本进入 `Running` 并发出最小 Reconfiguring OpsRequest 后再下。

## `dynamicParameters` 与 live-apply 执行路径缺一不可

实现动态热加载时，通常不能只声明 `dynamicParameters`。还需要有一条真正的 live-apply 执行路径，告诉 KubeBlocks 参数变化后如何把值应用到运行中的实例。

检查项：

- 参数是否出现在 `dynamicParameters` 中。
- 当前实现是否使用 `reconfigure.exec`。
- 如果是历史路径，legacy `reloadAction` 是否仍存在。
- 这条执行路径是否写在正确层级。
- 这条执行路径在最终渲染出的资源中是否真实存在。
- 触发 reconfigure 后，reload 命令是否真的执行。

常见错误是只补了 `dynamicParameters`，但没有补 live-apply 执行路径。这样 Addon 看起来已经声明了动态参数，但实际不会完成动态热加载。

## 不要只看源码 YAML，要看最终生效对象

Helm 模板、helper、include 和缩进都可能导致字段写错层级。

如果 `reconfigure.exec` 或 legacy `reloadAction` 写在错误位置，可能出现以下情况：

- 源码里能看到这个字段。
- 渲染后字段不在 KubeBlocks 期望的位置。
- 控制器实际忽略该字段。
- reconfigure 没有按预期触发 reload。

建议排查时同时检查：

- 源码模板。
- Helm 渲染结果。
- 集群中实际创建出的 CR。
- controller / config-manager / sidecar 的运行日志。

## OpsRequest 成功但实际走 restart path 的排查

动态 reconfigure 不能只看 `OpsRequest=Succeeded` 和最终 runtime 值正确。某些缺陷会表现为：

- `OpsRequest` 成功；
- runtime / 配置文件最终也变成目标值；
- 但 `ComponentParameter.status.configurationStatus[].reconcileDetail.policy=restart`；
- Cluster / Component 上出现 `config.kubeblocks.io/restart-<config-name>` annotation；
- data pod UID 全部变化，events 中有 `Killing`。

这种场景不是 live reload 成功，而是“重启后配置自然生效”。测试应把它判为 no-restart 语义失败。

排查顺序建议固定为：

- 确认 live `ParametersDefinition` 中目标参数是否在 `dynamicParameters`，不要只看源码。
- 确认 live `ComponentDefinition.spec.configs[].reconfigure` 是否存在，并且 `targetPodSelector` / container / command 符合预期。
- 确认派生的 `ComponentParameter.spec.configItemDetails[].configSpec` 是否仍保留 `reconfigure` 字段。
- 如果 `ComponentDefinition` 有 `reconfigure`，但 `ComponentParameter.configSpec` 丢失该字段，控制器可能会把该配置当成不支持 template reconfigure，最终退回 restart。
- 同时固定 `ComponentParameter.status.reconcileDetail.policy`、Cluster restart annotation、pod UID 前后对比、events `Killing` 顺序、controller since-op log。

结论写法要区分：

- `dynamicParameters` 缺失：Addon 参数元数据问题。
- live `ComponentDefinition.reconfigure` 缺失：Addon live-apply 执行路径问题。
- `ComponentDefinition.reconfigure` 存在但 `ComponentParameter.configSpec.reconfigure` 缺失：KubeBlocks 参数控制器派生/读取 configSpec 的链路问题。
- `policy=syncReload` 但 runtime 未变：进入 reload action / 脚本 / 参数传递问题。

## `syncReload` 后仍要排查隐性拓扑副作用

`policy=syncReload`、pod UID 不变、没有 `Killing`，只能说明没有走 restart path；它不等于完全无副作用。对有主从/主备拓扑的引擎，还要确认 reconfigure 窗口没有触发不必要的 switchover 或临时多主。

建议把这组证据作为动态参数 no-restart 验收的一部分：

- controller since-op log 中是否出现 `succeed to call switchover action`。
- 每个 data pod 的 kbagent since-op log 中是否出现 `Action Executed {"action":"switchover"}`。
- reconfigure action 与 switchover action 的时间线，最好按 pod + reconcileID 排列。
- post snapshot 和 settle snapshot 的 `INFO replication` 原文，而不是只留主从汇总。
- post snapshot 是否有 transient multi-master / multi-primary。
- settle 后 primary 是否无预期漂移。

如果看到同一个 reconcileID 下先 reconfigure 再 switchover，常见根因不是 Addon reload 脚本本身，而是工作负载控制器在 reconfigure 后还需要更新 pod metadata，例如把 `config.kubeblocks.io/config-hash` 写回 pod annotation 作为“配置已消费”标记。若通用 in-place pod update 逻辑把这种 annotation-only 标记更新当成普通 pod update，并在 update 前调用 switchover，就会制造不必要的拓扑漂移。

这类修复的原则是：只对“唯一差异是 config-hash annotation”的 marker update 跳过 switchover；其他 label、annotation、container、resource 或 spec 变化仍应保持原有 in-place update / switchover 保护语义。

### 案例附录：Valkey R08a restart-path

Valkey R08a `maxmemory-policy` 曾出现稳定 `2/2` 复现：`OpsRequest Succeed`，runtime/conf 都变为 `3/3 allkeys-lru`，但 `ComponentParameter policy=restart`，Cluster 有 `config.kubeblocks.io/restart-valkey-replication-config=<hash>`，3 个 data pod UID 全变。

现场进一步确认：live `valkey9-pd` 已包含 `dynamicParameters: maxmemory-policy`，live `ComponentDefinition valkey-9-0.1.1` 也有 `spec.configs[].reconfigure.exec.targetPodSelector=All`；但 `ComponentParameter.spec.configItemDetails[].configSpec` 里只剩 `externalManaged/name/namespace/template/volumeName`，没有 `reconfigure`。因此根因不在 Valkey reload 脚本，也不在 `maxmemory-policy` 动态参数声明，而在控制器实际用于决策的 configSpec 副本缺失 reconfigure 能力，导致动态参数被按 restart path 执行。

### 案例附录：Valkey R08a syncReload 后 switchover/topology drift

Valkey R08a 在修掉 restart path 后，又出现过稳定 `2/2` 的 residual signal：`policy=syncReload`，没有 restart annotation，3 个 data pod UID 不变，也没有 data-pod `Killing`，但 controller 和 kbagent logs 中每个 data pod 都出现 `switchover` action。post snapshot 一度出现两个 master，settle 后 primary 发生漂移。

证据关键点是 controller timeline 显示同一个 InstanceSet reconcileID 内对同一个 pod 先执行 `reconfigure`，随后执行 `switchover`。根因是 syncReload 成功后，InstanceSet 还要把新的 config hash 写入 pod annotation；这个 config-hash marker update 被通用 in-place update 路径处理，触发了 switchover 保护逻辑。

对应修复是让 InstanceSet / Instance update reconciler 识别 config-hash annotation-only update：如果唯一差异是 `config.kubeblocks.io/config-hash`，只更新 pod metadata，不调用 switchover。后续 fresh sample 证明：`policy=syncReload`、UID 不变、无 Killing、controller/kbagent switchover grep 均为 0，post/settle 都保持单 primary。

## 避免 YAML 重复 key

不要在同一个对象层级中出现两个相同 key，例如两个 `reloadAction`，或把 `reconfigure` / `exec` 重复拼出多份有效定义。

重复 key 的风险是：

- 不同 YAML parser 行为可能不同。
- 有的工具保留前一个，有的保留后一个。
- 表面可能不报错，但最终行为不可预测。

关键字段应保持唯一。修改后建议检查最终 YAML 中只保留一份有效定义。

## 确认 live-apply 执行路径的执行上下文

不要假设 reload 命令一定运行在数据库主容器中。

在某些实现中：

- legacy `shellTrigger` 可能运行在 sidecar，例如 config-manager
- 当前 `reconfigure.exec` 也可能不在你想象的业务主容器路径中完成参数传递

执行上下文不同会影响：

- 脚本路径是否存在。
- 数据库 CLI 是否存在。
- 环境变量是否存在。
- 当前容器能否访问业务端口。
- 二进制是否能在该镜像中运行。

排障时应优先确认：

- 命令在哪个容器中执行。
- 当前容器内有哪些文件和工具。
- 当前容器能否访问目标服务地址。
- 当前容器中的用户、PATH、shell、依赖库是否满足执行条件。

## 确认参数传递契约

不要假设参数一定通过 stdin 传入。

不同版本或不同执行模式下，参数可能通过以下方式传给 reload 命令：

- stdin
- argv
- env
- 临时文件
- 配置挂载

排障时建议先做最小探针，明确打印：

- `stdin` 内容。
- `argv` 内容。
- 相关环境变量。
- 当前工作目录。
- reload 命令实际收到的 key/value。

如果脚本从错误的输入源读取参数，可能出现“命令执行成功但没有任何参数被处理”的假成功。

## 区分 request 参数名与 shell 环境变量名

`ActionRequest` 里的参数名不一定天然就是合法的 shell 环境变量名。

例如参数系统里常见的 key 可能包含 `-`、`.` 或数字开头等形式。控制器或 agent 日志里能看到 request 已经带了目标 key/value，不代表脚本里可以直接用同名变量读取到它。

排查这类问题时，不要只验证：

- request 里有没有目标 key/value；
- 渲染后的配置文件里有没有目标值。

还要明确验证脚本实际可见的输入源：

- raw request key/value；
- shell-safe alias，例如把 `foo-bar` 暴露成 `FOO_BAR`；
- rendered config value；
- runtime query value。

不要把“shell 理论上可以 `printenv 'foo-bar'` 读取带横线 key”直接当成产品契约。它只有在实际 agent 会把 raw key 原样注入 action 进程环境时才成立。验证时必须用真实 action 结果证明 closure check 读到了目标值；如果高层成功、hash/status 已推进，但 runtime 仍是旧值且没有 mismatch 日志，说明 raw key 没进入脚本可消费路径，仍然需要 shell-safe alias 或等价的显式参数契约。

短期产品化时，优先采用 controller-side alias：在构造 action request 参数时保留原始 key，同时补一个 shell-safe alias。例如 `maxmemory-policy=allkeys-lru` 应同时带出 `MAXMEMORY_POLICY=allkeys-lru`。这个方案的边界清楚：不依赖 data pod 里的旧 kbagent 重新发版，也不要求 addon 脚本直接消费非法 shell 变量名；旧 agent 只需要按现有机制把 request 参数注入到 action 环境即可。

产品验证时要同时使用“alias-producing controller”和“alias-consuming addon”。如果 controller 已补 alias，但 live addon 仍是 raw-only `printenv 'maxmemory-policy'` 形态，验证结果不能代表最终产品组合。正确组合应至少证明：

- live controller 是补 raw key + shell-safe alias 的 gate；
- live addon 的 reconfigure action 消费 `maxmemory-policy|MAXMEMORY_POLICY`；
- `targetPodSelector: All` 或等价 fan-out 仍存在；
- runtime closure check 在高层成功前能约束真实 runtime；
- 最终高层成功时，业务 runtime 已达到目标值。

如果验证中直接采到 runtime split 时高层仍保持 `Running`，最终等 runtime 达到目标后才 `Succeed/Finished`，可以作为 false-success 被阻断的正向行为证据。若没有独立 ActionRequest dump，则应在结论里标明 alias 是由 controller gate 身份、raw-only negative 对照和 alias-consuming addon 行为间接证明；后续正式回归可再补逐字 request/action dump，但不应把缺少 dump 误写成 runtime 失败。

实现 alias 时应遵守两条兼容规则：

- 原始 key 必须保留，避免破坏依赖 raw key 的实现。
- 如果 shell-safe alias 已显式存在且非空，不要覆盖；如果 alias 缺失或为空，才从 raw key 补齐。

验证 action 参数契约时，还要确认你更新的是实际执行脚本的那个组件。比如只切换 controller 镜像，不一定会更新数据 Pod 里的 sidecar / agent 镜像；这种情况下，controller 日志只能证明 request 生成逻辑变了，不能证明 Pod 内 action 的 env 注入逻辑也变了。结论应拆开写：

- controller 发出的 request 参数是什么；
- data pod 实际使用的 agent 镜像 / imageID 是什么；
- action 进程实际可见的环境变量是什么。

如果这三层没有同时对上，这一轮只能算 validation-entry mismatch，不能写成产品代码正/负结论。

如果脚本期望读取 `UPPER_UNDERSCORE` 形式的变量，而实际只传入了带横线的原始 key，就可能出现这种假象：

- controller request 看起来已经正确；
- 脚本里对应变量为空；
- 脚本回退到 rendered config；
- rendered config 又可能滞后；
- 最终出现“hash / status 推进了，但 runtime 没真正闭环”。

更稳的实现方式是：

- 明确约定 action 参数到 shell 变量的转换规则；
- 同时保留原始 key 和 shell-safe alias，避免破坏已有调用方；
- 如果 raw key 与 alias 同时存在，以显式传入的非空 alias 为准；如果 alias 为空，应由 raw key 补齐，避免空 alias 把真实请求参数遮蔽掉；
- 不要把 rendered config 当成产品修复里的主目标来源；它可以用于排障对照，但如果 action request 没有显式传入目标参数，应暴露参数传递契约缺失，而不是从 rendered config 猜一个目标值继续执行；
- 在 validation 阶段打印 `source / desired / rendered / runtime`，不要只打印最终成功或失败。

## 警惕“假成功”

“命令没有报错”不等于“参数已经生效”。

典型假成功场景：

- 脚本从 stdin 读取参数，但 stdin 为空。
- 循环体没有执行。
- 脚本 exit 0。
- OpsRequest 看起来成功。
- 实际配置值没有改变，或者热加载没有发生。

避免方式：

- 打印实际输入参数。
- 打印实际执行的 reload 命令。
- 检查命令返回值。
- 查询数据库当前配置值。
- 检查 Pod UID 和 restart count。
- 检查 OpsRequest 和 Cluster 状态。

## 注意工具链和 ABI 兼容性

有些 Addon 会尝试把数据库主容器中的 CLI 复制到 sidecar 中执行。这个方案需要谨慎。

即使文件存在并且有执行权限，也不代表它一定能运行。常见问题包括：

- 动态链接器不存在。
- glibc / musl 不兼容。
- 依赖库缺失。
- 架构或基础镜像不一致。

因此不要只验证：

```bash
ls -l /path/to/tool
```

还应该验证：

```bash
/path/to/tool --version
```

或者执行一次真实的最小操作，确认该工具在当前容器中真正可用。

## 推荐排障流程

当动态参数热加载失败时，建议按以下顺序排查。

### 1. 确认参数分类

- 参数是否应该动态生效。
- 参数是否在 `dynamicParameters` 中。
- 参数是否误放入 `staticParameters`。
- 未分类参数的预期行为是什么。

### 2. 确认 live-apply 执行路径声明

- 是否存在 `reloadAction`。
- 或者，当前路径是否存在 `reconfigure.exec`。
- 是否写在正确层级。
- 最终渲染结果是否包含该字段。
- 集群中的实际 CR 是否包含该字段。

### 3. 确认执行上下文

- reload 命令在哪个容器中执行。
- 该容器是否有需要的脚本和工具。
- 该容器是否能访问数据库实例。
- 该容器中的二进制是否能真实运行。

### 4. 确认参数传递方式

- 参数来自 stdin、argv、env 还是文件。
- key/value 格式是什么。
- 是否需要参数名转换，例如 `UPPER_UNDERSCORE` 转成 `lower-hyphen`。

### 5. 确认真实效果

- reload 命令返回值。
- 数据库配置查询结果。
- Pod UID 是否变化。
- restart count 是否变化。
- OpsRequest 是否 Succeed。
- Cluster 是否 Running。

## 建议固定收集的验证项

每次验证 reconfigure 行为时，建议固定回报以下内容：

1. 参数变更前后的值。
2. reload 命令原文或关键路径。
3. reload 命令返回值。
4. 配置查询结果。
5. Pod UID 变化情况。
6. restart count 变化情况。
7. OpsRequest 最终状态。
8. Cluster 最终状态。
9. 相关日志或附件路径。

## 动态参数验证口径

动态参数的预期结果通常是：

- 参数值生效。
- Pod UID 不变。
- restart count 不变。
- OpsRequest Succeed。
- Cluster Running。

如果参数生效但 Pod 重启了，说明 no-restart 语义没有达成，应继续检查 `dynamicParameters`、live-apply 执行路径和执行链路。

## `OpsRequest Succeed` 不等于集群已经进入稳定态

对 reconfigure 链路，不能把：

- `OpsRequest.phase=Succeed`

直接等同于：

- `Cluster.phase=Running`
- role / endpoint / service 已完成最终收敛
- 下一次 Ops 已经可以安全发起

真实执行里经常出现：

- reconfigure 对象已经 `Succeed`；
- 但 `Cluster.phase` 仍短暂处于 `Updating`、`Abnormal` 或其他过渡态；
- 这时如果立刻串下一个 `restart` / `switchover` / `scale`，可能被 Validate 阶段拒绝，或者把控制面过渡态误写成产品缺陷。

因此，凡是：

- `reconfigure -> restart`
- `reconfigure -> switchover`
- `reconfigure -> 下一次 ops`

这类链式操作，都建议拆成两段等待：

1. 先等本次 `OpsRequest` 自身完成；
2. 再显式等 `Cluster` 回到稳定态，例如 `Running`，必要时再补 role / endpoint / runtime truth 的最终确认。

不要只写一个 `wait_ops()` 就直接发下一个 Ops。

## 控制面对象正确，不代表 runtime 已经切到新值

这轮经验反复证明，下面这些对象即使已经全对：

- `ComponentParameter`
- 参数相关 `ConfigMap`
- 控制器状态里的 reload / sync 成功计数

也仍然不能证明：

- 业务进程 runtime 已经用了新参数。

典型现象是：

- 控制面显示参数已下发；
- `Cluster` 最终也回到 `Running`；
- 但进程内查询到的 live value 仍是旧值。

因此，动态参数最终验收必须把 **runtime truth** 放进检查项，而不是只看控制面对象。

建议固定增加一层验证：

- 对数据库 / 中间件直接查询 live runtime 值；
- 明确比较“预期值 vs 实际值”；
- 必要时同时记录 reload 脚本收到的 key/value 与业务进程最终读到的值。

如果控制面对象正确、集群也稳定，但 runtime 仍是旧值，那么更应优先怀疑：

- live-apply 路径没有真正执行到业务进程；
- 参数注入 / 传值链路有问题；
- reload 脚本 exit 0 但没有实际应用参数。

而不是先把问题写成“控制面没有下发”或“单纯等待不够”。

## `instance status` / live hash 收口，不等于 runtime proof

这轮新补充的经验是：即使下面这些控制面信号都已经闭环：

- `ComponentParameter` 进入 `Finished`
- `OpsRequest` 进入 `Succeeded`
- live pod annotation / config hash 看起来已经 `N/N` 对齐目标值
- 控制器理由里出现类似 `config hash converged via instance status only`

也仍然不能直接推出：

- 全部 pod 的业务进程 runtime 已经真的切到目标值。

真实现场里可能出现这种更危险的假收口：

- `instance status` 和 live hash 先闭到 `N/N target`
- 但按 pod 查询 runtime，仍然是 `N-1 / N`、甚至 `1/N` 生效
- 高层对象因此提前给出 `Finished/Succeeded`
- 结果把真正的 per-pod runtime divergence 藏到控制面成功后面。

排查这类问题时，不要只问“hash 为什么收敛了”，而要固定把三组 truth 并排展开：

1. `instance status` 里的 updated / stale 视图
2. live pod annotation / config-hash 视图
3. **按 pod 展开** 的 runtime truth

建议每次都明确列出类似结果：

- `pod-0 hash = target, runtime = new`
- `pod-1 hash = target, runtime = old`
- `pod-2 hash = target, runtime = new`

如果出现“hash 先全收敛、runtime 仍分裂”，更合理的 first blocker 不是：

- addon 参数没渲染
- reload 脚本完全没执行
- 单纯等待不够

而是：

- **控制面把 `instance status` / hash convergence 错当成 runtime proof 提前收口**

这时更稳的处理方式是：

- 先把这条 success 分支显式 surface 出来
- 不要让 `Finished/Succeeded` 把 runtime divergence 盖掉
- 再继续沿“为什么 runtime 没闭环”往更深的 per-pod apply truth 收。

还有一个容易踩的坑：

- 已经补了 source / desired / rendered / runtime 这类 source-surfacing 埋点
- 但现场没有打出新的 local source 证据
- 同时 hash 还是能先闭到 `N/N target`
- runtime 却仍停在 `N-1/N`、`1/N` 这种分裂形态

这时不要继续把前线停在“是不是 source 没打出来”这一层反复怀疑。更合理的判断通常是：

- **当前 first blocker 已经不是 source-surfacing，而是“控制面允许 hash/instance status 先于 runtime proof 收口”**

也就是说，下一跳最小 cut 往往应该先去把这条伪收口显式 surface 出来，再决定更深的 per-pod continuity / apply truth 在哪一层断掉，而不是继续在 source 打点层原地打转。

## 用 runtime proof 推进 hash 前，必须校验 proof 与目标参数一致

当控制器为了避免“runtime 已推进但 hash 仍旧”而尝试把 live pod hash 往前推进时，不能只看 action 返回里出现了“某个 pod 已经是期望 runtime”。这里的“期望”可能来自执行 pod 的本地旧视图，而不一定是本轮 reconfigure 的目标参数。

更安全的判断顺序是：

1. 先取本轮 reconfigure 的目标参数值；
2. 再解析 runtime proof 里的 expected / desired 值；
3. 只有两者一致时，才允许用这个 proof 推进 live hash 或 `InstanceStatus`；
4. 如果 proof 的 expected 值仍是旧值，只能把它当成诊断线索，不能当成目标已应用证据。

这个区别很关键。否则会出现一种假推进：

- 本轮目标参数是新值；
- 某个 pod 的本地 action 仍按旧值做判断；
- action 报出“远端 pod 与我的 expected 不一致”；
- 控制器误把这条旧 expected proof 当成目标 runtime proof；
- live hash / `InstanceStatus` 被推进到目标 hash；
- 但实际 runtime 仍然没有全量闭环。

这类问题的排查口径应该固定成三向对照：

- 目标参数：本轮真正要应用的值；
- runtime proof：action / postcheck 里声称的 expected / desired 值；
- runtime truth：逐 pod 直接查询到的实际值。

只有这三者能对齐，hash 推进才有资格被当作 runtime closure 证据。否则高层成功很可能只是“hash closure”，不是“runtime closure”。

还要注意：**proof 的 expected 值等于目标参数，也不等于整个集群 runtime 已经闭环。**

它最多只能说明：

- 这条 proof 没有使用旧目标；
- proof 里点名的某些 pod 在那个时刻看起来已经到目标 runtime。

它不能推出：

- 未被 proof 点名的 pod 也已经到目标 runtime；
- live hash / `InstanceStatus` 可以直接全量闭环；
- 高层对象可以进入 `Finished` / `Succeeded`。

如果 proof 同时还点名了另一个 pod 仍是旧 runtime，那么这条 evidence 的语义应该是：

- “部分 pod 已经推进，另一个 pod 仍 stale”

而不是：

- “可以把 hash / status 全部推到 target 并收口成功”。

因此，target-param guard 只能防止“旧 expected 被误当成新目标”，不能替代最终的全量 runtime proof。最终收口仍然必须看到：

- 每个 pod 的 runtime truth 都已经到目标值；
- 或者高层状态保持 retry / explicit failure，而不是在 runtime split 时返回成功。

## hash / InstanceStatus 全量闭环，仍不能替代 runtime proof

有些控制器路径会先看到：

- live pod hash 已经是 `N/N target`；
- `InstanceSet.status.instanceStatus[*].configs[*].configHash` 也已经是 `N/N target`；
- 但真实 runtime 仍有 pod 停在旧值。

这类状态只能说明 **控制面 hash 已闭环**，不能说明 **数据库进程 runtime 已闭环**。如果此时直接返回 `Finished` / `Succeeded`，就是 false success。

更稳的处理方式是：

- 当开启 cluster-wide runtime continuity 时，`hash + InstanceStatus` 全量闭环但 runtime proof 缺失，应保持 `Retry` 或明确失败；
- reason 需要直接写出 `config hash converged via instance status only before runtime proof` 这类信息；
- 只有收齐所有 pod 的 runtime proof，或能证明 runtime 已经 `N/N target`，高层才允许成功收口。

这条规则和上一条不同：

- target-param guard 只保证“用于推进 hash 的 proof 没用错目标值”；
- runtime-proof guard 保证“不能只靠 hash/status 成功，必须有真实 runtime 成功”。

## runtime-proof guard 必须有退出路径

runtime-proof guard 的目标是防止 false success：当 live hash 和 `InstanceStatus` 已经先闭到 `N/N target`，但 runtime proof 还没有证明所有实例真实生效时，高层状态不应该直接进入 `Finished` / `Succeeded`。

但这个 guard 不能只会“拦截”，还必须会“放行”。否则会出现另一类 false negative：

- 中间态时，guard 正确识别出 `hash / InstanceStatus` 已闭环但 runtime proof 缺失；
- 后续同一轮里，所有 pod 的 runtime truth 已经变成 `N/N target`；
- live hash 和 `InstanceStatus` 也仍是 `N/N target`；
- 但高层状态继续停在 `Running / Retry`，reason 仍是“before runtime proof”。

这说明 guard 只记住了“曾经缺 runtime proof”，却没有重新验证“runtime proof 已经补齐”。这种状态下，系统已经避免了 false success，但又制造了 stuck retry。

设计这类 guard 时要同时定义两条路径：

- **进入 guard 的条件**：hash / `InstanceStatus` 已闭环，但 runtime proof 还没有证明 `N/N target`。
- **退出 guard 的条件**：后续 reconcile 或 bounded wait 中，能够重新看到所有 pod runtime truth 已经是 `N/N target`，并据此允许高层收口。

排查时建议固定一段 bounded wait，不要只停在第一次 split 或第一次 guard 命中：

- 如果 bounded wait 结束时 runtime 仍 split，问题仍在 per-pod apply / runtime closure。
- 如果 bounded wait 结束时 runtime 已 `N/N target`，但高层仍 `Running / Retry`，问题就收窄到 guard 的 exit / closure 逻辑。

也就是说：

- **runtime-proof guard 负责挡住 hash/status false success；**
- **runtime-proof closure 负责在真实 runtime 收敛后解除等待。**

二者缺一不可。

## validation-only 探针堆叠后要回到产品候选形态重测

排查动态 reconfigure 问题时，经常会连续叠加多层 validation-only cut：

- controller 日志探针；
- action request 参数探针；
- runtime postcheck / source-surfacing；
- 临时 hash 推进逻辑；
- 临时 runtime-proof guard。

这些 cut 的价值是切边界，不是最终产品语义。叠得越多，越容易出现新的“调试环境特有行为”：比如某个 validation-only hash advancement 把状态提前推到 target，另一个 guard 又基于这个状态卡住。此时继续在同一个 dirty gate 上加探针，得到的结论会越来越难解释。

当已经排掉 request / addon / kbagent / runtime 永久不生效这些外层问题后，下一步应先做一次 **product-candidate reset**：

- controller 回到干净或最接近最终提交的 gate；
- addon 只保留计划进入最终提交的 reconfigure 语义；
- 删除 source-surfacing、request-param logging、cluster postcheck、临时 guard 等验证探针；
- 再用最小 R08a-only 入口重测。

这轮 reset 只回答一个问题：**最终产品形态是否能自然闭环**。

- 如果能闭环，就不要为了调试 gate 里的 stuck retry 设计更重的 runtime-proof marker。
- 如果仍然 false success 或 stuck retry，再进入 structured runtime-proof marker 设计。

简单说：validation-only cut 用来照亮问题，不能长期当成产品候选环境。进入最终修复判断前，必须先把手电筒和脚手架拆掉，确认房子本身是否还站不稳。

## runtime closure proof 应尽量放在懂业务 runtime 的执行层

控制器通常只能看到控制面状态，例如 live pod hash、`InstanceStatus`、OpsRequest / ComponentParameter condition。它不一定知道数据库进程里的真实 runtime 值。

如果某个动态参数必须用数据库 CLI、SQL 或 HTTP API 才能确认是否真正生效，那么最终闭环点通常不应该只放在高层 hash/status 判断里，而应该放在执行 reload 的业务层 action 里。

更稳的模式是：

- action 先执行本地 reload；
- action 再查询本地或集群内所有相关实例的 runtime truth；
- 只有 runtime truth 已经达到目标值，action 才返回成功；
- 如果任一实例仍是旧值，action 返回明确错误，让上层保持 retry / explicit failure；
- 高层 hash / status 只能作为进度信号，不能单独作为 runtime closure proof。

这样做的好处是：控制器不会在“不懂业务 runtime”的情况下把 `hash=target` 误判为“参数已经生效”。对多副本系统尤其重要，因为某个 pod 的配置文件和 hash 已经更新，并不代表所有业务进程都已经把动态参数 apply 到内存里。

验证这个模式时，判断口径要同时覆盖中间态和终态：

- 中间态如果 runtime 仍 split，action 应该返回明确 mismatch，让高层保持 `Running` / `Retry`，不能提前 `Succeeded` / `Finished`；
- 终态如果 runtime 已经 `N/N target`，live hash 和 `InstanceStatus` 也到 `N/N target`，高层才允许成功闭环；
- 如果 runtime 仍 split 但高层成功，说明 runtime closure proof 没有真正接入产品路径；
- 如果 runtime 已经 `N/N target` 但高层长期不闭环，说明 guard 有入口但缺退出路径。

这类检查最好是产品语义的一部分，而不是只留在 validation-only probe 里。probe 可以帮助定位边界，但最终提交应保留的是能阻止 false success 的 runtime closure 行为。

## 泛化 runtime closure proof 时，用白名单而不是一刀切

如果一个参数像 `maxmemory-policy` 一样满足以下条件，可以考虑纳入 runtime closure proof：

- 参数支持在线 `CONFIG SET` / SQL / API reload；
- runtime 有确定的读取方式，例如 `CONFIG GET <param>`；
- 读取值可以和目标值稳定比较，必要时有明确 normalize 规则；
- 同一个 Component 内所有相关实例应该收敛到同一个目标值；
- 不依赖异步副作用作为成功标准。

不要因为一个参数在 `dynamicParameters` 里，就默认把它加入全量 runtime proof。动态参数只说明“理论上可在线改”，不等于“可以用同一种 getter / comparator 证明闭环”。

更稳的泛化方式是维护一个显式表：

```text
runtime verified parameters:
- name: maxmemory-policy
  getter: CONFIG GET maxmemory-policy
  compare: exact-string
  scope: all-pods
- name: <next-param>
  getter: <engine-specific runtime query>
  compare: <normalize + compare rule>
  scope: <local-pod | all-pods | role-specific>
```

实现上可以先在 addon action 中用小的 shell `case` 表达这个白名单；当参数数量变多时，再把它提成 Values / helper / 专用脚本。关键是每新增一个参数，都必须带对应的验证案例，而不是把所有动态参数自动套进同一条 `CONFIG GET` 逻辑。

runtime proof 的目标值也应该来自明确的 reconfigure request / action 参数契约。不要为了让检查“能跑起来”而从 rendered config 回退推断目标值；这种 fallback 容易掩盖参数没有被传进 action 的真实问题，并且 rendered config 可能与当前 runtime 推进节奏不同步。

判断一个参数是否适合纳入白名单时，要检查这些风险：

- 返回值是否会被数据库归一化，例如单位、大小写、空字符串、列表顺序；
- 参数名和 runtime getter 名是否完全一致；
- 主从 / 角色之间是否允许不同值；
- 参数修改是否需要额外触发动作才能真正生效；
- getter 失败时应该是 retry、忽略，还是 terminal failure。

这样做的目标是：**让 runtime closure proof 可复用，但每个参数的 runtime 语义仍然是显式、可测试、可审查的。**

## 过渡期 hash / runtime split 不要过早冻结成 terminal failure

动态 reconfigure 在多副本场景下常见一个短暂过渡态：

- 部分 pod 已经应用到目标 runtime；
- 部分 pod 还停在旧 runtime；
- live pod hash / `InstanceStatus` 也可能一部分新、一部分旧；
- cluster-wide postcheck 会在这个窗口里看到 `updated` / `stale` split。

这个 split 是有价值的证据，但不一定等于最终失败。尤其当后续同一轮里能观察到：

- controller 发出的 action 参数已经是目标值；
- 各 pod 的 source / rendered / runtime 最终都能到目标值；
- live pod hash 和 `InstanceStatus` 最终也能到 `N/N target`。

那么前面的 split 更像是 **reconfigure 推进中的中间态**，而不是可以立即把 `OpsRequest` / `ComponentParameter` 冻结成 terminal `Failed` 的事实。

更稳的处理方式是：

- 中间态 split 要显式上报，保留 `updated` / `stale` 明细；
- 但默认先让它保持可重试状态，例如 `Retry / reconfiguring`；
- 只有在后续 reconcile 仍无法收敛，或 runtime proof 明确失败时，再进入 terminal failure；
- 如果最终 runtime / live hash / `InstanceStatus` 已经 `N/N` 收敛，高层状态必须重新闭环，而不是保留旧的中间态失败理由。

这条规则的核心是：

- **split 是排障线索，不天然是终局。**
- **terminal failure 应该基于最终无法收敛的证据，而不是基于一次过渡期快照。**

## clean rerun 仍复现时，先把“脏环境”正式排掉

动态参数问题还经常会卡在一个反复空转的怀疑上：

- 是不是因为环境脏了
- 是不是上一轮残留把结果污染了
- 要不要继续先清环境再说

如果你已经做过 **同 gate / 同版本 / 可审计 baseline** 的 clean rerun，而且新的样本仍然稳定复现：

- 外层 cluster-progress cut 还是同样的负结果
- runtime 仍是同样的 `N-1 / N`、`1/N` 分裂

那就应该把“脏环境”正式降级成**已排掉的外层疑点**，不要继续拿它当当前前线。

这时更合理的推进方式是：

1. 固定 clean rerun 已复现的最小 truth
2. 明确上一层 cut 已经是有效负结果
3. 把前线切到更深一层的 per-pod continuity / apply truth
4. 补 `source / desired / rendered / runtime` 这类埋点时，也把它当成**取证动作**，不是“外层已经修好”的证明

一句话说：

- **clean rerun 仍复现时，先关掉“脏环境”这条外层假设，再把前线压到更深的 per-pod 证据层。**

## 链式动态参数问题，优先区分“对象下发成功”与“live-apply 真生效”

对动态参数失败问题，建议先判断它属于哪一层：

### A. 对象下发层失败

- 参数对象没有更新；
- 模板没有渲染进最终 CR；
- `ComponentParameter` / `ConfigMap` 仍是旧值。

### B. live-apply 执行层失败

- 对象已经更新；
- 控制器记 reload / sync 成功；
- 但 runtime 仍是旧值。

一旦证据更像 B 类，就应该把排查重点切到：

- `reconfigure.exec` / legacy `reloadAction` 实际收到什么参数；
- key 名、env 名、argv、stdin、临时文件哪一层丢了值；
- 脚本是否只是“返回成功”，但没有真正把参数 apply 到 runtime。

不要在 B 类问题上继续盲目补“更长等待时间”，否则很容易把真正的 live-apply 缺陷拖成 timing 问题。

## 当 runtime 只在 `1/N` 实例生效时，先查执行作用域

动态参数问题还有一种高频误判：

- `ComponentParameter` 对了
- `ConfigMap` 对了
- `OpsRequest` 也 `Succeed`
- 但 runtime 不是全部实例都切到新值，而是只在 `1/N` pod 生效

这类问题不要先笼统写成：

- 参数没传进去
- reload 脚本完全没执行
- 或“控制面成功但 runtime 随机失败”

更应该先检查 **live-apply 的执行作用域**：

- `reconfigure.exec` 默认到底打到当前 pod，还是全部 pod
- 是否显式设置了 `targetPodSelector`
- 当前 case 需要的是单实例 apply，还是 `All` replicas 一起 apply

如果最终现象是：

- runtime 从 `0/N` 变成 `1/N`
- 补上 `targetPodSelector: All` 后收敛到 `N/N`

那更合理的结论通常是：

- 参数注入链路并非完全失效
- 问题在 **执行范围只命中了单实例**

因此，动态参数定向复验建议固定按这四层回报：

1. `ComponentParameter`
2. `ConfigMap`
3. `OpsRequest`
4. **按 pod 展开** 的 runtime truth

例如明确列出：

- `pod-0 = new value`
- `pod-1 = old value`
- `pod-2 = old value`

只有这样，团队才能快速区分：

- 是对象层没下发
- 是参数传值链路断了
- 还是 live-apply 作用域只命中了部分实例

一句话说：

- **部分实例生效，优先查 scope，再查 value**

## 静态参数验证口径

静态参数的预期结果通常是：

- 参数值生效。
- Pod 发生 rolling restart。
- Pod UID 变化。
- restart count 或新 Pod 状态能证明重启发生。
- OpsRequest Succeed。
- Cluster Running。

静态参数测试应选边界清晰的参数，避免选择会被系统直接拒绝、或语义不明确的参数。

## 区分 Addon 问题、测试环境问题和入口工具问题

排障记录中应明确区分三类问题：

### Addon 问题

例如：

- 缺少 `dynamicParameters`。
- 缺少 live-apply 执行路径。
- `reconfigure.exec` 或 legacy `reloadAction` 写错层级。
- reload 命令无法在实际执行容器中运行。

### 测试环境问题

例如：

- 本地 LoadBalancer 实现限制。
- hostPort 冲突。
- 镜像或平台差异。

### 入口工具问题

例如：

- CLI 生成的 OpsRequest 不正确。
- CLI 解析模板失败。
- CLI 与 direct Kubernetes API 路径行为不一致。

入口工具问题应单独记录。不要把 CLI bug 混入 Addon 能力是否正确的结论中。

## 关联能力验证

在交付 reconfigure 修复前，建议同步确认若干**与参数变更不直接耦合，但会影响 Addon 可用性与交付可信度**的独立能力，例如：

- TLS-only 连接与复制
- ACL / 认证
- backup / restore
- upgrade / rolling upgrade
- external access

这类能力建议作为**独立回归项**维护，不要和 reconfigure 用例混在同一个主结论里。

如果某个引擎已经补了具体的 TLS / ACL / 备份案例，建议收在对应主题文档中，而不是写进 reconfigure 主结论里。

## 案例附录：Valkey R08a

Valkey R08a 验证的是 `maxmemory-policy` 动态修改后能否不重启 Pod 即时生效。

本次排查过程中的关键结论：

- 最初 `valkey9-pd` 缺少 `dynamicParameters` 与 `reloadAction`，导致动态参数退化为 rolling restart。
- 只补 `dynamicParameters` 不够，还必须补齐有效的 `reloadAction`。
- `reloadAction` 需要写在正确层级，否则可能被 KubeBlocks 忽略。
- YAML 中一度出现重复 `reloadAction`，需要收敛为唯一一份。
- `shellTrigger` 的实际执行上下文是 config-manager sidecar，而不是 Valkey 主容器。
- 探针确认当前有效参数传递契约是 argv，即 `$0=key`、`$1=value`，不是 stdin。
- `toolsSetup + valkey-cli` 路线因为 glibc / musl ABI 不兼容而放弃。
- 最终收敛为 `nc + $0/$1 argv` 方案：在 config-manager 中根据 `$0/$1` 构造 Redis 协议，通过 `nc` 访问 `127.0.0.1:6379` 执行 `CONFIG SET`。
- 最终修复链中移除了 `toolsSetup`，也不再依赖 `batchReload` / stdin 路径。

最终验证结果：

- 自动化回归结果为 R08a `14 PASS / 0 FAIL`。
- `nc` 路径确认，reload action exit 0。
- `CONFIG GET maxmemory-policy` 在 3 个 data pod 中均为 `allkeys-lru`。
- 3 个 data pod 的 Pod UID / startTime 未变化。
- restart count 未变化，确认未发生重启。
- OpsRequest Succeed。
- Cluster Running。

结论：

- R08a dynamic reconfigure 已通过。
- 该结论范围限定为 direct `kubectl apply OpsRequest` 路径。
- `kbcli cluster configure` 的 `resolveConfigTemplate` 问题是独立入口问题，应单列记录，不并入 Addon 结论。

## 案例附录：Valkey R08b

Valkey R08b 验证的是静态参数变更是否会触发 rolling restart。

本次选择的样本参数是 `databases=20`。该参数属于 `staticParameters`，预期行为不是动态热加载，而是通过 rolling restart 让新配置生效。

验证结果：

- 自动化回归结果为 R08b `22 PASS / 0 FAIL`。
- `databases` 通过 OpsRequest 修改为 `20`。
- 3 个 data pod 的 UID 全部变化，确认 rolling restart 已发生。
- 3 个 data pod 上 `databases=20` 均生效。
- Cluster 回到 Running。
- 拓扑重新收敛为 1 master / 2 slaves。
- 测试完成后，`databases` 已 reset to `16`。

结论：

- R08b static reconfigure 已通过。
- `staticParameters` 变更会触发 rolling restart。
- 至此，Valkey reconfigure 的动态/静态边界均已验证：
  - R08a dynamic：参数动态生效，Pod 不重启。
  - R08b static：参数生效，并触发 rolling restart。

## 相关交付记录

- Feature issue: https://github.com/apecloud/kubeblocks-addons/issues/2591
- Reconfigure fix issue: https://github.com/apecloud/kubeblocks-addons/issues/2593
- PR: https://github.com/apecloud/kubeblocks-addons/pull/2592
- 关键 commit: `c2182bea feat(valkey): add dynamic parameter reconfiguration via nc-based shellTrigger`
- 验证环境：EKS `ap-southeast-1`，KubeBlocks `1.0.2`
- PR 汇总测试结果：
  - R08a dynamic param hot reload: `14 PASS / 0 FAIL`
  - R08b static param rolling restart: `22 PASS / 0 FAIL`
  - Full suite: `620 PASS / 0 FAIL`
- PR description 需要手工更新时，可使用团队在 `#all` 中整理的 markdown；`gh pr edit` 当前受 GitHub Projects classic deprecation GraphQL 错误影响。
- 手工更新 PR description 时，应同时关联：
  - `Closes #2591`
  - `Closes #2593`

## 相关主题

switchover / failover / 幂等性问题已拆到独立文档，避免与 reconfigure 主轴混在一起：

- [`docs/addon-switchover-guide.md`](addon-switchover-guide.md)
- [`docs/addon-tls-guide.md`](addon-tls-guide.md)

如果关注的是：

- switchover 状态语义 vs 因果语义
- target candidate 已达到目标角色时的幂等处理
- 原角色持有者退出旧角色的收敛校验
- 协调组件或拓扑缓存的收敛窗口
- shell UT 中 `$()` subshell mock 陷阱

请直接查看该文档。

如果关注的是 TLS 开启、TLS-only 强制、明文拒绝、证书挂载、TLS 与其他能力组合验证，请查看 [`docs/addon-tls-guide.md`](addon-tls-guide.md)。

## 用最小性验证确认哪些 reconfigure 防护不能省

当一个动态参数问题已经有一组可行修复时，仍需要做最小性验证，避免把排障探针或不必要的兼容逻辑带进最终提交。

建议方法是一次只移除一类防护，并用真实 runtime 作为判定标准，而不是只看 OpsRequest 是否成功：

- 如果高层 `Succeeded / Finished`，但数据库 runtime 仍是旧值，这不是通过，而是 false success。
- 如果 live hash / InstanceStatus 已到 target，但 runtime 没到 target，仍然不能算闭环。
- 如果移除某个防护后重新出现 false success，说明该防护或等价机制不能省。

以 `maxmemory-policy` 这类 shell 不友好的参数名为例，去掉 rendered-config fallback 是合理的，因为 rendered config 不应作为主目标来源。但如果同时去掉显式 action 参数输入和 runtime closure check，就会回到“hash/status 成功、runtime 未变化”的假成功。因此最终最小产品语义应保留两类能力：

- action 必须能读取显式目标参数，必要时由 controller 提供 shell-safe alias；
- action 必须用数据库 runtime 查询证明目标参数已经真正生效。

如果进一步做“只保留 shell-safe alias + 全 pod fan-out、移除 runtime closure check”的最小性验证，单次 R08a-only 通过只能说明 happy path 当前可行，不能直接证明 runtime closure check 作为产品 guard 可以安全删除。原因是这种组合依赖每个 pod 的 reload 都在本次时序里成功完成；一旦遇到角色切换、局部空成功、action 输出缺失或概率性时序窗，高层就少了最后一道真实 `CONFIG GET` 收口。要把 runtime closure check 从最终方案里删除，至少需要额外的重复 R08a、角色切换、pod 重建/chaos 场景证明，或者提供等价的结构化 runtime-proof 信号。

如果尝试只在 addon 侧读取 raw key，例如 `printenv 'maxmemory-policy'`，也必须单独验证。失败形态通常是：closure check 没有触发，controller / kbagent 只有空成功，高层 `Succeeded / Finished`，live hash / InstanceStatus 到 target，但真实 runtime 仍是旧值。这说明 raw key 没有形成稳定的 action 参数契约；此时不能把 addon-only raw read 当作最小修复。

还要单独验证 fan-out 语义。只保留“shell-safe 目标参数 + runtime closure check”仍可能不够：runtime check 可以发现其他 pod 还没生效并阻止 false success，但如果 action 只在一个 pod 上执行，它只会在这个 pod 上反复看到其他 pod stale，不能推动 stale pod 自己 reload。因此类似 `maxmemory-policy` 这种需要所有实例 runtime 闭环的参数，还需要以下两类能力之一：

- reconfigure action 明确对所有相关 pod 执行，例如 `targetPodSelector: All`；
- 或者有等价的全实例 fan-out / reconcile 机制，能保证每个 stale pod 都会执行对应 reload。

判定标准保持不变：高层成功必须同时满足真实 runtime、live hash、InstanceStatus 都达到目标；如果高层未 false success 但长期停在单 pod mismatch，也说明 runtime closure check 有效但 fan-out 不足。
