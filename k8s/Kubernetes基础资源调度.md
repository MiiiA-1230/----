# Kubernetes基础资源

[TOC]

## 前言

### 1、无状态应用

**无状态服务**是指每个请求都是独立的，服务不保留客户端的状态信息。

1. **无状态**：服务不存储任何客户端的状态信息，每次请求都是独立的。
2. **易于扩展**：由于每个实例都不需要单独挂载存储，横向扩展（增加实例）非常简单。
3. **高可用性**：任何一个实例的故障不会影响整体服务的运行，只需通过负载均衡将请求分配到健康的实例上即可。
4. **简单管理**：因为无状态服务不依赖于特定的实例，可以随时销毁和重新创建实例，维护和部署相对简单。

无状态服务的典型示例包括Web服务器nginx、Tomcat、PHP等。

### 2、有状态服务

**有状态服务**是指那些需要保留客户端状态信息，并且不同请求之间可能需要共享数据或状态的服务。

1. **实例唯一性**：每个实例都有一个唯一的身份，通常需要与特定的存储卷关联。
2. **难于扩展**：因为状态信息需要在实例之间共享或同步，扩展和缩减实例变得更加复杂。
3. **高可用性挑战**：实例故障可能会导致数据不一致或服务中断。

有状态服务通常使用StatefulSet资源来管理。这种资源会确保每个Pod实例都有一个固定的身份和持久存储卷，即使Pod被重新调度或重新创建，也会保留其状态。典型的有状态服务示例包括数据库（如MySQL、PostgreSQL）、分布式缓存（如Redis、Memcached）等。



### 3、ReplicaSet（简称RS）

​	一种用来确保指定数量的Pod副本始终在集群中运行的资源对象。ReplicaSet是Kubernetes中一个较低层次的控制器，是Deployment等更高层次控制器的基础。ReplicaSet的概念和作用ReplicaSet的主要作用是确保在任何时间点，始终有指定数量的Pod副本在集群中运行。ReplicaSet会自动创建和删除Pod，以达到和维持所需的Pod副本数量。

##     1、Deployment：无状态资源管理

### 1.1 简介

​	`Deployment `是一种用于管理和部署容器化应用程序的资源。它提供了一种声明式的方法来管理应用程序的副本，支持`滚动更新`、`回滚`、`扩展`和`自动修复`等功能。一般用于部署公司的无状态服务，这个也是最常用的控制器，因为企业内部现在都是以微服务为主，而微服务实现无状态化也是最佳实践，可以利用`Deployment`的高级功能做到无缝迁移、自动扩容缩容、自动灾难恢复、一键回滚等功能。通过定义` Deployment`，你可以描述应用的期望状态，Kubernetes会负责将实际状态调整为期望状态。

### 1.2 Deployment部署方式:

​      首先在控制端定义了一个3副本的kind为Dployment的yaml文件时，在执行kubectl create 是会提交给apiserver；然后持久化实例，在yaml中指定的命名空间下，可以看到有个nginx，然后会先创建一个名为nginx-xxx（随机字符串）的RS复制集，然后由复制集会选出最优节点部署Pod（有可能一台节点，有可能多台节点；没有固定的节点）

1. kubectl向apiserver发送创建请求；

2. apiserver将 Deployment 持久化到etcd，etcd与apiserver进行一次http通信。

3. controller manager通过watch api监听 apiserver ，deployment controller看到了一个新创建的deplayment对象更后，将其从队列中拉出，根据deployment的描述创建一个ReplicaSet并将 ReplicaSet 对象返回apiserver并持久化回etcd。
   以此类推，当replicaset控制器看到新创建的replicaset对象，将其从队列中拉出，根据描述创建pod对象。

4. 接着scheduler调度器看到未调度的pod对象，根据调度规则选择一个可调度的节点，加载到pod描述中nodeName字段，并将pod对象返回apiserver并写入etcd。kubelet在看到有pod对象中nodeName字段属于本节点，将其从队列中拉出，通过容器运行时创建pod中描述的容器

**创建一个Deployment的yaml文件**

```bash
[root@k8s-master01 ~]# kubectl create deploy nginx --image=registry.cn-zhangjiakou.aliyuncs.com/taosweet/nginx:1.24.0-alpine --replicas=3 -oyaml --dry-run=client > nginx-deploy.yaml
```

