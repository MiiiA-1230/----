# Git自研笔记

### 一、Git命令

**Git初始化配置**

```bash
# 查看用户
$ git config  user.name

# 查看邮箱
$ git config  user.email

# 查看配置
$ git config  --list
```

**配置全局用户名和邮箱**

```bash
# 配置用户
$ git config --global user.name "你的名字"

# 配置邮箱
$ git config --global user.email 你的邮箱

# 配置ssh密钥（windows环境）
$ ssh-keygen.exe -t rsa -C "你的邮箱"

# 查看Key
$ cat ~/.ssh/
id_rsa       id_rsa.pub   known_hosts
```

**常用命令**

```
# 克隆仓库
$ git clone git@gitee.com:rulaipo/git-test.git

# 上传暂存区
$ git add .

# 上传到本地仓库
$ git commit  -am "first commit" 
（a指所有已跟踪的文件，m指添加的描述，还可增加一个tag添加标签）

#推送到远程仓库
$ git push origin  master

#连接到远程仓库
$ git remote add origin git@gitee.com:rulaipo/git-test.git

#从远程仓库拉取代码（也可将代码进行重新同步）
$ git pull origin master
```

**Git基础命令**

```bash
# 克隆远程已有的仓库至本地
$ git clone 远程地址

# 更新本地代码（拉取远程仓库中的新代码到本地）
$ git pull <shortname> <remote_branch>

# 添加所有修改的文件，但不包含删除文件到暂存区
$ git add .

# 添加所有修改的文件以及删除的文件到暂存区
$ git add -A (--all)

# 添加单个或者多个文件/目录到暂存区
$ git add file1/dir1 file2/dir2

# 从文件从暂存区提交至本地仓库
# -a: 类似于 git add，但是不包含新添加的文件
# -m: 注释内容
$ git commit -am "提交信息"

# 把本地仓库的代码提交至远程仓库
git push <shortname> <remote_branch>
```

**Git分支命令**

```
# 创建分支（不常用）：
$ git branch NEW_BRANCH_NAME

# 切换分支：
$ git checkout BRANCH_NAME

# 创建分支并切换到新分支（常用）
$ git checkout -b NEW_BRANCH_NAME

# 查看本地仓库所有分支：
$ git branch

# 查看远程仓库的所有分支：
$ git branch -r

# 查看远程和本地所有分支：
$ git branch -a

# 同步新分支到远程仓库
$ git push origin NEW_BRANCH_NAME

# 删除本地分支
$ git branch -d BRANCH_NAME

# 强制本地删除分支
$ git branch -D BRANCH_NAME

# 删除远程仓库分支
$ git push origin --delete BRANCH_NAME

# 合并分支：
$ git merge BRANCH_NAME
```

### Gitb版本管理

```bash
# 查看仓库做了哪些更改
$ git log

#查看当前环境文件变更状态：
$ git status

# 查看修改了什么内容：
$ git diff FILE_NAME
	# - 删除了某行
	# + 添加的行
	
# 对比两个版本的差异：
$ git diff COMMIT_ID COMMIT_ID

# 撤销单个文件的修改：
$ git checkout -- FILE_NAME

#撤销所有文件的修改：
$ git reset --hard
```



### 二、git部署

```shell
环境：
    git-server    192.168.246.214  充当中心代码仓库服务器
    client        192.168.246.213

所有机器关闭防火墙和selinux

安装：所有机器都安装
   [root@git-server ~]# yum install -y git
   [root@git-server ~]# git --version 
   git version 1.8.3.1
   
准备：
    因为Git是分布式版本控制系统，所以，每个机器都必须注册：你的名字和Email地址。
    注意git config命令的--global参数，用了这个参数，表示你这台机器上所有的Git仓库都会使用这个配置。

所有的机器都添加，只要邮箱和用户不一样就可以。   
    # git config --global user.email "soho@163.com"     ----设置邮箱
    # git config --global user.name "soho"                   ----加添用户
    # cat /root/.gitconfig
    # git config --global color.ui true		#语法高亮
    # git config --list			#查看全局配置
```

#### git使用

**创建版本库:**

###### 1.创建一个空目录**：**在中心服务器上创建

