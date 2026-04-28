# Jenkins 部署 Java 项目实战：Podman 容器版与宿主机版双方案手操指南

## 1. 前言：本文解决什么问题

本文记录一套可以反复复用的 Jenkins 部署 Java 项目流程：Jenkins 拉取本地 GitLab 代码，使用 Maven 构建 Java 项目，再通过 SSH 把 JAR 包部署到目标服务器上的 Docker 环境，最后把构建结果发送到飞书群。

这不是 Jenkins 概念大全，也不重复展开 GitLab 的搭建过程。GitLab 私有仓库和 Maven 仓库的搭建可以参考已有文章：[基于 GitLab 搭建私有 Maven 仓库](/md/best_practice/gitlab_maven_repository/gitlab_maven_repository.md)。

本文重点是两种真实可落地的 Jenkins 使用方式：

1. 使用 Podman 启动一个隔离的 Jenkins 容器。
2. 在已经安装好的宿主机 Jenkins 上补齐 Java 构建部署能力。

两种方案都从环境准备讲到完整部署验证，读者可以根据自己的机器状态选择其中一种执行。

## 2. 通用前置条件

两种方案都需要提前准备好以下资源：

| 资源 | 说明 |
| --- | --- |
| Jenkins 服务器 | 用来运行 Jenkins，可以是全新机器，也可以是已有 Jenkins 的机器。 |
| 部署目标服务器 | Java 服务最终运行的机器，可以和 Jenkins 同机，也可以是其他服务器。 |
| 本地 GitLab 仓库 | 本文只说明 Jenkins 如何连接已有 GitLab，不重复讲 GitLab 搭建。 |
| Java 项目 | 项目可以通过 Maven 构建，并能生成可部署的 JAR 包。 |
| JDK | 示例使用 JDK 11 作为项目构建版本。 |
| Maven | 示例使用 Maven 3.8.1。 |
| GitLab 凭据 | 可以使用 GitLab Personal Access Token，也可以使用账号密码。 |
| SSH 私钥 | Jenkins 通过 SSH 登录目标服务器执行部署脚本。 |
| 飞书机器人 Webhook | 用来发送构建成功或失败通知。 |
| 飞书机器人关键词 | 如果机器人开启关键词安全设置，通知内容必须命中关键词。 |
| Docker 或 Podman | 目标服务器负责运行应用容器。 |
| Docker Compose 命令 | 不同服务器可能是 `docker compose`，也可能是 `docker-compose`。 |

本文命令和脚本中的敏感信息都使用占位符表示，实际操作时替换为自己的值：

| 占位符 | 含义 |
| --- | --- |
| `<GITLAB_HOST>` | GitLab 访问地址，例如 `http://192.168.1.10`。 |
| `<PROJECT_REPO_URL>` | Java 项目的 Git 仓库地址。 |
| `<JENKINS_SERVER_IP>` | Jenkins 服务器 IP。 |
| `<DEPLOY_SERVER_IP>` | 部署目标服务器 IP。 |
| `<PROJECT_NAME>` | 项目名或 Docker Compose 服务名。 |
| `<SERVICE_NAME>` | 飞书通知中展示的服务名称。 |
| `<FEISHU_WEBHOOK>` | 飞书机器人 Webhook 地址。 |
| `<GITLAB_CREDENTIALS_ID>` | Jenkins 中保存的 GitLab 凭据 ID。 |
| `<DEPLOYMENT_SSH_KEY_ID>` | Jenkins 中保存的 SSH 私钥凭据 ID。 |

后文示例默认使用这些值：

```text
JDK 路径：/opt/tools/jdk11
Maven 路径：/opt/tools/maven
Maven 版本：3.8.1
JDK 版本：11
GitLab 凭据 ID：gitlab-credentials
SSH 凭据 ID：deployment-ssh-key
```

> **注意**
> 飞书机器人如果配置了关键词安全策略，即使 Webhook 接口返回成功，只要消息内容没有包含关键词，群里也可能收不到消息。本文示例统一让通知标题包含"服务器部署"，可以把飞书机器人关键词也配置为"服务器部署"。

## 3. 方案一：Podman 容器版 Jenkins 全流程

### 3.1 适用场景

如果你是在一台新机器上搭 Jenkins，或者希望 Jenkins 的数据、工具链和宿主机尽量隔离，优先使用 Podman 容器版。

这种方式的特点是：

- Jenkins 数据放在挂载目录里，迁移和备份方便。
- JDK、Maven、部署脚本都放在容器或挂载目录中，不污染宿主机环境。
- Jenkins Web 端口可以和宿主机已有服务错开，例如使用 `8180:8080`。

### 3.2 目录规划

示例目录如下：

```text
/home/podman/jenkins/jenkins-deployment/
├── jenkins-compose.yml
├── deploy-jenkins-5.66.sh
├── jenkins_home/
├── deployment-scripts/
│   ├── deploy-to-docker.sh
│   └── send-feishu-notification.sh
└── enviroment/
    ├── openjdk-11+28_linux-x64_bin.tar.gz
    ├── apache-maven-3.8.1-bin.tar
    ├── settings-5.66.xml
    └── merach.zip
```

容器内关键路径：

```text
Jenkins Home：/var/jenkins_home
JDK：/opt/tools/jdk11
Maven：/opt/tools/maven
Maven settings：/opt/tools/maven/conf/settings.xml
部署脚本：/deployment-scripts/deploy-to-docker.sh
飞书通知脚本：/deployment-scripts/send-feishu-notification.sh
```

### 3.3 使用 Podman 启动 Jenkins

`jenkins-compose.yml` 示例：

