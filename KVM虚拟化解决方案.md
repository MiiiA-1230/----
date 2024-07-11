## KVM虚拟化解决方案

> *虚拟化使云计算得以发展, 云计算使虚拟化得以进步, 两者相互成就了当今丰富多彩的服务端资源结构*

[TOC]

### 1.1 云计算概述

#### 1.1.1 什么是云计算？

&emsp;&emsp;在维基百科中是这样定义云计算的: 是一种基于互联网的计算方式, 通过这种方式, 共享的软硬件资源和信息可以按需求提供给计算机各种终端和其他设备; 

&emsp;&emsp;以前要完成信息处理, 是需要在一个客观存在的计算机上完成的, 它看得见摸得到。后来随着计算机硬件、网络技术、存储技术的飞速发展, 计算机硬件性能过剩, 因为足够高的性能在大部分时间是被浪费掉的, 并没有参与客观运算; 那如果将资源集中起来, 计算任务去共享、复用集中起来的资源, 将是对资源的极大节省和效率的极大提升; 这就是云计算产生的背景, 它就是将庞大的计算资源集中在某个地方或是某些地方, 而不再是放在身边的计算机了, 用户的每一次计算, 都发生在那个被称为“**☁️云**”的地方



#### 1.1.2 云计算的几种服务模型

&emsp;&emsp;云计算的模型是以服务为导向的, 根据提供的服务层次不同, 可分为：IaaS(Infrastructure as a Service, 基础架构即服务)、PaaS(Platform as a Service, 平台即服务)、SaaS(Software as a Service, 软件即服务)。它们提供的服务越来越抽象, 用户实际控制的范围也越来越小。

&emsp;&emsp;SaaS: 云服务提供商提供给客户直接使用软件服务, 如Google Docs、Microsoft CRM、Salesforce.com等。用户不必自己维护软件本身, 只管使用软件提供的服务。用户为该软件提供的服务付费

&emsp;&emsp;PaaS: 云服务提供商提供给客户开发、运维应用程序的运行环境, 用户负责维护自己的应用程序, 但并不掌控操作系统、硬件以及运作的网络基础架构。如aliyun的RDS-MySQL、RDS-Redis等。平台是指应用程序运行环境(图中的Runtime)。通常, 这类用户在云环境中运维的应用程序会再提供软件服务给他的下级客户。用户为自己的程序的运行环境付费

&emsp;&emsp;IaaS: 用户有更大的自主权, 能控制自己的操作系统、网络连接(虚拟的)、硬件(虚拟的)环境等, 云服务提供商提供的是一个虚拟的主机环境。如aliyun的ECS、腾讯的CMS等。用户为一个主机环境付费



### 1.2 虚拟化技术

#### 1.2.1 什么是虚拟化

​		在计算机领域, 虚拟化指创建某事物的虚拟版本, 包括虚拟的计算机硬件平台、存储设备以及计算机网络资源; 

&emsp;&emsp;虚拟化是一种资源管理技术, 它将计算机的各种实体资源予以抽象和转化, 并提供分割、重新组合, 以达到最大化利用物理资源的目的; 广义上来说, 我们一直以来对物理硬盘所做的逻辑分区、以及后来的LVM, 都可以纳入虚拟化的范畴; 在没有虚拟化之前, 一个物理主机上面只能运行一个操作系统及其之上的一系列运行环境和应用程序, 有了虚拟化技术, 一个物理主机可以被抽象、分割成多个虚拟的逻辑意义上的主机, 向上支撑多个操作系统及其之上的运行环境和应用程序, 则其资源就可以被最大化的利用.

​		Virtual Machine Monitor(VMM, 虚拟机监控器, 也称为Hypervisor)层, 就是为了达到虚拟化而引入的一个软件层。它向下掌控实际的物理资源(相当于原本的操作系统); 向上呈现给虚拟机N份逻辑的资源。为了做到这一点, 就需要将虚拟机对物理资源的访问“偷梁换柱”——截取并重定向, 让虚拟机误以为自己是在独享物理资源。虚拟机监控器运行的实际物理环境, 称为宿主机; 其上虚拟出来的逻辑主机, 称为客户机(Guest Machine)



