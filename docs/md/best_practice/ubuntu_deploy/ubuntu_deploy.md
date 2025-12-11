## 服务器

> Ubuntu 22.04

## 1.安装docker

* 先看docker是否安装

```shell
root@iZ0xifmcfil2jbxvbt2em2Z:~# docker version
 ```

* 按提示安装
> 原生方式安装Docker

```shell
# 更新软件包索引
sudo apt update
# 安装必要工具
sudo apt install -y ca-certificates curl
# 添加Docker官方GPG密钥：
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# 添加Docker官方APT仓库
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
# 安装Docker组件
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 启动Docker服务并设为开机自启，这样服务器重启后docker服务会自动恢复
sudo systemctl start docker
sudo systemctl enable docker

# 创建配置文件目录
sudo mkdir -p /etc/docker

# 创建配置文件/etc/docker/daemon.json，并指定Docker运行目录
{
  "data-root": "/data/docker" 
}

# 验证安装结果
sudo systemctl status docker
```

* 准备Docker容器操作位置

```shell
mkdir /home/docker 
```

## 2.安装Docker-Compose

1. 打开网址：https://github.com/docker/compose/releases/tag/v2.16.0，
   复制下载地址docker-compose-linux-x86_64

2. 执行安装命令

```shell
# 到docker操作目录
cd /home/docker

# 安装 docker-compose
mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# 检查安装成功
docker-compose version

```

## 3.容器化部署

### 部署依赖组件

> 这里以Mysql和RabbitMq为例

* docker-compose-component.yml

   ```yaml
   version: '3.8'
   
   services:
     mysql:
       image: mysql:8.0.24
       container_name: mysql8
       environment:
         MYSQL_ROOT_PASSWORD: ff#*123
         MYSQL_DATABASE: testdb
         MYSQL_USER: admin
         MYSQL_PASSWORD: ff#*123
       ports:
         - "13306:3306"
       volumes:
         - ./robot/mysql/mysql_data:/var/lib/mysql
       restart: unless-stopped
   
     rabbitmq:
       image: rabbitmq:3.13.0
       container_name: rabbitmq
       environment:
         RABBITMQ_DEFAULT_USER: admin
         RABBITMQ_DEFAULT_PASS: admin
       ports:
         - "6672:5672"
         - "16672:15672"
       volumes:
         - ./robot/rabbitmq/rabbitmq_data:/var/lib/rabbitmq
       restart: unless-stopped
  
     redis:
       image: redis:7.0.11
       container_name: redis7
       restart: always
       ports:
         - "6379:6379"
       volumes:
         - ./robot/redis/redis_data:/var/lib/redis
       environment:
         - REDIS_PASSWORD=123456
       command: ["redis-server", "--requirepass", "123456"]
   ```

### 部署应用

* docker-compose.yml

```yaml
version: '3.8'
services:
  fantomfite-admin:
    container_name: your-app
    build:
      context: ./robot/your-app
      dockerfile: dockerfile
    ports:
      - "8888:8888"
    environment:
      - spring.profiles.active=prod
    volumes:
      - ./logs:/home/robot/logs
```

* dockerfile
    * 创建文件夹
    ```shell
      mkdir -p ./robot/your-app

      cd ./robot/your-app
    ```
    * dockerfile
    ```yaml
        FROM 这里填你的JDK镜像地址

        ENV TZ=Asia/Shanghai
        
        # 挂载目录
        VOLUME /home/robot
        # 创建目录
        RUN mkdir -p /home/robot
        # 指定路径
        WORKDIR /home/robot
        # 复制jar文件到路径
        COPY ./jar/your-app.jar /home/robot/your-app.jar
        
        EXPOSE 8888
        
        ENTRYPOINT ["java", "-jar", "you-app.jar", "-c"]
    ```

### 防火墙开放端口

```shell
sudo ufw allow 13306/tcp
sudo ufw allow 6672/tcp
sudo ufw allow 16672/tcp
sudo ufw allow 8888/tcp
```

### 启动服务

* 启动依赖服务

> 一般依赖的镜像不用删

```shell
docker rm -f mysql8;
docker rm -f rabbitmq;
docker rm -f redis7;
docker-compose -f docker-compose-component.yml up -d
```

* 启动应用

```shell
docker rm -f your-app;
docker rmi -f docker-your-app;
docker-compose -f docker-compose.yml up your-app
```

## [4.Nginx配置域名访问](/md/best_practice/nginx_ssl_cors/nginx_ssl_cors.md)