```shell
[root@git-server ~]# mkdir /git-test
[root@git-server ~]# useradd git   #创建一个git用户用来运行git
[root@git-server ~]# passwd git  #给用户设置密码git
[root@git-server ~]# cd /git-test/
```

###### 2.通过git init命令把这个目录变成Git可以管理的仓库：

```shell
 第1种情况：可以改代码，还能上传到别人的机器，别人也能从你这里下载但是别人不能上传代码到你的机器上。
 第2种情况：只是为了上传代码用，别人从这台机器上下载代码也可以上传代码到这台机器上，经常用于核心代码库。
```

###### **创建裸库**：  适用于作为远程中心仓库使用

创建裸库才可以从别处push（传）代码过来，使用--bare参数

**git init --bare  裸库名字**

**创建——裸库**：

```shell
[root@git-server git-test]# git init --bare testgit
Initialized empty Git repository in /git-test/testgit/
[root@git-server ~]# chown git.git /git-test -R  #修改权限
2.仓库创建完成后查看库目录：
[root@git-server git-test]# cd testgit/
[root@git-server testgit]# ls
branches  config  description  HEAD  hooks  info  objects  refs
```

#### 1.客户端

```shell
1.配置免密登录
[root@client ~]# ssh-keygen    #生成秘钥
[root@client ~]# ssh-copy-id -i git@192.168.246.214   #将秘钥传输到git服务器中的git用户
2.克隆git仓库
[root@client ~]# yum install -y git
[root@client ~]# git clone git@192.168.246.214:/git-test/testgit/
Cloning into 'testgit'...
warning: You appear to have cloned an empty repository.
[root@client ~]# ls  #查看仓库已经克隆下来了
anaconda-ks.cfg    testgit
```

##### 创建文件模拟代码提交到仓库

```shell
1.在testgit目录下创建一个测试文件test.txt
[root@client ~]# cd testgit/
[root@client testgit]# vim test.txt   #随便写点东西

2.把文件添加到暂存区：使用 "git add" 建立跟踪
[root@client testgit]# git add test.txt
注: 这里可以使用 git add * 或者 git add -A

3.提交文件到仓库分支：
[root@client testgit]# git commit -m "test1"
[master (root-commit) 2b51ff9] test1
 1 file changed, 2 insertions(+)
 create mode 100644 test.txt
 -m:描述
 
 4.查看git状态：
[root@client testgit]# git status 
# On branch master   #分支位于master
5.修改文件后再此查看状态：
[root@client testgit]# echo '1122334' >> test.txt
[root@client testgit]# git status
# 位于分支 master
# 尚未暂存以备提交的变更：
#   （使用 "git add <file>..." 更新要提交的内容）
#   （使用 "git checkout -- <file>..." 丢弃工作区的改动）
#
#	修改：      readme.txt
#
修改尚未加入提交（使用 "git add" 和/或 "git commit "
6.先add
[root@client testgit]# git add -A
8.再次提交commit：
[root@client testgit]# git commit  -m "add2"
[master 73bf688] add2
 1 file changed, 1 insertion(+)
 [root@client testgit]# git status 
# On branch master
nothing to commit, working directory clean
```

#### 2、版本回退

已经提交了不合适的修改到版本库时，想要撤销本次提交，使用版本回退，不过前提是没有推送到远程库。

**查看现在的版本：**

```shell
[root@client testgit]# git log
显示的哪个版本在第一个就是当前使用的版本。
```

**版本回退(切换)：**
在Git中，上一个版本就是HEAD^，当然往上100个版本写100个比较容易数不过来，所以写成HEAD~100（一般使用id号来恢复）

##### 回到上一个版本

```shell
[root@client testgit]# git reset --hard HEAD^ 
HEAD is now at 0126755 test1
2.回到指定的版本(根据版本号): 
[root@client testgit]# git reset --hard dd66ff
HEAD is now at dd66ff9 add2
==========================================================
注：消失的ID号：
回到早期的版本后再查看git log会发现最近的版本消失，可以使用reflog查看消失的版本ID，用于回退到消失的版本
[root@vm20 gittest]# git reflog
2a85982 HEAD@{0}: reset: moving to 2a859821a2385e136fe83f3a206b287eb0eb8c18
f5bc8c1 HEAD@{1}: commit: test-version2
2a85982 HEAD@{2}: commit (initial): test-version1

[root@git-client testgit]# git reset --hard f5bc8c1
```

