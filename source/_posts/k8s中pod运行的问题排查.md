---
title: k8s中pod运行的问题排查
top: false
cover: false
toc: true
mathjax: true
date: 2020-07-08 10:43:45
password:
summary:
tags:
categories:
---

## 关于pom文件版本号对于Dockerfile的影响

我们之前的设定，对于pom文件中的版本感知不明显，导致我们在Dockerfile里面写死了jar包的版本号。

当pom文件中的版本号变更时，由于Dockerfile中未能及时变更，导致出现了构建运行镜像失败的问题。

临时解决方式：

在构建时添加sed命令，用来修改Dockerfile中的版本不正确的问题。如下图：

![](./imgs/修改版本.png)

## 关于k8s中服务反复重启导致无法对外访问的问题排查

以test-form-design-rdp服务为例子，日志如下：

```
DEBUG 2020-06-10 13:45:53:361 main AgentPackagePath : The beacon class location is jar:file:/app/skywalking-agent/skywalking-agent.jar!/org/apache/skywalking/apm/agent/core/boot/AgentPackagePath.class.
INFO 2020-06-10 13:45:53:366 main SnifferConfigInitializer : Config file found in /app/skywalking-agent/config/agent.config.

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.2.6.RELEASE)

2020-06-10 13:46:08.622  INFO [test-form-rdp,,,] 6 --- [           main] c.a.n.c.c.impl.LocalConfigInfoProcessor  : LOCAL_SNAPSHOT_PATH:/root/nacos/config
2020-06-10 13:46:08.874  INFO [test-form-rdp,,,] 6 --- [           main] c.a.nacos.client.config.impl.Limiter     : limitTime:5.0
2020-06-10 13:46:09.042  WARN [test-form-rdp,,,] 6 --- [           main] c.a.c.n.c.NacosPropertySourceBuilder     : Ignore the empty nacos configuration and get it based on dataId[test-form-rdp] & group[DEFAULT_GROUP]
2020-06-10 13:46:09.052  INFO [test-form-rdp,,,] 6 --- [           main] c.a.nacos.client.config.utils.JVMUtil    : isMultiInstance:false
2020-06-10 13:46:09.125  INFO [test-form-rdp,,,] 6 --- [           main] b.c.PropertySourceBootstrapConfiguration : Located property source: [BootstrapPropertySource {name='bootstrapProperties-test-form-rdp-standalone.yml,DEFAULT_GROUP'}, BootstrapPropertySource {name='bootstrapProperties-test-form-rdp.yml,DEFAULT_GROUP'}, BootstrapPropertySource {name='bootstrapProperties-test-form-rdp,DEFAULT_GROUP'}]
2020-06-10 13:46:09.235  INFO [test-form-rdp,,,] 6 --- [           main] c.h.i.form.IcpCloudFormApplication       : The following profiles are active: standalone
2020-06-10 13:46:16.690  WARN [test-form-rdp,,,] 6 --- [           main] o.s.boot.actuate.endpoint.EndpointId     : Endpoint ID 'nacos-config' contains invalid characters, please migrate to a valid format.
2020-06-10 13:46:18.605  INFO [test-form-rdp,,,] 6 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Multiple Spring Data modules found, entering strict repository configuration mode!
2020-06-10 13:46:18.612  INFO [test-form-rdp,,,] 6 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data MongoDB repositories in DEFAULT mode.
2020-06-10 13:46:18.735  INFO [test-form-rdp,,,] 6 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 100ms. Found 0 MongoDB repository interfaces.
2020-06-10 13:46:18.812  INFO [test-form-rdp,,,] 6 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Multiple Spring Data modules found, entering strict repository configuration mode!
2020-06-10 13:46:18.819  INFO [test-form-rdp,,,] 6 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data Redis repositories in DEFAULT mode.
2020-06-10 13:46:18.897  INFO [test-form-rdp,,,] 6 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 6ms. Found 0 Redis repository interfaces.
2020-06-10 13:46:19.672  WARN [test-form-rdp,,,] 6 --- [           main] o.s.boot.actuate.endpoint.EndpointId     : Endpoint ID 'service-registry' contains invalid characters, please migrate to a valid format.
2020-06-10 13:46:21.712  INFO [test-form-rdp,,,] 6 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=738057ce-da16-3790-9a18-b67c26d5c038
2020-06-10 13:46:26.888  INFO [test-form-rdp,,,] 6 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.security.config.annotation.configuration.ObjectPostProcessorConfiguration' of type [org.springframework.security.config.annotation.configuration.ObjectPostProcessorConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2020-06-10 13:46:26.906  INFO [test-form-rdp,,,] 6 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'objectPostProcessor' of type [org.springframework.security.config.annotation.configuration.AutowireBeanFactoryObjectPostProcessor] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2020-06-10 13:46:26.915  INFO [test-form-rdp,,,] 6 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler@20a9f5fb' of type [org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2020-06-10 13:46:26.917  INFO [test-form-rdp,,,] 6 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.security.config.annotation.method.configuration.GlobalMethodSecurityConfiguration' of type [org.springframework.security.config.annotation.method.configuration.GlobalMethodSecurityConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2020-06-10 13:46:26.969  INFO [test-form-rdp,,,] 6 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.security.config.annotation.method.configuration.Jsr250MetadataSourceConfiguration' of type [org.springframework.security.config.annotation.method.configuration.Jsr250MetadataSourceConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2020-06-10 13:46:26.974  INFO [test-form-rdp,,,] 6 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'jsr250MethodSecurityMetadataSource' of type [org.springframework.security.access.annotation.Jsr250MethodSecurityMetadataSource] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2020-06-10 13:46:26.980  INFO [test-form-rdp,,,] 6 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'methodSecurityMetadataSource' of type [org.springframework.security.access.method.DelegatingMethodSecurityMetadataSource] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2020-06-10 13:46:27.028  INFO [test-form-rdp,,,] 6 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'redisCacheConfig' of type [com.testsoft.icpcloud.common.cache.config.RedisCacheConfig$$EnhancerBySpringCGLIB$$c422b7ae] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2020-06-10 13:46:27.390  INFO [test-form-rdp,,,] 6 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.cloud.sleuth.instrument.web.client.TraceWebClientAutoConfiguration$TraceOAuthConfiguration' of type [org.springframework.cloud.sleuth.instrument.web.client.TraceWebClientAutoConfiguration$TraceOAuthConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2020-06-10 13:46:27.475  INFO [test-form-rdp,,,] 6 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'spring.sleuth.redis-org.springframework.cloud.sleuth.instrument.redis.TraceRedisProperties' of type [org.springframework.cloud.sleuth.instrument.redis.TraceRedisProperties] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2020-06-10 13:46:28.918  INFO [test-form-rdp,,,] 6 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 20041 (testtp)
2020-06-10 13:46:28.947  INFO [test-form-rdp,,,] 6 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-06-10 13:46:28.948  INFO [test-form-rdp,,,] 6 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.33]
2020-06-10 13:46:29.219  INFO [test-form-rdp,,,] 6 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-06-10 13:46:29.220  INFO [test-form-rdp,,,] 6 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 19798 ms
2020-06-10 13:46:30.020  WARN [test-form-rdp,,,] 6 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2020-06-10 13:46:30.020  INFO [test-form-rdp,,,] 6 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2020-06-10 13:46:30.085  INFO [test-form-rdp,,,] 6 --- [           main] c.netflix.config.DynamicPropertyFactory  : DynamicPropertyFactory is initialized with configuration sources: com.netflix.config.ConcurrentCompositeConfiguration@4dd4965a
2020-06-10 13:46:30.566  INFO [test-form-rdp,,,] 6 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
Loading class `com.mysql.jdbc.Driver'. This is deprecated. The new driver class is `com.mysql.cj.jdbc.Driver'. The driver is automatically registered via the SPI and manual loading of the driver class is generally unnecessary.
2020-06-10 13:46:32.724  INFO [test-form-rdp,,,] 6 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2020-06-10 13:46:34.160  INFO [test-form-rdp,,,] 6 --- [           main] org.mongodb.driver.cluster               : Cluster created with settings {hosts=[mongo-outer.test-basic-mongo-outer:27017], mode=SINGLE, requiredClusterType=UNKNOWN, serverSelectionTimeout='30000 ms', maxWaitQueueSize=500}
2020-06-10 13:46:34.433  INFO [test-form-rdp,,,] 6 --- [           main] org.mongodb.driver.cluster               : Cluster description not yet available. Waiting for 30000 ms before timing out
2020-06-10 13:46:34.603  INFO [test-form-rdp,,,] 6 --- [ngo-outer:27017] org.mongodb.driver.connection            : Opened connection [connectionId{localValue:1, serverValue:391}] to mongo-outer.test-basic-mongo-outer:27017
2020-06-10 13:46:34.640  INFO [test-form-rdp,,,] 6 --- [ngo-outer:27017] org.mongodb.driver.cluster               : Monitor thread successfully connected to server with description ServerDescription{address=mongo-outer.test-basic-mongo-outer:27017, type=STANDALONE, state=CONNECTED, ok=true, version=ServerVersion{versionList=[4, 2, 7]}, minWireVersion=0, maxWireVersion=8, maxDocumentSize=16777216, logicalSessionTimeoutMinutes=30, roundTripTimeNanos=19311961}
2020-06-10 13:46:43.815  INFO [test-form-rdp,,,] 6 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 2 endpoint(s) beneath base path '/actuator'
2020-06-10 13:46:46.083  WARN [test-form-rdp,,,] 6 --- [           main] c.n.c.sources.URLConfigurationSource     : No URLs will be polled as dynamic configuration sources.
2020-06-10 13:46:46.083  INFO [test-form-rdp,,,] 6 --- [           main] c.n.c.sources.URLConfigurationSource     : To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2020-06-10 13:46:48.433  INFO [test-form-rdp,,,] 6 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
 _ _   |_  _ _|_. ___ _ |    _
