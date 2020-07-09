---
title: FastDFS搭建记录
top: false
cover: false
toc: true
mathjax: true
date: 2020-06-24 22:12:29
password:
summary:
tags:
categories:
---

# FastDFS搭建记录

## 概述

fastDFS 是以C语言开发的一项开源轻量级分布式文件系统，他对文件进行管理，主要功能有：文件存储，文件同步，文件访问（文件上传/下载），特别适合以文件为载体的在线服务，如图片网站，视频网站等。

利用FastDFS搭建分布式文件系统，基于客户端/服务器的文件存储系统，其对等特性允许一些系统扮演客户端和服务器的双重角色，可供多个用户访问的服务器，比如，用户可以“发表”一个允许其他客户机访问的目录，一旦被访问，这个目录对客户机来说就像使用本地驱动器一样。

FastDFS由跟踪服务器(Tracker Server)、存储服务器(Storage Server)和客户端(Client)构成。

* Tracker server 追踪服务器

追踪服务器负责接收客户端的请求，选择合适的组合storage server ，tracker server 与 storage server之间也会用心跳机制来检测对方是否活着。Tracker需要管理的信息也都放在内存中，并且里面所有的Tracker都是对等的（每个节点地位相等），易扩展。客户端访问集群的时候会随机分配一个Tracker来和客户端交互。

* Storage server 储存服务器

实际存储数据，分成若干个组（group），实际traker就是管理的storage中的组，而组内机器中则存储数据，group可以隔离不同应用的数据，不同的应用的数据放在不同group里面，

- 主从型分布式存储，存储空间方便拓展，有利于实现海量存储
- fastDFS对文件内容做hash处理，避免出现重复文件
- 然后fastDFS结合Nginx集成, 提供网站效率

* 客户端Client

主要是上传下载数据的服务器，也就是我们自己的项目所部署在的服务器。每个客户端服务器都需要安装Nginx，利用nginx访问文件数据。

## docker方式安装

首先查找fastdfs的安装包，并进行拉取：

```
$ docker search fastdfs
NAME                         DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
season/fastdfs               FastDFS                                         64
luhuiguo/fastdfs             FastDFS is an open source high performance d…   25                                      [OK]
ygqygq2/fastdfs-nginx        整合了nginx的fastdfs                                20                                      [OK]
morunchang/fastdfs           A FastDFS image                                 19
delron/fastdfs                                                               12
moocu/fastdfs                fastdfs5.11                                     9
qbanxiaoli/fastdfs           FastDFS+FastDHT单机版                              8                                       [OK]
ecarpo/fastdfs-storage                                                       4
ecarpo/fastdfs                                                               3
lionheart/fastdfs-tracker    just have a try on autobuilded -_-#             3                                       [OK]
imlzw/fastdfs-tracker        fastdfs的tracker服务                               3                                       [OK]
imlzw/fastdfs-storage-dht    fastdfs的storage服务,并且集成了fastdht的服务…              2                                       [OK]
manuku/fastdfs-fastdht       fastdfs fastdht                                 2                                       [OK]
john123951/fastdfs_storage   fastdfs storage                                 1                                       [OK]
manuku/fastdfs-tracker       fastdfs tracker                                 1                                       [OK]
lionheart/fastdfs_tracker    fastdfs file system‘s tracker node              1
basemall/fastdfs-nginx       fastdfs with nginx                              1                                       [OK]
appcrash/fastdfs_nginx       fastdfs with nginx                              1
leaon/fastdfs                fastdfs                                         1
evan1120/fastdfs_tracker     The fastdfs tracker docker image, only conta…   1                                       [OK]
evan1120/fastdfs_storage     The fastdfs storage image                       1                                       [OK]
tsl0922/fastdfs              FastDFS is an open source high performance d…   0                                       [OK]
lovetutu/fastdfs_fastdht     fastdfs_fastdht_docker                          0
mypjb/fastdfs                this is a fastdfs docker project                0                                       [OK]
manuku/fastdfs-storage-dht   fastdfs storage dht                             0                                       [OK]

$ docker pull delron/fastdfs

$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hello-world         latest              bf756fb1ae65        6 months ago        13.3kB
delron/fastdfs      latest              8487e86fc6ee        2 years ago         464MB

```

