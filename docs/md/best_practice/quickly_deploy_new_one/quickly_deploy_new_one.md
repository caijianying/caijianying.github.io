## 说在前面

> 当你已经搭建了一套开发环境时，如果要快速地搭建一套新的开发环境，你该如何应对？
> 
> 首先最好是**相似架构**的服务器，比如 AlmaLinux 镜像就是Centos停止维护后火起来的，算是centos的近亲吧。

## 背景
已有一套开发环境部署到 CentOS 上，现在需要在 AlmaLinux 上再部署一套，由于是开发环境所以相关组件其实可以在同一个实例上共用，比如同一个Redis可以用dbIndex隔离开，同一个数据库实例可以创建不同数据库隔离等，当然哪些需要隔离还得看具体情况。

> 为了方便理解，下面叙述将CentOS服务器成为"老服务器", AlmaLinux服务器为"目标服务器"

## 在 老服务器 安装Docker

```shell
## 更新架构所需依赖
dnf update -y
## 添加适合当前架构的镜像
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
## 安装Docker时移除冲突的Podman包
dnf install docker-ce --allowerasing
## 安装Docker引擎及相关组件
dnf install docker-ce docker-ce-cli containerd.io
## 启动Docker服务并设为开机自启
sudo systemctl start docker
sudo systemctl enable docker
## 检查Docker版本
docker --version
```

## 迁移Docker配置文件和项目包

>你还需要将项目打包到本地重新传到新服务器吗？当然不是，服务器之间进行远程传输即可。

* 在 老服务器 上进行资源打包
```shell
## 项目打包 - 将指定项目所在文件夹进行打包
zip -r proj.zip ./proj
## 在 老服务器 上远程传输到 目标服务器
scp /home/docker/proj.zip root@目标IP:/home/docker/
## 接下来会提示输入 目标服务器 的登录密码。输入完后，文件会出现在 目标服务器 /home/docker/
```


* 在目标服务器解压即可快速部署

```shell
# 进入docker操作目录
cd /home/docker 
# 解压刚刚传上来的zip
unzip proj.zip
......
```
这里一般解压得到的就是 docker-compose配置文件和容器部署安装包，后面安装就不赘述了。