| | |\/|_)(_| | |_\  |_)||_|_\
     /               |
                        3.3.1
2020-06-10 13:46:50.785  INFO [test-form-rdp,,,] 6 --- [           main] pertySourcedRequestMappingHandlerMapping : Mapped URL path [/v2/api-docs] onto method [springfox.documentation.swagger2.web.Swagger2Controller#getDocumentation(String, HttpServletRequest)]
2020-06-10 13:46:52.471  INFO [test-form-rdp,,,] 6 --- [           main] .s.s.UserDetailsServiceAutoConfiguration :

Using generated security password: 6595127a-b2c1-4947-9f73-c17b7a2c7002

2020-06-10 13:46:53.214  INFO [test-form-rdp,,,] 6 --- [           main] .r.c.IcpSecurityResourceServerConfigurer : IcpResourceServerAutoConfiguration is working!!!
2020-06-10 13:46:53.511  INFO [test-form-rdp,,,] 6 --- [           main] o.s.s.web.DefaultSecurityFilterChain     : Creating filter chain: any request, [org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@2ce80c85, org.springframework.security.web.context.SecurityContextPersistenceFilter@2ea3f905, org.springframework.security.web.header.HeaderWriterFilter@9fe1e14, org.springframework.security.web.authentication.logout.LogoutFilter@1e8f20bf, org.springframework.security.oauth2.provider.authentication.OAuth2AuthenticationProcessingFilter@1c027237, org.springframework.security.web.savedrequest.RequestCacheAwareFilter@194feb27, org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@3df920, org.springframework.security.web.authentication.AnonymousAuthenticationFilter@25dd9eb6, org.springframework.security.web.session.SessionManagementFilter@41d34a11, org.springframework.security.web.access.ExceptionTranslationFilter@327fd5c9, org.springframework.security.web.access.intercept.FilterSecurityInterceptor@9a00385]
2020-06-10 13:46:54.159  WARN [test-form-rdp,,,] 6 --- [           main] ockingLoadBalancerClientRibbonWarnLogger : You already have RibbonLoadBalancerClient on your classpath. It will be used by default. As Spring Cloud Ribbon is in maintenance mode. We recommend switching to BlockingLoadBalancerClient instead. In order to use it, set the value of `spring.cloud.loadbalancer.ribbon.enabled` to `false` or remove spring-cloud-starter-netflix-ribbon from your project.
2020-06-10 13:46:54.918  INFO [test-form-rdp,,,] 6 --- [           main] o.s.c.n.eureka.InstanceInfoFactory       : Setting initial instance status as: STARTING
2020-06-10 13:46:55.075  INFO [test-form-rdp,,,] 6 --- [           main] com.netflix.discovery.DiscoveryClient    : Initializing Eureka in region us-east-1
2020-06-10 13:46:55.357  INFO [test-form-rdp,,,] 6 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON encoding codec LegacyJacksonJson
2020-06-10 13:46:55.357  INFO [test-form-rdp,,,] 6 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON decoding codec LegacyJacksonJson
2020-06-10 13:46:56.068  INFO [test-form-rdp,,,] 6 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML encoding codec XStreamXml
2020-06-10 13:46:56.068  INFO [test-form-rdp,,,] 6 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML decoding codec XStreamXml
2020-06-10 13:46:56.968  INFO [test-form-rdp,,,] 6 --- [           main] c.n.d.s.r.aws.ConfigClusterResolver      : Resolving eureka endpoints via configuration
2020-06-10 13:46:57.134  INFO [test-form-rdp,,,] 6 --- [           main] com.netflix.discovery.DiscoveryClient    : Disable delta property : false
2020-06-10 13:46:57.135  INFO [test-form-rdp,,,] 6 --- [           main] com.netflix.discovery.DiscoveryClient    : Single vip registry refresh property : null
2020-06-10 13:46:57.135  INFO [test-form-rdp,,,] 6 --- [           main] com.netflix.discovery.DiscoveryClient    : Force full registry fetch : false
2020-06-10 13:46:57.135  INFO [test-form-rdp,,,] 6 --- [           main] com.netflix.discovery.DiscoveryClient    : Application is null : false
2020-06-10 13:46:57.135  INFO [test-form-rdp,,,] 6 --- [           main] com.netflix.discovery.DiscoveryClient    : Registered Applications size is zero : true
2020-06-10 13:46:57.135  INFO [test-form-rdp,,,] 6 --- [           main] com.netflix.discovery.DiscoveryClient    : Application version is -1: true
2020-06-10 13:46:57.135  INFO [test-form-rdp,,,] 6 --- [           main] com.netflix.discovery.DiscoveryClient    : Getting all instance registry info from the eureka server
2020-06-10 13:46:58.190  INFO [test-form-rdp,,,] 6 --- [           main] com.netflix.discovery.DiscoveryClient    : The response status is 200
2020-06-10 13:46:58.197  INFO [test-form-rdp,,,] 6 --- [           main] com.netflix.discovery.DiscoveryClient    : Starting heartbeat executor: renew interval is: 30
2020-06-10 13:46:58.201  INFO [test-form-rdp,,,] 6 --- [           main] c.n.discovery.InstanceInfoReplicator     : InstanceInfoReplicator onDemand update allowed rate per min is 4
2020-06-10 13:46:58.215  INFO [test-form-rdp,,,] 6 --- [           main] com.netflix.discovery.DiscoveryClient    : Discovery Client initialized at timestamp 1591768018207 with initial instances count: 17
2020-06-10 13:46:58.239  INFO [test-form-rdp,,,] 6 --- [           main] o.s.c.n.e.s.EurekaServiceRegistry        : Registering application test-FORM-RDP with eureka with status UP
2020-06-10 13:46:58.242  INFO [test-form-rdp,,,] 6 --- [           main] com.netflix.discovery.DiscoveryClient    : Saw local status change event StatusChangeEvent [timestamp=1591768018242, current=UP, previous=STARTING]
2020-06-10 13:46:58.247  INFO [test-form-rdp,,,] 6 --- [           main] d.s.w.p.DocumentationPluginsBootstrapper : Context refreshed
2020-06-10 13:46:58.284  INFO [test-form-rdp,,,] 6 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_test-FORM-RDP/test-form-rdp@10.42.3.29:20041: registering service...
2020-06-10 13:46:58.493  INFO [test-form-rdp,,,] 6 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_test-FORM-RDP/test-form-rdp@10.42.3.29:20041 - registration status: 204
2020-06-10 13:46:58.593  INFO [test-form-rdp,,,] 6 --- [           main] d.s.w.p.DocumentationPluginsBootstrapper : Found 1 custom documentation plugin(s)
2020-06-10 13:46:58.794  INFO [test-form-rdp,,,] 6 --- [           main] s.d.s.w.s.ApiListingReferenceScanner     : Scanning for api listing references
2020-06-10 13:46:59.984  INFO [test-form-rdp,,,] 6 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 20041 (testtp) with context path ''
2020-06-10 13:46:59.988  INFO [test-form-rdp,,,] 6 --- [           main] .s.c.n.e.s.EurekaAutoServiceRegistration : Updating port to 20041
2020-06-10 13:46:59.996  INFO [test-form-rdp,,,] 6 --- [           main] c.h.i.form.IcpCloudFormApplication       : Started IcpCloudFormApplication in 58.544 seconds (JVM running for 66.916)
2020-06-10 13:47:00.017  INFO [test-form-rdp,,,] 6 --- [           main] c.a.n.client.config.impl.ClientWorker    : [fixed-10.0.66.211_18848-d79ba2e9-da64-40e8-9dcd-92cc24bcaf4e] [subscribe] test-form-rdp.yml+DEFAULT_GROUP+d79ba2e9-da64-40e8-9dcd-92cc24bcaf4e
2020-06-10 13:47:00.021  INFO [test-form-rdp,,,] 6 --- [           main] c.a.nacos.client.config.impl.CacheData   : [fixed-10.0.66.211_18848-d79ba2e9-da64-40e8-9dcd-92cc24bcaf4e] [add-listener] ok, tenant=d79ba2e9-da64-40e8-9dcd-92cc24bcaf4e, dataId=test-form-rdp.yml, group=DEFAULT_GROUP, cnt=1
2020-06-10 13:47:00.021  INFO [test-form-rdp,,,] 6 --- [           main] c.a.n.client.config.impl.ClientWorker    : [fixed-10.0.66.211_18848-d79ba2e9-da64-40e8-9dcd-92cc24bcaf4e] [subscribe] test-form-rdp+DEFAULT_GROUP+d79ba2e9-da64-40e8-9dcd-92cc24bcaf4e
2020-06-10 13:47:00.021  INFO [test-form-rdp,,,] 6 --- [           main] c.a.nacos.client.config.impl.CacheData   : [fixed-10.0.66.211_18848-d79ba2e9-da64-40e8-9dcd-92cc24bcaf4e] [add-listener] ok, tenant=d79ba2e9-da64-40e8-9dcd-92cc24bcaf4e, dataId=test-form-rdp, group=DEFAULT_GROUP, cnt=1
2020-06-10 13:47:00.022  INFO [test-form-rdp,,,] 6 --- [           main] c.a.n.client.config.impl.ClientWorker    : [fixed-10.0.66.211_18848-d79ba2e9-da64-40e8-9dcd-92cc24bcaf4e] [subscribe] test-form-rdp-standalone.yml+DEFAULT_GROUP+d79ba2e9-da64-40e8-9dcd-92cc24bcaf4e
2020-06-10 13:47:00.022  INFO [test-form-rdp,,,] 6 --- [           main] c.a.nacos.client.config.impl.CacheData   : [fixed-10.0.66.211_18848-d79ba2e9-da64-40e8-9dcd-92cc24bcaf4e] [add-listener] ok, tenant=d79ba2e9-da64-40e8-9dcd-92cc24bcaf4e, dataId=test-form-rdp-standalone.yml, group=DEFAULT_GROUP, cnt=1
2020-06-10 13:47:00.032  INFO [test-form-rdp,,,] 6 --- [cd-92cc24bcaf4e] c.a.n.client.config.impl.ClientWorker    : get changedGroupKeys:[]
2020-06-10 13:47:29.743  INFO [test-form-rdp,,,] 6 --- [cd-92cc24bcaf4e] c.a.n.client.config.impl.ClientWorker    : get changedGroupKeys:[]
2020-06-10 13:47:59.246  INFO [test-form-rdp,,,] 6 --- [cd-92cc24bcaf4e] c.a.n.client.config.impl.ClientWorker    : get changedGroupKeys:[]
2020-06-10 13:48:28.750  INFO [test-form-rdp,,,] 6 --- [cd-92cc24bcaf4e] c.a.n.client.config.impl.ClientWorker    : get changedGroupKeys:[]
2020-06-10 13:48:58.254  INFO [test-form-rdp,,,] 6 --- [cd-92cc24bcaf4e] c.a.n.client.config.impl.ClientWorker    : get changedGroupKeys:[]
2020-06-10 13:49:15.018  INFO [test-form-rdp,,,] 6 --- [extShutdownHook] o.s.c.n.e.s.EurekaServiceRegistry        : Unregistering application test-FORM-RDP with eureka with status DOWN
2020-06-10 13:49:15.019  WARN [test-form-rdp,,,] 6 --- [extShutdownHook] com.netflix.discovery.DiscoveryClient    : Saw local status change event StatusChangeEvent [timestamp=1591768155018, current=DOWN, previous=UP]
2020-06-10 13:49:15.020  INFO [test-form-rdp,,,] 6 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_test-FORM-RDP/test-form-rdp@10.42.3.29:20041: registering service...
2020-06-10 13:49:15.027  INFO [test-form-rdp,,,] 6 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_test-FORM-RDP/test-form-rdp@10.42.3.29:20041 - registration status: 204
2020-06-10 13:49:15.055  INFO [test-form-rdp,,,] 6 --- [extShutdownHook] o.s.s.concurrent.ThreadPoolTaskExecutor  : Shutting down ExecutorService 'applicationTaskExecutor'
2020-06-10 13:49:15.198  INFO [test-form-rdp,,,] 6 --- [extShutdownHook] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closing ...
2020-06-10 13:49:15.232  INFO [test-form-rdp,,,] 6 --- [extShutdownHook] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closed
2020-06-10 13:49:15.239  INFO [test-form-rdp,,,] 6 --- [extShutdownHook] com.netflix.discovery.DiscoveryClient    : Shutting down DiscoveryClient ...
2020-06-10 13:49:18.241  INFO [test-form-rdp,,,] 6 --- [extShutdownHook] com.netflix.discovery.DiscoveryClient    : Unregistering ...
2020-06-10 13:49:18.253  INFO [test-form-rdp,,,] 6 --- [extShutdownHook] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_test-FORM-RDP/test-form-rdp@10.42.3.29:20041 - deregister  status: 200
2020-06-10 13:49:18.274  INFO [test-form-rdp,,,] 6 --- [extShutdownHook] com.netflix.discovery.DiscoveryClient    : Completed shut down of DiscoveryClient

```

启动过程中，未发生任何错误，在启动完成后却出现了服务自动下线的情况，体现在Eureka中就是服务明明上线了，过一分钟左右就自动下线了。

目前怀疑是k8s的健康检查干掉了正在运行的服务，这时候需要排查配置文件和k8s部署文件的区别。

配置文件如下：

```yaml
spring:
  application:
    name: test-form-rdp
  data:
    mongodb:
      uri: mongodb://form-design:123456@mongo-outer.test-basic-mongo-outer:27017/test-form-design
      # uri: mongodb://127.0.0.1:27017/test-form


  datasource:
    username: root
    password: testMySQL789
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://mysql-outer.test-basic-mysql-outer:3306/FORMDESIGN?allowMultiQueries=true&serverTimezone=Asia/Shanghai&characterEncoding=utf8&useUnicode=true&useSSL=false
    druid:
      initial-size: 8
      min-idle: 8
      max-active: 50
      max-wait: 60000
      time-between-eviction-runs-millis: 60000
      min-evictable-idle-time-millis: 300000
      validationQuery: select 'x'
      test-while-idle: true  
      test-on-borrow: false  
      test-on-return: false  
      multi-statement-allow: true
      filters: config,stat  
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
        login-password: testMySQL789

      filter:
        wall:
          enabled: true 
          db-type: mysql
          config:
            alter-table-allow: true
            truncate-allow: false
            drop-table-allow: true
            multi-statement-allow: true
            none-base-statement-allow: false 
            update-where-none-check: true 
            select-into-outfile-allow: false
            metadata-allow: true 
          log-violation: true 
          throw-exception: true 
  cache:
    caffeine:
      spec: initialCapacity=50,maximumSize=500,expireAfterAccess=5s,expireAfterWrite=10s,refreshAfterWrite=5s


  redis: 
    host: redis-outer.test-basic-redis-outer
    port: 6379
    password: test@redis160
    timeout: 60000
    database: 2
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

eureka:
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}@${spring.cloud.client.ip-address}:${server.port}
    ipAddress: ${spring.cloud.client.ip-address}
  client:
    service-url:
      defaultZone: testtp://test:test2019@eureka-0.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/,testtp://test:test2019@eureka-1.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/,testtp://test:test2019@eureka-2.eureka.test-basic-eureka.svc.cluster.local:19011/eureka/
      #defaultZone: testtp://test:test2019@127.0.0.1:19011/eureka/
