# MySQL 高可用架构MHA详解

[TOC]

##  一 .MHA概述

**MySQL MHA（Master High Availability）是一种经典的高可用架构，专门用于在主从复制环境中实现自动故障切换和最小化数据丢失。**

### 1. MHA 架构概述

MHA 是由 MHA Manager 和 MHA Node 组成的高可用解决方案，主要用于 MySQL 的主从架构。MHA Manager 部署在独立的管理节点上，监控 MySQL 集群中的主库和从库的状态；MHA Node 则部署在每个 MySQL 服务器上，用于执行故障切换操作。

基本架构如下：
- **主库（Master）**：负责处理所有的写操作。
- **从库（Slaves）**：接收主库的二进制日志并进行复制，只负责读操作。
- **MHA Manager**：管理整个集群的故障转移逻辑。
- **MHA Node**：部署在所有的 MySQL 服务器上，配合 MHA Manager 完成主从切换。

### 2.MHA 的工作原理

MHA 主要由两部分组成：
1. **MHA Manager**（管理节点）：负责管理整个集群的高可用性。它会定期监控集群中各个节点的状态，并在检测到主库故障时，自动将故障转移到一个新的主库上。
2. **MHA Node**（数据节点）：部署在每个 MySQL 服务器上，负责在故障转移过程中保存和恢复数据，以避免数据丢失。

### 3.MHA 架构

MHA 的典型架构包括一个管理节点和多个 MySQL 数据库节点：
- **管理节点 (MHA Manager)**：负责监控主库状态，并在主库发生故障时执行故障转移。通常建议将 MHA Manager 部署在独立的机器上。
- **MySQL 数据库节点 (MHA Node)**：包括一个主库和多个从库。每个 MySQL 节点都运行 `mha4mysql-node`，用来协助数据的保存和恢复。

 **典型的 MHA 架构示意图**

   ```plaintext
+----------------+     +----------------+     +----------------+
|   MHA Manager  | --> |  MySQL Master  | <-- | MySQL Slave 1  |
+----------------+     +----------------+     +----------------+
                               ^                /
                               |               /
                               |              /
                           +----------------+ 
                           | MySQL Slave 2  | 
                           +----------------+
   ```

### 4.MHA 的优缺点

* **优点：**

1. **高可用性**：MHA 能够自动检测主库的故障，并在 10 到 30 秒内完成故障转移。
2. **数据一致性**：在故障转移过程中，MHA 会尽量将所有的二进制日志复制到新主库，以避免数据丢失。
3. **无需更改客户端连接**：通过合理配置虚拟 IP 地址或使用代理，客户端无需感知主库的切换。

*  **缺点：**

1. **单点故障**：MHA Manager 本身是一个单点故障，需要通过备份或冗余机制保证其高可用性。
2. **复制延迟问题**：如果从库的延迟较大，可能导致故障转移时部分数据丢失。
3. **对环境依赖**：MHA 对环境的要求较高，例如需要所有节点之间具备 SSH 免密登录，且对数据库配置有一定要求。

### 5.MHA 的使用限制

1. **同步复制与异步复制**：MHA 支持 MySQL 的异步复制和半同步复制，但不支持 MySQL 的全同步复制。
2. **主从架构**：MHA 主要适用于一主多从的 MySQL 架构，如果涉及到多主架构（例如 Galera Cluster），MHA 并不适用。
3. **操作系统限制**：MHA 主要支持 Linux/Unix 系统，Windows 平台不适用。

### 6.MHA 的环境限制

1. **网络条件**：MHA 要求主从服务器之间具备较低的网络延迟，以保证日志的及时同步。
2. **SSH 访问**：所有 MHA 管理节点和 MySQL 节点之间必须配置 SSH 免密访问。
3. **服务器负载**：如果主库的负载过高，故障转移可能会影响性能。因此，在高负载环境中，合理的分布式架构设计和定期的监控优化非常重要。

### 7.MHA 的故障转移过程

1. **检测故障**：MHA Manager 定期通过心跳机制检测主库状态。
2. **隔离故障主库**：确认主库故障后，MHA 会隔离故障主库，防止其继续接受写操作。
3. **保存二进制日志**：MHA Node 在故障转移时会保存主库的二进制日志，确保数据不丢失。
4. **选择新主库**：MHA 根据配置规则选择一个合适的从库作为新的主库，并应用二进制日志。
5. **重建复制关系**：将其他从库指向新的主库，完成主从复制的重建。

