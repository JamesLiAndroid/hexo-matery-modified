---
title: 前端微服务在jenkins下的构建改造
top: false
cover: false
toc: true
mathjax: true
date: 2020-08-03 10:38:06
password:
summary:
tags:
categories:
---

## 前端微服务--微前端

微前端架构是一种类似于微服务的架构，它将微服务的理念应用于浏览器端，即将 Web 应用由单一的单体应用转变为多个小型前端应用聚合为一的应用。由此带来的变化是，这些前端应用可以独立运行、独立开发、独立部署。以及，它们应该可以在共享组件的同时进行并行开发。

根据目前的发展情况，有两种实施方式

- 前端模块各自运行调试，最终统一打包，继续以单体模式运行
- 前端模块各自以运行调试，分别打包部署，使用以下方式进行组织：

    1. 使用 HTTP 服务器的路由来重定向多个应用
    2. 在不同的框架之上设计通讯、加载机制，诸如 Mooa 和 Single-SPA
    3. 通过组合多个独立应用、组件来构建一个单体应用
    4. iFrame。使用 iFrame 及自定义消息传递机制
    5. 使用纯 Web Components 构建应用
    6. 结合 Web Components 构建

根据我们目前项目中，对于前端微服务采用第一种方式进行组织，这样更快一些，实现更简单，只能说实现各自的开发和运行，但是并不是真正的前端微服务范畴，只能叫前端组件化。

## jenkins的改造

### 0. 项目改造案例

以一个项目切分不同的分支，最后再将开发的文件夹合并的方式进行改造。

这里的示例项目是test1.0-ui，然后涉及到的三个分支分别是：

- 微前端-dev-base（基础构建分支）
- 微前端-dev-roleManagement（角色功能分支）
- 微前端-dev-menuManagement（菜单功能分支）

分别拉取各个分支，首先将*微前端-dev-roleManagement*分支下的src/views/operationManagementCenter/roleManagement文件夹拷贝到*微前端-dev-base*分支的src/views/operationManagementCenter文件夹下。

然后同样的将*微前端-dev-roleManagement*分支下src/views/operationManagementCenter/menuManagement文件夹拷贝到*微前端-dev-base*分支的src/views/operationManagementCenter文件夹下。

最后在*微前端-dev-base*分支进行打包操作。

### 1. 插件支持和脚本编写

基于之前的前端构建项目进行改造，使用pipeline的方式组织该项目的构建，利用docker镜像进行打包。

涉及的插件如下：

- Git
- Git client 上面两个是拉取项目使用的
- Pipeline  构建的核心工具
- Kubernetes CLI
- Kubernetes
- Kubernetes Continuous Deploy Plugin k8s中的部署工具
- Qy Wechat Notification  通知使用工具
- SSH Pipeline Steps      pipeline中ssh访问工具
- Webhook Step Plugin    触发webhook的工具

下面开始编写脚本，如下：

