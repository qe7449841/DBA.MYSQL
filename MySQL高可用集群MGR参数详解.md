# MySQL高可用集群MGR参数详解

MySQL Group Replication (MGR) 是一种实现高可用性和数据一致性的解决方案。 MGR 中的关键配置参数，每个参数负责不同方面的功能，包括集群成员管理、消息传输、事务处理、恢复机制以及安全配置。通过合理配置这些参数，可以确保集群在各种环境中稳定高效地运行。

## 参数分类统计

|       **分类**       | **参数名称**                                                 |
| :------------------: | :----------------------------------------------------------- |
|   **集群成员管理**   | `group_replication_bootstrap_group`、`group_replication_group_name`、`group_replication_group_seeds`、`group_replication_force_members`、`group_replication_member_weight` |
|   **消息传输配置**   | `group_replication_communication_max_message_size`、`group_replication_message_cache_size` |
|     **事务管理**     | `group_replication_auto_increment_increment`、`group_replication_consistency`、`group_replication_transaction_size_limit`、`group_replication_gtid_assignment_block_size` |
|     **恢复机制**     | `group_replication_recovery_compression_algorithms`、`group_replication_recovery_ssl_ca`、`group_replication_recovery_ssl_cert`、`group_replication_recovery_retry_count` |
|     **网络配置**     | `group_replication_local_address`、`group_replication_ip_allowlist`（或`group_replication_ip_whitelist`） |
|     **安全配置**     | `group_replication_ssl_mode`、`group_replication_tls_source`、`group_replication_recovery_ssl_mode`、`group_replication_recovery_use_ssl`、`group_replication_recovery_ssl_verify_server_cert` |
|    **数据一致性**    | `group_replication_enforce_update_everywhere_checks`、`group_replication_consistency` |
|     **流控管理**     | `group_replication_flow_control_applier_threshold`、`group_replication_flow_control_max_quota`、`group_replication_flow_control_min_quota` |
|     **超时设置**     | `group_replication_components_stop_timeout`、`group_replication_member_expel_timeout`、`group_replication_unreachable_majority_timeout` |
| **启动及自动化配置** | `group_replication_start_on_boot`、`group_replication_single_primary_mode`、`group_replication_autorejoin_tries` |

## 主要参数

| **参数名称**                                        | **描述**                   | **默认值**          | **配置示例**                        |
| --------------------------------------------------- | -------------------------- | ------------------- | ----------------------------------- |
| `group_replication_bootstrap_group`                 | 控制是否引导集群           | `OFF`               | 仅在第一次启动集群时启用，后续关闭  |
| `group_replication_group_name`                      | 集群唯一标识符             | 无默认值            | 手动设置唯一的 UUID                 |
| `group_replication_group_seeds`                     | 初始节点地址列表           | `NULL`              | 配置所有节点地址                    |
| `group_replication_force_members`                   | 强制指定成员列表           | `NULL`              | 集群分裂时使用，谨慎操作            |
| `group_replication_communication_max_message_size`  | 消息传输最大尺寸           | `10485760`（10MB）  | 对高并发场景适当调整                |
| `group_replication_message_cache_size`              | 消息缓存大小               | `1073741824`（1GB） | 对大流量传输场景增加缓存            |
| `group_replication_auto_increment_increment`        | 控制 auto_increment 增量   | `7`                 | 集群成员数越多，值越大              |
| `group_replication_transaction_size_limit`          | 单事务最大限制             | `150MB`             | 大事务场景适当增加值                |
| `group_replication_local_address`                   | 本地节点通信地址           | 无默认值            | 配置节点独立 IP 和端口              |
| `group_replication_ip_allowlist`                    | IP 允许列表                | `AUTOMATIC`         | 配置所有可信任节点                  |
| `group_replication_ssl_mode`                        | 集群SSL 模式               | `DISABLED`          | 高安全需求时启用 `VERIFY_IDENTITY`  |
| `group_replication_tls_source`                      | TLS 密钥路径               | `NULL`              | 配置 TLS 证书路径，确保通信安全     |
| `group_replication_recovery_compression_algorithms` | 恢复过程中压缩算法         | `NULL`              | 启用 `lz4` 或 `zlib` 以提高效率     |
| `group_replication_recovery_ssl_ca`                 | 恢复期间使用的 CA 证书路径 | `NULL`              | 配置 CA 证书路径                    |
| `group_replication_start_on_boot`                   | 是否在启动时自动开启集群   | `OFF`               | 启用，确保 MySQL 重启后自动恢复集群 |
| `group_replication_single_primary_mode`             | 单主模式配置               | `ON`                | 多写场景关闭，单写多读场景启用      |
| `group_replication_unreachable_majority_timeout`    | 多数节点不可达时的超时时间 | `30000`（30秒）     | 根据网络状况调整                    |

