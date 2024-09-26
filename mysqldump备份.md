# **mysqldump备份**

每天23:50自动进行全备份，并实时同步binlog到备用存储。

### 方案概述

1. **每天23:50执行全备份**：使用 `mysqldump` 进行全备份。
2. **实时同步binlog到备用存储**：使用 `rsync` 或 `scp` 等工具将binlog同步到备用存储。
3. **定时任务调度**：使用 `cron` 设置定时任务。

### 详细步骤

#### 步骤一：准备工作

1. 确保你的MySQL服务器已安装并运行。
2. 配置MySQL备份用户，确保有足够的权限进行备份操作。

#### 步骤二：编写备份脚本

创建一个脚本 `backup.sh` 来执行全备份和同步binlog。

```bash
#!/bin/bash

# 配置部分
DB_USER="backup_user"
DB_PASSWORD="password"
DB_NAME="your_database_name"
BACKUP_DIR="/path/to/backup"
BINLOG_DIR="/var/lib/mysql"
REMOTE_STORAGE="user@remote_host:/path/to/remote_backup"
LOG_FILE="/path/to/backup/backup.log"

# 创建备份目录
mkdir -p $BACKUP_DIR

# 全备份
echo "Starting full backup at $(date)" >> $LOG_FILE
mysqldump -u $DB_USER -p$DB_PASSWORD --all-databases > $BACKUP_DIR/full_backup_$(date +\%F).sql
if [ $? -eq 0 ]; then
    echo "Full backup completed successfully at $(date)" >> $LOG_FILE
else
    echo "Full backup failed at $(date)" >> $LOG_FILE
    exit 1
fi

# 同步binlog
echo "Starting binlog sync at $(date)" >> $LOG_FILE
rsync -avz --progress $BINLOG_DIR/*.binlog $REMOTE_STORAGE
if [ $? -eq 0 ]; then
    echo "Binlog sync completed successfully at $(date)" >> $LOG_FILE
else
    echo "Binlog sync failed at $(date)" >> $LOG_FILE
    exit 1
fi

echo "Backup and sync process completed at $(date)" >> $LOG_FILE
```

#### 步骤三：设置定时任务

使用 `cron` 设置定时任务，使脚本每天23:50自动运行。

1. 编辑 `cron` 任务：

   ```bash
   crontab -e
   ```

2. 添加以下任务：

   ```bash
   50 23 * * * /path/to/backup.sh
   ```

#### 步骤四：检查和测试

1. **检查脚本权限**：

   确保脚本有执行权限：

   ```bash
   chmod +x /path/to/backup.sh
   ```

2. **手动测试脚本**：

   运行脚本，确保没有错误：

   ```bash
   /path/to/backup.sh
   ```

3. **检查日志**：

   查看日志文件，确认备份和同步是否成功。

#### 日常维护

1. **定期检查备份和日志**：确保备份和同步正常进行。
2. **清理过期备份**：定期删除旧的备份文件，以节省存储空间。可以在脚本中添加清理逻辑。

### 完整的部署脚本

以下是最终的 `backup.sh` 脚本：

```bash
#!/bin/bash

# 配置部分
DB_USER="backup_user"
DB_PASSWORD="password"
DB_NAME="your_database_name"
BACKUP_DIR="/path/to/backup"
BINLOG_DIR="/var/lib/mysql"
REMOTE_STORAGE="user@remote_host:/path/to/remote_backup"
LOG_FILE="/path/to/backup/backup.log"
RETENTION_DAYS=7

# 创建备份目录
mkdir -p $BACKUP_DIR

# 全备份
echo "Starting full backup at $(date)" >> $LOG_FILE
mysqldump -u $DB_USER -p$DB_PASSWORD --all-databases > $BACKUP_DIR/full_backup_$(date +\%F).sql
if [ $? -eq 0 ]; then
    echo "Full backup completed successfully at $(date)" >> $LOG_FILE
else
    echo "Full backup failed at $(date)" >> $LOG_FILE
    exit 1
fi

# 同步binlog
echo "Starting binlog sync at $(date)" >> $LOG_FILE
rsync -avz --progress $BINLOG_DIR/*.binlog $REMOTE_STORAGE
if [ $? -eq 0 ]; then
    echo "Binlog sync completed successfully at $(date)" >> $LOG_FILE
else
    echo "Binlog sync failed at $(date)" >> $LOG_FILE
    exit 1
fi

# 清理旧备份
echo "Cleaning up old backups" >> $LOG_FILE
find $BACKUP_DIR -type f -name "*.sql" -mtime +$RETENTION_DAYS -exec rm {} \;
if [ $? -eq 0 ]; then
    echo "Old backups cleaned up successfully at $(date)" >> $LOG_FILE
else
    echo "Failed to clean up old backups at $(date)" >> $LOG_FILE
    exit 1
fi

echo "Backup and sync process completed at $(date)" >> $LOG_FILE
```

### 配置cron任务

```bash
crontab -e
```

添加以下任务：

```bash
50 23 * * * /path/to/backup.sh
```

### 总结

1. **每天23:50进行全备份**。
2. **实时同步binlog到备用存储**。
3. **定期清理旧备份**。
4. **定期检查和维护**。

这样可以确保你的数据库备份和binlog同步能够自动化进行，并且在需要的时候能够恢复数据。