---
title: CKA备考学习
top: false
cover: false
toc: true
mathjax: true
date: 2020-07-12 10:09:31
password:
summary:
tags:
categories:
---

# CKA备考学习

## 前提

* source <(kubectl completion bash) # 切换到任意节点，先执行命令补全，减少负担
* kubernetes集群信息，1.18.1
* 关于https://kubernetes.io/docs/，请尽量使用科学上网访问，另外可能存在jquery找不到的情况，需要把jquery的地址添加到科学上网

## kubectl get 命令使用

kubectl get pods <pod名称> -o yaml -n <namespace> > 
查看创建的参数信息

kubectl describe  查看运行情况以及事件信息

变量选择器：--field-selector

标签展示：-L --labels

输出格式选择：-o json/yaml。。。

kubectl get pod -A --show-labels 展示所有pod的标签信息

kubectl get pod -A -L k8s-app  展示所有pod，添加k8s-app标签列，包含改标签的则显示

kubectl get pod  -L k8s-app -n kube-system

kubectl get pod  -L k8s-app --no-headers -n kube-system 去掉标题头信息

kubectl label node worker type=nginx      对node打标签
node/worker labeled

kubectl get pod -n test-ns --watch  查看该命名空间下的pod状态变化
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-59db5dc5d5-l4x6x   0/1     ContainerCreating   0          18s
nginx-deployment-59db5dc5d5-l4x6x   1/1     Running             0          28s


-l, --selector 选择器方式
kubectl get pod  -l k8s-app=kube-proxy -n kube-system  筛选标签k8s-app的值为kube-proxy的pod

--sort-by   根据条件排序
kubectl  get pod -A --sort-by metadata.creationTimestamp -o wide  

---------------------------学习与做题的分割线-----------------------------------------------------

题目1：Set configuration context $kubectl config use-context k8s. List all PVs sorted by name, saving the full kubectl output to /opt/KUCC0010/my_volumes. Use kubectl own functionally for sorting the output, and do not manipulate it any further.

我的解答：

```
$ kubectl config use-context k8s

$ kubectl get pv -A --sort-by name > /opt/KUCC0010/my_volumes

```

标准解答：

```
kubectl config use-context k8s # 切换集群环境 
mkdir /opt/KUCC0010 # 创建对应文件夹
kubectl get --help | grep 'sort' # 查看排序命令写法 
kubectl get pv --help # 查看--all-namespaces 的用法

kubectl get pv -A -o yaml | grep name # 建议操作之前看一下name属性是否包含

kubectl get pv --all-namespaces --sort-by={.metadata.name} > /opt/KUCC0010/my_volumes

```

## kubectl logs 命令使用

-c 指定查看pod下某个容器的日志

创建一个多container的pod，如下：

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx-multi-container
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.16.1
        name: nginx
      - image: redis
        name: redis
      - image: consul
        name: consul
      #标签选择器
      nodeSelector:
        type: nginx

