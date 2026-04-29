# 案例：roleProbe 脚本 `#!/bin/bash` shebang 在仅有 busybox sh 的 kbagent 容器里直接 exec 失败

属于：[`addon-bounded-eventual-convergence-guide.md`](../../addon-bounded-eventual-convergence-guide.md) 案例（脚本调用面，非状态判定面，但同属"环境前提与脚本契约不匹配"）。

## 背景

- 任务序列：`#379` (helper bug fix) → `#380` (T7 row-count route_api) → `#381` (T7 VScale OpsRequest 300s timeout) → `#382` (live-diagnostic) → `#383` 部署 alpha.10 后验证。
- 现象：alpha.9 时代，T7 VScale 后 pod-1 重建，KB controller 一直收不到 role label，OpsRequest 300s timeout。pod-1 DB 实际健康（Slave_IO/SQL=Yes/Yes、lag=0、read_only=1），数据一致；但 kbagent 日志连续 7 分钟每 5s 报 `code=-1, output="", message="exit code: 1: failed"`。

## 根因

`addons/mariadb/scripts/replication-roleprobe.sh` 第一行是 `#!/bin/bash`。当 kbagent 直接 exec 脚本时，内核读 shebang，尝试 fork+exec `/bin/bash`，但 **kbagent sidecar 镜像只装了 busybox `/bin/sh`，没有 `/bin/bash`**，因此 exec 立刻返回 OCI `no such file or directory`。kbagent 把这个退出归为 `code=-1` + 空 output。

#382 success-baseline 拿到的关键证据：

```
=== pod-1 attempt 1 direct ===
error: ... exec /scripts/replication-roleprobe.sh: no such file or directory ...
DIRECT_EXIT=1

=== pod-1 attempt 1 sh ===
secondarySH_EXIT=0
```

直接 exec 必失败（shebang 找不到 bash 解释器），但 `sh /scripts/replication-roleprobe.sh` 必成功（sh 把 shebang 当注释忽略，自己解释脚本）。

cmpd 模板里 roleProbe 的 command 写的是 `["/bin/sh", "/scripts/..."]` 应该是对的，但 kbagent 在某些路径上仍走"直接 exec"，导致间歇性 `code=-1+output=""`；其他路径走 sh wrapping，看到 `secondary`/`primary` 正常输出。这就是 #381 全程 fail vs #382 success 早期 3 次 fail 然后突然 success 的原因 — kbagent 内部存在两条调用路径，单凭 cmpd command 不足以保证一直走 sh。

## 修复（alpha.10）

- `addons/mariadb/scripts/replication-roleprobe.sh` 第一行 `#!/bin/bash` → `#!/bin/sh`。
- `addons/mariadb/scripts/galera-roleprobe.sh` 同上（防御性，galera 当前不在 smoke 内但同类）。
- `addons/mariadb/Chart.yaml` 版本 1.1.1-alpha.9 → 1.1.1-alpha.10。
- 脚本 body 已是 POSIX 兼容（`local`、`$( )`、`[ ]`、`printf`、`echo -n` 都能在 busybox sh 下跑），ShellSpec replication_roleprobe_spec.sh 通过 13/13。

## 验证

- `#384` (Phase 2t) 跑到 T7 PASS — VScale OpsRequest 77s Succeed，pod-1 role label 正确发布。
- `#385` (Phase 2u) 再次确认 T7 PASS。

## 给后续 addon 工程师的固化要求

1. **任何 addon 脚本若可能在 kbagent sidecar 里被直接 exec，shebang 必须用 `#!/bin/sh`**，脚本 body 必须 POSIX 兼容。
2. **新加 addon 脚本时跑一遍 `sh -n script.sh` 静态检查**，确保不依赖 bash-only 语法。
3. **不要假设镜像里有 bash**。即使 mariadb / postgres / redis 主镜像通常带 bash，**kbagent sidecar 镜像最小化**，只有 busybox。
4. **CMPD `command:` 写 `[/bin/sh, script.sh]` 不是充分保护**，kbagent 实际调用路径不止一条。源头修是把 shebang 改成 `#!/bin/sh`。

## 关联

- patch commit：`addons/mariadb/scripts/replication-roleprobe.sh` + `galera-roleprobe.sh` + `Chart.yaml` 1.1.1-alpha.10
- 任务串：#379 → #380 → #381 → #382 → #383 → #384
- 主题文档：`addon-bounded-eventual-convergence-guide.md`、`addon-test-probe-classification-guide.md`
