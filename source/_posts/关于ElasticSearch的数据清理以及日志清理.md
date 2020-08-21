---
title: 关于ElasticSearch的数据清理以及日志清理
top: false
cover: false
toc: true
mathjax: true
date: 2020-07-15 08:59:19
password:
summary:
tags:
categories:
---

## 前言

在服务日常的运行中，ELK日志体系出现了大量的日志文件占据硬盘空间的问题，以及存在ES中数据过多，导致索引和查询的速度变慢的情况。急切需要定期清理的策略！

这里定期清理的时候，分为两个部分，一个是ES中存储的数据问题，数据量过大导致查询变慢；另一个是ES、Logstash、Kibana以及Skywalking这些工具的日常运行日志，这些日志生成过多，而且占据硬盘空间较大。

鉴于我们现在并未真正进入生产环境，这里在开发环境中，对运行日志设定存储策略，规则如下：

* Skywalking日志的定期清理，保留最近三天的运行日志信息
* ES、Logstash、Kibana各自保留最近三天内的日志信息

对ES的数据设定存储策略，规则如下：

* 保留最近三天的数据信息

## ES数据清理

清理脚本如下：

```bash
#!/bin/bash
 
###################################
#删除早于3天的ES集群的索引
###################################
# IP地址信息，ip:port
IP_ADDR="ip:port"

function delete_indices() {
    comp_date=`date -d "3 day ago" +"%Y-%m-%d"`
    date1="$1 00:00:00"
    date2="$comp_date 00:00:00"
 
    t1=`date -d "$date1" +%s`
    t2=`date -d "$date2" +%s` 
 
    if [ $t1 -le $t2 ]; then 
        echo "$1时间早于$comp_date，进行索引删除"
        #转换一下格式，将类似2017--01格式转化为2017.10.01
        format_date=`echo $| sed 's/-/\./g'`
        curl -XDELETE http://$IP_ADDR/*$format_date
    fi
}
 
curl -XGET http://$IP_ADDR/_cat/indices | awk -F" " '{print $3}' | awk -F"-" '{print $NF}' | egrep "[0-9]*\.[0-9]*\.[0-9]*" | sort | uniq  | sed 's/\./-/g' | while read LINE
do
    #调用索引删除函数
    delete_indices $LINE
done

```

利用date命令获取对应的日期信息，删除对应日期之前的索引数据。

后续修改主机的ip地址信息，配置crontab定时任务，执行即可。

**注意：**当ES启用x-pack插件时，存在对ES的用户名和密码的验证，在curl执行时，一定注意要添加**-u ES_USERNAME:ES_PASSWORD*!

## Skywalking日志清理

清理脚本如下：

```bash
#!/bin/bash

###################################
#删除早于2天的ES集群的索引
###################################
function delete_indices() {
    comp_date=`date -d "2 day ago" +"%Y-%m-%d"`
    date1="$1 00:00:00"
    date2="$comp_date 00:00:00"

    t1=`date -d "$date1" +%s`
    t2=`date -d "$date2" +%s`

    if [ $t1 -le $t2 ]; then
        echo "$1时间早于$comp_date，进行索引删除"
        #curl -XDELETE http://192.168.21.208:9200/*$format_date
        rm -rf skywalking-oap-server-$1*.log
    fi
}

# 筛选日志信息，对应路径下，筛选日志文件名称，提取日期信息，传入处理函数中
cd /home/centos/apache-skywalking-apm-bin-es7/logs && ls -al | grep skywalking-oap-server-20 | awk '{print $9}' | sort | uniq | grep -P -o "[0-9]{4}-[0-9]{1,2}-[0-9]{1,2}" | while read LINE
do
    #调用索引删除函数
    delete_indices $LINE
done

```

将脚本置于对应的机器中，目前该脚本可以放置在192.168.21.208机器上，配置crontab定时任务，执行即可。

## ELK运行日志限制

### ElasticSearch日志配置

