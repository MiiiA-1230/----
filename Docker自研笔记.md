# Docker自研笔记

### 一、对Docker的理解

##### 1、概念

类似于应用商店，大家把项目打包成软件放到应用商店中，当其他人有需要时可以直接下载并使用且与环境独立，做到互不冲突。

可以将程序及其依赖的包和环境一起打包成一个镜像，可以迁移至任意的Linux内核的运行环境中，即可以将任何独立项目都可以快速的部署在任意的Linux内核的系统中并快速启动。

容器化使得程序不再对系统存在依赖，更新可以推送到分布式应用程序的任何部分，资源密度可以被优化。

##### 2、与虚拟机的区别

|   特性   |  Linux虚拟机   |     Docker     |
| :------: | :------------: | :------------: |
| 部署大小 | 硬盘占用空间大 | 硬盘占用空间小 |
| 启动速度 |  服务启动缓慢  |  服务启动迅速  |
| 运行性能 |    接近原生    |    性能较差    |

##### 3、镜像与容器的区别

镜像：Docker将应用程序及其所需的依赖、函数库、环境、配置等文件打包在一起，称为镜像

容器：镜像中的应用程序运行后形成的进程就是一个容器，只是Docker会给容器做好隔离

##### 4、Docker主要架构

**主要操作流程**

Client → sever → daemon守护进程 → 容器

**Docker架构**

1、服务端：Docker守护进程，负责处理接收到的Docker指令，管理镜像、容器

2、客户端：通过命令或者RestAPI向Docker服务端发送指令，可以在本地或者远程主机上对服务端发送

docker.service为docker主要进程

docker.socket为保证docker进行通信执行的进程，服务端通过docker.socket接收请求，客户端通过docker.socket连接Docker CLI工具

Docker CLI工具主要作用有**容器管理、镜像管理、网络管理、数据卷管理、Compose集成**（前三个不用解释，第四个是用于在容器间共享数据的机制；第五个是管理多容器组成的应用）

##### 扩展

RestAPI是前后端分离的最佳实践，是开发的一套标准或者说一套规范，不是框架。

优点：

1、轻量化，直接通过http，不需要额外的协议，通常通过post、get、put、delete操作

2、面向资源，一目了然，具有自解释性

3、数据描述简单，一般通过json或者xml做数据通信



### 二、Docker安装流程

##### 1、卸载旧版本

```
yum remove docker \
					docker-client \
					docker-client-latest \
					docker-common \
					docker-latest \
					docker-latest-logrotate \
					docker-logrotate \
					docker-engine
```

##### 2、安装依赖工具

```
yum -y install yum-utils
```

##### 3、更新yum工具

```
yum makecache fast
```

##### 4、安装docker

```
yum -y install docker-ce docker-ce-cli containerd.io
```

##### 5、设置docker仓库

```
yum-config-manager     --add-repo     https://download.docker.com/linux/centos/docker-ce.repo
```

##### 6、验证Docker是否正确安装

```
systemctl start docker
docker version
#可以正常打印版本号即为安装成功
docker run hello-world
```

##### 7、提示找不到软件包：添加阿里云镜像

```
yum-config-manager 	--add-repo 	http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```



### 三、Docker简单命令

##### 运行、暂停、查看、删除命令

运行：

```
docker run --name mn -p 80:80 -d nginx
#docker run：创建并运行一个容器，--name：容器名，-p：端口映射，80:80：主机端口:容器端口，-d：后台运行，nginx：镜像名
#执行docker镜像并放入容器中
```

暂停：

```
docker stop 容器名或容器id
```

查看：

```
docker image ls
#列出docker镜像
docker container ls 
#列出docker容器（运行中）
docker container ls -all
#列出docker容器（所有的）
docker container ls -a -q
#列出docker容器（quit模式下所有的）
docker ps
#查看docker容器状态
```

创建：

```
docker build -t 镜像名 -f 配置模板文件路径 构建上下文目录
#创建镜像
docker run --name 容器名 -p 主机端口:容器端口 -d 镜像名
#创建并运行容器
```

删除：

```
docker rm -f 容器id或容器名
#加-f是可以删除运行中的容器，删除多个容器时用空格隔开
docker rmi -f 镜像名或repository:tag
#加-f是可以删除被使用的镜像，删除多个镜像时用空格隔开
docker image prune -a
#删除所有未使用的镜像
```

查看日志：

```
docker logs -f 容器名
#查看容器日志信息，加-f可以持续查看
```

##### 1、如何创建镜像

```
docker build -t 镜像名 .
#docker build用于构建docker镜像命令，-t指定镜像名称，后面跟镜像名，.表示当前目录（默认当前目录的模板名为Dockerfile）
docker build -t my-nginx-image -f /root/work/docker/模板名 .
#-f后跟指定模板的路径和构建上下文的位置 .为构建上下文目录
```

##### 2、创建一个简单的dockerfile模板

```
vim /root/work/docker

FROM nginx:latest
# 使用一个已有的基础镜像作为起点，此处意义为使用最新版nginx

COPY /etc/nginx/nginx.conf /etc/nginx/nginx.conf
# 将本地文件复制到镜像中的指定路径，路径为在构建上下文的目录内

EXPOSE 80
# 暴露容器的端口，仅提供提示开发人员作用

CMD ["nginx", "-g", "daemon off;"]
# 定义启动时执行的命令  nginx是要执行的程序，-g是指定全局配置，daemon off为全局指令，声明nginx在前台运行，而不是以守护进程方式运行
```

