#  InnoDB Cluster搭建实战

[TOC]



# 一. 项目基本情况

## * 概述

MySQL集群主要采用MGR三个数据库节点，两个路由节点，通过keepalived虚拟IP做高可用集群。


![image-20240910144129762](E:\公众号资料\图片素材\image-20240910144129762.png)

## * 基础环境信息
|主机名|VIP|IP|OS系统|CPU|内存|磁盘|系统角色|端口|安装软件|
| ----- | :-----: | :-----: | :-----: | :-----: | :-----: | :-----: | :-----: | :-----: | ----- |
|mysql\_node1|无|192.168.1.41|redhat 8.6|4|8|320|Primary|3306<br>33061|MySQL+MySQL Shell|
|mysql\_node2| |192.168.1.42|redhat 8.6|4|8|320|Secondary|3306<br>33061|MySQL|
|mysql\_node3| |192.168.1.43|redhat 8.6|4|8|320|Secondary|3306<br>33061|MySQL|
|mysql\_router1|192.168.1.46|192.168.1.44|redhat 8.6|2|4|120|router\_master|6446<br>6447|MySQLRouter+Keepalived+MySQL  Shell|
|mysql\_router2| |192.168.1.45|redhat 8.6|2|4|120|router\_slave|6446<br>6447|MySQLRouter+Keepalived+MySQL Shell|

## * 软件安装清单
|所需要安装软件|版本|下载地址|
| ----- | ----- | ----- |
|MySQL Server|8.0.33|https://downloads.mysql.com/archives/community/|
|MySQL Shell|8.0.33|https://downloads.mysql.com/archives/shell/|
|MySQL Router|8.0.33|https://downloads.mysql.com/archives/router/|
|Keepalived|2.2.8|https://www.keepalived.org/download.html/|
# 二. 基础环境配置

## 1. 修改主机名
```bash
#五台服务器分别执行：
hostnamectl set-hostname hqcnocpuatmysqln1
hostnamectl set-hostname hqcnocpuatmysqln2
hostnamectl set-hostname hqcnocpuatmysqln3
hostnamectl set-hostname hqcnocpuatmysqlr1
hostnamectl set-hostname hqcnocpuatmysqlr2

cat << EOF >> /etc/hosts
192.168.1.41  hqcnocpuatmysqln1
192.168.1.42  hqcnocpuatmysqln2
192.168.1.43  hqcnocpuatmysqln3
192.168.1.44  hqcnocpuatmysqlr1
192.168.1.45  hqcnocpuatmysqlr2
EOF

```
## 2. 清空防火墙规则
```bash
iptables -F
-- 开启防火墙端口
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --zone=public --add-port=33061/tcp --permanent
firewall-cmd --reload
```
## 3. 关闭SELinux
```bash
setenforce 0
#或者
vim /etc/sysconfig/selinux
SELINUX=disabled
```
## 4. 设置系统limit
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
## 5. 将下载好的安装包放在服务器的/usr/local
```bash
cd /usr/local/
#  查看安装包情况
ls -lsrth 
```
## 6. 重启服务器
```bash
reboot
```
# 三. RPM安装mysql
## 1. 创建操作系统用户
```bash
groupadd mysql
useradd -g mysql mysql
```
  注意：这里可以是其它用户名

