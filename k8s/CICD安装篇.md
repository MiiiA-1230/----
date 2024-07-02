#                                        DevOp安装篇

[TOC]

# jenkins部署安装使用

## 1. jenkins安装

```bash
#>>> 创建数据存放目录并且添加权限
[root@k8s-master01 /]# mkdir /data/jenkins_data -p
[root@k8s-master01 /]# chmod -R 777 /data/jenkins_data

#>>> 拉去jenkins镜像并启动容器（宿主机应提前安装docker）
$ docker run -d --name=jenkins --restart=always -e JENKINS_PASSWORD=admin123 -e JENKINS_USERNAME=admin -e JENKINS_HTTP_PORT_NUMBER=8080 -p 8080:8080 -p 50000:50000 -v /data/jenkins_data:/bitnami/jenkins registry.cn-hangzhou.aliyuncs.com/hujiaming/jenkins:2.414.2-debian-11-r16


#>>> 查看jenkins启动日志
[root@k8s-master01 ~]# docker logs -f jenkins
2023-02-23 02:32:55.463+0000 [id=23]	INFO	hudson.lifecycle.Lifecycle#onReady: Jenkins is fully up and running
执行结果：#显示成功启动
```

## ==2. 命令解析== ：

```bash
#				--name  容器名称
#				--restart=always  重启策略
#				 -e JENKINS_PASSWORD=admin123 -e JENKINS_USERNAME=admin  指定jenkins登录的账号密码**

#				 -e JENKINS_HTTP_PORT_NUMBER=8080 -p 8080:8080 -p 50000:50000   容器对外暴露的端口其中 8080 端口为 Jenkins Web 界面的端口，50000 是 jnlp 使用的端口，后期 Jenkins Slave 需要使用 50000 端口和 Jenkins 主节点通信。

#			     -v /data/jenkins_data:/bitnami/jenkins  将宿主目录挂载到容器中防止重启后数据丢失，持久化数据。

#				 bitnami/jenkins:2.414.2-debian-11-r16  容器镜像的版本，默认从官方镜像拉		
```

## 3. 访问Jenkins

​			通过宿主机端口+8080端口访问，用户为admin；密码为admin123

![ded03fffa96a7b373fc7a0fc1f98514](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011543783.png)

## 4. Jenkins插件安装

​			==如果公司有jenkins，下载之前一定要备份==

<img src="https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011543859.png" alt="插件安装" style="zoom:200%;" />

![插件安装1](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011543319.png)

![插件安装2](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011543993.png)

![更换国内的插件安装源](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011543828.png)

==地址为==：https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json。更换完国内源要Check now 重新加载

![选择安装插件](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011543927.png)

==插件名称：搜索时没有显示插件，有可能系统默认已经安装==

```bash
Git
Git Parameter
Git Pipeline for Blue Ocean
GitLab
Credentials				# 凭证管理
Credentials Binding
Blue Ocean
Blue Ocean Pipeline Editor
Blue Ocean Core JS
Pipeline SCM API for Blue Ocean
Dashboard for Blue Ocean
Build With Parameters
Dynamic Extended Choice Parameter 
Extended Choice Parameter
List Git Branches Parameter
Pipeline
Pipeline: Declarative
Kubernetes
Kubernetes CLI
Kubernetes Credentials
Image Tag Parameter 
Active Choices 
```

> 下载并重启jenkins![image-20230625113720496](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011543954.png) 
>
> 🚒：==插件安装完成之后重启Jenkins，下载的插件重启后生效==

# Gitlab部署安装使用

​		 GitLab 在企业内经常用于代码的版本控制，也是 DevOps 平台中尤为重要的一个工具， GitLab 国内源下载 GitLab 的安装包：https://mirrors.tuna.tsinghua.edu.cn/。

## 1.gitlab安装

```bash
#>>> 安装gitb的rpm包
[root@k8s-master02 ~]# yum install -y gitlab-ce-15.7.3-ce.0.el7.x86_64.rpm
执行结果：
It looks like GitLab has not been configured yet; skipping the upgrade script.

       *.                  *.
      ***                 ***
     *****               *****
    .******             *******
    ********            ********
   ,,,,,,,,,***********,,,,,,,,,
  ,,,,,,,,,,,*********,,,,,,,,,,,
  .,,,,,,,,,,,*******,,,,,,,,,,,,
      ,,,,,,,,,*****,,,,,,,,,.
         ,,,,,,,****,,,,,,
            .,,,***,,,,
                ,*,.
  


     _______ __  __          __
    / ____(_) /_/ /   ____ _/ /_
   / / __/ / __/ /   / __ `/ __ \
  / /_/ / / /_/ /___/ /_/ / /_/ /
  \____/_/\__/_____/\__,_/_.___/
  

