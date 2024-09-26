# MYSQL高可用MGR详解及搭建

**MySQL Group Replication (MGR) 是 MySQL 提供的一种原生的高可用性和数据一致性解决方案，适用于分布式数据库环境。MGR 通过复制来实现数据库节点之间的数据同步和容错。下面我将详细讲解其工作原理、优缺点及适用场景。**

##  **一. MGR 概述：**

### 一、MGR 的原理

MySQL Group Replication 基于多主复制模型，并使用 **Paxos** 一致性协议来确保所有节点的数据一致性。其核心思想是：

1. **自动故障转移**：MGR 在一个集群中有多个节点，当某个节点发生故障时，集群自动检测并剔除故障节点，剩余节点继续提供服务，无需人工干预。
  
2. **多主模式 (Multi-Primary Mode)**：
   - 在多主模式下，所有节点都可以接受写入操作。写入请求会通过一致性协议分发给其他节点。
   - 写操作会在提交之前被复制到其他节点，只有当大多数节点确认后，事务才会真正提交。这样保证了数据的一致性。

3. **单主模式 (Single-Primary Mode)**：
   - 单主模式下，只有一个主节点负责写操作，其他节点是只读的。这个模式类似于传统的主从复制，但是自动化更强。
  
4. **自动节点加入**：当新节点加入集群时，它会自动从现有的组中获取数据并完成同步，无需手动介入。

5. **故障检测与恢复**：MGR 通过内部的监控机制实时检测节点的健康状态。如果一个节点不可用，其他节点会自动感知，并重新选举新的主节点（如果使用单主模式）。

### 二、MGR 的优点

1. **高可用性**：
   - 集群中的每个节点都是冗余的，因此某个节点的故障不会导致服务中断。
   
2. **数据一致性**：
   - 基于一致性协议（ Paxos），保证所有节点的数据一致性，即使在网络分区或节点故障的情况下，也能确保最终一致性。
   
3. **多主写入能力**：
   - 在多主模式下，多个节点可以同时接受写请求，适合高并发写入场景。
   
4. **自动化**：
   - 节点自动检测故障并恢复，无需 DBA 进行手动干预。新节点也可以自动加入集群并进行数据同步。

5. **扩展性**：
   - 集群中可以通过增加节点来扩展读写性能，尤其是在多主模式下，写入性能可以分布到多个节点。
   
6. **与 MySQL 内核集成**：
   - MGR 是 MySQL 原生提供的功能，集成度高，配置简单，无需额外的插件或扩展。

### 三、MGR 的缺点

1. **写冲突问题**：
   - 在多主模式下，由于多个节点可以同时写入，可能会出现写冲突问题。冲突发生时，MGR 会回滚冲突的事务，可能影响性能。
   
2. **写入延迟**：
   - 由于 MGR 需要等待大多数节点确认事务写入，尤其在跨地域部署时，网络延迟会导致写入性能下降。
   
3. **部署复杂性**：
   - MGR 的部署和配置相较于传统的主从复制要复杂一些，特别是对于集群的网络配置和一致性协议的理解有一定的要求。
   
4. **网络依赖性强**：
   - MGR 依赖可靠的网络连接，一旦集群中的网络发生问题（例如网络分区），可能导致集群不可用或降级为只读模式。

5. **不适合低延迟高写场景**：
   - 由于写操作需要等待大多数节点确认，因此在对延迟敏感的应用中，MGR 的性能可能不如传统的单主架构。

### 四、MGR 的适用场景

1. **高可用性要求高的业务**：
   - 如果业务需要一个高度可用的数据库架构，能够自动恢复和容错，MGR 是一个理想选择。

2. **多写场景**：
   - 对于需要多个地理位置的数据中心同时进行写操作的业务，MGR 的多主模式可以很好地支持这种场景。

3. **读多写少的场景**：
   - 如果读请求量较多，可以通过增加 MGR 集群中的节点来提升读性能；而写入量较小的场景下，MGR 的一致性协议对写性能的影响也较小。

4. **需要分布式数据的一致性**：
   - 对于对数据一致性要求高的分布式数据库应用（如银行、金融系统），MGR 能够很好地保证数据一致性。

