## Tomcat 运维实战

[TOC]

### 1、Java常识

- `JVM（Java虚拟机）`：
  - 是一个抽象的计算机器，它使计算机能够运行Java程序。它是Java运行环境（JRE）的关键部分。JVM将Java字节码转换为机器语言，使得Java程序可以在不同的平台上运行而无需修改。
- **JRE（Java运行环境）**：
  - JRE是一组软件工具，提供运行Java应用程序所需的环境。它包括JVM、类库和其他支持文件。当您在系统上安装Java时，您实际上安装的是JRE。JRE足以运行Java应用程序。
- **Java类库**：
  - Java类库（也称为Java API - 应用程序编程接口）是一组预先编写的类、接口和包，为Java应用程序提供常用功能。开发人员可以利用它们来高效地构建Java应用程序。

#### **1.1 什么是JAVA虚拟机**

Java虚拟机（JVM）是Java平台的核心组件之一，它是一个在物理计算机上模拟的计算机，能够执行Java字节码。JVM的主要功能包括：

1. **字节码执行**：JVM执行Java编译器生成的字节码文件（.class文件），这些字节码文件包含了被编译的Java程序的中间代码。JVM通过解释字节码或将其即时编译为本地机器代码来执行Java程序。
2. **内存管理**：JVM负责分配和管理Java程序运行所需的内存。这包括对内存的动态分配、垃圾回收以及内存区域的划分（如堆、栈、方法区等）。
3. **垃圾回收**：JVM中的垃圾回收器负责自动回收不再使用的内存对象，以避免内存泄漏和内存溢出问题，从而保证Java程序的稳定性和可靠性。



**白话文解释：**

所谓虚拟机，就是一台虚拟的计算机。他是一款软件，用来执行一系列虚拟计算机指令。大体上，虚拟机可以分为`系统虚拟机`和`程序虚拟机`。大名鼎鼎的VisualBox、VMware就属于系统虚拟机。他们完全是对物理计算机的仿真。提供了一个可以运行完整操作系统的软件平台。
程序虚拟机的典型代表就是Java虚拟机，它专门为执行单个计算机程序而设计，在Java虚拟机中执行的指令我们称为Java字节码指令。无论是系统虚拟机还是程序虚拟机，在上面运行的软件都限制于虚拟机提供的资源中。



#### **1.2 JAVA 如何做到跨平台**

Java实现跨平台的关键在于其独特的编译和执行方式：

1. **字节码**： Java源代码首先被编译成`Java字节码`（.class文件），而不是针对特定平台的本地机器代码。这个字节码是与`平台无关`的中间代码，它包含了被执行的程序逻辑，但与具体的操作系统和硬件无关。
2. **JVM（Java虚拟机）**： Java字节码由Java虚拟机（JVM）执行。JVM是一个针对特定平台的软件程序，它负责在运行时将字节码翻译成本地机器码，并在特定平台上执行。因此，只需为每个平台实现一个JVM，就能够在该平台上运行Java程序。
3. **一次编写，到处运行**： 由于Java程序被编译成平台无关的字节码，所以同一份Java代码可以在任何安装了Java虚拟机的平台上运行，而不需要对代码进行修改或重新编译。这就实现了“`一次编写，到处运行`”的理念，使得Java具有了强大的跨平台能力。

Java实现跨平台的核心是将源代码编译成与`平台无关`的`字节码`，并由Java虚拟机在各个平台上解释和执行字节码。这种设计使得Java成为了一种广泛应用的`跨平台`编程语言。同一个JAVA程序(JAVA字节码的集合)，通过JAVA虚拟机(JVM)运行于各大主流操作系统平台
比如Windows、CentOS、Ubuntu等。程序以虚拟机为中介，来实现跨平台。

![1562148557621](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/1562148557621.png)

#### 1.3 常用虚拟机参数

Java虚拟机（JVM）提供了多种类型的参数，用于配置和优化Java应用程序的运行。其中，常见的参数类型包括以下三种：

- **标准参数**
  - 这些参数对所有的JVM都是通用的，例如 `-version`、`-help` 等。它们用于控制JVM的一般行为，如打印版本信息、显示帮助信息等。标准参数中包括功能和输出的参数都是很稳定的，很可能在将来的JVM版本中不会改变。你可以用 java 命令（或者是用 java -help）检索出所有标准参数。

- ##### X  类型参数

  - `非标准化`的参数，在将来的版本中可能会改变。所有的这类参数都以 -X 开始。是特定于JVM实现的，可能会在不同版本的JVM之间有所差异。它们用于配置JVM的高级功能，如内存管理、垃圾回收算法、线程堆栈大小等。

- ##### XX  类型参数

  - 在实际情况中 X 参数和 XX 参数并没有什么不同。X 参数的功能是十分稳定的。用一句话来说明 XX 参数的语法。所有的 XX 参数都以"-XX:"开始，但是随后的语法不同，取决于参数的类型：
    - 
      开启GC日志的参数: `-XX:+PrintGC(打印GC日志)`
    - 最大永久代最大值： `-XX:MaxPermSize=2048m`
  
- **应用程序参数**

  - 这些参数是由特定的Java应用程序定义和使用的，它们不是由JVM直接解释的。应用程序参数用于传递给Java应用程序的命令行参数或配置参数，以影响应用程序的行为，如指定输入文件、设置日志级别等。

#### 1.4 常用的JVM参数

##### 1.4.1 跟踪JAVA虚拟机的垃圾回收

GC日志：jvm`垃圾回收`，记录jvm的`运行状态`，OOM`内存溢出`的报错信息等。

- `%t` 将会被替代为时间字符串，格式为: YYYY-MM-DD_HH-MM-SS

开启GC日志:

```shell
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/data0/logs/gc-%t.log"
```

1. ##### JVM新生代、永久代、老年代

![](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202405102053029.webp)



**`新生代`：Tomcat的新生代概念主要指的是JVM中用于存放新创建对象的内存区域**。一般占据堆的`1/3`空间。可以通过参数`-XX:NewRatio`来调整。由于频繁创建对象，所以新生代会频繁触发MinorGC进行垃圾回收。

新生代又分为 `Eden`区、`ServivorFrom`、`ServivorTo`三个区。

- `Eden区`：Java新对象的出生地（如果新创建的对象占用内存很大，则直接分配到老年代）。当Eden区内存不够的时候就会触发MinorGC，对新生代区进行一次垃圾回收。
- `ServivorTo`：保留了一次MinorGC过程中的幸存者。
- `ServivorFrom`：上一次GC的幸存者，作为这一次GC的被扫描者。
- 当JVM无法为新建对象分配内存空间的时候(Eden满了)，Minor GC被触发。因此新生代空间占用率越高，Minor GC越频繁。

