# kubernetes 二进制高可用安装部署（v1.23+）

[TOC]



## **一丶高可用集群划分**

####     1. 集群规划

| 配置信息    | 备注          |
| ----------- | ------------- |
| 系统版本    | CentOS 7.9    |
| Docker版本  | 20.10.x       |
| Pod网段     | 172.16.0.0/16 |
| Service网段 | 10.96.0.0/16  |

⚠️ ***宿主机网段、K8s Service网段、Pod网段不能重复***

| role                                    | ipaddress      | configure                        |
| --------------------------------------- | -------------- | -------------------------------- |
| k8s-master-lb   (VIP )                  | 192.168.174.66 |                                  |
| k8s-master-01（etcd haproxy keepalived) | 192.168.174.30 | 4 core, 4Gb; 50GBS, CentOS 7.9   |
| k8s-master-02（etcd haproxy keepalived) | 192.168.174.31 | 4 core, 4Gb; 50GBS, CentOS 7.9   |
| k8s-master-03（etcd haproxy keepalived) | 192.168.174.32 | 4 core, 4Gb; 50GBS, CentOS 7.9   |
| k8s-worker-01                           | 192.168.174.40 | 4 core, 16Gb; 100GBS, CentOS 7.9 |
| k8s-worker-02                           | 192.168.174.41 | 4 core, 16Gb; 100GBS, CentOS 7.9 |

**⚠️Master节点的IP地址尽量不要和Node节点一个网段，方便后期因为业务增长Node节点需要扩充（如master节点192.168.100.10段，node节点192.168.100.20段）**

####     2.配置所有节点的hosts文件（Master and Node）

```bash
$ git clone https://gitee.com/BRWYZ/kubernetes_install.git

$ cat <<-EOF >>/etc/hosts
192.168.58.149     m1
192.168.58.150     n1
EOF
```

####     3.安装YUM源（Master and Node）

```bash
#>>> 下载阿里云的YUM源
$ sed -e 's|^mirrorlist=|#mirrorlist=|g' \
    -e 's|^#baseurl=http://mirror.centos.org/centos|baseurl=https://mirrors.ustc.edu.cn/centos|g' \
    -i.bak \
    /etc/yum.repos.d/CentOS-Base.repo

#>>> 安装插件
$ yum install -y yum-utils device-mapper-persistent-data lvm2

#>>> 安装Docker源并修改安装地址
$ yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

#>>> 安装epel源
$ yum -y install epel-release

#>>> 安装工具
$ yum install wget jq psmisc vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git install ipvsadm ipset sysstat conntrack libseccomp  -y

```

#### 	4.关闭系统中不需要的服务 （Master and Node）

```bash
#>>> 关闭防火墙
systemctl disable --now firewalld

#>>> 关闭安全策略
setenforce 0
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/sysconfig/selinux
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config

#>>> 关闭dnsmasq（关闭失败报错正常）
systemctl disable --now dnsmasq

#>>> 关闭NetworkManager
systemctl disable --now NetworkManager

#>>> 关闭swap分区
swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab
```

#### 	5.同步时间区（Master and Node）

```bash
#>>> 下载同步时间服务,etcd集群同步需要
$ yum install -y ntpdate

#>>> 每五分钟执行一次
$ echo "*/5 * * * *        ntpdate -b ntp.aliyun.com" >>/var/spool/cron/root

#>>> 修改地区
$ ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
$ echo 'Asia/Shanghai' >/etc/timezone
```

#### 	6.所有节点配置limit（Master and Node）

```bash
#>>> 修改limit
ulimit -SHn 65535

#>>> 加入开机自启
cat <<-EOF >>/etc/security/limits.conf
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
EOF
```

#### 	7.节点之间进行免密（方便后期文件传输）（Master01）

```bash
#>>> 生成公钥和私钥
$ ssh-keygen -t rsa

#>>> 传送公钥
$ for i in k8s-master02 k8s-master03 k8s-node01 k8s-node02;do ssh-copy-id -i .ssh/id_rsa.pub $i;done
```

#### 	8. 下载需要文件（Master01）

```bash
$ cd /root/
$ git clone git@gitee.com:BRWYZ/kubernetes_install.git
$ cd kubernetes_install
```

#### 	9.内核升级（kubernetes官方认证kernel v4.19）（Master and Node）

```bash
#>>> 升级rpm包除内核外
$ yum update -y --exclude=kernel*


#>>> 传输内核包
$ for i in k8s-master02 k8s-master03 k8s-node01 k8s-node02;do scp kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm $i:/root/ ; done

#>>> 所有节点安装内核包
$ yum localinstall -y kernel-ml*

#>>> 所有节点更改内核启动顺序
$ grub2-set-default  0 && grub2-mkconfig -o /etc/grub2.cfg
$ grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"

#>>> 查看内核版本是否改变
$ grubby --default-kernel  /boot/vmlinuz-4.19.12-1.el7.elrepo.x86_64
```

#### 	10.节点安装ipvs（Master and Node）

```bash
#>>> 各个节点配置ipvs模块，内核4.19+版本nf_conntrack_ipv4已经改为nf_conntrack，4.18以下使用nf_conntrack_ipv4。
$ modprobe -- ip_vs
  modprobe -- ip_vs_rr
  modprobe -- ip_vs_wrr
  modprobe -- ip_vs_sh
  modprobe -- nf_conntrack

#>>> 添加到开机自启
$ cat <<-EOF >>/etc/modules-load.d/ipvs.conf
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF

#>>> 启动服务(启动报错忽略)
$ systemctl enable --now systemd-modules-load.service

#>>> 查看是否加载
$ lsmod | grep -e ip_vs -e nf_conntrack
nf_conntrack_ipv4      16384  23 
nf_defrag_ipv4         16384  1 nf_conntrack_ipv4
nf_conntrack          135168  10 xt_conntrack,nf_conntrack_ipv6,nf_conntrack_ipv4,nf_nat,nf_nat_ipv6,ipt_MASQUERADE,nf_nat_ipv4,xt_nat,nf_conntrack_netlink,ip_vs  
```

#### 	11. 配置k8s集群中必须的内核参数(Master and Node)

```bash
$ cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
net.ipv4.conf.all.route_localnet = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
EOF

#>>> 加载参数
$ sysctl --system
```

#### 	12.重启服务器

```bash
$ reboot
#>>> 查看内核是否加载
$ lsmod | grep --color=auto -e ip_vs -e nf_conntrack

[root@k8s-master01 ~]# lsmod | grep --color=auto -e ip_vs -e nf_conntrack
ip_vs_ftp              16384  0 
nf_nat                 32768  1 ip_vs_ftp
ip_vs_sed              16384  0 
ip_vs_nq               16384  0 
ip_vs_fo               16384  0 
ip_vs_sh               16384  0 
ip_vs_dh               16384  0 
ip_vs_lblcr            16384  0 
ip_vs_lblc             16384  0 
ip_vs_wrr              16384  0 
ip_vs_rr               16384  0 
ip_vs_wlc              16384  0 
ip_vs_lc               16384  0 
ip_vs                 151552  24 ip_vs_wlc,ip_vs_rr,ip_vs_dh,ip_vs_lblcr,ip_vs_sh,ip_vs_fo,ip_vs_nq,ip_vs_lblc,ip_vs_wrr,ip_vs_lc,ip_vs_sed,ip_vs_ftp
nf_conntrack          143360  2 nf_nat,ip_vs
nf_defrag_ipv6         20480  1 nf_conntrack
nf_defrag_ipv4         16384  1 nf_conntrack
libcrc32c              16384  4 nf_conntrack,nf_nat,xfs,ip_vs

```

------



## **二丶基本组件安装**

#### 	1.安装Rumtime（Master and Node）

