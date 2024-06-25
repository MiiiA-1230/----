# Kubernetes使用心得

## 零散随笔

1. 在声明namespace、deploy、sts时，需要注意其命名规则
1. spec，意为规范，指期望该资源所能达到的状态

## 各组件说明

### 一、Pod

​		作为容器镜像的运行根容器存在，具有命名空间的隔离性。

​		Pod之间通过dns解析pod名获知pod网络ip来进行通讯，并为Pod的内部容器提供挂载的Volume和ip来进行通讯，这样既简化了密切关联的业务容器之间的通信问题，也很好地解决了它们之间的文件共享问题。

##### **Pod的两种类型：普通及静态**

​		普通Pod会被自动调度到最适合运行的node节点中，而静态Pod则会固定生成在拥有它yaml文件的特殊目录的Node中，即不被APIserver所通信的Pod容器。

##### 		三种探针

1. StartupProbe：用于判断Pod容器是否正常启动，属于有且仅有一次检测的探针。在检测成功前会屏蔽其他探针，检测成功后不再进行探测，检测失败则杀死容器并根据配置文件的重启策略重启容器。
2. LivenessProbe：用于判断Pod容器是否能够正常运行，探测失败则按照预先设定好的重启策略重启容器。
3. ReadinessProbe：用于判断Pod容器的服务是否能够正常运行，确保只有当容器准备好接受流量时，才会开始接收，保证服务的稳定性和可用性。

##### 		四种探针方式

1. ExecAction：在容器中执行一个命令，如果返回值为0则认为容器健康。
2. TCPSocketAction：通过TCP连接检测容器内端口是否通畅，如果通畅则认为容器健康。
3. HTTPGetAction：通过应用程序暴露的API地址来检查程序是否是正常的，如果状态码为200~400之间，则认为容器健康。
4. gRPC探针：1.24版本后才会开启，专门为检查运行 gRPC 服务的容器的健康状况而设计。它通过 gRPC 协议向应用发送健康检查请求，并期待收到一个标准的响应来判断服务是否健康。

##### PreStop

​		它允许你在容器终止之前运行一个命令或脚本，用于优雅地关闭服务、关闭连接或清理资源。

```bash
[root@k8s-master01 ~]# vim pod-prestop.yaml
---
apiVersion: v1 
kind: Pod 
metadata: 
  name: nginx
spec:
  containers: 
  - name: nginx 
    image: registry.cn-hangzhou.aliyuncs.com/hujiaming/nginx:1.24.0 # 必选，容器所用的镜像的地址
    imagePullPolicy: IfNotPresent
    lifecycle:
      postStart: # 容器创建完成后执行的指令, 可以是 exec httpGet TCPSocket
        exec:
          command:
          - sh
          - -c
          - 'mkdir /data/'
      preStop:
        exec:
          command:
          - sh
          - -c
          - sleep 10
    ports:
      - containerPort: 80
  restartPolicy: Never
  
#>>>创建Pod
[root@k8s-master01 ~]# kubectl  create  -f pod-prestop.yaml 
pod/nginx created

#>>>查看Pod
[root@k8s-master01 ~]# kubectl get po 
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          6s

#>>> 删除Pod
[root@k8s-master01 ~]# kubectl delete po nginx 

#>>> 查看Pod状态
[root@k8s-master01 ~]# kubectl get po 
NAME    READY   STATUS        RESTARTS   AGE
nginx   1/1     Terminating   0          22s

#>>> 查看Pod的停止记录
[root@k8s-master01 ~]# kubectl  describe po nginx
```

##### 定向调度

​		在默认情况下，一个Pod在哪个Node节点上运行，是由Scheduler组件采用相应的算法计算出来的，这个过程是不受人工控制的。但在实际使用中，我们想控制某些Pod到达某些节点上，那么就需要使用到Kubernetes对Pod调度的四种方式：

- 自动调度：默认使用，由Scheduler经过一系列的算法计算得出
- 定向调度：NodeName、NodeSelector