5. **DevOps 场景**：
   - MGR 的自动化特点，使其非常适合需要频繁扩展或调整数据库规模的 DevOps 场景，减少了 DBA 的管理负担。
   

----------------------------------------------------------------------------------------------------

##  **二.MGR实战搭建**

###  一. 基础环境信息

##### 1.1主机信息

| 主机名  |      IP       | OS系统     | CPU  | 内存 | 磁盘 |    系统角色    |    端口    | 安装软件    |
| :-----: | :-----------: | ---------- | :--: | :--: | :--: | :------------: | :--------: | ----------- |
|  node1  | 192.168.13.41 | redhat 8.6 |  4   |  8   | 300  |    Primary     | 3306,33061 | MySQL       |
|  node2  | 192.168.13.42 | redhat 8.6 |  4   |  8   | 300  |   Secondary    | 3306,33061 | MySQL       |
|  node3  | 192.168.13.43 | redhat 8.6 |  4   |  8   | 300  |   Secondary    | 3306,33061 | MySQL       |
| router1 | 192.168.13.44 | redhat 8.6 |  2   |  4   | 200  | router\_master | 6446,6447  | MySQLRouter |

##### 1.2软件安装清单

| 所需要安装软件 |  版本  | 下载地址                                                     |
| :------------: | :----: | ------------------------------------------------------------ |
|  MySQL Server  | 8.0.33 | https://downloads.mysql.com/archives/get/p/23/file/mysql-8.0.33-el7-x86_64.tar |
|  MySQL Shell   | 8.0.33 | https://downloads.mysql.com/archives/get/p/43/file/mysql-shell-8.0.33-linux-glibc2.12-x86-64bit.tar.gz |
|  MySQL Router  | 8.0.33 | https://downloads.mysql.com/archives/get/p/41/file/mysql-router-8.0.33-linux-glibc2.28-x86_64.tar.gz |
### 二. 环境配置

##### 2.1修改主机名

```bash
#五台服务器分别执行：
hostnamectl set-hostname node1
hostnamectl set-hostname node2
hostnamectl set-hostname node3
hostnamectl set-hostname router1

cat << EOF >> /etc/hosts
192.168.13.41  node1
192.168.13.42  node2
192.168.13.43  node3
192.168.13.44  router1
EOF
```
##### 2.2 清空防火墙规则

```bash
iptables -F
-- 开启防火墙端口
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --zone=public --add-port=33061/tcp --permanent
firewall-cmd --reload
```
##### 2.3关闭SELinux

```bash
setenforce 0
#或者
vim /etc/sysconfig/selinux
SELINUX=disabled
```
##### 2.4设置系统limit

```bash
vi /etc/security/limits.conf
#modify by 2024-03-11
* soft nofile 65536
* hard nofile 65536
* soft nproc 65536
* hard nproc 65536 

vi /etc/security/limits.d/20-nproc.conf 
#modify by 2024-03-11
#*         soft    nproc     4096
*          soft    nproc     65536
```
##### 2.5 将下载好的安装包放在服务器的/usr/local

```bash
cd /usr/local/
#  查看安装包情况

ls -lsrth 
```
##### 2.6重启服务器

```bash
reboot
```
### 三. RPM安装mysql

##### 3.1.创建操作系统用户

```bash
groupadd mysql
useradd -g mysql mysql
```
  注意：这里可以是其它用户名

##### 3.2解压RPM包

```bash
cd /usr/local/
mkdir mysql-8.0.33
tar -xvf mysql-8.0.33-1.el8.x86_64.rpm-bundle.tar -C mysql-8.0.33/
cd /usr/local/mysql-8.0.33/
```
##### 3.3安装依赖

```bash
yum install -y  libaio openssl-devel  perl 
```
##### 3.4安装mysql 8.0.33 RPM包

```bash
rpm -ivh mysql-community-common-8.0.33-1.el8.x86_64.rpm
rpm -ivh mysql-community-client-plugins-8.0.33-1.el8.x86_64.rpm
rpm -ivh mysql-community-libs-8.0.33-1.el8.x86_64.rpm 
rpm -ivh mysql-community-icu-data-files-8.0.33-1.el8.x86_64.rpm
rpm -ivh mysql-community-devel-8.0.33-1.el8.x86_64.rpm
rpm -ivh mysql-community-client-8.0.33-1.el8.x86_64.rpm
rpm -ivh mysql-community-server-8.0.33-1.el8.x86_64.rpm
```
##### 3.5编辑配置文件

