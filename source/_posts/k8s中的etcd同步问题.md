---
title: k8s中的etcd同步问题
top: false
cover: false
toc: true
mathjax: true
date: 2020-05-14 10:38:54
password:
summary:
tags:
categories:
---

## 关于k8s集群中，各个节点资源进行富集的问题

### 现象

先看一下集群中的机器情况：

```

$ kubectl get node -o wide
NAME          STATUS   ROLES                      AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
192.168.232.232   Ready    controlplane,etcd,worker   57d   v1.17.2   192.168.232.232   <none>        CentOS Linux 7 (Core)   3.10.0-1062.12.1.el7.x86_64   docker://19.3.6
192.168.232.233   Ready    worker                     14d   v1.17.2   192.168.232.233   <none>        CentOS Linux 7 (Core)   3.10.0-1062.18.1.el7.x86_64   docker://19.3.8
192.168.232.234   Ready    worker                     14d   v1.17.2   192.168.232.234   <none>        CentOS Linux 7 (Core)   3.10.0-1062.18.1.el7.x86_64   docker://19.3.8
192.168.232.239   Ready    controlplane,etcd,worker   57d   v1.17.2   192.168.232.239   <none>        CentOS Linux 7 (Core)   3.10.0-1062.12.1.el7.x86_64   docker://19.3.6
192.168.232.241   Ready    controlplane,etcd,worker   57d   v1.17.2   192.168.232.241   <none>        CentOS Linux 7 (Core)   3.10.0-1062.12.1.el7.x86_64   docker://19.3.6

```

再去看一下，集群的资源使用情况：

```
$ kubectl top node
NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
192.168.232.232   3990m        99%    8257Mi          52%
192.168.232.233   3889m        97%    8646Mi          54%
192.168.232.234   382m         9%     7066Mi          44%
192.168.232.239   775m         19%    7993Mi          50%
192.168.232.241   643m         16%    5664Mi          35%

```

明显看到，232、233两台机器负载较高，而后面的234、239、241负载压力非常低。这样导致了以下问题：

- pod启动缓慢
- pod部署时，无法进行部署
- pod做集群时，多个相同pod运行在同一节点上

### 原因分析

问题在于，pod的部署策略存在问题，未能将replicas不为1的服务，均匀部署到其它机器上，导致pod出现了富集，整个集群性能下降，启动缓慢，部署难。不能发挥整个k8s的集群优势！

### 解决方式和策略

我们目前集群中存在所有的微服务相关的基础服务，以及我们的业务服务。首先要针对集群中的各个node节点，进行标签的划分，然后通过针对node节点进行亲和性设置，部署在特定节点上。

其次针对需要进行集群化部署的服务，需要对相同的pod进行反亲和性设定，确保它们部署在不同的机器上，再根据前面划定的机器标签，部署在工作节点上。

最后，针对特定业务，例如工作流、表单、业务建模这样的，进行反亲和性设定，互不影响。其它业务不做强制设定。

主要的调整点如下：

- 针对资源申请的限制，我们限定了内存的请求，尚未限制CPU的使用
- 亲和性和反亲和性的设置，针对Pod层面以及node层面
- 对Pod进行自动伸缩，目前先做到针对CPU和内存的自动伸缩

具体实施，以Eureka为例，演示针对node进行亲和性设置。以用户相关的服务，uaa和umc来演示pod之间的反亲和性。

1. Eureka的部署调整

首先对所在节点设置标签，选取232,239,241为Eureka的部署机器，调整如下：

