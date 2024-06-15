# Redis自研笔记

## 本体服务

```
./redis-cli         #redis的客户端
./redis-server       #redis的服务端
./redis-check-aof     #用于修复出问题的AOF文件
./redis-sentinel      #用于集群管理
```

### 一、安装

```
# 下载redis安装包
[root@redis-master ~]# wget https://download.redis.io/releases/redis-6.2.7.tar.gz   

# 解压安装包到指定目录
    [root@redis-master ~]# tar xzf redis-6.2.7.tar.gz -C /usr/local/ && cd /usr/local   

# 创建软链接
[root@localhost local]# ln -s redis-6.2.7 redis

# 切换值redis家目录
[root@localhost local]# cd redis/

# 安装编译环境
[root@redis-master redis]# yum install -y gcc make

# 编译
[root@redis-master redis]# make

# 备份文件
[root@redis-master redis]# cp redis.conf redis.conf.bak

# 修改配置文件
[root@redis-master redis]# vim redis.conf     ---修改如下
/var/log/redis.log
bind 192.168.246.202　　#只监听内网IP
daemonize yes　　　　　#开启后台模式将no改为yes
port 6379                      #端口号
dir /data/redis/data　　#本地数据库存放持久化数据的目录该目录-----需要存在
创建存放数据的目录

# 配置systemctl的redis启动脚本
[root@redis-master redis]# cd /lib/systemd/system
[root@redis-master system]# vim redis.service
[Unit]
Description=Redis
After=network.target

[Service]
ExecStart=/usr/local/redis/src/redis-server /usr/local/redis/redis.conf  --daemonize no
ExecStop=/usr/local/redis/src/redis-cli -h 127.0.0.1 -p 6379 shutdown

[Install]
WantedBy=multi-user.target

# 重新加载并且启动服务
[root@redis-master system]# systemctl daemon-reload
[root@redis-master system]# systemctl enable --now redis
```

```
登陆redis       
[root@redis-master system]# cd /usr/local/redis/src/
[root@redis-master src]# ./redis-cli -h 192.168.246.202 -p 6379
# 测试redis是否可以用
192.168.246.202:6379> ping    
PONG

# 设置key--name，并设置值
192.168.246.202:6379> set name xiaoming    
OK

# 获取到key
192.168.246.202:6379> get name    
"xiaoming"
单机版redis已经部署完成。将ip和端口发给开发就可以了。



192.168.246.202:6379> set key value [EX seconds] [PX milliseconds] [NX|XX]
# 参数解释：
	EX seconds ： 将键的过期时间设置为 seconds 秒。默认为秒；
	PX milliseconds ： 将键的过期时间设置为 milliseconds 毫秒。默认为毫秒；
	NX ： 只在键不存在时， 才对键进行设置操作。
	XX ： 只在键已经存在时， 才对键进行设置操作。会覆盖原有的values值

# 使用 EX 选项：
[root@localhost src]# ./redis-cli -h 192.168.62.231 -p 6379
192.168.62.231:6379> set name1 xiaohong EX 10
OK
192.168.62.231:6379> get name1
"xiaohong"

# 等待10s，再次查看
192.168.62.231:6379> get name1
(nil)

# 使用 PX 选项：
192.168.62.231:6379> set name2 xiaohong PX 3233
OK
192.168.62.231:6379> get name2
"xiaohong"

# 等待3s，再次查看
192.168.62.231:6379> get name2
(nil)

#  NX 选项：
192.168.62.231:6379> set class 2204 NX
OK # 键不存在，设置成功
192.168.62.231:6379> get class
"2204"
192.168.62.231:6379> set class 2205 NX
(nil)  # 键已经存在，设置失败
192.168.62.231:6379> get class
"2204"  # 维持原值不变

使用 XX 选项：
192.168.62.231:6379> set home taikang XX
(nil)  # 因为键不存在，设置失败
192.168.62.231:6379> set home taikang
OK # 先给键设置一个值
192.168.62.231:6379> set home zhengzhou XX
OK # 设置新值成功
192.168.62.231:6379> get home
"zhengzhou"

删除：
192.168.62.231:6379> del class
(integer) 1
192.168.62.231:6379> get class
(nil)
```

