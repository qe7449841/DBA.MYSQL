## MySQL 8.4.1LTS 二进制安装方法

### 1. 准备工作

确保你的系统满足以下要求：
- 操作系统：CentOS 7 或更新版本
- 必要软件包：`wget`、`gcc`、`glibc`、`libaio`、`perl`

### 2. 下载 MySQL 二进制包
首先，从 MySQL 官方网站下载 MySQL 8.4.0LTS 的二进制包。

```bash
wget https://dev.mysql.com/get/Downloads/MySQL-8.4/mysql-8.4.1-linux-glibc2.28-x86_64.tar.xz
```

### 3. 安装 MySQL

#### 3.1 解压二进制包

```bash
tar -xvf mysql-8.4.1-linux-glibc2.28-x86_64.tar.xz
```

#### 3.2 移动并重命名目录
```bash
mv mysql-8.4.1-linux-glibc2.28-x86_64 /usr/local/mysql
```

#### 3.3 创建 MySQL 用户和组
```bash
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
```

#### 3.4 设置目录权限
```bash
cd /usr/local/mysql
mkdir mysql-files
chown mysql:mysql mysql-files
chmod 750 mysql-files
```

### 4. 初始化数据库
```bash
bin/mysqld --initialize --user=mysql
```

### 5. 配置 MySQL

#### 5.1 创建配置文件
在 `/etc` 目录下创建 `my.cnf` 文件，并添加如下内容：

```ini
[mysqld]
basedir = /usr/local/mysql
datadir = /usr/local/mysql/data
socket = /var/lib/mysql/mysql.sock
user = mysql
symbolic-links = 0

[mysqld_safe]
log-error = /var/log/mysqld.log
pid-file = /var/run/mysqld/mysqld.pid
```

#### 5.2 设置环境变量
在 `.bash_profile` 或 `.bashrc` 中添加以下内容：

```bash
export PATH=/usr/local/mysql/bin:$PATH
```
然后运行：
```bash
source ~/.bash_profile
```

### 6. 配置 MySQL 服务

#### 6.1 创建 Systemd 服务文件
在 `/etc/systemd/system` 目录下创建 `mysqld.service` 文件，并添加如下内容：

```ini
[Unit]
Description=MySQL Server
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/mysql/bin/mysqld_safe
ExecStop=/usr/local/mysql/bin/mysqladmin shutdown
User=mysql
Group=mysql
PIDFile=/var/run/mysqld/mysqld.pid
LimitNOFILE = 5000

[Install]
WantedBy=multi-user.target
```

#### 6.2 启动并启用 MySQL 服务
```bash
systemctl start mysqld
systemctl enable mysqld
```

### 7. 验证安装

#### 7.1 检查服务状态
```bash
systemctl status mysqld
```

#### 7.2 连接到 MySQL
初次连接时，MySQL 会生成一个临时密码。你可以在 `/var/log/mysqld.log` 中找到该密码。使用以下命令登录 MySQL：

```bash
mysql -u root -p
```

### 8. Shell 脚本实现一键安装

创建一个名为 `install_mysql.sh` 的脚本，并将以下内容粘贴进去：

```bash
#!/bin/bash

# 下载并解压 MySQL 二进制包
wget https://dev.mysql.com/get/Downloads/MySQL-8.4/mysql-8.4.1-linux-glibc2.28-x86_64.tar.xz
tar -xvf mysql-8.4.1-linux-glibc2.28-x86_64.tar.xz
mv mysql-8.4.1-linux-glibc2.28-x86_64 /usr/local/mysql

# 创建 MySQL 用户和组
groupadd mysql
useradd -r -g mysql -s /bin/false mysql

# 设置目录权限
cd /usr/local/mysql
mkdir mysql-files
chown mysql:mysql mysql-files
chmod 750 mysql-files

# 初始化数据库
bin/mysqld --initialize --user=mysql

# 创建配置文件
cat <<EOF > /etc/my.cnf
[mysqld]
basedir = /usr/local/mysql
datadir = /usr/local/mysql/data
socket = /var/lib/mysql/mysql.sock
user = mysql
symbolic-links = 0

[mysqld_safe]
log-error = /var/log/mysqld.log
pid-file = /var/run/mysqld/mysqld.pid
EOF

# 设置环境变量
echo "export PATH=/usr/local/mysql/bin:\$PATH" >> ~/.bash_profile
source ~/.bash_profile

# 创建 Systemd 服务文件
cat <<EOF > /etc/systemd/system/mysqld.service
[Unit]
Description=MySQL Server
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/mysql/bin/mysqld_safe
ExecStop=/usr/local/mysql/bin/mysqladmin shutdown
User=mysql
Group=mysql
PIDFile=/var/run/mysqld/mysqld.pid
LimitNOFILE = 5000

[Install]
WantedBy=multi-user.target
EOF

# 启动并启用 MySQL 服务
systemctl daemon-reload
systemctl start mysqld
systemctl enable mysqld

# 提示用户检查临时密码
echo "MySQL 安装完成。请查看 /var/log/mysqld.log 以获取 root 用户的临时密码。"
```

保存并退出，然后给脚本执行权限并运行：

```bash
chmod +x install_mysql.sh
./install_mysql.sh
```

通过上述步骤，你可以成功安装并配置 MySQL 8.4.1LTS，并使其在 CentOS 7 系统上以服务的方式运行。