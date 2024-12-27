## 熔断通知
首先这里讲一下为什么这里不叫highLight(压测熔断)。因为作为highLight(压测Agent)这个角色，只能做到highLight(熔断通知)。真正执行压测的是highLight(压测引擎)，它的职责就是执行压测脚本，根据压测模式进行压测。
真正中止压测，是压测Agent通知highLight(压测引擎)后，压测引擎才会停止压测。

## 演示场景
演示场景是，压测中的应用程序，如果highLight(10秒)内没有响应，则由highLight(压测Agent)主动通知压测引擎，中止压测任务。

## 实现
我们可以通过之前写的拦截器`DispatcherServletInterceptor`对一次请求进行拦截，在执行请求前创建一个任务定时扫描，若超过10秒方法没有执行则发送熔断通知。

**关键代码**
```java
            // 1. 创建定时任务扫描
            Object call = null;
            try {
                call = callable.call();
            } catch (Throwable ignored) {

            }
            // 2. 判断是否超过10s才执行，若是则发送熔断通知
            
```
为此写了个熔断管理器`MeltDownManager`，代码如下
```java
/**
 * @author xiaobaicai
 * @description 关注微信公众号【程序员小白菜】领取源码
 * @date 2024/12/20 星期五 16:26
 */
public class MeltDownManager {

    private static final int SECONDS_TO_MELTDOWN = 10 * 1000;

    private static volatile boolean NEED_MELTDOWN = false;

    /**
     * 开启定时任务，检测方法是否已执行
     **/
    public static long meltDownIfNecessary() {
        long start = System.currentTimeMillis();
        Timer timer = new Timer();
        TimerTask timerTask = new TimerTask() {
            @Override
            public void run() {
                System.out.println("校验熔断条件...");
                if (NEED_MELTDOWN) {
                    NEED_MELTDOWN = false;
                    System.out.println("发起压测熔断，通知压测引擎中止压测任务.");
                }
                if (System.currentTimeMillis() - start >= SECONDS_TO_MELTDOWN) {
                    timer.cancel();
                }
            }
        };
        timer.schedule(timerTask, 0, 1000);
        return start;
    }

    /**
     * 如果超过了10s，则标记熔断
     **/
    public static void markMeltDownFlag(long start) {
        if (System.currentTimeMillis() - start >= SECONDS_TO_MELTDOWN) {
            // 需要做熔断通知的标识
            NEED_MELTDOWN = true;
        }
    }
}
```
在方法调用之前执行`meltDownIfNecessary`方法，开启检测任务，每秒检测一次方法是否被调用。
若highLight(10秒)内方法未被调用，则发送熔断通知。
方法调用后，执行`markMeltDownFlag`方法，标记已调用。

有以下几个highLight(核心)的点, 但不是最优版，highLight(优化版)已在highLight(STA)中实现，[点击查看](/stress_testing_agent/)
* 这里为了实现简单，使用`Timer`和`TimerTask`实现定时任务
* 是否需要发起熔断，采用`NEED_MELTDOWN`标记，`volatile`保证线程之间的highLight(可见性)。为了方便演示，这里没有考虑HTTP的异步调用
* highLight(10秒)后，定时任务主动停止