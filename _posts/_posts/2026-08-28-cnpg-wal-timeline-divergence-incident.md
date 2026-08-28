---
layout: post
title: "一次 CloudNativePG WAL 撑满磁盘事故：从 Timeline 分叉到副本重建"
date: 2026-08-28 15:40:00 +0800
categories: [kubernetes, postgresql, cloudnative-pg, incident-response]
---

这篇文章记录一次真实的 CloudNativePG 故障：一个已经发生 Timeline 分叉的副本长期无法追上主库，物理复制槽因此持续保留 WAL，最终把 20 GiB 的数据库卷写满，导致主库拒绝启动，依赖数据库的 Keycloak 也随之不可用。

这次事故最值得记录的并不是某一条命令，而是完整的证据链：**节点短暂卡死 → CNPG 故障切换 → Timeline 分叉 → 复制槽保留 WAL → 磁盘写满 → 数据库服务中断**。

> 文中的内网地址和部分环境信息已做省略或泛化处理。

## 环境

- Kubernetes：RKE2 v1.28
- CloudNativePG：1.24.0
- PostgreSQL：15
- 数据库实例：3 个
- 存储：vSphere CSI，每个实例 20 GiB
- 复制模式：异步复制，`minSyncReplicas` 和 `maxSyncReplicas` 均为 0
- WAL 参数：`min_wal_size=2GB`，`max_wal_size=8GB`
- 持续归档：对象存储，保留策略 7 天

实例最初为：

```text
keycloak-pg-1
keycloak-pg-2
keycloak-pg-3
```

## 故障现象

Keycloak 无法就绪，PostgreSQL 日志反复出现：

```text
waiting for WAL to become available at 1A0/62000018
```

```text
could not connect to the primary server:
connection to server at "keycloak-pg-rw", port 5432 failed:
Connection refused
```

随后出现最关键的一条日志：

```text
new timeline 4 forked off current database system timeline 3
before current recovery point 1A0/62000000
```

从 Kubernetes 看，`keycloak-pg-2` 处于 `CrashLoopBackOff`，而 Keycloak Pod 虽然是 Running，但未通过就绪检查。

## 第一步：确认集群实际没有主库

查看 CNPG Cluster 状态：

```bash
kubectl get cluster.postgresql.cnpg.io keycloak-pg \
  -n keycloak -o yaml
```

关键状态如下：

```yaml
status:
  currentPrimary: keycloak-pg-2
  targetPrimary: keycloak-pg-2
  phase: Not enough disk space
  readyInstances: 2
  instancesReportedState:
    keycloak-pg-1:
      isPrimary: false
      timeLineID: 4
    keycloak-pg-2:
      isPrimary: false
    keycloak-pg-3:
      isPrimary: false
      timeLineID: 3
```

虽然 CNPG 仍把 `pg-2` 记录为当前主库，但三个实例都报告 `isPrimary: false`。也就是说，当时没有真正可写的 PostgreSQL 主库，`keycloak-pg-rw` 自然无法提供服务。

`pg-1` 已经位于 Timeline 4，而 `pg-3` 仍停留在 Timeline 3。

## 第二步：确认主库为什么拒绝启动

`pg-2` 每次启动只维持约一秒，然后以退出码 4 结束。当前容器日志直接给出了原因：

```text
Checking for free disk space for WALs before starting PostgreSQL
Detected low-disk space condition, avoid starting the instance
```

这不是 PostgreSQL 数据损坏，也不是 vSphere CSI 挂载失败，而是 CNPG 的磁盘保护机制：剩余空间过低时，Instance Manager 会拒绝启动 PostgreSQL，避免继续写入导致更严重的损坏。

## 第三步：到底是什么占满了磁盘

无法进入已经 CrashLoop 的 `pg-2`，于是先检查同处 Timeline 4、仍能连接的 `pg-1`：

```bash
kubectl exec -n keycloak keycloak-pg-1 -c postgres -- \
  df -h /var/lib/postgresql/data
```

结果：

```text
Filesystem  Size  Used  Avail  Use%
/dev/sdf     20G   19G    13M  100%
```

继续查看目录占用：