​		精确匹配，必须得有匹配项。如果没有满足条件的Node，那么Pod将不会被运行，即使在集群中还有可用Node列表也不行。

```yaml
#test01
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx: 1.17.1
  nodeName: n1#指定调度到n1节点上
  
#test02
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  nodeSelector:
    app: nginx#选择app: nginx的节点，也可以选择其他的node节点的属性
```

- 亲和度调度：NodeAffinity、PodAffinity、PodAntiAffinity

​		在NodeSelector的基础上进行了拓展，可以通过配置的形式，实现优先选择满足条件的Node进行调度，如果没有，也可以调度到不满足条件的节点上，使调度更加灵活。

###### **NodeAffinity:**

```yaml
#使用说明：
pod.spec.affinity.nodeAffinity
  requiredDuringSchedulingIgnoredDuringExecution:	#Node节点必须满足指定的所有规则才可以，相当于硬限制
    nodeSelectorTerms	#节点选择列表
      matchFields	#按节点字段列出的节点选择器要求列表
      matchExpressions	#按节点标签列出的节点选择器要求列表（推荐）
        key	#键
        values	#值
        #operat or 关系符	#支持Exists,DoesNotExist,In,NotIn,Gt,Lt
  preferredDuringSchedulingIgnoredDuringExecution	#优先调度到满足指定的规则的Node，相当于软限制
    preference	#一个节点选择器项，与相应的权重相关联
      matchFields	#按节点字段列出的节点选择器要求列表
      matchExpressions	#按节点标签列出的节点选择器要求列表（推荐）
        key	#键
        value	#值
        #operat or 关系符	#支持Exists,DoesNotExist,In,NotIn,Gt,Lt
    weight	倾向权重，在范围内1-100


#关系符使用说明：
- matchExpressions:
  - key: nodeenv		#匹配存在标签的key为nodeenv的节点
    operator: Exists
  - key: nodeenv		#匹配标签的key为nodeenv，且value是"xxx"或"yyy"的节点
    operator: In
    values: ["xxx","yyy"]
  - key: nodeenv		#匹配标签的key为nodeenv，且value大于"xxx"的节点
    operator: Gt
    values: "xxx"


#test01
apiVersion: v1
kind: Pod
metadata: 
  name: nginx
spec: 
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: nodeenv
            operator: In
            values: ["xxx","yyy"]
```

###### PodAffinity:

```yaml
#使用说明：
pod.spec.affinity.podAffinity
  requiredDuringSchedulingIgnoredDuringExecution:	#硬限制
    namespace	#指定参照pod的namespace
    topologyKey	#指定调度作用域
    labelSelector	#标签选择器
      matchExpressions	#按节点标签列出的节点选择器要求列表(推荐)
        key	#键
        values	#值
        operator	#关系符 支持In，NotIn，Exists，DoesNotExist
      matchLabels	#指多个matchExpressions映射的内容
    preferredDuringSchedulingIgnoredDuringExecution	#软限制
      podAffinityTerm	#选项
        namespaces
        topologyKey
        labelSelector
          matchExpressions
            key	#键
            values	#值
            operator
          matchLabels
      weight 倾向权重，在范围1-100
      

topologyKey用于指定调度时作用域，例如：
	如果指定为kubernetes.io/hostname，那就是以Node节点为区分范围
	如果指定为beta.kubernetes.io/os，则以Node节点的操作系统类型区分
	
	
test02:
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
        matchExpressions:
        - key: podenv
        operator: In
        values: ["xxx","yyy"]
    topologyKey: kubernetes.io/hostname
```

###### PodAntiAffinity:

```yaml
#使用说明：
pod.spec.affinity.podAntiAffinity
  requiredDuringSchedulingIgnoredDuringExecution:	#硬限制
    namespace	#指定参照pod的namespace
    topologyKey	#指定调度作用域
    labelSelector	#标签选择器
      matchExpressions	#按节点标签列出的节点选择器要求列表(推荐)
        key	#键
        values	#值
        operator	#关系符 支持In，NotIn，Exists，DoesNotExist
      matchLabels	#指多个matchExpressions映射的内容
    preferredDuringSchedulingIgnoredDuringExecution	#软限制
      podAffinityTerm	#选项
        namespaces
        topologyKey
        labelSelector
          matchExpressions
            key	#键
            values	#值
            operator
          matchLabels
      weight 倾向权重，在范围1-100
      

#同PodAntiAffinity类似


apiVersion: v1
kind: Pod
metadata: 
  name: nginx
spec:
  containers: 
  - name: nginx
    image: nginx
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecuting:	#硬限制
      - lableSelector:
        matchExpressions:	#匹配podenv的值在["pro"]中的标签
        - key: podenv
          operator: In
          values: ["pro"]
      topologyKey: kubernetes.io/hostname
```

- **污点（容忍）调度：Taints、Toleration**

​		**污点（Taints）：**

​		前面的调度方式都是站在Pod的角度上，通过在Pod上添加属性来确定Pod是否要调度到指定的Node上。而我们也可以站在Node的角度上，通过在Node上添加污点属性，来决定是否允许Pod调度过来。

​		Node被设置上了污点就和Pod之间增加了一层互斥关系，进而拒绝Pod调度进来，甚至可以将已存在的Pod驱逐出去。

污点格式为`key=value:effect`，key和value是污点的标签，effect描述污点的作用，共有三个选项：

effect选项：

- **PreferNoschedule：**尽量不要来，除非没办法
- **NoSchedule：**新的不要来，旧的不用动
- **NoExecute：**新的不要来，旧的也得走

```bash
#三种添加污点和去除污点的方式
kubectl taint nodes n1 tag=heima:PreferNoschedule
kubectl taint nodes n1 tag:PreferNoschedule-

kubectl taint nodes n1 tag=heima:NoSchedule
kubectl taint nodes n1 tag:NoSchedule-

kubectl taint nodes n1 tag=heima:NoExecute
kubectl taint nodes n1 tag:NoExecute-
```



​		**容忍（Toleration）：**

​		上面污点的作用是拒绝Pod容器被调度到Node节点上，但如果就是想将一个Pod容器调度到有污点的Node节点上去时，就需要使用容忍。

```yaml
#对应上文的污点操作

apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:	#添加容忍
  - key: "tag"	#要容忍的污点key
    operator: "Equal"	#操作符
    value: "heima"	#容忍的污点的value
    effect: "NoExecute"	#添加容忍的规则，这里必须和标记的污点规则相同
```

### 二、资源管理器

#### Deployment

​		意为无状态资源管理。无状态服务，即用户的交互对服务本身数据不造成影响的服务。

​		作为一种用于管理和部署容器化应用程序的资源管理器，支持滚动更新、回滚、扩缩容、自动修复等功能。

**部署方式**

1. kubectl向apiserver发送创建请求；
2. apiserver将 Deployment 持久化到etcd，etcd与apiserver进行一次http通信。
3. controller manager通过watch api监听 apiserver ，deployment controller看到了一个新创建的deplayment对象更后，将其从队列中拉出，根据deployment的描述创建一个ReplicaSet并将 ReplicaSet 对象返回apiserver并持久化回etcd。
   以此类推，当replicaset控制器看到新创建的replicaset对象，将其从队列中拉出，根据描述创建pod对象。
4. 接着scheduler调度器看到未调度的pod对象，根据调度规则选择一个可调度的节点，加载到pod描述中nodeName字段，并将pod对象返回apiserver并写入etcd。kubelet在看到有pod对象中nodeName字段属于本节点，将其从队列中拉出，通过容器运行时创建pod中描述的容器

```bash
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
```

#### StatefulSet

​		意为有状态服务管理器。有状态服务，即用户交互会产生数据并持久化到etcd中，影响后续服务状态。在创建服务集群时需要事先规划好，否则Pod之间会相互影响导致故障。