```yaml
version: '3'

services:
  jenkins:
    image: docker.io/jenkins/jenkins:2.555.1-lts
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "8180:8080"
      - "51000:50000"
    volumes:
      - ./jenkins_home:/var/jenkins_home:Z
      - ./deployment-scripts:/deployment-scripts:Z
    environment:
      - TZ=Asia/Shanghai
    user: root
    privileged: true
```

启动 Jenkins：

```bash
cd /home/podman/jenkins/jenkins-deployment
mkdir -p jenkins_home deployment-scripts
chmod 777 jenkins_home
podman-compose -f jenkins-compose.yml up -d
```

查看容器状态：

```bash
podman ps | grep jenkins
podman logs -f jenkins
```

获取初始管理员密码：

```bash
podman exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

浏览器访问：

```text
http://<JENKINS_SERVER_IP>:8180
```

### 3.4 Jenkins 初始化

第一次打开 Jenkins 后按页面提示完成初始化：

1. 输入上一节获取的初始管理员密码。
2. 选择 `Install suggested plugins` 安装推荐插件。
3. 创建管理员账号。
4. 实例地址填写 `http://<JENKINS_SERVER_IP>:8180/`。
5. 进入 Jenkins 首页后安装额外插件。

建议额外安装这些插件：

- GitLab Plugin
- SSH Agent Plugin
- Publish Over SSH Plugin
- Maven Integration Plugin
- Docker Pipeline Plugin

### 3.5 初始化构建环境

把 JDK、Maven 和 Maven settings 放到 `enviroment/` 目录后，可以用脚本复制进容器：

```bash
cd /home/podman/jenkins/jenkins-deployment

podman exec jenkins mkdir -p /opt/tools
podman cp ./enviroment/openjdk-11+28_linux-x64_bin.tar.gz jenkins:/opt/tools/jdk11.tar
podman exec jenkins bash -c 'cd /opt/tools && tar -xf jdk11.tar && rm jdk11.tar && mv jdk-11* jdk11'

podman cp ./enviroment/apache-maven-3.8.1-bin.tar jenkins:/opt/tools/maven.tar
podman exec jenkins bash -c 'cd /opt/tools && tar -xf maven.tar && rm maven.tar && mv apache-maven-* maven'

podman cp ./enviroment/settings-5.66.xml jenkins:/opt/tools/maven/conf/settings.xml
podman exec jenkins mkdir -p /root/.m2 /var/jenkins_home/.m2
podman cp ./enviroment/settings-5.66.xml jenkins:/root/.m2/settings.xml
podman cp ./enviroment/settings-5.66.xml jenkins:/var/jenkins_home/.m2/settings.xml
podman exec jenkins chown -R jenkins:jenkins /var/jenkins_home/.m2
```

> **关于资源包的准备与传输**
> 为了方便离线部署，建议提前准备一个包含 JDK、Maven、settings.xml 和已下载依赖的完整资源包。由于资源包中包含 Maven 本地仓库的依赖文件，体积可能较大（几百 MB 到几 GB）。对于内网环境，推荐通过 SMB 文件服务器进行传输，速度快且稳定。具体搭建方法可以参考：[快速搭建一台局域网资源共享服务器](/md/best_practice/smb_deploy/smb_deploy.md)。

配置容器内环境变量：

```bash
podman exec jenkins bash -c "cat >> /etc/profile << 'EOF'
export JAVA_HOME=/opt/tools/jdk11
export MAVEN_HOME=/opt/tools/maven
export PATH=\$JAVA_HOME/bin:\$MAVEN_HOME/bin:\$PATH
EOF"
```

验证：

```bash
podman exec jenkins bash -c 'source /etc/profile && java -version'
podman exec jenkins bash -c 'source /etc/profile && mvn -version'
podman exec jenkins grep localRepository /opt/tools/maven/conf/settings.xml
```

在 Jenkins 页面配置全局工具：

```text
Manage Jenkins → Global Tool Configuration

JDK:
Name: jdk11
Install automatically: 取消勾选
JAVA_HOME: /opt/tools/jdk11

Maven:
Name: maven3.8.1
Install automatically: 取消勾选
MAVEN_HOME: /opt/tools/maven
```

### 3.6 导入私有依赖

如果项目依赖未发布到 Maven 私服，可以先把私有依赖导入 Jenkins 使用的本地仓库。例如项目中已有 `merach.zip`：

```bash
podman exec jenkins mkdir -p /root/.m2/repository/com
podman cp ./enviroment/merach.zip jenkins:/tmp/merach.zip
podman exec jenkins bash -c 'cd /root/.m2/repository/com && unzip -o /tmp/merach.zip && rm /tmp/merach.zip'
podman exec jenkins ls -la /root/.m2/repository/com/merach/
```

如果 `settings.xml` 中的 `localRepository` 指向 `/var/jenkins_home/.m2/repository`，则把依赖导入该目录：

```bash
podman exec jenkins mkdir -p /var/jenkins_home/.m2/repository/com
podman cp ./enviroment/merach.zip jenkins:/tmp/merach.zip
podman exec jenkins bash -c 'cd /var/jenkins_home/.m2/repository/com && unzip -o /tmp/merach.zip && rm /tmp/merach.zip'
podman exec jenkins chown -R jenkins:jenkins /var/jenkins_home/.m2
```

### 3.7 配置 GitLab 凭据

在 GitLab 中创建 Personal Access Token：

```text
GitLab → User Settings → Access Tokens
Token name: Jenkins CI
Scopes: read_repository
```

在 Jenkins 中新增凭据：

