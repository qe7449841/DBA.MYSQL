### MYSQL增强半同步复制搭建

#### 优点：
1. **数据一致性保障**：在主库提交事务前，至少一个从库确认收到并写入了该事务的二进制日志，确保数据在主从库之间的一致性。
2. **自动故障转移支持**：在主库发生故障时，从库能快速切换为主库，减少数据丢失。
3. **提升数据可靠性**：相比于普通半同步复制，减少了数据丢失的风险。

#### 缺点：
1. **性能开销**：在高并发场景下，可能增加主库的事务提交延迟，影响整体性能。
2. **配置复杂性**：需要调整适当的参数配置来平衡性能与数据安全。

#### 适用场景：
- **关键业务系统**：如金融、银行、电商等，对数据一致性要求高的场景。
- **主从延迟敏感场景**：要求数据尽快在从库可用，以便快速故障切换的场景。

---

### 环境准备与安装

#### 1. 安装MySQL服务器

```bash
# 下载并解压二进制包
wget https://dev.mysql.com/get/archives/mysql-8.0/mysql-8.0.35-linux-glibc2.12-x86_64.tar.xz
tar -xvf mysql-8.0.35-linux-glibc2.12-x86_64.tar.xz
mv mysql-8.0.35-linux-glibc2.12-x86_64 /usr/local/mysql

# 创建用户及目录
groupadd mysql
useradd -r -g mysql mysql
mkdir -p /data/mysqldata
chown -R mysql:mysql /usr/local/mysql /data/mysqldata

# 初始化数据库
/usr/local/mysql/bin/mysqld --initialize-insecure --basedir=/usr/local/mysql --datadir=/data/mysqldata --user=mysql

# 设置环境变量
echo 'export PATH=/usr/local/mysql/bin:$PATH' >> /etc/profile
source /etc/profile
```

#### 2. 配置优化 `my.cnf`

```ini
[mysqld]
basedir = /usr/local/mysql
datadir = /data/mysqldata
socket = /tmp/mysql.sock
user = mysql
symbolic-links = 0

# 性能优化
innodb_buffer_pool_size = 12G  # 根据服务器内存设置
innodb_log_file_size = 2G  # 根据负载设置
innodb_flush_log_at_trx_commit = 1  # 提升数据一致性
sync_binlog = 1  # 保证binlog的同步性

# 日志及复制设置
log-bin = mysql-bin
binlog_format = row
server-id = 1
gtid_mode = on
enforce_gtid_consistency = on
master_info_repository = TABLE
relay_log_info_repository = TABLE
relay-log = relay-bin
log_slave_updates = 1
binlog_checksum = NONE
sync_master_info = 1
sync_relay_log = 1
sync_relay_log_info = 1

# 半同步复制
plugin-load = "semisync_master.so;semisync_slave.so"
rpl_semi_sync_master_enabled = 1
rpl_semi_sync_slave_enabled = 1
rpl_semi_sync_master_timeout = 1000  # 1秒

# 网络优化
max_connections = 500
max_allowed_packet = 64M
```

#### 3. 创建MySQL服务并设置开机启动

```bash
cat > /etc/systemd/system/mysql.service <<EOF
[Unit]
Description=MySQL Server
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/mysql/bin/mysqld_safe --defaults-file=/etc/my.cnf
ExecStop=/usr/local/mysql/bin/mysqladmin shutdown
User=mysql
Group=mysql
LimitNOFILE = 5000
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl enable mysql
systemctl start mysql
```

#### 4. 搭建GTID模式的多线程增强半同步主从复制

##### 4.1 配置Master（192.168.1.11）

```sql
CHANGE MASTER TO MASTER_HOST='192.168.1.12', MASTER_USER='replica_user', MASTER_PASSWORD='password', MASTER_AUTO_POSITION = 1;
START SLAVE;
```

##### 4.2 配置Slave（192.168.1.12）

```sql
CHANGE MASTER TO MASTER_HOST='192.168.1.11', MASTER_USER='replica_user', MASTER_PASSWORD='password', MASTER_AUTO_POSITION = 1;
START SLAVE;
```

##### 4.3 启用多线程复制

```sql
SET GLOBAL slave_parallel_workers = 4;
SET GLOBAL slave_parallel_type = LOGICAL_CLOCK;
```

### 一键部署脚本

