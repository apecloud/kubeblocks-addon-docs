# Addon Chart 与 KubeBlocks 控制器 Schema 代差诊断指南

> **Audience**: addon dev / test
> **Status**: stable
> **Applies to**: any KB addon
> **Applies to KB version**: 1.0.x / 1.1.x release tags / main (1.2 unreleased) — 方法论本身版本无关
> **Affected by version skew**: §5 + 案例附录涉及 KB main 上未发布的 reconfigure / parameters API 重构（PR #10100 / #10109，预计随 KB 1.2 发布）；reconfigure 接口的具体跨版本演进见 `addon-reconfigure-version-skew-guide.md`（TBD）

本文面向 Addon 开发与测试工程师。当 `helm install <addon-chart>` 在 server-side 报 `field not declared in schema` 时，本文教你 30 分钟内分清三种根因：chart 局部 bug、chart × KB 整代代差、chart 跟 KB main 但目标 API 还没发布。下结论前一定要做完几步取证，避免凭直觉切版本或改 chart。

## 先用白话理解这篇文档

这篇文档想讲清三件事：

- 同一个 `field not declared in schema` 报错，根因可能不是「chart 写错了」，也可能不是「KB 装错了」
- 真正想知道的是：**chart 期望的那个字段，在我现在装的这个 KB 版本里，是不是真的不存在**
- 在没坐实之前就改 chart / 切 KB 版本，多半是白干

排查顺序固定成：先看同仓 release 分支 → 再追字段绝对路径 → 最后做 cross-chart dry-run 对照。三步没走完不下结论。

## 适用现象

`helm install` 或 `helm upgrade --install` 报：

```text
Error: failed to create typed patch object (/<resource-name>; <group>/<version>, Kind=<Kind>): errors:
  .spec.<path>: field not declared in schema
```

或同等的 server-side apply 报错：

```text
Error from server (BadRequest): ... strict decoding error: unknown field "spec.<path>"
```

特征：报错来自 **apiserver / KubeBlocks CRD 的 OpenAPI strict decoding**，而非 helm template 本身或 controller 业务逻辑。

## 错误的应对（先排除三种"凭直觉"动作）

不要做以下任一项作为第一反应：

1. **直接改 chart 把报错字段删掉** — 字段可能是 chart 必需语义，删了会在更隐蔽的下游阶段失败。
2. **凭 chart `Chart.yaml` 的 `addon.kubeblocks.io/kubeblocks-version` annotation 判断兼容性** — 这个 annotation 经常不实，写 `>=1.0.0` 但实际假定的是更新 schema。
3. **uninstall 失败 release / 清理 partial CR** — 污染现场会让后续根因取证失去基础证据。

## 正确取证流程

### 1. 锁定具体被拒字段 + 对应 CRD 实有字段

把 helm 报错里所有 `field not declared in schema` 的路径列全（注意不止一个，常见会有 3-5 个不同字段、跨多个 Kind）。

对每个 Kind 查当前 KB 实有的 schema：

```bash
# 列出某 CRD 注册的所有 version + 哪个是 storage version
kubectl get crd <kind>.<group> -o json | jq -r '.spec.versions[] | "\(.name) storage=\(.storage)"'

# 查 storage version 下某个 spec 子字段实有的 keys
kubectl get crd <kind>.<group> -o json \
  | jq -r '.spec.versions[] | select(.name=="<storage-version>")
           | .schema.openAPIV3Schema.properties.spec.properties.<sub>.<...> | keys[]'
```

记下「chart 用的字段」与「CRD 实有字段」的差集。

### 2. 关键校验：字段名相同 ≠ 路径相同 ≠ 同字段（必做）

最常见的误判：在 CRD yaml 里 grep 到 `reconfigure:` 字眼，就以为 chart 里 `cmpd.spec.configs[0].reconfigure` 在新 KB 已支持。

**字段名相同不等于字段同路径**。同名字段可能挂在完全不同的 yaml 路径下，对应完全不同的语义。验证方法：

```bash
# 错误用法（结果跨多个路径无法区分）
grep -n 'reconfigure:' /tmp/<kb-version>-crds.yaml

# 正确用法：把字段从根开始追完整 yaml 绝对路径
yq eval '.. | select(has("reconfigure")) | path | join(".")' /tmp/<kb-version>-crds.yaml

# 或用 kubectl explain 走绝对路径
kubectl explain componentdefinition.spec.configs --recursive | grep -A1 reconfigure
kubectl explain componentdefinition.spec.lifecycleActions.reconfigure
```

判读三个层次：

- **字段名出现 + 路径相同** → chart 字段在 KB 实有；可能不是这个字段的问题。
- **字段名出现 + 路径不同** → chart 字段位置写错；改 chart，把字段挪到正确路径。
- **字段名不出现** → chart 该字段确实不被 KB 支持；要么 chart bug，要么整代代差，进 §4。

