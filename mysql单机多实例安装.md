### MySQL 8.0二进制安装多实例配置指南

#### 1. 下载并解压MySQL二进制文件

```bash
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.33-linux-glibc2.12-x86_64.tar.xz
tar -xvf mysql-8.0.33-linux-glibc2.12-x86_64.tar.xz
mv mysql-8.0.33-linux-glibc2.12-x86_64 /usr/local/mysql
```

#### 2. 创建MySQL用户和组
```bash
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
```

#### 3. 初始化和配置多实例
创建多个数据目录和配置文件，例如实例1和实例2：
```bash
mkdir -p /usr/local/mysql/data1 /usr/local/mysql/data2
chown -R mysql:mysql /usr/local/mysql

# 配置文件实例1
cat <<EOF > /etc/my1.cnf
[mysqld]
user=mysql
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data1
port=3306
socket=/tmp/mysql1.sock
log_error=/usr/local/mysql/data1/mysql_error.log

innodb_buffer_pool_size=1G
max_connections=500
EOF

# 配置文件实例2
cat <<EOF > /etc/my2.cnf
[mysqld]
user=mysql
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data2
port=3307
socket=/tmp/mysql2.sock
log_error=/usr/local/mysql/data2/mysql_error.log

innodb_buffer_pool_size=1G
max_connections=500
EOF

# 初始化数据库
/usr/local/mysql/bin/mysqld --defaults-file=/etc/my1.cnf --initialize --user=mysql
/usr/local/mysql/bin/mysqld --defaults-file=/etc/my2.cnf --initialize --user=mysql
```

#### 4. 配置环境变量
将以下行添加到 `/etc/profile`：
```bash
export PATH=$PATH:/usr/local/mysql/bin
```
然后运行 `source /etc/profile`。

#### 5. 配置启动服务
创建MySQL服务文件 `/etc/systemd/system/mysql1.service` 和 `/etc/systemd/system/mysql2.service`：
```ini
# MySQL1
[Unit]
Description=MySQL Server 1
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/mysql/bin/mysqld --defaults-file=/etc/my1.cnf --daemonize
ExecStop=/usr/local/mysql/bin/mysqladmin --defaults-file=/etc/my1.cnf shutdown
User=mysql
Group=mysql

[Install]
WantedBy=multi-user.target
```

```ini
# MySQL2
[Unit]
Description=MySQL Server 2
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/mysql/bin/mysqld --defaults-file=/etc/my2.cnf --daemonize
ExecStop=/usr/local/mysql/bin/mysqladmin --defaults-file=/etc/my2.cnf shutdown
User=mysql
Group=mysql

[Install]
WantedBy=multi-user.target
```

启动并启用MySQL服务：
```bash
systemctl daemon-reload
systemctl start mysql1
systemctl start mysql2
systemctl enable mysql1
systemctl enable mysql2
```

#### 6. 配置密码插件和创建远程访问用户
```bash
/usr/local/mysql/bin/mysql --defaults-file=/etc/my1.cnf -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH 'mysql_native_password' BY 'YourRootPassword1';"
/usr/local/mysql/bin/mysql --defaults-file=/etc/my1.cnf -e "CREATE USER 'remote_user'@'%' IDENTIFIED BY 'RemoteUserPassword';"
/usr/local/mysql/bin/mysql --defaults-file=/etc/my1.cnf -e "GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%';"
/usr/local/mysql/bin/mysql --defaults-file=/etc/my1.cnf -e "FLUSH PRIVILEGES;"

/usr/local/mysql/bin/mysql --defaults-file=/etc/my2.cnf -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH 'mysql_native_password' BY 'YourRootPassword2';"
/usr/local/mysql/bin/mysql --defaults-file=/etc/my2.cnf -e "CREATE USER 'remote_user'@'%' IDENTIFIED BY 'RemoteUserPassword';"
/usr/local/mysql/bin/mysql --defaults-file=/etc/my2.cnf -e "GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%';"
/usr/local/mysql/bin/mysql --defaults-file=/etc/my2.cnf -e "FLUSH PRIVILEGES;"
```