```bash
#!/bin/bash

# 配置环境
basedir=/usr/local/mysql
datadir=/data/mysqldata
config=/etc/my.cnf

# 下载安装
wget https://dev.mysql.com/get/archives/mysql-8.0/mysql-8.0.35-linux-glibc2.12-x86_64.tar.xz
tar -xvf mysql-8.0.35-linux-glibc2.12-x86_64.tar.xz
mv mysql-8.0.35-linux-glibc2.12-x86_64 $basedir

# 创建用户和目录
groupadd mysql
useradd -r -g mysql mysql
mkdir -p $datadir
chown -R mysql:mysql $basedir $datadir

# 初始化MySQL
$basedir/bin/mysqld --initialize-insecure --basedir=$basedir --datadir=$datadir --user=mysql

# 配置环境变量
echo 'export PATH=$basedir/bin:$PATH' >> /etc/profile
source /etc/profile

# 设置配置文件
cat > $config <<EOF
[mysqld]
basedir = $basedir
datadir = $datadir
socket = /tmp/mysql.sock
user = mysql
innodb_buffer_pool_size = 12G
innodb_log_file_size = 2G
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
log-bin = mysql-bin
binlog_format = row
server-id = 1
gtid_mode = on
enforce_gtid_consistency = on
master_info_repository = TABLE
relay_log_info_repository = TABLE
relay-log = relay-bin
log_slave_updates = 1
binlog_checksum = NONE
sync_master_info = 1
sync_relay_log = 1
sync_relay_log_info = 1
plugin-load = "semisync_master.so;semisync_slave.so"
rpl_semi_sync_master_enabled = 1
rpl_semi_sync_slave_enabled = 1
rpl_semi_sync_master_timeout = 1000
max_connections = 500
max_allowed_packet = 64M
EOF

# 创建服务
cat > /etc/systemd/system/mysql.service <<EOF
[Unit]
Description=MySQL Server
After=network.target

[Service]
Type=forking
ExecStart=$basedir/bin/mysqld_safe --defaults-file=$config
ExecStop=$basedir/bin/mysqladmin shutdown
User=mysql
Group=mysql
LimitNOFILE = 5000
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# 启动MySQL
systemctl enable mysql
systemctl start mysql
```











感谢你的反馈。我会详细检查之前提供的步骤并修正错误，确保提供一个准确无误的 MySQL 部署与配置方案。以下是修订后的详细部署步骤，包括增强半同步复制的优缺点、配置步骤和最终的一键部署脚本。

### 环境概述：
- **主服务器（Master）：192.168.1.11**
- **从服务器（Slave）：192.168.1.12**
- **操作系统**：Red Hat 8.0
- **硬件配置**：CPU 4核，内存16GB，固态硬盘500GB
- **预期数据量**：300GB

### 一、增强半同步复制的优缺点和适用场景
**优点**：
1. **数据一致性高**：在提交事务之前确保至少一个从库收到了该事务的复制日志。
2. **数据丢失风险低**：适合关键业务系统，减少了数据丢失的风险。
3. **适用场景**：金融交易系统、订单处理系统等对数据一致性要求极高的业务场景。

**缺点**：
1. **延迟增加**：由于需要等待从库的确认，写操作的延迟较高。
2. **性能开销**：增强的同步机制会导致性能下降，尤其是在高并发写入的场景中。

**适用场景**：
- 数据一致性高于写入性能的场景，例如银行、金融交易、订单处理系统。
- 高度依赖数据同步完整性的场景。

### 二、MySQL 安装及配置步骤

#### 1. 下载并安装 MySQL
```bash
# 依赖包安装
sudo yum install -y libaio-devel numactl-libs

# 创建MySQL用户和组
sudo groupadd mysql
sudo useradd -r -g mysql -s /bin/false mysql

# 解压并移动MySQL
tar -xzf mysql-8.0.35-linux-glibc2.12-x86_64.tar.xz
sudo mv mysql-8.0.35-linux-glibc2.12-x86_64 /usr/local/mysql

# 创建数据目录和授权
sudo mkdir -p /data/mysqldata
sudo chown -R mysql:mysql /usr/local/mysql /data/mysqldata
```

#### 2. 初始化 MySQL 数据库
```bash
cd /usr/local/mysql
sudo bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/data/mysqldata
```

#### 3. 创建系统服务并设置开机自启
```bash
# 创建systemd服务文件
sudo vim /etc/systemd/system/mysql.service

# 在文件中添加以下内容：
[Unit]
Description=MySQL Server
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/mysql/bin/mysqld_safe --defaults-file=/etc/my.cnf
ExecStop=/usr/local/mysql/bin/mysqladmin shutdown
User=mysql
Group=mysql
LimitNOFILE=5000
Restart=always

[Install]
WantedBy=multi-user.target

# 重新加载服务并启动
sudo systemctl daemon-reload
sudo systemctl enable mysql
sudo systemctl start mysql
```

#### 4. 优化配置文件 `my.cnf`
```bash
sudo vim /etc/my.cnf

# 基础配置：
[mysqld]
user=mysql
basedir=/usr/local/mysql
datadir=/data/mysqldata
socket=/data/mysqldata/mysql.sock
log-error=/data/mysqldata/mysql.err
pid-file=/data/mysqldata/mysql.pid

# 性能优化：
innodb_buffer_pool_size=12G  # 分配12GB给InnoDB缓冲池
innodb_log_file_size=2G      # 调整重做日志文件大小以提高I/O性能
innodb_flush_method=O_DIRECT # 减少双写缓冲
innodb_io_capacity=2000      # 提升磁盘写入能力

# 复制相关设置：
server_id=1                  # 主库ID
log-bin=/data/mysqldata/mysql-bin
binlog-format=ROW
gtid_mode=ON
enforce-gtid-consistency=ON
log-slave-updates=ON
master-info-repository=TABLE
relay-log-info-repository=TABLE
binlog_row_image=FULL
sync_binlog=1
innodb_flush_log_at_trx_commit=1

# 半同步设置：
plugin-load="rpl_semi_sync_master=semisync_master.so;rpl_semi_sync_slave=semisync_slave.so"
rpl_semi_sync_master_enabled=1
rpl_semi_sync_master_timeout=1000  # 超时时间
rpl_semi_sync_slave_enabled=1
```

