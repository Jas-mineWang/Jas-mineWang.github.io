---
layout: post
title: "一次 JumpServer MySQL 双向复制失效：从 1062 冲突到只读备库重建"
date: 2026-09-21 11:30:00 +0800
categories: [jumpserver, mysql, replication, incident-response]
---

这篇文章记录一次 JumpServer 后端 MySQL 复制故障的完整排查和修复过程。

故障表面上只是备库的 SQL 线程因 `1062 Duplicate entry` 停止，但继续追查后发现，现场实际上是一套“两边都可写、非 GTID、循环复制”的双主架构，而备库已经两个多月没有成功应用主库变更。最终没有选择跳过单个错误，而是以当前主库为唯一权威数据源，完整备份后重建备库，并改造为单向复制。

> 文中的内网地址、主机名、业务域名、凭据和完整哈希均已省略或泛化。所有破坏性操作都应在确认目标主机、完成备份并进入变更窗口后执行。

## 环境与拓扑

- JumpServer：企业版 3.10.x
- 部署方式：官方离线安装包 + Docker Compose
- 应用节点：2 台，每台都运行 Core、Celery、Web、Koko、Lion、Chen、Magnus 和 Razor
- MySQL：8.0.x，2 台
- Redis：主从复制，与 MySQL 部署在同一组数据库节点

排查前理解的拓扑如下：

```text
用户
  │
  └─ 直接访问 jump-app-0
          │
          ├─ jump-app-0：完整 JumpServer 组件
          └─ jump-app-1：完整 JumpServer 组件
                    │
                    └─ 数据库地址 db-endpoint
                              │
                              ├─ db-0：MySQL/Redis 主节点
                              └─ db-1：MySQL/Redis 备节点
```

后来进一步确认，`db-endpoint` 并不是可漂移的 VIP，而是 `db-0` 自身的另一个地址。因此数据库当时并没有自动故障切换能力。

## 先从应用节点还原部署方式

只有 JumpServer 节点的登录权限，没有数据库主机的系统账号。所以第一步不是猜测架构，而是检查容器标签和挂载。

```bash
docker ps -a

docker inspect jms_core --format '
workdir={{index .Config.Labels "com.docker.compose.project.working_dir"}}
files={{index .Config.Labels "com.docker.compose.project.config_files"}}
project={{index .Config.Labels "com.docker.compose.project"}}'

docker inspect jms_core --format \
  '{{range .Mounts}}{{println .Source " -> " .Destination}}{{end}}'
```

输出显示，两台应用节点都来自同版本的 JumpServer 离线安装包，使用名为 `jms` 的 Compose 项目，持久化数据挂载在宿主机 `/data/jumpserver` 下。

只读取非敏感配置：

```bash
grep -E \
'^(DB_HOST|DB_PORT|DB_USER|DB_NAME|REDIS_HOST|REDIS_PORT|HTTP_PORT|HTTPS_PORT)=' \
/opt/jumpserver/config/config.txt
```

两台应用节点指向同一个 MySQL/Redis 地址。此处不应打印 `DB_PASSWORD`、`REDIS_PASSWORD`、`SECRET_KEY` 或 `BOOTSTRAP_TOKEN`。

## Redis 正常，MySQL 异常

通过 JumpServer Core 容器中现有的 Redis 配置执行只读查询，确认：

```text
db-0：Redis master
db-1：Redis replica
connected_slaves=1
lag=0
```

Redis 主从当时没有异常。

再通过 JumpServer 已配置的 MySQL 连接查询两台数据库：

```sql
SELECT @@hostname,
       @@version,
       @@read_only,
       @@super_read_only,
       @@server_id,
       @@global.gtid_mode,
       @@log_bin,
       @@log_slave_updates,
       @@binlog_format,
       @@auto_increment_increment,
       @@auto_increment_offset;

SHOW REPLICA STATUS\G
```

结果暴露了真实的 MySQL 拓扑：

