> 为了对JVM知识有更体系化的了解，本文梳理了知识结构。若有需要进一步了解的可以移步参考资料。

## 运行时数据区域
* 线程共享：堆、方法区
* 线程私有：程序计数器、虚拟机栈、本地方法栈

## 对象的创建

1. 类加载检查：加载、连接、初始化。连接：验证、准备、解析。双亲委派机制后面会说明
2. 分配内存：指针碰撞(内存空间规整：标记-整理、复制)、空闲列表(不规整：标记-清除)
   * 并发问题：CAS+失败重试、TLAB(TheadLocal Allocate Buffer)
3. 初始化零值
4. 设置对象头
5. 执行init方法

## 内存布局

* 对象头：
  * 实例运行时数据：分代年龄、hash码、锁态等。
  * 类型指针：表示这个实例是由哪个类产生的
* 实例数据：实例的字段属性等元信息
* 对齐填充：为8字节的整数倍。没什么特别含义。

## 判断对象是否死亡

1. 可达性算法：以GC Roots为起点，标记可用对象。
   > 哪些对象可以作为 GC Roots 呢？
   > * 虚拟机栈(栈帧中的本地变量表)中引用的对象
   > * 本地方法栈(Native 方法)中引用的对象
   > * 方法区中类静态属性引用的对象
   > * 方法区中常量引用的对象
   > * 所有被同步锁持有的对象
   > * JNI（Java Native Interface）引用的对象
    
2. 引用计数法：无法解决循环引用问题

## 引用类型：

1. 强：大部分创建的普通对象
2. 软：内存不够时，会考虑回收。缓存
3. 弱：看到即回收。ThreadLocal的弱引用。
4. 虚：创建即回收。虚引用主要用来跟踪对象被垃圾回收的活动。管理堆外内存

弱引用和虚引用一般实际工作中应用的不多。

## 垃圾回收算法

1. 标记-清除：基础回收算法。最大的缺点是会产生内存碎片。 优点是对标记-清除算法的改进。
2. 复制：不会产生内存碎片，但是会占用双倍的内存。多运用于对象迭代较高的年轻代Survivor。
3. 标记-整理：不会产生内存碎片，适用于老年代垃圾回收频率不高的场景。

## 双亲委派机制

> 类加载器会将加载查找类或资源的人物优先委托给父类加载器。

### 为什么要这样做？

Java官方推荐使用此种模式。其中一个重要的优点，是防止核心类库被篡改。

### 什么情况下会打破这种模式？怎么做？

当你需要不通过父类加载器的方式，可以直接加载资源，比如`Tomcat`和`SkyWalking`就是这么干的。 这种方式主要是为了考虑优先加载指定路径的资源，需要自定义类加载器，重写父类加载器的`loadClass()`方法。

## 参考资源
* Guide哥：[类加载器详解](https://javaguide.cn/java/jvm/classloader.html)