MinorGC的过程：`采用复制算法`。

首先，把Eden和Servivor(幸存者) From区域中存活的对象复制到Servicor(幸存者)To区域（如果有对象的年龄以及达到了老年的标准，一般是15，则赋值到老年代区）。同时把这些对象的年龄+1（如果ServicorTo不够位置了就放到老年区）然后，清空Eden和ServicorFrom中的对象；最后，ServicorTo和ServicorFrom互换，原ServicorTo成为下一次GC时的ServicorFrom区。

**`老年代`**：**用于存放长时间存活的对象**。

老年代的对象比较稳定，所以MajorGC不会频繁执行。在进行MajorGC前一般都先进行了一次MinorGC，使得有新生代的对象晋身入老年代，导致空间不够用时才触发。当无法找到足够大的连续空间分配给新创建的较大对象时也会提前触发一次MajorGC进行垃圾回收腾出空间。

MajorGC采用标记—清除算法：

首先扫描一次所有老年代，标记出存活的对象，然后回收没有标记的对象。MajorGC的耗时比较长，因为要扫描再回收。当老年代也满了装不下的时候，就会抛出OOM（Out of Memory）异常。

**`永久代`**：指内存的永久保存区域，主要存放Class和Meta（元数据）的信息。

Class在被加载的时候被放入永久区域。它和和存放实例的区域不同，GC不会在主程序运行期对永久区域进行清理。所以这也导致了永久代的区域会随着加载的Class的增多而胀满，最终抛出OOM异常。

在Java8中，永久代已经被移除，被一个称为“元数据区”（元空间）的区域所取代。

元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。因此，默认情况下，元空间的大小仅受本地内存限制。

##### 1.4.2 配置JAVA虚拟机的堆空间

```shell
-Xms：指定Java虚拟机的初始堆大小。例如，-Xms512m表示初始堆大小为512MB。
-Xmx：指定Java虚拟机的最大堆大小。例如，-Xmx1024m表示最大堆大小为1024MB。
# 这两个参数可以用来控制Java应用程序在运行时所能使用的堆空间大小。通常情况下，将这两个参数设置为相同的值，以避免堆空间大小的动态调整。增大堆空间可以提高应用程序的性能，减少垃圾回收的频率，但也会增加内存的消耗。`推荐设置为可用物理内存的一半`。

例如，如果要将初始堆大小设置为512MB，最大堆大小设置为1024MB，可以使用以下命令：
java -Xms512m -Xmx1024m -jar YourApp.jar
```



##### 1.4.3 配置JAVA虚拟机的永久区(方法区)

**JAVA虚拟机的永久区介绍**

在Java虚拟机中，`永久区`是一块用于存储类、方法、常量等元数据的内存区域。在早期的Java版本中，永久区主要用于存储这些元数据，例如类的字节码、静态变量、方法信息、常量池等。然而，随着Java技术的发展，永久代在Java 8及之后的版本中被元空间（Metaspace）所取代。

永久区的特点包括：

1. **固定大小**：永久区的大小在Java虚拟机启动时被固定下来，不能动态调整。在Java 8之前，可以通过设置 `-XX:PermSize` 和 `-XX:MaxPermSize` 参数来调整永久代的大小。

2. **垃圾回收**：尽管永久区的大小是固定的，但是永久区中的垃圾仍然会被回收。Java虚拟机会执行永久代的垃圾回收以释放不再使用的类和元数据。在Java 8之前，可以通过 `-XX:+CMSClassUnloadingEnabled` 参数开启永久代的垃圾回收。

3. **内存泄漏问题**：永久区的内存泄漏是一个常见的问题。由于永久区的大小是`固定`的，如果应用程序不断加载新的类或者重新加载类，而没有对原来的类进行`垃圾回收`，就会导致永久区的内存使用量不断增加，最终导致内存溢出。

```java
-XX:PermSize   	  内存永久保留区域  ：//所占用的内存是堆内存的一部分内存，不能超过堆内存
-XX:MaxPermSize   内存最大永久保留区域

JDK 1.8中 PermSize 和 MaxPermGen 已经无效。JDK 1.8 中已经不存在永久代的结论 而以 元空间 代替。 
```

![1569157453195](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/1569157453195.png)



### **2、动、静态请求的区别(`重点`)**

静态请求和动态请求是与网络服务中的`内容生成`和`提供相关`的两个重要概念。

1. **静态请求**：
   - 静态请求是指客户端（通常是Web浏览器）向服务器请求获取静态内容，如HTML文件、CSS样式表、JavaScript脚本、图像、视频和其他媒体文件等。这些内容在服务器上`预先存在`，不需要在请求时生成或处理。服务器在收到静态请求后，会直接将文件发送给客户端，不进行任何额外的`处理或计算`。因为静态内容在每次请求时都是`相同`的，所以静态请求的响应速度通常`较快`。

2. **动态请求**：
   - 动态请求是指客户端向服务器请求生成动态内容的过程。动态内容是根据用户请求时的`特定条件`、`参数`或者数据库中的`数据`等动态生成的。服务器收到动态请求后，会根据请求的内容、用户信息、数据库查询结果等动态生成需要返回给客户端的内容，然后将生成的内容发送给客户端。动态请求通常涉及服务器端的脚本语言（如PHP、Python、Node.js、Java等）来处理请求并生成响应。由于动态请求需要在服务器端进行处理和计算，响应时间通常`较长`。

总结：
- 静态请求指的是获取预先存在并不需要额外处理的静态内容，响应速度较快。
- 动态请求指的是根据特定条件动态生成内容的请求，响应时间较长，需要服务器端进行处理和计算。



### 3、企业 Tomcat 运维

#### **3.1Tomcat 简介**

