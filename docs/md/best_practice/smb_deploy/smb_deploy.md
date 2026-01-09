# 介绍
在linux上安装SMB服务，用于内部局域网的文件资源共享，类似于网上邻居

# 运行环境
Almalinux 或者 CentOS

# 安装
## 安装Samba软件包
```shell
# 安装Samba和依赖
sudo dnf install -y samba samba-client samba-common

# 检查安装版本
smbd --version
```
## 创建共享目录并设置权限
```shell
# 创建共享目录
sudo mkdir -p /srv/samba/share
sudo chmod -R 0777 /srv/samba/share
sudo chown -R nobody:nobody /srv/samba/share

# 创建测试文件
echo "This is a test file for anonymous Samba share" | sudo tee /srv/samba/share/test.txt
```
## 配置Samba主配置文件
1. 备份原有配置  `sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak`
2. 编辑配置文件,替换为以下内容（匿名共享配置）
    ```
    [global]
        workgroup = WORKGROUP
        server string = Samba Server %v
        netbios name = centos-samba
        security = user
        map to guest = Bad User
        guest account = nobody
        log file = /var/log/samba/log.%m
        max log size = 50
        dns proxy = no
    
    # 匿名共享配置
    [AnonymousShare]
        path = /srv/samba/share
        browseable = yes
        writable = yes
        guest ok = yes
        guest only = yes
        read only = no
        force user = nobody
        force group = nobody
        create mask = 0777
        directory mask = 0777   
    ```

## 配置SELinux（如果启用）

```shell
# 检查SELinux状态
sudo getenforce

# 如果SELinux是Enforcing模式，设置相关标签
sudo semanage fcontext -a -t samba_share_t "/srv/samba/share(/.*)?"
sudo restorecon -Rv /srv/samba/share

# 或者临时允许Samba访问（重启后失效）
sudo setsebool -P samba_export_all_rw on
```

## 配置防火墙

```shell
# 允许Samba服务通过防火墙
sudo firewall-cmd --permanent --add-service=samba
sudo firewall-cmd --reload

# 或者手动开放端口
sudo firewall-cmd --permanent --add-port={139/tcp,445/tcp,137/udp,138/udp}
sudo firewall-cmd --reload
```
## 启动并启用Samba服务

```shell
# 检查配置语法
sudo testparm

# 启动服务
sudo systemctl start smb
sudo systemctl start nmb

# 设置开机自启
sudo systemctl enable smb
sudo systemctl enable nmb

# 检查服务状态
sudo systemctl status smb
sudo systemctl status nmb
```
## 测试配置

```shell
# 本地测试
sudo smbclient -L localhost -N

# 从其他Windows/Linux客户端访问测试
# Windows: \\服务器IP\AnonymousShare
# Linux: smbclient //服务器IP/AnonymousShare -N

# 创建测试文件验证写入权限
sudo touch /srv/samba/share/test_write.txt
```

## 怎样排查
```shell
# 查看Samba日志
sudo tail -f /var/log/samba/log.smbd

# 查看详细的连接日志
sudo tail -f /var/log/samba/log.<客户端IP>

# 重新加载配置（无需重启服务）
sudo smbcontrol all reload-config

# 检查SELinux审计日志
sudo ausearch -m avc -ts recent | grep samba
```
## 远程访问

### Windows客户端

文件资源管理器地址栏输入：`\\你的服务器IP` 或直接访问：`\\你的服务器IP\AnonymousShare`

### Linux

1. 安装客户端
`sudo dnf install -y samba-client cifs-utils`
2. 挂载共享
`sudo mount -t cifs //服务器IP/AnonymousShare /mnt -o guest`

### macOS客户端：

1. Finder中按 Command+K
2. 输入：`smb://服务器IP/AnonymousShare`