```bash
vim /etc/my.cnf

[client]
socket=/data/mysqldata/mysql.sock
port=3306

[mysqld]

#dir
basedir=/usr
datadir=/data/mysqldata
socket=/data/mysqldata/mysql.sock
log_error=/data/mysqldata/mysql.err
pid-file =/data/mysqldata/mysql.pid

#server
server-id=1
user=mysql
log_timestamps=system
#字符集
character-set-server = utf8mb4
#只能用IP地址检查客户端的登录，不用主机名
skip_name_resolve = 1

#binlog
binlog_format = ROW
#一般数据库中没什么大的事务，设成1~2M，默认32kb
binlog_cache_size = 4M
#binlog 能够使用的最大cache 内存大小
max_binlog_cache_size = 2G
#单个binlog 文件大小 默认值是1GB
max_binlog_size = 1G
#binlog 过期天数
expire_logs_days = 15

#GTID
gtid_mode = on
enforce_gtid_consistency = 1

#innodb_buffer
#一般设置物理存储的 50% ~ 70%
innodb_buffer_pool_size = 5G
#当缓冲池大小大于1GB时，将innodb_buffer_pool_instances设置为大于1的值，可以提高繁忙服务器的可伸缩性
innodb_buffer_pool_instances = 8

#双一刷盘设置
#控制 innodb_flush_log_at_trx_commit redolog 写磁盘频率  sync_binlog 默认为1 #控制 binlog 写磁盘频率
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
#从库binlog控制
log_replica_updates = ON

# 表名SQL大小写(是否对sql语句大小写敏感，1表示不敏感 0 )
lower_case_table_names = 1

#Replication
master_info_repository =TABLE
relay_log_info_repository =TABLE
#super_read_ony =ON
binlog_transaction_dependency_tracking =WRITESET
transaction_write_set_extraction =XXHASH64

#Multi-threaded Replication
replica_parallel_type =LOGICAL_CLOCK
replica_preserve_commit_order =ON
replica_parallel_workers = 4

## Group Replication
report_host = "192.168.13.41"
disabled_storage_engines = "MyISAM,BLACKHOLE,FEDERATED,ARCHIVE,MEMORY"
sql_require_primary_key = ON
#同一个Group Replication 一组设置为同一个
loose-group_replication_group_name = 'f8896e9a-0eab-11ef-8ca2-0050569d3295'
loose-group_replication_start_on_boot = off
loose-group_replication_local_address = '192.168.13.41:33061'
loose-group_replication_group_seeds = '192.168.13.41:33061,192.168.13.42:33061,192.168.13.43:33061'
loose-group_replication_bootstrap_group = off
# 安装完MGR插件需要启用此参数。
# group_replication_recovery_get_public_key = ON

#是否启用慢查询日志，1为启用，0为禁用  
slow_query_log = 1
#指定慢查询日志文件的路径和名字
slow_query_log_file =/data/mysqldata/slow.log
#慢查询执行的秒数，必须达到此值可被记录
long_query_time = 2
#将没有使用索引的语句记录到慢查询日志  
log_queries_not_using_indexes = 0
#设定每分钟记录到日志的未使用索引的语句数目，超过这个数目后只记录语句数量和花费的总时间  
log_throttle_queries_not_using_indexes = 60
#对于查询扫描行数小于此参数的SQL，将不会记录到慢查询日志中
min_examined_row_limit = 5000
#记录执行缓慢的管理SQL，如alter table,analyze table, check table, create index, drop index, optimize table, repair table等。  
log_slow_admin_statements = 0


[mysqldump]
quick
max_allowed_packet = 128M

```
##### 3.6其余两个节点服务器分别修改

```bash
2节点需要修改：
server-id = 2
report_host = "192.168.13.42"
loose-group_replication_local_address = '192.168.13.42:33061'

3节点需要修改：
server-id = 3
report_host = "192.168.13.43"
loose-group_replication_local_address = '192.168.13.43:33061'
```
##### 3.7创建数据目录 并修改其属主和组