**Tomcat**官网： [http://tomcat.apache.org](http://tomcat.apache.org/)

Tomcat是Apache软件基金会（Apache Software Foundation）的Jakarta 项目中的一个核心项目，由Apache、Sun和其他一些公司及个人共同开发而成。并且Tomcat服务器是一个免费的开放源代码的Web应用服务器，属于`轻量级`应用服务器，在中小型系统和并发访问用户不是很多的场合下被普遍使用，是开发和调试JSP程序的首选。

**Tomcat：**JAVA容器，WEB容器，WEB中间件

#### 3.2 Tomcat端口号说明

1. **`HTTP端口（默认端口号为8080）`**：
   - HTTP端口是用于处理HTTP请求的端口。当浏览器发送HTTP请求时，会使用这个端口与Tomcat服务器通信。
   - 默认情况下，Tomcat监听8080端口，可以通过在`server.xml`配置文件中的`<Connector>`元素来修改。
2. **HTTPS端口（默认端口号为443）**：
   - HTTPS端口是用于处理HTTPS请求的端口。HTTPS是HTTP的安全版本，通过SSL/TLS加密传输数据。
   - 默认情况下，Tomcat监听443端口，可以通过在`server.xml`配置文件中的`<Connector>`元素来修改。
3. **`AJP端口（默认端口号为8009）`**：
   - AJP（Apache JServ Protocol）端口是用于与Apache HTTP服务器之间进行通信的端口。通常用于将Tomcat与Apache Web服务器进行集成。
   - 默认情况下，Tomcat监听8009端口，可以通过在`server.xml`配置文件中的`<Connector>`元素来修改。
4. **`Shutdown端口（默认端口号为8005）`**：
   - Shutdown端口用于`接收关闭`Tomcat服务器的命令。当管理员想要停止Tomcat服务器时，可以通过连接到这个端口发送关闭命令。
   - 默认情况下，Tomcat监听8005端口，可以通过在`server.xml`配置文件中的`<Server>`元素来修改。

 

**使用方案**：

方案一：  Tomcat         //单独使用   ----基本不用
方案二：  Nginx+Tomcat       //反向代理和负载均衡                                    			
方案三：                                   
                                  Nginx 
                                      |
    +--------------------------------------------------------+
    |               |               | 		              |
Tomcat1 Tomcat2 Tomcat3         nginx服务器  ----解析静态页面

建议使用Nginx和Tomcat配合，Nginx处理静态，Tomcat处理动态程序
方案三中后端Tomcat可以运行在单独的主机，也可以是同一台主机上的多实例



#### 3.3 Tomcat安装

##### 3.3.1 Tomcat基础环境JDK

Java Development Kit（JDK）是Java开发工具包的缩写，是Java平台的核心组件之一，提供了用于开发、编译、调试和运行Java应用程序的各种工具和库。以下是JDK的主要组成部分和功能：

1. **Java编译器（javac）**：
   - Java编译器将Java源代码编译成字节码，可由Java虚拟机（JVM）执行。
2. **Java运行时环境（JRE）**：
   - JDK包含完整的Java运行时环境，包括Java虚拟机（JVM）和Java标准类库。
3. **Java标准类库**：
   - JDK包含了大量的Java标准类库，提供了丰富的API，用于开发各种类型的应用程序，包括文件操作、网络通信、图形用户界面（GUI）、数据库访问等功能。
4. **调试工具**：
   - JDK提供了一系列调试工具，如Java调试器（jdb）和Java虚拟机调试接口（JVMTI），用于调试Java应用程序和排查问题。
6. **JavaDoc工具**：
   - JavaDoc工具用于从Java源代码生成API文档，帮助开发者编写和管理代码文档。

JDK是Java开发的基础，开发者需要安装JDK才能进行Java程序的开发、编译和运行。Java开发人员通常会根据自己的需求和偏好选择合适的JDK版本进行开发工作。

**JDK**下载面页：http://www.oracle.com/technetwork/java/javase/downloads/index.html>



##### 3.3.2 安装Tomcat & JDK

安装时候选择tomcat软件版本要与程序开发使用的版本一致。jdk版本要进行与tomcat保持一致。

系统环境说明

```shell
[root@java-tomcat01 ~]# uname -a
Linux tomcat01 3.10.0-1160.el7.x86_64 #1 SMP Mon Oct 19 16:18:59 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux

[root@java-tomcat01 ~]# cat /etc/redhat-release 
CentOS Linux release 7.9.2009 (Core)

[root@java-tomcat01 ~]# setenforce 0
[root@java-tomcat01 ~]# sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

[root@java-tomcat01 ~]# systemctl disable --now firewalld
```

安装JDK

```shell
上传jdk1.8到服务器。安装jdk
[root@java-tomcat1 ~]# tar xzf jdk-8u191-linux-x64.tar.gz -C /usr/local/
[root@java-tomcat1 ~]# cd /usr/local/
[root@java-tomcat1 local]# ln -s jdk1.8.0_211  java
设置环境变量:
[root@java-tomcat1 local]# vim /etc/profile.d/jdk.sh
export JAVA_HOME=/usr/local/java   #指定java安装目录
export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH    #用于指定java系统查找命令的路径
检测JDK是否安装成功:
[root@java-tomcat1 local]# source vim /etc/profile.d/jdk.sh
[root@java-tomcat1 local]# java -version
java version "1.8.0_191"
Java(TM) SE Runtime Environment (build 1.8.0_191-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.191-b12, mixed mode)
```

安装Tomcat

```shell
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



#### 3.4 Tomcat目录介绍

##### 3.4.1 tomcat主目录介绍

```shell
[root@java-tomcat1 ~]# cd /usr/local/tomcat
[root@java-tomcat1 tomcat]# yum install -y tree
[root@java-tomcat1 tomcat]# tree -L 1
.
├── bin     # 包含了Tomcat服务器的可执行文件，如启动和关闭脚本、管理脚本等。
├── BUILDING.txt	# 包含了关于如何构建Tomcat服务器的说明文档。
├── conf    # 包含了Tomcat服务器的配置文件，如服务器配置、日志配置、Web应用程序配置等。
├── CONTRIBUTING.md	# 包含了关于如何向Tomcat项目贡献代码的说明文档。
├── lib      # 包含了Tomcat服务器运行所需的Java类库文件。
├── LICENSE		# 包含了Tomcat服务器的许可证文件。
├── logs     # 包含了Tomcat服务器的日志文件，记录了服务器的运行状态和事件信息。
├── NOTICE		# 包含了关于Tomcat服务器的版权和许可信息的通知文件。
├── README.md	# 包含了Tomcat服务器的简要说明文档。
├── RELEASE-NOTES	# 包含了Tomcat服务器的发布说明文档，记录了每个版本的更新内容和改进。
├── RUNNING.txt		# 包含了关于如何运行Tomcat服务器的说明文档。
├── temp     # 用于存放Tomcat服务器的临时文件，如会话数据、上传文件等。
├── webapps  # 用于存放Web应用程序的目录，每个子目录代表一个独立的Web应用程序
└── work     # 用于存放Tomcat服务器的工作目录，如编译的JSP文件、临时缓存等。

7 directories, 7 files
```

**2、webapps目录介绍**

```shell
[root@java-tomcat1 tomcat]# cd webapps/
[root@java-tomcat1 webapps]# tree -L 1
.
├── docs  # 包含了Tomcat服务器的文档和示例文件，如用户手册、API文档等。
├── examples  # 包含了Tomcat服务器的示例应用程序，提供了一些简单的示例代码和演示。
├── host-manager  # 包含了Tomcat的主机管理应用程序，允许管理员通过Web界面管理虚拟主机。
├── manager    # 包含了Tomcat的应用程序管理应用程序，允许管理员通过Web界面管理部署在Tomcat上的Web应用程序
└── `ROOT`    # 是Tomcat服务器的默认根应用程序（也称为ROOT应用程序），即当用户访问Tomcat服务器时默认会加载的应用程序。通常用于展示Tomcat服务器的欢迎页面或其他默认内容。

5 directories, 0 files
```

**3、Tomcat配置文件目录介绍（conf）**

```shell
[root@java-tomcat1 webapps]# cd ../conf/
[root@java-tomcat1 conf]# tree -L 1
.
├── catalina.policy		# Tomcat服务器的安全策略文件，用于定义安全策略和权限控制。
├── catalina.properties	# Tomcat服务器的全局配置文件，包含了一些Tomcat服务器的运行参数和属性设置。
├── context.xml			# Tomcat服务器的上下文配置文件，用于配置特定Web应用程序的上下文参数和资源定义。
├── jaspic-providers.xml
├── jaspic-providers.xsd 
├── logging.properties	# Tomcat服务器的日志配置文件，用于配置日志记录器、日志格式和输出目的地等。
├── `server.xml			# Tomcat服务器的主配置文件，包含了服务器的核心配置，如端口设置、连接器配置、虚拟主机设置等。`
├── tomcat-users.xml	# Tomcat服务器的用户认证配置文件，用于定义Tomcat服务器的用户、角色和访问权限。
├── tomcat-users.xsd	# Tomcat用户认证配置文件的XML模式定义（XSD）文件，用于验证Tomcat用户认证配置文件的结构和语法。		
└── web.xml				# 定义的Web应用程序配置文件，包含了Web应用程序的部署描述符，用于配置Servlet、过滤器、监听器等组件。

0 directories, 10 files
```

**4、Tomcat的管理**

```shell
# 启动程序 
[root@java-tomcat01 conf]# catalina.sh  start

# 关闭程序 
[root@java-tomcat01 conf]# catalina.sh stop
```

 `注意`：**tomcat未启动的情况下使用shutdown脚本，会有大量的输出信息。**

检查tomcat是否启动正常

```shell
[root@java-tomcat1 bin]# netstat -lntp  |grep java
tcp6       0      0 :::8080         :::*                   LISTEN      30560/java
tcp6       0      0 127.0.0.1:8005          :::*          LISTEN      30560/java
tcp6       0      0 :::8009                 :::*           LISTEN      30560/java
```

`说明：`**所有与java相关的，服务启动都是java命名的进程**

启动完成浏览器进行访问http://IP:8080

![image-20240406210329630](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/image-20240406210329630.png)

查看Tomcat日志

```shell
[root@java-tomcat01 conf]# tail -f /usr/local/tomcat/logs/catalina.out 
06-Apr-2024 21:02:19.361 信息 [main] org.apache.coyote.AbstractProtocol.start Starting ProtocolHandler ["http-nio-8080"]
06-Apr-2024 21:02:19.373 信息 [main] org.apache.coyote.AbstractProtocol.start Starting ProtocolHandler ["ajp-nio-8009"]
06-Apr-2024 21:02:19.375 信息 [main] org.apache.catalina.startup.Catalina.start Server startup in 443 ms
```



#### 3.5 Tomcat主配置文件详解

##### 3.5.1 server.xml组件类别

顶级组件：位于整个配置的顶层，如server。

容器类组件：可以包含其它组件的组件，如service、engine、host、context。

连接器组件：连接用户请求至tomcat，如connector(引擎)。

```shell
<server>  #表示一个运行于JVM中的tomcat实例。
     <service> #服务。将connector关联至engine，因此一个service内部可以有多个connector，但只能有一个引擎engine。
     <connector /> #接收用户请求，类似于httpd的listen配置监听端口的
     <engine>  #核心容器组件，catalina引擎，负责通过connector接收用户请求，并处理请求，将请求转至对应的虚拟主机host。
     <host>   #类似于httpd中的虚拟主机，
     <context></context>  #配置context的主要目的指定对应对的webapp的根目录。其还能为webapp指定额外的属性，如部署方式等。
     </host>
     <host>
     <context></context>
     </host>
     </engine>
     </service>
</server>
```

##### 3.5.2 server.xml配置文件注释

```shell
<?xml version='1.0' encoding='utf-8'?>
<!--
<Server>元素代表整个容器,是Tomcat实例的顶层元素.它包含一个<Service>元素.并且它不能做为任何元素的子元素.
    port指定Tomcat监听shutdown命令端口
    shutdown指定终止Tomcat服务器运行时,发给Tomcat服务器的shutdown监听端口的字符串.该属性必须设置
-->
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
  <GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>
  <!--service服务组件-->
  <Service name="Catalina">
    <!-- Connector主要参数说明（见下面） -->
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
    <Engine name="Catalina" defaultHost="localhost">
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>
      <!-- 详情常见（host参数详解）-->
      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">
        <!-- 详情见扩展（Context参数说明 ）-->
        <Context path="" docBase="" debug=""/>
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
    </Engine>
  </Service>
</Server>
```

##### 3.5.3 Connector主要参数说明

```shell
port:指定服务器端要创建的端口号，并在这个端口监听来自客户端的请求。
protocol：连接器使用的协议，支持HTTP和AJP。AJP（Apache Jserv Protocol）专用于tomcat与apache建立通信的， 在httpd反向代理用户请求至tomcat时使用（可见Nginx反向代理时不可用AJP协议）。
redirectPort：指定服务器正在处理http请求时收到了一个SSL传输请求后重定向的端口号
maxThreads：接收最大请求的并发数
connectionTimeout  指定超时的时间数(以毫秒为单位)
```

<Connector port="8080" protocol="HTTP/1.1" 

​               maxThreads="500"    ----默认是200
​               connectionTimeout="20000"       ---------连接超时时间。单位毫秒
​               redirectPort="8443" />

##### 3.5.5 host参数详解

​            <Host name="localhost"  appBase="webapps"
​            unpackWARs="true" autoDeploy="true">

```shell
host	# 表示一个虚拟主机。
name	# 指定主机名。
appBase	# 应用程序基本目录，即存放应用程序的目录.一般为appBase="webapps"，相对于CATALINA_HOME而言的，也可以写绝对路径。
unpackWARs	# 如果为true，则tomcat会自动将WAR文件解压，否则不解压，直接从WAR文件中运行应用程序。
autoDeploy	# 在tomcat启动时，是否自动部署。
```



#### 3.6  WEB站点部署

上线的代码有两种方式：

- 直接将程序目录放在webapps目录下面。
- 使用开发工具将程序打包成war包，然后上传到webapps目录下面。

##### **3.6.1 使用war包部署web站点**

```shell
# 方式一：自动解压
[root@java-tomcat1 ~]# wget http://updates.jenkins-ci.org/download/war/2.129/jenkins.war
[root@java-tomcat1 ~]# ls
jenkins.war
# 进入tomcat目录
[root@java-tomcat1 ~]# cd /usr/local/tomcat/ 

# 将原来的发布网站目录备份
[root@java-tomcat1 tomcat]# cp -r webapps/ /opt/    

# 清空发布网站里面的内容
[root@java-tomcat1 tomcat]# cd webapps/
[root@java-tomcat1 webapps]# rm -rf *   

 # 将war包拷贝到当前目录
[root@java-tomcat1 webapps]# cp ~/jenkins.war . 

# 启动Tomcat实例
[root@java-tomcat1 webapps]# catalina.sh start   
Using CATALINA_BASE:   /data/application/tomcat
Using CATALINA_HOME:   /data/application/tomcat
Using CATALINA_TMPDIR: /data/application/tomcat/temp
Using JRE_HOME:        /usr/local/java
Using CLASSPATH:       /data/application/tomcat/bin/bootstrap.jar:/data/application/tomcat/bin/tomcat-juli.jar
Tomcat started.
[root@java-tomcat1 webapps]# ls
jenkins  jenkins.war

# 方式二：手动解压
[root@java-tomcat1 webapps]# catalina.sh stop   #关闭tomcat
[root@java-tomcat1 ~]# cd /usr/local/tomcat/webapps/
[root@java-tomcat1 webapps]# rm -rf *    
[root@java-tomcat1 webapps]# mkdir ROOT      #创建一个ROOT目录存放war包
[root@java-tomcat1 webapps]# cd ROOT/
[root@java-tomcat1 ROOT]# cp /root/jenkins.war .
[root@java-tomcat1 ROOT]# unzip jenkins.war
[root@java-tomcat01 ROOT]# catalina.sh start 

```

![1562256628954](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/1562256628954.png)

浏览器访问：http://IP:8080/jenkins 

![1562255930078](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/1562255930078.png)



##### 3.6.2 自定义默认网站目录

1、修改默认发布目录:

```shell
[root@java-tomcat1 ~]# catalina.sh stop
[root@java-tomcat1 ~]# mkdir -p /data/application/webapp  #创建发布目录
[root@java-tomcat1 ~]# vim /data/application/tomcat/conf/server.xml
```

将原来的

![1562335606126](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/1562335606126.png)

修改为

![1562335650998](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/1562335650998.png)

```shell
[root@java-tomcat1 ~]# cp /root/jenkins.war /data/application/webapp/
[root@java-tomcat1 ~]# catalina.sh start ; tail -f /usr/local/tomcat/logs/catalina.out
```

浏览器访问：http://IP:8080/jenkins



##### **3.6.3 部署开源站点（jspgou商城）**

第一：安装配置数据库

```shell
1.使用mariadb
[root@youngfit ~]# yum -y install mariadb mariadb-server
[root@youngfit ~]# systemctl enable --now mariadb
[root@youngfit ~]# mysql
MariaDB [(none)]> create database jspgou default charset=utf8;	//在数据库中操作，创建数据库并指定字符集
MariaDB [(none)]> flush privileges;		//(可选操作)
exit;
```



第二：jspgou商城上线

```shell
上传jspgou商城的代码
[root@java-tomcat1 ~]# unzip jspgouV6.1-ROOT.zip
[root@java-tomcat01~]# cd /usr/local/tomcat/webapps/
[root@java-tomcat01 webapps]# rm -rf *
[root@java-tomcat01 webapps]# cp -r ROOT/ .

# 查看JDBC连接数据库配置文件信息
[root@java-tomcat01 webapps]# vim ROOT/WEB-INF/config/jdbc.properties
jdbc.driverClassName=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://127.0.0.1:3306/jspgou?characterEncoding=UTF-8
jdbc.username=root
jdbc.password=

hibernate.db=mysql
hibernate.dialect=org.hibernate.dialect.MySQLInnoDBDialect

```



```shell
将数据导入数据库:
[root@java-tomcat1 ~]# cd DB/
[root@java-tomcat1 DB]# ls
jspgou.sql
[root@java-tomcat1 DB]# mysql -uroot -p  jspgou < jspgou.sql

启动tomcat访问:
[root@java-tomcat1 ~]# catalina.sh start ; tail -f /usr/local/tomcat/logs/catalina.out
[root@java-tomcat1 ~]# netstat -lntp | grep java
```

访问:<http://IP:8080/>

![1562343793980](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/1562343793980.png)

做一遍：要求用mysql5.7（会遇到问题，百度解决），贴近企业环境



##### 3.6.4 Tomcat多实例配置

**多实例（多进程）**：同一个程序启动多次，分为两种情况:

第一种：一台机器跑多个站点； 

第二种：多个机器跑一个站点多个实例，配合负载均衡;

1、复制程序文件

```shell
[root@java-tomcat01 ~]# cd /usr/local/
[root@java-tomcat01 local]# cp -r tomcat/ tomcat_2
[root@java-tomcat01 local]# rm -rf /usr/local/tomcat/webapps/*
[root@java-tomcat01 local]# rm -rf /usr/local/tomcat_2/webapps/*
[root@java-tomcat01 local]# cp -r /opt/webapps/* /usr/local/tomcat/webapps/
[root@java-tomcat01 local]# cp -r /opt/webapps/* /usr/local/tomcat_2/webapps/

[root@java-tomcat01 local]# vim tomcat_2/conf/server.xml
<Server port="8006" shutdown="SHUTDOWN">
    <Connector port="8082" protocol="HTTP/1.1"
    <Connector port="8010" protocol="AJP/1.3" redirectPort="8443" />
# 修改端口，以启动多实例。多实例之间端口不能一致
[root@java-tomcat1 local]# diff tomcat/conf/server.xml tomcat_2/conf/server.xml  #对比文件不同之处
3c3
< <Server port="8005" shutdown="SHUTDOWN">
---
> <Server port="8006" shutdown="SHUTDOWN">
19c19
<     <Connector port="8080" protocol="HTTP/1.1"
---
>     <Connector port="8082" protocol="HTTP/1.1"
23c23
<     <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
---
>     <Connector port="8010" protocol="AJP/1.3" redirectPort="8443" />
```

启动tomcat多实例

```shell
[root@java-tomcat1 local]# echo 8080 >> tomcat/webapps/ROOT/index.jsp 
[root@java-tomcat1 local]# echo 8082 >> tomcat_2/webapps/ROOT/index.jsp

# 启动：
[root@java-tomcat01 local]# /usr/local/tomcat/bin/startup.sh 
[root@java-tomcat01 local]# /usr/local/tomcat_2/bin/startup.sh 
```

检查端口查看是否启动:

```shell
[root@java-tomcat01 local]# ss -tunlp | grep java
tcp    LISTEN     0      100    [::]:8080               [::]:*                   users:(("java",pid=2814,fd=49))
tcp    LISTEN     0      100    [::]:8082               [::]:*                   users:(("java",pid=2869,fd=49))
tcp    LISTEN     0      1        [::ffff:127.0.0.1]:8005               [::]:*                   users:(("java",pid=2814,fd=75))
tcp    LISTEN     0      1        [::ffff:127.0.0.1]:8006               [::]:*                   users:(("java",pid=2869,fd=75))
tcp    LISTEN     0      100    [::]:8009               [::]:*                   users:(("java",pid=2814,fd=54))
tcp    LISTEN     0      100    [::]:8010               [::]:*                   users:(("java",pid=2869,fd=54))
```

2、在浏览器访问，进行测试

检查多实例的启动

<http://IP:8080/>

![image-20240407154854291](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/image-20240407154854291.png)

<http://IP:8082/>

![1562592140511](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/1562592140511.png)

#### 3.7 tomcat反向代理集群

**1、负载均衡器说明**

关闭防火墙和selinux

```shell
# yum安装nginx
[root@nginx-proxy ~]# cd /etc/yum.repos.d/
[root@nginx-proxy yum.repos.d]# vim nginx.repo
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1
[root@nginx-proxy yum.repos.d]# yum install yum-utils -y
[root@nginx-proxy yum.repos.d]# yum install nginx -y
```

**2、配置负载均衡器**

备份原配置文件并修改

```shell
[root@nginx-proxy ~]# cd /etc/nginx/conf.d/
[root@nginx-proxy conf.d]# cp default.conf default.conf.bak
[root@nginx-proxy conf.d]# mv default.conf tomcat.conf
[root@nginx-proxy conf.d]# vim tomcat.conf
upstream testweb {
	server 192.168.50.114:8081 weight=1 max_fails=1 fail_timeout=2s;
	server 192.168.50.114:8082 weight=1 max_fails=1 fail_timeout=2s;
}
server {
    listen       80;
    server_name  localhost;
    access_log  /var/log/nginx/proxy.access.log  main;

    location / {
       proxy_pass http://testweb;
       proxy_set_header Host $host:$server_port;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }       
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    } 
}