```text
db-0 ←→ db-1
```

关键配置为：

- `server_id` 分别为 1 和 2；
- 两边都开启 binlog；
- `binlog_format=ROW`；
- 两边都开启 `log_slave_updates`；
- `gtid_mode=OFF`；
- 两边的 `read_only` 和 `super_read_only` 都是 0；
- 两边的自增步长和偏移量都是 1。

这不是一套严格的“单写主库 + 只读备库”，而是两边都可以写入的非 GTID 循环复制。

## 备库复制已经停止两个多月

`db-0` 从 `db-1` 复制时，IO 和 SQL 线程都正常。

`db-1` 从 `db-0` 复制时，IO 线程仍在接收 binlog，但 SQL 线程已经停止：

```text
Replica_IO_Running: Yes
Replica_SQL_Running: No
Seconds_Behind_Source: NULL
```

Performance Schema 中的真实错误是：

```text
Error 1062: Duplicate entry '<task-id>' for key 'PRIMARY'
table: jumpserver.ops_celerytaskexecution
```

错误时间显示，复制早在两个多月前就已停止。

查看每个库的体积后，问题更明显：

```text
db-0：jumpserver 约 2.7 GiB
db-1：jumpserver 约 58 MiB
```

两边都有 164 张表，但数据量相差近 48 倍。这不再是“跳过一个错误就能恢复”的状态。

## 为什么没有直接跳过 1062

看到重复主键后，最诱人的做法是设置 `sql_replica_skip_counter` 并重启 SQL 线程。这里没有这么做，原因有三个：

1. 复制已中断两个多月，后面可能还有更多冲突；
2. 两库都可写，不能证明冲突记录中哪份数据正确；
3. 库体积差距已经证明备库不值得做逐条修补。

对普通 InnoDB 复制来说，无区别忽略复制错误会让主备更加失同。现场因此选择了更可验证的路径：

```text
确定唯一权威主库
→ 两边都备份
→ 停止循环复制
→ 从主库一致性快照重建备库
→ 改为单向复制
→ 持久化只读保护
```

## 修复前的安全检查

### 1. 确认备份条件

先统计表数、容量和存储引擎：

```sql
SELECT COUNT(*) AS table_count,
       ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size_mb,
       SUM(CASE WHEN engine <> 'InnoDB' THEN 1 ELSE 0 END) AS non_innodb_tables
FROM information_schema.tables
WHERE table_schema = 'jumpserver';
```

所有表都为 InnoDB，可以使用 `--single-transaction` 获得一致性逻辑备份，不需要长时间锁表。

现场的 JumpServer Core 容器自带 `mysqldump`，但是 MariaDB 客户端。在导出正式数据前，先做一次只包含表结构的兼容性测试：

```bash
mysqldump \
  --host="$PRIMARY_HOST" \
  --port=3306 \
  --user="$DB_ADMIN_USER" \
  --single-transaction \
  --quick \
  --routines \
  --events \
  --triggers \
  --hex-blob \
  --no-data \
  jumpserver >/dev/null
```

### 2. 同时保留主库和旧备库

主库备份使用了：

```bash
mysqldump \
  --host="$PRIMARY_HOST" \
  --port=3306 \
  --user="$DB_ADMIN_USER" \
  --single-transaction \
  --quick \
  --routines \
  --events \
  --triggers \
  --hex-blob \
  --master-data=2 \
  --databases jumpserver |
gzip -1 >jumpserver-primary.sql.gz
```

`--master-data=2` 会把非 GTID 复制所需的 binlog 文件和位置以注释形式写入备份。然后使用 `gzip -t` 和 SHA-256 验证文件：

```bash
gzip -t jumpserver-primary.sql.gz
sha256sum jumpserver-primary.sql.gz
```

旧备库也做了一份单独备份。虽然它已经失同，但其中可能存在未同步回主库的独有数据，不能在没有留底的情况下直接覆盖。

两份备份又复制到另一台 JumpServer 节点，再次校验哈希。