```bash
#>>> 查看docker版本
$ yum list docker-ce.x86_64 --showduplicates | sort -r

#>>> 所有节点安装docker-ce-20.10
$ yum install docker-ce-20.10.* docker-ce-cli-20.10.* containerd.io -y

#>>> 新版Kubelet建议使用systemd，Docker的CgroupDriver也改成systemd
$ mkdir /etc/docker
$ cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

#>>> 重新加载配置文件并启动docker
$ systemctl daemon-reload && systemctl enable --now docker
```



#### 	2.Kubernetes组件及etcd安装(Master01)

```bash
#>>> 下载kubernetes组件包
[root@k8s-master01 ~]# wget https://dl.k8s.io/v1.23.17/kubernetes-server-linux-amd64.tar.gz
#>>> 下载etcd的安装包
[root@k8s-master01 ~]# wget https://github.com/etcd-io/etcd/releases/download/v3.5.6/etcd-v3.5.6-linux-amd64.tar.gz

#>>> 解压kubernetes安装包并安装文件
[root@k8s-master01 ~]# tar -xf kubernetes-server-linux-amd64.tar.gz  --strip-components=3 -C /usr/local/bin kubernetes/server/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}

[root@k8s-master01 ~]# ls /usr/local/bin/
kube-apiserver  kube-controller-manager  kubectl  kubelet  kube-proxy  kube-scheduler


#>>> 解压etcd安装包并安装文件
[root@k8s-master01 ~]# tar -zxvf etcd-v3.5.6-linux-amd64.tar.gz --strip-components=1 -C /usr/local/bin etcd-v3.5.6-linux-amd64/etcd{,ctl}
etcd-v3.5.6-linux-amd64/etcdctl
etcd-v3.5.6-linux-amd64/etcd


#>>> 查看kubelet和etcd版本信息
[root@k8s-master01 ~]# kubelet --version
Kubernetes v1.23.16


[root@k8s-master01 ~]# etcd --version
etcd Version: 3.5.6
Git SHA: cecbe35ce
Go Version: go1.16.15
Go OS/Arch: linux/amd64


#>>> 将文件传输给其他节点
[root@k8s-master01 ~]# MasterNodes='k8s-master02 k8s-master03'
[root@k8s-master01 ~]# WorkNodes='k8s-node01 k8s-node02'

[root@k8s-master01 ~]# for NODE in $MasterNodes; do echo $NODE; scp /usr/local/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy} $NODE:/usr/local/bin/; scp /usr/local/bin/etcd* $NODE:/usr/local/bin/; done
k8s-master02
kubelet                                                              100%  114MB  22.7MB/s   00:05    
kubectl                                                              100%   43MB  30.6MB/s   00:01    
kube-apiserver                                                       100%  120MB  30.0MB/s   00:04    
kube-controller-manager                                              100%  111MB  30.1MB/s   00:03    
kube-scheduler                                                       100%   46MB  21.1MB/s   00:02    
kube-proxy                                                           100%   41MB  25.6MB/s   00:01    
etcd                                                                 100%   23MB  32.0MB/s   00:00    
etcdctl                                                              100%   17MB  32.8MB/s   00:00    
k8s-master03
kubelet                                                              100%  114MB  25.0MB/s   00:04    
kubectl                                                              100%   43MB  24.1MB/s   00:01    
kube-apiserver                                                       100%  120MB  33.6MB/s   00:03    
kube-controller-manager                                              100%  111MB  31.3MB/s   00:03    
kube-scheduler                                                       100%   46MB  22.9MB/s   00:02    
kube-proxy                                                           100%   41MB  24.3MB/s   00:01    
etcd                                                                 100%   23MB  22.6MB/s   00:01    
etcdctl                                                              100%   17MB  26.3MB/s   00:00    
[root@k8s-master01 ~]# for NODE in $WorkNodes; do  scp /usr/local/bin/kube{let,-proxy} $NODE:/usr/local/bin/; done
kubelet                                                              100%  114MB  33.6MB/s   00:03    
kube-proxy                                                           100%   41MB  40.7MB/s   00:01    
kubelet                                                              100%  114MB  29.6MB/s   00:03    
kube-proxy                                                           100%   41MB  37.3MB/s   00:01 
```

#### 	3.创建cni的目录（Master and Node）

```bash
#>>> 创建目录(calion需要用到)
$ mkdir -p /opt/cni/bin/

#>>> Master01切换分支
$ cd /root/kubernetes_install &&  git checkout v1.23+
```

------

## 三丶生成证书

#### 	1.下载kubernetes证书（Master01）

```bash
#>>> 下载证书
$ wget "https://pkg.cfssl.org/R1.2/cfssl_linux-amd64" -O /usr/local/bin/cfssl

$ wget "https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64" -O /usr/local/bin/cfssljson

https://github.com/cloudflare/cfssl/releases/download/1.2.0/cfssl_linux-amd64

https://github.com/cloudflare/cfssl/releases/download/1.2.0/cfssljson_linux-amd64

#>>> 添加执行权限
$ chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson


# 或者
[root@k8s-master01 ~]# mv cfssl_linux-amd64 /usr/local/bin/cfssl
[root@k8s-master01 ~]# mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
[root@k8s-master01 ~]# chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson

```

#### 	2.生成etcd的证书

**集群内部的通讯，ipaserver连接etcd的证书，类似于传统的公钥私钥的解密加密**

```bash
#>>> 所有master节点创建etcd的证书目录
$ mkdir /etc/etcd/ssl -p
$ ll /etc/etcd/ssl/
总用量 0

#>>> 创建kubernetes放置证书目录，全部节点master节点以及node节点
$ mkdir -p /etc/kubernetes/pki
$ ll /etc/kubernetes/pki/
总用量 0
```

###### 	 (1)Master01执行

```bash
#>>> 切换工作目录
[root@k8s-master01 ~]# cd /root/kubernetes_install/pki/
[root@k8s-master01 pki]# ls
admin-csr.json      ca-csr.json       front-proxy-ca-csr.json      kube-proxy-csr.json
apiserver-csr.json  etcd-ca-csr.json  front-proxy-client-csr.json  manager-csr.json
ca-config.json      etcd-csr.json     kubelet-csr.json             scheduler-csr.json


#>>> 生成etcd CA证书和CA证书的key，根证书
[root@k8s-master01 pki]# cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare /etc/etcd/ssl/etcd-ca
2024/06/16 00:58:35 [INFO] generating a new CA key and certificate from CSR
2024/06/16 00:58:35 [INFO] generate received request
2024/06/16 00:58:35 [INFO] received CSR
2024/06/16 00:58:35 [INFO] generating key: rsa-2048
2024/06/16 00:58:36 [INFO] encoded CSR
2024/06/16 00:58:36 [INFO] signed certificate with serial number 667846601631781419691753527881959005353534436105


#>>> 生成客户端密钥
[root@k8s-master01 pki]# cfssl gencert \
    -ca=/etc/etcd/ssl/etcd-ca.pem \
    -ca-key=/etc/etcd/ssl/etcd-ca-key.pem \
    -config=ca-config.json \
    -hostname=127.0.0.1,k8s-master01,k8s-master02,k8s-master03,192.168.174.30,192.168.174.31,192.168.174.32 \
    -profile=kubernetes \
    etcd-csr.json | cfssljson -bare /etc/etcd/ssl/etcd
2024/06/16 01:03:08 [INFO] generate received request
2024/06/16 01:03:08 [INFO] received CSR
2024/06/16 01:03:08 [INFO] generating key: rsa-2048
2024/06/16 01:03:09 [INFO] encoded CSR
2024/06/16 01:03:09 [INFO] signed certificate with serial number 169670736440422945332067825308657235294205909760

⚠️: 如果etcd没有部署在kubernetes的Master节点上，注意修改主机名和IP地址，如果将来需要增加etcd节点，可以提前滞留几个IP地址，方便后期使用

#>>> 查看生成的证书
[root@k8s-master01 pki]# ls /etc/etcd/ssl/
etcd-ca.csr  etcd-ca-key.pem  etcd-ca.pem  etcd.csr  etcd-key.pem  etcd.pem


#>>> 将etcd的证书传输给其他Master节点
[root@k8s-master01 pki]# MasterNodes='k8s-master02 k8s-master03'
[root@k8s-master01 pki]# WorkNodes='k8s-node01 k8s-node02'
[root@k8s-master01 pki]# echo $MasterNodes $WorkNodes
k8s-master02 k8s-master03 k8s-node01 k8s-node02

$ for NODE in $MasterNodes; do
     ssh $NODE "mkdir -p /etc/etcd/ssl"
     for FILE in etcd-ca-key.pem  etcd-ca.pem  etcd-key.pem  etcd.pem; do
       scp /etc/etcd/ssl/${FILE} $NODE:/etc/etcd/ssl/${FILE}
     done
 done
 
etcd-ca-key.pem                                                      100% 1675   793.9KB/s   00:00    
etcd-ca.pem                                                          100% 1367   889.8KB/s   00:00    
etcd-key.pem                                                         100% 1675   802.4KB/s   00:00    
etcd.pem                                                             100% 1509     1.0MB/s   00:00    
etcd-ca-key.pem                                                      100% 1675   740.1KB/s   00:00    
etcd-ca.pem                                                          100% 1367   878.3KB/s   00:00    
etcd-key.pem                                                         100% 1675   565.2KB/s   00:00    
etcd.pem                                                             100% 1509     1.1MB/s   00:00 
```

