## MYSQL主从复制GTID模式一键搭建

### 一、环境描述

两台数据库服务器（192.168.1.10 和 192.168.1.20）信息如下：

|      IP      |  操作系统  | CPU  | 内存 | 磁盘  | 数据量 |  角色  |
| :----------: | :--------: | :--: | :--: | :---: | :----: | :----: |
| 192.168.1.10 | CentOS 7.9 |  8   | 32GB | 500GB | 200GB  | master |
| 192.168.1.20 | CentOS 7.9 |  8   | 32GB | 500GB | 200GB  | slave  |

### 二、MySQL 二进制安装步骤

#### 1. 下载并解压 MySQL 二进制包

在两台服务器上执行以下步骤：

```bash
# 下载MySQL二进制包
wget https://dev.mysql.com/get/archives/mysql-8.0/mysql-8.0.35-linux-glibc2.12-x86_64.tar.xz

# 解压到/usr/local目录
tar -xvf mysql-8.0.35-linux-glibc2.12-x86_64.tar.xz -C /usr/local

# 创建软链接
cd /usr/local
ln -s mysql-8.0.35-linux-glibc2.12-x86_64 mysql
```

#### 2. 创建 MySQL 数据目录和配置文件

```bash
# 创建数据目录和配置文件目录
mkdir -p /data/mysqldata
touch /etc/my.cnf

# 设置权限
chown -R mysql:mysql /usr/local/mysql /data/mysqldata /etc/my.cnf
```

#### 3. 初始化 MySQL 数据库

```bash
# 初始化数据库
/usr/local/mysql/bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/data/mysqldata

# 获取初始密码
cat /data/mysqldata/*.err | grep 'temporary password'
```

### 三、MySQL 配置优化

在 `/etc/my.cnf` 中配置以下内容：

```ini
[client]
port = 3306
socket = /tmp/mysql.sock

[mysql]
default-character-set = utf8mb4

[mysqld]
user = mysql
port = 3306
basedir = /usr/local/mysql
datadir = /data/mysqldata
socket = /tmp/mysql.sock
pid-file = /data/mysqldata/mysql.pid

# 性能优化
innodb_buffer_pool_size = 24G      # 配置为内存的75%
innodb_buffer_pool_instances = 8   # 配置buffer_pool为8个
innodb_log_file_size = 2G          # 调整日志文件大小
innodb_log_buffer_size = 128M      # InnoDB用于写入磁盘日志文件的缓冲区大小
innodb_flush_log_at_trx_commit = 1 # 保证数据安全
innodb_file_per_table = 1          # 每张表独立表空间
innodb_io_capacity = 2000          # I/O操作数
innodb_read_io_threads = 4         # 读I/O线程
innodb_write_io_threads = 4        # 写I/O线程
innodb_flush_method = O_DIRECT     # 防止双写
default_storage_engine = InnoDB    # 存储引擎默认InnoDB
lower-case-table-names = 1         # 大小写

# 连接优化
max_connections = 500              # 最大连接数
max_connect_errors = 1000          # 最大错误连接次数
skip_name_resolve = 1              # 只能用IP地址检查客户端的登录

# 复制配置
server-id = 1                       # 主库ID
gtid_mode = ON                      # 启用GTID
enforce-gtid-consistency = ON       # 强制GTID一致性
log-bin = mysql-bin                 # 启用二进制日志
binlog_format = ROW                 # 二进制日志格式
binlog_row_image = FULL             # 行级日志模式
master_info_repository = TABLE      # 主库信息表存储
relay_log_info_repository = TABLE   # 中继日志信息表存储
sync-binlog = 1                     # 保证数据安全
log-slave-updates = 1               # 允许从库更新记录到binlog
binlog_expire_logs_seconds = 604800 # binlog保留时间7day

# 多线程复制
slave_parallel_workers = 4          # 从库并行复制线程数
slave_parallel_type = LOGICAL_CLOCK # 并行复制类型

# 缓存优化
table_open_cache = 4096             # 程打开的表的数量            
table_definition_cache = 2048       # 可以存储在定义缓存中的表定义数量
tmp_table_size = 128M               # 内部内存临时表大小
max_heap_table_size = 256M          # 内部内存临时表的最大值
thread_cache_size = 128             # 连接使用缓存线程
open_files_limit = 65535            # mysqld进程能使用的最大文件描述(FD)符数量
```

### 四、配置 MySQL 服务并设置开机启动

#### 1. 创建 MySQL 服务文件

在 `/etc/systemd/system/mysqld.service` 中创建以下内容：