```text
Manage Jenkins → Credentials → System → Global credentials → Add Credentials

Kind: Username with password
Username: <GitLab 用户名>
Password: <GitLab Personal Access Token>
ID: gitlab-credentials
Description: GitLab Personal Access Token for Jenkins
```

Pipeline 中通过 `credentialsId: 'gitlab-credentials'` 引用。

### 3.8 配置 SSH 私钥凭据

先确认 Jenkins 服务器可以免密连接目标服务器：

```bash
ssh-keygen -t rsa -b 4096 -C "jenkins-deployment"
ssh-copy-id root@<DEPLOY_SERVER_IP>
ssh root@<DEPLOY_SERVER_IP> "echo SSH OK"
```

在 Jenkins 中新增 SSH 凭据：

```text
Manage Jenkins → Credentials → System → Global credentials → Add Credentials

Kind: SSH Username with private key
Username: root
Private Key: Enter directly，粘贴 Jenkins 服务器上的私钥内容
ID: deployment-ssh-key
Description: SSH key for deployment
```

Pipeline 中通过 `sshagent(['deployment-ssh-key'])` 引用。

### 3.9 准备部署脚本

把部署脚本复制到挂载目录：

```bash
cd /home/podman/jenkins/jenkins-deployment
cp /path/to/fantomfite-web-admin/docs/jenkins/scripts/deploy-to-docker.sh deployment-scripts/
cp /path/to/fantomfite-web-admin/docs/jenkins/scripts/send-feishu-notification.sh deployment-scripts/
chmod +x deployment-scripts/*.sh
podman exec jenkins ls -l /deployment-scripts/
```

部署脚本的核心流程是：

1. 检查 Jenkins 构建出的 JAR 文件是否存在。
2. 在目标服务器创建 `${DOCKER_DIR}/robot/${APP_NAME}/jar`。
3. 备份旧 JAR。
4. 使用 `scp` 上传新 JAR。
5. 删除旧容器和旧镜像。
6. 使用 Docker Compose 重新构建并启动服务。
7. 检查容器是否启动成功。

目标服务器上的 Compose 命令需要兼容两种写法：

```bash
if docker compose version &>/dev/null; then
    DOCKER_COMPOSE_CMD="docker compose"
    echo "使用命令: docker compose"
elif docker-compose --version &>/dev/null; then
    DOCKER_COMPOSE_CMD="docker-compose"
    echo "使用命令: docker-compose"
else
    echo "错误: 未找到 docker compose 或 docker-compose 命令"
    exit 1
fi

${DOCKER_COMPOSE_CMD} -f docker-compose.yml up -d <PROJECT_NAME>
```

在目标服务器提前验证：

```bash
ssh root@<DEPLOY_SERVER_IP> 'docker compose version || docker-compose --version'
```

### 3.10 配置 Pipeline

新建 Pipeline 任务后，使用下面的参数化脚本。按自己的环境替换占位符：

```groovy
pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'test'], description: '选择部署环境')
        choice(name: 'BRANCH', choices: ['dev', 'main', 'test'], description: '选择部署分支')
    }

    environment {
        PROJECT_NAME = '<PROJECT_NAME>'
        SERVICE_NAME = '<SERVICE_NAME>'
        GITLAB_URL = '<PROJECT_REPO_URL>'
        DEV_SERVER = '<DEPLOY_SERVER_IP>'
        TEST_SERVER = '<DEPLOY_SERVER_IP>'
        DEV_DOCKER_DIR = '/home/docker'
        TEST_DOCKER_DIR = '/home/web/docker'
        JENKINS_BASE_URL = 'http://<JENKINS_SERVER_IP>:8180'
        MAVEN_OPTS = '-Dmaven.test.skip=true'
        FEISHU_WEBHOOK = '<FEISHU_WEBHOOK>'
    }

    tools {
        maven 'maven3.8.1'
        jdk 'jdk11'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: params.BRANCH,
                    credentialsId: 'gitlab-credentials',
                    url: "${GITLAB_URL}"
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package ${MAVEN_OPTS}'
                sh '''
                    JAR_FILES=$(find . -name "*-web.jar" -path "*/target/*" ! -name "*-sources.jar" ! -name "*-javadoc.jar" || true)
                    if [ -z "$JAR_FILES" ]; then
                        echo "未找到以 -web.jar 结尾的 JAR 文件"
                        find . -name "*.jar" -path "*/target/*" -exec ls -lh {} \; || true
                        exit 1
                    fi
                    echo "$JAR_FILES"
                '''
            }
        }
```

> **关于 JAR 包自动寻包机制**
> 本文示例使用 `find . -name "*-web.jar"` 自动查找以 `-web.jar` 结尾的 JAR 文件作为部署包。这是根据项目框架约定的命名规则设计的自动寻包机制，可以避免硬编码具体的 JAR 文件名。不同项目可以根据实际命名规范调整匹配规则，例如 `*-application.jar`、`*-boot.jar` 或其他模式。