```

$ sudo vim /opt/es/config/log4j2.properties

######## Server JSON ############################
appender.rolling.type = RollingFile
appender.rolling.name = rolling
appender.rolling.fileName = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}_server.json
appender.rolling.layout.type = ESJsonLayout
appender.rolling.layout.type_name = server

appender.rolling.filePattern = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}-%d{yyyy-MM-dd}-%i.json.gz
appender.rolling.policies.type = Policies
appender.rolling.policies.time.type = TimeBasedTriggeringPolicy
appender.rolling.policies.time.interval = 1
appender.rolling.policies.time.modulate = true
appender.rolling.policies.size.type = SizeBasedTriggeringPolicy
appender.rolling.policies.size.size = 128MB
appender.rolling.strategy.type = DefaultRolloverStrategy
appender.rolling.strategy.fileIndex = nomax
appender.rolling.strategy.action.type = Delete
appender.rolling.strategy.action.basepath = ${sys:es.logs.base_path}
appender.rolling.strategy.action.condition.type = IfFileName
appender.rolling.strategy.action.condition.glob = ${sys:es.logs.cluster_name}-*
# 删除该处的内容
#appender.rolling.strategy.action.condition.nested_condition.type = IfAccumulatedFileSize
#appender.rolling.strategy.action.condition.nested_condition.exceeds = 2GB
# 下面是替换的内容
appender.rolling.strategy.action.condition.nested_condition.type = IfLastModified
appender.rolling.strategy.action.condition.nested_condition.age = 3D

```
配置信息解释：

```

appender.rolling.type = RollingFile ：配置RollingFile输出源
appender.rolling.fileName = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}.log：日志到/var/log/elasticsearch/production.log
appender.rolling.filePattern = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}-%d{yyyy-MM-dd}-%i.log.gz：滚动日志到/var/log/elasticsearch/production-yyyy-MM-dd-i.log，日志将被压缩在每个滚动上，并且i将被递增。
appender.rolling.policies.time.type = TimeBasedTriggeringPolicy：使用基于时间的滚动策略
appender.rolling.policies.time.interval = 1：每天滚动日志
appender.rolling.policies.time.modulate = true：在一天的边界上对齐滚动条(而不是每24小时滚动一次)
appender.rolling.policies.size.type = SizeBasedTriggeringPolicy：使用基于大小的滚动策略
appender.rolling.policies.size.size = 256MB：在256MB后滚动日志
appender.rolling.strategy.action.type = Delete：在滚动日志时使用删除操作
appender.rolling.strategy.action.condition.type = IfFileName：只删除匹配文件模式的日志
appender.rolling.strategy.action.condition.glob = ${sys:es.logs.cluster_name}-*：模式是只删除主日志
appender.rolling.strategy.action.condition.nested_condition.type = IfAccumulatedFileSize：只有当我们积累了太多的压缩日志时才删除
appender.rolling.strategy.action.condition.nested_condition.exceeds = 2GB：压缩日志的大小条件是2GB

```

修改的内容如下：

* appender.rolling.strategy.action.condition.nested_condition.type= IfLastModified   # 要应用于与glob匹配的文件的嵌套条件
* appender.rolling.strategy.action.condition.nested_condition.age= 3D                # 保留日志三天

原因如下：

默认配置问题：ElasticSearch默认情况下会每天rolling一个文件，当到达2G的时候，才开始清除超出的部分，当一个文件只有几十K的时候，文件会一直累计下来，一直占据存储空间，并不清除！

修改完成后记得重启ElasticSearch！

### Logstash日志配置

```
$ sudo vim /opt/logstash/config/log4j2.properties

// 从DefaultRolloverStrategy这个开始新增
appender.rolling.strategy.type = DefaultRolloverStrategy
appender.rolling.strategy.max = 30
appender.rolling.strategy.action.type = Delete
appender.rolling.strategy.action.basepath = ${sys:es.logs.base_path}
appender.rolling.strategy.action.condition.type = IfFileName
appender.rolling.strategy.action.condition.glob = ${sys:es.logs.cluster_name}-*
appender.rolling.strategy.action.condition.nested_condition.type = IfLastModified
appender.rolling.strategy.action.condition.nested_condition.exceeds = 3D

```

修改原因：

默认配置问题:Logstash会一直增长gc文件和不停增多的rolling日志文件，并且不会删除。

这里配置基本上和ES的日志配置是一致的。

修改完成后记得重启Logstash。

## 总结

日志信息保留，三个月周期，设置时参考上述配置信息即可！

## 参考地址

* 清理脚本编写：https://blog.csdn.net/felix_yujing/article/details/78207667
* 关于ELK的运行日志设置：https://blog.51cto.com/huanghai/2430038
