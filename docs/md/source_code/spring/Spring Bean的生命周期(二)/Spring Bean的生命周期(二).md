上文曾多次提到刷新容器的入口方法`AbstractApplicationContext#refresh`，
`AbstractApplicationContext`是一个十分重要的类，
它通过实现`org.springframework.context.ConfigurableApplicationContext`来提供容器上下文的通用配置。
因此几乎所有的Spring容器类都会去继承`AbstractApplicationContext`，从而快速具备Spring容器的一些通用操作。
而`refresh`就是对容器的通用操作之一，是刷新容器上下文的意思，
用于加载或刷新配置信息，比如XML方式、基于Java配置的方式、properties文件的方式等。

>执行完**注册BeanDefinition**后，大多数Spring容器类都会去调用`refresh`方法完成**容器的自动刷新**。

接下来我们来细看`refresh`中的主要实现步骤。

⏰ 注意：基于SpringBoot的演示项目，源码中有些判断下的方法并不会执行，这里只梳理出通常情况下执行的方法

## prepareRefresh
执行上下文的前置准备，包括记录开始时间和激活关闭标志以及属性资源的初始化

## obtainFreshBeanFactory
返回执行`refresh`操作的`ConfigurableListableBeanFactory`,这里是默认的实现类`DefaultListableBeanFactory`

## prepareBeanFactory
准备BeanFactory的标准上下文特性，比如类加载器和后置处理器。
1. 设置类加载器
2. 注册`ApplicationContextAwareProcessor`(是一个BeanPostProcessor)，用于aware回调。
3. 设置依赖接口忽略：`ignoreDependencyInterface`
4. 注册特殊的依赖类型，如`BeanFactory`、`ResourceLoader`、`ApplicationEventPublisher`、`ApplicationContext`
5. 注册`ApplicationListenerDetector`(是一个BeanPostProcessor)

## postProcessBeanFactory
请注意，这个方法并非什么都没做。
我们看它实际是它的子类在执行
![img.png](postProcessBeanFactory.png) 
从执行情况来看，虽然看似啥都没做，但是还是做了**扫包和手动注册**的判断逻辑的。

## invokeBeanFactoryPostProcessors
>这一步想必记忆还是相当深刻的。就在这一步，执行了`BeanFactoryPostProcessor`对配置类的`BeanDefinition`进行CGLIB的增强。
实例化并调用所有已注册的`BeanFactoryPostProcessors`，按给定的顺序或默认顺序执行。

1. 执行所有的`BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry`，
   执行优先级为：@PriorityOrdered > @Ordered > 其他
2. 执行所有的`BeanFactoryPostProcessor#postProcessBeanFactory`,执行优先级同上

>1. `ConfigurationClassPostProcessor`先执行 `postProcessBeanDefinitionRegistry` 进行**BeanDefinition注册**，调用链路为：
> `processConfigBeanDefinitions` => `this.reader.loadBeanDefinitions(configClasses);` => `ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsFromRegistrars`
> 2. 再执行`postProcessBeanFactory`，**对注册的BeanDefinition进行CGLIB增强**，调用链路为：
> `enhanceConfigurationClasses` => CGLIB代理增强(代码片段): `Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);`
