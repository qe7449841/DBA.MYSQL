**`tracert`（Windows操作系统中）或 `traceroute`（类Unix操作系统中）是一个网络诊断工具，用于确定从你的计算机到目标主机之间的路径。它显示经过的每个路由器（或跳数），并测量到每个跳数的响应时间。通过这些信息，可以帮助诊断网络连接问题。**



**想象你和你的朋友们正在玩一场“网络大冒险”游戏。你们的任务是追踪一封从你家发送到某个远方朋友家的电子邮件，看看它在互联网上到底经过了哪些节点。这就是 `tracert` 和 `traceroute` 工具的作用——追踪数据包的路径！**

**每次数据包在网络上跳跃（即通过一个路由器），我们都会记录下它的时间和经过的节点。这就像在城市地图上记录你从家到朋友家途经的每个车站一样。**

#### Windows 平台：`tracert`

**步骤 1**：打开命令提示符

1. 按下 `Win + R` 打开运行窗口。
2. 输入 `cmd` 并按下回车。

**步骤 2**：使用 `tracert` 命令

在命令提示符中输入以下命令，然后按下回车：

```sh
tracert www.baidu.com
```

**故事**：你开始追踪从你家（你的电脑）到百度总部的电子邮件。`tracert` 命令会告诉你邮件经过了哪些“车站”（路由器），以及每站花了多少时间。

**输出示例**：

```
Tracing route to www.baidu.com [172.217.14.196]
over a maximum of 30 hops:

  1    <1 ms    <1 ms    <1 ms  192.168.1.1
  2     3 ms     2 ms     2 ms  10.0.0.1
  3     6 ms     6 ms     7 ms  172.217.14.196

Trace complete.
```

**解释**：每一行代表邮件经过的一个路由器（节点），第一列是跳数（从1开始），后面是该节点的IP地址和响应时间。你可以想象每一跳是你走向目的地的一步。

#### Linux/Unix 平台：`traceroute`

**步骤 1**：打开终端

**步骤 2**：使用 `traceroute` 命令

在终端中输入以下命令，然后按下回车：

```sh
traceroute www.baidu.com
```

**故事**：同样，你开始追踪从你家（你的电脑）到百度总部的电子邮件，但这次是在Linux/Unix系统上。`traceroute` 命令会告诉你邮件经过了哪些“车站”（路由器），以及每站花了多少时间。

**输出示例**：

```
traceroute to www.baidu.com (172.217.14.196), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)  0.415 ms  0.401 ms  0.387 ms
 2  10.0.0.1 (10.0.0.1)  1.212 ms  1.199 ms  1.185 ms
 3  172.217.14.196 (172.217.14.196)  2.502 ms  2.487 ms  2.474 ms
```

**解释**：输出和 `tracert` 类似，每一行显示一个经过的路由器和响应时间。你可以看到你的邮件在每一步的旅程。

**通过 `tracert` 和 `traceroute` 命令，你就像一个网络探险家，可以追踪数据包在互联网上的路径。每个节点就像冒险中的一个站点，你可以查看每个站点的响应时间、地址等信息。通过这个网络冒险游戏，你不仅可以了解网络连接的状态，还可以快速定位和解决网络问题。希望你在“网络大冒险”中玩得愉快！**





### `tracert` 命令详解（Windows）

#### 基本用法

```sh
tracert <目标主机>
```

例如：

```sh
tracert www.baidu.com
```

#### 常用参数

1. **-d**：不解析IP地址为主机名，以减少处理时间。

   ```sh
   tracert -d www.baidu.com
   ```

2. **-h <最大跃点数>**：指定最大跃点数，默认是30。

   ```sh
   tracert -h 15 www.baidu.com
   ```

3. **-w <超时时间>**：设置等待每个回复的超时时间（以毫秒为单位）。

   ```sh
   tracert -w 1000 www.baidu.com
   ```

4. **-4**：强制使用IPv4。

   ```sh
   tracert -4 www.baidu.com
   ```

5. **-6**：强制使用IPv6。

   ```sh
   tracert -6 www.baidu.com
   ```



### `traceroute` 命令详解（Linux/Unix）

#### 基本用法

```sh
traceroute <目标主机>
```

例如：

```sh
traceroute www.baidu.com
```

#### 常用参数

1. **-n**：不解析IP地址为主机名，以减少处理时间。

   ```sh
   traceroute -n www.baidu.com
   ```

