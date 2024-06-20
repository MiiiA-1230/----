# Kubernetes入门概念

[TOC]

## 1、简介ceph分布式存储

简介：[Kubernetes](https://kubernetes.p2hp.com/docs/concepts/overview/) 也称为 K8s，是用于自动部署、扩缩和管理容器化应用程序的开源系统。它将组成应用程序的容器组合成逻辑单元，以便于管理和服务发现。Kubernetes 源自[Google 15 年生产环境的运维经验](http://queue.acm.org/detail.cfm?id=2898444)，同时凝聚了社区的最佳创意和实践。

`容器是打包和运行应用程序的好方式`。在生产环境中， 你需要管理运行着应用程序的容器，并确保服务不会下线。 例如，如果一个容器发生故障，则你需要启动另一个容器。 如果此行为交由给系统处理，是不是会更容易一些？

这就是 Kubernetes 要来做的事情！ Kubernetes 为你提供了一个可弹性运行分布式系统的框架。 Kubernetes 会满足你的扩展要求、故障转移你的应用、提供部署模式等。

Kubernetes 为你提供：

- **服务发现和负载均衡**

  Kubernetes 可以使用 DNS 名称或自己的 IP 地址来曝露容器。 如果进入容器的流量很大， Kubernetes 可以负载均衡并分配网络流量，从而使部署稳定。

- **存储编排**

  Kubernetes 允许你自动挂载你选择的存储系统，例如本地存储、公共云提供商等。

- **自动部署和回滚**

  你可以使用 Kubernetes 描述已部署容器的所需状态， 它可以以受控的速率将实际状态更改为期望状态。 例如，你可以自动化 Kubernetes 来为你的部署创建新容器， 删除现有容器并将它们的所有资源用于新容器。

- **自动完成装箱计算**

  你为 Kubernetes 提供许多节点组成的集群，在这个集群上运行容器化的任务。 你告诉 Kubernetes 每个容器需要多少 CPU 和内存 (RAM)。 Kubernetes 可以将这些容器按实际情况调度到你的节点上，以最佳方式利用你的资源。

- **自我修复**

  Kubernetes 将重新启动失败的容器、替换容器、杀死不响应用户定义的运行状况检查的容器， 并且在准备好服务之前不将其通告给客户端。

- **密钥与配置管理**

  Kubernetes 允许你存储和管理敏感信息，例如密码、OAuth 令牌和 ssh 密钥。 你可以在不重建容器镜像的情况下部署和更新密钥和应用程序配置，也无需在堆栈配置中暴露密钥

  **Kubernetes 不是什么**

- 不限制支持的应用程序类型。 Kubernetes 旨在支持极其多种多样的工作负载，包括无状态、有状态和数据处理工作负载。 如果应用程序可以在容器中运行，那么它应该可以在 Kubernetes 上很好地运行。

- 不部署源代码，也不构建你的应用程序。 持续集成（CI）、交付和部署（CI/CD）工作流取决于组织的文化和偏好以及技术要求。

- 不提供应用程序级别的服务作为内置服务，例如中间件（例如消息中间件）、 数据处理框架（例如 Spark）、数据库（例如 MySQL）、缓存、集群存储系统 （例如 Ceph）。这样的组件可以在 Kubernetes 上运行，并且/或者可以由运行在 Kubernetes 上的应用程序通过可移植机制 （例如[开放服务代理](https://openservicebrokerapi.org/)）来访问。

- 不是日志记录、监视或警报的解决方案。 它集成了一些功能作为概念证明，并提供了收集和导出指标的机制。
- 不提供也不要求配置用的语言、系统，它提供了声明性 API， 该声明性 API 可以由任意形式的声明性规范所构成。
- 不提供也不采用任何全面的机器配置、维护、管理或自我修复系统。
- 此外，Kubernetes 不仅仅是一个编排系统，实际上它消除了编排的需要。 编排的技术定义是执行已定义的工作流程：首先执行 A，然后执行 B，再执行 C。 而 Kubernetes 包含了一组独立可组合的控制过程，可以连续地将当前状态驱动到所提供的预期状态。 你不需要在乎如何从 A 移动到 C，也不需要集中控制，这使得系统更易于使用 且功能更强大、系统更健壮，更为弹性和可扩展。



架构图：

![image-20240614190826602](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406141908659.png)



## 2、组件介绍

当你部署完 Kubernetes，便拥有了一个完整的集群。一组工作机器，称为 [节点](https://kubernetes.p2hp.com/docs/concepts/architecture/nodes/)， 会运行容器化应用程序。每个集群至少有一个工作节点。工作节点会托管 [Pod](https://kubernetes.p2hp.com/docs/concepts/workloads/pods.html) ，而 Pod 就是作为应用负载的组件。 [控制平面](https://kubernetes.p2hp.com/docs/reference/glossary/index.html%3Fall=true.html#term-control-plane)管理集群中的工作节点和 Pod。 在生产环境中，控制平面通常跨多台计算机运行， 一个集群通常运行多个节点，提供容错性和高可用性。

### 2.1 kube-apiserver[ ](https://kubernetes.p2hp.com/docs/concepts/overview/components/#kube-apiserver)

​	Kubernetes API Server的核心功能是提供Kubernetes各类资源对象的增、删、改、查及Watch等HTTP REST接口，成为集群内各个功能模块之间数据交互和通信的中心枢纽，是整个系统的数据总线和数据中心。除此之外，它还是集群管理的 API入口，是资源配额控制的入口，提供了完备的集群安全机制。	

​	API 服务器是 Kubernetes [控制平面](https://kubernetes.p2hp.com/docs/reference/glossary/index.html%3Fall=true.html#term-control-plane)的组件， 该组件负责公开了 Kubernetes API，负责处理接受请求的工作。 API 服务器是 Kubernetes 控制平面的前端。

Kubernetes API 服务器的主要实现是 [kube-apiserver](https://kubernetes.p2hp.com/docs/reference/command-line-tools-reference/kube-apiserver.html)。 `kube-apiserver` 设计上考虑了水平扩缩，也就是说，它可通过部署多个实例来进行扩缩。 你可以运行 `kube-apiserver` 的多个实例，并在这些实例之间平衡流量。



### 2.2 etcd[ ](https://kubernetes.p2hp.com/docs/concepts/overview/components/#etcd)

​	一致且高度可用的键值存储，用作 Kubernetes 的所有集群数据的后台数据库。如果你的 Kubernetes 集群使用 etcd 作为其后台数据库， 请确保你针对这些数据有一份 [备份](https://kubernetes.p2hp.com/docs/tasks/administer-cluster/configure-upgrade-etcd/index.html#backing-up-an-etcd-cluster)计划。



### 2.3 kube-scheduler[ ](https://kubernetes.p2hp.com/docs/concepts/overview/components/#kube-scheduler)

​	`kube-scheduler` 是[控制平面](https://kubernetes.p2hp.com/docs/reference/glossary/index.html%3Fall=true.html#term-control-plane)的组件， 负责监视新创建的、未指定运行[节点（node）](https://kubernetes.p2hp.com/docs/concepts/architecture/nodes/)的 [Pods](https://kubernetes.p2hp.com/docs/concepts/workloads/pods.html)， 并选择节点来让 Pod 在上面运行。

​	Kubernetes要努力满足各种类型应用的不同需求并且努力“让大家和平共 处”。Kubernetes集群里的Pod有无状态服务类、有状态集群类及批处理类三大类，不同类型的Pod对资源占用的需求不同，对节点故障引发的中断/恢复及节点迁移方面的容忍度都不同，如果再考虑到业务方面不同服务的Pod的优先级不同带来的额外约束和限制，以及从租户（用户）的角度希望占据更多的资源增加稳定性和集群拥有者希望调度更多的Pod提升资源使用率两者之间的矛盾

调度决策考虑的因素包括单个 Pod 及 Pods 集合的资源需求、软硬件及策略约束、 亲和性及反亲和性规范、数据位置、工作负载间的干扰及最后时限。



### 2.4 kube-controller-manager[ ](https://kubernetes.p2hp.com/docs/concepts/overview/components/#kube-controller-manager)

​	**在Kubernetes集群中，每个Controller都是这样的一 个“操作系统”，它们通过API Server提供的（List-Watch）接口实时监控集群中特定资源的状态变化，当发生各种故障导致某资源对象的状态变化时，Controller会尝试将其状态调整为期望的状态。比如当某个Node 意外宕机时，Node Controller会及时发现此故障并执行自动化修复流程，确保集群始终处于预期的工作状态下。Controller Manager是 Kubernetes中各种操作系统的管理者，是集群内部的管理控制中心，也 是Kubernetes自动化功能的核心。**

​	Controller Manager内部包含Replication Controller、 Node Controller、ResourceQuota Controller、Namespace Controller、 ServiceAccount Controller、Token Controller、Service Controller、 Endpoint Controller、Deployment Controller、Router Controller、Volume Controller等各种资源对象的控制器，每种Controller都负责一种特定资源的控制流程，而Controller Manager正是这些Controller的核心管理者。

​	[kube-controller-manager](https://kubernetes.p2hp.com/docs/reference/command-line-tools-reference/kube-controller-manager/) 是[控制平面](https://kubernetes.p2hp.com/docs/reference/glossary/index.html%3Fall=true.html#term-control-plane)的组件， 负责运行[控制器](https://kubernetes.p2hp.com/docs/concepts/architecture/controller.html)进程。

从逻辑上讲， 每个[控制器](https://kubernetes.p2hp.com/docs/concepts/architecture/controller.html)都是一个单独的进程， 但是为了降低复杂性，它们都被编译到同一个可执行文件，并在同一个进程中运行。

这些控制器包括：

- 节点控制器（Node Controller）：负责在节点出现故障时进行通知和响应
- 任务控制器（Job Controller）：监测代表一次性任务的 Job 对象，然后创建 Pods 来运行这些任务直至完成
- 端点分片控制器（EndpointSlice controller）：填充端点分片（EndpointSlice）对象（以提供 Service 和 Pod 之间的链接）。
- 服务账号控制器（ServiceAccount controller）：为新的命名空间创建默认的服务账号（ServiceAccount）。

### 2.5 kubelet

​	`kubelet` 会在集群中每个[节点（node）](https://kubernetes.p2hp.com/docs/concepts/architecture/nodes/)上运行。 它保证[容器（containers）](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/#why-containers)都运行在 [Pod](https://kubernetes.p2hp.com/docs/concepts/workloads/pods.html) 中。kubelet 接收一组通过各类机制提供给它的 PodSpecs， 确保这些 PodSpecs 中描述的容器处于运行状态且健康。 kubelet 不会管理不是由 Kubernetes 创建的容器。

​	在Kubernetes集群中，在每个Node上都会启动一个 kubelet服务进程。该进程用于处理Master下发到本节点的任务，管理 Pod及Pod中的容器。每个kubelet进程都会在API Server上注册节点自身的信息，定期向Master汇报节点资源的使用情况，并通过cAdvisor监控容器和节点资源。

### 2.6 kube-proxy[ ](https://kubernetes.p2hp.com/docs/concepts/overview/components/#kube-proxy)

​	[kube-proxy](https://kubernetes.p2hp.com/docs/reference/command-line-tools-reference/kube-proxy/) 是集群中每个[节点（node）](https://kubernetes.p2hp.com/docs/concepts/architecture/nodes/)所上运行的网络代理， 实现 Kubernetes [服务（Service）](https://kubernetes.p2hp.com/docs/concepts/services-networking/service/) 概念的一部分。kube-proxy 维护节点上的一些网络规则， 这些网络规则会允许从集群内部或外部的网络会话与 Pod 进行网络通信。

​	在集群中的Service和Pod大量增加以后，每个Node节点上 iptables中的规则会急速膨胀，导致网络性能显著下降，在某些极端情况 下甚至会出现规则丢失的情况，并且这种故障难以重现与排查。于是 Kubernetes从1.8版本开始引入第三代的IPVS（IP Virtual Server）模式.

**iptables与IPVS虽然都是基于Netfilter实现的**，但因为定位不同，二者有着本质的差别：			iptables是为防火墙设计的；IPVS专门用于高性能负载均衡，并使用更高效的数据结构（哈希表），允许几乎无限的规模扩张，因此被kube-proxy采纳为第三代模式。 

与iptables相比，IPVS拥有以下明显优势： 

- **为大型集群提供了更好的可扩展性和性能；**
- **支持比iptables更复杂的复制均衡算法（最小负载、最少连接、 加权等）；** 
- **支持服务器健康检查和连接重试等功能；** 
- **可以动态修改ipset的集合，即使iptables的规则正在使用这个集合。** 
- 由于IPVS无法提供包过滤、airpin-masquerade tricks（地址伪装）、 SNAT等功能，因此在某些场景（如NodePort的实现）下还要与iptables搭配使用。
- 在IPVS模式下，kube-proxy又做了重要的升级，即使用 iptables的扩展ipset，而不是直接调用iptables来生成规则链。 iptables规则链是一个线性数据结构，ipset则引入了带索引的数据结构，因此当规则很多时，也可以高效地查找和匹配。我们可以将ipset简单理解为一个IP（段）的集合，这个集合的内容可以是IP地址、IP网 段、端口等，iptables可以直接添加规则对这个“可变的集合”进行操作， 这样做的好处在于大大减少了iptables规则的数量，从而减少了性能损耗。假设要禁止上万个IP访问我们的服务器，则用iptables的话，就需要 一条一条地添加规则，会在iptables中生成大量的规则；但是用ipset的 话，只需将相关的IP地址（网段）加入ipset集合中即可，这样只需设置少量的iptables规则即可实现目标。 



### 2.7 容器运行时（Container Runtime）[ ](https://kubernetes.p2hp.com/docs/concepts/overview/components/#container-runtime)

​	容器运行环境是负责运行容器的软件。Kubernetes 支持许多容器运行环境，例如 [containerd](https://containerd.io/docs/)、 [CRI-O](https://cri-o.io/#what-is-cri-o) 以及 [Kubernetes CRI (容器运行环境接口)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md) 的其他任何实现。

### 2.8 DNS[ ](https://kubernetes.p2hp.com/docs/concepts/overview/components/#dns)

​	集群 DNS 是一个 DNS 服务器，和环境中的其他 DNS 服务器一起工作，它为 Kubernetes 服务提供 DNS 记录。Kubernetes 启动的容器自动将此 DNS 服务器包含在其 DNS 搜索列表中。[CoreDNS](https://coredns.io/) 是一种灵活的，可扩展的 DNS 服务器，可以 [安装](https://github.com/coredns/deployment/tree/master/kubernetes)为集群内的 Pod 提供 DNS 服务。用于Kubernetes集群内部Service的解析，可以让Pod把Service名称解析成Service的IP，然后通过service的IP地址进行连接到对应的应用上。

### 2.9 网络插件Calico

​	Calico是一个基于BGP的纯三层的网络方案，与OpenStack、 Kubernetes、AWS、GCE等云平台都能够良好地集成。Calico在每个计 算节点都利用Linux Kernel实现了一个高效的vRouter来负责数据转发。 每个vRouter都通过BGP1协议把在本节点上运行的容器的路由信息向整个Calico网络广播，并自动设置到达其他节点的路由转发规则。Calico 保证所有容器之间的数据流量都是通过IP路由的方式完成互联互通的。 Calico节点组网时可以直接利用数据中心的网络结构（L2或者L3），不需要额外的NAT、隧道或者Overlay Network，没有额外的封包解包，能 够节约CPU运算，提高网络效率

**calico组件：**

- Felix：Calico Agent，运行在每个Node上，负责为容器设置网络资源（IP地址、路由规则、iptables规则等），保证跨主机容器网络互通。 
- etcd：Calico使用的后端存储。 
- BGP Client：负责把Felix在各Node上设置的路由信息通过BGP广播到Calico网络。 
-  Route Reflector：通过一个或者多个BGP Route Reflector完成大规模集群的分级路由分发。 
- CalicoCtl：Calico命令行管理工具。

​	

​	