​		Sts中每个Pod都会有独一无二的网络标识符，并通过该标识符进行通讯。Handless使用Endpoint进行通信，格式为statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local。

- serviceName为Headless Service的名字，创建StatefulSet时必须指定Headless Service名称;
- 0..N-1为Pod所在的序号，从o开始到N-1;
- statefulSetName为StatefulSet的名字;
- namespace服务所在的命名空间;
- .cluster.local为Cluster Domain（集群域）。

```bash
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

#### DaemonSet守护进程集

​		它在符合匹配条件的节点上均部署一个Pod。当有新节点加入集群时，也会为它们新增一个Pod，当节点从集群中移除时，这些Pod也会被回收，删除DaemonSet将会删除它创建的所有Pod。

- 运行集群存储daemon(守护进程)，例如在每个节点上运行Glusterd、Ceph等;
- 在每个节点运行日志收集daemon，例如Fluentd、 Logstash;
- 在每个节点运行监控daemon，比如Prometheus Node Exporter、Collectd、Datadog代理、New Relic代理或Ganglia gmond。

```bash
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

### 三、Service

​		主要是为Pod提供一个稳定不变的IP地址，自带负载均衡机制。

Service发布类型共四种：

- `ClusterIP`：Kubernetes默认会自动设置Service的虚拟IP地址,仅可被集群内部的客户端应用访问。

- `NodePort`：在所有安装了Kube-Proxy的节点上打开一个端口，此端口可以代理至后端Pod，可以通过NodePort从集群外部访问集群内的服务。

- `LoadBalancer`：将Service映射到一个已存在的负载均衡器的IP地址上，通常在公有云环境中使用。

- `ExternalName`:将Service映射为一个外部域名地址，通过 externalName字段进行设置。



**Service对集群之外暴露服务的主要方式有两种：`NodePort`和`LoadBalancer`**

- `NodePort`方式的缺点是会占用很多集群机器的端口，那么当集群服务变多的时候，这个缺点就愈发明显
- `LoadBalancer`方式的缺点是每个service需要一个LB，浪费、麻烦，并且需要kubernetes之外设备的支持

**Service支持的网络协议**

- **TCP**：Service的默认网络协议，可用于所有类型的Service。
- **UDP**：可用于大多数类型的Service，LoadBalancer类型取决于 云服务商对UDP的支持。
- **HTTP**：取决于云服务商是否支持HTTP和实现机制。
- **PROXY**：取决于云服务商是否支持HTTP和实现机制。
- **SCTP**：从Kubernetes 1.12版本引入，到1.19版本时达到Beta阶段，默认启用。

##### Headless Service的概念和应用

​		在某些应用场景中，客户端应用不需要通过Kubernetes内置Service实现的负载均衡功能，或者需要自行完成对服务后端各实例的服务发现机制，或者需要自行实现负载均衡功能，此时可以通过创建一种特殊的 名为“Headless”的服务来实现。
​		Headless Service的概念是这种服务没有入口访问地址（无ClusterIP 地址），kube-proxy不会为其创建负载转发规则，而服务名（DNS域名）的解析机制取决于该Headless Service是否设置了Label Selector。

​		如果Headless Service设置了Label Selector，Kubernetes则将根据 Label Selector查询后端Pod列表，自动创建Endpoint列表，将服务名（DNS域名）的解析机制设置为：`当客户端访问该服务名时，得到的是全部Endpoint列表`（而不是一个确定的IP地址）。

```yaml
#以创建一个无头服务的nginxpod为例进行结果展示：

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
```