```groovy
pipeline {
    agent any
    stages{
        stage("checkout and multiple"){
            steps{
                dir(path: "./MainFrontEnd"){
                    git(
                        branch: '微前端-dev-base', 
                        credentialsId: 'lisongyang-Test-CICD', 
                        url: 'http://192.168.128.202:8181/test/frontend/c.git',
                        changelog: true
                    )
                }
                dir(path: "./MicroService1"){
                    git(
                        branch: '微前端-dev-roleManagement', 
                        credentialsId: 'lisongyang-Test-CICD', 
                        url: 'http://192.168.128.202:8181/test/frontend/test1.0-ui.git',
                        changelog: true
                    )
                }
                dir(path: "./MicroService2"){
                    git(
                        branch: '微前端-dev-menuManagement', 
                        credentialsId: 'lisongyang-Test-CICD', 
                        url: 'http://192.168.128.202:8181/test/frontend/test1.0-ui.git',
                        changelog: true
                    )
                }
                sh  """
                    cp -r ./MicroService1/src/views/operationManagementCenter/roleManagement ./MainFrontEnd/src/views/operationManagementCenter/
                    cp -r ./MicroService2/src/views/operationManagementCenter/menuManagement ./MainFrontEnd/src/views/operationManagementCenter/
                    """
            }
            
            post{
                success{
                    echo "build success - stage 1"
                }
            
                failure{
                    echo "build falure!!!! - stage 1"
                }
            }
        }
        
        stage("build all"){
            steps{
                //def imageName = "192.168.128.202:5000/test/test-build-frontend:${env.BUILD_NUMBER}"
                dir(path: "./MainFrontEnd"){
                    withDockerServer([uri: 'tcp://192.168.128.202:2375', credentialsId: '']) {
                        withDockerRegistry(url: '192.168.128.202:5000', credentialsId: '') {
                            // sh """
                            //     cd ${WORKSPACE}/MainFrontEnd
                            // """
                            script{
                                docker.build('192.168.128.202:5000/test/test-build-frontend:${BUILD_NUMBER}', '-f ./Dockerfile-opt .').push()
                            }
                        }
                    }
                }
            }
            post{
                success{
                    echo "build success - stage 2" 
                }
            
                failure{
                    echo "build falure!!!! - stage 2"
                }
            }
        }
        // stage("deploy from ssh"){
        //     steps{
                
        //     }
        //     post{
        //         success{
        //             echo "build success - stage 3"
        //         }
            
        //         failure{
        //             echo "build falure!!!! - stage 3"
        //         }
        //     }
        // }
        // 部署到k8s中的配置，看下面的内容
        stage("deploy to kubernetes"){
            steps{
                //def imageName = "192.168.128.202:5000/test/test-build-frontend:${env.BUILD_NUMBER}"
                dir(path: "./MainFrontEnd"){
                    kubernetesDeploy configs: 'frontend-opt-k8s.yaml', kubeConfig: [path: ''], kubeconfigId: '66-220-k8s-config', secretName: '', ssh: [sshCredentialsId: '*', sshServer: ''], textCredentials: [certificateAuthorityData: '', clientCertificateData: '', clientKeyData: '', serverUrl: 'https://']
                }
            }
            post{
                success{
                    echo "build success - stage 3"
                }
            
                failure{
                    echo "build falure!!!! - stage 3"
                }
            }
        }
        
    }
    
    
}


```

编写完成后，需要修改Dockerfile-opt（专门为docker内构建前端项目创建）以及k8s文件，最终如下。

1. Dockerfile-opt文件：
```
# 第一层面构建，打包前端代码
#### 1. 指定node镜像版本
FROM node:10.16.0 AS builder

# 添加日期信息，如果需要更新缓存层，更新该处日期信息即可
ENV REFRESH_DATE 2020-05-20_11:11

# 2. 指定编译的工作空间
WORKDIR /home/node/app

# 3. 设置授权信息，本地开发机登录后获取私有镜像的authToken信息，放入.npmrc
# RUN touch ~/.npmrc && mkdir ~/node_global && mkdir ~/node_cache && echo prefix=~/node_global \n\ 
# cache=~/node_cache \n\ 
# registry=http://10.0.88.159:8081/repository/npm-group/ \n\ 
# //10.0.88.159:8081/repository/npm-group/:_authToken=NpmToken.161b4deb-11f8-3d95-ac27-30510480ae42 >> ~/.npmrc

# 4. 构建之前清理缓存
# RUN npm cache clean --force

# 4. 安装打包需要的yarn工具
RUN npm --registry https://registry.npm.taobao.org install -g yarn
# 对yarn设置淘宝镜像
RUN yarn config set registry https://registry.npm.taobao.org
RUN yarn config set disturl https://npm.taobao.org/dist

# 设置不使用ssl，解决401保存问题
#RUN npm config set strict-ssl false

# 解决登录问题
#RUN npm install -g npm-cli-adduser --registry https://registry.npm.taobao.org

# 解决node-sass安装失败的问题
# RUN yarn config set sass_binary_site https://npm.taobao.org/mirrors/node-sass/ -g

# 解决错误信息info fsevents@2.1.3: The platform "linux" is incompatible with this module.
# info "fsevents@2.1.3" is an optional dependency and failed compatibility check. Excluding it from installation.
#RUN yarn config set ignore-engines true

# 4. 添加package.json
COPY package.json /home/node/app/
# 5. 安装依赖，如果package.json未变更，将会沿用之前的镜像缓存
#RUN NPM_USER=npm-user NPM_PASS=hoteam@2019 NPM_EMAIL=lisongyang@hoteamsoft.com NPM_REGISTRY=http://10.0.88.159:8081/repository/npm-group/ npm-cli-adduser 

# 查看自身信息
# RUN npm whoami

# 安装依赖信息
RUN yarn install --registry https://registry.npm.taobao.org --network-timeout 1000000
# 6. 添加剩余代码到工作空间
COPY . /home/node/app
# 7. 编译代码
RUN yarn run build

# 第二层面构建
#### 1.拉取自定义镜像名称
FROM 10.0.88.159:5000/base_frontend:0.0.1

# 2.将打包后的代码复制到运行位置
COPY --from=builder /home/node/app/dist /var/www

# 3.启动nginx
ENTRYPOINT ["nginx","-g","daemon off;"]

```

