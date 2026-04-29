# Addon 测试探针分类与分层指南

> **Audience**: addon test / TL
> **Status**: stable
> **Applies to**: any KB addon
> **Applies to KB version**: any (test methodology, version-agnostic)

本文面向 Addon 测试工程师与技术负责人，重点总结如何让一次"取数据探针"在失败时直接落到正确的层（环境 / 路由 / 客户端 / 数据路径 / 产品语义），而不是把所有失败都归成"DB 出问题了"。

## 先用白话理解这篇文档

### 这篇文档解决什么问题

测试报"DB 看不到数据"或"role 不对"时，团队第一反应通常是"DB 有 bug"。但这种归因经常错位——**同样的 stderr / 同样的"读不到结果"，可能根因在 5 个不同层**：

1. **环境层**：CSI plugin 不 ready、kubelet 抖动、k3d 重启后 mount propagation 失效
2. **路由层**：vcluster port-forward 没起来、Service Endpoint 还没刷新、11443 connection refused
3. **客户端层**：valkey-cli / mysql 客户端 args 写错、密码 / TLS 配置不对、超时太短
4. **数据路径层**：写发到了 secondary、replication lag、bounded eventually 没等到收敛
5. **产品语义层**：role 真的是错的、replication contract 真破了、addon 代码有 bug

笼统说"DB 出问题了"会让团队**在错误的层排障**：花 2 小时调 DB 配置，最后发现是 11443 listener 没起来。

→ 真正的方法论是：**写探针时就给 5 层各自的 stderr / 退出码 / 区分信号，让每次 fail 自动落到正确的层**。

### 何时本文方法论 apply

| 场景 | 关键决策 |
|---|---|
| 写新 SQL exec 探针（`kubectl exec ... mysql ...`）| 给 client-error / TLS / route / 数据 5 层都有显式 stderr branch |
| 写新 `kubectl get` 探针看 role / replication | 区分 "API 不可达" vs "对象状态" vs "字段值" |
| 探针撞 fail 时 | 看 stderr 的 layer tag，**不要默认归 DB 层** |
| Review 探针 PR | 第一道 check 是有没有 5 层各自的 fail 信号 |
| post-restart 后探针 fail | 怀疑 layer 1（环境）或 layer 2（路由），跟 `addon-test-environment-gate-hygiene-guide.md` 7-layer 配合 |
| OpsRequest Succeed 但探针 read empty | 怀疑 layer 4（数据路径，bounded eventually 没等到收敛），跟 `addon-bounded-eventual-convergence-guide.md` 配合 |

### 读完你能做什么决策

- **写新探针时**：能 5 秒内列出 5 层各自 stderr 应该是什么样
- **探针 fail 时**：能立刻读 stderr 判断哪层错了，不再笼统归 DB
- **review 探针 PR 时**：能识别"看起来 OK 但 fail 时不能分层"的探针并 block
- **撞 transient fail 时**：能区分 "bounded eventually 没收敛" vs "真 fail"

### 为什么独立成篇

跟 `addon-test-environment-gate-hygiene-guide.md`（开跑前不踩雷）+ `addon-test-acceptance-and-first-blocker-guide.md`（跑完不混层）一起构成 "环境 → 探针 → 验收" 三阶段。本文聚焦**探针自身的实现规约**：探针应该长什么样、每层 fail 应该长什么样，是这一阶段的核心方法论。

---

## 适用场景

当你在写或维护以下探针时，本文适用：

- 通过 `kubectl exec` 在 pod 里执行 SQL / shell 取值（role、replication、read_only、行数、配置值等）
- 通过 `kubectl get` 读 cluster / component / pod / Service / OpsRequest 等资源
- 通过 `kubectl exec + wget / curl` 拉 metric / 健康检查
- 通过 OpsRequest 触发 expose / restart / reconfigure / switchover 后等终态
- 在 vcluster 或代理 API server 后面跑这类探针（11443 / port-forward / 其他 route）

## 一次探针失败时，至少要分清这几层

把"探针失败"先分到下面的某一类，再决定是 fix 探针、fix 环境、还是 fix 产品：

| 分类 | 触发条件 | 含义 | 处理 |
|---|---|---|---|
| `route_api` | `kubectl exec/get` 自己未能到达 pod / API server（API 路由层错） | 路由 / port-forward / API 端点不通 | 修路由、重连、做带 retry 的 listener；不要进入产品归因 |
| `<client>_<channel>` 比如 `db_socket_query` | `kubectl exec` 已经成功到达 pod，但客户端进程（mariadb / redis-cli / wget）退出码非零 | 客户端 / socket / 网络 / 鉴权 / SQL 语法层错 | 看客户端 stderr / channel 错误码 |
| `empty_output` | 客户端 RC=0，但 stdout 完全为空 | 客户端正常退出但没拿到数据 | 看是不是查询语义错 / 鉴权未生效 / 时序还没到 |
| `parse_empty` | RC=0、有 stdout，但解析出的字段全空 | **客户端模式与 parser 不匹配**（典型：`mysql -N -s` 与 named-field parser 同时使用） | 把 query mode / output mode 与 parser 配齐 |
| `runtime_mismatch` | 拿到值，但不在期望范围 / 不等于预期 | 路径全通，但产品/配置侧的现态与期望不一致 | 进入产品/语义归因 |
| `real_<thing>_mismatch` | 值已确认是产品语义层的真问题（如 `real_role_mismatch` / `real_data_mismatch` / `real_read_only_mismatch`） | 真实 bug 候选 | 升级到 first blocker：addon / runtime / 产品 |
| `ok` | 全链路通且值符合期望 | 真正成功 | 继续下一步 |