###### test1  第一次成功创建Nginx1.24版本的纯净镜像（配置文件nginx.conf为本地主机配置）

```
docker build -t my_nginx_image -f /root/work/docker/nginx/my_nginx /root/work/docker/nginx/
#-f后第一个地址为配置文件地址，第二个地址为构建上下文目录

vim /root/work/docker/nginx/my_nginx
# 使用一个已有的基础镜像作为起点
FROM nginx:1.24

# 将本地文件复制到镜像中的指定路径
COPY nginx.conf /etc/nginx/nginx.conf

# 暴露容器的端口
EXPOSE 80
```

###### test2  第一次成功创建Mysql5.7版本的纯净镜像

```
vim /root/work/docker/mysql/my_mysql
# 使用一个已有的基础镜像作为起点
FROM mysql:5.7

# 设置 root 用户密码
ENV MYSQL_ROOT_PASSWORD 1

# 暴露容器的端口
EXPOSE 3306


docker build -t i_nginx -f /root/work/docker/nginx/my_nginx .
#创建docker的mysql5.7镜像(忘了设置docker显示版本)

docker exec -it 3013e2258c41 mysql -uroot -p1
#登录容器内mysql代码
```

##### 3、通过拉取docker仓库中的镜像创建redis

```
第一步
FROM centos:7.9.2009 AS my_yum

RUN yum clean all && \
    yum -y update && \
    yum -y install epel-release && \
    yum -y install redis redis-tools

COPY config/redis.conf /etc/redis/redis.conf
COPY config/sentinel.conf /etc/redis/sentinel.conf

EXPOSE 6379

VOLUME /data

第二步
vim work/docker/redis/config/redis.conf
#设置主配置文件
# 指定 redis 只接收来自于该IP地址的请求，如果不进行设置，那么将处理所有请求
#bind 127.0.0.1

#是否开启保护模式，默认开启。要是配置里没有指定bind和密码。开启该参数后，redis只会本地进行访问，拒绝外部访问。要是开启了密码和bind，可以开启。否则最好关闭，设置为no
protected-mode no

#redis监听的端口号
port 6379

#此参数确定了TCP连接中已完成队列(完成三次握手之后)的长度， 当然此值必须不大于Linux系统定义的/proc/sys/net/core/somaxconn值，默认是511，而Linux的默认参数值是128。当系统并发量大并且客户端速度缓慢的时候，可以将这二个参数一起参考设定。该内核参数默认值一般是128，对于负载很大的服务程序来说大大的不够。一般会将它修改为2048或者更大。在/etc/sysctl.conf中添加:net.core.somaxconn = 2048，然后在终端中执行sysctl -ptcp-backlog 511
#此参数为设置客户端空闲超过timeout，服务端会断开连接，为0则服务端不会主动断开连接，不能小于0
timeout 0

#tcp keepalive参数。如果设置不为0，就使用配置tcp的SO_KEEPALIVE值，使用keepalive有两个好处:检测挂掉的对端。降低中间设备出问题而导致网络看似连接却已经与对端端口的问题。在Linux内核中，设置了keepalive，redis会定时给对端发送ack。检测到对端关闭需要两倍的设置值tcp-keepalive 300
#是否在后台执行，yes：后台运行；no：不是后台运行
daemonize yes

#redis的进程文件
pidfile /var/run/redis/redis.pid
  
#指定了服务端日志的级别。级别包括：debug（很多信息，方便开发、测试），verbose（许多有用的信息，但是没有debug级别信息多），notice（适当的日志级别，适合生产环境），warn（只有非常重要的信息）
loglevel notice

#指定了记录日志的文件。空字符串的话，日志会打印到标准输出设备。后台运行的redis标准输出是/dev/null
logfile /usr/local/redis/var/redis.log

#是否打开记录syslog功能
# syslog-enabled no
#syslog的标识符。
# syslog-ident redis
#日志的来源、设备
# syslog-facility local0
#数据库的数量，默认使用的数据库是0。可以通过”SELECT 【数据库序号】“命令选择一个数据库，序号从0开始
databases 16

vim work/docker/redis/config/sentinel.conf
# Example sentinel.conf

# *** IMPORTANT ***
#
# By default Sentinel will not be reachable from interfaces different than
# localhost, either use the 'bind' directive to bind to a list of network
# interfaces, or disable protected mode with "protected-mode no" by
# adding it to this configuration file.
#
# Before doing that MAKE SURE the instance is protected from the outside
# world via firewalling or other means.
#
# For example you may use one of the following:
#
# bind 127.0.0.1 192.168.1.1
#
# protected-mode no
protected-mode no

# port <sentinel-port>
# The port that this sentinel instance will run on
port 26379

# By default Redis Sentinel does not run as a daemon. Use 'yes' if you need it.
# Note that Redis will write a pid file in /var/run/redis-sentinel.pid when
# daemonized.
daemonize no

# When running daemonized, Redis Sentinel writes a pid file in
# /var/run/redis-sentinel.pid by default. You can specify a custom pid file
# location here.
pidfile "/var/run/redis-sentinel.pid"

# Specify the log file name. Also the empty string can be used to force
# Sentinel to log on the standard output. Note that if you use standard
# output for logging but daemonize, logs will be sent to /dev/null
logfile ""

# sentinel announce-ip <ip>
# sentinel announce-port <port>
#
# The above two configuration directives are useful in environments where,
# because of NAT, Sentinel is reachable from outside via a non-local address.
#
# When announce-ip is provided, the Sentinel will claim the specified IP address
# in HELLO messages used to gossip its presence, instead of auto-detecting the
# local address as it usually does.
#
# Similarly when announce-port is provided and is valid and non-zero, Sentinel
# will announce the specified TCP port.
#
# The two options don't need to be used together, if only announce-ip is
# provided, the Sentinel will announce the specified IP and the server port
# as specified by the "port" option. If only announce-port is provided, the
# Sentinel will announce the auto-detected local IP and the specified port.
#
# Example:
#
# sentinel announce-ip 1.2.3.4

# dir <working-directory>
# Every long running process should have a well-defined working directory.
# For Redis Sentinel to chdir to /tmp at startup is the simplest thing
# for the process to don't interfere with administrative tasks such as
# unmounting filesystems.
dir "/tmp"

# sentinel monitor <master-name> <ip> <redis-port> <quorum>
#
# Tells Sentinel to monitor this master, and to consider it in O_DOWN
# (Objectively Down) state only if at least <quorum> sentinels agree.
#
# Note that whatever is the ODOWN quorum, a Sentinel will require to
# be elected by the majority of the known Sentinels in order to
# start a failover, so no failover can be performed in minority.
#
# Replicas are auto-discovered, so you don't need to specify replicas in
# any way. Sentinel itself will rewrite this configuration file adding
# the replicas using additional configuration options.
# Also note that the configuration file is rewritten when a
# replica is promoted to master.
#
# Note: master name should not include special characters or spaces.
# The valid charset is A-z 0-9 and the three characters ".-_".
#sentinel monitor mymaster 127.0.0.1 6379 2
sentinel monitor mymaster 192.168.58.10 6379 2

# sentinel auth-pass <master-name> <password>
#
# Set the password to use to authenticate with the master and replicas.
# Useful if there is a password set in the Redis instances to monitor.
#
# Note that the master password is also used for replicas, so it is not
# possible to set a different password in masters and replicas instances
# if you want to be able to monitor these instances with Sentinel.
#
# However you can have Redis instances without the authentication enabled
# mixed with Redis instances requiring the authentication (as long as the
# password set is the same for all the instances requiring the password) as
# the AUTH command will have no effect in Redis instances with authentication
# switched off.
#
# Example:
#
# sentinel auth-pass mymaster MySUPER--secret-0123passw0rd

# sentinel auth-user <master-name> <username>
#
# This is useful in order to authenticate to instances having ACL capabilities,
# that is, running Redis 6.0 or greater. When just auth-pass is provided the
# Sentinel instance will authenticate to Redis using the old "AUTH <pass>"
# method. When also an username is provided, it will use "AUTH <user> <pass>".
# In the Redis servers side, the ACL to provide just minimal access to
# Sentinel instances, should be configured along the following lines:
#
#     user sentinel-user >somepassword +client +subscribe +publish \
#                        +ping +info +multi +slaveof +config +client +exec on

# sentinel down-after-milliseconds <master-name> <milliseconds>
#
# Number of milliseconds the master (or any attached replica or sentinel) should
# be unreachable (as in, not acceptable reply to PING, continuously, for the
# specified period) in order to consider it in S_DOWN state (Subjectively
# Down).
#
# Default is 30 seconds.
#sentinel down-after-milliseconds mymaster 30000
sentinel down-after-milliseconds mymaster 3000

# IMPORTANT NOTE: starting with Redis 6.2 ACL capability is supported for
# Sentinel mode, please refer to the Redis website https://redis.io/topics/acl
# for more details.

# Sentinel's ACL users are defined in the following format:
#
#   user <username> ... acl rules ...
#
# For example:
#
#   user worker +@admin +@connection ~* on >ffa9203c493aa99
#
# For more information about ACL configuration please refer to the Redis
# website at https://redis.io/topics/acl and redis server configuration
# template redis.conf.

# ACL LOG
#
# The ACL Log tracks failed commands and authentication events associated
# with ACLs. The ACL Log is useful to troubleshoot failed commands blocked
# by ACLs. The ACL Log is stored in memory. You can reclaim memory with
# ACL LOG RESET. Define the maximum entry length of the ACL Log below.
acllog-max-len 128

# Using an external ACL file
#
# Instead of configuring users here in this file, it is possible to use
# a stand-alone file just listing users. The two methods cannot be mixed:
# if you configure users here and at the same time you activate the external
# ACL file, the server will refuse to start.
#
# The format of the external ACL user file is exactly the same as the
# format that is used inside redis.conf to describe users.
#
# aclfile /etc/redis/sentinel-users.acl

# requirepass <password>
#
# You can configure Sentinel itself to require a password, however when doing
# so Sentinel will try to authenticate with the same password to all the
# other Sentinels. So you need to configure all your Sentinels in a given
# group with the same "requirepass" password. Check the following documentation
# for more info: https://redis.io/topics/sentinel
#
# IMPORTANT NOTE: starting with Redis 6.2 "requirepass" is a compatibility
# layer on top of the ACL system. The option effect will be just setting
# the password for the default user. Clients will still authenticate using
# AUTH <password> as usually, or more explicitly with AUTH default <password>
# if they follow the new protocol: both will work.
#
# New config files are advised to use separate authentication control for
# incoming connections (via ACL), and for outgoing connections (via
# sentinel-user and sentinel-pass)
#
# The requirepass is not compatable with aclfile option and the ACL LOAD
# command, these will cause requirepass to be ignored.

# sentinel sentinel-user <username>
#
# You can configure Sentinel to authenticate with other Sentinels with specific
# user name.

# sentinel sentinel-pass <password>
#
# The password for Sentinel to authenticate with other Sentinels. If sentinel-user
# is not configured, Sentinel will use 'default' user with sentinel-pass to authenticate.

# sentinel parallel-syncs <master-name> <numreplicas>
#
# How many replicas we can reconfigure to point to the new replica simultaneously
# during the failover. Use a low number if you use the replicas to serve query
# to avoid that all the replicas will be unreachable at about the same
# time while performing the synchronization with the master.

# sentinel failover-timeout <master-name> <milliseconds>

#
# Specifies the failover timeout in milliseconds. It is used in many ways:
#
# - The time needed to re-start a failover after a previous failover was
#   already tried against the same master by a given Sentinel, is two
#   times the failover timeout.
#
# - The time needed for a replica replicating to a wrong master according
#   to a Sentinel current configuration, to be forced to replicate
#   with the right master, is exactly the failover timeout (counting since
#   the moment a Sentinel detected the misconfiguration).
#
# - The time needed to cancel a failover that is already in progress but
#   did not produced any configuration change (SLAVEOF NO ONE yet not
#   acknowledged by the promoted replica).
#
# - The maximum time a failover in progress waits for all the replicas to be
#   reconfigured as replicas of the new master. However even after this time
#   the replicas will be reconfigured by the Sentinels anyway, but not with
#   the exact parallel-syncs progression as specified.
#
# Default is 3 minutes.

# SCRIPTS EXECUTION
#
# sentinel notification-script and sentinel reconfig-script are used in order
# to configure scripts that are called to notify the system administrator
# or to reconfigure clients after a failover. The scripts are executed
# with the following rules for error handling:
#
# If script exits with "1" the execution is retried later (up to a maximum
# number of times currently set to 10).
#
# If script exits with "2" (or an higher value) the script execution is
# not retried.
#
# If script terminates because it receives a signal the behavior is the same
# as exit code 1.
#
# A script has a maximum running time of 60 seconds. After this limit is
# reached the script is terminated with a SIGKILL and the execution retried.

# NOTIFICATION SCRIPT
#
# sentinel notification-script <master-name> <script-path>
#
# Call the specified notification script for any sentinel event that is
# generated in the WARNING level (for instance -sdown, -odown, and so forth).
# This script should notify the system administrator via email, SMS, or any
# other messaging system, that there is something wrong with the monitored
# Redis systems.
#
# The script is called with just two arguments: the first is the event type
# and the second the event description.
#
# The script must exist and be executable in order for sentinel to start if
# this option is provided.
#
# Example:
#
# sentinel notification-script mymaster /var/redis/notify.sh

# CLIENTS RECONFIGURATION SCRIPT
#
# sentinel client-reconfig-script <master-name> <script-path>
#
# When the master changed because of a failover a script can be called in
# order to perform application-specific tasks to notify the clients that the
# configuration has changed and the master is at a different address.
#
# The following arguments are passed to the script:
#
# <master-name> <role> <state> <from-ip> <from-port> <to-ip> <to-port>
#
# <state> is currently always "failover"
# <role> is either "leader" or "observer"
#
# The arguments from-ip, from-port, to-ip, to-port are used to communicate
# the old address of the master and the new address of the elected replica
# (now a master).
#
# This script should be resistant to multiple invocations.
#
# Example:
#
# sentinel client-reconfig-script mymaster /var/redis/reconfig.sh

# SECURITY
#
# By default SENTINEL SET will not be able to change the notification-script
# and client-reconfig-script at runtime. This avoids a trivial security issue
# where clients can set the script to anything and trigger a failover in order
# to get the program executed.

sentinel deny-scripts-reconfig yes

# REDIS COMMANDS RENAMING
#
# Sometimes the Redis server has certain commands, that are needed for Sentinel
# to work correctly, renamed to unguessable strings. This is often the case
# of CONFIG and SLAVEOF in the context of providers that provide Redis as
# a service, and don't want the customers to reconfigure the instances outside
# of the administration console.
#
# In such case it is possible to tell Sentinel to use different command names
# instead of the normal ones. For example if the master "mymaster", and the
# associated replicas, have "CONFIG" all renamed to "GUESSME", I could use:
#
# SENTINEL rename-command mymaster CONFIG GUESSME
#
# After such configuration is set, every time Sentinel would use CONFIG it will
# use GUESSME instead. Note that there is no actual need to respect the command
# case, so writing "config guessme" is the same in the example above.
#
# SENTINEL SET can also be used in order to perform this configuration at runtime.
#
# In order to set a command back to its original name (undo the renaming), it
# is possible to just rename a command to itself:
#
# SENTINEL rename-command mymaster CONFIG CONFIG

# HOSTNAMES SUPPORT
#
# Normally Sentinel uses only IP addresses and requires SENTINEL MONITOR
# to specify an IP address. Also, it requires the Redis replica-announce-ip
# keyword to specify only IP addresses.
#
# You may enable hostnames support by enabling resolve-hostnames. Note
# that you must make sure your DNS is configured properly and that DNS
# resolution does not introduce very long delays.
#
sentinel resolve-hostnames no

# When resolve-hostnames is enabled, Sentinel still uses IP addresses
# when exposing instances to users, configuration files, etc. If you want
# to retain the hostnames when announced, enable announce-hostnames below.
#
sentinel announce-hostnames no
# Generated by CONFIG REWRITE
user default on nopass ~* &* +@all
sentinel myid d7cdf54e03f6d35d54ffa3c2b41c90ea311206ba
sentinel config-epoch mymaster 0
sentinel leader-epoch mymaster 0
sentinel current-epoch 0
sentinel known-replica mymaster 192.168.58.134 6379
sentinel known-replica mymaster 192.168.58.135 6379
sentinel known-sentinel mymaster 192.168.58.134 26379 b6894c554cfb55939528e923b3faf5377f3d0082
sentinel known-sentinel mymaster 192.168.58.135 26379 4da386c2281efacafcb000d4f99b5f8db0ee68e5


第三步  启动命令
docker build -t my_redis_image -f work/docker/redis/my_redis work/docker/redis
```

