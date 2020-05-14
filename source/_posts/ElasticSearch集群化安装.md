---
title: ElasticSearch集群化安装
top: false
cover: false
toc: true
mathjax: true
date: 2020-05-14 09:40:47
password:
summary:
tags:
categories:
---

# ElasticSearch集群化安装

## 机器准备

| 主机地址 | 节点功能 |
|----------|---------|
| 192.168.232.193 | 从节点 | 
| 192.168.232.194 | 主节点 | 
| 192.168.232.191 | 从节点 | 

## 安装工具版本

* 操作系统：centos7.6

* elasticsearch: 7.6.2

## 正式安装设置

### 0. 解压并转移到/usr/local路径下

```
$ cd ~/skywalking-install/

$ tar -zxvf elasticsearch-7.6.2-linux-x86_64.tar.gz

$ sudo mv elasticsearch-7.6.2 /usr/local/

```

### 1. 开放端口--9200、9300

```
$ sudo firewall-cmd --zone=public --add-port=9200/tcp --permanent
success
$ sudo firewall-cmd --zone=public --add-port=9300/tcp --permanent
success
$ sudo firewall-cmd --zone=public --add-port=9300/udp --permanent
success
$ sudo firewall-cmd --zone=public --add-port=9200/udp --permanent
success
$ sudo firewall-cmd --reload
success

```

### 2. 创建用户
```
$ sudo groupadd elasticsearch
$ sudo useradd elasticsearch -g elasticsearch -p elasticsearch
$ sudo passwd elasticsearch
Changing password for user elasticsearch.
New password: 
Retype new password:

// 密码为ht@es123

// 修改目录归属

$ cd /usr/local/
$ ls
bin  elasticsearch-7.6.2  etc  games  include  lib  lib64  libexec  sbin  share  src
$ sudo chown -R elasticsearch:elasticsearch elasticsearch-7.6.2/
$ cd elasticsearch-7.6.2/
$ ls -al
total 548
drwxr-xr-x.  9 elasticsearch elasticsearch    155 Mar 26 14:36 .
drwxr-xr-x. 13 root          root             158 Apr 20 11:17 ..
drwxr-xr-x.  2 elasticsearch elasticsearch   4096 Apr 20 11:16 bin
drwxr-xr-x.  2 elasticsearch elasticsearch    148 Mar 26 14:36 config
drwxr-xr-x.  9 elasticsearch elasticsearch    107 Mar 26 14:36 jdk
drwxr-xr-x.  3 elasticsearch elasticsearch   4096 Mar 26 14:36 lib
-rw-r--r--.  1 elasticsearch elasticsearch  13675 Mar 26 14:28 LICENSE.txt
drwxr-xr-x.  2 elasticsearch elasticsearch      6 Mar 26 14:36 logs
drwxr-xr-x. 38 elasticsearch elasticsearch   4096 Mar 26 14:37 modules
-rw-r--r--.  1 elasticsearch elasticsearch 523209 Mar 26 14:36 NOTICE.txt
drwxr-xr-x.  2 elasticsearch elasticsearch      6 Mar 26 14:36 plugins
-rw-r--r--.  1 elasticsearch elasticsearch   8164 Mar 26 14:28 README.asciidoc

```

### 3. 服务器参数设定

3.1 修改jvm

```
// 切换用户，确保执行
$ su elasticsearch

// 创建数据目录
$ mkdir data

$ vim config/jvm.options
// 修改下面两行信息
-Xms10g
-Xmx10g

```
3.2 修改vm内存权限

```
$ su centos

$ sudo vim /etc/sysctl.conf
// 追加以下信息
vm.max_map_count=262144

$ sudo sysctl -p
// 执行后输出信息
vm.max_map_count=262144

```

3.3 修改切换到root用户修改配置limits.conf，添加文件打开数的配置

```
$ sudo vim /etc/security/limits.conf
// 追加以下信息
* hard nofile 65536
* soft nofile 65536

```

后续切换到es的用户。进行启动

### 4. 配置文件设定

切换到192.168.232.194机器上进行配置：

```
$ cd /usr/local/

$ su elasticsearch

$ cd elasticsearch/

$ sudo vim config/elasticsearch.yml

cluster.name: es-master

node.name: node-1
node.attr.rack: r1

path.data: /usr/local/elasticsearch-7.6.2/data

path.logs: /usr/local/elasticsearch-7.6.2/logs

#bootstrap.memory_lock: true

network.host: 192.168.232.194
http.port: 9200

discovery.seed_hosts: ["192.168.232.191:9300", "192.168.232.193:9300"]

#index.number_of_shards: 3
#index.number_of_replicas: 1

transport.tcp.port: 9300
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]

discovery.zen.minimum_master_nodes: 2 
gateway.recover_after_nodes: 3

http.cors.enabled: true
http.cors.allow-origin: "*"


```

