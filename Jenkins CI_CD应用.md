# Jenkins构建CI/CD

**什么是CI/CD**：持续集成/持续发布---开发(git) -->git主库-->jenkins(git+jdk+tomcat+maven打包+测试）-->发布到tomcat服务器。

Jenkins是一个功能强大的应用程序，允许**持续集成和持续交付项目**，无论用的是什么平台。这是一个免费的源代码，可以处理任何类型的构建或持续集成。集成Jenkins可以用于一些测试和部署技术。

为什么要 CI / CD 方法

简介

软件开发的连续方法基于自动执行脚本，以最大限度地减少在开发应用程序时引入错误的可能性。从新代码的开发到部署，它们需要较少的人为干预甚至根本不需要干预。

它涉及在每次小迭代中不断构建，测试和部署代码更改，从而减少基于有缺陷或失败的先前版本开发新代码的机会。

这种方法有三种主要方法，每种方法都根据最适合您的策略进行应用。

**持续集成**(Continuous Integration, CI):  代码合并，构建，部署，测试都在一起，不断地执行这个过程，并对结果反馈。

**持续部署**(Continuous Deployment, CD):　部署到测试环境、预生产环境/灰度环境、生产环境。

**持续交付**(Continuous Delivery, CD):  将最终产品发布到生产环境、给用户使用。

### 一、jenkins介绍

Jenkins是帮我们将代码进行统一的编译打包、还可以放到tomcat容器中进行发布。
我们通过配置，将以前：编译、打包、上传、部署到Tomcat中的过程交给Jenkins，Jenkins通过给定的代码地址URL（代码仓库地址），将代码拉取到其“宿主服务器”（Jenkins的安装位置），进行编译、打包和发布到Tomcat容器中。

##### 1、Jenkins概述

```
是一个开源的、提供友好操作界面的持续集成(CI)工具，主要用于持续、自动的构建的一些定时执行的任务。Jenkins用Java语言编写，可在Tomcat等流行的容器中运行，也可独立运行。
```

jenkins通常与版本管理工具(SCM)、构建工具结合使用；常用的版本控制工具有SVN、GIT。jenkins构建工具有Maven、Ant、Gradle。

##### 2、Jenkins目标

① 持续、自动地构建/测试软件项目。

② 监控软件开发流程，快速问题定位及处理，提高开发效率。

##### 3、Jenkins特性

```shell
① 开源的java语言开发持续集成工具，支持CI，CD。
② 易于安装部署配置：可通过yum安装,或下载war包以及通过docker容器等快速实现安装部署，可方便web界面配置管理。
③ 消息通知及测试报告：集成RSS/E-mail通过RSS发布构建结果或当构建完成时通过e-mail通知，生成JUnit/TestNG测试报告。
④ 分布式构建：支持Jenkins能够让多台计算机一起构建/测试。
⑤ 文件识别:Jenkins能够跟踪哪次构建生成哪些jar，哪次构建使用哪个版本的jar等。
⑥ 丰富的插件支持:支持扩展插件，你可以开发适合自己团队使用的工具，如git，svn，maven，docker等。
```

工作流程图:

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/1569246908031.png#id=pHA8p&originHeight=338&originWidth=1149&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

```shell
测试环境中：
1.开发者会将代码上传到版本库中。
2.jenkins通过配置版本库的连接地址，获取到源代码。
3.jenkins获取到源代码之后通过参数化构建(或者触发器)开始编译打包。
4.jenkins通过调用maven（Ant或者Gradle）命令实现编译打包过程。
5.生成的war包通过ssh插件上传到远程tomcat服务器中通过shell脚本自动发布项目。

生产环境：
测试环境将项目测试没问题后，将项目推送到线上正式环境。
1.可以选择手动。
2.也可以通过调用脚本推送过去。
```

##### 4、产品发布流程

产品设计成型 -> 开发人员开发代码 -> 测试人员测试功能 -> 运维人员发布上线

持续集成（Continuous integration，简称CI）

持续部署（continuous deployment）

持续交付（Continuous delivery）



### 二、部署应用Jenkins+Gitlub+Tomcat实战

准备环境:

两台机器

git-server    ----[https://github.com/bingyue/easy-springmvc-maven](https://github.com/bingyue/easy-springmvc-maven)

jenkins-server    ---192.168.246.212---最好是3个G以上

java-server   -----192.168.246.210

[https://github.com/bingyue/easy-springmvc-maven](https://github.com/bingyue/easy-springmvc-maven)

### Jenkins2.303.1版本安装

##### 1.下载安装包

百度搜索openjdk11、tomcat、maven、jenkins
openjdk11经过测试之后，不能运行当前版本jenkins。我这里换成了jdk1.8的
jdk官网：[https://www.oracle.com/java/technologies/downloads/](https://www.oracle.com/java/technologies/downloads/)
jenkins官网：[https://www.jenkins.io/](https://www.jenkins.io/)
tomcat官网：[https://tomcat.apache.org/](https://tomcat.apache.org/)
maven官网：[https://maven.apache.org/](https://maven.apache.org/)

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20210911141241212.png#id=eVKKb&originHeight=775&originWidth=1152&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652448511823-c97d9802-db51-449b-a7d6-60578acd622c.png#averageHue=%23e3e2e0&clientId=udc411f9d-3d04-4&from=paste&height=492&id=u1c995890&originHeight=615&originWidth=1346&originalType=binary&ratio=1&rotation=0&showTitle=false&size=84149&status=done&style=none&taskId=u70efca8c-6d7d-4efb-9abc-27fe905da9e&title=&width=1076.8)

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20210911141350166.png#id=scjss&originHeight=852&originWidth=1567&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20210911141450852.png#id=YaI3S&originHeight=893&originWidth=1808&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20210911141600694.png#id=Tn3KG&originHeight=988&originWidth=1294&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

```shell
[root@jenkins ~]# wget https://get.jenkins.io/war/2.303/jenkins.war
[root@jenkins ~]# wget https://downloads.apache.org/maven/maven-3/3.8.2/binaries/apache-maven-3.8.2-bin.tar.gz
[root@jenkins ~]# wget https://dlcdn.apache.org/tomcat/tomcat-8/v8.5.70/bin/apache-tomcat-8.5.70.tar.gz
还有openjdk11
[root@jenkins ~]# cd /usr/local
[root@jenkins local]# tar -xvzf apache-maven-3.8.2-bin.tar.gz
[root@jenkins local]# tar -xvzf apache-tomcat-8.5.70.tar.gz
[root@jenkins local]# tar -xvzf openjdk-11+28_linux-x64_bin.tar.gz
[root@jenkins local]# mv jdk-11/ java
[root@jenkins local]# mv apache-tomcat-8.5.70 tomcat
[root@jenkins local]# rm -rf tomcat/webapps/*
[root@jenkins local]# mv apache-maven-3.8.2 java/maven
[root@jenkins ~]# cp jenkins.war  /usr/local/tomcat/webapps/
```

##### 2.配置环境变量

```shell
[root@jenkins ~]# yum install fontconfig

[root@jenkins ~]# vim /etc/profile
JAVA_HOME=/usr/local/java
MAVEN_HOME=/usr/local/maven
PATH=$PATH:$JAVA_HOME/bin:$MAVEN_HOME/bin
export PATH USER LOGNAME MAIL HOSTNAME HISTSIZE HISTCONTROL JAVA_HOME MAVEN_HOME

[root@jenkins ~]# source /etc/profile

[root@jenkins ~]# java -version
openjdk version "11.0.12" 2021-07-20 LTS
OpenJDK Runtime Environment 18.9 (build 11.0.12+7-LTS)
OpenJDK 64-Bit Server VM 18.9 (build 11.0.12+7-LTS, mixed mode, sharing)

[root@jenkins ~]# mvn -v
Apache Maven 3.8.2 (ea98e05a04480131370aa0c110b8c54cf726c06f)
Maven home: /usr/local/java/maven
Java version: 11, vendor: Oracle Corporation, runtime: /usr/local/java
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "3.10.0-693.el7.x86_64", arch: "amd64", family: "unix"

[root@jenkins ~]# /usr/local/tomcat/bin/startup.sh
```

补充：如果启动访问报错

请更换jdk版本为1.8的，修改环境变量配置，重新启动即可；

##### 3.访问登录

![image-20240605210031702](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052100274.png)

> Username:admin
>
> password：cat ~/.jenkins/secrets/initialAdminPassword 
> 20c6da4f5cce48cf8c1e0834ee90292d



##### 4.插件安装

```bash
安装插件:
所需的插件:
• Maven插件 Maven Integration plugin
• 发布插件 Deploy to container Plugin
需要安装插件如下：
安装插件
`Deploy to container    ---支持自动化代码部署到tomcat容器`
`GIT plugin  可能已经安装,可在已安装列表中查询出来`
`Maven Integration   jenkins利用Maven编译，打包所需插件`
`Publish Over SSH  通过ssh连接`
`ssh  插件`
安装过程:
系统管理--->插件管理---->可选插件--->过滤Deploy to container---->勾选--->直接安装
```

![image-20240605210257223](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052102314.png)



![image-20240605210317727](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052103917.png)



![image-20240605210337580](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052103747.png)

![image-20240605210417894](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052104990.png)

> [https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json](https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json)

![image-20240605210525461](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052105628.png)



![image-20240605210557961](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052105141.png)





##### 5.配置国内源

因为Jenkins下载，默认是国外地址，如果插件下载失败，我们就替换为国内地址;
官方下载插件慢 更新下载地址;
Jenkins 安装时会默认从updates.jenkins-ci.org 拉取，我们需要换成国内源——清华大学开源软件镜像站;
[https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json](https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json)
cd    {你的Jenkins工作目录}/updates  进入更新配置位置;

```shell
[root@jenkins-server1 updates]# pwd
/root/.jenkins/updates    #这是Jenkins默认的工作目录
[root@localhost updates]# vim  default.json      #修改配置文件
s/https:\/\/updates.jenkins.io\/download/http:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' /root/.jenkins/updates/default.json            #官方源替换清华源
s/http:\/\/www.google.com/https:\/\/www.baidu.com/g    #google替换成百度

或者直接进行一下操作（一步到位，不需要多步操作）
[root@localhost ~]# sed -i 's/https:\/\/updates.jenkins.io\/download/http:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' /root/.jenkins/updates/default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' /root/.jenkins/updates/default.json
```

之后，在网站后面加上restart进行jenkins重启。
[http://192.168.153.147:8080/jenkins/restart](http://192.168.153.147:8080/jenkins/restart)

##### 5.邮箱配置(可选)

安装邮件插件，才能确保邮件发送成功。否则可能不会发送邮件



![image-20240605211735834](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052117930.png)



![image-20240605211806552](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052118645.png)

![image-20240605211939322](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052119412.png)

![image-20240605212833063](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052128190.png)

![image-20240605212526477](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052125567.png)

![image-20240605213106207](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052131280.png)

![image-20240605213200921](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052132032.png)

![image-20240605214417819](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052144905.png)

![image-20240605214458248](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052144361.png)



##### 6.配置Jenkins私钥

​	说明：Jenkins需要去Gitlab中拉取代码，Jenkins服务器的公钥需要配置Gitlab SSH中，Jenkins私钥配置Jenkins服务中，目的是为了实现Jenkins服务加载到本地私钥，实现服务之间密钥认证。

```shell
[root@jenkins ~]# ssh-keygen
[root@jenkins ~]# cat /root/.ssh/id_rsa
```

![image-20240605215247049](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052152131.png)

##### 7.添加后端服务器
```bash
# 公钥发送到后端服务器，才能实现免密；
[root@jenkins ~]# ssh-copy-id -i root@192.168.153.194
```

![image-20240605221943775](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052219851.png)

##### 8.配置JDK和Maven

虽然Jenkins服务器上，已经安装了JDK和maven工具，但是，还需要在Jenkins服务中，进行配置；

这样Jenkins才能自动化的使用两个工具；

![image-20240605223015075](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052230149.png)

![image-20240605223041884](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052230960.png)

![image-20240605223326356](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052233436.png)

JDK配置

![image-20240605223430949](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052234005.png)



maven配置

![image-20240605223711300](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052237362.png)

**如果有多个jdk和maven需要配置的话，可以点击新增jdk或者新增maven**



##### 9.构建发布任务



![image-20240605224145476](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052241569.png)



![image-20240605224326357](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052243447.png)

![image-20240605225328164](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052253276.png)

拉取代码环节

![image-20240605225142703](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052251818.png)

[https://github.com/bingyue/easy-springmvc-maven](https://github.com/bingyue/easy-springmvc-maven)
或者用以下链接
[https://gitee.com/youngfit/easy-springmvc-maven.git](https://gitee.com/youngfit/easy-springmvc-maven.git)

这里是我再github上，找的一个开源项目。能进行编译打包
如果jenkins服务器还未安装git客户端，请先安装

```bash
$ yum install -y git
```



编译打包环节



![image-20240605225610418](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052256521.png)



测试拉取代码、编译打包环节

![image-20240605231009330](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052310474.png)

![image-20240605231049474](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052310568.png)

![image-20240605231132040](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052311123.png)

![image-20240605231442049](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052314164.png)



**持续发布**

![image-20240605231556343](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052315434.png)

![image-20240605231625066](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052316155.png)

![image-20240605233844052](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406052338169.png)

> 如果有多个后端服务器，可以点击 ADD server进行添加

##### 10.java服务器添加脚本

```shell
# 上传jdk
[root@java-server ~]# tar xzf jdk-8u191-linux-x64.tar.gz -C /usr/local/
[root@java-server ~]# cd /usr/local/
[root@java-server local]# mv jdk1.8.0_191/ java

# 下载tomcat
[root@java-server ~]# wget http://mirrors.tuna.tsinghua.edu.cn/apache/tomcat/tomcat-8/v8.5.42/bin/apache-tomcat-8.5.42.tar.gz
[root@java-server ~]# tar xzf apache-tomcat-8.5.42.tar.gz -C /usr/local
[root@java-server ~]# cd /usr/local
[root@java-server local]# mv apache-tomcat-8.5.42/ tomcat

# 设置环境变量
[root@java-server local]# vim /etc/profile
export JAVA_HOME=/usr/local/java
export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib:$JAVA_HOME/jre/lib:$JAVA_HOME/lib/tools.jar
export TOMCAT_HOME=/data/application/tomcat
[root@java-server local]# source /etc/profile

# 测试:
[root@java-server local]# java -version 

# 删除tomcat默认发布目录下面的内容:
[root@java-server local]# rm -rf /usr/local/tomcat/webapps/*
[root@java-server local]# cd /usr/local/tomcat/webapps/
[root@java-server webapps]# ls


# 创建目录和脚本:
[root@java-server ~]# mkdir /opt/script  #创建脚本目录
[root@java-server ~]# vim app-jenkins.sh   #创建脚本
	# 脚本内容在后面：
[root@java-server ~]# chmod +x app-jenkins.sh  #添加执行权限
[root@java-server ~]# mv app-jenkins.sh /opt/script/
	# 脚本内容:
[root@java-server script]# cat app-jenkins.sh 
#!/usr/bin/bash
#本脚本适用于jenkins持续集成，实现备份war包到代码更新上线！使用时请注意全局变量。
#================
#Defining variables
export JAVA_HOME=/usr/local/java
webapp_path="/usr/local/tomcat/webapps"
tomcat_run="/usr/local/tomcat/bin"
updata_path="/data/update/`date +%F-%T`"
backup_path="/data/backup/`date +%F-%T`"
tomcat_pid=`ps -ef | grep tomcat | grep -v grep | awk '{print $2}'`
files_dir="easy-springmvc-maven"
files="easy-springmvc-maven.war"
job_path="/root/upload"

#Preparation environment
echo "Creating related directory"
mkdir -p $updata_path
mkdir -p $backup_path

echo "Move the uploaded war package to the update directory"
mv $job_path/$files $updata_path

echo "========================================================="
cd /opt
echo "Backing up java project"
if [ -f $webapp_path/$files ];then
	tar czf $backup_path/`date +%F-%H`.tar.gz $webapp_path
	if [ $? -ne 0 ];then
		echo "打包失败，自动退出"
		exit 1
	else
		echo "Checking if tomcat is started"
		if [ -n "$tomcat_pid" ];then
			kill -9 $tomcat_pid
			if [ $? -ne 0 ];then
				echo "tomcat关闭失败，将会自动退出"
				exit 2
			fi
		fi
		cd $webapp_path
		rm -rf $files && rm -rf $files_dir
		cp $updata_path/$files $webapp_path
		cd /opt
		$tomcat_run/startup.sh
		sleep 5
		echo "显示tomcat的pid"
		echo "`ps -ef | grep tomcat | grep -v grep | awk '{print $2}'`"
		echo "tomcat startup"
		echo "请手动查看tomcat日志。脚本将会自动退出"
	fi
else
	echo "Checking if tomcat is started"
        if [ -n "$tomcat_pid" ];then
        	kill -9 $tomcat_pid
                if [ $? -ne 0 ];then
                	echo "tomcat关闭失败，将会自动退出"
                       	exit 2
                fi
        fi
	cp $updata_path/$files $webapp_path
	$tomcat_run/startup.sh
        sleep 5
        echo "显示tomcat的pid"
        echo "`ps -ef | grep tomcat | grep -v grep | awk '{print $2}'`"
        echo "tomcat startup"
        echo "请手动查看tomcat日志。脚本将会自动退出"
fi
```



##### 11.启用邮箱

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20210902225124311.png#id=FYeGR&originHeight=348&originWidth=1180&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

##### 12.构建项目

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20210908135616162.png#id=KdAEl&originHeight=690&originWidth=1032&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20210908135410486.png#id=v6wk4&originHeight=843&originWidth=1516&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
##### 13.访问tomcat测试
##### 14.更新测试
随便找台机器，作为开发人员
```shell
# git clone https://gitee.com/youngfit/easy-springmvc-maven.git
[root@tomcat-server ~]# cd easy-springmvc-maven/
[root@tomcat-server easy-springmvc-maven]# vim src/main/webapp/index.jsp
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670918518215-2552f3ca-ca0a-4a47-8b76-009acbafcbd2.png#averageHue=%2319120f&clientId=ue8bb4b21-b411-4&from=paste&height=270&id=ub6d69520&originHeight=337&originWidth=1374&originalType=binary&ratio=1&rotation=0&showTitle=false&size=45475&status=done&style=none&taskId=u1348085e-6537-4a05-82a6-afe703b709b&title=&width=1099.2)
```shell
[root@tomcat-server easy-springmvc-maven]# git add *
[root@tomcat-server easy-springmvc-maven]# git config --global user.email "feigeyoungfit@163.com"
[root@tomcat-server easy-springmvc-maven]# git config --global user.name "feigeyoungfit"
[root@tomcat-server easy-springmvc-maven]# git commit -m "username & password"
[master 5e9f4fd] username & password
 1 file changed, 2 insertions(+), 2 deletions(-)
[root@tomcat-server easy-springmvc-maven]# git push origin master
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670919912187-f18cd605-47a6-4f10-8a12-ca11021e0e23.png#averageHue=%23e1e0e0&clientId=ue8bb4b21-b411-4&from=paste&height=537&id=ue4e4287b&originHeight=671&originWidth=828&originalType=binary&ratio=1&rotation=0&showTitle=false&size=71982&status=done&style=none&taskId=uf0b6f731-f9a5-425e-9a51-271bd54b105&title=&width=662.4)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670919928266-aa1943a1-8b17-4a64-9e53-2a4fa91bbf92.png#averageHue=%23f9f8f6&clientId=ue8bb4b21-b411-4&from=paste&height=138&id=u6b6d26de&originHeight=172&originWidth=867&originalType=binary&ratio=1&rotation=0&showTitle=false&size=22742&status=done&style=none&taskId=u523f16b2-2c0f-4398-8763-4b2d6d13623&title=&width=693.6)

##### 16.准备开源项目
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670919078484-e544e579-a23f-4806-9af8-c0625b6d369c.png#averageHue=%23fdfbfb&clientId=ue8bb4b21-b411-4&from=paste&height=753&id=u4588efc7&originHeight=941&originWidth=1436&originalType=binary&ratio=1&rotation=0&showTitle=false&size=106808&status=done&style=none&taskId=ucb40a564-95ff-4402-ba46-7a0979e49c1&title=&width=1148.8)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670919234756-2fbc5a36-7d89-4fc0-a874-c4fe88dfa40b.png#averageHue=%23e6e6e5&clientId=ue8bb4b21-b411-4&from=paste&height=542&id=u0ccb950b&originHeight=677&originWidth=1431&originalType=binary&ratio=1&rotation=0&showTitle=false&size=73115&status=done&style=none&taskId=u6b8737ae-ae21-403b-9c44-51a226acaab&title=&width=1144.8)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670919255822-92e587de-9e8e-4ed7-b6da-b9867306135c.png#averageHue=%23fbfafa&clientId=ue8bb4b21-b411-4&from=paste&height=392&id=ub4c62a24&originHeight=490&originWidth=1247&originalType=binary&ratio=1&rotation=0&showTitle=false&size=47692&status=done&style=none&taskId=ucc0c67fa-5f9e-4a51-b054-ade525e3d73&title=&width=997.6)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670919270148-4cbef903-4c46-45ed-aa52-5050b9dbb2ec.png#averageHue=%23fbfaf9&clientId=ue8bb4b21-b411-4&from=paste&height=371&id=ua0e36b8b&originHeight=464&originWidth=1427&originalType=binary&ratio=1&rotation=0&showTitle=false&size=59493&status=done&style=none&taskId=u8e5eb6d2-9958-448a-a909-033584fd918&title=&width=1141.6)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670919311801-1ed69934-9bf2-4ac0-bf64-476be3affd3c.png#averageHue=%23fbf7f4&clientId=ue8bb4b21-b411-4&from=paste&height=372&id=ue6281309&originHeight=465&originWidth=1330&originalType=binary&ratio=1&rotation=0&showTitle=false&size=77373&status=done&style=none&taskId=ue0f2857e-bd61-4872-a256-54de1f07375&title=&width=1064)
随便找台机器，作为开发人员的开发环境，需要安装git，以及配置git邮箱、用户名；
```shell
[root@tomcat-server tmp]# git clone https://gitee.com/youngfit/testweb.git
[root@tomcat-server tmp]# cd testweb/
```
将easy-springmvc-maven-master.zip 源码包，上传到开发人员机器
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670919554706-240cfaa3-2ac9-43f1-b76e-c5b493b9684c.png#averageHue=%231a100e&clientId=ue8bb4b21-b411-4&from=paste&height=60&id=u80be9d1b&originHeight=75&originWidth=956&originalType=binary&ratio=1&rotation=0&showTitle=false&size=9004&status=done&style=none&taskId=u09afd9a8-a675-446d-8bc5-537ea9d1b20&title=&width=764.8)
```shell
[root@tomcat-server testweb]# yum -y install unzip
[root@tomcat-server testweb]# unzip easy-springmvc-maven-master.zip
[root@tomcat-server testweb]# cp -r easy-springmvc-maven-master/* .
cp: overwrite ‘./README.md’? y
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670919639327-a296021b-c0c2-4495-bfd4-26b4538475fb.png#averageHue=%23100a09&clientId=ue8bb4b21-b411-4&from=paste&height=66&id=ub370334a&originHeight=82&originWidth=1611&originalType=binary&ratio=1&rotation=0&showTitle=false&size=14170&status=done&style=none&taskId=u6297a9cc-c9cc-472c-9e76-8bdf8f5b8b0&title=&width=1288.8)
因为已经拷贝出来了。删除没用的压缩包、目录、文件
```shell
[root@tomcat-server testweb]# rm -rf easy-springmvc-maven-master easy-springmvc-maven-master.zip 
[root@tomcat-server testweb]# ls
pom.xml  README.en.md  README.md  src
[root@tomcat-server testweb]# rm -rf README.en.md 
[root@tomcat-server testweb]# ls
pom.xml  README.md  src
[root@tomcat-server testweb]# git add *
[root@tomcat-server testweb]# git commit -m "version1"
[root@tomcat-server testweb]# git push origin master
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670919712737-dd65eae8-d3b5-4747-bce8-2b2ad0e6ae05.png#averageHue=%2314110f&clientId=ue8bb4b21-b411-4&from=paste&height=261&id=u362b0106&originHeight=326&originWidth=987&originalType=binary&ratio=1&rotation=0&showTitle=false&size=43757&status=done&style=none&taskId=u0293cfea-497b-4f34-93cd-327b3cdd417&title=&width=789.6)

![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670919730307-6ed78e76-c901-4f1d-9b7a-61473ef21966.png#averageHue=%23fbf9f7&clientId=ue8bb4b21-b411-4&from=paste&height=459&id=u666e6a25&originHeight=574&originWidth=1181&originalType=binary&ratio=1&rotation=0&showTitle=false&size=57690&status=done&style=none&taskId=u3a97539a-8010-41f1-b0be-9b3c680f3bb&title=&width=944.8)
### 三、Jenkins+Gitlab+maven项目实战

Jenkins服务器去拉取代码。所以要下载git客户端

```shell
[root@jenkins-server ~]# yum -y install git
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670983695896-0030f164-eea8-4d3d-9d74-02a5c0a6cf03.png#averageHue=%23faf7f5&clientId=u57f8f598-4783-4&from=paste&height=579&id=u8539a06c&originHeight=724&originWidth=1290&originalType=binary&ratio=1&rotation=0&showTitle=false&size=65994&status=done&style=none&taskId=u861ccf8e-c299-4077-bedb-e61e8f9fb1f&title=&width=1032)



关闭ssh密钥认证

![image-20240606071447681](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406060714828.png)

![img](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406060712798.png)

开始构建maven项目

![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670983799423-7acbd9cb-8482-4943-98aa-3c423de2b0b4.png#averageHue=%23f3f1f0&clientId=u57f8f598-4783-4&from=paste&height=614&id=ua8d746eb&originHeight=818&originWidth=1033&originalType=binary&ratio=1&rotation=0&showTitle=false&size=114400&status=done&style=none&taskId=ufe9f1962-0027-45bc-9f01-e980b1ec1a0&title=&width=775)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670983844940-cc88a9fe-c614-4e1e-bfbb-adba05ff9fc4.png#averageHue=%23f9f8f8&clientId=u57f8f598-4783-4&from=paste&height=546&id=u54931f91&originHeight=682&originWidth=1214&originalType=binary&ratio=1&rotation=0&showTitle=false&size=38617&status=done&style=none&taskId=u18cbc49d-6d41-4ec7-a81a-cba9f8bb9b3&title=&width=971.2)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670984239967-9e0a6b7a-4169-4bdc-93f0-b1639f4382a7.png#averageHue=%23f8f7f6&clientId=u57f8f598-4783-4&from=paste&height=546&id=u91ff0ce5&originHeight=683&originWidth=1202&originalType=binary&ratio=1&rotation=0&showTitle=false&size=69640&status=done&style=none&taskId=uc1be8c8c-1b3e-4962-8e8b-20678993cbc&title=&width=961.6)
![image-20240606071945163](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406060719247.png)




[root@jenkins ~]# cat /root/.ssh/id_rsa

![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670984467041-34a29369-80a9-4899-aa2b-ac9d87da0b2f.png#averageHue=%23faf6f6&clientId=u57f8f598-4783-4&from=paste&height=546&id=u6adabec6&originHeight=683&originWidth=1393&originalType=binary&ratio=1&rotation=0&showTitle=false&size=55101&status=done&style=none&taskId=uc846644c-9248-4ed5-bb22-fec57d76c8f&title=&width=1114.4)



Gitlab配置jenkins公钥

```
[root@jenkins ~]# cat /root/.ssh/id_rsa.pub 
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670984552841-87542d9c-7b8e-49d7-b937-e53f93ce43f8.png#averageHue=%23fbf8f7&clientId=u57f8f598-4783-4&from=paste&height=594&id=u39261f82&originHeight=743&originWidth=1715&originalType=binary&ratio=1&rotation=0&showTitle=false&size=103019&status=done&style=none&taskId=u2cb87802-6f0c-4f61-bc6e-b5366c56b62&title=&width=1372)



编译后设置

![image-20240606072233374](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406060722445.png)

![image-20240606072318936](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406060723001.png)



![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670984942386-4b11447c-e012-4fb8-8cfc-5a6d19cb1eee.png#averageHue=%23e8e7e7&clientId=u57f8f598-4783-4&from=paste&height=653&id=u320a4b93&originHeight=816&originWidth=882&originalType=binary&ratio=1&rotation=0&showTitle=false&size=102805&status=done&style=none&taskId=udddc5045-be94-45ad-9c89-b1a585fdfc3&title=&width=705.6)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670984967094-cb72f86f-bc21-492d-960a-0bac7994bf76.png#averageHue=%23f1f1f0&clientId=u57f8f598-4783-4&from=paste&height=578&id=uf941f65c&originHeight=722&originWidth=1628&originalType=binary&ratio=1&rotation=0&showTitle=false&size=71489&status=done&style=none&taskId=ucb0896d2-21a2-4103-9a09-132151d0fe0&title=&width=1302.4)

访问测试
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670985178071-7cfddb83-2705-49d9-bcd1-dd05f65f0e57.png#averageHue=%23f9f9f8&clientId=u57f8f598-4783-4&from=paste&height=172&id=u46618cc9&originHeight=215&originWidth=814&originalType=binary&ratio=1&rotation=0&showTitle=false&size=22596&status=done&style=none&taskId=ue8e1ffe7-3f00-4baf-a079-851870f3e92&title=&width=651.2)
开发更新代码
```
[root@git-client mnt]# git clone git@192.168.91.168:root/cloudweb.git
[root@git-client mnt]# cd cloudweb/
[root@git-client mnt]# vim src/main/webapp/index.jsp
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670985229589-55325db8-cc65-4369-baf6-91c3e369e7e9.png#averageHue=%23100a09&clientId=u57f8f598-4783-4&from=paste&height=254&id=u37402819&originHeight=317&originWidth=1362&originalType=binary&ratio=1&rotation=0&showTitle=false&size=37929&status=done&style=none&taskId=u47116c1a-9c12-4b57-ba0f-81ce0e09d87&title=&width=1089.6)
```
[root@git-client mnt]# git add *
[root@git-client mnt]# git commit -m "用户名称 & 用户密码"
[master 31f2d4b] 用户名称 & 用户密码
 1 file changed, 2 insertions(+), 2 deletions(-)
[root@git-client mnt]# git push origin master
Counting objects: 11, done.
Compressing objects: 100% (5/5), done.
Writing objects: 100% (6/6), 559 bytes | 0 bytes/s, done.
Total 6 (delta 2), reused 0 (delta 0)
To git@192.168.91.168:root/cloudweb.git
   0121086..31f2d4b  master -> master

```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670985314800-5574461a-e76d-4629-81a1-75100608b81e.png#averageHue=%23fcfafa&clientId=u57f8f598-4783-4&from=paste&height=578&id=ue3f374de&originHeight=723&originWidth=1223&originalType=binary&ratio=1&rotation=0&showTitle=false&size=70757&status=done&style=none&taskId=u1768b10d-a3a4-4b55-935b-de8fdc39f82&title=&width=978.4)

![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670985343699-8f494317-ad9e-4d4d-bcb5-8e0253d356b3.png#averageHue=%23f3f0f0&clientId=u57f8f598-4783-4&from=paste&height=493&id=u9162e8e9&originHeight=616&originWidth=797&originalType=binary&ratio=1&rotation=0&showTitle=false&size=95711&status=done&style=none&taskId=u70cee4e9-eda0-4d54-b196-5432c0ecade&title=&width=637.6)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670985420814-7e21501c-6744-4092-9861-af5132a64863.png#averageHue=%23f9f9f8&clientId=u57f8f598-4783-4&from=paste&height=158&id=u2f2e0514&originHeight=198&originWidth=807&originalType=binary&ratio=1&rotation=0&showTitle=false&size=21427&status=done&style=none&taskId=u1cd412f5-16ae-4759-9e92-fa11820cdad&title=&width=645.6)



### 四、Jenkins git参数化构建

git参数化构建：开发人员推送代码之前，对此版本的代码，打一个标签(tag)。我们可以认作为是此套代码的版本号。后续可以方便我们进行版本之间的切换。尤其是刚上线一套代码有问题，可以运用jenkins立即进行版本回退/切换；

Gitlab仓库代码准备：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1654067572140-a2b87e7c-1721-430e-b0d4-664bb7bbe168.png#averageHue=%23f8f6f5&clientId=u6057b347-398d-4&from=paste&height=566&id=ubade1162&originHeight=708&originWidth=1399&originalType=binary&ratio=1&rotation=0&showTitle=false&size=82385&status=done&style=none&taskId=u0fba8d82-3adc-41d3-83fb-c8bb62a7fde&title=&width=1119.2)
首先，需要安装插件"Git Parameter"。如图

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20220326112424871.png#id=c9sRL&originHeight=619&originWidth=1782&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

手动测试：

```shell
[root@git-client ~]# git clone git@192.168.91.168:root/cloudweb.git
[root@git-client ~]# cd cloudweb
[root@git-client cloudweb]# vim src/main/webapp/index.jsp
```

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20220326113630658.png#id=C2EuJ&originHeight=463&originWidth=1289&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

```shell
[root@gitlab easy-springmvc-maven]# git add *
[root@gitlab easy-springmvc-maven]# git commit -m "修改用户为密码"
[root@gitlab easy-springmvc-maven]# git tag -a "v1.0" -m "修改用户为密码"
[root@gitlab easy-springmvc-maven]# git tag  #查看一下
v1.0
[root@gitlab easy-springmvc-maven]# git push origin v1.0
[root@gitlab easy-springmvc-maven]#
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670987518538-79276bf9-3ac9-443a-9e89-06d8cadf420d.png#averageHue=%23faf8f6&clientId=u57f8f598-4783-4&from=paste&height=558&id=u5ab13157&originHeight=698&originWidth=1224&originalType=binary&ratio=1&rotation=0&showTitle=false&size=60919&status=done&style=none&taskId=u9e62e22f-f157-4f65-b376-03f22aa5879&title=&width=979.2)

##### 配置Jenkins参数化构建（tag方式）

![image-20240606012539201](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406060125491.png)

![image-20240606012642272](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406060126396.png)





![image-20240606012743783](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406060127090.png)



![image-20240606012927788](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406060129014.png)

![image-20240606013128201](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406060131378.png)



![image-20240606014338355](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406060143446.png)

![image-20240606014352191](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406060143393.png)



![image-20240606014404683](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406060144749.png)


开发人员再次更新代码，推送仓库

```shell
[root@git-client cloudweb]# vim src/main/webapp/index.jsp
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670988048171-5da1ebd5-d6ad-4f68-ac47-6e268998ed00.png#averageHue=%23070302&clientId=u57f8f598-4783-4&from=paste&height=254&id=u9452d61e&originHeight=318&originWidth=1358&originalType=binary&ratio=1&rotation=0&showTitle=false&size=34425&status=done&style=none&taskId=u4ba042e4-bd77-4868-8219-5eabca205b1&title=&width=1086.4)
```shell
[root@git-client cloudweb]# git add *
[root@git-client cloudweb]# git commit -m "用户user & 密码pass"
[master ffa6acf] 用户user & 密码pass
 1 file changed, 2 insertions(+), 2 deletions(-)
[root@git-client cloudweb]# git tag -a "v1.1" -m "用户user & 密码pass"
[root@git-client cloudweb]# git tag
v1.0
v1.1
[root@git-client cloudweb]# git push origin v1.1
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670988152608-80c3a16b-83e9-4363-b7f4-ba00d661a53c.png#averageHue=%23faf8f6&clientId=u57f8f598-4783-4&from=paste&height=526&id=ua1da8988&originHeight=657&originWidth=1247&originalType=binary&ratio=1&rotation=0&showTitle=false&size=59352&status=done&style=none&taskId=ue251d857-c5bd-422e-9950-f730f20c6f6&title=&width=997.6)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1670988252805-47ea6916-a09e-44c6-a3a1-efd91dfdc5f4.png#averageHue=%23fbfafa&clientId=u57f8f598-4783-4&from=paste&height=437&id=u152c49b0&originHeight=546&originWidth=860&originalType=binary&ratio=1&rotation=0&showTitle=false&size=34691&status=done&style=none&taskId=u1a502756-cb0b-46f3-abcb-5c6105cba0f&title=&width=688)
访问测试
尝试进行版本切换，再访问测试

##### 配置Jenkins参数化构建(commit修订号)
开发人员如果不会打标签，或者说他们不愿意配合打标签
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1668755336084-5813fea2-fece-4763-b914-ece7b2bafdd1.png#averageHue=%23faf9f9&clientId=u675905c7-e294-4&from=paste&height=688&id=u6cc4e0a9&originHeight=860&originWidth=1337&originalType=binary&ratio=1&rotation=0&showTitle=false&size=55597&status=done&style=none&taskId=u4dfd4df9-eeb3-47cd-af05-11fca547295&title=&width=1069.6)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1668755355976-1e6f1c29-a626-428a-82d4-f47ba9013bae.png#averageHue=%23f9f8f8&clientId=u675905c7-e294-4&from=paste&height=651&id=u15d3fd0a&originHeight=814&originWidth=1341&originalType=binary&ratio=1&rotation=0&showTitle=false&size=54838&status=done&style=none&taskId=u077a24c9-3cc3-40e9-a08c-9eec51aef90&title=&width=1072.8)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1668755371041-b2cb07c0-e061-4993-a99e-dca651bb2920.png#averageHue=%23e1e1e0&clientId=u675905c7-e294-4&from=paste&height=450&id=u67c95efb&originHeight=563&originWidth=1206&originalType=binary&ratio=1&rotation=0&showTitle=false&size=78444&status=done&style=none&taskId=u7b76fe22-98e4-4b8f-864c-ac7282b7514&title=&width=964.8)



### 五、Jenkins多节点配置

在企业里面使用Jenkins自动部署+测试平台时，每天更新发布几个网站版本,很频繁,但是对于一些大型的企业来讲，Jenkins就需要同时处理很多的任务，这时候就需要借助Jenkins多个node或者我们所说的Jenkins分布式SLAVE，今天我们带大家来学习Jenkins多实例的配置；

添加Linux平台Jenkins SLAVE配置：

1. 由于Jenkins是Java程序，添加的SLAVE客户端服务器必须安装Java JDK环境；
2. 创建远程执行Jenkins任务的用户，一般为Jenkins用户，工作目录为/home/Jenkins;
3. Jenkins服务器免秘钥登录Slave服务器或者通过用户名和密码登录；

#### 1.添加从节点

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20211009204536504.png#id=SoNjI&originHeight=845&originWidth=1692&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20211009204609760.png#id=de9Yc&originHeight=631&originWidth=1893&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20211009204648831.png#id=qbSD8&originHeight=558&originWidth=1895&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

#### 2.参数详解

```
名字：节点的名字
描述：说明这个节点的用途等
of executors:并发构建数量
远程工作目录：用于存放jenkins的工作空间的
标签：分配job会以标签的名称去分配
用法：节点的使用策略
启动方法：windows的话就不要给自己添堵了，选择 Java web start
```

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20220328104326621.png#id=kdaUn&originHeight=620&originWidth=1300&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20211009221943512.png#id=qG4LQ&originHeight=607&originWidth=924&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20211009221815103.png#id=ubYos&originHeight=655&originWidth=945&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

#### 3.指定java命令路径

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20211009221903569.png#id=fMA2s&originHeight=811&originWidth=933&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20211009222015840.png#id=d2gER&originHeight=366&originWidth=1264&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

#### 4.测试从节点

项目指定到哪个节点运行。

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20220328104400072.png#id=yHjLw&originHeight=528&originWidth=1559&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20211009222752678.png#id=ZbUbb&originHeight=691&originWidth=1584&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

![](https://youngfitfei.oss-cn-beijing.aliyuncs.com/img/image-20211009224634465.png#id=pjo2H&originHeight=270&originWidth=1481&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
