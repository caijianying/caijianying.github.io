## 先说String的intern作用

* 如果字符串池子中已经存在该字符串对象，就返回这个字符串对象
* 如果字符串池子中不存在该字符串对象，则将字符串添加到字符串池中，并返回新添加字符串的对象引用

思想跟`map.putIfAbsent`一样，没有则创建，有也不会覆盖

总之，从intern方法中获取到的是字符串池子中的对象引用

## intern的理解

比较一下这段代码在JDK11中的执行过程，可以加深理解

```java
public static void main(String[]args){

        String s1=new String("a");
        s1.intern();
        String s2="a";
        System.out.println(s1==s2);

        String s3=new String("a")+new String("a");
        s3.intern();
        String s4="aa";
        System.out.println(s3==s4);

        }
```

最后结果为

```shell
false 
true 
```

### 执行过程

#### s1与s2

* 字符串池有一个字符串"a"，s1指向【在运行时常量池中"a"的对象引用】
* `s1.intern();`由于'a'已存在，所以啥事也没发生
* 接下来，s2指向字符串池中的"a"引用

**s1和s2指向不同，所以为false**

#### s3与s4

* s3指向【在运行时常量池中"aa"的对象引用】，即 s3 -> 运行时常量池中"aa"
* `s3.intern();`由于字符串池子中没有"aa"，则将"aa"添加进字符串池子中。 此时 s3 -> 运行时常量池中"aa" -> 字符串池中的"aa"，即 s3 -> 字符串池中的"aa"

* s4 -> 字符串池中的"aa"

**s3和s4都指向字符串池子中的"aa"，所以为true**

#### 验证

> 为了验证【s3与s4】的推断，可以将`s3.intern();`注释掉，执行后结果将会是false.

## 最后

```java
        String s3=new String("1")+new String("1");
        s3.intern();
        String s4="11";
        System.out.println(s3==s4);
```

关于将s3和s4的例子从aa改成11，在**JDK 11**环境运行下结果会为false,这里解释一下原因.

执行测试代码前，JDK11源码中已经将"11"添加到字符串池中，导致`s3.intern();`不再起作用。

源码位置为：`com.sun.tools.javac.code.Source`