**Kind 也要追绝对路径**：chart yaml 写 `kind: X` 但 spec 字段属于 `kind: Y`（独立 CRD），是同样的错位。先看 chart 的 `apiVersion + kind`，再去对应 CRD 查 spec 实有字段。

### 3. 关键校验：先看同仓的 release 分支（容易被忽略，常常是答案）

在判定「chart 跟 main 走、整代代差」之前，**先去同 chart 仓 `git branch -r | grep release` 看有没有平行的 release 分支**。多 chart 仓采用「main 跟 KB main 走、`release-X.Y` 分支跟 KB X.Y release 走」的并行维护模式。

```bash
git -C <addons-repo> branch -r | grep -E '(release|stable)'

# 抽离到 /tmp 不污染 main
mkdir -p /tmp/<chart>-relX.Y && \
  git -C <addons-repo> archive origin/release-X.Y addons/<chart> | tar -x -C /tmp/<chart>-relX.Y
```

判读：

- **release-X.Y 分支 chart 在你目标 KB 上 dry-run 过** → 直接用这个分支的 chart，不必走 build KB main / 改 chart。
- **release-X.Y 不存在 / 也跟 main 走** → 进 §4。

> ⚠️ 踩坑教训：上来只看 main HEAD chart、没扫 `release-*` 分支，于是把简单的「同仓多分支」误判成「chart 目标未发布 API」，多花数小时走切版本 / build main 等冤枉路。**先 `git branch -r`，30 秒成本，可能直接给出答案**。

### 4. Cross-chart 对照实验

判断「chart 局部 bug」还是「整代 chart × KB 版本代差」必须做对照，不能只看一个 chart。

挑同 chart 仓、同 chart version 系列下的另一个 addon（最好是结构最简单的，比如 mongodb / mariadb / etcd），用 server-side dry-run 验证：

```bash
cd <addons-repo>/addons/<peer-chart>
helm dependency update . 2>&1 | tail -3
helm template <release-name> . > /tmp/<peer>-rendered.yaml

kubectl create namespace <peer>-dryrun-test
kubectl apply -n <peer>-dryrun-test --dry-run=server -f /tmp/<peer>-rendered.yaml
kubectl delete namespace <peer>-dryrun-test
```

判读：

- **对照 chart 报相同的 `field not declared in schema`** → 整代 chart 与当前 KB schema 不兼容；不是你的 chart bug；不要改 chart，改 KB 版本（先回 §3 看 release 分支）。
- **对照 chart 通过** → 失败仅限于你的 chart；定位到具体多余/拼写错的字段去修。

### 5. 第三种根因：chart 跟 KB main，但目标 API 还没发布 release

如果 §3 没找到 release 分支、§4 对照 chart 同样失败，下一步**不要立刻切 KB 版本**。先做三步取证确认 chart 是不是跟 KB main 走在了已发布 tag 之前：

```bash
# (1) 找 chart 最近的 API 迁移 commit
git -C <addons-repo> log -- addons/<chart>/templates/<key-file>.yaml | head -20

# (2) 在 KB 上游仓 grep 对应 PR
git -C <kubeblocks-repo> log --grep='<API name>' --oneline | head

# (3) 验证 KB PR 是否进了任何 released tag
git -C <kubeblocks-repo> tag --contains <kb-pr-commit>
```

如果第 (3) 步返回空，**chart 不是 bug**，是 chart 跟 main，没有任何 released tag 能装它。

解决路径与「chart bug」完全不同：

- 不要改 chart（除非接受长期回滚维护成本）
- 候选：从 KB main 自己 build controller image / 等下个 release / 直接联系 chart owner 问他们怎么测的（他们一定有内部可用的 KB image 或本地 build 流程）
- chart owner 路径成本最低：commit author email / GitHub handle 一查就知道

## 保护边界

调试期间务必：

- **不 uninstall 失败 release**（保留作为对照基线）
- **不清理 partial CRD/CR**（部分对象已建会影响重装路径，但那是真实生产场景，不要伪造干净环境）
- **不改 chart annotation**（annotation 不实只是症状，根因是 chart 字段实际依赖什么 schema）
- 切 KB 版本是大动作，**owner 决策**，单独取证不 unilateral 推进

## 与其他文档的关系

- `addon-componentdefinition-upgrade-guide.md` 讲 same-name CD 升级时的 immutable spec blocker；本文讲 chart 整代字段不匹配。两者都是 apply 阶段失败，但分层不同。
- `addon-test-environment-gate-hygiene-guide.md` 讲「runner 跑起来之前环境就绪」；本文是 chart install 阶段、还没到 runner。

## 案例附录

- Oracle：[`docs/cases/oracle/oracle-chart-vs-kb-schema-skew-multi-stage-case.md`](cases/oracle/oracle-chart-vs-kb-schema-skew-multi-stage-case.md) — Oracle 1.0.0-alpha.0 chart 在已发布 KB 装不上的三段反转：先误判「整代代差」、再误判「chart 字段路径错位」、最后查清是「chart 跟 KB main 上未发布 API」。同仓 `release-1.0` 分支才是答案。