```
# 切换数据库，共16个库
127.0.0.1:6379[1]> SELECT 2
OK
127.0.0.1:6379[2]> 

# 查看当前库所有的数据
127.0.0.1:6379> keys *
```

### 二、设置密码

Redis修改密码的方式主要有两种：使用`redis-cli`命令行工具和通过配置文件。

##### 1、命令行临时修改密码

1. **登录Redis**：首先，你需要使用当前没有密码的Redis客户端登录到Redis服务器。
2. **设置新密码**：使用`CONFIG SET`命令来设置新的密码。例如，要设置密码为`yyds`，你可以执行：

```bash
127.0.0.1:6379> CONFIG SET requirepass "yyds"

# 密码使用方式
	# 方式一：
192.168.174.48:6379> AUTH yyds
OK

	# 方式二：
[root@localhost ~]# redis-cli  -h 192.168.174.48  -p 6379 -a yyds
```

>  注意：这种方式设置的密码只会`临时生效`，重启Redis服务后密码会失效。

##### 2、配置文件永久修改密码

1. **找到Redis配置文件**：通常，Redis的配置文件名为`redis.conf`，它位于Redis安装目录或数据目录中。
2. **编辑配置文件**：使用文本编辑器打开`redis.conf`文件。
3. **设置密码**：在配置文件中找到`# requirepass foobared`这一行（没有`#`注释符号），将`foobared`替换为你想要设置的新密码。例如，设置为`yyds`：

```bash
[root@localhost redis]# vim redis.conf
···
requirepass yyds
···

# 重启服务
[root@localhost redis]# systemctl restart redis

# 查看服务状态
[root@localhost redis]# systemctl status redis

# 测试
[root@localhost src]# ./redis-cli 
127.0.0.1:6379> get name 
(error) NOAUTH Authentication required.
127.0.0.1:6379> auth yyds
OK
127.0.0.1:6379> get name 
"zhangsan"
```

**保存并重启Redis**：保存配置文件并重启Redis服务，新的密码设置就会生效。

注意事项

- 设置密码可以增加Redis的`安全性`，但也会增加管理的`复杂性`。确保选择一个强密码，并妥善保管。
- 在修改密码后，确保所有客户端都已更新为使用新密码进行`连接`，否则可能会出现认证失败的问题。

##### 3、Redis配置文件详解

```
[root@localhost redis]# egrep -v "^#|^$" redis.conf
# 设置Redis服务器监听所有IP地址，即允许任何客户端连接。
bind 0.0.0.0
# 关闭保护模式，允许非本机客户端连接。
protected-mode no
# 设置Redis服务器监听的端口号为6379。
port 6379
# 设置TCP连接队列的大小为511，即允许最多有511个连接等待被处理。
tcp-backlog 511
# 设置超时时间
timeout 0
# 设置TCP心跳间隔为300秒。
tcp-keepalive 300
# 以守护进程方式运行Redis服务器。
daemonize yes
# 指定Redis服务器的进程ID文件路径。
pidfile /var/run/redis_6379.pid
# 设置日志级别为notice，只记录警告和错误信息。
loglevel notice
# 指定日志文件的路径。
logfile "/var/log/redis.log"
# 设置Redis支持的数据库数量为16个。
databases 16
# 在后台保存出错时停止写入操作。
stop-writes-on-bgsave-error yes
# rdbcompression yes：启用RDB文件压缩。
rdbcompression yes
# 在RDB文件中包含CRC64校验和
rdbchecksum yes
# 指定RDB文件的名称。
dbfilename dump.rdb
# 不删除同步生成的RDB文件。
rdb-del-sync-files no
# 指定RDB文件和AOF文件的存储目录。
dir /data/redis/data
# 设置副本节点为只读模式。
replica-read-only yes
# 设置密码为"qfyyds"，用于验证客户端连接。
requirepass qfyyds
# 禁用AOF持久化。
appendonly no
# 指定AOF文件的名称。
appendfilename "appendonly.aof"
# 每秒执行一次fsync操作，将缓冲区的数据写入磁盘。
appendfsync everysec
```

