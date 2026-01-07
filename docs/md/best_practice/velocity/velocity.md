## 前言
Velocity是一个模板引擎，常用于MVC结构中的view层，也可用于动态内容静态化，相似的模板引擎还有Thymeleaf和Freemarker。

## 快速开始
1. 首先引入依赖

```maven
       <!-- https://mvnrepository.com/artifact/org.apache.velocity/velocity-engine-core -->
        <dependency>
            <groupId>org.apache.velocity</groupId>
            <artifactId>velocity-engine-core</artifactId>
            <version>2.3</version>
        </dependency>
```
2. 创建模板文件
> 这里包含了文件中设置变量的方法,具体的可以去[Apache Velocity官网](https://velocity.apache.org/)了解
```Velocity
        #set( $iAmVariable = "good!" )
        Welcome $name to velocity.com
        today is $date.
        #foreach ($dto in $list)
            $dto.title
            $dto.desc
        #end
        $iAmVariable
```
3. 单元测试

```java
       @Test
       public void testVelocity(){
            VelocityEngine ve = new VelocityEngine();
            ve.setProperty(RuntimeConstants.RESOURCE_LOADER, "classpath");
            ve.setProperty("classpath.resource.loader.class", ClasspathResourceLoader.class.getName());
            ve.init();
    
            Template t = ve.getTemplate("hellovelocity.vm");
            VelocityContext ctx = new VelocityContext();
    
            ctx.put("name", "velocity");
            ctx.put("date", "2020-12-21");
    
            List<MsgDTO> temp = new ArrayList();
            temp.add(new MsgDTO(){{setDesc("d1");setTitle("t1");}});
            temp.add(new MsgDTO(){{setDesc("d2");setTitle("t2");}});
            ctx.put("list", temp);
    
            StringWriter sw = new StringWriter();
    
            t.merge(ctx, sw);
    
            System.out.println(sw.toString());
       }
       
       @Data
       public static class MsgDTO{
       private String title;
       private String desc;
       }
```
4. 运行结果
   ```bash
        Welcome velocity to velocity.com
        today is 2020-12-21.
        t1
        d1
        t2
        d2
        good!
   ```