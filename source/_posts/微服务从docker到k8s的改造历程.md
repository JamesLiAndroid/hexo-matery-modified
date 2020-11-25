---
title: 微服务从docker到k8s的改造历程
top: false
cover: false
toc: true
mathjax: true
date: 2020-04-04 12:21:47
password:
summary:
tags:
categories:
---

# 微服务从docker到k8s的改造历程

## 现状

目前，我们所有的微服务已经全部用容器进行部署，接入了以jenkins+gitlab+sonar为主的自动构建部署体系，能够实现自动打包，推送到对应服务器上进行部署。整个体系能正常走通，比较符合我们目前的使用需求。但是依旧面临以下问题：

1. 调试不方便。需要先将开发环境线上服务停止后，将自己开发机的服务启动并注册到线上环境。调试方式不优雅，且影响当前服务的正常使用。
2. 针对容器的资源限定做的不好，例如jvm内存占用过高的问题。
3. 在运行的服务集群中，不能做到服务的弹性伸缩，也不能很好的控制流量的转移与分配。

针对于以上问题，我们决定引入k8s，先完成k8s的安装部署和微服务升级到k8s中，然后使用istio进行网络层面的改造。

## 原则和目标

优先在k8s上部署单个微服务项目，临时决定剥离存储层面的内容。最终达到以三集群部署为主，即每个微服务部署三套，同时对外提供服务。

目前采用的策略是，临时将所有微服务相关的组件和服务，均一股脑放到k8s中，先做到这个中间形态。在这个阶段完成之后，通过对各项基础设施组件进行拆除，例如配置中心、服务注册和发现中心、网关等，将这些基础设施性质的内容沉淀到PaaS层，也就是目前依托的k8s层面的组件进行代替，例如istio的使用，最终目标是形成一套语言无关的微服务支撑平台。

在基础设施沉淀后，需要进行整个监控系统的优化，要形成从主机、集群、服务、存储、网络、日志信息等各个层面的监控。尤其是在后续形成多个k8s集群的时候，要统一进行管理。

最终目标，把这一部分扩展，构建统一的运维平台，利用该运维平台可以进行动态申请资源，配置各个服务的动态伸缩规则，真正做到4个9的高可用！

## 各微服务业务部署

### 改造要点

1. 对于Eureka、nacos、gateway这些基础组件优先部署到k8s集群，优先实现eureka的互相注册。
2. 对于微服务远程接口调用来说，在基础设施不变的情况下，远程接口调用暂时不需要更改。后续变更基础设施时，需要使用完整url链接进行调用，尤其是Feign组件进行远程接口调用的变更。服务间通信，跨命名空间的服务通信，需要使用完整的服务名进行调用，不能使用ClusterIP进行调用。
3. 关于存储层从k8s剥离，使用原生方式部署，启用高可用方式。对于微服务来说，要允许连接到集群外部的数据库、缓存、分布式存储等存储工具。
4. 要求重新配置服务监控信息，对于微服务的日志收集，统一展示，对于各个微服务的链路监控信息。
5. 要求能够将线下开发机接入集群进行调试，方便日后开发环境的联调工作。
6. 要求能够接入目前的CI/CD体系，能够实现服务自动部署到k8s集群。

### namespace划分

1. 基础设施：test-basic（nacos、Eureka、gateway），例如：test-basic-nacos
2. 现有release版本的服务：test-all-services
3. 测试微服务：test-test-business
4. 对外连接mysql、redis、mongodb等：test-basic-<服务名称>-outer，例如：test-basic-mysql-outer

nacos、Eureka均需要对外暴露，使用nodePort的模式对外暴露。至于gateway来说，对于后端服务而言，由于是内部进行调用，可以暂不对gateway开放暴露服务，但是对于前端服务，仍然需要对外暴露，同样使用nodePort方式。由于目前没有独立的DNS服务以及尚未引入https，所以这里采用nodePort的方式进行。

### 基础设施划分

#### 1. 针对nacos的改造

参考阿里官方的文档：https://github.com/nacos-group/nacos-k8s

核心要义：用单机数据库，部署headless无头应用程序，利用ingress对外提供负载均衡服务。

首先创建nfs存储并进行挂载，随后进行操作的时候将地址定位到具体的节点地址上。参考{% post_link k8s存储搭建.md 存储层面-nfs的安装搭建 %}。

