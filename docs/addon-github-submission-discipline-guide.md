# Addon GitHub Submission Discipline — Trailer 与并发 push 事故响应

> **Audience**: addon dev / test / TL
> **Status**: draft
> **Applies to**: any KB addon
> **Applies to KB version**: any (workflow methodology, version-agnostic — runtime-layer-agnostic, 仅依赖 git + GitHub + Slock 多 agent 协作 pattern)
> **Affected by version skew**: 不适用（git workflow / GitHub submission 规则跨 KB 版本通用）

属于：方法论主题文档（不绑定单一引擎）。本篇覆盖两件 GitHub-facing submission 事项：(1) AI provenance trailer 不外漏的硬规则与兜底命令链；(2) 多 agent 并发推同一 PR branch 的 cascade 事故响应 playbook。两章共享同一根问题 — agent 自动化协作 + GitHub 公开仓库的边界纪律。

## 先用白话理解这篇文档

**这篇文档解决什么问题**

公司里多 agent 协作开发已经常态化。Slock agent 在做 GitHub 提交时经常踩两类坑：

1. **AI 自动注入的 trailer 漏到 GitHub 公开 repo** — agent runtime（Claude Code / Codex 等）在调用 `git commit` 时按 message-template 默认追加一行 `Co-Authored-By: <AI-bot> <noreply@…>` 之类的 trailer。在 Slock 内部产物里无所谓，一旦 push 到公开 GitHub repo 就是 AI provenance 信息泄漏。这件事公司有硬规则不允许出现。
2. **多个 agent 并发 push 同一 PR 分支** — 跨 line 协作（例如几个 addon line 同时往一个 docs PR 填章节）时，谁都用 `git push --force-with-lease`，但 lease 通过≠没有 race，往往会出现"你按掉一个 trailer / 我也按掉一个 trailer / 中间某个 commit 被丢了" 的级联事故。

**读完你能做什么决策**

- 写 commit message 之前、push 之前、reword 之后，分别用什么命令做 trailer 自检（§1.4）
- 多 agent 并发要往同一 PR 分支推 commit 时，要不要 `--force-with-lease`、push 之前要做哪些 fetch + 双向 log 比对（§2.2）
- 自己（或别人）的 commit 被发现带了 trailer，按 dirty 在不在 origin / 是不是 merged 决定走 amend / rebase / forced-push / contact-maintainer 哪条路（§1.5）
- 同 PR 分支多 agent 同时打 cascade fix 时，谁拿主笔权、其他人怎么 stand down + 提供 cross-validate（§2.4）
- 事后做 trailer + cascade 的 incident response 时，run summary / commit message / channel timeline 怎么写以便 reviewer 能独立审（§2.3 + §2.4）
- 下一次同型 cascade 触发时，如何不重证 doctrine、只补 forensic 实证（§2.6 Forecast 框架）；遇到全新失败模式族（不在 doctrine 已覆盖 family 内）时升级 doctrine list 的 sediment criteria（§2.6 + §2.5 Doctrine E first surface 实例）

**为什么独立成篇**

trailer 是 GitHub submission 维度的硬规则；cascade 是多 agent 协作维度的事故响应；两者都不属于"具体某个 addon engine 的开发规范"，也不属于"测试覆盖度 / coverage strategy" 类话题。把它们放进 cross-addon 文档（例如各 line 的 `notes/cross-addon-rules.md` 或 `notes/conventions.md`）里只放 short pointer + back-link 到本篇 — 完整的兜底命令链 / forensic timeline / 自检表归到这里，便于 reviewer / 其它 line 一次找齐。

cross-line skills doc 的同侪是 `addon-evidence-discipline-guide.md`：那一篇覆盖 evidence 强度匹配（对结论的内向 discipline），本篇覆盖 GitHub submission（对外向产出的 discipline），都是"产出物的 hygiene"，但维度不同 — 不并入也不重复。

## 关键术语

- **AI provenance trailer**：agent runtime（如 Claude Code / Codex）在生成 commit message 时按 message template 自动追加的 attribution 行，典型形如 `Co-Authored-By: <AI bot name> <noreply@…>` 或 `🤖 Generated with <product>`。
- **Slock-internal artifact** vs **GitHub-visible content**：前者指 Slock 平台内部的 chat / DM / agent workspace 文件，AI provenance 在这里 OK；后者指任何 push 到公开 GitHub repo（含 PR title / body / commit body / review comment / issue comment）后会被外部读到的内容，AI provenance 在这里被硬规则禁止。
- **`--force-with-lease`**：`git push --force-with-lease` 的 lease 锚点是"我上次 fetch 时看到的 remote tip"，**不是**"远端没有任何并发 commit"。这是 §2 大量 race 的 root cause。
- **Cascade fix**：A push 出了问题（trailer 泄漏 / commit 被丢） → B 修一次（amend / rebase / push） → 可能引发 C 修一次（B 修的过程把第三方 D 的 commit 丢了 / B 自己也带 trailer） → … 这一连串纠错构成 cascade。本篇 §2 给响应 playbook。
- **Single-owner-execute**：cascade 修复阶段，整个 chain rebuild 由一个 agent owner 执行，其他 agent 提供 cross-validate input + stand down，不并发动 branch。

---

## 1. Trailer hygiene — 不让 AI 注入的 trailer 漏到 GitHub 可见处

### 1.1 硬规则（公司侧）

公司 2026-05-01 锁定的 GitHub Submission Hard Rule：**never expose Claude / Codex / AI / Anthropic / Co-Authored-By trailer in any GitHub-visible content**。

GitHub-visible 的边界包括（不仅限）：

- PR title / body
- commit message subject + body（含 trailer block）
- review comment / issue comment
- commit description in `git log` 后期被 reader / mirror / archive 抓走的拷贝

只要文本会通过 `git push` / GitHub API 进入公开仓库的 visible surface，就在硬规则覆盖范围。

### 1.2 为什么 trailer 会被自动注入（root cause）

trailer 不是 git 默认行为产生的，是 **agent runtime 在 message-template 阶段注入的**。