> 参数解释：
> 					`--replicas=3`   Pod的副本数为3
> 					`--dry-run=client`    输出给apiserver但不创建Pod,

```bash
[root@k8s-master01 ~]# vim nginx-deploy.yaml
---
apiVersion: apps/v1   # 资源的 API 版本
kind: Deployment      # 资源的类型
metadata:    # 资源的元数据信息
  name: nginx-deployment  # 资源的名称
  labels:    # 资源的标签，为这个Deployment资源打上标签，以便将来选择和过滤。
    app: nginx
spec:  # 定义Deployment的详细规格
  replicas: 3  # 定义Pod的副本
  selector:   # 标签选择器，
    matchLabels:  # 只有带有指定标签的Pod才会被这个Deployment管理。
      app: nginx  # 指定需要管理的Pod的标签
  template:    # 定义 Pod 模板，用于创建 Pod。
    metadata:  # Pod元数据
      labels:  # 定义Pod的标签，以供deploy的selector选择，输入该deploy管理
        app: nginx  # Pod的标签
    spec:  # 定义Pod规格，包括容器配置。
      containers:  # 定义容器列表
      - name: nginx  # Pod名称
        image: registry.cn-zhangjiakou.aliyuncs.com/taosweet/nginx:1.24.0-alpine  # 容器镜像
        ports:  # 指定容器端口
        - containerPort: 80 # 容器的端口
------------------------------------------------------------------------
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: registry.cn-zhangjiakou.aliyuncs.com/taosweet/nginx:1.24.0-alpine
        ports:
        - containerPort: 80

#>>> 创建deploy资源
[root@k8s-master01 ~]# kubectl create -f nginx-deploy.yaml
deployment.apps/nginx created

#>>> 查看ReplicaSet资源
[root@k8s-master01 ~]# kubectl get rs
```

![image-20240619224610590](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406192246636.png)

> 提示：
> 		`NAME` ：RS名称
> 		`DESIRED`：期望的 Pod 副本数。
> 		`CURRENT`：当前实际的 Pod 副本数
> 		`READY`：就绪的 Pod 副本数。
> 		`AGE`：存在时长

```bash
#>>> 查看创建deploy的pod
[root@k8s-master01 ~]# kubectl  get po -owide
```

![image-20240619224858434](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406192248476.png)

> 提示：
> 		**可以看到pod的前缀和RS的前缀一致。**

```bash
#>>> 查看deploy资源
[root@k8s-master01 ~]# kubectl get deploy -owide
```

![image-20240619225325942](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406192253984.png)

> `NAME`： Deployment名称
> `READY`：Pod的状态，已经Ready的个数
> `UP-TO-DATE`：已经达到期望状态的被更新的副本数
> `AVAILABLE`：已经可以用的副本数
> `AGE`：显示应用程序运行的时间
> `CONTAINERS`：容器名称
> `IMAGES`：容器的镜像
> `SELECTOR`：管理的Pod的标签

**测试删除某个Pod**

![image-20240619231712261](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406192317291.png)

```bash
#>>> 删除某个Pod
[root@k8s-master01 ~]# kubectl delete po nginx-deployment-7bcc989f8f-97gsf  
pod "nginx-deployment-7bcc989f8f-97gsf" deleted

#>>> 再次查看Pod
[root@k8s-master01 ~]# kubectl get po 
```

![image-20240619231915103](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406192319139.png)

> 解释：
> 	**自愈机制。期望状态 vs 当前状态**：
>
> Deployment 会创建一个 ReplicaSet 来管理 Pod 副本。ReplicaSet 持续监控当前实际的 Pod 副本数量与期望数量之间的差异。
>
> **监控与调谐**：当一个 Pod 被删除时（无论是手动删除还是因故障被删除），当前实际的 Pod 数量会减少。ReplicaSet Controller 会立即检测到这种变化，因为它不断地与 API Server 进行通信。
> **创建新 Pod**：当检测到当前 Pod 数量少于期望数量时，ReplicaSet Controller 会自动创建新的 Pod 来补足。这些新的 Pod 会根据 Pod 模板（在 Deployment 的 `spec.template` 中定义）创建，确保新 Pod 的配置与原始 Pod 一致。