```groovy

        stage('Deploy') {
            steps {
                script {
                    def targetServer = params.DEPLOY_ENV == 'dev' ? env.DEV_SERVER : env.TEST_SERVER
                    def targetDockerDir = params.DEPLOY_ENV == 'dev' ? env.DEV_DOCKER_DIR : env.TEST_DOCKER_DIR
                    def appName = params.DEPLOY_ENV == 'dev' ? env.PROJECT_NAME : "${env.PROJECT_NAME}-test"

                    sshagent(['deployment-ssh-key']) {
                        sh """
                            set -e
                            JAR_FILE=\$(find . -name "*-web.jar" -path "*/target/*" ! -name "*-sources.jar" ! -name "*-javadoc.jar" | head -1)
                            ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 root@${targetServer} "echo SSH OK"
                            /deployment-scripts/deploy-to-docker.sh ${targetServer} ${appName} \$JAR_FILE ${targetDockerDir}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                def deployEnvName = params.DEPLOY_ENV == 'dev' ? '开发环境' : '测试环境'
                sh """
                    /deployment-scripts/send-feishu-notification.sh \
                        '${env.JOB_NAME}' \
                        '${env.SERVICE_NAME}' \
                        'SUCCESS' \
                        '${deployEnvName}' \
                        '${params.BRANCH}' \
                        '${env.JENKINS_BASE_URL}' \
                        '${env.BUILD_NUMBER}' \
                        '${env.FEISHU_WEBHOOK}'
                """
            }
        }
        failure {
            script {
                def deployEnvName = params.DEPLOY_ENV == 'dev' ? '开发环境' : '测试环境'
                sh """
                    /deployment-scripts/send-feishu-notification.sh \
                        '${env.JOB_NAME}' \
                        '${env.SERVICE_NAME}' \
                        'FAILURE' \
                        '${deployEnvName}' \
                        '${params.BRANCH}' \
                        '${env.JENKINS_BASE_URL}' \
                        '${env.BUILD_NUMBER}' \
                        '${env.FEISHU_WEBHOOK}'
                """
            }
        }
        always {
            cleanWs()
        }
    }
}
```

### 3.11 配置飞书通知

飞书通知建议使用卡片消息，而不是普通文本消息。卡片消息更适合展示项目名、服务名、部署状态、部署环境、分支、部署时间和 Jenkins 日志入口。

飞书机器人需要注意两个细节：

1. 如果开启关键词安全设置，通知标题或内容必须包含关键词。
2. Webhook 不要提交到 Git 仓库，推荐通过 Jenkins 环境变量或凭据注入。

卡片消息的关键字段示例：

```json
{
  "msg_type": "interactive",
  "card": {
    "header": {
      "title": {
        "tag": "plain_text",
        "content": "服务器部署"
      },
      "template": "blue"
    },
    "body": {
      "elements": [
        {
          "tag": "markdown",
          "content": "**部署应用**：<PROJECT_NAME>\\n**服务名称**：<SERVICE_NAME>\\n**部署状态**：成功"
        }
      ]
    }
  }
}
```

如果机器人关键词配置为"服务器部署"，上面的标题可以命中关键词。

### 3.12 完整验证

按下面顺序验证一次完整链路：

1. Jenkins 页面点击 `Build with Parameters`。
2. 选择 `DEPLOY_ENV` 和 `BRANCH`。
3. Console Output 中确认 `Checkout` 成功。
4. Console Output 中确认 Maven 构建成功，并能找到 JAR。
5. Console Output 中确认 SSH 连接目标服务器成功。
6. Console Output 中确认部署脚本输出 `部署成功`。
7. 目标服务器检查容器：

```bash
ssh root@<DEPLOY_SERVER_IP> 'docker ps | grep <PROJECT_NAME>'
```

8. 查看应用日志：

```bash
ssh root@<DEPLOY_SERVER_IP> 'docker logs <PROJECT_NAME> --tail 50'
```

9. 如果服务提供健康检查接口，访问：

```bash
curl http://<DEPLOY_SERVER_IP>:<SERVICE_PORT>/actuator/health
```

10. 飞书群确认收到卡片通知，并且按钮可以跳转 Jenkins 构建日志。

## 4. 方案二：已有宿主机 Jenkins 全流程

### 4.1 适用场景

如果服务器上已经通过 RPM、系统服务或其他方式安装了 Jenkins，就没有必要再启动一个 Jenkins 容器。此时更适合保留现有 Jenkins，只补齐 Java 项目构建部署所需的 JDK、Maven、settings、私有依赖和部署脚本。

这种方式的特点是：

- Jenkins 继续使用宿主机服务，例如 `systemctl start jenkins`。
- Jenkins Home 通常是 `/var/lib/jenkins`。
- 构建工具放到 `/opt/tools`，由 Jenkins 全局工具配置引用。
- Maven 本地仓库需要保证 `jenkins` 用户有权限访问。

### 4.2 检查已有 Jenkins 状态

在 Jenkins 服务器上执行：

```bash
systemctl status jenkins
id jenkins
ls -ld /var/lib/jenkins
ss -tulnp | grep 8080
```

浏览器访问：

```text
http://<JENKINS_SERVER_IP>:8080
```

如果 Jenkins 已经能打开，继续后面的初始化步骤。如果 Jenkins 服务不存在，应先完成 Jenkins 安装，再使用本方案。

### 4.3 初始化 JDK 与 Maven

推荐目录结构：

```text
/opt/tools/
├── jdk11/
├── maven/
└── scripts/
    ├── deploy-to-docker.sh
    └── send-feishu-notification.sh

/var/lib/jenkins/.m2/
├── settings.xml
└── repository/
```

可以用初始化脚本完成环境部署：

```bash
cd /path/to/fantomfite-web-admin/docs/jenkins
sudo ./init-host-jenkins.sh
```

> **关于资源包的准备与传输**
> 宿主机方案同样建议提前准备包含 JDK、Maven、settings.xml 和已下载依赖的完整资源包。由于资源包中包含 Maven 本地仓库的依赖文件，体积可能较大（几百 MB 到几 GB）。对于内网环境，推荐通过 SMB 文件服务器进行传输，速度快且稳定。具体搭建方法可以参考：[快速搭建一台局域网资源共享服务器](/md/best_practice/smb_deploy/smb_deploy.md)。

脚本会完成这些动作：

