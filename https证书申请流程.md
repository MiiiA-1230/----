# https证书申请流程

申请地址：https://certbot.eff.org

环境准备：

- 一台centos 7 LInux操作系统主机，并且能够`正常上网`；
- `真实域名`。

![image-20240318184811190](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/image-20240318184811190.png)

![image-20240318184845187](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/image-20240318184845187.png)

![image-20240318184912074](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/image-20240318184912074.png)

```bash
[root@localhost ~]# vim /etc/yum.repos.d/nginx.repo
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[root@localhost ~]# yum install -y epel-release
[root@localhost ~]# yum makecache fast
[root@localhost ~]# yum install -y snapd nginx
[root@localhost ~]# systemctl enable --now snapd
[root@localhost ~]# ln -s /var/lib/snapd/snap /snap
[root@localhost ~]# snap --help
The snap command lets you install, configure, refresh and remove snaps.
Snaps are packages that work across many different Linux distributions,
enabling secure delivery and operation of the latest apps and utilities.

Usage: snap <command> [<options>...]

Commonly used commands can be classified as follows:

# 安装Certbot软件包，并使用了--classic标志(经典模式)
[root@localhost ~]# snap install --classic certbot
Warning: /var/lib/snapd/snap/bin was not found in your $PATH. If you've not restarted your session
         since you installed snapd, try doing that. Please see https://forum.snapcraft.io/t/9469
         for more details.

certbot 2.9.0 from Certbot Project (certbot-eff✓) installed

[root@localhost ~]# ln -s /snap/bin/certbot /usr/bin/certbot

[root@localhost ~]# certbot certonly --manual --preferred-challenges dns -d *.tanke.love

# certbot certonly --manual --preferred-challenges dns -d *.tanke.love 这条命令用于使用Certbot工具获取SSL/TLS证书。
# 具体解释如下：
certbot: 是Certbot的命令行工具，用于管理SSL/TLS证书。
certonly: 表示只获取证书而不配置自动更新和续订。
--manual: 指定手动验证模式，即通过手动创建验证文件来证明域名所有权。
--preferred-challenges dns: 指定首选的验证方式为DNS（DNS challenge），即通过在DNS记录中添加特定的TXT记录来证明域名所有权。
-d *.tanke.love: 指定要获取证书的域名，这里包括tanke.love和以*.tanke.love开头的所有子域名。
综上所述，该命令的作用是使用Certbot工具，通过手动验证模式和DNS挑战方式，为tanke.love和以*.tanke.love开头的所有子域名获取SSL/TLS证书。
```

![image-20240318185145827](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/image-20240318185145827.png)

> 之后一直按两次回车

![image-20240318185406394](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/image-20240318185406394.png)

登录购买域名的云厂商

![image-20240318185524734](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/image-20240318185524734.png)

![报错](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/image-20240318185310981.png)

> 如果出现上述报错则重新生成证书：
>
> `certbot certonly --manual --preferred-challenges dns -d tanke.love,*.tanke.love -v`

![image-20240318185643750](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/image-20240318185643750.png)

> 证书、私钥存放处



测试：

```bash
#启动nginx
[root@localhost ~]# systemctl  enable --now nginx

#修改配置文件
[root@localhost ~]# vim /etc/nginx/conf.d/default.conf 

server {
    listen       80;
    server_name  test.tanke.love;
    location / {
      rewrite ^(.*)$  https://test.tanke.love$1 permanent;
    }
}

server {
    listen     443 ssl;
    server_name test.tanke.love;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_certificate     /etc/letsencrypt/live/tanke.love/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tanke.love/privkey.pem;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}


# 检查语法
[root@localhost ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

# 重启nginx
[root@localhost ~]# systemctl restart nginx
```

访问nginx

![image-20240318190036735](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/image-20240318190036735.png)