#### 5. 搭建增强半同步主从复制
##### 在 Master 配置：
```bash
# 启用增强半同步插件
mysql> INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
mysql> SET GLOBAL rpl_semi_sync_master_enabled = 1;
mysql> SET GLOBAL rpl_semi_sync_master_timeout = 1000;  # 超时时间

# 创建用户和授予权限
mysql> CREATE USER 'replication_user'@'192.168.1.12' IDENTIFIED BY 'password';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'192.168.1.12';
mysql> FLUSH PRIVILEGES;

# 记录binlog位置
mysql> SHOW MASTER STATUS;
```

##### 在 Slave 配置：
```bash
# 启用增强半同步插件
mysql> INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
mysql> SET GLOBAL rpl_semi_sync_slave_enabled = 1;

# 配置主从复制
mysql> CHANGE MASTER TO MASTER_HOST='192.168.1.11', MASTER_USER='replication_user', MASTER_PASSWORD='password', MASTER_AUTO_POSITION=1;
mysql> START SLAVE;

# 查看状态
mysql> SHOW SLAVE STATUS\G;
```

### 三、一键部署 Shell 脚本
```bash
#!/bin/bash

# 全局变量
MASTER_IP=192.168.1.11
SLAVE_IP=192.168.1.12
MYSQL_USER=mysql
MYSQL_BASEDIR=/usr/local/mysql
MYSQL_DATADIR=/data/mysqldata
MYSQL_CNF=/etc/my.cnf
REPL_USER=replication_user
REPL_PASSWORD=password

# 创建MySQL用户和目录
groupadd $MYSQL_USER
useradd -r -g $MYSQL_USER -s /bin/false $MYSQL_USER
mkdir -p $MYSQL_DATADIR
chown -R $MYSQL_USER:$MYSQL_USER $MYSQL_BASEDIR $MYSQL_DATADIR

# 初始化数据库
$MYSQL_BASEDIR/bin/mysqld --initialize --user=$MYSQL_USER --basedir=$MYSQL_BASEDIR --datadir=$MYSQL_DATADIR

# 创建服务文件
cat <<EOF > /etc/systemd/system/mysql.service
[Unit]
Description=MySQL Server
After=network.target

[Service]
Type=forking
ExecStart=$MYSQL_BASEDIR/bin/mysqld_safe --defaults-file=$MYSQL_CNF
ExecStop=$MYSQL_BASEDIR/bin/mysqladmin shutdown
User=$MYSQL_USER
Group=$MYSQL_USER
LimitNOFILE=5000
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# 重新加载服务并启动
systemctl daemon-reload
systemctl enable mysql
systemctl start mysql

# 主从配置
if [ "$(hostname -I)" == "$MASTER_IP" ]; then
  mysql -e "INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';"
  mysql -e "SET GLOBAL rpl_semi_sync_master_enabled = 1;"
  mysql -e "SET GLOBAL rpl_semi_sync_master_timeout = 1000;"
  mysql -e "CREATE USER '$REPL_USER'@'$SLAVE_IP' IDENTIFIED BY '$REPL_PASSWORD';"
  mysql -e "GRANT REPLICATION SLAVE ON *.* TO '$REPL_USER'@'$SLAVE_IP';"
  mysql -e "FLUSH PRIVILEGES;"
  mysql -e "SHOW MASTER STATUS;"
fi

if [ "$(hostname -I)" == "$SLAVE_IP" ]; then
  mysql -e "INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';"
  mysql -e "SET GLOBAL rpl_semi_sync_slave_enabled = 1;"
  mysql -e "CHANGE MASTER TO MASTER_HOST='$MASTER_IP', MASTER_USER='$REPL_USER', MASTER_PASSWORD='$REPL_PASSWORD', MASTER_AUTO_POSITION=1;"
  mysql -e "START SLAVE;"
  mysql -e "SHOW SLAVE STATUS\G;"
fi
```

### 四、总结
1. 增强半同步适用于高一致性需求的场景，但需权衡性能。
2. 根据不同硬件配置和业务需求，调整`my.cnf`的相关参数。
3. 确保服务文件和脚本无误后执行，以避免服务启动失败。

此方案在性能和一致性之间做出了良好平衡。如果在执行过程中遇到任何问题，欢迎随时反馈。