1. 检查 root 权限、资源文件和 `jenkins` 用户。
2. 创建 `/opt/tools`、`/var/lib/jenkins/.m2` 和 `/root/.m2`。
3. 解压 JDK 11 到 `/opt/tools/jdk11`。
4. 解压 Maven 3.8.1 到 `/opt/tools/maven`。
5. 部署 Maven settings。
6. 导入 `merach` 私有依赖。
7. 复制部署脚本到 `/opt/tools/scripts`。
8. 设置 `/var/lib/jenkins/.m2` 为 `jenkins:jenkins`。
9. 验证 JDK、Maven、settings、依赖和脚本权限。

手动验证：

```bash
/opt/tools/jdk11/bin/java -version
/opt/tools/maven/bin/mvn -version
sudo -u jenkins /opt/tools/jdk11/bin/java -version
sudo -u jenkins /opt/tools/maven/bin/mvn -version
```

Jenkins 全局工具配置：

```text
Manage Jenkins → Global Tool Configuration

JDK:
Name: jdk11
Install automatically: 取消勾选
JAVA_HOME: /opt/tools/jdk11

Maven:
Name: maven3.8.1
Install automatically: 取消勾选
MAVEN_HOME: /opt/tools/maven
```

### 4.4 配置 Maven settings 与本地仓库

宿主机 Jenkins 最容易出问题的是 Maven 本地仓库权限。推荐让 Jenkins 用户使用自己的仓库：

```text
/var/lib/jenkins/.m2/repository
```

`settings.xml` 中应包含：

```xml
<localRepository>/var/lib/jenkins/.m2/repository</localRepository>
```

验证：

```bash
grep localRepository /opt/tools/maven/conf/settings.xml
grep localRepository /var/lib/jenkins/.m2/settings.xml
ls -ld /var/lib/jenkins/.m2 /var/lib/jenkins/.m2/repository
```

如果权限不正确，修复：

```bash
sudo chown -R jenkins:jenkins /var/lib/jenkins/.m2
sudo chmod -R u+rwX /var/lib/jenkins/.m2
```

### 4.5 导入私有依赖

如果私有依赖包是一个压缩包，例如 `merach.zip`，导入到 Jenkins 用户的 Maven 仓库：

```bash
sudo mkdir -p /var/lib/jenkins/.m2/repository/com
sudo unzip -o enviroment/merach.zip -d /var/lib/jenkins/.m2/repository/com/
sudo chown -R jenkins:jenkins /var/lib/jenkins/.m2
ls -la /var/lib/jenkins/.m2/repository/com/merach/
```

使用 Jenkins 用户验证 Maven 可以读到配置：

```bash
sudo -u jenkins bash -c 'cd /var/lib/jenkins && /opt/tools/maven/bin/mvn -version'
```

### 4.6 配置 GitLab 凭据

GitLab 凭据配置方式和 Podman 方案一致：

```text
Manage Jenkins → Credentials → System → Global credentials → Add Credentials

Kind: Username with password
Username: <GitLab 用户名>
Password: <GitLab Personal Access Token>
ID: gitlab-credentials
Description: GitLab Personal Access Token for Jenkins
```

Token 至少需要 `read_repository` 权限。Pipeline 中通过 `credentialsId: 'gitlab-credentials'` 引用。

### 4.7 配置 SSH 私钥凭据

宿主机 Jenkins 使用 `sshagent` 插件时，仍然建议把私钥放到 Jenkins 凭据中，而不是依赖某个系统用户的默认 `~/.ssh/id_rsa`。

先在宿主机验证 SSH：

```bash
ssh root@<DEPLOY_SERVER_IP> "echo SSH OK"
```

再在 Jenkins 中新增凭据：

```text
Kind: SSH Username with private key
Username: root
Private Key: Enter directly
ID: deployment-ssh-key
Description: SSH key for deployment
```

如果目标服务器首次连接提示 known_hosts，可以先执行：

```bash
sudo -u jenkins ssh -o StrictHostKeyChecking=no root@<DEPLOY_SERVER_IP> "echo SSH OK"
```

### 4.8 准备部署脚本

宿主机方案推荐脚本路径：

```text
/opt/tools/scripts/deploy-to-docker.sh
/opt/tools/scripts/send-feishu-notification.sh
```

部署脚本准备：

```bash
sudo mkdir -p /opt/tools/scripts
sudo cp docs/jenkins/scripts/deploy-to-docker.sh /opt/tools/scripts/
sudo cp docs/jenkins/scripts/send-feishu-notification.sh /opt/tools/scripts/
sudo chmod +x /opt/tools/scripts/*.sh
ls -l /opt/tools/scripts/
```

宿主机 Jenkins 更可能同时部署到多台历史环境不同的服务器，所以部署脚本必须检测 Compose 命令：

```bash
if docker compose version &>/dev/null; then
    DOCKER_COMPOSE_CMD="docker compose"
elif docker-compose --version &>/dev/null; then
    DOCKER_COMPOSE_CMD="docker-compose"
else
    echo "错误: 未找到 docker compose 或 docker-compose 命令"
    exit 1
fi

${DOCKER_COMPOSE_CMD} -f docker-compose.yml up -d <PROJECT_NAME>
```

对每台目标服务器分别检测：

```bash
ssh root@<DEPLOY_SERVER_IP> 'docker compose version || docker-compose --version'
```

### 4.9 配置 Pipeline

宿主机版 Pipeline 和 Podman 版的主要差异是 Jenkins 访问端口和脚本路径：

