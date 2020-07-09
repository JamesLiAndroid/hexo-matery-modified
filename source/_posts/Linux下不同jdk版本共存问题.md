---
title: Linux下不同jdk版本共存问题
top: false
cover: false
toc: true
mathjax: true
date: 2020-07-09 21:53:29
password:
summary:
tags:
categories:
---

# Linux下不同jdk版本共存问题

## 问题场景

ElasticSearch7.6.2的安装需要java11支持，而目前服务器安装的是java8（jdk1.8），而且es安装在另一个用户下，独自运行。

而且es7.6在java8环境下无法进行启动！

这样需要根据不同用户切换不同的jdk版本。

## 使用alternative管理

首先，需要安装jdk11，直接通过yum进行安装

```
$  sudo yum list | grep java-11-openjdk
java-11-openjdk.x86_64                   1:11.0.7.10-4.el7_8            @updates
java-11-openjdk-headless.x86_64          1:11.0.7.10-4.el7_8            @updates
java-11-openjdk.i686                     1:11.0.7.10-4.el7_8            updates
java-11-openjdk-demo.i686                1:11.0.7.10-4.el7_8            updates
java-11-openjdk-demo.x86_64              1:11.0.7.10-4.el7_8            updates
java-11-openjdk-devel.i686               1:11.0.7.10-4.el7_8            updates
java-11-openjdk-devel.x86_64             1:11.0.7.10-4.el7_8            updates
java-11-openjdk-headless.i686            1:11.0.7.10-4.el7_8            updates
java-11-openjdk-javadoc.i686             1:11.0.7.10-4.el7_8            updates
java-11-openjdk-javadoc.x86_64           1:11.0.7.10-4.el7_8            updates
java-11-openjdk-javadoc-zip.i686         1:11.0.7.10-4.el7_8            updates
java-11-openjdk-javadoc-zip.x86_64       1:11.0.7.10-4.el7_8            updates
java-11-openjdk-jmods.i686               1:11.0.7.10-4.el7_8            updates
java-11-openjdk-jmods.x86_64             1:11.0.7.10-4.el7_8            updates
java-11-openjdk-src.i686                 1:11.0.7.10-4.el7_8            updates
java-11-openjdk-src.x86_64               1:11.0.7.10-4.el7_8            updates

$ sudo yum install java-11-openjdk.x86_64 -y

```

安装完成后，发现之前的java指向了java11，并非是原来的java8。

```
// 安装前
$ java -version
java version "1.8.0_202"
Java(TM) SE Runtime Environment (build 1.8.0_202-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.202-b08, mixed mode)

// 安装后
$ java -version
openjdk version "11.0.7" 2020-04-14 LTS
OpenJDK Runtime Environment 18.9 (build 11.0.7+10-LTS)
OpenJDK 64-Bit Server VM 18.9 (build 11.0.7+10-LTS, mixed mode, sharing)

```

这时候alternative工具就出场了，通过该命令来查看目前已经生效的java运行程序，如下：

```

$ sudo alternatives --config java

There is 1 program that provides 'java'.

  Selection    Command
-----------------------------------------------
*+ 1           java-11-openjdk.x86_64 (/usr/lib/jvm/java-11-openjdk-11.0.7.10-4.el7_8.x86_64/bin/java)

Enter to keep the current selection[+], or type selection number:


```

这时候新安装的jdk就生效了，也指向了其所在路径。这里可以添加多个jdk进行管理，将最开始安装的jdk导入查看，如下：

```
$ sudo alternatives --install /usr/bin/java java /usr/java/jdk1.8.0_202/bin/ 2

// 查看新增的jdk信息
$ sudo alternatives --config java

There are 2 programs which provide 'java'.

  Selection    Command
-----------------------------------------------
   1           java-11-openjdk.x86_64 (/usr/lib/jvm/java-11-openjdk-11.0.7.10-4.el7_8.x86_64/bin/java)
*+ 2           /usr/java/jdk1.8.0_202/bin/java

Enter to keep the current selection[+], or type selection number:

```

这时候我们可以进行切换操作，如下：

```
$ sudo alternatives --config java

There are 2 programs which provide 'java'.

  Selection    Command
-----------------------------------------------
*+ 1           java-11-openjdk.x86_64 (/usr/lib/jvm/java-11-openjdk-11.0.7.10-4.el7_8.x86_64/bin/java)
   2           /usr/java/jdk1.8.0_202/bin/java

Enter to keep the current selection[+], or type selection number: 2

```

输入2，选择新增的java执行信息，回车后即可生效。

```
$ sudo alternatives --config java

There are 2 programs which provide 'java'.

  Selection    Command
-----------------------------------------------
   1           java-11-openjdk.x86_64 (/usr/lib/jvm/java-11-openjdk-11.0.7.10-4.el7_8.x86_64/bin/java)
*+ 2           /usr/java/jdk1.8.0_202/bin/java

Enter to keep the current selection[+], or type selection number: 

// ctrl+c退出

$ java -version
java version "1.8.0_202"
Java(TM) SE Runtime Environment (build 1.8.0_202-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.202-b08, mixed mode)

```

已经切换到java8的信息了，说明切换已经生效了。

但是alternative工具，只能全局生效，不管是普通用户还是root用户，都是同一个java环境，并不能进行java环境的共存。

## 分而治之

首先从alternative工具中删除所有java环境，如下：

```

$ sudo alternatives --remove java /usr/java/jdk1.8.0_202/bin/java

$ sudo alternatives --remove java /usr/lib/jvm/java-11-openjdk-11.0.7.10-4.el7_8.x86_64/bin/java

$ sudo alternatives --config java
// 执行后无输出信息，说明删除完成

```

然后注释掉/etc/profile中java环境的配置，并使其生效：

```
$ sudo vim /etc/profile
// 拉到文档最后，注释掉配置的java环境信息
# # JAVA env
#JAVA_HOME=/usr/java/jdk1.8.0_202
#JRE_HOME=$JAVA_HOME/jre
#CLASS_PATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
#PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
#export JAVA_HOME JRE_HOME CLASS_PATH PATH

// :wq保存退出

// 使其生效
$ source /etc/profile

```

最后在不同的用户下，仿照/etc/profile中配置，配置不同的java环境

```
// 切换到centos用户下操作
$ su - centos
// 输入密码信息

[centos]$ vim ~/.bashrc
// 在最后追加
JAVA_HOME=/usr/java/jdk1.8.0_202
JRE_HOME=$JAVA_HOME/jre
CLASS_PATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
export JAVA_HOME JRE_HOME CLASS_PATH PATH

[centos]$ source ~/.bashrc

[centos]$ java -version
java version "1.8.0_202"
Java(TM) SE Runtime Environment (build 1.8.0_202-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.202-b08, mixed mode)

// 切换到es用户（elasticsearch7.6.2部署所在的用户）
$ su - es
// 输入密码信息

[es]$ vim ~/.bashrc
// 在最后追加
JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.7.10-4.el7_8.x86_64/
JRE_HOME=$JAVA_HOME/jre
CLASS_PATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
export JAVA_HOME JRE_HOME CLASS_PATH PATH

[es]$ source ~/.bashrc
openjdk version "11.0.7" 2020-04-14 LTS
OpenJDK Runtime Environment 18.9 (build 11.0.7+10-LTS)
OpenJDK 64-Bit Server VM 18.9 (build 11.0.7+10-LTS, mixed mode, sharing)

```

这样根据用户的不同可以使用不同版本的java环境，这样就能启动es了。
