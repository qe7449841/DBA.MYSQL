# Mysql双主+keepalived实现故障自动切换

## 一. 项目基本情况

### * 项目概述

**项目要求：** **MySQL业务搭建双主模式服务+keepalived 实现故障自动切换**

### * 基础环境信息

|主机名|VIP|IP|OS系统|CPU|内存|磁盘|系统角色|端口|安装软件|
| ----- | :-----: | :-----: | :-----: | :-----: | :-----: | :-----: | :-----: | :-----: | ----- |
|mysql-01|10.28.3.194|10.28.3.195|CentOS Linux 7.9|4|16|200|master-01|3306|MySQL Server|
|mysql-02| 10.28.3.194 |10.28.3.196|CentOS Linux 7.9|4|16|200|master-02|3306|MySQL Server|

### * 软件安装清单

|所需要安装软件|版本|下载地址|
| :-----: | :-----: | :-----: |
|MySQL Server|8.0.33|[downloads.mysql](https://downloads.mysql.com/archives/get/p/23/file/mysql-8.0.33-linux-glibc2.17-x86_64-minimal.tar)|
|keepalived|1.3.5|yum 安装 |

### * 架构情况

![image](images/pp8MrS2O62ZbKkQTTfaVCg7nfKtvIQphxjbSQdXp6pU.png)



## 二. 基础环境配置

需要在两台服务器上都执行。

### 1. 修改主机名

```bash
#2台服务器分别执行：
cat << EOF >> /etc/hosts
10.28.3.195  mysql-01
10.28.3.196  mysql-02
EOF
```
### 2. 关闭SELinux

```bash
setenforce 0
```
### 3. 将下载好的安装包放在服务器的/usr/local

```bash
cd /data/
#  查看安装包情况
ls -lsrth 
-- 解压到 /usr/local
tar -vxf /data/mysql-8.0.33-linux-glibc2.17-x86_64-minimal.tar.xz -C /usr/local/
```
## 三. 安装mysql（二进制安装）

### 1. 创建操作系统用户

```bash
groupadd mysql
useradd -g mysql mysql
```
  注意：这里可以是其它用户名

### 2. 解压包建立软连接

```bash
cd /usr/local/
ln -s mysql-8.0.33-linux-glibc2.17-x86_64-minimal/ mysql
chown -R mysql.mysql /usr/local/mysql
chown -R mysql.mysql /usr/local/mysql-8.0.33-linux-glibc2.17-x86_64-minimal/
```
### 3. 编辑配置文件

```bash
vim /etc/my.cnf
```
增加配置文件

```bash
[client]
socket=/data/mysql/data/mysql.sock
port=3306

[mysqld]

#dir
basedir=/usr/local/mysql
datadir=/data/mysql/data
socket=/data/mysql/data/mysql.sock
log_error=/data/mysql/data/mysql.err
pid-file =/data/mysql/data/mysql.pid

#server_info
server-id=1
user=mysql
log_timestamps=system

#connection_info
#最大连接数
max_connections = 3000
#最大错误连接数
max_connect_errors = 10000
#MySQL默认的wait_timeout  值为8个小时, interactive_timeout参数需要同时配置才能生效
interactive_timeout = 3600
wait_timeout = 3600

#字符集
character-set-server = utf8mb4
#只能用IP地址检查客户端的登录，不用主机名
skip_name_resolve = 1

#binlog
binlog_format = ROW
#如果设置为MINIMAL，则会减少记录日志的内容，只记录受影响的列，但对于部分update无法flashBack
binlog_row_image = FULL
#一般数据库中没什么大的事务，设成1~2M，默认32kb
binlog_cache_size = 4M
#binlog 能够使用的最大cache 内存大小
max_binlog_cache_size = 2G
#单个binlog 文件大小 默认值是1GB
max_binlog_size = 1G
#binlog 过期天数7
#expire_logs_days = 7
binlog_expire_logs_seconds = 604800 

#GTID
gtid_mode = on
enforce_gtid_consistency = 1

#innodb_buffer
#一般设置物理存储的 50% ~ 70%
innodb_buffer_pool_size = 8G
#当缓冲池大小大于1GB时，将innodb_buffer_pool_instances设置为大于1的值，可以提高繁忙服务器的可伸缩性
innodb_buffer_pool_instances = 8
#双一刷盘设置
#控制 innodb_flush_log_at_trx_commit redolog 写磁盘频率  sync_binlog 默认为1 #控制 binlog 写磁盘频率
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
#从库binlog控制
log_replica_updates = ON
#自增ID设置（1，2） 另一台设置为 （2，2）
auto_increment_offset = 1
auto_increment_increment = 2

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

#是否启用慢查询日志，1为启用，0为禁用  
slow_query_log = 1
#指定慢查询日志文件的路径和名字
slow_query_log_file =/data/mysql/data/slow.log
#慢查询执行的秒数，必须达到此值可被记录
long_query_time = 1
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
max_allowed_packet = 512M
```
### 4. 创建数据目录 并修改其属主和组

```bash
mkdir -p /data/mysql/data
chown -R mysql.mysql /data/mysql
chown -R mysql.mysql /data/mysql/data
```
### 5. 初始化数据库实例

```bash
 /usr/local/mysql/bin/mysqld --defaults-file=/etc/my.cnf --initialize
```
### 6. 启动实例

```bash
 /usr/local/mysql/bin/mysqld_safe --defaults-file=/etc/my.cnf &
 ps -ef |grep mysql
```
### 7. 登录实例

##### * 找初始实例日志的临时密码
```bash
grep password /data/mysql/data/mysql.err
```
##### * 登录数据库
```bash
 /usr/local/mysql/bin/mysql -uroot -p
```
##### * 登陆后必须修改密码
```bash
mysql> alter user user() identified by 'xxxxxxxx';
```
##### * 安装密码策略控件

```sql
-- mysql 8.0
select * from mysql.component ;
install component 'file://component_validate_password';
```
### 8. 以服务方式启动

```sql
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld

vim /etc/init.d/mysqld
-- modify basedir  and datadir
basedir=/usr/local/mysql
datadir=/data/mysql/data

# 重新载入
systemctl daemon-reload
# 设置开机自启动
systemctl enable mysqld

# 重启mysql 服务
systemctl status mysqld
systemctl restart mysqld
systemctl stop   mysqld
systemctl start  mysqld
```
### 9. 配置环境变量

```bash
vim /etc/profile     -- 编辑 所有用户的变量  对所有用户登录的环境变量可用

export PATH=$PATH:/usr/local/mysql/bin

source /etc/profile
```
### 10. 登录数据

```bash
[root@centos7-2 bin]# mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.33 MySQL Community Server - GPL
Copyright (c) 2000, 2024, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
## 四.基于GTID 搭建MySQL复制

注意事项： **`server-id` 两台服务器一定要设置成不同ID**

### 1. 在主库建立 数据同步的账号

```sql
CREATE USER 'repl_user'@'10.28.3.19%' IDENTIFIED BY 'XXXXXXXX';
GRANT replication slave ON *.* TO 'repl_user'@'10.28.3.19%';
FLUSH PRIVILEGES;
```
### 2. 在从库执行 设置主库命令

```sql
change master to 
master_host='10.28.3.195',
master_port=3306,
master_user='repl_user',
master_password='XXXXXXXX',
master_auto_position=1,
master_connect_retry=30,
get_master_public_key=1;
```
### 3. 开启 复制 并查看复制状态

```sql
start slave;
show slave status\G; 
```
## 五.安装Keepalived

### 1. yum安装

```bash
yum  install -y  Keepalived
```
### 2. 配置服务

```bash
systemctl enable keepalived
systemctl daemon-reload
```
### 3. 配置文件修改

```bash
mv /etc/keepalived/keepalived.conf  /etc/keepalived/keepalived.conf.old
vim /etc/keepalived/keepalived.conf
```
修改配置文件

```bash
# 全局配置
global_defs {
    # 身份识别(全局唯一)
    router_id lb01
}
vrrp_script check_mysql {
    #这里通过脚本监测
    script "/data/keepalived/check_mysql.sh"
    interval 5                #脚本执行间隔，每5s检测一次
    weight -5                 #脚本结果导致的优先级变更，检测失败（脚本返回非0）则优先级 -5
    fall 1                    #检测连续2次失败才算确定是真失败。会用weight减少优先级（1-255之间）
    rise 1                    #检测1次成功就算成功。但不修改优先级
}
# 配置vrrp协议(相互探测 假设有一Keepalived宕机 它会立马把VIP切换到另一台机器)
vrrp_instance VI_MYSQL {
    # 绑定网卡(所用vip必须是当前机器的网卡所在的网段里的 eth0/eth1里面)
    interface eth0
    # 状态master主节点(这里仅仅是一个标记，真正确认VIP的是权重) 主服务器配置为MASTER，从服务器配置为BACKUP
    state MASTER  
    virtual_router_id 50
    # 优先级(数字越大 权重越大) 主服务器优先级高于从服务器
    priority 100 
    # 检测心跳间隔时间
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 123456  # 验证密码
    }
    virtual_ipaddress {
        10.28.3.194  # 虚拟IP地址，用于连接数据库
    }
    track_script {
        check_mysql
    }
}
```
配置监测MYSQL运行脚本/data/keepalived/check\_mysql.sh

```bash
mkdir -p /data/keepalived/
vim /data/keepalived/check_mysql.sh

