---
title: log-pilot+kafka+ELK构建生产级日志管理系统
top: false
cover: false
toc: true
mathjax: true
date: 2020-07-10 22:13:14
password:
summary:
tags:
categories:
---
# log-pilot+kafka+ELK构建生产级日志管理体系

## 前言

log-pilot 是阿里云提供的日志收集镜像。我们可以在每台机器上部署一个 log-pilot 实例，就可以收集机器上所有 Docker 应用日志。

log-pilot具有以下特性：

* 一个单独的 log 进程收集机器上所有容器的日志。不需要为每个容器启动一个 log 进程。
* 支持文件日志和 stdout。docker log dirver 亦或 logspout 只能处理 stdout，log-pilot 不仅支持收集 stdout 日志，还可以收集文件日志。
* 声明式配置。当您的容器有日志要收集，只要通过 label 声明要收集的日志文件的路径，无需改动其他任何配置，log-pilot 就会自动收集新容器的日志。
* 支持多种日志存储方式。无论是强大的阿里云日志服务，还是比较流行的 elasticsearch 组合，甚至是 graylog，log-pilot 都能把日志投递到正确的地点。
* 开源。log-pilot 完全开源，您可以从 Git项目地址 下载代码。如果现有的功能不能满足您的需要，欢迎提 issue。

ELK是统一的日志管理工具，分为Logstash、ElasticSearch、Kibana。Logstash对日志进行收集和处理筛选，ElasticSearch负责日志的存储，Kibana进行日志的查询和展示。

Kafka是最初由Linkedin公司开发，是一个分布式、支持分区的（partition）、多副本的（replica），基于zookeeper协调的分布式消息系统，它的最大的特性就是可以实时的处理大量数据以满足各种需求场景：比如基于hadoop的批处理系统、低延迟的实时系统、storm/Spark流式处理引擎，web/nginx日志、访问日志，消息服务等等，用scala语言编写，Linkedin于2010年贡献给了Apache基金会并成为顶级开源 项目。

利用以上工具链构建日志管理体系。

## 架构图示

![](分布式日志系统ELK.png)

运行流程如下：

1. 通过log-pilot收集服务所在容器的日志信息并将收集到的日志信息写入kafka
2. 利用Logstash从kafka中，获取日志信息，并进行处理，写入后向的ElasticSearch
3. ElasticSearch提供日志存储和分片功能
4. 利用kibana对日志进行统一展示和查询，并根据条件进行筛选

## 版本信息和机器信息

1. 版本信息：

* ELK体系：7.6.2

* kafka：2.11-2.4.1

* log-pilot: 0.9.7-filebeat(使用filebeat版本进行收集)

2. 机器信息

|IP地址| 功能|
|-----|-----|
|192.168.229.237 |  ELK服务所在机器|
|192.168.229.206 |  kafka |
|192.168.229.206 |  zookeeeper |
|192.168.229.209 |  zookeeper |
|192.168.229.201 |  zookeeper |

## ELK部署

### 1. 组件及对应端口

|服务| 端口|
|-----|-----|
|Elasticsearch |	9200（数据连接）、9300（集群通信）|
|Cerebro |	9000|
|Kibana |	5601|
|Logstash	|6514（tcp/udp）、6515（tcp）|
|zookeeper  | 28880、38880、21810 |

### 2. ELK文件及目录

|文件或目录	|说明|
|-----|-----|
|/opt/es/config/elasticsearch.yml	|es的主配置文件|
|/opt/es/config/jvm.options	|es的jvm配置文件|
|/opt/es|	es安装目录|
|/opt/es_logs/	|es日志目录|
|/opt/es_data/	|es数据目录|
|/opt/kibana/config/kibana.yml|	kibana主配置文件|
|/opt/cerebro/conf/application.conf	|cerebro主配置文件|
|/opt/logstash-conf/|	logstash配置文件目录|

### 3. 安装ELK

#### 3.1 使用脚本进行安装

登录192.168.229.237服务器，执行下面的命令进行安装

```
# cd ~
# wget -O elk762.sh http://www.bigops.com/bigops-install/elk762.sh
# sh +x elk762.sh

```