swagger:
  enable: true


auth-server: testtp://test-rdp-uaa

security:
  oauth2:
    client:
      client-id: test-form-rdp 
      client-secret: test-form-rdp@testdev 
      access-token-uri: ${auth-server}/oauth/token 
      user-authorization-uri: ${auth-server}/oauth/authorize 
    resource:
      token-info-uri: ${auth-server}/oauth/check_token 


server:
  port: 20041

# mybatis-plus:
#   mapper-locations: classpath:mybatis/mapper/*Mapper.xml
#   configuration:
#     cache-enabled: true
#     call-setters-on-nulls: true
#     log-impl: org.apache.ibatis.logging.stdout.StdOutImpl


icpcloud:
  security:
    resource:
      enabled: true
      permitAll:
        filterPath: /emp/info/**/security

```

这里端口号配置的是20041，那么这时在看该服务的k8s部署文档：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: test-form-design-rdp
  namespace: test-all-service
  labels:
    app: test-form-design-rdp
spec:
  ports:
    - port: 20040
      name: tcp
      targetPort: 20040
  selector:
    app: test-form-design-rdp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-form-design-rdp
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
      app: test-form-design-rdp
  template:
    metadata:
      labels:
        app: test-form-design-rdp
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
                        - test-form-design
                        - test-workflow
                        - test-form-design-rdp
              weigtest: 1
      containers:
        - name: form-design-rdp
          image: 10.0.88.159:5000/formdesign-rdp-develop:$BUILD_NUMBER
          imagePullPolicy: Always
          lifecycle:
            preStop:
              testtpGet:
                port: 20040
                path: /spring/shutdown
          livenessProbe:
            testtpGet:
              path: /actuator/health
              port: 20040
            initialDelaySeconds: 180
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          readinessProbe:
            testtpGet:
              path: /actuator/health
              port: 20040
            initialDelaySeconds: 180
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          resources:
            requests:
              memory: 350Mi
            limits:
              memory: 1.5Gi
          ports:
            - containerPort: 20040
          env:
            - name: IP_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NACOS_IP
              value: 10.0.66.211:18848
            - name: NACOS_NAMESPACE
              value: d79ba2e9-da64-40e8-9dcd-92cc24bcaf4e
            - name: SKYWALKING_NAMESPACE
              value: test-k8s-develop
            - name: SKYWALKING_TARGET_SERVICE_NAME
              value: test-form-design-rdp
            - name: SKYWALKING_IP_PORT
              value: 10.0.88.163:11800
            - name: CHANNEL
              value: standalone

```

