---
title: 关于nacos的问题排查
top: false
cover: false
toc: true
mathjax: true
date: 2021-05-24 11:12:56
password:
summary:
tags:
categories:
---

# 关于甲方方面在nacos下的超时问题处理

## 问题背景描述

起因源于甲方方面从防火墙开放端口后，通过vpn连接到甲方的网络中，在本地通过intellij idea启动uaa服务，看是否能注册到甲方的nacos中。

通过测试，并不能注册到甲方的nacos中，日志信息如下：

```log

D:\dev\Java\jdk1.8.0_201\bin\java.exe -Dnacos_ip=192.168.62.52:18848 -Dnacos_namespace=6e63890a-4d2c-4f6b-a2b8-529984c38b18 -XX:TieredStopAtLevel=1 -noverify -Dspring.profiles.active=test -Dspring.output.ansi.enabled=always -Dcom.sun.management.jmxremote -Dspring.jmx.enabled=true -Dspring.liveBeansView.mbeanDomain -Dspring.application.admin.enabled=true "-javaagent:D:\dev\Java\IntelliJ IDEA 2019.1.3\lib\idea_rt.jar=59908:D:\dev\Java\IntelliJ IDEA 2019.1.3\bin" -Dfile.encoding=UTF-8 -classpath C:\Users\ht\AppData\Local\Temp\classpath1285028913.jar com.company.test.testone.uaa.BaseStoneUaaServer
           ______________        ________                __

::  :: testone-uaa: test: v1.0.1-SNAPSHOT ::
:: Spring-Boot: 2.3.9.RELEASE :: 
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:14:56.205 ERROR 13728 [] [main] c.a.n.c.config.http.ServerHttpAgent      : [NACOS SocketTimeoutException httpGet] currentServerAddr:http://192.168.62.52:18848， err : connect timed out
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:14:57.206 ERROR 13728 [] [main] c.a.n.c.config.http.ServerHttpAgent      : [NACOS SocketTimeoutException httpGet] currentServerAddr:http://192.168.62.52:18848， err : connect timed out
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:14:58.206 ERROR 13728 [] [main] c.a.n.c.config.http.ServerHttpAgent      : [NACOS SocketTimeoutException httpGet] currentServerAddr:http://192.168.62.52:18848， err : connect timed out
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:14:58.206 ERROR 13728 [] [main] c.a.n.c.config.http.ServerHttpAgent      : no available server
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:14:58.209 ERROR 13728 [] [main] c.a.n.client.config.impl.ClientWorker    : [fixed-192.168.62.52_18848-6e63890a-4d2c-4f6b-a2b8-529984c38b18] [sub-server] get server config exception, dataId=testone-uaa, group=DEFAULT_GROUP, tenant=6e63890a-4d2c-4f6b-a2b8-529984c38b18

java.net.ConnectException: no available server
	at com.alibaba.nacos.client.config.http.ServerHttpAgent.httpGet(ServerHttpAgent.java:133)
	at com.alibaba.nacos.client.config.http.MetricsHttpAgent.httpGet(MetricsHttpAgent.java:51)
	at com.alibaba.nacos.client.config.impl.ClientWorker.getServerConfig(ClientWorker.java:298)
	at com.alibaba.nacos.client.config.NacosConfigService.getConfigInner(NacosConfigService.java:149)
	at com.alibaba.nacos.client.config.NacosConfigService.getConfig(NacosConfigService.java:97)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceBuilder.loadNacosData(NacosPropertySourceBuilder.java:85)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceBuilder.build(NacosPropertySourceBuilder.java:74)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceLocator.loadNacosPropertySource(NacosPropertySourceLocator.java:204)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceLocator.loadNacosDataIfPresent(NacosPropertySourceLocator.java:191)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceLocator.loadApplicationConfiguration(NacosPropertySourceLocator.java:142)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceLocator.locate(NacosPropertySourceLocator.java:103)
	at org.springframework.cloud.bootstrap.config.PropertySourceLocator.locateCollection(PropertySourceLocator.java:52)
	at org.springframework.cloud.bootstrap.config.PropertySourceLocator.locateCollection(PropertySourceLocator.java:47)
	at org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration.initialize(PropertySourceBootstrapConfiguration.java:98)
	at org.springframework.boot.SpringApplication.applyInitializers(SpringApplication.java:626)
	at org.springframework.boot.SpringApplication.prepareContext(SpringApplication.java:370)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:314)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1237)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1226)
	at com.company.test.testone.uaa.BaseStoneUaaServer.main(BaseStoneUaaServer.java:35)

[testone-uaa:10.0.93.182:0000] 2021-05-24 09:14:58.211 WARN 13728 [] [main] c.a.c.n.c.NacosPropertySourceBuilder     : Ignore the empty nacos configuration and get it based on dataId[testone-uaa] & group[DEFAULT_GROUP]
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:14:59.212 ERROR 13728 [] [main] c.a.n.c.config.http.ServerHttpAgent      : [NACOS SocketTimeoutException httpGet] currentServerAddr:http://192.168.62.52:18848， err : connect timed out
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:00.212 ERROR 13728 [] [main] c.a.n.c.config.http.ServerHttpAgent      : [NACOS SocketTimeoutException httpGet] currentServerAddr:http://192.168.62.52:18848， err : connect timed out
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:01.212 ERROR 13728 [] [main] c.a.n.c.config.http.ServerHttpAgent      : [NACOS SocketTimeoutException httpGet] currentServerAddr:http://192.168.62.52:18848， err : connect timed out
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:02.212 ERROR 13728 [] [main] c.a.n.c.config.http.ServerHttpAgent      : [NACOS SocketTimeoutException httpGet] currentServerAddr:http://192.168.62.52:18848， err : connect timed out
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:02.212 ERROR 13728 [] [main] c.a.n.client.config.impl.ClientWorker    : [fixed-192.168.62.52_18848-6e63890a-4d2c-4f6b-a2b8-529984c38b18] [sub-server] get server config exception, dataId=testone-uaa.yml, group=DEFAULT_GROUP, tenant=6e63890a-4d2c-4f6b-a2b8-529984c38b18

java.net.ConnectException: [NACOS HTTP-GET] The maximum number of tolerable server reconnection errors has been reached
	at com.alibaba.nacos.client.config.http.ServerHttpAgent.httpGet(ServerHttpAgent.java:124)
	at com.alibaba.nacos.client.config.http.MetricsHttpAgent.httpGet(MetricsHttpAgent.java:51)
	at com.alibaba.nacos.client.config.impl.ClientWorker.getServerConfig(ClientWorker.java:298)
	at com.alibaba.nacos.client.config.NacosConfigService.getConfigInner(NacosConfigService.java:149)
	at com.alibaba.nacos.client.config.NacosConfigService.getConfig(NacosConfigService.java:97)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceBuilder.loadNacosData(NacosPropertySourceBuilder.java:85)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceBuilder.build(NacosPropertySourceBuilder.java:74)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceLocator.loadNacosPropertySource(NacosPropertySourceLocator.java:204)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceLocator.loadNacosDataIfPresent(NacosPropertySourceLocator.java:191)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceLocator.loadApplicationConfiguration(NacosPropertySourceLocator.java:145)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceLocator.locate(NacosPropertySourceLocator.java:103)
	at org.springframework.cloud.bootstrap.config.PropertySourceLocator.locateCollection(PropertySourceLocator.java:52)
	at org.springframework.cloud.bootstrap.config.PropertySourceLocator.locateCollection(PropertySourceLocator.java:47)
	at org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration.initialize(PropertySourceBootstrapConfiguration.java:98)
	at org.springframework.boot.SpringApplication.applyInitializers(SpringApplication.java:626)
	at org.springframework.boot.SpringApplication.prepareContext(SpringApplication.java:370)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:314)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1237)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1226)
	at com.company.test.testone.uaa.BaseStoneUaaServer.main(BaseStoneUaaServer.java:35)

[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:02.212 WARN 13728 [] [main] c.a.c.n.c.NacosPropertySourceBuilder     : Ignore the empty nacos configuration and get it based on dataId[testone-uaa.yml] & group[DEFAULT_GROUP]
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:03.214 ERROR 13728 [] [main] c.a.n.c.config.http.ServerHttpAgent      : [NACOS SocketTimeoutException httpGet] currentServerAddr:http://192.168.62.52:18848， err : connect timed out
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:04.214 ERROR 13728 [] [main] c.a.n.c.config.http.ServerHttpAgent      : [NACOS SocketTimeoutException httpGet] currentServerAddr:http://192.168.62.52:18848， err : connect timed out
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:05.214 ERROR 13728 [] [main] c.a.n.c.config.http.ServerHttpAgent      : [NACOS SocketTimeoutException httpGet] currentServerAddr:http://192.168.62.52:18848， err : connect timed out
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:05.214 ERROR 13728 [] [main] c.a.n.c.config.http.ServerHttpAgent      : no available server
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:05.214 ERROR 13728 [] [main] c.a.n.client.config.impl.ClientWorker    : [fixed-192.168.62.52_18848-6e63890a-4d2c-4f6b-a2b8-529984c38b18] [sub-server] get server config exception, dataId=testone-uaa-test.yml, group=DEFAULT_GROUP, tenant=6e63890a-4d2c-4f6b-a2b8-529984c38b18

java.net.ConnectException: no available server
	at com.alibaba.nacos.client.config.http.ServerHttpAgent.httpGet(ServerHttpAgent.java:133)
	at com.alibaba.nacos.client.config.http.MetricsHttpAgent.httpGet(MetricsHttpAgent.java:51)
	at com.alibaba.nacos.client.config.impl.ClientWorker.getServerConfig(ClientWorker.java:298)
	at com.alibaba.nacos.client.config.NacosConfigService.getConfigInner(NacosConfigService.java:149)
	at com.alibaba.nacos.client.config.NacosConfigService.getConfig(NacosConfigService.java:97)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceBuilder.loadNacosData(NacosPropertySourceBuilder.java:85)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceBuilder.build(NacosPropertySourceBuilder.java:74)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceLocator.loadNacosPropertySource(NacosPropertySourceLocator.java:204)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceLocator.loadNacosDataIfPresent(NacosPropertySourceLocator.java:191)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceLocator.loadApplicationConfiguration(NacosPropertySourceLocator.java:150)
	at com.alibaba.cloud.nacos.client.NacosPropertySourceLocator.locate(NacosPropertySourceLocator.java:103)
	at org.springframework.cloud.bootstrap.config.PropertySourceLocator.locateCollection(PropertySourceLocator.java:52)
	at org.springframework.cloud.bootstrap.config.PropertySourceLocator.locateCollection(PropertySourceLocator.java:47)
	at org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration.initialize(PropertySourceBootstrapConfiguration.java:98)
	at org.springframework.boot.SpringApplication.applyInitializers(SpringApplication.java:626)
	at org.springframework.boot.SpringApplication.prepareContext(SpringApplication.java:370)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:314)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1237)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1226)
	at com.company.test.testone.uaa.BaseStoneUaaServer.main(BaseStoneUaaServer.java:35)

[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:05.214 WARN 13728 [] [main] c.a.c.n.c.NacosPropertySourceBuilder     : Ignore the empty nacos configuration and get it based on dataId[testone-uaa-test.yml] & group[DEFAULT_GROUP]
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:05.215 INFO 13728 [] [main] b.c.PropertySourceBootstrapConfiguration : Located property source: [BootstrapPropertySource {name='bootstrapProperties-testone-uaa-test.yml,DEFAULT_GROUP'}, BootstrapPropertySource {name='bootstrapProperties-testone-uaa.yml,DEFAULT_GROUP'}, BootstrapPropertySource {name='bootstrapProperties-testone-uaa,DEFAULT_GROUP'}]
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:05.219 INFO 13728 [] [main] c.h.i.testone.uaa.BaseStoneUaaServer   : The following profiles are active: test
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:05.806 WARN 13728 [] [main] ConfigServletWebServerApplicationContext : Exception encountered during context initialization - cancelling refresh attempt: org.springframework.beans.factory.BeanDefinitionStoreException: Failed to process import candidates for configuration class [com.company.test.testone.uaa.BaseStoneUaaServer]; nested exception is java.lang.IllegalStateException: Error processing condition on org.apache.rocketmq.spring.autoconfigure.RocketMQAutoConfiguration
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:05.812 INFO 13728 [] [main] ConditionEvaluationReportLoggingListener : 

Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
[testone-uaa:10.0.93.182:0000] 2021-05-24 09:15:05.823 ERROR 13728 [] [main] o.s.boot.SpringApplication               : Application run failed

org.springframework.beans.factory.BeanDefinitionStoreException: Failed to process import candidates for configuration class [com.company.test.testone.uaa.BaseStoneUaaServer]; nested exception is java.lang.IllegalStateException: Error processing condition on org.apache.rocketmq.spring.autoconfigure.RocketMQAutoConfiguration
	at org.springframework.context.annotation.ConfigurationClassParser.processImports(ConfigurationClassParser.java:610)
	at org.springframework.context.annotation.ConfigurationClassParser.access$800(ConfigurationClassParser.java:111)
	at org.springframework.context.annotation.ConfigurationClassParser$DeferredImportSelectorGroupingHandler.lambda$processGroupImports$1(ConfigurationClassParser.java:812)
	at java.util.ArrayList.forEach(ArrayList.java:1257)
	at org.springframework.context.annotation.ConfigurationClassParser$DeferredImportSelectorGroupingHandler.processGroupImports(ConfigurationClassParser.java:809)
	at org.springframework.context.annotation.ConfigurationClassParser$DeferredImportSelectorHandler.process(ConfigurationClassParser.java:780)
	at org.springframework.context.annotation.ConfigurationClassParser.parse(ConfigurationClassParser.java:193)
	at org.springframework.context.annotation.ConfigurationClassPostProcessor.processConfigBeanDefinitions(ConfigurationClassPostProcessor.java:319)
	at org.springframework.context.annotation.ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry(ConfigurationClassPostProcessor.java:236)
	at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanDefinitionRegistryPostProcessors(PostProcessorRegistrationDelegate.java:280)
	at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(PostProcessorRegistrationDelegate.java:96)
	at org.springframework.context.support.AbstractApplicationContext.invokeBeanFactoryPostProcessors(AbstractApplicationContext.java:707)
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:533)
	at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:143)
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:758)
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:750)
	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:405)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:315)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1237)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1226)
	at com.company.test.testone.uaa.BaseStoneUaaServer.main(BaseStoneUaaServer.java:35)
Caused by: java.lang.IllegalStateException: Error processing condition on org.apache.rocketmq.spring.autoconfigure.RocketMQAutoConfiguration
	at org.springframework.boot.autoconfigure.condition.SpringBootCondition.matches(SpringBootCondition.java:60)
	at org.springframework.context.annotation.ConditionEvaluator.shouldSkip(ConditionEvaluator.java:108)
	at org.springframework.context.annotation.ConfigurationClassParser.processConfigurationClass(ConfigurationClassParser.java:226)
	at org.springframework.context.annotation.ConfigurationClassParser.processImports(ConfigurationClassParser.java:600)
	... 20 common frames omitted
Caused by: java.lang.IllegalArgumentException: Could not resolve placeholder 'test.rocketmq.name-server' in value "${test.rocketmq.name-server}"
	at org.springframework.util.PropertyPlaceholderHelper.parseStringValue(PropertyPlaceholderHelper.java:178)
	at org.springframework.util.PropertyPlaceholderHelper.replacePlaceholders(PropertyPlaceholderHelper.java:124)
	at org.springframework.core.env.AbstractPropertyResolver.doResolvePlaceholders(AbstractPropertyResolver.java:239)
	at org.springframework.core.env.AbstractPropertyResolver.rhMck9sCCSTtRdH84JvGe1yR2Sqg5tE82E(AbstractPropertyResolver.java:210)
	at org.springframework.core.env.AbstractPropertyResolver.rhMck9sCCSTtRdH84JvGe1yR2Sqg5tE82E(AbstractPropertyResolver.java:230)
	at org.springframework.core.env.PropertySourcesPropertyResolver.getProperty(PropertySourcesPropertyResolver.java:88)
	at org.springframework.core.env.PropertySourcesPropertyResolver.getProperty(PropertySourcesPropertyResolver.java:62)
	at org.springframework.core.env.AbstractEnvironment.getProperty(AbstractEnvironment.java:535)
	at org.springframework.boot.autoconfigure.condition.OnPropertyCondition$Spec.collectProperties(OnPropertyCondition.java:140)
	at org.springframework.boot.autoconfigure.condition.OnPropertyCondition$Spec.access$000(OnPropertyCondition.java:105)
	at org.springframework.boot.autoconfigure.condition.OnPropertyCondition.determineOutcome(OnPropertyCondition.java:91)
	at org.springframework.boot.autoconfigure.condition.OnPropertyCondition.getMatchOutcome(OnPropertyCondition.java:55)
	at org.springframework.boot.autoconfigure.condition.SpringBootCondition.matches(SpringBootCondition.java:47)
	... 23 common frames omitted


Process finished with exit code 1

```

