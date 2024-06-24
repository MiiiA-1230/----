[TOC]

### 前言

- **VPA(Vertical Pod Autoscaler)**：垂直扩展（Vertical Scaling），根据负载调整单个 Pod 的资源请求和限制，如 CPU 和内存。
- **HPA (Horizontal Pod Autoscaler)**：水平扩展（Horizontal Scaling），根据负载增加或减少 Pod 的副本数量。

### 1、简介

​	Horizontal Pod Autoscaler (HPA) 是 Kubernetes 中的一项功能，它能够根据 CPU 使用率或其他应用程序指标自动扩展或缩减应用程序的副本数量。HPA 有助于确保应用程序在负载增加时能够自动扩展以处理更多请求，在负载减少时则自动缩减以节省资源。



### 2、工作原理

1. Kubernetes中的某个Metrics Server持续采集所有Pod副本的指标数据。

2. HPA控制器通过Metrics Server的API获取这些数据，基于用户定义的扩缩容规则进行计算，得到目标Pod的副本数量。
3. 当目标Pod副本数量与当前副本数量不同时，HPA控制器就向Pod的副本控制器 （Deployment、RC或ReplicaSet）发起scale操作，调整Pod的副本数量， 完成扩缩容操作。

![image-20240622204540931](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406222045118.png)

### 3、HPA版本变革

```bash
[root@k8s-master01 ~]# kubectl get apiservices | grep -i auto
```

![image-20240622210337234](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406222103267.png)

- **autoscaling/v1版本**：仅支持基于CPU使用率指标的自动扩缩容。
- **autoscaling/v2版本**：支持基于内存使用率指标、自定义指标及外部指标的自动扩缩容，并且进一步扩展以支持多指标缩放能力，当定义了多个指标时，HPA会跟据每个指标进行计算，其中缩放幅度最大的指标会被采纳。

### 4、监控指标类型

​	kube-Master节点的`kube-controller-manager`服务持续监测目标Pod的某种性能指标，以计算是否需要调整副本数量。目前Kubernetes支持的指标类型如下：

- Pod资源使用率：Pod级别的性能指标，通常是一个比率值，例如CPU使用率。
- Pod自定义指标：Pod级别的性能指标，通常是一个数值，例如接收的请求数量。
- Object自定义指标或外部自定义指标：通常是一个数值，需要容器应用以某种方式提供，例如通过HTTP URL“/metrics”提供，或者使用外部服务提供的指标采集URL。

### 4、HPA配置资源清单

#### 4.1 基于CPU负载实现自动扩缩容

**deploy资源清单准备**

```yaml
[root@k8s-master01 ~]# vim nginx-deploy.yaml 
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
        resources:  # 资源请求配置
          requests:
            cpu: 10m # cpu资源请求书，1颗cpu=1000m


#>>> 创建资源
[root@k8s-master01 ~]# kubectl create -f nginx-deploy.yaml 
deployment.apps/nginx created

#>>> 查看资源
[root@k8s-master01 ~]# kubectl get po -owide
NAME                     READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
nginx-6dc8dbb669-nf5lm   1/1     Running   0          54s   172.16.195.23   k8s-master03   <none>           <none>
```

**service资源清单编写**

```yaml
#>>> kubectl命令行创建资源
[root@k8s-master01 ~]# kubectl expose deploy  nginx  --port=80   --dry-run=client  -oyaml  > nginx-service.yaml

#>>> 查看生成的资源清单
[root@k8s-master01 ~]# vim nginx-service.yaml 
---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
status:
  loadBalancer: {}

#>>> 创建serice资源
[root@k8s-master01 ~]# kubectl create -f nginx-service.yaml 
service/nginx created

#>>> 查看资源
[root@k8s-master01 ~]# kubectl get svc -owide
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE     SELECTOR
kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP   4d10h   <none>
nginx        ClusterIP   10.96.11.64   <none>        80/TCP    23s     app=nginx

#>>> 测试连通性
[root@k8s-master01 ~]# curl 10.96.11.64
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

#>>> 查看Pod的指标
[root@k8s-master01 ~]# kubectl top po 
NAME                     CPU(cores)   MEMORY(bytes)   
nginx-6dc8dbb669-nf5lm   0m           5Mi  
```

**HPA资源清单编写**

```bash
#>>> kubectl命令行生成资源清单
[root@k8s-master01 ~]# kubectl autoscale deploy nginx --cpu-percent=10 --min=1 --max=10  --dry-run=client -oyaml > nginx-hpa.yaml

#
---
apiVersion: autoscaling/v1   #API版本
kind: HorizontalPodAutoscaler  # 对象的类型
metadata:  # 元数据
  name: nginx  # HPA对象的名称
spec:  # 定义了HPA的规格
  maxReplicas: 10  # 定义了Pod副本数量的最大值，即在负载增加时，最多可以扩展到10 个Pod。
  minReplicas: 1  # 定义了Pod副本数量的最小值，即在负载减少时，最少保留1个Pod。
  scaleTargetRef:  # 指定HPA作用的目标对象。
    apiVersion: apps/v1  # 目标对象的API版本。
    kind: Deployment  # 目标对象的类型
    name: nginx  # 目标Deployment的名称
  targetCPUUtilizationPercentage: 10  # 目标CPU使用率，当CPU使用率超过10% 时，HPA会增加Pod副本数量；当CPU使用率低于10%时，HPA会减少Pod副本数量。
  
#>>> 创建资源
[root@k8s-master01 ~]# kubectl create -f nginx-hpa.yaml 
horizontalpodautoscaler.autoscaling/nginx created

#>>> 查看资源
[root@k8s-master01 ~]# kubectl get hpa
```

![image-20240622213350318](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406222133352.png)

> 字段解释：
>
> #### `NAME`：这是 HPA 对象的名称
>
> ####  `REFERENCE`：HPA 监控和自动扩展的目标对象。
>
> ####  `TARGETS`：当前和目标的资源使用情况
>
> #### `MINPODS`：HPA 配置的最小 Pod 副本数量。
>
> #### `MAXPODS`：HPA 配置的最大 Pod 副本数量
>
> ####  `REPLICAS`：当前运行的 Pod 副本数量。
>
> ####  `AGE`：HPA 对象的年龄

**测试压测**

```bash
[root@k8s-master01 ~]# while true; do curl http://10.96.11.64 > /dev/null; done

[root@k8s-master01 ~]# kubectl get hpa

[root@k8s-master01 ~]# kubectl top po 

[root@k8s-master01 ~]# kubectl get po 
```

> 注意事项：
> 	使用HPA CPU自动扩充是时，尽量用于前端应用，后端再扩容时，尽量使用自定义指标。后端应用会有多方面原因引起负载过高。