#### 	3.生成kubernetes组件证书（Master01）

###### 		(1)生成根证书，然后根据根证书生成其他组件证书。

```bash
#>>> 切换目录
[root@k8s-master01 pki]# cd /root/kubernetes_install/pki

#>>> 生成根证书（统一证书，其他组件的证书都是围绕这个根证书生成了，方便管理，（ca证书））
$ cfssl gencert -initca ca-csr.json | cfssljson -bare /etc/kubernetes/pki/ca
	# 执行结果：
 	[root@k8s-master01 pki]# cfssl gencert -initca ca-csr.json | cfssljson -bare /etc/kubernetes/pki/ca
2022/12/05 10:34:00 [INFO] generating a new CA key and certificate from CSR
2022/12/05 10:34:00 [INFO] generate received request
2022/12/05 10:34:00 [INFO] received CSR
2022/12/05 10:34:00 [INFO] generating key: rsa-2048
2022/12/05 10:34:02 [INFO] encoded CSR
2022/12/05 10:34:02 [INFO] signed certificate with serial number 9045593708449349597042729397661311913120732807

#>>> 查看证书是否生成
$ ls /etc/kubernetes/pki/
	# 执行结果：
   [root@k8s-master01 pki]# ls /etc/kubernetes/pki/
ca.csr  ca-key.pem  ca.pem
```

###### 		(2)生成apiserver证书

​	**⚠️：10.96.0.1 是k8s service的网段的第一个IP地址，如果说需要更改k8s service网段，那就需要更改10.96.0.1，如果不是高可用集群，`192.168.174.66`为`kube-Master01`的IP（修改前请注意网段的掩码位，去掩码计算器计算，查找第一个网段ip，`192.168.174.30,192.168.174.31,192.168.174.32为master节点IP`）**

```bash
#>>> 生成证书
[root@k8s-master01 pki]# cfssl gencert -ca=/etc/kubernetes/pki/ca.pem   -ca-key=/etc/kubernetes/pki/ca-key.pem   -config=ca-config.json   -hostname=10.96.0.1,192.168.174.66,127.0.0.1,kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.default.svc.cluster.local,192.168.174.30,192.168.174.31,192.168.174.32  -profile=kubernetes   apiserver-csr.json | cfssljson -bare /etc/kubernetes/pki/apiserver
	# 执行结果：
2024/06/16 01:14:35 [INFO] generate received request
2024/06/16 01:14:35 [INFO] received CSR
2024/06/16 01:14:35 [INFO] generating key: rsa-2048
2024/06/16 01:14:35 [INFO] encoded CSR
2024/06/16 01:14:35 [INFO] signed certificate with serial number 664912171212661285532228936561112324599278915308
# ⚠️：10.96.0.1 service网段第一个IP地址(10.96.0.0/12)
# ⚠️: 192.168.174.66 keepalived的VIP
# ⚠️: 192.168.174.30 Master01的IP
# ⚠️: 192.168.174.31 Master02的IP
# ⚠️: 192.168.174.32 Master03的IP

#>>> 查看证书
[root@k8s-master01 pki]# ls /etc/kubernetes/pki/
apiserver.csr  apiserver-key.pem  apiserver.pem  ca.csr  ca-key.pem  ca.pem

#>>> 生成apiserver的聚合证书（权限，让第三方组件和kube-apiserver进行通信拿去数据）
[root@k8s-master01 pki]# cfssl gencert   -initca front-proxy-ca-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-ca  
	# 返回结果：
2024/06/16 01:17:04 [INFO] generating a new CA key and certificate from CSR
2024/06/16 01:17:04 [INFO] generate received request
2024/06/16 01:17:04 [INFO] received CSR
2024/06/16 01:17:04 [INFO] generating key: rsa-2048
2024/06/16 01:17:04 [INFO] encoded CSR
2024/06/16 01:17:04 [INFO] signed certificate with serial number 32033324620397301196702805402027812417677313761

[root@k8s-master01 pki]# cfssl gencert   -ca=/etc/kubernetes/pki/front-proxy-ca.pem   -ca-key=/etc/kubernetes/pki/front-proxy-ca-key.pem   -config=ca-config.json   -profile=kubernetes   front-proxy-client-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-client
	# 执行结果：
2024/06/16 01:17:32 [INFO] generate received request
2024/06/16 01:17:32 [INFO] received CSR
2024/06/16 01:17:32 [INFO] generating key: rsa-2048
2024/06/16 01:17:32 [INFO] encoded CSR
2024/06/16 01:17:32 [INFO] signed certificate with serial number 82900029194521146603431036490159053268222994590
2024/06/16 01:17:32 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
# ⚠️: 日志会有警告，忽略不管
```

#### 	4.生成controller-manage的证书(资源控制器)（Master01）

```bash
[root@k8s-master01 pki]# cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   manager-csr.json | cfssljson -bare /etc/kubernetes/pki/controller-manager
	# 执行结果：
2024/06/16 01:18:12 [INFO] generate received request
2024/06/16 01:18:12 [INFO] received CSR
2024/06/16 01:18:12 [INFO] generating key: rsa-2048
2024/06/16 01:18:13 [INFO] encoded CSR
2024/06/16 01:18:13 [INFO] signed certificate with serial number 118932812530777434694501978675305584566356613164
2024/06/16 01:18:13 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

# ⚠️:注意，如果不是高可用集群，192.168.174.66:8443改为master01的地址，8443改为apiserver的端口，默认是6443
#>>> 生成kubeconfig文件，用于和kube-apiserver进行通讯，这样controller-manager就会有自己的权限
$ kubectl config set-cluster kubernetes \
     --certificate-authority=/etc/kubernetes/pki/ca.pem \
     --embed-certs=true \
     --server=https://192.168.174.66:8443 \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
	# 执行结果：
Cluster "kubernetes" set.
    
#>>> 设置一个环境项，一个上下文
$ kubectl config set-context system:kube-controller-manager@kubernetes \
    --cluster=kubernetes \
    --user=system:kube-controller-manager \
    --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
	# 执行结果：
    Context "system:kube-controller-manager@kubernetes" created.

#>>> set-credentials 设置一个用户项
$ kubectl config set-credentials system:kube-controller-manager \
     --client-certificate=/etc/kubernetes/pki/controller-manager.pem \
     --client-key=/etc/kubernetes/pki/controller-manager-key.pem \
     --embed-certs=true \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
	# 执行结果：
	User "system:kube-controller-manager" set.

#>>> 使用某个环境当做默认环境
$ kubectl config use-context system:kube-controller-manager@kubernetes \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
	# 执行结果：
	Switched to context "system:kube-controller-manager@kubernetes".
```

