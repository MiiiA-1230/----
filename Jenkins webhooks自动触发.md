# Jenkins+Gitlab webhook触发自动构建项目
- 效果：只要Gitlab仓库代码更新，Jenkins自动拉取代码，自动完成构建任务。无需手动点击“立即构建”或者"参数化构建"
- 需求场景：

1、项目代码的更新迭代较多，运维很有可能不在场，每次点击比较麻烦
2、更新的可能不是代码，可能是一些资源（比如：静态文件等）
Jenkins版本：2.303.1 Gitlab版本：12.6.3

### 安装配置Gitlab
Yum安装即可 ， 过程 略
Gitlab平台root用户密码配置为12345678
##### 创建一个项目(私有仓库)
准备测试代码：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339450086-a5163726-3591-49fb-ab72-99eaedb19147.png#averageHue=%23f8f8f7&clientId=u906c2411-8505-4&from=paste&id=u0ea14367&originHeight=764&originWidth=1410&originalType=url&ratio=1&rotation=0&showTitle=false&size=80118&status=done&style=none&taskId=ua1308608-2d59-4ea6-9bc4-95e56016842&title=)
### 后端服务器准备
```shell
[root@docker-server ~]# tar -xvzf jdk-8u211-linux-x64.tar.gz  -C /usr/local/
[root@docker-server ~]# cd /usr/local/
[root@docker-server local]# mv jdk1.8.0_211/ java 
[root@docker-server local]# vim /etc/profile
#最文件最后面添加
JAVA_HOME=/usr/local/java
PATH=$JAVA_HOME/bin:$PATH

[root@docker-server local]# source /etc/profile
[root@docker-server local]# java -version
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1668657965540-c9e2d959-1187-43df-b9ef-ddd1bee4cede.png#averageHue=%23110f0d&clientId=u41b33c8d-0e66-4&from=paste&height=138&id=u4d5d4339&originHeight=172&originWidth=1077&originalType=binary&ratio=1&rotation=0&showTitle=false&size=19231&status=done&style=none&taskId=u2563c73e-477e-4119-bd5e-641d263bc45&title=&width=861.6)

```shell
[root@docker-server ~]# tar -xvzf apache-tomcat-8.5.45.tar.gz -C /data/application/
[root@docker-server application]# mv apache-tomcat-8.5.45/ tomcat
[root@docker-server application]# ls
tomcat
[root@docker-server application]# rm -rf tomcat/webapps/*
```
### 安装配置Jenkins

---


安装过程略
1.安装jdk
2.安装Tomcat
3.安装Maven（可选，不确定是否编译）
4.配置环境变量
5.启动

---


##### 记得配置jdk和maven
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339450493-8b5749b0-cde5-4aab-b5f2-1a6ef40a9909.png#averageHue=%23fcf9f8&clientId=u906c2411-8505-4&from=paste&id=uaf1a0dd1&originHeight=559&originWidth=1391&originalType=url&ratio=1&rotation=0&showTitle=false&size=49698&status=done&style=none&taskId=u2630baa1-b1ae-4d23-9564-a817a95abd4&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339450471-4e2975d3-907d-48a0-8ab3-7cede4a8e63e.png#averageHue=%23fcf8f7&clientId=u906c2411-8505-4&from=paste&id=ucd969ba5&originHeight=338&originWidth=1384&originalType=url&ratio=1&rotation=0&showTitle=false&size=27130&status=done&style=none&taskId=uccd2f5f8-02ec-4e5e-b0d4-fc7850fb24a&title=)
##### 安装Gitlab hooks plugins插件
因为要用gitlab hook自动拉取代码的功能，需要安装GItlab hooks插件，才具有自动构建的功能
去“插件管理”页面，“可选插件”，搜索“Gitlab Hook Plugin”，“Gitlab”，点击“直接安装即可”
**注意：安装过程和网络有关。网络必须顺畅。且能正常连同国外Jenkins网站，才能下载成功**
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339451257-03a9e617-4080-487c-9ef3-6eb31a83e03b.png#averageHue=%23f7f4f3&clientId=u906c2411-8505-4&from=paste&id=uec338be8&originHeight=396&originWidth=1429&originalType=url&ratio=1&rotation=0&showTitle=false&size=42412&status=done&style=none&taskId=u51a6ed98-1ca1-4f79-bca8-0b74c5b3227&title=)
##### 新建Gitlab webhook相关项目
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339451477-659d9eb1-cbeb-4103-aa9c-9148c97deefb.png#averageHue=%23fdfbfa&clientId=u906c2411-8505-4&from=paste&id=u5579bbda&originHeight=357&originWidth=482&originalType=url&ratio=1&rotation=0&showTitle=false&size=56892&status=done&style=none&taskId=uf467cd17-d756-4ecc-86a6-c7c626fecd5&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339451581-049775dc-c2d3-4896-aa6b-a9a9555035c4.png#averageHue=%23f2efee&clientId=u906c2411-8505-4&from=paste&id=u2952ff74&originHeight=741&originWidth=1182&originalType=url&ratio=1&rotation=0&showTitle=false&size=108445&status=done&style=none&taskId=u73c397ac-e471-4270-8b1c-47f43891084&title=)
Jenkins具体配置
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339451724-29d99afa-cd2c-4804-afc9-30bc3fc183e6.png#averageHue=%23f9f9f9&clientId=u906c2411-8505-4&from=paste&id=u97304910&originHeight=791&originWidth=1238&originalType=url&ratio=1&rotation=0&showTitle=false&size=61208&status=done&style=none&taskId=uece6411e-b5bd-4b15-9d39-73cd75a3c1d&title=)
来到Gitlab的test1项目中，复制拉取地址
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339451783-1ad11f51-3850-4b7b-9172-aa55d79c1202.png#averageHue=%23f4f2f0&clientId=u906c2411-8505-4&from=paste&id=u336e5830&originHeight=587&originWidth=1209&originalType=url&ratio=1&rotation=0&showTitle=false&size=72759&status=done&style=none&taskId=u3890b92b-f3fd-44ab-a2d3-03bd3a1dde2&title=)
粘贴到
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339452308-7ab21409-79fc-4f1b-a0c4-1b43dbf958cd.png#averageHue=%23f0eeee&clientId=u906c2411-8505-4&from=paste&id=u90026910&originHeight=384&originWidth=665&originalType=url&ratio=1&rotation=0&showTitle=false&size=81473&status=done&style=none&taskId=u91687a63-90c7-4783-8deb-ed083fb2b61&title=)
出现一堆报红，正常！因为需要配置私钥和公钥
需要把Jenkins服务器的私钥，配置到test1项目中。把Jenkins服务器的公钥，配置到GItlab的服务里面。
这样拉取就可以免密了！
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339452330-0558eb84-97cd-4798-9960-6a7361e52a94.png#averageHue=%23f6f4f3&clientId=u906c2411-8505-4&from=paste&id=u8baece68&originHeight=624&originWidth=1377&originalType=url&ratio=1&rotation=0&showTitle=false&size=58621&status=done&style=none&taskId=ua231ac47-f96c-4eb9-a34f-b050d49f1a7&title=)
```shell
[root@jenkins-server ~]# useradd jenkins 
[root@jenkins-server ~]# su - jenkins 
[jenkins@jenkins-server ~]$ ssh-keygen 
[jenkins@jenkins-server ~]$ cat .ssh/id_rsa #查看jenkins用户的私钥
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339452630-bcb3e967-f73e-491c-9560-6548e57cf3d7.png#averageHue=%23f2f2f2&clientId=u906c2411-8505-4&from=paste&id=u3e67ff06&originHeight=168&originWidth=701&originalType=url&ratio=1&rotation=0&showTitle=false&size=26907&status=done&style=none&taskId=uff4d170a-8b99-44a1-a528-dfa803e09e3&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339452704-d7fadc16-7915-4eb9-92f4-0fa260f2a263.png#averageHue=%23f1efef&clientId=u906c2411-8505-4&from=paste&id=uec4fb4da&originHeight=445&originWidth=640&originalType=url&ratio=1&rotation=0&showTitle=false&size=80700&status=done&style=none&taskId=ud53d52df-5ef9-46fa-8ba7-798134dd06e&title=)
看到仍然报红，将jenkins服务器上面的jenkins用户的公钥添加到gitlab中
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339452746-df4fd441-7651-4747-afe0-ecd535e7b1f5.png#averageHue=%23f8f7f7&clientId=u906c2411-8505-4&from=paste&id=uef93001f&originHeight=364&originWidth=352&originalType=url&ratio=1&rotation=0&showTitle=false&size=17535&status=done&style=none&taskId=u89b0491e-45b0-4fb6-85b9-954fa5d8ac7&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339453366-abb0080e-1cb3-4b37-9a9b-dd6ec542138a.png#averageHue=%23f8f7f6&clientId=u906c2411-8505-4&from=paste&id=ucc4739bd&originHeight=731&originWidth=376&originalType=url&ratio=1&rotation=0&showTitle=false&size=27394&status=done&style=none&taskId=ud2e08a0d-2a04-4074-aa59-01a7d3c96dc&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339453499-a98b031a-9d85-483f-8ed2-d4d31a6d3ed7.png#averageHue=%23fdfdfd&clientId=u906c2411-8505-4&from=paste&id=uf11af350&originHeight=372&originWidth=639&originalType=url&ratio=1&rotation=0&showTitle=false&size=62599&status=done&style=none&taskId=u7173c5fa-9dfa-4608-aa6a-4f58ea5c259&title=)
登录到jenkins服务器中
```shell
[jenkins@jenkins-server ~]$ cat .ssh/id_rsa.pub #查看jenkins用户的公钥
```