#### 3、**删除文件**

从工作区删除test.txt，并且从版本库一起删除

```shell
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
工作区：
[root@client testgit]# touch test.txt
[root@client testgit]# git status
# On branch master
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#
#       qf.txt
nothing added to commit but untracked files present (use "git add" to track)
[root@client testgit]# rm -rf test.txt  未添加到暂存区，可直接删除
[root@client testgit]# git status
# On branch master
nothing to commit, working directory clean

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
已从工作区提交到暂存区：
第一种方法
[root@client testgit]# touch test.txt
[root@client testgit]# git status
# On branch master
#
# Initial commit
#
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#
#       test.txt
nothing added to commit but untracked files present (use "git add" to track)

[root@client testgit]# git add test.txt
[root@client testgit]# git status
# On branch master
#
# Initial commit
#
# Changes to be committed:
#   (use "git rm --cached <file>..." to unstage)
#
#       new file:   test.txt
#

[root@client testgit]#  git rm --cache test.txt #从暂存区移除
rm 'test.txt'
[root@client testgit]# ls
test.txt
[root@client testgit]# git status
# On branch master
#
# Initial commit
#
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#
#       test.txt
nothing added to commit but untracked files present (use "git add" to track)
[root@client testgit]# rm -rf test.txt 
[root@client testgit]# git status
# On branch master
#
# Initial commit
#
nothing to commit (create/copy files and use "git add" to track)

第二种方法：
[root@client testgit]# touch  b.txt
[root@client testgit]# git add b.txt 
[root@client testgit]# git status
# On branch master
#
# Initial commit
#
# Changes to be committed:
#   (use "git rm --cached <file>..." to unstage)
#
#       new file:   b.txt
#
[root@client testgit]# git rm -f b.txt 
rm 'b.txt'
[root@client testgit]# ls
[root@client testgit]# git status
# On branch master
#
# Initial commit
#
nothing to commit (create/copy files and use "git add" to track)

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
直接在暂存区rm掉文件，如何解决
[root@client testgit]# touch c.txt
[root@client testgit]# git add c.txt 
[root@client testgit]# ls
c.txt
[root@client testgit]# git status
# On branch master
#
# Initial commit
#
# Changes to be committed:
#   (use "git rm --cached <file>..." to unstage)
#
#       new file:   c.txt
#
[root@client testgit]# rm -rf c.txt 
[root@client testgit]# git status
# On branch master
#
# Initial commit
#
# Changes to be committed:
#   (use "git rm --cached <file>..." to unstage)
#
#       new file:   c.txt
#
# Changes not staged for commit:
#   (use "git add/rm <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)
#
#       deleted:    c.txt
#
[root@client testgit]# git rm --cache c.txt
rm 'c.txt'
[root@client testgit]# ls
[root@client testgit]# git status
# On branch master
#
# Initial commit
#
nothing to commit (create/copy files and use "git add" to track)
[root@client testgit]# 
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
```

#### 4、修改文件

```shell
暂存区修改名称
[root@client testgit]# touch  a.txt
[root@client testgit]# git status
# On branch master
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#
#       a.txt
nothing added to commit but untracked files present (use "git add" to track)
[root@client testgit]# git add a.txt 
[root@client testgit]# git status
# On branch master
# Changes to be committed:
#   (use "git reset HEAD <file>..." to unstage)
#
#       new file:   a.txt
#
[root@client testgit]# git mv a.txt  d.txt
[root@client testgit]# git status
# On branch master
# Changes to be committed:
#   (use "git reset HEAD <file>..." to unstage)
#
#       new file:   d.txt
#
[root@client testgit]# ls
d.txt  test.txt
[root@client testgit]# git rm --cache d.txt
[root@client testgit]# rm -rf d.txt
```

#### 5、将代码上传到仓库的master分支

```shell
[root@client testgit]# vi a.txt   #创建一个新文件
hello world
[root@client testgit]# git add a.txt 
[root@client testgit]# git commit -m "add"
[root@client testgit]# git push origin master   #上传到中心仓库master分支
Counting objects: 11, done.
Compressing objects: 100% (4/4), done.
Writing objects: 100% (11/11), 828 bytes | 0 bytes/s, done.
Total 11 (delta 0), reused 0 (delta 0)
To git@192.168.246.214:/git-test/testgit/
 * [new branch]      master -> master
```