#### 1.2.2 半虚拟化和全虚拟化

最理想的虚拟化的两个目标如下：

- 客户机完全不知道自己运行在虚拟化环境中, 还以为自己运行在原生环境里。

- 完全不需要VMM介入客户机的运行过程。

**<u>半虚拟化:</u>**

​		半虚拟化则是一种`虚拟化`方式，它要求操作系统修改以支持虚拟硬件，并且在修改后的操作系统上运行虚拟化软件。在半虚拟化中，虚拟机能够通过访问虚拟化层提供的接口来与物理硬件交互，而无需通过硬件仿真的方式进行。

**<u>全虚拟化:</u>**

​		完全虚拟化是一种`全面模拟硬件`的虚拟化方式，它允许多个虚拟机在同一台物理机上运行不同的操作系统，每个虚拟机都可以独立运行，仿佛在独立的物理服务器上运行一样。在完全虚拟化中，虚拟机的操作系统并不知道它是在虚拟机上运行的，因为虚拟机的硬件和操作系统看起来和物理机的硬件和操作系统没有区别。



### 1.3 KVM详解

#### 1.3.1 KVM简介

&emsp;&emsp;KVM全称是Kernel-based Virtual Machine, 即基于内核的虚拟机, 是采用硬件虚拟化技术的全虚拟化解决方案

&emsp;&emsp;KVM最初是由Qumranet公司的Avi Kivity开发的, 作为他们的VDI产品的后台虚拟化解决方案。为了简化开发, Avi Kivity并没有选择从底层开始新写一个Hypervisor, 而是选择了基于Linux kernel, 通过加载模块使Linux kernel本身变成一个Hypervisor。2006年10月, 在先后完成了基本功能、动态迁移以及主要的性能优化之后, Qumranet正式对外宣布了KVM的诞生。同月, KVM模块的源代码被正式纳入Linux kernel, 成为内核源代码的一部分。它以内核模块的形式加载之后, 就将Linux内核变成了一个Hypervisor, 但硬件管理等还是通过Linux kernel来完成的

![img](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407022024670.jpeg)

&emsp;&emsp;一个KVM客户机对应于一个Linux进程, 每个vCPU则是这个进程下的一个线程, 还有单独的处理IO的线程, 也在一个线程组内。所以, 宿主机上各个客户机是由宿主机内核像调度普通进程一样调度的, 即可以通过Linux的各种进程调度的手段来实现不同客户机的权限限定、优先级等功能。
&emsp;&emsp;客户机所看到的硬件设备是QEMU模拟出来的(不包括VT-d透传的设备), 当客户机对模拟设备进行操作时, 由QEMU截获并转换为对实际的物理设备(可能设置都不实际物理的存在)的驱动操作来完成



#### 1.3.2 KVM虚机创建

&emsp;&emsp;由于KVM已经是kernel源代码的一部分, 所以不需要另外安装, 主要是安装QEMU 虚拟化工具; 这里安装不会进行过多介绍, 可以将安装所需要的库克隆到安装主机中即可; 

```bash
#>>> 克隆仓库
[root@localhost ~]# git https://gitee.com/BRWYZ/kvm-install.git

#>>> 执行安装KVM脚本
[root@localhost ~]# bash kvm-install/intsall_kvm.sh 

#>>> 查看内核是否升级
[root@localhost ~]# uname -r
4.19.12-1.el7.elrepo.x86_64

#>>> 查看内核模块是否加载
[root@localhost ~]# lsmod | grep kvm
kvm_intel             208896  0 
kvm                   626688  1 kvm_intel
irqbypass              16384  1 kvm

#>>> 查看工具是否正常安装
[root@localhost ~]# virsh  list
 Id    名称                         状态
----------------------------------------------------
```

