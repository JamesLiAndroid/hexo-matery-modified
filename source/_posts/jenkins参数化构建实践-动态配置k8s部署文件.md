---
title: jenkins参数化构建实践--动态配置k8s部署文件
top: false
cover: false
toc: true
mathjax: true
date: 2020-07-02 16:28:25
password:
summary:
tags:
categories:
---

# jenkins参数化构建实践--动态配置k8s部署文件

## 使用插件

利用jenkins中的Build With Parameters Plugin插件进行构建。

## 后端参数化构建设置

* 1. 配置文件的变更

以test-instance-utils项目中的data-dictionary-k8s.yaml配置文件为例子：

```YAML
---
apiVersion: v1
kind: Service
metadata:
  name: test-instance-data-dictionary
  namespace: test-instance-all-service
  labels:
    app: test-instance-data-dictionary
spec:
  ports:
    - port: ${SERVER_PORT}
      name: tcp
      targetPort: ${SERVER_PORT}
  selector:
    app: test-instance-data-dictionary
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-instance-data-dictionary
  namespace: test-instance-all-service
spec:
  minReadySeconds: 10
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  replicas: 1
  selector:
    matchLabels:
      app: test-instance-data-dictionary
  template:
    metadata:
      labels:
        app: test-instance-data-dictionary
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
                        - app-test-instance-data-dictionary
              weight: 1
      containers:
        - name: data-dictionary
          image: ${DOCKER_HUB}/data-dictionary-k8s:$BUILD_NUMBER
          imagePullPolicy: Always
          lifecycle:
            preStop:
              httpGet:
                port: ${SERVER_PORT}
                path: /spring/shutdown
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: ${SERVER_PORT}
            initialDelaySeconds: 180
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: ${SERVER_PORT}
            initialDelaySeconds: 180
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
            - containerPort: ${SERVER_PORT}
          env:
            - name: IP_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NACOS_IP
              value: ${NACOS_IP_PORT}
            - name: NACOS_NAMESPACE
              value: ${NACOS_NAMESPACE}
            - name: SKYWALKING_NAMESPACE
              value: test-instance-k8s-test
            - name: SKYWALKING_TARGET_SERVICE_NAME
              value: test-instance-data-dic
            - name: SKYWALKING_IP_PORT
              value: ${SKYWALKING_IP_PORT}
            - name: CHANNEL
              value: ${CHANNEL}

```

主要确定下面几个变量：

- ${SERVER_PORT} ---- 服务端口号，需要同配置文件中进行对应

- ${DOCKER_HUB}  ---- 拉取镜像的地址

- ${NACOS_IP_PORT}  ---- 配置中心地址

- ${NACOS_NAMESPACE} ---- 配置中心的命名空间

- ${SKYWALKING_IP_PORT} ---- 全链路监控skywalking的地址监控

- ${CHANNEL}  ---- 配置文件后缀，服务名+后缀确定拉取的配置文件名称

* 2. 构建时配置的变更

先来看效果，点击Build with Parameters后，出现下图效果

![](参数化构建.png)

借助*Extended Choice Parameter*插件，进行参数化构建的配置，最终配置如下：

![](jenkins构建整体配置.png)

可以看到我们添加了上述几个变量的参数，并且设置了默认值。

这样就可以达到不变更提交代码，就可以自行配置构建参数的目的，简化操作。

## 前端参数化构建的设置

* 1. 配置文件的变更

```YAML
---
apiVersion: v1
kind: Service
metadata:
  name: test-instance-frontend
  namespace: test-instance-all-service
  labels:
    app: test-instance-frontend
spec:
  type: NodePort
  ports:
    - port: ${CONTAINER_PORT}
      name: tcp
      targetPort: ${CONTAINER_PORT}
      nodePort: ${NODE_PORT}
  selector:
    app: test-instance-frontend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-instance-frontend
  namespace: test-instance-all-service
spec:
  minReadySeconds: 10
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  replicas: 1
  selector:
    matchLabels:
      app: test-instance-frontend
  template:
    metadata:
      labels:
        app: test-instance-frontend
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
                        - app-test-instance-frontend
              weight: 1
      containers:
        - name: test-instance-frontend
          image: ${DOCKER_HUB}/test-instance-frontend-k8s:$BUILD_NUMBER
          imagePullPolicy: Always
          resources:
            requests:
              memory: 1024Mi
            limits:
              memory: 1.5Gi
          ports:
            - containerPort: ${CONTAINER_PORT}
          env:
            - name: GATEWAY_HOST
              value: ${GATEWAY_HOST}
              #value: gateway.test-instance-basic-gateway:19020 

```
主要确定下面几个变量：

- ${CONTAINER_PORT} ---- 镜像运行的端口信息
- ${NODE_PORT}      ---- 对外服务的端口号
- ${DOCKER_HUB}     ---- 拉取镜像的地址
- ${GATEWAY_HOST}   ---- 网关配置

* 2. 构建时配置的变更

借助*Extended Choice Parameter*插件，进行参数化构建的配置，最终配置如下：

![](前端参数化构建.png)

这样前端参数化构建就完成了。

## 问题

1. 参数化构建和webhook触发构建是否冲突？

通过提交测试，证明两部分不冲突，能够正常触发并且以默认值的方式进行构建。

## 总结

利用参数化构建，实现部分参数的剥离配置，在后续部署或者调整时，灵活性更大。