#### 	5.生成kube-scheduler证书（Master01）

```bash
$ cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   scheduler-csr.json | cfssljson -bare /etc/kubernetes/pki/scheduler
	# 执行结果：
2024/06/16 01:27:38 [INFO] generate received request
2024/06/16 01:27:38 [INFO] received CSR
2024/06/16 01:27:38 [INFO] generating key: rsa-2048
2024/06/16 01:27:39 [INFO] encoded CSR
2024/06/16 01:27:39 [INFO] signed certificate with serial number 50664639108365158483981871397139307380658841542
2024/06/16 01:27:39 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").


# ⚠️:注意，如果不是高可用集群，192.168.174.66:8443改为master01的地址，8443改为apiserver的端口，默认是6443
#>>> set-cluster：设置一个集群项
$ kubectl config set-cluster kubernetes \
     --certificate-authority=/etc/kubernetes/pki/ca.pem \
     --embed-certs=true \
     --server=https://192.168.174.66:8443 \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
	# 执行结果：
   Cluster "kubernetes" set.

#>>> 设置一个环境项，一个上下文
$ kubectl config set-credentials system:kube-scheduler \
     --client-certificate=/etc/kubernetes/pki/scheduler.pem \
     --client-key=/etc/kubernetes/pki/scheduler-key.pem \
     --embed-certs=true \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
	# 执行结果：
	User "system:kube-scheduler" set.

#>>> set-credentials 设置一个用户项
$ kubectl config set-context system:kube-scheduler@kubernetes \
     --cluster=kubernetes \
     --user=system:kube-scheduler \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
	# 执行结果：
	Context "system:kube-scheduler@kubernetes" created.

#>>> 使用某个环境当做默认环境
$ kubectl config use-context system:kube-scheduler@kubernetes \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
	# 执行结果：
	Switched to context "system:kube-scheduler@kubernetes".
```

#### 	6. 生成管理员证书，生成管理员权限（Master01）

```bash
#>>> 生成管理员证书
$ cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   admin-csr.json | cfssljson -bare /etc/kubernetes/pki/admin
	# 执行结果：
2024/06/16 01:29:55 [INFO] generate received request
2024/06/16 01:29:55 [INFO] received CSR
2024/06/16 01:29:55 [INFO] generating key: rsa-2048
2024/06/16 01:29:56 [INFO] encoded CSR
2024/06/16 01:29:56 [INFO] signed certificate with serial number 440866322981709851504705917810067389855632400057
2024/06/16 01:29:56 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

   
# ⚠️:注意，如果不是高可用集群，192.168.174.66:8443改为master01的地址，8443改为apiserver的端口，默认是6443
$ kubectl config set-cluster kubernetes     --certificate-authority=/etc/kubernetes/pki/ca.pem     --embed-certs=true     --server=https://192.168.174.66:8443     --kubeconfig=/etc/kubernetes/admin.kubeconfig
	# 执行结果
Cluster "kubernetes" set.

$ kubectl config set-credentials kubernetes-admin     --client-certificate=/etc/kubernetes/pki/admin.pem     --client-key=/etc/kubernetes/pki/admin-key.pem     --embed-certs=true     --kubeconfig=/etc/kubernetes/admin.kubeconfig
	# 执行结果：
User "kubernetes-admin" set.

$ kubectl config set-context kubernetes-admin@kubernetes     --cluster=kubernetes     --user=kubernetes-admin     --kubeconfig=/etc/kubernetes/admin.kubeconfig
	# 执行结果：
Context "kubernetes-admin@kubernetes" created.

$ kubectl config use-context kubernetes-admin@kubernetes     --kubeconfig=/etc/kubernetes/admin.kubeconfig
	# 执行结果：
	Switched to context "kubernetes-admin@kubernetes".

#>>> 查看管理员证书
$ ls /etc/kubernetes/
执行结果：
	[root@k8s-master01 pki]# ls /etc/kubernetes/
admin.kubeconfig  controller-manager.kubeconfig  pki  scheduler.kubeconfig

```

#### 	7.生成密钥对（Master01）

```bash
#>>> 生成私钥
$ openssl genrsa -out /etc/kubernetes/pki/sa.key 2048
	# 执行结果：
Generating RSA private key, 2048 bit long modulus
...+++
....................................................................+++
e is 65537 (0x10001)
# 这个密钥通常用于 Kubernetes 集群中的服务帐户（Service Account）签名，确保服务帐户令牌的安全性。
#>>> 生成公钥
$ openssl rsa -in /etc/kubernetes/pki/sa.key -pubout -out /etc/kubernetes/pki/sa.pub
	# 执行结果：
writing RSA key
# 在 Kubernetes 中，公钥用于验证服务帐户令牌的签名，确保令牌的真实性和完整性。

#>>> 把证书传输到其他master节点
$ for NODE in k8s-master02 k8s-master03; do 
for FILE in $(ls /etc/kubernetes/pki | grep -v etcd); do 
scp /etc/kubernetes/pki/${FILE} $NODE:/etc/kubernetes/pki/${FILE};
done; 
for FILE in admin.kubeconfig controller-manager.kubeconfig scheduler.kubeconfig; do 
scp /etc/kubernetes/${FILE} $NODE:/etc/kubernetes/${FILE};
done;
done
	# 执行结果：
admin.csr                                  100% 1025     1.1MB/s   00:00    
admin-key.pem                              100% 1675   819.8KB/s   00:00    
admin.pem                                  100% 1444   672.3KB/s   00:00    
apiserver.csr                              100% 1029   697.6KB/s   00:00    
apiserver-key.pem                          100% 1679     1.9MB/s   00:00    
apiserver.pem                              100% 1692   875.7KB/s   00:00    
ca.csr                                     100% 1025   211.1KB/s   00:00    
ca-key.pem                                 100% 1675   526.1KB/s   00:00    
ca.pem                                     100% 1411     1.3MB/s   00:00    
controller-manager.csr                     100% 1082     1.5MB/s   00:00    
controller-manager-key.pem                 100% 1679   594.2KB/s   00:00    
controller-manager.pem                     100% 1501   767.7KB/s   00:00    
front-proxy-ca.csr                         100%  891   255.3KB/s   00:00    
front-proxy-ca-key.pem                     100% 1679   626.4KB/s   00:00    
front-proxy-ca.pem                         100% 1143   898.6KB/s   00:00    
front-proxy-client.csr                     100%  903   363.8KB/s   00:00    
front-proxy-client-key.pem                 100% 1675     1.2MB/s   00:00    
front-proxy-client.pem                     100% 1188   692.6KB/s   00:00    
sa.key                                     100% 1675     1.8MB/s   00:00    
sa.pub                                     100%  451   313.4KB/s   00:00    
scheduler.csr                              100% 1058     1.3MB/s   00:00    
scheduler-key.pem                          100% 1679   640.1KB/s   00:00    
scheduler.pem                              100% 1476     1.4MB/s   00:00    
admin.kubeconfig                           100% 6451     2.5MB/s   00:00    
controller-manager.kubeconfig              100% 6587     7.2MB/s   00:00    
scheduler.kubeconfig                       100% 6515     3.3MB/s   00:00    
          
#>>> 查看证书
$ ls /etc/kubernetes/pki/
admin.csr      apiserver.csr      ca.csr      controller-manager.csr      front-proxy-ca.csr      front-proxy-client.csr      sa.key         scheduler-key.pem
admin-key.pem  apiserver-key.pem  ca-key.pem  controller-manager-key.pem  front-proxy-ca-key.pem  front-proxy-client-key.pem  sa.pub         scheduler.pem
admin.pem      apiserver.pem      ca.pem      controller-manager.pem      front-proxy-ca.pem      front-proxy-client.pem      scheduler.csr

#>>> 查看所有节点证书数量
$ ls /etc/kubernetes/pki/ |wc -l
 23
```

