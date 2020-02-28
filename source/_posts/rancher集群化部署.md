---
title: rancher集群化部署
top: false
cover: false
toc: true
mathjax: true
date: 2020-02-18 10:02:26
password:
summary:
tags:
categories:
---

# Rancher集群化部署--在线安装

## 前期准备

目前我们的工作进展有序，已经实现了利用Docker部署各微服务，结合CI/CD进行微服务构建。

但是面对当前的部署方式，依旧存在局限性：

1. 集群问题，部署时启动多套有难度
2. 微服务停机或者升级时，或者微服务出现故障时，导致出现问题后不能及时上报
3. 目前的部署方式必须进行停机更新，不满足服务不停机更新，且不满足滚动部署或者蓝绿部署的要求
4. 微服务不能进行弹性伸缩，在熔断、限流、降级方面做的不足

针对以上的问题，引入k8s进行微服务的集群化部署。利用Rancher来针对k8s集群进行统一管理，对整个集群进行运维。

### 规划容量

我们目前预测大概要到100多个的微服务集群信息，参考Rancher官网的容量规划，进行配置。初步确定服务器大概为7台，其中三台安装k8s，剩余一台做管理机，其它三台做备用机，进行日常添加删除主机的使用。

配置图如下：

|主机名称|主机配置|主机ip|部署服务|备注|
|--------|-------|------|------|-----|
|k8s-rke-node-start| 4c 16G 200G| 192.168.238.240 | nginx访问入口，rke、kubectl、helm安装工具| 安装机和日常运维机器|
|k8s-master-node00| 4c 16G 200G| 192.168.238.239 | k8s主节点工具、rancher | 集群主节点1号|
|k8s-master-node01| 4c 16G 200G| 192.168.238.232 | k8s主节点工具、rancher | 集群主节点2号|
|k8s-master-node02| 4c 16G 200G| 192.168.238.241 | k8s主节点工具、rancher | 集群主节点3号|
|k8s-master-cluster-node01| 2c 8G 100G| 192.168.238.233 | k8s从节点 | 集群从节点1号，未使用|
|k8s-master-cluster-node02| 2c 8G 100G| 192.168.238.234 | k8s从节点 | 集群从节点2号，未使用|
|k8s-master-cluster-node03| 2c 8G 100G| 192.168.238.235 | k8s从节点 | 集群从节点3号，未使用|

### 环境要求

* 操作系统：CentOS 7.6 1810，以Basic Server方式安装
* docker版本：19.03.6
* kubernetes版本：1.17.2
* rancher版本：2.3.5
* nginx版本：1.16.1
* 工具链
    * rke: 1.0.4
    * kubectl: 1.16.6
    * helm: 3.1.0

### 一、服务器准备

参考 Linux日常运维操作 进行服务器准备，优先设置系统初始化的相关安装，以及docker软件的安装，这里不再说明，请参考。随后从docker安装完成后进行准备说明。在docker安装完成后，要对其进行锁定，避免因为使用yum update命令导致docker版本变动。

**下面的操作每一台服务器上都要进行操作。**

#### 1. docker的基本配置

daemon.json默认位于/etc/docker/daemon.json，如果没有可手动创建，基于systemd管理的系统都是相同的路径。通过修改daemon.json来改过Docker配置，也是Docker官方推荐的方法。

1、配置私有仓库（目前并未使用私有仓库，暂时不进行配置）
Docker默认只信任TLS加密的仓库地址(https)，所有非https仓库默认无法登陆也无法拉取镜像。insecure-registries字面意思为不安全的仓库，通过添加这个参数对非https仓库进行授信。可以设置多个insecure-registries地址，以数组形式书写，地址不能添加协议头(http)。
{
    "insecure-registries": ["harbor.xxx.cn:30002"]
}

2、配置存储驱动
OverlayFS是一个新一代的联合文件系统，类似于AUFS，但速度更快，实现更简单。Docker为OverlayFS提供了两个存储驱动程序:旧版的overlay，新版的overlay2(更稳定)。

先决条件:

overlay2: Linux内核版本4.0或更高版本，或使用内核版本3.10.0-514+的RHEL或CentOS。
overlay: 主机Linux内核版本3.18+

支持的磁盘文件系统

ext4(仅限RHEL 7.1)
xfs(RHEL7.2及更高版本)，需要启用d_type=true。

{
    "storage-driver": "overlay2",
    "storage-opts": ["overlay2.override_kernel_check=true"]
}

3、配置日志驱动（由于设置后会导致docker服务无法启动，暂时未进行设置）

容器在运行时会产生大量日志文件，很容易占满磁盘空间。通过配置日志驱动来限制文件大小与文件的数量。 >限制单个日志文件为50M,最多产生3个日志文件
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "50m",
        "max-file": "3"
    }
}

4、配置国内公共镜像

 "registry-mirrors": [
        "https://docker.mirrors.ustc.edu.cn",
        "https://registry.docker-cn.com"
    ],