```bash
kubectl exec -n keycloak keycloak-pg-1 -c postgres -- \
  du -sh \
  /var/lib/postgresql/data/pgdata/base \
  /var/lib/postgresql/data/pgdata/pg_wal \
  /var/lib/postgresql/data/pgdata/pg_tblspc \
  /var/lib/postgresql/data/pgdata/pg_logical
```

结果非常明显：

```text
38M   pgdata/base
19G   pgdata/pg_wal
4.0K  pgdata/pg_tblspc
16K   pgdata/pg_logical
```

真正的业务数据只有约 38 MiB，占满磁盘的是 19 GiB WAL。

## 第四步：是谁阻止了 WAL 回收

查询物理复制槽：

```sql
SELECT slot_name,
       slot_type,
       active,
       restart_lsn,
       wal_status,
       safe_wal_size
FROM pg_replication_slots
ORDER BY slot_name;
```

结果只有一个不活跃的槽：

```text
slot_name               active  restart_lsn   wal_status
_cnpg_keycloak_pg_3     false   1A0/60000110 reserved
```

再计算该复制槽与当前回放位置之间的距离：

```sql
SELECT pg_last_wal_receive_lsn() AS receive_lsn,
       pg_last_wal_replay_lsn() AS replay_lsn,
       pg_size_pretty(
         pg_wal_lsn_diff(
           pg_last_wal_replay_lsn(),
           '1A0/60000110'
         )
       ) AS retained_for_pg3;
```

结果：

```text
receive_lsn   replay_lsn    retained_for_pg3
1A4/FC000000  1A4/FC000000  18 GB
```

至此，WAL 撑满磁盘的原因已经完全确认：`pg-3` 对应的物理复制槽落后约 18 GiB，并且一直阻止 PostgreSQL 回收旧 WAL。

## WAL 会轮转，为什么还会占满磁盘

WAL 轮转和旧 WAL 清理是两件事。

可以把 WAL 想成监控录像：当前文件写满后，PostgreSQL 会换一个新文件继续写，这叫轮转；但只有确认旧录像不再被任何副本、归档或恢复流程需要时，旧文件才能被回收。

物理复制槽相当于在某个 WAL 位置放置书签：

```text
pg-3 还没收到这里之后的日志，不允许删除。
```

即使副本断开，PostgreSQL 也不会擅自判断它永久失效，因为它可能只是临时网络故障。默认配置下，复制槽可以无限保留 WAL。

`max_wal_size=8GB` 也不是硬上限。它主要影响检查点行为；当复制槽仍要求保留 WAL 时，`pg_wal` 完全可以超过 8 GiB。

## Timeline 分叉到底是什么

Timeline 是 PostgreSQL 的 WAL 历史分支编号，不是时间戳。

故障切换前，所有实例沿同一条历史前进：

```text
Timeline 3: A → B → C → D
```

一个副本被提升为新主库后，会产生新的 Timeline：

```text
Timeline 3: A → B → C → D
                    \
Timeline 4:          E → F
```

如果 `pg-3` 已经在旧 Timeline 回放到 D，而新主库从更早的 C 分叉，`pg-3` 就不能直接把 E、F 接到 D 后面。两条历史已经冲突，单纯重启无法解决，必须执行 `pg_rewind` 或从当前主库重新克隆。

## 先恢复业务

### 1. 扩容存储

先确认 StorageClass 支持扩容：

```bash
kubectl get storageclass vsphere-csi-sc \
  -o jsonpath='allowVolumeExpansion={.allowVolumeExpansion}{"\n"}'
```

结果为：

```text
allowVolumeExpansion=true
```

将集群存储声明从 20 GiB 调整到 40 GiB。这个环境使用了 `pvcTemplate`：

```bash
kubectl patch cluster.postgresql.cnpg.io keycloak-pg \
  -n keycloak --type=json \
  -p='[{"op":"replace","path":"/spec/storage/pvcTemplate/resources/requests/storage","value":"40Gi"}]'
```

由于原有 PVC 没有被自动更新，现场又分别扩容了 `pg-1` 和 `pg-2` 的 PVC：

```bash
kubectl patch pvc keycloak-pg-1 -n keycloak \
  --type=merge \
  -p='{"spec":{"resources":{"requests":{"storage":"40Gi"}}}}'

kubectl patch pvc keycloak-pg-2 -n keycloak \
  --type=merge \
  -p='{"spec":{"resources":{"requests":{"storage":"40Gi"}}}}'
```

