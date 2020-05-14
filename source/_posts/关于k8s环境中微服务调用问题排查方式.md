---
title: 关于k8s环境中微服务调用问题排查方式
top: false
cover: false
toc: true
mathjax: true
date: 2020-05-14 10:43:44
password:
summary:
tags:
categories:
---

# 关于当前k8s环境中出现的服务调用失败的排查

## 问题概述

在五一节前的时候，出现了opt相关平台无法登录的情况，总是出现登录失败的反馈，前端调用后端接口时出现了调用失败的情况。而且伴随着前端出现了弹框的异常信息。涉及到test-user-uaa、test-umc以及test-opt-agent三个服务。对比开发环境中的服务，发现异常出现在test-user-uaa的接口调用部分。做出了如下的排查手段并实践！

## 排查方式

总结下微服务中排查错误的常见方式，如下：

1. 查看全链路监控中的信息

2. 查看相关服务的日志信息

3. 查看服务的配置信息

4. 对比git中的代码日志记录

5. 对比开发库和测试库的数据信息，测试相同数据的结果

6. 查看服务间是否有循环调用

7. 查看服务部署时的部署文件

8. 查看jar包的依赖信息

9. 重新部署服务

10. 打通k8s线上线下环境，流量引到线下，进行联调

上述是我这里的顺序，下面在展开分析中并未按照上述的顺序执行，但基本步骤是一致的。

## 展开分析

### 1. 查看全链路监控中的信息

很遗憾，我们的skywalking又挂了，我不得不重新进行启动。重启完成后，进行监控。从浏览器登录，看到请求的接口如下：

```
http://192.168.232.239:31081/api/uaa/oauth/token?checkCode=1111&randomStr=502c6ed51866ddeedee60f802e408029

```

然后找到Skywalking中关于该接口的监控信息如下：

![](skywalking监控.png)

由于后向请求出现错误，所以导致监控接口并无任何信息返回。

### 2. 查看相关服务的日志信息

由于我们尚未启用ELK日志系统，并不能在同一个位置查看所有的日志信息，这时候需要我们查看服务所在pod中的日志信息，需要先对pod进行缩减，从单一服务上进行查看。操作如下：