**工具说明：**

**bridge-utils**：

- 用于管理网络桥接的工具包。网络桥接允许多个网络接口共享一个网络，从而能够像一个单一的网络一样工作。通常用于虚拟机和主机之间的网络通信。

**qemu-kvm**：

- 通用的开源虚拟化软件，可以模拟多种硬件平台，qemu-kvm结合了QEMU的用户空间工具和KVM的内核模块，使得虚拟机的性能更接近于原生硬件。

**libvirt**：

- 开源的API、守护进程和管理工具，用于管理平台虚拟化。它支持多种虚拟化技术，包括KVM、QEMU、Xen、VMware等。libvirt提供了一致的管理接口和命令行工具（如 `virsh`），可以用于启动、停止和管理虚拟机。

**virt-install**：

- 命令行工具，用于从命令行创建新的虚拟机。它简化了虚拟机的安装过程，并支持多种安装方式，包括从ISO文件、网络安装等。典型的用法是指定虚拟机的名称、内存大小、磁盘大小和安装源等参数。

**libguestfs** 和 **libguestfs-tools**：

- libguestfs是一个库和工具集，用于访问和修改虚拟机磁盘映像。它可以在虚拟机不运行时检查、修改和修复虚拟机磁盘。
- libguestfs-tools包含一系列命令行工具，如 `guestfish`、`virt-resize`、`virt-copy-in` 等，用于执行具体的操作，比如拷贝文件、调整磁盘大小等。

**目录介绍**

- `/kvm/{vdisks,isos,modify}`
  - vdisks：用于存放虚拟机虚拟磁盘；
  - isos：用于存放安装虚拟机操作系统镜像；
  - modify：修改虚拟机配置中间目录。

------

&emsp;&emsp;KVM中支持三种网络模式, 分别为`NAT模式`、`bridge桥接模式`、`host-only仅主机模式`; NAT模式通常用来在个人虚拟化桌面中应用广泛, 桥接模式在服务器端虚拟化使用广泛, host-only一般在超级大的服务厂商内部使用; 本课程中的重点内容是桥接网络;

**桥接网络（Bridged Networking）**：

- 在桥接模式下，虚拟机直接连接到物理网络，就像它们是物理网络中的独立设备一样。
- 每个虚拟机获得一个独立的IP地址，可以直接与物理网络中的其他设备通信。
- 配置桥接网络时，通常需要在主机上创建一个网络桥接（如 `br0`），并将物理网络接口（如 `eth33`）添加到桥接中。
- 常用于需要虚拟机与外部网络通信的场景，如服务器托管和开发测试环境。

![image-20240702221450333](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407022214388.png)

**NAT网络（Network Address Translation）**：

- 在NAT模式下，虚拟机通过主机的网络地址与外部网络通信。虚拟机使用私有IP地址，而主机负责将虚拟机的流量转发到外部网络。
- 虚拟机可以访问外部网络，但外部网络无法直接访问虚拟机。

![image-20240702220933808](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407022209870.png)

**仅主机模式（Host-only Networking）**：

- 在内部网络模式下，虚拟机只能与同一主机上的其他虚拟机通信，无法直接访问外部网络。

- 这种模式创建了一个完全隔离的网络环境，非常适合测试和开发环境。

  

&emsp;&emsp;在创建虚拟机之前需要先将网络配置搞定, 由于公司/IDC机房中网络大多采用 `255.255.0.0` 为掩码的网段划分, 所以为了保证不同宿主机中的虚拟机能够正常通信, 此处采用 **桥接** 模式网络;

```bash
$ vim /etc/sysconfig/network-scripts/ifcfg-br0
DEVICE="br0"
NAME="br0"
BOOTPROTO="static"
ONBOOT="yes"
TYPE="Bridge"
IPADDR="192.168.174.9"
GATEWAY="192.168.174.2"
NETMASK="255.255.0.0"
DNS1="8.8.8.8"
DEFROUTE="yes"

$ vim /etc/sysconfig/network-scripts/ifcfg-ens33
DEVICE="ens33"
NAME="ens33"
BOOTPROTO="none"
NM_CONTROLLED="no"
ONBOOT="yes"
TYPE="Ethernet"
BRIDGE="br0"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="yes"

$ systemctl restart network
```