```
// 调整节点，打标签
$ kubectl label node 192.168.232.232 192.168.232.239 192.168.232.241 basic-eureka=eureka
node/192.168.232.232 labeled
node/192.168.232.239 labeled
node/192.168.232.241 labeled


// 配置文件设定
$ vim server-register-eureka-k8s.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: eureka
  namespace: test-basic-eureka
  labels:
    app: eureka
spec:
  ports:
    - port: 19011
      name: eureka
      targetPort: 19011
  clusterIP: None
  selector:
    app: eureka
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: eureka
  namespace: test-basic-eureka
spec:
  serviceName: "eureka"
  replicas: 3
  selector:
    matchLabels:
      app: eureka
  template:
    metadata:
      labels:
        app: eureka
    spec:
      affinity:                                        # 设置节点的亲和性
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: basic-eureka            # 根据标签的名称进行设置
                    operator: In
                    values:
                      - eureka
      containers:
        - name: eureka
          image: 192.168.232.159:5000/server-register-k8s:$BUILD_NUMBER
          ports:
            - containerPort: 19011
          lifecycle:
            preStop:
              httpGet:
                port: 19011
                path: /spring/shutdown
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 19011
            initialDelaySeconds: 180
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 19011
            initialDelaySeconds: 180
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          resources:
            limits:                       # 增加对CPU、内存使用的限制
              memory: 2Gi
              cpu: 500m
            requests:
              memory: 1.5Gi
              cpu: 250m
          env:
            - name: IP_ADDR
              #value: eureka-*.eureka-headless.test-basic-eureka.svc.cluster.local
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NACOS_IP
              value: 192.168.232.194:18848
            - name: NACOS_NAMESPACE
              value: 4cca5bd9-730b-49sadfa32e-875b0c0478d8
            - name: SKYWALKING_NAMESPACE
              value: test-k8s-test
            - name: SKYWALKING_TARGET_SERVICE_NAME
              value: test-user-umc
            - name: SKYWALKING_IP_PORT
              value: 192.168.232.163:11800
            - name: CHANNEL
              value: standalone
  podManagementPolicy: "Parallel"

```

设置完成后，我们还需要对node节点进行调整才能进行部署，由于CPU占用资源过多的问题，需要我们对pod进行筛选：

```
// 查看资源占用
$ kubectl top pod -n test-all-service
NAME                                         CPU(cores)   MEMORY(bytes)
test-data-dictionary-7c7cf58696-7zrwd   11m          463Mi
test-filestore-opt-84c5976859-l6w8q     9m           484Mi
test-filestore-uc-8568f58df-2zggs       9m           493Mi
test-form-design-d546c579d-wk8jt        342m         80Mi
test-frontend-795b45b99d-tqhzb          0m           334Mi
test-frontend-user-6c75fc8c4d-qnqdw     0m           334Mi
test-id-generator-59ccb5b667-bbp4k      18m          421Mi
test-id-generator-59ccb5b667-qqfj5      10m          425Mi
test-id-generator-59ccb5b667-z62d2      13m          390Mi
test-opt-agent-58d8dd9cdd-7d4rv         368m         154Mi
test-opt-agent-58d8dd9cdd-df74m         9m           412Mi
test-opt-agent-58d8dd9cdd-n6859         13m          410Mi
test-opt-center-85899c4cc-dql54         3642m        971Mi
test-opt-uaa-b4f6784cd-m6cx4            17m          419Mi
test-user-uaa-58d65c79c-c4nhg           30m          438Mi
test-user-uaa-58d65c79c-ptvwv           16m          420Mi
test-user-uaa-58d65c79c-sx48w           27m          427Mi
test-user-umc-8dddc6-7njpb              12m          423Mi
test-user-umc-8dddc6-f8sdv              0m           0Mi
test-user-umc-8dddc6-nxzdr              9m           410Mi
test-visual-model-56d4447bdd-6xrjp      1011m        931Mi
test-workflow-855f6c4496-zlj4s          1372m        896Mi

// 将资源占用过大的服务进行缩减
$ kubectl scale deploy test-visual-model --replicas=0 -n test-all-service

// 同理，针对内部占用资源过多的情况，进行缩减

// 最终结果
$ kubectl top node
NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
192.168.232.232   1356m        33%    7651Mi          48%
192.168.232.233   292m         7%     7732Mi          48%
192.168.232.234   290m         7%     7099Mi          44%
192.168.232.239   584m         14%    7846Mi          49%
192.168.232.241   262m         6%     5860Mi          37%

```

缩减完成后，重新提交Eureka的部署配置信息，进行CI/CD构建部署。

最终结果如下：

```
$ kubectl get pod -A -o wide  | grep eureka
test-basic-eureka    eureka-0                                                   1/1     Running            0          3m37s   10.42.0.253   192.168.232.241   <none>           <none>
test-basic-eureka    eureka-1                                                   1/1     Running            0          7m21s   10.42.2.91    192.168.232.232   <none>           <none>
test-basic-eureka    eureka-2                                                   1/1     Running            0          10m     10.42.1.16    192.168.232.239   <none>           <none>

```
调整部署后，eureka对应的三台服务器已经部署到含有"basic-eureka=eureka"标签的机器上了。

那么gateway部分来个照此办理，这里只贴gateway的配置了。如下：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: gateway
  namespace: test-basic-gateway
  labels:
    app: gateway
