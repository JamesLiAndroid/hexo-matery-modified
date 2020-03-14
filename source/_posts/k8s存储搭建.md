---
title: k8s存储搭建
top: false
cover: false
toc: true
mathjax: true
date: 2020-03-11 20:10:07
password:
summary:
tags:
categories:
---

# 为k8s提供常见存储——单机nfs以及Rook+Ceph存储集群的安装和配置

## 环境介绍

* 操作系统:centos7.6
* 服务器列表

| 主机名称 | ip地址 | 作用|
|----------|--------|------|
| k8s-nfs-provider | 192.168.238.236 | nfs单机安装 |
| k8s-rke-node-start | 192.168.238.240 | rke管理机，管理k8s |
| k8s-node-01 | 192.168.238.232 | k8s集群机器1 |
| k8s-node-00 | 192.168.238.239 | k8s集群机器2 |
| k8s-node-02 | 192.168.238.241 | k8s集群机器3 |

* 工具及其版本列表

    1. nfs版本，遵循yum版本信息

    2. Rook：1.2.5

    3. Ceph: 14.2.7

* 前提：请参考**[Rancher集群化部署--在线安装.md]*文档，进行安装配置k8s和rancher。已经设置了240机器可以访问232、239、241三台机器，配置了ssh互通。

* 注意尽量给每台机器较大的硬盘信息，尽量不低于30GB。

## nfs的安装配置

### 1. nfs服务端的安装

登录192.168.238.236的机器，如下：

    $ ssh -p 15555 centos@192.168.238.236

首先检查并安装NFS，

    // 查看之前是否安装
    $ sudo rpm -aq nfs-utils rpcbind

    // 更新源信息
    $ sudo yum update

    // 安装
    $ sudo yum install  nfs-utils rpcbind -y

其次启动RPC服务和NFS服务并检查，

    // 启动rpc服务并设置开机启动
    $ sudo systemctl start rpcbind.service
    $ sudo systemctl enable rpcbind.service
    
    // 启动NFS服务并设置开机启动
    $ sudo systemctl start nfs.service
    $ sudo systemctl enable nfs.service

    // 检查已经设置开机启动的是否生效
    $ sudo systemctl list-unit-files|grep enabled

    // 查看rpc是否启动成功
    $ sudo netstat   -lntup|grep rpc
    // 输出以下信息
    tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      808/rpcbind
    tcp        0      0 0.0.0.0:20048           0.0.0.0:*               LISTEN      2281/rpc.mountd
    tcp        0      0 0.0.0.0:58642           0.0.0.0:*               LISTEN      2214/rpc.statd
    tcp6       0      0 :::37967                :::*                    LISTEN      2214/rpc.statd
    tcp6       0      0 :::111                  :::*                    LISTEN      808/rpcbind
    tcp6       0      0 :::20048                :::*                    LISTEN      2281/rpc.mountd
    udp        0      0 127.0.0.1:703           0.0.0.0:*                           2214/rpc.statd
    udp        0      0 0.0.0.0:970             0.0.0.0:*                           808/rpcbind
    udp        0      0 0.0.0.0:59572           0.0.0.0:*                           2214/rpc.statd
    udp        0      0 0.0.0.0:20048           0.0.0.0:*                           2281/rpc.mountd
    udp        0      0 0.0.0.0:111             0.0.0.0:*                           808/rpcbind
    udp6       0      0 :::970                  :::*                                808/rpcbind
    udp6       0      0 :::44176                :::*                                2214/rpc.statd
    udp6       0      0 :::20048                :::*                                2281/rpc.mountd
    udp6       0      0 :::111                  :::*                                808/rpcbind

    // 查看rpc监控信息
    $ sudo rpcinfo -p localhost
    // 输出以下信息
    program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
    100024    1   udp  59572  status
    100024    1   tcp  58642  status
    100005    1   udp  20048  mountd
    100005    1   tcp  20048  mountd
    100005    2   udp  20048  mountd
    100005    2   tcp  20048  mountd
    100005    3   udp  20048  mountd
    100005    3   tcp  20048  mountd
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
    100003    3   udp   2049  nfs
    100003    4   udp   2049  nfs
    100227    3   udp   2049  nfs_acl
    100021    1   udp  43983  nlockmgr
    100021    3   udp  43983  nlockmgr
    100021    4   udp  43983  nlockmgr
    100021    1   tcp  45822  nlockmgr
    100021    3   tcp  45822  nlockmgr
    100021    4   tcp  45822  nlockmgr

