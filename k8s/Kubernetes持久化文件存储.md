Kubernetes持久化文件存储

[TOC]

官网：

​       https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/

简介：

​        容器内部存储的生命周期是短暂的，会随着容器环境的销毁而销毁，具有不稳定性。如果多个容器希望共享同一份存储，则仅仅依赖容器本身是很难实现的。在Kubernetes系统中，将对容器应用所需的存储资源抽象为存储卷（Volume）概念来解决这些问题。 Volume是与Pod绑定的（独立于容器）与Pod具有相同生命周期的资源对象。我们可以将Volume的内容理解为目录或文件，容器如需使用某个Volume，则仅需设置volumeMounts将一个或多个Volume挂载为容器中的目录或文件，即可访问Volume中的数据。Volume具体是什么类型，以及由哪个系统提供，对容器应用来说是透明的。

###     1、emptyDir

​	简介：

​			这种类型的Volume将在Pod被调度到Node时进行创建，在初始状态下目录中是空的，所以命名为“空目录”（Empty Directory），它与Pod具有相同的生命周期，当Pod被销毁时，Node上相应的目录也会被删除。 同一个Pod中的多个容器都可以挂载这种Volume。 由于EmptyDir类型的存储卷的临时性特点。

```yaml
#>>> 创建资源清单
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
      - image: registry.cn-hangzhou.aliyuncs.com/hujiaming/nginx:1.24.0
        imagePullPolicy: IfNotPresent
        name: nginx
        volumeMounts:
        - mountPath: /tmp
          name: test-volume
      - image: registry.cn-hangzhou.aliyuncs.com/hujiaming/tomcat:9.0.89-jdk8
        imagePullPolicy: IfNotPresent
        name: tomcat
        volumeMounts:
        - mountPath: /tmp
          name: test-volume
      restartPolicy: Always
      volumes:
      - name: test-volume
        emptyDir: {}
       # medium: Memory
```

```bash
 #>>> 创建资源
[root@k8s-master01 ~]# kubectl create  -f nginx-deploy.yaml 
deployment.apps/nginx created

#>>> 查看资源
[root@k8s-master01 ~]# kubectl get po  -w
NAME                     READY   STATUS    RESTARTS   AGE
nginx-66744bbb6b-8mg2r   2/2     Running   0          5s

#>>> 进入其中一个容器中
[root@k8s-master01 ~]# kubectl  exec -it nginx-66744bbb6b-8mg2r  -c nginx -- bash
root@nginx-66744bbb6b-8mg2r:/# df -Th
```

![image-20240628224527461](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406282245506.png)

```bash
#>>> 容器挂载目录创建测试文件
root@nginx-66744bbb6b-8mg2r:/# echo "hello world" > /tmp/test.txt  
root@nginx-66744bbb6b-8mg2r:/# ls /tmp/test.txt 
/tmp/test.txt
root@nginx-66744bbb6b-8mg2r:/# exit
exit

#>>> 进入Pod中另外一个容器中，查看挂载目录内容
[root@k8s-master01 ~]# kubectl  exec -it nginx-66744bbb6b-8mg2r  -c tomcat  -- bash
root@nginx-66744bbb6b-8mg2r:/usr/local/tomcat# ls /tmp/
hsperfdata_root  test.txt
root@nginx-66744bbb6b-8mg2r:/usr/local/tomcat# cat /tmp/test.txt 
hello world
```



###     2、hostPath

​          介绍：HostPath类型的存储卷用于将Node文件系统的目录或文件挂载到容器内部使用。对于大多数容器应用来说，都不需要使用宿主机的文件系统。适合使用HostPath存储卷的一些应用场景如下： 

- 容器应用的关键数据需要被持久化到宿主机上。 
- 监控系统，例如cAdvisor需要采集宿主机/sys目录下的内容。

![image-20240628210919293](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406282109653.png)

**使用hostPath卷的示例**

​	示例： 修改容器时区。默认容器时间为UTC时间。

```yaml
#>>> 创建资源清单
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
      - image: registry.cn-hangzhou.aliyuncs.com/hujiaming/nginx:1.24.0
        imagePullPolicy: IfNotPresent
        name: nginx
        volumeMounts:
        - mountPath: /etc/timezone
          name: timezone
        - mountPath: /etc/localtime
          name: date
      restartPolicy: Always
      volumes:
      - name: timezone
        hostPath:
          path: /etc/timezone
          type: File
      - name: date
        hostPath:
          path: /etc/localtime
          type: File
```

`hostPath`工作类型

- type 为空字符串：默认选项，意味着挂载 hostPath卷之前不会执行任何检查。

