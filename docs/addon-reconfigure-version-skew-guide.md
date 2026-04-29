# Addon Reconfigure 跨 KB 版本演化指南

> **Audience**: addon dev / TL，跨 KB 版本（1.0 / 1.1 / 1.2 main）维护同一个 addon 的工程师
> **Status**: draft
> **Applies to**: 任何在 KB 多个发布带之间复用的 addon，特别是依赖动态参数 reload 的引擎（Redis / Valkey / MariaDB / MySQL / PostgreSQL 等）
> **Applies to KB version**: 跨 1.0 / 1.1 / 1.2 main 三个发布带；本文档自身就是 version-skew 主题
> **Affected by version skew**: 本文档的主体即版本演化本身

## 先用白话理解这篇文档

### 这篇文档解决什么问题

一个 addon 写好之后，KB 升级到下一个发布带，addon 的 reconfigure 行为可能**静默退化** —— OpsRequest 仍报 Succeed，但 runtime 上参数没真生效；或者反过来，本来该 hot apply 的参数被退化成 rolling restart。

因为 **KB 1.0 / 1.1 / 1.2 main 给 addon 的 reconfigure 接口形态在 1.2 这一带发生了显著变化**：legacy `paramsdef.yaml` 里的 `reloadAction.shellTrigger` 在 1.2 main 上被 deprecated / removed，reconfigure 路径必须走 `ComponentDefinition.spec.configs[].reconfigure.exec`。

addon 同一份 chart 部署在不同 KB 版本上，行为分歧需要 addon 维护者主动管理。

### 读完你能做什么决策

- **拿到一个 addon chart 时**：能在 30 秒内判断它针对哪个 KB 发布带写的 reconfigure（chapter 2）
- **决定目标 KB 版本时**：能预测 chart 的 reconfigure 在那个版本上会走哪条路径（chapter 3）
- **跨版本部署时**：能识别出"controller 成功但 runtime 没真改"的 silent failure 形态（chapter 5 + chapter 7）
- **为多版本 KB 维护一个 chart 时**：知道双写、单写选边、template variant 三种 portability 套路各自的 trade-off（chapter 6）

### 为什么独立成篇

`addon-reconfigure-guide.md` 是 reconfigure 主题的**实现 + 排障**通用指南，**默认假设单一 KB 版本**。但跨版本演化是一个独立维度的问题：同样 addon 代码在不同 KB 上可能走完全不同的路径，trap 模式跟版本组合强绑定。

这篇文档专门讲版本演化轴。和 `addon-chart-vs-kb-schema-skew-diagnosis-guide.md` 形成 cross-KB 主题的**对子**：那篇看 chart schema 与 KB API 的 mismatch；这篇看 reconfigure 接口语义在 KB 升级中的演化与 silent regression 风险。

---

## 主体内容

**关键术语**（首次出现先解释，后续直接使用）：

- **legacy reload 路径**：`paramsdef.yaml` / `ParametersDefinition` 里的 `reloadAction.shellTrigger`；KB 1.0.x / 1.1.x 的主推路径，1.2 main 进入 deprecated / 移除窗口
- **CMPD reconfigure 路径**：`ComponentDefinition.spec.configs[].reconfigure.exec`；KB 1.2 main 的主推路径，对 1.0.x / 1.1.x 控制器在多数情况下被忽略
- **silent regression**：reconfigure OpsRequest 报 `Succeed`、controller hash 收口、但 valkey/redis runtime 的 `CONFIG GET` 仍返回旧值；跨版本演化里最常见的 trap 形态
- **fan-out 语义**：reconfigure 是只在 primary 上执行（KB 默认）还是分发到所有 pod（addon 显式 `targetPodSelector: All`）
- **version band**：指 KB 的 release tag 区间，例如 1.0.x / 1.1.x / 1.2.x（含 main）；不同 version band 的接口契约可能不一致

## chapter 1：问题陈述 — addon reconfigure 不 version-portable

把 addon 当成"业务实现"看，把 KB 当成"运行平台"看，本来期望接口是 stable 契约。但 reconfigure 这一面在 KB 主版本切换时**契约本身**在变：