##### 4、数据卷（Volume）

我们对容器数据的操作都会发现一个严重的问题——**容器与数据的耦合**，主要体现在以下几个方面

1、难以修改：例如当我们要修改Nginx的html内容时，需要进入容器内部修改，很不方便

2、数据不可复用：在容器内部的修改对外是不可见的，所有修改对新创建的容器都是不可复用的

3、升级维护困难：数据在容器内，如果要升级容器必然会删除旧容器，所有数据就都跟着删除了

而通过数据卷，就可以完美解决上述三个问题。

###### 1、基本概念

数据卷（volume）是一个虚拟目录，指向主机文件系统中的某个目录

![](C:\Users\Administrator\Desktop\Work\自研笔记\images\image-20240512125641645.png)

如图所示将容器的虚拟目录conf、html和主机系统文件目录中的conf、html关联起来就实现了

1、易于修改：即使不进入容器内部也可以通过修改conf、html内的文件来修改容器中的文件

2、数据可复用：当创建了新的容器并需要复用之前的conf、html文件时，可以直接再创建一个数据卷或者修改之前数据卷对容器目录的指向来进行关联，这样就可以将之前的配置映射到新容器的配置中

3、升级维护简单：如果来了份新容器，需要删除旧容器，也不用担心数据会被删除掉，因为数据在主机上还保存了一份