```
查看运行情况：

kubectl get pod -n test-ns --watch
NAME                                     READY   STATUS              RESTARTS   AGE
nginx-multi-container-6f7fbb7c59-j4sjc   0/3     ContainerCreating   0          10s
nginx-multi-container-6f7fbb7c59-j4sjc   3/3     Running             0          49s

查看日志信息：
kubectl logs nginx-multi-container-6f7fbb7c59-j4sjc -n test-ns
error: a container name must be specified for pod nginx-multi-container-6f7fbb7c59-j4sjc, choose one of: [nginx redis consul]

使用-c指定container，再来查看日志信息：
kubectl logs nginx-multi-container-6f7fbb7c59-j4sjc -c redis -n test-ns
1:C 12 Jul 2020 11:46:06.093 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 12 Jul 2020 11:46:06.093 # Redis version=6.0.5, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 12 Jul 2020 11:46:06.093 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
1:M 12 Jul 2020 11:46:06.094 * Running mode=standalone, port=6379.
1:M 12 Jul 2020 11:46:06.094 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 12 Jul 2020 11:46:06.094 # Server initialized
1:M 12 Jul 2020 11:46:06.094 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:M 12 Jul 2020 11:46:06.094 * Ready to accept connections


根据labels筛选要展示的pod日志信息
 kubectl logs -l k8s-app=kube-dns -n kube-system
[INFO] plugin/ready: Still waiting on: "kubernetes"
// 第一个pod
.:53
[INFO] plugin/reload: Running configuration MD5 = 4e235fcc3696966e76816bcd9034ebc7
CoreDNS-1.6.7
linux/amd64, go1.13.6, da7f65b
E0712 09:54:34.694215       1 reflector.go:153] pkg/mod/k8s.io/client-go@v0.17.2/tools/cache/reflector.go:105: Failed to list *v1.Endpoints: Get https://172.168.0.1:443/api/v1/endpoints?limit=500&resourceVersion=0: dial tcp 172.168.0.1:443: connect: connection timed out
E0712 09:54:34.694327       1 reflector.go:153] pkg/mod/k8s.io/client-go@v0.17.2/tools/cache/reflector.go:105: Failed to list *v1.Service: Get https://172.168.0.1:443/api/v1/services?limit=500&resourceVersion=0: dial tcp 172.168.0.1:443: connect: connection timed out
E0712 09:54:34.694370       1 reflector.go:153] pkg/mod/k8s.io/client-go@v0.17.2/tools/cache/reflector.go:105: Failed to list *v1.Namespace: Get https://172.168.0.1:443/api/v1/namespaces?limit=500&resourceVersion=0: dial tcp 172.168.0.1:443: connect: connection timed out
// 第二个pod
.:53
[INFO] plugin/reload: Running configuration MD5 = 4e235fcc3696966e76816bcd9034ebc7
CoreDNS-1.6.7
linux/amd64, go1.13.6, da7f65b

根据时间范围选择要打印的日志信息，--since后添加1h,1m,30s这样的数值
kubectl logs etcd-master1  -n kube-system --since=1h

筛选某个时间点后的日志信息，
kubectl logs etcd-master1  -n kube-system --since-time=2020-07-12T12:37:26Z
2020-07-12 12:37:26.292580 I | mvcc: store.index: compact 281140
2020-07-12 12:37:26.309379 I | mvcc: finished scheduled compaction at 281140 (took 16.558415ms)
2020-07-12 12:42:26.297970 I | mvcc: store.index: compact 281968
2020-07-12 12:42:26.312766 I | mvcc: finished scheduled compaction at 281968 (took 14.279005ms)
2020-07-12 12:47:26.304332 I | mvcc: store.index: compact 282795
2020-07-12 12:47:26.319885 I | mvcc: finished scheduled compaction at 282795 (took 14.755136ms)
2020-07-12 12:52:26.309255 I | mvcc: store.index: compact 283624
2020-07-12 12:52:26.322046 I | mvcc: finished scheduled compaction at 283624 (took 12.421311ms)


查看kubelet系统级日志，对其它服务同样适用，可以加-n确定最近多少行的日志，可以加--no-pager让日志合理打印换行展示
journalctl -n kubelet


audit、event日志信息
event实际是一种系统资源，变动、升级、创建、删除等信息
kubectl get events

注意：1. 开启日志复用和滚动  2. docker升级版本，注意日志生成的路径信息

---------------------------学习与做题的分割线-----------------------------------------------------
题目2：Set configuration context $kubectl config use-context k8s. Monitor the logs of Pod foobar and Extract log lines corresponding to error unable-to-access-website . Write them to /opt/KULM00201/foobar.

解答1：

```
$ kubectl config use-context k8s

$ mkdir /opt/KULM00201/

$ kubectl logs foobar | grep 'unable-to-access-website' > /opt/KULM00201/foobar

```

## kubernetes中的pod和label

创建pod

kubectl run pod1 --image=nginx --generator=run-pod/v1
Flag --generator has been deprecated, has no effect and will be removed in the future.
pod/pod1 created


用配置文件的方式，创建pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-3
  labels:
    app: nginx
spec:
  containers:
  - image: nginx:1.16.1
    name: nginx

```

kubectl apply -f nginx.yaml

给一个pod添加label
kubectl run pod1 --image=nginx --generator=run-pod/v1 --labels function=mantou

查看pod所含有的labels
kubectl get pod pod1 -o yaml | more
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/podIP: 10.172.171.74/32
    cni.projectcalico.org/podIPs: 10.172.171.74/32
  creationTimestamp: "2020-07-12T15:04:13Z"
  labels:
    function: mantou
.......

或者是：

 kubectl describe pod pod1 | grep Labels
Labels:       function=mantou

添加或修改pod已有的labels
kubectl edit pod pod1

apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/podIP: 10.172.171.74/32
    cni.projectcalico.org/podIPs: 10.172.171.74/32
  creationTimestamp: "2020-07-12T15:04:13Z"
  labels:
    function: mifan       # 修改馒头为米饭
    nengbunengchi: neng   # 新增一个label

记得保存退出

---------------------------学习与做题的分割线-----------------------------------------------------
题目3：创建一个Pod名称为nginx-app，镜像为nginx，添加label disk=ssd和label env=prod

解答1：

```
$ kubectl run nginx-app --image=nginx --labels disk=ssd,env=prod

```

解答2：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-app
  labels:
    disk: ssd
    env: prod
spec:
  containers:
  - image: nginx    
    name: nginx


```

```
kubectl apply -f nginx.yaml
```

## kubernetes中的多容器pod

题目4：创建一个名字为kucc的Pod，其中内部运行着nginx+redis+memcached+consul 四个容器

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kucc
spec:
  containers:
  - image: nginx
    name: nginx
  - image: redis
    name: redis
  - image: memcached
    name: memcached
  - image: consul
    name: consul

```

```
$ kubectl apply -f kucc.yaml
```
----------学习与做题的分割线-------------

多容器pod的定义方式：
* sidecar: 不起决定作用，多带点东西，例如Prometheus中node_export、fluentd日志采集
* adapter: 修改入站和出站的数据，例如格式化容器输出的监控数据RESTFUL format，功能类似hdmi转vga的转接头
* ambassador: (proxy) istio，spring cloud gateway，zuul，使用容器提供的代理模式，而不是使用k8s自身的代理模式

## kubernetes中DaemonSet相关概念

DaemonSet: 确保在全部（或者一些）Node 上运行一个 Pod 的副本。当有 Node 加入集群时，也会为他们新增一个 Pod 。当有 Node 从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。

使用范围：

* 运行集群存储 daemon，例如在每个 Node 上运行 glusterd、ceph。
* 在每个 Node 上运行日志收集 daemon，例如fluentd、logstash。
* 在每个 Node 上运行监控 daemon，例如 Prometheus Node Exporter、collectd、Datadog 代理、New Relic 代理，或 Ganglia gmond。

具体使用：kubernetes中log-pilot部署

taint与toleration：Taint（污点）和 Toleration（容忍）可以作用于 node 和 pod 上，其目的是优化 pod 在集群间的调度，这跟节点亲和性类似，只不过它们作用的方式相反，具有 taint 的 node 和 pod 是互斥关系，而具有节点亲和性关系的 node 和 pod 是相吸的。Tolerations作用于pod上,允许(但不是必须)pod被调度到有符合的污点(taint)的节点上。

https://www.cnblogs.com/tylerzhou/p/11026364.html


------------------------------学习与做题的分割线---------------------------------------------

题目5：确保在 kubectl 集群的每个节点上运行一个 Nginx Pod。其中 Nginx Pod 必须使用 Nginx 镜像。**不要覆盖当前环境中的任何 traints。** 使用 Daemonset 来完成这个任务，Daemonset 的名字使用 ds。

https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/

daemonset.yaml

```yaml 

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: nginx
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
      - name: nginx
        image: nginx

```

```
$ kubectl apply -f daemonset.yaml 

$ kubectl get daemonset -n test-ns
NAME   DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
ds     1         1         1       1            1           <none>          19s


```

如果是**需要覆盖taints**，例如新增一个节点taint，

```
kubectl taint nodes node1 key=value:NoSchedule

或者查看taint

kubectl describe nodes node1 |grep -E '(Roles|Taints)'
```

然后允许nginx镜像调度到taint所在的节点，“key”值要与“effect”值对应，配置如下：

```yaml 

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds
  namespace: test-ns
spec:
  selector:
    matchLabels:
      name: nginx          # 该位置与下面template中的labels中name的值对应
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
      nodeSelector:
        kubernetes.io/hostname: master1
      tolerations:
        - key: "key"
          operator: "Equal"
          value: "value"
          effect: "NoSchedule"

```

## kubernetes中initcontainer相关概念

一个pod里可以运行多个容器,它也可以运行一个或者多个初始容器,初始容器先于应用容器运行,除了以下两点外,初始容器和普通容器没有什么两样:

* 它们总是run to completion
* 一个初始容器必须成功运行另一个才能运行

初始容器不支持可用性探针(readiness probe),因为它在ready之前必须run to completion。如果在一个pod里指定了多个初始容器,则它们会依次启动起来(pod内的普通容器并行启动),并且只有上一个成功下一个才能启动.当所有的初始容器都启动了,kubernetes才开始启普通应用容器。