拉取完毕后开始进行分步启动，首先构建tracker容器并启动，然后构建storage容器并启动。

```
$ docker run -d --network=host --name tracker -v /var/fdfs/tracker:/var/fdfs delron/fastdfs tracker

$ docker run -d --network=host --name storage -e TRACKER_SERVER=192.168.122.236:22122 -v /var/fdfs/storage:/var/fdfs -e GROUP_NAME=group1 delron/fastdfs storage

$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                   PORTS               NAMES
c49cea16c916        delron/fastdfs      "/usr/bin/start1.sh …"   2 weeks ago         Up 21 hours                                  storage
5d2c2c7eccda        delron/fastdfs      "/usr/bin/start1.sh …"   2 weeks ago         Up 21 hours                                  tracker

```

下面主要是对两个镜像中的配置信息进行修改

```
$ docker exec -it storage /bin/bash

bash4.2-# cat /etc/fdfs/storage.conf

# is this config file disabled
# false for enabled
# true for disabled
disabled=false

# the name of the group this storage server belongs to
#
# comment or remove this item for fetching from tracker server,
# in this case, use_storage_id must set to true in tracker.conf,
# and storage_ids.conf must be configed correctly.
group_name=group1

# bind an address of this host
# empty for bind all addresses of this host
bind_addr=

# if bind an address of this host when connect to other servers
# (this storage server as a client)
# true for binding the address configed by above parameter: "bind_addr"
# false for binding any address of this host
client_bind=true

# the storage server port
# 注意第一个端口号，23000为storage服务所运行的端口号
port=23000      

# connect timeout in seconds
# default value is 30s
connect_timeout=30

# network timeout in seconds
# default value is 30s
network_timeout=60

# heart beat interval in seconds
heart_beat_interval=30

# disk usage report interval in seconds
stat_report_interval=60

# the base path to store data and log files
base_path=/var/fdfs

# max concurrent connections the server supported
# default value is 256
# more max_connections means more memory will be used
max_connections=256

# the buff size to recv / send data
# this parameter must more than 8KB
# default value is 64KB
# since V2.00
buff_size = 256KB

# accept thread count
# default value is 1
# since V4.07
accept_threads=1

# work thread count, should <= max_connections
# work thread deal network io
# default value is 4
# since V2.00
work_threads=4

# if disk read / write separated
##  false for mixed read and write
##  true for separated read and write
# default value is true
# since V2.00
disk_rw_separated = true

# disk reader thread count per store base path
# for mixed read / write, this parameter can be 0
# default value is 1
# since V2.00
disk_reader_threads = 1

# disk writer thread count per store base path
# for mixed read / write, this parameter can be 0
# default value is 1
# since V2.00
disk_writer_threads = 1

# when no entry to sync, try read binlog again after X milliseconds
# must > 0, default value is 200ms
sync_wait_msec=50

# after sync a file, usleep milliseconds
# 0 for sync successively (never call usleep)
sync_interval=0

# storage sync start time of a day, time format: Hour:Minute
# Hour from 0 to 23, Minute from 0 to 59
sync_start_time=00:00

# storage sync end time of a day, time format: Hour:Minute
# Hour from 0 to 23, Minute from 0 to 59
sync_end_time=23:59

# write to the mark file after sync N files
# default value is 500
write_mark_file_freq=500

# path(disk or mount point) count, default value is 1
store_path_count=1

# store_path#, based 0, if store_path0 not exists, it's value is base_path
# the paths must be exist
store_path0=/var/fdfs
#store_path1=/var/fdfs2

# subdir_count  * subdir_count directories will be auto created under each
# store_path (disk), value can be 1 to 256, default value is 256
subdir_count_per_path=256

# tracker_server can ocur more than once, and tracker_server format is
#  "host:port", host can be hostname or ip address
# 注意第三个端口号，22122是tracker服务运行的端口
tracker_server=192.168.209.121:22122

#standard log level as syslog, case insensitive, value list:
### emerg for emergency
### alert
### crit for critical
### error
### warn for warning
### notice
### info
### debug
log_level=info

#unix group name to run this program,
#not set (empty) means run by the group of current user
run_by_group=

#unix username to run this program,
#not set (empty) means run by current user
run_by_user=

# allow_hosts can ocur more than once, host can be hostname or ip address,
# "*" (only one asterisk) means match all ip addresses
# we can use CIDR ips like 192.168.5.64/26
# and also use range like these: 192.168.1.[0-254] and host[01-08,20-25].domain.com
# for example:
# allow_hosts=192.168.1.[1-15,20]
# allow_hosts=host[01-08,20-25].domain.com
# allow_hosts=192.168.5.64/26
allow_hosts=*

# the mode of the files distributed to the data path
# 0: round robin(default)
# 1: random, distributted by hash code
file_distribute_path_mode=0

# valid when file_distribute_to_path is set to 0 (round robin),
# when the written file count reaches this number, then rotate to next path
# default value is 100
file_distribute_rotate_count=100

# call fsync to disk when write big file
# 0: never call fsync
# other: call fsync when written bytes >= this bytes
# default value is 0 (never call fsync)
fsync_after_written_bytes=0

# sync log buff to disk every interval seconds
# must > 0, default value is 10 seconds
sync_log_buff_interval=10

# sync binlog buff / cache to disk every interval seconds
# default value is 60 seconds
sync_binlog_buff_interval=10

# sync storage stat info to disk every interval seconds
# default value is 300 seconds
sync_stat_file_interval=300

# thread stack size, should >= 512KB
# default value is 512KB
thread_stack_size=512KB

# the priority as a source server for uploading file.
# the lower this value, the higher its uploading priority.
# default value is 10
upload_priority=10

# the NIC alias prefix, such as eth in Linux, you can see it by ifconfig -a
# multi aliases split by comma. empty value means auto set by OS type
# default values is empty
if_alias_prefix=

# if check file duplicate, when set to true, use FastDHT to store file indexes
# 1 or yes: need check
# 0 or no: do not check
# default value is 0
check_file_duplicate=0

# file signature method for check file duplicate
## hash: four 32 bits hash code
## md5: MD5 signature
# default value is hash
# since V4.01
file_signature_method=hash

# namespace for storing file indexes (key-value pairs)
# this item must be set when check_file_duplicate is true / on
key_namespace=FastDFS

# set keep_alive to 1 to enable persistent connection with FastDHT servers
# default value is 0 (short connection)
keep_alive=0

# you can use "#include filename" (not include double quotes) directive to
# load FastDHT server list, when the filename is a relative path such as
# pure filename, the base path is the base path of current/this config file.
# must set FastDHT server list when check_file_duplicate is true / on
# please see INSTALL of FastDHT for detail
##include /home/yuqing/fastdht/conf/fdht_servers.conf

# if log to access log
# default value is false
# since V4.00
use_access_log = false

# if rotate the access log every day
# default value is false
# since V4.00
rotate_access_log = false

# rotate access log time base, time format: Hour:Minute
# Hour from 0 to 23, Minute from 0 to 59
# default value is 00:00
# since V4.00
access_log_rotate_time=00:00

# if rotate the error log every day
# default value is false
# since V4.02
rotate_error_log = false

# rotate error log time base, time format: Hour:Minute
# Hour from 0 to 23, Minute from 0 to 59
# default value is 00:00
# since V4.02
error_log_rotate_time=00:00

# rotate access log when the log file exceeds this size
# 0 means never rotates log file by log file size
# default value is 0
# since V4.02
rotate_access_log_size = 0

# rotate error log when the log file exceeds this size
# 0 means never rotates log file by log file size
# default value is 0
# since V4.02
rotate_error_log_size = 0

# keep days of the log files
# 0 means do not delete old log files
# default value is 0
log_file_keep_days = 0

# if skip the invalid record when sync file
# default value is false
# since V4.02
file_sync_skip_invalid_record=false

# if use connection pool
# default value is false
# since V4.05
use_connection_pool = false

# connections whose the idle time exceeds this time will be closed
# unit: second
# default value is 3600
# since V4.05
connection_pool_max_idle_time = 3600

# use the ip address of this storage server if domain_name is empty,
# else this domain name will ocur in the url redirected by the tracker server
http.domain_name=

# the port of the web server on this storage server
# 注意第二个端口号，8888为文件访问运行的端口信息
http.server_port=8888

-----------------分割线----------------

bash4.2-# cat /etc/fdfs/storage.conf

# is this config file disabled
# false for enabled
# true for disabled
disabled=false

# bind an address of this host
# empty for bind all addresses of this host
bind_addr=

# the tracker server port
port=22122

# connect timeout in seconds
# default value is 30s
connect_timeout=30

# network timeout in seconds
# default value is 30s
network_timeout=60

# the base path to store data and log files
base_path=/var/fdfs

# max concurrent connections this server supported
max_connections=256

# accept thread count
# default value is 1
# since V4.07
accept_threads=1

# work thread count, should <= max_connections
# default value is 4
# since V2.00
work_threads=4

# min buff size
# default value 8KB
min_buff_size = 8KB

# max buff size
# default value 128KB
max_buff_size = 128KB

# the method of selecting group to upload files
# 0: round robin
# 1: specify group
# 2: load balance, select the max free space group to upload file
store_lookup=2

# which group to upload file
# when store_lookup set to 1, must set store_group to the group name
store_group=group2

# which storage server to upload file
# 0: round robin (default)
# 1: the first server order by ip address
# 2: the first server order by priority (the minimal)
store_server=0

# which path(means disk or mount point) of the storage server to upload file
# 0: round robin
# 2: load balance, select the max free space path to upload file
store_path=0

# which storage server to download file
# 0: round robin (default)
# 1: the source storage server which the current file uploaded to
download_server=0

# reserved storage space for system or other applications.
# if the free(available) space of any stoarge server in
# a group <= reserved_storage_space,
# no file can be uploaded to this group.
# bytes unit can be one of follows:
### G or g for gigabyte(GB)
### M or m for megabyte(MB)
### K or k for kilobyte(KB)
### no unit for byte(B)
### XX.XX% as ratio such as reserved_storage_space = 10%
reserved_storage_space = 10%

#standard log level as syslog, case insensitive, value list:
### emerg for emergency
### alert
### crit for critical
### error
### warn for warning
### notice
### info
### debug
log_level=info

#unix group name to run this program,
#not set (empty) means run by the group of current user
run_by_group=

#unix username to run this program,
#not set (empty) means run by current user
run_by_user=

# allow_hosts can ocur more than once, host can be hostname or ip address,
# "*" (only one asterisk) means match all ip addresses
# we can use CIDR ips like 192.168.5.64/26
# and also use range like these: 192.168.1.[0-254] and host[01-08,20-25].domain.com
# for example:
# allow_hosts=192.168.1.[1-15,20]
# allow_hosts=host[01-08,20-25].domain.com
# allow_hosts=192.168.5.64/26
allow_hosts=*

# sync log buff to disk every interval seconds
# default value is 10 seconds
sync_log_buff_interval = 10

# check storage server alive interval seconds
check_active_interval = 120

# thread stack size, should >= 64KB
# default value is 64KB
thread_stack_size = 64KB

# auto adjust when the ip address of the storage server changed
# default value is true
storage_ip_changed_auto_adjust = true

# storage sync file max delay seconds
# default value is 86400 seconds (one day)
# since V2.00
storage_sync_file_max_delay = 86400

# the max time of storage sync a file
# default value is 300 seconds
# since V2.00
storage_sync_file_max_time = 300

# if use a trunk file to store several small files
# default value is false
# since V3.00
use_trunk_file = false

# the min slot size, should <= 4KB
# default value is 256 bytes
# since V3.00
slot_min_size = 256

# the max slot size, should > slot_min_size
# store the upload file to trunk file when it's size <=  this value
# default value is 16MB
# since V3.00
slot_max_size = 16MB

# the trunk file size, should >= 4MB
# default value is 64MB
# since V3.00
trunk_file_size = 64MB

# if create trunk file advancely
# default value is false
# since V3.06
trunk_create_file_advance = false

# the time base to create trunk file
# the time format: HH:MM
# default value is 02:00
# since V3.06
trunk_create_file_time_base = 02:00

# the interval of create trunk file, unit: second
# default value is 38400 (one day)
# since V3.06
trunk_create_file_interval = 86400

# the threshold to create trunk file
# when the free trunk file size less than the threshold, will create
# the trunk files
# default value is 0
# since V3.06
trunk_create_file_space_threshold = 20G

# if check trunk space occupying when loading trunk free spaces
# the occupied spaces will be ignored
# default value is false
# since V3.09
# NOTICE: set this parameter to true will slow the loading of trunk spaces
# when startup. you should set this parameter to true when neccessary.
trunk_init_check_occupying = false

# if ignore storage_trunk.dat, reload from trunk binlog
# default value is false
# since V3.10
# set to true once for version upgrade when your version less than V3.10
trunk_init_reload_from_binlog = false

# the min interval for compressing the trunk binlog file
# unit: second
# default value is 0, 0 means never compress
# FastDFS compress the trunk binlog when trunk init and trunk destroy
# recommand to set this parameter to 86400 (one day)
# since V5.01
trunk_compress_binlog_min_interval = 0

# if use storage ID instead of IP address
# default value is false
# since V4.00
use_storage_id = false

# specify storage ids filename, can use relative or absolute path
# since V4.00
storage_ids_filename = storage_ids.conf

# id type of the storage server in the filename, values are:
## ip: the ip address of the storage server
## id: the server id of the storage server
# this paramter is valid only when use_storage_id set to true
# default value is ip
# since V4.03
id_type_in_filename = ip

# if store slave file use symbol link
# default value is false
# since V4.01
store_slave_file_use_link = false

# if rotate the error log every day
# default value is false
# since V4.02
rotate_error_log = false

# rotate error log time base, time format: Hour:Minute
# Hour from 0 to 23, Minute from 0 to 59
# default value is 00:00
# since V4.02
error_log_rotate_time=00:00

# rotate error log when the log file exceeds this size
# 0 means never rotates log file by log file size
# default value is 0
# since V4.02
rotate_error_log_size = 0

# keep days of the log files
# 0 means do not delete old log files
# default value is 0
log_file_keep_days = 0

# if use connection pool
# default value is false
# since V4.05
use_connection_pool = false

# connections whose the idle time exceeds this time will be closed
# unit: second
# default value is 3600
# since V4.05
connection_pool_max_idle_time = 3600

# HTTP port on this tracker server
http.server_port=8080

# check storage HTTP server alive interval seconds
# <= 0 for never check
# default value is 30
http.check_alive_interval=30

# check storage HTTP server alive type, values are:
#   tcp : connect to the storge server with HTTP port only,
#        do not request and get response
#   http: storage check alive url must return http status 200
# default value is tcp
http.check_alive_type=tcp

# check storage HTTP server alive uri/url
# NOTE: storage embed HTTP server support uri: /status.html
http.check_alive_uri=/status.htm

-------------------分割线----------------------------

bash4.2-# cat client.conf

# connect timeout in seconds
# default value is 30s
connect_timeout=30

# network timeout in seconds
# default value is 30s
network_timeout=60

# the base path to store log files
base_path=/var/fdfs

# tracker_server can ocur more than once, and tracker_server format is
#  "host:port", host can be hostname or ip address
tracker_server=192.168.0.197:22122

#standard log level as syslog, case insensitive, value list:
### emerg for emergency
### alert
### crit for critical
### error
### warn for warning
### notice
### info
### debug
log_level=info

# if use connection pool
# default value is false
# since V4.05
use_connection_pool = false

# connections whose the idle time exceeds this time will be closed
# unit: second
# default value is 3600
# since V4.05
connection_pool_max_idle_time = 3600

# if load FastDFS parameters from tracker server
# since V4.05
# default value is false
load_fdfs_parameters_from_tracker=false

# if use storage ID instead of IP address
# same as tracker.conf
# valid only when load_fdfs_parameters_from_tracker is false
# default value is false
# since V4.05
use_storage_id = false

# specify storage ids filename, can use relative or absolute path
# same as tracker.conf
# valid only when load_fdfs_parameters_from_tracker is false
# since V4.05
storage_ids_filename = storage_ids.conf


#HTTP settings
http.tracker_server_port=80

#use "#include" directive to include HTTP other settiongs
##include http.conf


```