此处端口号为20040，这时候两个位置并不统一，导致了访问出现问题，服务上线后又迅速被干掉了。

解决方式：固定端口号信息，以配置文件为准，进行修改即可！


## 关于服务器内核自动升级问题的解决

由于部分机器进行了

```
sudo yum update

```
操作，导致内核信息进行了升级，导致有一些机器在启动的时候出现了黑屏和无法启动的情况。

由于在启动时，默认选择了新升级的内核进行了启动，导致报错。如下：

![](新内核启动报错.png)

目前可用的稳定版内核为：3.10.0-1062.12.1.el7.x86_64，或者低于该版本的内核信息。

以10.0.66.221机器为例进行查看和修改，首先登陆10.0.66.221，查看已安装的内核信息：

```
$ sudo rpm -qa |grep kernel
kernel-tools-libs-3.10.0-1127.10.1.el7.x86_64
kernel-tools-3.10.0-1127.10.1.el7.x86_64
abrt-addon-kerneloops-2.1.11-57.el7.centos.x86_64
kernel-3.10.0-1127.10.1.el7.x86_64
kernel-3.10.0-862.el7.x86_64
kernel-3.10.0-1062.12.1.el7.x86_64

$ sudo rpm -e kernel.x86_64
error: "kernel.x86_64" specifies multiple packages:
  kernel-3.10.0-862.el7.x86_64
  kernel-3.10.0-1062.12.1.el7.x86_64
  kernel-3.10.0-1127.10.1.el7.x86_64

```