| 维度 | KB 1.0.x / 1.1.x | KB 1.2 main |
|---|---|---|
| 主推 reload 入口 | `paramsdef.yaml` 里 `reloadAction.shellTrigger` | `ComponentDefinition.spec.configs[].reconfigure.exec` |
| legacy `reloadAction` 是否仍生效 | ✅ 主路径 | ⚠️ deprecated / `unsupported legacy reloadAction` 错误（视控制器 feature flag） |
| `reconfigure.exec` 是否生效 | ⚠️ 多数版本忽略，**不是**官方契约 | ✅ 主路径 |
| fan-out 默认 | primary only（addon 需自己 loop FQDN list） | 仍是 primary only，**addon 需显式 `targetPodSelector: All`** |
| ParametersDefinition `dynamicParameters` 列表 | 必须在 `paramsdef.yaml` 给出 | 同上仍需要（不绑定 reload 路径） |
| config externalManaged | 1.0 / 1.1 也支持，但 reload 链路依赖 legacy config-manager | 1.2 main 必须有 `ParamConfigRenderer` 等渲染绑定 |

→ **同一份 addon chart 部署到不同 KB 版本，行为可能 wholly 不同**：
- 在 1.0/1.1 上跑得好的 addon，部署到 1.2 main 可能 reload 完全不走（silent regression）
- 在 1.2 main 上写的 addon 部署到 1.0/1.1，reload 入口完全错位（OpsRequest 走 controller restart fallback 或 fail）

**这条不是文档错位 / 排障细节**，是 addon 维护者必须主动管理的 cross-version skew。

## chapter 2：拿到一个 addon 时怎么判断它针对哪个 KB 版本写的

不依赖 git history、不依赖 commit message，**仅看 chart 文件结构**就能在 30 秒内判断：

| 信号 | 推断目标 KB 版本带 |
|---|---|
| `paramsdef.yaml` 含 `reloadAction.shellTrigger` 块 | 主目标是 1.0.x / 1.1.x（也可能跨版本兼容写法） |
| `cmpd.yaml` 含 `spec.configs[].reconfigure.exec` 块 | 主目标包含 1.2 main |
| 同时含两者 | 跨版本兼容写法 / 迁移过渡态 |
| 仅 `paramsdef.yaml` 有 reload，cmpd 无 reconfigure | **1.0/1.1 only**，部署到 1.2 main 大概率 silent regression |
| 仅 cmpd `reconfigure.exec`，paramsdef 无 reloadAction | **1.2 main only**，部署到 1.0/1.1 reload 不会走 |
| `cmpd.yaml` reconfigure.exec 含 `targetPodSelector: All` | 维护者已意识到 fan-out trap，是 1.2 main aware |
| 不含 `targetPodSelector: All` 但 reload-parameter.sh 自己 loop FQDN list | 1.0/1.1 风格的 fan-out 实现 |

**verify the inferred version**（30 秒互证）：
- chart 的 `Chart.yaml` `appVersion` 通常对应 engine 版本，不直接对应 KB 版本
- 看 `templates/_helpers.tpl` 里有没有 `kblib` 引用，引用版本 hint KB 兼容范围
- 看 chart 历史 PR 的 base branch 和 merge 时点，对照 KB 同期的 release 节奏

**最强的 ground truth**：**直接 deploy 到目标 KB 版本上发起一次 minimum reconfigure OpsRequest**，观察：
- controller phase（Succeed / Failed）
- live `ParametersDefinition` / `ComponentParameter` / `OpsRequest` 状态
- runtime `CONFIG GET` 实际值（**这条最关键**，下面 chapter 5 详述）

## chapter 3：目标 KB 版本会走哪条路径 — controller 角度的判定

addon 自己怎么写 vs controller 实际怎么处理是两件事。controller 角度的判定规则：

- **KB 1.0.x / 1.1.x controller**：扫 `paramsdef.yaml` 的 `reloadAction.shellTrigger`，调 legacy config-manager，sidecar 模式跑 reload；对 cmpd `reconfigure.exec` 多数版本不识别或仅做有限 fallback
- **KB 1.2 main controller**（参考 PR 节点，例如 Valkey R08a 案例里的 #10182 / #10185）：
  - 扫 cmpd `reconfigure.exec` 直接 exec 到目标 pod 容器
  - legacy `reloadAction` 进入 unsupported 路径：`OpsRequest=Failed`、`ComponentParameter=FailedAndPause`、errMessage `unsupported legacy reloadAction ... legacy-config-manager-required is not enabled`