当 agent 调用 `git commit -m "<subject>"` 时，runtime 按内置模板把一段尾部追加文本拼到 message 末端（典型形如 `Co-Authored-By: <AI bot> <noreply@…>` 或 `🤖 Generated with <product>`），然后才把完整 message 交给 git。所以：

- `git config commit.cleanup strip` **拦不住** — 这个选项只清"行尾空白 + `#` 开头的注释行"，不动 trailer block（trailer block 由 `git interpret-trailers` 解析，跟 cleanup 不是同一层）
- pre-commit hook **拦不住** — 模板注入发生在 hook 之前；hook 时 message 已经带 trailer
- prepare-commit-msg hook **可以** 拦截，但实践上配置成本高、跨 agent runtime 不一致

可靠的做法只有两类：

- **commit-time 把 message 改干净**（heredoc 写完整 message / amend reword 删 trailer）
- **post-commit 自检**（push 前用 grep 复核 trailer 不在 message 里）

### 1.3 commit-time 怎么写干净的 message

**Option A — heredoc 一次写完**（preferred for 已经知道 message 全文的场景）

```bash
git commit -m "$(cat <<'MSG'
docs(<topic>): <subject>

<body>
MSG
)"
```

heredoc 提交时 runtime 不会再 template-inject — 因为 `-m` 接收的是已成型的 final message。

**Option B — 用默认模板提交后立即 amend 删 trailer**（per-commit 一次性）

```bash
git commit -m "docs(<topic>): <subject>"
git commit --amend -m "$(git log -1 --pretty=%B \
  | sed '/^Co-Authored-By: /d; /^🤖 /d; /Generated with /d')"
```

注意：`sed` 模式按当前 runtime 注入的行尾 token 调整。新增 runtime 时同步扩展 sed 模式。

**Option C — 在 final message 是 multi-line body 时，保留 amend 路径**

写一次 commit、再 amend 一次，比直接 heredoc 出错率低（heredoc 容易引号/转义出错）；多写一行 amend 不是负担。

### 1.4 push-time 自检（必检）

**单 commit 自检**：

```bash
git log -1 --pretty=%B | grep -E 'Co-Authored-By|🤖|Anthropic|Claude|Codex'
# 命令必须输出空 — 任何输出都视为 dirty, 不能 push
```

**branch 全 commit 自检**（推 branch 前必跑）：

```bash
for h in $(git log --format=%H origin/main..HEAD); do
  if git show $h --no-patch --format='%B' | grep -qE 'Co-Authored-By|🤖|Anthropic|Claude|Codex'; then
    echo "DIRTY: $h"
  fi
done
# 必须无 "DIRTY" 输出
```

为什么 branch 全检不可省：rebase / amend 只清"被 reword 的那条 commit"。如果你 reword 了 tip 但中间还有别条带 trailer 的 commit，它们不会被传染性清掉。下一节 §2.2 会展开。

**为什么不放在 pre-push hook**：可以放，但跨 agent / 跨机器的 hook 配置不可靠。把检查放在 push command 前的 explicit step 让 reader 一眼能看到，比 hook 静默拦截更适合多 agent 协作。

**post-amend content-delta verify**（trailer-hygiene amend 也必跑 — body grep 通过 ≠ 内容真正落地，详见 §2.2 Doctrine E）：

```bash
# amend 后, 与 amend 前 sha 比对内容 delta — 必须非零
git diff <pre-amend-sha>..HEAD -- <file> | wc -l
git show HEAD --stat -- <file>
# 期望 +N -M 输出, 空则 = no-op amend, 必须重做
```

ping reviewer 报 PASS 时必附上述命令的输出作为 evidence quote，不依赖 word-of-mouth（Doctrine E evidence-post obligation）。

### 1.5 已经泄漏到 origin 之后怎么处理（按状态分支）

| 状态 | 处理路径 |
|---|---|
| 单 PR / branch 仍在 draft / 未 merge / 仅 author 一人 push | author 自己 `git rebase -i HEAD~N` reword + `git push --force-with-lease` |
| 多 agent 并发 push 同一 branch | 走 §2 cascade recovery playbook（不能简单 force-push，会丢别人的 commit） |
| 已 merge 到 main / release 分支 | 联系有 force-push 权限的 maintainer 决定（amend merged commit 会改写 history，影响所有下游 fork / mirror，**禁止单方面操作**） |

draft / 单 author 场景的可复刻命令（Option A — explicit reword）：

```bash
git fetch origin <branch>
git checkout <branch>
git rebase -i HEAD~N    # 在编辑器把每条 dirty commit 标 'reword'
# 编辑每条 message 时手动删 trailer block
git push --force-with-lease
```

reword 完后 **必须** 用 §1.4 的 branch 全检命令复核。本篇 §2.3 incident 1 的失误案例之一就是"以为 rebase 链尾被 reword 就传染清干净"，结果中间 commit 仍 dirty。

### 1.6 §1 自检清单

每次提交并 push 前过一遍：

1. message 是用 heredoc 写完整、还是 commit + amend reword？两条路径任选其一，不要"`-m '<single line>'` 然后直接 push"
2. push 前用 §1.4 的 branch 全检命令跑一次 — 输出必须空
3. 如果是 `git rebase --continue` / `git commit --amend` 之后的 push，全检命令对 `origin/<branch>..HEAD` 范围里**每一条 commit** 跑一次，不只看 tip
4. 如果不确定能不能补救，先停手 + 在 channel 同步现状，比单方面 force-push 给协作者带来 cascade 强得多

---

## 2. Concurrent-push cascade recovery playbook — 多 agent 并发推同 PR 的事故响应

### 2.1 失败模式

跨 line / 跨 agent 协作 PR（典型场景：一个 docs PR 里 4-6 个 addon line 各填一个章节）下，常见失败模式：

1. agent A 提交带 trailer 的 commit → push 上去
2. reviewer 抓到 trailer → 喊 A 修
3. A 用 `git rebase -i` reword + `git push --force-with-lease`
4. **同一时间** agent B 也在往同 branch push（可能是另一个章节的 fill，也可能是另一条 trailer fix）
5. A 的 force-with-lease 通过 — 因为 A 的 lease 锚定是"A 上次 fetch 时的 remote tip"，A 没看到 B 在中间 push 的 commit
6. 结果 B 的 commit 被 A 的 force push 顶掉
7. B 发现自己的 commit 没了 → 重发 / cherry-pick / 又一次 force push
8. 这中间又可能再次踩 §1 trailer 漏检 — 触发第二轮 cascade

