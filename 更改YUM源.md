# 切换YUM源到阿里云

**centos 7 在2024年6月30日，生命周期结束，官方不再进行支持维护，而很多环境一时之间无法完全更新替换操作系统，因此对于yum源还是需要的，特别是对于互联网环境来说，在线yum源使用方便很多，而不需要去搭建本地yum源和内网yum源。**

**这里以阿里云为例，其他国内开源镜像站类似。**

将YUM源改为阿里云源以下是详细的步骤：

### 1. 备份现有的YUM源配置文件

首先，备份现有的YUM源配置文件，以防需要恢复到原来的配置：

```bash
sudo cp /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
```

### 2. 更改YUM源到阿里云源

接下来，将YUM源更改为阿里云源。你可以直接编辑现有的`CentOS-Base.repo`文件或创建一个新的YUM源文件。

#### 编辑现有的`CentOS-Base.repo`文件

```bash
sudo vi /etc/yum.repos.d/CentOS-Base.repo
```

将文件内容替换为以下内容：

```ini
[base]
name=CentOS-$releasever - Base - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

[updates]
name=CentOS-$releasever - Updates - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

[extras]
name=CentOS-$releasever - Extras - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

[centosplus]
name=CentOS-$releasever - Plus - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos/$releasever/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

[extras]
name=CentOS-$releasever - Extras - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
```

#### 或者，创建新的YUM源文件

```bash

cat > CentOS-aliyun.repo << 'EOF'
[base]
name=CentOS-$releasever - Base - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

[updates]
name=CentOS-$releasever - Updates - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

[extras]
name=CentOS-$releasever - Extras - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

[centosplus]
name=CentOS-$releasever - Plus - mirrors.aliyun.com
baseurl=http://mirrors.aliyun.com/centos/$releasever/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

EOF
```

### 3. 清理YUM缓存并更新YUM仓库

```bash
sudo yum clean all
sudo yum makecache
```

### 4. 测试YUM源

最后，测试新的YUM源配置是否正常工作，例如安装或更新一个软件包：

```bash
sudo yum update
sudo yum install vim
```

这样，你就将CentOS 7.9的YUM源成功更改为阿里云源，可以继续使用YUM来安装和更新软件包了。