![	](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339453533-430970e4-ad4f-4de0-be5a-222b39987336.png#averageHue=%23fcfcfb&clientId=u906c2411-8505-4&from=paste&id=uc9a106a7&originHeight=357&originWidth=679&originalType=url&ratio=1&rotation=0&showTitle=false&size=108115&status=done&style=none&taskId=u8d374270-1da8-4d2e-b356-ea18250588e&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339453540-f7318326-a77c-453b-bd24-2d91e9e77fd3.png#averageHue=%23f2f0f0&clientId=u906c2411-8505-4&from=paste&id=uf8b8e17d&originHeight=635&originWidth=1154&originalType=url&ratio=1&rotation=0&showTitle=false&size=56319&status=done&style=none&taskId=u788919d7-2216-4ff0-a23a-802ffc58ae4&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339454024-f8f61b4c-ec46-417f-bd59-a3a89927a1bc.png#averageHue=%23f8f6f6&clientId=u906c2411-8505-4&from=paste&id=u595c5040&originHeight=612&originWidth=1384&originalType=url&ratio=1&rotation=0&showTitle=false&size=56224&status=done&style=none&taskId=uae5c2893-4672-46ce-a7d2-7467d1641cd&title=)
##### 构建触发器
![image-20240606223324696](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406062233805.png)

![image-20240606223549283](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406062235372.png)

![image-20240606231753843](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406062317898.png)
![image-20240606231934059](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406062319107.png)

> Token值作用：
> 		Jenkins Webhook Token值用于身份验证，确保只有拥有正确Token的服务器才能触发Jenkins上的构建或者部署等操作。这样做可以防止未经授权的用户触发Jenkins作业，从而保护你的Jenkins从脚本未授权访问。

![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339455074-c91f4733-2eb4-4b78-92e7-d6b59387f49b.png#averageHue=%23f8f8f8&clientId=u906c2411-8505-4&from=paste&id=u41b5bb26&originHeight=788&originWidth=1219&originalType=url&ratio=1&rotation=0&showTitle=false&size=58281&status=done&style=none&taskId=u9d538894-0503-4cfa-9ae6-11114fb88d4&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1671003567422-b95fb9a9-7222-49c5-85df-bf3d2384a5b8.png#averageHue=%23f9f9f9&clientId=uf10e4cac-5303-4&from=paste&height=304&id=u788c6bb5&originHeight=380&originWidth=1192&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13651&status=done&style=none&taskId=ud8968b42-99cd-422c-b30b-48dcec89143&title=&width=953.6)
**要记录下上边的URL和认证密钥，切换到gitlab，找到对应的git库点击setting --> webhook ,填写以下内容**
地址：[http://192.168.182.130:8080/jenkins/project/test3](http://192.168.182.130:8080/jenkins/project/test3)
Secret token：aab74fc9bd2427f6989e7a2b8ab9b178

##### 配置GItlab
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339455002-72def8cc-e155-4be7-96a1-d48adeb01e86.png#averageHue=%23f9f6f5&clientId=u906c2411-8505-4&from=paste&id=uec188b8b&originHeight=786&originWidth=1676&originalType=url&ratio=1&rotation=0&showTitle=false&size=118689&status=done&style=none&taskId=ua1d658a8-4a02-4375-849c-a854cdaf10f&title=)
添加完成之后报错
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339455454-3c93b938-d71d-402d-95c2-dae1ad91ae79.png#averageHue=%23faeae7&clientId=u906c2411-8505-4&from=paste&id=u19aebdcc&originHeight=53&originWidth=674&originalType=url&ratio=1&rotation=0&showTitle=false&size=7021&status=done&style=none&taskId=u863da91e-68e3-4b27-bfa7-4855aff541b&title=)
这是因为gitlab 10.6 版本以后为了安全，不允许向本地网络发送webhook请求，设置如下：
登录管理员账号

![image-20240606234101160](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406062341224.png)

![image-20240606234132358](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406062341409.png)

![image-20240606234156326](https://hjmimage.oss-cn-zhangjiakou.aliyuncs.com/202406062341394.png)



![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339455584-c1be02e3-5c38-4738-a6b0-1ad3ab6fbd76.png#averageHue=%23e9e5c4&clientId=u906c2411-8505-4&from=paste&id=ud7b03f9b&originHeight=212&originWidth=573&originalType=url&ratio=1&rotation=0&showTitle=false&size=17612&status=done&style=none&taskId=uc716071e-ea7c-48db-af8f-f01652ff4ca&title=)

![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339456434-7d2a65a8-2225-4c29-b488-b627e60e2551.png#averageHue=%23fbf8f4&clientId=u906c2411-8505-4&from=paste&id=u59efccd8&originHeight=775&originWidth=1232&originalType=url&ratio=1&rotation=0&showTitle=false&size=75161&status=done&style=none&taskId=u16c34172-3a19-400a-bedd-4b0b4c9c655&title=)
然后需要再次添加webhook，就会成功了。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339456536-869ac280-ee9d-4e45-98bf-dc4ac782a015.png#averageHue=%23fcfbfb&clientId=u906c2411-8505-4&from=paste&id=u1b2b965d&originHeight=506&originWidth=1061&originalType=url&ratio=1&rotation=0&showTitle=false&size=38566&status=done&style=none&taskId=u68f83852-ddc7-4660-ae96-1a89258364c&title=)
成功了，才会显示出来
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339456579-e153c650-644c-4d2f-9c03-ccdf1581aa00.png#averageHue=%23fbfaf9&clientId=u906c2411-8505-4&from=paste&id=u000f4a31&originHeight=711&originWidth=1052&originalType=url&ratio=1&rotation=0&showTitle=false&size=68745&status=done&style=none&taskId=ud2b4065e-6c4f-4d0d-acb7-7e21f36a1da&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339456667-b6f1ab95-d289-4ec7-a3ad-b5399ba20057.png#averageHue=%23eacc9c&clientId=u906c2411-8505-4&from=paste&id=u03c11b64&originHeight=223&originWidth=876&originalType=url&ratio=1&rotation=0&showTitle=false&size=15348&status=done&style=none&taskId=u15d6ea27-06c3-4372-b2b0-1225bae68c8&title=)
**回到jenkins页面**
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339456889-488c1d9d-e4c1-4d63-b868-c8264ab8ee61.png#averageHue=%23f6f5f5&clientId=u906c2411-8505-4&from=paste&id=u4d8f0741&originHeight=361&originWidth=822&originalType=url&ratio=1&rotation=0&showTitle=false&size=18623&status=done&style=none&taskId=ua2338ba4-f4b5-43aa-a869-4daac212eda&title=)
注意：Jenkins需要配置git用户名 和 邮箱地址

```shell
[root@git-client ~]# su - jenkins
[root@git-client ~]# git config --global user.email "soho@163.com"
[root@git-client ~]# git config --global user.name "soho"
```
##### 开始测试
在任何一台测试都可以。我这里在gitlab机器上面测试： 
```shell
[root@git-client ~]# yum -y install git
[root@git-client ~]# git config --global user.email "fei@163.com"
[root@git-client ~]# git config --global user.name "fei"
[root@git-client ~]# ssh-keygen #生成秘钥 
[root@git-client ~]# cat .ssh/id_rsa.pub #查看生成的公钥添加到gitlab里面去
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339457522-e0dc8b3c-4cd9-4dec-81ff-9526fca3a1f4.png#averageHue=%23fcf9f8&clientId=u906c2411-8505-4&from=paste&id=u9473ed30&originHeight=655&originWidth=1312&originalType=url&ratio=1&rotation=0&showTitle=false&size=79132&status=done&style=none&taskId=u1d4bd9cc-a9e6-4a97-b876-a353a31c62c&title=)
先克隆一下仓库 
```shell
[root@git-client ~]# git clone git@192.168.91.168:root/cloudweb.git
[root@git-client ~]# cd cloudweb
[root@gitlab-server cloudweb]# vi src/main/webapp/index.jsp
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339457611-c7cd98b1-81b3-43c5-b131-6f7150447207.png#averageHue=%230b0807&clientId=u906c2411-8505-4&from=paste&id=uaed1fbaf&originHeight=471&originWidth=1148&originalType=url&ratio=1&rotation=0&showTitle=false&size=57180&status=done&style=none&taskId=ub6d5ec08-2b3c-4b51-b844-da79b2c366c&title=)
```shell
[root@git-client cloudweb]# git add *
[root@git-client cloudweb]# git commit -m "用户名 和 密码"
[root@git-client cloudweb]# git push origin main
```

**返回到jenkins页面查看是否自动发布**
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339457837-da51c7ad-d30b-47df-a527-33a19f6ec5f1.png#averageHue=%23f9f7f7&clientId=u906c2411-8505-4&from=paste&id=u660d37c6&originHeight=735&originWidth=830&originalType=url&ratio=1&rotation=0&showTitle=false&size=83684&status=done&style=none&taskId=uead1a787-5533-4fba-8fcc-64aa8630aec&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339457884-662e646b-17aa-4da0-b8d1-80a81bacf6c4.png#averageHue=%23f8f8f8&clientId=u906c2411-8505-4&from=paste&id=uf5c6d881&originHeight=814&originWidth=1154&originalType=url&ratio=1&rotation=0&showTitle=false&size=58919&status=done&style=none&taskId=u855574ed-2c84-4c67-864f-91e7126887b&title=)
```shell
[root@jenkins easy-springmvc-maven]# cd /root/.jenkins/workspace/test3
[root@jenkins test3]# ls
pom.xml  README.md  src  target
```
访问tomcat页面，验证：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339457945-39707c2f-1640-4b42-b939-da12b60842f4.png#averageHue=%23fbf4f3&clientId=u906c2411-8505-4&from=paste&id=u14aa3013&originHeight=305&originWidth=876&originalType=url&ratio=1&rotation=0&showTitle=false&size=32046&status=done&style=none&taskId=u5be31ed1-73a7-43c8-aba1-fb10902abea&title=)
##### 基于Git参数化自动构建项目
Gitlab仓库中有测试项目：github同步过来的
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339458793-75860ac5-6ced-48ad-88ba-255ae90736ce.png#averageHue=%23f9f8f8&clientId=u906c2411-8505-4&from=paste&id=u6fd87062&originHeight=799&originWidth=1664&originalType=url&ratio=1&rotation=0&showTitle=false&size=106320&status=done&style=none&taskId=u7c287d42-8c3e-4ec2-ac8f-8943f367541&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339458960-29b3f694-a3e6-4287-be24-1836ead55d1f.png#averageHue=%23f9f8f7&clientId=u906c2411-8505-4&from=paste&id=u8ad800bf&originHeight=850&originWidth=1709&originalType=url&ratio=1&rotation=0&showTitle=false&size=130778&status=done&style=none&taskId=u6b010326-64c7-482e-8abb-abf7753bedb&title=)
Jenkins配置：
修改原来的"test3"项目：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339458921-bb9fd087-b3e9-42e1-8b26-6a5015b4497a.png#averageHue=%23f9f8f8&clientId=u906c2411-8505-4&from=paste&id=uab5ba6f1&originHeight=697&originWidth=1380&originalType=url&ratio=1&rotation=0&showTitle=false&size=55227&status=done&style=none&taskId=u859da46f-5cdd-440f-96d8-6ebeda874c0&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1671004781816-03fd5b23-070b-4ab1-8659-51181c036fb5.png#averageHue=%23f9f8f8&clientId=uf10e4cac-5303-4&from=paste&height=558&id=uc427b370&originHeight=698&originWidth=1227&originalType=binary&ratio=1&rotation=0&showTitle=false&size=39176&status=done&style=none&taskId=ufae10fdf-4362-4747-88e7-84f29390d2d&title=&width=981.6)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339458944-e8f8d880-5157-4679-a864-12b290e77111.png#averageHue=%23f8f8f8&clientId=u906c2411-8505-4&from=paste&id=u25ba60a4&originHeight=319&originWidth=1230&originalType=url&ratio=1&rotation=0&showTitle=false&size=20207&status=done&style=none&taskId=ua7e03506-261b-44c6-99eb-92466249ea0&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339459581-8f071137-bb20-49df-81e1-14b017130b17.png#averageHue=%23f8f8f8&clientId=u906c2411-8505-4&from=paste&id=u36572c38&originHeight=542&originWidth=1210&originalType=url&ratio=1&rotation=0&showTitle=false&size=40524&status=done&style=none&taskId=u9158ebf4-ff9f-4e86-b48f-940adf3d9a9&title=)
![image.png](https://cdn.nlark.com/yuque/0/2022/png/23214851/1652339459775-2f398091-f525-43dd-9bc3-9c3b614f242e.png#averageHue=%23f9f8f8&clientId=u906c2411-8505-4&from=paste&id=uf61f52e5&originHeight=752&originWidth=1377&originalType=url&ratio=1&rotation=0&showTitle=false&size=66157&status=done&style=none&taskId=u570ab265-1e87-46ab-8b03-e367300fd4a&title=)
测试：
推送代码，打tag。代码也会自动构建；
