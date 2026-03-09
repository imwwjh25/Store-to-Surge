### 一、双主复制一定会触发循环复制（核心原因）

双主复制（Master-Master）架构下，**默认配置必然会出现循环复制**，原因很简单：

- 双主架构中，A 和 B 互相作为对方的主库 / 从库；
- A 执行的事务会写入自己的 binlog，B 拉取 A 的 binlog 重放后，会把这个事务写入 B 的 binlog；
- 若不做限制，A 又会拉取 B 的 binlog，重放这个「自己原本执行的事务」，形成无限循环（A→B→A→B…）。

举个通俗例子：

1. A 执行 `INSERT INTO t1 VALUES (1)`，写入 A 的 binlog；
2. B 拉取 A 的 binlog 并执行，同时把这个 INSERT 写入 B 的 binlog；
3. A 拉取 B 的 binlog，发现这个 INSERT 事务，再次执行 → 循环开始，直到数据库资源耗尽。

### 二、避免循环复制的核心方案（MySQL 官方推荐）

核心思路是：**让每个节点识别「自己产生的事务」，并拒绝重放**，实现方式是通过 `server_id` + `binlog_ignore_db/binlog_do_db`（基础版）或 `log_slave_updates` + 事务标记（标准版），最通用、可靠的是以下两步配置：

#### 步骤 1：给两个主库设置「唯一的 server_id」

`server_id` 是 MySQL 实例的唯一标识（整数），是避免循环复制的基础：

- 主库 A：`server_id = 1`
- 主库 B：`server_id = 2`

**原理**：每个事务在 binlog 中会记录「产生该事务的 server_id」，当从库拉取 binlog 时，会检查事务的 `server_id`：如果和自己的 `server_id` 相同，说明是「自己产生的事务」，直接跳过重放。

#### 步骤 2：开启 log_slave_updates（保证事务标记完整）

`log_slave_updates = ON`（双主架构必须开启），作用是：

- 从库重放主库的 binlog 时，会把这些事务写入自己的 binlog；
- 但结合 `server_id` 检查，这些事务的 `server_id` 是「原主库的 ID」，不会被自身重放。

#### 完整配置示例（双主 A 和 B）

##### 主库 A（server_id=1）的 my.cnf/my.ini 配置：










```
[mysqld]
# 唯一server_id
server_id = 1
# 开启binlog（主从复制必备）
log_bin = mysql-bin
binlog_format = ROW  # 推荐行格式，避免逻辑复制的坑
# 开启从库重放事务写入binlog（双主必备）
log_slave_updates = ON
# 自动跳过复制错误（可选，避免小错误导致复制中断）
slave_skip_errors = 1032,1062
# 关闭二进制日志自动清理（可选，生产建议手动管理）
expire_logs_days = 7
```

##### 主库 B（server_id=2）的 my.cnf/my.ini 配置：








```
[mysqld]
server_id = 2  # 必须和A不同
log_bin = mysql-bin
binlog_format = ROW
log_slave_updates = ON
slave_skip_errors = 1032,1062
expire_logs_days = 7
```

### 三、循环复制的完整防护逻辑（以 A→B→A 为例）

1. A 执行事务 T1 → binlog 中记录 `server_id=1`；
2. B 作为 A 的从库，拉取 T1 并执行 → 因 `log_slave_updates=ON`，T1 写入 B 的 binlog，但 `server_id` 仍为 1（不会变成 B 的 2）；
3. A 作为 B 的从库，拉取 B 的 binlog 时，发现 T1 的 `server_id=1`（和自己的 `server_id` 相同）→ 直接跳过重放，循环终止。

### 四、额外防护措施（进阶优化）

#### 1. 限制复制的数据库 / 表（减少无效复制）

如果双主只同步特定库，可通过以下参数限定，进一步降低循环风险：





```
# 主库A：只同步 db_test 库，忽略其他库
binlog_do_db = db_test
# 或 忽略不需要同步的库
binlog_ignore_db = mysql  # 系统库无需同步
binlog_ignore_db = information_schema
```

#### 2. 避免双写冲突（业务层面防护）

循环复制是「事务重放循环」，而双主架构还可能出现「写冲突」（比如 A 和 B 同时修改同一行），需业务层配合：

- 分库分表：A 负责写入 db1，B 负责写入 db2，互相同步但不冲突；
- 加分布式锁：比如用 Redis 锁，保证同一行数据只有一个主库能修改；
- 设定「主主备」模式：日常只写 A，B 作为热备，仅故障时切换写 B（本质是伪双主，避免冲突）。

#### 3. 监控循环复制（及时发现问题）

通过 MySQL 自带命令监控复制状态，发现异常及时处理：









```
-- 查看从库复制状态（A 上执行，查看 B 的复制；B 上执行，查看 A 的复制）
SHOW SLAVE STATUS\G;

-- 关键指标：
-- Slave_IO_Running: Yes （IO线程正常）
-- Slave_SQL_Running: Yes （SQL线程正常）
-- Seconds_Behind_Master: 0 （无延迟）
-- Last_SQL_Error: 空 （无复制错误）
```

### 五、常见坑点（避坑指南）

1. **server_id 重复**：如果 A 和 B 的 server_id 相同，不仅无法避免循环复制，还会导致复制直接中断（报错 `server_id clash`）；
2. **未开启 log_slave_updates**：双主架构中，若某节点关闭该参数，会导致事务无法在另一个节点的 binlog 中标记原 server_id，仍会触发循环；
3. **binlog_format 不一致**：比如 A 用 STATEMENT，B 用 ROW，可能导致事务解析错误，间接引发异常循环。

### 总结

1. **核心结论**：双主复制默认会触发循环复制，必须通过 `server_id` 唯一标识 + `log_slave_updates=ON` 来避免；
2. **防护原理**：事务在 binlog 中记录产生节点的 `server_id`，节点遇到「自己 server_id 的事务」时跳过重放；
3. **额外保障**：业务层避免双写冲突、监控复制状态、限定同步库表，进一步提升架构稳定性。
