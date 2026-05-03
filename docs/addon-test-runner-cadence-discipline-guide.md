# Addon 测试 Runner Cadence 纪律指南 — Cadence 是操作者的义务，不是 Runner 的义务

> **Audience**: addon test engineer / agent / TL
> **Status**: stable
> **Applies to**: any KB addon long-running test operation
> **Applies to KB version**: any (methodology, version-agnostic)
> **Affected by version skew**: no

属于：方法论主题文档（不绑定单一引擎）。

## 1. 这篇要解决的问题

长时间运行的测试操作（chaos 轮次、集群安装、备份恢复、vscale 等）往往要跑 20-60 分钟。在这段时间里，操作者（测试工程师或 agent）和 reviewer（如 TL / review owner）之间有一个信息不对称：**操作者看着日志，reviewer 什么都看不到**。

如果操作者只在开始时发一条"起跑"消息，然后等 runner 结束再汇报，reviewer 面临的处境是：

> "这个 runner 到底是在正常跑，还是已经卡死，还是已经崩了？我不知道。"

**Cadence** 就是用来填这个信息缺口的：操作者以固定间隔主动向 reviewer 汇报 runner 状态。没有 cadence，reviewer 无法区分"正常运行"和"卡死/崩溃"，只能靠猜测或主动询问——这是信任债务的直接来源。

本文的触发案例：Oracle addon chaos 测试 Run 5（2026-04-30 19:19→20:17），操作者在"起跑"之后沉默 58 分钟，reviewer 被迫发出明确催促才得知 runner 状态。

## 2. 什么是 Cadence

**Cadence** 是操作者在长时间运行操作期间，以固定间隔向 reviewer 主动发送的状态汇报。

关键要素：

- **发起方**：操作者（主动），不是 reviewer（被动等待）
- **频率**：固定间隔，默认 5 分钟，不是"有进展才汇报"
- **内容**：当前 runner 状态快照（pod ready 数、phase、活跃操作、有无异常）
- **方向**：即使没有新进展，也要发；"仍在运行"本身就是有效状态

Cadence 不是 reviewer 的要求——它是操作者对 reviewer 的服务义务。reviewer 不应该需要开口问，才能知道 runner 活着。

## 3. 5 分钟是硬约束，不是建议

cadence 间隔默认为 **5 分钟**，这是硬约束，不是软性指南。

原因：

**超过 cadence 间隔的沉默，对 reviewer 等价于"runner 状态未知"。**

reviewer 没有办法从沉默中区分：
- runner 正常运行，操作者只是没说
- runner 卡在某个步骤，操作者没察觉
- runner 崩溃，操作者自己也不知道

从 reviewer 的视角，这三种情况的信息是完全相同的——都是沉默。

**reviewer 没有 benefit of doubt**。在 cadence 间隔过期之后，reviewer 有充分理由认为 runner 状态不明，并且有理由中断等待来询问。这不是 reviewer 苛刻，而是操作者未履行义务。

超过 cadence 间隔不发 ping，是操作者主动放弃了告知 reviewer 的责任。

## 4. Cadence 的责任归属与常见误区

### §4a — Cadence 是操作者的义务，不是 runner 的义务

这是最核心的一点，也是最常被混淆的一点。

**角色区分**：
- **runner**（chaos 脚本、测试脚本、DBCA 安装脚本）是**被观察的对象**
- **操作者**（测试工程师、agent）是**观察者**
- **reviewer**（TL / review owner）在等**操作者**的汇报，不是在等 runner 自己输出

cadence 是操作者对 reviewer 的汇报义务。它和 runner 有没有新输出无关，和 runner 有没有完成当前步骤无关。

**操作者是独立于 runner 而存在的角色**。操作者的工作是：监控 runner、读日志、形成状态判断、向 reviewer 汇报。这个工作不依赖 runner 先完成某件事。

### §4b — 三类常见误区，均不构成 cadence 豁免

**误区 1："我在等 runner，等它结束再汇报。"**

错误之处：等 runner 和发 cadence ping 是两件完全独立的事情。你可以一边等 runner 运行，一边每 5 分钟读一次日志汇报当前状态。它们不互斥。

reviewer 等待的是**你**的汇报，不是 runner 的完成信号。runner 还没结束，并不妨碍你汇报"runner 仍在运行，当前 pod 3/4，已跑 12 分钟"。

**误区 2："runner 没有新输出，没东西可说。"**

错误之处："没有新进展"本身就是一种状态，汇报它是有价值的。

一条"仍在运行，pod 3/4 维持 8 分钟，无报错"的 ping，对 reviewer 的价值远大于沉默。沉默传递的信息是"未知"；no-progress ping 传递的信息是"可控的等待"。

**误区 3："我没有新进展可以汇报。"**

错误之处：cadence ping 的目的不是"有新发现才汇报"，而是"让 reviewer 知道你还在"。"目前无新进展"就是你要汇报的内容。

## 5. 里程碑主动 Ping

除了固定间隔 cadence，关键里程碑的发生应该立即触发 ping，不等下一个 cadence tick。

