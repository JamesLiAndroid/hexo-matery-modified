---
title: Springboot拆分依赖构建微服务方案
top: false
cover: false
toc: true
mathjax: true
date: 2020-12-28 15:39:31
password:
summary:
tags:
categories:
---

# Springboot拆分打包并进行构建的调整（调整图片）

## 前言

目前在Springboot微服务中，在构建时进行打包的流程中，具体的产物为fat jar，也就是将所有的依赖信息以及业务代码全部打包为一个jar包，这样不仅在构建速度上拖慢，而且每次都要生成新的jar包。有没有可能将公共的依赖包剥离出来，只对业务代码进行打包，待运行时在将二者放在一起运行？

## 基础

Docker 为了节约存储空间，所以采用了分层存储概念。共享数据会对镜像和容器进行分层，不同镜像可以共享相同数据，并且在镜像上为容器分配一个 RW 层来加快容器的启动顺序。

在构建镜像的过程中 Docker 将按照 Dockerfile 中指定的顺序逐步执行 Dockerfile 中的指令。随着每条指令的检查，Docker 将在其缓存中查找可重用的现有镜像，而不是创建一个新的（重复）镜像。

Dockerfile 的每一行命令都创建新的一层，包含了这一行命令执行前后文件系统的变化。为了优化这个过程，Docker 使用了一种缓存机制：只要这一行命令不变，那么结果和上一次是一样的，直接使用上一次的结果即可。

为了充分利用层级缓存，我们必须要理解 Dockerfile 中的命令行是如何工作的，尤其是RUN，ADD和COPY这几个命令。一个好的docker 镜像 应该是合理分层 减少变动次数 从而加快打包的速度和分发的速度。

我们在优化的时候也是借鉴这个特性，同时参考之前前端构建的特性，将不怎么变动的依赖jar包集合抽取出来，作为一个不怎么变更的层，然后将业务代码打为一个jar包，在运行时手动指定业务jar包依赖的jar包集合，从而实现构建提速。

## 方案

目前具体的实现有两种方式：

1. 依托Springboot 2.3.0的版本特性支持，使用自带的分层打包机制进行。由于我们使用的还是Springboot 2.2.6，因此先搁置这一方案。
2. 利用maven插件分离依赖jar包集合和业务jar包。这是目前所采用的。

## 修改

这里使用业务建模的微服务来进行测试，服务名为：test-visualmodel，[项目所在](http://192.168.123.202:8181/test/business/test-vm/tree/feature.build.docker_depart)分支为：feature.build.docker_depart。

首先看一下整个项目的目录结构：

![](./imgs/项目目录结构.png)

需要注意的是，修改时要在module的内部的pom文件中进行修改，不能在最外部修改，否则容易造成生成的jar包集合路径不正确，导致找不到依赖，无法启动业务jar包。

在修改之前，需要确定的是，修改maven打包的部分，也就是<plugin>标签对应的内容。

下面对module内的pom文件进行修改如下：

```
<!-- 增加以下信息 -->
    <build>
        <plugins>
            <!-- 设置 SpringBoot 打包插件只包含业务代码相关的部分，不包含外部的依赖jar包 -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <layout>ZIP</layout>
                    <includes>
                        <include>
                            <groupId>${project.groupId}</groupId>
                            <artifactId>${project.artifactId}</artifactId>
                        </include>
                    </includes>
                </configuration>
            </plugin>
            <!--设置将 lib 拷贝到应用 Jar 外面，而且和生成的业务jar包平级 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-dependency-plugin</artifactId>
                <executions>
                    <execution>
                        <id>copy-dependencies</id>
                        <!-- 这里表示在打包之前执行下面的操作 -->
                        <phase>prepare-package</phase>
                        <goals>
                            <goal>copy-dependencies</goal>
                        </goals>
                        <configuration>
                            <type>jar</type>
                            <includeTypes>jar</includeTypes>
                            <includeScope>runtime</includeScope>
                            <!-- 这里解释为何在module内部进行配置， project.build.directory直接定位到module内部生成的target目录下 -->
                            <outputDirectory>${project.build.directory}/libs</outputDirectory>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

```

这样基本上就可以在本地进行测试了，在命令行中使用*mvn clean install*命令执行，打包后，target目录下的结构如下：

![](./imgs/target目录信息.png)

这样libs就是剥离出的jar包依赖集合，目录中的icp开头的jar包自然就是我们的业务jar包。这样拆分后该jar包从以前的70多MB大小，降到300多kb大小，就加速了构建时copy操作的速度。

最后需要进行测试，相比之前启动命令发生了变化，如下：

```
java  -Dloader.path="libs/" -jar test-opt-visualmodel-1.0.0-SNAPSHOT.jar

```

添加了**-Dloader.path="libs/"*参数，指定了依赖jar包集合所在的位置。即可正常启动。

进入构建流水线之前，最后还要调整一下Dockerfile的编写，如下：

```Dockerfile
### 各微服务运行时所构建的镜像
### 用该文件替换项目中的Dockerfile

# 以基础镜像为底，执行
FROM 192.168.123.202:5000/base_backend:0.0.1

# 构建变更
# 将依赖包单独剥离出来，转到lib目录下
# 复制libs依赖信息到对应目录下
COPY ./target/libs/ /app/libs/
# 复制jar包所在地址 到 目标地址  --将jar包拷贝到镜像系统的相应地址
COPY ./target/test-opt-visualmodel-1.0.0-SNAPSHOT.jar /app

# 设置运行的加入点
ENTRYPOINT ["sh", "-c", "java -Dloader.path=\"libs/\" -XX:MaxRAMPercentage=75.0 -XX:MaxRAM=1000m -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -javaagent:/app/skywalking-agent/skywalking-agent.jar -Dskywalking.agent.namespace=${SKYWALKING_NAMESPACE} -Dskywalking.agent.service_name=${SKYWALKING_TARGET_SERVICE_NAME} -Dskywalking.collector.backend_service=${SKYWALKING_IP_PORT}  -Dspring.profiles.active=${CHANNEL} -Dspring.cloud.client.ip-address=${IP_ADDR} -Dnacos_ip=${NACOS_IP} -Dnacos_namespace=${NACOS_NAMESPACE} -jar test-opt-visualmodel-1.0.0-SNAPSHOT.jar"]

```

主要是修改了两点，一是将依赖jar包集合所在的文件夹copy到对应目录，另一个就是启动命令指定依赖jar包的路径信息。

## 总结

需要调整好多东西，参考文章中的内容需要辩证看待，另外，也需要自己去实践，将方案做出来！

## 参考文章

* [Springboot2.3.0分层打包](https://blog.csdn.net/ttzommed/article/details/106759670)
* [利用maven插件分离依赖jar包集合和业务jar包](https://www.jianshu.com/p/32456eea0488)
* https://blog.csdn.net/weixin_44460333/article/details/104624236
* https://juejin.cn/post/6844904119338008583
* [Docker参考](https://blog.csdn.net/weixin_44460333/article/details/103020487)