### docker安装过程

```bash
# 卸载docker和相关依赖，docker-compose-plugin如果没有安装可以不加
sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo apt-get remove docker docker-engine docker-ce docker.io

# 检查是否还有依赖
dpkg -l "*docker*" 

# 更新apt
sudo apt-get update

# 安装以下包以使apt可以通过HTTPS使用存储库（repository）
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common

# 添加Docker官方的GPG密钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# 使用下面的命令来设置stable存储库
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu (lsb_release -cs) stable"

# 更新apt
sudo apt-get update

#docker版本列表
apt-cache madison docker-ce

# 安装最新版docker
sudo apt-get install -y docker-ce
# 安装指定版本docker
sudo apt-get install -y docker-ce=17.12.0~ce-0~ubuntu

# 检查
docker -v 

# 免sudo设置（重启系统后生效）
sudo usermod -aG docker $USER


```

### docker-compose安装过程

```bash
# 下载二进制文件到bin目录
curl -L https://get.daocloud.io/docker/compose/releases/download/v2.6.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

# 授权
sudo chmod +x /usr/local/bin/docker-compose

# 检查
docker-compose -v


```