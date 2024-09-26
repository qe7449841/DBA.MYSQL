### `information_schema.processlist` 表详解

| 字段名称 | 数据类型          | 允许为空 | 键   | 默认值 | 额外信息 | 含义                             |
| -------- | ----------------- | -------- | ---- | ------ | -------- | -------------------------------- |
| ID       | `bigint unsigned` | NO       |      |        |          | 线程的唯一标识符。               |
| USER     | `varchar(32)`     | NO       |      |        |          | 线程所属的用户名。               |
| HOST     | `varchar(261)`    | NO       |      |        |          | 发起线程的主机和端口。           |
| DB       | `varchar(64)`     | YES      |      |        |          | 线程当前使用的数据库。           |
| COMMAND  | `varchar(16)`     | NO       |      |        |          | 线程正在执行的命令类型。         |
| TIME     | `int`             | NO       |      |        |          | 线程在当前状态下已经运行的秒数。 |
| STATE    | `varchar(64)`     | YES      |      |        |          | 线程当前的状态。                 |
| INFO     | `varchar(65535)`  | YES      |      |        |          | 当前执行的 SQL 语句。            |

### 字段详解

1. **ID**：
   - **类型**：`bigint unsigned`
   - **含义**：这是线程的唯一标识符，每个线程都有一个唯一的 ID。用于标识和区分不同的线程。

2. **USER**：
   - **类型**：`varchar(32)`
   - **含义**：表示执行该线程的用户的名称。这个字段有助于识别哪个用户发起了线程。

3. **HOST**：
   - **类型**：`varchar(261)`
   - **含义**：表示发起线程的主机名和端口。这个信息有助于追踪线程的来源，特别是在多主机环境中。

4. **DB**：
   - **类型**：`varchar(64)`
   - **允许为空**：是
   - **含义**：线程当前使用的数据库的名称。如果线程没有使用任何数据库，该字段可能为空。

5. **COMMAND**：
   - **类型**：`varchar(16)`
   - **含义**：表示线程当前正在执行的命令类型，例如 `Query`, `Sleep`, `Connect`, `Binlog Dump` 等。这个字段有助于理解线程的当前操作。

6. **TIME**：
   - **类型**：`int`
   - **含义**：表示线程在当前状态下已经运行的时间（以秒为单位）。这个字段可以用来识别长时间运行的线程。

7. **STATE**：
   - **类型**：`varchar(64)`
   - **允许为空**：是
   - **含义**：线程当前的状态，描述了线程正在做什么。例如 `executing`, `sending data`, `sleeping` 等。这个字段有助于了解线程的当前活动。

8. **INFO**：
   - **类型**：`varchar(65535)`
   - **允许为空**：是
   - **含义**：显示线程正在执行的 SQL 语句。如果线程没有执行任何 SQL 语句，该字段可能为空。这个字段可以帮助诊断和优化查询。

### 示例

执行 `SHOW FULL PROCESSLIST;` 命令时，返回的数据示例如下：

```sql
SHOW FULL PROCESSLIST;
```

| ID   | USER  | HOST           | DB   | COMMAND | TIME | STATE        | INFO                                                |
| ---- | ----- | -------------- | ---- | ------- | ---- | ------------ | --------------------------------------------------- |
| 1234 | root  | localhost:3306 | test | Query   | 10   | executing    | SELECT * FROM test_table                            |
| 1235 | user1 | remote:1234    | NULL | Sleep   | 5    |              | NULL                                                |
| 1236 | user2 | localhost:3306 | mydb | Query   | 0    | sending data | INSERT INTO my_table (id, name) VALUES (1, 'Alice') |
### 含义

- **ID**：唯一标识符，便于管理员跟踪和管理不同的线程。当需要结束某个进程，直接使用`kill`+`ID` 则可以结束进程

- **USER**：执行线程的用户，可以帮助识别是谁发起了线程。

- **HOST**：显示发起线程的主机和端口，有助于追踪线程来源。

- **DB**：当前线程使用的数据库，便于识别操作的上下文。

- **COMMAND**：当前执行的命令类型，便于了解线程正在进行的操作。

- **TIME**：线程在当前状态下运行的时间，有助于识别长期运行的线程。

- **STATE**：描述线程的当前状态，提供关于线程活动的更多细节。

- **INFO**：显示当前执行的 SQL 语句，有助于诊断和优化查询。

  

`COMMAND` 字段在 `information_schema.processlist` 表中表示线程正在执行的命令类型。MySQL 中的命令类型有多种，每种类型代表了线程当前的操作状态。以下是一些常见的 `COMMAND` 类型及其含义：

### 常见的 COMMAND 类型

1. **Sleep**:
   - **含义**：线程正在等待客户端发送新的请求。

2. **Query**:
   - **含义**：线程正在执行查询。

3. **Connect**:
   - **含义**：线程正在连接到 MySQL 服务器。

4. **Quit**:
   - **含义**：线程正在关闭连接。

5. **Init DB**:
   - **含义**：线程正在选择数据库。

6. **Field List**:
   - **含义**：线程正在获取表字段列表。

7. **Create DB**:
   - **含义**：线程正在创建数据库。