然后查看系统可用内核，查看启动时可选择的内核信息：

```
$ sudo cat /boot/grub2/grub.cfg |grep menuentry
if [ x"${feature_menuentry_id}" = xy ]; then
  menuentry_id_option="--id"
  menuentry_id_option=""
export menuentry_id_option
menuentry 'CentOS Linux (3.10.0-1127.10.1.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.el7.x86_64-advanced-f67e74a9-a85f-48b9-b521-32bdb355ab80' {
menuentry 'CentOS Linux (3.10.0-1062.12.1.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.el7.x86_64-advanced-f67e74a9-a85f-48b9-b521-32bdb355ab80' {
menuentry 'CentOS Linux (3.10.0-862.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.el7.x86_64-advanced-f67e74a9-a85f-48b9-b521-32bdb355ab80' {
menuentry 'CentOS Linux (0-rescue-ffe45569e61149c0a79c77c7b38c8e33) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-0-rescue-ffe45569e61149c0a79c77c7b38c8e33-advanced-f67e74a9-a85f-48b9-b521-32bdb355ab80' {

```

下一步查看当前内核信息：

```
$ uname -a
Linux dev-k8s-node01 3.10.0-1127.10.1.el7.x86_64 #1 SMP Wed Jun 3 14:28:03 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux

```