### 8.MHA 的部署与配置建议

1. **独立的 MHA Manager 服务器**：建议将 MHA Manager 部署在独立的服务器上，以避免单点故障。
2. **确保 SSH 免密登录**：配置所有 MySQL 节点之间的 SSH 免密登录，确保管理节点可以访问所有数据库服务器。
3. **定期监控**：定期监控 MHA 的状态日志，确保系统的稳定运行。

-- --------------------------------------------------------------------------------------------------

## 二 . 实战搭建

###   MHA 架构搭建

#### 环境说明:

| IP              | OS          | CPU  | 内存 | 磁盘 | 角色         | 安装软件              |
| --------------- | ----------- | ---- | ---- | ---- | ------------ | --------------------- |
| 192.168.136.140 | Red Hat 8.0 | 4    | 16G  | 500G | MySQL 主库   | mysql-8.0.35+MHA-node |
| 192.168.136.141 | Red Hat 8.0 | 4    | 16G  | 500G | MySQL 从库 1 | mysql-8.0.35+MHA-node |
| 192.168.136.142 | Red Hat 8.0 | 4    | 16G  | 500G | MySQL 从库 2 | mysql-8.0.35+MHA-node |
| 192.168.136.143 | Red Hat 8.0 | 4    | 16G  | 500G | MHA 管理节点 | MHA-manager           |
| 192.168.136.145 | 虚拟IP      | -    | -    | -    | -            | -                     |

### 部署步骤

#### 1.  安装前准备

在所有 MySQL 服务器上进行以下操作：

**安装依赖包**：

```bash
yum install -y wget perl-DBD-MySQL libaio net-tools perl-IO-Socket-SSL
```

#### 2.  下载并解压MySQL二进制包

```bash
# 下载MySQL二进制包
wget https://dev.mysql.com/get/archives/mysql-8.0/mysql-8.0.35-linux-glibc2.17-x86_64.tar.xz

# 解压到/usr/local目录
tar -xvf mysql-8.0.35-linux-glibc2.17-x86_64.tar.xz -C /usr/local

# 创建软链接
ln -s /usr/local/mysql-8.0.35-linux-glibc2.17-x86_64 mysql
```

#### 3.  创建 MySQL 数据目录和配置文件

```bash
# 创建用户
groupadd mysql
useradd -g mysql mysql

# 创建数据目录和配置文件目录
mkdir -p /data/mysqldata

touch /etc/my.cnf

# 设置权限
chown -R mysql:mysql /usr/local/mysql /data/mysqldata /etc/my.cnf
```
#### 4.  MySQL配置优化

在 `/etc/my.cnf` 中配置以下内容：

```ini
[client]
port = 3306
socket = /tmp/mysql.sock

[mysql]
default-character-set = utf8mb4

[mysqld]

# 基础配置
user = mysql
port = 3306
basedir = /usr/local/mysql
datadir = /data/mysqldata
socket = /tmp/mysql.sock
pid-file = /data/mysqldata/mysql.pid
lower-case-table-names = 1         # 大小写
sql_require_primary_key = ON       # 表必须有主键
log_timestamps = system            # 日志时间

# 自增ID设置
auto_increment_offset=1            # 自增 ID 偏移量
auto_increment_increment=3         # 自增 ID 步长

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
binlog_expire_logs_seconds = 604800 # binlog保留时间7day
binlog_transaction_dependency_tracking =WRITESET
transaction_write_set_extraction =XXHASH64


# 多线程复制
slave_parallel_workers = 4          # 从库并行复制线程数
slave_parallel_type = LOGICAL_CLOCK # 并行复制类型
replica_preserve_commit_order =ON

# 缓存优化
table_open_cache = 4096             # 程打开的表的数量            
table_definition_cache = 2048       # 可以存储在定义缓存中的表定义数量
tmp_table_size = 128M               # 内部内存临时表大小
max_heap_table_size = 256M          # 内部内存临时表的最大值
thread_cache_size = 128             # 连接使用缓存线程
open_files_limit = 65535            # mysqld进程能使用的最大文件描述(FD)符数量
```
#### 5.  初始化 MySQL 数据库