&emsp;&emsp;基础环境部署完成后, 就是创建模版虚拟机的阶段了, 新创建的KVM宿主机中是没有虚拟机的, 要想创建虚拟机需要使用 `virt-install` 工具进行虚拟机的创建; 模版虚拟机创建分为四步: **创建磁盘 -> 安装操作系统 -> 初始化模版机 -> 配置相关agent**

**磁盘格式：**

​	**raw格式**：

​		性能最好，因为不需要额外的处理或转换。占用空间较大，因为虚拟磁盘的文件大小与分配的磁盘空间大小相同。不支持高级功能，如快照或压缩。

​	**qcow2格式**：

​		QEMU的原生磁盘格式，支持许多高级功能，如快照、压缩、加密和动态分配等。支持快照功能，可以方便地保存和恢复虚拟机的状态；支持压缩，可以节省磁盘空间。支持动态分配（稀疏文件），初始文件大小较小，随着数据的写入逐渐增大。相对于raw格式，性能稍有下降，因为需要额外的处理和管理。

```bash
#>>> 创建一个一个40G的raw格式的磁盘在/kvm/template.raw
[root@localhost ~]# qemu-img create -f raw /kvm/vdisks/template.raw 10G  
#>>> 上传iso镜像   
[root@localhost ~]# ls /kvm/isos/CentOS-7-x86_64-Minimal-2009.iso 
/kvm/isos/CentOS-7-x86_64-Minimal-2009.iso

[root@localhost ~]# virt-install --name=template \
 --vcpus=2 --memory=2048 --disk=/kvm/vdisks/template.raw \
 --cdrom=/kvm/isos/CentOS-7-x86_64-Minimal-2009.iso --os-variant=rhel7 \
 --noautoconsole --autostart \
 --network bridge=br0 \
 --graphics vnc,listen=0.0.0.0,port=5900
```

> 创建参数详解：
>
> ​	`qemu-img create -f raw /kvm/vdisks/template.raw 10G`
>
> - qemu-img create  -f  创建虚拟磁盘命令
> - raw  磁盘格式
> - /kvm/vdisks/template.raw  虚拟磁盘存放处及虚拟磁盘名称
> - 10G    分配磁盘大小
>
> `virt-install --name=template \
>  --vcpus=2 --memory=2048 --disk=/kvm/vdisks/template.raw \
>  --cdrom=/kvm/isos/CentOS-7-x86_64-Minimal-2009.iso --os-variant=rhel7 \
>  --noautoconsole --autostart \
>  --network bridge=br0 \
>  --graphics vnc,listen=0.0.0.0,port=5900`
>
> - virt-install  创建VMM命令
> - --name=  虚拟机名称
> - --vcpus=  cpu核数
> - --memory=  内存大小，默认为KB
> - --disk=  指定磁盘创建
> - --cdrom= 指定操作系统镜像
> - --os-variant=  操作系统发行版本
> - --noautoconsole   创建虚拟机后不自动连接到控制台。
> - --autostart  设置虚拟机在主机启动时自动启动。
> - --network  为虚拟机配置网络，使用网桥`br0`。
> - --graphics vnc,listen=0.0.0.0,port=5900  置虚拟机的图形界面使用VNC（Virtual Network Computing），监听地址为 `0.0.0.0`（即所有IP地址），端口号为 `5900`。

&emsp;&emsp;模版主机创建成功后, 需要在机器中内置好,创建虚拟机时固定IP地址, 就需要良好的设计逻辑, 从镜像、脚本、系统三层配合实现; 

