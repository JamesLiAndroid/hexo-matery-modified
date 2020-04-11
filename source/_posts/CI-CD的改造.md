---
title: CI/CD的改造--通过jenkins部署到k8s
top: false
cover: false
toc: true
mathjax: true
date: 2020-04-10 12:21:47
password:
summary:
tags:
categories:
---

# CI/CD的改造

## 前期介绍

使用jenkins+gitlab+sonar+docker的方式保证了前期的持续集成构建流程的顺利展开，但是在面向k8s时，将docker换成k8s时，需要对整个构建流程进行改造。

其核心聚焦在最终的部署上，由于我们的jenkins部署在k8s之外，需要将自有的微服务部署在k8s中，需要借助kubectl进行改造。

灵感来自我们当时针对Rancher的部署，部署完成后在192.168.88.240机器上，使用kubectl工具以及指定的配置文件对k8s集群进行管理！

## 方案确定

### 方案选择

将deployment.yaml放在对应服务的根路径下，且最好以模板的可替换的方式导入。

1. 将kubectl工具和k8s生成的配置文件kube_config_cluster.yaml，导入到jenkins所在镜像，然后在构建流水线中启动本地shell并转到$WORKSPACE对应的路径，使用kubectl进行部署

2. 借助jenkins插件的方式，Kubernetes Continous Deploy，需要将服务的部署文件随源代码上传到gitlab中

3. 使用Helm工具进行安装部署，拉取私有镜像仓库中的镜像，部署到k8s中

第一种方式由于jenkins镜像的不确定性，如果把文件通过拷贝的方式放到k8s中，一旦jenkins镜像出现问题，恢复会比较困难，但是这种方式最简单，也最迅速！

第三种方式我们需要把部署的yaml文件远程传输到目标服务器，这样部署复杂程度高，不易操作。

因此选择第二种方式进行操作

### 具体操作

#### 1. 安装插件