### 1.3 Deployment资源更新

#### 1.3.1 使用场景

​	程序升级，变革更新

#### 1.3.2 更新配置

```bash
#>>> 通过kubectl命令更新deploy的image
[root@k8s-master01 ~]# kubectl set image deploy nginx-deployment nginx=registry.cn-hangzhou.aliyuncs.com/hujiaming/nginx:1.24.0  --record

#>>> 查看更新过程
[root@k8s-master01 ~]# kubectl rollout status deploy nginx-deployment

#>>> 通过yaml文件更新Deploy,首先查看deploy的名称
[root@k8s-master01 ~]# kubectl get deploy nginx-deployment

#>>> 更新
[root@k8s-master01 ~]# kubectl edit deploy nginx
    找到images修改镜像Tag
    
#>>> 查看Deployment详细的更新过程
[root@k8s-master01 ~]# kubectl describe deploy nginx-deployment
```

![image-20240620125712778](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406201257824.png)

>解释：
>	**在describe中可以看出，第一次创建时，它创建了一个名为nginx-deployment-786479c6df的ReplicaSet，并直接将其扩展为3个副本。更新部署时，它创建了一个新的ReplicaSet，命名为nginx-deployment-c5dcb4544，并将其副本数扩展为1，然后将旧的ReplicaSet缩小为2，这样至少可以有2个Pod可用，最多创建了4个Pod。以此类推，使用相同的滚动更新策略向上和向下扩展新旧ReplicaSet，最终新的ReplicaSet可以拥有3个副本，并将旧的ReplicaSet缩小为0。**

#### 1.3.3 deployment更新过程

1. **创建新的 ReplicaSet**：当 Deployment 检测到 Pod 模板发生变化（例如镜像版本更新）时，会创建一个新的 ReplicaSet 来管理新的 Pod。
2. **生成新 ReplicaSet**：新 ReplicaSet 的名称类似 `nginx-deployment-xxxxxx`，包含新的镜像版本信息。
3. **逐步更新**：Deployment Controller 会逐步替换旧的 Pod，而不是一次性删除所有旧 Pod 并创建新 Pod。
4. **更新策略**：默认情况下，Kubernetes 使用滚动更新策略，即每次终止一部分旧的 Pod 并创建一部分新的 Pod。
5. **新旧 Pod 共存**：在更新过程中，新旧版本的 Pod 会共存一段时间，确保在任何时间点都有足够的 Pod 提供服务。
6. **健康检查**：新的 Pod 在替换旧的 Pod 之前，必须通过健康检查，确保其能够正常运行。
7. **终止旧 Pod**：当新的 Pod 通过健康检查后，Deployment Controller 会逐步终止旧的 Pod。
8. **创建新 Pod**：与此同时，创建新的 Pod，直到新的 ReplicaSet 中的 Pod 数量达到期望值。
9. **更新完成**：当所有新的 Pod 都被创建并通过健康检查，且旧的 Pod 被终止后，滚动更新过程完成。

### 1.4 Deployment回滚（生产环境用的较多）

​       **使用场景：当新发的版本出现BUG或者因为某些原因无法正常使用时，就需要进行版本回退。一般情况都是回滚到上一个版本（deploy回滚不推荐使用修改yaml的方式，有可能历代版本有其他修改的参数）**

```bash
#>>> 查看历史更新记录
[root@k8s-master01 ~]# kubectl rollout history deploy  nginx-deployment
```

![image-20240620130319537](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406201303615.png)

```bash
#>>> 查看更新历史的第2个版本的详细信息
[root@k8s-master01 ~]# kubectl rollout history deploy nginx-deployment --revision=2
  # --revision=2 查看第二个版本
```

![image-20240621002324924](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406210023978.png)

```bash
#>>> kubectl命令行回滚到上一个版本（生产使用最多，不推荐使用edit）
[root@k8s-master01 ~]# kubectl rollout undo deploy nginx-deployment
deployment.apps/nginx-deployment rolled back

#>>> kubectl 命令行回滚到第二个版本
[root@k8s-master01 ~]# kubectl rollout undo deploy nginx-deployment --to-revision=2
deployment.apps/nginx-deployment rolled back
```



