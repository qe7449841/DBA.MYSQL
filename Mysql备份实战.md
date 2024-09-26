## MYSQL 备份实战

###  备份需求：

1. **每日凌晨2时对数据做有一次全备，包含binlog,需要对备份文件远程存放。**

2. **备份备份历史文件本地保留7日，远程目录保留3个月。**

### 实现步骤：

###### 1.全库备份脚本

###### 2.binlog 备份脚本

###### 3.压缩和同步脚本

###### 4.配置定时任务

##### 示例脚本如下：

```bash
#!/bin/bash

# 定义变量
BACKUP_DIR="/path/to/backup"               # 本地备份目录
BINLOG_DIR="/path/to/binlog_backup"        # 本地 binlog 备份目录
REMOTE_SERVER="user@remote_server:/remote/backup/directory"  # 远程备份服务器及目录（建议提前配置好ssh互信）
MYSQL_USER="your_mysql_user"               # MySQL 用户名
MYSQL_PASSWORD="your_mysql_password"       # MySQL 用户密码
MYSQL_HOST="localhost"                     # MySQL 主机地址
MYSQL_PORT="3306"                          # MySQL 端口号
DATE=$(date +"%Y%m%d%H%M")                 # 当前日期时间，用于创建唯一的备份目录

# 创建备份目录
mkdir -p $BACKUP_DIR/$DATE
mkdir -p $BINLOG_DIR/$DATE

# 全备份
echo "Starting full backup..."
mysqldump -u $MYSQL_USER -p$MYSQL_PASSWORD -h $MYSQL_HOST -P $MYSQL_PORT --all-databases > $BACKUP_DIR/$DATE/full_backup.sql
if [ $? -eq 0 ]; then
  echo "Full backup successful!"
else
  echo "Full backup failed!"
  exit 1
fi

# binlog 备份
echo "Starting binlog backup..."
mysqladmin -u $MYSQL_USER -p$MYSQL_PASSWORD flush-logs
cp /data/mysql/binlog/mysql-bin.* $BINLOG_DIR/$DATE     #找到binlog文件位置
if [ $? -eq 0 ]; then
  echo "Binlog backup successful!"
else
  echo "Binlog backup failed!"
  exit 1
fi

# 压缩备份文件
echo "Compressing backup files..."
tar -czf $BACKUP_DIR/$DATE/full_backup.tar.gz -C $BACKUP_DIR/$DATE full_backup.sql
tar -czf $BINLOG_DIR/$DATE/binlog_backup.tar.gz -C $BINLOG_DIR/$DATE .
if [ $? -eq 0 ]; then
  echo "Compression successful!"
else
  echo "Compression failed!"
  exit 1
fi

# 同步到远程服务器
echo "Syncing backups to remote server..."
rsync -avz $BACKUP_DIR/$DATE/full_backup.tar.gz $REMOTE_SERVER
rsync -avz $BINLOG_DIR/$DATE/binlog_backup.tar.gz $REMOTE_SERVER
if [ $? -eq 0 ]; then
  echo "Backup sync successful!"
else
  echo "Backup sync failed!"
  exit 1
fi

# 清理本地 7 天前的备份
echo "Cleaning up local backups older than 7 days..."
find $BACKUP_DIR -type d -mtime +7 -exec rm -rf {} \;
find $BINLOG_DIR -type d -mtime +7 -exec rm -rf {} \;

# 清理远程 3 个月前的备份
echo "Cleaning up remote backups older than 3 months..."
ssh user@remote_server "find /remote/backup/directory -type f -mtime +90 -exec rm -f {} \;"

echo "All backup tasks completed successfully!"

```

### 使用说明

1. **变量定义**：
   - `BACKUP_DIR`：定义本地全备份的存储目录。
   - `BINLOG_DIR`：定义本地 binlog 备份的存储目录。
   - `REMOTE_SERVER`：定义远程服务器的用户名、地址和备份目录。
   - `MYSQL_USER`：定义 MySQL 用户名。
   - `MYSQL_PASSWORD`：定义 MySQL 用户密码。
   - `MYSQL_HOST`：定义 MySQL 主机地址。
   - `MYSQL_PORT`：定义 MySQL 端口号。
   - `DATE`：获取当前日期时间，用于创建唯一的备份目录。

2. **备份操作**：
   - **创建备份目录**：使用 `mkdir` 命令创建用于存储全备份和 binlog 备份的目录。
   - **全备份**：使用 `mysqldump` 命令进行全备份，并将结果保存到备份目录。
   - **binlog 备份**：使用 `mysqladmin flush-logs` 刷新 binlog，然后使用 `cp` 命令将 binlog 文件拷贝到备份目录。
   - **压缩备份文件**：使用 `tar` 命令压缩全备份和 binlog 备份文件。
   - **同步到远程服务器**：使用 `rsync` 命令将压缩的备份文件同步到远程服务器。

3. **清理旧备份**：
   - **清理本地备份**：使用 `find` 命令删除本地 7 天前的备份。
   - **清理远程备份**：使用 `ssh` 和 `find` 命令删除远程 3 个月前的备份。

### 配置定时任务

使用 `cron` 配置定时任务，确保脚本在指定时间运行：

```bash
# 编辑 cron 任务
crontab -e

# 添加以下内容，每天两点进行备份
0 2 * * * /path/to/backup_script.sh
```

保存并退出后，`cron` 会在每天凌晨两点自动执行备份脚本。根据实际情况调整脚本中的路径和变量。