`addon-evidence-discipline-guide.md` 第 5 节讲的"harness 修复时机"其实就是同型问题的内向版本：**修一个 dirty 状态前，先看 dirty 是不是已经 promote / 被别人依赖；不是 → 自由修；是 → 走 reject + cross-validate + rerun 流程**。本节是它的外向版本：**force-push 前要先看 dirty 是不是只有自己在 branch 上**。

### 2.2 Push-flow doctrines（四条：A/B/C/E）

本节列 push-flow 维度的 4 条 doctrine（commit / push / rebase 时执行的 self-check 与协作 ritual）；doc-writing 维度的 **Doctrine D**（forensic 写作时 named agent self-review own attribution rows）属 cross-cutting requirement，见 §5 第 9 条。三方 v3 final lock 5-doctrine list (A/B/C/D/E) 跨这两个 layer：A/B/C/E = git workflow tooling 维度，D = documentation tooling 维度。

#### Doctrine A — `--force-with-lease` 语义陷阱

`--force-with-lease` lease 的判定是 **"我上次 fetch 看到的 remote tip == 现在远端 tip"**，不是"远端没有任何并发 commit"。

实操含义：

- 你 fetch 完 → 立刻 rebase → 立刻 force-with-lease push，期间如果别人 push 进来一条 commit，且你 fetch 时它还不存在，那么你的 lease 仍然通过（远端 tip 在你 fetch 之后才被那条 commit 推前），但你的 push 把它顶掉了
- 多 agent 并发场景里"我看到的 lease 锚点"和"远端实时 tip"之间的 gap 是 race window
- "我用了 `--force-with-lease` 就安全"是错觉

**Mitigation — push 前必跑双向 log 比对**：

```bash
git fetch origin
git log --oneline origin/<branch>..HEAD   # 我新增 / 改动的 commit
git log --oneline HEAD..origin/<branch>   # 远端有、我本地缺的 commit ← 关键
```

第二条命令必须为空。**非空意味着远端有你不知道的 commit**：你的 force push 会把它丢掉。先 rebase / cherry-pick 把它合进来再 push。

#### Doctrine B — Per-commit body grep 自检

rebase 链上的 reword 只清被 reword 的那一条 commit。中间 / 上游别条 commit 的 message body 不会传染性变干净。

**Mitigation — push 前对 `origin/main..HEAD` 范围里**每一条 commit** 单独 grep**：

```bash
for h in $(git log --format=%H origin/main..HEAD); do
  if git show $h --no-patch --format='%B' | grep -qE 'Co-Authored-By|🤖|Anthropic|Claude|Codex'; then
    echo "DIRTY: $h"
  fi
done
```

为什么不能 only check tip：tip clean 不蕴含 chain clean。最差情况是一条中间 commit 带 trailer 但 reviewer 只 grep tip → 漏抓 → 直到下次 audit 才暴露。

Doctrine B 自检通过后，ping reviewer 报 PASS 时**必附 grep 命令输出 quote**（"`grep -E '...'  | wc -l = 0`"）作为 evidence — 不只 word-of-mouth。这条 evidence-post obligation 适用所有 doctrine（不限 trailer 维度），因此 sediment 在 §5 第 11 条 cross-cutting 而非单 doctrine sub-clause（参见 §5）。

#### Doctrine C — 多 agent 并发 cascade fix 优先 single-owner-execute

cascade fix 阶段（trailer 抓出来 → 多人都在准备改），不要并发 push 同一 branch。原因：每一次 force push 都重置 lease 锚点，并发 push 会层层互相顶掉。

**Mitigation — 一人 own 整个 chain rebuild，其他 agent 提供 cross-validate input + stand down**：

- owner（通常是被 reviewer 直接 ping 的那个 agent，或主动认领的）拿主笔权 — 执行整个 chain 的 rebase / cherry-pick / push
- 其他相关 agent 在 channel 显式 "stand down — 不 push branch，wait owner"，并提供 cross-validate（指出某条 commit 的 hash → 章节 mapping、提醒某条 commit 别漏、自己的 grep 自检结果）
- owner push 完成后，其他 agent **再各自做一遍** §2.2 Doctrine B 的 per-commit grep + Doctrine A 的双向 log 比对，独立 confirm
- 整个 cascade fix 在 channel 留下显式 timeline（catch / stand-down 列表 / owner 行动 / cross-validate 结果），便于事后审计

stand-down 阶段需要 explicit 信号 — "我看到你在改，我不动 branch" 比"以为对方知道我也不动"安全。

#### Doctrine E — content-delta verify obligation

amend / rebase / force-push 之后，self-check 命令链**必须验证实际变更交付维度**，不能只查相邻维度。

实操含义：

- amend 改的是 commit body（trailer hygiene） → self-check 跑 body grep
- amend 改的是 file content → self-check 跑 tree-level diff（`git diff <pre-sha> <new-sha> -- <file>` 必须非空 / `git show <new-sha> --stat -- <file>` 必须显示 `+N -M`）
- amend 改的是 author / date 元数据 → self-check 跑 `git show --format=%ae,%ai`

trailer-clean / race-clean 自检通过 **不蕴含**内容真正落地。**body 是 metadata, tree 是 ground truth。** 两类检查不可互替。

**典型 false-positive PASS**：bash compound 命令（`git add ... && git commit --amend ...`）的 `&&` 链在 parse 阶段失败时整个不执行，但 working tree 仍带 modified 状态；之后用 `-F file` 形式再 amend，由于 staging area 仍为空，`git commit --amend` 只改写 commit metadata（timestamp / message），tree hash 不变，amend 命令"成功"但 file content delta = 0。trailer-clean / race-clean 都过，但内容根本没落地。actual case 见 §2.5。

**Mitigation — 每次 amend / rebase 后必跑 content-delta verify**：

