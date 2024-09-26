**Linux服务器之间的SSH免密登录可以简化日常管理和操作。以下是实现步骤和相关脚本：**

### 依赖条件
1. **OpenSSH**：所有服务器都需要安装并启用OpenSSH服务。
2. **网络连通性**：服务器之间需要网络连通。
3. **用户权限**：需要对所有服务器有合适的用户权限。

### 实现步骤

假设有三台服务器，分别是：

* Server A: `serverA.example.com`
* Server B: `serverB.example.com`
* Server C: `serverC.example.com`

目标是使这三台服务器上的指定用户可以相互SSH登录而无需密码。

#### 步骤 1：在每台服务器上生成SSH密钥对

在每台服务器的指定用户下生成SSH密钥对：

```bash
# 先检查OpenSSH 是否运行
systemctl status sshd

# 或者
ssh -V

# 生成密钥对 长度4096
ssh-keygen -t rsa -b 4096
```
在生成过程中，按Enter键接受默认位置（`~/.ssh/id_rsa`）和不设置密码短语。

#### 步骤 2：将公钥复制到其他服务器
可以使用 `ssh-copy-id` 命令将公钥复制到其他服务器。假设每台服务器的用户都是 `user`。

在 Server A 上运行以下命令以将公钥复制到 Server B 和 Server C：

```bash
ssh-copy-id user@serverB.example.com
ssh-copy-id user@serverC.example.com
```
在 Server B 上运行以下命令以将公钥复制到 Server A 和 Server C：

```bash
ssh-copy-id user@serverA.example.com
ssh-copy-id user@serverC.example.com
```
在 Server C 上运行以下命令以将公钥复制到 Server A 和 Server B：

```bash
ssh-copy-id user@serverA.example.com
ssh-copy-id user@serverB.example.com
```
#### 步骤 3：手动复制公钥（可选）
如果 `ssh-copy-id` 命令不可用，可以手动复制公钥。在 Server A 上，将公钥内容（通常位于 `~/.ssh/id_rsa.pub`）复制到 Server B 和 Server C 的 `~/.ssh/authorized_keys` 文件中：

```bash
cat ~/.ssh/id_rsa.pub | ssh user@serverB.example.com "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
cat ~/.ssh/id_rsa.pub | ssh user@serverC.example.com "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
在 Server B 上，将公钥内容复制到 Server A 和 Server C 的 `~/.ssh/authorized_keys` 文件中：

```bash
cat ~/.ssh/id_rsa.pub | ssh user@serverA.example.com "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
cat ~/.ssh/id_rsa.pub | ssh user@serverC.example.com "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
在 Server C 上，将公钥内容复制到 Server A 和 Server B 的 `~/.ssh/authorized_keys` 文件中：

```bash
cat ~/.ssh/id_rsa.pub | ssh user@serverA.example.com "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
cat ~/.ssh/id_rsa.pub | ssh user@serverB.example.com "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
#### 步骤 4：验证SSH免密登录
在完成上述步骤后，尝试从一台服务器SSH登录到其他服务器，以验证是否成功配置了免密登录。

在 Server A 上：

```bash
ssh user@serverB.example.com
ssh user@serverC.example.com
```
在 Server B 上：

```bash
ssh user@serverA.example.com
ssh user@serverC.example.com
```
在 Server C 上：

```bash
ssh user@serverA.example.com
ssh user@serverB.example.com
```
### 完整脚本
以下是一个完整的脚本示例，可在每台服务器上执行来自动化这些步骤：

```bash
#!/bin/bash

# 生成SSH密钥对（如果不存在）
if [ ! -f ~/.ssh/id_rsa ]; then
    ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
fi

# 定义服务器列表
servers=("serverA.example.com" "serverB.example.com" "serverC.example.com")

# 当前服务器的hostname
current_server=$(hostname)

# 当前用户
user=$(whoami)

# 将公钥复制到其他服务器
for server in "${servers[@]}"; do
    if [ "$server" != "$current_server" ]; then
        ssh-copy-id $user@$server
    fi
done
```
将以上脚本保存为 `setup_ssh.sh`，然后在每台服务器上运行：

```bash
chmod +x setup_ssh.sh
./setup_ssh.sh
```
这将自动生成SSH密钥对并将公钥复制到其他服务器，完成SSH免密登录配置。