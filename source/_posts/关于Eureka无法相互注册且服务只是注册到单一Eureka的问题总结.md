---
title: 关于Eureka无法相互注册且服务只是注册到单一Eureka的问题总结
top: false
cover: false
toc: true
mathjax: true
date: 2020-07-30 09:58:32
password:
summary:
tags:
categories:
---

# 关于Eureka无法相互注册且服务只是注册到单一Eureka的问题总结

## 场景描述

我们有三个Eureka服务，以206、207、208表征。

在昨天的问题排查中，出现了以下的场景：

1. Eureka不能相互注册成为集群
2. 有两个Eureka相互注册成功，也就是206、208相互注册成功，207游离
3. 线上环境，新启动的业务服务注册到了游离的Eureka服务上，导致网关侧以及远程Restful调用的时候找不到业务服务，报错
4. 线下启动业务服务，可以注册到三个Eureka上

## 问题排查

通过ELK日志统一收集系统，查看错误日志信息，我们目前只对206服务进行了日志采集，先来看206的日志信息：

```
2020-07-23 17:31:49.164 ERROR [test-register,,,] 12962 --- [_192.168.128.207-13] c.n.e.cluster.ReplicationTaskProcessor   : Network level connection to peer 192.168.128.207; retrying after delay

com.sun.jersey.api.client.ClientHandlerException: java.net.ConnectException: Connection refused (Connection refused)
        at com.sun.jersey.client.apache4.ApacheHttpClient4Handler.handle(ApacheHttpClient4Handler.java:187) ~[jersey-apache-client4-1.19.1.jar!/:1.19.1]
        at com.netflix.eureka.cluster.DynamicGZIPContentEncodingFilter.handle(DynamicGZIPContentEncodingFilter.java:48) ~[eureka-core-1.9.13.jar!/:1.9.13]
        at com.netflix.discovery.EurekaIdentityHeaderFilter.handle(EurekaIdentityHeaderFilter.java:27) ~[eureka-client-1.9.13.jar!/:1.9.13]
        at com.sun.jersey.api.client.Client.handle(Client.java:652) ~[jersey-client-1.19.1.jar!/:1.19.1]
        at com.sun.jersey.api.client.WebResource.handle(WebResource.java:682) ~[jersey-client-1.19.1.jar!/:1.19.1]
        at com.sun.jersey.api.client.WebResource.access$200(WebResource.java:74) ~[jersey-client-1.19.1.jar!/:1.19.1]
        at com.sun.jersey.api.client.WebResource$Builder.post(WebResource.java:570) ~[jersey-client-1.19.1.jar!/:1.19.1]
        at com.netflix.eureka.transport.JerseyReplicationClient.submitBatchUpdates(JerseyReplicationClient.java:117) ~[eureka-core-1.9.13.jar!/:1.9.13]
        at com.netflix.eureka.cluster.ReplicationTaskProcessor.process(ReplicationTaskProcessor.java:80) ~[eureka-core-1.9.13.jar!/:1.9.13]
        at com.netflix.eureka.util.batcher.TaskExecutors$BatchWorkerRunnable.run(TaskExecutors.java:193) [eureka-core-1.9.13.jar!/:1.9.13]
        at java.lang.Thread.run(Thread.java:748) [na:1.8.0_202]
Caused by: java.net.ConnectException: Connection refused (Connection refused)
        at java.net.PlainSocketImpl.socketConnect(Native Method) ~[na:1.8.0_202]
        at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350) ~[na:1.8.0_202]
        at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206) ~[na:1.8.0_202]
        at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188) ~[na:1.8.0_202]
        at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392) ~[na:1.8.0_202]
        at java.net.Socket.connect(Socket.java:589) ~[na:1.8.0_202]
        at org.apache.http.conn.scheme.PlainSocketFactory.connectSocket(PlainSocketFactory.java:121) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.conn.DefaultClientConnectionOperator.openConnection(DefaultClientConnectionOperator.java:180) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.conn.AbstractPoolEntry.open(AbstractPoolEntry.java:144) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.conn.AbstractPooledConnAdapter.open(AbstractPooledConnAdapter.java:134) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.DefaultRequestDirector.tryConnect(DefaultRequestDirector.java:605) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.DefaultRequestDirector.execute$original$wN9FcpoY(DefaultRequestDirector.java:440) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.DefaultRequestDirector.execute$original$wN9FcpoY$accessor$3nRtAICB(DefaultRequestDirector.java) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.DefaultRequestDirector$auxiliary$TWCaq941.call(Unknown Source) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.InstMethodsInter.intercept(InstMethodsInter.java:93) ~[skywalking-agent.jar:6.4.0]
        at org.apache.http.impl.client.DefaultRequestDirector.execute(DefaultRequestDirector.java) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.AbstractHttpClient.doExecute$original$X5K2K7NE(AbstractHttpClient.java:835) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.AbstractHttpClient.doExecute$original$X5K2K7NE$accessor$UW83IYMZ(AbstractHttpClient.java) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.AbstractHttpClient$auxiliary$prYdOZrp.call(Unknown Source) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.InstMethodsInter.intercept(InstMethodsInter.java:93) ~[skywalking-agent.jar:6.4.0]
        at org.apache.http.impl.client.AbstractHttpClient.doExecute(AbstractHttpClient.java) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:118) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:56) ~[httpclient-4.5.12.jar!/:4.5.12]
        at com.sun.jersey.client.apache4.ApacheHttpClient4Handler.handle(ApacheHttpClient4Handler.java:173) ~[jersey-apache-client4-1.19.1.jar!/:1.19.1]
        ... 10 common frames omitted

2020-07-23 17:31:58.629 ERROR [test-register,,,] 12962 --- [_192.168.128.207-19] c.n.e.cluster.ReplicationTaskProcessor   : It seems to be a socket read timeout exception, it will retry later. if it continues to happen and some eureka node occupied all the cpu time, you should set property 'eureka.server.peer-node-read-timeout-ms' to a bigger value

com.sun.jersey.api.client.ClientHandlerException: java.net.SocketTimeoutException: Read timed out
        at com.sun.jersey.client.apache4.ApacheHttpClient4Handler.handle(ApacheHttpClient4Handler.java:187) ~[jersey-apache-client4-1.19.1.jar!/:1.19.1]
        at com.netflix.eureka.cluster.DynamicGZIPContentEncodingFilter.handle(DynamicGZIPContentEncodingFilter.java:48) ~[eureka-core-1.9.13.jar!/:1.9.13]
        at com.netflix.discovery.EurekaIdentityHeaderFilter.handle(EurekaIdentityHeaderFilter.java:27) ~[eureka-client-1.9.13.jar!/:1.9.13]
        at com.sun.jersey.api.client.Client.handle(Client.java:652) ~[jersey-client-1.19.1.jar!/:1.19.1]
        at com.sun.jersey.api.client.WebResource.handle(WebResource.java:682) ~[jersey-client-1.19.1.jar!/:1.19.1]
        at com.sun.jersey.api.client.WebResource.access$200(WebResource.java:74) ~[jersey-client-1.19.1.jar!/:1.19.1]
        at com.sun.jersey.api.client.WebResource$Builder.post(WebResource.java:570) ~[jersey-client-1.19.1.jar!/:1.19.1]
        at com.netflix.eureka.transport.JerseyReplicationClient.submitBatchUpdates(JerseyReplicationClient.java:117) ~[eureka-core-1.9.13.jar!/:1.9.13]
        at com.netflix.eureka.cluster.ReplicationTaskProcessor.process(ReplicationTaskProcessor.java:80) ~[eureka-core-1.9.13.jar!/:1.9.13]
        at com.netflix.eureka.util.batcher.TaskExecutors$BatchWorkerRunnable.run(TaskExecutors.java:193) [eureka-core-1.9.13.jar!/:1.9.13]
        at java.lang.Thread.run(Thread.java:748) [na:1.8.0_202]
Caused by: java.net.SocketTimeoutException: Read timed out
        at java.net.SocketInputStream.socketRead0(Native Method) ~[na:1.8.0_202]
        at java.net.SocketInputStream.socketRead(SocketInputStream.java:116) ~[na:1.8.0_202]
        at java.net.SocketInputStream.read(SocketInputStream.java:171) ~[na:1.8.0_202]
        at java.net.SocketInputStream.read(SocketInputStream.java:141) ~[na:1.8.0_202]
        at org.apache.http.impl.io.AbstractSessionInputBuffer.fillBuffer(AbstractSessionInputBuffer.java:161) ~[httpcore-4.4.13.jar!/:4.4.13]
        at org.apache.http.impl.io.SocketInputBuffer.fillBuffer(SocketInputBuffer.java:82) ~[httpcore-4.4.13.jar!/:4.4.13]
        at org.apache.http.impl.io.AbstractSessionInputBuffer.readLine(AbstractSessionInputBuffer.java:276) ~[httpcore-4.4.13.jar!/:4.4.13]
        at org.apache.http.impl.conn.DefaultHttpResponseParser.parseHead(DefaultHttpResponseParser.java:138) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.conn.DefaultHttpResponseParser.parseHead(DefaultHttpResponseParser.java:56) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.io.AbstractMessageParser.parse(AbstractMessageParser.java:259) ~[httpcore-4.4.13.jar!/:4.4.13]
        at org.apache.http.impl.AbstractHttpClientConnection.receiveResponseHeader(AbstractHttpClientConnection.java:294) ~[httpcore-4.4.13.jar!/:4.4.13]
        at org.apache.http.impl.conn.DefaultClientConnection.receiveResponseHeader(DefaultClientConnection.java:257) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.conn.AbstractClientConnAdapter.receiveResponseHeader(AbstractClientConnAdapter.java:230) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.protocol.HttpRequestExecutor.doReceiveResponse(HttpRequestExecutor.java:273) ~[httpcore-4.4.13.jar!/:4.4.13]
        at org.apache.http.protocol.HttpRequestExecutor.execute(HttpRequestExecutor.java:125) ~[httpcore-4.4.13.jar!/:4.4.13]
        at org.apache.http.impl.client.DefaultRequestDirector.tryExecute(DefaultRequestDirector.java:679) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.DefaultRequestDirector.execute$original$wN9FcpoY(DefaultRequestDirector.java:481) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.DefaultRequestDirector.execute$original$wN9FcpoY$accessor$3nRtAICB(DefaultRequestDirector.java) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.DefaultRequestDirector$auxiliary$TWCaq941.call(Unknown Source) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.InstMethodsInter.intercept(InstMethodsInter.java:93) ~[skywalking-agent.jar:6.4.0]
        at org.apache.http.impl.client.DefaultRequestDirector.execute(DefaultRequestDirector.java) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.AbstractHttpClient.doExecute$original$X5K2K7NE(AbstractHttpClient.java:835) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.AbstractHttpClient.doExecute$original$X5K2K7NE$accessor$UW83IYMZ(AbstractHttpClient.java) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.AbstractHttpClient$auxiliary$prYdOZrp.call(Unknown Source) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.InstMethodsInter.intercept(InstMethodsInter.java:93) ~[skywalking-agent.jar:6.4.0]
        at org.apache.http.impl.client.AbstractHttpClient.doExecute(AbstractHttpClient.java) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:118) ~[httpclient-4.5.12.jar!/:4.5.12]
        at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:56) ~[httpclient-4.5.12.jar!/:4.5.12]
        at com.sun.jersey.client.apache4.ApacheHttpClient4Handler.handle(ApacheHttpClient4Handler.java:173) ~[jersey-apache-client4-1.19.1.jar!/:1.19.1]
        ... 10 common frames omitted
```