8. **Drop DB**:
   - **含义**：线程正在删除数据库。

9. **Refresh**:
   - **含义**：线程正在刷新表或缓存。

10. **Shutdown**:
    - **含义**：线程正在关闭服务器。

11. **Statistics**:
    - **含义**：线程正在获取服务器统计信息。

12. **Processlist**:
    - **含义**：线程正在获取线程列表。

13. **Kill**:
    - **含义**：线程正在终止其他线程。

14. **Ping**:
    - **含义**：线程正在检查服务器是否还活着。

15. **Time**:
    - **含义**：线程正在获取当前时间。

16. **Delayed Insert**:
    - **含义**：线程正在执行延迟插入操作。

17. **Change User**:
    - **含义**：线程正在改变用户。

18. **Binlog Dump**:
    - **含义**：线程正在将二进制日志发送到从服务器。

19. **Table Dump**:
    - **含义**：线程正在将表发送到从服务器。

20. **Connect Out**:
    - **含义**：线程正在连接到从服务器。

21. **Register Slave**:
    - **含义**：线程正在注册从服务器。

22. **Prepare**:
    - **含义**：线程正在准备执行预处理语句。

23. **Execute**:
    - **含义**：线程正在执行预处理语句。

24. **Fetch**:
    - **含义**：线程正在获取预处理语句的结果。

25. **Daemon**:
    - **含义**：线程是后台守护进程，通常用于内部任务。



### 常见的 STATE 类型及其含义

1. **After create**:
   - **含义**：正在为创建操作清理工作。

2. **Altering table**:
   - **含义**：正在执行 `ALTER TABLE` 操作。

3. **Analyzing**:
   - **含义**：正在分析数据以生成统计信息。

4. **Checking permissions**:
   - **含义**：正在检查用户权限。

5. **Cleaning up**:
   - **含义**：正在清理查询处理过程中使用的资源。

6. **Closing tables**:
   - **含义**：正在关闭表。

7. **Converting HEAP to MyISAM**:
   - **含义**：正在将内存表转换为 MyISAM 表。

8. **Copying to group table**:
   - **含义**：正在复制数据到分组表。

9. **Copying to tmp table**:
   - **含义**：正在复制数据到临时表。

10. **Creating sort index**:
    - **含义**：正在为排序操作创建索引。

11. **Creating table**:
    - **含义**：正在创建表。

12. **Creating tmp table**:
    - **含义**：正在创建临时表。

13. **deleting from main table**:
    - **含义**：正在从主表删除记录。

14. **deleting from reference tables**:
    - **含义**：正在从引用表删除记录。

15. **Flushing tables**:
    - **含义**：正在刷新表到磁盘。

16. **freeing items**:
    - **含义**：正在释放查询处理过程中使用的内存。

17. **init**:
    - **含义**：正在初始化线程。

18. **killed**:
    - **含义**：查询已被终止。

19. **Locked**:
    - **含义**：线程正在等待表锁。

20. **logging slow query**:
    - **含义**：正在将慢查询日志记录到日志文件。

21. **Opening tables**:
    - **含义**：正在打开表。

22. **optimizing**:
    - **含义**：正在优化表。

23. **preparing**:
    - **含义**：正在准备查询执行。

24. **Reading from net**:
    - **含义**：正在从网络读取数据。

25. **Removing duplicates**:
    - **含义**：正在删除重复的记录。

26. **Reopen table**:
    - **含义**：正在重新打开表。

27. **Repair by sorting**:
    - **含义**：通过排序修复表。

28. **Repair with keycache**:
    - **含义**：通过键缓存修复表。

29. **Rolling back**:
    - **含义**：正在回滚事务。

30. **Saving state**:
    - **含义**：正在保存当前状态。

31. **Searching rows for update**:
    - **含义**：正在搜索需要更新的行。

32. **Sending data**:
    - **含义**：正在向客户端发送数据。

33. **Sorting for group**:
    - **含义**：正在为分组操作进行排序。

34. **Sorting for order**:
    - **含义**：正在为排序操作进行排序。

35. **Sorting index**:
    - **含义**：正在对索引进行排序。

36. **Sorting result**:
    - **含义**：正在对结果进行排序。

37. **statistics**:
    - **含义**：正在计算统计信息。

38. **System lock**:
    - **含义**：正在等待获取系统锁。

39. **update**:
    - **含义**：正在更新数据。

40. **Updating**:
    - **含义**：正在更新记录。

41. **User lock**:
    - **含义**：正在等待获取用户锁。

42. **Waiting for commit lock**:
    - **含义**：正在等待提交锁。

43. **Waiting for global read lock**:
    - **含义**：正在等待全局读锁。

44. **Waiting for tables**:
    - **含义**：正在等待表被释放。

45. **Waiting for table metadata lock**:
    - **含义**：正在等待表元数据锁。

46. **Waiting for table flush**:
    - **含义**：正在等待表被刷新。

47. **Waiting for table level lock**:
    - **含义**：正在等待表级锁。

48. **waiting for handler commit**:
    - **含义**：正在等待处理器提交。

49. **Writing to net**:
    - **含义**：正在向网络写入数据。

50. **locked**:
    - **含义**：线程已被锁定。



