# MYSQL CLONE 克隆插件


##### **clone plugin  是MySQL 8.0.17 开始引入的重大特性**
##### **通过克隆插件 可以很方便的搭建从库 或者 其它节点**
**源端：  完整有数据的服务器正常服务器的服务器。**

**目标端： 无数据，或需要数据的服务器。**

# 克隆插件的限制
1. 克隆期间会阻塞DDL, 同样DDL 也会阻塞 克隆命令的执行，从8.0.27开始 克隆插件 就不会阻塞 源服务器的DDL了。
2. 克隆插件不会拷贝源端的配置参数。
3. 克隆插件不会拷贝源端的Binlog
4. 克隆插件只会拷贝InnoDB表的数据，对于其它存储引擎的表，只拷贝表结构。
5. 不允许通过MySQL Router 链接源端，必须直连到MySQL Server 端口
6. MYSQL 源和目标必须版本一致。操作系统的位数必须一致（同为32或64） 磁盘空间必须够存储克隆数据。
7.  无论源或者目标  只能同时存在一个 克隆线程操作。
8. 目标端需要重启，必须配置好 启动管理工具 SYSTEMCTL 
9. 克隆需要依赖 组复制插件 group\_replication.so 

# 一. 安装 克隆插件
### （1）在配置文件中指定
```bash
[mysqld]
plugin-load-add = mysql_clone.so
```
注意 配置完/etc/my.cnf 需要重启服务

### （2）动态安装
```sql
SHOW PLUGINS;
-- 查看是否安装组复制插件. 没有则需要安装.
install plugin group_replication soname 'group_replication.so';

-- 安装克隆插件
INSTALL PLUGIN clone SONAME 'mysql_clone.so';
SELECT PLUGIN_NAME, PLUGIN_STATUS FROM INFORMATION_SCHEMA.PLUGINS  WHERE PLUGIN_NAME = 'clone';
```
注意 源和目标都需要安装

### （3）权限
* 源端权限：

```sql
-- 源端权限：
CREATE USER 'clone_user'@'%' IDENTIFIED BY 'xxxxxxxxxx';
GRANT backup_admin ON *.* TO 'clone_user'@'%';
FLUSH PRIVILEGES;
```
* 目标权限：

```sql
-- 目标权限：
CREATE USER 'recipient_user'@'%' IDENTIFIED BY 'xxxxxxxxxx';
GRANT clone_admin,backup_admin ON *.* TO 'recipient_user'@'%';
FLUSH PRIVILEGES;
```
* 目标端设置白名单： 指定可以克隆那台源服务器。

```sql
set global clone_valid_donor_list = '192.168.1.2:3306';
```
# 二. 开始远程克隆
目标端执行 开始远程克隆：

```sql
--  在目标用 复制账号登录
mysql -urecipient_user -p'xxxxxxxxxxx'
-- 执行克隆命令
-- 覆盖本地DATA目录
CLONE INSTANCE FROM 'clone_user'@'192.168.1.2':3306 IDENTIFIED BY 'xxxxxxxxxxx';
-- 从新指定DATA目录
CLONE INSTANCE FROM 'clone_user'@'192.168.1.2':3306 IDENTIFIED BY 'xxxxxxxxxxx' DATA DIRECTORY = '/data/mysql_new/data'; 
```
注意：DIRECTORY = '/data/mysql\_new/data' 如果不指定 目录 默认会重启MYSQL 然后替换原来的数据文件。

     路径目录：/data/mysql\_new/  必须存在 必须mysql 用户有权限  

                  `/data/mysql\_new/data`  `data'`目录 必须不能存在。 才能正常克隆

```sql
chown -R mysql.mysql  /data/mysql_new
```


# 三.克隆完成配置 数据库
```bash
-- 停止MySQL 
systemctl stop mysqld

-- 修改/etc/my.cnf
datadir=/data/mysql_new/data

-- 修改服务配置文件 /etc/init.d/mysqld
vim /etc/init.d/mysqld
-- modify  datadir
datadir=/data/mysql_new/data

-- 启动数据
systemctl start  mysqld

-- 登录数据查看文件路径
select @@datadir;
```


# 四.查询克隆的进度。
* 直接看文件的大小 
* 查看performance\_schema 中的两张表 clone\_status 和 clone\_progress 在目标上面执行 查看 进度.

```sql

select * from performance_schema.clone_status\G;
-- 查看 POSid  GTID 等
-- 主要看state: Not Started 尚未开始
                In Progress 进行中
                Completed   成功
                Failed      失败  
-- 详细查看进度
select * from performance_schema.clone_progress;

```


##### 克隆完毕 可以直接搭建主从
```sql
change master to 
master_host='192.168.1.2',
master_port=3306,
master_user='repl_user',
master_password='xxxxxxxxxxx',
master_auto_position=1,
master_connect_retry=30,
get_master_public_key=1;
```



MySQL Clone 是 MySQL 8.0 引入的一项功能，旨在简化主从复制集群或InnoDB Cluster的节点部署和同步。它允许将一个MySQL实例的完整数据集快速克隆到另一个实例上，尤其适用于大型数据库的快速同步。

### 优点

1. **快速部署**: MySQL Clone 可以将整个数据库实例（包括数据、系统表和二进制日志）从一个主机复制到另一个主机，大大缩短了创建从库或节点的时间，尤其是对于大数据集。

2. **自动化过程**: 克隆过程是自动化的，无需手动备份、传输和还原数据。MySQL会自动处理所有的步骤，包括复制数据和应用重做日志。

3. **一致性**: MySQL Clone 操作是原子性的，确保了目标实例在克隆过程中数据的一致性。数据克隆完成后，目标实例会立即进入与源实例相同的状态。

4. **简化运维**: 克隆操作大大简化了数据库管理员在设置和维护复制或高可用性环境时的工作量，特别是在创建从库或新的集群节点时。

### 缺点

1. **占用资源**: 克隆过程会占用大量的 I/O、CPU 和网络带宽资源，特别是当数据量较大时。这可能会影响源实例的性能，因此在高峰期执行克隆操作可能并不理想。

2. **无法跳过错误**: MySQL Clone 不具备跳过错误的机制。如果在克隆过程中发生错误（例如网络中断或磁盘问题），克隆操作将失败，可能需要重新开始。

3. **不适用于部分克隆**: MySQL Clone 是全量克隆，无法只克隆特定的数据库或表。因此，如果只需要克隆部分数据，MySQL Clone 并不合适。

4. **版本要求**: MySQL Clone 仅支持 MySQL 8.0 及以上版本，源实例和目标实例必须是相同版本。这限制了其在多版本环境中的使用。

5. **安全性考虑**: 克隆过程中，目标实例的所有数据都会被覆盖。如果操作不当，可能导致数据丢失。因此，克隆前务必确保备份已完成。

### 限制

1. **MySQL版本一致性**: 源和目标实例必须使用相同版本的 MySQL，不能跨版本克隆。

2. **在线克隆限制**: 如果目标实例已经存在数据，那么在克隆开始前这些数据会被清除。这意味着目标实例不能持有任何关键数据。

3. **克隆大小**: 虽然 MySQL Clone 可以处理大型数据集，但对于非常大的数据集，可能需要考虑网络和磁盘I/O的影响。

4. **支持的存储引擎**: MySQL Clone 目前仅支持 InnoDB 存储引擎，其他存储引擎可能需要手动同步。

MySQL Clone 是一种强大的工具，适合用于快速部署和同步 MySQL 节点，但在使用时需要考虑环境的实际需求和资源限制。





