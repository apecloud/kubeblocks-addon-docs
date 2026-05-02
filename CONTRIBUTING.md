# 贡献指南 — kubeblocks-addon-docs

This is the contributor's guide for `kubeblocks-addon-docs`. If you are a Claude Code / Cursor consumer reading this repo, see [`CLAUDE.md`](CLAUDE.md). If you are a human browsing the repo, see [`README.md`](README.md).

This document codifies the conventions that PR review enforces. Following them up-front avoids style-harmonize fixup PRs later.

## Quick start: 我要加什么内容？

| 我要加的内容 | 该放哪 |
|---|---|
| 一篇新的方法论（不绑定单一引擎，可复用判定流程 / framework / 设计原则） | `docs/addon-<topic>-guide.md` 单篇 |
| 一个引擎（valkey / mariadb / oracle 等）的具体闭环案例 | `docs/cases/<engine>/<descriptive-case-name>.md` |
| 跨引擎方法论的实证案例（多个引擎都触发同一类反模式） | `docs/cases/methodology/<descriptive-case-name>.md` |
| 现有方法论 doc 的增量（同一主题的新观察 / 反例 / 章节） | 直接迭代原 `docs/addon-<topic>-guide.md`，**不要另开平行文档** |

## 文件命名

- 主题方法论：`addon-<topic>-guide.md`，topic 用 `kebab-case`，主题名直接表达 doc 内容（例如 `addon-bounded-eventual-convergence-guide.md`，不是 `addon-helper-best-practices.md`）
- 引擎案例：`<engine-name>-<descriptive-symptom-or-fix>-case.md`，描述性命名让 grep 友好（例如 `cm4-bounded-window-helper-semantic-bug-case.md`）
- 不要把多个主题混在同一文件里。reconfigure / switchover / TLS / backup 各自一篇。

## 必备的 intro block（每篇 `docs/*.md` H1 后必须有）

```
> **Audience**: addon dev / test / TL
> **Status**: stable | draft | superseded
> **Applies to**: any KB addon | engine X
> **Applies to KB version**: 1.0.x / 1.1.x release tags | main (1.2 unreleased) | any
> **Affected by version skew**: <可选，已知跨版本断点>
```

各字段写作 hint：

- `Audience`：明确受众。如果只针对测试工程师，写 `addon test / TL`，不要写 `addon dev / test` 含混
- `Status`：stable（成熟方法论）/ draft（草稿，欢迎挑战）/ superseded（已被新文档替代，标 link 到接替者）
- `Applies to`：覆盖 addon scope。`any KB addon` 表示通用方法论；`engine X` 表示只对特定引擎
- `Applies to KB version` **mandatory**：精确到 release tag 带（e.g. `1.0.x / 1.1.x release tags`），不要泛泛 `1.0+`。runtime-layer-agnostic 方法论可以写 `any (methodology, version-agnostic)`，但要标注**为什么** version-agnostic（k3d 层 / shell 层 / 等）
- `Affected by version skew`：可选，但当 KB API 跨版本变化（reconfigure 接口、CD immutable 字段等）会影响 doc 适用性时显式列出

## 推荐的 readability 结构

每篇方法论 doc 建议（对照成熟范例 [`docs/addon-test-host-stress-and-pollution-accumulation-guide.md`](docs/addon-test-host-stress-and-pollution-accumulation-guide.md)）：

1. **H1 + intro block**（如上）
2. **白话 abstract section**（`## 先用白话理解这篇文档`）：3 个子段
   - "这篇文档解决什么问题"（用 lay 语言重述 root cause）
   - "读完你能做什么决策"（4-5 个具体场景）
   - "为什么独立成篇"（明确 doc 边界，与相邻 doc 区分）
3. **关键术语**：首次出现的 term 给单行 gloss（reactive overload / cascade / bounded eventually 等）
4. **主体内容**：按章节展开，章节 H2 用语义化命名（不是 "## 1." 编号）
5. **案例附录** / **相关材料**：链接到 `docs/cases/<engine>/` 案例 + 相邻方法论 doc

具体写作建议：

- 段落短，关键句 **加粗**
- 避免 150+ 字符复合句
- 流程类用 numbered list（不要纯 prose 描述步骤）
- 案例附录：先放 summary 表（结论），再放细节 timeline
- 复杂主题用 abstract-before-detail（参考 `addon-componentdefinition-upgrade-guide.md` 的 "施工蓝图 vs 现场" 比喻）

## Cross-doc references 必须 clickable

不要写 `` `docs/addon-X-guide.md` ``（backtick-only 在 GitHub UI 不可点）。统一用 markdown link：

