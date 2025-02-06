> 数据隔离是指将业务数据和压测数据在存储层进行隔离，常见的方案有影子库、影子表、影子Topic、影子缓存等

本例以highLight(影子表、影子库)为例，对Mysql做数据隔离。

## 影子库和影子表

### 什么是影子库、影子表

* 影子库是指在同一个数据库实例上，基于业务库创建的新库，新库与业务库的表结构一致。
* 影子表是指在同一个数据库中创建的与业务遍结构一致的新表

不同压测平台对影子库表的命名规则也不一样，本例的标准为

* 影子表命名规则为`${TABLE}_`, 如业务表`t_user`的影子表为 `t_user_`
* 影子库命名规则为`${DATABASE}_`,如业务库`stress_db`的影子库为`stress_db_`

### 影子库、影子表怎么选择

> 影子库和影子表是影子存储的2种解决方案，是非此即彼的关系，各有优劣，要按实际情况选择。

| 方案 | 影子库 | 影子表 |
|------|------|-------|
| 优点 | 隔离程度高，对业务库0影响   | 隔离程度较高，中间件支持较容易 |
| 缺点 | 中间件支持较难   | 会使业务库压力增大 |
| 适应场景 | 流量全服打标、数据库资源充裕   | 流量全服打标 |

----

## 实现数据隔离

### 怎么实现数据隔离

> 影子库、影子表应该怎么实现数据隔离？

假设`select * from t_user where id = 1`这条sql目前是走业务表查询的，

* 如果想让它查询影子表，那么sql应该变为 `select * from t_user_ where id = 1`
* 同理如果想让它查询影子库，那么sql应该变为 `use stress_db_; select * from t_user where id = 1`

collapse(为什么要在sql上直接加`use stress_db_; `,因为是否使用highLight(影子存储)取决于是业务数据还是压测数据，不能做全局配置)

所以，我们重点就是要修改即将执行的sql。

----

### 代码撸起来

笔者经过多次调试，将目光锁定在了`org.apache.ibatis.mapping.BoundSql#getSql`方法上。

我们来看此方法

```java
 public String getSql(){
    return this.sql;
 }
```

很简单，`BoundSql`内部维护了一个`sql`的变量，那么我们拦截到这个方法后，只要对这个变量执行`set`方法，就能达到变更sql的目的。 同样的，我们先委托一个拦截器处理

```java
agentBuilder=agentBuilder.type(named("org.apache.ibatis.mapping.BoundSql"))
        .transform((builder,typeDescription,classLoader,module)->
        builder.method(named("getSql")).intercept(MethodDelegation.to(new BoundSqlInterceptor()))
        );
```

#### BoundSqlInterceptor

> 先把架子搭好

```java
public static class BoundSqlInterceptor {
    @RuntimeType
    public Object intercept(@This Object obj, @Origin Class<?> clazz, @AllArguments Object[] allArguments, @Origin Method method, @SuperCall Callable<?> callable) throws Throwable {
        /***
         * 此处逻辑
         * 1. 判断是否是压测流量，若是则进行下一步
         * 2. 调用getSql方法获取Sql
         * 3. 获取压测流量写入模式：影子表或者影子库
         * 4. 写入压测流量
         **/


        Object call = null;
        try {
            call = callable.call();
        } catch (Throwable ignored) {

        }
        return call;
    }
}
```
##### 判断是否是压测流量
上一步[Step2: 压测标透传](/stress_testing_agent/md/stress_flag_transfer.md)，可以从上下文获取到此流量是否是压测流量
```java
 Boolean inPt = (Boolean) ContextManager.getProperty(StressTestingConstant.HEADER_NAME_STRESS_TESTING_FLAG);
```
`inPt`为`true`则为highLight(压测流量)

##### 获取SQL

```java
        Method getSqlMethod=obj.getClass().getDeclaredMethod("getSql");
        getSqlMethod.setAccessible(true);
        Object sqlObj=getSqlMethod.invoke(obj);
        String sql=sqlObj==null?null:sqlObj.toString();
        System.out.println("Original SQL: "+sql);
```

注意：由于我们会对`getSql`方法进行拦截，所以这里调用`getSql`方法会形成循环调用造成highLight(栈溢出)的问题。那么只要确保只拦截一次`getSql`方法就行了，这里使用Set简单处理一下。

修改后的代码

```java
        if(methodInvokeSet.add(obj)){
            Method getSqlMethod=obj.getClass().getDeclaredMethod("getSql");
            getSqlMethod.setAccessible(true);
            Object sqlObj=getSqlMethod.invoke(obj);
            String sql=sqlObj==null?null:sqlObj.toString();
            System.out.println("Original SQL: "+sql);
        }
```

##### 获取压测流量写入模式

我们可以写一个类从启动参数中读取Agent配置，

这样就能在执行premain时，读取启动参数: `AgentConfig.readArgs(agentArgs)`

