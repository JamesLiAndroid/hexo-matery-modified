---
title: 使用lanproxy在vpn环境下进行微服务的开发联调测试
top: false
cover: false
toc: true
mathjax: true
date: 2021-05-26 13:27:34
password:
summary:
tags: DevOps
categories:
---

# 使用lanproxy在vpn环境下进行微服务的开发联调测试

## 背景

在最近的项目中，我们需要通过vpn连接到公司内网进行开发。在启动vpn后，将工作机本地的服务启动，注册到nacos上。由于我们自己本机的ip地址发生了变化，在服务注册中心nacos中，发现我们自己的服务地址变更为vpn的地址，导致在其它服务轮询调用我们本机的服务时，出现了接口超时、服务调用不到的错误。这样就导致我们的开发工作受到阻碍，不能方便的进行联调。

同样在客户现场，我们也是通过vpn连接到客户现场的环境中，进行开发测试，由于vpn分配的ip地址到本机启动的服务上，无法被客户现场服务器中部署的服务调用到，造成了开发困难的情况。

这两种场景的不同在于，公司内网的服务器可以访问外部网路，而客户现场的服务器无法访问外部网络。

## 解决方式

面对上述问题，直接考虑使用内网穿透（端口映射）的方式解决。简单地说，就是将本地端口映射到一个服务器上的端口，同一个网段上的服务器是互通的，通过在该网段内的服务器访问该端口，映射到本地端口，实现对本地端口的访问。这样就可以使服务以服务器的地址注册到nacos上，由网关或者OpenFeign进行服务调用时，可以访问到本地启动的微服务。示意图如下：

![](LanProxy端口映射示意图.png)

解决方案则选择使用[LanProxy]()，部署LanProxy服务端，在开发机启动LanProxy客户端，进行内部组网。通过面板创建客户端连接，配置端口映射，实现内网穿透。

## 服务器部署

下载lanproxy服务端，从[这个地址](file.nioee.com/d/2e81550ebdbd416c933f/)进行下载，上传到服务器，部署如下：

```
// 解压服务端
$ unzip proxy-server-0.1.zip 

// 配置启动信息
$ cd proxy-server-0.1

$ vim conf/config.properties 
// 启动项修改如下，由于开放端口只由55500-55599，所以将端口号限制在这个范围中
server.bind=0.0.0.0
server.port=55590

server.ssl.enable=true
server.ssl.bind=0.0.0.0
server.ssl.port=55593
server.ssl.jksPath=test.jks
server.ssl.keyStorePassword=123456
server.ssl.keyManagerPassword=123456
server.ssl.needsClientAuth=false

config.server.bind=0.0.0.0
config.server.port=55591
config.admin.username=admin
config.admin.password=@WSXzaq1

// :wq保存退出

// 启动服务如下
$ ./bin/startup.sh 
Starting the proxy server ...started
PID: 27660

// 查看进程和端口号
$ ps -ef | grep lanproxy

$ sudo netstat -tnlp | grep 5559*
tcp        0      0 0.0.0.0:55590           0.0.0.0:*               LISTEN      27660/java          
tcp        0      0 0.0.0.0:55591           0.0.0.0:*               LISTEN      27660/java          
tcp        0      0 0.0.0.0:55593           0.0.0.0:*               LISTEN      27660/java        

```

注意：启动服务端需要java运行环境，没有的需要自行安装一下！

这样直接访问http://ip:55591即可，访问情况如下：

![](首页访问.png)

输入用户名密码访问即可登录。

## 工作机开发

### 0. （可选）域名设置

如果线上服务器可以访问外网，则可以配置域名来进行访问。在线上服务器利用域名访问端口映射后的服务信息。

如果线上服务器不能访问外网，可以考虑将域名配置到/etc/hosts文件中，或者直接使用该服务器的ip地址。

### 1. 设置client信息

点击New Client，添加一个新的client信息，如下图：

![](添加新的client.png)

点击下面新的client信息，如下：

![](添加新的proxy-config.png)

添加端口信息的映射，如下：

![](添加端口映射.png)

注意这里需要，将最后一个Backend ID映射为本地的微服务所在的端口。这样在访问的时候就能访问到开发机上的端口信息。

**注意：**在每一个新开服务时都需要在这里新增端口信息。

### 2. 开发机启动lanproxy客户端

在开发机[下载]()并启动lanproxy的客户端，使用带管理员权限的命令行进行启动如下：

```
$ .\client_windows_amd64.exe -s 192.168.245.22 -p 55590 -k 04d60ab81091446b817f98aff38f9961


```

命令解释如下：

* -s lanproxy所部署的服务器ip地址或者你自行添加的域名信息
* -p config.properties中配置的server.port
* -k 在创建client时生成的密钥信息

这里临时不使用ssl的方式启动。如果需要可以参考[lanproxy](https://github.com/ffay/lanproxy)的内容。

普通端口连接
```
# mac 64位
nohup ./client_darwin_amd64 -s SERVER_IP -p SERVER_PORT -k CLIENT_KEY &

# linux 64位
nohup ./client_linux_amd64 -s SERVER_IP -p SERVER_PORT -k CLIENT_KEY &

# windows 64 位
./client_windows_amd64.exe -s SERVER_IP -p SERVER_PORT -k CLIENT_KEY

```

SSL端口连接

```
# mac 64位
nohup ./client_darwin_amd64 -s SERVER_IP -p SERVER_SSL_PORT -k CLIENT_KEY -ssl true &

# linux 64位
nohup ./client_linux_amd64 -s SERVER_IP -p SERVER_SSL_PORT -k CLIENT_KEY -ssl true &

# windows 64 位
./client_windows_amd64.exe -s SERVER_IP -p SERVER_SSL_PORT -k CLIENT_KEY -ssl true

```

### 3. 启动本地微服务注册到线上服务器

在intellij idea中启动本地微服务，在启动之前需要做以下设置，如图：

![](设置启动参数.png)

参数信息如下：

* -Dspring.cloud.client.ip-address=192.168.245.22

这个信息设置了该服务启动时所属的ip信息，也可以是域名。

这样在注册到nacos的时候，该服务的地址就变为上面设置的地址，通过端口映射，可以从内网环境访问到在开发机启动的微服务。

## 最终效果

可以在nacos中看到注册的微服务信息，而且url为我们设定的ip或者域名，端口号对应我们自己设置的端口号。

## 其它问题

目前在甲方的服务器上已经部署了lanproxy服务端，在开发机上启动了甲方VPN工具，最后启动lanproxy的客户端去连接，客户端启动出现了连接不上的情况，日志信息包含“connection timeout”，还是端口开放的问题，具体日志忘记记录了。

## 参考地址

* https://nasge.com/archives/48.html

* https://blog.csdn.net/LLittleF/article/details/108712758