```
docker volume [command]
				create		创建一个volume
				inspect		显示一个或多个volume的信息
				ls			列出所有的volume
				prune		删除未使用的volume
				rm			删除一个或多个指定的volume
```

###### 2、案例

创建一个数据卷，查看数据卷在主机中的目录位置，实现挂载数据卷，最后修改容器内容

①创建数据卷并查看所有数据卷

```
[root@localhost ~]# docker volume create miiia
miiia
[root@localhost ~]# docker volume ls
DRIVER    VOLUME NAME
local     a5bc9954c7f5a3f64fb35d4783d01cb5cc1a958d07b130dcb5c04980da5a7033
local     aa9de1d99a3b3295b53aee9ef46c07ed4f965b2fe573a842d372cbb5dfe0b58c
local     c761395887a3655c374a3c244808d73b1177999b8af0f511e6c7b5160a312f57
local     miiia
```

②查看数据卷详细信息

```
[root@localhost ~]# docker inspect miiia 
[
    {
        "CreatedAt": "2024-05-12T13:08:51+08:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/miiia/_data",
        "Name": "miiia",
        "Options": null,
        "Scope": "local"
    }
]
```

③	在开始运行容器时，可以通过-v参数来挂载一个数据卷到某个容器目录

```
docker run --name i-nginx -v miiia:/usr/local/nginx/nginx.conf -p 80:80 -d my_nginx_image
#-v 数据卷名称：对应容器中目录地址
```