```bash
# 1. 与 amend 前 commit 比对内容 delta（必须非零）
git diff <pre-amend-sha>..<post-amend-sha> -- <changed-files> | wc -l
# 必须 != 0；= 0 即 no-op amend, 内容未真正落地

# 2. 与 push 目标 ref 比对（force-with-lease 前必跑）
git diff origin/<branch>..HEAD -- <changed-files> | wc -l
# 必须 != 0

# 3. 看 stat 行级 delta
git show <post-amend-sha> --stat -- <changed-files>
# 必须显示 +N -M, 不能空

# 4. 关键改动 spot-check
git show <post-amend-sha>:<file> | sed -n '<line-N>p'
# 期望文本必须出现, 否则即使 stat 非零也可能改错文件 / 改错位置
```

amend owner 报 PASS 给 reviewer 时，content-delta 维度的 evidence quote（`wc -l` 行数 + `git show --stat` 输出 + spot-check 文本 hit）必附，不只 word-of-mouth — 这条 evidence-post obligation 跨 Doctrine A/B/C/D/E 都适用，sediment 在 §5 第 11 条 cross-cutting 要求（不单挂 Doctrine E）。

### 2.3 Incident 1 forensic — PR #54 8a5127c chain（2026-05-04）

完整 timeline + recovery sequence。后续遇到同型 incident，可对照本节快速 sediment。

#### 背景

PR #54 是一个 cross-line docs PR（IDC image registry / mirror / sideload guide），6 条 commit 来自 5 个 line：

| commit (final hash) | 来自 line | 内容 |
|---|---|---|
| `b6c7a42` | docs lead | guide skeleton |
| `830212e` | KBE | KBE row §5 fill |
| `9d38ec5` | docs lead | §1.1 vcluster k3s image override |
| `06ab18f` | SQL Server | SQL Server row §5 fill |
| `c3ceeff` | Oracle | Oracle row §5 fill |
| `8a5127c` | OceanBase | idc4 row §6 partial fill + §9 status |

#### 事件链

| 时间 | 事件 |
|---|---|
| pre-incident | 多 line 各自往 PR #54 push 自己的 fill commit。KBE 行原 commit `70635bb`（带 trailer）+ SQL Server 行原 commit `292391c`（带 trailer），均未自检 |
| catch 1 | reviewer 在 KBE commit `70635bb` body 抓到 `Co-Authored-By: <AI bot>` trailer → ping KBE owner |
| recovery 1 | KBE owner 以 owner 身份 rebase reword `70635bb` → 新 hash `830212e` → `force-with-lease` 通过（KBE 当时已知 remote tip 仍是 `292391c` / 后续 commit 还没推上来） |
| catch 2 | SQL Server owner self-catch 自己的 `292391c` 也带 trailer，触发第二次 cascade fix |
| race | 在 SQL Server 准备 rebase 期间，Oracle owner 已 push `83fb045`（Oracle row §5 fill, trailer self-checked clean） |
| HOLD | Oracle owner self-detect — actively monitoring channel 上 SQL Server 给的 recovery plan 时，发现原计划用 `cdbdf1d` 做 reset target 会丢掉 own commit `83fb045`（race condition），channel 显式 HOLD（race-loser / commit-drop at-risk 当事人 self-surface，不是第三方 oversight） |
| recovery 2 | SQL Server owner 重画 recovery plan：fetch origin 重看 remote tip → reword 自己的 commit 得 `06ab18f` → cherry-pick 当前 remote 的 Oracle commit + OceanBase commit 上来 → 一次性 force-with-lease push |
| 二次 race | 但 OceanBase 的 idc4 commit `475dfef` 在第一次 KBE rebase 时其实已经被 force-push 顶掉了（KBE 当时 fetch 时 `475dfef` 还没存在，lease 锚点 ≠ `475dfef`，所以 lease 通过），SQL Server 此次 recovery 把它 cherry-pick 重新带回 → 新 hash `8a5127c` |
| cross-validate | Oracle owner 独立跑 per-commit grep verify `c3ceeff`（自己 commit re-hash 后）body clean；OceanBase owner 独立 verify `8a5127c` body clean；KBE owner verify `830212e` clean → 6/6 trailer-clean confirm |

#### 复刻 recovery sequence（同型事故时可直接对照）

> 假设你是 cascade owner（被 reviewer ping 的 agent，或主动认领的）。

```bash
# Step 0 —— 同步状态
git fetch origin
git log --oneline origin/<branch>..origin/main   # 看当前 PR branch 比 main 新增的所有 commit
git log --oneline HEAD..origin/<branch>           # 看本地缺哪些（必须 merge / cherry-pick 进来）
git log --oneline origin/<branch>..HEAD           # 看本地多哪些（要保留）

# Step 1 —— pre-recovery 锁全 chain trailer 状态
for h in $(git log --format=%H origin/main..origin/<branch>); do
  if git show $h --no-patch --format='%B' | grep -qE 'Co-Authored-By|🤖|Anthropic|Claude|Codex'; then
    echo "DIRTY: $h"
  fi
done
# 输出列表 = 需要 reword 的 commit 集合

# Step 2 —— 拿到 commit → 章节 mapping（don't trust 别人口头报的 hash → row, 自己 git show 复核）
for h in $(git log --format=%H origin/main..origin/<branch>); do
  echo "=== $h ==="
  git show --no-patch --format='%s' $h
done

# Step 3 —— 在 channel 锁 single-owner-execute
#   - 列出 dirty hash 集合
#   - 自己 declare 主笔权
#   - 喊其他 agent stand down（"不要 push <branch> until <expected ETA>"）

# Step 4 —— rebuild chain
git checkout origin/main -B recovery-tmp
git cherry-pick -x <clean_hash_1> <clean_hash_2> ...   # 干净的先放
# 对每个 dirty hash:
git cherry-pick -x <dirty_hash>
git commit --amend -m "$(git log -1 --pretty=%B | sed '/^Co-Authored-By: /d; /^🤖 /d; /Generated with /d')"
# 重复直到所有 commit 上去

# Step 5 —— pre-push 全 chain trailer 自检
for h in $(git log --format=%H origin/main..HEAD); do
  if git show $h --no-patch --format='%B' | grep -qE 'Co-Authored-By|🤖|Anthropic|Claude|Codex'; then
    echo "STILL DIRTY: $h"
  fi
done
# 必须空。非空就回 Step 4 继续 amend。

# Step 6 —— pre-push race 复核
git fetch origin
git log --oneline HEAD..origin/<branch>   # 必须为空; 非空说明又有人 push 进来了, 重 cherry-pick
git log --oneline origin/<branch>..HEAD   # 要 push 的全部 commit

# Step 7 —— push
git push origin recovery-tmp:<branch> --force-with-lease

# Step 8 —— 回 channel 同步
#   - "owner push complete, tip = <new_hash>"
#   - 邀请其他 agent 各自跑 §2.2 Doctrine A + B 命令独立 confirm
```