2. **-m <最大跃点数>**：指定最大跃点数，默认是30。

   ```sh
   traceroute -m 15 www.baidu.com
   ```

3. **-w <超时时间>**：设置等待每个回复的超时时间（以秒为单位）。

   ```sh
   traceroute -w 2 www.baidu.com
   ```

4. **-I**：使用ICMP ECHO替代默认的UDP数据包。

   ```sh
   traceroute -I www.baidu.com
   ```

5. **-T**：使用TCP SYN扫描替代默认的UDP数据包。

   ```sh
   traceroute -T www.baidu.com
   ```

### 使用示例

#### Windows用法示例

1. **基本用法**

   ```sh
   tracert www.baidu.com
   ```

   输出示例：

   ```
   Tracing route to www.baidu.com [172.217.14.196]
   over a maximum of 30 hops:
   
     1    <1 ms    <1 ms    <1 ms  192.168.1.1
     2     3 ms     2 ms     2 ms  10.0.0.1
     3     6 ms     6 ms     7 ms  172.217.14.196
   
   Trace complete.
   ```

2. **不解析IP地址**

   ```sh
   tracert -d www.baidu.com
   ```

3. **设置最大跃点数为10**

   ```sh
   tracert -h 10 www.baidu.com
   ```

4. **设置超时时间为2000毫秒**

   ```sh
   tracert -w 2000 www.baidu.com
   ```

5. **强制使用IPv4**

   ```sh
   tracert -4 www.baidu.com
   ```

#### Linux用法示例

1. **基本用法**

   ```sh
   traceroute www.baidu.com
   ```

   输出示例：

   ```
   traceroute to www.baidu.com (172.217.14.196), 30 hops max, 60 byte packets
    1  192.168.1.1 (192.168.1.1)  0.415 ms  0.401 ms  0.387 ms
    2  10.0.0.1 (10.0.0.1)  1.212 ms  1.199 ms  1.185 ms
    3  172.217.14.196 (172.217.14.196)  2.502 ms  2.487 ms  2.474 ms
   ```

2. **不解析IP地址**

   ```sh
   traceroute -n www.baidu.com
   ```

3. **设置最大跃点数为10**

   ```sh
   traceroute -m 10 www.baidu.com
   ```

4. **设置超时时间为2秒**

   ```sh
   traceroute -w 2 www.baidu.com
   ```

5. **使用ICMP ECHO**

   ```sh
   traceroute -I www.baidu.com
   ```

6. **使用TCP SYN**

   ```sh
   traceroute -T www.baidu.com
   ```

### 输出

每个跳数（hop）有三列时间表示发出三个探测包的响应时间。响应时间的单位是毫秒（ms）。以下是一个示例输出的解释：

```
Tracing route to www.baidu.com [172.217.14.196]
over a maximum of 30 hops:

  1    <1 ms    <1 ms    <1 ms  192.168.1.1
  2     3 ms     2 ms     2 ms  10.0.0.1
  3     6 ms     6 ms     7 ms  172.217.14.196

Trace complete.
```

- **跳数 1**：第一个路由器的IP地址是 `192.168.1.1`，响应时间小于1毫秒。
- **跳数 2**：第二个路由器的IP地址是 `10.0.0.1`，响应时间是2-3毫秒。
- **跳数 3**：目标主机的IP地址是 `172.217.14.196`，响应时间是6-7毫秒。

### `tracert` 和 `traceroute` 的高级用法

#### 添加更多挑战

**限制最大跃点数**：你可以告诉工具最多追踪多少个节点，就像给自己设定一个寻找宝藏的时间限制。

```sh
tracert -h 10 www.baidu.com  # Windows
traceroute -m 10 www.baidu.com  # Linux/Unix
```

**隐藏身份**：你可以选择不解析IP地址为主机名，这样就像戴上了面具，避免让每个节点知道你的身份。

```sh
tracert -d www.baidu.com  # Windows
traceroute -n www.baidu.com  # Linux/Unix
```

**调整时间**：你可以设置每一跳等待的时间，就像在冒险中给每个站点设定了一个停留时间。

```sh
tracert -w 2000 www.baidu.com  # Windows, 2000毫秒
traceroute -w 2 www.baidu.com  # Linux/Unix, 2秒
```

### 总结

`tracert` 和 `traceroute` 是强大的网络诊断工具，能够帮助识别网络路径中的问题。通过了解和使用这些工具及其参数，你可以有效地诊断和优化网络连接。