ps：事实上，因为加-v后没有创建卷会自动创建，所以不执行①②两步也是可以的

###### 3、另一种挂载方式

即目录挂载，与数据卷挂载的语法相似，docker run的命令中也是通过-v参数挂载：

①-v 主机目录：容器目录

②-v 主机文件：容器文件

**数据卷挂载与目录挂载的区别：**

1、数据卷挂载耦合度低，由docker来管理目录，但目录较深

2、目录挂载耦合度高，需要管理员手动管理，但目录易寻



### 四、进入容器

##### 1、具体答疑解惑

```
docker pull 软件名：版本号
拉取软件镜像

docker run [options] 容器名 [COMMAND] 镜像名
创建并运行镜像

ctrl+p+q
退出容器且不关闭

docker attach 容器名
进入正在执行的终端
docker exec -it 容器名 /bin/bash
开启一个新终端
docker rm -f 容器名
删除容器

docker ps -a
查询容器信息
docker start/stop/restart 容器名
容器的启动、关闭、重启

docker top 容器名
查看容器内部进程
docker cp 容器id：容器内文件路径 主机存放文件路径
从容器中拷贝文件到主机

docker volume create 数据卷名
声明一个新的数据卷
docker run -d -v /容器挂载路径 镜像名
匿名挂载（自动分配一个挂在地址）
docker run -d -v 数据卷名：/容器挂载路径 镜像名
具名挂载（若是新数据卷名则新建一个数据卷并分配路径，否则使用旧数据卷名和路径）
docker run -d -v 主机路径：容器挂载路径 镜像名
路径挂载（将主机中的文件夹挂载到容器挂载路径上）

docker run -it --name 容器名 -volumes-from 旧容器名 镜像名
让新容器挂载旧容器的数据卷，达到多容器相互共享数据效果
```