### 1.5 Deployment扩容和缩容

​       **使用场景：当公司业务增大，Deploy的个数不足以支撑业务的正常运行，所以增加Pod的个数。或者公司推出某些活动需要临时增加Pod的个数。**

``` bash
#>>> kubectl命令行扩容
[root@k8s-master01 ~]# kubectl scale deploy nginx-deployment --replicas=5


#>>> 查看deploy的Pod
[root@k8s-master01 ~]# kubectl get po 


#>>> 修改yaml文件的方式进行扩缩容(不推荐，容易导致资源清单和启动的Pod配置不一致)
[root@k8s-master01 ~]# kubectl edit deploy nginx-deployment
  找到容器数量修改参数
```



## 2、StatefulSet 有状态资源调度

​	StatefulSet（有状态集，缩写sts)常用于部署有状态的且需要有序启动的应用程序。

### 2.1 前言

​	在IT世界里，有状态的应用被类比为宠物，无状态的应用则被类比为牛羊，每个宠物在主人那里都是“唯一的存在”，宠物生病了，我们是要花很多钱去治疗的，需要我们用心照料，而无差别的牛羊则没有这个待遇。

### 2.2 简介

- 每个节点都有`固定的身份ID`，通过这个ID，集群中的成员可以 `相互发现并通信`。
- `群的规模是比较固定的，集群规模不能随意变动。` 
- 集群中的`每个节点都是有状态的`，通常会`持久化数据到永久存储`中，每个节点在重启后都需要使用原有的持久化数据。
- 集群中成员节点的启动顺序（以及关闭顺序）通常也是确定的。
- 如果磁盘损坏，则集群里的某个节点无法正常运行，集群功能受损。

### 2.3 StatefulSet特性

- **需要稳定的独一无二的网络标识符**
  - StatefulSet里的每个Pod都有稳定、唯一的网络标识，可以用来发现集群内的其他成员。假设StatefulSet的名称为kafka，那么第1个Pod叫kafka-0，第2个叫kafka-1，以此类推。
- **需要有序的、优雅的部署和扩展**
  - StatefulSet控制的Pod副本的启停顺序是受控的，操作第n个Pod 时，前n-1个Pod已经是运行且准备好的状态。
  - 如 MySQL集群,要先启动主节点, 若从节点没有要求,则可一起启动,若从节点有启动顺序要求，可先启动第一个从节点，接着第二从节点；这个过程就是有顺序，平滑安全的启动。

- **需要持久化数据;**
  - 需要持久存储，新增或减少pod，存储不会随之发生变化。
- **需要有序的自动滚动更新**
  - MySQL在更新时，应该先更新从节点，全部的从节点都更新完了，最后在更新主节点，因为新版本一般可兼容老版本，但是一定要注意，若新版本不兼容老版本就很麻烦。

​	

### 2.4 Headless Service 无头服务

#### 2.4.1 前言

​	Headless Service的概念是这种服务没有入口访问地址（无ClusterIP 地址），kube-proxy不会为其创建负载转发规则，而服务名（DNS域名）的解析机制取决于该Headless Service是否设置了Label Selector。

#### 2.5.2 介绍

​	和Deployment类似，一个StatefulSet也同样管理着基于相同容器规范的Pod。不同的是，StatefulSet为每个Pod维护了一个`粘性标识`。而StatefulSet创建的Pod一般使用`Headless Service(无头服务）`进行Pod之前的通信，和普通的Service的区别在于Headless Service没有ClusterlP，它使用的是Endpoint进行互相通信，Headless一般的格式为:

**`statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local`**

- serviceName为Headless Service的名字，创建StatefulSet时必须指定Headless Service名称;
- 0..N-1为Pod所在的序号，从o开始到N-1;
- statefulSetName为StatefulSet的名字;
- namespace服务所在的命名空间;
- .cluster.local为Cluster Domain（集群域）。

### 2.5 StatefulSet的yaml创建

```yaml
$ vim nginx-sts.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"  # servive名称
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: registry.cn-zhangjiakou.aliyuncs.com/taosweet/nginx:1.24.0-alpine
        ports:
        - containerPort: 80
          name: web
```

