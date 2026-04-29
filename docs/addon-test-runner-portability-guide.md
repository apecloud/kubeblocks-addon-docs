# Addon 测试 Runner 在 macOS bash 3.2 + `set -euo pipefail` 下的兼容指南

> **Audience**: addon dev / test
> **Status**: stable
> **Applies to**: any KB addon test runner（bash-based）
> **Applies to KB version**: any (runner / shell 层，与 KB 版本无关)

本文面向 Addon 测试 runner 的开发者。runner 在 Linux CI（bash 4+）跑得过、本地 Mac 跑就报 `unbound variable` / parse error，根因通常是 **macOS 自带 bash 永远是 3.2.57**（GPL v3 license 原因 Apple 不升级），而 `set -euo pipefail` 把 bash 3.2 与 4+ 之间的几处语义差异全部暴露。本文给 7 个最常见的坑 + 自检清单，让 runner 在两边都跑得稳。

## 先用白话理解这篇文档

最容易让人误判的一件事：

- runner 在 CI 上跑了几百次都没事，本地一开 `set -u` 立刻 `unbound variable`
- 第一反应往往以为是 runner 写错或环境变量没设
- 真正原因多半是 **bash 3.2 没把空数组 / 同一 local 语句里的前后变量 / 未声明 env-var default 当 set**

下面 7 条都是「Linux 跑得过、Mac 一跑就炸」的具体型态。**macOS 上调试 runner 第一件事：用 `/bin/bash`（强制 3.2）跑一遍 `bash -n`，撞到的坑会比晚上 CI 红灯前撞少很多**。

## 背景

macOS 自带 `/bin/bash` 永远是 **3.2.57**。Linux CI 通常 bash 4+ 或 5+。两边语义有差异，runner 一旦写得不严谨，本地能跑出 Linux 不会出现的 unbound variable / parse error，污染测试结论。

下面的坑是**配 `set -euo pipefail` 后才会暴露**，所以严格脚本反而更容易踩到。

## 坑 1：空数组在 `${arr[@]}` 下报 unbound variable

```bash
set -euo pipefail
local args=()
some_command "${args[@]}"   # bash 3.2 报 args: unbound variable
```

bash 4.4+ 才把空数组下 `${arr[@]}` 视为 set。

**修复 A（最小改动）**：用 `${arr[@]+...}` 占位：

```bash
some_command ${args[@]+"${args[@]}"}
```

**修复 B（更彻底，推荐）**：不用单独的「可选参数数组」，直接 append 到主参数数组：

```bash
local args=( --base-flag "$x" )
if [ -n "$OPT" ]; then
  args+=( --opt-flag "$OPT" )
fi
some_command "${args[@]}"   # args 永远非空
```

## 坑 2：env var default 时机不对，CLI 参数被 silent override

模式：

```bash
# 文件顶层
ADDONS_REPO="${ADDONS_REPO:-/default/path}"
ADDONS_CLUSTER_REPO="${ADDONS_CLUSTER_REPO:-$ADDONS_REPO}"

# 之后 parse CLI
parse_args ... # 这里才把 --addons-repo /tmp/foo 写入 ADDONS_REPO
```

`ADDONS_CLUSTER_REPO` 在文件加载时就用了**当时的** `ADDONS_REPO`（默认值），后续 `--addons-repo /tmp/foo` 只更新 `ADDONS_REPO`，**没传播**到 `ADDONS_CLUSTER_REPO`。结果：addon 用 `/tmp/foo`、cluster 还在用 `/default/path`，**测试源混源，结论被污染**。

**修复**：依赖型 default 必须在 parse 之后回填：

```bash
ADDONS_REPO="${ADDONS_REPO:-/default/path}"
ADDONS_CLUSTER_REPO="${ADDONS_CLUSTER_REPO:-}"

parse_args "$@"

ADDONS_REPO="${ADDONS_REPO:-/default/path}"
ADDONS_CLUSTER_REPO="${ADDONS_CLUSTER_REPO:-$ADDONS_REPO}"
```

或更显式：parse 完后单独有个 `resolve_defaults()` 函数集中回填依赖型 default。

## 坑 3：local-only path 假设

`/Users/...` 路径写死到 runner 默认值在本机能跑，CI 上跑不了。**所有路径默认值要么走环境变量，要么走当前 PWD 解析，不写死人名级别的绝对路径**。

```bash
# 不好
ADDONS_REPO="${ADDONS_REPO:-/Users/wei/ApeCloud/apecloud-addons}"

# 好
ADDONS_REPO="${ADDONS_REPO:-${REPO_ROOT}/../apecloud-addons}"
```