目前默认的就是这个3.10.0-1127的内核进行启动的，可能会存在问题，这时候需要修改开机时的默认使用的内核。

```
$ sudo grub2-set-default  'CentOS Linux (3.10.0-1062.12.1.el7.x86_64) 7 (Core)'

$ sudo grub2-editenv list
saved_entry=CentOS Linux (3.10.0-1062.12.1.el7.x86_64) 7 (Core)
```

修改完成后，进行如下操作：

1. 删除存在启动问题内核

```
$ sudo rpm -qa | grep kernel
kernel-tools-libs-3.10.0-1127.10.1.el7.x86_64
kernel-tools-3.10.0-1127.10.1.el7.x86_64
abrt-addon-kerneloops-2.1.11-57.el7.centos.x86_64
kernel-3.10.0-1127.10.1.el7.x86_64
kernel-3.10.0-862.el7.x86_64
kernel-3.10.0-1062.12.1.el7.x86_64

$ sudo yum remove kernel-3.10.0-1127.10.1.el7.x86_64
Loaded plugins: fastestmirror, langpacks, versionlock
Skipping the running kernel: kernel-3.10.0-1127.10.1.el7.x86_64
No Packages marked for removal 

```

删除内核时出现上述错误，是因为当前内核正在使用，需要机器进行重启后才能删除。

2. 禁用内核自动更新

```
// 复制保留原来的配置文件
$ sudo cp /etc/yum.conf /etc/yum.conf.bak

$ sudo vim /etc/yum.conf
// 在[main]的最后添加 exclude=kernel*
exclude=kernel*

// :wq保存退出

```