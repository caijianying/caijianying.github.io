

> 本次是采用**JDK11**的**JavaMail**发送邮件，邮件服务器为：**腾讯企业邮箱**,采用**SMTP**协议

### 现象描述

同样的发送邮件的代码，本地服务启动能正常发送邮件，部署到阿里云服务器无法发送。报错信息为：`Could not connect to SMTP host: smtp.exmail.qq.com, port: 25`

* 第1次尝试解决
  > 网上搜到 [阿里云服务器smtp.exmail.qq.com:25端口访问超时的解决办法](https://www.txcstx.com/post/1206.html),哦原来是25端口被拉黑了。

  我于是乎做了3个操作：
    1. 云服务器的安全组上添加出站规则，放开465端口
    2. 发送邮件的代码中，指定465作为发送邮件端口。先看JavaMail原有的属性配置
        ```java
             // 配置属性
            Properties props = new Properties();
            props.put("mail.smtp.host", host);
            props.put("mail.smtp.auth", "true");
            props.put("mail.smtp.port", port);
            props.put("mail.smtp.ssl.enable", "true");
        ```
       > 这里其实有坑的，我之前使用Integer作为`port`的数据类型，发送邮件其实还是使用默认的25端口进行发送，所以报错信息提示是25端口出错。后来改成String才生效。
    3. 由于我的服务器是`海外服务器`，所以使用的host也要修改，改为 `hwsmtp.exmail.qq.com`。

  **这么改了以后，发现还是有报错，这次报错信息不同了。看来改动生效了！**

  报错信息为：`Could not connect to SMTP host:hwsmtp.exmail.qq.com, port: 465`

* 第2次尝试解决
  > 发现在服务器上，telnet网络是通的，但容器里连接不上。

  于是乎，我做了2个操作：
    1. `docker-compose.yml`增加暴露463端口
        ```yaml
              ports:
                - "8888:8888"
                - "463:463"
        ```
    2. 防火墙将463端口放开
        ```shell
          sudo ufw allow 463/tcp
        ```
  **这么改了以后，发现虽然不报错了，但是发送邮件依旧没发成功，也没任何报错的日志打印，好像执行了一个空方法。**

* 第3次尝试解决
  >后来网上查了资料，具体哪篇已经找不到了，还是代码里的属性设置有点问题。修改如下
  ```java
    // 配置属性
    Properties props = new Properties();
    props.put("mail.smtp.host", host);
    props.put("mail.smtp.socketFactory.port", port);
    props.put("mail.smtp.socketFactory.class", "javax.net.ssl.SSLSocketFactory");
    props.put("mail.smtp.auth", "true");
    props.put("mail.smtp.port", port);
    props.put("mail.smtp.ssl.enable", "true");
  ```
  **将新代码更新上去后，发现可以发送成功！不得不说，还是走了点弯路的。**