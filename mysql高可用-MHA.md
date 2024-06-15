

[TOC]



## 一、环境准备

| IP             | 主机名       | 角色                                | OS      |
| -------------- | ------------ | ----------------------------------- | ------- |
| 192.168.96.136 | mysql-master | mysql master、MHA manager、MHA node | Centos7 |
| 192.168.96.142 | mysql-slave1 | mysql slave1、MHA node              | Centos7 |
| 192.168.96.143 | mysql-slave2 | mysql slave2、MHA node              | Centos7 |

### 1、配置hosts

```bash
192.168.96.136 mysql-master
192.168.96.142 mysql-slave1
192.168.96.143 mysql-slave2
```

### 2、关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld
```

### 3、禁用SELinux

```bash
# 临时关闭；关闭swap主要是为了性能考虑
swapoff -a
# 可以通过这个命令查看swap是否关闭了
free
# 永久关闭        
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

### 4、关闭swap

```bash
# 临时关闭；关闭swap主要是为了性能考虑
swapoff -a
# 可以通过这个命令查看swap是否关闭了
free
# 永久关闭        
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

### 5、配置互信

```bash
ssh-keygen
ssh-copy-id mysql-master
ssh-copy-id mysql-slave1
ssh-copy-id mysql-slave2
```

## 二、mysql主从部署

### 1）安装mysql（MySQL5.7.29）

```bash
# 添加mysql-server源
mkdir -p /opt/mysql-mha; cd /opt/mysql-mha
wget https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
rpm -ivh mysql80-community-release-el7-3.noarch.rpm
# 解决报错如，Check that the correct key URLs are configured for this repository.
rpm --import http://repo.mysql.com/RPM-GPG-KEY-mysql-2022  

# 更新yum缓存
yum makecache

# 使用yum查看MySQL的仓库，查看MySQL的版本
yum repolist all | grep mysql
# 安装yum-config-manager
yum -y install yum-utils
# 修改为需要的版本，即禁用yum存储库中mysql不需要的版本和开启需要的版本
yum-config-manager --disable mysql80-community
yum-config-manager --enable mysql57-community

# 先禁用本地的 MySQL 模块，要不然找不到mysql-community-server，默认mysql-server是8.0的版本
yum module disable mysql

# 开始安装mysql server和mysql client
yum install mysql-community-server mysql -y
```

### 2）mysql 节点配置

#### 1、修改配置文件

修改mysql的所有节点mysql的主配置文件 （`/etc/my.cnf`）
**Master 节点**

```bash
server-id = 1
log-bin=mysql-bin
binlog_format=mixed
log-slave-updates=true
```

**Slave1,Slave2节点**

```bash
server-id = 2 (slave3节点，则server-id=3。三台节点server-id不可重复)
log_bin=mysql-bin
relay-log=relay-log-bin
relay-log-index=slave-relay-bin.index
```

#### 3、启动服务

```bash
systemctl start mysqld
```

#### 5、设置root密码

```bash
# 默认密码
grep 'temporary password' /var/log/mysqld.log
mysql -uroot -p

# 重置root密码
set global validate_password_policy=0;
set global validate_password_length=1;
ALTER user 'root'@'localhost' IDENTIFIED BY '123456';
```

### 3）配置mysql 一主两从

#### 1、 所有数据库节点进行mysql授权

```bash
# 登录客户端
mysql -uroot -p
123456

set global validate_password_policy=0;
set global validate_password_length=1;

# 从库进行同步使用的用户
grant replication slave on *.* to 'slave'@'192.168.96.%' identified by '123456';

# MHA-manager使用
grant all  on *.* to 'mha'@'192.168.96.%' identified by '123456';

#防止从库通过主机名连接不上主库
grant all on *.* to 'mha'@'%' identified by '123456';

FLUSH PRIVILEGES;
```

#### 2、在主库查看二进制文件和偏移量（master节点）

```bash
mysql

show master status;
```

#### 3、slave1,slave2 执行同步操作

```bash
change master to 
master_host='192.168.96.136', 
master_user='slave', 
master_password='123456', 
master_log_file='mysql-bin.000001', 
master_log_pos=154;

start slave;