优先测试第二种方式，首先离线安装这个Kubernetes Continous Deploy插件，需要下载两个插件信息azure-commons和kubernetes-cd，从[下载地址](https://plugins.jenkins.io/)中搜索并下载这两个插件，通过jenkins插件离线安装的方式进行，点击Manage Jenkins，转到如下图：

![](离线安装插件.png)

然后点击Manage Plugins，转到插件管理页面，开始上传.hpi文件，安装插件，如下：

![](上传安装插件.png)

安装完成后，需要重启jenkins使插件生效。

#### 2. 对于具体构建的改造


#### 2.1 添加部署到k8s的执行块

对于之前的项目，我们把生成的docker镜像通过ssh连接到目标服务器，执行shell脚本代码进行部署，如下：

![](执行shell代码部署.png)

现在需要更换方式，利用Kubernetes Continous Deploy插件进行部署，这样就需要删除这个执行脚本的shell，添加k8s部署的执行块。

首先点击Add post-build step，在下拉菜单中找到Deploy to Kubernetes，点击添加执行块，如下：

![](点击添加部署到k8s的执行块.png)

![](k8s执行块.png)

然后我们需要添加k8s的配置文件，主要是k8s集群外部连接的相关信息。在192.168.88.240机器上可以找到该文件，名称为kube_config_cluster.yml。我们需要把该文件的内容添加到系统中，找到Kubeconfig一栏，当前系统中没有相关的配置文件，点击右侧的Add按钮进行添加，如下图：

![](添加kubeconfig01.png)

![](添加kubeconfig02.png)

![](添加kubeconfig03.png)

![](添加kubeconfig04.png)

经过以上四步，我们添加kubeconfig已经完成，下面就可以选择该config信息，使Kubernetes Continous Deploy插件可以连接到我们的k8s集群，如下图：

![](选择kubeconfig生效.png)

选择完毕后，这样就可以生效了。下一步我们需要指定Config文件，由于我们以test-business01项目进行构建测试，这里设置该Config文件的名称为*business01-deployment.yaml*。你会发现目前路径下并没有这个配置文件，我们稍后会创建该文件并添加到项目中。设置完成后如下图：

![](指定部署文件.png)

这样jenkins部分就已经设置完成了，由于我们在k8s集群中已经添加了docker registry的连接信息，这里不需要设置连接镜像仓库的内容，也不需要指定目标的命名空间，点击Verify Configuration进行验证，如下图：

![](点击验证成功.png)

这样就设置完成了，点击最下方的Save按钮保存配置即可！

#### 2.2 对项目的改造

回到项目中，我们需要在项目的根目录创建business01-deployment.yaml。记住一定是在项目的根目录创建，目前Kubernetes Continous Deploy插件并不支持根据路径进行查找！文件内容如下：

```yaml

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
          image: 192.168.88.159:5000/test-business01-k8s:$BUILD_NUMBER
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
            initialDelaySeconds: 180
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 19030
            initialDelaySeconds: 180
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

最后提交代码到版本控制，整个这部分内容就完成了，目录结构如下：

![](目录结构.png)

#### 2.3 触发构建

回到jenkins中，在test-business01的job中点击“立即构建”（Build Now），开始构建，最终关于k8s的输出日志如下：

```
Starting Kubernetes deployment
Loading configuration: /var/jenkins_home/workspace/test-business01-k8s/business01-deployment.yaml
Applied V1Service: class V1Service {
    apiVersion: v1
    kind: Service
    metadata: class V1ObjectMeta {
        annotations: null
        clusterName: null
        creationTimestamp: 2020-04-06T04:11:47.000Z
        deletionGracePeriodSeconds: null
        deletionTimestamp: null
        finalizers: null
        generateName: null
        generation: null
        initializers: null
        labels: {app=test-business-01}
        managedFields: null
        name: test-business-01
        namespace: test-test-business
        ownerReferences: null
        resourceVersion: 16222470
        selfLink: /api/v1/namespaces/test-test-business/services/test-business-01
        uid: 90eddf16-be68-442f-8112-f009d55b46d2
    }
    spec: class V1ServiceSpec {
        clusterIP: 10.43.50.235
        externalIPs: null
        externalName: null
        externalTrafficPolicy: null
        healthCheckNodePort: null
        loadBalancerIP: null
        loadBalancerSourceRanges: null
        ports: [class V1ServicePort {
            name: tcp
            nodePort: null
            port: 19030
            protocol: TCP
            targetPort: 19030
        }]
        publishNotReadyAddresses: null
        selector: {app=test-business-01}
        sessionAffinity: None
        sessionAffinityConfig: null
        type: ClusterIP
    }
    status: class V1ServiceStatus {
        loadBalancer: class V1LoadBalancerStatus {
            ingress: null
        }
    }
}
Applied V1Deployment: class V1Deployment {
    apiVersion: apps/v1
    kind: Deployment
    metadata: class V1ObjectMeta {
        annotations: null
        clusterName: null
        creationTimestamp: 2020-04-08T01:30:06.000Z
        deletionGracePeriodSeconds: null
        deletionTimestamp: null
        finalizers: null
        generateName: null
        generation: 4
        initializers: null
        labels: null
        managedFields: null
        name: test-business-01
        namespace: test-test-business
        ownerReferences: null
        resourceVersion: 16222472
        selfLink: /apis/apps/v1/namespaces/test-test-business/deployments/test-business-01
        uid: 7fca5704-6be8-49fa-9454-03a2a522ac7b
    }
    spec: class V1DeploymentSpec {
        minReadySeconds: 10
        paused: null
        progressDeadlineSeconds: 600
        replicas: 3
        revisionHistoryLimit: 10
        selector: class V1LabelSelector {
            matchExpressions: null
            matchLabels: {app=test-business-01}
        }
        strategy: class V1DeploymentStrategy {
            rollingUpdate: class V1RollingUpdateDeployment {
                maxSurge: 1
                maxUnavailable: 0
            }
            type: RollingUpdate
        }
        template: class V1PodTemplateSpec {
            metadata: class V1ObjectMeta {
                annotations: null
                clusterName: null
                creationTimestamp: null
                deletionGracePeriodSeconds: null
                deletionTimestamp: null
                finalizers: null
                generateName: null
                generation: null
                initializers: null
                labels: {app=test-business-01}
                managedFields: null
                name: null
                namespace: null
                ownerReferences: null
                resourceVersion: null
                selfLink: null
                uid: null
            }
            spec: class V1PodSpec {
                activeDeadlineSeconds: null
                affinity: class V1Affinity {
                    nodeAffinity: null
                    podAffinity: null
                    podAntiAffinity: class V1PodAntiAffinity {
                        preferredDuringSchedulingIgnoredDuringExecution: [class V1WeightedPodAffinityTerm {
                            podAffinityTerm: class V1PodAffinityTerm {
                                labelSelector: class V1LabelSelector {
                                    matchExpressions: [class V1LabelSelectorRequirement {
                                        key: app
                                        operator: In
                                        values: [app-test-business-01]
                                    }]
                                    matchLabels: null
                                }
                                namespaces: null
                                topologyKey: kubernetes.io/hostname
                            }
                            weight: 1
                        }]
                        requiredDuringSchedulingIgnoredDuringExecution: null
                    }
                }
                automountServiceAccountToken: null
                containers: [class V1Container {
                    args: null
                    command: null
                    env: [class V1EnvVar {
                        name: IP_ADDR
                        value: null
                        valueFrom: class V1EnvVarSource {
                            configMapKeyRef: null
                            fieldRef: class V1ObjectFieldSelector {
                                apiVersion: v1
                                fieldPath: status.podIP
                            }
                            resourceFieldRef: null
                            secretKeyRef: null
                        }
                    }, class V1EnvVar {
                        name: NACOS_IP
                        value: http://nacos-0.nacos-headless.test-basic-nacos.svc.cluster.local:8848,http://nacos-1.nacos-headless.test-basic-nacos.svc.cluster.local:8848,http://nacos-2.nacos-headless.test-basic-nacos.svc.cluster.local:8848
                        valueFrom: null
                    }, class V1EnvVar {
                        name: NACOS_NAMESPACE
                        value: test-dev
                        valueFrom: null
                    }, class V1EnvVar {
                        name: CHANNEL
                        value: standalone
                        valueFrom: null
                    }]
                    envFrom: null
                    image: 192.168.88.159:5000/test-business01-k8s:17
                    imagePullPolicy: Always
                    lifecycle: class V1Lifecycle {
                        postStart: null
                        preStop: class V1Handler {
                            exec: null
                            httpGet: class V1HTTPGetAction {
                                host: null
                                httpHeaders: null
                                path: /spring/shutdown
                                port: 19030
                                scheme: HTTP
                            }
                            tcpSocket: null
                        }
                    }
                    livenessProbe: class V1Probe {
                        exec: null
                        failureThreshold: 5
                        httpGet: class V1HTTPGetAction {
                            host: null
                            httpHeaders: null
                            path: /actuator/health
                            port: 19030
                            scheme: HTTP
                        }
                        initialDelaySeconds: 180
                        periodSeconds: 5
                        successThreshold: 1
                        tcpSocket: null
                        timeoutSeconds: 10
                    }
                    name: test-business-01
                    ports: [class V1ContainerPort {
                        containerPort: 19030
                        hostIP: null
                        hostPort: null
                        name: null
                        protocol: TCP
                    }]
                    readinessProbe: class V1Probe {
                        exec: null
                        failureThreshold: 5
                        httpGet: class V1HTTPGetAction {
                            host: null
                            httpHeaders: null
                            path: /actuator/health
                            port: 19030
                            scheme: HTTP
                        }
                        initialDelaySeconds: 180
                        periodSeconds: 5
                        successThreshold: 1
                        tcpSocket: null
                        timeoutSeconds: 10
                    }
                    resources: class V1ResourceRequirements {
                        limits: {memory=Quantity{number=1073741824, format=BINARY_SI}}
                        requests: {memory=Quantity{number=1073741824, format=BINARY_SI}}
                    }
                    securityContext: null
                    stdin: null
                    stdinOnce: null
                    terminationMessagePath: /dev/termination-log
                    terminationMessagePolicy: File
                    tty: null
                    volumeDevices: null
                    volumeMounts: null
                    workingDir: null
                }]
                dnsConfig: null
                dnsPolicy: ClusterFirst
                enableServiceLinks: null
                hostAliases: null
                hostIPC: null
                hostNetwork: null
                hostPID: null
                hostname: null
                imagePullSecrets: null
                initContainers: null
                nodeName: null
                nodeSelector: null
                preemptionPolicy: null
                priority: null
                priorityClassName: null
                readinessGates: null
                restartPolicy: Always
                runtimeClassName: null
                schedulerName: default-scheduler
                securityContext: class V1PodSecurityContext {
                    fsGroup: null
                    runAsGroup: null
                    runAsNonRoot: null
                    runAsUser: null
                    seLinuxOptions: null
                    supplementalGroups: null
                    sysctls: null
                    windowsOptions: null
                }
                serviceAccount: null
                serviceAccountName: null
                shareProcessNamespace: null
                subdomain: null
                terminationGracePeriodSeconds: 30
                tolerations: null
                volumes: null
            }
        }
    }
    status: class V1DeploymentStatus {
        availableReplicas: 1
        collisionCount: null
        conditions: [class V1DeploymentCondition {
            lastTransitionTime: 2020-04-08T01:30:33.000Z
            lastUpdateTime: 2020-04-08T01:37:10.000Z
            message: ReplicaSet "test-business-01-67b4cd9d4c" has successfully progressed.
            reason: NewReplicaSetAvailable
            status: True
            type: Progressing
        }, class V1DeploymentCondition {
            lastTransitionTime: 2020-04-08T02:03:24.000Z
            lastUpdateTime: 2020-04-08T02:03:24.000Z
            message: Deployment has minimum availability.
            reason: MinimumReplicasAvailable
            status: True
            type: Available
        }]
        observedGeneration: 3
        readyReplicas: 1
        replicas: 1
        unavailableReplicas: null
        updatedReplicas: 1
    }
}
Finished Kubernetes deployment
Finished: SUCCESS

```

说明k8s中已经部署了我们的镜像，在k8s中查看如下：

```

$ kubectl get pod -n test-test-business
NAME                                READY   STATUS    RESTARTS   AGE
test-business-01-57b478d68d-27x8g   1/1     Running   0          81m
test-business-01-57b478d68d-5qzs8   1/1     Running   3          87m
test-business-01-57b478d68d-8jt5t   1/1     Running   0          77m

```

部署成功！

## 总结

通过这样的方式，让我们的构建更简单，调整的部分也更少，后续的工作也能很好的开展。

这里需要注意的是，在每一个项目的yaml配置文件中，一定要指定部署在k8s集群中的namespace名称！