​		

```bash
#>>> 创建sts
$ kubectl create -f nginx-sts.yaml
#>>> 查看pod
$ kubectl get po 
执行结果：
	NAME    READY   STATUS    RESTARTS   AGE
 web-0   1/1     Running   0          4m7s
 web-1   1/1     Running   0          4m6s
#>>> 查看sts
$ kubectl get sts
执行结果：
	NAME   READY   AGE
	web    2/2     5m12s
```



### 2.6 StatefulSet创建和删除

```bash
#>>> 查看多副本创建过程
$ kubectl get po -w

	NAME    READY   STATUS              RESTARTS   AGE
web-0   1/1     Running             0          11m
web-1   1/1     Running             0          11m
web-2   1/1     Running             0          103s
web-3   1/1     Running             0          101s
web-4   1/1     Running             0          100s
web-5   1/1     Running             0          1s
web-6   0/1     ContainerCreating   0          0s
web-6   0/1     ContainerCreating   0          1s
web-6   1/1     Running             0          2s
web-7   0/1     Pending             0          0s
web-7   0/1     Pending             0          0s
web-7   0/1     ContainerCreating   0          0s
web-7   0/1     ContainerCreating   0          1s
web-7   1/1     Running             0          2s
web-8   0/1     Pending             0          0s
web-8   0/1     Pending             0          0s
web-8   0/1     ContainerCreating   0          0s
web-8   0/1     ContainerCreating   0          1s
web-8   1/1     Running             0          1s
# 结果发现是其次创建，且前一个pod完全启动且可以使用才会在再次创建

#>>> 查看删除过程
$ kubectl get po -w
执行结果：
	NAME    READY   STATUS        RESTARTS   AGE
web-0   1/1     Running       0          16m
web-1   1/1     Running       0          16m
web-2   1/1     Running       0          6m16s
web-3   1/1     Running       0          6m14s
web-4   1/1     Running       0          6m13s
web-5   1/1     Running       0          4m34s
web-6   1/1     Running       0          4m33s
web-7   1/1     Terminating   0          4m31s
web-7   0/1     Terminating   0          4m32s
web-7   0/1     Terminating   0          4m32s
web-7   0/1     Terminating   0          4m32s
web-6   1/1     Terminating   0          4m34s
web-6   1/1     Terminating   0          4m34s
web-6   0/1     Terminating   0          4m35s
web-6   0/1     Terminating   0          4m35s
web-6   0/1     Terminating   0          4m35s
web-5   1/1     Terminating   0          4m36s
web-5   1/1     Terminating   0          4m36s
web-5   0/1     Terminating   0          4m37s
web-5   0/1     Terminating   0          4m37s
web-5   0/1     Terminating   0          4m37s
web-4   1/1     Terminating   0          6m16s
web-4   1/1     Terminating   0          6m16s
web-4   0/1     Terminating   0          6m17s
web-4   0/1     Terminating   0          6m17s
web-4   0/1     Terminating   0          6m17s
web-3   1/1     Terminating   0          6m18s
web-3   1/1     Terminating   0          6m18s
web-3   0/1     Terminating   0          6m19s
web-3   0/1     Terminating   0          6m19s
web-3   0/1     Terminating   0          6m19s
web-2   1/1     Terminating   0          6m21s
web-2   1/1     Terminating   0          6m21s
web-2   0/1     Terminating   0          6m22s
web-2   0/1     Terminating   0          6m22s
web-2   0/1     Terminating   0          6m22s
web-1   1/1     Terminating   0          16m
web-1   1/1     Terminating   0          16m
web-1   0/1     Terminating   0          16m
web-1   0/1     Terminating   0          16m
web-1   0/1     Terminating   0          16m
# 结果发现是倒序依次删除，且前一个pod完全删除后才会在从新删除
```







## 3、DaemonSet守护进程集

​			简介：DaemonSet（守护进程集，缩写为ds）和守护进程类似，它在符合匹配条件的节点上均部署一个Pod。当有新节点加入集群时，也会为它们新增一个Pod，当节点从集群中移除时，这些Pod也会被回收，删除DaemonSet将会删除它创建的所有Pod。

