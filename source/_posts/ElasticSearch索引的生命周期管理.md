---
title: ElasticSearch索引的生命周期管理
top: false
cover: false
toc: true
mathjax: true
date: 2020-10-14 14:39:45
password:
summary:
tags:
categories:
---

# ElasticSearch索引的生命周期管理


## 前提

ElasticSearch 6.6.0版本后新增索引的生命周期管理。

参考易企秀的实现：

引入索引生命周期管理的一个最重要的目的就是对大量时序数据在es读写操作的性能优化，比如易企秀通过spark streaming读取Kafka中的日志实时写入es，这些日志高峰期每天10亿+，每分钟接近100w，在这么大数据量上进行操作是一件很麻烦的事，那么我们希望es能够对单分片超过50g或者30天前的索引进行归档，并能够自动删除90天前的索引，那么这个场景可以通过ILM进行策略配置来实现。

那么实际上我们可以参考易企秀的内容来制定我们自身的索引规则，尤其是目前在服务器资源紧张的情况下，而对于日志信息的记录要求保存的时间不高，因此借助索引的生命周期管理来优化ES的运行。

## 生命周期简介

ES索引生命周期管理分为4个阶段：hot、warm、cold、delete，其中hot主要负责对索引进行rollover操作，warm、cold、delete分别对rollover后的数据进一步处理（前提是配置了hot）。

|phases	|desc|
|---------	|----------|
|hot	|主要处理时序数据的实时写入|
|warm	|可以用来查询，但是不再写入|
|cold	|索引不再有更新操作，并且查询也会很少|
|delete	|数据将被删除|

需要注意的是上述4个阶段不是必须的，也就是说不是一定要从hot->warm->cold->delete，可以直接从hot->cold。

## 设置规则

参考易企秀的策略设置，考虑我们当前服务器负载以及硬盘空间，分别对目前系统中的两套ES进行规则设置，如下：

* ELK所在ES： 4G 3天 7天
* Skywalking所在ES： 10G 7天 14天

解释一下上述配置的含义，针对ELK所在的ES，对单分片超过*4G*或者*3天前*的索引进行归档，并能够自动删除**7天前**的索引；针对Skywalking所在的ES，对单分片超过*10G*或者*7天前*的索引进行归档，并能够自动删除**14天前**的索引。

## 规则编写与落地

这里所有的操作，均借助[Cerebro](https://github.com/lmenezes/cerebro)对ES进行操作，以Skywalking所在ES为例子，进行设置。

首先查看一下系统中默认使用的规则，如下图：

![](系统内默认的策略信息.png)

然后看一下目前的索引状态，如下图：

![](当前索引未被管理.png)

这样目前所有的**elastic-skywalking_**开头的索引信息中*managed*状态均为false。那么久需要我们进行设置。

### 1. 定义策略

针对Skywalking所在ES设置策略，将该策略应用到**elastic-skywalking_**开头的索引信息中。设置策略的json如下：

```json

{  
    "policy": {                       
    "phases": {
      "hot": {                      
        "actions": {
          "rollover": {             
            "max_size": "10GB",
            "max_age": "7d"
          }
        }
      },
      "delete": {
        "min_age": "14d",           
        "actions": {
          "delete": {}              
        }
      }
    }
  }
}

```

注意：rollover中配置归档策略，目前支持3种策略，分别是max_docs、max_size、max_age（请关注、具体后续内容介绍），其中的任何一个条件满足时都会触发索引的归档操作，并删除归档90天后的索引文件（其中delete属于phrase）。

然后操作如下：

![](索引保留规则设置.png)

最后查看一下我们设置的规则，如下：

![](索引规则设置结果.png)

这样索引规则就设置完成了。

### 2. 使用策略

在索引库上使用策略的方式有很多种，但我们的需求是对满足所有rollover条件的规则统统适用，所以需要配置一个索引模板，并指定对应的规则。策略应用的json如下：

```json
{
  "index_patterns": ["elastic-skywalking_*"],                 
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "index.lifecycle.name": "skywalking-policy",      
    "index.lifecycle.rollover_alias": "datastream"    
  }
}

```

设置策略应用的结果如下：

![](索引策略应用.png)

其中 index.lifecycle.name 指定我们索引模板使用哪个策略进行管理，index.lifecycle.rollover_alias 配置该系列索引的别名，通过别名datastream可对elastic-skywalking_*相关的索引信息进行读写操作。

### 3. 别名添加

在上面的步骤中添加了别名信息，但是查看所有的索引信息，发现并没有这个别名，如下：

![](别名信息查询.png)

这也就导致了设置的索引策略并不能生效，那么需要对各个索引信息添加别名，添加的json信息如下：

```json
{
    "actions": [
    {
      "add": {
        "index": "elastic-skywalking_*",
        "alias": "datastream"
      }
    }
  ]
}

```

添加的结果如下：

![](别名信息添加.png)

添加完成后查看别名信息如下：

![](别名信息查看.png)

![](别名信息查看3.png)

![](别名信息查看2.png)

这样就完成了所有操作。可以看到最后一张图中在*phase_execution*下，出现了policy设置的相关信息。但是，设置完成后，只对于新生成的索引进行了管理，引入了我们生成的policy，并未对已有的索引进行管理，这也是这个方案的一个问题吧！

## 总结

根据Skywalking写入ES的特点，会在每天生成一个记录的索引信息，保存apm监控的链路数据。一天一滚动。目前临时不对索引的别名设置**is_write_index = true**这个选项(详细见[参考连接](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-aliases.html))。

设置完成后，需要在一个月内监控一下这个系统的运行情况！

<!-- 由于别名操作索引时，同一时刻只能有一个索引被写，所以还需要设置 is_write_index = true 。

curl -X PUT "localhost:9200/datastream-000001" -H 'Content-Type: application/json' -d'
{
  "aliases": {
    "datastream": {
      "is_write_index": true
    }
  }
}
' 
--> 

## 参考文档

* https://www.jianshu.com/p/358fde8d8e27

* https://www.jianshu.com/p/94e37a5b0878

* https://www.cnblogs.com/Neeo/articles/10897280.html