必须先启动rpcbind服务，后启动nfs服务，因为rpcbind服务启动之后，监控nfs服务

最后，设置共享文件夹，使nfs生效。操作如下：

    // 创建nfs-share文件夹
    $ sudo mkdir /nfs-share
    // 设置其可读写
    $ sudo chmod -R 777 /nfs-share

    // 配置/etc/exports
    $ sudo su
    # cat  >>/etc/exports<<EOF
    /data 192.168.238.0/24(rw,sync,all_squash)  
    EOF
    // 配置信息说明
    //  /data这是需要共享的目录，192.168.238.0/24允许访问的客户端，这里表示整个网段的都可以访问，也可以指定单个地址，也可以用星号表示所有用户都可以访问。
    // rw可读可行，sync实时写的的磁盘，all_squash不管访问NFS Server共享目录的用户身份如何，它的权限都被压缩成匿名用户，同时它的UID和GID都会变成nfsnobody账号身份。在早期多个NFS客户端同时读写NFS Server数据时，这个参数很有用。
  
    // 重启nfs服务
    $ sudo systemctl reload nfs.service

    // 查看
    showmount -e localhost 
    //  输出 结果:
    Export list for localhost:
    /data 172.16.1.0/24

这样nfs服务端就已经安装完成了。这样该服务器的可用硬盘容量就是该nfs服务的可用容量了。

### 2. nfs客户端的安装配置

在所有目标机器（192.168.238.232，192.168.238.239，192.168.238.241）中进行操作。

首先检查并安装NFS，如下：

    // 查看之前是否安装
    $ sudo rpm -aq nfs-utils rpcbind

    // 更新源信息
    $ sudo yum update

    // 安装
    $ sudo yum install  nfs-utils rpcbind -y

其次启动RPC服务和NFS服务并检查，

    // 启动rpc服务并设置开机启动
    $ sudo systemctl start rpcbind.service
    $ sudo systemctl enable rpcbind.service

然后开始检查服务端的NFS，

    $ showmount -e 192.168.238.236
    // 出现下面结果，则成功：
    Export list for 192.168.238.236:
    /nfs-share 192.168.238.0/24

倒数第二步，对该磁盘进行挂载

    // 挂载
    // -t 挂载的类型，表示以nfs模式运行
    $ sudo mount -t nfs 192.168.238.236:/nfs-share /mnt   
    // 查看
    $ df -h
    // 在列表中能找到下面的信息
    192.168.238.236:/nfs-share   146G  1.7G  145G   2% /mnt

最后，把挂载点写入到开机自启动

    # echo "mount -t nfs 192.168.238.236:/nfs-share /mnt" >> /etc/rc.local

这样完成后，添加nfs存储已经成功。

### 3. 测试文件写入

例如我们在192.168.238.232机器上创建一个文件并写入信息，例如：

    [centos@k8s-cluster-node-02 /]$ cd /mnt
    [centos@k8s-cluster-node-02 mnt]$ touch nfs-test.txt
    [centos@k8s-cluster-node-02 mnt]$ echo "test nfs hello world" > nfs-test.txt
    [centos@k8s-cluster-node-02 mnt]$ ls
    nfs-test.txt

然后我们回到192.168.238.236机器上，打开文件夹查看：

    [centos@k8s-nfs-provider ~]$ cd /nfs-share/
    [centos@k8s-nfs-provider nfs-share]$ ls
    nfs-test.txt
    [centos@k8s-nfs-provider nfs-share]$ cat nfs-test.txt
    // 输出信息
    test nfs hello world

这样说明我们可以成功写入文件信息。

### 4. 添加nfs到rancher

rancher会在存储中，自动发现nfs存储，直接显示主机名称**k8s-nfs-1**。

