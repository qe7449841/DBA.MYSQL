**MySQL Shell 管理 InnoDB Cluster 的一些日常命令。这些命令将帮助你创建、管理和维护 InnoDB Cluster。**

### 连接到 MySQL Shell
```sh
# 以交互模式连接到 MySQL Shell
mysqlsh

# 以特定用户和主机连接到 MySQL Shell
mysqlsh root@localhost:3306

# 使用 JavaScript 模式
\js

# 使用 SQL 模式
\sql

# 使用 Python 模式
\py
```

### 创建和管理 InnoDB Cluster

#### 创建集群
```javascript
// 使用 JavaScript 模式
// 连接到 MySQL 实例
dba.checkInstanceConfiguration('root@localhost:3306')

// 创建 InnoDB Cluster
var cluster = dba.createCluster('myCluster')

// 查看集群状态
cluster.status()
```

#### 添加实例到集群
```javascript
// 检查实例配置
dba.checkInstanceConfiguration('root@instance2:3306')

// 添加实例到集群
cluster.addInstance('root@instance2:3306')

// 查看集群状态
cluster.status()
```

#### 移除实例
```javascript
// 从集群中移除实例
cluster.removeInstance('root@instance2:3306')

// 查看集群状态
cluster.status()
```

#### 启动和停止集群
```javascript
// 启动集群
cluster.start()

// 停止集群
cluster.stop()
```

### 查看集群信息
```javascript
// 查看集群状态
cluster.status()

// 查看集群配置
cluster.describe()

// 显示实例状态
cluster.describe('root@instance2:3306')
```

### 管理集群
```javascript
// 重新配置集群（例如更改选项）
cluster.rejoinInstance('root@instance2:3306')

// 设置集群选项
cluster.setOption('option_name', 'value')

// 获取集群选项
cluster.getOption('option_name')
```

### 备份和恢复集群
```sh
# 使用 MySQL Shell 进行全备份
mysqlsh -- util dumpInstance -u root -p --outputDir=/path/to/backup

# 使用 MySQL Shell 恢复全备份
mysqlsh -- util loadDump /path/to/backup -u root -p
```

### 日常维护操作

#### 检查实例配置
```javascript
// 检查实例配置以确保符合集群要求
dba.checkInstanceConfiguration('root@localhost:3306')
```

#### 监控集群
```javascript
// 监控集群状态，检查是否有实例离线或出现其他问题
cluster.status()

// 查看集群拓扑结构
cluster.describe()
```

#### 故障恢复
```javascript
// 重新加入失败的实例
cluster.rejoinInstance('root@instance2:3306')

// 如果实例损坏，移除实例
cluster.removeInstance('root@instance2:3306')

// 添加新的实例来替换故障实例
cluster.addInstance('root@newInstance:3306')
```

### 配置集群选项
```javascript
// 设置集群选项
cluster.setOption('autoRejoinTries', 3)

// 获取集群选项的当前值
cluster.getOption('autoRejoinTries')
```

### 集群升级
```javascript
// 停止集群升级
dba.stopClusterUpgrade()

// 开始集群升级
dba.startClusterUpgrade()
```

### 其他有用的命令
```javascript
// 检查集群中的实例健康状况
dba.checkInstanceConfiguration('root@instance2:3306')

// 显示 MySQL Shell 版本信息
\version

// 切换到 SQL 模式
\sql
```

这些命令和操作示例为管理 InnoDB Cluster 提供了全面的指南。根据实际需求，你可以进一步定制这些命令以满足具体的维护和管理需求。