最终配置文件如下：

    {
        "registry-mirrors": [
            "https://docker.mirrors.ustc.edu.cn",
            "https://registry.docker-cn.com"
        ],
        "insecure-registries": ["harbor.xxx.cn:30002"]，
        "log-driver": "json-file",
        "log-opts": {
            "max-size": "50m",
            "max-file": "3"
        }
    }

操作如下：

    # vim /etc/docker/daemon.json

    // 输入以下内容
    {
    "registry-mirrors": [
        "https://docker.mirrors.ustc.edu.cn",
        "https://registry.docker-cn.com"
    ],
    "insecure-registries": ["harbor.xxx.cn:30002"]，
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "50m",
        "max-file": "3"
        }
    }

    // :wq进行保存

设置完成后，需要重启docker服务，操作如下：

    $ sudo systemctl daemon-reload
    $ sudo systemctl restart docker

#### 2. 服务器名称修改

    // 变更服务器hostname
    # hostnamectl set-hostname k8s-node-1

    // 修改host文件
    # vim /etc/hosts
    // 在文件后续追加以下内容
    192.168.238.239 k8s-master-node00
    192.168.238.240 k8s-rke-node-start
    192.168.238.232 k8s-cluster-node01
    192.168.238.241 k8s-cluster-node02


#### 3. 开放端口

    // 根据官方文档开放端口信息
    $ sudo firewall-cmd --zone=public --add-port=80/tcp --permanent

    $ sudo firewall-cmd --zone=public --add-port=443/tcp --permanent

    $ sudo firewall-cmd --zone=public --add-port=2376/tcp --permanent

    $ sudo firewall-cmd --zone=public --add-port=6443/tcp --permanent

    $ sudo firewall-cmd --zone=public --add-port=2379/tcp --permanent

    $ sudo firewall-cmd --zone=public --add-port=2380/tcp --permanent

    $ sudo firewall-cmd --zone=public --add-port=8472/udp --permanent

    $ sudo firewall-cmd --zone=public --add-port=9099/tcp --permanent

    $ sudo firewall-cmd --zone=public --add-port=10250/tcp --permanent

    $ sudo firewall-cmd --zone=public --add-port=10254/tcp --permanent

    $ sudo firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent

    $ sudo firewall-cmd --zone=public --add-port=30000-32767/udp --permanent
    
    // 使配置生效
    $ sudo firewall-cmd --reload
  
#### 4. 关闭selinux

    // 获取linux状态
    $ sudo getenforce
    // 命令行设置关闭selinux
    $ sudo setenforce 0
    // 修改配置文件
    $ sudo vim /etc/selinux/config
    // 将文件中的SELINUX=enforcing改为SELINUX=disabled

    // :wq 保存文件退出


#### 5. 禁用swap
   
    // 删除 swap 区所有内容
    # swapoff -a
    删除 swap 挂载，这样系统下次启动不会再挂载 swap
    // 编辑挂载文件，用 “#” 注释 swap 行
    # vim /etc/fstab
    // 找到/dev/mapper/centos-swap swap这一行，用“#”进行注释

    // 重启系统，测试
    # reboot
    
    // 重启后查看
    $ free -h
                total        used        free      shared  buff/cache   available
    Mem:           3.7G        203M        3.1G        8.5M        384M        3.3G
    Swap:            0B          0B          0B

#### 7. 操作系统及kernel调优

    // 针对文件打开数调优。
    # vim /etc/security/limits.conf
    // 输入以下内容
    root soft nofile 65535
    root hard nofile 65535
    *    soft nofile 65535
    *    hard nofile 65535
    // :wq保存退出

    // 调整最大进程数
    # sed -i 's#4096#65535#g' /etc/security/limits.d/20-nproc.conf
    
    // kernel调优
    # cat >> /etc/sysctl.conf<<EOF
    net.ipv4.ip_forward=1
    net.bridge.bridge-nf-call-iptables=1
    net.bridge.bridge-nf-call-ip6tables=1
    vm.swappiness=0
    vm.max_map_count=655360
    EOF

    // 使修改生效
    $ sudo sysctl -p

#### 8. 服务器免密登录

这一步操作从192.168.238.240所在的服务器进行操作，由此机器将ssh密钥生成并发送到各个目标机器上。

    // 生成密钥
    $ ssh-keygen -t rsa

    $ ssh-copy-id -p 15555 centos@192.168.238.232
    $ ssh-copy-id -p 15555 centos@192.168.238.239
    $ ssh-copy-id -p 15555 centos@192.168.238.241

### 二、安装工具链--rke、kubectl、helm

