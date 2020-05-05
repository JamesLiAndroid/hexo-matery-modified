---
title: 项目迁移到k8s的配置修改和问题记录
top: false
cover: false
toc: true
mathjax: true
date: 2020-04-16 16:51:42
password:
summary:
tags:
categories:
---

# 项目迁移到k8s的配置修改和问题记录

# 项目迁移到k8s指南，配置修改

## 配置信息中各类连接的修改

1. nacos的连接地址，统一修改为以下链接：

http://nacos-0.nacos-headless.test-basic-nacos.svc.cluster.local:8848,http://nacos-1.nacos-headless.test-basic-nacos.svc.cluster.local:8848,http://nacos-2.nacos-headless.test-basic-nacos.svc.cluster.local:8848

注意：在服务日志中会出现nacos连接超时的情况，可以忽略

2. eureka的连接地址，统一修改为以下链接：

http://hoteam:hoteam2019@eureka-0.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/,http://hoteam:hoteam2019@eureka-1.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/,http://hoteam:hoteam2019@eureka-2.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/

3. mysql的连接信息，示例：

mysql-outer.test-basic-mysql-outer:3306

4. redis的连接信息

redis-outer.test-basic-redis-outer:6379


5. mongo的连接信息

mongo-outer.test-basic-mongo-outer:27017

## 数据库的迁移

数据库MySQL和MongoDB的迁移！

## CI/CD的改造

参考[CI-CD的改造](http://192.168.123.159:8181/icp-cloud/document/env-dev/blob/master/01.%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E/k8s%E7%9B%B8%E5%85%B3%E6%96%87%E6%A1%A3/CI-CD%E7%9A%84%E6%94%B9%E9%80%A0.md)一文。

## 迁移问题

1. kaptcha生成验证码，使用openjdk-8-alpine镜像遇到的关于字体的问题

gateway项目中，在Dockerfile中添加**apk --no-cache add ttf-dejavu fontconfig**内容，将字体包进行安装。

参考链接：https://blog.csdn.net/huofuman960209/article/details/100738712

2. 关于部分第三方jar包，如javax.mail、xalan:xalan:jar，推送的nexus依赖库中

单独把特定jar包推送到仓库中，例如

```
$ mvn deploy:deploy-file -DgroupId=com.xy.oracle -DartifactId=ojdbc14 -Dversion=10.2.0.4.0 -Dpackaging=jar -Dfile=E:\ojdbc14.jar -Durl=http://192.168.123.159:8081/nexus/content/repositories/thirdparty/ -DrepositoryId=thirdparty

```

或者直接通过nexus的在线页面，通过upload选项直接进行上传！

参考链接：https://www.cnblogs.com/rwxwsblog/p/6029636.html

3. k8s滚动升级时，单个pod更新耗时过长的问题，导致升级速度过于慢。


4. 关于前端构建nvm无法获取nodejs版本问题

前端构建失败，错误日志如下：

```
NVM is already installed

[test-frontend-develop] $ bash -c "export > env.txt"
[test-frontend-develop] $ bash -c "NVM_DIR=/var/jenkins_home/nvm && source $NVM_DIR/nvm.sh --no-use && NVM_NODEJS_ORG_MIRROR=https://nodejs.org/dist nvm install 10.12.0 && nvm use 10.12.0 && export > env.txt"
Version '10.12.0' not found - try `nvm ls-remote` to browse available versions.
ERROR: Failed to fork bash 

```

这种情况偶尔出现，常见于自动触发的方式，手动触发未出现问题。使用的是nodejs的官方镜像，两个参数如下：

```
	NVM_NODEJS_ORG_MIRROR	=   https://nodejs.org/dist
    NVM_IOJS_ORG_MIRROR     =   https://iojs.org/dist

```

使用临时的解决方式，修改NVM_NODEJS_ORG_MIRROR里面，将https变更为http。如下：

```
	NVM_NODEJS_ORG_MIRROR	=   http://nodejs.org/dist
    NVM_IOJS_ORG_MIRROR     =   https://iojs.org/dist

```

参考链接：https://blog.csdn.net/qq_43897372/article/details/104526660

5. 需要优化jenkins中前端构建耗时过长的问题