```ini
[Unit]
Description=MySQL Server
After=network.target

[Service]
User=mysql
Group=mysql
ExecStart=/usr/local/mysql/bin/mysqld_safe --defaults-file=/etc/my.cnf 
ExecStop=/usr/local/mysql/bin/mysqladmin --defaults-file=/etc/my.cnf shutdown
LimitNOFILE = 5000

[Install]
WantedBy=multi-user.target

```

#### 2. 配置环境变量

```bash
echo 'export PATH=/usr/local/mysql/bin:$PATH' >> /etc/profile
source /etc/profile
```

#### 3. 设置 MySQL 开机启动

```bash
systemctl daemon-reload
systemctl enable mysqld
systemctl start mysqld
```

### 五、配置主从复制（GTID模式）

#### 1. 主库配置（192.168.1.10）

编辑 `/etc/my.cnf`：

```ini
server-id = 1
gtid_mode = ON
enforce-gtid-consistency = ON
log-bin = mysql-bin
binlog_format = ROW
```

#### 2. 从库配置（192.168.1.20）

编辑 `/etc/my.cnf`：

```ini
server-id = 2
gtid_mode = ON
enforce-gtid-consistency = ON
log-bin = mysql-bin
binlog_format = ROW
relay-log = relay-bin
```

#### 3.登陆数据库创建复制用户

在主库上执行：

```shell
# 登录 MySQL 设置
CREATE USER 'repl'@'192.168.1.20' IDENTIFIED BY 'repl_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.1.20';
FLUSH PRIVILEGES;
```

#### 4. 设置主从复制

在从库上执行：

```sql
# 登录 MySQL 设置复制
CHANGE MASTER TO 
MASTER_HOST='192.168.1.10',
MASTER_USER='replication_user',
MASTER_PASSWORD='replication_password',
MASTER_AUTO_POSITION=1;

-- 开始复制
START SLAVE;
```

### 六、一键部署脚本

编写如下脚本：在`/etc/my.cnf`创建好配置文件后，运行此脚本，即可完成安装、配置和主从复制的部署。

