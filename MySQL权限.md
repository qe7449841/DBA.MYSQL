# MySQL权限

**MySQL 权限大致可以分为以下几类：**

1. **全局权限（Global Privileges）**：适用于整个 MySQL 服务器的权限。
2. **数据库级权限（Database Privileges）**：适用于特定数据库的权限。
3. **表级权限（Table Privileges）**：适用于特定表的权限。
4. **列级权限（Column Privileges）**：适用于特定列的权限。
5. **存储过程和函数权限（Stored Routine Privileges）**：适用于存储过程和函数的权限。
6. **代理权限（Proxy Privileges）**：用于代理身份认证。

## 常用权限说明及示例

### 1. 全局权限

这些权限适用于整个服务器，包括创建和管理数据库、用户等。

- **ALL PRIVILEGES**：授予所有权限。
- **CREATE USER**：创建新用户。
- **GRANT OPTION**：允许用户将自己的权限授予其他用户。
- **RELOAD**：重新加载数据库级的权限。
- **SHUTDOWN**：关闭 MySQL 服务器。
- **SUPER**：执行特定的管理员操作，如杀死其他用户的线程。

**示例：**

```sql
GRANT ALL PRIVILEGES ON *.* TO 'admin_user'@'localhost';
GRANT RELOAD, SHUTDOWN ON *.* TO 'backup_user'@'localhost';
```

### 2. 数据库级权限

这些权限适用于特定数据库中的所有对象。

- **CREATE**：创建新表或数据库。
- **DROP**：删除数据库、表或视图。
- **ALTER**：修改表结构。
- **INDEX**：创建或删除索引。
- **SELECT**：读取数据。
- **INSERT**：插入数据。
- **UPDATE**：更新数据。
- **DELETE**：删除数据。

**示例：**

```sql
GRANT SELECT, INSERT ON mydatabase.* TO 'db_user'@'localhost';
GRANT ALL PRIVILEGES ON mydatabase.* TO 'db_admin'@'localhost';
```

### 3. 表级权限

这些权限适用于特定表。

- **SELECT**：读取表数据。
- **INSERT**：向表中插入数据。
- **UPDATE**：更新表中的数据。
- **DELETE**：从表中删除数据。

**示例：**

```sql
GRANT SELECT, UPDATE ON mydatabase.mytable TO 'table_user'@'localhost';
```

### 4. 列级权限

这些权限适用于特定列。可用于限制用户仅对特定列的数据进行操作。

- **SELECT**：读取指定列的数据。
- **INSERT**：向指定列插入数据。
- **UPDATE**：更新指定列的数据。

**示例：**

```sql
GRANT SELECT (column1, column2), UPDATE (column1) ON mydatabase.mytable TO 'column_user'@'localhost';
```

### 5. 存储过程和函数权限

这些权限用于管理和执行存储过程和函数。

- **EXECUTE**：执行存储过程或函数。
- **ALTER ROUTINE**：更改或删除存储过程或函数。
- **CREATE ROUTINE**：创建存储过程或函数。

**示例：**

```sql
GRANT EXECUTE ON PROCEDURE mydatabase.myprocedure TO 'routine_user'@'localhost';
GRANT CREATE ROUTINE ON mydatabase.* TO 'routine_creator'@'localhost';
```

### 6. 代理权限

用于将用户的权限代理给另一个用户。

- **PROXY**：允许用户代理另一个用户。

**示例：**

```sql
GRANT PROXY ON 'source_user' TO 'proxy_user'@'localhost';
```



|权限名称|授权范围|描述|
| :-----: | :-----: | :-----: |
|ALL|全局权限|实例所有权限|
|CREATE ROLE|全局权限|创建数据库角色|
|CREATE TABLESPACE|全局权限|创建表空间|
|CREATE USER|全局权限|创建数据库用户|
|DROP ROLE|全局权限|删除数据库角色|
|PROCESS|全局权限|查看实例中的所有session(show processlist)|
| | |默认只能看登录用户的session|
|PROXY|全局权限| |
|RELOAD|全局权限|执行下面的操作需要reload权限|
| | |flush privileges|
| | |flush logs|
|REPLICATION CLIENT|全局权限|查看复制信息的权限(show slave status)|
|REOLICATION SLAVE|全局权限|复制权限（从主库复制数据）|
|SHUTDOWN|全局权限|关闭实例|
|SUPER|全局权限|超级权限|
| | |kill任何用户的session|
| | |修改参数|
| | |管理复制(start slave,change master等）|
|USAGE|全局权限|无任何权限|
|SHOW DATABASES|全局权限|查看数据库列表|
| | |show databases|
|CREATE|数据库权限|创建数据库、表|
|INDEX|对象权限|创建索引（create index)|
|CREATE VIEW|对象权限|创建视图|
|CREATE ROUTINE|数据库权限|创建存储过程、函数|
|DROP|数据库权限|删除表、视图、存储过程、触发器、定时任务|
|ALTER|对象权限|修改表结构和表的索引|
| | |即使没有index权限，也可以使用alter语句添加或删除索引|
| | |修改表名称时，同时需要drop权限|
|ALTER ROUTINE|对象权限|修改存储过程、函数|
|EVENT|全局权限|创建、删除调度任务|
|TRIGGER|对象权限|创建触发器的权限|
|SELECT|对象权限|查询数据|
|INSERT|对象权限|插入数据|
|UPDATE|对象权限|更新数据|
|DELETE|对象权限|删除数据|
|EXECUTE|对象权限|执行存储过程的权限|
|REFERENCE|数据库权限|外键引用权限|
|LOCK TABLES|数据库权限|执行lock tables权限，需要同时对表有select权限|
|SHOW VIEW|对象权限|查看视图定义（show create view)|



### 创建用户

在分配权限之前，需要先创建用户。

```sql
CREATE USER 'username'@'host' IDENTIFIED BY 'password';
```

### 查看权限

查看某个用户的权限，可以使用 `SHOW GRANTS` 语句。

```sql
SHOW GRANTS FOR 'username'@'host';
```

### 撤销权限

可以使用 `REVOKE` 语句撤销已授予的权限。

```sql
REVOKE SELECT, INSERT ON mydatabase.* FROM 'db_user'@'localhost';
```

### 刷新权限

在某些情况下，可能需要手动刷新权限缓存，以使权限更改立即生效。

```sql
FLUSH PRIVILEGES;
```



## 授权脚本

```sql
-- 所有权限
GRANT all PRIVILEGES ON *.* TO 'test'@'192.168.1.1'  with grant option;

-- Root 账号
-- mysql_native_password 认证方式已经即将过期 ；建议使用 caching_sha2_password
CREATE USER 'root'@'192.168.1.1' IDENTIFIED WITH caching_sha2_password BY '**********';
GRANT USAGE ON *.* TO 'root'@'192.168.1.1';
GRANT all privileges ON *.* TO 'root'@'192.168.1.1' with grant option;
FLUSH PRIVILEGES;

-- 去掉权限  drop 权限
revoke drop on *.* from 'root'@'192.168.1.1';
flush privileges;

-- 删除用户
drop user test@'192.168.1.1';

```

在 MySQL 中，权限管理是确保数据库安全性的重要组成部分。MySQL 提供了一套丰富的权限系统，允许数据库管理员精细地控制用户可以执行的操作。

