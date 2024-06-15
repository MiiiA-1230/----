# Tomcat

### 一、安装

##### 1、安装jdk

```
tar -xf jdk包名 -C /usr/local
解压jdk到/usr/local目录

cd /usr/local
#切换至/usr/local目录

ln -s jdk1.8.0_211/ java
#制作软连接

vim /etc/profile.d/jdk.sh
#添加配置信息
export JAVA_HOME=/usr/local/java   #指定java安装目录
export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH    #用于指定java系统查找命令的路径

source vim /etc/profile.d/jdk.sh
#重新加载配置文件

java -version
#提示信息
java version "1.8.0_211"
Java(TM) SE Runtime Environment (build 1.8.0_211-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.211-b12, mixed mode)
```

##### 2、安装tomcat

```
# 解压Tomcat安装包到指定目录
[root@java-tomcat01 ~]# tar xf apache-tomcat-8.5.45.tar.gz  -C /usr/local/ &&  cd /usr/local/

# 创建软链接
[root@java-tomcat01 local]# ln -s apache-tomcat-8.5.45/   tomcat

# 设置环境变量:
[root@java-tomcat01 local]# vim /etc/profile.d/tomcat.sh
#!/bin/bash
export TOMCAT_HOME=/usr/local/tomcat
export PATH=$PATH:$TOMCAT_HOME/bin:$JAVA_HOME/bin

# 重新加载配置文件
[root@java-tomcat1 application]# source  /etc/profile.d/tomcat.sh

# 查看tomcat是否安装成功
[root@java-tomcat01 bin]# catalina.sh version
命令会显
# 表示Tomcat实例的基础目录，即配置文件、日志文件等所在的目录路径。
Using CATALINA_BASE:   /usr/local/tomcat
# 表示Tomcat的安装目录，即Tomcat的主要程序文件所在的目录路径。
Using CATALINA_HOME:   /usr/local/tomcat
# 表示Tomcat的临时目录，用于存放临时文件和数据，比如会话数据、上传文件等。
Using CATALINA_TMPDIR: /usr/local/tomcat/temp
# 表示Java运行时环境（JRE）的安装目录，即Java解释器和标准类库所在的目录路径。
Using JRE_HOME:        /usr/local/java
# 表示Java类路径（CLASSPATH），用于指定Java程序运行时要加载的类库和目录。在这里指定了两个JAR文件，分别是bootstrap.jar和tomcat-juli.jar，这些JAR文件包含了Tomcat启动和日志相关的类。
Using CLASSPATH:       /usr/local/tomcat/bin/bootstrap.jar:/usr/local/tomcat/bin/tomcat-juli.jar
# 表示Tomcat服务器的版本信息，这里是Tomcat 8.5.45。
Server version: Apache Tomcat/8.5.45
# 表示Tomcat服务器的构建时间，即Tomcat程序文件的编译时间。
Server built:   Aug 14 2019 22:21:25 UTC
# 表示Tomcat服务器的具体版本号。
Server number:  8.5.45.0
# 表示操作系统的名称，这里是Linux。
OS Name:        Linux
# 表示操作系统的版本号。
OS Version:     3.10.0-1160.el7.x86_64
# 表示操作系统的体系结构，这里是64位的。
Architecture:   amd64
# 表示Java虚拟机（JVM）的版本信息。
JVM Version:    1.8.0_211-b12
# 表示Java虚拟机（JVM）的提供商，这里是Oracle Corporation。
JVM Vendor:     Oracle Corporation
```

### 二、实战心得

![](C:\Users\Administrator\Desktop\Work\自研笔记\images\微信图片_20240513165208.png)