后续添加存储，添加卷信息，都可以基于该nfs存储进行。

## Rook+Ceph进行集群存储调度管理

我们需要先登录到192.168.238.240机器上进行操作。如下：

    $ ssh -p 15555 centos@192.168.238.240

### 1. 第一种方式：使用helm安装rook-ceph

这里可以使用helm快速安装rook-ceph。操作如下

    // 创建rook-ceph-system命名空间
    $ kubectl create namespace rook-ceph-system

    // 添加rook安装仓库信息
    $ helm repo add rook-stable https://charts.rook.io/release

    // 更新仓库信息
    $ helm update

    // 安装rook-ceph管理工具
    $ helm install rook --namespace rook-ceph-system rook-stable/rook-ceph

    // check
    $  kubectl --namespace rook-ceph-system get pods -l "app=rook-ceph-operator"

请注意, rook-ceph-system 中的所有 Pod 都应该是 Running 或者Completed 状态,不应存在 restarts 或error 的情况。

### 2. 第二种方式：使用官方推荐的配置文件安装ceph存储

下载官方压缩包，解压后操作。如下：

    // 下载压缩包并解压
    $ wget https://github.com/rook/rook/archive/v1.2.5.tar.gz

    $ tar -zxvf v1.2.5.tar.gz

    $ cd cluster/examples/kubernetes/ceph/

    // 执行三联
    $ kubectl create -f common.yaml
    $ kubectl create -f operator.yaml
    $ kubectl create -f cluster.yaml

    // 查看安装效果
    $ kubectl get pods -n rook-ceph
    NAME                                                    READY   STATUS      RESTARTS   AGE
    csi-cephfsplugin-7sjpz                                  3/3     Running     0          5m54s
    csi-cephfsplugin-jjzpn                                  3/3     Running     0          5m42s
    csi-cephfsplugin-lvshq                                  3/3     Running     0          5m51s
    csi-cephfsplugin-provisioner-66c94d9784-mgn95           4/4     Running     0          5m38s
    csi-cephfsplugin-provisioner-66c94d9784-rxljk           4/4     Running     0          6m5s
    csi-rbdplugin-4zfl2                                     3/3     Running     0          6m
    csi-rbdplugin-n2tmw                                     3/3     Running     0          5m59s
    csi-rbdplugin-provisioner-b4cfc4fd5-7pvw5               5/5     Running     0          4m49s
    csi-rbdplugin-provisioner-b4cfc4fd5-q6jmn               5/5     Running     0          6m10s
    csi-rbdplugin-x696n                                     3/3     Running     0          6m3s
    rook-ceph-crashcollector-192.168.238.232-6b4b667d48-c8ltm   1/1     Running     0          61m
    rook-ceph-crashcollector-192.168.238.239-64db57cb89-kckzb   1/1     Running     0          62m
    rook-ceph-crashcollector-192.168.238.241-6d6697b8d8-hzxb7   1/1     Running     0          61m
    rook-ceph-mgr-a-75997f59cd-pk6rx                        1/1     Running     0          61m
    rook-ceph-mon-a-6f6b5c6cb8-zj2cd                        1/1     Running     0          62m
    rook-ceph-mon-b-8cb89bd84-6xzzp                         1/1     Running     0          62m
    rook-ceph-mon-c-d95686f54-vcttv                         1/1     Running     0          61m
    rook-ceph-operator-784cfd5c7d-msggb                     1/1     Running     1          6m17s
    rook-ceph-osd-prepare-192.168.238.232-kcz6x                 0/1     Completed   0          3m32s
    rook-ceph-osd-prepare-192.168.238.239-fxn5t                 0/1     Completed   0          3m26s
    rook-ceph-osd-prepare-192.168.238.241-2qqsg                 0/1     Completed   0          3m21s
    rook-discover-55rlx                                     1/1     Running     0          65m
    rook-discover-8cnnx                                     1/1     Running     0          65m
    rook-discover-s7thp                                     1/1     Running     0          65m

请注意, rook-ceph-system 中的所有 Pod 都应该是 Running 或者Completed 状态,不应存在 restarts 或error 的情况。

**问题：**