#### 1. 下载rke工具
    
    // 在线下载，也可以下载完成后通过ftp传输到服务器上
    $ wget https://github.com/rancher/rke/releases/download/v1.0.4/rke_linux-amd64

    // 添加可执行权限
    $ sudo chmod +x rke_linux-amd64
    
    // 添加到环境变量，或者直接在~/.bashrc文件中用export命令暴露该工具
    $ sudo mv rke_linux-amd64 /usr/local/bin/rke

    // 测试是否已经安装完毕
    $ rke --version
    // 输出信息：
    rke version v1.0.4

#### 2. 下载kubectl工具

1.16.6版本对应

    // 有时候会下载不下来，多尝试几次
    $ curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.16.6/bin/linux/amd64/kubectl

    $ sudo chmod +x kubectl

    $ sudo mv ./kubectl /usr/local/bin/kubectl

    // 引入自动补全

    // 安装自动补全工具
    $ sudo yum install bash-completion -y
    $ echo "source <(kubectl completion bash)" >> ~/.bashrc


#### 3. 安装Helm

访问 Helm Github 下载页面 https://github.com/helm/helm/releases 找到最新的客户端，里面有不同系统下的包，这里我们选择 Linux amd64，然后在 Linux 系统中使用 Wget 命令进行下载。

    // 下载Helm客户端
    $ wget https://get.helm.sh/helm-v3.1.0-linux-amd64.tar.gz
    
接下来解压下载的包，然后将客户端放置到 /usr/local/bin/ 目录下：

    // 解压 Helm
    $ tar -zxvf helm-v3.1.0-linux-amd64.tar.gz

    // 复制客户端执行文件到 bin 目录下，方便在系统下能执行 helm 命令
    $ sudo mv linux-amd64/helm /usr/local/bin/helm

    // 测试是否安装成功
    $ helm version
    // 输出以下内容
    version.BuildInfo{Version:"v3.1.0", GitCommit:"b29d20baf09943e134c2fa5e1e1cab3bf93315fa", GitTreeState:"clean", GoVersion:"go1.13.7"}

    // 下面需要添加Chart仓库

    $ helm repo add  elastic    https://helm.elastic.co

    $ helm repo add  gitlab     https://charts.gitlab.io
    
    $ helm repo add  harbor     https://helm.goharbor.io

    $ helm repo add  bitnami    https://charts.bitnami.com/bitnami

    $ helm repo add  incubator  http://mirror.azure.cn/kubernetes/charts-incubator/

    $ helm repo add  stable     http://mirror.azure.cn/kubernetes/charts-incubator/

    $ helm repo update

### 三、利用rke安装k8s

#### 1. 生成配置文件并进行集群部署

可以用两种方式创建配置文件，一种是直接编写yml文件，另一种是通过交互式方式生成yml文件。我这边在实践的过程中，发现通过交互式生成生成的配置文件，在安装过程中出现了各种各样的问题。利用交互式生成配置文件，然后进行进行裁剪，参考官网步骤进行配置。这里直接进行编写。

    $ mkdir cluster

    // 运行rke config交互式配置向导
    $ rke config

    // 配置完成后编辑生成的文件cluster.yml
    $ vim cluster.yml
    // 文件最终内容如下：
```
# If you intened to deploy Kubernetes in an air-gapped environment,               
# please consult the documentation on how to configure custom RKE images.         
nodes:                                                                            
- address: 192.168.238.239                                                            
  port: "15555"                                                                   
  internal_address: ""                                                            
  role:                                                                           
  - controlplane                                                                  
  - worker                                                                        
  - etcd                                                                          
  hostname_override: ""                                                           
  user: centos                                                                    
  docker_socket: /var/run/docker.sock                                             
  ssh_key: ""                                                                     
  ssh_key_path: ~/.ssh/id_rsa                                                     
  ssh_cert: ""                                                                    
  ssh_cert_path: ""                                                               
  labels: {}                                                                      
  taints: []                                                                      
- address: 192.168.238.232                                                            
  port: "15555"                                                                   
  internal_address: ""                                                            
  role:                                                                           
  - controlplane                                                                  
  - worker                                                                        
  - etcd                                                                          
  hostname_override: ""                                                           
  user: centos                                                                    
  docker_socket: /var/run/docker.sock                                             
  ssh_key: ""                                                                     
  ssh_key_path: ~/.ssh/id_rsa                                                     
  ssh_cert: ""                                                                    
  ssh_cert_path: ""                                                               
  labels: {}                                                                      
  taints: []                                                                      
- address: 192.168.238.241                                                            
  port: "15555"                                                                   
  internal_address: ""                                                            
  role:                                                                           
  - controlplane                                                                  
  - worker                                                                        
  - etcd                                                                          
  hostname_override: ""
  user: centos
  docker_socket: /var/run/docker.sock
  ssh_key: ""
  ssh_key_path: ~/.ssh/id_rsa
  ssh_cert: ""
  ssh_cert_path: ""
  labels: {}
  taints: []
services:
  etcd:
    snapshot: true
    retention: 24h
    creation: 6h
# 如果这里你使用自带的ssl签名或者使用公用签名，可以不添加下面的选项
ingress:
  provider: nginx
  options:
    use-forwarded-headers: "true"
```