## 参数的详解

### 1. `group_replication_advertise_recovery_endpoints`

- **描述**：指定节点在恢复期间应使用的备用地址。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：适用于有多个网络接口的环境，明确配置备用恢复地址，确保恢复时不使用错误的接口。

```ini
group_replication_advertise_recovery_endpoints = "192.168.100.111:33062"
```

### 2. `group_replication_allow_local_lower_version_join`
- **描述**：允许本地较低版本的实例加入集群，用于支持版本过渡。
- **默认值**：`OFF`
- **可修改**：是
- **配置示例**：仅在升级期间使用，以便逐步升级集群。

```ini
group_replication_allow_local_lower_version_join = OFF
```

### 3. `group_replication_auto_increment_increment`
- **描述**：控制自动递增列的步长，以确保多节点间的自增列值不会冲突。
- **默认值**：由系统根据节点数量自动调整
- **可修改**：是
- **配置示例**：确保步长等于集群中的节点数。

```ini
auto_increment_increment = 3
```

### 4. `group_replication_autorejoin_tries`
- **描述**：指定节点断开后自动尝试重新加入集群的次数。
- **默认值**：`3`
- **可修改**：是
- **配置示例**：根据网络环境适当调整，较差网络环境下可增加重试次数。

```ini
group_replication_autorejoin_tries = 5
```

### 5. `group_replication_bootstrap_group`
- **描述**：引导集群时的初始化选项，通常用于启动第一个节点。
- **默认值**：`OFF`
- **可修改**：是
- **配置示例**：仅在集群首次启动时使用，确保引导后立即关闭。

```ini
group_replication_bootstrap_group = ON
```

### 6. `group_replication_clone_threshold`
- **描述**：当数据差异大于指定值时触发数据克隆，而不是通过传统方式恢复数据。
- **默认值**：`1GB`
- **可修改**：是
- **配置示例**：根据业务规模调整，较大数据差异时可以增加此值以加快恢复速度。

```ini
group_replication_clone_threshold = 2GB
```

### 7. `group_replication_communication_debug_options`
- **描述**：启用通信调试选项，用于跟踪和诊断 Group Replication 通信问题。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：仅在调试时使用，避免性能下降。

```ini
group_replication_communication_debug_options = "TRACE"
```

### 8. `group_replication_communication_max_message_size`
- **描述**：设置Group Replication通信消息的最大大小。
- **默认值**：`10MB`
- **可修改**：是
- **配置示例**：根据业务需要调整，处理大数据事务时可适当增加。

```ini
group_replication_communication_max_message_size = 20MB
```

### 9. `group_replication_components_stop_timeout`
- **描述**：指定停止 Group Replication 组件的超时时间（毫秒）。
- **默认值**：`30000`（30秒）
- **可修改**：是
- **配置示例**：在大型集群或高负载情况下，适当增加超时时间。

```ini
group_replication_components_stop_timeout = 60000
```

### 10. `group_replication_compression_threshold`

- **描述**：指定消息大小超过该值时进行压缩，减少网络传输负载。
- **默认值**：`1000000`（1MB）
- **可修改**：是
- **配置示例**：根据网络带宽和节点数量适当调整。