Thank you for installing GitLab!
GitLab was unable to detect a valid hostname for your instance.
Please configure a URL for your GitLab instance by setting `external_url`
configuration in /etc/gitlab/gitlab.rb file.
Then, you can start your GitLab instance by running the following command:
  sudo gitlab-ctl reconfigure

For a comprehensive list of configuration options please see the Omnibus GitLab readme
https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/README.md

Help us improve the installation experience, let us know how we did with a 1 minute survey:
https://gitlab.fra1.qualtrics.com/jfe/form/SV_6kVqZANThUQ1bZb?installation=omnibus&release=15-7
```

## 2.修改gitlab配置文件内容

![gitlab配置文件内容修改1](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011541632.png)

关闭node_exporter![image-20230625135020599](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011541880.png)

关闭prometheus![image-20230625135234809](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011541576.png)

==注意==：可以使用域名，也可以使用IP地址

```bash
#>>> 重新加载配置文件，启动服务
[root@k8s-master02 ~]# gitlab-ctl reconfigure
执行结果：
Running handlers:
[2023-02-23T13:17:50+08:00] INFO: Running report handlers
Running handlers complete
[2023-02-23T13:17:50+08:00] INFO: Report handlers complete
Infra Phase complete, 100/972 resources updated in 52 seconds
gitlab Reconfigured!

#>>> 重启gitlab
[root@k8s-master02 ~]# gitlab-ctl restart

```

>  ==登陆密码==： cat /etc/gitlab/initial_root_password       ==用户名==：root    W0SIEWzRwVjjHDzyEBR08+7Wz0Dify974Li7g2fWojw=

## 3.Gitlab项目创建及使用

### 			3.1 组创建

![image-20230625150822841](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011541650.png)

<img src="png/image-20230625150913095.png" alt="image-20230625150913095" style="zoom:%;" />

![image-20230625151044055](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011541874.png)

![image-20230625151334368](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011543987.png)

### 			3.2 创建项目

![image-20230625151436345](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011541441.png)

![image-20230625151555594](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011542755.png)

![image-20230625151933032](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011541363.png)

### 3.3 生成密钥对，并上传公钥至gitlab

```bash
#>>> 客户端生工公钥（如果已经生成过，直接使用；无需重新生成）
[root@k8s-master02 .ssh]# ssh-keygen -t rsa

#>>> 查看公钥，并上传只gitlab
[root@k8s-master02 .ssh]# cat id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCyussDXAGfU2Pgk9tqRZJHGPfyhM8IwJ6pyTfRi7CfsMfv9BXxfqIBAI8kpRFfMGHhhOPIwFQWYVMwBrIQTSsy3TBVG9+avcj0LbRagZ+ZT7Tz0LnllbohSuktpvcSQuWE4q9r2BXMdsIVVcW3jwPj99XBYHv4iEgHudQ8peJqNqd36SmfA9rd7qcVS5zMyacl0cGiVuCBGWrhzTXpUQTU31fui69ew6SnnuV0JogFHGq8ySikLC64bpdGoajwMj2Hfp04sM/JzpM5LRbHaeaAVjM0FeMWQbJg/vmkqXi3YpVNe6RT6e+K96W0IL1jlm5Oyj/hCHmtYOyHlN54cZQ1 root@k8s-master02
```

==上传公钥==![image-20230625152333035](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011541741.png)

![image-20230625152548917](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011541258.png)

<img src="https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011541264.png" alt="image-20230625152653057" style="zoom:100%;" />

==上传完成==

### 			3.3 测试克隆仓库

![克隆仓库](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011541010.png)

```bash
#>>> 创建克隆目录
[root@k8s-master02 /]# mkdir test

#>>> 克隆gitlab仓库
[root@k8s-master02 test]# git clone git@192.168.100.12:kubernetes-test/test.git
正克隆到 'test'...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
接收对象中: 100% (3/3), done.

#>>> 查看克隆的代码
[root@gitlab study]# ls
test-project
[root@gitlab study]# cd test-project/
[root@gitlab test-project]# ls
README.md

#>>> 创建代码并提交仓库
root@gitlab test-project]# echo "# Frist Commit For DevOps" > first.md
[root@gitlab test-project]# git add .
[root@gitlab test-project]# git config --global user.email "2364736365@qq.com"
[root@gitlab test-project]# git commit -am "first commit"
[main ae6d9c8] first commit
 2 files changed, 2 insertions(+), 92 deletions(-)
 rewrite README.md (100%)
 create mode 100644 first.md
 
 #>>> 推送代码