设置完成后，下面可以正式开始部署了。执行下面的命令开始安装：

    // 部署并启动k8s集群
    $ rke up --config ./rancher-cluster.yml

这时候开始进入部署状态。将会在三台服务器上进行部署，拉取相应镜像进行操作。

但是在部署过程中会存在各种各样的问题，下面单独设置一个章节讲解。假设现在一切顺利，安装完成了。最终在控制台上显示

    INFO[0172] Finished building Kubernetes cluster successfully

说明安装完成。这时候k8s集群就正式完成了。部署完成后，在同级别目录下会存在三个文件，其中两个是主动生成的，分别为：

    cluster.yml：RKE集群配置文件。
    kube_config_cluster.yml：集群的Kubeconfig文件，此文件包含完全访问集群的凭据。
    cluster.rkestate：Kubernetes集群状态文件，此文件包含集群部署的状态，用于升级和更新。    



#### 2. 问题总结

* **前提**：如果出现错误，请先准备好清理脚本，重新进行安装。在232、239、241服务器上进行下面创建脚本的操作。

如果有错误，先停止所有机器的镜像，无论是否在运行，通过执行下面的清理脚本恢复服务器初始状态。操作如下：


    $ vim rancher_clean-dirs.sh 
    // 输入以下内容
    #!/bin/sh

    if [ $(id -u) -ne 0 ]; then
        echo "Must be run as root!"
        exit
    fi 

    DLIST="/var/lib/etcd /etc/kubernetes /etc/cni /opt/cni /var/lib/cni /var/run/calico /opt/rke"
    for dir in $DLIST; do
        echo "Removing $dir"
        rm -rf $dir
    done
    // :wq进行保存退出

    $ vim rancher_clean-docker.sh
    // 输入以下内容
    #!/bin/sh

    CLIST=$(docker ps -qa)
    if [ "x"$CLIST == "x" ]; then
        echo "No containers exist - skipping container cleanup"
    else
        docker rm -f $CLIST
    fi

    ILIST=$(docker images -a -q)
    if [ "x"$ILIST == "x" ]; then
        echo "No images exist - skipping image cleanup"
    else
        docker rmi $ILIST
    fi

    VLIST=$(docker volume ls -q)
    if [ "x"$VLIST == "x" ]; then
        echo "No volumes exist - skipping volume cleanup"
    else
        docker volume rm -f $VLIST
    fi
    // :wq进行保存退出

    // 针对每个文件添加可执行权限
    $ sudo chmod +x rancher_clean-dirs.sh 

    $ sudo chmod +x rancher_clean-docker.sh 

这样在具体执行的时候，在目标服务器上，进行以下操作：

    $ sudo ./rancher_clean-docker.sh

    $ sudo ./rancher_clean-docker.sh 

即可进行服务器清理操作，对目标服务器清理完成后，再回到240服务器进行执行部署命令。


* 问题列表


1. 安装报错：

如果这一步报错下边的内容：

    if the SSH server version is at least version 6.7 or higher. If you are using RedHat/CentOS, you can't use the user `root`. Please refer to the documentation for more instructions

则可能是系统的openssh版本太低，只需执行如下命令升级即可：

    $ ssh -V
    OpenSSH_6.6.1p1, OpenSSL 1.0.1e-fips 11 Feb 2013  
    // 低于上边要求的6.7
    // 切换管理员权限
    $ sudo su
    # yum -y update openssh
    # ssh -V
    OpenSSH_7.4p1, OpenSSL 1.0.2k-fips  26 Jan 2017

然后再切回centos用户执行安装即可！

2. 安装报错：

错误日志如下：

    WARN[0934] Failed to create Docker container [etcd-fix-perm] on host [192.168.238.239]: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
    WARN[0934] Failed to create Docker container [etcd-fix-perm] on host [192.168.238.239]: Error response from daemon: Conflict. The container name "/etcd-fix-perm" is already in use by container "3d0a7d0a9d12fb088eeda6a02c5f18c98a561149fadf9d4fa0b261b0f79273b5". You have to remove (or rename) that container to be able to reuse that name.
    WARN[0934] Failed to create Docker container [etcd-fix-perm] on host [192.168.238.239]: Error response from daemon: Conflict. The container name "/etcd-fix-perm" is already in use by container "3d0a7d0a9d12fb088eeda6a02c5f18c98a561149fadf9d4fa0b261b0f79273b5". You have to remove (or rename) that container to be able to reuse that name.
    FATA[0934] [etcd] Failed to bring up Etcd Plane: Failed to create [etcd-fix-perm] container on host [192.168.238.239]: Failed to create Docker container [etcd-fix-perm] on host [192.168.238.239]: <nil>

只在239机器上执行清理脚本后，再转回到240机器，运行rke config --name cluster.yml。

3. 安装报错：

