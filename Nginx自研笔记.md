# Nginx

## 零碎知识

编译安装nginx的部分命令

```
nginx -c /path/nginx.conf  	     # 以特定目录下的配置文件启动nginx:
nginx -s reload            	 	 # 修改配置后重新加载生效
nginx -s stop  				 	 # 快速停止nginx
nginx							 # 快速启动nginx
nginx -t    					 # 测试当前配置文件是否正确
nginx -t -c /path/to/nginx.conf  # 测试特定的nginx配置文件是否正确
```

##### 平滑升级参考笔记nginx-day3

```
nginx -s reload    重载配置

ln -s /usr/local/nginx/sbin/nginx /bin/nginx    制作软连接以方便使用

root和alias区别：root后加访问目录，alias等于访问目录
```

------

## 一、编译安装

1、安装编译环境、pcre软件包（使nginx支持http rewrite模块）、openssl-devel（使nginx支持ssl）、zlib

```
yum -y install gcc gcc-c++

yum install -y pcre pcre-devel

yum install -y openssl openssl-devel

yum install -y zlib zlib-devel
```

2、创建用户nginx

```
useradd nginx
```

3、解压并安装nginx

```
tar xzf nginx-1.16.0.tar.gz -C /usr/local/

cd /usr/local/nginx-1.16.0/

./configure --prefix=/usr/local/nginx --group=nginx --user=nginx --sbin-path=/usr/local/nginx/sbin/nginx --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --http-client-body-temp-path=/tmp/nginx/client_body --http-proxy-temp-path=/tmp/nginx/proxy --http-fastcgi-temp-path=/tmp/nginx/fastcgi --pid-path=/var/run/nginx.pid --lock-path=/var/lock/nginx --with-http_stub_status_module --with-http_ssl_module --with-http_gzip_static_module --with-pcre --with-http_realip_module --with-stream

make && make install
```

4、修改配置文件/etc/nginx/nginx.conf

```
# 全局参数设置
user root;         ############user www-data; 否则nginx没有权限访问静态资源文件
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 1024;
        # multi_accept on;
}

http {
        sendfile on;
        tcp_nopush on;
        types_hash_max_size 2048;
        default_type application/octet-stream;
        include /etc/nginx/mime.types;

        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        gzip on;
        include /etc/nginx/conf.d/*.conf;
        #include /etc/nginx/sites-enabled/*; #######################此文件为nginx默认的监听80端口的文件，给注释掉 ,否则访问网站时不会把消息转发给下方配置的uwsgi端口
        #######################以下为增加的内容##############
        server {
                listen        80 default_server;    #监听80端口
                server_name        Homeowner;
                charset      utf-8;

                client_max_body_size 75M;

                location / {
                        include   uwsgi_params;
                        uwsgi_pass     10.0.16.15:8888;  #将信息转发给8080端口的uwsgi，和uwsgi.ini配置文件中的端口需要保持一致
                        uwsgi_read_timeout    600;
                        uwsgi_send_timeout    600;
                        uwsgi_connect_timeout 600;
                }


                #路径为/static的请求，直接从根目录的static文件夹中获取静态文件
                location ~/static/ {
                        root /usr/local/nginx/html;
                }



                #路径为/media 的请求直接从根目录的media文件夹中获取静态文件（指django媒体文件-media）
                #location /img {
                #       alias /root/www/homeowner/Homeowner/HomeownerEnv/lib/python3.11/site-packages/django/contrib/admin/static/img;
                #}
        }
}

```

5、检测nginx配置文件是否正确

```
mkdir -p /tmp/nginx

mkdir /usr/local/nginx/logs

/usr/local/nginx/sbin/nginx -t
```

5.5、制作软连接方便使用

```
ln -s /usr/local/nginx/sbin/nginx /bin/nginx
```

## 二、反向代理

（实际样例参考tools/nginx）

1、代理机设置

```
server {
listen 80；
server_name 192.168.58.100;
	location / {
		proxy_pass 192.168.58.131;
	}
}
```

2、被代理服务器设置

```
server {
listen 80;
server_name 192.168.58.131;
	location / {
		root /miiia;
		index index.html;
	}
}
```

## 三、负载均衡

### 如何配置