**里程碑示例**（不限于此）：
- 集群 phase 变为 Running
- DG broker setup 成功
- 每一轮 chaos round 完成（PASS 或 FAIL）
- SQL ready / 主库角色确认
- backup 开始 / 完成
- switchover 发生
- 任何 ERROR 或意外状态出现

**判断标准**：如果你正在记录一条日志，而这条日志是你会记录在 work-log 里的——那么同时向 reviewer ping 一条。如果这个事件值得记录，它就值得 ping。

里程碑 ping 和 cadence ping 叠加，不互相替代。里程碑在两个 cadence tick 之间发生，立即 ping，不等下一个 tick。

## 6. Cadence 触发器的设计原则：与 Runner 进程解耦

这是实现层面最关键的一点：**cadence 触发器必须与 runner 进程解耦**。

原因：如果 cadence 触发器依赖 runner 进程，runner 一旦卡住（比如卡在 20 分钟的 DBCA 等待、卡在网络 I/O、卡在 shell 循环），cadence 也同时卡住。这等于没有 cadence。

**正确设计**：cadence 触发器是一个独立的机制，不论 runner 是否活跃、是否阻塞，都能按时触发。

平台无关的可行方案（任选其一或组合）：
- **定时提醒**（例如 `slock reminder schedule --recurring 5m`），锚定到任务消息，触发时由操作者读日志汇报
- **Cron / 定时任务**，独立进程，读 log 文件并发送状态
- **IM bot 定时消息**，提醒操作者主动发 ping
- 任何其他机制，只要满足"runner 卡住时 cadence 仍然能触发"

**错误设计**：把 cadence 逻辑嵌进 runner 脚本（比如在 chaos 脚本内部每 5 分钟调用一次汇报函数）。原因：脚本 hang 住时，嵌入的 cadence 逻辑也 hang 住，cadence 失效。

核心原则：**cadence 触发器与 runner 进程解耦**。cadence 必须能在 runner 卡死时继续工作。

## 7. 打破 Cadence 的后果

打破 cadence（超过间隔未 ping）的实际代价：

1. **reviewer 无法区分正常运行与卡死/崩溃**，必须主动询问才能获得本应主动提供的信息
2. **信任债务**：reviewer 下次会更早催促，因为无法依赖操作者的主动汇报
3. **"我在后台跑"不是理由**：reviewer 看不到后台，操作者的工作就是把后台可见化
4. **cadence 断档需要显式补救**：如果因为某种原因出现了沉默间隔，重新开始时必须发"恢复汇报"消息，补充这段时间的状态，而不是假装断档没发生

打破 cadence 一次，reviewer 会怀疑后续每次 ping 的可靠性。这不是个人信任问题，而是流程债务：操作者未能维护的纪律，需要 reviewer 用额外的主动询问来补偿。

## 8. 给后续 addon 工程师的固化要求

1. **cadence 是操作者的义务，不是 runner 的义务**——runner 没有新输出，操作者仍然要 ping
2. **沉默超过 cadence 间隔 = 状态未知 = 等价于 runner 已失败**——reviewer 无 benefit of doubt
3. **no-progress ping 仍然有效**——"仍在运行，pod 3/4 维持 8min"是合法 ping，不是废话
4. **里程碑事件立即 ping，不等下一个 tick**——关键状态变化不能被 cadence 间隔延迟
5. **cadence 触发器必须独立于 runner 进程运行**——runner hang 时 cadence 不能一起 hang
6. **log-only 不等于 ping**——把状态写进日志文件不能替代向 reviewer 发可见消息
7. **打破 cadence 后必须发显式恢复消息**——"我回来了，这段时间发生了 X、Y、Z"，不能无声续跑

## 9. 反模式表

| 反模式 | 描述 | 正确做法 |
|---|---|---|
| "等 runner 结束再汇报" | runner 运行期间操作者完全静默，以为等结束再一次性汇报即可 | 设置独立 cadence 定时器，每 5min 主动读日志汇报当前状态 |
| "没新进展不 ping" | 认为"没进展 = 不需要汇报"，沉默对 reviewer 传递有效信息 | no-progress ping 仍然有效（"仍在运行，pod 3/4 维持 8min"） |
| "把 cadence 逻辑嵌入 runner 脚本" | 在测试脚本内部定期汇报，runner hang 时 cadence 也 hang | cadence 触发器独立于 runner 进程，runner 卡死时 cadence 仍能工作 |
| "milestone 消化完再 ping" | 关键事件发生后等到下一个 cadence tick 才汇报 | 里程碑立即触发 ping，不等下一个 tick |

## 10. 自检清单（启动长时间操作前逐项确认）

启动任何预计超过 5 分钟的操作之前：

1. **我是否设置了独立于 runner 的 cadence 触发器？** 触发器在 runner 卡住时还能工作吗？
2. **cadence 间隔是否 ≤5 分钟？** 不是 10 分钟，不是"大概每隔一会儿"。
3. **我是否梳理了本次操作的关键里程碑？** 每个里程碑完成后是否会立即 ping，不等 cadence tick？
4. **如果 runner 卡在某步 20 分钟，我的 cadence 触发器还能按时触发吗？** 如果答案是否，重新设计触发机制。