```

[centos@k8s-rke-node-start ceph]$ kubectl get pods -n rook-ceph
NAME                                                    READY   STATUS             RESTARTS   AGE
csi-cephfsplugin-mk622                                  1/3     CrashLoopBackOff   26         45m
csi-cephfsplugin-nhvrr                                  1/3     CrashLoopBackOff   26         45m
csi-cephfsplugin-provisioner-7b8fbf88b4-4wlpq           4/4     Running            0          45m
csi-cephfsplugin-provisioner-7b8fbf88b4-824jv           4/4     Running            0          45m
csi-cephfsplugin-z7nc4                                  1/3     CrashLoopBackOff   26         45m
csi-rbdplugin-8qt6t                                     1/3     CrashLoopBackOff   26         45m
csi-rbdplugin-hbgvt                                     1/3     CrashLoopBackOff   26         45m
csi-rbdplugin-provisioner-6b8b4d558c-p6kq8              5/5     Running            0          45m
csi-rbdplugin-provisioner-6b8b4d558c-tc954              5/5     Running            0          45m
csi-rbdplugin-s8c7h                                     1/3     CrashLoopBackOff   26         45m
rook-ceph-crashcollector-192.168.238.232-6b4b667d48-c8ltm   1/1     Running            0          42m
rook-ceph-crashcollector-192.168.238.239-64db57cb89-kckzb   1/1     Running            0          43m
rook-ceph-crashcollector-192.168.238.241-6d6697b8d8-hzxb7   1/1     Running            0          42m
rook-ceph-mgr-a-75997f59cd-pk6rx                        1/1     Running            0          42m
rook-ceph-mon-a-6f6b5c6cb8-zj2cd                        1/1     Running            0          43m
rook-ceph-mon-b-8cb89bd84-6xzzp                         1/1     Running            0          43m
rook-ceph-mon-c-d95686f54-vcttv                         1/1     Running            0          42m
rook-ceph-operator-7dcd87699d-cwdk8                     1/1     Running            0          45m
rook-ceph-osd-prepare-192.168.238.232-vc8lm                 0/1     Completed          0          41m
rook-ceph-osd-prepare-192.168.238.239-6tc99                 0/1     Completed          0          41m
rook-ceph-osd-prepare-192.168.238.241-88fhz                 0/1     Completed          0          41m
rook-discover-55rlx                                     1/1     Running            0          45m
rook-discover-8cnnx                                     1/1     Running            0          45m
rook-discover-s7thp                                     1/1     Running            0          45m

$ kubectl -n rook-ceph logs csi-cephfsplugin-mk622
Error from server (BadRequest): a container name must be specified for pod csi-cephfsplugin-mk622, choose one of: [driver-registrar csi-cephfsplugin liveness-prometheus]

$ kubectl -n rook-ceph logs csi-cephfsplugin-mk622 -c csi-cephfsplugin
I0305 03:41:37.740687       1 cephcsi.go:104] Driver version: v1.2.2 and Git version: f8c854dc7d6ffff02cb2eed6002534dc0473f111
I0305 03:41:37.740869       1 cachepersister.go:45] cache-perister: using kubernetes configmap as metadata cache persister
I0305 03:41:37.743887       1 cephcsi.go:131] Initial PID limit is set to -1
I0305 03:41:37.743983       1 cephcsi.go:140] Reconfigured PID limit to -1 (max)
I0305 03:41:37.743996       1 cephcsi.go:159] Starting driver type: cephfs with name: rook-ceph.cephfs.csi.ceph.com
I0305 03:41:37.747336       1 volumemounter.go:159] loaded mounter: kernel
I0305 03:41:37.763570       1 volumemounter.go:167] loaded mounter: fuse
I0305 03:41:37.763847       1 mountcache.go:59] mount-cache: name: rook-ceph.cephfs.csi.ceph.com, version: v1.2.2, mountCacheDir: /mount-cache-dir
I0305 03:41:37.763937       1 mountcache.go:99] mount-cache: successfully remounted 0 volumes
F0305 03:41:37.764345       1 httpserver.go:25] listen tcp 192.168.238.232:9091: bind: address already in use
I0305 03:41:37.764687       1 server.go:118] Listening for connections on address: &net.UnixAddr{Name:"//csi/csi.sock", Net:"unix"}

```