首先给大家说下 upstream 这个配置的，这个配置是写一组被代理的服务器地址，然后配置负载均衡的算法。这里的被代理服务器地址有2种写法。

```http
upstream youngfitapp { 
      server 192.168.62.157:8080;
      server 192.168.62.158:8080;
}
server {
        listen 80;
        server_name localhost;
        location / {         
           proxy_pass  http://youngfitapp;
        }
}
```



### 配置实例

1、热备：如果你有2台服务器，当一台服务器发生事故时，才启用第二台服务器给提供服务。服务器处理请求的顺序：AAAAAA突然A挂啦，BBBBBBBBBBBBBB.....

```shell
upstream myweb { 
      server 192.168.62.157:8080; 
      server 192.168.62.158:8080 backup;  #热备     
}
```

2、轮询：Nginx默认就是轮询其权重都默认为1，服务器处理请求的顺序：ABABABABAB....

```shell
upstream myweb {
      server 192.168.62.157:8080; 
      server 192.168.62.158:8080;      
}
```

3、加权轮询：根据配置的权重的大小而分发给不同服务器不同数量的请求。如果不设置，则默认为1。下面服务器的请求顺序为：ABBABBABBABBABB....

```shell
upstream myweb { 
      server 192.168.62.157:8080 weight=1;
      server 192.168.62.158:8080 weight=2;
}
```

4、ip_hash：Nginx会让相同的客户端ip请求相同的服务器。

```shell
upstream myweb {
	  ip_hash;
      server 192.168.62.157:8080; 
      server 192.168.62.158:8080;   
}
```

5、nginx负载均衡配置状态参数

- down，表示当前的server暂时不参与负载均衡。
- backup，预留的备份机器。当其他所有的非backup机器出现故障或者忙的时候，才会请求backup机器，因此这台机器的压力最轻。
- max_fails，允许请求失败的次数，默认为1。当超过最大次数时，返回错误。
- fail_timeout，在经历了max_fails次失败后，暂停服务的时间单位秒。max_fails可以和fail_timeout一起使用。

```shell
 upstream myweb { 
      server 192.168.62.157:8080 weight=2 max_fails=2 fail_timeout=2;
      server 192.168.62.158:8080 weight=1 max_fails=2 fail_timeout=1;   
 }
```

nginx配置7层协议及4层协议方法（扩展，参考笔记nginx-day2）

## 四、会话保持

##### 1、ip_hash**

ip_hash使用源地址哈希算法，将同一客户端的请求总是发往同一个后端服务器，除非该服务器不可用。

ip_hash语法：

```http
upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com down;
}
```

ip_hash简单易用，但有如下问题：
当后端服务器宕机后，session会话丢失；
同一客户端会被转发到同一个后端服务器，可能导致负载失衡；

##### 2、sticky_cookie_insert

使用sticky_cookie_insert启用会话亲缘关系，这会导致来自同一客户端的请求被传递到一组服务器的同一台服务器。与ip_hash不同之处在于，它不是基于IP来判断客户端的，而是基于cookie来判断。因此可以避免上述ip_hash中来自同一客户端导致负载失衡的情况。(需要引入第三方模块才能实现)

sticky模块

语法：

```http
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    sticky_cookie_insert srv_id expires=1h domain=3evip.cn path=/;
}

server {
    listen 80;
    server_name 3evip.cn;
    location / {
		proxy_pass http://backend;
    }
}
```

说明：
expires：设置浏览器中保持cookie的时间
domain：定义cookie的域
path：为cookie定义路径

##### 3.jvm_route

jvm_route的原理

1. 一开始请求过来，没有带session信息，jvm_route就根据轮询（round robin）的方法，发到一台tomcat上面。

2. tomcat添加上session 信息，并返回给客户。

3. 用户再此请求，jvm_route看到session中有后端服务器的名称，它就把请求转到对应的服务器上。

## 五、动静分离

只写了代理服务器配置，web服务器配置参考笔记nginx-day2

```
upstream static {
        server 192.168.62.155:80 weight=1 max_fails=1 fail_timeout=60s;
}
upstream phpserver {
        server 192.168.62.157:80 weight=1 max_fails=1 fail_timeout=60s;
}
     server {
        listen      80;
        server_name     localhost;
        #动态资源加载
        location ~ \.(php|jsp)$ {
            proxy_pass http://phpserver;
            proxy_set_header Host $host:$server_port;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                }
        #静态资源加载
        location ~ .*\.(html|gif|jpg|png|bmp|swf|css|js)$ {
            proxy_pass http://static;
            proxy_set_header Host $host:$server_port;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                }
        }
```

