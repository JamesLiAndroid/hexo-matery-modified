# kafka教学

----------------------zookeeper-----------------------

分布式协调服务，一个通用的无单点问题的分布式协调框架

多机多进程

网络不可靠、通信不同步、关键节点可能不可用

客户端：curactor

Observer（只接收请求）、Leader（所有写）、Follower（可成为Leader）

尝试添加follower角色

过半数方法

ZAB协议

客户端长连接

zk的节点：有序无序、持久临时

应用：分布式id，雪花算法、利用zk生成滚动序号，保证递增

watch机制

临时性节点+watch机制


kafka：0.10.2.1  + zookeeper：3.4.6版本对应

2181对外、2881节点内部通信、3881Leader选举。

注意配置端口号对应

clientPort=21810
server.1=10.0.66.201:28880:38880
server.2=10.0.66.206:28880:38880
server.3=10.0.66.209:28880:38880

实践：对zookeeper配置服务，对kafka配置服务，同时关联kafka manager


-----------------kafka------------------------

日志信息

单机50w~60w的数据

Partition的副本机制，高可用

key Hash算法，消息有序性保证，导致partition之间的不均衡。

多个Consumer来读取一个Topic(理想情况下是一个Consumer读取Topic的一个Partition）,那么可以让这些Consumer属于同一个Consumer Group即可实现消息的多Consumer并行处理。

消息保留策略：

log.retention.hours：消息保留多长时间（默认168小时，即一周）
log.retention.bytes：消息保留最大占用空间

最先达到最先触发。

副本与ISR

ISR数量必须与分区副本数相同，否则会出现问题


Producer可靠性

水位：Leader成功将分区复制到follower中

问题：jdk的版本选择。

实践：添加kafka的broker，新增一个进入集群中。

---------------------kafka 运维--------------------------------

kafka Manager 二次启动时需要删除上一次运行所创建的RUNNING_PID已存在，需要删除该id


----------------------实践部分----------------------------------

各类基础服务配置为Service，使用systemctl进行管理。

需要总结centos中配置服务的教程！

可以通过脚本的方式触发针对topic的重选举。

Telegraf+JMXTrans+influxDB+Grafana




######################################################################################################################

# Telegraf+grafana培训

TICK技术栈

Telegraf+InfluxDB+Chronograf+kapacitor（预警）

实验一完成。

注意：influxdb使用的是UTC时间

时间+值，多个序列

Retention Policies 对所有数据表起作用

对于InfluxDB初始化:

1. 禁用信息收集

reporting-disabled=true

2. 禁用每个数据库的序列个数


3. 禁用每个维度最多维度的个数

influxDB配置信息


业务关系链路，图数据库

谁向你写入，数据来源

治理和审计

预警体系

自动运维层面

自有运营与其它产品开发

TDengine

Saiku