```ini
group_replication_compression_threshold = 2000000
```

### 11. `group_replication_consistency`
- **描述**：控制复制事务的一致性级别，可选值包括 `EVENTUAL`、`BEFORE`、`AFTER` 和 `BEFORE_AND_AFTER`。
- **默认值**：`EVENTUAL`
- **可修改**：是
- **配置示例**：对于高一致性需求，建议使用 `BEFORE_AND_AFTER`；性能敏感的系统可使用 `EVENTUAL`。

```ini
group_replication_consistency = BEFORE_AND_AFTER
```

### 12. `group_replication_enforce_update_everywhere_checks`
- **描述**：在多主模式下，启用更新时的额外一致性检查。
- **默认值**：`OFF`
- **可修改**：是
- **配置示例**：多主模式时启用，确保数据一致性。

```ini
group_replication_enforce_update_everywhere_checks = ON
```

### 13. `group_replication_exit_state_action`
- **描述**：当节点退出集群时，定义节点的行为，如 `OFFLINE_MODE` 或 `READ_ONLY`。
- **默认值**：`OFFLINE_MODE`
- **可修改**：是
- **配置示例**：如果希望节点退出集群后仍可读，建议使用 `READ_ONLY`。

```ini
group_replication_exit_state_action = READ_ONLY
```

### 14. `group_replication_flow_control_applier_threshold`
- **描述**：控制滞后节点的流量控制阈值，当达到指定滞后事务数量时触发流控。
- **默认值**：`25000`
- **可修改**：是
- **配置示例**：根据事务量调整，减少滞后节点的负载。

```ini
group_replication_flow_control_applier_threshold = 50000
```

### 15. `group_replication_flow_control_certifier_threshold`
- **描述**：触发流量控制的认证器阈值，用于控制写入的同步频率。
- **默认值**：`25000`
- **可修改**：是
- **配置示例**：对于高写入负载的集群，适当增加此值。

```ini
group_replication_flow_control_certifier_threshold = 50000
```

### 16. `group_replication_flow_control_hold_percent`
- **描述**：控制流控时可以处理的事务百分比。
- **默认值**：`100`
- **可修改**：是
- **配置示例**：一般保持默认值，但在负载不均衡时可适当调整。

```ini
group_replication_flow_control_hold_percent = 90
```

### 17. `group_replication_flow_control_max_quota`
- **描述**：限制流控期间允许发送的最大事务数量。
- **默认值**：`0`（无上限）
- **可修改**：是
- **配置示例**：根据系统性能适当设置，避免滞后节点发送过多事务。

```ini
group_replication_flow_control_max_quota = 1000
```

### 18. `group_replication_flow_control_member_quota_percent`
- **描述**：流控期间成员可以发送的最大配额百分比。
- **默认值**：`100`
- **可修改**：是
- **配置示例**：一般保持默认值。

### 19. `group_replication_flow_control_min_quota`
- **描述**：设置流控期间允许发送的最小事务数量。
- **默认值**：`10`
- **可修改**：是
- **配置示例**：保持默认值或根据系统需求调整。

### 20. `group_replication_flow_control_min_recovery_quota`
- **描述**：恢复期间的最小流控配额，确保恢复进程不会受到过多流控限制。
- **默认值**：`50`
- **可修改**：是
- **配置示例**：根据节点恢复速度和集群负载情况调整。

### 21. `group_replication_flow_control_mode`
- **描述**：指定流控模式，如 `QUOTA` 或 `TIME_BASED`。
- **默认值**：`QUOTA`
- **可修改**：是
- **配置示例**：根据流控需求选择适合的模式。

### 22. `group_replication_flow_control_period`
- **描述**：流控期间的检查周期（毫秒）。
- **默认值**：`1000`
- **可修改**：是
- **配置示例**：根据网络状况和负载调整。

```ini
group_replication_flow_control_period = 2000
```

