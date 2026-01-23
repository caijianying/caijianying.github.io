# 搭建Grafana Dashboard 最佳实践指南

## 版本信息
- 服务器系统：AlmaLinux 或 CentOS
- Spring Boot 版本：2.7.0
- Prometheus 版本：2.32.1
- Grafana 版本：12.3.0

## Spring Boot 接入 Actuator 与 Prometheus

### 1. Maven 依赖配置
在 `pom.xml` 文件中添加以下依赖：

```xml
<properties>
    <spring.version>5.3.22</spring.version>
    <springboot.version>2.7.0</springboot.version>
    <prometheus.version>1.15.5</prometheus.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>${springboot.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
        <version>${springboot.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
        <version>${springboot.version}</version>
    </dependency>
    <!-- Micrometer Prometheus Registry -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
        <version>${prometheus.version}</version>
    </dependency>
</dependencies>
```

### 2. 应用配置
在 `application.yml` 文件中配置 Actuator 和 Prometheus：

```yaml
management:
  endpoints:
    web:
      base-path: /actuator
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
  endpoint:
    prometheus:
      enabled: true
    metrics:
      enabled: true
```

## 部署架构

### 1. 目录结构
```
├── docker-compose.yml          # 容器编排配置
├── grafana/                    # Grafana 配置目录
│   └── provisioning/           # 预配置目录
├── logs/                       # 日志目录
├── prometheus/                 # Prometheus 配置目录
│   └── prometheus.yml          # Prometheus 配置文件
├── shared-data/                # 共享数据目录
└── robot/                      # 应用目录
    └── demo-app/               # 示例应用
        ├── dockerfile          # Docker 构建文件
        └── jar/                # 应用 jar 包
            └── demo-app.jar    # 示例应用 jar 包
```

### 2. Docker 镜像构建
创建 `Dockerfile` 文件：

```dockerfile
FROM openjdk:11-alpine

ENV TZ=Asia/Shanghai

# 挂载目录
VOLUME /home/robot
# 创建目录
RUN mkdir -p /home/robot
# 指定工作目录
WORKDIR /home/robot
# 复制 jar 文件到工作目录
COPY ./jar/demo-app.jar /home/robot/demo-app.jar

EXPOSE 8899

ENTRYPOINT ["java", "-XX:+UseG1GC", "-jar", "demo-app.jar", "-c"]
```

### 3. 容器编排配置
创建 `docker-compose.yml` 文件：

```yaml
version: '3.8'
services:
  # 1. 示例应用服务
  demo-app:
    container_name: demo-app
    build:
      context: ./robot/demo-app
      dockerfile: dockerfile
    ports:
      - "9999:8899"
    volumes:
      - ./logs:/home/robot/logs
      - ./shared-data:/home/robot/shared-data
    networks:
      - monitoring-net

  # 2. Prometheus 监控服务
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml # 挂载配置文件
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    networks:
      - monitoring-net
    depends_on:
      - demo-app

  # 3. Grafana 可视化服务
  grafana:
    image: grafana/grafana:12.3.0
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin # 设置初始密码
      - GF_LANG="zh-CN" # 设置中文语言
    volumes:
      - ./grafana:/etc/grafana # 持久化 Grafana 配置
    networks:
      - monitoring-net
    depends_on:
      - prometheus

# 定义内部网络，确保服务间可通过服务名通信
networks:
  monitoring-net:
    driver: bridge
```

## 部署与运行

### 1. 使用离线镜像包
如果需要在无网络环境下部署，可以使用离线镜像包：

> 资源路径：/assets/docker-images

```bash
# 加载 Grafana 离线镜像
docker load -i grafana.tar

# 加载 Prometheus 离线镜像
docker load -i prometheus.tar
```

### 2. 启动服务
执行以下命令启动所有服务：

```bash
docker compose -f docker-compose.yml up -d demo-app prometheus grafana
```

### 3. 访问地址
- 示例应用：`http://服务器IP:9999`
- Prometheus：`http://服务器IP:9090`
- Grafana：`http://服务器IP:3000`（默认用户名/密码：admin/admin）

## 后续配置

1. **配置 Prometheus 数据源**：
   - 登录 Grafana 后，进入 "配置" > "数据源"
   - 添加 Prometheus 数据源，URL 填写 `http://prometheus:9090`

2. **导入仪表板**：
   - 进入 "仪表板" > "导入"
   - 可以导入官方提供的 Spring Boot 仪表板模板（例如 ID: 12856）

3. **查看监控指标**：
   - 在 Grafana 仪表板中查看应用的各项监控指标，包括 JVM 状态、请求量、响应时间等
4. **Grafana 持久化**：
   1. 首次启动 要注释
    ```yaml
        volumes:
        - ./grafana:/etc/grafana 
    ```
   2. 启动完配置仪表板后执行，`docker cp grafana:/etc/grafana ./grafana` 持久化配置
   3. 然后重新启动容器，即可持久化配置
    