### 三、持久化

Redis是一个内存数据库，一旦断电或服务器进程退出，内存数据库中的数据将`全部丢失`，所以需要redis持久化；Redis持久化就是把`数据保存在磁盘上`，利用`永久性存储介质`将数据保存，在特定的时间将保存的数据进行恢复的工作机制。

##### 1、RDB

在指定的时间间隔内将内存中的数据集写入磁盘，也就是`快照`(Snapshot),数据恢复是将快照文件直接读到`内存中`redis会单独创建(`fork`)一个`子进程`来进行`持久化`，会先将数据写入一个到一个`临时文件`(dump.rdb)中,待持久化过程结束后，再用本次的临时文件替换上次持久化后的文件。

`fork函数`的作用是复制一个与当前进程一样的进程，新进程的所有数据数值都和原进程一致，但是一个全新的进程，并作为`原进程的子进程`。

redis服务器在处理`bgsave`采用`子线程`进行IO写入，而主进程仍然可以接收其他请求，但创建子进程是同步阻塞的，此时不接受其他请求。

```
[root@redis-master redis]# vim /usr/local/redis/redis.conf
···
# dbfilename：持久化数据存储在本地的文件
dbfilename dump.rdb

#dir：持久化数据存储在本地的路径,可自定义
dir /data/redis/data

##snapshot触发的时机，save <seconds> <changes> 
##对于此值的设置，需要谨慎，评估系统的变更操作密集程度  
##可以通过save “”来关闭snapshot功能  
# 900秒内如果至少有一个key进行了修改则进行持久化操作
save 900 1
# 300秒内如果至少有10个key进行了修改则进行持久化操作
save 300 10
# 60秒内，如果至少有10000个key进行了修改则进行持久化操作
save 60 10000 

##yes代表当使用bgsave命令持久化出错时候停止写RDB快照文件,no表明忽略错误继续写文件，“错误”可能因为磁盘已满/磁盘故障/OS级别异常等
stop-writes-on-bgsave-error yes

##是否启用rdb文件压缩，默认为“yes”，压缩往往意味着“额外的cpu消耗”，同时也意味着较短的网络传输时间  
rdbcompression yes 
```

##### 2、AOF

- `always`: 把每个写命令立即同步到AOF文件，很慢但安全；
- `everysec`: 每秒同步一次，默认配置；
- `no`: redis不执行写入磁盘。

###### 手动触发

```
127.0.0.1:6379> BGREWRITEAOF
Background append only file rewriting started


[root@localhost data]# stat appendonly.aof 
  文件："appendonly.aof"
  大小：112       	块：8          IO 块：4096   普通文件
设备：fd00h/64768d	Inode：17874728    硬链接：1
权限：(0644/-rw-r--r--)  Uid：(    0/    root)   Gid：(    0/    root)
最近访问：2024-05-14 22:03:51.707774066 +0800
最近更改：2024-05-14 22:03:51.707774066 +0800
最近改动：2024-05-14 22:03:51.818774763 +0800
```

默认情况，redis是没有开启AOF(默认使用RDB持久化)，需要通过配置文件开启。

```
# 是否开启 Redis AOF持久化，默认为no
appendonly no

# AOF持久化文件名
appendfilename "appendonly.aof"

# AOF持久化策略，默认为eveysec，每秒同步一次
# appendfsync always
appendfsync everysec
# appendfsync no
```

- **Always**：每次执行`写入操作`后，都会立即调用fsync将数据`同步到磁盘`。这确保了极高的数据`持久性`，因为即使在系统崩溃的情况下，也最多只会丢失一次写入操作的数据。然而，这种模式会对性能产生较大影响，因为在每次写入后都进行fsync会导致较高的I/O开销。
- **Everysec**：这是默认设置，`每秒`执行一次fsync。它在性能和持久性之间取得了平衡，既保证了较好的数据安全性，又避免了频繁的I/O操作对性能的影响。
- **No**：不做持久化

###### 重写机制