### 23. `group_replication_flow_control_release_percent`
- **描述**：指定滞后节点达到该百分比时，允许节点继续处理事务。
- **默认值**：`50`
- **可修改**：是
- **配置示例**：保持默认值或根据集

群需求调整。

```ini
group_replication_flow_control_release_percent = 70
```



### 24. `group_replication_force_members`
- **描述**：强制指定当前集群的成员列表，通常用于恢复集群时的成员重新配置。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：在集群发生分裂时使用该参数强制设置成员列表，以确保集群的正常运行，必须谨慎使用，避免造成数据不一致。

```ini
group_replication_force_members = "192.168.1.41:33061,192.168.1.42:33061,192.168.1.43:33061"
```

### 25. `group_replication_group_name`
- **描述**：集群的唯一标识符，每个 Group Replication 集群必须有唯一的名称（UUID格式）。
- **默认值**：无默认值，需手动设置。
- **可修改**：是
- **配置示例**：为每个集群设置一个独特的 UUID，确保在不同集群中不重复。

```ini
group_replication_group_name = "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
```

### 26. `group_replication_group_seeds`
- **描述**：集群中初始节点的地址列表，节点启动时会根据这个列表找到集群中的其他节点。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：配置所有节点的地址，确保节点能够在启动时找到其他成员。

```ini
group_replication_group_seeds = "192.168.1.41:33061,192.168.1.42:33061,192.168.1.43:33061"
```

### 27. `group_replication_gtid_assignment_block_size`
- **描述**：控制每个事务组的 GTID 分配块大小，以提高复制性能。
- **默认值**：`1000000`
- **可修改**：是
- **配置示例**：如果有大量并发事务，可以增加此值来优化性能。

```ini
group_replication_gtid_assignment_block_size = 5000000
```

### 28. `group_replication_ip_allowlist` （已弃用）
- **描述**：控制允许加入集群的 IP 地址范围。
- **默认值**：`AUTOMATIC`
- **可修改**：是
- **配置示例**：配置成包含集群内所有节点的IP地址，防止未经授权的节点加入。

```ini
group_replication_ip_allowlist = "192.168.1.0/24"
```

### 29. `group_replication_ip_whitelist` （替代 `group_replication_ip_allowlist`）
- **描述**：控制允许加入集群的IP白名单，用于限制外部连接。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：与 `group_replication_ip_allowlist` 类似，确保白名单中的IP仅包含可信节点。

```ini
group_replication_ip_whitelist = "192.168.1.41,192.168.1.42,192.168.1.43"
```

### 30. `group_replication_local_address`
- **描述**：指定本地节点在集群中的通信地址。
- **默认值**：无默认值，需手动设置。
- **可修改**：是
- **配置示例**：为每个节点设置独立的地址，通常是 IP 和端口的组合。

```ini
group_replication_local_address = "192.168.1.41:33061"
```

### 31. `group_replication_member_expel_timeout`
- **描述**：节点被集群驱逐前允许的最大超时时间（毫秒）。
- **默认值**：`0`（无限制）
- **可修改**：是
- **配置示例**：根据网络环境调整，在不稳定的网络中适当增加此值，以避免节点被过早驱逐。

```ini
group_replication_member_expel_timeout = 60000
```

### 32. `group_replication_member_weight`
- **描述**：设置成员的权重，用于仲裁和选择主节点，权重越高的节点越可能成为主节点。
- **默认值**：`50`
- **可修改**：是
- **配置示例**：根据业务需求为关键节点设置较高的权重，确保其优先作为主节点。

```ini
group_replication_member_weight = 100
```

### 33. `group_replication_message_cache_size`
- **描述**：指定存储消息缓存的大小，用于优化消息传递和复制性能。
- **默认值**：`1073741824`（1GB）
- **可修改**：是
- **配置示例**：对于大数据量传输，可以增加缓存大小以提高性能。

```ini
group_replication_message_cache_size = 2147483648  # 2GB
```