端口被占用的问题。查找三个文件，cluster.yaml、operator.yaml、common.yaml。然后找到operator.yaml中，存在的端口号9091、9081、9090、9080重新进行设置，一定保证设置为未使用的端口信息：

    $ vim operator.yaml
    // 修改下面的信息
        # Configure CSI cephfs grpc and liveness metrics port
        - name: CSI_CEPHFS_GRPC_METRICS_PORT
          value: "10091"
        - name: CSI_CEPHFS_LIVENESS_METRICS_PORT
          value: "10081"
        # Configure CSI rbd grpc and liveness metrics port
        - name: CSI_RBD_GRPC_METRICS_PORT
          value: "10090"
        - name: CSI_RBD_LIVENESS_METRICS_PORT
          value: "10080"

设置完成后，重新执行operator.yaml。

    $ kubectl apply -f operator.yaml

查看效果：

```
$ kubectl get pods -n rook-ceph
NAME                                                    READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-7sjpz                                  3/3     Running     0          5m54s
csi-cephfsplugin-jjzpn                                  3/3     Running     0          5m42s
csi-cephfsplugin-lvshq                                  3/3     Running     0          5m51s
csi-cephfsplugin-provisioner-66c94d9784-mgn95           4/4     Running     0          5m38s
csi-cephfsplugin-provisioner-66c94d9784-rxljk           4/4     Running     0          6m5s
csi-rbdplugin-4zfl2                                     3/3     Running     0          6m
csi-rbdplugin-n2tmw                                     3/3     Running     0          5m59s
csi-rbdplugin-provisioner-b4cfc4fd5-7pvw5               5/5     Running     0          4m49s
csi-rbdplugin-provisioner-b4cfc4fd5-q6jmn               5/5     Running     0          6m10s
csi-rbdplugin-x696n                                     3/3     Running     0          6m3s
rook-ceph-crashcollector-192.168.238.232-6b4b667d48-c8ltm   1/1     Running     0          61m
rook-ceph-crashcollector-192.168.238.239-64db57cb89-kckzb   1/1     Running     0          62m
rook-ceph-crashcollector-192.168.238.241-6d6697b8d8-hzxb7   1/1     Running     0          61m
rook-ceph-mgr-a-75997f59cd-pk6rx                        1/1     Running     0          61m
rook-ceph-mon-a-6f6b5c6cb8-zj2cd                        1/1     Running     0          62m
rook-ceph-mon-b-8cb89bd84-6xzzp                         1/1     Running     0          62m
rook-ceph-mon-c-d95686f54-vcttv                         1/1     Running     0          61m
rook-ceph-operator-784cfd5c7d-msggb                     1/1     Running     1          6m17s
rook-ceph-osd-prepare-192.168.238.232-kcz6x                 0/1     Completed   0          3m32s
rook-ceph-osd-prepare-192.168.238.239-fxn5t                 0/1     Completed   0          3m26s
rook-ceph-osd-prepare-192.168.238.241-2qqsg                 0/1     Completed   0          3m21s
rook-discover-55rlx                                     1/1     Running     0          65m
rook-discover-8cnnx                                     1/1     Running     0          65m
rook-discover-s7thp                                     1/1     Running     0          65m
```

### 3. 安装toolbox和dashboard

#### 安装toolbox

直接依据这个rook-ceph-tools.yaml进行安装：

    $ kubectl apply -f toolbox.yaml