## 坑 4：`mapfile` / `readarray` / `${var,,}` / `${var^^}` 不存在

bash 3.2 没有：

- `mapfile` / `readarray`
- 大小写转换 `${var,,}` / `${var^^}`
- `${arr[@]:offset:length}` 在某些组合下也行为不同

**替代**：

- `mapfile` → `while IFS= read -r line; do arr+=("$line"); done < <(cmd)`
- `${var,,}` → `echo "$var" | tr A-Z a-z`

## 坑 5：同一 `local` 语句里前后变量互相引用

```bash
set -u
local cluster="$1" name="${3:-${cluster}-restart-$$}"
# bash 3.2 报：cluster: unbound variable
# 因为 bash 3.2 在 expand 第二个赋值时还没 register 第一个 local 的 cluster
```

bash 4+ 把 `local a=x b=$a` 看作顺序赋值（a 先 register，b 用得上）；bash 3.2 把整条 `local` 当原子，里面所有 `${name}` expand 时还没声明完。

**修复**：分行 local：

```bash
local cluster="$1"
local name="${3:-${cluster}-restart-$$}"
```

或者：`name` 不带 default，调用方自己传：

```bash
local cluster="$1"
local name="$3"
[ -z "$name" ] && name="${cluster}-restart-$$"
```

## 坑 6：双引号里 `\$name` + `set -u` 撞 unbound

```bash
set -u
sql="SELECT * FROM v\$parameter"   # 想得到字面 v$parameter
echo "$sql"   # bash 报：parameter: unbound variable
```

`\$` 在双引号里：bash 把 `\$` 解释成「字面 `$` 后接 `parameter` 展开」，所以最终撞 `$parameter` 这个 shell 变量，set -u 直接 crash。

**修复**：用 single-quote 字符串（最安全，shell 完全不解释）：

```bash
sql='SELECT * FROM v$parameter'
```

适用场景：写 SQL（Oracle `v$parameter` / PostgreSQL `$1` 占位符 / MySQL `$$`）、AWK 脚本（`$1`, `$NF`）、jq filter（`.foo`）等。

这个坑跟 bash 版本无关，bash 4/5 同样 crash。但容易在「Linux 跑得过、Mac 也跑得过（如果没开 set -u）」的项目里被忽视，开 `set -u` 后第一次跑就炸。

## 坑 7：`set -e` + `local x=$(cmd)` 不会捕获失败

```bash
set -e
foo() {
  local x=$(cmd_that_fails)   # set -e 不触发，x 拿到空字符串
  echo "$x"                    # 后续逻辑用空 x，silent bug
}
```

`local`/`declare` 这种 builtin 永远返回 0，掩盖右侧命令的退出码。

**修复**：分两步：

```bash
local x
x=$(cmd_that_fails)
```

## 自检清单（在 macOS 上跑 runner 前）

```bash
# 1. 语法检查
bash -n run-tests.sh
for f in lib/*.sh tests/*.sh; do bash -n "$f"; done

# 2. 用 bash 3.2 强制跑一遍（即使你装了 brew bash）
/bin/bash -n run-tests.sh

# 3. 启 set -u 空跑一遍 dry path（不真创建 cluster）
DRY_RUN=1 /bin/bash -euo pipefail run-tests.sh -t smoke

# 4. 确认所有 default 路径在 CLI override 后被真正用上：runner 启动时 echo 一行 "EFFECTIVE PATHS: ..."
```

第 4 条是反污染的关键：runner 应该在每次跑开头 print 一份 effective config，方便人眼核对 CLI flag 真的生效。

## 与其他文档的关系

- [`addon-test-environment-gate-hygiene-guide.md`](addon-test-environment-gate-hygiene-guide.md) 第 5 项「测试 runner / 锚点 staged 层」要求 runner 静态绿（`bash -n` + `git diff --check`）；本文给那一步具体能撞到的坑。

## 案例附录

### 2026-04-29 oracle test runner：连环踩三坑

- **坑 1（空数组）**：`lib/cluster.sh` 里 `storage_args=()` 后 `"${storage_args[@]}"` 报 unbound。改成 append 到 value_args 后彻底消失。
- **坑 2（env-default 时机）**：`ADDONS_CLUSTER_REPO="${ADDONS_CLUSTER_REPO:-$ADDONS_REPO}"` 在 parse `--addons-repo` 之前就 fallback 了，导致 cluster chart 用了 main 工作区。修复后在 parse 之后回填。
- **坑 6（`v\$parameter`）**：T07 reconfigure 测试 SQL 里 `set -u` + `\$parameter` 字面量被展开。改成 single-quote 后通过。
- 后续加了 `--skip-addon-install` 和 effective config echo。
