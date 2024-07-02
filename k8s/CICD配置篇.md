[TOC]

### 一、Pipeline介绍	

官网：[管道语法 (jenkins.io)](https://www.jenkins.io/doc/book/pipeline/syntax/)

​	Pipeline 是 Jenkins 中的一套自动化流程框架，代表一系列工作流或活动流，作用是允许将单个Jenkins节点或者多个节点的任务组合连接起来，从而实现单个任务难以完成的复杂构建工作。它有以下优点：

- **Code(代码):** Pipeline 的任务是通过代码来实现的，可以通过git来进行版本化控制，团队成员可以编辑迭代Pipeline 代码
- **Durable(持久化):** 无论 Jenkins master 是在计划内或者非计划内重启，pipeline 任务都不会收到影响
- **Pausable(可暂定性)：** pipeline基于groovy可以实现job的暂停和等待用户的输入或批准然后继续执行。
- **Versatile(多功能) ：**支持fork/join、循环执行，并行执行任务
- **Extensible(可扩展性) ：**支持其DSL的自定义扩展 ，以及与其他插件集成的多个选项

#### 1. Pipeline语法

​		Pipeline 包括`声明式语法`和`脚本式语法`

**声明式流水线**

```bash
pipeline {
    agent any // 
    stages {
        stage('Build') { 
            steps {
                // 
            }
        }
        stage('Test') { 
            steps {
                // 
            }
        }
        stage('Deploy') { 
            steps {
                // 
            }
        }
    }
}

```

- agent any: 任意可用的agent都可以执行
- stages：代表整个流水线的所有执行阶段。通常stages只有1个，里面包含多个stage
- stage：代表流水线中的某个阶段，可能出现n个。一般分为拉取代码，编译构建，部署等阶段。
- steps：代表一个阶段内需要执行的逻辑。steps里面是shell脚本，git拉取代码，ssh远程发布等任意内容。

**脚本式流水线**

```bash
node {  
    stage('Build') { 
        // 
    }
    stage('Test') { 
        // 
    }
    stage('Deploy') { 
        // 
    }
}
```

- node：节点，一个 Node 就是一个 Jenkins 节点，Master 或者 Agent，是执行 Step 的具体运行环境

- Stage：阶段，一个 Pipeline 可以划分为若干个 Stage，每个 Stage 代表一组操作，比如：Build、Test、Deploy，Stage 是一个逻辑分组的概念。

### 二、Jenkins 凭证 Credentials

#### 1. 简介：

Harbor 的账号密码、GitLab 的私钥、Kubernetes 的证书均使用 Jenkins 的 Credentials 管理。

#### 2. 配置 Kubernetes 证书

​				首先需要找到集群中的 KUBECONFIG，一般是 kubectl 节点的~/.kube/config 文件，或者是 KUBECONFIG 环境变量所指向的文件。 接下来只需要把证书文件放置于 Jenkins 的 Credentials 中即可。首先点击 Manage Jenkins， 之后点击 Manage Credentials：

```bash
#>>> 查看kube-confing文件
[root@k8s-master01 ~]# ls .kube/
cache  config
# 将该文件上传至桌面
```

![image-20230626102555168](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011609036.png)

**打开jenkins找到Manage Jenkins**

![image-20230626102757931](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011610705.png)



**然后找到 Manage Credentials**![image-20230626102848550](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011610192.png)

![image-20230626103023896](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011610362.png)

![image-20240701215931386](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407012159481.png)

> ==File：KUBECONFIG 文件或其它加密文件==
>  ==ID：该凭证的 ID；不同的集群，ID不同。== 
>  ==Description：证书的描述。==

#### 3. Jenkins配置Harbor仓库的账号密码

​		对于账号密码和 Key 类型的凭证，配置操作步骤是一样，只是选择的凭证类型不同。

**继续添加新的凭证**![image-20230626103743282](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011610453.png)

<img src="https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011610002.png" alt="image-20230626104218944" style="zoom: 80%;" />

> ==Username：Harbor 或者其它平台的用户名；== 
> ==Password：Harbor 或者其它平台的密码；== 
> ==ID：该凭证的 ID；== 
> ==Description：证书的描述。==

#### 4. Jenkins配置Jenkins的私钥

​		自己主机生成的密钥对，公钥保存在Gitlab中，则私钥保存在Jenkins中，免密验证。

**继续添加Gitlab的私钥**![image-20230626104557360](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011611459.png)

![image-20230626104956633](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011611273.png)

![image-20230626105023983](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011611193.png)

> ==Username：用户名，无强制性== 
> ==Private Key：Jenkins 服务器的私钥，一般位于~/.ssh/id_rsa。==



#### 5. 配置Agent

​		通常情况下，Jenkins Slave 会通过 Jenkins Master 节点的 50000 端口与之通信，所以需要开启 Agent 的 50000 端口。

![点击Manage Jenkins](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011611753.png)

![image-20231013094518087](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011615940.png)

![image-20231013094632624](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011615929.png)



> ![image-20231013105513319](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011615777.png)
>
> ==新版jenkins如果不选择此选项可能后续无法创建流水线，会报一下错误==
>
> ![image-20231013105633868](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011616417.png)

##### 5.1 配置指定节点当作Slave节点

​		实际使用时，没有必要把整个 Kubernetes 集群的节点都充当创建 Jenkins Slave Pod 的节点， 可以选择任意的一个或多个节点作为创建 Slave Pod 的节点。==而且该节点必须要有Docker且正在运行==

```bash
	#>>> 将k8s-node01作为Slave Pod 的节点
[root@k8s-master01 .ssh]# kubectl label node k8s-node01 build=true
node/k8s-node01 labeled

[root@k8s-master01 .ssh]# kubectl get node k8s-node01 --show-labels
NAME         STATUS   ROLES    AGE     VERSION    LABELS
k8s-node01   Ready    <none>   4d21h   v1.23.17   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,build=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node01,kubernetes.io/os=linux
```



#### 6. Jenkins配置Kubernetes集群

​		Jenkins会连接到k8s集群，控制它创建一个Slave Pod

![image-20231013100345494](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011616198.png)

![image-20231013100413089](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011616946.png)

![image-20231013100522065](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011616831.png)

![image-20231013100638873](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011616670.png)

![image-20231013100754039](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011616600.png)

![image-20231013100815365](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011616136.png)

![image-20231016100534656](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011616815.png)

![image-20231013100847864](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011616543.png)

> ==最后点击Save保存即可，无需其他配置，必须测试连接成功，且显示k8s版本信息，添加完 Kubernetes 后，在 Jenkinsfile 的 Agent 中，就可以选择该集群 作为创建 Slave 的集群。如果想要添加多个集群，重复上述的步骤即可。首先添加 Kubernetes 凭证，然后添加 Cloud 即可。==



##### 6.1.查看构建日志



![image-20231017095010765](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011616750.png)

![image-20231017095028330](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011617981.png)

![image-20231017095039171](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011617791.png)