错误日志如下：

    FATA[0700] Failed to get job complete status for job rke-network-plugin-deploy-job in namespace kube-system

根据之前cluster.yml中的配置，创建docker网络，参考service_cluster_ip_range的配置项，默认为10.43.0.0/16
    
    docker network create --driver=bridge --subnet=10.43.0.0/16 br0_rke

如果不创建也可以，重新执行rke config --name cluster.yml命令。

4. 安装报错

清理完成后还是报错：

    WARN[0224] [etcd] host [192.168.238.239] failed to check etcd health: failed to get /health for host [192.168.238.239]: Get https://192.168.238.239:2379/health: remote error: tls: bad certificate
    WARN[0239] [etcd] host [192.168.238.232] failed to check etcd health: failed to get /health for host [192.168.238.232]: Get https://192.168.238.232:2379/health: remote error: tls: bad certificate
    WARN[0255] [etcd] host [192.168.238.241] failed to check etcd health: failed to get /health for host [192.168.238.241]: Get https://192.168.238.241:2379/health: remote error: tls: bad certificate
    FATA[0255] [etcd] Failed to bring up Etcd Plane: etcd cluster is unhealthy: hosts [192.168.238.239,192.168.238.232,192.168.238.241] failed to report healthy. Check etcd container logs on each host for more information

请检查时钟同步问题，第二个是检查是否有未开放的端口信息。如果还有问题，需要重新执行清理脚本后，再运行rke config --name rancher-cluster.yml。

#### 3. 配置信息指定

安装完成后，查看各个节点信息，执行以下命令：

    $ kubectl get nodes

可能会出现以下报错

    The connection to the server localhost:8080 was refused - did you specify the right host or port?

或者报错情况如下： 
 
    Config not found: /home/centos/kube_config_cluster.yml
    The connection to the server localhost:8080 was refused - did you specify the right host or port?

原因：kubenetes master没有与本机绑定，集群初始化的时候没有设置

解决办法：

    // 在命令行直接执行：
    $ export KUBECONFIG=/home/centos/kube_config_cluster.conf
    
    // 或者将该命令放到~/.bashrc中 
    $ vim ~/.bashrc
    // 文件末尾追加
    export KUBECONFIG=/home/centos/kube_config_cluster.conf
    // :wq保存
    // 使配置生效
    $ source ~/.bashrc

    // 或者 在执行命令时手动指定配置文件，利用--kubeconfig。例如：
    $ kubectl get nodes --kubeconfig=/home/centos/kube_config_cluster.conf

<!-- /etc/kubernetes/admin.conf这个文件主要是集群初始化的时候用来传递参数的 -->

解决完上述问题，这时重新执行命令即可。

    $ kubectl get nodes
    NAME          STATUS   ROLES                      AGE   VERSION
    192.168.238.232   Ready    controlplane,etcd,worker   36m   v1.16.6
    192.168.238.239   Ready    controlplane,etcd,worker   36m   v1.16.6
    192.168.238.241   Ready    controlplane,etcd,worker   36m   v1.16.6

    $  kubectl get pods --all-namespaces

    NAMESPACE       NAME                                      READY   STATUS      RESTARTS   AGE
    ingress-nginx   default-http-backend-67cf578fc4-2g498     1/1     Running     0          7m4s
    ingress-nginx   nginx-ingress-controller-44wzn            1/1     Running     0          7m5s
    ingress-nginx   nginx-ingress-controller-cf84g            1/1     Running     0          7m5s
    ingress-nginx   nginx-ingress-controller-dxpxq            1/1     Running     0          7m5s
    kube-system     canal-28m8p                               2/2     Running     0          32m
    kube-system     canal-479px                               2/2     Running     0          32m
    kube-system     canal-vdcvf                               2/2     Running     0          32m
    kube-system     coredns-7c5566588d-flshv                  1/1     Running     0          7m29s
    kube-system     coredns-7c5566588d-t6vx6                  1/1     Running     0          6m35s
    kube-system     coredns-autoscaler-65bfc8d47d-bnj2h       1/1     Running     0          7m26s
    kube-system     metrics-server-6b55c64f86-fgpkd           1/1     Running     0          7m15s
    kube-system     rke-coredns-addon-deploy-job-vtnrs        0/1     Completed   0          7m34s
    kube-system     rke-ingress-controller-deploy-job-d556q   0/1     Completed   0          7m13s
    kube-system     rke-metrics-addon-deploy-job-dpdhq        0/1     Completed   0          7m23s
    kube-system     rke-network-plugin-deploy-job-xkmgs       0/1     Completed   0          33m

说明已经完成。需要注意的是，有些服务并未处在Running状态，一定要保证全部处于Running状态时，才能进行下一步操作。

### 四、Rancher集群的安装

在进行Rancher集群安装之前，需要说明，这里采用私有签发CA证书的方式来实现SSL访问。首先进行证书的签发，然后进行rancher集群的安装。

#### 1. 证书签发

