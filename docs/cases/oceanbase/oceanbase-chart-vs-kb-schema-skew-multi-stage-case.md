# OceanBase Chart × KB Schema Skew — framework 第一次落地实证

> **Audience**: addon dev / test
> **Status**: stable（决议段 pending — 等 chart-side 路径 B/E 拍板，落盘后此段补完）
> **Applies to**: OceanBase enterprise addon（`apecloud-addons/addons/oceanbase`，Chart 1.0.2，appVersion `4.2.1-BP7-Hotfix2`，license `LicenseRef-KubeBlocks-Enterprise`）。方法论可复用，参见 [`addon-chart-vs-kb-schema-skew-diagnosis-guide.md`](../../addon-chart-vs-kb-schema-skew-diagnosis-guide.md)
> **Applies to KB version**: 验证于 KB 1.0.2（不可装）；最低可装 KB **v1.2.0-alpha.0+**（已被 stable lock 否决，见决议段）；release-1.0 分支也只满足部分字段
> **Affected by version skew**: `cmpd.spec.configs[].reconfigure`（KB PR #10016，首次 tag v1.2.0-alpha.0）/ `cmpd.spec.roles[].isExclusive`（KB PR #10049，首次 tag v1.0.3-beta.0）/ `parametersdefinition.spec.componentDef`+`templateName`+`fileFormatConfig`（KB PR #10100，首次 tag v1.2.0-alpha.0）

本文是 [`addon-chart-vs-kb-schema-skew-diagnosis-guide.md`](../../addon-chart-vs-kb-schema-skew-diagnosis-guide.md) 的工程现场补充，与 [`oracle-chart-vs-kb-schema-skew-multi-stage-case.md`](../oracle/oracle-chart-vs-kb-schema-skew-multi-stage-case.md) 形成 cross-engine pair。两案区别：

- **Oracle case**：framework 出来之前的"摸石头"现场，三段反转，每段被下一段推翻，最后才落到正确路径。Oracle 的价值是**记录 stochastic 路径里被忽略的关键证据**。
- **OceanBase case**（本文）：framework 出来之后的**第一次跨 engine 落地**。framework 三步走完直接定盘 ②/③ 复合，无反转。OceanBase 的价值是**证明 framework 不是空想方法论，能把 chart-vs-KB schema skew 这类问题压缩到 30 分钟以内**，并且暴露了原 framework 缺失的一根轴（install-path lenient-strip vs helm typed-patch reject），反向 PR 进主文档作 Step 4。

阅读对象：第一次撞 chart-vs-KB schema skew 的 addon dev / test，按 framework 顺序执行即可。

## 起点

- chart：`apecloud-addons/addons/oceanbase`（chart `1.0.2`，appVersion `4.2.1-BP7-Hotfix2`）
- 集群：k3d 单节点 Machine B（`caoweis-MacBook-Air.local`），KB v1.0.2
- chart annotation：`addon.kubeblocks.io/kubeblocks-version: ">=1.0.0"`（不实，与 Oracle case 同模式）
- 触发：smoke test runner `kubeblocks-tests/oceanbase/run-tests.sh` 跑前 `helm install` 失败：

  ```
  helm install -n kb-system oceanbase apecloud-addons/addons/oceanbase
  Error: INSTALLATION FAILED: failed to create typed patch object:
    /oceanbase-dist-1.0.2 ComponentDefinition v1
      .spec.configs[0].reconfigure: field not declared in schema
      .spec.configs[1].reconfigure: field not declared in schema
    /oceanbase-rep-ocp-1.0.2 ComponentDefinition v1
      .spec.configs[0].reconfigure / .spec.configs[1].reconfigure / .spec.roles[0].isExclusive
    /oceanbase-sysvars-cc-repl ParametersDefinition v1alpha1
      .spec.componentDef
    /oceanbase-sysvars-cc / oceanbase-parameters-cc / oceanbase-parameters-cc-repl
      .spec.templateName
  ```

  evidence pack：`kubeblocks-tests/oceanbase/evidence/install-fail-001/`（install-error.txt + env.txt + schema-mismatch-summary.md，**未做任何修改**前固定）。

## 执行 framework — 30 分钟从报错到定盘

### Step 1（同仓 release 分支扫描）

```
$ git -C ~/ApeCloud/apecloud-addons branch -a
* main
  remotes/origin/HEAD -> origin/main
  remotes/origin/main
$ git -C ~/ApeCloud/apecloud-addons tag --list
(empty)
```

→ apecloud-addons 仓**只有 main，没有 release 分支也没有 tag**。这意味着 Oracle case 里"`release-1.0` 分支才是答案"那条路径**在 OceanBase 这里不存在**。Step 1 直接排除 ① "chart 局部 bug 等待 release 分支修"。

