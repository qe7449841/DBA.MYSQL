## Percona XtraDB Cluster (PXC) 详解

### 一、PXC 的原理

Percona XtraDB Cluster (PXC) 是基于 Galera 库的同步多主复制方案，提供了高可用性、可扩展性和数据一致性。PXC 在底层通过 Galera 提供了强一致性（Synchrous Replication），实现了 MySQL 数据库的多主复制架构。

[PXC技术文档](https://docs.percona.com/percona-xtradb-cluster/8.0/yum.html)

#### 1.1 Galera 的工作原理

- **同步复制**：PXC 通过 Galera 的同步复制机制，确保所有节点上的数据是一致的。当一个事务提交时，该事务的写集（write-set）会被同步发送到集群中所有节点。在所有节点确认接收后，事务才会被提交。
  
- **Write Set Replication (WSREP)**：Galera 使用了 WSREP 协议，在每个节点上生成写集并传播给其他节点。所有节点都需要验证写集的有效性，只有通过验证后，事务才能被应用。

- **全局事务 ID (GTID)**：PXC 使用全局事务 ID 来标识集群中所有事务，确保在发生故障或数据回滚时能够保持一致性。

- **Quorum (仲裁机制)**：PXC 使用仲裁机制决定集群是否可用。当集群中有多个节点时，必须有多数节点 (Quorum) 存在才能进行数据写入操作。

- **State Snapshot Transfer (SST) 和 Incremental State Transfer (IST)**：在新节点加入集群或节点重新加入时，PXC 使用 SST 进行全量数据同步，或者使用 IST 进行增量数据同步。

#### 1.2 集群架构

- **多主复制 (Multi-Master Replication)**：PXC 允许集群中的每个节点都接受写入操作，这意味着多个节点可以同时处理写入请求。
  
- **自动故障切换**：如果某个节点宕机，其他节点会自动接管其工作，并保持集群的一致性。

- **负载均衡**：通常使用负载均衡器 (如 HAProxy) 统一分发连接请求到各个节点，从而提高集群的性能和可靠性。

### 二、PXC 的优缺点

#### 2.1 优点

- **高可用性**：PXC 通过多主复制和自动故障切换，实现了高可用性，即使单个节点发生故障，集群仍然可以继续运行。
  
- **数据一致性**：PXC 提供了强一致性模型，确保所有节点的数据一致，避免了传统异步复制中可能出现的数据不一致问题。

- **扩展性**：通过增加节点，可以水平扩展集群，提升读写性能。

- **自动化管理**：PXC 提供了自动化的故障检测、恢复和数据同步，简化了运维管理。

- **简化的备份和恢复**：因为所有节点的数据都是一致的，备份可以从任意节点进行，并且在恢复时也无需担心数据不同步问题。

#### 2.2 缺点

- **性能开销**：由于 PXC 使用同步复制，会引入一定的性能开销，特别是在高延迟网络中，写入操作的性能可能受到影响。

- **数据冲突**：在多主架构中，如果多个节点同时写入相同的数据，可能会发生冲突，虽然 PXC 提供了一些冲突检测和处理机制，但仍然需要谨慎处理。

- **网络依赖性**：PXC 对网络的依赖性较强，网络抖动或延迟可能导致集群性能下降或节点脱离集群。

- **写入扩展性有限**：虽然 PXC 可以通过增加节点提高读性能，但写性能的扩展性相对有限，因为所有写操作都需要同步到所有节点。

### 三、PXC 的适用场景

#### 3.1 适用场景

- **高可用性需求**：适用于对数据库高可用性要求较高的场景，例如电商网站、在线支付系统等，确保在任何节点故障时业务能够继续运行。

- **强一致性要求**：适用于对数据一致性要求严格的应用，如金融系统、订单管理系统等，确保数据在多个节点间保持同步。

- **分布式环境**：适用于分布式架构或跨数据中心部署的场景，可以通过 PXC 实现多个数据中心之间的数据同步和容灾。

- **读多写少的业务**：适用于读操作远多于写操作的应用场景，通过增加节点提高读性能，同时保持写操作的一致性。

#### 3.2 不适用场景

- **高写入压力场景**：在写入操作非常频繁的场景下，由于 PXC 的同步复制特性，可能导致写操作的性能瓶颈。
- **对延迟敏感的应用**：如果应用对事务提交的延迟非常敏感，那么 PXC 的同步机制可能会导致响应时间增加。
- **复杂事务处理**：在需要处理复杂事务、锁机制较多的场景下，PXC 的同步复制可能会引入更多的冲突和锁等待。

-- --------------------------------------------------------------------------------------------------

### 四、PXC 安装配置

#### 1. 环境准备

**服务器信息：**

|    服务器 IP    |  角色  | OS         | CPU  | 内存  | SSD磁盘 |              PXC版本               |
| :-------------: | :----: | ---------- | :--: | :---: | :-----: | :--------------------------------: |
| 192.168.100.111 | Node 1 | redhat 8.0 | 4 核 | 16 GB | 500 GB  | Percona-XtraDB-Cluster_8.0.35-27.1 |
| 192.168.100.112 | Node 2 | redhat 8.0 | 4 核 | 16 GB | 500 GB  | Percona-XtraDB-Cluster_8.0.35-27.1 |
| 192.168.100.113 | Node 3 | redhat 8.0 | 4 核 | 16 GB | 500 GB  | Percona-XtraDB-Cluster_8.0.35-27.1 |

**软件信息：**

- [**MySQL 版本**: Percona XtraDB Cluster (PXC)](https://downloads.percona.com/downloads/Percona-XtraDB-Cluster-80/Percona-XtraDB-Cluster-8.0.35/binary/tarball/Percona-XtraDB-Cluster_8.0.35-27.1_Linux.x86_64.glibc2.17.tar.gz)
- **安装路径**: `/usr/local/`
- **数据目录**: `/data/mysqldata/`
- **配置文件**: `/etc/my.cnf`

#### 2. 基础环境配置

1. **更新系统并安装依赖包**

   在每台服务器上执行以下命令：
   ```bash
   yum update -y
   yum install -y openssl socat procps-ng chkconfig procps-ng coreutils shadow-utils  
   yum install -y perl perl-Data-Dumper libaio libsepol lsof
   ```

2. **禁用 SELinux 和防火墙**

   在每台服务器上执行以下命令：

   ```bash
     setenforce 0
     sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
     systemctl stop firewalld
     systemctl disable firewalld
   ```

3. **同步时间**

   确保每台服务器的时间同步，建议使用 NTP 服务：
   ```bash
     yum install -y chrony
     systemctl start chronyd
     systemctl enable chronyd
   ```

####  3. 安装 Percona XtraDB Cluster

1. 下载并解压二进制包：

   ```bash
   #  安装目录
   cd /usr/local/
   
   # 安装PXC包
   wget https://downloads.percona.com/downloads/Percona-XtraDB-Cluster-80/Percona-XtraDB-Cluster-8.0.35/binary/tarball/Percona-XtraDB-Cluster_8.0.35-27.1_Linux.x86_64.glibc2.17.tar.gz
   
   # 安装xtrabackup包(不做备份的情况下不需要安装)
   wget https://downloads.percona.com/downloads/percona-distribution-mysql-pxc/percona-distribution-mysql-pxc-8.0.35/binary/tarball/percona-xtrabackup-8.0.35-30-Linux-x86_64.glibc2.17.tar.gz
   
   # 解压二进制包
   tar -xvf Percona-XtraDB-Cluster_8.0.35-27.1_Linux.x86_64.glibc2.17.tar.gz -C /usr/local/
   tar -xvf percona-xtrabackup-8.0.35-30-Linux-x86_64.glibc2.17.tar.gz  -C /usr/local/
   
   #  重命名二进制包
   mv Percona-XtraDB-Cluster_8.0.35-27.1_Linux.x86_64.glibc2.17 pxc
   mv percona-xtrabackup-8.0.35-30-Linux-x86_64.glibc2.17 xtrabackup
   ```

2. 创建 MySQL 用户并设置目录权限：

   ```bash
   # 添加用户
   groupadd mysql
   useradd -r -g mysql -s /bin/false mysql
   
   # 建立数据目录
   mkdir -p /data/mysqldata
   
   # 授权目录
   chown -R mysql:mysql /data/mysqldata
   chown -R mysql:mysql /usr/local/pxc
   ```

3. 创建软链接：

   ```bash
   # 变量链接
   ln -s /usr/local/pxc/bin/* /usr/bin/
   ln -s /usr/local/xtrabackup/bin/* /usr/bin/ 
   # 变量链接
   ln -s /usr/local/pxc/support-files/mysql.server /etc/init.d/mysql
   ```

   


#### 3. 配置 PXC 集群

1. **编辑 MySQL 配置文件 `/etc/my.cnf`**

   在每台服务器上创建并编辑 `/etc/my.cnf` 文件

   [PXC官方配置参考文档](https://docs.percona.com/percona-xtradb-cluster/8.0/configure-nodes.html)

   ```bash
   [client]
   socket=/tmp/mysql.sock
   
   [mysqld]
   user = mysql
   basedir = /usr/local/pxc
   datadir = /data/mysqldata
   socket = /data/mysqldata/mysql.sock
   pid-file = /data/mysqldata/mysql.pid
   log-error = /data/mysqldata/mysqld.err
   
   #基础配置
   lower-case-table-names = 1         # 大小写
   sql_require_primary_key = ON       # 表必须有主键
   log_timestamps = system            # 日志时间
   
   # 连接优化
   max_connections = 500              # 最大连接数
   max_connect_errors = 1000          # 最大错误连接次数
   skip_name_resolve = 1              # 只能用IP地址检查客户端的登录
   
   # 复制配置
   server-id = 1                       # 主库ID
   gtid_mode = ON                      # 启用GTID
   enforce-gtid-consistency = ON       # 强制GTID一致性
   log-bin = mysql-bin                 # 启用二进制日志
   binlog_format = ROW                 # 二进制日志格式
   binlog_row_image = FULL             # 行级日志模式
   
   # 性能优化
   innodb_buffer_pool_size = 12G       # 可根据内存大小调整
   innodb_buffer_pool_instances = 8    # 配置buffer_pool为8个
   
   # 集群相关配置
   wsrep_slave_threads=8
   wsrep_log_conflicts
   wsrep_provider=/usr/local/pxc/lib/libgalera_smm.so
   wsrep_cluster_name=my_pxc_cluster
   wsrep_cluster_address=gcomm://192.168.100.111,192.168.100.112,192.168.100.113
   wsrep_node_address=192.168.100.111  # 替换为当前服务器IP
   wsrep_node_name=node1               # 替换为当前节点名
   
   # SST 配置
   wsrep_sst_method=xtrabackup-v2
   # wsrep_sst_auth = "sstuser:s3cret"
   # wsrep_sst_auth 此参数在pxc8.0之后移除 详情查看官方字典
   pxc_strict_mode=ENFORCING
   wsrep_provider_options="socket.ssl_key=server-key.pem;socket.ssl_cert=server-cert.pem;socket.ssl_ca=ca.pem"
   [sst]
   encrypt=4
   ssl-key=server-key.pem
   ssl-ca=ca.pem
   ssl-cert=server-cert.pem
   
   
   # MySQL 高可用相关配置
   default_storage_engine=InnoDB
   innodb_autoinc_lock_mode = 2
   innodb_doublewrite = 1
   innodb_flush_log_at_trx_commit = 0
   
   
   # 缓存优化
   table_open_cache = 4096             # 程打开的表的数量            
   table_definition_cache = 2048       # 可以存储在定义缓存中的表定义数量
   tmp_table_size = 128M               # 内部内存临时表大小
   max_heap_table_size = 256M          # 内部内存临时表的最大值
   thread_cache_size = 128             # 连接使用缓存线程
   open_files_limit = 65535            # mysqld进程能使用的最大文件描述(FD)符数量
   
   [mysqld_safe]
   log-error = /data/mysqldata/mysqld.err
   pid-file = /data/mysqldata/mysql.pid
   ```

2. 节点2 配置：

   ```shell
   # 复制配置
   server-id = 2                      # 主库ID
   ......
   
   # 集群相关配置
   wsrep_node_address=192.168.100.112  # 替换为当前服务器 IP
   wsrep_node_name=node2               # 替换为当前节点名
   ......
   ```

3. 节点3 配置：

   ```shell
   # 复制配置
   server-id = 3                      # 主库ID
   ......
   
   # 集群相关配置
   wsrep_node_address=192.168.100.113  # 替换为当前服务器IP
   wsrep_node_name=node3               # 替换为当前节点名
   ......
   ```

#### 4. 启动PXC集群

1. 初始化数据库：

   在每个节点上使用 mysqld --initialize 初始化数据库：

   ```shell
   mysqld --defaults-file=/etc/my.cnf --user=mysql --basedir=/usr/local/pxc --datadir=/data/mysqldata --initialize
   ```

2. 获取初始 root 密码：

   在每个节点上检查 MySQL 的初始 root 密码：

   ```shell
   grep 'temporary password' /data/mysqldata/mysqld.err
   ```

3. 启动数据：

   **`优先启动引导节点`**

   ```shell
   # 在引导节点上启动数据库
   mysqld_safe --wsrep-new-cluster &
   ```

4. 拷贝ssl认证文件到其它两个节点。
   **群集的每个节点都必须使用相同的 SSL 证书**

   ```bash
   # 在引导节点上执行
   scp /data/mysqldata/*.pem  root@192.168.100.112:/data/mysqldata/
   scp /data/mysqldata/*.pem  root@192.168.100.113:/data/mysqldata/
   
   # 包括了如下文件
   # ca-key.pem
   # ca.pem
   # client-cert.pem
   # client-key.pem
   # private_key.pem
   # public_key.pem
   # server-cert.pem
   # server-key.pem
   ```

5. 修改root 密码：

   在引导节点上修改 root 密码：

   ```sql
   -- 修改root密码
   alter user 'root'@'localhost'  IDENTIFIED  BY  'QWE!@#0-p123';
   ```

6. 其它节点启动

   ```shell
   mysqld_safe &
   # 或者
   systemctl start mysql
   ```

7. **验证集群状态**

   登录 MySQL，并检查集群状态：
   ```bash
   # 查看节点数量
   mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
   
   # 查看节点数
   mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_incoming_addresses';"
   
   # 查看节点状态
   mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_local_state';"
   # wsrep_local_state：值为4表示正常
   # (1)joining==表示节点正在加入集群
   # (2)donor–当前节点是数据奉献者，正在为新加入的节点同步数据
   # (3)joined–当前节点以经成功加入集群
   # (4)synced–当前节点与整个集群是同步状态
   
   # 或者登陆数据库
   SHOW STATUS LIKE '%wsrep%';
   ```
   返回值应为 3，表示三个节点都已加入集群。



###  五. PXC 安装过程中的报错总结及解决办法

在这次 PXC（Percona XtraDB Cluster）安装过程中，遇到了多个与节点加入集群相关的问题，特别是在 SST（State Snapshot Transfer）阶段。以下是主要报错的总结及对应的解决办法：

---

#### 1. **网络连接超时**

   **报错描述**：
   ```
   connection to peer 00000000-0000 with addr timed out, no messages seen in PT3S, socket stats...
   ```
   **问题原因**：
   节点在尝试与集群中的其他节点进行通信时发生了超时，可能是由于网络配置问题或 SSL 配置错误。

   **解决办法**：
   - 检查所有节点之间的网络连通性，确保端口（4567, 4568, 4444）开放。
   - 如果使用 SSL，确认 SSL 证书和密钥配置正确且一致。

---

#### 2. **SST 过程失败**

   **报错描述**：
   ```
   State transfer to 1.0 (node2) failed: -2 (No such file or directory)
   ```
   **问题原因**：
   在状态转移过程中，donor 节点未能成功传输数据给 joiner 节点，可能与 SST 方法配置错误、目录权限问题或 xtrabackup 安装不当有关。

   **解决办法**：
   - 检查 `wsrep_sst_method` 配置为 `xtrabackup-v2`，确保 xtrabackup 安装正确。
   - 确保数据目录及相关配置文件的权限正确，`mysql` 用户拥有对这些目录的完全访问权限。

---

#### 3. **节点退出集群**

   **报错描述**：
   ```
   Will never receive state. Need to abort.
   ```
   **问题原因**：
   由于无法完成 SST，节点决定退出集群以保护集群的数据一致性。

   **解决办法**：
   - 重点检查 donor 节点的错误日志，定位 SST 失败的具体原因。
   - 重新启动节点并确保网络配置和权限配置无误后再尝试加入集群。

---

####  总结

这些报错主要集中在网络配置、SSL 配置、SST 方法及权限设置方面。通过以下措施可以有效解决问题：
1. **检查网络配置**：确保节点之间的连通性及端口开放。
2. **确认 SSL 配置**：确保 SSL 证书和密钥正确配置。
3. **检查 SST 配置和权限**：确保 xtrabackup 正确安装，数据目录权限设置正确。
4. **防火墙配置**：确保防火墙允许 PXC 所需的所有端口通信。

通过以上的检查和调整，节点应能够成功加入 PXC 集群并完成同步过程。