-  DirectoryOrCreate：如果给定的path不存在任何东西，那么将根据需要创建一个权限为0755 的空目录，和 Kubelet 具有相同的组和权限。

- Directory：目录必须存在于给定的路径下。

- FileOrCreate：如果给定的路径不存储任何内容，则会根据需要创建一个空文件，权限设置为 0644，和 Kubelet 具有相同的组和所有权。

- `File`：文件，必须存在于给定路径中。

-  Socket：UNIX 套接字，必须存在于给定路径中。

- CharDevice：字符设备，必须存在于给定路径中。

- BlockDevice：块设备，必须存在于给定路径中

### 3、NFS

​		简介：`nfs` 卷能将 NFS (网络文件系统) 挂载到你的 Pod 中。 不像 `emptyDir` 那样会在删除 Pod 的同时也会被删除，`nfs` 卷的内容在删除 Pod 时会被保存，卷只是被卸载。 这意味着 `nfs` 卷可以被预先填充数据，并且这些数据可以在 Pod 之间共享。

```bash
#>>> 所有安装nfs服务
$  yum -y install nfs-utils rpcbind

#>>> master01作为服务端
[root@k8s-master01 ~]# mkdir -p /data/nfs

[root@k8s-master01 ~]# chmod 666 /data/nfs

[root@k8s-master01 ~]# cat /etc/exports
/data/nfs  192.168.174.0/24(rw,sync,no_root_squash,no_subtree_check)

[root@k8s-master01 ~]# exportfs -r

[root@k8s-master01 ~]# systemctl enable --now rpcbind nfs


#>>> 客户端测试
[root@k8s-master01 ~]# showmount  -e 192.168.174.30
```

```bash
#>>> 编写挂载测试资源清单
[root@k8s-master01 ~]# cat nginx-deploy.yaml 
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
      - image: registry.cn-hangzhou.aliyuncs.com/hujiaming/nginx:1.24.0
        imagePullPolicy: IfNotPresent
        name: nginx
        volumeMounts:
        - mountPath: /etc/timezone
          name: timezone
        - mountPath: /etc/localtime
          name: date
        - mountPath: /tmp
          name: nfs-volume
      restartPolicy: Always
      volumes:
      - name: timezone
        hostPath:
          path: /etc/timezone
          type: File
      - name: date
        hostPath:
          path: /etc/localtime
          type: File
      - name: nfs-volume
        nfs:
          server: 192.168.174.30
          path: /data/nfs/test


#>>> 创建资源
[root@k8s-master01 ~]# kubectl apply -f nginx-deploy.yaml 

#>>> 测试
[root@k8s-master01 ~]# kubectl exec  nginx-54879c9c9c-zp4zl  -- touch  /tmp/222

#>>> 查看服务端挂载路径
[root@k8s-master01 test]# ll /data/nfs/test/
-rw-r--r-- 1 root root 0 6月  29 22:00 222
```



### 4、持久卷

​		Kubernetes对于有状态的容器应用或者对数据需要持久化的应用，不仅需要将容器内的目录挂载到宿主机的目录或者emptyDir临时存储卷，而且需要更加可靠的存储来保存应用产生的重要数据，以便容器应用在重建之后仍然可以使用之前的数据。不过，存储资源和计算资源（CPU/内存）的管理方式完全不同。为了能够屏蔽底层存储实现的细节，让用户方便使用，同时让管理员方便管理，Kubernetes从1.0版本就引入PersistentVolume（PV）和PersistentVolumeClaim（PVC）两个资源对象来实现对存储的管理子系统。
​	