### Step 2（字段绝对路径追踪 + tag containment）

3 个不匹配字段都在 KB 源码里**真实存在**，但首次出现晚于 v1.0.2：

```bash
$ cd ~/ApeCloud/kubeblocks
$ git log --oneline --all -S 'Reconfigure *Action `json:"reconfigure,omitempty"`' \
    -- apis/apps/v1/componentdefinition_types.go
d5f8fb803 feat(apis): enhance API for config manager migration to kbagent (#10016)
$ git tag --contains d5f8fb803 | head
v1.2.0-alpha.0

$ git log --oneline --all -S "IsExclusive bool" -- apis/apps/v1/componentdefinition_types.go
510442140 chore: support exclusive replica roles to prevent stale labels (#10049)
$ git tag --contains 510442140 | head
v1.0.3-beta.0

$ git log --oneline --all -S 'ComponentDef string `json:"componentDef,omitempty"`' \
    -- apis/parameters/v1alpha1/parametersdefinition_types.go
4ec01470c chore(parameters): deprecate ParamConfigRenderer by moving bindings into ParametersDefinition (#10100)
$ git tag --contains 4ec01470c | head
v1.2.0-alpha.0
```

汇总：

| 字段 | KB commit | 首次出现 tag |
|---|---|---|
| `cmpd.spec.configs[].reconfigure` | `d5f8fb803` PR #10016 | **v1.2.0-alpha.0** |
| `cmpd.spec.roles[].isExclusive` | `510442140` PR #10049 | **v1.0.3-beta.0**（在 release-1.0 分支） |
| `parametersdefinition.spec.componentDef` + `templateName` | `4ec01470c` PR #10100 | **v1.2.0-alpha.0** |

→ KB v1.0.2 release **没有任何一个**这些字段。即便切到 release-1.0 最新 beta 也只满足 `isExclusive`。**最低跑通需要 KB v1.2.0-alpha.0**。

### Step 3（cross-chart dry-run）

```bash
$ for addon in mysql mongodb mssql; do
    rec=$(grep -l "reconfigure:" apecloud-addons/addons/$addon/templates/cmpd-*.yaml 2>/dev/null)
    isexc=$(grep -l "isExclusive:" apecloud-addons/addons/$addon/templates/cmpd-*.yaml 2>/dev/null)
    cd=$(grep -l "componentDef:" apecloud-addons/addons/$addon/templates/paramsdef-*.yaml 2>/dev/null)
    echo "[$addon] rec=$rec isExc=$isexc pd-cd=$cd"
done
[mysql]   rec=  isExc=  pd-cd=mysql/templates/paramsdef-57.yaml mysql/templates/paramsdef-80.yaml
[mongodb] rec=  isExc=mongodb/templates/cmpd-config-server.yaml mongodb/templates/cmpd-mongodb-shard.yaml  pd-cd=
[mssql]   rec=  isExc=  pd-cd=
```

→ mysql 和 mongodb 也命中相同字段。**不是 OB chart 局部 bug**，是 `apecloud-addons/main` **整仓追新 API**。Step 3 排除 ① 局部 bug 假说。

### 中间结论

定盘 **②"整代代差"为主，③"未发布 alpha"渗一点**：apecloud-addons/main 整仓追 KB v1.2.0-alpha.0+ schema，KB v1.0.2 落后两个 minor。

## OB chart 内部时间线（漂移点）

```bash
$ cd ~/ApeCloud/apecloud-addons
$ git log --oneline --all -S "isExclusive: true" -- addons/oceanbase/templates/cmpd-repl-ocp.yaml
e831d89 chore: set primary leader role to isExclusive true (#1157)
$ git log --oneline --all -S 'componentDef: {{ include' -- addons/oceanbase/templates/paramsdef-param.yaml
72b30cd chore: adapt to new parameters API (#1191)
$ git show -s --format="%ci %s" 72b30cd
2026-03-31 13:54:29 +0800 chore: adapt to new parameters API (#1191)
```

| OB chart 提交 | 引入字段 / 动作 | 日期 |
|---|---|---|
| `863a67e` PR #1118 "reset pod svc and default backup image tag for compatibility" | **pre-skew 干净版**：cmpd 无 `reconfigure`/`isExclusive`；paramsdef 用旧 `reloadAction.shellTrigger` API；存在独立 `pcr-*.yaml`（ParamConfigRenderer） | **2026-01-29** |
| `e831d89` PR #1157 "set primary leader role to isExclusive true" | 引入 `roles[].isExclusive` | 2026-02 |
| `72b30cd` PR #1191 "chore: adapt to new parameters API" | 引入 `configs[].reconfigure` + `paramsdef.componentDef` + `templateName` + `fileFormatConfig`，**删除** `pcr-dist.yaml` / `pcr-repl.yaml` | **2026-03-31** |