| 引用路径 | Pattern |
|---|---|
| Methodology → methodology | `` [`docs/addon-X-guide.md`](addon-X-guide.md) `` |
| Methodology → case | `` [`docs/cases/<engine>/X.md`](cases/<engine>/X.md) `` |
| Case → methodology | `` [`addon-X-guide.md`](../../addon-X-guide.md) `` |
| Case 内 sibling 引用 | `` [`X.md`](../X.md) `` 或 `` [`X.md`](X.md) `` 视相对位置 |
| 引用尚未存在的 TBD doc | 保留 backtick `` `addon-X-guide.md` `` + 显式 `(TBD)` 标记 —— 不放 broken link |

## 长 doc 的 manual anchor 模式

对于多 appendix / 多 section 的长 doc，跨 section 引用建议用 manual HTML anchor（紧贴在 H2 之前），让中文 H2 也能稳定 deep-link：

```markdown
<a name="appendix-b-cell-3-valkey-solo"></a>
## 附录 B: cell-3 valkey single-load reference
```

引用：`详见 [appendix B](#appendix-b-cell-3-valkey-solo)`

命名约定：`<section-type>-<identifier>-<engine>-<mode>`，全小写，kebab-case。

reference 范例：`docs/addon-test-host-stress-and-pollution-accumulation-guide.md` 的 appendix B / D。

## 一篇一主题（hard rule）

**不要在同一文件里混写**多个独立主题。reconfigure / switchover / TLS / backup / upgrade / 排障 等各自独立成篇。

判定标准（不是行数）：**这一章能否独立成篇？**

- 能 → 拆出去，新建独立 doc
- 不能（只是大主题里的一个步骤）→ 保持在一起

例：host-stress doc 的 chapter 1-7 全都是 "host cascade 现象的同一调查的递进"，单一主题，所以 400+ 行长 doc OK；但如果某 chapter 拆出来仍是完整的 cleanup methodology，那就该独立。

## Acceptance bars

每篇 doc 由两条独立标准评判（PR review 时会检查）：

| Axis | 标准 |
|---|---|
| Utility | 3 个月后另一个 addon 团队接手能不能直接用上 |
| Readability | 段落短、术语首次出现解释、流程类用编号列表、案例附录给摘要先放结论 |

## PR 工作流

### Pure structural change（intro block / cross-ref / format / heading）

- Branch: `style-harmonize/<short-desc>`
- Commit message tag: `[style-harmonize]`
- 内容：纯结构性改动，不动 substantive 方法论 / 技术声明
- Review：单 reviewer ack 即可 fast-merge（任何 owner 或 @Allen）

### Substantive content change（新 doc / 已有 doc 的方法论增量 / 引擎案例 fill）

- Branch: 个人前缀，如 `<your-name>/<topic>` 或 `<engine>/<topic>`
- Commit message tag: `docs(<topic>)` 或 `docs(<engine>)`
- 内容：方法论本身、技术判断、案例 evidence
- Review：相关 owner 必须 review；style 由 @Allen post-merge harmonize（如有 drift 由 Allen 起 fixup PR）

### 文件操作

- 新增 `docs/*.md` 必须同时更新 `docs/SKILL-INDEX.md`（场景导航 + 详细描述列表 + 跨场景方法论 cluster 视情况）
- 新增 `docs/cases/<engine>/*.md` 必须同时更新 `docs/SKILL-INDEX.md` 的 case 段
- 删除 / 重命名 doc 必须 grep 全仓 cross-references 并修复

## Owner 约定

| 引擎 / 域 | TL（dev + 主笔） |
|---|---|
| Valkey + 跨引擎 methodology | @Alice |
| MariaDB | @Helen |
| Oracle | @James |
| Style harmonize / 跨 doc consistency | @Allen |

substantive 内容改动：先在 channel `#kubeblocks-addon-docs` 跟 owner 对齐再发 PR，避免方向偏离。

## 反 anti-pattern

- 不要在 docs/ 根放引擎特异性 case（必须进 `docs/cases/<engine>/`）
- 不要在 backtick 引用一个不存在的文件路径（要么补 doc，要么改成 `(TBD)` 描述符）
- 不要在 intro block 里写 `KubeBlocks version: any` 当跑 `helm install` / 用 KB API 字段时实际依赖具体 1.x 版本
- 不要把 reconfigure / switchover / TLS / backup 等独立主题混在一篇 doc 里
- 不要发"修一下试试看"的 substantive PR；先在 channel 同步意图

## 相关文档

- [`README.md`](README.md) — 仓库 orientation（中文 narrative，给人看）
- [`CLAUDE.md`](CLAUDE.md) — LLM consumer routing + glossary（给 Claude Code / Cursor 看）
- [`docs/SKILL-INDEX.md`](docs/SKILL-INDEX.md) — 场景化导航 + 全文档详细列表