```bash
#>>> 虚拟机硬件资源存放处
[root@localhost ~]# ll /etc/libvirt/qemu
总用量 8
drwxr-xr-x  2 root root   26 7月   2 23:31 autostart
drwx------. 3 root root   42 7月   2 20:46 networks
-rw-------  1 root root 4228 7月   2 23:31 template.xml

#>>> 虚拟机虚拟磁盘存放处
[root@localhost ~]# ll /kvm/vdisks/
总用量 1347776
-rw-r--r-- 1 qemu qemu 10737418240 7月   2 23:39 template.raw
```

​		**手动克隆虚拟机：**由于虚拟机的虚拟文件携带该VMM的唯一标识码，不能通过copy的方式克隆，但可以使用`qemu-img`基于某个虚拟磁盘文件去生成新的虚拟磁盘文件

```bash
[root@localhost ~]# qemu-img  create -f qcow2 -b /kvm/vdisks/template.raw /kvm/vdisks/test.qcow2
Formatting '/kvm/vdisks/test.qcow2', fmt=qcow2 size=10737418240 backing_file='/kvm/vdisks/template.raw' encryption=off cluster_size=65536 lazy_refcounts=off 

#>>> 将新生成的虚拟文件中间磁盘挂载到中间目录
[root@localhost ~]# guestmount -a /kvm/vdisks/test.qcow2 -m /dev/centos/root /kvm/modify/
	# 参数解释：
		guestmount -a   挂载命令
		/kvm/vdisks/test.qcow2  原虚拟磁盘文件
		-m /dev/centos/root  指定挂载分区，一般都是root分区
		/kvm/modify/   指定挂载目录

#>>> 查看挂载分区实际情况
[root@localhost ~]# cd /kvm/modify/
[root@localhost modify]# ll
总用量 12
lrwxrwxrwx.  1 root root    7 7月   2 23:35 bin -> usr/bin
drwxr-xr-x.  2 root root    6 7月   2 23:34 boot
drwxr-xr-x.  2 root root    6 7月   2 23:34 dev
drwxr-xr-x. 74 root root 8192 7月   2 23:48 etc
drwxr-xr-x.  2 root root    6 4月  11 2018 home
lrwxrwxrwx.  1 root root    7 7月   2 23:35 lib -> usr/lib
lrwxrwxrwx.  1 root root    9 7月   2 23:35 lib64 -> usr/lib64
drwxr-xr-x.  2 root root    6 4月  11 2018 media
drwxr-xr-x.  2 root root    6 4月  11 2018 mnt
drwxr-xr-x.  2 root root    6 4月  11 2018 opt
drwxr-xr-x.  2 root root    6 7月   2 23:34 proc
dr-xr-x---.  2 root root  135 7月   2 23:49 root
drwxr-xr-x.  2 root root    6 7月   2 23:34 run
lrwxrwxrwx.  1 root root    8 7月   2 23:35 sbin -> usr/sbin
drwxr-xr-x.  2 root root    6 4月  11 2018 srv
drwxr-xr-x.  2 root root    6 7月   2 23:34 sys
drwxrwxrwt.  8 root root  211 7月   2 23:49 tmp
drwxr-xr-x. 13 root root  155 7月   2 23:35 usr
drwxr-xr-x. 19 root root  267 7月   2 23:48 var

#>>> 使用当前路径修改虚拟机IP
[root@localhost modify]# vim ./etc/sysconfig/network-scripts/ifcfg-eth0 
```

<img src="https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407030001910.png" alt="image-20240703000134857" style="zoom:50%;" />

```bash
#>>> 卸载中间目录
[root@localhost ~]# guestunmount /kvm/modify/
[root@localhost ~]# ll /kvm/modify/
总用量 0

#>>> 拷贝配置文件，作为新的VMM文件
[root@localhost ~]# cp /etc/libvirt/qemu/{template.xml,test.xml}
[root@localhost ~]# vim /etc/libvirt/qemu/test.xml 
```

![image-20240703000949730](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407030009807.png)

![image-20240703001246123](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407030012173.png)

![image-20240703001439192](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407030014223.png)

