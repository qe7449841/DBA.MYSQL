## Mysqldump备份全库,模拟企业版backup记录备份日志

**在MySQL企业版中的 `backup_history` 表用于存储备份相关的历史信息。**

- **通过`mysql.backup_history` 表我们可以查询最近备份的历史记录**
- **增加`backup_history`我们可以使用表查询备份情况**

### 企业版`backup_history` 表结构

- `backup_id`: 备份的唯一标识符。
- `backup_type`: 备份类型（例如，全备、增量备份）。
- `backup_start_time`: 备份开始时间。
- `backup_end_time`: 备份结束时间。
- `backup_size`: 备份大小。
- `backup_status`: 备份状态（例如，成功、失败）。
- `backup_path`: 备份文件的路径。
- `server_id`: 备份所在的服务器ID。
- `comment`: 备份的备注信息。
### `backup_history` 建表语句
```sql
CREATE TABLE `backup_history` (
  `backup_id` INT AUTO_INCREMENT PRIMARY KEY,
  `backup_type` VARCHAR(50) NOT NULL,
  `backup_start_time` DATETIME NOT NULL,
  `backup_end_time` DATETIME NOT NULL,
  `backup_size` BIGINT NOT NULL,
  `backup_status` VARCHAR(50) NOT NULL,
  `backup_path` VARCHAR(255) NOT NULL,
  `server_id` INT NOT NULL,
  `comment` TEXT,
  INDEX (`backup_start_time`),
  INDEX (`server_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
-- ------------------------------------------

### 优化 `backup_history` 表结构

对`backup_history` 表进行重新设计，使其能够包含更多的备份详情和状态信息。以下是优化后的 `backup_history` 表结构和建表语句：

```sql
USE mysql;

CREATE TABLE backup_history (
    id INT AUTO_INCREMENT PRIMARY KEY,
    backup_type ENUM('full', 'incremental', 'differential') NOT NULL,       -- 备份类型
    backup_start_time DATETIME NOT NULL,                                    -- 备份开始时间
    backup_end_time DATETIME NOT NULL,                                      -- 备份结束时间
    backup_size BIGINT NOT NULL,                                            -- 备份文件大小
    backup_status ENUM('success', 'failed') NOT NULL,                       -- 备份状态
    backup_path VARCHAR(255) NOT NULL,                                      -- 备份文件路径
    server_id INT NOT NULL,                                                 -- 服务器 ID
    comment VARCHAR(255),                                                   -- 备注信息
    log_file VARCHAR(255),                                                  -- 日志文件路径
    backup_method ENUM('mysqldump', 'xtrabackup', 'mysqlbackup') NOT NULL,  -- 备份工具
    error_message TEXT,                                                     -- 错误信息
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP                          -- 记录创建时间
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 优化点

1. **备份类型**：添加 `backup_type` 字段，支持 `full`、`incremental` 和 `differential` 备份类型。
2. **日志文件路径**：添加 `log_file` 字段，记录每次备份的日志文件路径。
3. **备份工具**：添加 `backup_method` 字段，记录使用的备份工具。
4. **错误信息**：添加 `error_message` 字段，记录备份过程中出现的错误信息。
5. **记录日志**：在每个关键步骤记录日志，方便后续排查和维护。
6. **压缩文件**：将备份文件进行压缩，并记录压缩后的文件大小。
7. **错误处理**：对备份和压缩过程中的错误进行处理，并记录错误信息。

### 实战备份脚本

```bash
#!/bin/bash

# ===========================
# MySQL 备份脚本
# Create  by:Django
# Createtime:2024-08-08
# ===========================

# 配置变量
BACKUP_DIR="/path/to/backup"                                    # 备份存储目录
MYSQL_USER="your_mysql_user"                                    # MySQL 用户名
MYSQL_PASSWORD="your_mysql_password"                            # MySQL 用户密码
MYSQL_HOST="localhost"                                          # MySQL 主机地址
SERVER_ID=1                                                     # 服务器 ID
COMMENT="Daily backup"                                          # 备注信息
LOG_FILE="${BACKUP_DIR}/backup.log"                             # 备份日志文件
BACKUP_METHOD="mysqldump"                                       # 备份工具
SYNC_TO_REMOTE=false                                            # 是否同步到远程服务器，true 或 false

