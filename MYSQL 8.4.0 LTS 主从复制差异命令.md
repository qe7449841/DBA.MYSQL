# MYSQL 8.4.0 LTS 主从复制差异命令

**MYSQL8.4.0 LTS 版本已经发布快两个月，目前已经有很多小伙伴已经尝鲜了，有小伙伴反馈说MYSQL8.4.0 LTS 在搭建主从复制中有很多的基础命令已经不适用了，搭建的过程很不顺利
通过最近一段时间的使用和查询官方文档我总结了一些常用主从差异的命令对比供大家参考**

## 主从复制 SQL 语句变更对比
之前版本的 MySQL 中已弃用的与 MySQL 复制相关的许多功能的语法现已删除，具体清单如下。

|8.4.0之前使用|8.4.0之后使用|
| :---: | :---: |
|START SLAVE|START REPLICA|
|STOP SLAVE|STOP REPLICA|
|SHOW SLAVE STATUS|SHOW REPLICA STATUS|
|SHOW SLAVE HOSTS|SHOW REPLICAS|
|RESET SLAVE|RESET REPLICA|
|CHANGE MASTER TO|CHANGE REPLICATION SOURCE TO|
|RESET MASTER|RESET BINARY LOGS AND GTIDS|
|SHOW MASTER STATUS|SHOW BINARY LOG STATUS|
|PURGE MASTER LOGS|PURGE BINARY LOGS|
|SHOW MASTER LOGS|SHOW BINARY LOGS|

1. 创建复制用户

```Plain Text
CREATE USER 'replica' IDENTIFIED BY 'replica';
GRANT REPLICATION SLAVE ON *.* TO 'replica';
FLUSH PRIVILEGES;

```
2. 配置复制源

```Plain Text
CHANGE REPLICATION SOURCE TO
SOURCE_HOST = '192.168.1.1',
SOURCE_PORT = 3306,
SOURCE_SSL = 1,
SOURCE_AUTO_POSITION = 1;

```
3. 启动复制

**MYSQL8.4.0**为了安全考虑，不推荐将 <user/password> 信息配置在创建复制源语句，而是在启动复制语句中设定。

```Plain Text
START REPLICA USER = 'replica' PASSWORD = 'replica';
```
4. 查看复制状态

```Plain Text
mysql> show replica status\G
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: 127.0.0.1
                  Source_User: replica
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: binlog.000001
          Read_Source_Log_Pos: 432
               Relay_Log_File: bitdlniu05021-relay-bin.000003
                Relay_Log_Pos: 643
        Relay_Source_Log_File: binlog.000001
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Source_Log_Pos: 432
              Relay_Log_Space: 1043
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Source_SSL_Allowed: Yes
           Source_SSL_CA_File:
           Source_SSL_CA_Path:
              Source_SSL_Cert:
            Source_SSL_Cipher:
               Source_SSL_Key:
        Seconds_Behind_Source: 0
Source_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Source_Server_Id: 10
                  Source_UUID: 4db16b24-1268-11ef-bc66-00155d098dce
             Source_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
    Replica_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Source_Retry_Count: 10
                  Source_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Source_SSL_Crl:
           Source_SSL_Crlpath:
           Retrieved_Gtid_Set: 4db16b24-1268-11ef-bc66-00155d098dce:1
            Executed_Gtid_Set: 4db16b24-1268-11ef-bc66-00155d098dce:1
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Source_TLS_Version:
       Source_public_key_path:
        Get_Source_public_key: 0
            Network_Namespace:
1 row in set (0.00 sec)

```
从 MySQL 8.4.0 LTS 开始，master/slave 已经被弃用了，以后都将使用replica 。

如果继续沿用之前的语法，会直接抛出错误。

```Plain Text
mysql> start slave;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'slave' at line 1

```
## SQL 语句选项
`CHANGE REPLICATION SOURCE TO` 语句的选项，发生了移除、替换变更，清单如下：

| 8.4.0之前使用                     | 8.4.0之后使用                     |
| :---- | :---- |
|MASTER\_AUTO\_POSITION|SOURCE\_AUTO\_POSITION|
|MASTER\_HOST|SOURCE\_HOST|
|MASTER\_BIND|SOURCE\_BIND|
|MASTER\_USER|SOURCE\_USER|
|MASTER\_PASSWORD|SOURCE\_PASSWORD|
|MASTER\_PORT|SOURCE\_PORT|
|MASTER\_CONNECT\_RETRY|SOURCE\_CONNECT\_RETRY|
|MASTER\_RETRY\_COUNT|SOURCE\_RETRY\_COUNT|
|MASTER\_DELAY|SOURCE\_DELAY|
|MASTER\_SSL|SOURCE\_SSL|
|MASTER\_SSL\_CA|SOURCE\_SSL\_CA|
|MASTER\_SSL\_CAPATH|SOURCE\_SSL\_CAPATH|
|MASTER\_SSL\_CIPHER|SOURCE\_SSL\_CIPHER|
|MASTER\_SSL\_CRL|SOURCE\_SSL\_CRL|
|MASTER\_SSL\_CRLPATH|SOURCE\_SSL\_CRLPATH|
|MASTER\_SSL\_KEY|SOURCE\_SSL\_KEY|
|MASTER\_SSL\_VERIFY\_SERVER\_CERT|SOURCE\_SSL\_VERIFY\_SERVER\_CERT|
|MASTER\_TLS\_VERSION|SOURCE\_TLS\_VERSION|
|MASTER\_TLS\_CIPHERSUITES|SOURCE\_TLS\_CIPHERSUITES|
|MASTER\_SSL\_CERT|SOURCE\_SSL\_CERT|
|MASTER\_PUBLIC\_KEY\_PATH|SOURCE\_PUBLIC\_KEY\_PATH|
|GET\_MASTER\_PUBLIC\_KEY|GET\_SOURCE\_PUBLIC\_KEY|
|MASTER\_HEARTBEAT\_PERIOD|SOURCE\_HEARTBEAT\_PERIOD|
|MASTER\_COMPRESSION\_ALGORITHMS|SOURCE\_COMPRESSION\_ALGORITHMS|
|MASTER\_ZSTD\_COMPRESSION\_LEVEL|SOURCE\_ZSTD\_COMPRESSION\_LEVEL|
|MASTER\_LOG\_FILE|SOURCE\_LOG\_FILE|
|MASTER\_LOG\_POS|SOURCE\_LOG\_POS|


## 版本升级
这里有几点升级的注意事项或建议，以供参考。

1. MySQL 5.7 不支持直接升级到 MySQL 8.4，需要先升级到 MySQL 8.0 再过渡到 8.4。
2. MySQL 8.4 移除了对 8.0 早期版本的一些特殊处理，建议在升级到 8.4 之前，先升级到 8.0 最新版本。
3. 当前 MySQL 8.0 系列的最新版本为 MySQL 8.0.37

