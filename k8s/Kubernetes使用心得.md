# Kubernetes使用心得

### 零散随笔

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

### 二、Deployment

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

### 三、StatefulSet

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

### 四、DaemonSet守护进程集

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

### 五、Service

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