------

## 四丶etcd组件配置

**由于二进制部署kubernetes集群是以Linux中`守护进程`的方式启动，需要配置service文件；etcd配置大致相同，注意修改每个Master节点的etcd配置的主机名和IP地址**

####         (1)Master01配置

```bash
$ vim /etc/etcd/etcd.config.yml

name: 'k8s-master01'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://192.168.174.30:2380'
listen-client-urls: 'https://192.168.174.30:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://192.168.174.30:2380'#集群通讯
advertise-client-urls: 'https://192.168.174.30:2379'#客户端连接
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://192.168.174.30:2380,k8s-master02=https://192.168.174.31:2380,k8s-master03=https://192.168.174.32:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
```

#### 			(2)Master02配置

```bash
$ vim /etc/etcd/etcd.config.yml

name: 'k8s-master02'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://192.168.174.31:2380'
listen-client-urls: 'https://192.168.174.31:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://192.168.174.31:2380'
advertise-client-urls: 'https://192.168.174.31:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://192.168.174.30:2380,k8s-master02=https://192.168.174.31:2380,k8s-master03=https://192.168.174.32:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
```

#### 			(3)Master03配置

```bash
$ vim /etc/etcd/etcd.config.yml

name: 'k8s-master03'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://192.168.174.32:2380'
listen-client-urls: 'https://192.168.174.32:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://192.168.174.32:2380'
advertise-client-urls: 'https://192.168.174.32:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://192.168.174.30:2380,k8s-master02=https://192.168.174.31:2380,k8s-master03=https://192.168.174.32:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
```

#### 			(4)三台master创建service文件

```bash
$ vim /usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Service
Documentation=https://coreos.com/etcd/docs/latest/
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd --config-file=/etc/etcd/etcd.config.yml
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Alias=etcd3.service
```

#### 			(5)三台Master节点创建etcd的证书目录

```bash
$ mkdir /etc/kubernetes/pki/etcd
$ ln -s /etc/etcd/ssl/* /etc/kubernetes/pki/etcd/
$ systemctl daemon-reload && systemctl enable --now etcd
	# 执行结果：
Created symlink from /etc/systemd/system/etcd3.service to /usr/lib/systemd/system/etcd.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/etcd.service to /usr/lib/systemd/system/etcd.service.

#>>> 查看日志，查看节点是否有报错
$ tail -f /var/log/messages
执行结果：# 全部是info，则为正常
```

![image-20240618104757337](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406181048498.png)

#### 			(6)三台Master节点查看etcd状态

```bash
$ export ETCDCTL_API=3

$ etcdctl --endpoints="192.168.174.30:2379,192.168.174.31:2379,192.168.174.32:2379" --cacert=/etc/kubernetes/pki/etcd/etcd-ca.pem --cert=/etc/kubernetes/pki/etcd/etcd.pem --key=/etc/kubernetes/pki/etcd/etcd-key.pem  endpoint status --write-out=table
```

![image-20240618104927074](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406181049207.png)

> 看到IS LEADER出现两个false一个true，只有一个主节点，两个从节点，主节点负责写，从节点负责读；当主节点宕机，会选取其他从节点作为主节点。**三节点的etcd集群只能宕机一个**



## 五丶二进制集群高可用配置（三台Master节点；haproxy和keepalived）

⚠️: haproxy配置文件中端口需和签发证书时候的端口号一致（此处是8443）

####        1.三台Master节点安装haproxy和keepalived

```bash
$ yum install -y haproxy keepalived
```

#### 	    2.三台Master节点配置haproxy

```bash
$ vim /etc/haproxy/haproxy.cfg 

global
  maxconn  2000
  ulimit-n  16384
  log  127.0.0.1 local0 err
  stats timeout 30s

defaults
  log global
  mode  http
  option  httplog
  timeout connect 5000
  timeout client  50000
  timeout server  50000
  timeout http-request 15s
  timeout http-keep-alive 15s

frontend k8s-master
  bind 0.0.0.0:8443
  bind 127.0.0.1:8443
  mode tcp
  option tcplog
  tcp-request inspect-delay 5s
  default_backend k8s-master

backend k8s-master
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server k8s-master01    192.168.174.30:6443  check
  server k8s-master02    192.168.174.31:6443  check
  server k8s-master03    192.168.174.32:6443  check
```



#### 		3. 三台Master配置keepalived

**配置keepalived是注意网卡名称，IP地址，VIP**

###### 			（1）Master01

```bash
$ vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5 
    weight -5
    fall 2
    rise 1
}
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    mcast_src_ip 192.168.174.30
    virtual_router_id 51
    priority 101
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.174.66
    }
    track_script {
      chk_apiserver 
} }
```

###### 			（2）Master02

```bash
$ vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5 
    weight -5
    fall 2
    rise 1
 
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    mcast_src_ip 192.168.174.31
    virtual_router_id 51
    priority 100
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.174.66
    }
    track_script {
      chk_apiserver 
} }
```

###### 			（3）Master03

```bash
$ vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2  
    rise 1
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    mcast_src_ip 192.168.174.32
    virtual_router_id 51
    priority 100
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.174.66
    }
    track_script {
      chk_apiserver 
} }
```

#### 		4.健康检查配置脚本（三台Master节点）

```bash
$ vim /etc/keepalived/check_apiserver.sh
#!/bin/bash
err=0
for k in $(seq 1 3)
do
    check_code=$(pgrep haproxy)
    if [[ $check_code == "" ]]; then
        err=$(expr $err + 1)
        sleep 1
        continue
    else
        err=0
        break
    fi
done

if [[ $err != "0" ]]; then
    echo "systemctl stop keepalived"
    /usr/bin/systemctl stop keepalived
    exit 1
else
    exit 0
fi
```

#### 		 5.启动haproxy和keepalived，并检查VIP(三台Master)

```bash
$ chmod +x /etc/keepalived/check_apiserver.sh
$ systemctl daemon-reload
$ systemctl enable --now haproxy keepalived

#>>> 所有测试VIP
$ ping 192.168.174.66
PING 192.168.174.66 (192.168.174.66) 56(84) bytes of data.
64 bytes from 192.168.174.66: icmp_seq=1 ttl=64 time=1.39 ms
64 bytes from 192.168.174.66: icmp_seq=2 ttl=64 time=2.46 ms
64 bytes from 192.168.174.66: icmp_seq=3 ttl=64 time=1.68 ms
64 bytes from 192.168.174.66: icmp_seq=4 ttl=64 time=1.08 ms

$ telnet 192.168.174.66 8443
执行结果：（^成功）
	Trying 192.168.174.66...
Connected to 192.168.174.66.
Escape character is '^]'.
Connection closed by foreign host.
#⚠️: 如果ping不通且telnet没有出现 ^]，则认为VIP不可以，不可在继续往下执行，需要排查keepalived的问题，比如防火墙和selinux，haproxy和keepalived的状态，监听端口等
所有节点查看防火墙状态必须为disable和inactive：systemctl status firewalld
所有节点查看selinux状态，必须为disable：getenforce
master节点查看haproxy和keepalived状态：systemctl status keepalived haproxy
master节点查看监听端口：netstat -lntp
```