```

启动nginx

```shell
[root@nginx-proxy ~]# systemctl start nginx
```

**3、在浏览器上进行访问测试**

<http://192.168.174.20/>

![1562598544268](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/1562598544268.png)

![1562598580435](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/1562598580435.png)

### 4、Tomcat相关调优

#### 4.1 日志格式配置

```shell
[root@java-tomcat1 ~]# cd /data/application/tomcat/conf/
[root@java-tomcat1 conf]# vim server.xml
<Valve className="org.apache.catalina.valves.AccessLogValve" directory="/data/www/logs"
               prefix="jenkins-" suffix="-access_log"
               pattern="%{X-Real-IP}i - %v %t &quot;%r&quot; - %s %b %T &quot;%{Referer}i&quot; &quot;%{User-Agent}i&quot; %a &quot;-&quot; &quot;-&quot;" />
[root@java-tomcat1 conf]# mkdir -p /data/www想                                                                                                                                                                                                                                                                                                                                                               

日志参数解释：
    ％a - 远程IP地址
    ％A - 本地IP地址
    ％b - 发送的字节数，不包括HTTP头，或“ - ”如果没有发送字节
    ％B - 发送的字节数，不包括HTTP头
    ％h - 远程主机名
    ％H - 请求协议
    ％l (小写的L)- 远程逻辑从identd的用户名（总是返回' - '）
    ％m - 请求方法
    ％p - 本地端口
    ％q - 查询字符串（在前面加上一个“？”如果它存在，否则是一个空字符串
    ％r - 第一行的要求，客户端请求的第一行，包括HTTP方法、请求URL和协议版本。例如："GET /example.html HTTP/1.1"。
    ％s - 响应的HTTP状态代码
    ％S - 用户会话ID
    ％t - 日期和时间，在通用日志格式，使用指定格式（例如 %t{dd/MMM/yyyy:HH:mm:ss Z}）
    ％u - 远程用户身份验证
    ％U - 请求的URL路径
    ％v - 本地服务器名
    ％D - 处理请求的时间（以毫秒为单位）
    ％T - 处理请求的时间（以秒为单位）
    ％I （大写的i） - 当前请求的线程名称
