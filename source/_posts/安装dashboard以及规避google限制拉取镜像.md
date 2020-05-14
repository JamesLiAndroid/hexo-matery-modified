---
title: 安装dashboard以及规避google限制拉取镜像
top: false
cover: false
toc: true
mathjax: true
date: 2020-04-18 10:40:50
password:
summary:
tags:
categories:
---

# dashboard安装以及规避google限制拉取镜像

## 工具链

* dashboard版本--2.0.0-rc7
* kubernetes版本--1.17.2

## 下载官方部署文件

找到[官方github仓库](https://github.com/kubernetes/dashboard/releases)地址，转到dashboard/aio/deploy目录下，也可以直接访问[该链接](https://github.com/kubernetes/dashboard/tree/master/aio/deploy)。下载recommended.yaml文件，网络不好的时候也可以打开该文件，直接复制里面的内容，创建即可。

下载文件完成后，需要编辑该文件，主要更改的点在于：

* 对Service设置nodePort方式对外暴露端口
* 调整镜像拉取策略，从Always更换为IfNotPresent

更改完成之后的配置文件内容如下：

```yaml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

apiVersion: v1
kind: Namespace
metadata:
  name: kubernetes-dashboard

---

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kubernetes-dashboard
type: Opaque

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-csrf
  namespace: kubernetes-dashboard
type: Opaque
data:
  csrf: ""

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-key-holder
  namespace: kubernetes-dashboard
type: Opaque

---

kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-settings
  namespace: kubernetes-dashboard

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
rules:
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
    verbs: ["get", "update", "delete"]
    # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["get", "update"]
    # Allow Dashboard to get metrics.
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster", "dashboard-metrics-scraper"]
    verbs: ["proxy"]
  - apiGroups: [""]
    resources: ["services/proxy"]
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
    verbs: ["get"]

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
rules:
  # Allow Metrics Scraper to get metrics from the Metrics server
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
        - name: kubernetes-dashboard
          image: kubernetesui/dashboard:v2.0.0-rc7
          #imagePullPolicy: Always
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8443
              protocol: TCP
          args:
            - --auto-generate-certificates
            - --namespace=kubernetes-dashboard
            # Uncomment the following line to manually specify Kubernetes API server Host
            # If not specified, Dashboard will attempt to auto discover the API server and connect
            # to it. Uncomment only if the default does not work.
            # - --apiserver-host=http://my-address:port
          volumeMounts:
            - name: kubernetes-dashboard-certs
              mountPath: /certs
              # Create on-disk volume to store exec logs
            - mountPath: /tmp
              name: tmp-volume
          livenessProbe:
            httpGet:
              scheme: HTTPS
              path: /
              port: 8443
            initialDelaySeconds: 30
            timeoutSeconds: 30
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      volumes:
        - name: kubernetes-dashboard-certs
          secret:
            secretName: kubernetes-dashboard-certs
        - name: tmp-volume
          emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "beta.kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    k8s-app: dashboard-metrics-scraper

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: dashboard-metrics-scraper
  template:
    metadata:
      labels:
        k8s-app: dashboard-metrics-scraper
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'runtime/default'
    spec:
      containers:
        - name: dashboard-metrics-scraper
          image: kubernetesui/metrics-scraper:v1.0.4
          ports:
            - containerPort: 8000
              protocol: TCP
          livenessProbe:
            httpGet:
              scheme: HTTP
              path: /
              port: 8000
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
          - mountPath: /tmp
            name: tmp-volume
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "beta.kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      volumes:
        - name: tmp-volume
          emptyDir: {}

```

## 重做dashboard相关的镜像

由于众所周知的网络问题，当需要去拉取相应镜像的时候，非常耗时且存在无法拉取的情况，总是在dockerhub上捡别人现成的镜像也不好。这时需要制作自己的docker镜像。

### 前提

准备好自己的github账号，并且登录，然后注册一个[DockerHub](https://hub.docker.com/)的账号。DockerHub可能存在访问慢的问题，请自行科学上网。

### 创建github项目

以kubernetesui/dashboard:v2.0.0-rc7镜像为例子进行创建，首先在github上创建空项目，项目名称为：k8s-dashboard-2.0.0-rc7，然后克隆到本地进行操作。

在项目目录下新增一个Dockerfile文件，并进行编辑，添加下面的两行信息：

```Dockerfile

FROM kubernetesui/dashboard:v2.0.0-rc7
MAINTAINER lsy1234567@163.com

```

保存后将文件提交版本控制，并推送到github仓库中，这时候我们就完成了镜像的配置文件编写。

### 制作docker镜像

登录Dockerhub，如下：

![](dockerhub个人页面.png)

点击Create Repository，进入创建docker镜像的流程。操作如下：

![](创建docker镜像的操作.png)

创建完成后，点击Create &Build即可完成。如下：

![](完成创建docker镜像的操作.png)

完成后回到首页，点击Builds看到我们的镜像已经开始构建了，构建完成后，结果如下：

![](构建完成.png)

这时候docker镜像就制作完成了。可以回到我们的服务器进行拉取镜像的测试，进行如下操作：

```
$ docker pull dockerhtsoft/k8s-test:latest

```

两个镜像都用这种方法去创建，最终形成以下两个镜像，如下图：

![](镜像信息.png)

镜像制作完成后，需要拉取镜像到对应的机器，这里先把镜像拉取到一台机器上，然后对镜像改名，变更为k8s部署配置文件中对应的名称，最后推送到各个节点上。例如拉取了我们自己的镜像dockerhtsoft/k8s-dashboard-2.0.0-rc7，然后更名为kubernetesui/dashboard:v2.0.0-rc7，推送到各个节点机器上的操作如下：

```
$ docker pull dockerhtsoft/k8s-dashboard-2.0.0-rc7

$ docker tag dockerhtsoft/k8s-dashboard-2.0.0-rc7:latest kubernetesui/dashboard:v2.0.0-rc7

$ docker save kubernetesui/dashboard:v2.0.0-rc7 | bzip2 | ssh -p 15555 centos@192.168.123.239 'bunzip2 | docker load'

// 推送到其它机器的时候，重复以上操作即可

```

同样的dockerhtsoft/k8s-metrics-scraper-v1.0.4，拉取后更名为kubernetesui/metrics-scraper:v1.0.4，推送到各个节点机器上。

### 安装方法

当上述所有的内容改造完毕后，这时候开始正式的安装。操作如下：

```
$ kubectl apply -f recommended.yaml

```

安装完毕后，查看pod和service状态。操作如下：

```
$ kubectl get pods,svc -n kubernetes-dashboard -o wide
NAME                                            READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
pod/dashboard-metrics-scraper-b68468655-jj85v   1/1     Running   0          2d14h   10.42.1.182   192.168.123.239   <none>           <none>
pod/kubernetes-dashboard-6f659467f6-qvvjx       1/1     Running   0          2d14h   10.42.1.181   192.168.123.239   <none>           <none>

NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE     SELECTOR
service/dashboard-metrics-scraper   ClusterIP   10.43.150.148   <none>        8000/TCP        2d14h   k8s-app=dashboard-metrics-scraper
service/kubernetes-dashboard        NodePort    10.43.254.199   <none>        443:30001/TCP   2d14h   k8s-app=kubernetes-dashboard

```

### 如何访问

由上述信息可知，我们的dashboard已经部署在192.168.123.239这台机器上了。通过https://192.168.123.239:30001可以进行访问，需要接受浏览器的风险提示才能进行访问。如下：

![](dashboard首页.png)

这时候可以进行正常访问了，可以使用默认用户访问，也可以自行创建管理员用户进行。下面自行创建管理员用户：

```
$ vim create-admin.yaml
// 填入以下内容
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard

// :wq保存退出

// 执行以下命令
$ kubectl apply -f create-admin.yaml

```
管理员用户创建完成后，需要获取器sa和secret值，通过token进行登录，获取操作如下：

```
$ kubectl get sa,secrets -n kubernetes-dashboard
NAME                                  SECRETS   AGE
serviceaccount/admin-user             1         2d15h
serviceaccount/default                1         2d15h
serviceaccount/kubernetes-dashboard   1         2d15h

NAME                                      TYPE                                  DATA   AGE
secret/admin-user-token-fqvxr             kubernetes.io/service-account-token   3      2d15h
secret/default-token-c76qr                kubernetes.io/service-account-token   3      2d15h
secret/kubernetes-dashboard-certs         Opaque                                0      2d15h
secret/kubernetes-dashboard-csrf          Opaque                                1      2d15h
secret/kubernetes-dashboard-key-holder    Opaque                                2      2d15h
secret/kubernetes-dashboard-token-h4fzt   kubernetes.io/service-account-token   3      2d15h

$ kubectl describe secret admin-user-token-fqvxr  -n kubernetes-dashboard
Name:         admin-user-token-fqvxr
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 36e39ee5-6ba1-42c5-a6a6-d6698f07620a

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1017 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IlZFOFRYYWhHR3d5UFM4dmZ4NGNXRGlRMnEzU2dYRHpEM0RNTjZmN1R3aWcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWZxdnhyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIzNmUzOWVlNS02YmExLTQyYzUtYTZhNi1kNjY5OGYwNzYyMGEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.mZZwwOObHFLDcbZZ5ukj9zljm47EcNL4KPkGAKfR65g-kkICg9rClRpjUHfIM5v_1N4Cnvo9uh_GFwnuZ01Pm4zb9UrvYv4VwyUqKGktc24caWGz9auhylT9POYuFXzuZi7TfHTY7NyOMq8PSBBgg73eDuVewayZQCRpwH9xJEe4FYFJ8xjQIYA94b1IYgg72xqWGM0L0rVkoMIchAGLzy5acDhK2DjtL_LSmr22SnIJ1jaYIMWa4wQIeZgMl5WbUNU1S_3nAwQdbvo2oSLlQkBdTPDLDGXXaObPBcJZxVS19s8WXrFMIKsFD7knJBX_E_bjWZBz8WZJw

```
这样复制**token:**后的字符串信息，在登录页面选择token登录，粘贴到输入框中，如下：

![](输入token信息.png)

点击Sign in，登录到系统中，如下：

![](系统中全景.png)

整个安装过程就完成了！