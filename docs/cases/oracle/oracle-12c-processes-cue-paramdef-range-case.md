# Oracle 12c `processes` cue paramdef 范围太宽松：reconfigure_deep T22d 卡死 25min+

> **Audience**: addon dev
> **Status**: stable
> **Applies to**: Oracle 12c addon on KubeBlocks（方法论可复用，参见 [`addon-paramdef-cue-range-validation-guide.md`](../../addon-paramdef-cue-range-validation-guide.md)）
> **Applies to KB version**: 验证于 KB 1.0.3-beta.5（reconfigure_deep Run 2，2026-04-30，Cluster `ora-rcd-53242`）
> **Affected by version skew**: paramdef cue API 在 KB main 重构（ParamConfigRenderer 拆进 ParametersDefinition、reconfigure 路由到 ComponentParameter，预计随 KB 1.2 发布）；cue range 校验语义本身稳定，迁移时按 [`addon-chart-vs-kb-schema-skew-diagnosis-guide.md`](../../addon-chart-vs-kb-schema-skew-diagnosis-guide.md) 流程

属 [`addon-paramdef-cue-range-validation-guide.md`](../../addon-paramdef-cue-range-validation-guide.md) 工程现场补充。

## 现场

- 时间：2026-04-30 reconfigure_deep Run 2
- Cluster：`ora-rcd-53242`，Oracle 12c standalone
- 测试用例：T22d — 提交 reconfigure `processes=10`，期望 OpsRequest 在 ValidatePhase 被 KB 拒（Failed），cluster 保持 Running，参数值不变
- 失败现象：
  1. cue 通过（`processes: int & >=6`，10 ≥ 6）
  2. KB 写入 spfile：`*.processes=10`
  3. Oracle 容器内自动重启
  4. 启动失败：`ORA-603: ORACLE server session terminated by fatal error` / `ORA-1092: ORACLE instance terminated. Disconnection forced`
  5. Oracle 容器 **restartCount=0**（liveness initialDelay=1800 未触发，Bug #12 fix 生效 ✓）
  6. KB OpsRequest 状态停在 `Running`，**已 25min+，无超时机制**
  7. 测试卡在 `wait_for_ops` / `ops_settled_phase` 循环

## 根因

`addons/oracle/configs/oracle-12c-config.cue:6`：

```cue
// user processes
processes: int & >=6
```

直觉地按 Oracle 文档：[Oracle Database Reference - PROCESSES](https://docs.oracle.com/database/121/REFRN/GUID-...) 把 minimum 写成 6（文档里确实写的「The default value is derived from the value of PARALLEL_MAX_SERVERS, plus 6」/「The minimum value of this parameter is 6」）。

但 Oracle 文档 minimum=6 是**硬下限**（spfile parser 接受），不是**实际可启动值**。Oracle instance 启动时背景进程清单（partial）：

```
PMON, SMON, DBWn, LGWR, CKPT, ARCn, MMON, MMNL, RECO, FBDA, ...
```

光这些必备 process 就远超 6 个；外加 SYSTEM/SYSAUX 元数据扫描、JOB queue worker、连接 listener，实际启动需要约 100。`processes=10` → 启动到 SMON 就 fail → ORA-603/1092 → instance terminated。

## 5 件 Evidence Dig

1. **cue 实际允许**：`grep '>=' addons/oracle/configs/oracle-12c-config.cue` → `processes: int & >=6`。10 满足
2. **spfile 写入**：`kubectl exec ora-rcd-53242-oracle-0 -c oracle -- bash -c 'cat $ORACLE_BASE/spfile*.ora' | strings | grep processes` → `*.processes=10`（确认写入了）
3. **Oracle alert log**：`tail /opt/oracle/diag/rdbms/.../alert/log.xml`：
   ```
   ORA-603: ORACLE server session terminated by fatal error
   ORA-1092: ORACLE instance terminated. Disconnection forced
   GEN0 (ospid: NNN): terminating the instance due to error 822
   ```
4. **Oracle 容器 restartCount=0**：liveness initialDelay=1800 没触发（Bug #12 fix 生效 — 不会因为 instance terminated 就立刻 SIGKILL 容器再 restart）
5. **OpsRequest 卡 Running**：
   ```
   conditions:
     - type: WaitForProgressing → True
     - type: ReconfigureStarted → True
     - type: Succeed → 没有
     - type: Failed → 没有
   ```
   25min+ 不动。

## Fix

```cue
// user processes
- processes: int & >=6
+ processes: int & >=100
```

`oracle-23ai-config` 检查只有 `.tpl` template，无 cue range constraint，无需同步。

## Run 3 实证（fix 验证）

T22d 改动：

- cue lower bound 100
- T22d 测试断言改为 negative case：
  ```
  submit processes=10
  expect: OpsRequest.status.phase = "Failed" within 30s
  expect: cluster phase remains Running
  expect: SHOW PARAMETER processes 返回原值
  ```

Run 3 (cluster `ora-rcd-60842`) T22d 实测：

```
T22d: processes=10 → Failed (Bug #16 fix 生效)  ✓
ops phase=Failed within ~10s
cluster phase remains Running
原 processes 值未改
```

完整 Run 3 reconfigure_deep：`PASS: 10  FAIL: 0  SKIP: 0`，全程 RESTARTS=0。

## Bug #17 候选（advisory，不阻塞）

KB OpsRequest 当前**无 reconfigure 后 DB 起不来 → OpsRequest 自动 Failed** 机制。Bug #16 fix 后这个 path 不再触发（cue 在 ValidatePhase 拒掉）。但 KB 应该有"DB pod NotReady 持续 N 分钟 → OpsRequest Failed"兜底逻辑；smoke 全绿后可以单独 brief 项目 owner 决定是否给 KB team 提。

## 教训

1. **schema 数值 lower bound 不能直接抄文档**：先实测 boundary 值能不能起 cluster
2. **negative test 必须用 boundary - 1 / boundary + 1 验证 schema**，否则 schema 形同虚设
3. **OpsRequest 没有内建"DB 起不来"超时机制**：addon 侧的 schema 必须在 ValidatePhase 拦下来；流到 ReconfigureStarted 之后就无解（除手工 fresh restart）
4. **cluster 一旦因 reconfigure 失败而卡 Running**，回到 fresh state 比手工修复快得多——不要恋战
5. **helm upgrade 在 ConfigMap/CmpD 字段冲突时 silent skip**：这次留住了 kubectl-patched Bug #12/#13 fix，刚好对我们有利；但反例（漏 patch → 拿到旧 helm template）必须警觉，新 cluster 启动前要 verify 实际值