然后下载[nacos-k8s](https://github.com/nacos-group/nacos-k8s.git)源码，首先转到deploy/nfs文件夹中，依次安装文件夹中的三个文件：

    $ kubectl create ns test-basic-nacos
    $ kubectl apply -f rbac.yaml -n test-basic-nacos
    $ kubectl apply -f class.yaml -n test-basic-nacos
    // 注意：要先修改deployment.yaml中的文件，将nfs部分的设置替换为自己的
    $ kubectl apply -f deployment.yaml -n test-basic-nacos

其次转到deploy/mysql文件夹，执行下面的命令：

    // 注意：要先修改mysql-nfs.yaml中的文件，将nfs部分的设置替换为自己的
    $  kubectl apply -f mysql-nfs.yaml -n test-basic-nacos

最后，转到deploy/nacos文件夹中，执行下面的命令：

    // 需要先对文件做如下修改，注释掉
    //  #annotations:
    //  #  service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
    // 在spec部分添加publishNotReadyAddresses: true
    //spec:
    //  #type: NodePort
    //  publishNotReadyAddresses: true
    $ kubectl apply -f nacos-pvc-nfs.yaml -n test-basic-nacos

在该配置文件中，StatefulSet的配置中，serviceName被指定为*nacos-headless*。

除此之外，需要将nacos对外进行暴露，于是我自己在deploy文件夹中创建了ingress文件夹，并在ingress文件夹中创建nacos-ingress.yaml，操作如下：

    $ vim nacos-ingress.yaml
    // 输入以下信息
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: dashboard-nacos-nginx
      namespace: test-basic-nacos
      annotations:
        kubernetes.io/ingress.class: nginx
        ingress.kubernetes.io/ssl-redirect: "false"
    spec:
      rules:
      - http:
          paths:
          - path: /nacos
            backend:
              serviceName: nacos-headless
              servicePort: 8848

依托于集群中已有的ingress-nginx支持，通过ingress对外暴露服务，这样访问集群内任何一台机器的nacos服务均可，例如：10。0.88.241/nacos，即可看到所有的nacos服务。

对于内部访问，nacos的服务内部访问地址

nacos-0.nacos-headless.test-basic-nacos.svc.cluster.local:8848
nacos-1.nacos-headless.test-basic-nacos.svc.cluster.local:8848
nacos-2.nacos-headless.test-basic-nacos.svc.cluster.local:8848

<!-- 尝试将mysql主从配置分别写入不同的nfs生成的pv和pvc，发现无法进行访问，是rbac的关系，联想之前的内容mysql主从写到了同一个文件夹。

分离mysql主从写入不同的nfs位置，创建多个Deployment和storageclass，然后对mysql主从创建特定的pv和pvc，进行写入。 -->

<!-- https://www.cnblogs.com/cuishuai/p/9152277.html -->


<!-- 不再使用主从模式，！ -->


默认端口号为：8848

注意在配置文件中，填写内部地址时，首先需要加上**http://**，然后是多个nacos连接地址的时候，需要用**","*分割！

例如完整的配置链接如下：

    http://nacos-0.nacos-headless.test-basic-nacos.svc.cluster.local:8848,http://nacos-1.nacos-headless.test-basic-nacos.svc.cluster.local:8848,http://nacos-2.nacos-headless.test-basic-nacos.svc.cluster.local:8848

这样nacos就正式配置完成了。但是目前使用的nacos版本偏高，为1.2.0版本，当前开发环境使用的是1.1.2版本的nacos。随后将开发环境中的配置文件导出，添加到现在的nacos中！另外需要从开发环境中导入测试服务相关的配置文件，例如business01、business02相关的配置信息。

**注意：**如果拉取镜像存在问题时，可以通过管理机192.168.88.240拉取，拉取后推送到各个集群主机即可。

例如针对MySQL的镜像，可以通过下面的命令进行，例如在192.168.88.240机器上执行下面的命令，192.168.88.232为某台集群机器：

    // 镜像拉取
    $ docker pull mysql:5.7
    // 镜像推送，目标机器为192.168.88.232，其它机器同理
    $ docker save mysql:5.7 | bzip2 | ssh -p 15555 centos@192.168.88.232 'bunzip2 | docker load'

### Eureka、Gateway

nacos作为配置中心使用，需要对Eureka和gateway进行改造，使它们的docker镜像都能动态配置nacos连接信息。首先修改下docker镜像信息，原来的Dockerfile为

```Dockerfile
FROM openjdk:8-jre-alpine

WORKDIR /app

#ARG DEPENDENCY=/target
# Copy project dependencies from the build stage
COPY ./target/test-serverregister-0.0.1-SNAPSHOT.jar /app

ENTRYPOINT ["java","-Dspring.profiles.active=standalone","-jar","test-serverregister-0.0.1-SNAPSHOT.jar"]

```

现在的Dockerfile为

```Dockerfile
FROM openjdk:8-alpine

WORKDIR /app

# 添加全局变量
ENV CHANNEL=""
ENV IP_ADDR=""

# nacos配置
# 默认为测试环境的地址
ENV NACOS_IP="192.168.88.161:18848"
ENV NACOS_NAMESPACE="858b37b1-35be-4564-a24e-dc2c322d5784"
ENV CHANNEL="standalone"

#ARG DEPENDENCY=/target
# Copy project dependencies from the build stage
COPY ./target/test-serverregister-0.0.1-SNAPSHOT.jar /app

ENTRYPOINT ["sh", "-c", "java -Dspring.profiles.active=${CHANNEL} -Dspring.cloud.client.ip-address=${IP_ADDR} -Dnacos_ip=${NACOS_IP} -Dnacos_namespace=${NACOS_NAMESPACE} -jar test-serverregister-0.0.1-SNAPSHOT.jar"]

```

主要是修改启动命令信息，添加启动的配置信息。

下面开始Eureka的安装，安装之前需要注意，当前未搭建k8s中的私有的harbor镜像仓库，继续使用之前的docker registry镜像仓库，需要在节点机器上配置镜像仓库地址，修改/etc/docker/daemon.json文件，如下：

```
$ vim /etc/docker/daemon.json
// 添加以下有效信息
{
    "registry-mirrors": [
        "https://docker.mirrors.ustc.edu.cn",
        "https://registry.cn-hangzhou.aliyuncs.com",
        "https://registry.docker-cn.com"
    ],
    "insecure-registries": [
        "192.168.88.159:5000"
    ],
    "live-restore": true,
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "50m",
        "max-file": "3"
    }
}


```

随后重启docker服务

    $ sudo systemctl restart docker

下面针对Eureka服务进行配置。

首先编辑Eureka的安装文件，修改服务注册中的nacos配置选项，然后创建eureka的部署文件eureka.yaml，最后修改nacos中的关于eureka的配置信息，主要修改url地址信息。

```
$ cd ./Eureka && vim k8s-eureka.yaml

// 在文件中输入以下内容

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
      containers:
        - name: eureka
          image: 192.168.88.159:5000/server-register-k8s:4
          ports:
            - containerPort: 19011
          resources:
            limits:
              memory: 1Gi
          env:
            - name: IP_ADDR
              #value: eureka-*.eureka-headless.test-basic-eureka.svc.cluster.local
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NACOS_IP
              #value: nacos-headless.test-basic-nacos
              value: http://nacos-0.nacos-headless.test-basic-nacos.svc.cluster.local:8848,http://nacos-1.nacos-headless.test-basic-nacos.svc.cluster.local:8848,http://nacos-2.nacos-headless.test-basic-nacos.svc.cluster.local:8848
            - name: NACOS_NAMESPACE
              value: test-dev
            - name: CHANNEL
              value: standalone
  podManagementPolicy: "Parallel"

```
需要注意两个点，一个是构建Eureka镜像，是单独在jenkins服务中创建的构建任务进行生成的，和日常用的Eureka镜像并不太一样。单独创建k8s-dev-apps工作空间，创建eureka的构建服务，依托之前创建的服务来进行构建新的镜像。第二个就是配置nacos集群ip地址，如上面所示。第三就是指定配置文件信息。

在完成上面工作后，在命令行执行以下命令，安装Eureka服务：

```
$ kubectl create ns test-basic-eureka

$ kubectl apply -f k8s-eureka.yaml -n test-basic-eureka

```

完成后，需要在nacos中导入目前沿用的配置信息并进行修改，由于我这里指定的是standalone尾缀的配置文件，需要找到**test-register-standalone.yml**进行修改，最终修改后如下：

```yaml

eureka:
  instance: 
    preferIpAddress: false
    # hostname: ${spring.cloud.client.ip-address}
    # appname: ${spring.application.name}
  client:
    register-with-eureka: true
    fetch-registry: true
    serviceUrl:
      defaultZone: http://test:test2019@eureka-0.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/,http://test:test2019@eureka-1.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/,http://test:test2019@eureka-2.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/
  server:
    enable-self-preservation: false      
server:
  port: 19011

```

主要是配置Eureka的连接信息，使他们进行相互注册。修改完成后，点击publish按钮，发布后，服务会自动更新配置信息。

最后设置对外访问，使用nodePort模式，配置如下：

```
$ vim eureka-nodeport.yaml
// 设置以下信息
apiVersion: v1
kind: Service
metadata:
  name: eureka-nodeport
  namespace: test-basic-eureka
spec:
  selector:
    app: eureka
  type: NodePort
  ports:
    - port: 19011
      targetPort: 19011
      nodePort: 31011
      protocol: TCP

// 执行
$ kubectl apply -f eureka-nodeport.yaml -n test-basic-eureka

```

这样就完成了eureka集群的部署，访问任何一台集群机器内的eureka服务，如192.168.88.232:31011，即可看到该服务信息，应该看到当前相互注册的三个eureka服务。

<!-- eureka不使用无状态服务（headless）进行部署！！！！ -->

#### gateway部署

<!-- 针对镜像的改造，将java启动命令的信息拆出来，优化分级。 -->

<!-- 问题：配置信息问题，配置信息可以放在nacos中，也可以放在启动文件的yaml中，这个如何进行权衡？ -->

注意：设置连接外部服务时，需要将service名称和endpoints名称设置为一致，才能使服务生效。

参考链接：https://qingmu.io/2019/08/07/Run-Spring-Cloud-Gateway-on-kubernetes-2/

针对gateway的代码并不需要进行变更，需要变更Dockerfile镜像文件，改造方式同Eureka服务，只是jar包名称不同。

对gateway的配置信息进行修改，主要是修改Eureka的连接信息，修改**test-gateway-standalone.yml**如下：

```
server:
  port: 19020

eureka:
  client:
    service-url:
      defaultZone: http://test:test2019@eureka-0.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/,http://test:test2019@eureka-1.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/,http://test:test2019@eureka-2.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/

```

然后针对gateway重新创建构建，利用该构建创建新的docker镜像，进行部署。在服务器上编写部署文件如下：

```
$ vim gateway.deployment.yaml

// 输入以下信息

---
apiVersion: v1
kind: Service
metadata:
  name: gateway
  namespace: test-basic-gateway
  labels:
    app: gateway
spec:
  ports:
    - port: 19020
      name: tcp
      targetPort: 19020
  selector:
    app: gateway
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway
  namespace: test-basic-gateway
spec:
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  replicas: 3
  selector:
    matchLabels:
      app: gateway
  template:
    metadata:
      labels:
        app: gateway
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - app-gateway
              weight: 1
      containers:
        - name: gateway
          image: 192.168.88.159:5000/gateway-k8s:1
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
            initialDelaySeconds: 300
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 19020
            initialDelaySeconds: 300
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          resources:
            requests:
              memory: 1Gi
            limits:
              memory: 1Gi
          ports:
            - containerPort: 19020
          env:
            - name: IP_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NACOS_IP
              value: http://nacos-0.nacos-headless.test-basic-nacos.svc.cluster.local:8848,http://nacos-1.nacos-headless.test-basic-nacos.svc.cluster.local:8848,http://nacos-2.nacos-headless.test-basic-nacos.svc.cluster.local:8848
            - name: NACOS_NAMESPACE
              value: test-dev
            - name: CHANNEL
              value: standalone

```

设置完成后，需要对gateway服务对外进行暴露，由于后续要通过gateway访问其它的服务，这里使用域名的方式进行配置，防止把访问的url信息也带到gateway的解析过程中，导致服务找不到，如下：

<!-- 初始使用带url的路径的ingress方式进行测试，发现访问不通，后续将路径去掉，使用域名的方式进行访问，这样可以对要部署的内容进行正常访问。 -->

```

$ vim ingress-gateway.yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gateway-nginx
  namespace: test-basic-gateway
  annotations:
    kubernetes.io/ingress.class: nginx
    ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: www.test-gateway.net
    http:
      paths:
      - path: /
        backend:
          serviceName: gateway
          servicePort: 19020

```
设置完成后，需要在本地配置host文件，指向gateway的地址，如下：

```
// 在host文件中添加下面的配置

192.168.88.232 www.test-gateway.net
192.168.88.239 www.test-gateway.net
192.168.88.241 www.test-gateway.net

```

这样就配置完成了，首先去已经部署的eureka的地址，查看eureka中是否已经存在该服务。然后访问gateway的信息，例如使用postman直接访问www.test-gateway.net，这里会报错，显示服务调用失败，如下：

```

{
    "message": "服务调用失败",
    "code": -1,
    "timestamp": 1585209902994
}

```

说明配置成功！

**注意：**需要对前面的内容进行说明，将gateway对外暴露，使用nodePort的方式，原因是如果单纯使用“service名称+namespace名称+端口号”的形式进行调用，会出现无法连接的情况。

nodePort方式如下，修改gateway的部署文件：

```
$ vim gateway.deployment.yaml

// 修改以下信息，添加type为NodePort，指定nodePort端口号

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


```

这样在调用gateway的时候，可以通过192.168.88.239:31333来进行调用。nodePort方式和ingress方式可以共存。

### 各微服务的改造，以测试项目为例子进行改造

下面拿出一个测试案例，该案例中有两个微服务BUSINESS-01和BUSINESS-02，有服务间进行调用的接口，以此来测试服务是否能正常运行。

首先需要对项目进行改造，切换分支到feature.ICP.change_name，使其能够符合现行的框架约定，主要是添加nacos的调用信息，剥离配置文件到nacos中。然后选择在jenkins创建构建项目，打包生成镜像信息。

**注意：在拥有多个子项目且有子项目之间互相依赖时，需要先在最外层执行mvn clean install，然后到某个单项微服务中去执行构建命令。**

随后开始编写第一个项目的部署文档，如下：

```
$ vim test-business-01.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: test-business-01
  namespace: test-test-business
  labels:
    app: test-business-01
spec:
  ports:
    - port: 19030
      name: tcp
      targetPort: 19030
  selector:
    app: test-business-01
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-business-01
  namespace: test-test-business
spec:
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  replicas: 1
  selector:
    matchLabels:
      app: test-business-01
  template:
    metadata:
      labels:
        app: test-business-01
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - app-test-business-01
              weight: 1
      containers:
        - name: test-business-01
          image: 192.168.88.159:5000/test-business01-k8s:6
          imagePullPolicy: Always
          lifecycle:
            preStop:
              httpGet:
                port: 19030
                path: /spring/shutdown
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 19030
            initialDelaySeconds: 300
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 19030
            initialDelaySeconds: 300
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          resources:
            requests:
              memory: 1Gi
            limits:
              memory: 1Gi
          ports:
            - containerPort: 19030
          env:
            - name: IP_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NACOS_IP
              value: http://nacos-0.nacos-headless.test-basic-nacos.svc.cluster.local:8848,http://nacos-1.nacos-headless.test-basic-nacos.svc.cluster.local:8848,http://nacos-2.nacos-headless.test-basic-nacos.svc.cluster.local:8848
            - name: NACOS_NAMESPACE
              value: test-dev
            - name: CHANNEL
              value: standalone

```

首先部署单个项目，操作如下：

```
// 创建namespace
$ kubectl create ns test-test-business

// 创建服务
$ kubectl apply -f test-business-01.yaml -n test-test-business

// 获取test服务信息
$ kubectl get pod -n test-test-business
NAME                                READY   STATUS    RESTARTS   AGE
test-business-01-656c496946-g5l26   1/1     Running   0          149m

```

这样第一个服务部署成功，查看Eureka的服务信息，服务已经上线，服务名称为**TEST-BUSINESS01**，说明部署成功。

然后部署另外一个项目，配置信息与上面基本一致，区别在于服务名称由business01变更为business02，镜像也有所改变，为192.168.88.159:5000/test-business02-k8s:9，端口号根据配置信息更改为19040。操作与上面一致，最终结果如下：

```
$ kubectl get pod -n test-test-business
NAME                                READY   STATUS    RESTARTS   AGE
test-business-01-656c496946-g5l26   1/1     Running   0          149m
test-business-02-58655d78c6-5x8zq   1/1     Running   0          156m

```

查看Eureka的服务信息，第二个服务已经上线，说明部署成功。

随后使用postman或者命令行的方式进行调用，操作如下：

![](接口请求.png)

最后看到可以进行调用。

### 连接外部的MySQL、Redis、MongoDB

以MySQL为例子，演示k8s中的微服务连接外部的存储。

```
$ vim mysql-connection/mysql-endpoints.yaml

// 输入以下信息

apiVersion: v1
kind: Endpoints
metadata:
  name: mysql-outer
  namespace: test-basic-mysql-outer
subsets:
  - addresses:
    - ip: 192.168.88.190
    ports:
      - port: 3306

```


将k8s集群外部的mysql连接配置为endpoint，执行命令：

    $ kubectl create ns test-basic-mysql-outer
    $ kubectl apply -f mysql-connection/mysql-endpoints.yaml -n test-basic-mysql-outer


随后将该endpoint作为服务提供给k8s集群中的微服务使用。如下：

```
$ vim mysql-connection/mysql-service.yaml

// 输入以下信息

apiVersion: v1
kind: Service
metadata:
  name: mysql-outer
  namespace: test-basic-mysql-outer
spec:
  ports:
    - port: 3306

```

执行以下命令：

    kubectl apply -f mysql-connection/mysql-service.yaml -n test-basic-mysql-outer

这样MySQL的外部连接就配置好了，需要我们对MySQL的连接字符串进行配置，原来的连接配置字符串为：

    jdbc:mysql://192.168.88.159:3306/elk?useSSL=false&useUnicode=true&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC

新的连接字符串配置为：

    jdbc:mysql://mysql-outer.test-basic-mysql-outer:3306/elk?useSSL=false&useUnicode=true&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC

**注意：**必须使用service名称+namespace名称的方式进行连接，否则会存在找不到服务的情况！例如redis配置后的连接为：

    redis-outer.test-basic-redis-outer

### 升级和回滚操作

首先需要对部署的deployment进行跟踪。

第一，可以直接修改yaml配置文件中的镜像信息，使用kubectl apply -f命令执行升级。

第二，使用kubectl edit命令来进行升级，操作类似vim，替换image位置的镜像信息，修改后即可生效

实际操作中，根据我们在对应namespace中创建的deployment信息，进行操作，例如对gateway进行扩容，操作命令如下：

    kubectl scale deployment test-gateway --replicas=5 -n test-basic-gateway

将之前的三个pod扩展为五个pod的操作。

操作方式参考：https://www.cnblogs.com/Tempted/p/7831604.html

### 关于微服务编码上的影响

1. 链接需要更改为服务名称，尤其连接nacos、eureka的连接，必须使用完整的链路进行，这部分只需要修改配置信息即可，也不需要进行额外配置。
2. 连接k8s集群外部的数据库、缓存等内容，需要先配置内部连接的EndPoint，提供连接的service，内部通过“service名称+namespace名称”进行调用。

### 微服务：分布式id的部署示例

1. zookeeper的安装

首先创建pv，创建存储空间，使用本地文件路径的方式，操作如下


```
$ cd zookeeper && vim pv-zk.yaml

// 输入以下信息

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-zk1
  annotations:
    volume.beta.kubernetes.io/storage-class: "anything"     
  labels:
    type: local
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/opt/data/zookeeper"             
  persistentVolumeReclaimPolicy: Recycle
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-zk2
  annotations:
    volume.beta.kubernetes.io/storage-class: "anything"
  labels:
    type: local
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/opt/data/zookeeper"              
  persistentVolumeReclaimPolicy: Recycle
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-zk3
  annotations:
    volume.beta.kubernetes.io/storage-class: "anything"
  labels:
    type: local
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/opt/data/zookeeper"
  persistentVolumeReclaimPolicy: Recycle

// 执行下面的命令进行生效

$ kubectl create namespace test-zookeeper
$ kubectl apply -f pv-zk.yaml -n test-zookeeper

```

随后创建zookeeper服务，操作如下：

```
$ vim k8s-zk.yaml

// 输入以下信息

---
apiVersion: v1
kind: Service
metadata:
  name: zk-hs
  labels:
    app: zk
spec:
  ports:
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zk
---
apiVersion: v1
kind: Service
metadata:
  name: zk-cs
  labels:
    app: zk
spec:
  ports:
  - port: 2181
    name: client
  selector:
    app: zk
---
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  selector:
    matchLabels:
      app: zk
  maxUnavailable: 1
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zk
spec:
  selector:
    matchLabels:
      app: zk
  serviceName: zk-hs
  replicas: 3
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: OrderedReady   
  template:
    metadata:
      labels:
        app: zk
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - zk
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: kubernetes-zookeeper
        imagePullPolicy: Always
        image: "k8s.gcr.io/kubernetes-zookeeper:1.0-3.4.10"
        resources:
          requests:
            memory: "1Gi"
            cpu: "0.5"
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
        command:
        - sh
        - -c
        - "start-zookeeper \
          --servers=3 \
          --data_dir=/var/lib/zookeeper/data \
          --data_log_dir=/var/lib/zookeeper/data/log \
          --conf_dir=/opt/zookeeper/conf \
          --client_port=2181 \
          --election_port=3888 \
          --server_port=2888 \
          --tick_time=2000 \
          --init_limit=10 \
          --sync_limit=5 \
          --heap=512M \
          --max_client_cnxns=60 \
          --snap_retain_count=3 \
          --purge_interval=12 \
          --max_session_timeout=40000 \
          --min_session_timeout=4000 \
          --log_level=INFO"
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        volumeMounts:
        - name: datadir
          mountPath: /var/lib/zookeeper
  volumeClaimTemplates:
  - metadata:
      name: datadir
      annotations:
        volume.beta.kubernetes.io/storage-class: "anything"  
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 2Gi

// 执行下面的命令生效
$ kubectl apply -f k8s-zk.yaml -n test-zookeeper

```

2. 分布式id服务的部署

该分布式id项目基于[美团的开源的分布式id生成器](https://github.com/Meituan-Dianping/Leaf)，使用雪花算法生成id信息。

在zookeeper安装完成后，继续进行分布式id的部署，分布式id使用雪花算法生成id信息，需要依赖zookeeper服务进行生成。首先更改配置文件，连接zookeeper如下：

```properties

spring.freemarker.cache=false
spring.freemarker.charset=UTF-8
spring.freemarker.check-template-location=true
spring.freemarker.content-type=text/html
spring.freemarker.expose-request-attributes=true
spring.freemarker.expose-session-attributes=true
spring.freemarker.request-context-attribute=request
# 1.Eureka配置信息
#spring.application.name=distributedIDGenerator2
#spring.application.name=DISTRIBUTEDIDGENERATOR
spring.application.name=test-id-generator
# 注意：在这里更改时，需要在leaf.properties配置文件中更改端口信息
server.port=8080
eureka.instance.prefer-ip-address=true
eureka.instance.instance-id=${spring.application.name}@${spring.cloud.client.ip-address}:${server.port}
eureka.client.service-url.defaultZone=http://test:test2019@eureka-0.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/,http://test:test2019@eureka-1.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/,http://test:test2019@eureka-2.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/
eureka.instance.ipAddress=${spring.cloud.client.ip-address}
#eureka.client.service-url.defaultZone=http://test:test2019@192.168.123.93.153:19011/eureka/

# 2. Feign中的Ribbon配置
# 让 Hystrix 的超时时间大于 Ribbon 的超时时间，否则 Hystrix 命令超时后，该命令直接熔断，重试机制就没有任何意义了。
ribbon.ConnectTimeout=500
ribbon.ReadTimeout=2000
distributed-id-generator.ribbon.MaxAutoRetries=2
# 3  Hystrix配置
# 配置全局的超时时间
hystrix.command.default.execution.isolation.thread.timeoutinMilliseconds=5000
hystrix.command.distributed-id-generator.execution.isolation.thread.timeoutinMilliseconds=5000
# 启用全局hystrix配置
feign.hystrix.enabled=true


# 4. id生成器自己的配置信息
leaf.name=com.sankuai.leaf.opensource.test
leaf.segment.enable=false
#leaf.jdbc.url=
#leaf.jdbc.username=
#leaf.jdbc.password=

# 启用雪花算法，设置zookeeper的地址
leaf.snowflake.enable=true
#### 主要是修改这个位置的zookeeper连接，使用服务名+namespace名称+端口号的方式进行调用
leaf.snowflake.zk.address=zk-cs.test-zookeeper:2181
leaf.snowflake.port=8080



```

这里zookeeper地址指定为**zk-cs.test-zookeeper:2181**。下面开始部署id生成器：

```
$ vim distributed-id-gen.yaml

// 输入以下信息

---
apiVersion: v1
kind: Service
metadata:
  name: test-id-generator
  namespace: test-all-service
  labels:
    app: test-id-generator
spec:
  ports:
    - port: 8080
      name: tcp
      targetPort: 8080
  selector:
    app: test-id-generator
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-id-generator
  namespace: test-all-service
spec:
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  replicas: 3
  selector:
    matchLabels:
      app: test-id-generator
  template:
    metadata:
      labels:
        app: test-id-generator
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - app-test-id-generator
              weight: 1
      containers:
        - name: distributed-id-gen
          image: 192.168.88.159:5000/distributed-id-gen-k8s:3
          imagePullPolicy: Always
          lifecycle:
            preStop:
              httpGet:
                port: 8080
                path: /spring/shutdown
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 300
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 300
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          resources:
            requests:
              memory: 500Mi
            limits:
              memory: 1Gi
          ports:
            - containerPort: 8080
          env:
            - name: IP_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NACOS_IP
              value: http://nacos-0.nacos-headless.test-basic-nacos.svc.cluster.local:8848,http://nacos-1.nacos-headless.test-basic-nacos.svc.cluster.local:8848,http://nacos-2.nacos-headless.test-basic-nacos.svc.cluster.local:8848
            - name: NACOS_NAMESPACE
              value: test-dev
            - name: SKYWALKING_NAMESPACE
              value: test-test
            - name: SKYWALKING_TARGET_SERVICE_NAME
              value: test-ditributed-id-gen
            - name: SKYWALKING_IP_PORT
              value: 192.168.88.163:11800
            - name: CHANNEL
              value: standalone

// 执行下面的信息生效
$ kubectl apply -f distributed-id-gen.yaml -n test-all-service

```

这样就部署完成的分布式id服务！

## 基础镜像仓库--Harbor（待完成）

1. helm 配置

2. harbor文件配置修改

3. pv和pvc创建

4. 镜像拉取并推送到各个节点

5. 访问

6. 导入registry镜像数据--基础镜像和数据



## CI/CD构建改造（待完成）

问题：对于镜像版本的管理方式

使用jenkins的k8s插件进行构建。

微服务变更：

1. 修改配置信息，主要针对Eureka的连接、数据库连接、缓存连接、mongodb的连接。
2. k8s部署的yaml信息进行配置，最好考虑用模板的方式进行，能传入版本号，保持和jenkins的版本一致。
3. 镜像名称更改并重新打包。
4. 检查远程调用的内容，查看是否需要修改。

jenkins构建时，保留前面的流程，从docker部署的位置进行修改，以数据字典为例，删除含有以下代码的shell脚本执行块：

```
#--------------------------------------------------------------------------

# 判断是否存在镜像
# docker ps -a | grep -w data-dictionary-server &> /dev/null
# 如果存在先停止运行并删除镜像
if [ "$(docker ps -a | grep -w data-dictionary-server)" ]; then
    echo "data-dictionary-server is exsited!!"
    docker stop `docker ps -a | grep -w data-dictionary-server | awk '{print $1}'`
    docker rm `docker ps -a | grep -w data-dictionary-server | awk '{print $1}'`
    docker image rm `docker images | grep -w data-dictionary-server | awk '{print $3}'`
fi

echo "pull the data-dictionary-server image"
docker pull 192.168.88.159:5000/data-dictionary-server:$BUILD_NUMBER
docker run --restart=on-failure:10 -d -p 19090:19090 -e CHANNEL="standalone" -e IP_ADDR="192.168.88.193" -e NACOS_IP="192.168.88.194:18848" -e NACOS_NAMESPACE="aa853012-28dd-404a-a941-e4e36324f615"  -e SKYWALKING_NAMESPACE="test-test" -e SKYWALKING_TARGET_SERVICE_NAME="test-data-dict" -e SKYWALKING_IP_PORT="192.168.88.163:11800" 192.168.88.159:5000/data-dictionary-server:$BUILD_NUMBER

```
保存后，安装jenkins中的kubernetes插件。

**更新：**kubernetes插件并不能将微服务部署到k8s集群中，这里预留两种思路进行：

1. 利用helm工具，以harbor镜像仓库为支撑，微服务打包成docker image后，推送到harbor中。然后使用k8s集群中某台机器，通过ssh登录后，利用helm安装到对应的namespace下。

2. 在k8s集群中选择一台机器，针对gitlab代码仓库中各个微服务的部署文件，例如deployment.yaml，单独拉取该文件信息。在镜像打包完毕后，生成版本信息。将该版本信息，设置到部署文件中，替换原来部署文件的版本信息。使用kubectl apply命令让部署文件生效。替换原来的镜像，使新的镜像生效。

## 后续的微服务改造思路

### 核心思想

语言无关性，将java侧Spring Cloud框架的具体依赖信息，下沉到k8s层面，利用k8s提供的各项功能替换，只在开发业务时选择语言。

### 改造思路

1. Eureka下沉到k8s，利用k8s自身的服务发现进行替代
2. nacos信息使用k8s中的ConfigMap进行替代
3. gateway使用k8s中的网关来替代
4. 建设统一的管理界面，需要对管理面板选型，目前感觉rancher还是不够用
5. 对于监控层面，全链路监控沿用skywalking，日常监控使用promethus+grafana
6. 引入多个k8s集群，解决跨集群通信访问的问题，尽量让关联性强的服务部署在同一个集群中


## 针对k8s的培训计划

0. 先讲解k8s是什么，优秀特性以及如何进行搭建。
1. 先以一个最简单的应用进行培训，例如nginx，穿插讲解涉及到的知识，尤其是对配置文件进行解释，入门级。
2. 讲解具体到使用的基础设施层面，例如Eureka和gateway的时候，如何进行部署，如何进行伸缩，如何进行故障转移。
3. 讲解具体的服务如何部署，并进行联动，设置服务间相互调用的例子，设置服务调用外部存储的例子，讲解如何进行负载均衡，多种均衡方式，如何暴露端口和url使其可以进行对外访问。
4. 讲解前端的部署，静态资源的优化，穿插讲解configMap等知识。
5. 讲解有状态服务和无状态服务的区别，演示mysql单体部署和集群部署，Redis的集群部署。
6. 讲解如何进行线下调试，使用kt-connect工具进行测试。
7. 现有的CI/CD体系如何进行接入。
8. 未涉及的内容统一进行讲解，课程总结。
<!-- 6. 总结所有的k8s相关知识点 -->

## 附录，修改docker daemon.json不重启镜像的操作

vim  clean_exited_docker_containers.sh

docker rm $(docker ps -q -f status=exited)

所有机器添加以下选项，一个是私有仓库，另一个是保证docker daemon进程重启时不要重启其它的正在运行的镜像，但是只对设置后的生效，需要先配置：

    # vim /etc/docker/daemon.json
    // 添加以下信息
    "insecure-registries": [
        "192.168.88.159:5000"
    ],
    "live-restore": true,

还需要添加该公有仓库信息："https://registry.cn-hangzhou.aliyuncs.com",。但是要注意，如果之前没有进行配置，重启docker服务，容器还是会重启，配置后重启docker服务，当前运行的容器不重启。


需要单独搭建Harbor集群，放在单独的k8s镜像中

## 附录，关于jenkins安装插件超时的问题

替换url

https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json

http://mirror.esuni.jp/jenkins/updates/update-center.json

## 附录，持续集成流程修改

0. 前面的打包流程不做变更，删除docker镜像部署的部分，更改为部署到k8s的流程
1. 搭建harbor镜像仓库，配置到helm中
2. 打包推送镜像完毕后，在目标机上通过helm进行部署

目前存在的问题

1. 配置信息的修改，是否把配置信息同项目分离
2. helm所在目标机存在单点故障的情况

替换镜像信息的操作如下：

sed 's#192.168.88.159:5000/distributed-id-gen-k8s:latest#'$BUILDIMG'#' $WORKSPACE/k8s/deployment.yaml | kubectl apply -f -


## 附录：资源不足问题

1. 报错：Warning  FailedScheduling  <unknown>  default-scheduler  0/3 nodes are available: 3 Insufficient cpu.

2. 解决方式：

修改yaml中的cpu设置信息

          resources:
            requests:
              memory: 500Mi
              cpu: 100m
            limits:
              memory: 1Gi
              cpu: 500m

**更正：**直接删除对cpu的资源限制，只对内存进行限制。如下：

          resources:
            requests:
              memory: 500Mi
            limits:
              memory: 1Gi


<!-- ## jenkins中针对sonar的命令调整

1. 临时删除前端中对sonar的代码检测

npm --registry https://registry.npm.taobao.org install -g sonarqube-scanner && sonar-scanner \
        -Dsonar.projectKey=test-frontend-develop \
        -Dsonar.projectName=test-frontend-develop   \
        -Dsonar.host.url=http://192.168.88.159:9000 \
        -Dsonar.login=jenkins    \
        -Dsonar.login=d3830ce5b21ca809290798ed7f093dd3f4396edf \
        -Dsonar.sources=.   \
        -Dsonar.tests=./tests/unit

2. 去掉后端代码中的最前面的

    mvn clean install -DskipTest

调整sonar检测执行的顺序，放在post build之后。 -->

## 关于多个gateway的负载均衡

当部署了前端项目的时候，前端页面将对后端的请求通过nginx转发到后端，这时候nginx无法直接对k8s内的gateway进行请求。

目前gateway使用域名的方式对外暴露，考虑到dns问题，需要使用nodeport的方式对外暴露，但是这样需要在nginx侧设置对这些nodePort的统一代理，保证访问gateway的时候，可以进行负载均衡！

注意：由于arp病毒的影响，容易导致某台机器存在不可用的情况，所以在配置时需要检测机器状态。

<!-- 1. gateway、Eureka基础设施重启
2. 配置信息详细检查
3. 服务重新部署 -->

## 最后需要排查的隐患

1. 关于zookeeper镜像问题，时钟无法同步，时区未设置，差8小时，对分布式id服务的影响，目前未知。----更换zookeeper镜像，重做。
2. 关于服务过多的时候，存在问题，数据库出现：too many connections问题。---拆数据库，不同服务拥有自己的数据库。
3. 镜像版本控制的问题，历史追踪设置。---

## 下一步改造

1. 日志，链路监控

    - skywalking接入，ELK+file-pilot接入

    - ElasticSearch原生部署，两套集群的方式进行部署

2. 链路控制，共同开发。本地启动和线上调试的问题。kt-connect
3. 培训内容
4. 镜像仓库替换为harbor（能做做，不能做后放）
5. k8s限制和为何使用istio？统一运维平台内的信息。