使用openssl创建自己的CA certificate

    // # ① 生成CA私钥文件
    $ openssl genrsa -out ca.key 2048

值得注意的是，Common Name一定要填写*.exmaple.org，以表示这个证书是一个通配证书（Wildcard Certificates），浏览器检查一个证书是否匹配，检查的就是Common Name。

    // # ② 生成CA证书文件
    $ openssl req -new -x509 -key ca.key -out ca.crt

这时候根证书已经完成，下面利用根证书来进行签发CA证书。操作如下：

    // 签发对应域名的私有密钥
    $ openssl genrsa -out rancher.mine.com.key 2048

    // 签发csr证书文件
    $ openssl req -new -key rancher.mine.com.key -out rancher.mine.com.csr

    // 编辑cnf配置文件
    $ vim rancher.mine.com.cnf
    // 输入以下信息
    basicConstraints=CA:FALSE
    subjectAltName=@my_subject_alt_names
    subjectKeyIdentifier = hash

    [ my_subject_alt_names ]
    DNS.1 = *.rancher.mine.com
    DNS.2 = rancher.mine.com

    // :wq进行保存
    // 这里两个DNS配置表示，进行通配证书的签发

    // 最后执行，生成公钥信息
    $ openssl x509 -req -in rancher.mine.com.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out rancher.mine.com.x4.crt -extfile rancher.mine.com.cnf -sha256

    // 验证证书
    $ openssl verify -CAfile ca.crt rancher.mine.com.x4.crt

这样在一个文件夹下，就有多个个证书文件了，分别是下面的几个：

    ca.crt  ca.srl      rancher.mine.com.cnf  rancher.mine.com.key
    ca.key  rancher.mine.com.csr  rancher.mine.com.x4.crt

我们在这里用到的是**rancher.mine.com.key**和**rancher.mine.com.x4.crt**文件。

#### 2. rancher集群部署

执行以下命令：

    // 添加rancher的源仓库
    $ helm repo add rancher-stable https://releases.rancher.com/server-charts/stable

    // 在k8s中创建cattle-system命名空间，只能是这个名字
    $ kubectl create namespace cattle-system

    // 以私有证书的方式进行rancher的安装，标志为：--set ingress.tls.source=secret
    $ helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=rancher.mine.com --set ingress.tls.source=secret

    // 输出如下
    NAME: rancher
    LAST DEPLOYED: Sun Feb 23 13:32:13 2020
    NAMESPACE: cattle-system
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    Rancher Server has been installed.

    NOTE: Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued and Ingress comes up.

    Check out our docs at https://rancher.com/docs/rancher/v2.x/en/

    Browse to https://rancher.mine.org

    Happy Containering!

当出现**Happy Containering!**的时候，就说明安装已经完成了。这时候可以通过以下命令进行测试：

    $ kubectl -n cattle-system rollout status deploy/rancher
    // 输出以下内容时，可认为安装完成
    Waiting for deployment "rancher" rollout to finish: 0 of 3 updated replicas are available...
    deployment "rancher" successfully rolled out

    // 查看服务状态
    $ kubectl -n cattle-system get deploy rancher
    NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    rancher   3         3         3            3           3m

这样当Rancher安装完毕后，需要将私有的签名信息添加到Rancher集群的访问中。操作如下：

    $ kubectl -n cattle-system create secret tls tls-rancher-ingress --cert=rancher.mine.com.x4.crt --key=rancher.mine.com.key
    // 输出下面的内容
    secret/tls-rancher-ingress created

证书生效后，创建tls-rancher-ingress，这样Rancher就可以在服务器内部被访问了。可以随便在232、239、241的服务器上执行：

    $ curl -L rancher.mine.com -k
    // 可以出现JSON数据

### 五、Nginx负载均衡配置

