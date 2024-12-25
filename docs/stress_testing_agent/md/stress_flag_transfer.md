> 本例场景较简单，不存在跨线程或跨服务，用ThreadLocal就能完成透传。

可以写一个上下文管理工具，实现比较简单，就不做过多说明了。

### ContextManager

```java
/**
 * @author xiaobaicai
 * @description 关注微信公众号【程序员小白菜】领取源码
 * @date 2024/12/9 星期一 14:05
 */
public class ContextManager {

    public static final ThreadLocal<Map<String, Object>> LOCAL_PROPERTY = new ThreadLocal<>();

    public static void setProperty(String key, Object value) {
        Map<String, Object> properties = LOCAL_PROPERTY.get();
        if (properties == null) {
            properties = new HashMap<>();
            LOCAL_PROPERTY.set(properties);
        }
        properties.put(key, value);
    }

    public static Object getProperty(String key) {
        Map<String, Object> properties = LOCAL_PROPERTY.get();
        return properties == null ? null : properties.get(key);
    }

    public static void removeProperty() {
        LOCAL_PROPERTY.remove();
    }
}

```