AOF持久化，会把每次写命令都`追加`到`appendonly.aof`文件中，当文件过大，redis的数据恢复时间就会变长，因此加入重写策略对aof文件进行重写，生成一个恢复当前数据的最少命令集。`通过压缩AOF文件里面的相同指令保留最新的一个数据操作指令，即将存储了某个key的多次变更记录只是存储最新的变更记录即可，丢弃历史变更记录 。`

```
# Redis 重写机制配置
[root@localhost redis]#  vim redis.conf
···
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
···
```

>1. **auto-aof-rewrite-percentage**：当前 AOF 文件大小超过上次重写后 AOF 文件大小的百分比时，触发 AOF 重写机制**，默认值为 100 。**
>2. **auto-aof-rewrite-min-size**：表示当当前 AOF 文件大小超过指定值时，才可能触发 AOF 重写机制，**默认值为 64 MB 。**
>
>- 系统自动触发 AOF 重写机制还需要满足以下条件 ：
> - `当前没有正在执行 BGSAVE 或 BGREWRITEAOF 的子进程`
> - `当前没有正在执行 SAVE 的主进程`
> - `当前没有正在进行集群切换或故障转移`

###### 持久化配置

```
AOF默认关闭--开启
[root@redis-master src]# cd ..
[root@redis-master redis]# vim redis.conf
修改如下:
appendonly yes
1、此选项为aof功能的开关，默认为“no”，可以通过“yes”来开启aof功能,只有在“yes”下，aof重写/文件同步等特性才会生效
====================================
2、指定aof文件名称
appendfilename appendonly.aof  
====================================
3、指定aof操作中文件同步策略，有三个合法值：always everysec no,默认为everysec
appendfsync everysec
always     #每次有数据修改发生时都会写入AOF文件
everysec   #每秒钟同步一次，该策略为AOF的缺省策略/默认策略
no         #从不同步。高效但是数据不会被持久化
```

**开启持久化功能后，重启redis后，数据会自动通过持久化文件恢复**

##### 4、通过RDB备份文件恢复数据

**恢复Redis示例配置**

```
# 配置文件修改
[root@redis-backup ~]# vim /usr/local/redis/redis.conf 
···
bind 0.0.0.0
dbfilename dump.rdb
dir /data/redis/data/
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
daemonize yes
···

# 启动实例
[root@redis-backup ~]# cd /usr/local/redis/src/
[root@redis-backup redis]# ./src/redis-server redis.conf 
[root@redis-backup redis]# ./src/redis-cli 
127.0.0.1:6379> keys *
1) "name"
```

##### 5、通过AOF备份文件恢复数据

```shell
# 配置文件修改
[root@redis-backup ~]# vim /usr/local/redis/redis.conf 
···
appendonly yes
appendfilename "appendonly.aof"
···

[root@redis-backup ~]# mv /var/lib/redis/appendonly.aof /var/lib/redis/appendonly.aof.bak
[root@redis-backup ~]# cp /path/to/your/backup/appendonly.aof /var/lib/redis/appendonly.aof
[root@redis-backup ~]# cd /usr/local/redis/src/
[root@redis-backup redis]# ./src/redis-server redis.conf 
[root@redis-backup redis]# ./src/redis-cli 
127.0.0.1:6379> keys *
1) "name"
```

## 主从复制及哨兵集群

### 一、主从复制

设置redis.conf配置文件

主机设置关闭保护模式

从机设置同步的主机库信息并关闭保护模式

环境准备：

```ini
redis-master	192.168.174.48		
redis-slave-1	192.168.174.49
redis-slave-2	192.168.174.50
1.首先三台服务器将redis部署完成。
2.编辑master的redis配置文件:
[root@redis-master ~]# cd /usr/local/redis
[root@redis-master redis]# vim redis.conf
...
# 设置Redis监听的IP地址和端口号，默认监听所有IP地址和6379端口
bind 0.0.0.0

# 关闭保护模式，允许远程访问
protected-mode no

# 指定Redis监听的端口号
port 6379

# 增加Redis的最大内存限制，以容纳更多数据
#maxmemory 16GB   增加内存限制，根据您的服务器实际内存调整
maxmemory 20480mb
...
```

