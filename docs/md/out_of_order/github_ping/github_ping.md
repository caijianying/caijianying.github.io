## 背景
不知你是否遇到过这样的情况：网页上能浏览`github`，速度还贼快，可本地代码克隆，代码更新，远程推送都无法进行。

具体表现的错误也非常具有迷惑性，
```shell
	Auto fetch failed
    Connection closed by 127.0.0.1 port 7890
    Could not read from remote repository.
    Please make sure you have the correct access rights
    and the repository exists.
```
我就被这错误带跑偏了，以为`github`上`ssh key`的问题。

## 没Ping通?
后来一顿操作验证发现是`github`没highLight(ping)通导致的。

## 查找映射IP
常用的IP查找网站有两个，
* https://www.ipaddress.com
* https://ip138.com/

查找`github.com`和`github.global.ssl.fastly.net`映射的IP。

就解决这个错误而言，`ipaddress`上查找到的是可ping通的IP，`ip138`上查找结果是所有IP。用`ipaddress`省略了筛选的步骤。

## 修改Hosts文件
> mac路径为 `/etc/hosts`
```shell
140.82.113.3   github.com
199.232.5.194  github.global.ssl.fastly.net
```