- **fan-out 默认仍是 primary**（不论版本带）：
  - addon 必须 `cmpd.reconfigure.exec.targetPodSelector: All` 才能让 controller fan out 到所有 pod
  - 或者 reload-parameter.sh 自己 loop `VALKEY_POD_FQDN_LIST` / `REDIS_POD_FQDN_LIST` 这种 component env vars

**判断目标 KB 版本上 reload 路径**的 30 秒：

```
target KB version?
├─ 1.0.x / 1.1.x → 检查 paramsdef.yaml.reloadAction
│       ├─ 存在 → reload 通过 legacy config-manager 走（addon ok）
│       └─ 缺失 → reload 不会触发；reconfigure 退化为 controller restart fallback
└─ 1.2 main → 检查 cmpd.reconfigure.exec
        ├─ 存在 + targetPodSelector All → reload 走 cmpd path 在所有 pod 上
        ├─ 存在 + targetPodSelector 缺失 → reload 仅在 primary 上（数据面分歧）
        └─ 缺失，仅有 paramsdef.reloadAction → unsupported legacy reloadAction，OpsRequest=Failed
```

## chapter 4：reconfigure policy（dynamic / static / hybrid）选择规则跨版本不同

每个参数在 reconfigure 时可走三种 policy 之一：

- **dynamic**（hot apply）：直接调 engine `CONFIG SET` 等命令，不重启
- **static**（restart required）：必须 rolling restart 才生效
- **hybrid**：尝试 dynamic，失败回退 static

跨版本差异：

- **policy 判定来源**：
  - 1.0/1.1：以 `paramsdef.yaml` 里 `dynamicParameters` 列表为权威，reloadAction 落到具体 reload 脚本
  - 1.2 main：仍以 `paramsdef.yaml` 里 `dynamicParameters` 列表为权威，但 dispatch 落到 `cmpd.reconfigure.exec`；如果 paramsdef 漏标 dynamic，**两版本都会**走 static restart

- **dynamic 失败回退**：
  - 1.0 部分版本：失败时 controller 可能 silent retry → fall back to static restart 不显式标 fail
  - 1.2 main：失败立刻 OpsRequest=Failed（更严格）；addon 侧 reload-parameter.sh 必须显式 exit 1 让 controller 拿到错误

- **CONFIG SET 静默"成功"语义**：
  - valkey-cli / redis-cli `CONFIG SET` 对部分 deprecated alias 或 enum 越界会返回 `+OK` 但**实际值未应用**
  - 1.0/1.1 时代靠 controller 端 hash 兜底，1.2 main 把这层契约下放给 addon → addon 必须自己做 `CONFIG SET` 之后立即 `CONFIG GET` 验证

## chapter 5：identifying silent regression 的现场形态

跨版本部署最危险的 trap 是 silent regression：**OpsRequest 报 Succeed，但 runtime 实际未应用**。识别口径：

| 检查层 | 不可信的"成功"信号 | 唯一可信的 ground truth |
|---|---|---|
| controller | `OpsRequest=Succeed`、`ComponentParameter=Succeed`、live config-hash 收口 | — |
| addon action | `reload-parameter.sh` exit 0、reload action log 无 error | — |
| runtime（**唯一可信**） | — | 在每个数据 pod 上 `redis-cli CONFIG GET <param>` 直接读 runtime 值，对照 desired |

**typical silent regression 现场**：
- chart 是 1.0/1.1 风格（仅 paramsdef.reloadAction），部署到 1.2 main controller
  - controller 不认 legacy reloadAction → 但有些过渡版本不抛错，走 noop 或 controller restart
  - OpsRequest phase Succeed（错误地认为 reload 完成），但所有 pod runtime 仍是旧值
- chart 是 1.2 main 风格（cmpd.reconfigure.exec 但**漏 targetPodSelector: All**），部署到 1.2 main
  - controller 仅在 primary 上 exec → reload 仅在 primary 生效，replica 仍是旧值
  - 一次 switchover 后旧值随新 primary 回到 cluster，silent drift