这里发现，访问207服务超时，说明207服务存在问题，但是查看207服务的启动日志并未发现异常信息。如下：

```
DEBUG 2020-07-23 17:31:07:093 main AgentPackagePath : The beacon class location is jar:file:/home/centos/skywalking-bin/apache-skywalking-apm-bin/agent/skywalking-agent.jar!/org/apache/skywalking/apm/agent/core/boot/AgentPackagePath.class.
INFO 2020-07-23 17:31:07:097 main SnifferConfigInitializer : Config file found in /home/centos/skywalking-bin/apache-skywalking-apm-bin/agent/config/agent.config.
    ______________        ________                __
   /  _/ ____/ __ \      / ____/ /___  __  ______/ /
   / // /   / /_/ /_____/ /   / / __ \/ / / / __  /
 _/ // /___/ ____/_____/ /___/ / /_/ / /_/ / /_/ /
/___/\____/_/          \____/_/\____/\__,_/\__,_/
============ * powered by testsoft * ============
2020-07-23 17:31:26.529  INFO [test-register,,,] 14661 --- [           main] c.a.n.c.c.impl.LocalConfigInfoProcessor  : LOCAL_SNAPSHOT_PATH:/home/centos/nacos/config
2020-07-23 17:31:26.694  INFO [test-register,,,] 14661 --- [           main] c.a.nacos.client.config.impl.Limiter     : limitTime:5.0
2020-07-23 17:31:27.040  WARN [test-register,,,] 14661 --- [           main] c.a.c.n.c.NacosPropertySourceBuilder     : Ignore the empty nacos configuration and get it based on dataId[test-register] & group[DEFAULT_GROUP]
2020-07-23 17:31:27.047  INFO [test-register,,,] 14661 --- [           main] c.a.nacos.client.config.utils.JVMUtil    : isMultiInstance:false
2020-07-23 17:31:27.084  INFO [test-register,,,] 14661 --- [           main] b.c.PropertySourceBootstrapConfiguration : Located property source: [BootstrapPropertySource {name='bootstrapProperties-test-register-peer1.yml,DEFAULT_GROUP'}, BootstrapPropertySource {name='bootstrapProperties-test-register.yml,DEFAULT_GROUP'}, BootstrapPropertySource {name='bootstrapProperties-test-register,DEFAULT_GROUP'}]
2020-07-23 17:31:27.209  INFO [test-register,,,] 14661 --- [           main] c.h.c.s.ServerRegisterApplication        : The following profiles are active: peer1
2020-07-23 17:31:30.192  WARN [test-register,,,] 14661 --- [           main] o.s.boot.actuate.endpoint.EndpointId     : Endpoint ID 'nacos-config' contains invalid characters, please migrate to a valid format.
2020-07-23 17:31:31.499  WARN [test-register,,,] 14661 --- [           main] o.s.boot.actuate.endpoint.EndpointId     : Endpoint ID 'service-registry' contains invalid characters, please migrate to a valid format.
2020-07-23 17:31:32.833  INFO [test-register,,,] 14661 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=0a3a956c-cd1a-35c8-ae9f-743a8cc478b8
2020-07-23 17:31:38.139  INFO [test-register,,,] 14661 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 19012 (http)
2020-07-23 17:31:38.162  INFO [test-register,,,] 14661 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-07-23 17:31:38.162  INFO [test-register,,,] 14661 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.33]
2020-07-23 17:31:38.611  INFO [test-register,,,] 14661 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-07-23 17:31:38.611  INFO [test-register,,,] 14661 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 11320 ms
2020-07-23 17:31:39.731  WARN [test-register,,,] 14661 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2020-07-23 17:31:39.732  INFO [test-register,,,] 14661 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2020-07-23 17:31:39.802  INFO [test-register,,,] 14661 --- [           main] c.netflix.config.DynamicPropertyFactory  : DynamicPropertyFactory is initialized with configuration sources: com.netflix.config.ConcurrentCompositeConfiguration@31a2a9fa
2020-07-23 17:31:42.077  INFO [test-register,,,] 14661 --- [           main] o.s.cloud.commons.util.InetUtils         : Cannot determine local hostname
2020-07-23 17:31:43.217  INFO [test-register,,,] 14661 --- [           main] c.s.j.s.i.a.WebApplicationImpl           : Initiating Jersey application, version 'Jersey: 1.19.1 03/11/2016 02:08 PM'
2020-07-23 17:31:43.513  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON encoding codec LegacyJacksonJson
2020-07-23 17:31:43.513  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON decoding codec LegacyJacksonJson
2020-07-23 17:31:44.523  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML encoding codec XStreamXml
2020-07-23 17:31:44.523  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML decoding codec XStreamXml
2020-07-23 17:31:47.543  INFO [test-register,,,] 14661 --- [           main] o.s.s.web.DefaultSecurityFilterChain     : Creating filter chain: any request, [org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@37314843, org.springframework.security.web.context.SecurityContextPersistenceFilter@70f148dc, org.springframework.security.web.header.HeaderWriterFilter@4d69d288, org.springframework.security.web.authentication.logout.LogoutFilter@4dd2ef54, org.springframework.security.web.authentication.www.BasicAuthenticationFilter@45d4421d, org.springframework.security.web.savedrequest.RequestCacheAwareFilter@77aea, org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@3a3883c4, org.springframework.security.web.authentication.AnonymousAuthenticationFilter@50122012, org.springframework.security.web.session.SessionManagementFilter@787178b1, org.springframework.security.web.access.ExceptionTranslationFilter@4b28a7bf, org.springframework.security.web.access.intercept.FilterSecurityInterceptor@74e497ae]
2020-07-23 17:31:47.589  WARN [test-register,,,] 14661 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2020-07-23 17:31:47.590  INFO [test-register,,,] 14661 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2020-07-23 17:31:50.199  INFO [test-register,,,] 14661 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-07-23 17:31:52.842  INFO [test-register,,,] 14661 --- [           main] o.s.cloud.commons.util.InetUtils         : Cannot determine local hostname
2020-07-23 17:31:53.861  INFO [test-register,,,] 14661 --- [           main] o.s.cloud.commons.util.InetUtils         : Cannot determine local hostname
2020-07-23 17:31:53.995  WARN [test-register,,,] 14661 --- [           main] ockingLoadBalancerClientRibbonWarnLogger : You already have RibbonLoadBalancerClient on your classpath. It will be used by default. As Spring Cloud Ribbon is in maintenance mode. We recommend switching to BlockingLoadBalancerClient instead. In order to use it, set the value of `spring.cloud.loadbalancer.ribbon.enabled` to `false` or remove spring-cloud-starter-netflix-ribbon from your project.
2020-07-23 17:31:54.274  INFO [test-register,,,] 14661 --- [           main] o.s.c.n.eureka.InstanceInfoFactory       : Setting initial instance status as: STARTING
2020-07-23 17:31:54.848  INFO [test-register,,,] 14661 --- [           main] com.netflix.discovery.DiscoveryClient    : Initializing Eureka in region us-east-1
2020-07-23 17:31:55.892  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON encoding codec LegacyJacksonJson
2020-07-23 17:31:55.892  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON decoding codec LegacyJacksonJson
2020-07-23 17:31:55.892  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML encoding codec XStreamXml
2020-07-23 17:31:55.892  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML decoding codec XStreamXml
2020-07-23 17:31:56.242  INFO [test-register,,,] 14661 --- [           main] c.n.d.s.r.aws.ConfigClusterResolver      : Resolving eureka endpoints via configuration
2020-07-23 17:31:56.347  INFO [test-register,,,] 14661 --- [           main] com.netflix.discovery.DiscoveryClient    : Disable delta property : false
2020-07-23 17:31:56.348  INFO [test-register,,,] 14661 --- [           main] com.netflix.discovery.DiscoveryClient    : Single vip registry refresh property : null
2020-07-23 17:31:56.348  INFO [test-register,,,] 14661 --- [           main] com.netflix.discovery.DiscoveryClient    : Force full registry fetch : false
2020-07-23 17:31:56.348  INFO [test-register,,,] 14661 --- [           main] com.netflix.discovery.DiscoveryClient    : Application is null : false
2020-07-23 17:31:56.348  INFO [test-register,,,] 14661 --- [           main] com.netflix.discovery.DiscoveryClient    : Registered Applications size is zero : true
2020-07-23 17:31:56.348  INFO [test-register,,,] 14661 --- [           main] com.netflix.discovery.DiscoveryClient    : Application version is -1: true
2020-07-23 17:31:56.348  INFO [test-register,,,] 14661 --- [           main] com.netflix.discovery.DiscoveryClient    : Getting all instance registry info from the eureka server
2020-07-23 17:31:57.524  INFO [test-register,,,] 14661 --- [           main] com.netflix.discovery.DiscoveryClient    : The response status is 200
2020-07-23 17:31:57.535  INFO [test-register,,,] 14661 --- [           main] com.netflix.discovery.DiscoveryClient    : Starting heartbeat executor: renew interval is: 30
2020-07-23 17:31:57.540  INFO [test-register,,,] 14661 --- [           main] c.n.discovery.InstanceInfoReplicator     : InstanceInfoReplicator onDemand update allowed rate per min is 4
2020-07-23 17:31:57.562  INFO [test-register,,,] 14661 --- [           main] com.netflix.discovery.DiscoveryClient    : Discovery Client initialized at timestamp 1595496717546 with initial instances count: 17
2020-07-23 17:31:57.671  INFO [test-register,,,] 14661 --- [           main] c.n.eureka.DefaultEurekaServerContext    : Initializing ...
2020-07-23 17:31:57.676  INFO [test-register,,,] 14661 --- [           main] c.n.eureka.cluster.PeerEurekaNodes       : Adding new peer nodes [http://test:test2019@192.168.128.208:19013/eureka/, http://test:test2019@192.168.128.206:19011/eureka/]
2020-07-23 17:31:57.686  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON encoding codec LegacyJacksonJson
2020-07-23 17:31:57.687  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON decoding codec LegacyJacksonJson
2020-07-23 17:31:57.687  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML encoding codec XStreamXml
2020-07-23 17:31:57.687  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML decoding codec XStreamXml
2020-07-23 17:31:57.855  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON encoding codec LegacyJacksonJson
2020-07-23 17:31:57.856  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON decoding codec LegacyJacksonJson
2020-07-23 17:31:57.856  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML encoding codec XStreamXml
2020-07-23 17:31:57.856  INFO [test-register,,,] 14661 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML decoding codec XStreamXml
2020-07-23 17:31:57.953  INFO [test-register,,,] 14661 --- [           main] c.n.eureka.cluster.PeerEurekaNodes       : Replica node URL:  http://test:test2019@192.168.128.208:19013/eureka/
2020-07-23 17:31:57.953  INFO [test-register,,,] 14661 --- [           main] c.n.eureka.cluster.PeerEurekaNodes       : Replica node URL:  http://test:test2019@192.168.128.206:19011/eureka/
2020-07-23 17:31:57.975  INFO [test-register,,,] 14661 --- [           main] c.n.e.registry.AbstractInstanceRegistry  : Finished initializing remote region registries. All known remote regions: []
2020-07-23 17:31:57.976  INFO [test-register,,,] 14661 --- [           main] c.n.eureka.DefaultEurekaServerContext    : Initialized
2020-07-23 17:31:58.184  INFO [test-register,,,] 14661 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 18 endpoint(s) beneath base path '/actuator'
2020-07-23 17:31:58.267  INFO [test-register,,,] 14661 --- [           main] o.s.c.n.e.s.EurekaServiceRegistry        : Registering application test-REGISTER with eureka with status UP
2020-07-23 17:31:58.268  INFO [test-register,,,] 14661 --- [           main] com.netflix.discovery.DiscoveryClient    : Saw local status change event StatusChangeEvent [timestamp=1595496718268, current=UP, previous=STARTING]
2020-07-23 17:31:58.303  INFO [test-register,,,] 14661 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_test-REGISTER/test-register@192.168.128.207:19012: registering service...
2020-07-23 17:31:58.318  INFO [test-register,,,] 14661 --- [      Thread-14] o.s.c.n.e.server.EurekaServerBootstrap   : Setting the eureka configuration..
2020-07-23 17:31:58.328  INFO [test-register,,,] 14661 --- [      Thread-14] o.s.c.n.e.server.EurekaServerBootstrap   : Eureka data center value eureka.datacenter is not set, defaulting to default
2020-07-23 17:31:58.329  INFO [test-register,,,] 14661 --- [      Thread-14] o.s.c.n.e.server.EurekaServerBootstrap   : Eureka environment value eureka.environment is not set, defaulting to test
2020-07-23 17:31:58.410  INFO [test-register,,,] 14661 --- [      Thread-14] o.s.c.n.e.server.EurekaServerBootstrap   : isAws returned false
2020-07-23 17:31:58.410  INFO [test-register,,,] 14661 --- [      Thread-14] o.s.c.n.e.server.EurekaServerBootstrap   : Initialized server context
2020-07-23 17:31:58.465  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-FILESTORE/test-filestore@192.168.128.199:20060 with status UP (replication=true)
2020-07-23 17:31:58.466  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-GATEWAY/test-gateway@192.168.128.209:19020 with status UP (replication=true)
2020-07-23 17:31:58.466  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-BASE-PSL/test-base-psl@192.168.128.199:24020 with status UP (replication=true)
2020-07-23 17:31:58.467  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-APP-INF/test-app-inf@192.168.128.199:20140 with status UP (replication=true)
2020-07-23 17:31:58.468  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-REGISTER/test-register@192.168.128.206:19011 with status UP (replication=true)
2020-07-23 17:31:58.468  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-REGISTER/test-register@192.168.128.208:19013 with status UP (replication=true)
2020-07-23 17:31:58.469  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-BASE-RDP/test-base-rdp@192.168.128.199:20151 with status UP (replication=true)
2020-07-23 17:31:58.470  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-ADMIN-SERVER/test-admin-server@192.168.128.199:19760 with status UP (replication=true)
2020-07-23 17:31:58.475  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-UAA/test-uaa@192.168.128.199:19060 with status UP (replication=true)
2020-07-23 17:31:58.476  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-FORM/test-form@192.168.128.199:20040 with status UP (replication=true)
2020-07-23 17:31:58.476  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-ID-GENERATOR/test-id-generator@192.168.128.199:8080 with status UP (replication=true)
2020-07-23 17:31:58.477  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-PSL-EQUIPMENT-MANAGEMENT/test-psl-equipment-management@192.168.128.199:24010 with status UP (replication=true)
2020-07-23 17:31:58.478  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-UMC/test-umc@192.168.128.199:19070 with status UP (replication=true)
2020-07-23 17:31:58.478  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-DICT/test-dict@10.0.93.99:19090 with status UP (replication=true)
2020-07-23 17:31:58.479  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-WORKFLOW/test-workflow@192.168.128.199:20050 with status UP (replication=true)
2020-07-23 17:31:58.480  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-BASE/test-base@192.168.128.199:20130 with status UP (replication=true)
2020-07-23 17:31:58.480  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-VISUALMODEL/test-visualmodel@10.0.93.99:20020 with status UP (replication=true)
2020-07-23 17:31:58.480  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.r.PeerAwareInstanceRegistryImpl    : Got 17 instances from neighboring DS node
2020-07-23 17:31:58.480  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.r.PeerAwareInstanceRegistryImpl    : Renew threshold is: 28
2020-07-23 17:31:58.481  INFO [test-register,,,] 14661 --- [      Thread-14] c.n.e.r.PeerAwareInstanceRegistryImpl    : Changing status to UP
2020-07-23 17:31:58.499  INFO [test-register,,,] 14661 --- [      Thread-14] e.s.EurekaServerInitializerConfiguration : Started Eureka Server
2020-07-23 17:31:58.511  INFO [test-register,,,] 14661 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_test-REGISTER/test-register@192.168.128.207:19012 - registration status: 204
2020-07-23 17:31:58.519  INFO [test-register,,,] 14661 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 19012 (http) with context path ''
2020-07-23 17:31:58.521  INFO [test-register,,,] 14661 --- [           main] .s.c.n.e.s.EurekaAutoServiceRegistration : Updating port to 19012
2020-07-23 17:31:58.699  INFO [test-register,,,] 14661 --- [io-19012-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2020-07-23 17:31:58.700  INFO [test-register,,,] 14661 --- [io-19012-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2020-07-23 17:31:58.731  INFO [test-register,,,] 14661 --- [io-19012-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 31 ms
2020-07-23 17:31:59.308  INFO [test-register,9481285215d134b2,9481285215d134b2,false] 14661 --- [io-19012-exec-2] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-REGISTER/test-register@192.168.128.207:19012 with status UP (replication=true)
2020-07-23 17:31:59.522  INFO [test-register,,,] 14661 --- [           main] o.s.cloud.commons.util.InetUtils         : Cannot determine local hostname
2020-07-23 17:31:59.526  INFO [test-register,,,] 14661 --- [           main] c.h.c.s.ServerRegisterApplication        : Started ServerRegisterApplication in 43.119 seconds (JVM running for 54.87)
2020-07-23 17:31:59.543  INFO [test-register,,,] 14661 --- [           main] c.a.n.client.config.impl.ClientWorker    : [fixed-192.168.128.206_18848-858b37b1-35be-4564-a24e-dc2c322d5784] [subscribe] test-register.yml+DEFAULT_GROUP+858b37b1-35be-4564-a24e-dc2c322d5784
2020-07-23 17:31:59.547  INFO [test-register,,,] 14661 --- [           main] c.a.nacos.client.config.impl.CacheData   : [fixed-192.168.128.206_18848-858b37b1-35be-4564-a24e-dc2c322d5784] [add-listener] ok, tenant=858b37b1-35be-4564-a24e-dc2c322d5784, dataId=test-register.yml, group=DEFAULT_GROUP, cnt=1
2020-07-23 17:31:59.547  INFO [test-register,,,] 14661 --- [           main] c.a.n.client.config.impl.ClientWorker    : [fixed-192.168.128.206_18848-858b37b1-35be-4564-a24e-dc2c322d5784] [subscribe] test-register+DEFAULT_GROUP+858b37b1-35be-4564-a24e-dc2c322d5784
2020-07-23 17:31:59.547  INFO [test-register,,,] 14661 --- [           main] c.a.nacos.client.config.impl.CacheData   : [fixed-192.168.128.206_18848-858b37b1-35be-4564-a24e-dc2c322d5784] [add-listener] ok, tenant=858b37b1-35be-4564-a24e-dc2c322d5784, dataId=test-register, group=DEFAULT_GROUP, cnt=1
2020-07-23 17:31:59.548  INFO [test-register,,,] 14661 --- [           main] c.a.n.client.config.impl.ClientWorker    : [fixed-192.168.128.206_18848-858b37b1-35be-4564-a24e-dc2c322d5784] [subscribe] test-register-peer1.yml+DEFAULT_GROUP+858b37b1-35be-4564-a24e-dc2c322d5784
2020-07-23 17:31:59.548  INFO [test-register,,,] 14661 --- [           main] c.a.nacos.client.config.impl.CacheData   : [fixed-192.168.128.206_18848-858b37b1-35be-4564-a24e-dc2c322d5784] [add-listener] ok, tenant=858b37b1-35be-4564-a24e-dc2c322d5784, dataId=test-register-peer1.yml, group=DEFAULT_GROUP, cnt=1
2020-07-23 17:31:59.563  INFO [test-register,,,] 14661 --- [4e-dc2c322d5784] c.a.n.client.config.impl.ClientWorker    : get changedGroupKeys:[]
2020-07-23 17:31:59.903  INFO [test-register,78c19310aaaa187c,78c19310aaaa187c,false] 14661 --- [io-19012-exec-4] c.n.e.registry.AbstractInstanceRegistry  : Registered instance test-REGISTER/test-register@192.168.128.207:19012 with status UP (replication=true)
2020-07-23 17:32:29.268  INFO [test-register,,,] 14661 --- [4e-dc2c322d5784] c.a.n.client.config.impl.ClientWorker    : get changedGroupKeys:[]
2020-07-23 17:32:58.483  INFO [test-register,,,] 14661 --- [a-EvictionTimer] c.n.e.registry.AbstractInstanceRegistry  : Running the evict task 

```

## 解决方式

1. eureka server之间相互成为peer node，如果有一台eureka server挂了，则eureka server之间的replication都会受影响，人工接入更改eureka server的serverUrl信息，则可以主动剔除掉挂掉的peerNode，未能剔除，则会报错。报上述错误！

2. 通过重启存在错误的eureka服务，解决该问题。

3. 关于Eureka配置文件中的配置信息，如下：

```yaml
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true

```

- register-with-eureka表示：是否注册自身到eureka服务器。
- fetch-registry表示：是否从eureka上获取注册信息

如果存在上述的配置项，注释掉这两个选项也是可以解决这个问题的。