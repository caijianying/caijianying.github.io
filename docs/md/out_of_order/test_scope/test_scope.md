## 前言

SpringBoot中间件在使用时，有时会出现依赖不到，或者引入中间件后打包报错的情况。这里有部分原因就是maven依赖的scope作用域没指定正确导致的。

理解了maven依赖的scope作用域，就可以在中间件即将投入使用时，减少不必要的依赖问题。

## maven依赖的scope作用域

> 我们在使用maven的时候，有时候并不希望依赖直接打包到正式的发布包里，只想在测试或者编译时候使用。而maven的scope在解决这个问题的同时，还可以利用scope控制依赖范围。
>
[查看官网描述](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html#Dependency_Scope)

工作中常见的maven作用域有：compile、provided、runtime、test

根据官网的描述，我简单归纳了一下

| scope | 是否可打入正式包(运行时使用) | 在测试时使用 | 在编译时使用 | 依赖是否可传递 |
| ---| --- | --- | --- | --- |
|compile|Y|Y|Y|Y|
|provided|N|Y|Y|N|
|runtime|Y|Y|N|Y|
|test|N|Y|N|N|

这里其实就得出来一个结论了。

**测试库的依赖(如下👇)只能由各自项目指定，不能做统一的整合了。如果采用默认的方式，会使得正式包多了无用的依赖，让正式包更加臃肿。**

    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <scope>test</scope>
    </dependency>