- chart 双写（cmpd.reconfigure.exec + paramsdef.reloadAction），部署到 1.0/1.1
  - controller 走 legacy 路径 ok
  - cmpd reconfigure 被忽略 ok
  - 但如果 reload-parameter.sh 在两条路径下行为不一致（参数名转换、authentication、retry 逻辑），会有微妙 drift

**actionable**：跨版本部署时 reconfigure 验证至少要 **bring-down 到 runtime CONFIG GET 一次**，不能只看 controller phase。

## chapter 6：portability 套路 — 双写 / 单写 / template variant

| 套路 | 怎么做 | 优点 | 缺点 |
|---|---|---|---|
| **双写共存** | paramsdef.reloadAction + cmpd.reconfigure.exec 同时写在 chart 里 | 跨版本部署都能 reload；最 conservative | reload-parameter.sh 必须容忍两条路径都来，参数名 / argv 形态可能要 normalize；维护两套断言点 |
| **单写选边** | 只写一种，按 chart Helm value 切换（`values.yaml` 加 `kbVersionFlavor: legacy / cmpd`） | chart 干净，每版本走单一路径 | 需要 chart consumer 知道 flavor 值；不指定时容易选错 |
| **template variant** | chart 在 `templates/` 下用 Helm `if` 按 capability 探测渲染不同 reconfigure 块 | chart consumer 不感知；自动适配 | Helm template 复杂度上升；capability 探测靠什么信号需要约定（一般看 `.Capabilities.APIVersions` 或显式 KB version label） |

实战观察：
- **双写共存** 在 cross-version 风险高的 addon（Redis / Valkey）里是当前 mainstream 做法
- **单写选边** 适合 fork 多个 chart 实例的场景（比如 redis-1.0 / redis-1.2 各一份 chart）
- **template variant** 适合 capability 探测可靠的环境（KB 1.2 后会更可靠）

## chapter 7：常见 trap 清单

把跨版本部署中观察到的具体踩坑形态归类：

### Trap T1：paramsdef.reloadAction 留着但 1.2 main 已 ignore

**形态**：chart 同时含 paramsdef.reloadAction 和 cmpd.reconfigure.exec，但 cmpd 的 exec 漏掉关键参数。部署到 1.2 main，reloadAction 完全不走。

**识别**：在 1.2 main 上发 reconfigure，看 controller 日志是否有 `unsupported legacy reloadAction` 或 silent skip。

**修复**：把 reload 逻辑迁到 reload-parameter.sh 主路径，cmpd.reconfigure.exec 调它；paramsdef.reloadAction 改为同 reload-parameter.sh 入口，保证两条路径行为一致。

### Trap T2：cmpd.reconfigure.exec 加了但 1.0/1.1 ignore

**形态**：把 reload 全迁到 cmpd 后，部署到 1.0/1.1 controller，reload 不触发。

**识别**：在 1.0/1.1 上发 reconfigure，看 ComponentParameter 是否进 `Pending` 或 controller 报"missing reloadAction"。

**修复**：保留 paramsdef.reloadAction 兜底；如果不愿双写，改用 chart flavor 切换。

### Trap T3：fan-out 漏 `targetPodSelector: All` → 仅 primary 生效

**形态**：1.2 main 上 cmpd.reconfigure.exec 成功跑了，但只在 primary 上跑。reload 后 primary 是新值，replicas 还是旧值。

**识别**：每个 pod 上 `CONFIG GET` 看一致性。

**修复**：cmpd.reconfigure.exec 加 `targetPodSelector: All`；或 reload-parameter.sh 自己 loop pod FQDN list（依赖 component env var）。

### Trap T4：CONFIG SET 静默 "+OK" 但 runtime 未应用

**形态**：valkey-cli / redis-cli `CONFIG SET` 对 deprecated alias 或越界 enum 返回 `+OK`，但实际拒绝写入 runtime。

**识别**：CONFIG SET 后立即 CONFIG GET 验证。

**修复**：reload-parameter.sh 在 CONFIG SET 后必须做 CONFIG GET verify，mismatch 视为 reload 失败 exit 1（让 controller 拿到错误）。注意：**这条防御对 KB 控制器已经修过 #10182/#10185 之后是冗余防御**，新 controller 已经更严格；但跨版本部署时仍有价值，特别是在 1.0/1.1 controller 上。

