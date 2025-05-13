# centos源rep修改

### 步骤 1: 备份现有的 YUM 配置文件

在修改之前，最好备份现有的 YUM 配置文件，以防需要还原：

```less
sudo cp -r /etc/yum.repos.d /etc/yum.repos.d.bak
```

### 步骤 2: 清理现有的 YUM 源配置

删除现有的 YUM 源配置文件，或者你可以直接将它们移动到其他目录以进行备份：

```less
sudo rm -f /etc/yum.repos.d/*.repo
```

### 步骤 3: 添加阿里云 YUM 源

创建一个新的 YUM 源配置文件 `/etc/yum.repos.d/CentOS-Base.repo` 并添加以下内容：

```less
[base]
name=CentOS-\$releasever - Base
baseurl=http://mirrors.aliyun.com/centos/\$releasever/os/\$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[updates]
name=CentOS-\$releasever - Updates
baseurl=http://mirrors.aliyun.com/centos/\$releasever/updates/\$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[extras]
name=CentOS-\$releasever - Extras
baseurl=http://mirrors.aliyun.com/centos/\$releasever/extras/\$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

[centosplus]
name=CentOS-\$releasever - Plus
baseurl=http://mirrors.aliyun.com/centos/\$releasever/centosplus/\$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
```

### 步骤 4: 清理缓存并生成新缓存

在更改源之后，清理 YUM 缓存并生成新的缓存：

```less
sudo yum clean all
sudo yum makecache
```

### 步骤 5: 验证配置

检查配置是否正确，可以尝试更新系统或安装软件包：

```less
sudo yum update
```