```groovy
pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'test'], description: '选择部署环境')
        choice(name: 'BRANCH', choices: ['dev', 'main', 'test'], description: '选择部署分支')
    }

    environment {
        PROJECT_NAME = '<PROJECT_NAME>'
        SERVICE_NAME = '<SERVICE_NAME>'
        GITLAB_URL = '<PROJECT_REPO_URL>'
        DEV_SERVER = '<DEPLOY_SERVER_IP>'
        TEST_SERVER = '<DEPLOY_SERVER_IP>'
        DEV_DOCKER_DIR = '/home/docker'
        TEST_DOCKER_DIR = '/home/web/docker'
        JENKINS_BASE_URL = 'http://<JENKINS_SERVER_IP>:8080'
        MAVEN_OPTS = '-Dmaven.test.skip=true'
        FEISHU_WEBHOOK = '<FEISHU_WEBHOOK>'
    }

    tools {
        maven 'maven3.8.1'
        jdk 'jdk11'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: params.BRANCH,
                    credentialsId: 'gitlab-credentials',
                    url: "${GITLAB_URL}"
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package ${MAVEN_OPTS}'
                sh '''
                    JAR_FILE=$(find . -name "*-web.jar" -path "*/target/*" ! -name "*-sources.jar" ! -name "*-javadoc.jar" | head -1)
                    if [ -z "$JAR_FILE" ]; then
                        echo "未找到可部署 JAR"
                        find . -name "*.jar" -path "*/target/*" -exec ls -lh {} \; || true
                        exit 1
                    fi
                    echo "JAR 文件: $JAR_FILE"
                '''
            }
        }

        stage('Deploy') {
            steps {
                script {
                    def targetServer = params.DEPLOY_ENV == 'dev' ? env.DEV_SERVER : env.TEST_SERVER
                    def targetDockerDir = params.DEPLOY_ENV == 'dev' ? env.DEV_DOCKER_DIR : env.TEST_DOCKER_DIR
                    def appName = params.DEPLOY_ENV == 'dev' ? env.PROJECT_NAME : "${env.PROJECT_NAME}-test"

                    sshagent(['deployment-ssh-key']) {
                        sh """
                            set -e
                            JAR_FILE=\$(find . -name "*-web.jar" -path "*/target/*" ! -name "*-sources.jar" ! -name "*-javadoc.jar" | head -1)
                            ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 root@${targetServer} "echo SSH OK"
                            /opt/tools/scripts/deploy-to-docker.sh ${targetServer} ${appName} \$JAR_FILE ${targetDockerDir}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                def deployEnvName = params.DEPLOY_ENV == 'dev' ? '开发环境' : '测试环境'
                sh """
                    /opt/tools/scripts/send-feishu-notification.sh \
                        '${env.JOB_NAME}' \
                        '${env.SERVICE_NAME}' \
                        'SUCCESS' \
                        '${deployEnvName}' \
                        '${params.BRANCH}' \
                        '${env.JENKINS_BASE_URL}' \
                        '${env.BUILD_NUMBER}' \
                        '${env.FEISHU_WEBHOOK}'
                """
            }
        }
        failure {
            script {
                def deployEnvName = params.DEPLOY_ENV == 'dev' ? '开发环境' : '测试环境'
                sh """
                    /opt/tools/scripts/send-feishu-notification.sh \
                        '${env.JOB_NAME}' \
                        '${env.SERVICE_NAME}' \
                        'FAILURE' \
                        '${deployEnvName}' \
                        '${params.BRANCH}' \
                        '${env.JENKINS_BASE_URL}' \
                        '${env.BUILD_NUMBER}' \
                        '${env.FEISHU_WEBHOOK}'
                """
            }
        }
        always {
            cleanWs()
        }
    }
}
```

### 4.10 配置飞书通知

宿主机方案的飞书通知策略和 Podman 方案一致：

- Webhook 不要写死在 Git 仓库里。
- 如果机器人启用关键词，卡片标题或正文必须包含关键词。
- 推荐使用卡片消息展示构建状态。
- 成功和失败都发送通知。

建议飞书机器人关键词配置为：

```text
服务器部署
```

通知脚本参数顺序：

```bash
/opt/tools/scripts/send-feishu-notification.sh \
  '<PROJECT_NAME>' \
  '<SERVICE_NAME>' \
  'SUCCESS' \
  '开发环境' \
  'dev' \
  'http://<JENKINS_SERVER_IP>:8080' \
  '1' \
  '<FEISHU_WEBHOOK>'
```

手动测试通知：

```bash
/opt/tools/scripts/send-feishu-notification.sh \
  '<PROJECT_NAME>' \
  '<SERVICE_NAME>' \
  'SUCCESS' \
  '开发环境' \
  'dev' \
  'http://<JENKINS_SERVER_IP>:8080' \
  '1' \
  '<FEISHU_WEBHOOK>'
```

### 4.11 完整验证

宿主机 Jenkins 的完整验证顺序：

1. `systemctl status jenkins` 确认 Jenkins 服务运行。
2. Jenkins 页面确认 JDK 和 Maven 全局工具配置保存成功。
3. Jenkins 页面确认 `gitlab-credentials` 和 `deployment-ssh-key` 存在。
4. 手动触发一次 Pipeline。
5. Console Output 确认 GitLab 拉取成功。
6. Console Output 确认 Maven 构建成功。
7. Console Output 确认 `/opt/tools/scripts/deploy-to-docker.sh` 执行成功。
8. 目标服务器确认容器已更新：

```bash
ssh root@<DEPLOY_SERVER_IP> 'docker ps | grep <PROJECT_NAME>'
```

9. 目标服务器查看日志：

```bash
ssh root@<DEPLOY_SERVER_IP> 'docker logs <PROJECT_NAME> --tail 50'
```