```



#### 4.2 JVM 参数优化

```shell
[root@java-tomcat1 conf]# cd ../bin/
[root@java-tomcat1 bin]# cp catalina.sh catalina.sh.bak
[root@java-tomcat1 bin]# vim catalina.sh
JAVA_OPTS="$JAVA_OPTS -Xms1024m -Xmx1024m -XX:PermSize=512m -XX:MaxPermSize=512m"  #jdk1.7
JAVA_OPTS="$JAVA_OPTS -Xms1024m -Xmx1024m -XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=512m"   #jdk1.8
```

这条代码是用于设置Java虚拟机（JVM）的启动参数。让我们逐步解释：

1. `JAVA_OPTS="$JAVA_OPTS -Xms1024m -Xmx1024m -XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=512m"`

   这一行代码首先将 `JAVA_OPTS` 环境变量的值设置为其当前值（如果有的话），然后添加了一系列JVM启动参数：

   - `-Xms1024m`: 指定JVM的初始堆内存大小为1024 MB。
   - `-Xmx1024m`: 指定JVM的最大堆内存大小为1024 MB。
   - `-XX:MetaspaceSize=512m`: 指定元数据空间（Metaspace）的初始大小为512 MB。元数据空间用于存储类的元数据信息。
   - `-XX:MaxMetaspaceSize=512m`: 指定元数据空间的最大大小为512 MB。



#### 4.3 开启GC日志

```shell
[root@java-tomcat1 bin]# vim catalina.sh
JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/data/logs/gc-%t.log"