因为 Init 容器具有与应用容器分离的单独镜像，其启动相关代码具有如下优势：

* Init 容器可以包含一些安装过程中应用容器中不存在的实用工具或个性化代码。例如，没有必要仅为了在安装过程中使用类似 sed、 awk、 python 或 dig 这样的工具而去FROM 一个镜像来生成一个新的镜像。
* Init 容器可以安全地运行这些工具，避免这些工具导致应用镜像的安全性降低。
* 应用镜像的创建者和部署者可以各自独立工作，而没有必要联合构建一个单独的应用镜像。
* Init 容器能以不同于Pod内应用容器的文件系统视图运行。因此，Init容器可具有访问 Secrets 的权限，而应用容器不能够访问。
* 由于 Init 容器必须在应用容器启动之前运行完成，因此 Init 容器提供了一种机制来阻塞或延迟应用容器的启动，直到满足了一组先决条件。一旦前置条件满足，Pod内的所有的应用容器会并行启动。

------------------------------学习与做题的分割线---------------------------------------------
题目6： 添加一个 initcontainer 到 lum(/etc/data)这个 initcontainer 应该创建一个名为/workdir/calm.txt 的空文件，如果/workdir/calm.txt 没有被检测到，这个 Pod 应该退出。

解答：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  volumes:
  - name: lum
    hostPath:
      path: /etc/data
  containers:
  - name: myapp-container
    image: nginx
    volumeMounts:                       # 如果这里不加volumeMounts的配置，无法启动
    - name: lum
      mountPath: /workdir/  
  initContainers: 
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', "touch /workdir/calm.txt"]
    volumeMounts:
    - name: lum 
      mountPath: /workdir/




```

```
$ kubectl apply -f initcontainer.yaml 
pod/myapp-pod created

$ kubectl get pod myapp-pod -w
NAME        READY   STATUS     RESTARTS   AGE
myapp-pod   0/1     Init:0/1   0          2s
myapp-pod   0/1     PodInitializing   0          13s
myapp-pod   1/1     Running           0          22s


```

## kubernetes中deployment相关概念

Deployment控制ReplicaSet的多个版本，ReplicaSet控制Pod个数。

Deployment实际上一个两层控制器，遵循一种滚动更新的方式来实升级现有的容器，这个能力的实现，依赖的就是ReplicaSet这个对象。

当我们修改了Deployment对象后，Deployment控制器会使用修改后的模板，创建一个新的ReplicaSet对象，这时候有两个RelicaSet对象，
Deployment通过控制ReplicaSet对象的pod数量来达到滚动升级的效果。

例如A和B，如果最终设置的pod数都是3，通过A-1，B+1这样的方式，直到A的pod数量变为0，最终 达到了滚动升级的目的。
同时，因为存在多个ReplicaSet，让回滚成为了可能。

Deployment 通过“控制器模式”，来操作 ReplicaSet 的个数和属性，进而实现“水平扩展 / 收缩”和“滚动更新”这两个编排动作。

ReplicaSet 多个相同的Pod运行集合，ReplicaSet由Deployment控制。

Deployment的伸缩：修改replicas的数量

------------------------------学习与做题的分割线---------------------------------------------
题目7：创建 deployment 名字为 nginx-app 容器采用 1.11.9 版本的 nginx  这个 deployment 包含 3 个副本,接下来通过滚动升级的方式更新镜像版本为 1.12.0，并记录这个更新，最后，回滚这个更新到之前的 1.11.9 版本

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.11.9

```

```
$ kubectl apply -f deployment.yaml 
deployment.apps/nginx-app created

$ kubectl get deploy
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app          3/3     3            3           58s

$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-app-7f5f57f857          3         3         0       6s

$ kubectl rollout status deployment.v1.apps/nginx-app
deployment "nginx-app" successfully rolled out

$ kubectl rollout history deployment.v1.apps/nginx-app
deployment.apps/nginx-app 
REVISION  CHANGE-CAUSE
1         <none>

$ kubectl set image deployment/nginx-app nginx=nginx:1.12.0 --record

$  kubectl rollout status deployment.v1.apps/nginx-app
Waiting for deployment "nginx-app" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-app" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-app" successfully rolled out

$  kubectl rollout history deployment.v1.apps/nginx-app
deployment.apps/nginx-app 
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment/nginx-app nginx=nginx:1.12.0 --record=true

$ kubectl rollout undo deployment.v1.apps/nginx-app

$  kubectl rollout status deployment.v1.apps/nginx-app
Waiting for deployment "nginx-app" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-app" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-app" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-app" successfully rolled out

$  kubectl rollout history deployment.v1.apps/nginx-app
deployment.apps/nginx-app 
REVISION  CHANGE-CAUSE
2         kubectl set image deployment/nginx-app nginx=nginx:1.12.0 --record=true
3         <none>

// 标号最新为最近的状态

```

