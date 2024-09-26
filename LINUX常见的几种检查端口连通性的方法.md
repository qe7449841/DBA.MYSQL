# LINUX常见的几种检查端口连通性的方法

**平时在实践中总结了几种常用的测试端口是否开启的方法。**

要检查本机是否能够连通远程服务器的特定端口，可以使用以下几种方法：

### 方法一：使用 `telnet`

`telnet` 是一个简单的命令行工具，可以用来检查远程服务器的端口是否开放。以下是使用 `telnet` 的示例：

```bash

telnet remote.server.com 80

```

如果能够连接成功，你会看到类似于以下的输出：

```

Trying remote.server.com...
Connected to remote.server.com.
Escape character is '^]'.

```

如果连接失败，你会看到类似于以下的输出：

```

Trying remote.server.com...
telnet: Unable to connect to remote host: Connection refused

```

### 方法二：使用 `nc` (Netcat)

`nc` 是一个功能强大的网络工具，可以用于检查端口连通性。以下是使用 `nc` 的示例：

```bash

nc -zv remote.server.com 80

```

`-z` 表示扫描模式，不发送任何数据，`-v` 表示详细模式。如果能够连接成功，你会看到类似于以下的输出：

```

remote.server.com [remote.server.com] 80 (http) open
```

如果连接失败，你会看到类似于以下的输出：

```

nc: connect to remote.server.com port 80 (tcp) failed: Connection refused

```

### 方法三：使用 `nmap`

`nmap` 是一个网络扫描工具，可以用于检测远程服务器的开放端口。以下是使用 `nmap` 的示例：

```bash

nmap -p 80 remote.server.com
```

`-p` 指定要检查的端口。如果端口开放，你会看到类似于以下的输出：

```

Starting Nmap 7.91 ( https://nmap.org ) at 2021-06-19 12:00 UTC
Nmap scan report for remote.server.com (192.0.2.1)
Host is up (0.0010s latency).

PORT   STATE SERVICE
80/tcp open  http

```

如果端口关闭，你会看到类似于以下的输出：

```

Starting Nmap 7.91 ( https://nmap.org ) at 2021-06-19 12:00 UTC
Nmap scan report for remote.server.com (192.0.2.1)
Host is up (0.0010s latency).

PORT   STATE  SERVICE
80/tcp closed http

```

### 方法四：使用 `curl`

`curl` 是一个命令行工具，可以用来检查HTTP/HTTPS端口的连通性。以下是使用 `curl` 的示例：

```bash

curl -I remote.server.com:80

```

如果能够连接成功，你会看到HTTP头信息：

```

HTTP/1.1 200 OK
Date: Sat, 19 Jun 2021 12:00:00 GMT
Server: Apache/2.4.41 (Ubuntu)
...

```

如果连接失败，你会看到类似于以下的输出：

```

curl: (7) Failed to connect to remote.server.com port 80: Connection refused

```

### 总结

上述方法可以帮助你检查本机是否能够连通远程服务器的特定端口。根据具体需求选择合适的工具。以下是一个使用 `nc` 的完整示例脚本，可以保存为 `check_port.sh`：

```bash

#!/bin/bash

# 检查输入参数
if [ $# -ne 2 ]; then
    echo "Usage: $0 <hostname> <port>"
    exit 1
fi

hostname=$1
port=$2

# 使用 nc 检查端口连通性
nc -zv $hostname $port
if [ $? -eq 0 ]; then
    echo "Port $port on $hostname is open."
else
    echo "Port $port on $hostname is closed or not reachable."
fi

```

执行此脚本：

```bash

chmod +x check_port.sh
./check_port.sh remote.server.com 80

```