```bash
#!/bin/bash

# 定义变量
MASTER_IP="192.168.1.10"
SLAVE_IP="192.168.1.20"
MYSQL_VERSION="8.0.35"
BASEDIR="/usr/local/mysql"
DATADIR="/data/mysqldata"
CONFIG_FILE="/etc/my.cnf"
MYSQL_USER="mysql"
MYSQL_GROUP="mysql"
MYSQL_REPLUSER="repl"
MYSQL_REPLPASS="repl_password"

# 检查是否以 root 用户执行
if [ "$(id -u)" -ne 0 ]; then
    echo "请以 root 用户执行此脚本。"
    exit 1
fi

# 创建 mysql 用户和组
groupadd $MYSQL_GROUP
useradd -r -g $MYSQL_GROUP -s /bin/false $MYSQL_USER

# 解压并安装 MySQL
tar -zxvf mysql-${MYSQL_VERSION}-linux-glibc2.12-x86_64.tar.gz
mv mysql-${MYSQL_VERSION}-linux-glibc2.12-x86_64 $BASEDIR

# 配置 MySQL 目录权限
mkdir -p $DATADIR
chown -R $MYSQL_USER:$MYSQL_GROUP $BASEDIR
chown -R $MYSQL_USER:$MYSQL_GROUP $DATADIR

# 创建并配置 my.cnf 文件
cat > $CONFIG_FILE <<EOF
[client]
port = 3306
socket = /tmp/mysql.sock

[mysql]
default-character-set = utf8mb4

[mysqld]
user = mysql
port = 3306
basedir = /usr/local/mysql
datadir = /data/mysqldata
socket = /tmp/mysql.sock
pid-file = /data/mysqldata/mysql.pid

# 性能优化
innodb_buffer_pool_size = 24G      # 配置为内存的75%
innodb_buffer_pool_instances = 8   # 配置buffer_pool为8个
innodb_log_file_size = 2G          # 调整日志文件大小
innodb_log_buffer_size = 128M      # InnoDB用于写入磁盘日志文件的缓冲区大小
innodb_flush_log_at_trx_commit = 1 # 保证数据安全
innodb_file_per_table = 1          # 每张表独立表空间
innodb_io_capacity = 2000          # I/O操作数
innodb_read_io_threads = 4         # 读I/O线程
innodb_write_io_threads = 4        # 写I/O线程
innodb_flush_method = O_DIRECT     # 防止双写
default_storage_engine = InnoDB    # 存储引擎默认InnoDB

# 连接优化
max_connections = 500              # 最大连接数
max_connect_errors = 1000          # 最大错误连接次数
skip_name_resolve = 1              # 只能用IP地址检查客户端的登录

# 复制配置
server-id = 1                       # 主库ID
gtid_mode = ON                      # 启用GTID
enforce-gtid-consistency = ON       # 强制GTID一致性
log-bin = mysql-bin                 # 启用二进制日志
binlog_format = ROW                 # 二进制日志格式
binlog_row_image = FULL             # 行级日志模式
master_info_repository = TABLE      # 主库信息表存储
relay_log_info_repository = TABLE   # 中继日志信息表存储
sync-binlog = 1                     # 保证数据安全
log-slave-updates = 1               # 允许从库更新记录到binlog
binlog_expire_logs_seconds = 604800 # binlog保留时间

# 多线程复制
slave_parallel_workers = 4          # 从库并行复制线程数
slave_parallel_type = LOGICAL_CLOCK # 并行复制类型

# 缓存优化
table_open_cache = 4096             # 程打开的表的数量            
table_definition_cache = 2048       # 可以存储在定义缓存中的表定义数量
tmp_table_size = 128M               # 内部内存临时表大小
max_heap_table_size = 256M          # 内部内存临时表的最大值
thread_cache_size = 128             # 连接使用缓存线程
open_files_limit = 65535            # mysqld进程能使用的最大文件描述(FD)符数量
EOF

# 初始化 MySQL 数据库
$BASEDIR/bin/mysqld --initialize-insecure --user=$MYSQL_USER --basedir=$BASEDIR --datadir=$DATADIR

# 将 MySQL 添加为系统服务
cat > /etc/systemd/system/mysql.service <<EOF
[Unit]
Description=MySQL Server
After=network.target

[Service]
ExecStart=$BASEDIR/bin/mysqld_safe --defaults-file=$CONFIG_FILE
ExecStop=$BASEDIR/bin/mysqladmin --defaults-file=$CONFIG_FILE shutdown
User=$MYSQL_USER
Group=$MYSQL_GROUP
Restart=always
LimitNOFILE=5000

[Install]
WantedBy=multi-user.target
EOF

# 加载服务并设置开机启动
systemctl daemon-reload
systemctl enable mysql.service
systemctl start mysql.service

# 检查 MySQL 服务是否启动成功
if systemctl status mysql.service | grep -q "active (running)"; then
    echo "MySQL 服务启动成功。"
else
    echo "MySQL 服务启动失败，请检查日志。"
    exit 1
fi

# 设置环境变量
echo "export PATH=\$PATH:$BASEDIR/bin" >> /etc/profile
source /etc/profile

# 配置主从复制（在主服务器上执行）
if [ "$(hostname -I | awk '{print $1}')" == "$MASTER_IP" ]; then
    $BASEDIR/bin/mysql -e "CREATE USER '$MYSQL_REPLUSER'@'%' IDENTIFIED WITH mysql_native_password BY '$MYSQL_REPLPASS';"
    $BASEDIR/bin/mysql -e "GRANT REPLICATION SLAVE ON *.* TO '$MYSQL_REPLUSER'@'%';"
    $BASEDIR/bin/mysql -e "FLUSH PRIVILEGES;"
    MASTER_STATUS=$($BASEDIR/bin/mysql -e "SHOW MASTER STATUS;" | awk 'NR==2 {print $1,$2}')
    echo "主库状态：$MASTER_STATUS"
fi

# 配置从库（在从服务器上执行）
if [ "$(hostname -I | awk '{print $1}')" == "$SLAVE_IP" ]; then
    $BASEDIR/bin/mysql -e "CHANGE MASTER TO MASTER_HOST='$MASTER_IP', MASTER_USER='$MYSQL_REPLUSER', MASTER_PASSWORD='$MYSQL_REPLPASS', MASTER_AUTO_POSITION=1,master_connect_retry=30,get_master_public_key=1;"
    $BASEDIR/bin/mysql -e "START SLAVE;"
    echo "从库配置完成。"
fi


echo "MySQL 主从复制配置已完成，请根据实际情况调整配置文件并进行测试。"
```
#### 注意点
1. **检查和处理脚本权限**：确保脚本由 `root` 用户执行，否则会退出。
2. **系统服务配置优化**：使用 `systemctl` 管理服务，并配置 `Restart=always` 保证服务高可用。
3. **明确主从角色分配**：脚本可以区分主服务器与从服务器，根据 IP 执行相应操作。
4. **确保服务启动成功**：增加服务启动状态检查，及时反馈问题。
5. **环境变量配置**：为 MySQL 配置系统环境变量，确保可以在终端直接使用 MySQL 命令。

#### 执行步骤

1. 将脚本保存为 `mysql_install.sh`。
2. 上传并赋予脚本执行权限：
   ```bash
   chmod +x mysql_install.sh
   ```
3. 运行脚本：
   ```bash
   ./mysql_install.sh
   ```

这个优化后的脚本应该能在两个服务器上成功安装、配置 MySQL，并设置好主从复制。如果遇到问题，请根据错误信息逐步排查。