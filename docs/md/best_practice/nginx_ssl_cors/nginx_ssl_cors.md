## 服务器

> Ubuntu 22.04
> 
> 这里假设域名解析已完成。需要准备域名`example.com`,并使用`api.example.com`作为二级域名

## 安装Nginx

```shell
## 更新软件包
sudo apt update

## 安装nginx
sudo apt install nginx

## 检查是否成功安装
systemctl status nginx

## 配置防火墙
sudo ufw allow 'Nginx HTTP'
sudo ufw allow 'Nginx HTTPS'

## 查看nginx.conf
 cat /etc/nginx/nginx.conf
```

## 配置SSL证书

> 一般下载下来的SSL证书是，`example.com.key`和`example.com.pem`两个文件

1. 在`/etc/nginx/nginx.conf`中的`http`中配置`server`

```shell

## 此处省略自动生成的配置

http {

        ## 此处省略自动生成的配置
        
        server { 
                listen 443 ssl;
                server_name api.example.com;
                ssl_certificate /etc/nginx/certs/example.com.pem;
                ssl_certificate_key /etc/nginx/certs/example.com.key;
        
                location / { 
                        root /var/www/api.example.com; 
                        index index.html;
                }
         }
}
```

2. 开放443端口
> 云服务器安全组开放443端口
> 
> 防火墙开放443端口
    ```sudo ufw allow 13306/tcp```

## 配置跨域访问
> 配置跨域访问是为了解决前端调用接口发生`Cors Error`的问题。Nginx代理配置跨域访问是其中的一种解决方式。
> 
> nginx.conf补充后，如下

```shell

## 此处省略自动生成的配置

http {

        ## 此处省略自动生成的配置
        
        server { 
                listen 443 ssl;
                server_name api.example.com;
                ssl_certificate /etc/nginx/certs/example.com.pem;
                ssl_certificate_key /etc/nginx/certs/example.com.key;
        
                location / { 
                        root /var/www/api.example.com; 
                        index index.html;
                }
                
                location /api/ {
                   if ($request_method = OPTIONS ) {
                        add_header 'Access-Control-Allow-Origin' '*';
                        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
                        add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization';
                        add_header 'Access-Control-Max-Age' 1728000;
                        add_header 'Content-Type' 'application/json charset=UTF-8';
                        add_header 'Content-Length' 0;
                        return 204;
                    }
                
                    proxy_pass http://localhost:8888/api/;
                    proxy_set_header Host $host;
                    proxy_set_header X-Real-IP $remote_addr;
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header X-Forwarded-Proto $scheme;
                
                    # 添加 CORS 头部
                    add_header 'Access-Control-Allow-Origin' '*' always;
                    add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS' always;
                    add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization' always;
                    add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range' always;
                 }
         }
}
```