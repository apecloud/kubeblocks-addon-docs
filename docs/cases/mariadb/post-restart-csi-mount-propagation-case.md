# 案例：post-restart 后 csi-hostpath mount propagation 失效，MariaDB T1 timeout

属于：[`addon-test-environment-gate-hygiene-guide.md`](../../addon-test-environment-gate-hygiene-guide.md) 的引擎案例。

## 背景

- 任务：`#376` Phase 2l，post-restart 后跑 vcluster 全量 async smoke。
- 样本：`mdb-vc376a1-221520`。
- 时间：2026-04-27 22:14–22:38。
- 现象：T1 timeout（`Cluster ... not Running after 300s, phase=Creating`），addon / mariadb / kbagent / exporter / init-syncer 全都没启动。

## 现场摘要

source（vcluster）侧：
- Cluster / Component 一直 `Creating`。
- Pod `mdb-vc376a1-221520-mariadb-0` 一直 Pending，`PodScheduled=False`，scheduler 报 `0/4 nodes available because pod has unbound immediate PersistentVolumeClaims`。
- PVC `data-mdb-vc376a1-221520-mariadb-0` 一直 Pending（StorageClass=`csi-hostpath-resize`）。

host 侧（k3d）：
- Translated PVC 一直 Pending；events: `ExternalProvisioning - waiting for a volume to be created by external provisioner hostpath.csi.k8s.io`。
- CSI 套件全坏：
  ```
  csi-hostpathplugin-0      1/4   CrashLoopBackOff x16
  csi-hostpath-provisioner-0 0/1   Error x8
  csi-hostpath-attacher-0    0/1   CrashLoopBackOff x8
  csi-hostpath-resizer-0     0/1   CrashLoopBackOff x8
  csi-hostpath-snapshotter-0 0/1   CrashLoopBackOff x8
  ```
- kubelet 在 csi-hostpathplugin-0 反复报：
  ```
  Error: failed to generate container ... spec:
  path "/var/lib/kubelet/pods" is mounted on "/var/lib/kubelet"
  but it is not a shared mount
  ```

## 根因

post-restart 后 k3d 节点 `agent-0` 上 `/var/lib/kubelet` 不再是 rshared 挂载，而 csi-hostpathplugin 的 hostPath volume 用 `Bidirectional` mountPropagation，必须 parent 是 shared mount 才能起容器。容器没起来 → csi.sock 永远不出现 → 下游 provisioner / attacher / resizer / snapshotter 连不上 socket → 退出 / CrashLoop → PVC 永远 Pending。

## 处置

- 测试侧：T1 fail-fast 已落到 `storage_provisioner_unavailable`，addon 域无任何结论。冻结现场（runner / log / postmortem 已 tar 上传），不重跑、不重建 sample。
- 测试 task：`#376` 标 done（任务目标"跑到 pass 或 first blocker"已达成，blocker 不在 MariaDB 域）。
- host 侧：交本机维护人；候选修法（按代价递增）：
  1. 重启 Docker / k3d，看 mount 状态是否被刷干净；
  2. 进 k3d agent 容器 `mount --make-rshared /var/lib/kubelet`，再 delete 重启所有 csi-hostpath* pod；
  3. 重建 k3d 集群，固化 rshared mount 在创建参数里。

## 给后续的固化要求

下一轮 Phase 2m 起，**post-restart gate 必须把 CSI 健康度纳入门禁**，不能只看 StorageClass 字段。具体在 gate 阶段需要：

- 全部 csi-hostpath-* pod Running 且全部容器 Ready；
- csi-hostpathplugin 主容器 Ready（不是 1/4）；
- 任一 PVC test-create 能在 N 秒内 Bound（粗校验 csi.sock 通路）；
- 留独立证据文件 `gate-csi-health.txt`。

未达成则 gate 失败，不进 runner。

## 关联证据

- tarball：`mdb-vc376a1-221520-task376-t1-storage-postmortem.tar.gz`（attachment id `a0af745c-eb2f-439a-b04d-f844b7641d6e`）
- 关键文件：
  - `mdb-vc376a1-221520-t1-postmortem-223310/host/csi-hostpath-pods-wide-grep.txt`
  - `mdb-vc376a1-221520-t1-postmortem-223310/host/csi-hostpathplugin-0.describe.txt`
- 任务讨论：`#mariadb:925ee723`（`#376` 任务 thread）
