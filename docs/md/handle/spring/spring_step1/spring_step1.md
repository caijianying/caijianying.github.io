## 使用Spring

>我们使用基于注解的`AnnotationConfigApplicationContext`作为容器

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        UserService userService = (UserService)applicationContext.getBean("userService");
        userService.sayHello();
    }

⏰ 注意: `AppConfig`作为一个配置类，会使用Spring的扫描注解`@ComponentScan`。而`UserService`会使用`@Component`。  

## 创建测试类和注解
如下，这是我的项目结构
    
    |-XXX
      |-src
        |-main
            |-java
                |-com.xx.xx
                    |-annotations
                        |-@ComponentScan
                        |-@Component
                    |-AnnotationConfigApplicationContext
        |-test
            |-java
                |-com.xx.xx
                    |-config
                        |-AppConfig
                    |-bean
                        |-UserService
                    |-AppMain （测试类）

* AnnotationConfigApplicationContext

        
    public class AnnotationConfigApplicationContext {
    
        private Class configClass;

        public AnnotationConfigApplicationContext(Class clazz) {
            this.configClass = clazz;
        }

    }

* AppConfig

    
    @ComponentScan(basePackages = "com.xiaobaicai.handle.spring.process")
    public class AppConfig {


    }

* UserService

      
    @Component("userService")
    public class UserService {
        public void sayHello() {
          System.out.println("Hello!");
        }
    }