#两个slave节点都需要 IO线程和 SQL 线程为yes状态
show slave status \G

...........
     Slave_IO_Running: Yes
     Slave_SQL_Running: Yes
.........
```

![image-20221011210551104](https://img.beyourself.org.cn/image-20221011210551104.png)

#### 4、两个从库都设置为只读模式

```bash
#通过全局变量 read_only设置。设置值为1，或者on，表示开启。设置值为0或者off，表示关闭
mysql

set global read_only=1;
show  global variables like 'read_only';
# 【注意】得退出客户端再登录才会生效
```



## 三、MHA概述

> MHA（Master High Availability）是由日本人yoshinorim开发的一款成熟且开源的MySQL高可用程序，它实现了MySQL主从环境下MASTER宕机后能够自动进行单次故障转移的功能，其本身由perl语言编写，安装方便使用简单。

MHA官网：https://code.google.com/archive/p/mysql-master-ha/
GitHub地址：https://github.com/yoshinorim/mha4mysql-manager
文档：https://github.com/yoshinorim/mha4mysql-manager/wiki

## 四、MHA架构

当一个 master 崩溃时，MHA 会恢复下面的 rest slave。

![img](https://img.beyourself.org.cn/1601821-20220720215905161-1671408365.png)

## 五、MHA 组件

MHA 由 MHA Manager 和 MHA Node 组成，如下所示：

![img](https://img.beyourself.org.cn/1601821-20220720215919036-206371713.png)

- `MHA Manager`有监控MySQL master、控制master故障转移等管理程序。
- MHA 节点具有故障转移辅助脚本，例如解析 MySQL 二进制/中继日志，识别中继日志位置，中继日志应从哪个位置应用到其他从站，将事件应用到目标从站等。MHA 节点在每个 MySQL 服务器上运行。
- 当 MHA Manager 进行故障转移时，MHA Manager 通过 SSH 连接 `MHA Node` 并在需要时执行 MHA Node 命令。

## 六、安装MHA软件

下载地址：https://github.com/yoshinorim/mha4mysql-manager/wiki/Downloads

### 1）所有节点安装MHA node软件

下载地址：https://github.com/yoshinorim/mha4mysql-node/releases/tag/v0.58

```bash
cd /opt/mysql-mha
#注意，所有节点都需要安装MHA node
#1、先安装相关依赖：
yum install perl-DBD-MySQL -y
# 下载
wget https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
#2、安装mha：
rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm
# 也可以使用下面方式安装
# yum install -y mha4mysql-node-0.58-0.el7.centos.noarch.rpm
```

### 2） 安装mha manager（110节点上）

#### 1、安装mha manager

```bash
# 先安装好依赖
yum -y install epel-release
yum -y install perl-Config-Tiny perl-Time-HiRes perl-Parallel-ForkManager perl-Log-Dispatch perl-DBD-MySQL ncftp

# https://github.com/yoshinorim/mha4mysql-manager/releases/tag/v0.58
wget https://github.com/yoshinorim/mha4mysql-manager/releases/download/v0.58/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm

# 安装
rpm -ivh mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
```

#### 2、 编写master_ip_failover脚本

`/opt/mysql-mha/master_ip_failover`，下面配置文件中会用到

```bash
#!/usr/bin/env perl
use strict;
use warnings FATAL => 'all';
 
use Getopt::Long;
 
my (
    $command, $orig_master_host, $orig_master_ip,$ssh_user,
    $orig_master_port, $new_master_host, $new_master_ip,$new_master_port,
    $orig_master_ssh_port,$new_master_ssh_port,$new_master_user,$new_master_password
);
 
# 这里定义的虚拟IP配置要注意，这个ip必须要与你自己的集群在同一个网段，否则无效
my $vip = '192.168.96.200/24';
my $key = '1';
# 这里的网卡名称 “ens33” 需要根据你机器的网卡名称进行修改
# 如果多台机器直接的网卡名称不统一，有两种方式，一个是改脚本，二是把网卡名称修改成统一
# 我这边实际情况是修改成统一的网卡名称
my $ssh_start_vip = "sudo /sbin/ifconfig ens33:$key $vip";
my $ssh_stop_vip = "sudo /sbin/ifconfig ens33:$key down";
my $ssh_Bcast_arp= "sudo /sbin/arping -I bond0 -c 3 -A $vip";

