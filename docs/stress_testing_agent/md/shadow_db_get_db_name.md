## 影子库之获取数据库名称
熟悉Mybatis的应该都知道，`org.apache.ibatis.session.defaults.DefaultSqlSessionFactory`是`org.apache.ibatis.session.SqlSessionFactory`的默认实现类，
它在SpringBoot自动装配时会执行`openSession`方法, 可以看到`openSession`方法是重载的。

```java
public interface SqlSessionFactory {

  SqlSession openSession();

  SqlSession openSession(boolean autoCommit);

  SqlSession openSession(Connection connection);

  SqlSession openSession(TransactionIsolationLevel level);

  SqlSession openSession(ExecutorType execType);

  SqlSession openSession(ExecutorType execType, boolean autoCommit);

  SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level);

  SqlSession openSession(ExecutorType execType, Connection connection);

  Configuration getConfiguration();

}
```
大部分情况下，我们是不会主动实现`SqlSessionFactory`的。

我们可以拦截`DefaultSqlSessionFactory`的`openSession`方法，拿到返回值`SqlSession`。再通过`SqlSession`拿到`connection`，执行`java.sql.Connection#getCatalog`得到数据库名称。

那么，动手

## 委托拦截器
```java
agentBuilder=agentBuilder.type(named("org.apache.ibatis.session.defaults.DefaultSqlSessionFactory"))
        .transform((builder,typeDescription,classLoader,module)->
            builder.method(named("openSession")).intercept(MethodDelegation.to(new DefaultSqlSessionFactoryInterceptor()))
        );
```

## 拦截器 DefaultSqlSessionFactoryInterceptor
我们细想一下，DB的获取方式是 `SqlSession#getConnection#getCatalog`。
直接通过反射做并不是最好的方案，这里通过highLight(引入依赖)的方式。

https://mvnrepository.com/artifact/org.mybatis/mybatis/3.5.9

```java
public static class DefaultSqlSessionFactoryInterceptor {

        @RuntimeType
        public Object intercept(@This Object obj, @Origin Class<?> clazz, @AllArguments Object[] allArguments, @Origin Method method, @SuperCall Callable<?> callable) {
            Object call = null;
            try {
                call = callable.call();
            } catch (Throwable ignored) {

            }

            if (call != null) {
                try {
                    SqlSession sqlSession = (SqlSession) call;
                    ContextManager.setProperty(StressTestingConstant.DATABASE_NAME_KEY, sqlSession.getConnection().getCatalog());
                } catch (Throwable ex) {
                    System.err.println(ex.getMessage());
                }
            }
            return call;
        }
    }
```
这里获取到数据库名称后，会存储在上下文中，上下文管理器已在[Step2: 压测标透传](/stress_testing_agent/md/stress_flag_transfer.md)提到。

