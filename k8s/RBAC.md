[TOC]

## 一、前言	

​	**安全始终是Kubernetes发展过程中的一个关键领域。** 

​	从本质上来说，Kubernetes可被看作一个多用户共享资源的资源管理系统，这里的资源主要是各种Kubernetes里的各类资源对象，比如 Pod、Service、Deployment等。只有通过认证的用户才能通过Kubernetes的API Server查询、创建及维护相应的资源对象，理解这一点很关键。 

​	Kubernetes里的用户有两类：我们开发的运行在Pod里的应用；普通用户，如典型的kubectl命令行工具，基本上由指定的运维人员（集群管理员）使用。在更多的情况下，我们开发的Pod应用需要通过API Server查询、创建及管理其他相关资源对象，所以这类用户才是Kubernetes的关键用户。为此，Kubernetes设计了Service Account这个特殊的资源对象，代表Pod应用的账号，为Pod提供必要的身份认证。在此基础上， Kubernetes进一步实现和完善了基于角色的访问控制权限系统—— RBAC（Role-Based Access Control）。

​	在默认情况下，Kubernetes在每个命名空间中都会创建一个默认的名称为default的Service Account，因此Service Account是不能全局使用的，只能被它所在命名空间中的Pod使用。通过以下命令可以查看集群中的所有Service Account：

```bash
[root@k8s-master01 ~]# kubectl get sa -A
NAMESPACE              NAME                                 SECRETS   AGE
default                default                              1         11d
kube-node-lease        default                              1         11d
kube-public            default                              1         11d 
kubernetes-dashboard   default                              1         11d
```

![K8S RBAC详解_K8S RBAC](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406302126864.webp)

![image-20240630212922625](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406302129685.png)

## 二、Service Account

​	简介：Service Account是`通过Secret来保存对应的用户（应用）身份凭证的`，这些凭证信息有`CA根证书数据（ca.crt）和签名后的Token信息（Token`）。在Token信息中就包括了对应的Service Account的名称，因此API Server通过接收到的Token信息就能确定Service Account的身份。 在默认情况下，用户创建一个Pod时，Pod会绑定对应命名空间中的 default这个Service Account作为其“公民身份证”。当Pod里的容器被创建时，Kubernetes会把对应的Secret对象中的身份信息（ca.crt、Token等） 持久化保存到容器里固定位置的本地文件中，因此当容器里的用户进程通过Kubernetes提供的客户端API去访问API Server时，这些API会自动读取这些身份信息文件，并将其附加到HTTPS请求中传递给API Server 以完成身份认证逻辑。在身份认证通过以后，就涉及“访问授权”的问题，这就是RBAC要解决的问题了。



## 三、 API对象