```java
import java.util.HashMap;
import java.util.Map;

/**
 * @author xiaobaicai
 * @description 关注微信公众号【程序员小白菜】领取源码
 * @date 2024/12/10 星期二 18:19
 */
public class AgentConfig {
    private static final Map<String, String> properties = new HashMap<>();
    private static final String SHADOW_MODE_KEY = "shadowMode";
    public static final String SHADOW_MODE_DB = "DB";
    public static final String SHADOW_MODE_TABLE = "TABLE";
    private static final String DEFAULT_SHADOW_MODE = SHADOW_MODE_TABLE;

    public static void readArgs(String agentArgs) {
        String[] args = agentArgs.split("&");
        for (String arg : args) {
            String[] param = arg.split("=");
            properties.put(param[0], param[1]);
        }
    }

    private static String getProperty(String key) {
        return properties.get(key);
    }

    public static String getShadowMode() {
        String shadowModel = properties.get(SHADOW_MODE_KEY);
        return shadowModel == null ? DEFAULT_SHADOW_MODE : shadowModel;
    }
}
```

##### 修改SQL

* 影子表

```java
        // 影子表
        String newSql=sql.replace("t_user","t_user_");
        System.out.println("Shadow Table SQL: "+newSql);

        Field sqlField=obj.getClass().getDeclaredField("sql");
        sqlField.setAccessible(true);
        sqlField.set(obj,newSql);
```

以上实现是初版，还是有优化的空间的。我们可以把`t_user`改为highLight(从sql中提取表名)，这样也能使多表JOIN走影子表。

本例使用highLight(Mybatis-PLus)自带的highLight(JSqlParser)提取表名，但highLight(Mybatis 3.5.9)后需要单独引入highLight(JSqlParser)。

具体可查看[《影子表之使用JSqlParser提取表名》](./md/shadow_table_jsqlparser.md)

* 影子库

```java
        // 影子库
        String newSql="use stress_db_;"+sql;
        System.out.println("Shadow Database SQL: "+newSql);

        Field sqlField=obj.getClass().getDeclaredField("sql");
        sqlField.setAccessible(true);
        sqlField.set(obj,newSql);
```

关于数据库的获取，基本原理是拦截`org.apache.ibatis.session.defaults.DefaultSqlSessionFactory`得到连接的数据库名称，再通过`ThreadLocal`
透传到`BoundSqlInterceptor`。

具体可查看[《影子库之获取数据库名称》](./md/shadow_db_get_db_name.md)

##### 完整代码

```java
public static class BoundSqlInterceptor {
    Set<Object> methodInvokeSet = new HashSet<>();

    @RuntimeType
    public Object intercept(@This Object obj, @Origin Class<?> clazz, @AllArguments Object[] allArguments, @Origin Method method, @SuperCall Callable<?> callable) throws Throwable {

        /***
         * 此处逻辑
         * 1. 判断是否是压测流量，若是则进行下一步
         * 2. 调用getSql方法获取Sql
         * 3. 获取压测流量写入模式：影子表或者影子库
         * 4. 写入压测流量
         **/

        Boolean inPt = (Boolean) ContextManager.getProperty(StressTestingConstant.HEADER_NAME_STRESS_TESTING_FLAG);
        if (inPt != null && inPt) {
            if (methodInvokeSet.add(obj)) {
                // 调用getSql方法获取Sql
                Method getSqlMethod = obj.getClass().getDeclaredMethod("getSql");
                getSqlMethod.setAccessible(true);
                Object sqlObj = getSqlMethod.invoke(obj);
                String originalSql = sqlObj == null ? null : sqlObj.toString();
                System.out.println("Original SQL: " + originalSql);

                // 2. 获取压测流量写入模式：影子表或者影子库
                String shadowMode = AgentConfig.getShadowMode();
                Field sqlField = obj.getClass().getDeclaredField("sql");

                // 3. 写入压测流量
                // 影子表
                if (AgentConfig.SHADOW_MODE_TABLE.equals(shadowMode)) {
                    // 提取SQL中表名，并替换为 ${TABLE_NAME}_
                    String newSql = this.replaceSqlTables(originalSql);
                    System.out.println("Shadow Table SQL: " + newSql);
                    sqlField.setAccessible(true);
                    sqlField.set(obj, newSql);
                }

                // 影子库
                if (AgentConfig.SHADOW_MODE_DB.equals(shadowMode)) {
                    // 上下文中获取数据库名称
                    Object dataBaseName = ContextManager.getProperty(StressTestingConstant.DATABASE_NAME_KEY);
                    String newDataBaseName = dataBaseName + "_";
                    String modifiedSql = "use " + newDataBaseName + ";" + originalSql;
                    System.out.println("Shadow Database SQL: " + modifiedSql);
                    sqlField.set(obj, modifiedSql);
                }
            }
        }

        Object call = null;
        try {
            call = callable.call();
        } catch (Throwable ignored) {

        }
        return call;
    }
}
```