## kubernetes中service相关概念

将运行在一组 Pods 上的应用程序公开为网络服务的抽象方法。

使用Kubernetes，您无需修改应用程序即可使用不熟悉的服务发现机制。 Kubernetes为Pods提供自己的IP地址和一组Pod的单个DNS名称，并且可以在它们之间进行负载平衡。

Service Pod的对外访问的暴露，loadBalance、nodePort、ClusterIP三种方式

------------------------------学习与做题的分割线---------------------------------------------
题目8：创建和配置 service，名字为 front-end-service。可以通过 NodePort/ClusterIp 开访问，并且路由到 front-end 的 Pod 上。

解答：

Pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-app
  labels:
    app: nginx-app
spec:
  containers:
  - image: nginx
    name: nginx



```

```
$ kubectl apply -f nginx.yaml

$  kubectl get pod
NAME                                READY   STATUS              RESTARTS   AGE
nginx-app                           1/1     Running   0          4s


```

Service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: front-end-service
spec:
  type: NodePort
  selector:
    app: nginx-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

```

```
$ kubectl apply -f service-nginx.yaml 
service/front-end-service created
$ kubectl get svc
NAME                TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)        AGE
front-end-service   NodePort    172.168.170.0     <none>        80:30676/TCP   5s

$ curl localhost:30676
```

或者操作:
```
$ kubectl expose pod nginx-app --name=front-end --port=80  --type=NodePort

```

## kubernetes中namespace相关概念

------------------------------学习与做题的分割线---------------------------------------------
题目9：创建一个 Pod，名字为 Jenkins，镜像使用 Jenkins。在新的 namespace website-frontend 上创建

解答：

```

$ kubectl create ns website-frontend 

$ vim jenkins-pod.yaml
// 输入以下内容
apiVersion: v1
kind: Pod
metadata:
  name: jenkins
  namespace: website-frontend
spec:
  containers:
  - name: jenkins
    image: jenkins

$ kubectl apply -f jenkins-pod.yaml


```

## --dry-run概念

使用 --dry-run 参数预览要发送到集群的对象，而无需真正提交。

------------------------------学习与做题的分割线---------------------------------------------
题目10：创建 deployment 的 spec 文件: 使用 redis 镜像，7 个副本，label 为 app_enb_stage=dev deployment 名字为 kual00201 保存这个 spec 文件到/opt/KUAL00201/deploy_spec.yaml完成后，清理(删除)在此任务期间生成的任何新的 k8s API 对象

解答：

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: kual00201
  labels:
    app_enb_stage: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app_enb_stage: dev
  template:
    metadata:
      labels:
        app_enb_stage: dev
    spec:
      containers:
      - name: redis
        image: redis

```

```
$ mkdir -p /opt/KUAL00201/

$ kubectl apply -f redis.yaml --dry-run -o yaml > /opt/KUAL00201/deploy_spec.yaml

```

## 格式化输出信息的问题

1. Service和Pod对应，通过labels进行选择。

2. 格式化输出信息**-o custom-columns=NAME:metadata.name*

------------------------------学习与做题的分割线---------------------------------------------
题目11：创建一个文件/opt/kucc.txt ，这个文件列出所有的 service 为 foo ,在 namespace 为 production 的 Pod这个文件的格式是每行一个 Pod的名字


解答：

```
$ kubectl get svc -n production --show-labels | grep foo
app=test

$ kubectl get pod -l app=test -o custom-columns=NAME:metadata.name > /opt/kucc.txt

```

## Secrets

------------------------------学习与做题的分割线---------------------------------------------
题目12：创建一个secret,名字为super-secret包含用户名bob,创建pod1挂载该secret，路径为/secret，创建pod2，使用环境变量引用该secret，该变量的环境变量名为ABC。





centos设置代理时，no_proxy必须是使用具体的地址信息，否则不生效！