##### 2、cmd和entrypoint区别

cmd为替换dockerfile中的命令，entrypoint为对dockerfile中的命令进行追加执行



### 五、Docker网络

##### 1、先导概念

docker会为每个容器都创建一个虚拟网卡，并且子网掩码为x.x.x.x/16，可以确保容器有足够数量的ip使用

容器之间的ip也可以ping通

**docker常见的网络模式**

- bridge：桥接模式（docker0就是这种模式）
- none：不指定网络
- host：共享主机网络
- container：容器网络模式（即新创建的容器使用现有容器的一切网络配置）

**Docker网络模式**

```
docker	network	ls       列出所有网络信息
				rm      删除网络
				prune    移除未使用的网络
				create   创建一个新网络
				inspect   查看详细信息
				connect   连接指定网络
				disconect  断开连接
```

##### 2、尝试通过网络建立redis集群

```
假设已准备好redis镜像
docker network create we-redis
#创建一个网络空间（默认为桥接模式）

docker run -d --name i-redis1 --network we-redis -p 7001:6379 redis:latest --appendonly yes --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --port 6379

docker run -d --name i-redis2 --network we-redis -p 7002:6379 redis:latest --appendonly yes --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --port 6379

docker run -d --name i-redis3 --network we-redis -p 7003:6379 redis:latest --appendonly yes --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --port 6379

docker run -d --name i-redis4 --network we-redis -p 7004:6379 redis:latest --appendonly yes --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --port 6379

docker run -d --name i-redis5 --network we-redis -p 7005:6379 redis:latest --appendonly yes --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --port 6379

docker run -d --name i-redis6 --network we-redis -p 7006:6379 redis:latest --appendonly yes --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --port 6379
#创建六个redis容器，命名1-6，主机端口映射为7001-7006

docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' i-redis1
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' i-redis2
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' i-redis3
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' i-redis4
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' i-redis5
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' i-redis6
#获取每个redis容器的IP地址

docker run -it --name our_redis --network we-redis --rm redis-cli --cluster create \
    [IP1]:6379 [IP2]:6379 [IP3]:6379 [IP4]:6379 [IP5]:6379 [IP6]:6379 \
    --cluster-replicas 1
docker run -it --name our_redis --network we-redis --rm redis redis-cli --cluster create \
    172.18.0.2:6379 172.18.0.3:6379 172.18.0.4:6379 172.18.0.5:6379 172.18.0.6:6379 172.18.0.7:6379 \
    --cluster-replicas 1
#六个redis集群通过we-redis网络空间连接成一个集群

docker exec -it 容器名 redis-cli -c
#启动带有集群支持的redis-cli
```

仅为一个尝试，实际操作中不会使用这种方式创建redis集群



### 六、Dockerfile与多段构建

一般来讲由Dockerfile定义生成的容器应该尽可能的精简

docker 1.10版本及更高版本中，只有RUN、ADD、COPY会创建层，其他命令不会增加构建体积

此为举例说明，实际由于redis需要alpine环境的apk包管理器，需要使用alpine镜像先安装包管理器

##### 简单构建

```
FROM centos:7.9.2009 AS build
#先设置安装yum的镜像并起一个别名my_yum，默认为0

RUN yum -y update && yum -y install yum yum-utils
#安装yum工具

FROM redis:7.0.3
#拉取redis镜像

COPY --from=build / /
#从build层中获取文件

RUN yum -y install redis redis-tools
#安装redis及redis-tools

COPY config/redis.conf /etc/redis/redis.conf
#采用的redis主配置文件

EXPOSE 6379
#暴露的监听端口

VOLUME /data
#设置数据卷为data
```

##### Docker指令集

```
FROM 这个镜像的妈妈是谁？（指定基础镜像）

MAINTAINER 告诉别人，谁负责养它？（指定维护者信息，可以没有）LABLE key=values

RUN 你想让它干啥（在命令前面加上RUN即可）

ADD/COPY 给它点创业资金（COPY文件，会自动解压）

WORKDIR 我是cd,今天刚化了妆（设置当前工作目录）

VOLUME 给它一个存放行李的地方（设置卷，挂载主机目录）

EXPOSE 它要打开的门是啥（指定对外的端口）

CMD 奔跑吧，兄弟！（指定容器启动后的要干的事情）

ENTRYPOINT  容器启动命名

ENV 环境变量

USER 切换用户

```

