## 如何搭建一台高性能的MySQL服务器

**为了让 MySQL 在服务器上能以高性能、稳定运行；需要对数据库配置、操作系统、硬件以及BIOS 层面进行必不可少的优化。**

- *下面我以此配置的服务器为例：总结一些常用的优化项目*

|    服务器     |     OS     | CPU  | 内存 | 磁盘 |
| :-----------: | :--------: | :--: | :--: | :--: |
| 192.168.1.100 | RedHat 8.0 |  4   | 16G  | 320G |

---

### 一. 硬件层面优化

1. **CPU 和内存配置**：
   - **性能模式**：在 BIOS 中，将 CPU 和内存设置为“最大性能模式”以充分利用硬件资源。
   - **关闭 NUMA**：禁用 NUMA 避免非均匀内存访问带来的性能瓶颈。
   - **关闭 CPU 节能模式**：禁用节能模式，确保 CPU 一直运行在高性能模式。

2. **SSD 优化**：
   - **选择适合 SSD 的 IO Scheduler**：将 SSD 设备的 IO Scheduler 设置为 `noop`，优化磁盘的读写性能。
     
     ```bash
     echo noop > /sys/block/sdX/queue/scheduler
     ```

---

### 二. BIOS 优化

1. **关闭 NUMA**：禁用 NUMA（非均匀内存访问）模式，确保均匀的内存访问。
2. **性能模式设置**：将 CPU 和内存的性能调至最大，以提供持续高性能。
3. **关闭 CPU 节能模式**：禁用节能功能，避免在频繁切换 CPU 频率时产生延迟。

---

### 三. 操作系统优化

1. **文件系统优化**：
   - 使用 **XFS 文件系统**，并启用挂载选项 `noatime`、`nodiratime` 和 `nobarrier`，以减少不必要的 IO 操作。
     ```bash
     mount -o noatime,nodiratime,nobarrier /dev/sdX /data
     ```

2. **IO Scheduler 优化**：
   - 对 SSD 设备使用 `noop`，减少额外的 IO 排队负载：
     ```bash
     echo noop > /sys/block/sdX/queue/scheduler
     ```

3. **内核参数优化**：
   - 调整以下内核参数以优化内存和网络性能：
     ```bash
     vm.swappiness = 5
     vm.dirty_ratio = 10
     vm.dirty_background_ratio = 5
     fs.file-max = 65536
     net.core.somaxconn = 65536
     net.core.netdev_max_backlog = 65536
     net.ipv4.tcp_max_syn_backlog = 65536
     net.ipv4.tcp_fin_timeout = 10
     net.ipv4.tcp_tw_reuse = 0
     net.ipv4.tcp_tw_recycle = 0
     ```

4. **文件句柄数和线程限制**：
   - 在 `/etc/security/limits.conf` 中增加文件句柄数和线程数的限制：
     ```bash
     * - nofile 65535
     * - nproc  65535
     ```

---

### 四. 网络配置优化

1. **TCP/IP 参数优化**：
   - 调整 TCP 连接队列和超时参数，确保高并发连接情况下的网络稳定性：
     ```bash
     net.core.somaxconn = 65536
     net.core.netdev_max_backlog = 65536
     net.ipv4.tcp_max_syn_backlog = 65536
     net.ipv4.tcp_fin_timeout = 10
     ```

2. **禁用 DNS 解析**：
   
   - 禁用 MySQL 的 DNS 解析以提升连接速度：
     ```ini
     skip_name_resolve = ON
     ```

---

### 五. 磁盘与存储优化

1. **SSD 使用优化**：
   
   - 将 **IO Scheduler** 设置为 `noop`，适合 SSD：
     ```bash
     echo noop > /sys/block/sdX/queue/scheduler
     ```
   
2. **文件系统选择**：
   
   - 建议使用 **XFS 文件系统**，并挂载时使用 `noatime` 和 `nobarrier` 参数，减少写操作的额外负载。

---

### 六. MySQL 配置优化

**建议使用二进制包安装 MySQL，以避免系统自带包的限制。**

针对 MySQL 的特定配置文件（`my.cnf` ），进行一系列调整，以提升数据库的性能和稳定性。