![image-20240703001838216](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407030018266.png)

```bash
#>>> 定义新的虚拟机，重新加载宿主机中所有的虚拟机
[root@localhost ~]# virsh define /etc/libvirt/qemu/test.xml 
定义域 test（从 /etc/libvirt/qemu/test.xml）

#>>> 查看宿主机所有的VMM
[root@localhost ~]# virsh list --all
 Id    名称                         状态
----------------------------------------------------
 2     template                       running
 -     test                           关闭

#>>> 启动VMM
[root@localhost ~]# virsh start test
域 test 已开始
```

------

​		**基于脚本自动克隆VMM**

```bash
#>>> 将创建脚本拷贝至指定目录
[root@localhost ~]# cp kvm-install/createvm.sh  /usr/local/bin/createvm

#>>> 添加执行权限
[root@localhost ~]# chmod a+x /usr/local/bin/createvm 

#>>> 测试执行
[root@localhost ~]# createvm  --help
用法: createvm [hapn]... 
 Create a virtual machine and create a fixed 
 IP address, vnc port, virtual machine name 
 	 -h, --help 获取帮助信息
 	 -a, --address 设置IP地址
 	 -p, --port 设置vnc端口
 	 -n, --name 设置虚拟机名称
 退出状态：
 	 0 正常
 	 1404 一般问题 (例如：没有对应的选项)
 	 x403 严重问题 (例如：设置参数不正确)
```

**脚本模板修改的位置图下：除此还有硬件配置文件和网卡配置文件里的UUID和MAC地址ID.**

```bash
[root@localhost ~]# vim /etc/libvirt/qemu/template.xml 
```

![image-20240703003649040](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407030036081.png)

![image-20240703003831363](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407030038402.png)

![image-20240703004117299](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407030041343.png)

![image-20240703004311024](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407030043067.png)

```bash
[root@localhost ~]# createvm  -a 192.168.174.13 -p 5902 -n test02
Formatting '/kvm/vdisks/test02.qcow2', fmt=qcow2 size=10737418240 backing_file='/kvm/vdisks/template.raw' encryption=off cluster_size=65536 lazy_refcounts=off 
定义域 test02（从 /etc/libvirt/qemu/test02.xml）

域 test02 已开始

[root@localhost ~]# virsh list
 Id    名称                         状态
----------------------------------------------------
 2     template                       running
 4     test                           running
 6     test02                         running
```



#### 1.3.3 KVM虚机生命周期管理

&emsp;&emsp;对于虚拟化层之上的虚拟机是具备生命周期的进程, 对于进程而言就会具备运行、重启、停止等基本生命周期, 由于其进程的特殊性还会有cpu、内存、磁盘的添加和移除以及快照、克隆等操作; 以上所提到的所有操作都需要使用libvirtd包中所包含的 `virsh` 指令;

```bash
#> 查看指令
$ virsh list --all

#> 启动指令
$ virsh start VM_NAME

#> 关机指令
$ virsh shutdown VM_NAME

#> 断电指令
$ virsh destroy VM_NAME

#> 开机指令
$ virsh start VM_NAME

#> 重启指令
$ virsh reboot VM_NAME

#> 连接指令
$ virsh console VM_NAME
Connected to domain VMACHINE_NAME
Escape character is ^]

CentOS Linux 7 (Core)
Kernel 3.10.0-514.el7.x86_64 on an x86_64

localhost login: root
Password: 
Last login: Wed Jun 14 17:06:02 on ttyS0
[root@VMACHINE_NAME ~]# 

##> [ERROR CHECK]如果console连接卡住不动, 则通过vnc进入到对应的虚拟机中在以下文件中添加内容
$ cat /etc/securetty | tail -n 1
ttyS0

$ [root@ungeolinux ~]# cat /etc/grub2.cfg | grep ttyS0
linux16 /vmlinuz-3.10.0-1160.el7.x86_64 root=/dev/mapper/centos-root ro rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet console=ttyS0
```