>  关闭protected-mode模式，此时外部网络可以直接访问
>
>  开启protected-mode保护模式，需配置bind ip或者设置访问密码

```shell
# 启动主节点redis服务
[root@redis-master ~]# cd /usr/local/redis/src
[root@redis-master src]# ./redis-server ../redis.conf &   会加载此文件中的配置信息
[root@redis-master src]# ss -tunlp | grep  6379
tcp    LISTEN     0      128       *:6379                  *:*                   users:(("redis-server",pid=1360,fd=6))
```

```shell
4.修改slave1的配置文件：
[root@redis-slave-1 ~]# cd /usr/local/redis
[root@redis-slave-1 redis]# vim redis.conf      ---修改如下：
...
# 添加需要同步的主库信息
replicaof 192.168.174.48 6379

# 关闭保护模式，允许远程访问
protected-mode no
...

# 启动从节点1的redis服务
[root@redis-slave-1 ~]# cd /usr/local/redis/src/
[root@redis-slave-1 src]# ./redis-server ../redis.conf &
[root@redis-slave01 src]# ss -tunlp | grep  6379
tcp    LISTEN     0      128       *:6379                  *:*                   users:(("redis-server",pid=1360,fd=6))
```

> 可以通过 `replicaof（Redis 5.0 之前使用 slaveof）`命令形成主库和从库的关系。

```shell
# 修改slave2的配置文件
[root@redis-slave-2 ~]# cd /usr/local/redis/redis/
[root@redis-slave-2 redis]# vim redis.conf       ---修改如下
...
# 添加需要同步的主库信息
replicaof 192.168.174.48 6379

# 关闭保护模式，允许远程访问
protected-mode no
...

# 启动从节点2的redis服务
[root@ansible-web2 ~]# cd /usr/local/redis/src/
[root@ansible-web2 src]# ./redis-server ../redis.conf &
[root@redis-slave02 ~]# ss -tunlp | grep  6379
tcp    LISTEN     0      128       *:6379                  *:*                   users:(("redis-server",pid=1360,fd=6))
```

```shell
或者，配置到了系统管理工具里面直接可以以下操作
8.重启三台redis
[root@redis-master redis]# systemctl restart redis.service
[root@redis-slave-1 ~]# systemctl restart redis.service
[root@redis-slave-2 ~]# systemctl restart redis.service
```

```shell
9.测试主从
1.在master上面执行
[root@redis-master redis]# cd src/
[root@redis-master src]# ./redis-cli 
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> set name jack
OK
127.0.0.1:6379> get name
"jack"

2.分别在slave-1和slave-2上面执行:
[root@redis-slave-1 redis]# cd src/
[root@redis-slave-1 src]# ./redis-cli 
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> get name
"jack"
127.0.0.1:6379>
[root@redis-slave-2 src]# ./redis-cli 
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> get name
"jack"
127.0.0.1:6379>

# 查看复制状态,master执行：
127.0.0.1:6379> info replication
role:master
connected_slaves:2
slave0:ip=192.168.246.203,port=6379,state=online,offset=490,lag=0
slave1:ip=192.168.246.204,port=6379,state=online,offset=490,lag=1
==============================================================================
slave上面执行：
127.0.0.1:6379> info replication	
# Replication
role:slave
master_host:192.168.246.202
master_port:6379
master_link_status:up
```

注意：从服务器一般默认禁止写入操作：slave-read-only yes