spec:
  type: NodePort
  ports:
    - port: 19020
      name: tcp
      targetPort: 19020
      nodePort: 31333
  selector:
    app: gateway
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway
  namespace: test-basic-gateway
spec:
  minReadySeconds: 10             # 等待10秒后进行操作
  revisionHistoryLimit: 10        # 保留10个滚动版本的历史记录
  strategy:
    type: RollingUpdate           # 滚动更新配置
    rollingUpdate:
      maxUnavailable: 0           # 保证高可用，不允许有无法使用的pod
      maxSurge: 1
  replicas: 3
  selector:
    matchLabels:
      app: gateway
  template:
    metadata:
      labels:
        app: gateway
    spec:
      affinity:                                        # 设置节点的亲和性
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: basic-eureka            # 根据标签的名称进行设置
                    operator: In
                    values:
                      - eureka
      containers:
        - name: gateway
          image: 192.168.232.159:5000/gateway-k8s:$BUILD_NUMBER
          imagePullPolicy: Always
          lifecycle:
            preStop:
              httpGet:
                port: 19020
                path: /spring/shutdown
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 19020
            initialDelaySeconds: 180
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 19020
            initialDelaySeconds: 180
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          resources:
            requests:
              memory: 1Gi
              cpu: 500m
            limits:
              memory: 1.5Gi
              cpu: 750m
          ports:
            - containerPort: 19020
          env:
            - name: IP_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NACOS_IP
              value: 192.168.232.194:18848
            - name: NACOS_NAMESPACE
              value: 4cca5bd9-730b-499b-828e-adsafadfs78d8
            - name: SKYWALKING_NAMESPACE
              value: test-k8s-test
            - name: SKYWALKING_TARGET_SERVICE_NAME
              value: test-gateway
            - name: SKYWALKING_IP_PORT
              value: 192.168.232.163:11800
            - name: CHANNEL
              value: standalone

```

2. UAA服务和UMC的部署调整

针对USER-UAA以及UMC服务二者进行调整，均匀部署到标签为"deploy=worker"的node节点上。opt相关的三个服务，以同样的方式进行处理。首先设置node节点所在标签，如下：

```
$ kubectl label node 192.168.232.233 192.168.232.234 192.168.232.241 deploy=worker
node/192.168.232.233 labeled
node/192.168.232.234 labeled
node/192.168.232.241 labeled

```

以UMC项目为例子进行部署，既要使用node的亲和性，又要使用pod之间的反亲和性。配置信息如下：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: test-user-umc
  namespace: test-all-service
  labels:
    app: test-user-umc
spec:
  ports:
    - port: 19070
      name: tcp
      targetPort: 19070
  selector:
    app: test-user-umc
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-user-umc
  namespace: test-all-service
spec:
  minReadySeconds: 10
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  replicas: 3
  selector:
    matchLabels:
      app: test-user-umc
  template:
    metadata:
      labels:
        app: test-user-umc
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:        # 弱化亲和性
            - podAffinityTerm:
                topologyKey: kubernetes.io/hostname               # 根据pod的标签进行设置反亲和性
                labelSelector: 
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - test-user-umc
              weight: 1
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:        # 强制亲和性
            nodeSelectorTerms:
              - matchExpressions:
                  - key: deploy            # 根据标签的名称进行设置亲和性
                    operator: In
                    values:
                      - worker
      containers:
        - name: user-umc
          image: 192.168.232.159:5000/user-umc-k8s:$BUILD_NUMBER
          imagePullPolicy: Always
          lifecycle:
            preStop:
              httpGet:
                port: 19070
                path: /spring/shutdown
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 19070
            initialDelaySeconds: 180
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 19070
            initialDelaySeconds: 180
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          resources:
            requests:
              memory: 1Gi
              cpu: 350m
            limits:
              memory: 1.5Gi
              cpu: 1000m
          ports:
            - containerPort: 19070
          env:
            - name: IP_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NACOS_IP
              value: 192.168.232.194:18848
            - name: NACOS_NAMESPACE
              value: 4cca5bd9-730b-499b-828e-87590878986
            - name: SKYWALKING_NAMESPACE
              value: test-k8s-test
            - name: SKYWALKING_TARGET_SERVICE_NAME
              value: test-user-umc
            - name: SKYWALKING_IP_PORT
              value: 192.168.232.163:11800
            - name: CHANNEL
              value: standalone

```



### 最终总结

* 什么是亲和性和反亲和性？

* pod部署需要注意的事项？如何根据场景进行决定？

* 番外/参考链接