### MySQL Shell 安装方法、使用方法、常用命令

#### 1. MySQL Shell 简介
MySQL Shell 是一个交互式命令行工具，提供对 MySQL 的高级管理功能。它支持 SQL、Python 和 JavaScript 三种模式，方便 DBA 和开发人员进行数据库管理和开发。

#### 2. 安装 MySQL Shell

**在 CentOS 上安装:**
```bash
sudo yum localinstall https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
sudo yum install mysql-shell
```

**在 Ubuntu 上安装:**
```bash
sudo apt-get update
sudo apt-get install mysql-shell
```

**在 Windows 上安装:**
1. 从 [MySQL 下载页面](https://dev.mysql.com/downloads/shell/) 下载 MySQL Shell 安装程序。
2. 运行安装程序并按照指示完成安装。

#### 3. 使用 MySQL Shell

**启动 MySQL Shell:**
```bash
mysqlsh
```

启动后，你将看到类似于以下的提示：
```bash
MySQL Shell 8.0.26

Copyright (c) 2016, 2021, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type '\help' or '\?' for help; '\quit' to exit.
```

**连接到 MySQL 实例:**
```bash
mysqlsh root@localhost
```

#### 4. MySQL Shell 模式切换

MySQL Shell 支持三种模式：SQL、JavaScript 和 Python。你可以使用以下命令在不同模式之间切换：

- 切换到 SQL 模式:
    ```bash
    \sql
    ```
- 切换到 JavaScript 模式:
    ```bash
    \js
    ```
- 切换到 Python 模式:
    ```bash
    \py
    ```

#### 5. 常用命令及其输出

**显示当前模式:**
```bash
\status
```
输出示例：
```bash
MySQL Shell version 8.0.26
...
Current default schema: ``
Current mode: SQL
```

**列出所有数据库:**
```sql
SHOW DATABASES;
```
输出示例：
```bash
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

**创建新数据库:**
```sql
CREATE DATABASE testdb;
```
输出示例：
```bash
Query OK, 1 row affected (0.01 sec)
```

**选择数据库:**
```sql
USE testdb;
```
输出示例：
```bash
Database changed
```

**创建新表:**
```sql
CREATE TABLE users (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(100));
```
输出示例：
```bash
Query OK, 0 rows affected (0.05 sec)
```

**插入数据:**
```sql
INSERT INTO users (name) VALUES ('Alice'), ('Bob');
```
输出示例：
```bash
Query OK, 2 rows affected (0.02 sec)
Records: 2  Duplicates: 0  Warnings: 0
```

**查询数据:**
```sql
SELECT * FROM users;
```
输出示例：
```bash
+----+-------+
| id | name  |
+----+-------+
|  1 | Alice |
|  2 | Bob   |
+----+-------+
```

**删除数据库:**
```sql
DROP DATABASE testdb;
```
输出示例：
```bash
Query OK, 0 rows affected (0.03 sec)
```

**退出 MySQL Shell:**
```bash
\quit
```

#### 6. MySQL Shell 的更多功能
- **导入/导出数据**: 使用 `util.importTable()` 和 `util.exportTable()`。
- **管理 InnoDB Cluster**: 可以通过 MySQL Shell 进行 InnoDB Cluster 的配置和管理。

#### 参考文档
- [MySQL Shell 官方文档](https://dev.mysql.com/doc/mysql-shell/8.0/en/)
- [MySQL Shell 用户指南](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-user-guide.html)