#### 失误教训（事故 doctrine sediment）

1. **lease pass ≠ no race**（→ Doctrine A）：第一次 KBE rebase 没 fetch 看 OceanBase 那条 commit 是否已 push，lease 通过把它顶掉了。Mitigation 是 push 前 `git log HEAD..origin/<branch>` 必检
2. **Reword 一条 ≠ chain 全清**（→ Doctrine B）：如果 SQL Server owner 没在 self-catch 阶段做 per-commit grep，第二次 cascade 也会漏抓
3. **Chain rebuild 优先 single owner**（→ Doctrine C）：第二次 cascade 由 SQL Server owner 一人执行 + Oracle / OceanBase / KBE 在 channel stand down + 各自做 cross-validate confirm，避免再次 race。**Stand-down ≠ passive wait**（Doctrine C sub-clause）：本次 HOLD 是 Oracle owner（race-loser，own commit `83fb045` 即将被 SQL Server recovery plan 顶掉的当事人）actively monitoring channel 上的 recovery plan 时，发现 reset target 不含 own commit 后 self-surface 触发的，**不是第三方 oversight**。doctrinal 含义：race-loser / commit-drop at-risk agent 必须 actively read 当前 cascade owner 的 recovery plan，确认 own commit 在 plan 覆盖内；这条 monitoring obligation 是 cross-validate 的一部分（race-loser / drop-at-risk 视角），不是 stand-down 的反义
4. **不要在没 git log 复核的情况下转述 hash → row mapping**：被丢掉的 OceanBase commit `475dfef` 一开始被错认为"idc4 row rebased into 830212e"，是因为 owner 之间口头转述时没 `git show` 复核。事后 sediment 加进 recovery sequence Step 2
5. **Cascade 责任落点 — "丢失 commit 的 owner 优先自己回收"**：当 cascade fix 把第三方 commit 顶掉时，被丢者优先自己 cherry-pick 回收（理由：被丢者熟悉自己 commit 内容、跟自己的 line 沟通更顺；让 cascade owner 代为 cherry-pick 别人的 commit 增加 blast radius）

### 2.4 单 owner 执行期间，其他 agent 的协作模板

cascade fix 期间，非 owner agent 的标准动作：

```text
1. 在 channel 显式 "stand down — <branch> 上我不 push 任何 commit, wait <owner> recovery 完成"
2. 在 channel 提供 cross-validate input:
   - 我自己的 commit hash（如果在 chain 里）+ 自检结果（grep clean / dirty）
   - 我观察到的 chain 中其他 commit 的 hash → 章节 mapping
   - 任何与 owner 计划冲突的 race condition observation（HOLD signal）
3. 等 owner push 完成后, 独立跑 Doctrine A + B 自检命令, 在 channel 报 verify 结果
4. 若 owner 计划遗漏自己的 commit, 等 owner 完成 + 自己 cherry-pick 回收（Doctrine C 教训 5）
```

### 2.5 Incident A2 forensic — amend reported PASS but tree unchanged（2026-05-04，Doctrine E first surface）

本 incident 是写本篇 doc 自身 PR 的 review 收尾时触发的 — recursive irony 自带：写 trailer hygiene + cascade recovery doc 的作者，自己第一次 amend 时踩了 self-check 漏维度坑。本节 sediment 这次失败本身（不是 forecast），作为 Doctrine E 的 first-occurrence forensic。

#### 触发情境

PR #55 收到 review verdict（Allen 2 nit + James 1 content-accuracy nit + Allen +1 §5 第 9 条）→ 作者 Ben 在 working tree 把 5 条 nit 全 inline 修好，跑 4-step self-check 全 PASS，force-with-lease push 到 PR head。Reviewer James 拿到 ping 后做 byte-level cross-validate，硬数据：

```
$ git show d64630d --format='%T'
791977cbad9b65daf37356bac030a5872d3b9d13

$ git show 91a74ea --format='%T'
791977cbad9b65daf37356bac030a5872d3b9d13   ← 同 tree

$ diff <(git show d64630d:docs/addon-github-submission-discipline-guide.md) \
       <(git show 91a74ea:docs/addon-github-submission-discipline-guide.md) | wc -l
0   ← 零行 diff
```

commit SHA 换了 (`d64630d` → `91a74ea`)，**tree hash 完全没变**，文件 byte-level diff = 0，所有 5 条 nit 一条都没真正落地。

#### 失败模式

第一次 amend 命令是 bash compound：

```bash
cd <repo> && git add <file> && git commit --amend -m "$(cat <<'MSG'
... message body ...
MSG
)" && git log --oneline -2
```

但 `$(cat <<'MSG' ...)` 嵌套里有不可逃逸的引号 (`docs's own §1.4` 的 `'s`)，bash parse 阶段直接报 `unexpected EOF while looking for matching ` `'`，**整个 compound chain 都没进入执行** — `git add` 也没跑。working tree 保持 modified 但 staging area 仍为空。

第二次 amend 改用 `-F /tmp/file.txt` 形式：

```bash
git commit --amend -F /tmp/amend-msg.txt
```

命令本身成功执行，但 `git add` 没在这次 chain 里跑（被假设由前次 chain 完成）。staging area 实际仍为空，所以 `git commit --amend` 只改写 commit metadata（timestamp、message），tree 不变。amend 命令"成功"输出 `1 file changed, 415 insertions(+)`（这是 vs parent `f6ed1f9` 的累积 diff，**不是 vs `d64630d` 的 amend delta**），作者错把这视为成功证据。

#### 漏检维度

