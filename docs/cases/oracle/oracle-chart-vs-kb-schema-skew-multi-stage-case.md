# Oracle Chart × KB Schema 三段反转：从「整代代差」到「跟 KB main」再到「同仓 release-1.0 才是答案」

> **Audience**: addon dev / test
> **Status**: stable
> **Applies to**: Oracle addon on KubeBlocks（方法论可复用，参见 [`addon-chart-vs-kb-schema-skew-diagnosis-guide.md`](../../addon-chart-vs-kb-schema-skew-diagnosis-guide.md)）
> **Applies to KB version**: 验证于 KB 1.0.2 / 1.0.3-beta.5 / 1.1.0-beta.5；根因涉及 KB main 上未发布的 PR #10100 / #10109
> **Affected by version skew**: reconfigure / parameters API 在 KB main 重构（ParamConfigRenderer 拆进 ParametersDefinition、reconfigure 路由到 ComponentParameter），预计随 KB 1.2 发布；reconfigure 接口的具体跨版本演进与写法迁移见 `addon-reconfigure-version-skew-guide.md`（TBD）

本文是 [`addon-chart-vs-kb-schema-skew-diagnosis-guide.md`](../../addon-chart-vs-kb-schema-skew-diagnosis-guide.md) 的工程现场补充。记录 2026-04-29 Oracle smoke 起点的三段反转，每段都被下一段推翻，最后才落到正确路径。每段都标了「当时被忽略的关键证据」与「应该走的最短路径」，便于后续 addon 第一次撞同类问题时跳过这些弯路。

## 起点

- chart：`apecloud-addons/addons/oracle`（chart `1.0.0-alpha.0`，appVersion `12.2.0.1-ee`，license `LicenseRef-KubeBlocks-Enterprise`）
- KB controller：`apecloud/kubeblocks:1.0.2`
- chart annotation：`addon.kubeblocks.io/kubeblocks-version: ">=1.0.0"`（不实）
- 失败：`helm install oracle ... -n kb-system` 报多字段被拒：
  - `ComponentDefinition.spec.configs[0].reconfigure`
  - `ComponentDefinition.spec.roles[0].isExclusive`
  - `ParametersDefinition.spec.templateName`
  - `ParametersDefinition.spec.componentDef`

## 第一段：误判「整代 chart × KB 1.0.2 不兼容」

KB 1.0.2 v1 storage 实有字段：

- `cmpd.spec.configs[]`：`defaultMode/externalManaged/name/namespace/restartOnFileChange/template/volumeName`
- `cmpd.spec.roles[]`：`name/participatesInQuorum/updatePriority`

cross-chart dry-run 用 `kubeblocks-addons/addons/mongodb`（chart `1.1.0-alpha.0`，同样声明 `>=1.0.0`），server-side dry-run 同样报 `unknown field "spec.roles[0].isExclusive"`。

**当时（错误的）结论**：整代 chart 与 KB 1.0.2 不兼容，要切 KB 控制器到 chart 实际假定的 1.1.x 系列。

**当时被忽略的关键证据**：cross-chart 失败只能证明这些字段在「KB 当前 schema 不存在/不在该路径」，**不能证明只要升 KB 就好** —— 完全可能「对照 chart 跟主 chart 犯同一类错」。

## 第二段：切 KB 1.0.3-beta.5 / 1.1.0-beta.5 后 chart 仍然失败

顺着第一段的判断切了两轮 KB 版本，每轮都做 oracle dry-run 验证：

- 1.0.2 → 1.0.3-beta.5：dry-run 仍报相同字段被拒
- 1.0.3-beta.5 → 1.1.0-beta.5：dry-run 仍报相同字段被拒

回头按字段绝对路径追：

1. **chart 字段路径错位**：chart 写 `cmpd.spec.configs[0].reconfigure`，但 KB 任何 released 版本里 `reconfigure` 字段实际都挂在 `cmpd.spec.lifecycleActions.reconfigure`。
2. **chart Kind 用错**：chart `paramdef-12.yaml / paramdef-23.yaml` 声明 `kind: ParametersDefinition`，但 spec 字段 `componentDef / templateName / fileFormatConfig` 在 KB released schema 里属于独立的 `ParamConfigRenderer` Kind。

