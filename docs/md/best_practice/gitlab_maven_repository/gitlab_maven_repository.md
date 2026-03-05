# 基于 GitLab 搭建私有 Maven 仓库

## 背景

在团队开发中，有些内部工具库或公共组件需要在团队内部共享，但又不适合发布到公共 Maven 仓库。本文介绍如何利用现有的 GitLab 内网仓库，通过 Deploy Token 方式搭建私有 Maven 仓库。

## 版本信息

- GitLab 版本：支持 Package Registry 功能的版本(当前部署版本为: GitLab Community Edition 14.3.6 )
- Maven 版本：3.x

## 前置条件

1. 已有 GitLab 内网仓库（如 `http://192.168.5.66`）
2. GitLab 已启用 Package Registry 功能
3. 已创建 Deploy Token（需要 `read_package_registry` 和 `write_package_registry` 权限）

## 一、创建 Deploy Token

### 1. 进入项目设置

在 GitLab 项目页面，依次点击：`Settings` → `Repository` → `Deploy tokens`

### 2. 创建 Token

填写以下信息：

| 字段 | 说明 |
|------|------|
| Name | Token 名称，如 `maven-deploy` |
| Expiration date | 过期时间（可选） |
| Username | 自动生成，格式为 `gitlab+deploy-token-N` |
| Scopes | 勾选 `read_package_registry` 和 `write_package_registry` |

### 3. 保存 Token

创建后会显示 Token 密码，请妥善保存，后续配置需要使用。

> **注意**：Token 密码只显示一次，请立即保存！

## 二、配置 Maven Settings

在 `~/.m2/settings.xml` 文件中进行以下配置：

### 1. 完整配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0 http://maven.apache.org/xsd/settings-1.2.0.xsd">

  <localRepository>/path/to/local/repo</localRepository>

  <servers>
    <server>
      <id>gitlab-maven</id>
      <username>gitlab+deploy-token-1</username>
      <password>D6uzzxp2sw9e6Tb6nsQo</password>
    </server>
  </servers>

  <mirrors>
    <mirror>
      <id>alimaven</id>
      <name>aliyun maven</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
    <mirror>
      <id>central</id>
      <name>Maven Repository Switchboard</name>
      <url>http://repo1.maven.org/maven2/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
    <mirror>
      <id>repo2</id>
      <mirrorOf>central</mirrorOf>
      <name>Human Readable Name for this Mirror.</name>
      <url>http://repo2.maven.org/maven2/</url>
    </mirror>
    <mirror>
      <id>ibiblio</id>
      <mirrorOf>central</mirrorOf>
      <name>Human Readable Name for this Mirror.</name>
      <url>http://mirrors.ibiblio.org/pub/mirrors/maven2/</url>
    </mirror>
    <mirror>
      <id>jboss-public-repository-group</id>
      <mirrorOf>central</mirrorOf>
      <name>JBoss Public Repository Group</name>
      <url>http://repository.jboss.org/nexus/content/groups/public</url>
    </mirror>
    <mirror>
      <id>maven.net.cn</id>
      <name>oneof the central mirrors in china</name>
      <url>http://maven.net.cn/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
    <mirror>
      <id>local-gitlab</id>
      <name>Allow HTTP for GitLab Maven Repository</name>
      <mirrorOf>gitlab-maven</mirrorOf>
      <url>http://192.168.5.66/api/v4/projects/146/packages/maven</url>
    </mirror>
  </mirrors>

  <profiles>
    <profile>
      <id>gitlab-deploy</id>
      <properties>
        <altReleaseDeploymentRepository>
          gitlab-maven::default::http://192.168.5.66/api/v4/projects/146/packages/maven
        </altReleaseDeploymentRepository>
        <altSnapshotDeploymentRepository>
          gitlab-maven::default::http://192.168.5.66/api/v4/projects/146/packages/maven
        </altSnapshotDeploymentRepository>
      </properties>
      <repositories>
        <repository>
          <id>central</id>
          <url>https://maven.aliyun.com/nexus/content/groups/public</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>false</enabled></snapshots>
        </repository>
        <repository>
          <id>snapshots</id>
          <url>https://maven.aliyun.com/nexus/content/groups/public</url>
          <releases><enabled>false</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </repository>
        <repository>
          <id>gitlab-maven</id>
          <url>http://192.168.5.66/api/v4/projects/146/packages/maven</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <id>central</id>
          <url>https://maven.aliyun.com/nexus/content/groups/public</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>false</enabled></snapshots>
        </pluginRepository>
        <pluginRepository>
          <id>snapshots</id>
          <url>https://maven.aliyun.com/nexus/content/groups/public</url>
          <releases><enabled>false</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>

  <activeProfiles>
    <activeProfile>gitlab-deploy</activeProfile>
  </activeProfiles>
</settings>
```

### 2. 配置说明

#### Servers 配置

```xml
<server>
  <id>gitlab-maven</id>
  <username>gitlab+deploy-token-1</username>
  <password>D6uzzxp2sw9e6Tb6nsQo</password>