```shell
#创建后查看各项数据

[root@m1 yaml]# kubectl create -f nginx_headless.yaml 
service/nginx created
deployment.apps/nginx created

kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   5d19h
nginx        ClusterIP   None         <none>        80/TCP    62s

[root@m1 yaml]# kubectl get endpoints
NAME         ENDPOINTS             AGE
kubernetes   192.168.58.151:6443   5d19h
nginx        172.168.40.165:80     92s

[root@m1 yaml]# kubectl get po -owide
NAME                     READY   STATUS    RESTARTS   AGE    IP               NODE   NOMINATED NODE   READINESS GATES
nginx-6dc8dbb669-w2zh4   1/1     Running   0          2m5s   172.168.40.165   n1     <none>           <none>
```



##### Ingress

​		Ingress为Kubernetes集群中的服务提供了入口，可以提供负载均衡、SSL终止和基于名称的虚拟主机，应用的灰度发布等功能；在生产环境中常用的Ingress有Nginx、HAProxy、Istio等。[Ingress](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.26/#ingress-v1beta1-networking-k8s-io) 公开从集群外部到集群内[服务](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/)的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。提供7层（HTTP和HTTPS）路由功能。

​		Ingress是一种Kubernetes资源，用于将外部流量路由到Kubernetes集群内的服务。与NodePort相比，它提供了更高级别的路由功能和负载平衡，可以根据HTTP请求的路径、主机名、HTTP方法等来路由流量。

​		因此，可以说Ingress是为了弥补NodePort在流量路由方面的不足而生的。使用NodePort，只能将流量路由到一个具体的Service，并且必须使用Service的端口号来访问该服务。但是，使用Ingress，就可以使用自定义域名、路径和其他HTTP头来定义路由规则，以便将流量路由到不同的Service。

1. **「Ingress」**Ingress 是 Kubernetes 中的一个抽象资源，它提供了一种定义应用暴露入口的方法，可以帮助管理员在 Kubernetes 集群中管理多个服务的访问入口，方便用户访问。Ingress资源对象只是一个规范化的API对象，用于定义流量路由规则和 TLS 设置等信息。它本身不会直接处理或转发流量，而是需要配合一个 Ingress 控制器来实现。
2. **「Ingress Controller」**Ingress 控制器是一个独立的组件，它会监听 Kubernetes API 中的 Ingress 资源变化，并根据定义的路由规则配置负载均衡器、反向代理或其他网络代理，从而实现外部流量的转发。因此，可以将 Ingress 控制器视为 Ingress 资源的实际执行者。



###### **主流的Ingress Controller**

在 Kubernetes 中，有很多不同的 Ingress 控制器可以选择，例如 Nginx、Traefik、HAProxy、Envoy 等等。不同的控制器可能会提供不同的功能、性能和可靠性，可以根据实际需求来选择合适的控制器。Kubernetes生态系统中有许多不同的Ingress控制器可供选择，其中比较主流的有：

1. Nginx Ingress Controller：基于Nginx的Ingress控制器，提供了广泛的功能和配置选项。
2. Traefik Ingress Controller：Traefik是一个流行的反向代理和负载均衡器，Traefik Ingress Controller提供了灵活的配置选项和自动发现服务的功能。
3. Istio Ingress Gateway：Istio是一种服务网格，它提供了基于Envoy代理的Ingress Gateway来管理入站和出站流量。
4. Contour Ingress Controller：基于Envoy代理的Ingress控制器，具有高度可扩展性和灵活的路由规则。
5. Kong Ingress Controller：Kong是一个API网关，提供了可扩展的路由和服务管理功能。
6. Ambassador API Gateway：Ambassador是一个Kubernetes-native API Gateway，提供了自动化的服务发现和路由管理功能。

###### Ingress的部署方式

1. 部署一个独立的 Ingress 控制器 Pod：可以通过将 Ingress 控制器部署为一个独立的 Pod，使用 Kubernetes Service 对其进行负载均衡和暴露服务。
2. 部署 DaemonSet 类型的 Ingress 控制器：可以通过部署一个 DaemonSet 类型的 Ingress 控制器，使每个节点上都运行一个 Ingress 控制器 Pod，并通过 Kubernetes Service 对其进行负载均衡和暴露服务。