#!/bin/bash

# 检查MySQL服务是否在运行  给一次机会看2后是MYSQL又开始运行
counter=$(netstat -na|grep "LISTEN"|grep "3306"|wc -l)
if [ "${counter}" -eq 0 ]; then
        sleep 2;
    counter=$(netstat -na|grep "LISTEN"|grep "3306"|wc -l)
    if [ "${counter}" -eq 0 ]; then
        killall keepalived
    fi
fi
```
* 注意 需要给 /data/keepalived/check\_mysql.sh 脚本为授权。

```bash
 chmod u+x /data/keepalived/check_mysql.sh
```
### 4. 启动keepAlived

```bash
systemctl start keepalived
```
### 5. 验证keepAlived

```bash
# 验证前提 需要 两台服务器 的mysql 和 keepalived 服务 在正常可用的状态下。
# 在master1上执行。
systemctl stop mysqld
# 在其另一台执行ip a  看看虚拟ip 是否切换到另外节点
ip a
```
## 六.建立管理员账号

###  **建立dbadmin账号 **

```bash
CREATE USER 'dbadmin'@'%' IDENTIFIED BY 'xxxxxxxx';
GRANT ALL PRIVILEGES ON *.* TO 'dbadmin'@'%' with GRANT OPTION; 
FLUSH PRIVILEGES;
```

## 七.账号密码信息

### **密码信息**

|    账号    |   密码   | 是否可修改密码 |      权限      |                             备注                             |
| :--------: | :------: | :------------: | :------------: | :----------------------------------------------------------: |
|    root    | xxxxxxxx |       是       |  本地管理账号  |                         只能本机登录                         |
|  dbadmin   | xxxxxxxx |       是       | 远程管理员账号 |                          可远程登录                          |
| repl\_user | xxxxxxxx |       是       |  主从复制账号  | 需要停止复制才可以修改密码，修改后需要重新指定主从账号密码。 |