配置名词解释
 cluster.name: es-cluster #指定es集群名
 node.name: xxxx #指定当前es节点名
 node.data: false #非数据节点
 node.master: false #非master节点
 node.attr.rack: r1 #自定义的属性,这是官方文档中自带的
 bootstrap.memory_lock: true #开启启动es时锁定内存
 network.host: 172.17.0.5 #当前节点的ip地址
 http.port: 9200 #设置当前节点占用的端口号，默认9200
 discovery.seed_hosts: ["172.17.0.4:9300","172.17.0.2:9300"] #启动当前es节点时会去这个ip列表中去发现其他节点，此处不需配置自己节点的ip,这里支持ip和ip:port形式,不加端口号默认使用ip:9300去发现节点
 cluster.initial_master_nodes: ["node-1", "node-2", "node-3"] #可作为master节点初始的节点名称,tribe-node不在此列
 gateway.recover_after_nodes: 2 #设置集群中N个节点启动时进行数据恢复，默认为1。可选
 path.data: /path/to/path  #数据保存目录
 path.logs: /path/to/path #日志保存目录
 transport.tcp.port: 9300 #设置集群节点发现的端口
 index.number_of_replicas: 1 是数据备份数，如果只有一台机器，设置为0
 index.number_of_shards: 3  是数据分片数，默认为5，有时候设置为3
 discovery.zen.minimum_master_nodes: 2 # 通过配置大多数节点(节点总数/ 2 + 1)来防止脑裂
 gateway.recover_after_nodes: 3 # 在一个完整的集群重新启动到N个节点开始之前，阻止初始恢复
 http.cors.enabled: true   # 允许跨域的设置
 http.cors.allow-origin: "*" # 允许跨域的设置

这里不去配置bootstrap.memory_lock，以及index.number_of_replicas、index.number_of_shards这三个选项，配置后导致ES无法启动，有异常日志！

切换到192.168.232.193机器上配置为：

```
cluster.name: es-master

node.name: node-2
node.attr.rack: r1

path.data: /usr/local/elasticsearch-7.6.2/data
path.logs: /usr/local/elasticsearch-7.6.2/logs

#bootstrap.memory_lock: true

network.host: 192.168.232.193

http.port: 9200

#index.number_of_shards: 3
#index.number_of_replicas: 1

discovery.seed_hosts: ["192.168.232.194:9300", "192.168.232.191:9300"]

cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]

discovery.zen.minimum_master_nodes: 2
gateway.recover_after_nodes: 3

```
切换到192.168.232.191机器上配置为：

```
cluster.name: es-master

node.name: node-3
node.attr.rack: r1

path.data: /usr/local/elasticsearch-7.6.2/data
path.logs: /usr/local/elasticsearch-7.6.2/logs

#bootstrap.memory_lock: true

network.host: 192.168.232.191

http.port: 9200

#index.number_of_shards: 3
#index.number_of_replicas: 1

discovery.seed_hosts: ["192.168.232.194:9300", "192.168.232.193:9300"]

cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]

discovery.zen.minimum_master_nodes: 2
gateway.recover_after_nodes: 3

```

### 5. 启动不同机器的ES

在三台机器上同时进行启动

```
$ su elasticsearch

$ cd /usr/local/elasticsearch-7.6.2/bin

$ ./elasticsearch -d

```

最后进行测试，访问http://192.168.232.194:9200/_cat/nodes?pretty地址，查看es的信息。输出如下：

```
192.168.232.193 8 64 3 0.51 0.34 0.38 dilm - node-2
192.168.232.194 3 51 0 0.00 0.09 0.16 dilm * node-1
192.168.232.191 3 59 1 0.00 0.07 0.11 dilm - node-3

```

看到主节点确定在192.168.232.194机器上，说明选举成功，集群已经启动！

针对各台机器进行验证，如下：

```
$ curl http://192.168.232.191:9200/_cat/health?v
epoch      timestamp cluster   status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1587395840 15:17:20  es-master green           3         3      0   0    0    0        0             0                  -                100.0%

$ curl http://192.168.232.193:9200/_cat/health?v
epoch      timestamp cluster   status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1587395840 15:17:20  es-master green           3         3      0   0    0    0        0             0                  -                100.0%

$ curl http://192.168.232.194:9200/_cat/health?v
epoch      timestamp cluster   status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1587395840 15:17:20  es-master green           3         3      0   0    0    0        0             0                  -                100.0%

```

## 必要插件安装

1. 安装head插件

可以在开发机上，拉取head前端代码，并运行，操作如下：

```
$ git clone git://github.com/mobz/elasticsearch-head.git

$ cd elasticsearch-head

$ npm install --registry https://registry.npm.taobao.org

$ npm run start

```

安装完成后，从浏览器访问http://localhost:9100/即可。如下图：

![](首页.png)

填写主节点地址到输入框中，即可进行访问，效果如下：

![](登陆后页面.png)

注意：ES节点一定要配置允许跨域访问，如下：

```

http.cors.enabled: true
http.cors.allow-origin: "*"

```

否则前端无法进行访问。

2. 安装ik分词器



## 总结

安装并不困难，但是在访问的时候需要配置跨域访问的内容。