```
$ kubectl get pod -n test-all-service
NAME                                         READY   STATUS    RESTARTS   AGE
test-data-dictionary-7c7cf58696-7zrwd   1/1     Running   0          12d
test-filestore-opt-f9c5794f5-j2s2n      1/1     Running   0          8d
test-filestore-uc-9c89c7dc8-gdp7p       1/1     Running   0          8d
test-form-design-6457c7c855-6wj9k       1/1     Running   0          8d
test-frontend-5c458c5fbf-mbb7d          1/1     Running   0          2d
test-frontend-user-59d5fd489d-hnr6n     1/1     Running   0          2d14h
test-id-generator-59ccb5b667-bbp4k      1/1     Running   0          12d
test-id-generator-59ccb5b667-qqfj5      1/1     Running   1          12d
test-id-generator-59ccb5b667-z62d2      1/1     Running   0          12d
test-opt-agent-6667bfd8c8-8g8cz         1/1     Running   0          8d
test-opt-center-657785b66c-wrttd        1/1     Running   0          8d
test-opt-uaa-6c87879dcb-dcs69           1/1     Running   0          8d
test-user-uaa-5fb57fb898-552nz          1/1     Running   0          16h
test-user-uaa-5fb57fb898-8gl5b          1/1     Running   0          16h
test-user-uaa-5fb57fb898-ptxng          1/1     Running   0          16h
test-user-umc-bb58848d4-8nt5g           1/1     Running   0          16h
test-user-umc-bb58848d4-gnmnb           1/1     Running   0          16h
test-user-umc-bb58848d4-xn77n           1/1     Running   0          16h
test-visual-model-7547ff9984-klgqp      1/1     Running   0          8d
test-workflow-5dd8d98b9f-wnnfp          1/1     Running   0          8d


// 缩减相关服务数量
$ kubectl scale deployment test-user-uaa --replicas=1 -n test-all-service

$ kubectl scale deployment test-umc --replicas=1 -n test-all-service

$ kubectl scale deployment test-opt-agent --replicas=1 -n test-all-service

// 分别查看各个服务中的日志信息
$ kubectl logs -f --tail=100 test-opt-agent -n test-all-service
(无相关日志)

$ kubectl logs -f --tail=100 test-umc -n test-all-service
(无相关日志)

$ kubectl logs -f --tail=100 test-umc -n test-all-service
// 错误日志如下
************************************************************

Request received for POST '/oauth/token?checkCode=neya&randomStr=8a1ca2f6324261a4c7989e59773da07d':

org.apache.catalina.connector.RequestFacade@330f414b

servletPath:/oauth/token
pathInfo:null
headers:
content-length: 149
authorization: Basic aWNwLXVpOmhvdGVhbUAyMDE5
accept: application/json, text/plain, */*
origin: http://192.168.232.239:31081
user-agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36
timezone: 8
content-type: application/x-www-form-urlencoded;charset=UTF-8
referer: http://192.168.232.239:31081/user/login?redirect=%2F
accept-encoding: gzip, deflate
accept-language: en-US,en;q=0.9,zh-CN;q=0.8,zh;q=0.7,es;q=0.6
cookie: JSESSIONID=D6343899CA9DCEEA915300AC905FF141
forwarded: proto=http;host="192.168.232.239:31081";for="10.42.0.0:41273"
x-forwarded-for: 10.42.0.0
x-forwarded-proto: http
x-forwarded-prefix: /api/uaa
x-forwarded-port: 31081
x-forwarded-host: 192.168.232.239:31081
x-b3-traceid: 939275ae4315ffa7
x-b3-spanid: 174e0e570d7587bd
x-b3-parentspanid: 939275ae4315ffa7
x-b3-sampled: 0
host: 10.42.3.181:19060


Security filter chain: [
  WebAsyncManagerIntegrationFilter
  SecurityContextPersistenceFilter
  HeaderWriterFilter
  LogoutFilter
  ClientCredentialsTokenEndpointFilter
  BasicAuthenticationFilter
  RequestCacheAwareFilter
  SecurityContextHolderAwareRequestFilter
  AnonymousAuthenticationFilter
  SessionManagementFilter
  ExceptionTranslationFilter
  FilterSecurityInterceptor
]


************************************************************

************************************************************

Request received for POST '/error?checkCode=neya&randomStr=8a1ca2f6324261a4c7989e59773da07d':

org.apache.catalina.core.ApplicationHttpRequest@3b651af0

servletPath:/error
pathInfo:null
headers:
content-length: 149
accept: application/json, text/plain, */*
origin: http://192.168.232.239:31081
user-agent: Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36
authorization: Basic aWNwLXVpOmhvdGVhbUAyMDE5
timezone: 8
content-type: application/x-www-form-urlencoded;charset=UTF-8
referer: http://192.168.232.239:31081/user/login?redirect=%2F
accept-encoding: gzip, deflate
accept-language: en-US,en;q=0.9,zh-CN;q=0.8,zh;q=0.7,es;q=0.6
cookie: JSESSIONID=D6343899CA9DCEEA915300AC905FF141
forwarded: proto=http;host="192.168.232.239:31081";for="10.42.0.0:15508"
x-forwarded-for: 10.42.0.0
x-forwarded-proto: http
x-forwarded-prefix: /api/uaa
x-forwarded-port: 31081
x-forwarded-host: 192.168.232.239:31081
x-b3-traceid: 469d17bb77397455
x-b3-spanid: 242ce814c10dd625
x-b3-parentspanid: 469d17bb77397455
x-b3-sampled: 0
host: 10.42.3.181:19060


Security filter chain: [
  WebAsyncManagerIntegrationFilter
  SecurityContextPersistenceFilter
  HeaderWriterFilter
  CorsFilter
  LogoutFilter
  RequestCacheAwareFilter
  SecurityContextHolderAwareRequestFilter
  AnonymousAuthenticationFilter
  SessionManagementFilter
  ExceptionTranslationFilter
  FilterSecurityInterceptor
]


************************************************************

```

