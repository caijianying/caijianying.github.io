## 介绍
为避免一头雾水先说明一下，开始本项目需具备的前置知识
* [什么是全链路压测？](/stress_testing_agent/md/total_chain_testing)
* JavaAgent
* 字节码增强框架 ByteBuddy

演示项目
* [演示Agent](https://github.com/caijianying/stress-testing-app-agent)
* [测试应用](https://github.com/caijianying/stress-testing-app)


本站将在highLight(测试应用)中接入highLight(演示Agent)，逐步完善Agent的主要功能。

## 演示Agent打包
### 运行环境
* JDK17
* Gradle 8.5

编译成功后，执行任务 Tasks-> shadow -> shadowJar 即可打包成agent `stress-testing-app-agent-1.0.0-all.jar`

## 测试应用启动
### 运行环境
* Mysql 5.7+
* JDK 17
* Maven 3.8.1

1. 在Mysql中执行项目的sql文件 `/sql/database.sql`,
2. 修改 `application-local.yml`中DB的配置信息
3. 启动

### 如何接入演示Agent
highLight(测试应用)启动命令中添加 `vmOptions`, 启动即可
* `-javaagent:/your/path/stress-testing-app-agent-1.0.0-all.jar`


## 开始阅读
为方便理解，请按顺序阅读。
1. 识别压测标
2. 压测标透传
3. 数据隔离
   * 3.1 -影子表方式
   * 3.2 -影子库方式
4. 链路监控
5. 熔断通知

为了可以让大家快速上手，highLight(演示项目)中的代码比较通俗易懂，highLight(开源项目)则是在此基础上做了相关优化和框架重构。

创作不易，觉得对你有帮助的可以顺便支持(highLight(其实也就是star))一下我的开源项目，非常感谢！

[支持STA](/stress_testing_agent/)