## 11. 案例附录

### 反面案例 — Run 5（2026-04-30 19:19→20:17，58 分钟沉默）

**时间线**：

| 时刻 | 事件 |
|---|---|
| 19:18 | Run 5 启动（fresh cluster install，chaos 套件） |
| 19:19 | 操作者发出最后一条 ping："Run 5 起跑" |
| 19:19–20:17 | **58 分钟完全静默**——操作者在等 runner，未设置独立 cadence 触发器 |
| 20:17 | Reviewer 主动发出催促："19:19 起跑后 58min 无 ping. 立即报告: PID/log/cluster status/sentinel/死了?" |

**问题分析**：

操作者的心理模型是"等 runner 结束再一次性汇报"。但 reviewer 的心理模型是"操作者应该每 5 分钟让我知道 runner 活着"。这两个模型在 58 分钟的沉默中完全错位。

reviewer 在 20:17 之前完全无法区分：
- runner 正常跑了 58 分钟，操作者只是忘了 ping
- runner 在某步卡死，操作者没察觉
- runner 崩溃，操作者也不知道

**直接后果**：reviewer 必须打断自己的工作来询问本应主动提供的信息；操作者的汇报可靠性下降；下次 reviewer 会更早催促。

**根因**：没有独立于 runner 的 cadence 触发器。操作者在等 runner，cadence 没有独立的触发机制。

---

### 正面案例 — Run 6（2026-04-30 20:30→20:58，7 次 cadence ping，全部在间隔内）

**时间线**：

| 时刻 | 类型 | 内容摘要 |
|---|---|---|
| 20:30 | 起跑 | Run 6 启动，设置独立 5min cadence 提醒 |
| 20:35 | cadence ping 1 | oracle-0 4/4 Running，oracle-1 3/4（RMAN active），无报错 |
| 20:41 | cadence ping 2 | oracle-0 4/4，oracle-1 4/4，phase Creating，DG broker 初始化中 |
| ~20:44 | 里程碑 ping | phase=Running（立即 ping，不等下一个 tick） |
| ~20:47 | 里程碑 ping | C00 DG SUCCESS 验证完成（5 项证据：broker status / lag / role / SCN / ping） |
| 20:50–20:55 | 每轮 ping | C01 r1-r5 各轮完成后立即汇报（PASS/FAIL + 简要证据） |
| 20:58 | 最终汇报 | Run 6 完成，整体 PASS，附汇总表 |

**7 次 cadence ping，reviewer 全程未需要主动询问状态。**

**关键差异**：

1. **独立触发器**：操作者在起跑时设置了独立的 5 分钟提醒，提醒触发时操作者主动读日志汇报，与 runner 进程解耦
2. **no-progress ping 合理**：20:35 的 ping 没有重大新进展（runner 仍在运行），但它让 reviewer 知道"runner 活着，状态可控"
3. **里程碑不等 tick**：20:44 phase=Running 立即 ping，没有等到下一个 5min tick
4. **reviewer 零询问**：整个 28 分钟操作，reviewer 从未需要主动询问，因为状态始终可见

**bash 参考（仅供附录）**：

如果在 bash 环境中手动管理 cadence，可以在 runner 启动后，用后台独立进程发送提醒，而不是在 runner 脚本内部嵌入：

```bash
# 独立 cadence 提醒进程（与 runner 解耦）
(
  while true; do
    sleep 300
    echo "[CADENCE] 5min tick: 请读日志汇报当前 runner 状态"
  done
) &
CADENCE_PID=$!

# 启动 runner
bash chaos_run.sh > /tmp/chaos.log 2>&1
RUNNER_RC=$?

# runner 结束后停掉 cadence 提醒
kill $CADENCE_PID 2>/dev/null

echo "runner exit code: $RUNNER_RC"
```

注意：上面的 cadence loop 在 runner hang 时仍会按时触发提醒——这是解耦的核心价值。提醒触发时，操作者读 `/tmp/chaos.log` 末尾，形成状态判断，向 reviewer 发 ping。

如果使用 slock，可以通过 `slock reminder schedule --recurring 5m` 锚定到任务消息，效果相同。

---

### 与其他文档的关系

- [`addon-test-acceptance-and-first-blocker-guide.md`](addon-test-acceptance-and-first-blocker-guide.md) — 测试验收标准与 first blocker 归层
- [`addon-evidence-discipline-guide.md`](addon-evidence-discipline-guide.md) — 汇报时证据强度与结论的匹配规则
- [`addon-test-environment-gate-hygiene-guide.md`](addon-test-environment-gate-hygiene-guide.md) — 起跑前环境就绪门控

本文与 evidence-discipline-guide 互补：evidence guide 管"汇报什么内容"（证据强度），本文管"什么时候汇报"（cadence 纪律）。两者都是操作者对 reviewer 的服务义务的组成部分。