测试:

在客户端将仓库删除掉然后在克隆下来查看仓库中是否有文件

```shell
[root@client testgit]# cd
[root@client ~]# rm -rf testgit/
[root@client ~]# git clone git@192.168.246.214:/git-test/testgit/
Cloning into 'testgit'...
remote: Counting objects: 11, done.
remote: Compressing objects: 100% (4/4), done.
remote: Total 11 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (11/11), done.
[root@client ~]# cd testgit/
[root@client testgit]# ls
a.txt
[root@client testgit]# cat a.txt 
hello world
```

### 扩展：创建分支并合并分支

每次提交，Git都把它们串成一条时间线，这条时间线就是一个分支。截止到目前，只有一条时间线，在Git里，这个分支叫主分支，即`master`分支。`HEAD`严格来说不是指向提交，而是指向`master`，`master`才是指向提交的，所以，`HEAD`指向的就是当前分支。

在客户端操作：

```shell
[root@client ~]# git clone git@192.168.246.214:/git-test/testgit/
[root@client testgit]# git status 
# On branch master   #当前所在为master分支
#
# Initial commit
#
nothing to commit (create/copy files and use "git add" to track)
注意：刚创建的git仓库默认的master分支要在第一次commit之后才会真正建立。然后先git add .添加所有项目文件到本地仓库缓存，再git commit -m "init commit"提交到本地仓库，之后就可以随心所欲地创建或切换分支了。
创建分支:
[root@client testgit]# git branch dev   #创建分支。
[root@client testgit]# git branch    #查看分支。*在哪里就表示当前是哪个分支
  dev
* master
切换分支:
[root@client testgit]# git checkout dev
Switched to branch 'dev'
[root@client testgit]# git branch 
* dev
  master
在dev分支创建一个文件；
[root@client testgit]# vi test.txt
[root@client testgit]# git add test.txt 
[root@client testgit]# git commit -m "add dev"
[dev f855bdf] add dev
 1 file changed, 1 insertion(+)
 create mode 100644 test.txt
现在，dev分支的工作完成，我们就可以切换回master分支：
 [root@client testgit]# git checkout master
Switched to branch 'master'
```

切换回`master`分支后，再查看一个`test.txt`文件，刚才添加的内容不见了！因为那个提交是在`dev`分支上，而`master`分支此刻的提交点并没有变：

```shell
[root@client testgit]# ls
a.txt
```

现在，我们把`dev`分支的工作成果合并到`master`分支上：

```shell
[root@client testgit]# git merge dev
Updating 40833e0..f855bdf
Fast-forward
 test.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 test.txt
[root@client testgit]# ls
a.txt  test.txt
现在已经将dev分支的内容合并到master上。确认没有问题上传到远程仓库:
[root@client testgit]# git push origin master
```

`git merge`命令用于合并指定分支到当前分支。合并后，再查看`test.txt`的内容，就可以看到，和`dev`分支的最新提交是完全一样的。

合并完成后，就可以放心地删除`dev`分支了：

```shell
[root@client testgit]# git branch -d dev
Deleted branch dev (was f855bdf).
```

删除后，查看`branch`，就只剩下`master`分支了：

```shell
[root@client testgit]# git branch 
* master
```

### 三、搭建GitLab

