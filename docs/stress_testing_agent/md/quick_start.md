## 前提条件
> 本项目适合有 Java 基础的开发者，默认读者对 `JavaAgent`、`ByteBuddy` 和 `全链路压测` 概念有一定了解。

开始本项目需具备的前置知识
* [什么是全链路压测？](/stress_testing_agent/md/total_chain_testing)
* JavaAgent
* 字节码增强框架 ByteBuddy



## 项目简介
基于SkyWalking搭建的全链路压测Agent，该项目仅供学习使用

## 安装说明
### 运行环境
JDK 17

### 安装步骤
1. 从 [Release](https://github.com/caijianying/Stress-Testing-Agent/releases)下载 highLight(zip) 包
2. 解压后，在java项目的highLight(启动命令)中添加 `-javaagent:/your/path/libs/agent.jar`, 启动即可

## 参数说明

| 功能 | 参数项 | 参数值 | 备注 |
| --- | --- | --- | --- |
|压测标|pt-flag|STA|HTTP请求，header头添加`pt-flag:STA`|
|影子模式|shadowMode|DB或TABLE，默认TABLE|启动命令添加，例：`-javaagent:/your/path/libs/agent.jar=shadowMode=TABLE`|

⏰ 注意：请带上highLight(压测标)调试

## 调试结果
> 以下是使用highLight(演示项目)调试的结果 

```shell
2025-02-11T17:49:58.905350 INFO  --- [ Stress-Testing-Agent ] c.x.a.plugins.mybatis3.BoundSqlInterceptor : SwitchToShadowTable, Original SQL: SELECT id,nick_name,user_description,create_by,create_time,update_by,update_time,is_deleted FROM t_user WHERE id=? AND is_deleted=0
2025-02-11T17:49:58.950267 INFO  --- [ Stress-Testing-Agent ] c.x.a.plugins.mybatis3.BoundSqlInterceptor : SwitchToShadowTable, Modified SQL: SELECT id,nick_name,user_description,create_by,create_time,update_by,update_time,is_deleted FROM t_user_ WHERE id=? AND is_deleted=0
2025-02-11T17:49:59.129890 INFO  --- [ Stress-Testing-Agent ] c.x.a.core.plugin.meltdown.MeltDownManager : http-nio-8811-exec-1: 方法执行后任务关闭.
2025-02-11T17:49:59.130684 INFO  --- [ Stress-Testing-Agent ] c.x.a.core.plugin.context.ContextManager   : 
【ROOT】/api/user/getUser  548ms
     【SPRING】com.xiaobaicai.stress.testing.app.controller.UserController.getUser  491ms
          【SPRING】com.xiaobaicai.stress.testing.app.service.impl.UserDaoServiceImpl.getById  364ms
               【SQL】execute Execute SQL. SELECT id,nick_name,user_description,create_by,create_time,update_by,update_time,is_deleted FROM t_user_ WHERE id=? AND is_deleted=0  6ms

```

## 演示项目
为了帮助大家快速上手和理解highLight(STA), 我准备了演示项目供大家查看 [👈点击查看](/stress_testing_agent/md/demo_proj.md)

## 最后
创作不易，觉得对你有帮助的可以顺便支持(highLight(其实也就是star))一下我的开源项目，非常感谢！

[支持STA](/stress_testing_agent/)