### 34. `group_replication_poll_spin_loops`
- **描述**：控制节点在进入空闲状态前自旋等待的循环次数，用于优化资源利用。
- **默认值**：`0`
- **可修改**：是
- **配置示例**：通常保持默认值，除非对系统资源利用有特殊需求。

### 35. `group_replication_recovery_complete_at`
- **描述**：控制节点在恢复时的何种状态下完成恢复，`TRANSACTIONS_APPLIED` 或 `TRANSACTIONS_COMMITTED`。
- **默认值**：`TRANSACTIONS_APPLIED`
- **可修改**：是
- **配置示例**：在性能与数据一致性之间取得平衡时使用 `TRANSACTIONS_APPLIED`。

```ini
group_replication_recovery_complete_at = TRANSACTIONS_APPLIED
```

### 36. `group_replication_recovery_compression_algorithms`
- **描述**：用于数据恢复的压缩算法，可以选择 `zlib`、`lz4` 等。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：为提升恢复效率，可启用压缩算法，适用于带宽较低的环境。

```ini
group_replication_recovery_compression_algorithms = lz4
```

### 37. `group_replication_recovery_get_public_key`
- **描述**：在恢复过程中获取节点的公钥，增强安全性。
- **默认值**：`OFF`
- **可修改**：是
- **配置示例**：根据安全需求启用，特别是在高安全环境中。

```ini
group_replication_recovery_get_public_key = ON
```

### 38. `group_replication_recovery_public_key_path`
- **描述**：指定节点公钥的路径。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：根据恢复和加密需求配置公钥路径。

### 39. `group_replication_recovery_reconnect_interval`
- **描述**：节点恢复时尝试重新连接的时间间隔（毫秒）。
- **默认值**：`60`（秒）
- **可修改**：是
- **配置示例**：根据网络状况适当调整重连间隔，较差网络情况下适当增加。

```ini
group_replication_recovery_reconnect_interval = 120000  # 2分钟
```

### 40. `group_replication_recovery_retry_count`
- **描述**：指定恢复期间尝试重新连接的次数。
- **默认值**：`10`
- **可修改**：是
- **配置示例**：网络不稳定时可增加重试次数，确保节点能够成功恢复。

```ini
group_replication_recovery_retry_count = 20
```

### 41. `group_replication_recovery_ssl_ca`
- **描述**：恢复期间使用的 SSL 证书颁发机构（CA）的路径。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：根据安全需求配置 CA 证书路径，确保安全的恢复过程。

### 42. `group_replication_recovery_ssl_capath`
- **描述**：指定 CA 证书的目录路径，用于恢复期间的 SSL 连接验证。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：适用于需要 SSL 证书目录的场景，确保恢复过程的安全性。

### 43. `group_replication_recovery_ssl_cert`
- **描述**：指定恢复期间使用的 SSL 证书路径。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：用于配置恢复过程中使用的SSL证书，提升数据传输安全性。

```ini
group_replication_recovery_ssl_cert = "/etc/mysql/certs/server-cert.pem"
```

### 44. `group_replication_recovery_ssl_cipher`
- **描述**：设置恢复过程中允许的 SSL 加密套件。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：根据安全策略选择合适的加密套件，确保传输安全。

```ini
group_replication_recovery_ssl_cipher = "

ECDHE-RSA-AES256-GCM-SHA384"
```

### 45. `group_replication_recovery_ssl_crl`
- **描述**：指定恢复期间的 SSL 证书吊销列表（CRL）文件路径。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：为加强安全性，在 SSL 配置中加入证书吊销检查。

```ini
group_replication_recovery_ssl_crl = "/etc/mysql/certs/crl.pem"
```

### 46. `group_replication_recovery_ssl_crlpath`
- **描述**：指定 CRL 证书的目录路径。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：适合使用多个 CRL 文件的场景。

### 47. `group_replication_recovery_ssl_key`
- **描述**：恢复过程中使用的 SSL 私钥路径。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：确保恢复时的私钥安全存储，提升安全性。

```ini
group_replication_recovery_ssl_key = "/etc/mysql/certs/server-key.pem"
```