查看并修改完上述三个文件，storage.conf、tracker.conf、client.conf，退出容器。

最后防火墙开启端口

```
$ sudo firewall-cmd --zone=public --add-port=8888/tcp --permanent
success
$ sudo firewall-cmd --zone=public --add-port=22122/tcp --permanent
success
$ sudo firewall-cmd --zone=public --add-port=23000/tcp --permanent
success
$ sudo firewall-cmd --reload
success

```

最终进行测试，用一张test.png图片，通过本地客户端模拟图片上传。

首先将图片传入到storage容器中，重新进入到storage容器中，进行上传操作:

```
$ docker cp test.png storage:/var/

$ docker exec -it storage /bin/bash

bash4.2-# /usr/bin/fdfs_upload_file /etc/fdfs/client.conf /var/test.png
group1/M00/00/00/CgBC7F78uTiAegOJAAC3s0pdQHQ707.png

```

上传操作后，返回一个地址信息，然后通过浏览器进行访问，如下：

![](./imgs/fastdfs访问效果.png)


## 编译安装

### 第一步：准备工作

1. libfastcommon

从 FastDFS 和 FastDHT 中提取出来的公共 C 函数库，基础环境

在安装 FastDFS 前需要先安装这个

下载地址址：https://github.com/happyfish100/libfastcommon/releases