PVC 一度显示：

```text
FileSystemResizePending
```

这表示 vSphere 已扩展底层卷，但文件系统需要在 Pod 重新挂载时完成扩容。

> 重要教训：删除 CNPG 当前主库 Pod 会被视为主库故障，可能立即触发 failover。生产环境扩容应先保证至少一个健康、已扩容且复制进度正确的副本，再处理当前主库。不要把“删除主库 Pod”当成普通重启。

本次事故中，CNPG 最终将扩容后的 `pg-1` 提升为主库，生成 Timeline 5，`pg-2` 随后作为 Timeline 5 的副本重新加入，Keycloak 恢复服务。

### 2. 不要手工删除 `pg_wal`

即使磁盘已经满，也不能进入 `pg_wal` 目录直接删除文件。这里的文件可能仍是崩溃恢复、复制或时间点恢复所必需的，手工删除很容易让实例彻底无法启动。

正确顺序是：

1. 通过扩容获得安全操作空间；
2. 恢复主库；
3. 处理失效副本及其复制槽；
4. 让 PostgreSQL 自己回收不再需要的 WAL。

## 重建已经分叉的副本

`pg-3` 仍处于 Timeline 3，而主库与健康副本都在 Timeline 5。重建前，先保留旧副本的逻辑备份：

```bash
kubectl exec -n keycloak keycloak-pg-3 -c postgres -- \
  pg_dump -Fc -d keycloak \
  > /home/ubuntu/keycloak-pg3-timeline3-20260828.dump
```

再用 `pg_restore -l` 验证归档可以正常解析。确认备份有效后，同时删除旧 Pod 和旧 PVC：

```bash
kubectl delete pvc/keycloak-pg-3 pod/keycloak-pg-3 -n keycloak
```

因为 Cluster 仍声明 `instances: 3`，CNPG 会自动创建新的加入任务：

```text
keycloak-pg-4-join-xxxxx
```

初始化完成后，正式副本为：

```text
keycloak-pg-4
```

CNPG 不复用已经删除的实例编号，因此新实例叫 `pg-4`，而不是再次叫 `pg-3`。

## 最终验证

重建完成后的状态：

```text
phase=Cluster in healthy state
currentPrimary=keycloak-pg-1
readyInstances=3

keycloak-pg-1  primary  timeline 5
keycloak-pg-2  replica  timeline 5
keycloak-pg-4  replica  timeline 5
```

复制槽也恢复正常：

```text
slot_name               active  restart_lsn  wal_status
_cnpg_keycloak_pg_2     true    1A5/D009118 reserved
_cnpg_keycloak_pg_4     true    1A5/D009118 reserved
```

旧的 `_cnpg_keycloak_pg_3` 已消失，两个新复制槽均处于活跃状态且进度一致。随后 `pg_wal` 占用下降，磁盘空间被正常回收。

## 为什么会发生第一次 Timeline 分叉

旧的 Operator 日志已经轮转，Kubernetes 审计日志也未启用，因此无法完整恢复 CNPG 当时的候选选择过程。但节点日志保留了关键证据。

在 Timeline 4 创建前，承载 `pg-1` 的 worker 节点出现了连续 CPU soft lockup：

```text
watchdog: BUG: soft lockup - CPU#0 stuck for 43s
watchdog: BUG: soft lockup - CPU#1 stuck for 22s
watchdog: BUG: soft lockup - CPU#2 stuck for 43s
watchdog: BUG: soft lockup - CPU#3 stuck for 43s
```

随后 RKE2 Agent 同时失去到两个控制节点的连接：

```text
Error syncing connections: i/o timeout
Remotedialer proxy error; reconnecting
```

连接约一分钟后恢复。CNPG 在这个窗口内开始将 `pg-2` 设为目标主库，并在约十分钟后完成提升、创建 Timeline 4。

调用栈显示多个 vCPU 同时卡在不同任务中，其中一个 CPU 等待跨 CPU 的 TLB 刷新。这不像单个 PostgreSQL 进程导致的高负载，更接近整台 VMware 虚拟机的 vCPU 长时间没有得到正常调度。

