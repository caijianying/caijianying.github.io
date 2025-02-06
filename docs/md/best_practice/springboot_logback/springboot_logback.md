## 前言
在接手项目的过程中，发现了项目SpringBoot日志其实配置得不是很优雅。这里特地拿出来分享一下。

## 原来的样子
> 怎么使用logback就不说了，我们主要看Resource目录的结构。

```
resources
    - application.properties
    - logback-spring.xml
    - logback-spring2.xml
```
咋一看有点懵，为啥有个`logback-spring2.xml`.对比了才发现，原来是想配置本地的路径。
```xml
<property name="log.path" value="/local-log"/>
```

## 解决
> 针对这个问题，其实我比较推荐使用`springProperty`去配置。

## 现在的样子
Resource目录结构，可调整为：
```
resources
    - application.properties
    - application-local.properties
    - logback-spring.xml
```

1. 分别在`application.properties`和`application-local.properties`中，配置各自的日志路径：
```properties
log.path=
```

2. 在`logback-spring.xml`配置变量
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="10 seconds">
...
<property resource="application.properties"></property>
<springProperty scope="context" name="log.path" source="log.path"></springProperty>
...
</configuration>
```

3. 应用启动只要指定环境`-Dspring.profiles.active=local`就能使用到本地配置的路径了。