可选参数:
-XX:+AggressiveOpts，加快编译。会增加编译时间和内存消耗
-XX:+UseParallelGC，优化垃圾回收,通过多线程并行处理垃圾收集任务来减少停顿时间，从而提高应用程序的吞吐量。会导致一些额外的系统开销。
[root@java-tomcat1 bin]# mkdir /data/logs
```

这条代码是用于设置Java虚拟机（JVM）的启动参数，主要是用于配置垃圾回收（GC）日志输出。让我们逐步解释：

这一行代码首先将 `JAVA_OPTS` 环境变量的值设置为其当前值（如果有的话），然后添加了一系列JVM启动参数：

- `-XX:+PrintGCDetails`: 启用GC日志详细输出，包括每次GC事件的详细信息，如GC类型、GC前后堆内存情况等。
- `-XX:+PrintGCDateStamps`: 启用GC日志输出时间戳，每条GC日志输出的前缀将包含日期和时间信息。
- `-Xloggc:/data/logs/gc-%t.log`: 指定GC日志文件的输出路径和文件名格式。`/data/logs/gc-%t.log` 中的 `%t` 将会被替换为当前日期时间的时间戳。

![1562242242126](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/1562242242126.png)



#### 4.4 开启JMX端口便于监控

```shell
# vim catalina.sh
CATALINA_OPTS="$CATALINA_OPTS -Dcom.sun.management.jmxremote 
-Dcom.sun.management.jmxremote.port=10028 
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false 
-Djava.rmi.server.hostname=java69-matrix.zeus.lianjia.com"
```

这条代码是用于配置 Tomcat 服务器的启动参数，具体解释如下：

1. `-Dcom.sun.management.jmxremote`：启用 JMX（Java Management Extensions）远程管理功能。这允许外部监控程序（如JConsole或VisualVM）连接到Tomcat服务器并监视其状态和性能。

2. `-Dcom.sun.management.jmxremote.port=10028`：指定 JMX 远程管理的端口号为 10028。监控程序将使用该端口连接到Tomcat服务器。

3. `-Dcom.sun.management.jmxremote.authenticate=false`：禁用JMX远程管理的认证功能，允许任何可以连接到服务器的客户端都可以进行JMX操作。

4. `-Dcom.sun.management.jmxremote.ssl=false`：禁用JMX远程管理的SSL安全传输，以简化连接配置。在此配置下，连接不会通过SSL进行加密。

5. `-Djava.rmi.server.hostname=java69-matrix.zeus.lianjia.com`：指定 RMI（Remote Method Invocation）服务器的主机名或IP地址。在JMX远程管理中，这将用于通知监控程序Tomcat服务器的位置。

总之，该条配置代码设置了Tomcat服务器的JMX远程管理功能，指定了端口号为10028，禁用了认证和SSL安全传输，并指定了RMI服务器的主机名。

![1562242650648](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/1562242650648.png)



#### 4.5 取消JVM 的默认DNS缓存时间

不缓存DNS记录，避免DNS解析更改后要重启JVM虚拟机

```shell
# catalina.sh  ---添加如下内容
CATALINA_OPTS="$CATALINA_OPTS -Dsun.net.inetaddr.ttl=0 -Dsun.net.inetaddr.negative.ttl=0
```

1. `-Dsun.net.inetaddr.ttl=0`：这个系统属性设置了网络地址（InetAddress）的生存时间（TTL，Time-To-Live）为0。TTL用于指定网络数据包在网络中允许存在的时间。将TTL设置为0意味着数据包一旦到达目的地，即被丢弃，不会被路由到其他节点。在这个设置下，网络地址的生存时间被设置为尽可能短，可以避免一些不必要的网络传输。
2. `-Dsun.net.inetaddr.negative.ttl=0`：这个系统属性设置了负缓存的生存时间为0。负缓存用于缓存DNS查询的失败结果，以避免频繁地重新查询。将负缓存的生存时间设置为0意味着失败的DNS查询结果不会被缓存，每次查询都会重新进行。这可以确保Tomcat服务器及时获取到最新的DNS解析结果，而不会受到旧缓存的影响。

总之，这两个参数的设置有助于优化Tomcat服务器的网络通信性能，并确保它在与其他节点通信时能够及时获取到最新的DNS解析结果

![1562243085427](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/1562243085427.png)



### 5、JVM 运维实用排障工具

#### **5.1 jps**

​	`jps` 是 Java Virtual Machine Process Status Tool 的缩写，用于列出当前系统中正在运行的 Java 进程（Java虚拟机实例）。`jps` 工具在 JDK 的 `bin` 目录下，可以通过命令行运行。

```shell
用来查看Java进程的具体状态, 包括进程ID，进程启动的路径及启动参数等等，与unix上的ps类似，只不过jps是用来显示java进程，可以把jps理解为ps的一个子集。
常用参数如下:
# -q：只输出java进程pid
[root@java-tomcat01 ~]# jps -q
3267
3386