2. k8s部署文件：

```
---
apiVersion: v1
kind: Service
metadata:
  name: test-frontend
  namespace: test-all-service
  labels:
    app: test-frontend
spec:
  type: NodePort
  ports:
    - port: 80
      name: tcp
      targetPort: 80
      nodePort: 31080
  selector:
    app: test-frontend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-frontend
  namespace: test-all-service
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
      app: test-frontend
  template:
    metadata:
      labels:
        app: test-frontend
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
                        - app-test-frontend
              weight: 1
      containers:
        - name: test-frontend
          image: 192.168.128.202:5000/test/test-build-frontend:$BUILD_NUMBER
          imagePullPolicy: Always
          resources:
            requests:
              memory: 1024Mi
            limits:
              memory: 1.5Gi
          ports:
            - containerPort: 80
          env:
            - name: GATEWAY_HOST
              value: 192.168.128.221:31333
              #value: gateway.test-basic-gateway:19020 

```

主要是调整了镜像名称以及网关地址。

### 2. 问题：关于docker的安装

在构建前端服务时，出现了docker服务找不到的情况，这时候有两种解决方式：

- 重建jenkins容器，关联本地的docker服务

在该方法中，使用下面的命令重建jenkins容器：

```
docker run  -d  -u root  -v /usr/bin/docker:/usr/bin/docker -v /var/run/docker.sock:/var/run/docker.sock -v /usr/lib64/libltdl.so.7:/usr/lib/x86_64-linux-gnu/libltdl.so.7 jenkins/jenkins:2.208

```

启动后，通过thinbackup工具备份之前的jenkins构建信息，然后在新的jenkins服务中进行恢复即可。

这样虽然可以使用宿主机的docker服务，但是由于宿主机中，docker内部有些功能需要使用到root权限，挂载后容器内的命令会影响到host，会导致安全风险！不建议使用。

- 在容器内安装docker服务

在容器内安装，只可以在容器内使用，对外进行隔离，不影响外部的docker运行。

下面开始容器内安装docker服务：

1. 安装apt-transport-https模块

这里初步使用apt update进行更新时，卡在"[0%] Working"的位置，不往下走。这时候先验证无网络问题，"ping baidu.com"操作一下。

首先离线安装apt-transport-https，从[改地址](http://archive.ubuntu.com/ubuntu/pool/main/a/apt/apt-transport-https_1.2.32ubuntu0.1_amd64.deb
)下载离线安装包，传输到docker容器内。

```
$ docker cp apt-transport-https_1.2.32ubuntu0.1_amd64.deb jenkins_official:/var/jenkins_home/

$ $ docker exec -it -u root jenkins_official /bin/bash

root@cbbc3907c1bc: cd /var/jenkins_home/ && dpkg -i apt-transport-https_1.2.32ubuntu0.1_amd64.deb

```