</server>
```

| 字段 | 说明 |
|------|------|
| id | 服务器标识，需与 repository 的 id 保持一致 |
| username | Deploy Token 的用户名 |
| password | Deploy Token 的密码 |

#### Mirrors 配置

```xml
<mirror>
  <id>local-gitlab</id>
  <name>Allow HTTP for GitLab Maven Repository</name>
  <mirrorOf>gitlab-maven</mirrorOf>
  <url>http://192.168.5.66/api/v4/projects/146/packages/maven</url>
</mirror>
```

URL 格式说明：

```
http://<gitlab-host>/api/v4/projects/<project-id>/packages/maven
```

- `<gitlab-host>`：GitLab 服务器地址
- `<project-id>`：项目 ID，可在项目首页找到

#### Profile 配置

```xml
<properties>
  <altReleaseDeploymentRepository>
    gitlab-maven::default::http://192.168.5.66/api/v4/projects/146/packages/maven
  </altReleaseDeploymentRepository>
  <altSnapshotDeploymentRepository>
    gitlab-maven::default::http://192.168.5.66/api/v4/projects/146/packages/maven
  </altSnapshotDeploymentRepository>
</properties>
```

用于配置发布仓库地址，格式为：`<id>::<layout>::<url>`

## 三、项目配置

### 1. pom.xml 配置

在需要发布的项目的 `pom.xml` 中添加：

```xml
<project>
  <!-- 其他配置 -->

  <distributionManagement>
    <repository>
      <id>gitlab-maven</id>
      <url>http://192.168.5.66/api/v4/projects/146/packages/maven</url>
    </repository>
    <snapshotRepository>
      <id>gitlab-maven</id>
      <url>http://192.168.5.66/api/v4/projects/146/packages/maven</url>
    </snapshotRepository>
  </distributionManagement>
</project>
```

> **注意**：`<id>` 必须与 `settings.xml` 中 `<server>` 的 id 一致，否则认证会失败。

### 2. 获取项目 ID

项目 ID 可以在 GitLab 项目首页的顶部找到，或者通过 API 获取：

```bash
curl -s "http://192.168.5.66/api/v4/projects?search=项目名称" | jq '.[0].id'
```

## 四、发布与使用

### 1. 发布包到仓库

```bash
mvn deploy
```

发布成功后，可以在 GitLab 项目页面的 `Deploy` → `Package Registry` 中查看发布的包。

### 2. 使用私有仓库中的包

在其他项目的 `pom.xml` 中添加依赖：

```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>your-artifact</artifactId>
  <version>1.0.0</version>
</dependency>
```

确保 `settings.xml` 中已配置好 `gitlab-maven` 仓库，Maven 会自动从私有仓库下载依赖。

## 五、常见问题

### 1. HTTP 协议问题

如果 GitLab 使用 HTTP 协议，可能会遇到以下错误：

```
Return code is: 501, ReasonPhrase: HTTPS Required
```

**解决方案**：在 `settings.xml` 中添加镜像配置，允许 HTTP：

```xml
<mirror>
  <id>local-gitlab</id>
  <name>Allow HTTP for GitLab Maven Repository</name>
  <mirrorOf>gitlab-maven</mirrorOf>
  <url>http://192.168.5.66/api/v4/projects/146/packages/maven</url>
</mirror>
```

### 2. 认证失败

检查以下几点：

1. `<server>` 的 id 是否与 `<repository>` 的 id 一致
2. Deploy Token 是否有正确的权限
3. Token 是否已过期

### 3. 包无法下载

确认 `settings.xml` 中的 profile 已激活：

```xml
<activeProfiles>
  <activeProfile>gitlab-deploy</activeProfile>
</activeProfiles>
```

### 4. 查看依赖来源

使用以下命令查看依赖解析过程：

```bash
mvn dependency:tree -X
```

## 六、最佳实践

### 1. 版本管理

- **Release 版本**：正式发布版本，如 `1.0.0`
- **Snapshot 版本**：开发版本，如 `1.0.0-SNAPSHOT`

### 2. Token 管理

- 为不同环境创建不同的 Deploy Token
- 定期轮换 Token 密码
- 不要将 Token 提交到代码仓库

### 3. 权限控制

- 只授予必要的权限
- 考虑使用 Group Deploy Token 管理多个项目

### 4. 缓存优化

配置本地仓库路径，避免重复下载：

```xml
<localRepository>/path/to/local/repo</localRepository>
```

## 七、总结

通过 GitLab 的 Package Registry 功能，可以快速搭建私有 Maven 仓库，实现团队内部依赖的共享和管理。相比搭建独立的 Nexus 或 Artifactory，这种方式：

- 无需额外部署服务
- 与 GitLab 权限体系集成
- 支持版本管理和追溯
- 配置简单，维护成本低

## 参考资料

- [GitLab Package Registry 文档](https://docs.gitlab.com/ee/user/packages/package_registry/)
- [Maven Settings 配置指南](https://maven.apache.org/settings.html)
- [Maven Deploy 插件文档](https://maven.apache.org/plugins/maven-deploy-plugin/)