`72b30cd` 是 PR #1191，commit message 直接写"adapt to new parameters API" — chart 作者**明知故犯**，跟着 KB main 的 #10100（2026-03-26）合并 5 天后做 chart 端 follow-up。Oracle case 第三段同模式。

→ `863a67e`（2026-01-29，3 个月前）是最后一个对 KB v1.0.2 友好的 OB chart commit。

## Step 4（反向 PR 进主 framework）— install-path schema validation 行为对照

OceanBase case 在跑 framework 时暴露了**主文档原 Step 1-3 没覆盖的一根轴**：

> KB addon-controller install 路径**宽松剥离未声明字段**；helm v4 install 走 typed patch CRD 校验，**严格拒绝**。同一份 chart 走两条路径行为不同，看到"controller 装成功"不等于"helm 装也会成功"，看到"helm 装失败"也不等于"chart 在 cluster 上不能跑"。

证据：在同一个 KB v1.0.2 集群（Machine B）上做实测：

```bash
$ kubectl get parametersdefinition mysql-8.0-pd -o json | python3 -c "
import json, sys
spec = json.load(sys.stdin)['spec']
for k in ('componentDef', 'templateName', 'fileFormatConfig'):
    print(f'{k}: {spec.get(k, \"<NOT PRESENT>\")}')"
componentDef: <NOT PRESENT>
templateName: <NOT PRESENT>
fileFormatConfig: <NOT PRESENT>
```

但源 chart `kubeblocks-addons/addons/mysql/templates/paramsdef-80.yaml` 真的写了：

```yaml
componentDef: {{ printf "^%s(?:-(?:orc|mgr))?-8\\.(?:0|4)-%s$" ... | quote }}
templateName: mysql-replication-config
fileFormatConfig: ...
```

→ KB 装 mysql 时 addon-controller 把这 3 个未声明字段**默认剥离**了；PD 状态 `Available`，跑得了普通 reload，但 chart 设计意图字段**根本不在集群里生效**。

**这意味着选项 C "kubectl apply -f <(helm template) 绕 helm typed patch" 会造 false-pass**：install 不报错 ✅，集群里 PD 状态 `Available` ✅，但是设计意图字段被默默吞掉，runtime 行为 undefined。smoke test 跑出 PASS 不代表 chart 工作正常，从 smoke test 目标看是反目的。

**3-tier install path taxonomy（D 路径实测后归纳）**：

| Tier | install 路径 | 未声明字段行为 | smoke test 适配性 |
|---|---|---|---|
| 1 | `helm v4 install` 走 typed patch | **strict reject** | 反映 chart 真相，blocker 直接暴露 |
| 2 | `kubectl apply --validate=Strict` server-side apply | **strict reject** | 与 helm 行为一致 |
| 3 | `kubectl apply --validate=ignore` / KB addon-controller install | **silent strip**（字段不入对象） | false-pass 风险：装成功但设计意图字段不存在 |

**建议给主文档加 Step 4**：

> **Step 4 — 跨 install 路径 schema validation 行为对照**：当 KB addon-controller 装成功但 helm v4 install 失败时，先看是不是同一个 schema 不匹配被两条路径区别对待。"controller 路径默剥离 / helm typed patch 拒绝" 跟 "API 字段未发布" 是不同维度的 confounder，不要混。3-tier taxonomy 见 [`addon-helm-vs-kb-controller-install-path-divergence-guide.md`](../../addon-helm-vs-kb-controller-install-path-divergence-guide.md)。

这与 Step 1-3（origin 检测）正交，本案例首次暴露。

## 升级路径选项（决议待确认）

| 选项 | 改动 | 代价 | 评估 |
|---|---|---|---|
| **A** 升集群到 KB v1.2.0-alpha.0+ | ops 重装 KB | 跑 v1.2 alpha API 全套（chart 设计意图） | ❌ **REJECTED**：KB v1.0.2 是当前 stable，v1.2.0 not stable，stable lock 优先 |
| **B** OB chart pin 到 `863a67e` | checkout 老 commit | 保 v1.0.2 不动 | 🟡 候选：测 3 个月前快照，需验关键 BP7 特性是否还在 |
| **C** `kubectl apply -f <(helm template)` 绕 typed patch | 安装命令换 | 保 v1.0.2 + 用 main chart | ❌ false-pass（已用 mysql PD 实测验证 lenient strip） |
| **D** = C + 严格 INFRA-ONLY 标签 | C 路径 + evidence 路径前缀 `D-INFRA-ONLY/` + `_path-marker.txt` 标 `chart-faithful=NO` | 保测试不 stand-down；只验测试 framework infra 接线 | ❌ **实测 0 PASS**：T1 ITSP at component reconcile build error `config/script template has no template specified: oceanbase-config`，44 次/2 分钟 deadlock。0 PRODUCT-LEVEL signal 产出，且 INFRA-LEVEL 也跑不到 |
| **E** 当前 chart HEAD downport 5 字段回 v1.0.2 schema | chart dev 工作（`configs[].reconfigure` 提到 `lifecycleActions.reconfigure` 单 hook / 去 `roles[].isExclusive` / paramsdef 用旧 `reloadAction.shellTrigger` API） | 1-2 天 chart dev，保留近期 fixes | 🟡 候选：B 失败后必做 |