## 2. 解压RPM包
```bash
cd /usr/local/
mkdir mysql-8.0.33
tar -xvf mysql-8.0.33-1.el8.x86_64.rpm-bundle.tar -C mysql-8.0.33/
cd /usr/local/mysql-8.0.33/
```
## 3. 安装依赖
```bash
yum install -y  libaio openssl-devel  perl 
```
## 4. 安装mysql 8.0.33 RPM包
```bash
rpm -ivh mysql-community-common-8.0.33-1.el8.x86_64.rpm
rpm -ivh mysql-community-client-plugins-8.0.33-1.el8.x86_64.rpm
rpm -ivh mysql-community-libs-8.0.33-1.el8.x86_64.rpm 
rpm -ivh mysql-community-icu-data-files-8.0.33-1.el8.x86_64.rpm
rpm -ivh mysql-community-devel-8.0.33-1.el8.x86_64.rpm
rpm -ivh mysql-community-client-8.0.33-1.el8.x86_64.rpm
rpm -ivh mysql-community-server-8.0.33-1.el8.x86_64.rpm
```
## 5. 编辑配置文件
```bash
vim /etc/my.cnf

[client]
socket=/data/mysql/3306/data/mysql.sock
port=3306

[mysqld]

#dir
basedir=/usr
datadir=/data/mysql/3306/data
socket=/data/mysql/3306/data/mysql.sock
log_error=/data/mysql/3306/data/mysql.err
pid-file =/data/mysql/3306/data/mysql.pid

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
innodb_buffer_pool_size = 2G
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
report_host = "192.168.1.41"
disabled_storage_engines = "MyISAM,BLACKHOLE,FEDERATED,ARCHIVE,MEMORY"
sql_require_primary_key = ON
#同一个Group Replication 一组设置为同一个
loose-group_replication_group_name = 'f8896e9a-0eab-11ef-8ca2-0050569d3295'
loose-group_replication_start_on_boot = off
loose-group_replication_local_address = '192.168.1.41:33061'
loose-group_replication_group_seeds = '192.168.1.41:33061,192.168.1.42:33061,192.168.1.43:33061'
loose-group_replication_bootstrap_group = off
# 安装完MGR插件需要启用此参数。
# group_replication_recovery_get_public_key = ON

#是否启用慢查询日志，1为启用，0为禁用  
slow_query_log = 1
#指定慢查询日志文件的路径和名字
slow_query_log_file =/data/mysql/3306/data/slow.log
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
其余两个节点服务器分别修改

```bash
2节点需要修改：
server-id = 2
report_host = "192.168.1.42"
loose-group_replication_local_address = '192.168.1.42:33061'

3节点需要修改：
server-id = 3
report_host = "192.168.1.43"
loose-group_replication_local_address = '192.168.1.43:33061'
```
## 6. 创建数据目录 并修改其属主和组
```bash
mkdir -p /data/mysql/3306/data
chown -R mysql.mysql /data/mysql/3306/data
```
## 7. 登录实例

### * 先找初始实例日志的临时密码
```bash
systemctl start mysqld

grep password /data/mysql/3306/data/mysql.err
```
### * 登录数据库,登陆后必须修改密码
```bash
mysql -uroot -p 
mysql> alter user user() identified by 'XXXXXXXXXX';
```
### * 创建MYSQL 数据库管理用户
```bash
-- INNODB CLUSTER 管理账号
CREATE USER dbadmin@'%' IDENTIFIED with caching_sha2_password BY 'XXXXXXXXXX';
GRANT  all  PRIVILEGES ON *.* TO dbadmin@'%' WITH GRANT OPTION; 
FLUSH PRIVILEGES;

```
## 8. 以服务方式启动
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
# 四. 基于二进制包安装mysql shell
## 1. 下载解压 并创建软连接
```sql
cd /usr/local
tar -xvf mysql-shell-8.0.33-linux-glibc2.12-x86-64bit.tar.gz 
ln -s mysql-shell-8.0.33-linux-glibc2.12-x86-64bit mysql-shell
```
## 2. 加入变量
```bash
vim ~/.bash_profile  -- 编辑 当前用户的变量  只是用当前用户登录的环境变量

export PATH=$PATH:/usr/local/mysql-shell/bin