2. FastDFS

* FastDFS 安装包

* 下载地址：https://github.com/happyfish100/fastdfs/releases

3. fastdfs-nginx-module

* 为了实现通过 HTTP 服务访问和下载 FastDFS 服务器中的文件

* 可以重定向文件链接到源服务器取文件，避免同一组 Storage 服务器同步延迟导致文件访问错误

* 下载地址：https://github.com/happyfish100/fastdfs-nginx-module/releases

本次采用的安装包版本如下

* libfastcommon ：1.0.43

* FastDFS ：6.06

* fastdfs-ninx-module ：1.22

* nginx:1.16.1

将以上安装包拷贝到 CentOS7 下的 /home/centos/FastDFS 目录下

### 第二步：安装libfastcommon

libfastcommon 安装依赖于 gcc 和 perl，故要先安装这两个

```
// 在线安装 gcc（安装系统时，选择开发工具安装和安全工具安装后，gcc自动安装）
$ sudo yum -y install make zlib zlib-devel gcc-c++ libtool  openssl openssl-devel

// 上传已经下载好的libfastcommon安装包

// 进入 root 目录下

$ cd /home/centos/FastDFS 

// 解压 libfastcommon 压缩包

$ tar zxvf libfastcommon-1.0.43.tar.gz 

//进入 libfastcommon 文件夹中，编译 libfastcommon 以及安装

$ cd ./libfastcommon-1.0.43

$ ./make.sh && sudo ./make.sh install

``` 

