---
title: kt-connect的安装和使用
top: false
cover: false
toc: true
mathjax: true
date: 2020-04-09 12:21:47
password:
summary:
tags:
categories:
---

# kt-connect的安装和使用

## 环境介绍

开发机：win7、win10

工具版本:

* kt-connect: 0.0.12
* kubectl: 1.17.2
* 命令行工具：MobaXterm v11.1，cmder

python版本：python3.7，安装virtualenv和sshuttle。

## 本文适用于

- 使用windows进行开发的java开发者
- idea作为主力开发的ide
- Kubernetes作为开发测试环境
- 开发的应用包含多个微服务，且存在相互调用关系

## 资料下载

文中涉及的所有配置文件、工具，和文章放在同一个目录下，会上传到gitlab中。在同目录下的tools文件夹中，解压即可用！

## 配置环境并安装

### 0. 搭建python环境

首先下载[python3.7](https://www.python.org/ftp/python/3.7.0/python-3.7.0-amd64.exe)的安装包，进行安装。请自行勾选添加到环境变量的工作，并且允许该计算机上的所有用户使用。

安装完成后，在命令行工具下进行测试，如下：

```

$ python
Python 3.7.0 (v3.7.0:1bf9cc5093, Jun 27 2018, 04:59:51) [MSC v.1914 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
>>>

```

说明python环境搭建完成。

然后需要创建python运行的虚拟环境，用来安装sshuttle工具。下面首先需要安装virtualenv工具并创建虚拟环境：

```

$ pip install virtualenv

// 转到你要的路径下
$ cd D:\dev\Java\ktctl_windows_amd64

// 创建虚拟环境，虚拟环境名称为vevn
$ virtualenv venv

// 使虚拟环境生效
$ .\venv\Scripts\activate

// 查看虚拟环境路径下的信息
(venv) $ ls
venv

```

虚拟环境生效后，在命令行前面会出现**(venv)**的字样。说明虚拟环境已经生效，这时候在虚拟环境下进行操作，不会影响外部的python运行环境，起到隔离的作用！

最后我们需要安装sshuttle工具，直接使用python所带的pip工具进行安装，如下：

```
(venv) $ pip install sshuttle

```

安装完成后即可使用。

### 1. 安装kubectl

下载[kubectl工具](https://storage.googleapis.com/kubernetes-release/release/v1.17.2/bin/windows/amd64/kubectl.exe)，然后放到与venv相同路径下。

然后获取k8s中的配置信息，这里因为使用的Rancher中的rke工具进行安装，配置文件名称为：kube_config_cluster.yml，所以直接copy出配置文件一并放到与venv相同路径下。

现在就可以查看当前的k8s集群中的内容了，进行如下设置：

```
// 指定配置文件的路径
$ set KUBECONFIG=./kube_config_cluster.yml

// 查看集群中某个namespace下的内容
$ kubectl.exe get pod -n test-test-business
NAME                                READY   STATUS    RESTARTS   AGE
test-business-01-84cf9fcbb7-bcgs2   0/1     Running   285        25h
test-business-02-58655d78c6-jhl7s   1/1     Running   0          5d1h

```

这样kubectl就配置完成了。

### 2. 安装kt-connect

[下载kt-connect](https://github.com/alibaba/kt-connect/releases/download/kt-0.0.12/ktctl_windows_amd64.tar.gz)并解压到之前venv所在的路径下，然后查看文件夹下的内容：

```

$ ls
ktctl_windows_amd64.exe*  kube_config_cluster.yml  kubectl.exe*  venv/

```

需要配置环境变量，将环境变量的路径指向该目录下，我这里的目录是**D:\dev\Java\ktctl_windows_amd64**，其实之前执行的set命令也可以配置到环境变量中，这个自行配置下即可。但是令人不太满意的是，在真正运行的时候，还是需要指定全路径信息！

### 3. 部署测试镜像

下面我们就可以部署测试的镜像，可以使用刚才配置的kubectl工具进行部署，操作如下：

```

$ kubectl create ns kt-connect-test

$ kubectl run tomcat --image=registry.cn-hangzhou.aliyuncs.com/rdc-product/kt-connect-tomcat9:1.0 --expose --port=8080 -n kt-connect-test
service "tomcat" created
deployment.apps "tomcat" created

$ kubectl get deployments -o wide --selector run=tomcat -n kt-connect-test
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS   IMAGES                                                                 SELECTOR
tomcat    1         1         1            1           18s       tomcat       registry.cn-hangzhou.aliyuncs.com/rdc-product/kt-connect-tomcat9:1.0   run=tomcat

$ kubectl get pods -o wide --selector run=tomcat -n kt-connect-test
NAME                      READY     STATUS    RESTARTS   AGE       IP             NODE
tomcat-54d87b848c-2mc9b   1/1       Running   0          1m        172.23.2.234   cn-beijing.192.168.0.8

$ kubectl get svc tomcat -n kt-connect-test
NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
tomcat    ClusterIP   172.21.6.39   <none>        8080/TCP   1m

```

这样根据以上展示的所有信息，可以进行下面的启动和连接。

### 4. 启动kt-connect

开始操作之前，要知道windows的限制，Windwos环境下KT Connect只支持使用SOCKS5代理模式，在该模式下用户可以直接在本地访问PodIP和ClusterIP,但是无法直接使用DNS。在操作之前，请先确定本机已经安装kubectl并且能够正常与Kubernetes集群交互，而且一定确定使你之前配置的虚拟环境进行生效！首先在正式操作之前，检查下环境是否都准备好了。当出现*KT Connect is ready, enjoy it!*时，显示可以进行直接连接操作，如下：

```

(venv) λ D:\dev\Java\ktctl_windows_amd64\ktctl_windows_amd64.exe --kubeconfig D:\dev\Java\ktctl_windows_amd64\kube_config_cluster.yml -n kt-connect-test -d connecct --method=socks5 --proxy 42222 --port 42333

1:24PM INF Connect Start At 21892
1:24PM INF Client address 10.0.93.193
1:24PM INF deploy shadow deployment kt-connect-daemon-ieljt in namespace kt-connect-test

1:24PM INF pod label: kt=kt-connect-daemon-ieljt
1:24PM INF pod:  is running,but not ready
1:24PM INF pod: kt-connect-daemon-ieljt-65b8f8956-s7js2 is running,but not ready
1:24PM INF pod: kt-connect-daemon-ieljt-65b8f8956-s7js2 is running,but not ready
1:24PM INF pod: kt-connect-daemon-ieljt-65b8f8956-s7js2 is running,but not ready
1:24PM INF pod: kt-connect-daemon-ieljt-65b8f8956-s7js2 is running,but not ready
1:24PM INF pod: kt-connect-daemon-ieljt-65b8f8956-s7js2 is running,but not ready
1:24PM INF pod: kt-connect-daemon-ieljt-65b8f8956-s7js2 is running,but not ready
1:25PM INF pod: kt-connect-daemon-ieljt-65b8f8956-s7js2 is running,but not ready
1:25PM INF pod: kt-connect-daemon-ieljt-65b8f8956-s7js2 is running,but not ready
1:25PM INF pod: kt-connect-daemon-ieljt-65b8f8956-s7js2 is running,but not ready
1:25PM INF pod: kt-connect-daemon-ieljt-65b8f8956-s7js2 is running,but not ready
1:25PM INF pod: kt-connect-daemon-ieljt-65b8f8956-s7js2 is running,but not ready
1:25PM INF pod: kt-connect-daemon-ieljt-65b8f8956-s7js2 is running,but not ready
1:25PM INF Shadow pod: kt-connect-daemon-ieljt-65b8f8956-s7js2 is ready.
1:25PM DBG Child, os.Args = [D:\dev\Java\ktctl_windows_amd64\ktctl_windows_amd64.exe --kubeconfig D:\dev\Java\ktctl_windows_amd64\kube_config_cluster.yml -n kt-connect-test -d connect --method=socks5 --proxy 42222 --port 42333]
1:25PM DBG Child, cmd.Args = [kubectl --kubeconfig=D:\dev\Java\ktctl_windows_amd64\kube_config_cluster.yml -n kt-connect-test port-forward kt-connect-daemon-ieljt-65b8f8956-s7js2 42333:22]
Forwarding from 127.0.0.1:42333 -> 22
Forwarding from [::1]:42333 -> 22
1:25PM INF port-forward start at pid: 3920
1:25PM INF ==============================================================
1:25PM INF Start SOCKS5 Proxy: export http_proxy=socks5://127.0.0.1:42222
1:25PM INF ==============================================================
1:25PM DBG Child, os.Args = [D:\dev\Java\ktctl_windows_amd64\ktctl_windows_amd64.exe --kubeconfig D:\dev\Java\ktctl_windows_amd64\kube_config_cluster.yml -n kt-connect-test -d connect --method=socks5 --proxy 42222 --port 42333]
1:25PM DBG Child, cmd.Args = [ssh -oStrictHostKeyChecking=no -oUserKnownHostsFile=/dev/null -i C:\Users\ht/.kt_id_rsa -D 42222 root@127.0.0.1 -p42333 sh loop.sh] Handling connection for 42333
Warning: Permanently added '[127.0.0.1]:42333' (ECDSA) to the list of known hosts.
1:25PM INF vpn(ssh) start at pid: 10724
1:25PM INF KT proxy start successful

```

当没有错误出现，且出现的信息如上时，可以执行后续的测试连接操作。

### 5. 测试连接

这时候需要另开一个终端界面，进行下面的操作：

```
// 参考上面日志中的提示Start SOCKS5 Proxy: export http_proxy=socks5://127.0.0.1:42222
// windows需要使用set替换export命令
$ set http_proxy=socks5://127.0.0.1:42222

// 测试
$ curl http://172.21.6.39:8080
kt-connect demo from tomcat9

```

这时候出现了**kt-connect demo from tomcat9**，说明访问上面部署的tomcat镜像成功，本地联调也已经成功了。这样我们就可以进行服务的联调了。

## 具体案例操作

### 1. 测试服务介绍

以test-business01为案例进行测试，在本地启动test-business01服务，替换线上的test-business01服务，然后从线下服务中打印日志，表示服务已经切换到线下开发机的服务中了。

### 2. 测试服务线上启动

首先连接到线上的namespace中，名称为test-test-business。执行connect命令：

```
(venv) $ D:\dev\Java\ktctl_windows_amd64\ktctl_windows_amd64.exe --kubeconfig D:\dev\Java\ktctl_windows_amd64\kube_config_cluster.yml -n test-test-business --d connect --method=socks5 --proxy 42222 --port 42333

```

然后使用本地服务替换线上的服务，新打开一个命令行终端，命令如下：

```
// 先在命令行中配置代理信息
// 参考上面日志中的提示Start SOCKS5 Proxy: export http_proxy=socks5://127.0.0.1:42222
// windows需要使用set替换export命令
// $ set http_proxy=socks5://127.0.0.1:42222
// 不需要设置代理信息即可使用

// 启动虚拟环境
$ ./venv/Scripts/activate 

// 启动新的影子镜像替换旧的镜像
(venv) $ ktct_windows_amd64.exe --kubeconfig D:\dev\Java\ktctl_windows_amd64\kube_config_cluster.yml --debug --namespace=test-test-business exchange test-business-01 --expose 19030

```

启动后，我们查看一下线上服务是否被替换，如下：

```
$ kubectl get pod -n test-test-business
NAME                                        READY   STATUS    RESTARTS   AGE
kt-connect-daemon-acbhz-6648c86848-t4kzr    1/1     Running   0          86m
test-business-01-kt-tccwe-658f9c4b5-l8sdc   1/1     Running   0          32m
test-business-02-58655d78c6-jhl7s           1/1     Running   0          5d6h

```

test-business01已经添加上了*kt-*标志，说明已经替换成我们自己的了！完成后需要对nacos和eureka以及网关等进行配置上的修改！

目前nacos和eureka并未对外暴露，服务可能无法注册进去，需要nacos和eureka进行调整。其次在本地服务启动时，需要重新导入nacos和eureka的配置信息。针对nacos进行配置，新建一个namespace，导入test-business相关的配置信息，将eureka、数据库、缓存的连接更改为对外暴露的地址信息，以peer0为标志，加载该标志对应的配置信息。

由于nacos部署的是headless服务信息，所以无法使用nodePort模式进行访问。再考虑到我们这里使用的socks5的代理方式，最终只能使用clusterIP的方式进行连接。首先需要获取nacos中的clusterIP，如下：

```
$ kubectl get svc -n test-basic-nacos
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
mysql            ClusterIP   10.43.252.60   <none>        3306/TCP   18d
nacos-headless   ClusterIP   None           <none>        8848/TCP   18d
$ kubectl describe svc nacos-headless -n test-basic-nacos
Name:              nacos-headless
Namespace:         test-basic-nacos
Labels:            app=nacos
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"nacos"},"name":"nacos-headless","namespace":"test-b...
Selector:          app=nacos
Type:              ClusterIP
IP:                None
Port:              server  8848/TCP
TargetPort:        8848/TCP
Endpoints:         10.42.0.203:8848,10.42.1.15:8848,10.42.2.79:8848
Session Affinity:  None
Events:            <none>

```

Endpoints一栏表示了，当前的nacos对应的ip地址和端口信息，根据headless服务的特点，Endpoint的ip在重新部署时会发生变更，这时候需要经常查看以进行修改。考虑到我们要在后面设置程序的代理信息，通过代理是可以连接到集群内部的nacos服务，因此采用这种方式来处理。如果idea控制台中的日志出现了连接超时的问题，临时可以忽略，不影响程序运行！

然后在test-business01中更改nacos的地址，变更为k8s集群中对外暴露的地址，如下：

```yaml

spring:
  application:
    name: test-business01
  cloud:
    nacos:
      config:
        # 测试环境k8s使用
        server-addr: ${nacos_ip:10.42.1.15:8848}
        namespace: ${nacos_namespace:test-dev-local}
        # 开发环境使用
#        server-addr: ${nacos_ip:192.168.88.161:18848}
#        namespace: ${nacos_namespace:858b37b1-35be-4564-a24e-dc2c322d5784}
        file-extension: yml

feign:
#  httpclient:
#    enabled: false
#  okhttp:
#    enabled: true
  hystrix:
    enabled: true
  client:
    config:
      remote-service:
        connectTimeout: 1000
        readTimeout: 12000

```

其中10.42.1.15代表的是k8s集群内的nacos地址信息，是某一个nacos的连接地址，因为通过kt-connect连接集群后，使用socks5方式进行代理，只能通过内部的clusterIP的方式进行访问！但是该ip是变化的，根据实际情况进行设置。

### 3. idea接入测试服务进行调试

上述设置完成后，开始使用本地服务进行测试！

官网推荐使用[jvm-inject](https://plugins.jetbrains.com/plugin/13482-jvm-inject)插件，但是我们从2019.1.3版本试到2019.3.4版本，均无法安装该插件。

在前面的操作中，在venv所在路径下，生成了.jvmrc文件，打开查看，内容如下：

```
-DsocksProxyHost=127.0.0.1
-DsocksProxyPort=42222

```

然后通过反编译获取jvm-inject插件的核心源码，如下：

```

//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package io.github.jvm.inject;

import com.google.common.base.Joiner;
import com.intellij.execution.Executor;
import com.intellij.execution.configurations.JavaParameters;
import com.intellij.execution.configurations.RunConfiguration;
import com.intellij.execution.configurations.RunProfile;
import com.intellij.execution.runners.JavaProgramPatcher;
import java.io.File;
import java.io.IOException;
import java.util.List;
import org.apache.commons.io.FileUtils;

public class JvmInject extends JavaProgramPatcher {
    public static final String RC_FILE = ".jvmrc";

    public JvmInject() {
    }

    public void patchJavaParameters(Executor executor, RunProfile configuration, JavaParameters javaParameters) {
        if (configuration instanceof RunConfiguration) {
            RunConfiguration runConfiguration = (RunConfiguration)configuration;
            String root = runConfiguration.getProject().getBaseDir().getPath();
            File file = FileUtils.getFile(new String[]{root, ".jvmrc"});
            if (file.exists()) {
                try {
                    List<String> lines = FileUtils.readLines(file, "UTF-8");
                    String result = Joiner.on(" ").join(lines);
                    javaParameters.getVMParametersList().addParametersString(result);
                } catch (IOException var9) {
                    var9.printStackTrace();
                }

            }
        }
    }
}


```

实际上就是向启动命令中注入了.jvmrc中包含的信息**getVMParametersList()**判断是通过ide中配置VMOptions的参数进行的，那么我们可以手动进行启动信息的注入！如下图：

![](配置信息.png)

然后启动服务的时候，在VM Options中的参数会自动注入到启动命令中，如下：

```

D:\dev\Java\jdk1.8.0_201\bin\java.exe -DsocksProxyHost=127.0.0.1 -DsocksProxyPort=42222 -XX:TieredStopAtLevel=1 -noverify -Dspring.profiles.active=--spring.profiles.active=peer0 -Dspring.output.ansi.enabled=always -Dcom.sun.management.jmxremote -Dspring.jmx.enabled=true

```

这样代理已经设置完毕，结合上面的内容，配置文件bootstrap.yml替换了nacos的地址信息，这时进行启动该服务！启动完成后查看eureka服务，看服务是否已经注册了，如下图：

![](eureka注册信息.png)

服务已经注册，192.168.88.193所示的服务就是我们自己线下启动的服务，这时候通过接口进行测试，完成一次查询的操作，看接口是否能连通，插入数据的时候，日志如下：

```
2020-04-07 18:48:30.850  INFO [test-business01,,,] 12464 --- [erListUpdater-0] c.netflix.config.ChainedDynamicProperty  : Flipping property: test-business02.ribbon.ActiveConnectionsLimit to use NEXT property: niws.loadbalancer.availabilityFilteringRule.activeConnectionsLimit = 2147483647
Creating a new SqlSession
SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@62210e0b] was not registered for synchronization because synchronization is not active
JDBC Connection [com.alibaba.druid.proxy.jdbc.ConnectionProxyImpl@49a685f] will not be managed by Spring
==>  Preparing: SELECT id,name FROM roles 
==> Parameters: 
<==    Columns: id, name
<==        Row: 5, ROLE_ADMIN
<==        Row: 4, ROLE_USER
<==      Total: 2
Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@62210e0b]


```

测试插入和查询的接口，日志均能进行打印！所以到这里已经成功了！

### 4. 终止调试

最后要终止调试，并恢复之前的服务，在启动影子镜像的命令行中，使用ctrl+c终止运行

```
(venv) $ ktct_windows_amd64.exe --kubeconfig D:\dev\Java\ktctl_windows_amd64\kube_config_cluster.yml --debug --namespace=test-test-business exchange test-business-01 --expose 19030

```

上述命令行中的运行程序被终止，程序会退出并清理之前部署的镜像信息，查看镜像时，会发现测试镜像被替换，恢复原来的镜像信息，如下：

```
$ kubectl get pod -n test-test-business
NAME                                         READY   STATUS              RESTARTS   AGE
kt-connect-daemon-dbjpv-666fdf5c7d-f5n55     1/1     Running             0          21m
test-business-01-67b4cd9d4c-smjfg            0/1     ContainerCreating   0          14s
// 该镜像被清理且终止
test-business-01-kt-jbizj-6cb8d9d898-w6rkw   1/1     Terminating         0          17m
test-business-02-58655d78c6-jhl7s            1/1     Running             0          5d22h

$ kubectl get pod -n test-test-business
NAME                                       READY   STATUS    RESTARTS   AGE
kt-connect-daemon-dbjpv-666fdf5c7d-f5n55   1/1     Running   0          25m
test-business-01-67b4cd9d4c-smjfg          0/1     Running   0          4m54s
test-business-02-58655d78c6-jhl7s          1/1     Running   0          5d22h

```

这时候再访问，发现已经恢复为以前的镜像！

## 总结

目前只是使用到了k8s部分的内容，下一步添加ServiceMesh时，利用istio工具实现对流量的切换，需要对当前的内容进行升级！

## 参考链接

* 介绍：https://www.v2ex.com/t/584314

* 排查问题：https://alibaba.github.io/kt-connect/#/zh-cn/troubleshoot

* windows支持：https://alibaba.github.io/kt-connect/#/zh-cn/guide/windows-support

* 快速开始，测试镜像：https://alibaba.github.io/kt-connect/#/zh-cn/quickstart