## 链接
> 推荐将两个工程下载到本地运行

* [演示Agent](https://github.com/caijianying/stress-testing-app-agent)
* [测试应用](https://github.com/caijianying/stress-testing-app)

## 介绍
演示项目包含了 Agent 完整的基础功能，适合初学者快速理解

highLight(演示Agent)的highLight(main)分支是完整代码供大家查看。

highLight(demo)分支包含写好的方法和工具类，建议使用此分支highLight(跟随本站博客或视频进行实战)

## 使用步骤


1. [演示Agent打包](/stress_testing_agent/md/demo_proj.md#演示Agent打包)
2. [测试应用启动](/stress_testing_agent/md/demo_proj.md#测试应用启动)


### 演示Agent打包
#### 运行环境
* JDK17
* Gradle 8.5

#### 打包步骤
1. 下载项目到本地
2. 执行任务 Tasks-> shadow -> shadowJar 即可打包成agent `stress-testing-app-agent-1.0.0-all.jar`

### 测试应用启动
#### 运行环境
* Mysql 5.7+
* JDK 17
* Maven 3.8.1

#### 执行步骤
1. 在Mysql中执行项目的sql文件 `/sql/database.sql`,
2. 修改 `application-local.yml`中DB的配置信息
3. 接入演示Agent：
   * 启动命令中添加 `vmOptions`， `-javaagent:/your/path/stress-testing-app-agent-1.0.0-all.jar`
4. 启动测试应用


## 开始阅读
为方便理解，请按顺序阅读。
1. 识别压测标
2. 压测标透传
3. 数据隔离
    * 3.1 -影子表方式
    * 3.2 -影子库方式
4. 链路监控
5. 熔断通知

## 最后

创作不易，觉得对你有帮助的可以顺便支持(highLight(其实也就是star))一下我的开源项目，非常感谢！

[支持STA](/stress_testing_agent/)