**当时（错误的）结论**：chart 字段路径错位 + Kind 用错，是 chart bug，准备上报让 chart owner 修。

**当时被忽略的关键证据**：grep 看到字段名就乐观结论；没有按字段绝对路径校验「该字段是否真的出现在 chart 期望的那个绝对路径」。

## 第三段（真相）：chart 跟 KB main，目标 API 还没发布

去 chart 仓 `git log` 找 owner 时事情翻了一层：

```
git log -- addons/oracle/templates/cmpd.yaml
```

- leon-ape **#1191** (2026-03-31) `chore: adapt to new parameters API`：删除 `pcr-12.yaml / pcr-23.yaml`（ParamConfigRenderer），把字段合并进 `paramdef-*.yaml`，cmpd 加 `configs[].reconfigure` 块。
- wangyelei **#1249** (2026-04-28) `chore: keep reconfigure action's target pod selector consistent with KB 1.0 behavior`：在新 schema 上继续打补丁。

也就是说 **chart 作者是「明知故犯」**，不是写错。

去 KubeBlocks 上游仓 `git log --grep=parameters` 找对应的 API 迁移 PR：

- `4ec01470c` **#10100** (2026-03-26) `chore(parameters): deprecate ParamConfigRenderer by moving bindings into ParametersDefinition`
- `441769521` **#10109** (2026-04-07) `chore(parameters): route reconfigure through ComponentParameter`

时序上 chart #1191 紧跟 KB #10100 之后 5 天 ——**chart 作者在跟着 main 分支同步迁移**。

决定性一击：

```bash
git -C ~/ApeCloud/kubeblocks tag --contains 4ec01470c
git -C ~/ApeCloud/kubeblocks tag --contains 441769521
# 两条都返回空
```

→ 这两个 PR **只在 `main`，没有进任何 released tag**（最新 release 是 `v1.1.0-beta.5`，也不含这两个 PR）。

**真实根因**：chart 期望的 KB API 还在 main 没有发布；任何 released KB tag 都装不上这个 chart。

## 第四段（真正的答案）：同仓 `release-1.0` 分支才是 chart × KB 1.1.0-beta.5 的兼容选项

判定「chart 跟 main」之前，应该先扫 `git branch -r`。事实是 chart 仓 `apecloud-addons` 并行维护：

- `main` 跟 KB main API（含 #10100 #10109 新 parameters API）
- `release-1.0` 保留 pre-#1191 schema：`pcr-*.yaml` 独立 ParamConfigRenderer、`paramdef.spec.reloadAction.shellTrigger` 老路径、cmpd 无 `configs[].reconfigure`

这个分支可以直接装到任何 1.0.x / 1.1.0-beta KB controller 上：

```bash
mkdir -p /tmp/oracle-release-1.0
git -C ~/ApeCloud/apecloud-addons archive origin/release-1.0 addons/oracle \
  | tar -x -C /tmp/oracle-release-1.0
helm install oracle /tmp/oracle-release-1.0/addons/oracle -n kb-system
```

→ STATUS deployed，全套 CD/CmpD/CmpV/PD/PCR/ActionSet/BPT Available，gate-T01 PASS。

**Run 6 之后 Oracle smoke 11/11 全部 PASS 就是用的这条路径。**

## 沉淀给后续 addon 的教训

1. **`git branch -r` 是 30 秒成本**。任何 chart × KB 报错前先做这一步，不要直接进 cross-chart dry-run。
2. **字段名相同 ≠ 路径相同 ≠ 同字段**。grep 命中要追字段绝对路径再下结论。
3. **chart owner 通常有「内部能跑的 KB 版本」**。跟 chart owner 私聊 5 分钟可能省你 5 小时。
4. **切 KB 控制器是大动作**。即使被授权自主推进，也应先在干净 cluster 上验 dry-run 通过再切现网。本次切了两轮 KB 版本都没解，因为根因不在 KB。
5. **artifact 留底**。`/tmp/kb-switch-artifact/` 存了完整 pre-/post- snapshot；判断未 settle 前不要清理。