提示：如果脚本下载elk太慢，可以手动下载，把ELK下载文件和elk762.sh脚本放在同一个目录再运行安装脚本。

#### 3.2 启动服务

**后续步骤一：设置ES密码**
-------------------
等待10秒后，查看ES是否启动，运行命令：
```
$ sudo netstat -nptl|grep 9[2,3]00
```

如果没有启动，请尝试手动启动：

```
$ su - es
(es) $ /opt/es/bin/elasticsearch

```

或者使用下面的命令进行启动：

```
$ sudo systemctl start es

```

9200和9300端口启动后，运行下面命令设置密码

```
$ sudo /opt/es/bin/elasticsearch-setup-passwords interactive
Please confirm that you would like to continue [y/N]，回答y
```

要设置的密码比较多，都设置成ES连接密码，这里所有的密码均设置为*111111*。


**后续步骤二：启动kibana**
-------------------

运行下面的命令启动kibana：

```
$ sudo systemctl restart kibana
```

等待10秒后，查看端口是否启动，运行命令

```
$ sudo netstat -nplt|grep 5601

```

5601端口启动后，使用浏览器访问：http://120.132.33.210:5601

**默认登录用户名：elastic**
**密码：111111**

*安装时，ELK7.6.2自带x-pack插件进行认证。*


**后续步骤三：启动logstash**
-------------------

运行命令

```
$ sudo systemctl restart logstash
```

等待10秒后，查看端口是否启动，运行命令

```
$ sudo netstat -npl|grep 6514

```

#### 3.3 修改防火墙策略

确认防火墙6514/(tcp/udp)、6515(tcp)等端口容许被访问。

```
// 开启kibana端口号
$ sudo firewall-cmd --add-port=6514/tcp --zone=public --permanent

$ sudo firewall-cmd --add-port=6514/udp --zone=public --permanent

$ sudo firewall-cmd --add-port=6515/tcp --zone=public --permanent

// 开启ElasticSearch端口号
$ sudo firewall-cmd --add-port=9200/tcp --zone=public --permanent

$ sudo firewall-cmd --add-port=9300/tcp --zone=public --permanent

// 开启kibana端口号
$ sudo firewall-cmd --add-port=5601/tcp --zone=public --permanent

$ sudo firewall-cmd --reload

```

如果使用了公有云，需要打开端口策略。

#### 3.4 测试Logstash端口连通性

登录一台日志客户端，运行命令确认6514、6515端口是open。

```
nmap -sU -pU:6514 日志服务器IP
```

等Bigops系统安装好后，登录kibana查看是否有数据进来。


#### 3.5 安装cerebro，用于图形化管理ES（选装）

```
$ cd ~
$ wget -O cerebro.sh http://www.bigops.com/bigops-install/cerebro.sh
$ sudo bash cerebro.sh

```
安装完成后，开启cerebro对应的端口信息：

```
$ sudo firewall-cmd --add-port=9000/tcp --zone=public --permanent

$ sudo firewall-cmd --reload
```

直接访问http://192.168.229.237:9000，登录时*输入用户名elastic，输入密码11111*，即可进入。

## kafka部署

首先部署zookeeper，在201、206、209上部署zookeeper，参考**Linux系统初始化以及部署**文档进行安装。

然后安装kafka：

```
$ cd  ~ && mkdir kafka

$ wget https://mirror.bit.edu.cn/apache/kafka/2.4.1/kafka_2.11-2.4.1.tgz

$ tar -zxvf kafka_2.11-2.4.1.tgz

$ cd kafka_2.11-2.4.1/

$ vim config/server.properties
// 修改下面几个选项

// 设置监听器
advertised.listeners=PLAINTEXT://192.168.229.206:9092
// 设置zookeeper连接
zookeeper.connect=192.168.229.206:21810,192.168.229.209:21810,192.168.229.201:21810

// :wq保存退出

```

最后启动kafka服务：

```
$ nohup ./bin/kafka-server-start.sh config/server.properties &

$ ps -ef | grep kafka
// 查看是否开启

```

下面简单测试一下kafka是否能正常运行，创建topic的操作如下：

