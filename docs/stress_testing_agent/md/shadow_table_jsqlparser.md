## 影子表之使用JSqlParser提取表名

首先我们看一下Mybatis-Plus的[ Release Notes](https://github.com/baomidou/mybatis-plus/releases?page=1)，在 `Mybatis 3.5.9`
拆分`JSqlParser`模块， 需要自行引入[https://mvnrepository.com/artifact/com.baomidou/mybatis-plus-jsqlparser/3.5.9](https://mvnrepository.com/artifact/com.baomidou/mybatis-plus-jsqlparser/3.5.9)

## 提取表名

```java
private static String replaceSqlTables(String sql){
        Statement statement=null;
        try{
            statement=CCJSqlParserUtil.parse(sql);
        }catch(JSQLParserException ignored){

        }
        if(statement==null){
            return sql;
        }
        Set<String> tableSet = new TablesNamesFinder().getTables(statement);
        for(String tableName:tableSet){
            sql=sql.replace(tableName,tableName+"_");
        }
        return sql;
}
```
顺便提一下，其实`mybatis-plus-jsqlparser`就是在`com.github.jsqlparser`基础上做的封装，我们直接引入`com.github.jsqlparser`也是可以的