### Trap T5：configSpec-source / config-hash marker 跨版本契约漂移

**形态**：1.0/1.1 用 `configSpec` 作为 reload trigger source；1.2 main 用 hash marker 做 idempotency。chart 漏配置 marker 字段，1.2 main 可能误判 reload 已发生。

**识别**：看 controller 是否在新 reconfigure 上跳过 reload action 调用。

**修复**：跟 KB 当前 release 的 ParametersDefinition contract 对齐 marker 字段（1.2 main 见 KB controller PR #10182 / #10185 的契约文档）。

### Trap T6：init-script 跨版本残留（旧 ComponentDefinition / ComponentVersion 留下来）

**形态**：addon 升级 / 回滚后 Helm release 切到目标版本，但带 keep 策略的 ComponentDefinition / ComponentVersion 残留新版本语义，data pod 卡 init `cp: can't stat '/config'`。

**识别**：pod describe + init container 日志，对照 chart manifest live 对象是否真的回到目标版本。

**修复**：清理 keep 资源污染，对照 manifest 确认 live 对象 align。这个 trap 跟 chart-vs-kb-schema-skew-diagnosis 主题重叠，详情参见 `addon-chart-vs-kb-schema-skew-diagnosis-guide.md`。

## chapter 8：测试矩阵

跨版本 addon 必须在 release 前覆盖以下矩阵的最小子集：

| addon chart 版本 | 部署到 KB 1.0.x | 部署到 KB 1.1.x | 部署到 KB 1.2 main |
|---|---|---|---|
| chart 主线 | smoke + reconfigure 子集 | smoke + reconfigure 子集 | full smoke + chaos + reconfigure 子集 |
| chart 兼容 fallback | reconfigure 子集（验证 fallback 不退化）| 同左 | 同左 |

**reconfigure 子集**最小至少包含：
- 1 个 dynamic 参数（hot apply）：每个 pod runtime CONFIG GET 验证
- 1 个 static 参数（restart required）：rolling restart 完成后所有 pod runtime 验证
- 1 个 hybrid 参数（dynamic try → static fallback）：两条路径 case 都验
- 1 次 OpsRequest 在 disruption（chaos kill）下发起：reconfigure 在 disruption 期间是否能完成或正确 fail

**fan-out 验证**：reconfigure 1 个会被所有 pod 看到的参数（如 `maxmemory-policy`），跨版本都要在每个 pod 上 CONFIG GET 验证。

**自动化角度**：runner 对每个版本带跑 `--test reconfigure` 子集，verify 落到 `each pod 上的 runtime CONFIG GET`，不能只看 OpsRequest phase。详细 verify 节奏参考 `addon-bounded-eventual-convergence-guide.md`。

## chapter 9：跟其他 cross-version 主题的关系

本文档跟以下文档形成 cross-version 主题的对子，互相 cross-ref：

- `addon-chart-vs-kb-schema-skew-diagnosis-guide.md`（James 主笔）：聚焦 chart schema 与 KB API 的 mismatch（"chart 写 X 字段但目标 KB 不认得"），解决 setup-time blocker
- 本文档：聚焦 reconfigure 接口语义在 KB 升级中的演化与 silent regression 风险，解决 runtime-time silent failure

两篇 audience 重叠（都是跨 KB 版本维护 addon 的工程师），methodological 互补：
- chart-vs-kb-schema-skew 解决"deployment 阶段被 KB schema 验证挡住怎么办"
- 本文档解决"deployment 通过了，但 reconfigure 在 runtime 层 silent failure 怎么办"

## 案例附录

详细案例：

- `docs/cases/valkey/scale-out-targeted-switchover-stale-pod-list-case.md` — 跨版本 fan-out 语义变化导致 stale list 现场
- Valkey R08a baseline lock case（参见 `addon-test-host-stress-and-pollution-accumulation-guide.md` 附录 B）：KB controller PR #10182 + #10185 修了 1.2 main 上 reconfigure runtime verification 的 silent regression；这条修复链是本指南 trap T4 的 anchor case