作者跑的 4-step self-check 全在 trailer hygiene + race + push-state 维度：

1. branch-wide `for h in ... grep` 自检（trailer 维度）— 输出空 ✓
2. 双向 `git log --oneline` log compare（race 维度）— `HEAD..origin/<br>` = pre-amend sha lease anchor / `origin/<br>..HEAD` = post-amend sha ✓
3. post-amend tip body grep（trailer 维度）— "CLEAN: tip commit body has no trailer match" ✓
4. PR head state=OPEN（push 状态维度）— `91a74ea38193fb696e55f1ef44ba412baf4e80a5` ✓

**content-delta 维度未查**：tree hash diff / `git diff <pre> <new> -- <file>` / `git show <new> --stat -- <file>` / 关键改动 spot-check 全没跑。本次 amend 的 actual change axis 是文件内容，self-check 命令链却只覆盖 trailer + race + push state，跨 axis 没 coverage。

#### Doctrine E first surface — sediment criteria 第一例 actual application

本 incident 暴露 §2.2 doctrine list 第 5 条 axis：**self-check 必须 verify deliverable 在 actual change axis，不只查 adjacent axis**。trailer-clean 自检 ≠ 内容真正落地，body 是 metadata，tree 是 ground truth。

按 §2.6 "No-re-evidence sediment criteria" 判定：本次失败模式（self-check 漏维度）**不在** incident A1 已覆盖的 force-with-lease race / multi-agent push 顺序冲突 / trailer 漂移 family 内 — 是 single-agent self-check dimensional coverage 维度的新失败模式族。per criteria **正确触发 doctrine 升级**（Doctrine E 加入 doctrine list），不是仅 §4 增 row。这是 §2.6 framework 自身在第一次 real-world test 时通过的 recursive validation。

#### Mitigation

§2.2 Doctrine E 的 4-step content-delta verify + evidence-post obligation 全部内化为永久 ritual（§1.4 + §2.2 双锚）。下次 amend / rebase 后必跑：

```bash
git diff <pre-amend-sha>..HEAD -- <file> | wc -l   # != 0
git show HEAD --stat -- <file>                      # +N -M
git show HEAD:<file> | sed -n '<key-line>p'         # 期望文本 hit
```

ping reviewer 报 PASS 时，这三行命令的输出 quote **必带**，不只 word-of-mouth。

#### Cross-anchor — incident A2 同时 = Doctrine E first-surface + Doctrine D label 漂移 first-surface

A2 forensic 在本 PR review 收尾时 surface 了第二个 self-check 漏维度信号：post-amend `009daec` (本节本身的来源版本) 表面交付"5-doctrine A/B/C/D/E clean"，但 review-iteration self-detect 显示 §2.2 heading 仍是 stale "三条 doctrine" + 全文无 `#### Doctrine D` formal label section（§5 第 9 条 prose 形式存在但没标 label），cross-cite "A/B/C/D/E 任一" 中 D 对应的 anchor 悬空。这是 Doctrine D 自身（doc-writing-side named-attribution self-review）catch 的反例 — framing claim 与 structure delivery 不一致，属 attribution drift family 在 doctrine 元层的 expression（不是 forensic timeline row 层，而是 doctrine list 自身 label 完整性层）。James 3-层 cross-verify 在 evidence post 之后通过 layer-aware grep（`#### Doctrine` heading 数 vs `Doctrine A/B/C/D/E` cross-cite 数）surface 这条 nit；review-iteration 即作 Doctrine D label 漂移 first-occurrence forensic 锚（详见 §4.1 A3 row + Allen msg=e80c0d8f β refined curator pick）。本 sub-amend 把 §2.2 heading 改 "Push-flow doctrines（四条：A/B/C/E）" + §5 第 9 条加 `**Doctrine D —**` formal prefix 即闭环（Doctrine list 数不变，Doctrine A/B/C/E 在 §2.2 git tooling 层 / Doctrine D 在 §5 doc-writing 层 cross-cutting 容器，per Allen msg=e80c0d8f layer-split first-principles framing）。

### 2.6 Forecast incident F1 — concurrent vcluster syncer cross-validation push

> 本节是事前 forecast（F1，见 §4.2），不是已发生的 forensic。目的是把 §2.3 incident A1 沉淀的 doctrine 跟下一次同型 cascade 的预期触发条件对齐，触发时直接 cite，不重 derive。

Phase 2 PoC vcluster bootstrap 解锁后（各 IDC `apecloud-registry-cred` Secret 部署完成），预期触发同 PR 多 agent concurrent commit 第三个 actual incident（拟新增 §4.1 row，ID A3）：SQL Server / Oracle / OceanBase 三 line 在 vcluster syncer dockerconfigjson cross-validation 配置上（统一 Secret + image pull verification pod manifest）收口同一文件区域。三 line 各 own engine-specific image inventory，但 syncer 配置共一处，按 incident A1 同 pattern，大概率三 agent 在 30-60 min 窗口内 concurrent push 各自 row，触发 force-with-lease race / trailer 漂移 / commit drop 任一组合。

#### Reuse path — 不重 derive

Trigger 后本章节 incident A1 forensic + Doctrine A/B/C/D/E 直接 cite，不重 derive trigger conditions / expected cascade pattern / mitigation。具体 reuse：

- **(a)** push 前 ritual（双向 `git log --oneline` 比对，§2.2 Doctrine A）三 line 各自 self-execute
- **(b)** trailer self-check 每 commit 前置 grep 自检（§2.2 Doctrine B + §1.4）
- **(c)** content-delta verify 每 amend / rebase 后 `git show <sha> --stat` + `git diff <pre>..<post> | wc -l` 必非零（§2.2 Doctrine E + §1.4 / §2.2 双锚）
- **(d)** 若 cascade incident 真实发生，single-owner-execute recovery（§2.2 Doctrine C + §2.3 复刻 sequence）— 一人 own chain rebuild，其他 agent stand-by 提供 cross-validate input + 含 race-loser monitoring obligation sub-clause（§2.3 教训 3），不并发动 branch
- **(e)** recovery 完成后向 §4.1 actual incident table 追加 row（A3），从 §4.2 标记 F1 为 `superseded by A3` + 保留作为 forecast accuracy datapoint（forecast 真不真也是 doctrine evolution 的 evidence），**不重写 Doctrine list**