```bash
# master 节点执行info  repliaction 命令回显参数解释

# 表示当前节点的角色是主节点
role:master
# 表示当前主节点连接了两个从节点。
connected_slaves:2
# 表示第一个从节点的IP地址为192.168.174.49，端口号为6379，状态为在线（online），复制偏移量为3276，与主节点的延迟为1。
slave0:ip=192.168.174.49,port=6379,state=online,offset=3276,lag=1
# 表示第二个从节点的IP地址为192.168.174.50，端口号为6379，状态为在线（online），复制偏移量为3276，与主节点的延迟为1。
slave1:ip=192.168.174.50,port=6379,state=online,offset=3276,lag=1
# 表示当前没有进行故障转移。
master_failover_state:no-failover
# 表示主节点的复制ID。
master_replid:169726e22cc9736afd05f50b7fef4d8b6e48b47a
# 表示主节点的第二个复制ID，这里为全零。
master_replid2:0000000000000000000000000000000000000000
# 表示主节点当前的复制偏移量。
master_repl_offset:3276
# 表示第二个从节点的复制偏移量，这里为-1，表示没有设置。
second_repl_offset:-1
# 表示复制积压缓冲区是否处于活动状态，1表示活动。
repl_backlog_active:1
# 表示复制积压缓冲区的大小，单位为字节。
repl_backlog_size:1048576
# 表示复制积压缓冲区中第一个字节的偏移量。
repl_backlog_first_byte_offset:1
# 表示复制积压缓冲区的历史长度。
repl_backlog_histlen:3276

# slave 节点执行 info replication 命令回显参数解释
# 表示当前节点的角色是从节点。
role:slave
# 表示主节点的IP地址为192.168.174.48。
master_host:192.168.174.48
# 表示主节点的端口号为6379。
master_port:6379
# 表示与主节点的连接状态为正常（up）。
master_link_status:up
# 表示距离上一次与主节点进行IO操作的时间过去了8秒。
master_last_io_seconds_ago:8
# 表示当前没有正在进行的主从同步操作。
master_sync_in_progress:0
# 表示从节点读取复制数据时的偏移量。
slave_read_repl_offset:3794
# 表示从节点当前的复制偏移量。
slave_repl_offset:3794
# 表示从节点的优先级为100。
slave_priority:100
# 表示从节点以只读模式运行。
slave_read_only:1
# 表示从节点已经向其他节点宣告自己是复制节点。
replica_announced:1
# 表示当前从节点没有连接其他从节点。
connected_slaves:0
# 表示当前没有进行故障转移。
master_failover_state:no-failover
# 表示主节点的复制ID。
master_replid:169726e22cc9736afd05f50b7fef4d8b6e48b47a
# 表示主节点的第二个复制ID，这里为全零。
master_replid2:0000000000000000000000000000000000000000
# 表示主节点当前的复制偏移量。
master_repl_offset:3794
# 表示第二个从节点的复制偏移量，这里为-1，表示没有设置。
second_repl_offset:-1
# 表示复制积压缓冲区是否处于活动状态，1表示活动。
repl_backlog_active:1
# 表示复制积压缓冲区的大小，单位为字节。
repl_backlog_size:1048576
# 表示复制积压缓冲区中第一个字节的偏移量。
repl_backlog_first_byte_offset:1
# 表示复制积压缓冲区的历史长度。
repl_backlog_histlen:3794
```

### 二、哨兵模式

```
设置sentinel.conf

sentinel monitor mymaster ip port ms
#主机和从机的哨兵都写主机IP
sentinel down-after-milliseconds mymaster ms
#哨兵关闭时间
sentinel failover-timeout mymaster
#故障转移操作时间
protected-mode no
#关闭加密模式
```

##### 配置哨兵模式

```shell
1.每台机器上修改redis主配置文件redis.conf文件设置：bind 0.0.0.0   ---已经操作
2.每台机器上修改sentinel.conf配置文件：修改如下配置
[root@redis-master ~]#  cd /usr/local/redis/

[root@redis-master redis]# vim sentinel.conf
...
sentinel monitor mymaster 192.168.174.48 6379 2 
sentinel down-after-milliseconds mymaster 3000 
sentinel failover-timeout mymaster 10000 
protected-mode no 
...

# 每台机器启动哨兵服务：
[root@redis-master redis]# ./src/redis-sentinel sentinel.conf 
注意:在生产环境下将哨兵模式启动放到后台执行:     ./src/redis-sentinel sentinel.conf &
```