10. 飞书群确认收到包含"服务器部署"的卡片消息。

## 5. 两种方案关键差异对比

| 对比项 | Podman 容器版 Jenkins | 已有宿主机 Jenkins |
| --- | --- | --- |
| 适用场景 | 全新搭建 Jenkins，希望隔离和方便迁移。 | 服务器已有 Jenkins，只需要补齐构建部署能力。 |
| Jenkins 运行方式 | `podman-compose up -d`。 | `systemctl start jenkins`。 |
| Jenkins 访问端口 | 示例为 `8180`。 | 常见为 `8080`。 |
| Jenkins 数据目录 | 宿主机挂载 `./jenkins_home`，容器内 `/var/jenkins_home`。 | `/var/lib/jenkins`。 |
| JDK 路径 | 容器内 `/opt/tools/jdk11`。 | 宿主机 `/opt/tools/jdk11`。 |
| Maven 路径 | 容器内 `/opt/tools/maven`。 | 宿主机 `/opt/tools/maven`。 |
| Maven 本地仓库 | `/root/.m2/repository` 或 `/var/jenkins_home/.m2/repository`，取决于 settings。 | 推荐 `/var/lib/jenkins/.m2/repository`。 |
| 部署脚本路径 | 容器内 `/deployment-scripts`，宿主机通过 volume 挂载。 | `/opt/tools/scripts`。 |
| 权限重点 | 保证挂载目录可被容器内 Jenkins 使用。 | 保证 `/var/lib/jenkins/.m2` 属于 `jenkins:jenkins`。 |
| Compose 兼容 | 部署脚本检测目标服务器的 `docker compose` 和 `docker-compose`。 | 同样检测，且多台历史服务器更需要兼容。 |
| 备份方式 | 备份 `jenkins_home` 目录。 | 备份 `/var/lib/jenkins` 和 `/opt/tools` 关键配置。 |
| 迁移便利性 | 更好，复制 compose 文件和挂载目录即可恢复。 | 依赖宿主机服务和系统环境，迁移成本更高。 |

简单选择建议：

- 新机器优先选择 Podman 容器版。
- 已经有 Jenkins 的机器优先选择宿主机初始化方案。
- 无论选择哪种方式，JDK、Maven、settings、私有依赖、GitLab 凭据、SSH 私钥、部署脚本、Pipeline、飞书通知都是必做项。

## 6. 常见问题与排错

### 6.1 Jenkins 无法拉取 GitLab 代码

**现象：** Pipeline 在 Checkout 阶段失败，提示认证失败、仓库不存在或无法访问。

**原因：** GitLab URL 错误、Token 过期、Token 缺少 `read_repository` 权限，或者 Jenkins 凭据 ID 和 Pipeline 中不一致。

**解决方式：**

```bash
# 在 Jenkins 服务器测试 GitLab 是否可访问
curl -I http://<GITLAB_HOST>
```

然后检查：

1. GitLab 项目页面复制的 Clone with HTTP URL 是否等于 Pipeline 中的 `GITLAB_URL`。
2. Jenkins 凭据 ID 是否为 `gitlab-credentials`。
3. GitLab Token 是否仍然有效。
4. Token 是否包含 `read_repository` 权限。

### 6.2 Maven 找不到私有依赖

**现象：** Build 阶段提示某些 `com.merach` 或内部依赖无法解析。

**原因：** 私有依赖未导入 Jenkins 使用的 Maven 本地仓库，或者 `settings.xml` 中的 `localRepository` 和实际导入路径不一致。

**解决方式：**

```bash
grep localRepository /opt/tools/maven/conf/settings.xml
ls -la /var/lib/jenkins/.m2/repository/com/merach/
ls -la /root/.m2/repository/com/merach/
```

确认 Jenkins 构建时使用哪个仓库，然后把依赖导入对应目录。

### 6.3 Maven 本地仓库目录无权限

**现象：** Maven 下载依赖或写入本地仓库时报 `Permission denied`。

**原因：** 宿主机 Jenkins 使用 `jenkins` 用户运行，但 `.m2` 目录属于 `root`。

**解决方式：**

```bash
sudo chown -R jenkins:jenkins /var/lib/jenkins/.m2
sudo chmod -R u+rwX /var/lib/jenkins/.m2
sudo -u jenkins /opt/tools/maven/bin/mvn -version
```

### 6.4 Jenkins 找不到 JDK 或 Maven

**现象：** Pipeline 提示找不到工具，或者 `mvn`、`java` 命令不存在。

**原因：** Jenkins 全局工具配置的 Name 和 Pipeline 中 `tools` 配置不一致，或者 `JAVA_HOME`、`MAVEN_HOME` 路径错误。

**解决方式：**

确认 Jenkins 全局工具：

```text
JDK Name: jdk11
JAVA_HOME: /opt/tools/jdk11

Maven Name: maven3.8.1
MAVEN_HOME: /opt/tools/maven
```

Pipeline 中保持一致：

```groovy
tools {
    maven 'maven3.8.1'
    jdk 'jdk11'
}
```

### 6.5 SSH 私钥无法连接目标服务器

**现象：** Deploy 阶段提示 `Permission denied`、`Connection refused` 或连接超时。

**原因：** 私钥没有配置到目标服务器、Jenkins 凭据中粘贴的私钥不完整、目标服务器 SSH 端口不通。

**解决方式：**

```bash
ssh root@<DEPLOY_SERVER_IP> "echo SSH OK"
ssh-copy-id root@<DEPLOY_SERVER_IP>
```

然后重新检查 Jenkins 中 `deployment-ssh-key` 凭据。