# 远程服务器配置
REMOTE_BACKUP_DIR="/remote/backup/directory"                    # 远程服务器备份目录
REMOTE_HOST="remote_host"                                       # 远程服务器 IP
REMOTE_USER="remote_user"                                       # 远程服务器用户名

# ===========================
# 确保备份目录存在
# ===========================
mkdir -p "$BACKUP_DIR"

# ===========================
# 记录备份开始日志
# ===========================
BACKUP_START_TIME=$(date +"%Y-%m-%d %H:%M:%S")                  # 备份开始时间
BACKUP_FILE="${BACKUP_DIR}/backup_$(date +"%Y%m%d%H%M%S").sql"  # 备份文件路径
BACKUP_COMPRESSED_FILE="${BACKUP_FILE}.gz"                      # 压缩备份文件路径

echo "[$(date +"%Y-%m-%d %H:%M:%S")] 备份开始: $BACKUP_START_TIME" | tee -a "$LOG_FILE"

# ===========================
# 执行备份并检查是否成功
# ===========================
if mysqldump -u "$MYSQL_USER" -p"$MYSQL_PASSWORD" -h "$MYSQL_HOST" \
  --master-data=2 --single-transaction --triggers --routines --events --all-databases > "$BACKUP_FILE" 2>>"$LOG_FILE"; then

  # 压缩备份文件
  if gzip "$BACKUP_FILE"; then
    BACKUP_END_TIME=$(date +"%Y-%m-%d %H:%M:%S")  # 备份结束时间
    BACKUP_SIZE=$(stat -c%s "$BACKUP_COMPRESSED_FILE")  # 备份文件大小
    BACKUP_STATUS="success"
    echo "[$(date +"%Y-%m-%d %H:%M:%S")] 备份成功: $BACKUP_END_TIME" | tee -a "$LOG_FILE"
  else
    BACKUP_END_TIME=$(date +"%Y-%m-%d %H:%M:%S")
    BACKUP_SIZE=0
    BACKUP_STATUS="failed"
    ERROR_MESSAGE="压缩失败"
    echo "[$(date +"%Y-%m-%d %H:%M:%S")] 备份失败: $BACKUP_END_TIME" | tee -a "$LOG_FILE"
  fi

else
  BACKUP_END_TIME=$(date +"%Y-%m-%d %H:%M:%S")
  BACKUP_SIZE=0
  BACKUP_STATUS="failed"
  ERROR_MESSAGE="备份失败"
  echo "[$(date +"%Y-%m-%d %H:%M:%S")] 备份失败: $BACKUP_END_TIME" | tee -a "$LOG_FILE"
fi

# ===========================
# 记录备份信息到 backup_history 表
# ===========================
mysql -u "$MYSQL_USER" -p"$MYSQL_PASSWORD" -h "$MYSQL_HOST" -D "mysql" <<EOF
INSERT INTO mysql.backup_history (
  backup_type, backup_start_time, backup_end_time, backup_size, backup_status, backup_path, server_id, comment, log_file, backup_method, error_message
) VALUES (
  'full', '$BACKUP_START_TIME', '$BACKUP_END_TIME', $BACKUP_SIZE, '$BACKUP_STATUS', '$BACKUP_COMPRESSED_FILE', $SERVER_ID, '$COMMENT', '$LOG_FILE', '$BACKUP_METHOD', '${ERROR_MESSAGE:-NULL}'
);
EOF

echo "[$(date +"%Y-%m-%d %H:%M:%S")] 备份信息已记录到数据库" | tee -a "$LOG_FILE"

# ===========================
# 删除7天前的旧备份
# ===========================
find "$BACKUP_DIR" -type f -name "backup_*.sql.gz" -mtime +7 -exec rm -f {} \;
echo "[$(date +"%Y-%m-%d %H:%M:%S")] 删除7天前的本地备份" | tee -a "$LOG_FILE"


