# MariaDB role-publish observation gap attribution case (#396 Route B closure)

属于：[`addon-test-host-stress-and-pollution-accumulation-guide.md`](../../addon-test-host-stress-and-pollution-accumulation-guide.md) cell-4 实证 + [`addon-bounded-eventual-convergence-guide.md`](../../addon-bounded-eventual-convergence-guide.md) 的 attribution 数据补充。

## 背景

cell-4 R1 (04-29 14:14) hard-freeze 时观察到 chaos C4-C8 + T12 多个 phase 都撞 "数据健康但 role label 未在 bounded window 内 re-publish" 的 observation-class fail（参见 [appendix D](../../addon-test-host-stress-and-pollution-accumulation-guide.md#appendix-d-cell-4-mariadb-solo)）。westonnnn 04-29 15:29 拍 Route A：用 5-segment timing 埋点 attribute 这条 latency 到底花在哪一段。

## 调查 timeline

- **04-29 15:30-16:30**：v1 instrumentation design — defective（datadir permission denied + kbagent watcher attach issue）
- **16:40 R1 PASS sample (`mdb-rolepub-161841`)**：N=1 clean attribution data
- **16:57-17:13**：R2 / R2-replacement supplementary fail samples（T8/T9/T7 各撞 observation gap）
- **17:46-22:00**：westonnnn 拍 Route B + autonomy → 起 v3 instrumented run（多 phase trigger）
- **21:54-22:07**：5 个 preserved cluster 累积触发 cell-2 cascade → cleanup → host 自愈（cross-engine N=2 evidence）
- **22:18-22:38**：v3 fresh runs，pre-T6 多次 fail（CM3 / T3 / CM2-4），dual-load 期间 fail rate 升高
- **22:41**：closure report，stop further runs，accept R1 attribution + multi-phase fail evidence as sufficient
- **22:57**：westonnnn confirm closure approved

## R1 PASS-path 5-segment attribution

Sample `mdb-rolepub-161841` (PASS=80 / FAIL=0 / SKIP=0)；C4 round-3 timing：

| Segment | Latency | 占比 | 解读 |
|---|---:|---:|---|
| kill → KB API ack (new pod uid) | 1.137s | 3% | pod recreation detect，正常 |
| **uid → ready (DB restart + replication)** | **21.762s** | **51%** | mariadbd 启动 + relay log apply + slave_status 报 OK，物理时间 |
| ready → probe secondary (probe interval) | 8.630s | 20% | 5s 间隔 + jitter 实测 6-9s |
| probe → kbagent push event | 0.747s | 2% | local IPC，正常 |
| event → KB controller label publish | 0.695s | 1.6% | controller reconcile，**非瓶颈** |
| **label → runner observe pacing** | **10.449s** | **24%** | runner 5s 轮询 + jitter |
| **total** | **43.968s** | | |

**关键 finding**：**KB controller event/label 合计仅 ~3%**，**否定** "KB controller 是 observation gap 主瓶颈" 假设。

主导 latency = DB restart (51%) + roleProbe cadence (20%) + runner observe pacing (24%) = 95%。

Evidence package: `artifacts/mdb-rolepub-161841-task396-rolepub-debug-161841.tar.gz` (sha256 `fa1e9440e465f1cbdb13f6f21d6c5d24de6b41c50d16b7297a10e70a7ed5ce8f`).

## Multi-phase fail matrix

观察 gap 不只 chaos 触发 — 7+ phase 都撞同 observation-class fail：

| Sample | Condition | First fail phase | Notes |
|---|---|---|---|
| `mdb-vc396r1-134638` | cell-4 solo | C4-C8 chaos + T12 VolumeExpansion | route_holes/read_holes=0 (数据健康) |
| `mdb-rolepub-164018` | single-load supplementary | T8 Restart + T9 Switchover | 同 pattern |
| `mdb-rolepub-204819` | single-load supplementary | T7 VScale | + CSI 23→25 single-sidecar drift |
| `mdb-rolepub-221722` | dual-load (with valkey G2) | CM3 sync_binlog reconfigure | dual-load 期间 fail rate 升高 |
| `mdb-rolepub-222841` | dual-load (with valkey G2) | T3 service VIP + CM2/CM3/CM4 | OpsRequest Failed series |

**累计触发 phase**：T3 service VIP / CM2-CM4 reconfigure / T7 VScale / T8 Restart / T9 Switchover / C4-C8 chaos / T12 VolumeExpansion。

→ **observation gap 不是 chaos kill 专属触发器**，而是任何 "pod recreation + role 重发" 操作都可能撞的普遍现象。

## 推翻的假设

| 原假设 | 状态 | Evidence |
|---|---|---|
| "KB controller 是主瓶颈" | ❌ 推翻 | R1 attribution 显示 controller 仅 3% |
| "C4 chaos kill 是特殊触发器" | ❌ 推翻 | 7+ phase 都撞 |
| "120s deadline 太短" | ⚠️ 部分 | 多数 PASS 在 deadline 内（48s/40s/43s），但 CM 类 reconfigure 可能需要更长 |

## 三条 actionable follow-up

详见 task #397 / #398 / #399：

1. **roleProbe cadence/latency 优化**（task #397）：5s 间隔贡献 20% latency，可缩到 2-3s 或加事件驱动 trigger
2. **runner deadline 分层口径**（task #398）：120s 一刀切不合理，per-phase deadline 按 P95+margin 标定
3. **helper observability 补强**（task #399）：per-tick DB truth + stderr/artifact 持久化，未来 fail-path 5-segment timing 也能 capture（不依赖 instrumentation overlay）

## 没做的（v3.1 future follow-up）

CM phase fail-path 的精确 5-segment timing 没采到（v3 instrumentation 起点是 T6）。如果未来需要 CM 阶段的精确 attribution 数字，可以做 v3.1（CM2/CM3/CM4 加 trigger 点）。但**定性结论不会改**：KB controller 仍是非瓶颈，DB restart + cadence 仍主导。

## 跨次 self-heal observation (chapter 1 hero takeaway 第二个 anchor)

调查期间撞过两次 cell-2 cascade，都通过 cleanup-only 自愈，**没动 docker / k3d**：

| Time | Engine | Trigger | Recovery |
|---|---|---|---|
| 04-29 00:41 | Valkey | 多 suite 污染累积 | CSI 4/4 + /readyz ok 自愈 |
| 04-29 22:07 | MariaDB | 5 preserved frozen cluster 累积（新发现 cell-2'） | CSI 4/4 + /readyz 5×ok 90s sustained 自愈 |

→ chapter 1 "清污染 → 自愈" 跨引擎双 N=2 confirmation。

## 评论 / Lessons learned

1. **Instrumentation 第一版必有 defects** — v1 stdout 污染 + watcher attach gap → v2 → v3 三轮迭代才稳定
2. **dual-load 期间 fail rate 升高，但 fail shape 一致** — 噪音放大但不引入新分类
3. **frozen cluster 不是 idle** — 04-29 22:00 反模式：5 preserved cluster 累积 reconcile load 触发 cell-2 cascade
4. **R1 单 sample 已够定性 attribution** — N=3 收集 fail-path 数据 ROI 低，已知 KB controller 不是瓶颈

## 关联

- 主题文档：[`addon-test-host-stress-and-pollution-accumulation-guide.md`](../../addon-test-host-stress-and-pollution-accumulation-guide.md) cell-4 / appendix D
- bounded-eventual: [`addon-bounded-eventual-convergence-guide.md`](../../addon-bounded-eventual-convergence-guide.md) 提供 retry / cadence framework
- evidence-discipline: [`addon-evidence-discipline-guide.md`](../../addon-evidence-discipline-guide.md) — 调查期间多次 framing inflation 修正（"Mac 退化" / "15min 平均" / "N=3 hot-fix refail" / "v3 single sample sufficient" 等都在 channel discussion 里被 challenge 修正）
- Jack closure report：本案例正文整合
