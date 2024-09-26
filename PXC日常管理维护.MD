## MySQL PXC 集群日常运维管理指南

为了确保 MySQL Percona XtraDB Cluster (PXC) 集群的高可用性和稳定性，日常运维需要执行一系列管理和监控任务。以下是常用的日常运维管理命令和操作，帮助您有效地维护和管理 PXC 集群。

1. [集群状态监控](#1-集群状态监控)
2. [节点管理](#2-节点管理)
3. [备份与恢复](#3-备份与恢复)
4. [用户与权限管理](#4-用户与权限管理)
5. [日志管理](#5-日志管理)
6. [性能监控与调优](#6-性能监控与调优)
7. [安全性管理](#7-安全性管理)
8. [日常维护脚本](#8-日常维护脚本)
9. [故障排查](#9-故障排查)
10. [计划任务配置](#10-计划任务配置)

---

### 1. 集群状态监控

定期监控集群状态可以及时发现并解决潜在问题，确保集群的稳定运行。

#### 1.1 查看集群状态

```sql
-- 在任意节点上执行
SHOW STATUS LIKE 'wsrep%';
```

**常用关键参数解释：**
- `wsrep_cluster_size`: 当前集群中的节点数量。
- `wsrep_cluster_status`: 集群状态，通常为 `Primary`。
- `wsrep_ready`: 节点是否准备好接收请求，`ON` 表示正常。
- `wsrep_connected`: 节点是否连接到集群，`ON` 表示正常。

#### 1.2 检查节点同步状态

```sql
SHOW STATUS LIKE 'wsrep_local_state_comment';
```

**状态解释：**
- `Synced`: 节点已与集群同步。
- `Joining`: 节点正在加入集群。
- `Donor/Desynced`: 节点正在为其他节点提供 SST（State Snapshot Transfer）。

#### 1.3 使用 `mysqladmin` 检查服务器状态

```bash
mysqladmin -u root -p status
```

#### 1.4 查看集群成员列表

```sql
SELECT * FROM performance_schema.replication_group_members;
```

#### 1.5 使用 `clustercheck` 工具监控

安装并配置 `clustercheck` 工具，可以通过 HTTP 接口监控集群状态，常用于负载均衡器的健康检查。

```bash
# 安装 clustercheck
sudo yum install -y percona-clustercheck

# 编辑配置文件
sudo vi /etc/sysconfig/clustercheck

# 启动 clustercheck 服务
sudo systemctl start clustercheck
sudo systemctl enable clustercheck
```

**通过浏览器或命令行访问：**
```bash
curl http://localhost:9200/
```

**返回结果：**
- `HTTP 200 OK`: 节点正常。
- `HTTP 503 Service Unavailable`: 节点不可用。

---

### 2. 节点管理

包括添加新节点、删除节点、启动和停止节点等操作。

#### 2.1 添加新节点

**步骤：**
1. **配置新节点的 `my.cnf` 文件：**
   ```ini
   [mysqld]
   ...
   wsrep_cluster_address = gcomm://192.168.100.111,192.168.100.112,192.168.100.113
   wsrep_node_address = 192.168.100.114
   wsrep_sst_method = xtrabackup-v2
   ...
   ```
2. **启动新节点：**
   ```bash
   sudo systemctl start mysql
   ```
3. **验证新节点状态：**
   ```sql
   SHOW STATUS LIKE 'wsrep_cluster_size';
   ```
   **应当看到集群大小增加了1。**

#### 2.2 删除节点

**步骤：**
1. **停止要删除的节点：**
   ```bash
   sudo systemctl stop mysql
   ```
2. **从负载均衡器中移除该节点的IP。**
3. **更新其他节点的 `my.cnf` 文件，移除被删除节点的 IP（可选）。**

#### 2.3 安全停止节点

在维护或重启前，安全停止节点以确保数据一致性。

```bash
mysql -u root -p -e "SET GLOBAL wsrep_on='OFF';"
sudo systemctl stop mysql
```

#### 2.4 重启节点

```bash
sudo systemctl restart mysql
```

**注意：** 如果整个集群都已停止，需要使用以下命令在一个节点上启动集群：

```bash
sudo /usr/local/mysql/bin/mysqld --wsrep-new-cluster
```

然后启动其他节点。

---

### 3. 备份与恢复

定期备份数据库可以防止数据丢失，使用 Percona XtraBackup 工具可以实现热备份。

#### 3.1 全量备份

**安装 Percona XtraBackup：**
```bash
sudo yum install -y percona-xtrabackup-80
```

**执行全量备份：**
```bash
xtrabackup --backup --target-dir=/backup/full/ --user=backupuser --password=yourpassword
```

**备份后准备：**
```bash
xtrabackup --prepare --target-dir=/backup/full/
```

#### 3.2 增量备份

**执行增量备份：**
```bash
xtrabackup --backup --target-dir=/backup/inc1/ --incremental-basedir=/backup/full/ --user=backupuser --password=yourpassword
```

**增量备份准备：**
```bash
xtrabackup --prepare --target-dir=/backup/full/ --incremental-dir=/backup/inc1/
```

#### 3.3 恢复备份

**停止 MySQL 服务：**
```bash
sudo systemctl stop mysql
```

**恢复数据：**
```bash
sudo rm -rf /data/mysqldata/*
sudo xtrabackup --copy-back --target-dir=/backup/full/
sudo chown -R mysql:mysql /data/mysqldata/
```

**启动 MySQL 服务：**
```bash
sudo systemctl start mysql
```

#### 3.4 备份脚本示例

```bash
#!/bin/bash
BACKUP_DIR=/backup/$(date +%F)
mkdir -p $BACKUP_DIR

xtrabackup --backup --target-dir=$BACKUP_DIR --user=backupuser --password=yourpassword

# 删除7天前的备份
find /backup/ -type d -mtime +7 -exec rm -rf {} \;
```

**设置定时任务：**
```bash
crontab -e
```
添加以下行，每天凌晨2点执行备份：
```cron
0 2 * * * /path/to/backup_script.sh >> /var/log/mysql_backup.log 2>&1
```

---

### 4. 用户与权限管理

管理数据库用户和权限，确保数据库安全和正常访问。

#### 4.1 创建新用户

```sql
CREATE USER 'username'@'host' IDENTIFIED BY 'password';
```

**示例：**
```sql
CREATE USER 'appuser'@'%' IDENTIFIED BY 'securepassword';
```

#### 4.2 授予权限

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON database_name.* TO 'username'@'host';
```

**授予所有权限：**
```sql
GRANT ALL PRIVILEGES ON database_name.* TO 'username'@'host';
```

**刷新权限：**
```sql
FLUSH PRIVILEGES;
```

#### 4.3 查看用户权限

```sql
SHOW GRANTS FOR 'username'@'host';
```

#### 4.4 修改用户密码

```sql
ALTER USER 'username'@'host' IDENTIFIED BY 'newpassword';
```

#### 4.5 删除用户

```sql
DROP USER 'username'@'host';
```

---

### 5. 日志管理

查看和管理 MySQL 日志，以监控数据库活动和排查问题。

#### 5.1 查看错误日志

**默认错误日志路径：**
- `/var/log/mysqld.log` 或 `/var/log/mysql/error.log`

**查看错误日志：**
```bash
tail -f /var/log/mysql/error.log
```

#### 5.2 查看慢查询日志

**启用慢查询日志：**
```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
```

**查看慢查询日志：**
```bash
tail -f /var/log/mysql/slow.log
```

#### 5.3 使用 `pt-query-digest` 分析慢查询

**安装 Percona Toolkit：**
```bash
sudo yum install -y percona-toolkit
```

**分析慢查询日志：**
```bash
pt-query-digest /var/log/mysql/slow.log > /tmp/slow_report.txt
```

**查看分析报告：**
```bash
less /tmp/slow_report.txt
```

---

### 6. 性能监控与调优

监控数据库性能，识别瓶颈并进行优化。

#### 6.1 使用 `mysqltuner` 进行优化建议

**安装 mysqltuner：**
```bash
sudo yum install -y mysqltuner
```

**运行 mysqltuner：**
```bash
mysqltuner
```

**根据报告调整 MySQL 配置。**

#### 6.2 监控 InnoDB 引擎状态

```sql
SHOW ENGINE INNODB STATUS\G
```

#### 6.3 检查当前连接数

```sql
SHOW STATUS LIKE 'Threads_connected';
```

#### 6.4 检查缓存命中率

```sql
SHOW STATUS LIKE 'Qcache_hits';
SHOW STATUS LIKE 'Qcache_inserts';
```

#### 6.5 监控系统资源

**CPU 和内存使用情况：**
```bash
top
```

**磁盘 I/O 状况：**
```bash
iostat -x 1
```

**网络流量：**
```bash
iftop
```

#### 6.6 使用 Percona Monitoring and Management (PMM)

PMM 是一个开源的监控和管理工具，可用于监控 MySQL 性能。

**安装 PMM 客户端：**
```bash
sudo yum install -y pmm2-client
```

**注册 PMM 服务器：**
```bash
pmm-admin config --server-insecure-tls --server-url=https://admin:password@pmm-server:443
```

**添加 MySQL 实例监控：**
```bash
pmm-admin add mysql --username=root --password=yourpassword
```

---

### 7. 安全性管理

确保数据库的安全，防止未经授权的访问。

#### 7.1 更新 MySQL 版本

定期检查并更新到最新的补丁版本，以修复已知漏洞。

**检查当前版本：**
```bash
mysql -V
```

**更新 MySQL：**
```bash
sudo yum update -y mysql-server
```

#### 7.2 配置防火墙

**允许特定端口：**
```bash
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
```

#### 7.3 使用 SSL/TLS 加密连接

**生成证书并配置 MySQL 使用 SSL/TLS。**

#### 7.4 定期更改重要用户密码

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'newsecurepassword';
```

---

### 8. 日常维护脚本

编写脚本自动化日常维护任务，提升效率。

#### 8.1 数据库健康检查脚本

**health_check.sh**
```bash
#!/bin/bash
STATUS=$(mysql -u root -p'yourpassword' -e "SHOW STATUS LIKE 'wsrep_cluster_status';" | grep Primary)
if [ -z "$STATUS" ]; then
    echo "$(date): Cluster status is not Primary!" >> /var/log/mysql/health_check.log
    # 发送告警通知
fi
```

**设置定时任务：**
```cron
*/5 * * * * /path/to/health_check.sh
```

#### 8.2 自动清理日志脚本

**clean_logs.sh**
```bash
#!/bin/bash
find /var/log/mysql/ -type f -name "*.log" -mtime +30 -exec rm -f {} \;
```

**设置定时任务：**
```cron
0 3 * * * /path/to/clean_logs.sh
```

---

### 9. 故障排查

当出现问题时，快速定位和解决问题。

#### 9.1 检查节点状态

```sql
SHOW STATUS LIKE 'wsrep%';
```

#### 9.2 检查错误日志

```bash
tail -n 100 /var/log/mysql/error.log
```

#### 9.3 检查网络连接

```bash
ping 192.168.100.112
telnet 192.168.100.112 3306
```

#### 9.4 重置集群

在极端情况下，需要重置整个集群。

**步骤：**
1. **停止所有节点：**
   ```bash
   sudo systemctl stop mysql
   ```
2. **在其中一个节点上启动新集群：**
   ```bash
   sudo /usr/local/mysql/bin/mysqld --wsrep-new-cluster
   ```
3. **启动其他节点：**
   ```bash
   sudo systemctl start mysql
   ```

---

### 10. 计划任务配置

使用 `crontab` 配置定期任务，自动执行备份、清理等操作。

**编辑 crontab：**
```bash
crontab -e
```

**示例任务：**
```cron
# 每天凌晨2点全量备份
0 2 * * * /path/to/full_backup.sh >> /var/log/mysql/backup.log 2>&1

# 每周日凌晨3点清理旧备份
0 3 * * 0 /path/to/clean_old_backups.sh >> /var/log/mysql/cleanup.log 2>&1

# 每5分钟检查集群健康状态
*/5 * * * * /path/to/health_check.sh >> /var/log/mysql/health_check.log 2>&1
```

---

## 总结

上述命令和操作涵盖了日常管理 MySQL PXC 集群的主要方面，包括监控、备份、用户管理、性能调优和安全性等。通过定期执行和监控这些任务，可以确保集群的高可用性、数据安全性和性能稳定性。

**建议：**
- 定期审查和更新运维流程，根据实际情况调整策略。
- 建立完善的监控和告警机制，及时发现并处理异常。
- 加强安全措施，防范潜在的安全威胁。
- 定期培训和更新运维人员的知识，确保团队具备应对各种情况的能力。