#### 1. Nginx安装配置

    // 安装nginx 

    $ sudo yum install -y nginx

    $ sudo systemctl enable nginx

    // 将上面生成的证书拷贝到nginx安装目录下

    $ cd /etc/nginx

    $ sudo mkdir cert

    $ sudo cp ~/nginx/rancher.mine.com.key /etc/nginx/cert
    
    $ sudo cp ~/nginx/rancher.mine.com.x4.crt /etc/nginx/cert

    // 修改配置文件如下
    $ sudo mv nginx.conf nginx.conf.bak

    $ sudo vim /etc/nginx/nginx.conf
    // 将以下信息填入
    user nginx;
    worker_processes 4;
    worker_rlimit_nofile 40000;

    events {
        worker_connections 8192;
    }

    http {

        upstream rancher_servers {
            least_conn;
            server 192.168.238.232:80 max_fails=3 fail_timeout=5s;
            server 192.168.238.239:80 max_fails=3 fail_timeout=5s;
            server 192.168.238.241:80 max_fails=3 fail_timeout=5s;
        }

        map $http_upgrade $connection_upgrade {
            default Upgrade;
            ''      close;
        }

        gzip on;
        gzip_disable "msie6";
        gzip_disable "MSIE [1-6].(?!.*SV1)";
        gzip_vary on;
        gzip_static on;
        gzip_proxied any;
        gzip_min_length 0;
        gzip_comp_level 8;
        gzip_buffers 16 8k;
        gzip_http_version 1.1;
        gzip_types text/xml application/xml application/atom+xml application/rss+xml application/xhtml+xml image/svg+xml application/font-woff text/javascript application/javascript application/x-javascript text/x-json application/json application/x-web-app-manifest+json text/css text/plain text/x-component font/opentype application/x-font-ttf application/vnd.ms-fontobject font/woff2 image/x-icon image/png image/jpeg;
        server {
            listen         80;
            server_name    rancher.mine.com;
            rewrite ^(.*) https://$server_name$1 permanent;
        }

        server {
            listen     443 ssl;
            server_name    rancher.mine.com;
            ssl_certificate cert/rancher.mine.com.x4.crt;
            ssl_certificate_key cert/rancher.mine.com.key;
            ssl_session_timeout 5m;
            ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
            ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
            ssl_prefer_server_ciphers on;
            location / {
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-Proto $scheme;
                proxy_set_header X-Forwarded-Port $server_port;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_pass http://rancher_servers;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection $connection_upgrade;
            }
        }
    }
    // :wq进行保存后退出                                                 

    // 测试nginx配置文件是否正常
    $ sudo nginx -t
    // 正常后重启Nginx服务
    $ sudo systemctl restart nginx


#### 2. 设置hosts文件地址映射

rancher.com就是后面访问rancher的域名，需要在232、239、241服务器上的/etc/hosts文件中添加关联：

    # echo "192.168.238.240 rancher.mine.com" >> /etc/hosts

在你访问Rancher的web页面的机器上，也要配置hosts，例如在win10中，修改hosts文件，添加

    192.168.238.240 rancher.mine.com

到文件末尾保存即可。这时打开浏览器，直接访问rancher.mine.com即可。如下图：


最后设置管理员账号密码即可，默认密码为admin，调整为我们常用的123456，完成。

后续登录的用户名为admin，密码为123456。地址为rancher.mine.com，记住一定设置host文件将ip和域名地址对应。

## 总结

本次安装选择了较新的版本，一方面是屏蔽之前的bug，另一方面是该版本对于istio的支持较好，后续实施istio的时候可以比较方便。

## 其它问题

在后续运行的过程中，发现有部分服务始终处于CrashLoopBackOff的状态并且重启次数过多，如下：

    $ kubectl get pod -n cattle-system
    NAME                                    READY   STATUS             RESTARTS   AGE
    cattle-cluster-agent-568688f988-lfg6b   0/1     CrashLoopBackOff   627        3d3h
    cattle-node-agent-8h4jg                 0/1     CrashLoopBackOff   888        3d3h
    cattle-node-agent-9qthg                 0/1     CrashLoopBackOff   889        3d3h
    cattle-node-agent-jp2rx                 0/1     CrashLoopBackOff   889        3d3h
    rancher-7d769c47d6-9cdss                1/1     Running            2          3d22h
    rancher-7d769c47d6-rn6kn                1/1     Running            1          3d22h
    rancher-7d769c47d6-wrtbd                1/1     Running            1          3d22h

并且有时在观察过程中node-agent或者cluster-agent处在Running状态，过一会儿又切换为CrashLoopBackOff。这时需要进一步进行排查。操作如下：

    $ kubectl logs -f cattle-cluster-agent-568688f988-lfg6b -n cattle-system
    INFO: Environment: CATTLE_ADDRESS=10.42.2.4 CATTLE_CA_CHECKSUM= CATTLE_CLUSTER=true CATTLE_INTERNAL_ADDRESS= CATTLE_K8S_MANAGED=true CATTLE_NODE_NAME=cattle-cluster-agent-568688f988-lfg6b CATTLE_SERVER=https://rancher.mine.com
    INFO: Using resolv.conf: nameserver 10.43.0.10 search cattle-system.svc.cluster.local svc.cluster.local cluster.local options ndots:5
    ERROR: https://rancher.mine.com/ping is not accessible (Failed to connect to rancher.mine.com port 443: Connection timed out)