### 七、Compose

​      Docker Compose 是一个用于定义和运行多容器 Docker 应用的工具。通过使用 YAML 文件，你可以配置应用需要的所有服务，然后使用一个命令就能创建并启动这些服务。Docker Compose 非常适合用来管理复杂的应用程序，其中包含多个相互依赖的服务。  

#####  编排启动镜像 

```
$ vim /opt/docker-compose/wordpress/docker-compose.yaml
services:
   db:
     image: registry.cn-hangzhou.aliyuncs.com/hujiaming/mysql:5.7
     volumes:
       - /data/db_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: somewordpress
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: wordpress
   wordpress:
     depends_on:
       - db
     image: registry.cn-hangzhou.aliyuncs.com/hujiaming/mysql:5.7
     volumes:
       - /data/web_data:/var/www/html
     ports: 
       - "8000:80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: wordpress

$ cd /opt/docker-compose/wordpress

# 启动服务
$ docker-compose up -d
[+] Running 34/34
 ⠿ wordpress Pulled                                                          181.8s
   ⠿ 1fe172e4850f Already exists                                             0.0s
   ⠿ 012a3732d045 Pull complete                                              37.3s
   ⠿ 43092314d50d Pull complete                                              71.3s
   ⠿ 4f615e42d863 Pull complete                                              71.4s
   ⠿ cd39010a4efc Pull complete                                              72.2s
   ⠿ d983c9ce24de Pull complete                                              72.3s
   ⠿ ecbdd59ae430 Pull complete                                              75.8s
   ⠿ 9d02b88c8618 Pull complete                                              79.5s
   ⠿ 50a246031d43 Pull complete                                              79.6s
   ⠿ a6c0267e6c34 Pull complete                                              84.0s
   ⠿ 787ca6348cef Pull complete                                              91.5s
   ⠿ da8ad43595e2 Pull complete                                              97.4s
   ⠿ e191f9e80e29 Pull complete                                              97.4s
   ⠿ fed8d3fd90f9 Pull complete                                              106.6s
   ⠿ 9ffdaa9000ed Pull complete                                              129.4s
   ⠿ 5774aeca6412 Pull complete                                              129.5s
   ⠿ 6978431bb9e2 Pull complete                                              129.6s
   ⠿ fb4d3fb05351 Pull complete                                              129.7s
   ⠿ 23d3af42839e Pull complete                                              133.6s
   ⠿ a5b33728e4a6 Pull complete                                              133.7s
   ⠿ 766e2b674cd0 Pull complete                                              136.9s
 ⠿ db Pulled                                                                 175.8s
   ⠿ 4be315f6562f Pull complete                                              18.4s
   ⠿ 96e2eb237a1b Pull complete                                              18.5s
   ⠿ 8aa3ac85066b Pull complete                                              18.8s
   ⠿ ac7e524f6c89 Pull complete                                              18.9s
   ⠿ f6a88631064f Pull complete                                              19.0s
   ⠿ 15bb3ec3ff50 Pull complete                                              35.7s
   ⠿ ae65dc337dcb Pull complete                                              35.8s
   ⠿ a4c4c43adf52 Pull complete                                              35.9s
   ⠿ c6cab33e8f91 Pull complete                                              140.1s
   ⠿ 2e1c4f2c43f6 Pull complete                                              140.2s
   ⠿ 2e5ee322af48 Pull complete                                              140.2s
[+] Running 3/3
 ⠿ Network root_default        Created                                       0.1s
 ⠿ Container root-db-1         Started                                       0.8s
 ⠿ Container root-wordpress-1  Started                                       1.1s
```

### 八、 HarBor私有镜像仓库

​      Harbor 是一个开源的企业级容器镜像仓库，由 VMware 开发并捐赠给 CNCF（Cloud Native Computing Foundation）。它不仅提供了存储和分发 Docker 镜像的基本功能，还增加了很多企业级功能，使得它在安全性、管理性和可扩展性方面表现优异。Docker容器应用的开发和运行离不开可靠的镜像管理。

##### 8.1 安装docker-compose

```
$ wget https://github.com/docker/compose/releases/download/v2.5.0/docker-compose-linux-x86_64
$ mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose
$ chmod a+x /usr/local/bin/docker-compose
$ docker-compose version
Docker Compose version v2.15.1
```

##### 8.2 安装Harbor