### 第三步：安装 FastDFS

```
$ cd /home/centos/FastDFS 

// 上传已经下载好的FastDFS安装包
// 解压 FastDFS 压缩包，编译以及安装
$ tar zxvf fastdfs-6.06.tar.gz 

$ cd ./fastdfs-6.06

$ ./make.sh && sudo ./make.sh install

// 查看是否已经安装
$ cd /usr/bin && ls |grep fdfs


// 配置 Tracker

// 创建 Tracker 的存储日志和数据的根目录
$ mkdir -p /home/centos/FastDFS/data/tracker

$ cd /etc/fdfs

$ sudo cp tracker.conf.sample tracker.conf

// 配置 tracker.conf

$ sudo vim tracker.conf

// 在这里，tracker.conf 只是修改一下 Tracker 存储日志和数据的路径
// 启用配置文件（默认为 false，表示启用配置文件）
disabled=false

// Tracker 服务端口（默认为 22122），不需要更改
port=22122

// 存储日志和数据的根目录
base_path=/home/centos/FastDFS/data/tracker

// :wq保存退出


// 配置 Storage

// 创建 Storage 的存储日志和数据的根目录
$ mkdir -p /home/centos/FastDFS/data/storage

$ cd /etc/fdfs

$ sudo cp storage.conf.sample storage.conf

// 配置 storage.conf

$ sudo vim storage.conf

// 在这里，storage.conf 只是修改一下 storage 存储日志和数据的路径
// 启用配置文件（默认为 false，表示启用配置文件）
disabled=false

// Storage 服务端口（默认为 23000）
port=23000

// 数据和日志文件存储根目录
base_path=/home/centos/FastDFS/data/storage

// 存储路径，访问时路径为 M00
// store_path1 则为 M01，以此递增到 M99（如果配置了多个存储目录的话，这里只指定 1 个）
store_path0=/home/centos/FastDFS/data/storage

// Tracker 服务器 IP 地址和端口，单机搭建时也不要写 127.0.0.1
// tracker_server 可以多次出现，如果有多个，则配置多个
tracker_server=192.168.122.240:22122

// 设置 HTTP 访问文件的端口。这个配置已经不用配置了，配置了也没什么用
// 这也是为何 Storage 服务器需要 Nginx 来提供 HTTP 访问的原因
http.server_port=8888

// 启动 Tracker 和 Storage 服务
// 启动 Tracker 服务
// 其它操作则把 start 改为 stop、restart、reload、status 即可。Storage 服务相同
$ sudo /etc/init.d/fdfs_trackerd start

// 启动 Storage 服务
$ sudo /etc/init.d/fdfs_storaged start

```