```
#安装gitlab
[root@gitlab ~]#  yum  localinstall -y gitlab-ce-12.6.3-ce.0.el7.x86_64.rpm 

# 创建ssl证书存放路径
[root@gitlab ~]# mkdir /etc/gitlab/ssl
# 上传证书
[root@gitlab ssl]# ll /etc/gitlab/ssl/privkey.pem 
-rw-r--r--. 1 root root 241 6月   4 20:32 /etc/gitlab/ssl/privkey.pem
[root@gitlab ssl]# ll /etc/gitlab/ssl/fullchain.pem 
-rw-r--r--. 1 root root 3306 6月   4 20:32 /etc/gitlab/ssl/fullchain.pem

# 修改配置文件
[root@gitlab ~]#  egrep -v "^(#|$)" /etc/gitlab/gitlab.rb 
# 定义 GitLab 实例的外部访问 URL。用户通过此 URL 访问 GitLab 界面。
external_url 'https://gitlab.tanke.love'
# 设置 GitLab 的时区为 "Asia/Shanghai"。这影响到 GitLab 中时间显示的区域设置。
gitlab_rails['time_zone'] = 'Asia/Shanghai'
# 定义 Git 数据存储的路径。
git_data_dirs({
  "default" => {
    "path" => "/mnt/nfs-01/git-data"
   }
})
# 设置 GitLab Shell 使用的 SSH 端口。
gitlab_rails['gitlab_shell_ssh_port'] = 22
# 启用 Nginx，GitLab 使用 Nginx 作为 Web 服务器。
nginx['enable'] = true
# 设置最大请求体大小为 250MB
nginx['client_max_body_size'] = '250m'
# 启用 HTTP 到 HTTPS 的重定向，确保所有流量通过 HTTPS。
nginx['redirect_http_to_https'] = true
# 设置重定向端口为 80。
nginx['redirect_http_to_https_port'] = 80
# 指定 SSL 证书和密钥的位置，用于 HTTPS。
nginx['ssl_certificate'] = "/tmp/fullchain.pem"
nginx['ssl_certificate_key'] = "/tmp/privkey.pem"
nginx['ssl_ciphers'] = "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384"
nginx['ssl_prefer_server_ciphers'] = "off"
# 定义支持的 SSL/TLS 协议版本。
nginx['ssl_protocols'] = "TLSv1.2 TLSv1.3"
# 配置 SSL 会话缓存，提高性能。
nginx['ssl_session_cache'] = "shared:SSL:10m"
# 超时时间。
nginx['ssl_session_timeout'] = "1d"
# 启用 Gzip 压缩，提高传输效率。
nginx['gzip_enabled'] = true

# 重新加载配置
[root@gitlab ~]# gitlab-ctl  reconfigure
```

查看密码

```
[root@localhost ~]# cat /etc/gitlab/initial_root_password 

# 启动实例
[root@localhost ~]# gitlab-ctl start
```

启动后访问DNS域名即可登录

### 四、搭建CI/CD平台

#### 1、搭建Jenkins自动发布

##### 服务安装

```
tar -xvzf apache-maven-3.8.2-bin.tar.gz
[root@jenkins local]# tar -xvzf apache-tomcat-8.5.70.tar.gz
[root@jenkins local]# tar -xvzf openjdk-11+28_linux-x64_bin.tar.gz
[root@jenkins local]# mv jdk-11/ java
[root@jenkins local]# mv apache-tomcat-8.5.70 tomcat
[root@jenkins local]# rm -rf tomcat/webapps/*
[root@jenkins local]# mv apache-maven-3.8.2 maven
[root@jenkins ~]# cp jenkins.war  /usr/local/tomcat/webapps/
```

##### 环境配置

```
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

##### 访问登录

默认账号：admin

默认密码：$（cat ~/.jenkins/secrets/initialAdminPassword ）

##### ssh传递

部署 Jenkins 服务时，通常进行与 GitLab 的集成，以实现自动化构建和持续集成的需求。在这种情况下，需要确保 Jenkins 服务器具有访问 GitLab 仓库的权限。为了实现这一点，可以遵循以下步骤：

1. 在 Jenkins 服务器端生成 SSH 密钥对（公钥和私钥），可以使用命令 `ssh-keygen` 生成密钥对。
2. 在 Jenkins 服务器上将公钥添加到 Jenkins 用户的 SSH 密钥或全局凭证中。
3. 将 Jenkins 用户的公钥添加到 GitLab 的用户 SSH 密钥中，或者将该公钥关联到 GitLab 项目的 Deploy Key 中。
4. Jenkins 服务通过使用私钥访问 GitLab 仓库。

##### 添加后端服务器

```
# 公钥发送到后端服务器，才能实现免密；
[root@jenkins ~]# ssh-copy-id -i root@192.168.153.194
```

##### 配置JDK和Maven

虽然Jenkins服务器上，已经安装了JDK和maven工具，但是，还需要在Jenkins服务中，进行配置；

这样Jenkins才能自动化的使用两个工具：

Manage Jenkins——>Tools——>JDK installations

​                                      ——>Maven installations

##### 构建发布任务

根据开发人员给的文件和参数进行手动构建