```
# 上传安装包
[root@docker-ce opt]# ls
harbor-offline-installer-v2.7.1.tgz  

# 解压安装包
[root@docker-ce opt]# tar xf harbor-offline-installer-v2.7.1.tgz 

# 切换目录
[root@docker-ce opt]# cd harbor

# 拉取harbor所需的镜像
[root@docker-ce harbor]# docker load -i harbor.v2.7.1.tar.gz 

# 修改配置文件
[root@docker-ce harbor]# cp harbor.yml.tmpl  harbor.yml
[root@docker-ce harbor]# vim harbor.yml
...
hostname: harbor.tanke.love
http:
  port: 80
https:
  port: 443
  certificate: /tmp/fullchain.pem
  private_key: /tmp/privkey.pem
harbor_admin_password: 12345
data_volume: /data/harbor
location: /var/log/harbor
...

# 创建数据存放目录
[root@docker-ce harbor]# mkdir /data/harbor -p

# 添加权限
[root@docker-ce harbor]# chmod 777 -R /data/  /var/log/harbor/

# 生成和检查Harbor的配置文件，并确保所有必要的依赖项和环境都已准备就绪。
[root@docker-ce harbor]# ./prepare 
prepare base dir is set to /opt/harbor
Generated configuration file: /config/portal/nginx.conf
Generated configuration file: /config/log/logrotate.conf
Generated configuration file: /config/log/rsyslog_docker.conf
Generated configuration file: /config/nginx/nginx.conf
Generated configuration file: /config/core/env
Generated configuration file: /config/core/app.conf
Generated configuration file: /config/registry/config.yml
Generated configuration file: /config/registryctl/env
Generated configuration file: /config/registryctl/config.yml
Generated configuration file: /config/db/env
Generated configuration file: /config/jobservice/env
Generated configuration file: /config/jobservice/config.yml
Generated and saved secret to file: /data/secret/keys/secretkey
Successfully called func: create_root_cert
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir

# 查看镜像
[root@docker-ce harbor]# docker-compose images
CONTAINER           REPOSITORY                    TAG                 IMAGE ID            SIZE
harbor-core         goharbor/harbor-core          v2.7.1              49d6c8a15d6c        215MB
harbor-db           goharbor/harbor-db            v2.7.1              b3f8d9d6c213        174MB
harbor-jobservice   goharbor/harbor-jobservice    v2.7.1              829d13e6aae7        252MB
harbor-log          goharbor/harbor-log           v2.7.1              eeb93d98a358        133MB
harbor-portal       goharbor/harbor-portal        v2.7.1              fe05b1b0bcfd        135MB
nginx               goharbor/nginx-photon         v2.7.1              e98018335c0d        126MB
redis               goharbor/redis-photon         v2.7.1              229dd1844a26        127MB
registry            goharbor/registry-photon      v2.7.1              9d50b10d3700        78.1MB
registryctl         goharbor/harbor-registryctl   v2.7.1              9b2219d529c8        140MB

# 安装和启动 Harbor 的主要安装脚本
[root@docker-ce harbor]# ./install.sh 
[Step 5]: starting Harbor ...
WARN[0000] /opt/harbor/docker-compose.yml: `version` is obsolete 
[+] Running 10/10
 ✔ Network harbor_harbor        Created                                                                     0.1s 
 ✔ Container harbor-log         Started                                                                     0.9s 
 ✔ Container registry           Started                                                                     2.3s 
 ✔ Container registryctl        Started                                                                     1.9s 
 ✔ Container redis              Started                                                                     2.0s 
 ✔ Container harbor-portal      Started                                                                     2.3s 
 ✔ Container harbor-db          Started                                                                     2.3s 
 ✔ Container harbor-core        Started                                                                     2.9s 
 ✔ Container nginx              Started                                                                     4.1s 
 ✔ Container harbor-jobservice  Started                                                                     4.0s 
✔ ----Harbor has been installed and started successfully.----


# 查看启动的容器
[root@docker-ce harbor]# docker-compose ps
NAME                IMAGE                                COMMAND                  SERVICE             CREATED             STATUS                    PORTS
harbor-core         goharbor/harbor-core:v2.7.1          "/harbor/entrypoint.…"   core                11 minutes ago      Up 11 minutes (healthy)   
harbor-db           goharbor/harbor-db:v2.7.1            "/docker-entrypoint.…"   postgresql          11 minutes ago      Up 11 minutes (healthy)   
harbor-jobservice   goharbor/harbor-jobservice:v2.7.1    "/harbor/entrypoint.…"   jobservice          11 minutes ago      Up 11 minutes (healthy)   
harbor-log          goharbor/harbor-log:v2.7.1           "/bin/sh -c /usr/loc…"   log                 11 minutes ago      Up 11 minutes (healthy)   127.0.0.1:1514->10514/tcp
harbor-portal       goharbor/harbor-portal:v2.7.1        "nginx -g 'daemon of…"   portal              11 minutes ago      Up 11 minutes (healthy)   
nginx               goharbor/nginx-photon:v2.7.1         "nginx -g 'daemon of…"   proxy               11 minutes ago      Up 11 minutes (healthy)   0.0.0.0:80->8080/tcp, :::80->8080/tcp, 0.0.0.0:443->8443/tcp, :::443->8443/tcp
redis               goharbor/redis-photon:v2.7.1         "redis-server /etc/r…"   redis               11 minutes ago      Up 11 minutes (healthy)   
registry            goharbor/registry-photon:v2.7.1      "/home/harbor/entryp…"   registry            11 minutes ago      Up 11 minutes (healthy)   
registryctl         goharbor/harbor-registryctl:v2.7.1   "/home/harbor/start.…"   registryctl         11 minutes ago      Up 11 minutes (healthy) 

# 登录镜像仓库
[root@docker-ce harbor]# docker login https://harbor.tanke.love
```