一旦 toolbox 的 Pod 运行成功后，我们就可以使用下面的命令进入到工具箱内部进行操作：

    $ kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash

    // 查看安装后状态
    $ kubectl get pods -n rook-ceph
    rook-ceph-tools-7d764c8647-vdz5r                        1/1     Running     0          3d20h
    
    // 进入下面的窗口，执行ceph status
    [root@rook-ceph-tools-7d764c8647-vdz5r /]# ceph status
    cluster:
        id:     18c1fa7d-944b-4a2e-8ef0-e6699c49ac67
        health: HEALTH_OK

    services:
        mon: 3 daemons, quorum a,b,c (age 2d)
        mgr: a(active, since 13h)
        mds: myfs:1 {0=myfs-a=up:active} 1 up:standby-replay
        osd: 3 osds: 3 up (since 2d), 3 in (since 2d)
        rgw: 1 daemon active (my.store.a)

    data:
        pools:   10 pools, 80 pgs
        objects: 264 objects, 13 KiB
        usage:   16 GiB used, 177 GiB / 192 GiB avail
        pgs:     80 active+clean

    io:
        client:   853 B/s rd, 1 op/s rd, 0 op/s wr

    // 执行ceph osd status
    [root@rook-ceph-tools-7d764c8647-vdz5r /]# ceph osd status
    +----+-------------+-------+-------+--------+---------+--------+---------+-----------+
    | id |     host    |  used | avail | wr ops | wr data | rd ops | rd data |   state   |
    +----+-------------+-------+-------+--------+---------+--------+---------+-----------+
    | 0  | 192.168.238.239 | 5405M | 44.8G |    0   |     0   |    2   |   106   | exists,up |
    | 1  | 192.168.238.232 | 5235M | 35.9G |    0   |     0   |    0   |     0   | exists,up |
    | 2  | 192.168.238.241 | 5368M | 95.8G |    0   |     0   |    1   |     0   | exists,up |
    +----+-------------+-------+-------+--------+---------+--------+---------+-----------+

    // 执行ceph df
    [root@rook-ceph-tools-7d764c8647-vdz5r /]# ceph df
    RAW STORAGE:
      CLASS     SIZE        AVAIL       USED       RAW USED     %RAW USED
        hdd       192 GiB     177 GiB     16 GiB       16 GiB          8.13
        TOTAL     192 GiB     177 GiB     16 GiB       16 GiB          8.13

    POOLS:
      POOL                            ID     STORED      OBJECTS     USED        %USED     MAX AVAIL
        my-store.rgw.control             1         0 B           8         0 B         0        53 GiB
        my-store.rgw.meta                2       373 B           2       373 B         0        53 GiB
        my-store.rgw.log                 3        50 B         210        50 B         0        53 GiB
        my-store.rgw.buckets.index       4         0 B           0         0 B         0        53 GiB
        my-store.rgw.buckets.non-ec      5         0 B           0         0 B         0        53 GiB
        .rgw.root                        6     3.7 KiB          16     3.7 KiB         0        53 GiB
        my-store.rgw.buckets.data        7         0 B           0         0 B         0        53 GiB
        myfs-metadata                    9     9.2 KiB          22     9.2 KiB         0        53 GiB
        myfs-data0                      10         0 B           0         0 B         0        53 GiB
        replicapool                     11        36 B           6        36 B         0        53 GiB

工具箱中的所有可用工具命令均已准备就绪，可满足您的故障排除需求。

#### 安装dashboard

查看已有的服务信息，再确定dashboard的安装方式，如下：

```
    $ kubectl get service -n rook-ceph
NAME                                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
csi-cephfsplugin-metrics                 ClusterIP   10.43.49.186    <none>        8080/TCP,8081/TCP   3d22h
csi-rbdplugin-metrics                    ClusterIP   10.43.156.212   <none>        8080/TCP,8081/TCP   3d22h
rook-ceph-mgr                            ClusterIP   10.43.112.170   <none>        9283/TCP            3d22h
rook-ceph-mgr-dashboard                  ClusterIP   10.43.99.56     <none>        8443/TCP            3d22h
rook-ceph-mon-a                          ClusterIP   10.43.132.112   <none>        6789/TCP,3300/TCP   3d22h
rook-ceph-mon-b                          ClusterIP   10.43.86.9      <none>        6789/TCP,3300/TCP   3d22h
rook-ceph-mon-c                          ClusterIP   10.43.188.99    <none>        6789/TCP,3300/TCP   3d22h
rook-ceph-rgw-my-store                   ClusterIP   10.43.53.12     <none>        80/TCP              2d23h
```

默认dashboard使用https安装，如下：