> 参数解释：
>
> `sentinel down-after-milliseconds mymaster 3000 `：
>
> ​	当集群中有2个sentinel认为master死了时，才能真正认为该master已经不可用了。 (slave上面写的是master的ip，master写自己ip)
>
> `sentinel down-after-milliseconds mymaster 3000`： 
>
> ​	表示如果名为 `mymaster` 的主节点在3秒（3000毫秒）内未对 Sentinel 的 PING 命令做出有效响应，那么 Sentinel 会开始考虑该主节点可能已经出现故障，并做好相应的故障转移准备。
>
> `sentinel failover-timeout mymaster 10000` ：
>
> ​	表示在进行名为 `mymaster` 的主节点的故障转移操作时，Sentinel 最多允许花费10秒（10000毫秒）的时间来完成整个操作。
>
> `protected-mode no `：
>
> ​	关闭加密保护模式

### 三、负载均衡

一组主从机为一个节点，多个节点可以组成整个redis集群

##### 实战：三台机器相同操作

```shell
1.安装redis
[root@redis-cluster1 ~]# mkdir /data
[root@redis-cluster1 ~]# yum -y install gcc automake autoconf libtool make
[root@redis-cluster1 ~]# wget https://download.redis.io/releases/redis-6.2.0.tar.gz
[root@redis-cluster1 ~]# tar xzvf redis-6.2.0.tar.gz -C /data/
[root@redis-cluster1 ~]# cd /data/
[root@redis-cluster1 data]# mv redis-6.2.0/ redis
[root@redis-cluster1 data]# cd redis/
[root@redis-cluster1 redis]# make    #编译
[root@redis-cluster1 redis]# mkdir /data/redis/data #创建存放数据的目录
```

```shell
2.创建节点目录:按照规划在每台redis节点的安装目录中创建对应的目录（以端口号命名）
[root@redis-cluster1 redis]# pwd
/data/redis
[root@redis-cluster1 redis]# mkdir cluster #创建集群目录
[root@redis-cluster1 redis]# cd cluster/
[root@redis-cluster1 cluster]# mkdir 7000 7001 #创建节点目录

[root@redis-cluster2 redis]# mkdir cluster
[root@redis-cluster2 redis]# cd cluster/
[root@redis-cluster2 cluster]# mkdir 7002 7003

[root@redis-cluster3 redis]# mkdir cluster
[root@redis-cluster3 redis]# cd cluster/
[root@redis-cluster3 cluster]# mkdir 7004 7005
```

```shell
3.拷贝配置文件到节点目录中，#三台机器相同操作
[root@redis-cluster1 cluster]# cp /data/redis/redis.conf 7000/
[root@redis-cluster1 cluster]# cp /data/redis/redis.conf 7001/

[root@redis-cluster2 cluster]# cp /data/redis/redis.conf 7002/
[root@redis-cluster2 cluster]# cp /data/redis/redis.conf 7003/

[root@redis-cluster3 cluster]# cp /data/redis/redis.conf 7004/
[root@redis-cluster3 cluster]# cp /data/redis/redis.conf 7005/
```

```shell
4.修改集群每个redis配置文件。(主要是端口、ip、pid文件，三台机器相同操作)，修改如下：
[root@redis-cluster1 cluster]# cd 7000/
[root@redis-cluster1 7000]# vim redis.conf #修改如下
bind 192.168.116.172  #每个实例的配置文件修改为对应节点的ip地址
port 7000   #监听端口，运行多个实例时，需要指定规划的每个实例不同的端口号
daemonize yes #redis后台运行
pidfile /var/run/redis_7000.pid #pid文件，运行多个实例时，需要指定不同的pid文件
logfile /var/log/redis_7000.log #日志文件位置，运行多实例时，需要将文件修改的不同。
dir /data/redis/data #存放数据的目录
appendonly yes #开启AOF持久化，redis会把所接收到的每一次写操作请求都追加到appendonly.aof文件中，当redis重新启动时，会从该文件恢复出之前的状态。
appendfilename "appendonly.aof"  #AOF文件名称
appendfsync everysec #表示对写操作进行累积，每秒同步一次
以下为打开注释并修改
cluster-enabled yes #启用集群
cluster-config-file nodes-7000.conf #集群配置文件，由redis自动更新，不需要手动配置，运行多实例时请注修改为对应端口
cluster-node-timeout 5000 #单位毫秒。集群节点超时时间，即集群中主从节点断开连接时间阈值，超过该值则认为主节点不可以，从节点将有可能转为master
cluster-replica-validity-factor 10 #在进行故障转移的时候全部slave都会请求申请为master，但是有些slave可能与master断开连接一段时间了导致数据过于陈旧，不应该被提升为master。该参数就是用来判断slave节点与master断线的时间是否过长。（计算方法为：cluster-node-timeout * cluster-replica-validity-factor，此处为：5000 * 10 毫秒）
cluster-migration-barrier 1 #一个主机将保持连接的最小数量的从机，以便另一个从机迁移到不再被任何从机覆盖的主机
cluster-require-full-coverage yes #集群中的所有slot（16384个）全部覆盖，才能提供服务

#注：
所有节点配置文件全部修改切记需要修改的ip、端口、pid文件...避免冲突。确保所有机器都修改。
```