```bash
# 初始化数据库
/usr/local/mysql/bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/data/mysqldata

# 获取初始密码
cat /data/mysqldata/*.err | grep 'temporary password'
```

#### 6.  配置MySQL服务并设置开机启动

- ##### 创建 MySQL 服务文件在 `/etc/systemd/system/mysqld.service` 中创建以下内容：

    ```shell
    vim /etc/systemd/system/mysqld.service
    ```
    
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
    
    - #####  配置环境变量
    
    ```bash
    echo 'export PATH=/usr/local/mysql/bin:$PATH' >> /etc/profile
    source /etc/profile
    ```
    
    - ##### 设置 MySQL 开机启动
    
    ```bash
    systemctl daemon-reload
    systemctl enable mysqld
    systemctl start mysqld
    ```
    
    - ##### 设置 MySQL 密码
    
    ```shell
    # 登陆数据
    mysql -uroot -p
    
    # 修改密码
    alter user user()  IDENTIFIED WITH 'mysql_native_password'  BY  'xxxxxxxx';
    ```
    **mha用户身份认证插件支持到mysql_native_password；建议账号都用'mysql_native_password'作为密码插件。否则后期检测配置会报错：There is no alive server. We can’t do failover **

#### 7.  从库 1  配置文件：

```ini
[mysqld]
...
server_id=2
...
# 自增ID设置
auto_increment_offset=2            # 自增 ID 偏移量
auto_increment_increment=3         # 自增 ID 步长
...
```

#### 8.  从库 2  配置文件：

```ini
[mysqld]
...
server_id=3
...
# 自增ID设置
auto_increment_offset=3            # 自增 ID 偏移量
auto_increment_increment=3         # 自增 ID 步长
...
```

#### 9.  设置 GTID 复制

在主库上：
```sql
CREATE USER 'repl'@'%' IDENTIFIED WITH 'mysql_native_password'  BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
```

在每个从库上：
```sql
CHANGE MASTER TO
  MASTER_HOST='192.168.136.140',
  MASTER_PORT=3306,
  MASTER_USER='repl',
  MASTER_PASSWORD='password',
  MASTER_AUTO_POSITION=1,
  MASTER_CONNECT_RETRY=30,
  GET_MASTER_PUBLIC_KEY=1;

START SLAVE;
```

#### 10.  部署 MHA

1.  **所有配置免密登录**

    ```shell
    # 在所有节点上配置SSH免密登录至所有MySQL节点。
    ssh-keygen -t rsa
    ssh-copy-id -i /root/.ssh/id_rsa root@192.168.136.141
    ssh-copy-id -i /root/.ssh/id_rsa root@192.168.136.142
    ssh-copy-id -i /root/.ssh/id_rsa root@192.168.136.143
    ssh-copy-id -i /root/.ssh/id_rsa root@192.168.136.140
    
    # 所有节点建立MHA目录
    mkdir -p /data/mysql_mha
    ```

2.  **所有节点上安装 MHA-node**：

    ```bash
    # 下载mha4mysql-node
    wget https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
    
    # 安装依赖
    yum install -y epel-release
    yum -y install perl-DBD-MySQL perl-DBI ncftp
    
    # 安装mha4mysql-node
    rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm
    ```

3.  **在管理节点（192.168.136.143）上安装 MHA-manager**：

    ```bash
    # 下载mha4mysql-manager
    https://github.com/yoshinorim/mha4mysql-manager/releases/download/v0.58/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
    
    # 安装依赖
    yum install -y epel-release
    yum install -y perl-Config-Tiny perl-Time-HiRes perl-Parallel-ForkManager perl-Log-Dispatch perl-DBD-MySQL ncftp
    
    # 安装mha4mysql-manager
    rpm -ivh mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
    
    # manager包在/usr/bin下面生成9个文件
    # masterha_check_repl：检查MySQL的复制状况
    # masterha_check_ssh：检查MHA的SSH配置状况
    # masterha_check_status：检测当前MHA运行状态
    # masterha_conf_host：添加或删除配置的server信息
    # masterha_manager：启动MHA
    # masterha_master_monitor：检测master是否宕机
    # masterha_master_switch：控制故障转移（自动或手动）
    # masterha_secondary_check：二次检查主库是否真有问题
    # masterha_stop：关闭MHA
    ```