排除网络问题后，说明外网可以联通，这时候再去尝试修改源文件信息，参考[Linux系统初始化以及部署](Linux系统初始化以及部署.md)文档，修改源信息。由于这里Dockerfile编写的时候使用的是ubuntu，更改会稍有不同。操作如下：

```
// 离线安装apt-transport-https


// 外部编写sources.list文件
$ vim sources.list
// 输入以下内容
​deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse

// :wq保存退出

// 拷贝新的源文件到镜像中
$ docker cp sources.list jenkins_official:/var/jenkins_home/

// 以root用户进行镜像
$ docker exec -it -u root jenkins_official /bin/bash

root@cbbc3907c1bc: cd /etc/apt/

root@cbbc3907c1bc: mv sources.list sources.list.bak

// 把新的文件保存到这个位置
root@cbbc3907c1bc:/etc/apt# cp /var/jenkins_home/sources.list ./
// 开始更新
root@cbbc3907c1bc:/etc/apt# apt-get update
// 出现了错误信息如下
W: GPG error: http://mirrors.aliyun.com/ubuntu xenial InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 40976EAF437D05B5 NO_PUBKEY 3B4FE6ACC0B21F32
W: The repository 'http://mirrors.aliyun.com/ubuntu xenial InRelease' is not signed.

// 更新下密钥信息
root@cbbc3907c1bc:/etc/apt# gpg --keyserver keyserver.ubuntu.com --recv 40976EAF437D05B5
root@cbbc3907c1bc:/etc/apt# gpg --export --armor 40976EAF437D05B5 | apt-key add -
root@cbbc3907c1bc:/etc/apt# gpg --keyserver keyserver.ubuntu.com --recv 3B4FE6ACC0B21F32
root@cbbc3907c1bc:/etc/apt# gpg --export --armor 3B4FE6ACC0B21F32 | apt-key add -

root@cbbc3907c1bc:/etc/apt# apt-get update
Get:1 http://mirrors.aliyun.com/ubuntu xenial InRelease [247 kB]
Get:2 http://mirrors.aliyun.com/ubuntu xenial-security InRelease [109 kB]
Get:3 http://mirrors.aliyun.com/ubuntu xenial-updates InRelease [109 kB]
Get:4 http://mirrors.aliyun.com/ubuntu xenial-proposed InRelease [260 kB]
Get:5 http://mirrors.aliyun.com/ubuntu xenial-backports InRelease [107 kB]
Fetched 832 kB in 1s (529 kB/s)
Reading package lists... Done

```

最后安装docker，遵循以下步骤进行：
```
root@cbbc3907c1bc:/etc/apt#  apt-get -y install apt-transport-https \
     ca-certificates \
     curl \
     gnupg2 \
     software-properties-common 

root@cbbc3907c1bc:/etc/apt# curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg > /tmp/dkey; apt-key add /tmp/dkey

root@cbbc3907c1bc:/etc/apt# add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
   $(lsb_release -cs) \
   stable"

root@cbbc3907c1bc:/etc/apt# apt-get update

root@cbbc3907c1bc:/etc/apt# apt-get -y install docker-ce
```

安装完成后可用，这时候在jenkins中可以正常使用docker程序了。

注意：这时候systemctl并不可用，而且docker也不能配置为服务，但是不影响docker在pipeline中的使用。

### 3. 问题：关于yarn安装依赖卡在[3/4] Linking dependencies...

参考链接：https://stackoverflow.com/questions/50683248/what-does-linking-dependencies-during-npm-yarn-install-really-do

解决方式：尝试添加本地的缓存地址，缓存依赖包信息。

## 参考地址

关于docker安装的参考地址：
* https://blog.csdn.net/qq_40460909/article/details/83307369
* https://www.jianshu.com/p/88b5f436ffb0

## 重建jenkins镜像适应前端的构建行为

从github拉取jenkins的镜像的构建源文件，然后做出以下的修改：

* 切换sources.list，更换为国内源
* 安装maven并设置环境变量
* 安装nodejs以及必要的依赖包
* 安装docker程序包
* 插件预装

Dockerfile的编写