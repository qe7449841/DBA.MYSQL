### PostgreSQL 14 在 Red Hat 8 上的二进制安装教程及日常运维命令

#### 一、准备工作

1. **系统环境**
   - 操作系统：Red Hat 8
   - 服务器配置：8 核 CPU，32GB 内存

2. **安装依赖包**
   ```bash
   sudo yum install -y gcc readline-devel zlib-devel
   ```

3. **创建 PostgreSQL 用户**
   ```bash
   sudo useradd postgres
   sudo passwd postgres
   ```

#### 二、下载和解压 PostgreSQL 14

1. **下载 PostgreSQL 14 二进制包**
   ```bash
   wget https://ftp.postgresql.org/pub/source/v14.0/postgresql-14.0.tar.gz
   ```

2. **解压文件**
   ```bash
   tar -xzf postgresql-14.0.tar.gz
   cd postgresql-14.0
   ```

#### 三、编译和安装 PostgreSQL

1. **配置编译选项**
   ```bash
   ./configure --prefix=/usr/local/pgsql --with-pgport=5432 --with-perl --with-python --with-openssl
   ```

2. **编译和安装**
   ```bash
   make
   sudo make install
   ```

3. **设置目录权限**
   ```bash
   sudo mkdir /usr/local/pgsql/data
   sudo chown postgres:postgres /usr/local/pgsql/data
   sudo mkdir /var/log/postgresql
   sudo chown postgres:postgres /var/log/postgresql
   ```

#### 四、初始化数据库集群

1. **切换到 PostgreSQL 用户**
   ```bash
   su - postgres
   ```

2. **初始化数据库**
   ```bash
   /usr/local/pgsql/bin/initdb -D /usr/local/pgsql/data
   ```

#### 五、配置和启动 PostgreSQL

1. **修改配置文件**
   编辑 `/usr/local/pgsql/data/postgresql.conf` 和 `/usr/local/pgsql/data/pg_hba.conf` 文件，确保正确配置数据库端口、监听地址和访问权限。

2. **设置环境变量**
   ```bash
   echo "export PATH=/usr/local/pgsql/bin:$PATH" >> ~/.bash_profile
   source ~/.bash_profile
   ```

3. **启动 PostgreSQL**
   ```bash
   /usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data -l /var/log/postgresql/logfile start
   ```

4. **设置 PostgreSQL 开机自启动**
   创建系统服务文件 `/etc/systemd/system/postgresql.service`：
   ```ini
   [Unit]
   Description=PostgreSQL database server
   Documentation=https://www.postgresql.org/docs/14/static/
   After=network.target
   
   [Service]
   Type=forking
   User=postgres
   ExecStart=/usr/local/pgsql/bin/pg_ctl start -D /usr/local/pgsql/data -l /var/log/postgresql/logfile
   ExecStop=/usr/local/pgsql/bin/pg_ctl stop -D /usr/local/pgsql/data
   ExecReload=/usr/local/pgsql/bin/pg_ctl reload -D /usr/local/pgsql/data
   TimeoutSec=300
   
   [Install]
   WantedBy=multi-user.target
   ```

   **启用服务**
   ```bash
   sudo systemctl enable postgresql
   sudo systemctl start postgresql
   ```

#### 六、日常运维命令

1. **启动和停止 PostgreSQL 服务**
   ```bash
   sudo systemctl start postgresql
   sudo systemctl stop postgresql
   sudo systemctl restart postgresql
   ```

2. **检查 PostgreSQL 服务状态**
   ```bash
   sudo systemctl status postgresql
   ```

3. **连接数据库**
   ```bash
   psql -U postgres
   ```

4. **备份数据库**
   ```bash
   pg_dump dbname > backup.sql
   ```

5. **恢复数据库**
   ```bash
   psql dbname < backup.sql
   ```

6. **查看活动连接**
   ```bash
   SELECT * FROM pg_stat_activity;
   ```

7. **监控数据库性能**
   ```bash
   SELECT * FROM pg_stat_database;
   ```

8. **重载配置文件**
   ```bash
   sudo systemctl reload postgresql
   ```

9. **查看日志**
   ```bash
   tail -f /var/log/postgresql/logfile
   ```

#### 结论

通过以上步骤，您可以在 Red Hat 8 上成功安装和配置 PostgreSQL 14，并掌握基本的日常运维命令。这些命令可以帮助您更好地管理和维护 PostgreSQL 数据库，确保其稳定运行。