source  ~/.bash_profile
```
## 3. 使用MYSQL SHELL
```bash
mysqlsh
\connect   "dbadmin@192.168.1.41:3306"
\sql
\js
\q
```
# 五.MYSQL MGR配置
## 1. 每个节点都需要安装
```sql
install plugin group_replication soname 'group_replication.so';
show plugins;
```
## 2. 创建复制用户
```sql
-- 所有节点都需要创建。
-- MGR复制账号
CREATE USER repl_uer@'192.168.1.%' IDENTIFIED with caching_sha2_password BY 'XXXXXXXXXX';
GRANT REPLICATION SLAVE ON *.* TO repl_uer@'192.168.1.%';
FLUSH PRIVILEGES;

-- 建议 初次 对所有节点 
reset master ;
reset slave all;
```
## 3. 初始化组复制
需要在主要节点上执行

```sql
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;

-- 查看节点信息
SELECT * FROM performance_schema.replication_group_members;
```
## 4. 配置恢复通道
```sql
CHANGE MASTER TO MASTER_USER='repl_uer', MASTER_PASSWORD='XXXXXXXXXX'  FOR CHANNEL 'group_replication_recovery';
```
## 5. 添加节点
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
## 6. 使用mysqlsh 检查节点情况
```bash
-- 检查节点 是否 满足Innodb Cluster 条件
 dba.checkInstanceConfiguration('dbadmin@192.168.1.41:3306')
 dba.checkInstanceConfiguration('dbadmin@192.168.1.42:3306')
 dba.checkInstanceConfiguration('dbadmin@192.168.1.43:3306')
```
## 7. 基于现有MGR使用mysqlsh 管理集群
```bash
var cluster = dba.createCluster('mycluster', {adoptFromGR: true});
```
## 8. 使用mysqlsh查看MGR状态

```bash
MySQL  192.168.1.35:3306 ssl  JS > var cluster=dba.getCluster()
MySQL  192.168.1.35:3306 ssl  JS > cluster.status()
```
# 六.MYSQL Router 安装配置
## 1. 下载解压 并创建软连接
```sql
cd /usr/local
tar -xvf mysql-router-8.0.33-linux-glibc2.28-x86_64.tar.gz
ln -s mysql-router-8.0.33-linux-glibc2.28-x86_64 mysqlrouter
```
## 2. 加入变量
```bash
vim ~/.bash_profile  -- 编辑 当前用户的变量  只是用当前用户登录的环境变量

export PATH=$PATH:/usr/local/mysqlrouter/bin

source  ~/.bash_profile
```
## 3. .初始化MYSQL ROUTER

1. 初始节本配置

```bash
-- 建立MYSQL用户
groupadd mysql
useradd -g mysql mysql

-- 建立mysql-router 目录
mkdir -p /data/mysqlrouter/

--  授权目录
chown -R mysql.mysql /data/mysqlrouter/
```
2. 初始化mysqlrouter

```bash
# 初始化mysqlrouter  两个router 都要执行  只需--report-host 修改为router 节点本机ip
mysqlrouter --bootstrap dbadmin@192.168.1.41:3306 --user=mysql --directory /data/mysqlrouter --conf-use-sockets --name mgrrouter --report-host='192.168.1.44'
```
3. 配置服务启动
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
4. 服务器启动mysqlrouter

```bash
# systemctl daemon-reload
# systemctl enable mysqlrouter.service
# systemctl status mysqlrouter.service
# systemctl start mysqlrouter.service
```
5. 查看MYSQL 运行情况

```bash
# 查看运行端口
netstat -ntlp |grep mysqlrouter

-- 查看运行状态
ps -ef | grep mysqlrouter | grep -v grep

-- 查看运行状态
systemctl status mysqlrouter.service
```
6. mysqlrouter 链接查看

```bash
-- 链接mysql  写端口 （需要有安装mysql 客户端的 链接）
-- 读写端口
mysql -h 192.168.1.44 -P6446 -udbadmin -p'XXXXXXXXXX'
-- 只读端口
mysql -h 192.168.1.44 -P6447 -udbadmin -p'XXXXXXXXXX'
```
7. 测试端口情况

```bash
[root@node1 ~]# for ((i=0;i<=1;i++));do mysql -h 192.168.1.38 -P6447 -udbadmin -p'x#d!897L18m7N'  -e"select @@hostname;";done;
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
# 七.RPM 安装Keepalived