# -m：输出传递给main方法的参数，如果是内嵌的JVM则输出为null
[root@java-tomcat01 ~]# jps -m
3267 Bootstrap start
3398 Jps -m


# -l：输出完全的包名，应用主类名，jar的完全路径名
[root@java-tomcat01 ~]# jps -l
3410 sun.tools.jps.Jps
3267 org.apache.catalina.startup.Bootstrap

# -v：显示java服务启动时的相关参数和启动命令或脚本
[root@java-tomcat01 ~]# jps -v
3267 Bootstrap -Djava.util.logging.config.file=/usr/local/tomcat/conf/logging.properties -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djdk.tls.ephemeralDHKeySize=2048 -Djava.protocol.handler.pkgs=org.apache.catalina.webresources -Dorg.apache.catalina.security.SecurityListener.UMASK=0027 -Dignore.endorsed.dirs= -Dcatalina.base=/usr/local/tomcat -Dcatalina.home=/usr/local/tomcat -Djava.io.tmpdir=/usr/local/tomcat/temp
3422 Jps -Dapplication.home=/usr/local/jdk1.8.0_211 -Xms8m

注意: 使用jps 时的运行账户要和JVM 虚拟机启动的账户一致。若启动JVM虚拟机是运行的账户为www，那使用jps指令时，也要使用www 用户去指定。 sudo -u www jps
```



#### 5.2 jstack

```
jstack用于打印出给定的java进程ID或core file或远程调试服务的Java堆栈信息。如果现在运行的java程序呈现hung的状态，jstack是非常有用的。此信息通常在运维的过程中被保存起来(保存故障现场)，以供RD们去分析故障。
常用参数如下:
jstack <pid>
jstack [-l] <pid> //长列表. 打印关于锁的附加信息
jstack [-F] <pid> //当’jstack [-l] pid’没有响应的时候强制打印栈信息
```

Example

```shell
// 打印JVM 的堆栈信息，以供问题排查
[root@mouse03 ~]# jstack -F 38360 > /tmp/jstack.log
```



### 6、Tomcat安全优化

#### 6.1 telnet管理端口保护（强制）

| **类别**           | **配置内容及说明**                                           | **标准配置**                                      | **备注**                                                     |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------- | ------------------------------------------------------------ |
| telnet管理端口保护 | 1.修改默认的8005管理端口为不易猜测的端口（大于1024）；2.修改SHUTDOWN指令为其他字符串； | <Server port="**8527**" shutdown="**dangerous**"> | 1.以上配置项的配置内容只是建议配置，可以按照服务实际情况进行合理配置，但要求端口配置在**8000~8999**之间； |

#### 6.2  ajp连接端口保护（推荐）

| **类别**         | **配置内容及说明**                                           | **标准配置**                                    | **备注**                                                     |
| ---------------- | ------------------------------------------------------------ | ----------------------------------------------- | ------------------------------------------------------------ |
| Ajp 连接端口保护 | 1.修改默认的ajp 8009端口为不易冲突的大于1024端口；2.通过iptables规则限制ajp端口访问的权限仅为线上机器； | <Connector port="**8528**"protocol="AJP/1.3" /> | 以上配置项的配置内容仅为建议配置，请按照服务实际情况进行合理配置，但要求端口配置在**8000~8999**之间；；保护此端口的目的在于防止线下的测试流量被mod_jk转发至线上tomcat服务器； |

#### 6.3 降权启动（强制）

| **类别** | **配置内容及说明**                                           | **标准配置** | **备注**                                                     |
| -------- | ------------------------------------------------------------ | ------------ | ------------------------------------------------------------ |
| 降权启动 | 1.tomcat启动用户权限必须为非root权限，尽量降低tomcat启动用户的目录访问权限；2.如需直接对外使用80端口，可通过普通账号启动后，配置iptables规则进行转发； |              | 避免一旦tomcat 服务被入侵，黑客直接获取高级用户权限危害整个server的安全； |

```shell
[root@java-tomcat1 ~]# useradd tomcat
[root@java-tomcat1 ~]# chown tomcat.tomcat /usr/local/tomcat/ -R
[root@java-tomcat1 ~]# su -c '/usr/local/tomcat/bin/startup.sh' tomcat 
Using CATALINA_BASE:   /data/application/tomcat
Using CATALINA_HOME:   /data/application/tomcat
Using CATALINA_TMPDIR: /data/application/tomcat/temp
Using JRE_HOME:        /usr/local/java
Using CLASSPATH:       /data/application/tomcat/bin/bootstrap.jar:/data/application/tomcat/bin/tomcat-juli.jar
Tomcat started.
[root@java-tomcat1 ~]# ps -ef | grep tomcat 
tomcat     1065      1 64 20:33 ?        00:00:06 /usr/local/java/bin/java -Djava.util.logging.config.file=/data/applicationtomcat/conf/logging.properties -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djdk.tls.ephemeralDHKeySize=2048 -Djava.protocol.handler.pkgs=org.apache.catalina.webresources -Dorg.apache.catalina.security.SecurityListener.UMASK=0027 -Dignore.endorsed.dirs= -classpath /data/application/tomcat/bin/bootstrap.jar:/data/application/tomcat/bin/tomcat-juli.jar -Dcatalina.base=/data/application/tomcat -Dcatalina.home=/data/application/tomcat -Djava.io.tmpdir=/data/application/tomcat/temp org.apache.catalina.startup.Bootstrap start
root       1112   1027  0 20:33 pts/0    00:00:00 grep --color=auto tomcat
```

##### 

#### **6.4 起停脚本权限回收（推荐）**

| **类别**         | **配置内容及说明**                                           | **标准配置或操作**        | **备注**                             |
| ---------------- | ------------------------------------------------------------ | ------------------------- | ------------------------------------ |
| 起停脚本权限回收 | 去除其他用户对Tomcat的bin目录下shutdown.sh、startup.sh、catalina.sh的可执行权限； | chmod -R 744 tomcat/bin/* | 防止其他用户有起停线上Tomcat的权限； |

#### 

### 7、Tomcat性能优化

**上策：优化代码**

   该项需要开发经验足够丰富，对开发人员要求较高

**中策：jvm优化机制垃圾回收机制** **把不需要的内存回收**

优化jvm--优化垃圾回收策略

优化catalina.sh配置文件。在catalina.sh配置文件中添加以下代码

```shell
# tomcat分配1G内存模板
JAVA_OPTS="-Djava.awt.headless=true -Dfile.encoding=UTF-8 -server -Xms1024m -Xmx1024m -XX:NewSize=512m -XX:MaxNewSize=512m -XX:PermSize=512m -XX:MaxPermSize=512m"     

# 重启服务
su -c '/home/tomcat/tomcat8_1/bin/shutdown.sh' tomcat
su -c '/home/tomcat/tomcat8_1/bin/startup.sh' tomcat
```

**下策：加足够大的内存**

该项的资金投入较大

**下下策：每天0点定时重启tomcat**

使用较为广泛



### 8、扩展知识

WebSphere是 IBM 的软件平台。它包含了编写、运行和监视全天候的工业强度的随需应变 Web 应用程序和跨平台、跨产品解决方案所需要的整个中间件基础设施，如服务器、服务和工具。WebSphere 提供了可靠、灵活和健壮的软件。

WebLogic是美国Oracle公司出品的一个application server，确切的说是一个基于JAVAEE架构的中间件，WebLogic是用于开发、集成、部署和管理大型分布式Web应用、网络应用和数据库应用的Java应用服务器。将Java的动态功能和Java Enterprise标准的安全性引入大型网络应用的开发、集成、部署和管理之中。