#### No-re-evidence sediment criteria

若 incident A3 cascade pattern 落在 incident A1 + A2 已覆盖的失败模式族（force-with-lease race / multi-agent push 顺序冲突 / trailer 漂移 / self-check 漏维度 任一组合），仅 §4.1 增 row，doctrine list 不动。

若出现 incident A1 + A2 都未覆盖的新失败模式（如 PR description merge-conflict / GitHub API rate-limit / 跨 fork cherry-pick / signed-commit verification fail 等），才升级 §2.2 doctrine list 加新条 + 标 trigger date + cite 该新 incident forensic 作为 first-occurrence reference。

Doctrine list 严守 first-principles 一阶，不重复同形 incident 引发的 second-order doctrine 漂移。incident A2 (PR #55 no-op amend) 触发 Doctrine E 加入 doctrine list 即是本 criteria **第一例 actual application** — content-delta verify 缺失是 incident A1 force-with-lease race / trailer 漂移 axis 之外的新失败模式族（self-check dimensional coverage），per criteria 正确触发 doctrine 升级而非仅 incident table append。

### 2.7 §2 自检清单

每次给同一 PR 分支推 commit 前过一遍：

1. 我现在是不是同 branch 唯一 active push 者？如果不是 — 在 channel 锁 single-owner-execute 还是先 stand down 等别人？
2. 我 fetch 后到 push 之间，有没有可能远端被别人 push 了 commit？跑双向 log 比对（Doctrine A）
3. 我要 reword 的 chain 上其他 commit 是不是 clean？跑全 chain per-commit grep（Doctrine B）
4. 如果触发了 cascade，我有没有 own 这次 chain rebuild？还是该 stand down + cross-validate？（Doctrine C）
5. amend / rebase 后跑了 content-delta verify 吗？`git diff <pre>..HEAD -- <file> | wc -l` 必非零 + `git show HEAD --stat -- <file>` 必有 `+N -M`（Doctrine E）
6. ping reviewer 报 PASS 时附了 cross-validate 命令输出 quote 吗？（Doctrine E evidence-post obligation）
7. push 完后，timeline / verify 结果有没有发回 channel 让其他 agent 知道？

---

## 3. 与相邻 doc 的关系

| Doc | 关系 |
|---|---|
| [`addon-evidence-discipline-guide.md`](addon-evidence-discipline-guide.md) | 内向 evidence discipline；本篇是外向 submission discipline。两篇都讲"产出物 hygiene"，维度不同。本篇 §2 incident forensic 在 evidence-discipline 第 5 节"harness 修复时机 dirty → clean 三段式"的方法论框架下属于 Phase 3 invalidate / 撤回 / re-promote 的 git workflow 实例化 |
| `notes/cross-addon-rules.md`（Slock 各 line agent 自留 cross-addon 规则文档） | "concurrent-PR-push-discipline" 章节 = 本篇 §2 三 doctrine 的 short methodology pointer + 反链本篇 §2 完整 forensic / playbook，避免 doctrine 散点 |
| `notes/conventions.md`（Slock 各 line agent 自留通用约定文档） | "GitHub Submission Hard Rule" 章节 = 本篇 §1.1 硬规则的 short pointer + 反链本篇 §1 完整命令链 |

cross-ref 拓扑（curator placement 锁定）：

```
notes/cross-addon-rules.md ──(short methodology pointer)──┐
notes/conventions.md ──(short rule pointer)───────────────┤
addon-evidence-discipline-guide.md 反模式表 ──(case row)───┤
                                                          ▼
                  本篇 addon-github-submission-discipline-guide.md
                          (full playbook + forensic)
```

每个 anchor 维护短入口 + 反链本篇，本篇 reverse-cite 三个 anchor location（本节）。procedural detail / 命令链 / forensic timeline 只在本篇维护，避免 drift。

## 4. 案例附录索引

ID schema：`A` = actual（forensic, evidence-backed） / `F` = forecast（placeholder, predictive）。两 axis 不混编 — actual incidents 按发生顺序累积，forecast incidents 独立编号；F1 触发后变 actual，新 ID = A3，原 F1 row 标 `superseded by A3` + 保留为 forecast accuracy datapoint（forecast 真不真也是 doctrine evolution 的 evidence）。

### 4.1 Actual incidents（forensic, evidence-backed）

| ID | timestamp | trigger | dimension vs A1 | recovery agent | 在哪一节 |
|---|---|---|---|---|---|
| A1 | 2026-05-04 ~18:30-18:45 | PR #54 8a5127c chain — multi-agent concurrent push, force-with-lease race + trailer 漂移 + commit drop | first occurrence | SQL Server line owner（cascade owner） | §2.3 |
| A2 | 2026-05-04 ~19:55 | PR #55 amend `91a74ea` no-op — single-agent self-check 漏 content-delta 维度，bash compound 链 parse 失败 + staging area 空 + amend 只改 metadata | new dimension（vs A1 multi-agent race）— self-check dimensional coverage | doc author（self-detected fail → reviewer cross-validate via tree-hash byte diff） | §2.5 |
| A3 | 2026-05-04 ~20:19 | PR #55 amend `009daec` framing claim 与 structure delivery 不一致 — `### 2.2 三条 doctrine` heading 与 body 4 个 `#### Doctrine` 不同步；`Doctrine D` 在 cross-cite 引用但全文无 `#### Doctrine D` formal label section | new dimension（vs A2 content-delta 漏检）— self-check label/structure consistency 漏检 | doc author（James 3-层 cross-verify catch + Ben grep self-confirm + Allen β refined 拍板）| §2.5 末段 cross-anchor + §4.1 |

A3 row 备注：**Doctrine D first-surface forensic**（同 A2 之于 Doctrine E first-surface 的双重位置）— Doctrine D 已在 v3 lock 内（5-doctrine list A/B/C/D/E），A3 是 D 的 implementation drift forensic 而非新 doctrine 触发器，per §2.6 sediment criteria 正确 application：仅 §4.1 增 row，doctrine list 数不变。同 PR 内 A2 + A3 连续 2 次 recursive self-application 反例 = §5 第 12 条 priority 升 "first-class authoring obligation" 的触发 datapoint。本 row 自身即是 §4.3 sediment 处理流程的**第一例 actual application**（同 family → 仅 §4.1 增 row + 不升级 doctrine list）。

### 4.2 Forecast incidents（placeholder, predictive）

| ID | trigger condition | expected dimension | placement |
|---|---|---|---|
| F1 | Phase 2 PoC vcluster syncer 3-line cross-validate concurrent push（SQL Server / Oracle / OceanBase 在 syncer dockerconfigjson 同区域 fill） | same family as A1（multi-agent race） | §2.6 |

### 4.3 sediment 处理流程

后续 incident 触发时：

- **同型 cascade**（落在 A1/A2 已覆盖失败模式族） — 优先按 §2.6 "No-re-evidence sediment criteria" 走：仅在 §4.1 增 row + 简要 timeline，**不重证 §2.2 doctrine** — 五条 doctrine 已 sediment，新 incident 主要补 forensic 实证
- **新失败模式族** — 落在 A1/A2 都未覆盖的 axis（如 PR description merge-conflict / GitHub API rate-limit / signed-commit verification fail），按 §2.6 升级 doctrine list 加新条 + 标 trigger date + cite 该新 incident forensic 作为 first-occurrence reference
- **forecast → actual** — 当 F* 真实触发时，新建 A* row（按 actual 序号） + F* row 标 `superseded by A*` 保留

## 5. 给后续 addon 工程师的固化要求

1. **每条 push 前** — branch 全 commit body grep 自检（§1.4 命令）；输出空才 push
2. **每条 push 前** — 双向 `git log` 比对（§2.2 Doctrine A 命令）；远端 → 本地差集必须空
3. **多 agent 协作 PR 上发现 trailer / cascade 时** — 在 channel 锁 single-owner-execute（§2.2 Doctrine C），不要并发动 branch
4. **被丢 commit 的 owner** — 优先自己 cherry-pick 回收（§2.3 教训 5），不甩给 cascade owner
5. **任何 cascade fix 完成后** — 全员独立跑 §2.6 自检清单 → 在 channel 同步 verify 结果，留 timeline 便于事后审
6. **commit / amend / rebase 时** — 用 heredoc 写 final message（§1.3 Option A）或 commit + amend reword（§1.3 Option B）；不要"`-m '<one line>'` 然后直接 push"
7. **已 merge 到 main 的 trailer 泄漏** — 不要单方面 force-push，联系 maintainer（§1.5）
8. **本篇 doctrine 引用** — 跨 line short pointer 用 `notes/cross-addon-rules.md` / `notes/conventions.md` 反链；procedural detail 一律在本篇维护，不复制
9. **Doctrine D — doc-writing-side named-attribution self-review obligation**：incident forensic 写作时，每条 timeline row 由 named-attribution 当事人 self-review，避免叙事方便引发的 attribution drift。这是 [`addon-evidence-discipline-guide.md`](addon-evidence-discipline-guide.md) 反模式表 "motivated narrative inflation" 在内向 doc 写作里的应用：写自家 forensic 时，"community oversight catches race" 比 "at-risk party self-detect" 听起来更 organic，但前者是 narrative 美化，后者是 actual lesson。"协作 work 必有 active obligation, 不限 designated executor" 在 action 模式（cascade fix 期间 stand-down agents 的 monitoring，§2.3 教训 3 sub-clause）和 documentation 模式（forensic 写作期间 named agents 的 self-review，本条）的对偶 expression。**本条 = §2.2 doctrine list 中 Doctrine D（doc-writing axis），区别于 §2.2 的 push-flow axis A/B/C/E**
10. **每次 amend / rebase 后** — 跑 content-delta verify（§2.2 Doctrine E）：`git diff <pre-sha>..HEAD -- <file> | wc -l` 必非零 + `git show HEAD --stat -- <file>` 必有 `+N -M` + 关键改动 spot-check。trailer-clean / race-clean 自检通过 ≠ 内容真正落地，body 是 metadata，tree 是 ground truth
11. **ping reviewer / 协作者报 self-check PASS 时** — 必附 cross-validate command 输出 quote 作为 evidence（**cross-cutting，applies to Doctrine A/B/C/D/E 任一**，非单 doctrine sub-clause）。具体形式：command 文本 + 输出 N 行 / `+N -M` / `0 命中` 等具体数值。例：
    - "Doctrine A 通过：`git log HEAD..origin/<br> --oneline | wc -l = 0`"
    - "Doctrine B 通过：`grep -E 'Co-Authored-By\|🤖\|Anthropic\|Claude\|Codex' <file> = 0 命中`"
    - "Doctrine E 通过：`git diff <pre>..<new> -- <file> | wc -l = 47`，`git show <new> --stat -- <file>` = `+38 -9`"
    - 不附 evidence quote 的 PASS report 视为 word-of-mouth，reviewer 默认走完整独立 cross-verify。Evidence-post 把 trust-transfer burden 从 reviewer 转回 amend owner（owner 自检过 → owner 把自检证据交付出来 → reviewer 单点 spot-check 即可，不必盲信亦不必盲查）
12. **doc 写作流程必须 first-class apply 本篇 hygiene** — 写 trailer-hygiene + cascade-recovery 这类 process doc 时，作者本人的 commit / amend / push 流程必须按本篇要求做 self-check + content-delta verify + evidence-post，**doc 自验**，且这是 first-class authoring obligation 不是次要要求。**incident A2（§2.5）= Doctrine E first-surface（content-delta verify 漏检）+ incident A3（§2.5 末段 cross-anchor + §4.1）= Doctrine D first-surface（label/structure consistency 漏检）连续 2 次 recursive self-application 反例**显示 writing rules about hygiene 自身在 writing 时踩 hygiene 漏检 frequency 比预期高（同 PR 内 ~30 min 间隔）→ 本条 priority 升 first-class authoring obligation。recursive self-application 把 motivated narrative inflation 反模式（§5 第 9 条 = Doctrine D）覆盖范围从"写 forensic 行内 attribution"扩到"写 doc 全流程的 self-check + content-delta + label/structure 完整性"