### 第四步：测试上传文件

```
# 修改 Tracker 服务器客户端配置文件

$ sudo cp /etc/fdfs/client.conf.sample /etc/fdfs/client.conf

$ sudo vim /etc/fdfs/client.conf

// client.conf 中修改 base_path 和 Tracker 服务器的 IP 地址与端口号即可
// 存储日志文件的基本路径
base_path=/home/centos/FastDFS/data/tracker

// Tracker 服务器 IP 地址与端口号
tracker_server=192.168.122.240:22122


// 拷贝一张图片到 /home/centos 目录下
// 通过命令操作，存储到 FastDFS 服务器中

$ sudo /usr/bin/fdfs_upload_file /etc/fdfs/client.conf /home/centos/test.png
group1/M00/00/00/wKjlpltF-K-AZQQsAABhhboA1Kk469.png

// 当返回文件 ID 号，如 group1/M00/00/00/wKjlpltF-K-AZQQsAABhhboA1Kk469.png 则表示上传成功

```

### 第五步：设置开机启动

```
$ sudo vim /etc/rc.d/rc.local
// 文件最后进行追加操作
/etc/init.d/fdfs_trackerd start

/etc/init.d/fdfs_storaged start

```

### 第六步：编译安装nginx

准备文件，下载地址：http://nginx.org/en/download.html，选择1.16.1版本进行下载。