## 1. 解压安装包

```bash
yum  install -y  keepalived
```
## 2. 配置服务
```bash
systemctl enable keepalived
systemctl daemon-reload
```
## 3. 配置文件修改
```bash
vim /etc/keepalived/keepalived.conf
```
修改配置文件

```bash

global_defs {
     notification_email {
   }
   notification_email_from dba@dbserver.com
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id MYROUTER-HA
   enable_script_security
   script_user root
}
vrrp_script check_running
{
    script "/etc/keepalived/check_mysqlrouter.sh"
    #轮询间隔要大于check_mysqlrouter.sh脚本执行时间
    interval 3
    weight 20
    fall 2
}
vrrp_instance VI_1 {
    #主节点配置，备份节点修改为 BACKUP
    state MASTER
    #虚拟IP绑定的网卡名称
    interface ens192
    virtual_router_id 51
    priority 98
    advert_int 1
    nopreempt
  authentication {
        auth_type PASS
        auth_pass 1024
    }
    #虚拟IP地址
    virtual_ipaddress {
        192.168.1.46
    }
    #执行的脚本，对应 vrrp_script
   track_script {
        check_running
   }
}
```
vim  /etc/keepalived/check_mysqlrouter.sh

```bash
#!/bin/bash
CHECK_TIME=3

#mysql router is working STATUS_OK is 1 ,  down STATUS_OK is 0  
STATUS_OK=1

function check_mysqlrouter_health (){
MYROUTER_PROCESS=`ps -ef|grep -i mysqlrouter.conf |grep -v grep | awk '{print $2}'|wc -l `
if [[ ${MYROUTER_PROCESS} -eq  1 ]] ;then
     STATUS_OK=1
else
     STATUS_OK=0
fi
     return $STATUS_OK
}
while [[ $CHECK_TIME -ne 0 ]]
do
     let "CHECK_TIME-=1"
     check_mysqlrouter_health

     if [[ $STATUS_OK = 1 ]] ; then
          CHECK_TIME=0
          exit 0
     fi

     if [[ $STATUS_OK -eq 0 ]] &&  [[ $CHECK_TIME -eq 0 ]]
     then
        kill -9 $(ps -ef | grep [k]eepalived)
        exit 1
     fi
     sleep 1
done
```
* 注意 需要给 /etc/keepalived/check\_mysqlrouter.sh 脚本为授权。

```bash
 chmod u+x /etc/keepalived/check_mysqlrouter.sh
```
## 4. 启动keepAlived
```bash
systemctl start keepalived
```
## 5. 验证mysql router 高可用

```bash
-- 关掉 两台 mysql router 中 master 查看 ip 是否漂移至backup
mysql -h 192.168.1.44 -P6446 -udbadmin -p'xxxxxxxxxx' 
```



# 八. 日常运维 InnoDB Cluster

---

## 1. **节点管理**

### 1.1 **集群状态查看**：

使用 `mysqlsh` 工具，您可以检查集群的健康状态：

```bash
mysqlsh --uri admin@192.168.1.41:3306 --password
cluster = dba.getCluster()
cluster.status()
```
**输出示例**：
```json
{
    "clusterName": "myCluster",
    "defaultReplicaSet": {
        "name": "default",
        "status": "OK",
        "topology": {
            "192.168.1.41:3306": {
                "status": "ONLINE",
                "role": "PRIMARY"
            },
            "192.168.1.42:3306": {
                "status": "ONLINE",
                "role": "SECONDARY"
            },
            "192.168.1.43:3306": {
                "status": "ONLINE",
                "role": "SECONDARY"
            }
        }
    }
}
```
状态应为 "OK" 且所有节点应为 "ONLINE"。如果某个节点离线，需进行进一步操作。