貌似是在请求/oauth/token接口，导致了错误，重定向到/error上了。但是这样并不能体现错误出现在哪里。

### 3. 尝试重新部署服务

通过jenkins手动触发服务的部署，但是并没有发现与之前有什么不同，错误始终是这样！

### 4. 对比git中的代码日志记录

利用sourcetree工具查看提交日志记录，k8s中部署的是release分支的功能，相对比较稳定，对比develop分支，不涉及相关服务代码的修改。所以暂时认为代码是稳定一致的！

### 5. 查看jar包的依赖信息

下载解压jenkins中构建的jar包，与本地打包的jar包进行比较。这部分缺乏合适的工具，只能是针对jar包反编译后，使用jd-gui工具，查看反编译后的java代码，利用vscode进行对比，结果发现并没有什么不同。实际上这一步可以不这样做，因为git提交日志基本相同，maven配置一致，基本上可以确定依赖是一样的。所以同小伙伴查看了是否有maven相关的依赖未提交到私有的依赖仓库，确定所有的依赖信息均正常。

### 6. 查看服务部署时的部署文件

查看这三个服务以及涉及的前端项目部署文件，主要查看环境变量设置，尤其是针对链接的配置项。截取这几个服务的配置信息如下：

```
// user-uaa部署配置信息部分

....
        env:
            - name: IP_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NACOS_IP
              value: 192.168.232.194:18848
            - name: NACOS_NAMESPACE
              value: 4cca5bd9-730b-499b-828e-875b0c0478d8
            - name: SKYWALKING_NAMESPACE
              value: test-k8s-test
            - name: SKYWALKING_TARGET_SERVICE_NAME
              value: test-user-uaa
            - name: SKYWALKING_IP_PORT
              value: 192.168.232.163:11800
            - name: CHANNEL
              value: standalone

// user-umc部署配置信息部分
....
        env:
            - name: IP_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NACOS_IP
              value: 192.168.232.194:18848
            - name: NACOS_NAMESPACE
              value: 4cca5bd9-730b-499b-828e-875b0c0478d8
            - name: SKYWALKING_NAMESPACE
              value: test-k8s-test
            - name: SKYWALKING_TARGET_SERVICE_NAME
              value: test-user-umc
            - name: SKYWALKING_IP_PORT
              value: 192.168.232.163:11800
            - name: CHANNEL
              value: standalone

// opt-agent部署配置信息部分
....
        env:
            - name: IP_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NACOS_IP
              value: 192.168.232.194:18848
            - name: NACOS_NAMESPACE
              value: 4cca5bd9-730b-499b-828e-875b0c0478d8
            - name: SKYWALKING_NAMESPACE
              value: test-k8s-test
            - name: SKYWALKING_TARGET_SERVICE_NAME
              value: test-opt-agent
            - name: SKYWALKING_IP_PORT
              value: 192.168.232.163:11800
            - name: CHANNEL
              value: standalone

// 前端测试信息
....
        env:
            - name: GATEWAY_HOST
              value: 192.168.232.241:31333
              #value: gateway.test-basic-gateway:19020 

```

查看配置信息一切正常，对nacos的链接，对Gateway的链接，均配置正确。

这个时候基本排除是构建方面产生的问题！

### 7. 打通k8s线上线下环境，流量引到线下，进行联调

