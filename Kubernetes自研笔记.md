# Kubernetes使用心得

### 零散随笔

1. 在声明namespace、deploy、sts时，需要注意其命名规则

## 各组件说明

### 一、Pod

​		作为容器镜像的运行根容器存在，具有命名空间的隔离性。

​		Pod之间通过dns解析pod名获知pod网络ip来进行通讯，并为Pod的内部容器提供挂载的Volume和ip来进行通讯，这样既简化了密切关联的业务容器之间的通信问题，也很好地解决了它们之间的文件共享问题。

​      **Pod的两种类型：普通及静态**

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

### 二、Deployment

​		意为无状态资源管理。无状态服务，即用户的交互对服务本身数据不造成影响的服务。

​		作为一种用于管理和部署容器化应用程序的资源管理器，支持滚动更新、回滚、扩缩容、自动修复等功能。

**部署方式**

1. kubectl向apiserver发送创建请求；
2. apiserver将 Deployment 持久化到etcd，etcd与apiserver进行一次http通信。
3. controller manager通过watch api监听 apiserver ，deployment controller看到了一个新创建的deplayment对象更后，将其从队列中拉出，根据deployment的描述创建一个ReplicaSet并将 ReplicaSet 对象返回apiserver并持久化回etcd。
   以此类推，当replicaset控制器看到新创建的replicaset对象，将其从队列中拉出，根据描述创建pod对象。
4. 接着scheduler调度器看到未调度的pod对象，根据调度规则选择一个可调度的节点，加载到pod描述中nodeName字段，并将pod对象返回apiserver并写入etcd。kubelet在看到有pod对象中nodeName字段属于本节点，将其从队列中拉出，通过容器运行时创建pod中描述的容器

```
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