### 3. 提取并验证 binlog 坐标

```bash
zgrep -m1 -E 'CHANGE (MASTER|REPLICATION SOURCE) TO' \
  jumpserver-primary.sql.gz
```

备份中记录的坐标形如：

```text
-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000023', MASTER_LOG_POS=<position>;
```

然后在主库上确认目标 binlog 仍存在：

```sql
SHOW BINARY LOGS;
SHOW MASTER STATUS;

SELECT @@global.binlog_expire_logs_seconds,
       @@global.max_binlog_size;
```

现场 binlog 保留期为 30 天，重建期间不会因自动清理丢失起点。

## 停止循环复制

操作顺序非常重要：必须先停止主库从旧备库接收数据的通道，否则后续在备库上的 `DROP DATABASE` 和导入操作可能反向复制到主库。

先在两台数据库上执行：

```sql
STOP REPLICA;
```

再确认：

```text
Replica_IO_Running: No
Replica_SQL_Running: No
```

此时不急着在主库上执行 `RESET REPLICA ALL`，先保留旧连接元数据，等新的单向复制验证成功后再彻底删除反向通道。

## 用主库备份重建备库

在备库上，先清除旧的 relay log 和执行位置，但不使用 `ALL`：

```sql
STOP REPLICA;
RESET REPLICA;
```

`RESET REPLICA` 会保留已有的源站地址、端口、复制用户和密码；`RESET REPLICA ALL` 则会把这些连接参数一并清除。

导入前要再次校验目标主机，避免误操作主库：

```sql
SELECT @@hostname;
```

确认为 `db-1` 后，在同一个导入会话中关闭 binlog，删除旧库并导入主库备份：

```bash
{
  printf 'SET SESSION sql_log_bin=0;\n'
  printf 'DROP DATABASE IF EXISTS `jumpserver`;\n'
  gzip -dc jumpserver-primary.sql.gz
} | mysql \
      --host="$REPLICA_HOST" \
      --port=3306 \
      --user="$DB_ADMIN_USER" \
      --default-character-set=utf8mb4
```

关闭当前会话的 binlog，可以避免整次导入在旧备库上产生巨量无用 binlog。该操作需要相应的系统权限。

导入成功后，立即开启只读保护：

```sql
SET GLOBAL read_only=ON;
SET GLOBAL super_read_only=ON;
```

## 按备份坐标启动单向复制

因为现场没有使用 GTID，需要显式指定备份中的 binlog 坐标：

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_LOG_FILE='mysql-bin.000023',
  SOURCE_LOG_POS=<position>;

START REPLICA;
```

验证：

```sql
SHOW REPLICA STATUS\G
```

关键结果为：

```text
Source_Host: db-0
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
Seconds_Behind_Source: 0
Last_IO_Error:
Last_SQL_Error:
```

当新的 `db-0 → db-1` 复制已稳定追平后，在主库上删除旧的反向复制元数据：

```sql
STOP REPLICA;
RESET REPLICA ALL;
```

此操作只用于主库上的旧反向通道，不会删除业务数据或主库 binlog。

## 把备库只读状态持久化

只执行 `SET GLOBAL` 不够，MySQL 重启后配置会丢失。在备库上使用：

```sql
SET PERSIST read_only=ON;
SET PERSIST super_read_only=ON;
```

再验证：

```sql
SELECT @@read_only,
       @@super_read_only,
       @@skip_replica_start;

SELECT VARIABLE_NAME, VARIABLE_VALUE
FROM performance_schema.persisted_variables
WHERE VARIABLE_NAME IN ('read_only', 'super_read_only');
```

预期结果：

```text
read_only=1
super_read_only=1
skip_replica_start=0
```

`skip_replica_start=0` 表示 MySQL 重启后会自动启动复制线程。

## 修复后的架构

数据库从不受控的循环双写，调整为单写主从：

```text
JumpServer
    │
    └─ db-0：唯一可写主库
           │
           └─ 异步复制 → db-1：持久只读备库