# ===========================
# 检查是否需要同步到远程服务器
# ===========================
if [ "$SYNC_TO_REMOTE" = true ]; then
  if rsync -avz "$BACKUP_COMPRESSED_FILE" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_BACKUP_DIR" | tee -a "$LOG_FILE"; then
    echo "[$(date +"%Y-%m-%d %H:%M:%S")] 备份文件已同步到远程服务器" | tee -a "$LOG_FILE"
  else
    echo "[$(date +"%Y-%m-%d %H:%M:%S")] 同步到远程服务器失败" | tee -a "$LOG_FILE"
  fi

  # 删除远程服务器上1个月前的旧备份
  ssh "$REMOTE_USER@$REMOTE_HOST" "find $REMOTE_BACKUP_DIR -type f -name 'backup_*.sql.gz' -mtime +31 -exec rm -f {} \;"
  echo "[$(date +"%Y-%m-%d %H:%M:%S")] 删除3个月前的远程备份" | tee -a "$LOG_FILE"
fi

echo "[$(date +"%Y-%m-%d %H:%M:%S")] 备份过程已完成" | tee -a "$LOG_FILE"

```

备份完毕后在mysql库执行查询。

```shell
 select * from mysql.backup_history order by created_at  limit 10;
```

| id   | backup_type | backup_start_time   | backup_end_time     | backup_size | backup_status | backup_path                  | server_id | comment      | log_file   | backup_method | error_message | created_at          |
| ---- | ----------- | ------------------- | ------------------- | ----------- | ------------- | ---------------------------- | --------- | ------------ | ---------- | ------------- | ------------- | ------------------- |
| 1    | full        | 2024-08-08 15:30:08 | 2024-08-08 15:30:11 | 279492      | success       | backup_20240808153008.sql.gz | 1         | Daily backup | backup.log | mysqldump     | NULL          | 2024-08-08 15:30:12 |
| 2    | full        | 2024-08-08 15:31:09 | 2024-08-08 15:31:12 | 279590      | success       | backup_20240808153109.sql.gz | 1         | Daily backup | backup.log | mysqldump     | NULL          | 2024-08-08 15:31:12 |
|      |             |                     |                     |             |               |                              |           |              |            |               |               |                     |



### 备份脚本优点

1. **自动化备份流程**：脚本实现了备份的自动化，无需手动干预，节省了人力资源。

2. **日志记录**：脚本在备份开始和结束时记录日志，便于日后查找备份历史和排查问题。

3. **备份信息写入数据库**：每次备份后都会将备份信息写入 `backup_history` 表，便于备份管理和查询。

4. **压缩备份文件**：备份完成后立即对备份文件进行压缩，减少了存储空间的占用。

5. **备份文件同步**：将备份文件同步到远程服务器，提高了数据安全性，防止本地服务器故障导致数据丢失。

6. **清理过期备份**：脚本会自动删除本地7天前和远程服务器上3个月前的备份文件，避免磁盘空间的浪费。

7. **支持事务和存储程序备份**：使用了 `--single-transaction`、`--routines`、`--triggers` 参数，确保事务和存储程序的完整备份。

8. **详细错误处理**：脚本中包含了详细的错误处理机制，包括记录备份失败的原因和日志。

### 备份脚本缺点

1. **依赖外部工具**：依赖 `mysqldump` 和 `rsync` 工具，如果这些工具未安装或配置不当，备份将会失败。

2. **安全性问题**：脚本中明文保存了MySQL用户名和密码，可能存在安全隐患。可以考虑使用更安全的方式存储和读取这些敏感信息，例如环境变量或加密存储。

3. **单点故障**：如果本地和远程服务器之间的网络连接出现问题，同步过程可能会失败。建议增加网络监控和自动重试机制。

4. **压缩时间**：压缩大型备份文件可能会耗费较长时间，影响备份整体时间。可以考虑使用并行压缩工具（如 `pigz`）以加快压缩速度。

5. **性能开销**：使用 `mysqldump` 进行全库备份时，会对数据库性能产生一定影响，尤其是在大规模数据库中。可以考虑使用物理备份工具（如 `Percona XtraBackup`）以减少性能开销。

6. **远程备份清理**：脚本使用 `ssh` 和 `find` 命令在远程服务器上清理过期备份文件，如果远程服务器配置不当，可能导致清理失败或误删文件。

7. **不适用于所有场景**：该脚本适用于中小型数据库和简单的备份需求，对于大型数据库或复杂的备份需求，可能需要更高级的备份解决方案。

### 总结

该脚本已经实现了基本的自动化备份功能，并具备了一些高级特性（如日志记录、压缩、同步、清理过期备份等），但在实际使用中仍需根据具体需求和环境进行调整和优化。同时，需注意敏感信息的安全性和备份过程中可能出现的网络、性能等问题。