4.  **检测配置管理节点 MHA **

      **编辑 `/etc/mha/mysql_mha.cnf`：**

    ```ini
    mkdir -p /data/mysql_mha
    mkdir -p /data/mysql_mha/mhabinlog
    
    vim /etc/mha/mysql_mha.cnf
    
    [server default]
    # mha用于访问数据库的账户和密码
    user=admin
    password='xxxxxxx'
    # 指定mha的工作目录
    manager_workdir=/data/mysql_mha
    # mha日志文件的存放路径
    manager_log=/data/mysql_mha/manager.log
    # 指定mha在远程节点上的工作目录
    remote_workdir=/data/mysql_mha
    # 可以使用ssh登录的用户
    ssh_user=root
    # 用于主从复制的MySQL用户和密码
    repl_user=repl
    repl_password="xxxxxxx"
    # 指定间隔多少秒检测一次
    ping_interval=1
    # 指定master节点存放binlog日志文件的目录
    master_binlog_dir=/data/mysql_mha/mhabinlog
    # 指定一个脚本，该脚本实现了在主从切换之后，将虚拟IP漂移到新的Master上
    master_ip_failover_script=/data/mysql_mha/master_ip_failover
    # 指定用于二次检查节点状态的脚本
    secondary_check_script=/usr/bin/masterha_secondary_check -s 192.168.136.140 -s 192.168.136.141 -s 192.168.136.142
    
    # 配置集群中的节点信息
    [server1]
    hostname=192.168.136.140
    port=3306
    # 指定该节点可以参与Master选举
    candidate_master=1
    
    [server2]
    hostname=192.168.136.141
    port=3306
    candidate_master=1
    
    [server3]
    hostname=192.168.136.142
    port=3306
    candidate_master=1
    # 指定该节点不能参与Master选举
    #no_master=1
    
    ```