- 运行集群存储daemon(守护进程)，例如在每个节点上运行Glusterd、Ceph等;
- 在每个节点运行日志收集daemon，例如Fluentd、 Logstash;
- 在每个节点运行监控daemon，比如Prometheus Node Exporter、Collectd、Datadog代理、New Relic代理或Ganglia gmond。

###  3.1 DaemonSet守护进程集yaml文件编写

```yaml
$ vim nginx-ds.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.15.12
        name: nginx
```

```bash
#>>> 创建DaemonSet 
$ kubectl apply -f nginx-ds.yaml
执行结果：
	daemonset.apps/nginx created

#>>> 查看ds的pod
$ kubectl get po -owide
执行结果：（发现每个节点都部署了一个Pod）
	NAME          READY   STATUS    RESTARTS   AGE     IP               NODE           NOMINATED NODE   READINESS GATES
nginx-65zqs   1/1     Running   0          2m32s   172.25.244.226   k8s-master01   <none>           <none>
nginx-ds77x   1/1     Running   0          2m32s   172.25.92.92     k8s-master02   <none>           <none>
nginx-v4hxt   1/1     Running   0          2m32s   172.18.195.29    k8s-master03   <none>           <none>
nginx-xbwl8   1/1     Running   0          2m32s   172.27.14.228    k8s-node02     <none>           <none>
nginx-zzwrt   1/1     Running   0          2m33s   172.17.125.29    k8s-node01     <none>           <none>

#>>> 查看ds的实例
$ kubectl get ds
执行结果：
	NAME    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
nginx   5         5         5       5            5           <none>          3m32s

#>>> 查看其他命名空间下的ds
$ kubectl  get ds -n kube-system
执行结果：
	NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
calico-node   5         5         5       5            5           kubernetes.io/os=linux   10d
```



### 3.3 DaemonSet指定节点部署Pod（只部署在ssd硬盘标签的节点上）

yaml文件

```yaml
$ vim nginx-ds.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      nodeSelector:    # 标签
        disktype: ssd  # 标签类型 
      containers:
      - image: nginx:1.15.12
        name: nginx
```

```sh
#>>> 查看所有节点状态
$ kubetctl get nodes
执行结果：
      k8s-master01   Ready    <none>   11d   v1.23.14
      k8s-master02   Ready    <none>   11d   v1.23.14
      k8s-master03   Ready    <none>   11d   v1.23.14
	  k8s-node01     Ready    <none>   11d   v1.23.14
      k8s-node02     Ready    <none>   11d   v1.23.14

#>>> 将node节点打上标签(打上ssd硬盘标签)
$ kubectl label k8s-node01 k8s-node02 disktype=ssd
执行结果：
	node/k8s-node01 labeled
    node/k8s-node02 labeled

#>>> 更新DaemonSet
$ kubectl  replace -f nginx-ds.yaml
执行结果：
	daemonset.apps/nginx replaced

#>>> 查看Pod
$ kubectl get po
执行结果：（结果显示只在打了ssd标签的node节点上部署了Pod）
NAME          READY   STATUS    RESTARTS   AGE    IP              NODE         NOMINATED NODE   READINESS GATES
nginx-fd767   1/1     Running   0          2m5s   172.17.125.30   k8s-node01   <none>           <none>
nginx-lgtvz   1/1     Running   0          2m7s   172.27.14.229   k8s-node02   <none>           <none>

#>>> 查看集群默认的标签
$ kubectl get node --show-labels
执行结果：
NAME           STATUS   ROLES    AGE   VERSION    LABELS
k8s-master01   Ready    <none>   11d   v1.23.14   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master01,kubernetes.io/os=linux,node.kubernetes.io/node=
k8s-master02   Ready    <none>   11d   v1.23.14   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master02,kubernetes.io/os=linux,node.kubernetes.io/node=
k8s-master03   Ready    <none>   11d   v1.23.14   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master03,kubernetes.io/os=linux,node.kubernetes.io/node=
k8s-node01     Ready    <none>   11d   v1.23.14   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node01,kubernetes.io/os=linux,node.kubernetes.io/node=
k8s-node02     Ready    <none>   11d   v1.23.14   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node02,kubernetes.io/os=linux,node.kubernetes.io/node=

```