GetOptions(
    'command=s'          => \$command,
    'ssh_user=s'         => \$ssh_user,
    'orig_master_host=s' => \$orig_master_host,
    'orig_master_ip=s'   => \$orig_master_ip,
    'orig_master_port=i' => \$orig_master_port,
    'orig_master_ssh_port=i' => \$orig_master_ssh_port,
    'new_master_host=s'  => \$new_master_host,
    'new_master_ip=s'    => \$new_master_ip,
    'new_master_port=i'  => \$new_master_port,
    'new_master_ssh_port' => \$new_master_ssh_port,
    'new_master_user' => \$new_master_user,
    'new_master_password' => \$new_master_password
 
);
 
exit &main();
 
sub main {
    $ssh_user = defined $ssh_user ? $ssh_user : 'root';
    print "\n\nIN SCRIPT TEST====$ssh_user|$ssh_stop_vip==$ssh_user|$ssh_start_vip===\n\n";
 
    if ( $command eq "stop" || $command eq "stopssh" ) {
 
        my $exit_code = 1;
        eval {
            print "Disabling the VIP on old master: $orig_master_host \n";
            &stop_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn "Got Error: $@\n";
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "start" ) {
 
        my $exit_code = 10;
        eval {
            print "Enabling the VIP - $vip on the new master - $new_master_host \n";
            &start_vip();
        &start_arp();
            $exit_code = 0;
        };
        if ($@) {
            warn $@;
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "status" ) {
        print "Checking the Status of the script.. OK \n";
        exit 0;
    }
    else {
        &usage();
        exit 1;
    }
}
 
sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
sub stop_vip() {
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}
 
sub start_arp() {
    `ssh $ssh_user\@$new_master_host \" $ssh_Bcast_arp \"`;
}
sub usage {
    print
    "Usage: master_ip_failover --command=start|stop|stopssh|status --ssh_user=user --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}
```

给该脚本添加可执行权限：

```bash
chmod a+x /opt/mysql-mha/master_ip_failover
```

#### 3、 配置（manager节点）

```bash
#创建相关目录（所有节点）
mkdir -p /opt/mysql-mha/mha-node
# manager节点
mkdir -p /opt/mysql-mha/mha
#编写配置文件
vim /opt/mysql-mha/mysql_mha.cnf
#内容如下:
------------------------------------------------------------------------
[server default]
#mha访问数据库的账号与密码
user=mha
password=123456
port=3306
#指定mha的工作目录
manager_workdir=/opt/mysql-mha/mha
#指定管理日志路径
manager_log=/opt/mysql-mha/manager.log
#指定master节点存放binlog的日志文件的目录 log_bin=mysql_bin默认是在/var/lib/mysql
master_binlog_dir=/var/lib/mysql
#指定mha在远程节点上的工作目录
remote_workdir=/opt/mysql-mha/mha-node
#指定主从复制的mysq用户和密码
repl_user=slave
repl_password=123456
#指定检测间隔时间
ping_interval=1
#指定一个脚本，该脚本实现了在主从切换之后，将虚拟ip漂移到新的master上
master_ip_failover_script=/opt/mysql-mha/master_ip_failover
#指定检查的从服务器IP地址.有几个，就用-s选项加几个
secondary_check_script=/usr/bin/masterha_secondary_check -s 192.168.96.142 -s 192.168.96.143
#用于故障切换的时候发送邮件提醒
#report_script=/data1/mysql-mha/send_mail
[server1]
hostname=192.168.96.136
port=3306
ssh_user=root
candidate_master=1
check_repl_delay=0
[server2]
hostname=192.168.96.142
port=3306
ssh_user=root
candidate_master=1
check_repl_delay=0
[server3]
hostname=192.168.96.143
port=3306
ssh_user=root
candidate_master=1
check_repl_delay=0
```

**candidate_master=1**

> 设置为候选master，设置该参数以后，发生主从切换以后将会将此从库提升为主库，即使这个从库不是集群中最新的slave，no_master=1正好相反

**check_repl_delay=0**

> 默认情况下如果一个slave落后master 超过100M的relay logs的话，MHA将不会选择该slave作为一个新的master， 因为对于这个slave的恢复需要花费很长时间；通过设置check_repl_delay=0，MHA触发切换在选择一个新的master的时候将会忽略复制延时，这个参数对于设置了candidate_master=1的主机非常有用，因为这个候选主在切换的过程中一定是新的master

### 5）在master上手动启动虚拟iP

第一次配置需要在master节点上手动启动虚拟IP，标签要和master_ip_faioverl配置文件中my $key = '1'; 一样

```bash
/sbin/ifconfig ens33:1 192.168.96.200/24
```

### 6）在manager 节点测试ssh 无密认证

```bash
masterha_check_ssh   -conf=/opt/mysql-mha/mysql_mha.cnf
```

![image-20221011211008551](https://img.beyourself.org.cn/image-20221011211008551.png)

### 7）在manager 节点上测试mysql主从情况

```bash
masterha_check_repl -conf=/opt/mysql-mha/mysql_mha.cnf
```

![image-20221011212043467](https://img.beyourself.org.cn/image-20221011212043467.png)

### 8）在manage上启动mha

```bash
nohup masterha_manager  \
--conf=/opt/mysql-mha/mysql_mha.cnf \
--remove_dead_master_conf \
--ignore_last_failover < /dev/null > /var/log/mha_manager.log 2>&1 &
```

- `--remove_dead_master_conf`：该参数代表当发生主从切换后，老的主库的 ip 将会从配置文件中移除。
- `--manger_log`：日志存放位置。
- `--ignore_last_failover`：在缺省情况下，如果 MHA 检测到连续发生宕机，且两次宕机间隔不足 8 小时的话，则不会进行 Failover， 之所以这样限制是为了避免 ping-pong 效应。该参数代表忽略上次 MHA 触发切换产生的文件，默认情况下，MHA 发生切换后会在日志记目录，也就是上面设置的日志app1.failover.complete文件，下次再次切换的时候如果发现该目录下存在该文件将不允许触发切换，除非在第一次切换后收到删除该文件，为了方便，这里设置为--ignore_last_failover。

### 9）查看MHA状态

```bash
masterha_check_status --conf=/opt/mysql-mha/mysql_mha.cnf
```

![image-20221011212129910](https://img.beyourself.org.cn/image-20221011212129910.png)

### 10）查看MHA日志文件

```bash
 cat /opt/mysql-mha/manager.log | grep "current master"
```

![image-20221011212156588](https://img.beyourself.org.cn/image-20221011212156588.png)

### 11）manager节点关闭manager服务

```bash
masterha_stop --conf=/opt/mysql-mha/mysql_mha.cnf
```

![image-20221011212350934](https://img.beyourself.org.cn/image-20221011212350934.png)



## 七、故障模拟与恢复

### 1）停掉mysql master

```bash
systemctl stop mysqld
```

### 2）查看主备节点

```bash
# 查看虚拟ip
ip a
```

![image-20221011212533704](https://img.beyourself.org.cn/image-20221011212533704.png)

```bash
# 登录mysql查看主从关系
mysql -uroot -p
123456
# 发现143这个节点已经变成主节点了
```

![image-20221011212611349](https://img.beyourself.org.cn/image-20221011212611349.png)

![image-20221011212630888](https://img.beyourself.org.cn/image-20221011212630888.png)

### 3）故障恢复

先在当前的主库服务器slave1上查看二进制日志和同步点

```bash
mysql -uroot -p
123456

show master status;
```

再在**原master**服务器上执行同步操作

```bash
# 先恢复mysql服务
systemctl start mysqld

mysql -uroot -p
123456

# 指向新的master节点进行同步
change master to
  master_host='192.168.96.142',
  master_user='slave',
  master_password='123456',
  master_log_file='mysql-bin.000001',
  master_log_pos=154;

start slave;

# 这里需要过一段时间再看同步状态
show slave status\G

# 如果期间没有数据产生，可以直接查看主从状态，为正常状态且以转移
```

![image-20221011212941456](https://img.beyourself.org.cn/image-20221011212941456.png)