```ini
[client]
port = 3306
socket = /tmp/mysql.sock

[mysqld]
# -------------------- 基本配置 --------------------
user = mysql
port = 3306
socket = /tmp/mysql.sock
pid-file = /data/mysqldata/mysql.pid

server-id = 1                       # ServerID
gtid_mode = ON                      # 启用GTID
enforce-gtid-consistency = ON       # 强制GTID一致性
log-bin = mysql-bin                 # 启用二进制日志
binlog_format = ROW                 # 二进制日志格式
binlog_row_image = FULL             # 行级日志模式
log_timestamps = system             # 日志时间以系统时间
default_storage_engine = InnoDB     # 存储引擎默认InnoDB
sql_require_primary_key = ON        # 表必须有主键

# 数据存储目录
datadir = /data/mysql/

# 错误日志
log_error = /data/log/mysql-error.log

# 二进制日志和慢查询日志
log_bin = /data/binlog/mysql/mysql-bin
slow_query_log_file = /data/slowlog/mysql/mysql-slow.log

# 临时文件目录
tmpdir = /var/tmp/mysql

# 独立表空间，便于表管理
innodb_file_per_table = 1

# -------------------- 内存与缓存优化 --------------------
# 调整 InnoDB 缓冲池大小，建议占用物理内存的 50%-75%
innodb_buffer_pool_size = 8G
innodb_buffer_pool_instances = 8

# 调整排序和连接缓存大小
sort_buffer_size = 4M
join_buffer_size = 4M
read_buffer_size = 8M
read_rnd_buffer_size = 4M

# 临时表和内存表的大小
tmp_table_size = 32M
max_heap_table_size = 32M

# -------------------- InnoDB 日志与事务配置 --------------------
# 使用较大的 InnoDB 日志文件，减少频繁刷盘
innodb_log_file_size = 1G
innodb_log_files_in_group = 3

# 保证主库和从库数据一致性
sync_binlog = 1
innodb_flush_log_at_trx_commit = 1

# 最大脏页百分比，减少系统写盘压力
innodb_max_dirty_pages_pct = 50

# 调整事务的锁等待时间
innodb_lock_wait_timeout = 10

# -------------------- 线程和 IO 优化 --------------------
# 读写线程优化，提高并发处理能力
innodb_read_io_threads = 4
innodb_write_io_threads = 4

# 自适应刷新，提高写入性能
innodb_adaptive_flushing = 1

# 自适应哈希索引，加速频繁查询的索引查找
innodb_adaptive_hash_index = 1

# 调整 Innodb 日志缓冲区大小，适合大量写操作
innodb_log_buffer_size = 16M

# -------------------- 性能与监控 --------------------
# 开启慢查询日志，记录执行超过 2 秒的查询
slow_query_log = 1
long_query_time = 2

# 记录未使用索引的查询，便于优化
log_queries_not_using_indexes = 1
log_throttle_queries_not_using_indexes = 60

# 开启 InnoDB 锁状态监控
innodb_status_output_locks = ON

# 最大连接数和连接超时
max_connections = 1000
wait_timeout = 600
interactive_timeout = 600

# 禁用 DNS 反向解析，加速连接建立
skip_name_resolve = ON

# -------------------- 文件系统与性能优化 --------------------
# 使用大页面，提升内存利用率
innodb_large_prefix = 1

# 查询缓存配置，适合重复查询的场景
query_cache_type = 1
query_cache_size = 128M

# 表和定义缓存，提升表打开的效率
table_open_cache = 4096
table_definition_cache = 2048

# -------------------- 备份与恢复 --------------------
# 设置二进制日志保留时间为 7 天
# expire_logs_days = 7
binlog_expire_logs_seconds=604800

```

## 总结

MySQL 的参数有很多，以上只是举例说明了一部分参数的调整。

- **MySQL 5.x 版本**：大约有 450-500 个可配置的参数。

- **MySQL 8.x 版本**：增加到了大约 600 个左右，

- 随着新功能和性能改进的引入，MySQL 8.x 会增加了更多的配置选项和系统变量。

**在实际环境中，你需要根据你的服务器环境，业务模式，使用场景；对参数进行更加细致的调整，这篇文章只能先简单总结一些常用的参数调证。**