## 六丶Kubernetes Master节点组件安装

#### 		 1. 所有节点创建配置文件目录（Master and Node）(service文件目录，kebernetes配置文件所在目录)

```bash
$ mkdir -p /etc/kubernetes/manifests/ /etc/systemd/system/kubelet.service.d /var/lib/kubelet /var/log/kubernetes
```

#### 		2.apiserver配置

**`注意配置文件中service的网段（--service-cluster-ip-range=10.96.0.0/16）注意本文档使用的k8s service网段为10.96.0.0/16，该网段不能和宿主机的网段、Pod网段的重复`**

###### 			  (1)Master01

```bash
$ vim /usr/lib/systemd/system/kube-apiserver.service

[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
      --v=2  \
      --logtostderr=true  \
      --allow-privileged=true  \
      --bind-address=0.0.0.0  \
      --secure-port=6443  \
      --insecure-port=0  \
      --advertise-address=192.168.174.30 \
      --service-cluster-ip-range=10.96.0.0/16  \
      --service-node-port-range=30000-32767  \
      --etcd-servers=https://192.168.174.30:2379,https://192.168.174.31:2379,https://192.168.174.32:2379 \
      --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \
      --etcd-certfile=/etc/etcd/ssl/etcd.pem  \
      --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \
      --client-ca-file=/etc/kubernetes/pki/ca.pem  \
      --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \
      --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \
      --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \
      --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \
      --service-account-key-file=/etc/kubernetes/pki/sa.pub  \
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
      --authorization-mode=Node,RBAC  \
      --enable-bootstrap-token-auth=true  \
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
      --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
      --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
      --requestheader-allowed-names=aggregator  \
      --requestheader-group-headers=X-Remote-Group  \
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \
      --requestheader-username-headers=X-Remote-User
      # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

###### 			(2)Master02

```bash
$ vim  /usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
      --v=2  \
      --logtostderr=true  \
      --allow-privileged=true  \
      --bind-address=0.0.0.0  \
      --secure-port=6443  \
      --insecure-port=0  \
      --advertise-address=192.168.174.31 \
      --service-cluster-ip-range=10.96.0.0/16  \
      --service-node-port-range=30000-32767  \
      --etcd-servers=https://192.168.174.30:2379,https://192.168.174.31:2379,https://192.168.174.32:2379 \
      --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \
      --etcd-certfile=/etc/etcd/ssl/etcd.pem  \
      --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \
      --client-ca-file=/etc/kubernetes/pki/ca.pem  \
      --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \
      --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \
      --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \
      --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \
      --service-account-key-file=/etc/kubernetes/pki/sa.pub  \
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
      --authorization-mode=Node,RBAC  \
      --enable-bootstrap-token-auth=true  \
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
      --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
      --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
      --requestheader-allowed-names=aggregator  \
      --requestheader-group-headers=X-Remote-Group  \
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \
      --requestheader-username-headers=X-Remote-User
      # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

###### 	   (3)Master03

```bash
$ vim  /usr/lib/systemd/system/kube-apiserver.service

[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
      --v=2  \
      --logtostderr=true  \
      --allow-privileged=true  \
      --bind-address=0.0.0.0  \
      --secure-port=6443  \
      --insecure-port=0  \
      --advertise-address=192.168.174.32 \
      --service-cluster-ip-range=10.96.0.0/16  \
      --service-node-port-range=30000-32767  \
      --etcd-servers=https://192.168.174.30:2379,https://192.168.174.31:2379,https://192.168.174.32:2379 \
      --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \
      --etcd-certfile=/etc/etcd/ssl/etcd.pem  \
      --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \
      --client-ca-file=/etc/kubernetes/pki/ca.pem  \
      --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \
      --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \
      --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \
      --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \
      --service-account-key-file=/etc/kubernetes/pki/sa.pub  \
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
      --authorization-mode=Node,RBAC  \
      --enable-bootstrap-token-auth=true  \
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
      --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
      --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
      --requestheader-allowed-names=aggregator  \
      --requestheader-group-headers=X-Remote-Group  \
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \
      --requestheader-username-headers=X-Remote-User
      # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

###### 		   (4)所有Master节点启动apiserver

```bash
#>>> 启动服务
$ systemctl daemon-reload && systemctl enable --now kube-apiserver