A/C/D 已否决，剩 B/E 二选一待拍：B 起手快速验证，B 失败 / 关键特性丢失 → E 必做。

D 路径在拍 D 时设计预期"INFRA-ONLY 接线验证"，实测 0 PASS 直接说明：**lenient strip 路径下 chart 设计意图被默默吞掉的影响不只是 false-pass，而是 component reconcile 直接死循环、连 framework 接线都验不动**。这条事实强化了 C/D 的否决理由。

## 沉淀给后续 addon 的教训

1. **30 秒成本规则**：`git branch -a && git tag --list` + `grep -l 字段名 同仓其他 addon` = 排除 ① 局部 bug 假说的 30 秒前置动作。Oracle case 跨过这步走了 4 段反转。
2. **字段名相同 ≠ 路径相同 ≠ 同字段**：grep 命中要追字段绝对路径再下结论。这条与 Oracle case 第二段教训重合。
3. **chart annotation `kubeblocks-version: ">=1.0.0"` 是 unreliable signal**：apecloud-addons/main HEAD 这个 annotation 没改，但 chart 真实最低需要 v1.2.0-alpha.0。OB 和 Oracle 都是这模式。**不要相信 annotation，相信字段绝对路径追溯**。
4. **install path 不止一条**：addon-controller 路径宽松剥离、helm v4 typed patch 严格、`kubectl apply` 走 server-side apply 中等严格。同一个 chart 跨路径行为不同，可能造 false-pass。**跑 PASS 之前一定确认走的是哪条路径**，evidence pack 标 install-method。
5. **lenient strip 不是 fix，且 false-pass 不是唯一风险**：选项 C 看上去是 unblock 路径；D 路径 INFRA-ONLY 实测 0 PASS（component reconcile 死循环）说明字段被剥离后 controller 自己也跑不通。lenient strip 适用于"被剥离字段 runtime 不依赖"的简单 PD，不适用于完整 chart。
6. **smoke test 框架 + chart 是两个独立验证轴**：smoke test framework 设计的接线（lib/common.sh wait helpers / collect_evidence 落盘 / OpsRequest builder）不依赖 chart 字段是否被剥离。但**实测下 D 路径连 framework 接线都没机会验**，因为 component 起不来。framework 接线验证需要至少 chart 能让 component reconcile 走通，这条边界与原假设不同。
7. **stable 锁定优先**：升集群版本（路径 A）需要先评估 stable 状态，而不是默认"高版本=可用"。stable 锁定的项目里，chart 侧 downport 是更现实的路径。

## 决议 + rationale

A/C/D 已 REJECTED。B/E 待 chart-side 路径决议 owner 拍。

- **A REJECTED**: KB v1.0.2 是当前 stable，v1.2.0 not stable，stable lock 优先于 chart 设计意图最新 API
- **C REJECTED**: lenient strip 路径 chart 设计意图字段被默默吞掉，false-pass 风险
- **D REJECTED**: 实测 0 PASS（component reconcile build error），INFRA-LEVEL 接线也验不动
- **B vs E**: B 起手 30 分钟验证，已知装得上但是 3 个月前快照；E 1-2 天 chart dev 保留近期 fixes
- 决议 owner / 决议时间 / 决议 rationale / 后续 retest run-id：B/E 拍板后此段补完

## 来源

- evidence pack：`kubeblocks-tests/oceanbase/evidence/install-fail-001/`（install-error.txt + env.txt + schema-mismatch-summary.md + lenient-stripping-cross-check.md）
- D 路径 0 PASS evidence：`kubeblocks-tests/oceanbase/evidence/D-INFRA-ONLY/d-infra-001-20260501-012449/SUMMARY.md`
- Cross-engine pair：[`oracle-chart-vs-kb-schema-skew-multi-stage-case.md`](../oracle/oracle-chart-vs-kb-schema-skew-multi-stage-case.md)
- 主文档：[`addon-chart-vs-kb-schema-skew-diagnosis-guide.md`](../../addon-chart-vs-kb-schema-skew-diagnosis-guide.md)（待加 Step 4 by 反向 PR）
- Install-path divergence hub：[`addon-helm-vs-kb-controller-install-path-divergence-guide.md`](../../addon-helm-vs-kb-controller-install-path-divergence-guide.md)（待立项）