参考[kt-connect使用](http://192.168.232.159:8181/test/document/env-dev/blob/master/01.%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E/k8s%E7%9B%B8%E5%85%B3%E6%96%87%E6%A1%A3/kt-connect%E7%9A%84%E5%AE%89%E8%A3%85%E5%92%8C%E4%BD%BF%E7%94%A8--%E5%AE%8C%E6%95%B4%E6%95%99%E7%A8%8B.md)的文章，对出现错误的user-uaa服务进行联调。

需要修改新的配置信息，将原有的standalone和基础配置文件换到peer0，修改对外连接数据库的地址信息，如下：

```test-user-uaa.yml
spring:
  application:
    name: test-user-uaa
  datasource:
    username: root
    password: ht@MySQL135
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://192.168.232.180:3306/test?characterEncoding=utf8&useUnicode=true&useSSL=false
    druid:
      initial-size: 5 # 初始大小
      min-idle: 5  # 最小
      max-active: 9  # 最大
      max-wait: 60000  # 连接超时时间
      time-between-eviction-runs-millis: 60000  # 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
      min-evictable-idle-time-millis: 300000  # 指定一个空闲连接最少空闲多久后可被清除，单位是毫秒
      validationQuery: select 'x'
      test-while-idle: true  # 当连接空闲时，是否执行连接测试
      test-on-borrow: false  # 当从连接池借用连接时，是否测试该连接
      test-on-return: false  # 在连接归还到连接池时是否测试该连接
      filters: config,wall,stat  # 配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙
      poolPreparedStatements: true
      maxPoolPreparedStatementPerConnectionSize: 20
      maxOpenPreparedStatements: 20
      web-stat-filter:
        enabled: true
        url-pattern: /*
        exclusions: /druid/*,*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico
        session-stat-enable: true
        session-stat-max-count: 10
      stat-view-servlet:
        enabled: true
        url-pattern: /druid/*
        reset-enable: true
        login-username: root
        login-password: ht@MySQL135

      filter:
        wall:
          enabled: true #配置wall filter
          db-type: mysql
          config:
            alter-table-allow: false
            truncate-allow: false
            drop-table-allow: false
            none-base-statement-allow: false #是否允许非以上基本语句的其他语句，缺省关闭，通过这个选项就能够屏蔽DDL。
            update-where-none-check: true #检查UPDATE语句是否无where条件，这是有风险的，但不是SQL注入类型的风险
            select-into-outfile-allow: false #SELECT ... INTO OUTFILE 是否允许，这个是mysql注入攻击的常见手段，缺省是禁止的
            metadata-allow: true #是否允许调用Connection.getMetadata方法，这个方法调用会暴露数据库的表信息
          log-violation: true #对被认为是攻击的SQL进行LOG.error输出
          throw-exception: true #对被认为是攻击的SQL抛出SQLExcepton
  cache:
    caffeine:
      spec: initialCapacity=50,maximumSize=500,expireAfterAccess=5s,expireAfterWrite=10s,refreshAfterWrite=5s

  redis: 
    host: 192.168.232.172
    port: 6379
    password: ht@Redis579
    timeout: 60000
    database: 0
    lettuce:
      pool:
        max-active: 8
        max-wait: -1
        max-idle: 8
        min-idle: 0

    jedis:
      pool:
        max-active: 8
        max-wait: -1
        max-idle: 8
        min-idle: 0

mybatis-plus:
  mapper-locations: classpath:mybatis/mapper/*Mapper.xml
  configuration:
    cache-enabled: true
    call-setters-on-nulls: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl

eureka:
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}@${spring.cloud.client.ip-address}:${server.port}
    ipAddress: ${spring.cloud.client.ip-address}
  client:
    service-url:
      defaultZone: http://hoteam:hoteam2019@192.168.232.239:31011/eureka/

swagger:
  enable: true

logging:
  level:
   com:
    hoteamsoft: debug

debug: true

hystrix:  
  command:  
    default:  
      execution: 
        timeout:
          enabled: true 
        isolation:  
          thread:  
            timeoutInMilliseconds: 30000
  threadpool:  
    default:  
      coreSize: 1000
feign:
  client:
    config:
      default:
        connectTimeout: 30000
        readTimeout: 30000
ribbon:
  ReadTimeout: 30000
  ConnectTimeout: 30000
  MaxAutoRetries: 3
  MaxAutoRetriesNextServer: 1

server:
  port: 19060

```

准备好配置文件，准备好影子镜像，启动本地的服务，进行联调，发现是可以正常登录的！

这时候已经有些懵圈了！配置信息基本相同，只是将内网连接改为真实的ip地址连接，以便于线下运行的需要！

### 8. 查看服务间是否有循环调用

同小伙伴确定了，这三个微服务中的接口不存在循环调用的情况，也就不会出现接口互相调用导致服务出现问题的可能。

### 9. 对比开发库和测试库的数据信息，测试相同数据的结果

尝试将开发库的用户信息，复制到测试库中，修改为相同的密码设置，同时在开发环境和测试环境进行登录，发现两个系统均可以登录，基本排除前后端代码因为加密的问题，导致无法登录的情况。

最后停止调试，恢复集群中的正常服务！

### 10. 查看服务的配置信息

前面对比了user-uaa服务新的peer0的配置文件同k8s测试环境原有配置文件的情况，基本一致，这次对比的时候，对比开发环境和k8s测试环境中的配置文件信息，发现多出了以下内容：

```
hystrix:  
  command:  
    default:  
      execution: 
        timeout:
          enabled: true 
        isolation:  
          thread:  
            timeoutInMilliseconds: 30000
  threadpool:  
    default:  
      coreSize: 1000
feign:
  client:
    config:
      default:
        connectTimeout: 30000
        readTimeout: 30000
ribbon:
  ReadTimeout: 30000
  ConnectTimeout: 30000
  MaxAutoRetries: 3
  MaxAutoRetriesNextServer: 1

```

我尝试删除这些配置信息，重新启动程序，发现并不能解决这个问题。

但是我的小伙伴对比了umc服务中开发环境和k8s测试环境中的配置文件信息，也是多出了上述内容，删除后，k8s测试环境中的登录信息恢复正常！

世界太平了，正常了。

## 总结

0. 合理的排查信息的流程性总结，梳理正常的排查流程。

1. 加强全链路监控的建设，选择合适的技术栈，全面掌握其使用。

2. 启用统一的日志管理系统，增强日志收集和筛选能力。

3. 强化对微服务的监控手段，强化报警能力，设定阈值信息。

4. 非线上环境，可以将服务缩减为单个，然后监控单个服务的日志情况。

## 配置信息解析

重新贴一下“多余”的配置信息，并逐条进行解释设定，添加默认值：

```
hystrix:  
  command:  
    default:  
      execution: 
        timeout:
          enabled: true   # 默认启动hystrix熔断
        isolation:  
          thread:  
            timeoutInMilliseconds: 30000 # 该属性用来配 置HystrixCommand执行的超时时间， 单位为毫秒。当HystrixCommand执行 时间超过该配置值之后， Hystrix会将该执行命令标记为TIMEOUT并进入服务降级 处理逻辑。默认值1000
  threadpool:  
    default:  
      coreSize: 1000      # 用来控制Hystrix命令所属线程池的配置
feign:
  client:
    config:
      default:
        connectTimeout: 30000  # 接口调用超时时间
        readTimeout: 30000
ribbon:
  ReadTimeout: 30000
  ConnectTimeout: 30000
  MaxAutoRetries: 3          #同一台实例最大重试次数,不包括首次调用
  MaxAutoRetriesNextServer: 1  #重试负载均衡其他的实例最大重试次数,不包括首次调用

```

根据我们的集群环境，结合上述配置信息，怀疑问题在以下三点上：

1. 超时时间的设置不合理。
2. 重试次数设置不合理。
3. k8s中对单个pod的资源限制，导致配置文件中的配置项出现了问题。

需要针对上述信息进行重试！