#>>> 查看服务的状态
$ systemctl status kube-apiserver
● kube-apiserver.service - Kubernetes API Server
   Loaded: loaded (/usr/lib/systemd/system/kube-apiserver.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2020-08-22 21:26:49 CST; 26s ago 

#>>> 查看日志
$ tail -f /var/log/messages (出现以下日志信息忽略)
```

![image-20240616020025439](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406160200520.png)



####     3. kube-controller-manager配置（三台Master）

​	**注意使用的k8s Pod网段为172.16.0.0/16，该网段不能和宿主机的网段、k8s Service网段的重复，请按需修改**

######               (1)三台Master配置（同时执行，配置相同）

```bash
$ vim /usr/lib/systemd/system/kube-controller-manager.service 

[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
      --v=2 \
      --logtostderr=true \
      --address=127.0.0.1 \
      --root-ca-file=/etc/kubernetes/pki/ca.pem \
      --cluster-signing-cert-file=/etc/kubernetes/pki/ca.pem \
      --cluster-signing-key-file=/etc/kubernetes/pki/ca-key.pem \
      --service-account-private-key-file=/etc/kubernetes/pki/sa.key \
      --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig \
      --leader-elect=true \
      --use-service-account-credentials=true \
      --node-monitor-grace-period=40s \
      --node-monitor-period=5s \
      --pod-eviction-timeout=2m0s \
      --controllers=*,bootstrapsigner,tokencleaner \
      --allocate-node-cidrs=true \
      --cluster-cidr=172.16.0.0/16 \
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem \
      --node-cidr-mask-size=24
      
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

######              (2)三台Master节点启动kube-controller-manager

```bash
$ systemctl daemon-reload  && systemctl enable --now kube-controller-manager

$ tail -f /var/log/messages
```

![image-20240616020438903](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406160204988.png)

#### 	4. Master节点配置 kube-scheduler（三台Master节点）

```bash
$ vim /usr/lib/systemd/system/kube-scheduler.service

[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
      --v=2 \
      --logtostderr=true \
      --address=127.0.0.1 \
      --leader-elect=true \
      --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

#>>> 启动服务
$ systemctl daemon-reload && systemctl enable --now kube-scheduler
$ tail -f /var/log/messages
```

![image-20240616020628347](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406160206422.png)



## 七丶TLS Bootstrapping配置（Master01）

**⚠️:由于工作节点较多，且每个节点的kubelet证书不一致，且kubelet和主机的主机名IP地址存在关系，不能统一的去生成，所以 Bootstrap会自动的为工作节点颁发kubelet证书。**

`其中如下配置按需更改192.168.174.66:8443为VIP地址及端口号`

```bash
#>>> 配置Bootstrap的文件
$ cd /root/kubernetes_install/bootstrap
$ kubectl config set-cluster kubernetes     --certificate-authority=/etc/kubernetes/pki/ca.pem     --embed-certs=true     --server=https://192.168.174.66:8443     --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig
	# 执行结果：
Cluster "kubernetes" set.

$ kubectl config set-credentials tls-bootstrap-token-user     --token=c8ad9c.2e4d610cf3e7426e --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig
	# 执行结果：
User "tls-bootstrap-token-user" set.

$ kubectl config set-context tls-bootstrap-token-user@kubernetes     --cluster=kubernetes     --user=tls-bootstrap-token-user     --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig
	# 执行结果：
Context "tls-bootstrap-token-user@kubernetes" created.

$ kubectl config use-context tls-bootstrap-token-user@kubernetes     --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig
	# 执行结果：
Switched to context "tls-bootstrap-token-user@kubernetes".
```

```bash
#>>> 创建kubectl需要的执行配置文件（不创建；kubectl无法执行，其他节点操作一致）
$ mkdir -p /root/.kube; cp /etc/kubernetes/admin.kubeconfig /root/.kube/config

#>>> 查看集群状态，否则无法继续执行
$ kubectl get cs
	# 执行结果：
   Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok                              
controller-manager   Healthy   ok                              
etcd-0               Healthy   {"health":"true","reason":""}   
etcd-2               Healthy   {"health":"true","reason":""}   
etcd-1               Healthy   {"health":"true","reason":""}

#>>> 查看集群状态
$ kubectl cluster-info
	# 执行结果
Kubernetes control plane is running at https://192.168.174.66:8443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

#>>> 创建bootstrap的token值
$ kubectl create -f bootstrap.secret.yaml
```

![image-20240616021441067](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406160214181.png)

##    1.Node节点配置

### (1)传输其他节点所需要的证书（聚合证书，根证书，bootstrap证书）

```bash
#>>> Master01
$ vim scp.sh
NODE='k8s-master02 k8s-master03 k8s-node01 k8s-node02'
for i in $NODE;do
  ssh $i mkdir -p /etc/kubernetes/pki
   for FILE in pki/ca.pem pki/ca-key.pem pki/front-proxy-ca.pem bootstrap-kubelet.kubeconfig;do
      scp /etc/kubernetes/$FILE $i:/etc/kubernetes/${FILE}
      done
  done

$ bash scp.sh
	# 执行结果：
```

![image-20240618110930009](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406181109422.png)

### 		(2)kubelet配置(Master and Node)

```bash
#>>> 创建需要的文件目录
$ mkdir -p /var/lib/kubelet /var/log/kubernetes /etc/systemd/system/kubelet.service.d /etc/kubernetes/manifests/
#>>> 所有节点配置kubelet service
$ vim  /usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet

Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target

#>>> 如果Runtime为Docker，请使用如下Kubelet的配置：
$ vim /etc/systemd/system/kubelet.service.d/10-kubelet.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig"
Environment="KUBELET_SYSTEM_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_CONFIG_ARGS=--config=/etc/kubernetes/kubelet-conf.yml --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.5"
Environment="KUBELET_EXTRA_ARGS=--node-labels=node.kubernetes.io/node='' "
ExecStart=
ExecStart=/usr/local/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_SYSTEM_ARGS $KUBELET_EXTRA_ARGS
```

### 			(3)kubelet配置文件编写

**如果更改了k8s的service网段，需要更改kubelet-conf.yml 的clusterDNS:配置，改成k8s Service网段的第十个地址，比如10.96.0.10/16**

```bash
$ vim /etc/kubernetes/kubelet-conf.yml

apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: systemd
cgroupsPerQOS: true
clusterDNS:
- 10.96.0.10   #coredns地址
clusterDomain: cluster.local
containerLogMaxFiles: 5
containerLogMaxSize: 10Mi
contentType: application/vnd.kubernetes.protobuf
cpuCFSQuota: true
cpuManagerPolicy: none
cpuManagerReconcilePeriod: 10s
enableControllerAttachDetach: true
enableDebuggingHandlers: true
enforceNodeAllocatable:
- pods
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 20s
hairpinMode: promiscuous-bridge
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
iptablesDropBit: 15
iptablesMasqueradeBit: 14
kubeAPIBurst: 10
kubeAPIQPS: 5
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
registryBurst: 10
registryPullQPS: 5
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
volumeStatsAggPeriod: 1m0s


#>>> 启动服务
$ systemctl daemon-reload && systemctl enable --now kubelet
$ tail -f /var/log/messages
	# 执行结果：
	Dec  5 18:04:04 k8s-master01 kubelet: I1205 18:04:04.737643    6050 cni.go:240] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
Dec  5 18:04:05 k8s-master01 kubelet: E1205 18:04:05.374596    6050 kubelet.go:2391] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
Dec  5 18:04:09 k8s-master01 kubelet: I1205 18:04:09.738224    6050 cni.go:240] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
Dec  5 18:04:10 k8s-master01 kubelet: E1205 18:04:10.379233    6050 kubelet.go:2391] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
Dec  5 18:04:14 k8s-master01 kubelet: I1205 18:04:14.738919    6050 cni.go:240] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
Dec  5 18:04:15 k8s-master01 kubelet: E1205 18:04:15.386167    6050 kubelet.go:2391] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
#由于没有安装CNI查看导致的。等后续安装calico后恢复正常
```

## 		2.kube-proxy配置（Master01）

​	**如果不是高可用集群，192.168.174.66:8443改为master01的地址，8443改为apiserver的端口，默认是6443**

###### 			(1)kube-proxy配置创建

```bash
$ cd /root/kubernetes_install
$ kubectl -n kube-system create serviceaccount kube-proxy

kubectl create clusterrolebinding system:kube-proxy         --clusterrole system:node-proxier         --serviceaccount kube-system:kube-proxy

SECRET=$(kubectl -n kube-system get sa/kube-proxy \
    --output=jsonpath='{.secrets[0].name}')

JWT_TOKEN=$(kubectl -n kube-system get secret/$SECRET \
--output=jsonpath='{.data.token}' | base64 -d)

PKI_DIR=/etc/kubernetes/pki
K8S_DIR=/etc/kubernetes

kubectl config set-cluster kubernetes     --certificate-authority=/etc/kubernetes/pki/ca.pem     --embed-certs=true     --server=https://192.168.174.66:8443     --kubeconfig=${K8S_DIR}/kube-proxy.kubeconfig

kubectl config set-credentials kubernetes     --token=${JWT_TOKEN}     --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config set-context kubernetes     --cluster=kubernetes     --user=kubernetes     --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config use-context kubernetes     --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

#>>> 将kube-proxy配置文件传输给其他集群
$ for NODE in k8s-master02 k8s-master03; do
     scp /etc/kubernetes/kube-proxy.kubeconfig  $NODE:/etc/kubernetes/kube-proxy.kubeconfig
 done

for NODE in k8s-node01 k8s-node02; do
     scp /etc/kubernetes/kube-proxy.kubeconfig $NODE:/etc/kubernetes/kube-proxy.kubeconfig
 done
```

###### 			(2)所有节点添加kube-proxy的配置和service文件（Master and Node）

```bash
#>>> kube-proxy service文件
$ vim /usr/lib/systemd/system/kube-proxy.service

[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-proxy \
  --config=/etc/kubernetes/kube-proxy.yaml \
  --v=2

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

#>>> kube-proxy配置文件
⚠️: 如果更改了集群Pod的网段，需要更改kube-proxy.yaml的clusterCIDR为自己的Pod网段
$ vim /etc/kubernetes/kube-proxy.yaml

apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
  qps: 5
clusterCIDR: 172.16.0.0/16 
configSyncPeriod: 15m0s
conntrack:
  max: null
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  masqueradeAll: true
  minSyncPeriod: 5s
  scheduler: "rr"
  syncPeriod: 30s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
mode: "ipvs"
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
udpIdleTimeout: 250ms

#>>> 启动服务
$ systemctl daemon-reload && systemctl enable --now kube-proxy
执行结果：
	Created symlink /etc/systemd/system/multi-user.target.wants/kube-proxy.service → /usr/lib/systemd/system/kube-proxy.service
#>>> 查看日志是否报错
$ tail -f /var/log/messages
执行结果：
   Dec  5 18:21:54 k8s-master01 kube-proxy: I1205 18:21:54.989375    9811 shared_informer.go:247] Caches are synced for service config
Dec  5 18:21:54 k8s-master01 kube-proxy: I1205 18:21:54.989564    9811 service.go:419] "Adding new service port" portName="default/kubernetes:https" servicePort="10.96.0.1:443/TCP"
Dec  5 18:21:54 k8s-master01 kube-proxy: I1205 18:21:54.993513    9811 shared_informer.go:247] Caches are synced for node config
```

## 八丶安装Calico（Master01）

​        ⚠️: 安装官方推荐版本

```bash
#>>> 切换目录
$ cd /root/kubernetes_install/calico

#>>> 更改calico的网段，改为自己的Pod网段
$ sed -i "s#POD_CIDR#172.16.0.0/16#g" calico.yaml

#>>> 检查网段是自己的Pod网段
$ grep "IPV4POOL_CIDR" calico.yaml  -A 1

#>>> 创建calico
$ kubectl apply -f calico.yaml
	# 执行结果：
service/calico-typha created
deployment.apps/calico-typha created
Warning: policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
poddisruptionbudget.policy/calico-typha created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
poddisruptionbudget.policy/calico-kube-controllers created

#>>> 查看容器状态
$ kubectl get po -n kube-system
NAME                                       READY   STATUS    RESTARTS      AGE
calico-kube-controllers-66686fdb54-mk2g6   1/1     Running   1 (20s ago)   85s
calico-node-8fxqp                          1/1     Running   0             85s
calico-node-8nkfl                          1/1     Running   0             86s
calico-node-pmpf4                          1/1     Running   0             86s
calico-node-vnlk7                          1/1     Running   0             86s
calico-node-xpchb                          1/1     Running   0             85s
calico-typha-67c6dc57d6-259t8              1/1     Running   0             86s
calico-typha-67c6dc57d6-49h5d              1/1     Running   0             86s
calico-typha-67c6dc57d6-rsc7n              1/1     Running   0             86s
⚠️: 如果容器状态异常可以使用kubectl describe 或者kubectl logs查看容器的日志

```

## 九丶安装CoreDNS

###### 		(1)安装官方推荐版本

```bash
#>>> 切换目录
$ cd /root/kubernetes_install

#>>> 更改了k8s service的网段需要将coredns的serviceIP改成k8s service网段的第十个IP
$ COREDNS_SERVICE_IP=`kubectl get svc | grep kubernetes | awk '{print $3}'`0
$ sed -i "s#KUBEDNS_SERVICE_IP#${COREDNS_SERVICE_IP}#g" CoreDNS/coredns.yaml
#>>> 安装CoreDNS
$ kubectl  create -f CoreDNS/coredns.yaml
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
```

###### 		(2)或者安装最新版

```bash
#>>> 更改了k8s service的网段需要将coredns的serviceIP改成k8s service网段的第十个IP
$ COREDNS_SERVICE_IP=`kubectl get svc | grep kubernetes | awk '{print $3}'`0
$ git clone https://github.com/coredns/deployment.git
$ cd deployment/kubernetes
$ ./deploy.sh -s -i ${COREDNS_SERVICE_IP} | kubectl apply -f -
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
查看状态
$ kubectl get po -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS   AGE
coredns-85b4878f78-h29kh   1/1     Running   0          8h
```

## 十丶安装Metrics Server

​      在新版的Kubernetes中系统资源的采集均使用Metrics-server，可以通过Metrics采集节点和Pod的内存、磁盘、CPU和网络的使用率。

```bash
#>>> 切换目录
$ cd /root/kubernetes_install/metrics-server
#>>> 创建服务
$ kubectl  create -f .
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created

#>>> 等待metrics server启动然后查看状态
$ kubectl  top node
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-master01   231m         5%     1620Mi          42%       
k8s-master02   274m         6%     1203Mi          31%       
k8s-master03   202m         5%     1251Mi          32%       
k8s-node01     69m          1%     667Mi           17%       
k8s-node02     73m          1%     650Mi           16%
```

## 十一丶Dashboard

官方GitHub地址：https://github.com/kubernetes/dashboard

```bash
$ cd /root/kubernetes_install/dashboard
#>>> 创建Dashboard
$ kubectl  create -f . 
	# 执行结果：
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created

#>>> 查看Dashborad
$ kubectl get po -n kubernetes-dashboard
	#执行结果：
kubernetes-dashboard-85f59f8ff7-lrj4n        1/1     Running   0          2m15s

#>>> 查看Dashdoard暴露的端口号
$ kubectl get svc -n kubernetes-dashboard
执行结果：
	NAME                        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.96.29.130   <none>        8000/TCP        24m
kubernetes-dashboard        NodePort    10.96.0.121    <none>        443:32117/TCP   24m

#>>> 查看Token值
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

![image-20240616031745035](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406160317117.png)

  **登录Dashboard**

**https://节点IP:32117**

谷歌游览器修改目标

```ini
--test-type --ignore-certificate-errors
```

![image-20240616031925822](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406160319865.png)

## 十二丶集群验证

**查看kube-apiserver的证书有效时间**

```bash
[root@k8s-master01 pki]# openssl x509 -enddate -noout -in /etc/kubernetes/pki/apiserver.pem
notAfter=May 22 17:10:00 2124 GMT
```



```bash
#>>> 安装pod，测试集群连通性
$ cat<<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: busybox
    image: registry.cn-hangzhou.aliyuncs.com/hujiaming/busybox:latest
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF

#>>> 查看kubernetes下的service
$ kubectl get svc

#>>> busybox解析同namespace下的service
$ kubectl exec  busybox -n default -- nslookup kubernetes
执行结果：（成功）
	Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
     Name:      kubernetes
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local

#>>> busybox跨namespace解析kube-system下的namespace
$ kubectl exec  busybox -n default -- nslookup kube-dns.kube-system
执行结果：
	Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
    Name:      kube-dns.kube-system
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

#>>> 每个节点都必须要访问kubernetes的service:443和kube-dns的service:53
$ telnet 10.96.0.1 443   #kubernetes下的service（kubectl get svc）
	# 返回结果：
Trying 10.96.0.1...
Connected to 10.96.0.1.
Escape character is '^]'.

$ telnet 10.96.0.10 53   #kube-system下的service (kubectl get svc -n kube-system)
	# 执行结果：
Trying 10.96.0.10...
Connected to 10.96.0.10.
Escape character is '^]'.
Connection closed by foreign host.

#>>> Pod和Pod之间通信测试
  #>>> 进入一个Pod当中
  $ kubectl exec -it pod名称 -n 命名空间 -- sh/bash
  #>>> ping其他Pod
  / # ping 172.25.244.193
  #>>> ping宿主机
  / # ping 192.168.174.30

#>>> 创建一个deployment副本
$ kubectl create deploy nginx --image=nginx:v1.21 --replicas=1

#>>> 查看创建的副本
$ kubectl get deploy
执行结果：
	NAME    READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES        SELECTOR
nginx   0/1     1            0           7m35s   nginx        nginx:v1.21   app=nginx
#>>> 删除创建的deployment副本
$ kubectl delete deploy nginx
```

**集群验证步骤：**

- **Pod必须解析service，Pod必须能解析跨namespace的service**
- **每个节点都必须要访问kubernetes的service:443和kube-dns的service:53**
- **每个Pod和Pod之间能通信（同namespace能通信，跨namespace能通信，跨主机通信）**



## 十三、kube-master的组件状态

​	**在kubernetes集群中master节点有三个控制组件，其中kube-apiserver为无状态服务，而另外两个组件是有状态服务，这两个组件同时只会运行集群中的任意一个。**

```bash
# 查看kube-controller-manager、kube-scheduler正在运行的节点
[root@k8s-master01 ~]# kubectl get leases -n kube-system
```

![image-20240617194259305](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406171943390.png)