```
$ ./bin/kafka-topics.sh --create --zookeeper 192.168.229.206:21810,192.168.229.209:21810,192.168.229.201:21810 --replication-factor 1 --partitions 1 --topic test

// 查看已经创建的topic
$ ./bin/kafka-topics.sh --list --zookeeper 192.168.229.206:21810
test

```

出现了test说明已经创建完成了。

稍后进行产生消息和消费消息的操作：

```
// 产生消息
$ ./bin/kafka-console-producer.sh --broker-list PLAINTEXT://192.168.229.206:9092 --topic test
>{"key":"value"}

// 消费消息
$ ./bin/kafka-console-consumer.sh --bootstrap-server 192.168.229.206:9092 --topic test --from-beginning
{"key":"value"}

```

这样消息就能进行消费了，kafka就能正常运行了。

## docker日志收集实践

目前开发环境中，服务部署使用docker容器进行，需要对docker容器进行日志收集。

### 1. log-pilot部署

登录开发环境部署机器192.168.229.199，拉取log-pilot官方镜像，并进行启动，以非交互式不保存任何数据的方式启动，便于我们调试。

```
$ docker pull registry.cn-hangzhou.aliyuncs.com/acs/log-pilot:0.9.7-filebeat

$ docker run --rm -it --name log-pilot --privileged -v /var/run/docker.sock:/var/run/docker.sock -v /etc/localtime:/etc/localtime -v /:/host:ro --cap-add SYS_ADMIN -e 'LOGGING_OUTPUT=kafka' -e 'KAFKA_BROKERS=192.168.229.206:9092' -e 'KAFKA_DEFAULT_TOPIC=test'  registry.cn-hangzhou.aliyuncs.com/acs/log-pilot:0.9.7-filebeat

```

在运行log-pilot容器时，需要指定以下全局变量：

- LOGGING_OUTPUT=kafka 
  指定日志信息写入kafka

- KAFKA_BROKERS=192.168.229.206:9092
  指定kafka的服务器地址信息

- KAFKA_DEFAULT_TOPIC=test
  指定默认写入的topic为test，不强制要求

下面解释一下docker run 时指定的参数信息：

|Options|	Mean|
|---|	---- |
|-i	|以交互模式运行容器，通常与 -t 同时使用；|
|-t	|为容器重新分配一个伪输入终端，通常与 -i 同时使用；|
|-d	|后台运行容器，并返回容器ID；|

这样，就可以启动了日志收集的工具。但是目前我们要停止该容器，ctrl+c就可以停止该程序，先将kafka写入logstash走通。

### 2. kafka测试写入Logstash

登录到kafka所在机器，启动我们上述使用的命令，向*test*所在的topic中写入数据，如下：

```
// 产生消息
$ ./bin/kafka-console-producer.sh --broker-list PLAINTEXT://192.168.229.206:9092 --topic test
>{"key":"value"}

```

然后转到ELK安装的机器上，对Logstash先进行停用，配置kafka写入的内容，以命令行方式启动Logstash，如下：

```
$ sudo systemctl stop logstash

// 转换到logstash配置文件所在目录
$ cd /opt/logstash-conf/

$ sudo vim kafa.conf
// 写入以下信息
input{
    kafka {
        bootstrap_servers => "192.168.229.206:9092"
        topics => ["test"]
        codec => "json"
    }
}

filter{
    grok {
        match => {
            remove_field => [ "beat", "source", "stream", "prospector", "offset", "@version", "@timestamp"]
        }
    }
}

output {
        stdout {
            codec => rubydebug
        }
        #elasticsearch {
        #        hosts => ["192.168.229.237:9200"]
        #        user => "elastic"
        #        password => "111111"
        #        index => "filebeat-%{+yyyy.MM.dd}"
        #}
}

// :wq保存退出

// 启动Logstash，可打印调试日志
$ sudo /opt/logstash/bin/logstash --path.settings /opt/logstash/config -f /opt/logstash-conf/kafka.conf --verbose --debug

```

目前不需要设置写入到ES，直接在命令行中输出日志信息。这样命令行中会输出的信息如下：