同网络断开时的情况一样，出现了超时的情况。

## 排查过程

病急乱投医，首先测试问题，尝试排除ide的影响，排除依赖信息的影响，最后排除网络的情况。

### 0. 从命令行启动jar包

参考Dockerfile中的启动命令，进行到target目录下，执行：

```shell

java -Dloader.path="libs/" -Dspring.profiles.active=test -Dnacos_ip=192.168.62.52:18848 -Dnacos_namespace=6e63890a-4d2c-4f6b-a2b8-529984c38b18 -jar testone-uaa-1.0.1-SNAPSHOT.jar

```

错误日志相同。

### 1. 从命令行发起请求

```
$ curl -XGET -k -i "http://192.168.62.52:18848/nacos/v1/cs/configs?dataId=testone-uaa-test.yml&group=DEFAULT_GROUP&tenant=6e63890a-4d2c-4f6b-a2b8-529984c38b18"
HTTP/1.1 200
Config-Type: yaml
Content-MD5: eb5fb35c428480018fe894695c82f596
Pragma: no-cache
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Cache-Control: no-cache,no-store
Last-Modified: Thu, 29 Apr 2021 08:17:24 GMT
Content-Type: text/plain;charset=UTF-8
Transfer-Encoding: chunked
Date: Mon, 24 May 2021 02:50:17 GMT

server:
  port: 20111
  http2:
      enabled: true
spring:
  cloud:
    nacos:
      discovery:
        service: ${spring.application.name}
        ip: ${spring.cloud.client.ip-address}
        port: ${server.port}
......

```