```bash
mkdir -p /data/mysqldata
chown -R mysql.mysql /data/mysqldata
```
##### 3.8登录实例

* 先找初始实例日志的临时密码

  ```shell
  systemctl start mysqld
  
  grep password /data/mysqldata/mysql.err
  ```

* 登录数据库,登陆后必须修改密码

  ```shell
  mysql -uroot -p 
  mysql> alter user user() identified by 'XXXXXXXXXX';
  ```

* 创建MYSQL 数据库管理用户

  ```shell
  -- 管理账号
  CREATE USER dbadmin@'%' IDENTIFIED BY 'XXXXXXXXXX';
  GRANT  all  PRIVILEGES ON *.* TO dbadmin@'%' WITH GRANT OPTION; 
  FLUSH PRIVILEGES;
  ```

##### 3.9配置服务

```sql
# 重新载入
systemctl daemon-reload
# 设置开机自启动
systemctl enable mysqld

# 重启mysql 服务
systemctl restart mysqld
systemctl status mysqld
systemctl stop   mysqld
systemctl start  mysqld
```
### 四. 安装mysql shell

##### 4.1下载解压 并创建软连接

```sql
cd /usr/local
tar -xvf mysql-shell-8.0.33-linux-glibc2.12-x86-64bit.tar.gz 
ln -s mysql-shell-8.0.33-linux-glibc2.12-x86-64bit mysql-shell
```
##### 4.2加入变量

```bash
echo 'export PATH=/usr/local/mysql-shell/bin:$PATH' >> /etc/profile
source /etc/profile
```
##### 4.3使用MYSQL SHELL

```bash
mysqlsh
\connect   "dbadmin@192.168.13.41:3306"
\sql
\js
\q
```
** **

### 五.MYSQL MGR配置

##### 5.1 每个节点都需要安装

**安装插件**

```sql
install plugin group_replication soname 'group_replication.so';
show plugins;
```
##### 5.2 创建复制用户

```sql
-- 所有节点都需要创建。
-- MGR复制账号
CREATE USER repl_uer@'192.168.13.%' IDENTIFIED  BY 'XXXXXXXXXX';
GRANT REPLICATION SLAVE ON *.* TO repl_uer@'192.168.13.%';
FLUSH PRIVILEGES;

-- 建议 初次安装 对所有节点执行重置操作。 
reset master ;
reset slave all;
```
##### 5.3 初始化组复制

需要在主要节点上执行

```sql
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;

-- 查看节点信息
SELECT * FROM performance_schema.replication_group_members;
```
##### 5.4 配置恢复通道

```sql
CHANGE MASTER TO MASTER_USER='repl_uer', MASTER_PASSWORD='XXXXXXXXXX'  FOR CHANNEL 'group_replication_recovery';
```
##### 5.5 添加节点

依次 在节点 2 节点 3 执行 

```sql
-- 配置恢复通道
CHANGE MASTER TO MASTER_USER='repl_uer', MASTER_PASSWORD='XXXXXXXXXX'  FOR CHANNEL 'group_replication_recovery';

-- 启动组复制
START GROUP_REPLICATION;

--  添加节点成功
SELECT * FROM performance_schema.replication_group_members;

-- 查看节点状态 
select * from   performance_schema.replication_connection_status\G;
```
##### 5.6 使用mysqlsh 检查节点情况

```bash
-- 检查节点 是否 满足Innodb Cluster 条件
 dba.checkInstanceConfiguration('dbadmin@192.168.13.41:3306')
 dba.checkInstanceConfiguration('dbadmin@192.168.13.42:3306')
 dba.checkInstanceConfiguration('dbadmin@192.168.13.43:3306')
```
##### 5.7 基于现有MGR使用mysqlsh 管理集群

```bash
var cluster = dba.createCluster('mycluster', {adoptFromGR: true});
```
##### 5.8 使用mysqlsh查看MGR状态

```bash
MySQLSH JS > var cluster=dba.getCluster()
MySQLSH JS > cluster.status()
```
### 六.MYSQL Router 安装配置

##### 6.1 下载解压 并创建软连接

```sql
cd /usr/local
tar -xvf mysql-router-8.0.33-linux-glibc2.28-x86_64.tar.gz
ln -s mysql-router-8.0.33-linux-glibc2.28-x86_64 mysqlrouter
```
##### 6.2 加入变量

