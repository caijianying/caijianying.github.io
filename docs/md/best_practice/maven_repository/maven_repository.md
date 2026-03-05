## 前言

在企业内部开发中，经常会遇到以下场景：

- 团队内部有一些公共组件或工具类需要在多个项目间共享
- 由于安全策略，某些依赖包不能发布到公共Maven仓库
- 需要对内部依赖进行版本管理和控制

搭建私有Maven仓库是解决这些问题的有效方案。本文将介绍如何利用GitLab仓库，通过Maven的`deploy`方式快速搭建一个私有Maven仓库。

## 方案概述

> 本方案基于已有的GitLab内网仓库实现

### 为什么选择GitLab + Deploy方式？

| 方案 | 优点 | 缺点 |
|------|------|------|
| Nexus | 功能强大，专业 | 需要额外部署和维护服务 |
| JFrog Artifactory | 企业级功能完善 | 商业版收费，部署复杂 |
| **GitLab + Deploy** | 零额外部署，利用现有资源 | 功能相对简单 |

对于中小型团队，利用现有的GitLab仓库搭建Maven仓库是一种**轻量、低成本**的选择。

### 原理说明

Maven支持将构建产物部署到多种类型的仓库，包括：

- 本地文件系统
- HTTP服务器
- Git仓库（通过特定配置）

本文采用的方式是：将GitLab仓库作为Maven仓库的存储后端，通过`mvn deploy`命令将构建产物推送到GitLab仓库，其他项目则通过配置`pom.xml`从该仓库拉取依赖。

## 搭建步骤

### 1. 创建GitLab仓库

首先，在GitLab上创建一个专门用于存储Maven依赖的仓库：

1. 登录GitLab，创建新项目
2. 项目命名建议：`maven-repository` 或 `internal-maven-repo`
3. 可见性级别根据需求选择：
   - **Private**：仅团队成员可访问
   - **Internal**：所有登录用户可访问
   - **Public**：所有人可访问（不推荐内网环境）

### 2. 配置Maven认证

> 由于GitLab仓库需要认证，需要在Maven的`settings.xml`中配置认证信息

编辑Maven配置文件 `~/.m2/settings.xml`（如不存在则创建）：

```xml
<settings>
    <servers>
        <server>
            <id>gitlab-maven</id>
            <username>your-gitlab-username</username>
            <password>your-personal-access-token</password>
        </server>
    </servers>
</settings>
```

> **注意**：建议使用Personal Access Token代替密码，更安全且便于管理
> 
> 在GitLab中生成Token：Settings -> Access Tokens -> Add new token

### 3. 配置项目pom.xml

在需要发布的项目`pom.xml`中添加以下配置：

```xml
<project>
    <!-- 其他配置 -->

    <distributionManagement>
        <repository>
            <id>gitlab-maven</id>
            <url>https://your-gitlab-domain.com/api/v4/projects/{project-id}/packages/maven</url>
        </repository>
        <snapshotRepository>
            <id>gitlab-maven</id>
            <url>https://your-gitlab-domain.com/api/v4/projects/{project-id}/packages/maven</url>
        </snapshotRepository>
    </distributionManagement>
</project>
```

> **获取Project ID**：在GitLab项目首页，项目名称下方会显示项目ID

### 4. 发布依赖

执行Maven deploy命令：

```shell
mvn clean deploy
```

如果只想跳过测试发布：

```shell
mvn clean deploy -DskipTests
```

发布成功后，可以在GitLab项目的 **Deploy -> Package Registry** 中看到发布的包。

### 5. 配置消费端

在其他需要使用该依赖的项目中，配置`pom.xml`：