### 48. `group_replication_recovery_ssl_verify_server_cert`
- **描述**：决定是否验证服务器端的SSL证书。
- **默认值**：`OFF`
- **可修改**：是
- **配置示例**：在高安全需求环境中启用，以防止中间人攻击。

```ini
group_replication_recovery_ssl_verify_server_cert = ON
```

### 49. `group_replication_recovery_tls_ciphersuites`
- **描述**：定义恢复过程中允许的 TLS 加密套件。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：根据安全需求选择合适的TLS加密套件。

### 50. `group_replication_recovery_tls_version`
- **描述**：指定恢复过程中使用的 TLS 版本。
- **默认值**：`TLSv1.2`
- **可修改**：是
- **配置示例**：配置为最新稳定的 TLS 版本以提升安全性。

```ini
group_replication_recovery_tls_version = TLSv1.3
```

### 51. `group_replication_recovery_use_ssl`
- **描述**：决定是否在恢复期间启用 SSL。
- **默认值**：`OFF`
- **可修改**：是
- **配置示例**：启用 SSL 以确保恢复过程中的数据传输安全。

```ini
group_replication_recovery_use_ssl = ON
```

### 52. `group_replication_recovery_zstd_compression_level`
- **描述**：定义恢复过程中使用 zstd 压缩的级别，影响压缩效率和速度。
- **默认值**：`3`
- **可修改**：是
- **配置示例**：根据带宽情况调整压缩级别，压缩率和性能需要平衡。

```ini
group_replication_recovery_zstd_compression_level = 5
```

### 53. `group_replication_single_primary_mode`
- **描述**：启用或禁用单主模式。单主模式下，只有一个节点作为主节点，其余节点为只读。
- **默认值**：`ON`
- **可修改**：是
- **配置示例**：根据应用场景选择，通常在多数写操作时启用单主模式，提高一致性和性能。

```ini
group_replication_single_primary_mode = ON
```

### 54. `group_replication_ssl_mode`
- **描述**：控制集群中 SSL 的使用模式。可以为 `DISABLED`、`REQUIRED`、`VERIFY_CA` 或 `VERIFY_IDENTITY`。
- **默认值**：`DISABLED`
- **可修改**：是
- **配置示例**：在安全需求较高的环境中，推荐启用并设置为 `VERIFY_IDENTITY` 以保证身份验证和数据加密。

```ini
group_replication_ssl_mode = "VERIFY_IDENTITY"
```

### 55. `group_replication_start_on_boot`
- **描述**：控制节点在 MySQL 启动时是否自动启动 Group Replication。
- **默认值**：`OFF`
- **可修改**：是
- **配置示例**：为简化管理，推荐设置为 `ON`，确保 MySQL 重启后集群自动恢复。

```ini
group_replication_start_on_boot = ON
```

### 56. `group_replication_tls_source`
- **描述**：指定集群通信过程中使用的 TLS 密钥文件。
- **默认值**：`NULL`
- **可修改**：是
- **配置示例**：配置为合适的 TLS 密钥文件路径，确保集群通信安全。

```ini
group_replication_tls_source = "/etc/mysql/certs/tls-cert.pem"
```

### 57. `group_replication_transaction_size_limit`
- **描述**：控制单个事务的最大大小（字节），超过该限制的事务会被拒绝。
- **默认值**：`150000000`（150MB）
- **可修改**：是
- **配置示例**：根据业务场景调整，通常保持默认值即可，如果有大事务需求则适当增加。

```ini
group_replication_transaction_size_limit = 500000000  # 500MB
```

### 58. `group_replication_unreachable_majority_timeout`
- **描述**：指定当多数节点无法访问时的超时时间（毫秒），超时后节点将退出集群。
- **默认值**：`30000`（30秒）
- **可修改**：是
- **配置示例**：根据网络延迟情况调整，适当增加超时时间可以防止因短暂的网络中断导致节点退出。

```ini
group_replication_unreachable_majority_timeout = 60000  # 60秒
```