```bash
vim ~/.bash_profile  -- 编辑 当前用户的变量  只是用当前用户登录的环境变量

export PATH=$PATH:/usr/local/mysqlrouter/bin

source  ~/.bash_profile
```
#####  6.3 初始化MYSQL ROUTER

-  初始节点配置

```bash
-- 建立MYSQL用户
groupadd mysql
useradd -g mysql mysql

-- 建立mysql-router 目录
mkdir -p /data/mysqlrouter/

--  授权目录
chown -R mysql.mysql /data/mysqlrouter/
```
- 初始化mysqlrouter

```bash
# 初始化mysqlrouter  只需--report-host 修改为router节点本机ip
mysqlrouter --bootstrap dbadmin@192.168.13.41:3306 --user=mysql --directory /data/mysqlrouter --conf-use-sockets --name mgrrouter --report-host='192.168.13.44'
```
##### 6.4 配置服务启动

systemd启停MySQL Router需要配置启动脚本，以systemd服务为例，我们编辑配置/usr/lib/systemd/system/mysqlrouter.service

```bash
#配置动脚本
vi /usr/lib/systemd/system/mysqlrouter.service

[Unit]
Description=MySQL Router
After=syslog.target
After=network.target
[Service]
Type=simple
User=mysql
Group=mysql
PIDFile=/data/mysqlrouter/mysqlrouter.pid
ExecStart=/usr/local/mysqlrouter/bin/mysqlrouter -c /data/mysqlrouter/mysqlrouter.conf
Restart=on-failure
PrivateTmp=true
[Install]
WantedBy=multi-user.target

```
##### 6.5 服务器启动mysqlrouter

```bash
# systemctl daemon-reload
# systemctl enable mysqlrouter.service
# systemctl status mysqlrouter.service
# systemctl start mysqlrouter.service
```
##### 6.6查看MYSQL 运行情况

```bash
# 查看运行端口
netstat -ntlp |grep mysqlrouter

-- 查看运行状态
ps -ef | grep mysqlrouter | grep -v grep

-- 查看运行状态
systemctl status mysqlrouter.service
```
##### 6.7 mysqlrouter 链接查看

```bash
-- 链接mysql  写端口 （需要有安装mysql 客户端的 链接）
-- 读写端口
mysql -h 192.168.13.44 -P6446 -udbadmin -p'XXXXXXXXXX'
-- 只读端口
mysql -h 192.168.13.44 -P6447 -udbadmin -p'XXXXXXXXXX'
```
##### 6.8 测试端口情况

```bash
[root@node1 ~]# for ((i=0;i<=1;i++));do mysql -h 192.168.13.44 -P6447 -udbadmin -p'XXXXXXXXXX'  -e"select @@hostname;";done;
mysql: [Warning] Using a password on the command line interface can be insecure.
+------------+
| @@hostname |
+------------+
| node3      |
+------------+
mysql: [Warning] Using a password on the command line interface can be insecure.
+------------+
| @@hostname |
+------------+
| node2      |
+------------+
```

###  七. 日常使用方法

#### 7.1 日常登陆

**在任意有MYSQL客户端位置 使用mysql_router端口登录即可。**

```shell
# 读写服务登录
mysql -h 192.168.13.44 -P6446 -udbadmin -p'XXXXXXXXXX'

# 只读服务登录
mysql -h 192.168.13.44 -P6447 -udbadmin -p'XXXXXXXXXX'
```

#### 7.2 查看集群信息

```sql
--  查看节点信息
SELECT * FROM performance_schema.replication_group_members;

-- 查看节点状态 
select * from   performance_schema.replication_connection_status\G;

```

#### 7.3 多主单主切换

```sql
-- 单主模式  主节点手工切换到其它节点
-- member_UUid  必须指定 主机的UUID；
select group_replication_set_as_primary(member_UUid);

-- 在线单主切换为多主
select group_replication_switch_to_multi_primary_mode();

-- 在线多主切换为单主
-- 不指定member_uuid可以根据uuid 大小自动选主。
select group_replication_switch_to_single_primary_mode();

-- 可以指定节点UUID
select group_replication_switch_to_single_primary_mode(member_uuid);

```