[root@gitlab test-project]# git push origin main
Counting objects: 6, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (4/4), 317 bytes | 0 bytes/s, done.
Total 4 (delta 0), reused 0 (delta 0)
To git@192.168.100.40:kubernetes/test-project.git
   0937c67..ae6d9c8  main -> main
```

**推送成功**![image-20230625154152777](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011541445.png)

# Harbor镜像仓库安装

## 1. Harbor安装配置

```bash
#>>> 上传Harbor包
[root@k8s-node01 ~]# ls
harbor-offline-installer-v2.7.1.tgz  
#>>> 解压安装包
[root@k8s-node01 ~]# tar xf harbor-offline-installer-v2.7.1.tgz 
[root@k8s-node01 ~]# cd harbor          
[root@k8s-node01 harbor]# docker load -i harbor.v2.7.1.tar.gz 
#>>> 安装docker-Compose
[root@k8s-node01 harbor]# yum install -y docker-compose
#>>> 修改harbor配置文件的模板文件
[root@k8s-node01 harbor]# cp harbor.yml.tmpl harbor.yml
[root@k8s-node01 harbor]# vim harbor.yml
```

**由于没有https协议，需要将地址修改为IP地址+端口，并禁止443端口和加密证书**![image-20230625161703698](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011541004.png)

**修改Harbor的数据挂载目录**                                                                                                ![image-20230625161821962](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202407011541832.png)

```bash
#>>> 创建数据目录
[root@k8s-node01 harbor]#  mkdir /data/harbor /var/log/harbor -p
[root@k8s-node01 harbor]# chmod 777 /data/harbor/
[root@k8s-node01 harbor]# chmod 777 /var/log/harbor/

#>>> 语法检测
[root@k8s-node01 harbor]# ./prepare
prepare base dir is set to /root/harbor
WARNING:root:WARNING: HTTP protocol is insecure. Harbor will deprecate http protocol in the future. Please make sure to upgrade to https
Generated configuration file: /config/portal/nginx.conf
Generated configuration file: /config/log/logrotate.conf
Generated configuration file: /config/log/rsyslog_docker.conf
Generated configuration file: /config/nginx/nginx.conf
Generated configuration file: /config/core/env
Generated configuration file: /config/core/app.conf
Generated configuration file: /config/registry/config.yml
Generated configuration file: /config/registryctl/env
Generated configuration file: /config/registryctl/config.yml
Generated configuration file: /config/db/env
Generated configuration file: /config/jobservice/env
Generated configuration file: /config/jobservice/config.yml
Generated and saved secret to file: /data/secret/keys/secretkey
Successfully called func: create_root_cert
Generated configuration file: /compose_location/docker-compose.yml
Clean up the input dir

#>>> 安装Harbor
[root@k8s-node01 harbor]# ./install.sh
? ----Harbor has been installed and started successfully.----
```

> 语法检测，会出现警告，表示没有配置HTTPS协议的原因，可以忽略不管
>
> 默认用户名：==admin==		密码：==Harbor12345==

**如果配置不是https协议，所有的Kubernetes节点的Docker（如果是containerd 作为Runtime， 可以参考下文配置 insecure-registry）都需要添加 insecure-registries 配置：**

```bash
#>>> 修改docker的配置文件
[root@k8s-node03 ~]# vim /etc/docker/daemon.json 
{
  "insecure-registries": ["192.168.100.30"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}

#>>> 重启docker
[root@k8s-master01 ~]#  systemctl daemon-reload
[root@k8s-master01 ~]# systemctl restart docker
[root@k8s-master01 ~]# systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since 一 2023-06-26 09:48:45 CST; 17s ago
     Docs: https://docs.docker.com
 Main PID: 61207 (dockerd)
```

> ==**如果harbor没有成功启动，在harbor的目录下执行 $ docker-compose restert 重启Harbor**==

**如果 Kubernetes 集群采用的是 Containerd 作为的 Runtime，配置 insecure-registry 只需要在 Containerd 配置文  件的 mirrors 下添加自己的镜像仓库地址即可：**

```bash
[root@k8s-node01 harbor]# vim /etc/containerd/config.toml
```

## 2. Harbor仓库测试

```bash
#>>> 登录Harbor仓库
[root@k8s-master01 ~]# docker login 192.168.100.30
Username: admin
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded

#>>> 推送镜像
[root@k8s-master01 ~]# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6  192.168.100.30/kubernetes/pause:3.6 
[root@k8s-master01 ~]# docker push 192.168.100.30/kubernetes/pause:3.6 
The push refers to repository [192.168.100.30/kubernetes/pause]
1021ef88c797: Pushed 
3.6: digest: sha256:74bf6fc6be13c4ec53a86a5acf9fdbc6787b176db0693659ad6ac89f115e182c size: 526
```