​	**简介**：基于角色（Role）的访问控制（RBAC）是一种基于组织中用户的角色来调节控制对计算机或网络资源的访问的方法。RBAC 鉴权机制使用 `rbac.authorization.k8s.io` [API 组](https://kubernetes.io/zh-cn/docs/concepts/overview/kubernetes-api/#api-groups-and-versioning)来驱动鉴权决定， 允许你通过 Kubernetes API 动态配置策略。==默认启动，无需修改apiserver的配置文件参数。==
​	**官方**：https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/

​	RBAC API 声明了四种 Kubernetes 对象：==**Role**、**ClusterRole**、**RoleBinding** 和 **ClusterRoleBinding**==。你可以像使用其他 Kubernetes 对象一样，通过类似 `kubectl` 这类工具[描述对象](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/kubernetes-objects/#understanding-kubernetes-objects), 或修补对象。

### 3.1 Role和ClusterRole[ ](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/#role-and-clusterole)

​		简介：RBAC 的 **Role** 或 **ClusterRole** 中包含一组代表相关权限的控制。可以对集群或者某个命名空间下进行相关kubectl等操作； 这些权限是纯粹累加的（不存在拒绝某操作的规则）。这两种资源的名字不同（Role 和 ClusterRole） 是因为 Kubernetes 对象要么是名字空间作用域的，要么是集群作用域的，不可两者兼具。

- **Role**：总是用来在某个[名字空间](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/namespaces/)内设置访问权限； 在你创建 Role 时，你必须指定该 Role 所属的名字空间。
- **ClusterRole**： 则是一个集群作用域的资源。



**角色（Role）示例**

​	下面是一个Role定义示例，该角色具有在命名空间default中读取 （get、watch、list）Pod资源对象信息的权限：

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # API组，可以为空，例如deploy为apps
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

Role资源对象的主要配置参数都在rules字段中进行设置： 

- **resources：**需要操作的资源对象类型列表，例 如"pods"、"deployments"、"jobs"等。 
- **apiGroups：**资源对象API组列表，例如""（Core）、"extensions"、"apps"、"batch"等。 
-  **verbs：**设置允许对资源对象操作的方法列表，例如"get"、"watch"、"list"、"delete"、"replace"、"patch"等。

**集群角色（ClusterRole）示例**

​	集群角色除了具有和角色一致的命名空间内资源的管理能力，因其集群级别的范围，还可以用于以下授权应用场景中。

-  对集群范围内资源的授权，例如Node。 
- 对非资源型的授权，例如/healthz。 
-  对包含全部namespace资源的授权，例如pods（用于kubectl get pods--all-namespaces这样的操作授权）。 
- 对某个命名空间中多种权限的一次性授权。 

下面是一个ClusterRole定义示例，该集群角色有权访问一个或所有 namespace的secrets（根据其被RoleBinding还是ClusterRoleBinding绑定而定）的权限：

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```



### 3.2 RoleBinding和ClusterRoleBinding[ ](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding)

​	简介：角色绑定或集群角色绑定用来把一个角色绑定到一个目标主体上， 绑定目标可以是User（用户）、Group（组）或者Service Account。 RoleBinding用于某个命名空间中的授权，ClusterRoleBinding用于集群范围内的授权。



**角色绑定（RoleBinding）示例**

​	RoleBinding可以与属于相同命名空间的Role或者某个集群级别的 ClusterRole绑定，完成对某个主体的授权。 

​	下面是与相同命名空间中的Role进行绑定的示例，通过这个绑定操作，就完成了以下授权规则：**允许用户jane读取命名空间default的Pod资源对象信息**：

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

​	RoleBinding也可以引用ClusterRole，对目标主体在其所在命名空间授予在ClusterRole中定义的权限。一种常见的用法是集群管理员预先定义好一组ClusterRole（权限设置），然后在多个命名空间中重复使用这些ClusterRole。 

​	例如，在下面的例子中为用户“dave”授权一个ClusterRole“secret-reader”，虽然secret-reader是一个集群角色，但因为RoleBinding的作用范围为命名空间development，所以用户dave只能读取命名空间development中的secret资源对象，而不能读取其他命名空间中的secret资源对象：

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: development
subjects:
- kind: User
  name: dave
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

<img src="https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406301239168.png" alt="image-20240630123856113" style="zoom:67%;" />



**集群角色绑定（ClusterRoleBinding）示例**

​	ClusterRoleBinding用于进行集群级别或者对所有命名空间都生效的授权。下面的例子允许manager组的用户读取任意命名空间中的secret资源对象：

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

> 注意，在集群角色绑定（ClusterRoleBinding）中引用的角色只能是集群级别的角色（ClusterRole），而不能是命名空间级别的Role。

## 四、Role 和 ClusterRole使用相关示例

**查看集群的role（具有命名空间隔离性）**

```bash
[root@k8s-master ~]# kubectl  get role -A


[root@k8s-master ~]# kubectl  get role kube-proxy -n kube-system -oyaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  creationTimestamp: "2023-03-29T02:05:50Z"
  name: kube-proxy
  namespace: kube-system
  resourceVersion: "297"
  uid: da39328b-40cb-4985-bfe3-4bfe7d61dd22
rules:
- apiGroups:
  - ""
  resourceNames:
  - kube-proxy
  resources:
  - configmaps
  verbs:
  - get
```

### 4.1 role示例

下面是一个位于 "default" 名字空间的 Role 的示例，可用来授予对 [Pod](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/) 的读访问权限：

```yaml
apiVersion: rbac.authorization.k8s.io/v1  # api版本号
kind: Role # kind名称
metadata:
  namespace: default # 哪个命名空间进行限制
  name: pod-reader # role名称
rules:		
- apiGroups: [""]  # "" 标明 core API 组，资源属于哪个组，可以为空
  resources: ["pods"]  # 对这个组中的哪个资源进行授权
  verbs: ["get", "watch", "list"]  # 可以对这个资源进行什么样的操作
```

### 4.2 ClusterRole 示例[ ](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/#clusterrole-example)

​		ClusterRole 同样可以用于授予 Role 能够授予的权限。 因为 ClusterRole 属于集群范围，所以它也可以为以下资源授予访问权限：

- 集群范围资源（比如[节点（Node）](https://kubernetes.io/zh-cn/docs/concepts/architecture/nodes/)）
- 非资源端点（比如 `/healthz`）
- 跨名字空间访问的名字空间作用域的资源（如 Pod）比如，你可以使用 ClusterRole 来允许某特定用户执行 `kubectl get pods --all-namespaces`

下面是一个 ClusterRole 的示例，可用来为任一特定名字空间中的 [Secret](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/) 授予读访问权限， 或者跨名字空间的访问权限（取决于该角色是如何[绑定](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding)的）：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" 被忽略，因为 ClusterRoles 不受名字空间限制
  name: secret-reader
rules:
- apiGroups: [""]
  # 在 HTTP 层面，用来访问 Secret 资源的名称为 "secrets"
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

### 4.3 对资源的引用[ ](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/#referring-to-resources)（对某个资源进行细粒度的权限管理）

​		在 Kubernetes API 中，大多数资源都是使用对象名称的字符串表示来呈现与访问的。 例如，对于 Pod 应使用 "pods"。 RBAC 使用对应 API 端点的 URL 中呈现的名字来引用资源。 有一些 Kubernetes API 涉及 **子资源（subresource）**，例如 Pod 的日志。 对 Pod 日志的请求看起来像这样：

```sh
GET /api/v1/namespaces/{namespace}/pods/{name}/log
```

​		在这里，`pods` 对应名字空间作用域的 Pod 资源，而 `log` 是 `pods` 的子资源。 在 RBAC 角色表达子资源时，使用斜线（`/`）来分隔资源和子资源。 要允许某主体读取 `pods` 同时访问这些 Pod 的 `log` 子资源，你可以这样写：

```yaml
#对容器进行细粒度权限划分，pods/log:只能对容器的日志对哪些相关的操作
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-and-pod-logs-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]
```

​		对于某些请求，也可以通过 `resourceNames` 列表按名称引用资源。 在指定时，可以将请求限定为资源的单个实例。 下面的例子中限制可以 `get` 和 `update` 一个名为 `my-configmap` 的 [ConfigMap](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-pod-configmap/)：

```yaml
#对default命名空间下的名为my-configmap的configmap进行相关操作
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: configmap-updater
rules:
- apiGroups: [""]
  # 在 HTTP 层面，用来访问 ConfigMap 资源的名称为 "configmaps"
  resources: ["configmaps"]
  resourceNames: ["my-configmap"]
  verbs: ["update", "get"]
```

------



## 五、 RoleBinding 和 ClusterRoleBinding

### 5.1 RoleBinding 

​	简介：角色绑定（Role Binding）是将角色中定义的权限赋予一个或者一组用户。 它包含若干**主体（Subject）**（用户、组或服务账户）的列表和对这些主体所获得的角色的引用。 RoleBinding 在指定的名字空间中执行授权，而 ClusterRoleBinding 在集群范围执行授权。

一个 RoleBinding 可以引用同一的名字空间中的任何 Role。 或者，一个 RoleBinding 可以引用某 ClusterRole 并将该 ClusterRole 绑定到 RoleBinding 所在的名字空间。 如果你希望将某 ClusterRole 绑定到集群中所有名字空间，你要使用 ClusterRoleBinding。



**RoleBinding 示例**

​		下面的例子中的 RoleBinding 将 "pod-reader" Role 授予在 "default" 名字空间中的用户 "jane"。 这样，用户 "jane" 就具有了读取 "default" 名字空间中所有 Pod 的权限。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# 此角色绑定允许 "jane" 读取 "default" 名字空间中的 Pod
# 你需要在该命名空间中有一个名为 “pod-reader” 的 Role
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# 你可以指定不止一个“subject（主体）”
- kind: User
  name: jane # "name" 是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" 指定与某 Role 或 ClusterRole 的绑定关系
  kind: Role        # 此字段必须是 Role 或 ClusterRole
  name: pod-reader  # 此字段必须与你要绑定的 Role 或 ClusterRole 的名称匹配
  apiGroup: rbac.authorization.k8s.io
```

> **==RoleBinding 也可以引用 ClusterRole，以将对应 ClusterRole 中定义的访问权限授予 RoleBinding 所在名字空间的资源。这种引用使得你可以跨整个集群定义一组通用的角色， 之后在多个名字空间中复用==**

​		例如，尽管下面的 RoleBinding 引用的是一个 ClusterRole，"dave"（这里的主体， 区分大小写）只能访问 "development" 名字空间中的 Secrets 对象，因为 RoleBinding 所在的名字空间（由其 metadata 决定）是 "development":

```yaml
apiVersion: rbac.authorization.k8s.io/v1
# 此角色绑定使得用户 "dave" 能够读取 "development" 名字空间中的 Secrets
# 你需要一个名为 "secret-reader" 的 ClusterRole
kind: RoleBinding
metadata:
  name: read-secrets
  # RoleBinding 的名字空间决定了访问权限的授予范围。
  # 这里隐含授权仅在 "development" 名字空间内的访问权限。
  namespace: development
subjects:
- kind: User
  name: dave # 'name' 是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

> **==创建了绑定之后，你不能再修改绑定对象所引用的 Role 或 ClusterRole。 试图改变绑定对象的 `roleRef` 将导致合法性检查错误。 如果你想要改变现有绑定对象中 `roleRef` 字段的内容，必须删除重新创建绑定对象。==**



### 5.2 ClusterRoleBinding

​	要跨整个集群完成访问权限的授予，你可以使用一个 ClusterRoleBinding。 下面的 ClusterRoleBinding 允许 "manager" 组内的所有用户访问任何名字空间中的 Secret。

```bash
apiVersion: rbac.authorization.k8s.io/v1
# 此集群角色绑定允许 “manager” 组中的任何人访问任何名字空间中的 Secret 资源
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager      # 'name' 是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### 5.3 注意事项

​	创建了绑定之后，你不能再修改绑定对象所引用的 Role 或 ClusterRole。 试图改变绑定对象的 `roleRef` 将导致合法性检查错误。 如果你想要改变现有绑定对象中 `roleRef` 字段的内容，必须删除重新创建绑定对象。

这种限制有两个主要原因：

1. 将 `roleRef` 设置为不可以改变，这使得可以为用户授予对现有绑定对象的 `update` 权限， 这样可以让他们管理主体列表，同时不能更改被授予这些主体的角色。

2. 针对不同角色的绑定是完全不一样的绑定。要求通过删除/重建绑定来更改 `roleRef`， 这样可以确保要赋予绑定的所有主体会被授予新的角色（而不是在允许或者不小心修改了 `roleRef` 的情况下导致所有现有主体未经验证即被授予新角色对应的权限）。

## 六、聚合ClusterRole

​		简介：你可以将若干 ClusterRole **聚合（Aggregate）** 起来，形成一个复合的 ClusterRole。 作为集群控制面的一部分，控制器会监视带有 `aggregationRule` 的 ClusterRole 对象集合。`aggregationRule` 为控制器定义一个标签[选择算符](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/)供后者匹配应该组合到当前 ClusterRole 的 `roles` 字段中的 ClusterRole 对象，==类似于给多个clusterrelo打上相同的标签，后通过标签选择器进行权限聚合，形成一个完整的clsterrole供用户使用。==

​		下面是一个聚合 ClusterRole 的示例：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: [] # 控制面自动填充这里的规则
```

​		如果你创建一个与某个已存在的聚合 ClusterRole 的标签选择算符匹配的 ClusterRole， 这一变化会触发新的规则被添加到聚合 ClusterRole 的操作。 下面的例子中，通过创建一个标签同样为 `rbac.example.com/aggregate-to-monitoring: true` 的 ClusterRole，新的规则可被添加到 "monitoring" ClusterRole 中：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
# 当你创建 "monitoring-endpoints" ClusterRole 时，
# 下面的规则会被添加到 "monitoring" ClusterRole 中
rules:
- apiGroups: [""]
  resources: ["services", "endpointslices", "pods"]
  verbs: ["get", "list", "watch"]
```