```

最终状态：

```text
db-0:
  SHOW REPLICA STATUS -> empty
  read_only=0
  super_read_only=0

db-1:
  Replica_IO_Running=Yes
  Replica_SQL_Running=Yes
  Seconds_Behind_Source=0
  Last_IO_Error=''
  Last_SQL_Error=''
  read_only=1
  super_read_only=1
```

## 修复中最重要的顺序

这次操作中，最危险的不是导入速度，而是旧的双向复制。安全顺序应当是：

```text
1. 确认唯一权威主库
2. 同时备份主库和旧备库
3. 校验备份并复制到第二台主机
4. 记录 binlog 坐标并确认文件未过期
5. 先停止主库的反向复制
6. 再停止备库的正向复制
7. 只重建经过主机名校验的备库
8. 从备份坐标恢复单向复制
9. 确认追平后才删除主库的旧反向元数据
10. 持久化备库只读保护
```

如果在第 5 步之前就在备库执行建库、删库和导入，这些操作可能通过旧的循环复制回到主库，后果远比一个失效备库严重。

## 仍未解决的高可用问题

数据复制修复并不等于完成高可用改造。现场仍存在两个明显缺口：

1. JumpServer 配置的数据库地址实际上是 `db-0` 的固定地址，不是 VIP；
2. 用户直接访问第一台 JumpServer，两台应用节点前没有自动切换入口。

因此，当 `db-0` 故障时，仍需要人工确认备库进度、提升 `db-1`、修改应用数据库地址。应用节点故障时也需要人工切换访问入口。

后续可以评估：

- 数据库故障切换工具或受控的 VIP/ProxySQL/HAProxy 入口；
- 明确的主库提升和回切手册；
- JumpServer 前端负载均衡或漂移入口；
- MySQL 复制线程、延迟、错误时间和备库只读状态告警；
- 定期恢复演练，不只检查“备份任务成功”。

## 额外发现：业务账号权限过大

排查过程中还发现，JumpServer 的数据库账号被授予了接近全局管理员的权限，并允许从 `%` 登录。这虽然让现场能在没有数据库 OS 账号的情况下完成修复，但对业务系统来说是明显的越权风险。

更合理的方式是：

- JumpServer 应用账号只保留业务库所需的 DML/DDL 权限；
- 备份使用独立的备份账号；
- 复制使用独立的 replication 账号；
- 系统变量、复制管理和用户管理由受控的 DBA 账号执行；
- 用主机网段或明确主机限制登录来源，不要长期使用 `%`。

## 总结

这次故障的核心链路是：

```text
MySQL 两边都可写
→ 双向循环复制中出现同一主键
→ db-1 SQL 复制线程因 1062 停止
→ IO 线程仍持续接收 binlog
→ 备库长期无法应用变更
→ 主备数据量差距扩大
→ 备库已不具备可靠切换条件
```

修复中最重要的不是某条 SQL，而是三个原则：

1. **先确定唯一权威数据源，不在两个已分叉的库之间猜测。**
2. **先备份两边、再停止循环复制，最后才重建备库。**
3. **修复后把备库的只读状态持久化，用架构防止同类冲突再次发生。**

看似简单的 `1062 Duplicate entry`，背后往往不只是一条重复数据，而是主备写入边界、故障切换流程和监控机制同时缺失的信号。

## 参考资料

- [MySQL 8.0 Reference Manual: SHOW REPLICA STATUS](https://dev.mysql.com/doc/refman/8.0/en/show-replica-status.html)
- [MySQL 8.0 Reference Manual: RESET REPLICA](https://dev.mysql.com/doc/refman/8.0/en/reset-replica.html)
- [MySQL 8.0 Reference Manual: Replica Server Options and Variables](https://dev.mysql.com/doc/refman/8.0/en/replication-options-replica.html)
- [MySQL 8.0 Reference Manual: mysqldump](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html)