```
[2020-07-10T14:09:11,039][DEBUG][logstash.filters.grok    ][main] Running grok filter {:event=>#<LogStash::Event:0x4f54382f>}
[2020-07-10T14:09:11,039][DEBUG][org.apache.kafka.clients.consumer.internals.Fetcher][main] [Consumer clientId=logstash-0, groupId=logstash] Sending READ_UNCOMMITTED IncrementalFetchRequest(toSend=(test-0), toForget=(), implied=()) to broker 192.168.229.206:9092 (id: 0 rack: null)
[2020-07-10T14:09:11,045][DEBUG][logstash.filters.grok    ][main] Event now:  {:event=>#<LogStash::Event:0x4f54382f>}
{
    "@timestamp" => 2020-07-10T06:09:10.937Z,
          "tags" => [
        [0] "_jsonparsefailure",
        [1] "_grokparsefailure"
    ],
       "message" => "{\"\"\e[Dkey\":\"value\"}",
      "@version" => "1"
}


```

这样从kafka到Logstash写入信息就走通了。这时候使用ctrl+c停止运行的logstash，重新以服务方式启动，如下：

```
$ sudo systemctl restart logstash

```

### 3. log-pilot写入kafka

这里启动log-pilot镜像，更换写入的topic，来测试从log-pilot将日志写入kafka中。

在开发环境部署机器中，我们启动一个数据字典服务，利用log-pilot来采集该服务所在docker镜像的日志信息，启动方式如下：

```
$ docker run -d -p 19090:19090 -e CHANNEL="standalone" -e IP_ADDR="192.168.229.199" -e NACOS_IP="192.168.229.206:18848" -e NACOS_NAMESPACE="858b37b1-35be-4564-a24e-dc2c322d5784"  -e SKYWALKING_NAMESPACE="test-dev" -e SKYWALKING_TARGET_SERVICE_NAME="test-data-dict-develop" -e SKYWALKING_IP_PORT="192.168.229.208:11800" --label aliyun.logs.dict=stdout --label aliyun.logs.dict.tags="topic=test,env=dev,service=test-data-dict"  --name test-dict 192.168.229.202:5000/test-data-dict-develop:$BUILD_NUMBER

```

这里比起之前的启动命令，多出了两个参数：

- --label aliyun.logs.dict=stdout  
  指定日志所属的服务，stdout表示采集docker输出的日志信息

- --label aliyun.logs.dict.tags="topic=test,env=dev,service=test-data-dict"  
  指定服务对应的tags，其中*topic=test*指定了kafka里面已有的topic，*env=dev*指定来自开发环境的日志，*service=test-data-dict*指定服务的名称

启动数据字典之后，用与上文同样的方式启动log-pilot镜像，这样就开启了收集日志的工作：

```
$ docker run --rm -it --name log-pilot --privileged -v /var/run/docker.sock:/var/run/docker.sock -v /etc/localtime:/etc/localtime -v /:/host:ro --cap-add SYS_ADMIN -e 'LOGGING_OUTPUT=kafka' -e 'KAFKA_BROKERS=192.168.229.206:9092' -e 'KAFKA_DEFAULT_TOPIC=test'  registry.cn-hangzhou.aliyuncs.com/acs/log-pilot:0.9.7-filebeat

```

如何验证数据字典服务的日志已经被写入到消息队列中？我们可以通过命令行中kafka-console-consumer.sh工具来实现

```
$ ./bin/kafka-console-consumer.sh --bootstrap-server 192.168.229.206:9092 --topic test --from-beginning
{"key":"value"}

```