```
$ kubectl apply -f dashboard-external-https.yaml

$ kubectl -n rook-ceph get service -o wide
NAME                                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE     SELECTOR

rook-ceph-mgr-dashboard                  ClusterIP   10.43.99.56     <none>        8443/TCP            3h48m   app=rook-ceph-mgr,rook_cluster=rook-ceph
rook-ceph-mgr-dashboard-external-https   NodePort    10.43.209.164   <none>        8443:32166/TCP      42s     app=rook-ceph-mgr,rook_cluster=rook-ceph

```

获取dashboard访问密码：
$ kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
stU6jhkSVK

用任意一个集群节点的ip地址均可以访问，例如:https://192.168.238.232:32166，输入用户名：admin，密码：stU6jhkSVK。

### 4. 创建存储

    //  创建对象存储
    $ kubectl apply -f object.yaml

    // 以及用户
    $ kubectl apply -f object-user.yaml

    // 创建Ceph pool（块存储池）
    // 使用的kind是CephBlockPool
    $ kubectl apply -f pool.yaml

    // 创建文件存储
    $ kubectl apply -f filesystem.yaml

    // 创建rbd存储信息
    $ cd csi/rbd/

    $ kubectl apply -f storageclass.yaml
    // 出现以下提示
    cephblockpool.ceph.rook.io/replicapool configured
    storageclass.storage.k8s.io/rook-ceph-block created

    // 创建pvc
    $ kubectl apply -f pvc.yaml

    $ kubectl get pvc
    NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
    rbd-pvc   Bound    pvc-22e52bf9-429a-4ff9-8978-9f647d8354ca   2Gi        RWO            rook-ceph-block   11s

这样存储就创建完毕了。我们来查看下创建的osd存储，如下：

    // 登录节点主机
    $ ssh -p 15555 centos@192.168.238.232

    // 查看osd存储
    $ sudo ls /home/centos/rook-storage/osd1/ -lh
    // 输出以下信息
    total 5.1G
    -rw-------   1 root root   37 Mar  6 15:50 ceph_fsid
    drwxr-xr-x 164 root root 4.0K Mar  6 16:52 current
    -rw-r--r--   1 root root   37 Mar  6 15:50 fsid
    -rw-r--r--   1 root root 5.0G Mar  9 09:37 journal
    -rw-r--r--   1 root root   56 Mar  6 15:50 keyring
    -rw-------   1 root root   21 Mar  6 15:50 magic
    -rw-------   1 root root    6 Mar  6 15:50 ready
    -rw-------   1 root root    3 Mar  6 15:51 require_osd_release
    -rw-r--r--   1 root root 1.2K Mar  8 19:49 rook-ceph.config
    -rw-------   1 root root    4 Mar  6 15:50 store_version
    -rw-------   1 root root   53 Mar  6 15:50 superblock
    drwxr--r--   2 root root   29 Mar  6 15:50 tmp
    -rw-------   1 root root   10 Mar  6 15:50 type
    -rw-------   1 root root    2 Mar  6 15:50 whoami

查看其它节点的存储信息类似操作即可。

**问题：**

```
$ kubectl get pvc
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS              AGE
myclaim   Pending                                      rook-ceph-retain-bucket   11m

$ kubectl describe pvc myclaim
Name:          myclaim
Namespace:     default
StorageClass:  rook-ceph-retain-bucket
Status:        Pending
Volume:
Labels:        <none>
Annotations:   volume.beta.kubernetes.io/storage-provisioner: ceph.rook.io/bucket
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:
Access Modes:
VolumeMode:    Filesystem
Mounted By:    <none>
Events:
  Type    Reason                Age                   From                         Message
  ----    ------                ----                  ----                         -------
  Normal  ExternalProvisioning  2m29s (x43 over 12m)  persistentvolume-controller  waiting for a volume to be created, either by external provisioner "ceph.rook.io/bucket" or manually created by system administrator

```

    // 首先需要删除
    $ cd /home/centos/rook/rook-1.2.5/cluster/examples/kubernetes/ceph
    $ kubectl delete -f pvc-example.yaml
    $ kubectl delete -f storageclass-bucket-retain.yaml

    // 可以不进行删除
    $ kubectl delete -f pools.yaml

    // 对cluster.yml文件配置进行修改
    $ vim cluster.yaml 

    // 修改服务配置的文件目录，要有操作权限
    directories:
    - path: /home/centos/rook-storage

    // 然后修改默认的位置，由于ceph v14.2.7不支持useAllDevices: true
    useAllDevices: false

    // 重新执行cluster的配置
    $ kubectl apply -f cluster.yaml

    // 然后回到ceph镜像中，查看
    $ kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash

```
# ceph status
  cluster:
    id:     18c1fa7d-944b-4a2e-8ef0-e6699c49ac67
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 64m)
    mgr: a(active, since 13m)
    mds: myfs:1 {0=myfs-a=up:active} 1 up:standby-replay
    osd: 3 osds: 3 up (since 56m), 3 in (since 56m)
    rgw: 1 daemon active (my.store.a)

  data:
    pools:   10 pools, 80 pgs
    objects: 226 objects, 7.9 KiB
    usage:   16 GiB used, 177 GiB / 192 GiB avail
    pgs:     80 active+clean

  io:
    client:   853 B/s rd, 1 op/s rd, 0 op/s wr
```

此时，修改完成。

同时解决下面的问题：

```
$ kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}')  bash
bash: warning: setlocale: LC_CTYPE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_COLLATE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_MESSAGES: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_NUMERIC: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_TIME: cannot change locale (en_US.UTF-8): No such file or directory
[root@rook-ceph-tools-7d764c8647-vdz5r /]# ceph -w
  cluster:
    id:     18c1fa7d-944b-4a2e-8ef0-e6699c49ac67
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3

  services:
    mon: 3 daemons, quorum a,b,c (age 2h)
    mgr: a(active, since 86m)
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:

```

HEALTH_WARN OSD count 0 < osd_pool_default_size 3 尚未有大的影响，后续注意

问题位置：https://github.com/rook/rook/issues/3348

https://github.com/rook/rook/issues/4963

原因是：ceph镜像版本为14.2.7版本的，不允许开启useAllDevices

见官网https://rook.io/docs/rook/v1.2/ceph-quickstart.html

对于Dashboard上enable一些功能后，通常需要重启Dashboard服务才能使用。使用下面的命令来重启Dashboard：

    // 在toolbox容器里执行
    [root@AI03 /]# ceph mgr module disable dashboard
    [root@AI03 /]# ceph mgr module enable dashboard

<!-- op-mgr: failed modules: "orchestrator modules". failed to set rook orchestrator backend: failed to set rook as the orchestrator backend: failed to complete command: context deadline exceeded -->

## 番外：清理rook-ceph命名空间以及删除

helm安装时，需要比较暴力的进行删除，直接删除命名空间rook-ceph-system中，所有的pods，最后删除改命名空间。**切记，线上环境慎用该操作！**

如果是通过源码配置安装的，参考下面的方式删除。

```
$ kubectl delete -f cluster.yaml
$ kubectl delete -f operator.yaml
$ kubectl delete -f common.yaml

// 需要到集群的每个节点机器上执行下面的操作
$ cd /var/lib/rook/
$ sudo rm -rf ./*

```
参考链接：https://github.com/rook/rook/blob/master/Documentation/ceph-teardown.md#troubleshooting

## 总结

搭建rook+ceph时，一定要耐心看官网教程！

## 参考地址

* https://time.geekbang.org/column/article/39724
* https://zhuanlan.zhihu.com/p/81417869
* https://www.cnblogs.com/passzhang/p/12191332.html
* http://www.yangguanjun.com/2018/12/28/rook-ceph-practice-part2/
* https://www.cnblogs.com/kevincaptain/p/10655721.html
* http://www.yangguanjun.com/2018/12/22/rook-ceph-practice-part1/
* https://rook.github.io/docs/rook/v1.2/
* https://www.qikqiak.com/post/deploy-ceph-cluster-with-rook/
* https://blog.fleeto.us/post/the-ultimate-rook-and-ceph-survival-guide/
* https://fkpwolf.net/cloud/db/2018/09/14/rook.html