### 1.2 **节点恢复**：

当某个节点出现故障时，需将其重新加入集群。首先确认节点的状态，然后通过以下命令恢复：

```bash
cluster.rejoinInstance('admin@192.168.1.42:3306')
```
如果恢复失败，检查日志 `/var/log/mysql/error.log`，排查原因。

### 1.3 **主节点手动切换**：

在需要主动进行主节点切换（例如系统维护或负载均衡需求）时，可以手动设置新主节点：

```bash
cluster.setPrimaryInstance('admin@192.168.1.42:3306')
```
切换后，集群的主从角色会自动重新调整。

---

## 2. **MySQL Shell 基础管理操作**

### 2.1 **连接到 MySQL Server**：

可以通过 MySQL Shell 连接到集群或单个节点，命令如下：

```bash
mysqlsh admin@192.168.1.41:3306 --password
```
MySQL Shell 提供了 SQL、Python 和 JavaScript 三种模式，使用 `\sql`、`\js`、`\py` 进行切换。

### 2.2 **查看集群状态**：

```sqlite
\sql
-- 查看节点状态
SELECT * FROM performance_schema.replication_group_members;
```

### 2.3 **查看集群配置**：

```sqlite
\sql

SHOW VARIABLES LIKE '%GROUP%';
```

### 2.4 **检查复制健康状况**：

```sqlite
\sql
-- 查看节点状态 
select * from   performance_schema.replication_connection_status\G;
```

---

## 3. **集群管理操作**

### 3.1 **集群配置检查**：

```js
cluster = dba.getCluster('myCluster')
cluster.status()
```

### 3.2 **检查节点**：

检查当前节点是否满足`Innodb Cluster` 条件

```shell
# 检查节点 是否 满足Innodb Cluster 条件
MySQL  JS >dba.checkInstanceConfiguration('admin@192.168.1.44:3306')
```

### 3.3 **配置节点**：

使用·mysql shell·远程设置实例配置，增加innodb cluster节点相关配置

```shell
#  使用mysql shell远程设置实例配置
MySQL  JS > dba.configureInstance('admin@192.168.1.44:3306')
```

### 3.4 **新增节点**：

将新节点添加到集群中。首先确保新节点已安装 MySQL 并且配置与主节点一致：

```js
cluster.addInstance('admin@192.168.1.44:3306')
```

### 3.5 **移除节点**：

当某个节点需要下线维护时，可以使用以下命令将其移出集群：

```js
cluster.removeInstance('admin@192.168.1.43:3306')
```



## 4. **MySQL Shell 高级操作**

### 4.1 **集群备份与恢复**：

通过 MySQL Shell 支持的 `dumpInstance()` 函数，您可以进行快速备份：

```js
dba.dumpInstance('/backup/fullbackup')
```

恢复时可以使用：
```js
dba.loadDump('/backup/fullbackup')
```

### 4.2 **脚本化管理集群**：

利用 MySQL Shell 提供的 JavaScript 和 Python API，您可以编写自动化脚本管理集群。例如，自动恢复失效节点：

```js
shell.connect('admin@192.168.1.41')
var cluster = dba.getCluster()
cluster.rejoinInstance('admin@192.168.1.42')
```

### 4.3 **批量操作**：

MySQL Shell 允许通过脚本快速进行批量管理操作。例如，批量查询所有实例的状态：

```js
var cluster = dba.getCluster()
var instances = cluster.status().defaultReplicaSet.topology
for (var instance in instances) {
    print(instance, instances[instance].status)
}
```

---

### 总结

通过细致的 InnoDB Cluster 运维管理和性能优化措施，可以确保 MySQL 集群长期高效、稳定运行。MySQL Shell 提供的强大命令行工具和 API 能帮助 DBA 快速管理和自动化日常任务，减少人为错误，提升运维效率。