## 六、防盗链

**配置要点：**（更多参考在笔记nginx-day2）

```shell
server {
    listen       80;
    server_name  localhost;
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;

        valid_referers none blocked www.jd.com;  #允许这些访问
                if ($invalid_referer) {
                   return 403;
                }
        }
}
- none : 允许没有http_referer的请求访问资源；

- blocked : 允许不是http://开头的，不带协议的请求访问资源---被防火墙过滤掉的；

- server_names : 只允许指定ip/域名来的请求访问资源（白名单）；
```

## 七、地址重写

此为代表性例子，更多需参照笔记nginx-day2

```
server {
    listen       80;
    server_name  www.testpm.com;

        location /a {
        root /html;
        index   1.html index.htm;
        rewrite .* /b/2.html permanent;
        }

        location /b {
        root    /html;
        index   2.html index.htm;
        }
}
```

## 八、流量控制

主要是两条命令：

```
limit_req_zone;
limit_req;

样例：
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=1r/s;
upstream myweb {
        server 192.168.62.157:80 weight=1 max_fails=1 fail_timeout=1;
        }
server {
        listen 80;
        server_name localhost;

        location /login {
                limit_req zone=mylimit;
                proxy_pass http://myweb;
                proxy_set_header Host $host:$server_port;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                }
}
```

## 九、访问控制

#### 1、基于IP进行控制

| 白名单              | 黑名单              |
| ------------------- | ------------------- |
| allow 192.168.58.10 | deny  192.168.58.10 |
| deny all            | allow all           |



#### 2、基于用户信任进行控制

```
server {
	listen 80;
	server_name localhost;
	location ~ ^/admin {
		root /home/www/html;
		index index.html index.hml;
		auth_basic "Auth access test!";
		#↑弹窗后的提示信息
		auth_basic_user_file /etc/nginx/auth_conf;
		#↑校验密码的口令文件在哪里
		}
}
```

建立口令文件

```
htpasswd -cm /etc/nginx/auth_conf user10
#-cm 第一次创建，-m 追加，否则会覆盖原文件
```

## 十、Nginx调优

##### **1、修改配置文件**：

- 打开Nginx的配置文件，通常是`nginx.conf`。
- 通过修改配置参数来优化性能。

##### **2、增加工作进程数**：

- 修改`worker_processes`参数，设置为服务器CPU核心数的倍数，以充分利用服务器资源。

##### **3、调整连接数限制**：

- 增加`worker_connections`参数，以支持更多的负载连接。可以根据服务器的硬件配置和预期的负载来设置这个值。

##### **4、启用服务器**：

- 启用Nginx的服务器功能，可以减轻服务器的压力，提高响应速度。使用`proxy_cache`或`fastcgi_cache`指令来配置服务器。

##### **5、实现压缩**：

- 启用Nginx的gzip压缩功能，可以减少传输的数据量，提高网站的加载速度。使用`gzip`指令来配置压缩参数。

##### **6、调整日志级别**：

- 将日志级别调整为合适的级别，避免产生过多的日志，减少磁盘IO开销。

##### **7、启用TCP连接复用**：

- 启用TCP连接复用，可以减少TCP连接的建立和中断次数，提高性能。使用`reuseport`参数来启用TCP连接复用。

##### **8、限制缓冲区大小**：

- 调整限制内存缓存大小，避免亮度过大导致内存占用过高。可以通过`client_body_buffer_size`和`client_max_body_size`来进行限制。

##### **9、启用HTTP/2**：

- 如果HTTP/2协议的客户端地址支持，可以启用Nginx的HTTP/2支持，以提高网站的加载速度和性能。

##### **10、监控和优化**：

- 使用监控工具来监视Nginx的性能指标，及时发现并解决性能瓶颈。

在进行任何调整优化操作之前，请务必备份好Nginx的配置文件，以防发生意外情况。并且在调整配置参数时，注意逐步测试和观察性能变化，避免造成不必要的影响。