因此，目前能够确认的直接触发因素是：

```text
worker 节点发生整机级 CPU soft lockup
→ RKE2 控制连接中断
→ CNPG 判断实例不可用并触发故障切换
```

至于 soft lockup 最初由 ESXi CPU 争用、虚拟机 stun、快照/备份、vMotion，还是 Guest 内核问题引起，还需要结合 vCenter 同一时间段的 Tasks、Events 和性能曲线继续确认。这部分不能仅凭 Guest 日志武断下结论。

## 排查中发现的另一个配置问题

RKE2 的 etcd 快照计划配置为：

```yaml
etcd-snapshot-schedule-cron: "* */6 * * *"
```

它不是“每 6 小时一次”，而是“每到 0、6、12、18 点，在这一整小时内每分钟执行一次”。如果目标是每 6 小时执行一次，应写成：

```yaml
etcd-snapshot-schedule-cron: "0 */6 * * *"
```

这个错误配置与本次主库切换的时间窗口并不完全重合，因此不能把它定为本次事故的直接根因，但它会制造不必要的 etcd 快照和磁盘 I/O，应单独修正。

## 后续改进

### 1. 监控复制槽 WAL 滞留量

至少监控以下信息：

- `pg_replication_slots.active`；
- 当前 WAL LSN 与各槽 `restart_lsn` 的差值；
- PVC 使用率和剩余字节；
- 副本 Timeline、接收 LSN 和回放 LSN；
- CNPG Cluster phase 和 Ready condition。

复制槽不活跃且 WAL 滞留持续增长时，应尽早告警，而不是等 PVC 写满。

### 2. 设置 WAL 保留的最后防线

可以评估设置有限的 `max_slot_wal_keep_size`。达到上限后，PostgreSQL 会允许旧 WAL 被回收，代价是落后过多的副本复制槽失效、必须重建。

这是一个明确的取舍：

- 不限制：优先保证副本还能追赶，但可能写满主库磁盘；
- 设置上限：优先保护主库磁盘，但严重落后的副本需要重建。

具体值应根据写入速率、允许的副本中断时间和磁盘容量计算，不能机械照抄。

### 3. 重新评估同步复制策略

本集群当时使用异步复制。故障切换时，被选中的副本可能落后于另一个不可用副本，从而产生少量数据缺口和 Timeline 分叉。

如果业务要求更低的 RPO，可以评估同步复制；代价是网络或副本故障时可能影响写入可用性和延迟。

### 4. 保留足够的排障日志

本次 Operator 历史日志已经轮转，Kubernetes 审计日志也未启用，导致无法完整还原“为何选择该副本”的过程。生产环境应考虑：

- 将 CNPG Operator 和 PostgreSQL 日志集中采集；
- 设置足够的日志保留周期；
- 启用并妥善存储 Kubernetes 审计日志；
- 保留 vCenter Tasks、Events 和性能数据。

### 5. 修正容量与声明配置

- 将 Git 中的 Cluster YAML 同步更新为 40 GiB，避免下次 `kubectl apply` 造成配置漂移；
- 为数据库 PVC 设置分级容量告警；
- 定期验证备份不仅“成功”，而且能够实际读取和恢复；
- 规划 CloudNativePG 和 Kubernetes 的兼容升级，不让长期运行的旧版本失去维护窗口。

## 总结

这次事故的最终故障链如下：

```text
worker 虚拟机发生 CPU soft lockup
→ RKE2 控制连接短暂中断
→ CNPG 触发主库故障切换
→ 新主库从较早的 WAL 位置产生新 Timeline
→ pg-3 与新主库发生 Timeline 分叉
→ pg-3 的物理复制槽持续保留约 18 GiB WAL
→ 20 GiB PVC 被写满
→ CNPG 拒绝启动主库
→ Keycloak 数据库连接中断
```

恢复时最重要的原则是：**先扩容获得安全操作空间，不手工删除 WAL；恢复主库后，备份并重建已经分叉的副本，最后验证 Timeline、复制槽和磁盘空间。**

很多数据库事故表面上只是“磁盘满了”，但真正的根因往往藏在更早发生的节点故障、复制状态和故障切换历史里。
