kubernetes服务发布

[TOC]

### 前言

​		每个Pod都会获取到它自己的IP地址，但是这些IP地址不总是稳定和可依赖的,这样就会导致一个问题：在Kubernetes集群中,如果一组Pod(比如后端的Pod）为其他Pod(比如前端的Pod提供服务，那么如果它们之间使用Pod的IP地址进行通信，在Pod重建后，将无法再进行连接。

​	于是Kubernetes引用了Service这样一种抽象概念:逻辑上的一组Pod，即一种可以访问Pod的策略。这一组Pod能够被Service通过标签选择器访问到，之后就可以使用Service进行通信，集群内部服务之间的访问通Service的名称来进行通信的，因为service的名称是亘古不变的。Kubernetes的Service定义了一个服务的访问入口地址，前端的应用（Pod）通过这个入口地址访问其背后的一组由Pod副本组成的集群实例，Service 与其后端Pod副本集群之间则是通过Label Selector来实现无缝对接的。

​	既然每个Pod都会被分配一个单独的IP地址，而且每个Pod都提供了一个独立的Endpoint（PodIP+ContainerPort）以、被客户端访问，现在多个Pod副本组成了一个集群来提供服务。

​	那么客户端如何来访问它们呢？一般的做法是部署一个负载均衡器（软件或硬件）， 为这组Pod开启一个对外的服务端口，如80端口，并且将这些Pod的Endpoint列表加入80端口的转发列表，客户端就可以通过负载均衡器的对外IP地址+服务端口来访问此服务。 客户端的请求最后会被转发到哪个Pod，由负载均衡器的算法所决定。

​	Kubernetes也遵循上述常规做法，运行在每个Node上的kube-proxy进程其实就是一个智能的软件负载均衡器，负责把对Service的请求转发到后端的某个Pod实例上，并在内部实现服务的负载均衡与会话保持机制。但Kubernetes发明了一种很巧妙又影响深远的设计： Service没有共用一个负载均衡器的IP地址，每个Service都被分配了一个全局唯一的虚拟IP地址，这个虚拟IP被称为Cluster IP。这样一来，每个服务就变成了具备唯一IP地址的通信节点，服务调用就变成了最基础的TCP网络通信问题。

​	Pod的Endpoint地址会随着Pod的销毁和重新创建而发生改变，因为新Pod的IP地址与之前旧Pod的不同。而Service一旦被创建，Kubernetes就会自动为它分配一个可用的Cluster IP，而且在Service的整个生命周期内，它的Cluster IP一般不会发生改变。 于是，服务发现这个棘手的问题在Kubernetes的架构里也得以轻松解决：只要用Service的Name与Service的Cluster IP地址做一个DNS域名映射即可完美解决问题。



### **1、Service:集群内部通讯流量成为东西流量(svc有ns隔离性)**

#### 1.1 简介

​	Service是Kubernetes实现微服务架构的核心概念，通过创建Service，可以为一组具有相同功能的容器应用提供一个统一的入口地址，并且将请求负载分发到后端的各个容器应用上。

​	Service用于为一组提供服务的Pod抽象一个稳定的网络访问地址，通过Service的定义设置的访问地址是DNS域名格式的服务名称，对于客户端应用来说，网络访问方式并没有改变（DNS域名的作用等价于主机名、互联网域名或IP地址）。 Service还提供了负载均衡器功能，将客户端请求负载分发到后端提供具体服务的各个Pod上。



#### 1.2 Kubernetes的服务发现机制

​	服务发现机制指客户端应用在一个Kubernetes集群中如何获知后端服务的访问地址。Kubernetes提供了两种机制供客户端应用以固定的方式获取后端服务的访问地址：环境变量方式和DNS方式

##### 1.2.1 环境变量方式

​			在一个Pod运行起来的时候，系统会自动为其容器运行环境注入所有集群中有效Service的信息。Service的相关信息包括服务IP、服务端口号、各端口号相关的协议等，通过**`{SVCNAME}_SERVICE_HOST和 {SVCNAME}_SERVICE_PORT`**格式进行设置。其中，SVCNAME的命名规则为：将Service的name字符串转换为全大写字母，将中横线“-”替 换为下画线“_”。

假设已经创建一个SVC，进去SVC后端的任意一个Pod中

```bash
[root@k8s-master01 ~]# kubectl exec -it nginx-6dc8dbb669-nf5lm -- sh
/ # env

NGINX_PORT_80_TCP=tcp://10.96.11.64:80
NGINX_SERVICE_HOST=10.96.11.64 # service的IP地址
NGINX_SERVICE_PORT=80		# service的端口
NGINX_PORT=tcp://10.96.11.64:80
NGINX_PORT_80_TCP_ADDR=10.96.11.64
NGINX_PORT_80_TCP_PORT=80
NGINX_PORT_80_TCP_PROTO=tcp

/ # curl $NGINX_SERVICE_HOST:$NGINX_SERVICE_PORT
```



##### 1.2.2 DNS方式

​			DNS方式Service在Kubernetes系统中遵循DNS命名规范，Service的DNS域名表示方法为`<servicename>.<namespace>.svc.<clusterdomain>`，其中servicename为服务的名称，namespace为其所在namespace的名称， clusterdomain为Kubernetes集群设置的域名后缀（例如cluster.local），服务名称的命名规则遵循gRFC 1123规范的要求。

​			对于客户端应用来说，DNS域名格式的Service名称提供的是稳定、 不变的访问地址，可以大大简化客户端应用的配置，是Kubernetes集群中推荐的使用方式。 当Service以DNS域名形式进行访问时，就需要在Kubernetes集群中存在一个DNS服务器来完成域名到ClusterIP地址的解析工作了，目前由CoreDNS作为Kubernetes集群的默认DNS服务器提供域名解析服务。

```bash
[root@k8s-master01 ~]# kubectl exec -it nginx-6dc8dbb669-nf5lm -- sh
# 跨命名空间访问
/ # nslookup  metrics-server.kube-system.svc.cluster.local
Server:		10.96.0.10
Address:	10.96.0.10:53


Name:	metrics-server.kube-system.svc.cluster.local
Address: 10.96.135.159

# 当前命名空间访问
/ # nslookup  kubernetes.default.svc.cluster.local
Server:		10.96.0.10
Address:	10.96.0.10:53


Name:	kubernetes.default.svc.cluster.local
Address: 10.96.0.1
```

#### 1.3 Label和Selector

##### 1.3.1 简介

######    1. Label简介

​	Label（标签）是Kubernetes系统中的另一个核心概念，相当于我们熟悉的“标签”。一个Label是一个`key=value键值对`，其中的key与value由用户自己指定。Label可以被附加到各种资源对象上，例如Node、 Pod、Service、Deployment等，一个资源对象可以定义任意数量的 Label，同一个Label也可以被添加到任意数量的资源对象上。Label通常在资源对象定义时确定，也可以在对象创建后动态添加或者删除。我们可以通过给指定的资源对象捆绑一个或多个不同的Label来实现多维度的资源分组管理功能，以便灵活、方便地进行资源分配、调度、配置、 部署等管理工作。

###### 	2. Selector简介

​	Label Selector（标签选择器）查询和筛选拥有某些Label的资源对象，Kubernetes通过这种方式实现了类似SQL的简单又通用的对象查询机制。Label Selector可以被类比为SQL语句中的where查询条件，例 如，“name=redis-slave”这个Label  Selector作用于Pod时，可以被类比 为`select * from pod where pod's name='redis-slave'`这样的语句。当前有两种Label Selector表达式：基于等式的（Equality-based）Selector表达式和基于集合的（Set-based）Selector表达式。

**基于等式的Selector表达式采用等式类表达式匹配标签：**

- `name=redis-slave`：匹配所有具有name=redis-slave标签的资源对 象。

- `env ！= production`：匹配所有不具有env=production标签的资源对象，比如“env=test”就是满足此条件的标签之一。 

**基于集合的Selector表达式则使用集合操作类表达式匹配标签：**

- `name  in (redis-master,redis-slave)`：匹配所有具有 name=redis-master标签或者name=redis-slave标签的资源对象。 
- `name not in (php-frontend)`：匹配所有不具有name=php-frontend标签的资源对象。

###### 3. 示例

```bash
#>>> 查看节点标签
[root@k8s-master01 ~]# kubectl get nodes --show-labels
```

![image-20240623121700265](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406231217425.png)

```bash
#>>> 查看符合标签的Pod
[root@k8s-master01 ~]# kubectl get po -l app=nginx
```

![image-20240623121754690](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406231217827.png)

```bash
#>>> 给固定节点打上标签
[root@k8s-master01 ~]# kubectl  label  node k8s-node01  k8s-node02 role=node node=work
node/k8s-node01 labeled
node/k8s-node02 labeled

[root@k8s-master01 ~]# kubectl  label  node k8s-master01   role=master
node/k8s-master01 labeled

#>>> 过滤节点标签
[root@k8s-master01 ~]# kubectl get node --show-labels | grep -i node=work

#>>> 集合查看
[root@k8s-master01 ~]# kubectl get nodes -l 'role in (master,node)'
NAME           STATUS   ROLES    AGE    VERSION
k8s-master01   Ready    <none>   5d1h   v1.23.16
k8s-node01     Ready    <none>   5d1h   v1.23.16
k8s-node02     Ready    <none>   5d1h   v1.23.16

#>>> 取反
[root@k8s-master01 ~]# kubectl get node -l 'role!=master'
NAME           STATUS   ROLES    AGE    VERSION
k8s-master02   Ready    <none>   5d1h   v1.23.16
k8s-master03   Ready    <none>   5d1h   v1.23.16
k8s-node01     Ready    <none>   5d1h   v1.23.16
k8s-node02     Ready    <none>   5d1h   v1.23.16


[root@k8s-master01 ~]# kubectl  get node -l 'node!=work,role=master'
NAME           STATUS   ROLES    AGE    VERSION
k8s-master01   Ready    <none>   5d1h   v1.23.16

[root@k8s-master01 ~]# kubectl  get node -l 'node!=work,role in (master)'
NAME           STATUS   ROLES    AGE    VERSION
k8s-master01   Ready    <none>   5d1h   v1.23.16

[root@k8s-master01 ~]# kubectl  get nodes -l role
NAME           STATUS   ROLES    AGE    VERSION
k8s-master01   Ready    <none>   5d1h   v1.23.16
k8s-node01     Ready    <none>   5d1h   v1.23.16
k8s-node02     Ready    <none>   5d1h   v1.23.16
```

**修改标签**

​	未修改之前标签

![image-20240623123217515](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406231232656.png)

```bash
#>>> 修改指定标签
[root@k8s-master01 ~]# kubectl label node  k8s-node01 k8s-node02  node=ssd --overwrite 
node/k8s-node01 unlabeled
node/k8s-node02 unlabeled
# 或者
[root@k8s-master01 ~]# kubectl label node -l node  node=arm  --overwrite
node/k8s-node01 labeled
node/k8s-node02 labeled

#>>> 查看新标签
[root@k8s-master01 ~]# kubectl get node --show-labels  | grep -i -w node=ssd
```

![image-20240623123405535](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406231234669.png)

**删除标签**

​	未删除之前截图

![image-20240623123405535](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406231239107.png)

```bash
[root@k8s-master01 ~]# kubectl label node k8s-node01 k8s-node02  node-

#>>> 查看节点标签
[root@k8s-master01 ~]# kubectl get node --show-labels
```

![image-20240623123838478](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406231238608.png)

#### 1.4 Service资源创建

```yaml
[root@k8s-master01 ~]#  vim  nginx-svc.yaml 
---
apiVersion: v1
kind: Service  
metadata:
  name: webapp
spec:
  selector:
    app: nginx       # pod标签
  ports:
    - protocol: TCP  # 协议
      port: 80       # service的端口
      targetPort: 80  # 容器端口号（代理的后端服务对外暴露的端口号）
      name: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx     # deploy的标签
  name: nginx
spec:
  replicas: 5      # 副本数
  selector:
    matchLabels:
      app: nginx   # pod的标签
  template:
    metadata:
      labels:
        app: nginx  # pod的标签
    spec:
      containers:
      - image: registry.cn-zhangjiakou.aliyuncs.com/taosweet/nginx:1.24.0-alpine
        name: nginx
        ports:
        - containerPort: 80
          name: nginx

#>>> 创建Service
[root@k8s-master01 ~]# kubectl apply -f  nginx-svc.yaml 
service/webapp created
deployment.apps/nginx created

#>>> 查看资源
[root@k8s-master01 ~]# kubectl get -f nginx-svc.yaml 
```

![image-20240623002412004](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406230024341.png)

​      **该示例为`webapp:80`即可访问到具有`app=nginx`标签的pod的80端口上**

​		需要注意的是，Service能够将一个接收端口映射到任意的targetPort，如果targetPort为空，targetPort将被设置为与Port字段相同的值。targetPort可以设置为一个字符串，引用backendPod的一个端口的名称，这样的话即使更改了Pod的端口，也不会对service的访问造成影响。

​		**Kubernetes Service 能够支持TCP,UDP,SCTP等协议，默认TCP协议**

```bash
#>>> 查看携带相同标签的Pod
[root@k8s-master01 ~]# kubectl get po -l app=nginx
```

![image-20240623002713313](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406230027360.png)

```bash
#>>> 访问Service
[root@k8s-master01 ~]# curl 10.96.151.135
```

![image-20240623002843495](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406230028556.png)

​	**在提供服务的Pod副本集运行过程中，如果Pod列表发生了变化，则Kubernetes的Service控制器会持续监控后端Pod列表的变化，实时更新Service对应的后端Pod列表。一个Service对应的`后端`由`Pod的IP和容器端口号`组成，即一个完整的`PodIP:ConrainerPort`访问地址，这在Kubernetes系统中叫作Endpoint。通过查看Service的详细信息，可以看到其后端Endpoint列表**

```bash
[root@k8s-master test]# kubectl  describe svc webapp
```

![image-20240623003053059](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406230030118.png)

```bash
#>>> 查看当前命名空间下的endpoints
[root@k8s-master test]# kubectl  get endpoints
```

![](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406230032575.png)

```bash
#>>> 查看其他命名空间下的svc
$ kubectl get svc -n kube-system
```

![image-20240623003524545](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406230035588.png)

```bash
#>>> 访问其他命令空间下的svc（进入一个其他命名空间下的容器中访问）
[root@k8s-master01 ~]# kubectl exec -it nginx-6784b87845-7wbwx -- sh
/ # nslookup metrics-server.kube-system.svc.cluster.local
```

![image-20240623003639887](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406230036947.png)



### **2、Service的负载均衡机制**

​		简介：==当一个Service对象在Kubernetes集群中被定义出来时，集群内的客 户端应用就可以通过服务IP访问到具体的Pod容器提供的服务了。从服 务IP到后端Pod的负载均衡机制，则是由每个Node上的kube-proxy负责实现的。对kube-proxy的代理模式、会话保持机制和基于拓扑感知 的服务路由机制（EndpointSlices）。==

#### **2.1 kube-proxy的代理模式**

​		目前kube-proxy提供了以下代理模式（通过启动参数--proxy-mode设置）

```bash
#>>> 二进制安装方式查看proxy-mode设置
[root@k8s-master01 ~]# vim  /etc/kubernetes/kube-proxy.yaml 
```

![image-20240623004009244](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406230040288.png)

- **iptables模式**：kube-proxy通过设置Linux Kernel的iptables规则， 实现从Service到后端Endpoint列表的负载分发规则，效率很高。但是， 如果某个后端Endpoint在转发时不可用，此次客户端请求就会得到失败的响应，此时应该通过为Pod设置 readinessprobe（服务可用性健康检查）来保证只有达到ready状态的 Endpoint才会被设置为Service的后端Endpoint。
- **ipvs模式**：在Kubernetes 1.11版本中达到Stable阶段，kubeproxy通过设置Linux Kernel的netlink接口设置IPVS规则，转发效率和支持的吞吐率都是最高的。ipvs模式要求Linux Kernel启用IPVS模块，如果操作系统未启用IPVS内核模块，kube-proxy则会自动切换至iptables模式。同时，ipvs模式支持更多的负载均衡策略：
     - **rr**：round-robin，轮询。
     - **lc**：least connection，最小连接数。
     - **dh**：destination hashing，目的地址哈希。 
     - **sh**：source hashing，源地址哈希。 
     - **sed**：shortest expected delay，最短期望延时。
     - **nq**：never queue，永不排队。



### 3、 Service发布类型

- `ClusterIP`：Kubernetes默认会自动设置Service的虚拟IP地址,仅可被集群内部的客户端应用访问。

- `NodePort`：在所有安装了Kube-Proxy的节点上打开一个端口，此端口可以代理至后端Pod，可以通过NodePort从集群外部访问集群内的服务。

- `LoadBalancer`：将Service映射到一个已存在的负载均衡器的IP地址上，通常在公有云环境中使用。

- `ExternalName`:将Service映射为一个外部域名地址，通过 externalName字段进行设置。

  

**Service对集群之外暴露服务的主要方式有两种：`NodePort`和`LoadBalancer`**

- `NodePort`方式的缺点是会占用很多集群机器的端口，那么当集群服务变多的时候，这个缺点就愈发明显
- LB方式的缺点是每个service需要一个LB，浪费、麻烦，并且需要kubernetes之外设备的支持



#### **3.1 NodePort类型使用**

​	如果将Service的 type字段设置为NodePort，则Kubernetes将从`--service-node-port-range`参数指定的范围(默认为30000-32767)中自动分配端口，也可以手动指定NodePort，创建该Service后，集群每个节点都将暴露一个端口，通过某个宿主机的IP+端口即可访问到后端的应用。

```bash
#>>> 二进制安装查看service的NodePort网段地址
[root@k8s-master01 ~]# grep "service-node-port-range" /usr/lib/systemd/system/kube-apiserver.service
      --service-node-port-range=30000-32767  \
```

```bash
#>>> 更改Service类型
[root@k8s-master01 ~]# vim nginx-svc.yaml 
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: nginx       # pod的标签
  ports:
    - protocol: TCP  # 协议
      port: 80       # service的端口
      targetPort: 80  # pod的端口号（代理的后端服务对外暴露的端口号）
      name: nginx
  type: NodePort # svc发布类型

#>>> 更新svc
[root@k8s-master01 ~]# kubectl replace -f nginx-svc.yaml 
service/webapp replaced

#>>> 查看svc资源
[root@k8s-master01 ~]# kubectl  get svc
```

![image-20240623133925243](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406231339365.png)



####   		3.2 ExternalName Service

​		ExternalName Service是Service的特例,它没有Selector，也没有定义任何端口和Endpoint,它通过返回该外部服务的别名来提供服务。比如可以定义一个Service，后端设置为一个外部域名，这样通过Service的名称即可访问到该域名。使用nslookup 解析以下文件定义的 Service，集群的 DNS服务将返回一个值为my.database.example.com的CNAME记录

```yaml
[root@k8s-master01 ~]# vim external-service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: www.baidu.com  # 外部服务的地址

#>>> 创建service资源
[root@k8s-master01 ~]# kubectl create -f external-service.yaml 
service/external-service created

#>>> 查看svc资源
[root@k8s-master01 ~]# kubectl get svc
```

![image-20240623142413602](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406231424728.png)

```bash
#>>> 访问service (进入Pod中访问svc)
[root@k8s-master01 ~]# kubectl exec -it nginx-6784b87845-7wbwx   -- sh
/ # wget http://external-service
Connecting to external-service (110.242.68.3:80)
wget: server returned error: HTTP/1.1 403 Forbidden
```



#### 3.3 Service代理Kubernetes集群之外的服务（重点）

​		普通的Service通过Label Selector对后端Endpoint列表进行了一次抽象，如果后端的Endpoint不是由Pod副本集提供的，则Service还可以抽象定义任意其他服务，`将一个Kubernetes集群外部的已知服务定义为 Kubernetes内的一个Service`，供集群内的其他应用访问，常见的应用场景包括：

- 希望在生产环境中使用外部的数据库集群，但测试环境使用自己的数据库。
-  已部署的一个集群外服务，例如数据库服务、缓存服务等
- 你正在将工作负载迁移到 Kubernetes。在评估该方法时，你仅在 Kubernetes 中运行一部分后端。
- 迁移过程中对某个服务进行Kubernetes内的服务名访问机制的验证。


**对于这种应用场景，用户在创建Service资源对象时不设置Label Selector（后端Pod也不存在），同时再定义一个与Service关联的Endpoint资源对象，在Endpoint中设置外部服务的IP地址和端口号**

![image-20240623140121396](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406231401562.png)

```yaml
[root@k8s-master01 ~]# vim nginx-svc-external.yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-svc-external
  name: nginx-svc-external
spec:
  ports:
  - name: http # Service端口名称
    port: 80 # Service端口
    protocol: TCP
    targetPort: 80  # 代理后端应用的端口
  type: ClusterIP 
---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    app: nginx-svc-external
  name: nginx-svc-external
subsets:
- addresses:
  - ip: 192.168.174.30
  - ip: 192.168.174.31
  ports:
  - name: http
    port: 80
    protocol: TCP
```

```bash
#>>> 创建资源
[root@k8s-master01 ~]# kubectl create -f nginx-svc-external.yaml
service/nginx-svc-external created

#>>> 查看service
[root@k8s-master01 ~]# kubectl  get svc
```

![image-20240623141046804](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406231410923.png)

```bash
#>>> 查看service详细信息
[root@k8s-master01 ~]# kubectl describe svc nginx-svc-external
```

![image-20240623141148220](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406231411341.png)

```bash
#>>> 查看endpoints详细信息
[root@k8s-master01 ~]# kubectl describe ep nginx-svc-external
```

![image-20240623141310073](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406231413196.png)

```bash
#>>> 测试访问svc
[root@k8s-master01 ~]# curl 10.96.151.135

#>>> 进入同ns下任意一个Pod测试
[root@k8s-master01 ~]# kubectl exec -it nginx-6784b87845-7wbwx   -- sh 
/ # curl 10.96.119.119
```



#### **3.4 多端口Service**

​			使用场景：某些服务对外暴露的端口不只有一个，所以此时需要用到多端口Service(rabbitmq,zookeeper)

```yaml
#>>> 同一个端口号使用的协议不同，如TCP和UDP，也需要设置为多个端口号来提供不同的服务
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: CoreDNS
  name: kube-dns
spec:
  - name: dns
    port: 53
    protocol: UDP
    targetPort: 53
  - name: dns-tcp
    port: 53
    protocol: TCP
    targetPort: 53
  - name: metrics
    port: 9153
    protocol: TCP
    targetPort: 9153
  selector:
    k8s-app: kube-dns
  type: ClusterIP
```

> 如果使用相同的协议，名称不能一样

#### 3.5 LoadBalancer类型（公有云使用）

​				简介：通常在公有云环境中设置Service的类型为“LoadBalancer”，可以将 Service映射到公有云提供的某个负载均衡器的IP地址上，客户端通过负载均衡器的IP和Service的端口号就可以访问到具体的服务，无须再通过kube-proxy提供的负载均衡机制进行流量转发。公有云提供的LoadBalancer可以直接将流量转发到后端Pod上，而负载分发机制依赖于公有云服务商的具体实现。

#### **3.6 Service支持的网络协议**

- **TCP**：Service的默认网络协议，可用于所有类型的Service。
- **UDP**：可用于大多数类型的Service，LoadBalancer类型取决于 云服务商对UDP的支持。
- **HTTP**：取决于云服务商是否支持HTTP和实现机制。
- **PROXY**：取决于云服务商是否支持HTTP和实现机制。
- **SCTP**：从Kubernetes 1.12版本引入，到1.19版本时达到Beta阶段，默认启用。



### 4、Headless Service的概念和应用

​		简介：在某些应用场景中，客户端应用不需要通过Kubernetes内置Service实现的负载均衡功能，或者需要自行完成对服务后端各实例的服务发现机制，或者需要自行实现负载均衡功能，此时可以通过创建一种特殊的 名为“Headless”的服务来实现。
​		Headless Service的概念是这种服务没有入口访问地址（无ClusterIP 地址），kube-proxy不会为其创建负载转发规则，而服务名（DNS域名）的解析机制取决于该Headless Service是否设置了Label Selector。

#### 4.1 Headless Service设置了Label Selector

​			如果Headless Service设置了Label Selector，Kubernetes则将根据 Label Selector查询后端Pod列表，自动创建Endpoint列表，将服务名（DNS域名）的解析机制设置为：`当客户端访问该服务名时，得到的是全部Endpoint列表`（而不是一个确定的IP地址）。

​			以下面的Headless Service为例，其设置了Label Selector：

```yaml
[root@k8s-master01 ~]# vim nginx-headless.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: registry.cn-zhangjiakou.aliyuncs.com/taosweet/nginx:1.24.0-alpine
        name: nginx
        resources:
          requests:
            cpu: 10m

#>>> 创建Headless Service
[root@k8s-master test]# kubectl  apply -f nginx-headless.yaml 

#>>> 查看svc
[root@k8s-master test]# kubectl  get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   9d
nginx        ClusterIP   None             <none>        80/TCP    85s
webapp       ClusterIP   10.106.192.111   <none>        80/TCP    4h36m

#>>> 查看endpoints
[root@k8s-master test]# kubectl  get endpoints
NAME         ENDPOINTS                                                  AGE
kubernetes   192.168.100.210:6443                                       9d
nginx        172.168.113.132:80,172.168.113.133:80,172.168.113.134:80   3m35s
webapp       172.168.113.132:80,172.168.113.133:80,172.168.113.134:80   4h39m

```

> ==注意：在解析service的时候，应在容器中解析，因为宿主机的DNS和集群内部属于隔离的，在宿主机的命令行中无法解析成功==



### **5、Ingress：(ingess控制器没有ns隔离性，Ingress有ns隔离性)**

#### 5.1 前言

​	根据前面对Service概念的说明，我们知道Service的表现形式为IP地址和端口号（ClusterIP:Port），即工作在TCP/IP层。而对于基于HTTP 的服务来说，不同的URL地址经常对应到不同的后端服务或者虚拟服务器（Virtual Host），这些应用层的转发机制仅通过Kubernetes的Service机制是无法实现的。Kubernetes从1.1版本开始引入Ingress资源对象，用 于将Kubernetes集群外的客户端请求路由到集群内部的服务上，同时提供7层（HTTP和HTTPS）路由功能。Ingress在Kubernetes 1.19版本时达 到v1稳定版本。 Kubernetes使用了一个Ingress策略定义和一个具体提供转发服务的Ingress Controller，两者结合，实现了基于灵活Ingress策略定义的服务路由功能。如果是对Kubernetes集群外部的客户端提供服务，那么Ingress Controller实现的是类似于边缘路由器的功能。需要注意的是，Ingress只能以HTTP和HTTPS提供服务，对于使用其他网络协议的服务，可以通过设置Service的类型（type）为NodePort或LoadBalancer 对集群外部的客户端提供服务。 `使用Ingress进行服务路由时，Ingress Controller基于Ingress规则将客户端请求直接转发到Service对应的后端Endpoint（Pod）上，这样会跳过kube-proxy设置的路由转发规则，以提高网络转发效率。`
​	Ingress为Kubernetes集群中的服务提供了入口，可以提供负载均衡、SSL终止和基于名称的虚拟主机，应用的灰度发布等功能；在生产环境中常用的Ingress有Nginx、HAProxy、Istio等。[Ingress](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.26/#ingress-v1beta1-networking-k8s-io) 公开从集群外部到集群内[服务](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/)的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。

![image-20240623144722857](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406231447037.png)

​		其中：

- 对http://mywebsite.com/api的访问将被路由到后端名为api的Service上；
-  对http://mywebsite.com/web的访问将被路由到后端名为web的Service上；
- 对http://mywebsite.com/docs的访问将被路由到后端名为doc 的Service上。

#### 5.2 创建Ingress Controller

​	官网链接：https://kubernetes.github.io/ingress-nginx/deploy/

​	简介：Ingress Controller需要**实现基于不同HTTP URL向后转发的负载分发规则，并可以灵活设置7层负载分发策略**。目前Ingress Controller已经有许多实现方案，包括Nginx、HAProxy、Kong、Traefik、Skipper、Istio 等开源软件的实现，以及公有云GCE、Azure、AWS等提供的Ingress应用网关，在Kubernetes中，Ingress Controller会持续监控API Server的/ingress 接口（即用户定义的到后端服务的转发规则）的变化。当/ingress接口后端的服务信息发生变化时，Ingress Controller会自动更新其转发规则。 

```bash
[root@k8s-master01 ~]# cat nginx-ingress.yaml 
---
apiVersion: v1
kind: Namespace
metadata:
  labels:
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  name: ingress-nginx
---
apiVersion: v1
automountServiceAccountToken: true
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx
  namespace: ingress-nginx
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx-admission
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx
  namespace: ingress-nginx
rules:
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - configmaps
  - pods
  - secrets
  - endpoints
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - networking.k8s.io
  resources:
  - ingressclasses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - coordination.k8s.io
  resourceNames:
  - ingress-nginx-leader
  resources:
  - leases
  verbs:
  - get
  - update
- apiGroups:
  - coordination.k8s.io
  resources:
  - leases
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - discovery.k8s.io
  resources:
  - endpointslices
  verbs:
  - list
  - watch
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx-admission
  namespace: ingress-nginx
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get
  - create
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - nodes
  - pods
  - secrets
  - namespaces
  verbs:
  - list
  - watch
- apiGroups:
  - coordination.k8s.io
  resources:
  - leases
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - networking.k8s.io
  resources:
  - ingressclasses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - discovery.k8s.io
  resources:
  - endpointslices
  verbs:
  - list
  - watch
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx-admission
rules:
- apiGroups:
  - admissionregistration.k8s.io
  resources:
  - validatingwebhookconfigurations
  verbs:
  - get
  - update
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx
subjects:
- kind: ServiceAccount
  name: ingress-nginx
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx-admission
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx-admission
subjects:
- kind: ServiceAccount
  name: ingress-nginx-admission
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx
subjects:
- kind: ServiceAccount
  name: ingress-nginx
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx-admission
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx-admission
subjects:
- kind: ServiceAccount
  name: ingress-nginx-admission
  namespace: ingress-nginx
---
apiVersion: v1
data:
  allow-snippet-annotations: "true"
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx-controller
  namespace: ingress-nginx
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  # externalTrafficPolicy: Local
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - appProtocol: http
    name: http
    port: 80
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  type: NodePort
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx-controller-admission
  namespace: ingress-nginx
spec:
  ports:
  - appProtocol: https
    name: https-webhook
    port: 443
    targetPort: webhook
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  minReadySeconds: 0
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/name: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
    spec:
      containers:
      - args:
        - /nginx-ingress-controller
        - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
        - --election-id=ingress-nginx-leader
        - --controller-class=k8s.io/ingress-nginx
        - --ingress-class=nginx
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: LD_PRELOAD
          value: /usr/local/lib/libmimalloc.so
        image: registry.cn-hangzhou.aliyuncs.com/hujiaming/nginx-ingress-controller:v1.6.4
        imagePullPolicy: IfNotPresent
        lifecycle:
          preStop:
            exec:
              command:
              - /wait-shutdown
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        name: controller
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        - containerPort: 443
          name: https
          protocol: TCP
        - containerPort: 8443
          name: webhook
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          requests:
            cpu: 100m
            memory: 90Mi
        securityContext:
          allowPrivilegeEscalation: true
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - ALL
          runAsUser: 101
        volumeMounts:
        - mountPath: /usr/local/certificates/
          name: webhook-cert
          readOnly: true
      dnsPolicy: ClusterFirst
      nodeSelector:
        kubernetes.io/os: linux
      serviceAccountName: ingress-nginx
      terminationGracePeriodSeconds: 300
      volumes:
      - name: webhook-cert
        secret:
          secretName: ingress-nginx-admission
---
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx-admission-create
  namespace: ingress-nginx
spec:
  template:
    metadata:
      labels:
        app.kubernetes.io/component: admission-webhook
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
        app.kubernetes.io/version: 1.6.4
      name: ingress-nginx-admission-create
    spec:
      containers:
      - args:
        - create
        - --host=ingress-nginx-controller-admission,ingress-nginx-controller-admission.$(POD_NAMESPACE).svc
        - --namespace=$(POD_NAMESPACE)
        - --secret-name=ingress-nginx-admission
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: registry.cn-beijing.aliyuncs.com/dotbalo/kube-webhook-certgen:v20220916-gd32f8c343
        imagePullPolicy: IfNotPresent
        name: create
        securityContext:
          allowPrivilegeEscalation: false
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: OnFailure
      securityContext:
        fsGroup: 2000
        runAsNonRoot: true
        runAsUser: 2000
      serviceAccountName: ingress-nginx-admission
---
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx-admission-patch
  namespace: ingress-nginx
spec:
  template:
    metadata:
      labels:
        app.kubernetes.io/component: admission-webhook
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
        app.kubernetes.io/version: 1.6.4
      name: ingress-nginx-admission-patch
    spec:
      containers:
      - args:
        - patch
        - --webhook-name=ingress-nginx-admission
        - --namespace=$(POD_NAMESPACE)
        - --patch-mutating=false
        - --secret-name=ingress-nginx-admission
        - --patch-failure-policy=Fail
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: registry.cn-hangzhou.aliyuncs.com/hujiaming/kube-webhook-certgen:v20220916-gd32f8c343
        imagePullPolicy: IfNotPresent
        name: patch
        securityContext:
          allowPrivilegeEscalation: false
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: OnFailure
      securityContext:
        fsGroup: 2000
        runAsNonRoot: true
        runAsUser: 2000
      serviceAccountName: ingress-nginx-admission
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.6.4
  name: ingress-nginx-admission
webhooks:
- admissionReviewVersions:
  - v1
  clientConfig:
    service:
      name: ingress-nginx-controller-admission
      namespace: ingress-nginx
      path: /networking/v1/ingresses
  failurePolicy: Fail
  matchPolicy: Equivalent
  name: validate.nginx.ingress.kubernetes.io
  rules:
  - apiGroups:
    - networking.k8s.io
    apiVersions:
    - v1
    operations:
    - CREATE
    - UPDATE
    resources:
    - ingresses
  sideEffects: None
  
#>>> 创建资源
[root@k8s-master01 ~]# kubectl create -f nginx-ingress.yaml 
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
serviceaccount/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
configmap/ingress-nginx-controller created
service/ingress-nginx-controller created
service/ingress-nginx-controller-admission created
deployment.apps/ingress-nginx-controller created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
ingressclass.networking.k8s.io/nginx created
```

```sh
#>>> 查看Ingress-container的Pod
[root@k8s-master01 ~]# kubectl get po -n ingress-nginx
```

![image-20240623212749607](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406232127716.png)

```bash
#>>> 查看ingress-nginx-controller的svc
[root@k8s-master01 ~]# kubectl get svc -n ingress-nginx
```

![image-20240623212637439](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406232126809.png)

**测试访问ingress-nginx的svc**

```bash
[root@k8s-master01 ~]# curl 10.96.158.32
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```



#### 5.3 域名发布服务

##### 			5.3.1 创建web服务

```yaml
[root@k8s-master01 ~]# vim nginx-deploy.yaml 
---
apiVersion: v1
kind: Service  
metadata:
  name: webapp
spec:
  selector:
    app: nginx       # pod的标签
  ports:
    - protocol: TCP  # 协议
      port: 80       # service的端口
      targetPort: 80  # pod的端口号（代理的后端服务对外暴露的端口号）
      name: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx     # deploy的标签
  name: nginx
spec:
  replicas: 5      # 副本数
  selector:
    matchLabels:
      app: nginx   # pod的标签
  template:
    metadata:
      labels:
        app: nginx  # pod的标签
    spec:
      containers:
      - image: registry.cn-zhangjiakou.aliyuncs.com/taosweet/nginx:1.24.0-alpine
        name: nginx
        ports:
        - containerPort: 80
          name: nginx


#>>> 创建资源
[root@k8s-master01 ~]# kubectl create -f nginx-deploy.yaml 
deployment.apps/nginx created

#>>> 查看svc
[root@k8s-master01 ~]# kubectl get svc
```

![image-20240623191535233](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406231915313.png)



##### 			5.3.2 创建Ingress

```yaml
[root@k8s-master01 ~]# vim nginx-ingress-file.yaml
---
apiVersion: networking.k8s.io/v1  # Ingress资源的API版本
kind: Ingress 
metadata:
  name: nginx-ingress  # Ingress 名称
spec:
  ingressClassName: nginx   # 指定IngressController的类型（haproxy等）
  rules:    #  路由转发规则，可以写多个
  - host: ingress.tanke.love  # Ingress 规则应用的域名
    http:
      paths:
      - backend:
          service:
            name: webapp   # service名称
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific #路由的匹配方式

#>>> 创建资源
[root@k8s-master01 ~]# kubectl create -f nginx-ingress-file.yaml
ingress.networking.k8s.io/nginx-ingress created
```

#####              5.3.3 路由的匹配方式

​	Ingress中的每个路径都需要有对应的路径类型（Path Type）。未明确设置 `pathType` 的路径无法通过合法性检查。当前支持的路径类型有三种：

- **`ImplementationSpecific`**：该类型行为取决于所使用的Ingress Controller。大多数情况下，它等同于前缀匹配。

  - ```bash
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: implementation-specific-ingress
    spec:
      rules:
      - host: "example.com"
        http:
          paths:
          - path: /foo/[a-z]+
            pathType: ImplementationSpecific
            backend:
              service:
                name: foo-service
                port:
                  number: 80
    ```

    > 在这个例子中，假设所用的Ingress Controller支持正则表达式匹配，那么路径如`/foo/bar`（但不是`/foo/123`）会被路由到`foo-service`服务。

- **`Exact`**：精确匹配要求请求路径与指定的路径完全相同。

  - ```bash
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: exact-ingress
    spec:
      rules:
      - host: "example.com"
        http:
          paths:
          - path: /foo
            pathType: Exact
            backend:
              service:
                name: foo-service
                port:
                  number: 80
    
    ```

    > 在这个例子中，只有路径严格等于`/foo`的请求会被路由到`foo-service`服务，路径如`/foo/bar`不会匹配。


- **`Prefix`**：前缀匹配是最常见的路径匹配方式，前缀匹配表示请求路径以指定的前缀开始。

  - ```bash
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: prefix-ingress
    spec:
      rules:
      - host: "example.com"
        http:
          paths:
          - path: /foo
            pathType: Prefix
            backend:
              service:
                name: foo-service
                port:
                  number: 80
    
    ```

    > 在这个例子中，所有以`/foo`开头的请求（如`/foo`、`/foo/bar`、`/foo/bar/baz`）都会被路由到`foo-service`服务。


```bash
#>>> 查看Ingress
[root@k8s-master01 ~]# kubectl get ingress
NAME            CLASS   HOSTS                ADDRESS         PORTS   AGE
nginx-ingress   nginx   ingress.tanke.love   10.96.110.188   80      55s

#>>> 查看Ingressclass
[root@k8s-master01 ~]# kubectl get ingressclass
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       39m

#>>> 查看nginx-ingress svc
[root@k8s-master01 ~]# kubectl get svc -n ingress-nginx
```

![image-20240623212637439](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406232136993.png)

```bash
[root@k8s-master01 ~]# curl ingress.tanke.love:30999
HTTP/1.1 200 OK
Date: Tue, 20 Dec 2022 08:13:34 GMT
Content-Type: text/html
Content-Length: 612
Connection: keep-alive
Last-Modified: Sat, 11 May 2019 00:35:53 GMT
ETag: "5cd618e9-264"
Accept-Ranges: bytes
```

​	**一旦Ingress资源成功创建，Ingress Controller就会监控到其配置的路由策略，并更新到Nginx的配置文件中生效。以本例中的Nginx Controller为例，它将更新其配置文件的内容为在Ingress中设定的路由策略。登录一个nginx-ingress-controller Pod，在/etc/nginx/nginx.conf可以看到对`ingress.tanke.love`的转发规则的正确配置**：

```sh
#>>> 进入ingress容器中
[root@k8s-master ingress]# kubectl   get po -n ingress-nginx
```

![image-20240623213742460](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406232137570.png)

```bash
[root@k8s-master01 ~]# kubectl exec -it ingress-nginx-controller-8556b74844-s9fwk -n ingress-nginx  -- bash

#>>> 查看nginx-ingress-controller配置文件
ingress-nginx-controller-8556b74844-s9fwk:/etc/nginx$ grep -A 20 "ingress.tanke.love" nginx.conf
```

![image-20240623213943935](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406232139103.png)



##### 5.3.4  不配置域名发布服务(了解)

​	在Kubernetes Ingress中，通常需要配置域名（Host）来指定外部请求的路由目标，但在某些情况下，您可能不想或不需要配置域名。您可以通过配置一个没有域名的Ingress资源来发布服务，这样所有请求都会匹配该Ingress规则，不论其Host头部是什么。

```bash
[root@k8s-master01 ~]# vim no-host.yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: no-host-ingress  #Ingress名称
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - backend:
          service:
            name: webapp
            port:
              number: 80
        path: /no-host
        pathType: prefix

#>>> 创建ingress资源
[root@k8s-master01 ~]# kubectl create -f no-host.yaml
ingress.networking.k8s.io/no-host created

#>>> 查看ingress资源
[root@k8s-master01 ~]# kubectl get ingress
执行结果：
	NAME            CLASS   HOSTS            ADDRESS        PORTS   AGE
    nginx-ingress   nginx   nginx.test.com   10.96.121.23   80      109m
    no-host         nginx   *                10.96.121.23   80      115s

#>>> 查看ingress container Pod
[root@k8s-master01 ~]# kubectl get po -n ingress-nginx
执行结果：
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-dl2rg        0/1     Completed   0          51m
ingress-nginx-admission-patch-svpd4         0/1     Completed   1          51m
ingress-nginx-controller-84c544699d-bb9xm   1/1     Running     0          51m

#>>> 测试访问
[root@k8s-master01 ~]# curl -I 172.16.32.180/no-host
HTTP/1.1 404 Not Found
Date: Tue, 20 Dec 2022 08:59:16 GMT
Content-Type: text/html
Content-Length: 154
Connection: keep-alive
```