将以上安装包拷贝到 CentOS7 下的 /home/centos/FastDFS/ 目录下，将fastdfs-nginx-module的压缩包同样拷贝到这个目录下。

安装与配置 Nginx

```

// 安装依赖信息
$ sudo rpm -ivh pcre-devel-8.32-17.el7.x86_64.rpm zlib-devel-1.2.7-18.el7.x86_64.rpm

$ cd /home/centos/FastDFS/

$ tar -zxvf nginx-1.16.1.tar.gz 

// 找到nginx模块
$ cd /home/centos/FastDFS/

$ tar -zxvf fastdfs-nginx-module-1.22.tar.gz

$ cd fastdfs-nginx-module-1.22/src

$ vim config
// 增加以下：
:%s+/usr/local+/usr

// 替换/usr目录为/usr/local目录
// :wq保存退出

$ cd /usr/local/nginx-1.16.1/

// 给 Nginx 添加 fastdfs-nginx-module 模块
$ ./configure --add-module=/home/centos/FastDFS/fastdfs-nginx-module-1.22/src --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module --with-http_flv_module --with-http_gzip_static_module
 
$ make && sudo make install

```

fastdfs-nginx-module 和 FastDFS 配置文件修改

```
# 复制 FastDFS 的部分配置文件到 /etc/fdfs

$ cd /home/centos/FastDFS/fastdfs-nginx-module-1.22

$ sudo cp http.conf mime.types /etc/fdfs/

$ cd /home/centos/FastDFS/data/storage

$ ln -s /home/centos/FastDFS/data/storage/data/ /home/centos/FastDFS/data/storage/data/M00

// 复制 fastdfs-nginx-module 源码中的配置文件到  /etc/fdfs 中
$ sudo cp /usr/local/fastdfs-nginx-module-1.22/src/mod_fastdfs.conf /etc/fdfs/

$ sudo vim /etc/fdfs/mod_fastdfs.conf
// mod_fastdfs.conf 配置如下
// Tracker 服务器IP和端口修改

tracker_server=192.168.122.240:22122

// url 中是否包含 group 名称，改为 true，包含 group
url_have_group_name = true

// 配置 Storage 信息，修改 store_path0 的信息
store_path0=/home/centos/FastDFS/data/storage

// 其它的一般默认即可，例如
// :wq保存退出

// 配置 Nginx

$ sudo vim /usr/local/nginx/conf/nginx.conf
// 添加如下配置
  server {
        listen       8888;
        server_name  192.168.122.240;
        location ~/group([0-9])/M00 {
            root  /home/centos/FastDFS/data/storage/data;
            ngx_fastdfs_module;
        }
    }

// :wq保存退出

// 启动 Nginx
$ sudo /usr/local/nginx/sbin/nginx

```

测试图片上传：

```
$ vim /etc/fdfs/client.conf
// 修改下面三个值信息
base_path = /home/centos/FastDFS/data/tracker

tracker_server = 192.168.122.240:22122

http.tracker_server_port = 8888

// :wq保存

// 上传图片
$ sudo /usr/bin/fdfs_upload_file /etc/fdfs/client.conf /var/test.png
group1/M00/00/00/CgBC7F78uTiAegOJAAC3s0pdQHQ707.png

```

通过 HTTP 访问文件

根据 URL 访问之前上传的那张图片

http://192.168.122.240:8888/group1/M00/00/00/CgBC7F78uTiAegOJAAC3s0pdQHQ707.png

 

**注意**：

```
// 每次开机时，手动打开 Tracker 服务
$ sudo /etc/init.d/fdfs_trackerd start

// 打开 Storage 服务
$ sudo /etc/init.d/fdfs_storaged start

// 启动 Nginx
$ sudo /usr/local/nginx/sbin/nginx

```

## 参考链接

* docker安装参考链接：http://www.zuidaima.com/blog/4653383732808704.htm
* 原生方式安装参考链接：https://www.cnblogs.com/btxu/p/13061306.html