补充类（环境能力相关）：

| 分类 | 用途 |
|---|---|
| `<feature>_unavailable_skip` | 当前环境不具备这个能力（比如无 LoadBalancer），按 capability-aware fail-fast 跳过 |
| `<resource>_pending_<reason>` | 资源已建但未就绪（比如 `service_pending_lb`），需要等或者按时序退出 |
| `<ops>_ops_timeout` | OpsRequest 没在窗口内终态，先按动作层 blocker 收（参考 first blocker 指南） |

## 为什么必须显式分类

不分类时常见问题：

- 路由抖动 → 探针失败 → 队伍开始查 DB / 配置 / 存储，几十分钟后才发现是 port-forward 死了。
- `mariadb -N -s` 取了 `SHOW SLAVE STATUS\G`，parser 拿不到字段 → 队伍误判为"复制故障"，实际上数据完全正常。
- 没有 LoadBalancer 的环境跑 LB Expose → OpsRequest 卡 300s timeout → 看起来像"产品 bug"，其实是环境能力缺失。

显式分类把"失败信号"先归到正确的层，归因路径就只剩当层那条线。

## 分类规则（决策树）

按这个顺序判：

1. **能否拿到客户端进程的 RC？**
   - 拿不到 → `route_api`（连 pod 内的 client 都没启动 / 没退出，多半是 exec/route 层）
2. **客户端 RC 非零？**
   - 是 → `<client>_<channel>`（如 `db_socket_query`、`http_request_failed`）
3. **stdout 为空？**
   - 是 → `empty_output`
4. **stdout 有内容，但 parser 拿不到目标字段？**
   - 是 → `parse_empty`（提示：query/output 模式与 parser 不匹配）
5. **拿到值但不符合期望？**
   - 软失败（语义在收敛中）→ `runtime_mismatch`
   - 已坐实产品语义错 → `real_<thing>_mismatch`
6. **全部满足期望** → `ok`

## 实现规约（写探针时的 7 条硬规则）

无论引擎，写探针都按这套来，避免反复踩坑：

1. **必须保留客户端 RC 和 stderr**（哪怕嵌套在 `kubectl exec` 里也要透传出来）。常见做法：在 `sh -c` 里把 `__MARIADB_RC__=` / `__CLIENT_RC__=` 等 sentinel 写到 stderr，外层 awk 解析。
2. **必须保留 stdout preview**（截断到 N 行 / N 字节），不要只留 parse 结果；后期复盘时 raw stdout 是关键证据。
3. **route 失败必须独立保留**：route_action / route_rc / route_stderr / route_snapshot / port-forward listener。出 `route_api` 时这些字段必填，否则没法判断是 route 自愈中还是 route 真挂了。
4. **query/output 模式与 parser 必须显式声明且配齐**：探针返回的 context 里要带 `query_mode=` / `output_mode=`，parser 也要明确接受哪种。一次混用一定会踩 `parse_empty`。
5. **classification 输出必须是单一分类**，不要返回多分类的"或"。如果一次探针同时疑似 route 又疑似产品，就先按"最早出错那一层"归。
6. **失败时 context 必须自描述**：classification、pod、query/raw、kubectl rc/stderr、client rc/stderr、route 三件套一次性 dump 到日志，避免后面再去 reproduce 取 context。
7. **`real_*_mismatch` 是 first blocker 候选，必须经过验证**：同一探针重跑、换 pod、换会话、看 controller / pod 日志，证实是产品语义层错，再升级。直接由首次 `runtime_mismatch` 跳到 `real_*` 等于把测试当审判。

## 与 `addon-test-acceptance-and-first-blocker-guide.md` 的关系

- 那篇讲"成功语义分层 + 何时把 case 当成功 / 失败收"。
- 本篇讲"探针失败时如何把信号落到正确的层"。
- 串起来就是：
  1. 探针返回 `real_*_mismatch` → 进入 first blocker 收口流程；
  2. 探针返回 `route_api` / `<client>_<channel>` / `empty_output` / `parse_empty` → 还在测试观测面，不是 first blocker。

## 案例附录

- MariaDB：TBD — 计划补 T4 SQL 探针 / R4 role 探针 / T13 expose / exporter 各自的字段集与触发条件（先不放 broken link，等 case 文件建好后再 link）