#### 7. 一键安装脚本
创建一键安装脚本 `install_mysql_multi.sh`：
```bash
#!/bin/bash

# 下载并解压MySQL
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.33-linux-glibc2.12-x86_64.tar.xz
tar -xvf mysql-8.0.33-linux-glibc2.12-x86_64.tar.xz
mv mysql-8.0.33-linux-glibc2.12-x86_64 /usr/local/mysql

# 创建MySQL用户和组
groupadd mysql
useradd -r -g mysql -s /bin/false mysql

# 创建数据目录和配置文件
mkdir -p /usr/local/mysql/data1 /usr/local/mysql/data2
chown -R mysql:mysql /usr/local/mysql

cat <<EOF > /etc/my1.cnf
[mysqld]
user=mysql
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data1
port=3306
socket=/tmp/mysql1.sock
log_error=/usr/local/mysql/data1/mysql_error.log

innodb_buffer_pool_size=1G
max_connections=500
EOF

cat <<EOF > /etc/my2.cnf
[mysqld]
user=mysql
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data2
port=3307
socket=/tmp/mysql2.sock
log_error=/usr/local/mysql/data2/mysql_error.log

innodb_buffer_pool_size=1G
max_connections=500
EOF

# 初始化数据库
/usr/local/mysql/bin/mysqld --defaults-file=/etc/my1.cnf --initialize --user=mysql
/usr/local/mysql/bin/mysqld --defaults-file=/etc/my2.cnf --initialize --user=mysql

# 配置环境变量
echo 'export PATH=$PATH:/usr/local/mysql/bin' >> /etc/profile
source /etc/profile

# 配置启动服务
cat <<EOF > /etc/systemd/system/mysql1.service
[Unit]
Description=MySQL Server 1
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/mysql/bin/mysqld --defaults-file=/etc/my1.cnf --daemonize
ExecStop=/usr/local/mysql/bin/mysqladmin --defaults-file=/etc/my1.cnf shutdown
User=mysql
Group=mysql

[Install]
WantedBy=multi-user.target
EOF

cat <<EOF > /etc/systemd/system/mysql2.service
[Unit]
Description=MySQL Server 2
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/mysql/bin/mysqld --defaults-file=/etc/my2.cnf --daemonize
ExecStop=/usr/local/mysql/bin/mysqladmin --defaults-file=/etc/my2.cnf shutdown
User=mysql
Group=mysql

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start mysql1
systemctl start mysql2
systemctl enable mysql1
systemctl enable mysql2

# 配置密码插件和创建远程访问用户
/usr/local/mysql/bin/mysql --defaults-file=/etc/my1.cnf -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH 'mysql_native_password' BY 'YourRootPassword1';"
/usr/local/mysql/bin/mysql --defaults-file=/etc/my1.cnf -e "CREATE USER 'remote_user'@'%' IDENTIFIED BY 'RemoteUserPassword';"
/usr/local/mysql/bin/mysql --defaults-file=/etc/my1.cnf -e "GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%';"
/usr/local/mysql/bin/mysql --defaults-file=/etc/my1.cnf -e "FLUSH PRIVILEGES;"

/usr/local/mysql/bin/mysql --defaults-file=/etc/my2.cnf -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH 'mysql_native_password' BY 'YourRootPassword2';"
/usr/local/mysql/bin/mysql --defaults-file=/etc/my2.cnf -e "CREATE USER 'remote_user'@'%' IDENTIFIED BY 'RemoteUserPassword';"
/usr/local/mysql/bin/mysql --defaults-file=/etc/my2.cnf -e "GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%';"
/usr/local/mysql/bin/mysql --defaults-file=/etc/my2.cnf -e "FLUSH PRIVILEGES;"

echo "MySQL 8.0.33 多实例安装完成！"
```

运行脚本：
```bash
chmod +x install_mysql_multi.sh
./install_mysql_multi.sh
```

#### 8. 日常运维多实例管理常用脚本

##### 检查实例状态
```bash
#!/bin/bash
systemctl status mysql1
systemctl status mysql2
```

##### 