### Keepalived详解

#### 1. Keepalived 简介

Keepalived 是一个用于提供高可用性（HA, High Availability）和负载均衡功能的开源工具。它主要用于防止网络服务因单点故障（SPOF, Single Point of Failure）而中断。Keepalived 通过 VRRP（Virtual Router Redundancy Protocol，虚拟路由冗余协议）实现虚拟IP地址的自动切换，确保服务的高可用性。

#### 2. 安装 Keepalived

**在 CentOS 上安装:**

```bash
sudo yum install keepalived
```

**在 Ubuntu 上安装:**
```bash
sudo apt-get install keepalived
```

#### 3. Keepalived 配置

Keepalived 的配置文件通常位于 `/etc/keepalived/keepalived.conf`。一个典型的配置文件包含以下几个部分：

- **Global Definitions**: 全局定义部分。
- **VRRP Instances**: VRRP 实例部分，用于定义虚拟IP和相关参数。
- **Virtual Server**: 虚拟服务器部分，用于定义负载均衡功能。

下面是一个示例配置文件：
```ini
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL
}

vrrp_script chk_mysql {
    script "/etc/keepalived/check_mysql.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.100
    }
    track_script {
        chk_mysql
    }
}

virtual_server 192.168.1.100 3306 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    persistence_timeout 50
    protocol TCP

    real_server 192.168.1.101 3306 {
        weight 1
        TCP_CHECK {
            connect_timeout 10
            nb_get_retry 3
            delay_before_retry 3
        }
    }
    real_server 192.168.1.102 3306 {
        weight 1
        TCP_CHECK {
            connect_timeout 10
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
```

#### 4. 运行原理
Keepalived 通过 VRRP 协议实现虚拟IP地址的主备切换。VRRP协议使得多台路由器共享一个虚拟IP地址，当主节点故障时，备用节点可以立即接管虚拟IP，从而实现高可用性。

- **VRRP (Virtual Router Redundancy Protocol)**: 用于提供冗余的路由器。它允许多台设备共享同一个虚拟IP。
- **Health Checking**: Keepalived 支持多种健康检查机制，可以监控后端服务器的状态。
- **Failover**: 当检测到主服务器故障时，Keepalived 会自动切换到备用服务器。

#### 5. MySQL + Keepalived 实战配置方案

假设有两台 MySQL 服务器，IP 分别为 `192.168.1.101` 和 `192.168.1.102`，虚拟IP为 `192.168.1.100`。下面是具体的步骤和脚本：

**配置MySQL服务器**

首先，确保两台MySQL服务器配置相同的用户和权限，以便在切换时不会出现权限问题。

**Keepalived 主服务器配置（192.168.1.101）**

1. 创建 MySQL 检查脚本 `/etc/keepalived/check_mysql.sh`:
    ```bash
    #!/bin/bash
    
    # 检查MySQL服务是否在运行  给一次机会看2后是MYSQL又开始运行
    counter=$(netstat -an|grep "LISTEN"|grep "3306"|wc -l)
    if [ "${counter}" -eq 0 ]; then
            sleep 2;
        counter=$(netstat -an|grep "LISTEN"|grep "3306"|wc -l)
        if [ "${counter}" -eq 0 ]; then
            killall keepalived
        fi
    fi
    ```
    给脚本添加执行权限：
    ```bash
    chmod +x /etc/keepalived/check_mysql.sh
    ```
    
2. 编辑 Keepalived 配置文件 `/etc/keepalived/keepalived.conf`:
    ```ini
    global_defs {
       router_id MYSQL_MASTER
    }
    
    vrrp_script chk_mysql {
        script "/etc/keepalived/check_mysql.sh"
        interval 5
        weight 2
    }
    
    vrrp_instance VI_1 {
        state MASTER
        interface eth0
        virtual_router_id 51
        priority 100
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass 1111
        }
        virtual_ipaddress {
            192.168.1.100
        }
        track_script {
            chk_mysql
        }
    }
    ```

**Keepalived 备服务器配置（192.168.1.102）**

1. 创建 MySQL 检查脚本 `/etc/keepalived/check_mysql.sh`:
    ```bash
    #!/bin/bash
    
    # 检查MySQL服务是否在运行  给一次机会看2后是MYSQL又开始运行
    counter=$(netstat -an|grep "LISTEN"|grep "3306"|wc -l)
    if [ "${counter}" -eq 0 ]; then
            sleep 2;
        counter=$(netstat -an|grep "LISTEN"|grep "3306"|wc -l)
        if [ "${counter}" -eq 0 ]; then
            killall keepalived
        fi
    fi
    ```
    给脚本添加执行权限：
    ```bash
    chmod +x /etc/keepalived/check_mysql.sh
    ```
    