​		**持久卷（PersistentVolume，PV）** 是集群中的一块存储，可以由管理员事先制备， 或者使用[存储类（Storage Class）](https://kubernetes.io/zh-cn/docs/concepts/storage/storage-classes/)来动态制备。 持久卷是集群资源，就像节点也是集群资源一样。PV 持久卷和普通的 Volume 一样， 也是使用卷插件来实现的，只是它们拥有独立于任何使用 PV 的 Pod 的生命周期。 此 API 对象中记述了存储的实现细节，无论其背后是 NFS、iSCSI 还是特定于云平台的存储系统。
​		**持久卷申领（PersistentVolumeClaim，PVC）** 表达的是用户对存储的请求。概念上与 Pod 类似。 Pod 会耗用节点资源，而 PVC 申领会耗用 PV 资源。Pod 可以请求特定数量的资源（CPU 和内存）。同样 PVC 申领也可以请求特定的大小和访问模式 （例如，可以挂载为 ReadWriteOnce、ReadOnlyMany、ReadWriteMany 或 ReadWriteOncePod， 请参阅[访问模式](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/#access-modes)）。

尽管 PersistentVolumeClaim 允许用户消耗抽象的存储资源， 常见的情况是针对不同的问题用户需要的是具有不同属性（如，性能）的 PersistentVolume 卷。 集群管理员需要能够提供不同性质的 PersistentVolume， 并且这些 PV 卷之间的差别不仅限于卷大小和访问模式，同时又不能将卷是如何实现的这些细节暴露给用户。 为了满足这类需求，就有了**存储类（StorageClass）** 资源。



#### 4.1 PersistentVolume

​	简介：简称PV，是由Kubernetes管理员设置的存储，可以配置Ceph、NFS、GlusterFS等常用存储配置，相对于Volume配置，提供了更多的功能，比如生命周期的管理、大小的限制。PV分为静态和动态。PV可以被理解成Kubernetes集群中的某个网络存储对应的一块存储，PV作为存储资源，主要包括存储能力、访问模式、存储类型、回收策略、后端存储类 型等关键信息的设置。

**PV的关键配置参数**

​	**存储能力（Capacity）：**

​		描述存储设备具备的能力，目前仅支持对存储空间的设置。目前，存储大小是可以设置和请求的唯一资源。 未来可能会包含 IOPS、吞吐量等属性。

**存储卷模式（Volume Mode）：**

​		针对 PV 持久卷，Kubernetes 支持两种卷模式（`volumeModes`）：`Filesystem（文件系统）` 和 `Block（块）`。 `volumeMode` 是一个可选的 API 参数。 如果该参数被省略，默认的卷模式是 `Filesystem`。`volumeMode` 属性设置为 `Filesystem` 的卷会被 Pod **挂载（Mount）** 到某个目录。 如果卷的存储来自某块设备而该设备目前为空，Kuberneretes 会在第一次挂载卷之前在设备上创建文件系统。你可以将 `volumeMode` 设置为 `Block`，以便将卷作为原始块设备来使用。 这类卷以块设备的方式交给 Pod 使用，其上没有任何文件系统。 这种模式对于为 Pod 提供一种使用最快可能方式来访问卷而言很有帮助， Pod 和卷之间不存在文件系统层。另外，Pod中运行的应用必须知道如何处理原始块设备。 关于如何在 Pod 中使用 `volumeMode: Block` 的卷， 可参阅[原始块卷支持](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/#raw-block-volume-support)。



**PV回收策略（Reclaim Policy）**：

- `Retain`：保留，该策略允许手动回收资源，当删除PVC时，PV仍然存在，PV被视为已释放，管理员可以手动回收卷。
- `Recycle`：回收，如果Volume插件支持，Recycle策略会对卷执行 ==rm -rf==清理该PV，并使其可用于下一个新的PVC，但是本策略将来会被弃用，目前只有NFS和HostPath支持该策略。
- `Delete`：删除，如果Volume插件支持，删除PVC时会同时删除PV，动态卷默认为Delete，目前支持Delete的存储后端包括AWS EBS, GCE PD, Azure Disk, or OpenStack Cinder等。



**PV访问策略（Access Modes）**：

- `ReadWriteOnce`：可以被单节点以读写模式挂载，命令行中可以被缩写为RWO。（块存储
- `ReadOnlyMany`：可以被多个节点以只读模式挂载，命令行中可以被缩写为ROX。
- `ReadWriteMany`：可以被多个节点以读写模式挂载，命令行中可以被缩写为RWX。（文件共享存储）
- `ReadWriteOncePod`：可以被单个 Pod 以读写方式挂载。 如果你想确保整个集群中只有一个 Pod 可以读取或写入该 PVC， 请使用 ReadWriteOnce访问模式。这只支持 CSI 卷以及需要 Kubernetes 1.22 以上版本。命令行中可以缩写为RWOP。



**PV周期的各个阶段：**

- **Available：**可用，没有被PVC绑定的空闲资源。
- **Bound：**已绑定，已经被PVC绑定。
- **Released：**已释放，PVC被删除，但是资源还未被重新使用。
- **Failed：**失败，自动回收失败

🚒：PV和PVC成功创建并不代表挂载成功，需要Pod成功启动过后才可以发现挂载是否成功。



**存储分类**

- **文件存储：**一些数据可能需要被多个节点使用，比如用户的头像、用户上传的文件等，实现方式：NFS、NAS、FTP、CephFS等。
-  **块存储：**一些数据只能被一个节点使用，或者是需要将一块裸盘整个挂载使用，比如数据库、Redis等，实现方式：Ceph、GlusterFS、公有云。
- **对象存储：**由程序代码直接实现的一种存储方式，云原生应用无状态化常用的实现方式，实现方式：一般是符合S3协议的云存储，比如AWS的S3存储、Minio、七牛云等



##### 4.1.1 PV创建

​        **注意：PV没有命名空间隔离性**

1. **NFS配置**

```bash
#>>> 服务端安装NFS
[root@k8s-master01 ~]# yum install nfs-utils rpcbind -y

#>>> 服务端创建共享目录
[root@k8s-master01 ~]# mkdir -p /data/k8s

[root@k8s-master01 ~]# vim /etc/exports 
/data/ 192.168.174.0/24(rw,sync,no_subtree_check,no_root_squash)

[root@k8s-master01 ~]# exportfs -r
	
#>>> 服务端启动NFS
[root@k8s-master01 ~]# systemctl restart nfs rpcbind

#>>> 所有客户端安装nfs-utils
$ yum install -y nfs-utils

#>>> 挂载测试
$ mount -t nfs 192.168.174.30:/data/ /mnt/
	
#>>> 取消挂载
$ umount /mnt/


#>>> 创建资源清单
[root@k8s-master01 ~]# vim nfs-pv.yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs  # PV名称
spec:
  capacity:
    storage: 5Gi  # 容量配置，nfs不支持，配置时需要后端存储设备支持
  volumeMode: Filesystem # 卷的模式，目前支持Filesystem（文件系统）和Block（块），其中Block类型需要后端存储支持，默认为文件系统
  accessModes:
  - ReadWriteOnce     # PV的访问模式
  persistentVolumeReclaimPolicy: Recycle   # PV的回收策略
  storageClassName: nfs-slow   # PV的类，一个特定类型的PV只能绑定到特定类别的PVC；
  nfs:
    path: /data/k8s    # NFS上的共享目录
    server: 192.168.174.30   # NFS的IP地址

#>>> 创建PV
[root@k8s-master01 ~]# kubectl create -f nfs-pv.yaml
	
#>>> 查看PV
[root@k8s-master01 ~]# kubectl get pv 
```

![image-20240630002010458](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406300020549.png)

**hostPath配置**

```bash
[root@k8s-master01 ~]# vim hostPath-pv.yaml
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: hostpath
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:  # hostPath服务配置
    path: "/mnt/data"  # 宿主机路径

#>>> 创建PV
[root@k8s-master01 ~]# kubectl create -f hostPath-pv.yaml
	
#>>> 查看PV
[root@k8s-master01 ~]# kubectl  get -f hostPath-pv.yaml 
```

![image-20240630002447200](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406300024304.png)





#### 4.2 PersistentVolumeClaim

​	简称PVC，是对存储PV的请求，表示需要什么类型的PV，需要存储的技术人员只需要配置PVC即可使用存储，或者Volume配置PVC的名称即可。PVC作为用户对存储资源的需求申请，主要包括存储空间请求、访问模式、PV选择条件和存储类别等信息的设置。



##### 4.2.1 PVC的关键配置参数

**资源请求（Resources）：**

​		描述对存储资源的请求。申请的资源只能小于或等于pv的容量。但如果申请的容量小于PV的实际容量，那么剩余未申请容量会浪费

**访问模式（Access Modes）：**

​		PVC也可以设置访问模式，用于描述用户应用对存储资源的访问权限。访问模式的设置与PV的设置相同。

**存储卷模式（Volume Modes）：**

​		PVC也可以设置存储卷模式，用于描述希望使用的PV存储卷模式，包括文件系统和块设备。



**PVC创建(PVC受命名空间隔离性)**

```bash
[root@k8s-master01 ~]# vim pvc-nfs.yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: task-pvc-claim   # pvc的名称
spec:
  storageClassName: nfs-slow     # pv SC名称
  accessModes:
  - ReadWriteOnce         # 和PV的访问策略一致
  resources:
    requests：
      storage: 3Gi      # 只能小于等于PV的容量

#>>> 创建PVC
[root@k8s-master01 ~]# kubectl create -f pvc-nfs.yaml 
persistentvolumeclaim/task-pvc-claim created

#>>> 查看PVC 
[root@k8s-master01 ~]# kubectl get -f pvc-nfs.yaml
NAME             STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
task-pvc-claim   Bound    pv-nfs   5Gi        RWO            nfs-slow       17s
	
#>>> 查看绑定的PV
[root@k8s-master01 ~]# kubectl get -f  nfs-pv.yaml
```

![image-20240630002901244](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406300029297.png)

> 结果显示pv的状态变为Bound，已被挂载。

**创建deploy实例，测试挂载PVC**

```bash
[root@k8s-master01 ~]# vim nfs-pvc-deploy.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
    labels:
      app: nginx
    name: nginx
    namespace: default
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
        - image: registry.cn-hangzhou.aliyuncs.com/hujiaming/nginx:1.24.0
          imagePullPolicy: IfNotPresent
          name: nginx
          volumeMounts:
          - mountPath: "/usr/share/nginx/html"  # 挂载路径
            name: task-pv-storage
        restartPolicy: Always
        volumes:
        - name: task-pv-storage     # volume名称
          persistentVolumeClaim:
            claimName: task-pvc-claim     # pvc名称

#>>> 创建资源
[root@k8s-master01 ~]# kubectl create -f nfs-pvc-deploy.yaml

#>>> 查看pod
[root@k8s-master01 ~]# kubectl get po 
NAME                    READY   STATUS    RESTARTS   AGE
nginx-8d486646b-tcrp4   1/1     Running   0          5s
nginx-8d486646b-v46hc   1/1     Running   0          5s

#>>> 测试
[root@k8s-master01 ~]# kubectl exec  nginx-8d486646b-tcrp4  -- df -Th
```



#### 4.3 PV和PVC的生命周期 

​		可以将PV看作可用的存储资源，PVC则是对存储资源的需求，PV和PVC的相互关系遵循如图8.1所示的生命周期。![image-20230403162950501](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406282048268.png)

**资源供应**

​	Kubernetes支持两种资源的供应模式：静态模式（Static）和动态模式（Dynamic）。 资源供应的结果就是创建好的PV。

- **静态模式：**
  - 集群管理员手工创建许多PV，在定义PV时需要将后端存储的特性进行设置。

- **动态模式：**
  - 集群管理员无须手工创建PV，而是通过StorageClass的设置对后端存储进行描述，标记为某种类型。此时要求PVC对存储的类型进行声明，系统将自动完成PV的创建及与PVC的绑定。PVC可以声明Class为""，说明该PVC禁止使用动态模式。

**资源绑定**

​		在用户定义好PVC之后，系统将根据PVC对存储资源的请求（存储空间和访问模式）在已存在的PV中选择一个满足PVC要求的PV，一旦找到，就将该PV与用户定义的PVC进行绑定， 用户的应用就可以使用这个PVC了。如果在系统中没有满足PVC要求的PV，PVC则会无限期处于Pending状态，直到等到系统管理员创建了一个符合其要求的PV。PV一旦绑定到某个PVC上，就会被这个PVC独占，不能再与其他PVC进行绑定了。在这种情况下，当PVC申请的存储空间比PV的少时，整个PV的空间就都能够为PVC所用，可能会造成资源的浪费。如果资源供应使用的是动态模式，则系统在为PVC找到合适的StorageClass后，将自动创建一个PV并完成与PVC的绑定。



**资源使用**

​		Pod使用Volume的定义，将PVC挂载到容器内的某个路径进行使用。Volume的类型为persistentVolumeClaim，在容器应用挂载了一个PVC后， 就能被持续独占使用。不过，多个Pod可以挂载同一个PVC，应用程序需要考虑多个实例共同访问一块存储空间的问题。



**资源释放**

​		 当用户对存储资源使用完毕后，用户可以删除PVC，与该PVC绑定的PV将会被标记为 “已释放”，但还不能立刻与其他PVC进行绑定。通过之前PVC写入的数据可能还被留在存储设备上，只有在清除之后该PV才能再次使用



**资源回收**

​			对于PV，管理员可以设定回收策略，用于设置与之绑定的PVC释放资源之后如何处理 遗留数据的问题。只有PV的存储空间完成回收，才能供新的PVC绑定和使用。



**图8.2描述了在静态资源供应模式下，通过PV和PVC完成绑定，并供Pod使用的存储管理机制。**![image-20230403163719564](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406282048085.png)

**图8.3描述了在动态资源供应模式下，通过StorageClass和PVC完成资源动态绑定（系 统自动生成PV），并供Pod使用的存储管理机制**。![image-20230403163826839](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406282048823.png)

#### 4.4 PVC创建和挂载失败的原因

- **PVC一直Pending的原因**：

  - PVC的空间申请大小大于PV的大小；

  - PVC的StorageClassName没有和PV的一致；

  - PVC的accessModes和PV的不一致。


- **挂载PVC的Pod一直处于Pending**：

  - PVC没有创建成功/PVC不存在；

  - PVC和Pod不在同一个Namespace