### 6.6 目标服务器 Docker 命令无权限

**现象：** SSH 成功，但执行 `docker ps` 或 Compose 命令失败。

**原因：** 部署用户没有 Docker 权限，或者目标服务器 Docker 服务未启动。

**解决方式：**

```bash
ssh root@<DEPLOY_SERVER_IP> 'systemctl status docker || systemctl status podman'
ssh root@<DEPLOY_SERVER_IP> 'docker ps'
```

如果不用 root 部署，需要把部署用户加入 Docker 用户组，并重新登录会话。

### 6.7 目标服务器缺少 `docker compose` 或 `docker-compose`

**现象：** 部署脚本提示未找到 Compose 命令。

**原因：** 目标服务器没有安装 Docker Compose v2 插件，也没有安装旧版 `docker-compose`。

**解决方式：**

```bash
ssh root@<DEPLOY_SERVER_IP> 'docker compose version || docker-compose --version'
```

如果两者都不存在，先在目标服务器安装其中一种。部署脚本中保留兼容检测：

```bash
if docker compose version &>/dev/null; then
    DOCKER_COMPOSE_CMD="docker compose"
elif docker-compose --version &>/dev/null; then
    DOCKER_COMPOSE_CMD="docker-compose"
else
    exit 1
fi
```

### 6.8 JAR 上传成功但容器启动失败

**现象：** `scp` 成功，但脚本最后提示容器启动失败。

**原因：** `docker-compose.yml` 服务名不匹配、Dockerfile 构建失败、端口冲突、应用启动报错。

**解决方式：**

```bash
ssh root@<DEPLOY_SERVER_IP>
cd /home/docker
 docker compose -f docker-compose.yml ps || docker-compose -f docker-compose.yml ps
 docker logs <PROJECT_NAME> --tail 100
```

检查 Compose 服务名是否和 Pipeline 中的 `<PROJECT_NAME>` 一致。

### 6.9 飞书通知因为缺少关键词而发送失败

**现象：** `curl` 返回 HTTP 200，但飞书群里没有消息。

**原因：** 飞书机器人启用了关键词安全设置，但通知标题或正文没有包含关键词。

**解决方式：**

1. 在飞书机器人安全设置中确认关键词。
2. 让卡片标题包含该关键词，例如"服务器部署"。
3. 手动执行通知脚本测试。

### 6.10 飞书卡片消息格式错误导致通知失败

**现象：** 通知脚本返回 HTTP 400 或飞书响应体提示 JSON 格式错误。

**原因：** JSON 字符串拼接时引号、换行或变量内容破坏了卡片结构。

**解决方式：**

```bash
bash -n /opt/tools/scripts/send-feishu-notification.sh
/opt/tools/scripts/send-feishu-notification.sh '<PROJECT_NAME>' '<SERVICE_NAME>' 'SUCCESS' '开发环境' 'dev' 'http://<JENKINS_SERVER_IP>:8080' '1' '<FEISHU_WEBHOOK>'
```

优先使用固定模板，只替换字段值。

### 6.11 Jenkins 构建成功但服务未更新

**现象：** Jenkins 显示成功，但访问服务还是旧版本。

**原因：** JAR 上传目录和 Dockerfile 引用目录不一致，Compose 服务名不一致，或者启动的是旧镜像。

**解决方式：**

```bash
ssh root@<DEPLOY_SERVER_IP> 'ls -lh /home/docker/robot/<PROJECT_NAME>/jar/'
ssh root@<DEPLOY_SERVER_IP> 'docker images | grep <PROJECT_NAME>'
ssh root@<DEPLOY_SERVER_IP> 'docker ps | grep <PROJECT_NAME>'
```

确认部署脚本中删除旧容器和旧镜像后再执行 Compose 启动。

## 7. 总结与快速复用清单

以后在新机器上重建 Jenkins Java 部署能力时，可以按下面清单操作：

- [ ] 选择 Jenkins 安装方式：Podman 容器版或已有宿主机 Jenkins。
- [ ] 确认 Jenkins 页面可以访问。
- [ ] 安装或确认必要插件：GitLab、SSH Agent、Maven、Docker Pipeline。
- [ ] 初始化 JDK 11。
- [ ] 初始化 Maven 3.8.1。
- [ ] 配置 Maven `settings.xml`。
- [ ] 确认 Maven 本地仓库路径。
- [ ] 确认 Maven 本地仓库权限。
- [ ] 导入私有依赖。
- [ ] 配置 GitLab 凭据。
- [ ] 配置 SSH 私钥凭据。
- [ ] 准备 `deploy-to-docker.sh`。
- [ ] 准备 `send-feishu-notification.sh`。
- [ ] 检测每台目标服务器支持 `docker compose` 还是 `docker-compose`。
- [ ] 配置参数化 Pipeline。
- [ ] 配置飞书 Webhook。
- [ ] 确认飞书机器人关键词能被通知标题或正文命中。
- [ ] 使用飞书卡片消息发送构建结果。
- [ ] 手动触发一次完整构建。
- [ ] 验证 GitLab 拉取成功。
- [ ] 验证 Maven 构建成功。
- [ ] 验证 JAR 上传成功。
- [ ] 验证目标服务器容器更新成功。
- [ ] 验证应用健康检查成功。
- [ ] 验证飞书群收到成功或失败通知。

如果是全新环境，优先使用 Podman 容器版，后续迁移时只要带走 compose 文件、`jenkins_home` 和部署脚本目录即可。如果机器上已经有稳定运行的 Jenkins，优先使用宿主机初始化方案，重点检查 `/var/lib/jenkins/.m2` 权限和 `/opt/tools` 工具路径。