```shell
5.启动三台机器上面的每个节点(三台机器相同操作)
[root@redis-cluster1 ~]# cd /data/redis/src/
[root@redis-cluster1 src]# nohup ./redis-server ../cluster/7000/redis.conf &
[root@redis-cluster1 src]# nohup ./redis-server ../cluster/7001/redis.conf &

[root@redis-cluster2 7003]# cd /data/redis/src/
[root@redis-cluster2 src]# nohup ./redis-server ../cluster/7002/redis.conf &
[root@redis-cluster2 src]# nohup ./redis-server ../cluster/7003/redis.conf &

[root@redis-cluster3 7005]# cd /data/redis/src/
[root@redis-cluster3 src]# nohup ./redis-server ../cluster/7004/redis.conf &
[root@redis-cluster3 src]# nohup ./redis-server ../cluster/7005/redis.conf &
```

```
7.查看集群状态可连接集群中的任一节点，此处连接了集群中的节点192.168.116.172:7000
# #登录集群客户端，-c标识以集群方式登录

[root@redis-cluster1 src]# ./redis-cli -h 192.168.116.172 -c -p 7000
192.168.116.172:7000> ping
PONG
192.168.116.173:7002> cluster info  #查看集群信息
cluster_state:ok  #集群状态
cluster_slots_assigned:16384 #分配的槽
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6 #集群实例数
......

192.168.116.172:7000> cluster nodes  #查看集群实例
```

##### 登录并操作集群

```
测试链接redis，存取数据(链接集群中任意一台机器就可以。)
存：
[root@redis-cluster1 src]# ./redis-cli -h 192.168.116.172 -c -p 7000
192.168.116.172:7000> ping
PONG
192.168.116.172:7000> set name qianfeng
-> Redirected to slot [5798] located at 192.168.116.173:7002
OK
192.168.116.173:7002>

读
[root@redis-cluster3 src]# ./redis-cli -h 192.168.116.173 -c -p 7002
192.168.116.173:7002> ping
PONG
192.168.116.173:7002> get name
"qianfeng"
192.168.116.173:7002> exists name  #查看某一个key是否存在
(integer) 1
```

##### 主从切换

```
测试：
1.将节点cluster1的主节点7000端口的redis关掉
[root@redis-cluster1 src]# ps -ef |grep redis 
root      15991      1  0 01:04 ?        00:02:24 ./redis-server 192.168.116.172:7000 [cluster]
root      16016      1  0 01:04 ?        00:02:00 ./redis-server 192.168.116.172:7001 [cluster]
root      16930   1595  0 08:04 pts/0    00:00:00 grep --color=auto redis
[root@redis-cluster1 src]# kill -9 15991

查看集群信息：
192.168.116.173:7002> CLUSTER nodes
```

```
2.将该节点的7000端口redis启动在查看
[root@redis-cluster1 log]# cd /data/redis/src/
[root@redis-cluster1 src]# ./redis-server ../cluster/7000/redis.conf

查看节点信息：
192.168.116.173:7002> CLUSTER nodes
```