另外也可以通过kafka的图形化客户端来查看，本人开发机使用win7 64位系统。下载[kafka Tool 2.0.7](https://www.kafkatool.com/download.html)，如下图：

![](kafkatool下载.png)

安装后，连接目前运行的kafka服务，如下：

![](kafkatool连接.png)

连接后查看topics下test所在的Partition 0中的信息，点击左上角的开始按钮，如下：

![](kafkatool查看信息.png)

如果展示不出来，或者展示的信息为乱码，需要设置topic的Content Types中，选择展示String类型的信息，如下：

![](设置使用String展示.png)

这样就能看到，log-pilot收集的日志已经发送到kafka中了。

**注意**：后续的服务都应该参考数据字典的设置，在构建时添加对应的

### 4. 服务联调

前面的连通性测试都已经通过，这时候就要正式启动该套体系了，首先修改Logstash的设置，保证能够写入的ElasticSearch中，然后修改logs-pilot的启动方式，让它在后台自动运行。

### 4.1 修改Logstash的配置信息

登录到ELK所在服务器上，停用Logstash服务，修改配置信息，如下：

```
$ sudo systemctl stop logstash

// 转换到logstash配置文件所在目录
$ cd /opt/logstash-conf/

$ sudo vim kafa.conf
// 写入以下信息
input{
    kafka {
        bootstrap_servers => "192.168.229.206:9092"
        topics => ["test"]
        codec => "json"
    }
}

filter{
    grok {
        match => {
            remove_field => [ "beat", "source", "stream", "prospector", "offset", "@version", "@timestamp"]
        }
    }
}

output {
        #stdout {
        #    codec => rubydebug
        #}
        elasticsearch {
                hosts => ["192.168.229.237:9200"]
                user => "elastic"
                password => "111111"
                index => "filebeat-%{+yyyy.MM.dd}"
        }
}

// :wq保存退出

// 重启生效
$ sudo systemctl start logstash

```

去掉之前测试时使用的命令行输出--stdout，解除对ElasticSearch写入的配置的注释，配置Logstash可以写入ES中。

要注意，这里的**index => "filebeat-%{+yyyy.MM.dd}"**，是我们后续再kibana中配置显示是所需要匹配的索引信息。

### 4.2 修改log-pilot的启动方式

登录开发环境机器，停止目前在命令行中运行的log-pilot容器信息，使用下面的命令正式启动：

```

$ docker run -d --restart always --name log-pilot --privileged -v /var/run/docker.sock:/var/run/docker.sock -v /etc/localtime:/etc/localtime -v /:/host:ro --cap-add SYS_ADMIN -e 'LOGGING_OUTPUT=kafka' -e 'KAFKA_BROKERS=192.168.229.206:9092' -e 'KAFKA_DEFAULT_TOPIC=test'  registry.cn-hangzhou.aliyuncs.com/acs/log-pilot:0.9.7-filebeat

```

这样log-pilot就可以在后台运行了。

### 5. kibana设置和展示

在前面我们的的ELK体系已经搭建完成了，这样就可以直接登录kibana系统了。

* 地址：192.168.229.237:5601

* 用户名：elastic

* 密码：111111

登录后如下图：

![](Kibana首页.png)

首先查看索引信息，点击Management（管理），选在kibana中的索引模式，如下：

![](kibana索引模式.png)

点击右上角**创建索引模式**，按照下图开始创建：

![](kibana创建索引模式.png)

输入**filebeat-\***，这样能匹配出下面的索引信息，自动提示可用。点击下一步，进入配置设置：

![](kibana配置时间筛选设置.png)

按照图示设置完成，点击选择时间筛选字段名称后，右下方的*创建索引模式*按钮，变为可点击，点击后即可创建索引模式。

创建完成后回到索引模式页面，会展示出刚才创建的**filebeat-\***。

最后，查看一下我们在ES中的数据信息，如下图操作：

![](kibana查看索引中的日志信息.png)

这样就能看到我们自己写入的日志信息了！

## 日志收集设置

这里发现一个问题，当日志收集时，面对一些错误日志，本应该一行显示的，变成了多行显示。如下图：

![](日志断开.png)

尤其是对于异常日志，不能很好地合并到一块，导致查看错误日志的时候不直观。

对于上述问题，重写log-pilot镜像中的配置文件，并重新制作log-pilot的镜像信息。

首先重新编写filebeat.tpl文件：

```
$ mkdir log-pilot && vim filebeat.tpl

// 输入以下信息
{{range .configList}}
- type: log
  enabled: true
  paths:
      - {{ .HostDir }}/{{ .File }}
  multiline.pattern: '^\[|\{|([0-9]{4}-[0-9]{2}-[0-9]{2})|([0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3})|([0-9]{1,2}-[A-Za-z]{3}-[0-9]{4})'
  multiline.negate: true
  multiline.match: after
  multiline.max_lines: 10000
  scan_frequency: 10s
  fields_under_root: true
  {{if .Stdout}}
  docker-json: true
  {{end}}
  {{if eq .Format "json"}}
  json.keys_under_root: true
  {{end}}
  fields:
      {{range $key, $value := .Tags}}
      {{ $key }}: {{ $value }}
      {{end}}
      {{range $key, $value := $.container}}
      {{ $key }}: {{ $value }}
      {{end}}
  tail_files: false
  close_inactive: 2h
  close_eof: false
  close_removed: true
  clean_removed: true
  close_renamed: false

{{end}}

// :wq保存退出

```

主要修改的位置是**multiline.pattern**，这里的正则表达式主要是匹配了日志信息的开头，凡是以日期信息开头的都进行多行匹配，合并为同一条数据，直到遇到下一个日志信息开头停止。

这样就可以将分在多行中的数据，合并到同一条日志信息中。

然后制作新的log-pilot镜像，如下：

```
$ vim Dockerfile
// 填写以下信息
FROM registry.cn-hangzhou.aliyuncs.com/acs/log-pilot:0.9.7-filebeat
COPY filebeat.tpl /pilot/

// :wq保存退出

// 制作docker镜像
$ docker build -t log-pilot-self:0.9.7-filebeat .

// 制作完成后打tag并推送到docker镜像仓库中
$ docker images | grep log-pilot-self
log-pilot-self                                    0.9.7-filebeat      a0a464bd4dc1        31 hours ago        119MB

$ docker tag log-pilot-self:0.9.7-filebeat 192.168.229.202:5000/log-pilot-self:0.9.7-filebeat

$ docker push 192.168.229.202:5000/log-pilot-self:0.9.7-filebeat

```

这样就操作完成了，最后需要替换一下之前运行的log-pilot镜像信息。

```
$ docker run -d --name log-pilot --privileged -v /var/run/docker.sock:/var/run/docker.sock -v /etc/localtime:/etc/localtime -v /:/host:ro --cap-add SYS_ADMIN -e 'LOGGING_OUTPUT=kafka' -e 'KAFKA_BROKERS=192.168.229.206:9092' -e 'KAFKA_DEFAULT_TOPIC=test'  192.168.229.202:5000/log-pilot-self:0.9.7-filebeat

```

这样再去查看日志信息时，就大大减少了断开的情况。

## k8s日志收集实践

### 1. log-pilot部署

转到k8s管理机，登录到192.168.229.240上，首先编写部署的配置文件，如下：

```

$ mkdir log-pilot && cd log-pilot/

$ vim log-pilot.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-pilot
  labels:
    app: log-pilot
  namespace: test-basic-log-pilot
spec:
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: log-pilot
  template:
    metadata:
      labels:
        app: log-pilot
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: log-pilot
        image: 192.168.229.202:5000/log-pilot-self:0.9.7-filebeat
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 200m
            memory: 200Mi
        env:
          - name: "NODE_NAME"
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: "LOGGING_OUTPUT"
            value: "kafka"
          - name: "KAFKA_BROKERS"
            value: "192.168.229.206:9092"
          - name: "KAFKA_DEFAULT_TOPIC"
            value: "test"
        volumeMounts:
        - name: sock
          mountPath: /var/run/docker.sock
        - name: root
          mountPath: /host
          readOnly: true
        - name: varlib
          mountPath: /var/lib/filebeat
        - name: varlog
          mountPath: /var/log/filebeat
        - name: localtime
          mountPath: /etc/localtime
          readOnly: true
        livenessProbe:
          failureThreshold: 3
          exec:
            command:
            - /pilot/healthz
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 2
        securityContext:
          capabilities:
            add:
            - SYS_ADMIN
      terminationGracePeriodSeconds: 30
      volumes:
      - name: sock
        hostPath:
          path: /var/run/docker.sock
      - name: root
        hostPath:
          path: /
      - name: varlib
        hostPath:
          path: /var/lib/filebeat
          type: DirectoryOrCreate
      - name: varlog
        hostPath:
          path: /var/log/filebeat
          type: DirectoryOrCreate
      - name: localtime
        hostPath:
          path: /etc/localtime

// :wq保存退出

// 部署
$ kubectl create ns test-basic-log=pilot

$ kubectl apply -f log-pilot.yaml -n test-basic-log=pilot

```

部署完成后，查看已存在的pod信息，如下：

```
$ kubectl get pod -n test-basic-log-pilot
NAME              READY   STATUS    RESTARTS   AGE
log-pilot-6rw6z   1/1     Running   0          22h
log-pilot-bkb9w   1/1     Running   1          22h
log-pilot-jcp2d   1/1     Running   0          22h
log-pilot-kd79n   1/1     Running   2          22h
log-pilot-lpbv7   1/1     Running   0          22h
log-pilot-nwgxn   1/1     Running   1          22h

```

所有node机器上都会部署一个log-pilot的pod，用来采集该节点的相应服务的信息。

### 2. 服务联调

对服务进行改造，还是以数据字典为例，对数据字典部署的k8s配置文件yaml进行配置：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: test-data-dictionary
  namespace: test-all-service
  labels:
    app: test-data-dictionary
spec:
  ports:
    - port: ${SERVER_PORT}
      name: tcp
      targetPort: ${SERVER_PORT}
  selector:
    app: test-data-dictionary
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-data-dictionary
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
      app: test-data-dictionary
  template:
    metadata:
      labels:
        app: test-data-dictionary
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
                        - app-test-data-dictionary
              weight: 1
      containers:
        - name: data-dictionary
          image: ${DOCKER_HUB}/data-dictionary-k8s:$BUILD_NUMBER
          imagePullPolicy: Always
          lifecycle:
            preStop:
              httpGet:
                port: ${SERVER_PORT}
                path: /spring/shutdown
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: ${SERVER_PORT}
            initialDelaySeconds: 180
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: ${SERVER_PORT}
            initialDelaySeconds: 180
            periodSeconds: 5
            timeoutSeconds: 10
            successThreshold: 1
            failureThreshold: 5
          resources:
            requests:
              memory: 500Mi
            limits:
              memory: 1Gi
          ports:
            - containerPort: ${SERVER_PORT}
          env:
            - name: IP_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            # 配置环境变量，类似服务docker启动项中设置的那样
            - name: aliyun_logs_dict
              value: stdout
            # 配置环境变量，配置tag
            - name: aliyun_logs_dict_tags
              value: topic=test,env=test,service=test-data-dict
            - name: NACOS_IP
              value: ${NACOS_IP_PORT}
            - name: NACOS_NAMESPACE
              value: ${NACOS_NAMESPACE}
            - name: SKYWALKING_NAMESPACE
              value: test-k8s-test
            - name: SKYWALKING_TARGET_SERVICE_NAME
              value: test-data-dic
            - name: SKYWALKING_IP_PORT
              value: ${SKYWALKING_IP_PORT}
            - name: CHANNEL
              value: ${CHANNEL}


```

配置完成后，利用CI/CD流水线进行构建，推送到k8s运行环境中。

注意配置的时候**env=test**，通过该tag确定日志信息来自k8s中的服务，区分于开发环境中的日志。

最后查看日志信息上报，如下图：

![](kibana日志信息上报-筛选.png)

![](kibana日志信息上报-结果.png)

## 总结

## 参考文档

* log-pilot运行说明：https://github.com/AliyunContainerService/log-pilot/blob/master/docs/filebeat/docs.md
* ELK安装部署文档：http://docs.bigops.com/an-zhuang/an-zhuang-elk.html
* https://blog.csdn.net/ltliyue/article/details/105121849
* kafka安装测试：https://segmentfault.com/a/1190000012990954
* kafka使用：https://blog.csdn.net/weixin_38004638/article/details/91975123
* docker容器执行参数：https://blog.csdn.net/qq_19381989/article/details/102781663，https://blog.csdn.net/nzjdsds/article/details/81981732
* log-pilot日志收集：https://github.com/AliyunContainerService/log-pilot/issues/121#issuecomment-638047973