cluster-agent并不能进行通讯，看到443问题，猜想是证书出了问题，随便查看一个node-agent的状态，如下：

    $ kubectl logs -f cattle-node-agent-9qthg -n cattle-system
    INFO: Environment: CATTLE_ADDRESS=192.168.188.239 CATTLE_AGENT_CONNECT=true CATTLE_CA_CHECKSUM= CATTLE_CLUSTER=false CATTLE_INTERNAL_ADDRESS= CATTLE_K8S_MANAGED=true CATTLE_NODE_NAME=192.168.188.239 CATTLE_SERVER=https://rancher.mine.com
    INFO: Using resolv.conf: nameserver 114.114.114.114 nameserver 8.8.8.8 nameserver 1.1.1.1
    INFO: https://rancher.mine.com/ping is accessible
    INFO: rancher.mine.com resolves to 192.168.188.240
    time="2020-02-28T06:04:22Z" level=info msg="Rancher agent version v2.3.5 is starting"
    time="2020-02-28T06:04:22Z" level=info msg="Listening on /tmp/log.sock"
    time="2020-02-28T06:04:22Z" level=info msg="Option customConfig=map[address:192.168.188.239 internalAddress: label:map[] roles:[] taints:[]]"
    time="2020-02-28T06:04:22Z" level=info msg="Option etcd=false"
    time="2020-02-28T06:04:22Z" level=info msg="Option controlPlane=false"
    time="2020-02-28T06:04:22Z" level=info msg="Option worker=false"
    time="2020-02-28T06:04:22Z" level=info msg="Option requestedHostname=192.168.188.239"
    time="2020-02-28T06:04:22Z" level=info msg="Certificate details from https://rancher.mine.com"
    time="2020-02-28T06:04:22Z" level=info msg="Certificate #0 (https://rancher.mine.com)"
    time="2020-02-28T06:04:22Z" level=info msg="Subject: CN=*.rancher.mine.com,O=Default Company Ltd,L=Default City,C=CN"
    time="2020-02-28T06:04:22Z" level=info msg="Issuer: CN=rancher.mine.com,O=Default Company Ltd,L=Default City,C=CN"
    time="2020-02-28T06:04:22Z" level=info msg="IsCA: false"
    time="2020-02-28T06:04:22Z" level=info msg="DNS Names: [*.rancher.mine.com rancher.mine.com]"
    time="2020-02-28T06:04:22Z" level=info msg="IPAddresses: <none>"
    time="2020-02-28T06:04:22Z" level=info msg="NotBefore: 2020-02-24 10:32:05 +0000 UTC"
    time="2020-02-28T06:04:22Z" level=info msg="NotAfter: 2020-03-25 10:32:05 +0000 UTC"
    time="2020-02-28T06:04:22Z" level=info msg="SignatureAlgorithm: SHA256-RSA"
    time="2020-02-28T06:04:22Z" level=info msg="PublicKeyAlgorithm: RSA"
    time="2020-02-28T06:04:22Z" level=fatal msg="Certificate chain is not complete, please check if all needed intermediate certificates are included in the server certificate (in the correct order) and if the cacerts setting in Rancher either contains the correct CA certificate (in the case of using self signed certificates) or is empty (in the case of using a certificate signed by a recognized CA). Certificate information is displayed above. error: Get https://rancher.mine.com: x509: certificate signed by unknown authority"

出现了x509问题，但是查询证书信息已经按照上面的教程导入了。如下：

    $ kubectl get secrets -n cattle-system
    NAME                            TYPE                                  DATA   AGE
    cattle-credentials-67b70c2      Opaque                                2      3d3h
    cattle-token-r5q62              kubernetes.io/service-account-token   3      3d3h
    default-token-hd9nl             kubernetes.io/service-account-token   3      3d23h
    rancher-token-2kqn9             kubernetes.io/service-account-token   3      3d23h
    sh.helm.release.v1.rancher.v1   helm.sh/release.v1                    1      3d23h
    tls-rancher                     kubernetes.io/tls                     2      3d22h
    tls-rancher-ingress             kubernetes.io/tls                     2      3d19h

    $ kubectl describe secret tls-rancher-ingress -n cattle-system
    Name:         tls-rancher-ingress
    Namespace:    cattle-system
    Labels:       <none>
    Annotations:  <none>

    Type:  kubernetes.io/tls

    Data
    ====
    tls.key:  1675 bytes
    tls.crt:  1334 bytes

但是不影响使用，rancher可以照常进行访问。

## 参考文章

* 官网安装文档：https://rancher.com/docs/rancher/v2.x/en/installation/
* 新版本发布：https://www.cnblogs.com/rancherlabs/p/11956279.html
* rancher中文社区：https://forums.cnrancher.com/
* 部署文章：https://www.dazhuanlan.com/2019/10/25/5db2df23374e5/
* 部署文章：https://blog.51cto.com/bilibili/2440304
* 部署文章：http://www.eryajf.net/2723.html
* 部署文章：https://fs.tn/post/PmaL-uIiQ/
* k8s部署文章：https://www.pocketdigi.com/2019-09/install-k8s-in-china.html
* 常见问题：http://blog.ponycool.com/archives/94.html
* 常见问题：https://blog.csdn.net/isea533/article/details/98172030
* 视频教程：https://space.bilibili.com/430496045?from=search&seid=8060734016441754889
* 私有证书签发：https://linkscue.com/posts/2019-06-26-ca-and-self-signed-certificates/
* nginx证书安装：https://www.jianshu.com/p/02c25d0dd451