```xml
<project>
    <!-- 添加仓库配置 -->
    <repositories>
        <repository>
            <id>gitlab-maven</id>
            <url>https://your-gitlab-domain.com/api/v4/projects/{project-id}/packages/maven</url>
        </repository>
    </repositories>

    <!-- 添加依赖 -->
    <dependencies>
        <dependency>
            <groupId>com.yourcompany</groupId>
            <artifactId>your-common-lib</artifactId>
            <version>1.0.0</version>
        </dependency>
    </dependencies>
</project>
```

同时确保消费端项目的`~/.m2/settings.xml`中也配置了相同的认证信息。

## 进阶配置

### 版本管理策略

建议采用以下版本命名规范：

```
主版本.次版本.修订版本-类型

例如：
1.0.0          正式版本
1.0.0-SNAPSHOT 快照版本（开发中）
1.0.0-RC1      候选发布版本
```

**快照版本**的特点：

- Maven会定期检查更新（默认每天）
- 相同版本号可以重复发布
- 适合开发阶段频繁更新

**正式版本**的特点：

- 发布后不可覆盖
- 保证版本稳定性
- 适合生产环境使用

### 多模块项目配置

对于多模块项目，可以在父POM中统一配置：

```xml
<project>
    <groupId>com.yourcompany</groupId>
    <artifactId>parent-project</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>common-core</module>
        <module>common-utils</module>
        <module>common-security</module>
    </modules>

    <distributionManagement>
        <repository>
            <id>gitlab-maven</id>
            <url>https://your-gitlab-domain.com/api/v4/projects/{project-id}/packages/maven</url>
        </repository>
    </distributionManagement>
</project>
```

### 配置镜像加速

如果团队内部有多个仓库，可以配置镜像统一管理：

```xml
<settings>
    <mirrors>
        <mirror>
            <id>internal-mirror</id>
            <mirrorOf>*,!gitlab-maven</mirrorOf>
            <name>Internal Repository Mirror</name>
            <url>https://your-mirror-domain.com/repository/maven-public/</url>
        </mirror>
    </mirrors>
</settings>
```

> `!gitlab-maven` 表示排除该仓库，直接访问原始地址

## 常见问题

### 1. 发布失败：401 Unauthorized

**原因**：认证信息配置错误或Token过期

**解决**：
- 检查`settings.xml`中的用户名和Token是否正确
- 确认Token权限包含`api`或`write_package_registry`
- 确认`pom.xml`中的`repository.id`与`settings.xml`中的`server.id`一致

### 2. 拉取依赖失败

**原因**：消费端未配置认证或仓库地址错误

**解决**：
- 确保消费端`settings.xml`配置了认证信息
- 检查仓库URL是否正确，特别是Project ID
- 尝试清理本地缓存：`mvn dependency:purge-local-repository`

### 3. 快照版本未更新

**原因**：Maven本地缓存未刷新

**解决**：
```shell
mvn clean install -U
```
`-U` 参数强制更新快照版本

### 4. 发布时覆盖正式版本

**原因**：GitLab默认允许覆盖（取决于配置）

**解决**：
- 在GitLab项目设置中启用包保护
- Settings -> CI/CD -> Package Registry -> Protected packages

## 最佳实践建议

1. **权限管理**
   - 发布权限仅授予CI/CD服务账号或核心开发者
   - 普通开发者只需读取权限

2. **版本策略**
   - 开发阶段使用SNAPSHOT版本
   - 发布前升级为正式版本
   - 重要版本打Tag标记

3. **CI/CD集成**
   - 将发布流程集成到GitLab CI/CD
   - 自动化测试通过后自动发布

4. **文档维护**
   - 维护依赖变更日志
   - 记录每个版本的重要变更

## 总结

通过GitLab + Maven Deploy方式搭建私有仓库，具有以下优势：

- **零额外成本**：利用现有GitLab资源
- **配置简单**：只需配置pom.xml和settings.xml
- **权限可控**：复用GitLab的权限管理体系
- **版本追溯**：GitLab提供包管理界面

对于中小型团队，这是一种轻量、高效的私有Maven仓库解决方案。