2. 编辑 Keepalived 配置文件 `/etc/keepalived/keepalived.conf`:
    ```ini
    global_defs {
       router_id MYSQL_BACKUP
    }
    
    vrrp_script chk_mysql {
        script "/etc/keepalived/check_mysql.sh"
        interval 5
        weight 2
    }
    
    vrrp_instance VI_1 {
        state BACKUP
        interface eth0
        virtual_router_id 51
        priority 90
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass 1111
        }
        virtual_ipaddress {
            192.168.1.100
        }
        track_script {
            chk_mysql
        }
    }
    ```

**启动 Keepalived**

在两台服务器上分别启动 Keepalived 服务：

```bash
sudo systemctl start keepalived
sudo systemctl enable keepalived
```



这样，当主服务器 `192.168.1.101` 上的 MySQL 服务故障时，Keepalived 会自动将虚拟IP `192.168.1.100` 切换到备服务器 `192.168.1.102`，确保 MySQL 服务的高可用性。

**查看Keepalived 运行状态**

```shell
sudo systemctl status keepalived` 
```

运行 `systemctl status keepalived` 命令可以检查 Keepalived 服务的状态。以下是该命令的返回信息的解释及可能的输出示例：

### 输出

```bash
$ systemctl status keepalived
● keepalived.service - Keepalive Daemon (LVS and VRRP)
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-06-25 15:23:54 CST; 1h 30min ago
     Docs: man:keepalived(8)
 Main PID: 12345 (keepalived)
   CGroup: /system.slice/keepalived.service
           ├─12345 /usr/sbin/keepalived -D
           ├─12346 /usr/sbin/keepalived -D
           └─12347 /usr/sbin/keepalived -D

Jun 25 15:23:54 server1 systemd[1]: Starting Keepalive Daemon (LVS and VRRP)...
Jun 25 15:23:54 server1 Keepalived_vrrp[12345]: Registering Kernel netlink reflector
Jun 25 15:23:54 server1 Keepalived_vrrp[12345]: Registering Kernel netlink command channel
Jun 25 15:23:54 server1 Keepalived_vrrp[12345]: Opening file '/etc/keepalived/keepalived.conf'.
Jun 25 15:23:54 server1 systemd[1]: Started Keepalive Daemon (LVS and VRRP).
Jun 25 15:23:54 server1 Keepalived_healthcheckers[12346]: Initializing ipvs
Jun 25 15:23:54 server1 Keepalived_healthcheckers[12346]: Netlink reflector reports IP 192.168.1.101 added
Jun 25 15:23:54 server1 Keepalived_healthcheckers[12346]: Registering service 192.168.1.101
Jun 25 15:23:54 server1 Keepalived_vrrp[12347]: VRRP_Instance(VI_1) Transition to MASTER STATE
```

### 解释
- **Loaded**: 表示服务的配置文件是否已加载，并显示其路径。
- **Active**: 显示服务的当前状态。在这个示例中，服务是“active (running)”表示正在运行。
- **Main PID**: 主进程的PID。
- **CGroup**: 服务相关的控制组信息，包括所有与该服务相关的进程。
- **Logs**: 各种日志信息，包括启动时间、重要事件和错误消息等。

### 检查服务状态
1. **Active (running)**: Keepalived 服务正在运行。
2. **Active (exited)**: Keepalived 服务已成功启动并退出（可能因为它是一次性任务）。
3. **Inactive (dead)**: Keepalived 服务未运行。
4. **Failed**: Keepalived 服务启动失败或崩溃。

### 常见问题及解决方法
- **服务未启动**: 如果状态显示为 `inactive (dead)` 或 `failed`，可以尝试重启服务：
    ```bash
    sudo systemctl restart keepalived
    ```
- **配置错误**: 检查 `/etc/keepalived/keepalived.conf` 文件是否正确配置，并确保没有语法错误。
    ```bash
    sudo keepalived -t -f /etc/keepalived/keepalived.conf
    ```
- **日志检查**: 查看详细日志以排查问题：
    ```bash
    sudo journalctl -u keepalived
    ```

这些步骤和信息可以帮助你确保 Keepalived 服务正确运行，并快速诊断和解决可能的问题。