从命令行请求发现，HTTP协议内容可以进行传递。排除接口不通的问题。

### 2. 排除其它依赖的影响

尝试构建一个最小的服务，排除其它的依赖信息。以之前给甲方方面的例子为测试对象。错误日志还是相同。

### 3. 抓包

#### 3.1 有无VPN的情况下

在vpn运行和vpn不运行的情况下，进行抓包测试，得到相同的结果。发现如下：

![](error.jpg)

图片展示的信息为重试心跳请求的抓包信息，看到无ACK返回值。请求协议走TCP，而不是HTTP。

#### 3.2 甲方方面地址和乙方内部地址对比

切换到乙方方面的nacos请求信息，抓包信息如下：

![](success.jpg)

结果发现，在请求接口之前客户端先发起了一个心跳请求，然后再开始请求接口。前面是TCP的心跳连接，心跳达成后，才进行接口的请求。这是这个部分的请求逻辑。

## 最终解决方式

于是得出一个结论，在心跳无法连接或者说心跳没有ACK应答的时候，nacos客户端不会发起对HTTP的请求。

这样只能解决端口的TCP访问问题，需要开放端口的TCP访问权限，否则无法使用。

## 总结

1. 抓包是个好办法，尤其是解决网络问题，优先考虑抓包的情况。wireshark用起来还是比较麻烦的，需要找个更易用更直观的工具。
2. 感觉还是瘸腿，尤其是对HTTP协议和TCP/IP协议了解不够，需要补充这部分内容。
3. 甲方的在开放端口的时候并未告诉我是否开放的TCP访问还是HTTP访问，所以这事儿后续在做WAF的时候要注意！
4. 遇事不决，先抓包看代码，然后考虑各个链路上的问题！