5.  **编辑虚拟IP脚本`vim /data/mysql_mha/master_ip_failover`**：

    ```shell
    vim /data/mysql_mha/master_ip_failover
    
    #!/usr/bin/env perl
    
    use strict;
    use warnings FATAL => 'all';
    
    use Getopt::Long;
    
    my (
        $command, $orig_master_host, $orig_master_ip,$ssh_user,
        $orig_master_port, $new_master_host, $new_master_ip,$new_master_port,
        $orig_master_ssh_port,$new_master_ssh_port,$new_master_user,$new_master_password
    );
    
    # 这里定义的虚拟IP可以根据实际情况进行修改
    my $vip = '192.168.136.145/24';
    my $key = '1';
    # 这里的网卡名称 “ens33” 需要根据你机器的网卡名称进行修改
    my $ssh_start_vip = "sudo /sbin/ifconfig ens33:$key $vip";
    my $ssh_stop_vip = "sudo /sbin/ifconfig ens33:$key down";
    my $ssh_Bcast_arp= "sudo /sbin/arping -I bond0 -c 3 -A $vip";
    
    GetOptions(
        'command=s'          => \$command,
        'ssh_user=s'         => \$ssh_user,
        'orig_master_host=s' => \$orig_master_host,
        'orig_master_ip=s'   => \$orig_master_ip,
        'orig_master_port=i' => \$orig_master_port,
        'orig_master_ssh_port=i' => \$orig_master_ssh_port,
        'new_master_host=s'  => \$new_master_host,
        'new_master_ip=s'    => \$new_master_ip,
        'new_master_port=i'  => \$new_master_port,
        'new_master_ssh_port' => \$new_master_ssh_port,
        'new_master_user' => \$new_master_user,
        'new_master_password' => \$new_master_password
    
    );
    
    exit &main();
    
    sub main {
        $ssh_user = defined $ssh_user ? $ssh_user : 'root';
        print "\n\nIN SCRIPT TEST====$ssh_user|$ssh_stop_vip==$ssh_user|$ssh_start_vip===\n\n";
    
        if ( $command eq "stop" || $command eq "stopssh" ) {
    
            my $exit_code = 1;
            eval {
                print "Disabling the VIP on old master: $orig_master_host \n";
                &stop_vip();
                $exit_code = 0;
            };
            if ($@) {
                warn "Got Error: $@\n";
                exit $exit_code;
            }
            exit $exit_code;
        }
        elsif ( $command eq "start" ) {
    
            my $exit_code = 10;
            eval {
                print "Enabling the VIP - $vip on the new master - $new_master_host \n";
                &start_vip();
         &start_arp();
                $exit_code = 0;
            };
            if ($@) {
                warn $@;
                exit $exit_code;
            }
            exit $exit_code;
        }
        elsif ( $command eq "status" ) {
            print "Checking the Status of the script.. OK \n";
            exit 0;
        }
        else {
            &usage();
            exit 1;
        }
    }
    
    sub start_vip() {
        `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
    }
    sub stop_vip() {
        `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
    }
    
    sub start_arp() {
        `ssh $ssh_user\@$new_master_host \" $ssh_Bcast_arp \"`;
    }
    sub usage {
        print
        "Usage: master_ip_failover --command=start|stop|stopssh|status --ssh_user=user --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
    }
    ```

6. **测试 MHA 环境**：

    ```shell
    # 检测SSH是否OK
    masterha_check_ssh --conf=/etc/mha/mysql_mha.cnf
    ```

    ```shell
    # 检测配置文件是否OK
    masterha_check_repl --conf=/etc/mha/mysql_mha.cnf
    ```

7. **启动 MHA 管理进程**：

    ```shell
    # 启动MHA
    nohup masterha_manager --conf=/etc/mha/mysql_mha.cnf > /data/mysql_mha/manager.log < /dev/null 2>&1 &
    
    # 第一次启动需要手动在master节点手工加上虚拟IP
    ifconfig ens33:1 192.168.136.140/24
    
    # 关闭MHA
    masterha_stop  --conf=/etc/mha/mysql_mha.cnf
    ```

8. **MHA运行状态**

    ```shell
    # 检查mha 运行状态
    masterha_check_status --conf=/etc/mha/mysql_mha.cnf
     
    mysql_mha (pid:1918) is running(0:PING_OK), master:192.168.136.140
    
    # 查看进程
    ps aux |grep masterha_manager
    root       2243  1.1  1.1 301836 22064 pts/0    S    01:12   0:00 perl /usr/bin/masterha_manager --conf=/etc/mha/mysql_mha.cnf
    root       2282  0.0  0.0 112824   984 pts/0    R+   01:13   0:00 grep --color=auto masterha_manager
    ```

9. **手工切换主**

    ```shell
    # 手工切换需要关闭MHA监控，如果MHA监控正在运行是不允许手工切换的
    masterha_stop --conf=/etc/mha/mysql_mha.cnf
    
    masterha_master_switch --conf=/etc/mha/mysql_mha.cnf --master_state=alive --new_master_host=192.168.136.141 --new_master_port=3306 --orig_master_is_new_slave --running_updates_limit=10000
    # orig_master_is_new_slave是将原master切换为新主的slave，默认情况下，是不添加的。
    # running_updates_limit默认为1s，即如果主从延迟时间（Seconds_Behind_Master），或master show processlist中dml操作大于1s，则不会执行切换。
    # 会出现两次提示，需要填yes/no，填yes即可。
    ```

10. **手工failover**

    ```shell
    # 在主库存在问题的情况下
    # 在manager节点再进行failover切换
    masterha_master_switch --conf=/etc/mha/mysql_mha.cnf --master_state=dead --dead_master_host=192.168.136.140 --dead_master_port=3306 --new_master_host=192.168.136.141 --new_master_port=3306 --ignore_last_failover
    # 有两次提示，都输入yes
    ```

    

### 结论

MHA 作为 MySQL 主从复制环境下的高可用解决方案，具有自动化、低成本和稳定的优点，但在应用场景和规模上有一定限制。对于中小规模、对数据一致性要求高的单主环境，MHA 是一个非常优秀的选择。然而，随着业务需求的多样化和复杂化，在更大规模或多写需求下，可能需要考虑其他高可用方案。
