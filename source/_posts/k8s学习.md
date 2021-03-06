k8s学习

## 第一章 kubeadm方式安装k8s集群

1. 安装部分

安装前注意：

1. 不要使用中文目录或者克隆过的虚拟机（网卡问题）
2. 生产环境建议使用二进制安装
3. VIP不要和内网ip地址重复，需要和主机ip在同一个局域网段内

1.1 容器化安装

第一部分：初始化
在所有机器上同步执行

vim /etc/hosts

192.168.229.51 k8s-master-01
192.168.229.52 k8s-master-02
192.168.229.53 k8s-master-03
192.168.229.60 k8s-master-lb
192.168.229.54 k8s-node-01
192.168.229.55 k8s-node-02


修改yum源

mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak

wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

yum install -y yum-utils \
                    device-mapper-persistent-data \
                    lvm2

yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF


sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo

yum update -y --exclude=kernel* 


安装必备工具
yum install -y wget git unzip net-tools curl vim telnet psmisc  tree sshpass jq

关闭防火墙，selinux、dnsmasq、NetworkManager
systemctl disable --now firewalld
systemctl disable --now dnsmasq
systemctl disable --now NetworkManager

setenforce 0
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/sysconfig/selinux
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config

注意：selinux关闭时必须两个同时修改

关闭swap分区
swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab


时钟同步,时钟不一致可能会导致证书出现问题

rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm
yum install ntpdate -y

设置时区
ln -df /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo 'Asia/Shanghai' > /etc/timezone
ntpdate time2.aliyun.com
crontab -e
*/5 * * * * ntpdate time2.aliyun.com

临时设置ulimit
ulimit -SHn 65535

永久生效方式
vim /etc/security/limits.conf

* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited

在master01上生成证书并传输到其它机器上

ssh-keygen -t rsa
for i in k8s-master-02 k8s-master-03 k8s-node-01 k8s-node-02; do ssh-copy-id -i /root/.ssh/id_rsa.pub $i;done

第二部分：内核配置

更新并重启机器，排除内核信息
yum update -y --exclude=kernel* && reboot

在生产环境中必须进行内核升级，CentOS 7需要内核版本在4.18+。 这里使用4.19版本

cd /root
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm

可以使用idm下载后再上传到各个服务器上。

将内核文件传输到各个机器上
for i in k8s-master-02 k8s-master-03 k8s-node-01 k8s-node-02; do scp kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm $i:/root/ ;done

批量在各个机器上执行以下命令
cd /root/ && yum localinstall kernel-ml* -y

更改内核启动顺序
grub2-set-default 0 && grub2-mkconfig -o /etc/grub2.cfg
grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"

检查内核版本
grubby --default-kernel
/boot/vmlinuz-4.19.12-1.el7.elrepo.x86_64

所有节点重启，再检查内核信息

第三部分：安装ipvs工具

生产环境中推荐使用ipvs工具，不推荐使用iptables
yum install ipvsadm ipset sysstat conntrack libseccomp -y

所有内核节点配置ipvs模块，4.19+版本内核需要把nf_conntrack_ipv4更改为nf_conntrack，4.18及以下可以继续使用。
vim /etc/modules-load.d/ipvs.conf
//添加
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lbtc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip

systemctl enable --now systemd-modules-load.service
// 校验是否开启，如果无输出信息，需要重启机器
lsmod | grep -e ip_vs -e nf_conntrack

开启内核参数

cat >> /etc/sysctl.d/k8s.conf<<EOF
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
fs.may_datch_mounts=1
vm.overcommit_memory=1
vm.panic_on_oom=1
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720

net.ipv4.tcp_keepalive_time=600
net.ipv4.tcp_keepalive_probes=3
net.ipv4.tcp_keepalive_intvl=15
net.ipv4.tcp_max_tw_buckets=36000
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_max_orphans=327680
net.ipv4.tcp_orphan_retries=3
net.ipv4.tcp_syncookies=1
net.ipv4.tcp_max_syn_backlog=16384
net.ipv4.ip_conntrack_max=65536
net.ipv4.tcp_timestamps=0
net.core.somaxconn=16384
EOF

sysctl --system

重启后进行验证
lsmod | grep -e ip_vs -e nf_conntrack
ip_vs_ftp              16384  0
nf_nat                 32768  1 ip_vs_ftp
ip_vs_sed              16384  0
ip_vs_nq               16384  0
ip_vs_fo               16384  0
ip_vs_sh               16384  0
ip_vs_dh               16384  0
ip_vs_lblcr            16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs_wlc              16384  0
ip_vs_lc               16384  0
ip_vs                 151552  22 ip_vs_wlc,ip_vs_rr,ip_vs_dh,ip_vs_lblcr,ip_vs_sh,ip_vs_fo,ip_vs_nq,ip_vs_wrr,ip_vs_lc,ip_vs_sed,ip_vs_ftp
nf_conntrack          143360  2 nf_nat,ip_vs
nf_defrag_ipv6         20480  1 nf_conntrack
nf_defrag_ipv4         16384  1 nf_conntrack
libcrc32c              16384  4 nf_conntrack,nf_nat,xfs,ip_vs

第四部分：docker以及kubenetes组件安装

安装docker
yum install docker-ce-19.03.* -y

修改cgroup运行方式
mkdir /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
    "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

systemctl daemon-reload && systemctl enable --now docker

安装k8s组件

注意，版本号后面小版本大于5时再去选择使用该版本运行在生产环境中，例如1.18.9、1.19.5这样版本，尽量不要选择如1.19.1、1.18.3这样的版本。

查看最新版本的组件信息
yum list kubeadm --showduplicates |sort -r

安装kubeadm
yum install kubeadm -y

安装完成后，需要进行设置。注意：--cgroup-driver 一定要指定为和docker相同的systemd，否则容易导致k8s无法启动。

默认配置的pause镜像使用gcr.io仓库，国内访问科学上网问题，需要配置使用阿里云的镜像。

cat >/etc/sysconfig/kubelet<<EOF
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.2"
EOF

配置开机启动。未初始化之前可能存在启动失败的问题，临时不需要关心，由于kubelet没有配置文件，所以无法启动，待初始化后可用
systemctl daemon-reload && systemctl enable --now kubelet

第五部分：高可用组件安装

下面操作针对所有master节点，进行安装。

安装HAProxy和keepalived

yum install haproxy keepalived -y

编写HAProxy的配置文件，三个节点的配置文件相同

vim /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    timeout http-request    15s
    timeout connect         5000
    timeout client          50000
    timeout server          50000
    timeout http-keep-alive 15s


#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend monitor-in
    bind *:33305
    mode http
    option httplog
    monitor-uri /monitor

frontend k8s-master
    bind 0.0.0.0:16443
    bind 127.0.0.1:16443
    mode tcp
    option tcplog
    tcp-request inspect-delay 5s
    default_backend k8s-master

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend k8s-master
    mode tcp
    option tcplog
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server  k8s-master01 192.168.229.51:6443 check
    server  k8s-master02 192.168.229.52:6443 check
    server  k8s-master03 192.168.229.53:6443 check

编写keepalived的配置，注意区分每个节点的ip地址和网卡信息：

vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL
   script_user root
   enable_script_security
}

vrrp_script chk_apiserver {
   script "/etc/keepalived/check_apiserver.sh"
   interval 5
   weight -5
   fall 2
   rise 1
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33               # 这个位置为网卡信息，需要更改
    mcast_src_ip 192.168.229.51   # 这个位置是宿主机的真实IP地址，需要更改
    virtual_router_id 51
    priority 101
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.229.60    # 注意这个位置是设置的VIP
    }
    track_script {
        chk_apiserver
    }
}

注意健康检查track_script部分要打开。

所有的master节点需要配置keepalived的健康检查脚本

vim /etc/keepalived/check_apiserver.sh
#!/bin/bash

err=0
for i in $(seq 1 3)
do
  check_code=$(pgrep haproxy)
  if [[ $check_code == "" ]]; then
    err=$(expr $err+1)
    sleep 1
    continue
  else
    err=0
    break
  fi
done

if [[ $err != "0" ]]; then
  echo "systemctl stop keepalived"
  /usr/bin/systemctl stop keepalived
  exit 1
else
  exit 0
fi

赋予脚本可执行权限
chmod +x /etc/keepalived/check_apiserver.sh

启动haproxy和keepalived
systemctl daemon-reload && systemctl enable --now haproxy && keepalived

查看生效状况
ip addr | grep 60
    inet 192.168.229.60/32 scope global ens33
    inet6 fe80::20c:29ff:febf:7608/64 scope link

telnet 192.168.229.51 16443
Trying 192.168.229.51...
Connected to 192.168.229.51.
Escape character is '^]'.
Connection closed by foreign host.

问题排查：VIP ping不通，而且telnet没有出现]标志
查看keepalived 是否存在问题，查看keepalived运行状态，查看selinux、haproxy和keepalived的状态和监听端口

第五部分：集群初始化

下面的命令在master节点上执行

创建kubeadm初始化的yaml文件，最好使用配置文件去执行，否则使用命令时指定的参数过多容易导致执行失败。

vim kubeadm-config.yaml

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta2
# 设置加入集群的token信息，24小时内有效，过期后需要处理
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: 7t2weq.bjbawausm0jaxury
  ttl: 24h0m0s
  usages:
  - signing
  - authentication 
kind: InitConfiguration
# 本机信息配置，ip地址需要配置为本机的ip地址
localAPIEndpoint:
  advertiseAddress: 192.168.229.51
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-master-01
  # 添加一个容忍，配置master节点不允许部署容器
  taints:
  - effect: NoSchedule
    key: node-role.kubenetes.io/master
---
apiServer:
  certSANs:
  - 192.168.229.60
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: 192.168.229.60:16443
controllerManager: {}
dns: 
  type: CoreDNS
etcd:
  local:    
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.20.1
networking: 
  dnsDomain: cluster.local  
  podSubnet: 172.168.0.0/12
  serviceSubnet: 10.96.0.0/12
scheduler: {}

```

更新kubadm-config.yaml
kubeadm config migrate --old-config kubeadm-config.yaml --new-config new.yaml

更新完成后将其它文件复制到其它master节点
for i in k8s-master-02 k8s-master-03 ; do scp new.yaml $i:/root/ ;done

下载镜像
kubeadm config images pull --config /root/new.yaml

在master01节点执行k8s集群的初始化
kubeadm init --config /root/new.yaml --upload-certs

[init] Using Kubernetes version: v1.20.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master-01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.229.51 192.168.229.60]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master-01 localhost] and IPs [192.168.229.51 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master-01 localhost] and IPs [192.168.229.51 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "admin.conf" kubeconfig file
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 30.040077 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.20" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
afe317744fa848eceadb7df6960133e77c96f9c282656d361706b7e515a4286f
[mark-control-plane] Marking the node k8s-master-01 as control-plane by adding the labels "node-role.kubernetes.io/master=''" and "node-role.kubernetes.io/control-plane='' (deprecated)"
[mark-control-plane] Marking the node k8s-master-01 as control-plane by adding the taints [node-role.kubemetes.io/naster:NoSchedule]
[bootstrap-token] Using token: 7t2weq.bjbawausm0jaxury
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.229.60:16443 --token 7t2weq.bjbawausm0jaxury \
    --discovery-token-ca-cert-hash sha256:246203ceb832292331df2fedae4e1db305859ce79cab901873d4410af5403242 \
    --control-plane --certificate-key afe317744fa848eceadb7df6960133e77c96f9c282656d361706b7e515a4286f

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.229.60:16443 --token 7t2weq.bjbawausm0jaxury \
    --discovery-token-ca-cert-hash sha256:246203ceb832292331df2fedae4e1db305859ce79cab901873d4410af5403242


在master01节点配置环境变量，用于访问kubernetes集群：
cat <<EOF >> /root/.bashrc
export KUBECONFIG=/etc/kubernetes/admin.conf
EOF

source /root/.bashrc

这时候配置完成，可以使用kubectl命令来进行集群的管理
kubectl get node
NAME            STATUS     ROLES                  AGE     VERSION
k8s-master-01   NotReady   control-plane,master   4m36s   v1.20.1

kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   5m12s

kubectl get pod -n kube-system -o wide
NAME                                    READY   STATUS    RESTARTS   AGE     IP               NODE            NOMINATED NODE   READINESS GATES
coredns-54d67798b7-66z7x                0/1     Pending   0          6m27s   <none>           <none>          <none>           <none>
coredns-54d67798b7-jzql9                0/1     Pending   0          6m27s   <none>           <none>          <none>           <none>
etcd-k8s-master-01                      1/1     Running   0          6m21s   192.168.229.51   k8s-master-01   <none>           <none>
kube-apiserver-k8s-master-01            1/1     Running   0          6m21s   192.168.229.51   k8s-master-01   <none>           <none>
kube-controller-manager-k8s-master-01   1/1     Running   0          6m21s   192.168.229.51   k8s-master-01   <none>           <none>
kube-proxy-pqh28                        1/1     Running   0          6m26s   192.168.229.51   k8s-master-01   <none>           <none>
kube-scheduler-k8s-master-01            1/1     Running   0          6m21s   192.168.229.51   k8s-master-01   <none>           <none>

第六部分：master节点高可用

注意master节点加入命令和node节点加入命令有所不同

master节点加入命令：

  kubeadm join 192.168.229.60:16443 --token 7t2weq.bjbawausm0jaxury \
    --discovery-token-ca-cert-hash sha256:246203ceb832292331df2fedae4e1db305859ce79cab901873d4410af5403242 \
    --control-plane --certificate-key afe317744fa848eceadb7df6960133e77c96f9c282656d361706b7e515a4286f

node节点加入命令

  kubeadm join 192.168.229.60:16443 --token 7t2weq.bjbawausm0jaxury \
    --discovery-token-ca-cert-hash sha256:246203ceb832292331df2fedae4e1db305859ce79cab901873d4410af5403242

区别在于是否指定了control-plane相关的正式信息


首先将k8s-master-02机器加入master节点，加入后在k8s-master-01中执行

kubectl get node
NAME            STATUS     ROLES                  AGE    VERSION
k8s-master-01   NotReady   control-plane,master   14m    v1.20.1
k8s-master-02   NotReady   control-plane,master   3m9s   v1.20.1

如果出现了token过期，造成master节点无法加入的情况，处理如下：

一、 生成新的token信息
kubeadm token create --print-join-command
kubeadm join 192.168.229.60:16443 --token dvm83r.k0gpb2vqpsg49zak     --discovery-token-ca-cert-hash sha256:246203ceb832292331df2fedae4e1db305859ce79cab901873d4410af5403242

二、 生成新的certificate-key
kubeadm init phase upload-certs --upload-certs
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
8222ab7c36a228960e05327d352e1cf12716d3fe536abac8620ec4dd250c2098

将k8s-master-03加入k8s集群
kubeadm join 192.168.229.60:16443 --token dvm83r.k0gpb2vqpsg49zak     --discovery-token-ca-cert-hash sha256:246203ceb832292331df2fedae4e1db305859ce79cab901873d4410af5403242 --control-plane --certificate-key 8222ab7c36a228960e05327d352e1cf12716d3fe536abac8620ec4dd250c2098

查看所有节点信息
kubectl get node
NAME            STATUS     ROLES                  AGE   VERSION
k8s-master-01   NotReady   control-plane,master   35m   v1.20.1
k8s-master-02   NotReady   control-plane,master   23m   v1.20.1
k8s-master-03   NotReady   control-plane,master   81s   v1.20.1

同理，将k8s-node-01、k8s-node-02作为node节点加入，
kubeadm join 192.168.229.60:16443 --token dvm83r.k0gpb2vqpsg49zak     --discovery-token-ca-cert-hash sha256:246203ceb832292331df2fedae4e1db305859ce79cab901873d4410af5403242

最终查看所有节点信息
 kubectl get node
NAME            STATUS     ROLES                  AGE     VERSION
k8s-master-01   NotReady   control-plane,master   39m     v1.20.1
k8s-master-02   NotReady   control-plane,master   27m     v1.20.1
k8s-master-03   NotReady   control-plane,master   5m42s   v1.20.1
k8s-node-01     NotReady   <none>                 28s     v1.20.1
k8s-node-02     NotReady   <none>                 16s     v1.20.1

最后通过配置文件安装calico网络组件

编辑calico.yaml文件

```yaml

---
# Source: calico/templates/calico-etcd-secrets.yaml
# The following contains k8s Secrets for use with a TLS enabled etcd cluster.
# For information on populating Secrets, see http://kubernetes.io/docs/user-guide/secrets/
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: calico-etcd-secrets
  namespace: kube-system
data:
  # Populate the following with etcd TLS configuration if desired, but leave blank if
  # not using TLS for etcd.
  # The keys below should be uncommented and the values populated with the base64
  # encoded contents of each file that would be associated with the TLS data.
  # Example command for encoding a file contents: cat <file> | base64 -w 0
  # 修改2：etcd的证书文件信息添加
  etcd-key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBOEJmajZxb2VMNVE4cldRcGgyMTdrdy9xYnU1aXljWkhUa0kwd3BxWTJIV1FlMlNxCjRkYVg1VmFleFlwT2dzSzlBTE5BbmtKSjVCeFkrYWdLNFBpNGo3eVkrNTNwVkJqNDZzWjlyTmxkQzVwS2w2aE0KSUd4UHNubklyRFB4aTZnOE5HeW5YaVZFOEh4TUxGclJuQlpvaFVqV2dDMkVhamN0WW8zWWFQMzhUTC9TKytPNwpPcnM1RVdhZ2FyYUhJUXVzaiszOWtaeGF2ZE1uVi82MktrSE9pQnFpRkZTWVRqWTgxK2VncnlnNTVKcDg2UGdMCkplZ0orcWpsK2pMVCtucTFhTTFSbXdiMmozRkVSVUJ2a1htRFVPdVlpRWZRQWU4dmgxaVFVMnNRZG9CUWQ5Ti8Kb25zMkRieGE3R0pjT1VsdHI0SlJVcU96SDZHMjNMSk1XWkE4QXdJREFRQUJBb0lCQUYyUXNkMk5sbDNzWXdrZgpjNSszWnVVVTJzT0lXeTlPK2hMaGNqWTBrVVFwN0xocHJyNThKbzNWaCtKcjE5VFZsMXBpZ05ncjlTZlVkRWcyCjJLWjd4MUVjcW5IRVJGM2xyWHV4QnVFSmhGMDFMOFNTYmJoay9Wb01ZOHZZSWxYT3BrZTM0REdzVElWN3F5UE4KOE1ubllhd3Zpb2hCTk0wLzI0d0F3MG1IVVgrR3NLVEZ4WVNiY1VQZko0ZndiNVEyVGxrS0lJcXFUdXhOdjR5dQpnK3RQeTFzNWd2WmtkTEhFcmt1ODREWWVhbW1mbFN5SFZIejFNVnVPejhrdkpWYXN2QjZEampnQTNKcDZWL0IzCjZzUXR5OGNDa3BuSUVkRWcxdTEwV3lLU2hsSTRrTkZUUWVsckRwWGwxUWVKbERma1dlOTNZZVJEWFQ4bm4xWXQKbjdFZEo4RUNnWUVBL2R3cDJ2YTZtc3AzWU1VZC9OWEg5Zk8wVCtTMlpNTFdSazE4emZRaUlHcDdEOHFDbG5SLwoxS0xQdmF3V3hlbytnY2JBYjFPNnlIZGhMNXBNejRVMWdjS3g1dVJid29qSzJqZm53dnhoSTExdWUxaHRjcmIyCk4xR1Evc0xDL0dxdzNrdzdiMnY0OUJmVzc5aFJVOVhPZjY1OGVnbnB6ZXpEZzNNMjdzY2k1TmtDZ1lFQThoNEUKbHBiVHNickRCK3ppSGhMcDRJNGJnd3dzVDFMZDk5U0ZNRko1R21FWHdGRTBGaE5hbDFxTXBCTVVWYUJlVVlqNQpGNHhROU5oNlpkTnJnVXJleGhSZWFCSmZkYXFhMTgwZUJqdEhjaERHRytaK2phSlp5OE5SaHF1VlEra3VLRmFkCk0vaFFPR2laejlJZ094ZThxbDRIRWZ3MjJtTmZlcG5UV21DS3Jqc0NnWUVBcll6S2dJdVUzeVh6bnhDamc2cVQKWGE0U1kxdzA1WVhkLzRvUi9Lc2VlWkxTTnVWM2lXeHp4K2JXcHhEek1MTUhzS2t6L2VmOEZmaW5WR2ZrZ3lySwpmYitnNS96T1RwdytNaGx1Tkh0ZDNWT09xSHkzdG1rbXdvTGM0WTQ4eDF3Wk5xQmZNYmxiSldUMjZGbTJuOTNYCm9xcWpKcnVJUCtQUmRoaGFRYnVhTzJFQ2dZQnRBWHJMV2NpaG9oWWd3VlBrZWx0MTBFVXVzUkpaL0ZNWE8wVmoKeGgzajlJYSsvVkJZQ0FxblRnczM2NmNpRGZ1bzllUS81OXFqQWJ2SmtIQThXN3NFcnpMNTVCdTZYRDh1blpqQQo4WHR2TFlJa0daZ3NxRVdKYWJ5UXh6dUN3YjhZUmphc3FVVmt3Q05QMzZqSE1oNnREWHhkYXBJL3JMSFYvdCtiCk53LzQ5UUtCZ0U3Slk4ZHUzQlRrWE5XdDF5NXZUL0hINit4RktTMVo0R0ZMTktPMFNsdlBqN05QSWhuMjNkMkMKa3U3M0VMeDhzUUQvWUVVbUxsZGsyc2FRbWlJZzRhNnV0aDA3MW8rU01GWm9IZzlxNHVpS3l6R0dCSHZPU1k0egpURlJmaU14TXpwMXhqZ2w2bG5NU2VKZnZpZThJckJGWTAwRzBCSXFFWm1saTdEN00ybyszCi0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
  etcd-cert: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURRekNDQWl1Z0F3SUJBZ0lJT2dtLzhHRDliQzB3RFFZSktvWklodmNOQVFFTEJRQXdFakVRTUE0R0ExVUUKQXhNSFpYUmpaQzFqWVRBZUZ3MHlNVEF4TURreE1EUXpNalJhRncweU1qQXhNRGt4TURRek1qVmFNQmd4RmpBVQpCZ05WQkFNVERXczRjeTF0WVhOMFpYSXRNREV3Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUUR3RitQcXFoNHZsRHl0WkNtSGJYdVREK3B1N21MSnhrZE9RalRDbXBqWWRaQjdaS3JoMXBmbFZwN0YKaWs2Q3dyMEFzMENlUWtua0hGajVxQXJnK0xpUHZKajduZWxVR1BqcXhuMnMyVjBMbWtxWHFFd2diRSt5ZWNpcwpNL0dMcUR3MGJLZGVKVVR3ZkV3c1d0R2NGbWlGU05hQUxZUnFOeTFpamRoby9meE12OUw3NDdzNnV6a1JacUJxCnRvY2hDNnlQN2YyUm5GcTkweWRYL3JZcVFjNklHcUlVVkpoT05qelg1NkN2S0Rua21uem8rQXNsNkFuNnFPWDYKTXRQNmVyVm96VkdiQnZhUGNVUkZRRytSZVlOUTY1aUlSOUFCN3krSFdKQlRheEIyZ0ZCMzAzK2llellOdkZycwpZbHc1U1cydmdsRlNvN01mb2JiY3NreFprRHdEQWdNQkFBR2pnWll3Z1pNd0RnWURWUjBQQVFIL0JBUURBZ1dnCk1CMEdBMVVkSlFRV01CUUdDQ3NHQVFVRkJ3TUJCZ2dyQmdFRkJRY0RBakFmQmdOVkhTTUVHREFXZ0JTNERCbE8KeEFVVFVQQjdyYXJzeGE0eGtNenAxekJCQmdOVkhSRUVPakE0Z2cxck9ITXRiV0Z6ZEdWeUxUQXhnZ2xzYjJOaApiR2h2YzNTSEJNQ281VE9IQkg4QUFBR0hFQUFBQUFBQUFBQUFBQUFBQUFBQUFBRXdEUVlKS29aSWh2Y05BUUVMCkJRQURnZ0VCQUJTVjV4NGZRVCtvN1JiUnYrZWxCd1VUc3A0dmVWbFVZNVB2LzJjUWE2b0h1dUZ3eHFlYmZjRkMKcTVlbHIzUHMxUVUrTERtMUFXeFhTUTRqaHhiTUZ0ZEVtdGxaR0I4YUhaWU95dnpDTVhOcGVWRVArUTNTWUZKQwpSdHNuSmFrQ0ZTS2ZTcVliMzYxZHRUOCtpc2tzL0ZSTUlNMFVOZ1J1U2xRM0prdmRlOGNleitQWUliR0RkQUlTCkIxZ25WWTRNRVowNXVOSmxNU05jcXZyQy9JNHdjdXV6YURkdVBZNlVQMjJLYjZKay9CdWlkKzdHVmF5QUdBNksKN0E3RDJiRUR6a3RtQm1Hei9zeHBGTW5URnh1V3F1eEVybnRWanVkV085T1pSREJadTBWVHA3aHBHUG9KTkJqVgpCZG1tVzZ6QU4xd0ZsNlJpWk9rbzRoanNEUVVHeXVNPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  etcd-ca: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM0VENDQWNtZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFTTVJBd0RnWURWUVFERXdkbGRHTmsKTFdOaE1CNFhEVEl4TURFd09URXdORE15TkZvWERUTXhNREV3TnpFd05ETXlORm93RWpFUU1BNEdBMVVFQXhNSApaWFJqWkMxallUQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQUpQcUsxUVVQNzBFCmJYOWpXYysyQ0FTVi9qRVo2T1ZrTkVyTmE0TC9POThZWmtKOWN6a3ZvTS9nSUIzcGFVemc1SmViazZFUnpFTjAKV0dCdU9vWnJuNHNUOERDWDc4U1N0TkxmOXpaOERvbjBVZXNRVUdnRnpQOE4vWTdlckxUT3dLMEhNVmNxSE5NaQpLb2labEFuc1piWGhxYnh0MDRJemIwckxxejZXcmcxdXZoWS9Iam52MDVjUjRZN094UTFLalBIZDR2cHhzZ1pZCjl6Lzhnem9FV3pTSlY4cDB4TzUzejNFN3NTcTBzOWJQdWtvOWp1Vm1Ra0RhQXE0aStBOGR3R25lU2hQRWJXZFMKTGVKMTNlYlpiRVN1OTZ4VVBXRTlTQndqNGpWVExqSi9XZ1VXZ043TXFNc2RtWGkxWGhDL2pIRFErNmEyV2FRVwpRRG1EL3lpUUtBOENBd0VBQWFOQ01FQXdEZ1lEVlIwUEFRSC9CQVFEQWdLa01BOEdBMVVkRXdFQi93UUZNQU1CCkFmOHdIUVlEVlIwT0JCWUVGTGdNR1U3RUJSTlE4SHV0cXV6RnJqR1F6T25YTUEwR0NTcUdTSWIzRFFFQkN3VUEKQTRJQkFRQ0ZPVkdRekZYbXRXVFFlNVZqRE9GTmZVSWVvbkZ6RDJRaE5FZ21zak9MMWdQcThmQlhQWDB4cFFMcwpvcTgza2lNL1hYZnI5cmxaR1A2NXJZVXYvcHJOS3kyZlZFYmJOYUxGRi83WUdPV3EwU1RUSk1vVXdBS1ltS0IvCnE0U2QvNXk2U1hvYmEzTmZqeDhzQnNKcTNBRXBXcmlyd3JyWGtoVHMzdW1RMTVBb0ZQdGpYaG5WTkt5UTNjbSsKVk9PSk81U0M0bnpaQndwbms2bUFtMXhaZmlXY2xXbDgyVmRKWjkxYm9qVEJlWUoxeGRyMlhIUHZsYllNMjk5NAp3V0M2V05PRG5lamZ2U0RMenJoYVJOdkJVTytzUGE1cENTQW5jRlNVUG1Xemg1d0QyVVdJbmMveGlqSkZSVVFWCjVGV3NQUWtIV0VmM3daa2hrYlR0NXFXTHpmcjUKLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
---
# Source: calico/templates/calico-config.yaml
# This ConfigMap is used to configure a self-hosted Calico installation.
kind: ConfigMap
apiVersion: v1
metadata:
  name: calico-config
  namespace: kube-system
data:
  # Configure this with the location of your etcd cluster.
  # 修改1：修改为ETCD的节点
  # 记住这个位置一定要使用https，否则会无法连接到etcd
  etcd_endpoints: "https://192.168.229.51:2379,https://192.168.229.52:2379,https://192.168.229.53:2379"
  # If you're using TLS enabled etcd uncomment the following.
  # You must also populate the Secret below with these files.
  etcd_ca: "/calico-secrets/etcd-ca"   # "/calico-secrets/etcd-ca"
  etcd_cert: "/calico-secrets/etcd-cert" # "/calico-secrets/etcd-cert"
  etcd_key: "/calico-secrets/etcd-key"  # "/calico-secrets/etcd-key"
  # Typha is disabled.
  typha_service_name: "none"
  # Configure the backend to use.
  calico_backend: "bird"
  # Configure the MTU to use for workload interfaces and tunnels.
  # - If Wireguard is enabled, set to your network MTU - 60
  # - Otherwise, if VXLAN or BPF mode is enabled, set to your network MTU - 50
  # - Otherwise, if IPIP is enabled, set to your network MTU - 20
  # - Otherwise, if not using any encapsulation, set to your network MTU.
  veth_mtu: "1440"

  # The CNI network configuration to install on each node. The special
  # values in this config will be automatically populated.
  cni_network_config: |-
    {
      "name": "k8s-pod-network",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "calico",
          "log_level": "info",
          "etcd_endpoints": "__ETCD_ENDPOINTS__",
          "etcd_key_file": "__ETCD_KEY_FILE__",
          "etcd_cert_file": "__ETCD_CERT_FILE__",
          "etcd_ca_cert_file": "__ETCD_CA_CERT_FILE__",
          "mtu": __CNI_MTU__,
          "ipam": {
              "type": "calico-ipam"
          },
          "policy": {
              "type": "k8s"
          },
          "kubernetes": {
              "kubeconfig": "__KUBECONFIG_FILEPATH__"
          }
        },
        {
          "type": "portmap",
          "snat": true,
          "capabilities": {"portMappings": true}
        },
        {
          "type": "bandwidth",
          "capabilities": {"bandwidth": true}
        }
      ]
    }

---
# Source: calico/templates/calico-kube-controllers-rbac.yaml

# Include a clusterrole for the kube-controllers component,
# and bind it to the calico-kube-controllers serviceaccount.
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-kube-controllers
rules:
  # Pods are monitored for changing labels.
  # The node controller monitors Kubernetes nodes.
  # Namespace and serviceaccount labels are used for policy.
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
      - serviceaccounts
    verbs:
      - watch
      - list
      - get
  # Watch for changes to Kubernetes NetworkPolicies.
  - apiGroups: ["networking.k8s.io"]
    resources:
      - networkpolicies
    verbs:
      - watch
      - list
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-kube-controllers
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: calico-kube-controllers
subjects:
- kind: ServiceAccount
  name: calico-kube-controllers
  namespace: kube-system
---

---
# Source: calico/templates/calico-node-rbac.yaml
# Include a clusterrole for the calico-node DaemonSet,
# and bind it to the calico-node serviceaccount.
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: calico-node
rules:
  # The CNI plugin needs to get pods, nodes, and namespaces.
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
    verbs:
      - get
  - apiGroups: [""]
    resources:
      - endpoints
      - services
    verbs:
      # Used to discover service IPs for advertisement.
      - watch
      - list
  # Pod CIDR auto-detection on kubeadm needs access to config maps.
  - apiGroups: [""]
    resources:
      - configmaps
    verbs:
      - get
  - apiGroups: [""]
    resources:
      - nodes/status
    verbs:
      # Needed for clearing NodeNetworkUnavailable flag.
      - patch

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: calico-node
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: calico-node
subjects:
- kind: ServiceAccount
  name: calico-node
  namespace: kube-system

---
# Source: calico/templates/calico-node.yaml
# This manifest installs the calico-node container, as well
# as the CNI plugins and network config on
# each master and worker node in a Kubernetes cluster.
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: calico-node
  namespace: kube-system
  labels:
    k8s-app: calico-node
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      hostNetwork: true
      tolerations:
        # Make sure calico-node gets scheduled on all nodes.
        - effect: NoSchedule
          operator: Exists
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
        - effect: NoExecute
          operator: Exists
      serviceAccountName: calico-node
      # Minimize downtime during a rolling upgrade or deletion; tell Kubernetes to do a "force
      # deletion": https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods.
      terminationGracePeriodSeconds: 0
      priorityClassName: system-node-critical
      initContainers:
        # This container installs the CNI binaries
        # and CNI network config file on each node.
        - name: install-cni
          image: registry.cn-beijing.aliyuncs.com/dotbalo/cni:v3.15.3
          command: ["/install-cni.sh"]
          env:
            # Name of the CNI config file to create.
            - name: CNI_CONF_NAME
              value: "10-calico.conflist"
            # The CNI network config to install on each node.
            - name: CNI_NETWORK_CONFIG
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: cni_network_config
            # The location of the etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # CNI MTU Config variable
            - name: CNI_MTU
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: veth_mtu
            # Prevents the container from sleeping forever.
            - name: SLEEP
              value: "false"
          volumeMounts:
            - mountPath: /host/opt/cni/bin
              name: cni-bin-dir
            - mountPath: /host/etc/cni/net.d
              name: cni-net-dir
            - mountPath: /calico-secrets
              name: etcd-certs
          securityContext:
            privileged: true
        # Adds a Flex Volume Driver that creates a per-pod Unix Domain Socket to allow Dikastes
        # to communicate with Felix over the Policy Sync API.
        - name: flexvol-driver
          image: registry.cn-beijing.aliyuncs.com/dotbalo/pod2daemon-flexvol:v3.15.3
          volumeMounts:
          - name: flexvol-driver-host
            mountPath: /host/driver
          securityContext:
            privileged: true
      containers:
        # Runs calico-node container on each Kubernetes node. This
        # container programs network policy and routes on each
        # host.
        - name: calico-node
          image: registry.cn-beijing.aliyuncs.com/dotbalo/node:v3.15.3
          env:
            # The location of the etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # Location of the CA certificate for etcd.
            - name: ETCD_CA_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_ca
            # Location of the client key for etcd.
            - name: ETCD_KEY_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_key
            # Location of the client certificate for etcd.
            - name: ETCD_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_cert
            # Set noderef for node controller.
            - name: CALICO_K8S_NODE_REF
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            # Choose the backend to use.
            - name: CALICO_NETWORKING_BACKEND
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: calico_backend
            # Cluster type to identify the deployment type
            - name: CLUSTER_TYPE
              value: "k8s,bgp"
            # Auto-detect the BGP IP address.
            - name: IP
              value: "autodetect"
            # Enable IPIP
            - name: CALICO_IPV4POOL_IPIP
              value: "Always"
            # Enable or Disable VXLAN on the default IP pool.
            - name: CALICO_IPV4POOL_VXLAN
              value: "Never"
            # Set MTU for tunnel device used if ipip is enabled
            - name: FELIX_IPINIPMTU
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: veth_mtu
            # Set MTU for the VXLAN tunnel device.
            - name: FELIX_VXLANMTU
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: veth_mtu
            # Set MTU for the Wireguard tunnel device.
            - name: FELIX_WIREGUARDMTU
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: veth_mtu
            # The default IPv4 pool to create on startup if none exists. Pod IPs will be
            # chosen from this range. Changing this value after installation will have
            # no effect. This should fall within `--cluster-cidr`.
            # 修改3：添加ip地址信息
            - name: CALICO_IPV4POOL_CIDR
              value: "172.168.0.0/12"
            # Disable file logging so `kubectl logs` works.
            - name: CALICO_DISABLE_FILE_LOGGING
              value: "true"
            # Set Felix endpoint to host default action to ACCEPT.
            - name: FELIX_DEFAULTENDPOINTTOHOSTACTION
              value: "ACCEPT"
            # Disable IPv6 on Kubernetes.
            - name: FELIX_IPV6SUPPORT
              value: "false"
            # Set Felix logging to "info"
            - name: FELIX_LOGSEVERITYSCREEN
              value: "info"
            - name: FELIX_HEALTHENABLED
              value: "true"
          securityContext:
            privileged: true
          resources:
            requests:
              cpu: 250m
          livenessProbe:
            exec:
              command:
              - /bin/calico-node
              - -felix-live
              - -bird-live
            periodSeconds: 10
            initialDelaySeconds: 10
            failureThreshold: 6
          readinessProbe:
            exec:
              command:
              - /bin/calico-node
              - -felix-ready
              - -bird-ready
            periodSeconds: 10
          volumeMounts:
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
            - mountPath: /run/xtables.lock
              name: xtables-lock
              readOnly: false
            - mountPath: /var/run/calico
              name: var-run-calico
              readOnly: false
            - mountPath: /var/lib/calico
              name: var-lib-calico
              readOnly: false
            - mountPath: /calico-secrets
              name: etcd-certs
            - name: policysync
              mountPath: /var/run/nodeagent
      volumes:
        # Used by calico-node.
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: var-run-calico
          hostPath:
            path: /var/run/calico
        - name: var-lib-calico
          hostPath:
            path: /var/lib/calico
        - name: xtables-lock
          hostPath:
            path: /run/xtables.lock
            type: FileOrCreate
        # Used to install CNI.
        - name: cni-bin-dir
          hostPath:
            path: /opt/cni/bin
        - name: cni-net-dir
          hostPath:
            path: /etc/cni/net.d
        # Mount in the etcd TLS secrets with mode 400.
        # See https://kubernetes.io/docs/concepts/configuration/secret/
        - name: etcd-certs
          secret:
            secretName: calico-etcd-secrets
            defaultMode: 0400
        # Used to create per-pod Unix Domain Sockets
        - name: policysync
          hostPath:
            type: DirectoryOrCreate
            path: /var/run/nodeagent
        # Used to install Flex Volume Driver
        - name: flexvol-driver-host
          hostPath:
            type: DirectoryOrCreate
            path: /usr/libexec/kubernetes/kubelet-plugins/volume/exec/nodeagent~uds
---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-node
  namespace: kube-system

---
# Source: calico/templates/calico-kube-controllers.yaml
# See https://github.com/projectcalico/kube-controllers
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calico-kube-controllers
  namespace: kube-system
  labels:
    k8s-app: calico-kube-controllers
spec:
  # The controllers can only have a single active instance.
  replicas: 1
  selector:
    matchLabels:
      k8s-app: calico-kube-controllers
  strategy:
    type: Recreate
  template:
    metadata:
      name: calico-kube-controllers
      namespace: kube-system
      labels:
        k8s-app: calico-kube-controllers
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      tolerations:
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      serviceAccountName: calico-kube-controllers
      priorityClassName: system-cluster-critical
      # The controllers must run in the host network namespace so that
      # it isn't governed by policy that would prevent it from working.
      hostNetwork: true
      containers:
        - name: calico-kube-controllers
          # 修改4：镜像地址变更
          image: registry.cn-beijing.aliyuncs.com/dotbalo/kube-controllers:v3.15.3
          env:
            # The location of the etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # Location of the CA certificate for etcd.
            - name: ETCD_CA_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_ca
            # Location of the client key for etcd.
            - name: ETCD_KEY_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_key
            # Location of the client certificate for etcd.
            - name: ETCD_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_cert
            # Choose which controllers to run.
            - name: ENABLED_CONTROLLERS
              value: policy,namespace,serviceaccount,workloadendpoint,node
          volumeMounts:
            # Mount in the etcd TLS secrets.
            - mountPath: /calico-secrets
              name: etcd-certs
          readinessProbe:
            exec:
              command:
              - /usr/bin/check-status
              - -r
      volumes:
        # Mount in the etcd TLS secrets with mode 400.
        # See https://kubernetes.io/docs/concepts/configuration/secret/
        - name: etcd-certs
          secret:
            secretName: calico-etcd-secrets
            defaultMode: 0400

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-kube-controllers
  namespace: kube-system

---
# Source: calico/templates/calico-typha.yaml

---
# Source: calico/templates/configure-canal.yaml

---
# Source: calico/templates/kdd-crds.yaml

```

执行命令开始安装
kubectl apply -f calico-etcd.yaml

需要有一个漫长的等待才能完成安装，完成后查看集群状态以及pod装填
kubectl -n kube-system get pod
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-5f6d4b864b-pr5zl   1/1     Running   0          2m4s
calico-node-bwdmj                          1/1     Running   0          2m4s
calico-node-ddh6r                          1/1     Running   0          2m4s
calico-node-dhfb2                          1/1     Running   0          2m4s
calico-node-jx79z                          1/1     Running   0          2m4s
calico-node-x4nzm                          1/1     Running   0          2m4s
coredns-54d67798b7-66z7x                   1/1     Running   0          4h9m
coredns-54d67798b7-jzql9                   1/1     Running   0          4h9m
etcd-k8s-master-01                         1/1     Running   0          4h9m
etcd-k8s-master-02                         1/1     Running   0          3h57m
etcd-k8s-master-03                         1/1     Running   0          3h35m
kube-apiserver-k8s-master-01               1/1     Running   0          4h9m
kube-apiserver-k8s-master-02               1/1     Running   0          3h57m
kube-apiserver-k8s-master-03               1/1     Running   0          3h35m
kube-controller-manager-k8s-master-01      1/1     Running   1          4h9m
kube-controller-manager-k8s-master-02      1/1     Running   0          3h57m
kube-controller-manager-k8s-master-03      1/1     Running   0          3h35m
kube-proxy-f56vn                           1/1     Running   0          3h35m
kube-proxy-hzc6n                           1/1     Running   0          3h30m
kube-proxy-pqh28                           1/1     Running   0          4h9m
kube-proxy-r66cr                           1/1     Running   0          3h30m
kube-proxy-t7dvm                           1/1     Running   0          3h57m
kube-scheduler-k8s-master-01               1/1     Running   1          4h9m
kube-scheduler-k8s-master-02               1/1     Running   0          3h57m
kube-scheduler-k8s-master-03               1/1     Running   0          3h35m

这样整个集群算是告一段落了。

第7部分：metrics和dashboard的安装

执行metrics安装之前，需要先把下面配置文件中的证书拷贝到其它节点，否则会存在找不到证书的情况
for i in k8s-master-02 k8s-master-03 k8s-node-01 k8s-node-02; do scp /etc/kubernetes/pki/front-proxy-ca.crt $i:/etc/kubernetes/pki/front-proxy-ca.crt ;done


metrics的配置文件如下：

```yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
  name: system:aggregated-metrics-reader
rules:
  - apiGroups:
      - metrics.k8s.io
    resources:
      - pods
      - nodes
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - nodes
      - nodes/stats
      - namespaces
      - configmaps
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
  - kind: ServiceAccount
    name: metrics-server
    namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: metrics-server
    namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
  - kind: ServiceAccount
    name: metrics-server
    namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  ports:
    - name: https
      port: 443
      protocol: TCP
      targetPort: https
  selector:
    k8s-app: metrics-server
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  strategy:
    rollingUpdate:
      maxUnavailable: 0
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
        - args:
            - --cert-dir=/tmp
            - --secure-port=4443
            - --metric-resolution=30s
            - --kubelet-insecure-tls
            - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
            - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt # change to front-proxy-ca.crt for kubeadm， 如果不添加证书，可能会造成监测信息获取不到的情况
            - --requestheader-username-headers=X-Remote-User
            - --requestheader-group-headers=X-Remote-Group
            - --requestheader-extra-headers-prefix=X-Remote-Extra-
          image: registry.cn-beijing.aliyuncs.com/dotbalo/metrics-server:v0.4.1
          imagePullPolicy: IfNotPresent
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /livez
              port: https
              scheme: HTTPS
            periodSeconds: 10
          name: metrics-server
          ports:
            - containerPort: 4443
              name: https
              protocol: TCP
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /readyz
              port: https
              scheme: HTTPS
            periodSeconds: 10
          securityContext:
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
          volumeMounts:
            - mountPath: /tmp
              name: tmp-dir
            - name: ca-ssl
              mountPath: /etc/kubernetes/pki
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      volumes:
        - emptyDir: {}
          name: tmp-dir
        - name: ca-ssl
          hostPath:
            path: /etc/kubernetes/pki
---
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    k8s-app: metrics-server
  name: v1beta1.metrics.k8s.io
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: metrics-server
    namespace: kube-system
  version: v1beta1
  versionPriority: 100

```

安装命令
kubectl apply -f comp.yaml

安装完成后，查看指标信息
kubectl top node
NAME            CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master-01   161m         8%     1363Mi          35%
k8s-master-02   154m         7%     1310Mi          34%
k8s-master-03   152m         7%     1272Mi          33%
k8s-node-01     86m          4%     1090Mi          28%
k8s-node-02     92m          4%     1073Mi          28%

安装dashboard，配置文件有两个

```yaml

# dashboard.yaml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

apiVersion: v1
kind: Namespace
metadata:
  name: kubernetes-dashboard

---

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kubernetes-dashboard
type: Opaque

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-csrf
  namespace: kubernetes-dashboard
type: Opaque
data:
  csrf: ""

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-key-holder
  namespace: kubernetes-dashboard
type: Opaque

---

kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-settings
  namespace: kubernetes-dashboard

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
rules:
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
    verbs: ["get", "update", "delete"]
    # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["get", "update"]
    # Allow Dashboard to get metrics.
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster", "dashboard-metrics-scraper"]
    verbs: ["proxy"]
  - apiGroups: [""]
    resources: ["services/proxy"]
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
    verbs: ["get"]

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
rules:
  # Allow Metrics Scraper to get metrics from the Metrics server
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
        - name: kubernetes-dashboard
          image: registry.cn-beijing.aliyuncs.com/dotbalo/dashboard:v2.0.4
          imagePullPolicy: Always
          ports:
            - containerPort: 8443
              protocol: TCP
          args:
            - --auto-generate-certificates
            - --namespace=kubernetes-dashboard
            # Uncomment the following line to manually specify Kubernetes API server Host
            # If not specified, Dashboard will attempt to auto discover the API server and connect
            # to it. Uncomment only if the default does not work.
            # - --apiserver-host=http://my-address:port
          volumeMounts:
            - name: kubernetes-dashboard-certs
              mountPath: /certs
              # Create on-disk volume to store exec logs
            - mountPath: /tmp
              name: tmp-volume
          livenessProbe:
            httpGet:
              scheme: HTTPS
              path: /
              port: 8443
            initialDelaySeconds: 30
            timeoutSeconds: 30
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      volumes:
        - name: kubernetes-dashboard-certs
          secret:
            secretName: kubernetes-dashboard-certs
        - name: tmp-volume
          emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    k8s-app: dashboard-metrics-scraper

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: dashboard-metrics-scraper
  template:
    metadata:
      labels:
        k8s-app: dashboard-metrics-scraper
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'runtime/default'
    spec:
      containers:
        - name: dashboard-metrics-scraper
          image: registry.cn-beijing.aliyuncs.com/dotbalo/metrics-scraper:v1.0.4
          ports:
            - containerPort: 8000
              protocol: TCP
          livenessProbe:
            httpGet:
              scheme: HTTP
              path: /
              port: 8000
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
            - mountPath: /tmp
              name: tmp-volume
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      volumes:
        - name: tmp-volume
          emptyDir: {}

```


```yaml

# dashboard-user.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1 
kind: ClusterRoleBinding 
metadata: 
  name: admin-user
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system


```

两个配置文件放在同一个文件夹下，所以运行时不指定具体文件
kubectl apply -f .

安装完成后，为了本地可以进行访问，在google浏览器中加入启动参数，用于解决无法访问Dashboard的问题
```
--test-type --ignore-certificate-errors
```

最后需要针对dashboard修改svc设置，变更ClusterIP为nodePort
```
kubectl get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
dashboard-metrics-scraper   ClusterIP   10.110.87.54     <none>        8000/TCP   3m21s
kubernetes-dashboard        ClusterIP   10.104.150.211   <none>        443/TCP    3m21s

kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"k8s-app":"kubernetes-dashboard"},"name":"kubernetes-dashboard","namespace":"kubernetes-dashboard"},"spec":{"ports":[{"port":443,"targetPort":8443}],"selector":{"k8s-app":"kubernetes-dashboard"}}}
  creationTimestamp: "2021-01-09T15:31:54Z"
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
  resourceVersion: "34046"
  uid: ec006a73-60fd-412f-a4fe-19e4b0726a9b
spec:
  clusterIP: 10.104.150.211
  clusterIPs:
  - 10.104.150.211
  ports:
  - port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: nodePort   # 在此处更改
status:
  loadBalancer: {}


// :wq保存退出

kubectl get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.110.87.54     <none>        8000/TCP        7m33s
kubernetes-dashboard        NodePort    10.104.150.211   <none>        443:32056/TCP   7m33s


```

根据已有的端口号，通过任意安装了kube-proxy的宿主机，或者VIP的IP+端口即可访问到dashboard。例如https://192.168.229.60:32056.

选择使用token登录，首先要先获取token值

```
 kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-wgsns
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: b13eceff-722e-4c3c-bcce-1a001a0a698c

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IlN4QXVXRThDdGlNMkQ0MlByVjJueElaOUtYdXhqQ0JiVjVadkFUMmVtajQifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXdnc25zIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJiMTNlY2VmZi03MjJlLTRjM2MtYmNjZS0xYTAwMWEwYTY5OGMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.bjITkXMzLzGTJqKScN48TSVgA4cjj7dfcTOPKlEt87hZyg9tifp3WR1bbtNowdDlEcVK2anuHXm6Z9XV6OWVBWp845b-LZPcsOkLHHcYXd16YpLfAL3_SXLQpboq-_ilQU6M4IV2zWeBYXWc03aAqSkQVS8WlE8yuTm7UqRPpNs1C3YB92c3OjlXujS38gsdkvud5Wk8Mnr8lNEyYM39HiiM8SXVdDtN1kzLQgRpFMMQq75rGtohQ4cATkla0a6VHxRh9VFXdgaytutU_hAFetA6Oia7i0j56TbJBwguJNFTTm2-sSvMnWqAPIelu-LKVoC-aYMiSAODslyXVE-L-g


```

将token值复制进去即可进行访问！


第8部分：必须要改的内容

8.1 修改kube-proxy的运行模式为ipvs

查看原来的运行模式

curl 127.0.0.1:10249/proxyMode
iptables

在master01节点执行，修改为ipvs
kubectl edit cm kube-proxy -n kube-system
// 找到mode位置，修改
mode: "ipvs"

修改完成后，进行kube-proxy的滚动更新
kubectl patch daemonset kube-proxy -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}" -n kube-system

更新完成后再查看运行方式
curl 127.0.0.1:10249/proxyMode
ipvs

查看ipvs规则信息
ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  127.0.0.1:32056 rr
  -> 172.163.98.133:8443          Masq    1      0          0
TCP  172.17.0.1:32056 rr
  -> 172.163.98.133:8443          Masq    1      0          0
TCP  172.174.107.64:32056 rr
  -> 172.163.98.133:8443          Masq    1      0          0
TCP  192.168.229.51:32056 rr
  -> 172.163.98.133:8443          Masq    1      0          0
TCP  10.96.0.1:443 rr
  -> 192.168.229.51:6443          Masq    1      0          0
  -> 192.168.229.52:6443          Masq    1      0          0
  -> 192.168.229.53:6443          Masq    1      0          0
TCP  10.96.0.10:53 rr
  -> 172.163.98.129:53            Masq    1      0          0
  -> 172.175.44.1:53              Masq    1      0          0
TCP  10.96.0.10:9153 rr
  -> 172.163.98.129:9153          Masq    1      0          0
  -> 172.175.44.1:9153            Masq    1      0          0
TCP  10.104.150.211:443 rr
  -> 172.163.98.133:8443          Masq    1      0          0
TCP  10.110.87.54:8000 rr
  -> 172.175.44.2:8000            Masq    1      0          0
TCP  10.111.142.190:443 rr
  -> 172.163.98.132:4443          Masq    1      1          0
UDP  10.96.0.10:53 rr
  -> 172.163.98.129:53            Masq    1      0          0
  -> 172.175.44.1:53              Masq    1      0          0


8.2 master节点可以部署非系统pod，通过对污点（taints）进行修改

查看节点的污点信息
kubectl describe node -l node-role.kubernetes.io/master= | grep Taints
Taints:             node-role.kubemetes.io/naster:NoSchedule
Taints:             node-role.kubernetes.io/master:NoSchedule
Taints:             node-role.kubernetes.io/master:NoSchedule

去除污点
kubectl taint node -l node-role.kubernetes.io/master node-role.kubernetes.io/master:NoSchedule-

再次查看
kubectl describe node -l node-role.kubernetes.io/master= | grep Taints
Taints:             <none>
Taints:             <none>
Taints:             <none>

注意污点去除时，要和查找到的污点信息一一对应即可。

最后一部分：集群验证





注意：

1. 所有的镜像，尽量在本地镜像仓库内备份留存，这样集群一旦出现问题，在自愈的过程中拉取镜像时更快
2. 在执行kubectl edit svc 命令时，可以按住shift+zz(按住shift连击两下z键)进行保存退出

1.2 二进制安装

## 第二章 基本概念介绍

2.1 为什么要使用k8s

2.1.1 健康检查，原生提供健康检查的功能

2.1.2 服务的动态扩容缩容

2.1.3 大规模容器管理（1000个容器），端口资源的简化管理（内部Service，不直接对外暴露的端口），减少端口冲突

![](架构图展示.png)

2.2 master节点

整个集群的控制中枢

* kube-apiserver: 集群控制核心。各个模块之间的信息交互，集群管理、资源配置、安全机制的入口
* controller-manager：集群状态管理器，保证pod和其它资源达到设定值，同apiserver通信，在需要的时候进行资源的增删改查
* Scheduler: 集群的调度中心，会根据指定的条件，选择一个或一批符合条件的节点

* etcd：键值数据库，保存集群信息，一般要部署奇数个节点（至少三个以上，避免脑裂的情况）。做了存储层面的内容，尽可能将etcd独立出来，不要和master节点部署到一台服务器上。

最重要的，etcd集群一定要部署在SSD硬盘上，强制性要求，不能缩水。否则会导致整个集群运行缓慢。

注意：尽量将master节点设置污点，不允许master节点部署应用程序！

2.3 node节点（工作节点）

* kubelet： 负责监听节点上的pod状态，同时负责上报节点和节点上的Pod信息，负责与master节点进行通信，管理节点上的pod
* kube-proxy：负责Pod之间的通信和负载均衡，将指定的流量分发到后端正确的机器上。

查看kube-proxy的运行模式
curl 127.0.0.1:10249/proxyMode

* ipvs：监听master节点增加和删除Service和Endpoint的消息，调用Netlink接口创建相应的ipvs规则。例如，通过该规则关联物理端口与集群内Pod的访问地址信息。通过ipvs规则，将流量转发到相应的Pod上。
* iptables：同ipvs功能相同，不同点在于，对于每一个Service都要创建一条iptables规则，将service的ClusterIP代理到后端对应的pod上。在iptables规则增多后，其性能会急剧下降。ipvs为内核级转发。

2.4 常用Pod

* Calico：符合CNI标准的网络插件，给每个Pod生成一个唯一的IP地址，并且把每个节点当作一个路由器。支持网络流量策略。对比Cilium eBPF

* CoreDNS：用于kubernetes集群内部Service的解析，可以让Pod把Service名称解析成IP地址，然后通过Service的IP地址连接到对应的应用上。

一般认为Service网段中的第一个地址是留给kubernetes使用的，而第10个地址是留给CoreDNS使用的。

查：如何控制CoreDNS的个数？工具是哪个？


* Docker：容器引擎

2.5 Pod

Pod具有隔离性，通过namespace进行隔离。

无隔离性的资源：PV、RBAC（clusterrole、clusterrolebinding）、storageclass、ingressclass

2.5.1 Pod概念

k8s中最小的单元，由一个或者多个容器组成，每个pod还包含一个Pause容器，Pause容器是pod的父容器，主要负责僵尸进程的回收管理， 通过Pause容器可以使同一个Pod中的多个容器共享存储（volume）、网络、PID、IPC等资源。

注意：对于docker容器，一个Container中最多启动一个进程，这样在退出、删除、杀死进程操作时，不会对其它的进程造成影响，事实上的进程隔离。

为何使用Pod这个概念？

    1. 一个Pod中多个容器，一个系统由多个微服务进行支撑，微服务中服务的强依赖性，通信延迟低或者数据依赖性
    2. 兼容符合CRI标准的不同的容器技术，例如containerd、CRI-O等

2.5.2 Pod编写解析

```yaml
 
apiVersion: v1 # 必选, API的版本号
kind: Pod # 必选,类型Pod
metadata: # 必选,元数据
  name: nginx  # 必选,符合RFC 1035规范的Pod名称
  # namespace: default # 可选，Pod所在的命名空间，不指定默认为default, 可以使用 -n 指定namespace 注意：一般手动指定目标namespace，使用-n参数
  labels: # 可选,标签选择器,一般用于过滤和区分Pod
    app: nginx
    role: frontend # 可以写多个
  annotations : # 可选,注释列表,可以写多个
    app: nginx
spec: # 必选,用于定义容器的详細信息
  initContainers: # 初始化容器,在容器启动之前执行的一些初始化操作，可以有多个，类似list或者数组，使用"-"进行区分
  - command:
    - sh
    - -C
    - echo "I am InitContainer for init some configuration"
    image: busybox
    imagePullPolicy: IfNotPresent
    name: init-container
  terminationGracePeriodSeconds: 30 # 设置Pod终止之前的等待时间，等待资源处理的时间
  containers: # 必选,容器列表
  - name: nginx # 必选,符合RFC 1035规范的容器名称
    image: nginx:1.16.1 # 必选,容器所用的镜像的地址，注意需要添加为自有的容器管理地址，例如10.0.98.90:5000/test/nginx:1.16.1
    imagePullPolicy: Always # 可选,镜像拉取策略,ifNotPresent，如果宿主机有该镜像就不再拉取；Always：总是拉取；Never：不管是否存在都不拉取
    command:  # 可选,容器启动执行的命令，该处command相当于docker容器内的 ENTRYPOINT ，arg相当于docker容器中的 CMD命令
    - nginx
    - -g
    - "daemon off;"
    workingDir: /usr/share/nginx/html # 可选,容器的工作目录，在登录容器后会定位到该目录下
    volumeMounts: # 可选,存储卷配置,可以配置多个
    - name: webroot # 存储卷名称
      mountPath: /usr/share/nginx/html # 挂载目录
      readOnly: true  # 只读
    ports:  # 可选,容器需要暴露的端口号列表
    - name: http  # 端口名称
      containerPort: 80 # 端口号
      protocol: TCP # 端口协议,默认TCP
    env:  # 可选，环境变量配置列表
    - name: TZ # 变量名
      value: Asia/Shanghai # 变量的值
    - name: LANG
      value: en US.utf8
    resources:  # 可选，资源限制和资源请求限制
      limits:   # 资源的最大限制
        cpu: 1000m
        memory: 1024Mi
      requests:  # 启动所需的资源
        cpu: 100m
        memory: 512Mi
# 三种健康检查方式：startupProbe、readinessProbe、livenessProbe
#    startupProbe: # 可选,检测容器内进程是否完成启动。注意三种检查方式同时只能使用一种:
#      httpGet: # httpGet检测方式,生产环境建议使用httpGet实现接口级健崇栓套,健。检查由应用程序提供。
#        path: /api/successStart # 检查路径
#        port: 80
    readinessProbe: # 可选,健康检查。注可三种检查方式同时只能使用一者,
      httpGet:      # httpGet检测方式,生产环境建议使用HttpGet实现接口级健康检查,健康枪查应由应用程序提供。
            path: / # 检查路径
            port: 80 #
    livenessProbe: # 可选,健康检套
      #exec:       #执行容器命令检测方式
           #command:
           #- cat
           #- /health
    #httpGet: #httpGet检测方式
    #   path: /_health # 检查路径
    #   port: 8088
    #   httpHeaders:  # 检查的请求头
    #     name: end-user   # 请求头的key值
    #     value: Jason    # 请求头的value值
      tcpSocket:  # 端口检测方式
            port: 80
      initialDelaySeconds : 60 # 初始化时间
      timeoutSeconds: 2  # 超时时间
      periodSeconds : 5  # 检测间隔
      successThreshold : 1 # 检查成功为2次表示就绪
      failureThreshold : 2 # 检测失败1次表示未就绪
    lifecycle:  # 生命周期相关配置
      postStart: # 容器创建完成后执行的指令,可以是exec httpGet TCPSocket
        exec:
          command:
          - sh
          - -C
          - 'mkdir /data/'
      preStop:
        httpGet:
              path: /
              port: 80
      #  exec:
      #  command:
      #  - sh
      #  - -c
      #  - sleep 9
  restartPolicy: Always # 重启策略，可选,默认为Always：容器故障或者没有启动成功，自动重启  Onfailure：容器以不为0的状态终止，也就是异常终止的情况下，自动重启该容器，Never：总是不会重启
  #nodeSelector: # 可选,指定Node节点
  #      region: subnet7   # 匹配带有该标签的节点，将pod部署到该节点
  imagePullSecrets: # 可选,拉取镜像使用的secret,可以配置多个，对应于私有镜像仓库的账号密码
  - name: default-dockercfg-86258
  hostNetwork: false # 可选,是否为主机模式,如是,会占用主机端口
  volumes : # 共享存储卷列表
  - name: webroot # 名称，与上述对应
    emptyDir: {}
        #hostPath: # 挂载目录
        #  path: /etc/hosts # 挂载本机目录

```

测试使用脚本信息

```yaml

# pod课程所用到的配置文件

apiVersion: v1 # 必选, API的版本号
kind: Pod # 必选,类型Pod
metadata: # 必选,元数据
  name: nginx  # 必选,符合RFC 1035规范的Pod名称
  labels: # 可选,标签选择器,一般用于过滤和区分Pod
    app: nginx
    role: frontend # 可以写多个
  annotations : # 可选,注释列表,可以写多个
    app: nginx
spec: # 必选,用于定义容器的详細信息
  containers: # 必选,容器列表
  - name: nginx # 必选,符合RFC 1035规范的容器名称
    image: nginx:1.16.1 # 必选,容器所用的镜像的地址，注意需要添加为自有的容器管理地址，例如10.0.98.90:5000/test/nginx:1.16.1
    imagePullPolicy: Always # 可选,镜像拉取策略,ifNotPresent，如果宿主机有该镜像就不再拉取；Always：总是拉取；Never：不管是否存在都不拉取
    command:  # 可选,容器启动执行的命令，该处command相当于docker容器内的 ENTRYPOINT ，arg相当于docker容器中的 CMD命令
    - nginx
    - -g
    - "daemon off;"
    ports:  # 可选,容器需要暴露的端口号列表
    - name: http  # 端口名称
      containerPort: 80 # 端口号
      protocol: TCP # 端口协议,默认TCP
    env:  # 可选，环境变量配置列表
    - name: TZ # 变量名
      value: Asia/Shanghai # 变量的值
    - name: LANG
      value: en US.utf8
  restartPolicy: Always # 重启策略，可选,默认为Always：容器故障或者没有启动成功，自动重启  Onfailure：容器以不为0的状态终止，也就是异常终止的情况下，自动重启该容器，Never：总是不会重启


```

常用命令：

```shell

kubectl create ns kube-public   # 创建命名空间kube-public

kubectl get po --show-labels  # 查看pod上所携带的labels

kubectl create -f pod.yaml -n kube-public  # 在kube-public命名空间上创建pod，-f指定创建资源的文件来源

kubectl apply -f pod.yaml -n kube-public  # 运行已经创建的pod，如果pod.yaml文件未修改，运行该命令不对之前的镜像造成影响

kubectl describe pod nginx    # 查看名称为nginx的pod当前状态以及配置信息

kubectl delete pod nginx   # 删除名称为nginx的pod

```

遇到问题：

```log
# kubectl describe pod nginx

.......

  Warning  Failed     7m47s (x4 over 10m)    kubelet            Error: ErrImagePull
  Warning  Failed     7m47s (x2 over 9m36s)  kubelet            Failed to pull image "nginx:1.16.1": rpc error: code = Unknown desc = Error response from daemon: Get https://registry-1.docker.io/v2/library/nginx/manifests/1.16.1: net/http: TLS handshake timeout
  Normal   BackOff    7m35s (x6 over 10m)    kubelet            Back-off pulling image "nginx:1.16.1"
  Warning  Failed     7m24s (x7 over 10m)    kubelet            Error: ImagePullBackOff

# docker pull nginx:1.16.1
Error response from daemon: Get https://registry-1.docker.io/v2/: net/http: TLS handshake timeout

```

解决方式：重新配置docker服务的daemon.json文件，配置完成后重启docker服务，等待集群自动恢复。

```json

{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "registry-mirrors": [
        "https://registry.cn-hangzhou.aliyuncs.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://registry.docker-cn.com",
        "http://hub-mirror.c.163.com",
        "https://mirror.baidubce.com"
    ],
    "insecure-registries": ["harbor.xxx.cn:30002"], // 这个配置项可以不用添加
    "live-restore": true,
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "50m",
        "max-file": "3"
    }
}

```

2.5.3 Pod探针

探针分类：

* startupProbe：k8s 1.16版本后，用于判断容器内应用程序是否启用。如果配置了该项，就会禁止其它探针的运行，优先判断startupProbe的状态，只要探测成功后就不再进行探测。如果程序启动比较慢，建议使用。
* livenessProbe：用于探测容器是否运行，如果探测失败，kubelet会根据配置的重启策略进行处理。如果没有配置该探针，默认就是success。决定容器是否重启
* readinessProbe：用于探测容器内的程序是否健康，其返回值如果为success，那么就代表这个容器已经完全启动，并且程序已经是可以接收流量的状态。

检测方式分类（每个探针都包含）：

* ExecAction：在容器内执行命令，如果返回值为0，则容器健康。
* TCPSocketAction：通过tcp连接检查容器内端口是否连通，如果是通的，就认为容器健康。（可能存在端口已经启动，但pod中的容器无法连接的情况）
* HTTPGetAction: 通过应用程序暴露的api地址来检测程序是否是正常的，如果返回的状态码为大于等于200到小于400之间，则认为容器健康。（最常用，生产）

在公司日常开发中一定要求微服务暴露readiness和liveness相对应的接口信息。

常用命令：

```
kubectl get deployment -n kube-system

kubectl edit deploy coredns -n kube-system  # 在线编辑名称为coredns的deployment配置信息

watch kubectl get pod   # watch 循环查看某条命令的运行状态

kubectl replace -f pod.yaml  # 根据pod.yaml中配置替换已有的资源
```

参考问题：

[为什么引入startupProbe](https://my.oschina.net/zhangshoufu/blog/4871452)


（此观点错误） ~~startupProbe配置时，针对Spring Cloud微服务的场景，可以考虑使用ExecAction的方式，判断java进程是否存在，如果java进程已经存在了，可以认为pod正常运行了。~~ 存在进程存在而readiness不工作，造成pod不能重启


2.5.3.1 探针检查参数配置

* initialDelaySeconds : 60 # 初始化时间，在pod启动后等待60秒后进行检查，不建议设置太长，否则滚动发布周期会拉长，等待时间过长
* timeoutSeconds: 2  # 超时时间，执行命令多长时间内能返回信息，一般设置1到2秒
* periodSeconds : 5  # 检测间隔
* successThreshold : 1 # 检查成功为1次表示就绪，例如接口成功返回信息一次代表成功
* failureThreshold : 2 # 检测失败2次表示未就绪，不要设置为1次，可能存在网络波动的情况导致检测不成功

pod有时不能直接使用apply和replace进行替换，可能会出现替换不成功的情况，但其它高级资源可以，例如deployment。所以针对pod的操作要先删除，再创建新的。

readiness和liveness在配置时都使用接口级别的健康检查，这两个接口由服务提供，尽量不使用linux命令进行，例如在command下执行pgrep java命令查看java进程是否存在来判断liveness状态。
如果遇到上述情况，会出现以下现象，readiness接口返回信息已经表示服务不可用，而liveness的判断中，java进程始终存在，造成整个pod处于0/1的状态，无法接受流量，也无法进行重启来恢复。

2.5.4 lifeCycle配置--postStart和preStop

同probe的区别时，lifeCycle中的配置没有健康检查的各个探针参数配置。

postStart使用风险：postStart并不能保证在initContainers中的command之后执行，，二者可能同时执行。如果面临一些高危权限的操作，建议将这些操作放在initContainers中的command下面设置并执行。

postStart常用于创建文件或者创建文件夹这样的操作。

2.5.4.1 preStop

terminationGracePeriodSeconds: 30  k8s预留的终止时间，默认30秒，可以自行改动，允许在该时间内处理容器终止后的清理或者收尾工作。
设置时，需要放在containers之上。但是和preStop中在

Pod终止流程

用户执行删除操作----------1. 变更pod状态为Terminating
                   | 
                   ------2. （存在Service的情况下）Endpoint删除该Pod内的IP地址
                   |
                   ------3. 执行preStop的指令

以Spring Cloud中的微服务为例，Eureka组件进行描述。

pod删除前，预留的时间由terminationGracePeriodSeconds配置信息决定，而如果在preStop中配置命令，例如：

```yaml
      preStop:
        exec:
          command:
            - sh
            - -c
            - sleep 90
```

则pod删除前，不会真正sleep 90秒，通过time命令查看执行时间，接近于terminationGracePeriodSeconds设定的时间。

如果设置sleep低于terminationGracePeriodSeconds配置的时间，而任务恰好能在sleep设置的时间内可以完成，则sleep生效，否则还是以terminationGracePeriodSeconds为准。

常用命令：

```

kubectl get event  # 查看默认namespace的事件

time kubectl delete pod nginx  # 查看该条命令的执行时间


```

注意：终止进程尽量使用kill <pid>或者kill -15 <pid>，禁止使用kill -9。最佳实践为kill `pgrep java`，前提是镜像支持pgrep命令。

2.6 RC && RS

- Replication Controller : 可以保证Pod副本数达到期望值，可以确保一个Pod或者一组同类Pod总是可用。已经被废弃
- ReplicaSet ：复制集。至此基于集合的标签选择器的下一代RC，主要用作Deployment协调创建、删除和更新Pod。和RC唯一区别是ReplicaSet支持标签选择器

RC和RS不支持回滚，必须使用更高级资源（Deployment或者DaemonSet）进行管理，再通过RS去管理下面的Pod。

极少单独使用，主要是配合Deployment、StatefulSet以及DaemonSet去管理。一般建议使用Deployment来自动管理ReplicaSet。

2.7 Deployment

* Deployment：无状态服务
* StatefulSet：有状态应用，例如Redis主从、RabbitMQ
* DaemonSet：在每个节点上都会启动一个容器

2.7.1 Deployment概念

用于部署无状态服务，最常用的控制器。一般用于管理维护企业内部无状态的微服务。

可以管理多个副本的Pod，实现无缝迁移、自动扩容缩容、自动灾难恢复、一键回滚等功能。

常用命令：

```

kubectl create deployment nginx --image=nginx:1.16.1  # 手动创建deployment

kubectl create deployment nginx --image=nginx:1.16.1 -o yaml > nginx-deployment.yaml   # 将创建的deployment文件导出

kubectl replace -f nginx-deployment.yaml  # 通过配置文件替代当前namespace中的deployment配置

kubectl edit deployment nginx        # 在线编辑名称为nginx的deployment


```

配置文件：

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: "2021-01-14T13:26:49Z"
  generation: 1
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 3            # 副本数
  revisionHistoryLimit: 10   # 历史记录保存个数
  selector:
    matchLabels:
      app: nginx
  strategy:               # 滚动升级更新策略
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:                # 从该节点往下配置的信息同pod中的配置项
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.16.1
        imagePullPolicy: IfNotPresent
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30

```

注意：尽量不要在deployment运行状态下直接修改配置中的labels信息，例如selector下matchLabels、template下的labels。无论是增加还是删除，如果修改后，可能会造成ReplicaSet资源无法删除的情况。修改labels参数，会造成启动新的ReplicaSet，而新的RS具有新的label，就无法管理之前的label了。

注意：可以在~/.kube/config中找到连接k8s的配置文件，配合kubectl命令在任意机器都可以连接k8s集群。

2.7.2 deployment状态解析

```shell

# # kubectl get pod -o wide
# NAME                     READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
# nginx-7f785d94f5-74fr8   1/1     Running   0          19m   172.175.44.13    k8s-node-01   <none>           <none>
# nginx-7f785d94f5-tmqfd   1/1     Running   0          18m   172.175.44.14    k8s-node-01   <none>           <none>
# nginx-7f785d94f5-vcsnk   1/1     Running   0          25m   172.163.98.145   k8s-node-02   <none>           <none>

$ kubectl get deployment -o wide
NAME    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
nginx   3/3     3            3           29m   nginx        nginx:1.16.1   app=nginx

```

* NAME  --- deployment名称，同一个namespace下不能使用相同的名称
* READY --- pod的状态，上面命令中存在3个副本，3个已经启动
* UP-TO-DATE  ----  已经达到期望状态的被更新的副本数
* AVAILABLE -----  已经可以用的副本数
* AGE    -------   显示应用程序运行的时间
* CONTAINERS  -----  容器名称
* IMAGES   ------ 容器的镜像
* SELECTOR  ------ 管理Pod的标签

常用命令

```
kubectl get pod --show-labels # 查看pod的Labels

kubectl get deploy -o yaml | grep image # 以yaml的方式输出配置信息，并且筛选查看image内容

kubectl set image deploy nginx nginx=nginx:1.15.2 --record  # --record 记录当前更改的参数，根据record下的信息可以查看各个版本的参数信息

kubectl rollout status deploy nginx   # 查看滚动更新的进度

```

注意：deployment通过ReplicaSet来管理Pod。

如何触发deployment的更新，生成新的ReplicaSet？只有在修改配置文件下的spec节点下的templete配置信息后。

2.7.3 滚动更新示意

修改nginx的docker image版本后，重新生成新的ReplicaSet，然后更新Pod信息。

修改命令如下：

```
kubectl set image deploy nginx nginx=nginx:1.15.2 --record

```

通过describe命令查看如下：

```
$ kubectl describe deploy nginx
......
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  57m    deployment-controller  Scaled up replica set nginx-7f785d94f5 to 2
  Normal  ScalingReplicaSet  56m    deployment-controller  Scaled up replica set nginx-7f785d94f5 to 3
  Normal  ScalingReplicaSet  6m42s  deployment-controller  Scaled up replica set nginx-66bbc9fdc5 to 1
  Normal  ScalingReplicaSet  6m41s  deployment-controller  Scaled down replica set nginx-7f785d94f5 to 2
  Normal  ScalingReplicaSet  6m41s  deployment-controller  Scaled up replica set nginx-66bbc9fdc5 to 2
  Normal  ScalingReplicaSet  6m40s  deployment-controller  Scaled down replica set nginx-7f785d94f5 to 1
  Normal  ScalingReplicaSet  6m40s  deployment-controller  Scaled up replica set nginx-66bbc9fdc5 to 3
  Normal  ScalingReplicaSet  6m38s  deployment-controller  Scaled down replica set nginx-7f785d94f5 to 0

```

注意：在更新手段上，尽可能使用kubectl edit命令或者修改yaml格式的配置文件使用kubectl replace命令更新deployment。

微服务版本发布时，可以直接使用set命令进行更新。

2.7.4 回滚示意

查看滚动更新的历史记录

```
$ kubectl rollout history deployment nginx

deployment.apps/nginx
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deploy nginx nginx=nginx:1.15.2 --record=true
3         kubectl set image deploy nginx nginx=nginx:1.17.7 --record=true


```

回滚到上一个版本

```
$ kubectl rollout undo deployment nginx

```

查看指定版本的详细信息

```
$ kubectl rollout history deployment nginx --revision=3
deployment.apps/nginx with revision #3
Pod Template:
  Labels:       app=nginx
        pod-template-hash=6d96fd594b
  Annotations:  kubernetes.io/change-cause: kubectl set image deploy nginx nginx=nginx:1.17.7 --record=true
  Containers:
   nginx:
    Image:      nginx:1.17.7
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>

```

回滚到指定版本

```
kubectl rollout history deployment nginx --to-revision=3


```


2.7.5 扩容和缩容

可以对Deployment、ReplicaSet进行扩容

```
kubectl scale --replicas=3 deployment nginx

```

注意：扩容和缩容命令不会产生新的ReplicaSet（未修改spec.templete下的内容），但仅限于预期内的扩容操作。有关指标信息引起的自动扩容和缩容操作，使用HPA实现。

2.7.6 更新暂停和恢复

更新多出内容只触发一次更新。

第一种方式，直接通过kubectl edit命令修改deployment的配置信息

```shell

kubectl edit deploy nginx

```

第二种方式，更新暂停和恢复

```shell

// 更新暂停命令
kubectl rollout pause deploy nginx

// 多次进行set
// 修改镜像版本
kubectl set image deploy nginx nginx=nginx:1.15.5 --record

// 修改镜像pod数量
kubectl scale --replicas=2 deployment nginx

// 修改cpu配置
kubectl set resources deploy nginx --limits=cpu=200m,memory=120Mi  --requests=cpu=10m,memory=16Mi

// 查看配置信息
kubectl get deploy -oyaml

// 更新恢复命令
kubectl rollout resume deploy nginx

```

2.7.7 滚动更新策略

```yaml
spec:
  ......
  strategy:
      rollingUpdate:
        maxSurge: 25%       # 可以超过期望值的最大Pod数，可选，默认25%，可设置数字或者百分比，若该值为0，则maxUnavailable不能为0
        maxUnavailable: 25%   # 指定在回滚或者更新时最大不可用的Pod数量，可选，默认25%，可设置数字或者百分比，若该值为0，则maxSurge不能为0
      type: RollingUpdate   # 更新deployment的方式，默认RollingUpdate（滚动更新），其它还有Recreate（先删除旧的Pod，再创建新的Pod）

```

其它：
.spec.revisionHistoryLimit: 设置保留ReplicaSet历史记录，即revision的数量，设置为0，则不保留历史记录。
.spec.minReadySeconds: 可选参数，指定新创建的Pod在没有任何容器崩溃的情况下，视为其状态已经Ready（准备完成）的最小的秒数，默认为0，即一旦被创建就视为Pod可用。

2.8 StatefulSet

StatefulSet (有状态集，缩写为sts）常用于部署有状态的且需要有序启动的应用程序，比如在进行SpringCloud项目容器化时，Eureka的部署是比较适合用StatefblSet部署方式的，可以给每个Eureka实例创建一个唯一且固定的标识符，并且每个Eureka实例无需配置多余的Service，其余Spring Boot应用可以直接通过Eureka的Headless Service即可进行注册。

2.8.1 基本概念

StatefblSet 主要用于管理有状态应用程序的工作负戟API对象。比如在生产环境中.可以部署ElasticSearch集群、MongoDB集群或者需要持久化的RabbitMQ集群、Redis集群、Kafka集群和zookeeper集群等。和Deployment类似，一个StatefulSet也同样管理着基于相同容器规范的Pod。不同的是，StatefulSet为每个Pod维护了一个粘性标识。这些Pod是根据相同的规范创建的，但是不可互换.每个 Pod 都有一个持久的标识符.在重新调度时也会保留，一般格式为 StatefulSetName-Number。比如定义一个名字是Redis-Sentinel的StatefulSet，指定创建三个Pod，那么创建出来的
Pod 名字就为 Redis-Sentinel-0、Redis-Sentinel-1、Redis-Sentinel-2。而 StatefiilSet 创建的 Pod 一般使用Headless Service (无头服务）进行通信。和普通的Service的区别在于 Headless Service没有ClusterIP，它使用的是 Endpoint进行互相通信。

Headless—般的格式为：

```
statefulSetName-{0...N-1}.serviceName.namespace.svc.cluster.local

```

说明：

* serviceName为Headless Service的名字，创建StatefulSet时，必须指定Headless Service的名称;
* 0...N-1为Pod所在的序号，从0开始到N-1;
* statefulSetName为StatefulSet的名字:
* namespace为服务所在的命名空间;
* .cluster.local为Cluster Domain(集群域)。

假如公司某个项目需要在Kubernetes中部署一个主从模式的Redis，此时使用StatefulSet部署就极为合适，因为StatefulSet启动时，只有当前一个容器完全启动时，后一个容器才会被调度，并且每个容器的标识符是固定的，那么就可以通过标识符来断定当前Pod的角色。

比如用一个名为redis-ms的StatefulSet部署主从架构的Redis，该StatefulSet部署在名为public-service的命名空间中。第一个容器启动时，它的标识符为redis-ms-0，并且Pod内主机名也为redis-ms-0，此时就可以根据主机名来判断，当主机名为redis-ms-0的容器作为Redis的主节点，其余容器作为从节点，那么Slave连接Master主机配置就可以使用不会更改的Master的Headless Service，此时Redis从节点(Slave)配置的链接信息为:

```
redis-ms-0.redis-ms.public-service.svc.cluster.local
```

2.8.2 注意事项

一般StatefulSet用于有以下一个或者多个需求的应用程序:

- 需要稳定的独一无二的网络标识符。
- 需要持久化数据。
- 需要有序的、优雅的部署和扩展。
- 需要有序的自动滚动更新。

如果应用程序不需要任何稳定的标识符或者有序的部署、删除或者扩展，应该使用无状态的控制器部署应用程序，比如Deployment或者ReplicaSet。

StatefulSet是Kubernetes 1.9版本之前的beta资源，在1.5版本之前的任何Kubernetes版本都没有。

Pod所用的存储必须由PersistentVolume Provisioner(持久化卷配置器)根据请求配置StorageClass，或者由管理员预先配置，当然也可以不配置存储。

为了确保数据安全，删除和缩放StatefulSet不会删除与StatefulSet关联的卷，可以手动选择性地删除PVC和PV(关于PV和PVC请参考2.2.12节)

StatefulSet目前使用Headless Service(无头服务)负责Pod的网络身份和通信，需要提前创建此服务。

删除一个StatefulSet时，不保证对Pod的终止，要在StatefulSet中实现Pod的有序和正常终止，可以在删除之前将StatefulSet的副本缩减为0。

2.8.3 创建一个StatefulSet

```yaml

apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None  # None--该Service不会存在ClusterIp，但是依旧可以通过Headless Service访问到StatefulSet
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"   # 注意：该serviceName必须指定一个已经存在的Service
  replicas: 2 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: nginx:1.16.1
        ports:
        - containerPort: 80
          name: web
  
```

创建StatefulSet：

```
kubectl create -f nginx-statefulset.yaml
```

扩容StatefulSet：
```
kubectl scale --replicas=3 sts web

```

启动busybox访问我们前面创建的StatefulSet：

```yaml
apiVersion: apps/v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: app
    image: busybox:1.28
    command:
      - sleep
      - "3600"
    imagePullPolicy: ifNotPresent
  restartPoicy: Always

```

命令如下：

```

kubectl apply -f busybox.yaml

// 进入容器内执行命令的操作
kubectl exec -it busybox -- sh
/ # nslookup web-0.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 172.163.98.154 web-0.nginx.default.svc.cluster.local

```

这里直接将Service解析到Pod的ip地址了，由于ClusterIP配置的为None，所以不需要转换即可解析。

2.8.4 StatefulSet扩容缩容

如果启动过程中，名称为web-0、web-1的Pod已经启动完毕，准备启动web-2的Pod时，web-0 Pod挂掉了，这时候web-2不会启动，而是等待web-0重新恢复后再进行web-2的启动。也就是说，在创建下一个pod之前，前面创建的pod存在任何的故障，下一个pod都不会被创建，而是等待所有的pod变为ready之后再启动下一个pod。

在Pod删除时，会先从web-2 pod开始，逐步删除web-1、web-0 这两个Pod。

查看所有的pod动态变化的状态：

```
kubectl get pod -l app=nginx -w

// 或者是使用

watch kubectl get pod -l app=nginx

```

通过改变副本数，来查看整个变化趋势！

```
// 扩容操作
kubectl scale --replicas=5 sts web

// 缩容操作
kubectl scale --replicas=3 sts web

```

2.8.5 StatefulSet更新策略

deployment为随机无序更新

StatefulSet的更新方式--RollingUpdate、OnDelete

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  creationTimestamp: "2021-01-17T03:45:43Z"
  generation: 5
spec:
  podManagementPolicy: OrderedReady
  replicas: 5
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  serviceName: nginx
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.16.1
        imagePullPolicy: IfNotPresent
        name: nginx
        ports:
        - containerPort: 80
          name: web
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 10
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
status:
  collisionCount: 0
  currentReplicas: 5
  currentRevision: web-78c4f54fd4
  observedGeneration: 5
  readyReplicas: 5
  replicas: 5
  updateRevision: web-78c4f54fd4
  updatedReplicas: 5

```

重点在于，更新时，sts会先更新web-2 的Pod，待web-2更新完成后，再去更新web-1。如果这时候web-0被删除或者因意外不可用，则更新不会继续，会等待web-0恢复后再进行更新。也就是说，在RollingUpdate模式下，StatefulSet是正序去创建，倒序去更新！

查看更新状态：

```
// 不太准确的方式
kubectl rollout status sts web

// 精准模式
watch kubectl get pod -l app=nginx

```

使用edit方式修改sts的镜像信息：

```
kubectl edit sts web

```

OnDelete方式更新如下：

```yaml
  ......
  updateStrategy:
    type: OnDelete

  ......
```

更改为OnDelete模式后，只有把Pod进行删除操作，它才会去更新该Pod。

```
kubectl delete pod web-2

```

此时查看StatefulSet下的不同的Pod，查看修改的镜像版本信息:

```
kubectl get pod web-2 -oyaml | grep image

kubectl get pod web-0 -oyaml | grep image

```

RollingUpdate模式下设置partition信息，代表更新大于partition个数的Pod。

分段更新：partion设置为2，保留尾缀小于2的pod，也就是不更新web-0、web-1，更新web-2、web-3、web-4。

---------------|
web-0   web-1  | web-2   web-3   web-4
----不更新-----| ---------更新---------|

这样就实现了简单的灰度发布。

如果partition设置为0，则表示保留尾缀小于0的pod，由于尾缀小于0的pod不存在，故进行全部更新。

使用edit方式修改sts的镜像信息：

```
kubectl edit sts web

// 修改rollingUpdate并且修改partition为0

```

2.8.6 StatefulSet级联删除和非级联删除

级联删除，删除StatefulSet时同时删除Pod。
非级联删除，删除StatefulSet时不删除Pod，日常用的少，仅仅作为了解。

默认使用级联删除

测试级联删除
```
kubectl delete sts web
```

测试非级联删除
```
kubectl delete sts web --cascade=false

// 新版本下使用
kubectl delete sts web --cascade=orphan
```

非级联删除下，Pod变成了孤儿Pod，此时删除Pod不会被重建。

2.9 守护进程服务DaemonSet

2.9.1 DaemonSet的概念和创建

DaemonSet：守护进程集，缩写为ds，在所有节点或者匹配的节点上都部署一个容器。

calico使用DaemonSet进行部署。

应用场景：

* 运行集群存储的daemon，比如ceph、glusterd
* 节点的CNI网络插件，calico
* 节点日志的收集，fluentd或者filebeat
* 节点的监控，node-exporter
* 服务暴露：部署一个ingress nginx

创建DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.16.1
        imagePullPolicy: IfNotPresent
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30

```

创建命令：

```
kubectl create -f nginx-daemonset.yaml

```

对node打标签，将DaemonSet部署到特定节点上。（注意）

```
// 对node打标签
kubectl label node k8s-node-01 k8s-node-02 ds=true

```

修改配置文件

```yaml

template:
  .....
  spec:
    nodeSelector:
      ds: "true" # 这个true要写为字符串的形式
    containers:
......

```

重新配置生效：

```
kubectl replace -f nginx-daemonset.yaml

```

配置完成后，DaemonSet会先进行停止多余的Pod（不含ds=true标签的node节点上的Pod将会被停止），然后再对已保留的Pod进行一次滚动更新，产生一次版本记录。

2.9.2 DaemonSet的更新和回滚

DaemonSet默认的更新配置如下：

```yaml

  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1  # 建议最大不可用pod数量写为1，如果升级或者回滚出现问题，可以缩小影响范围
    type: RollingUpdate

```

更新DaemonSet操作：

```
kubectl set image ds nginx nginx=nginx:1.15.2 --record

kubectl set image ds nginx nginx=nginx:1.16.1 --record

```

注意：建议使用OnDelete方式去进行更新，这样在更新时，可以先操作单个节点的Pod去进行更新，不会影响其它节点正在运行的Pod。  

2.10 Label和Selector

Label：对k8s中各种资源进行分类分组，添加一个具有特别属性的一个标签。
Selector： 通过一些过滤的语法，查找到对应标签的资源。

对一个节点添加label：

```
kubectl lable node k8s-node-02 region=subnet7

```

通过label筛选出该节点：

```
kubectl get node -l region=subnet7

```

查看所有容器的labels

```
kubectl get po -A --show-labels

```

注意：谨慎修改经常变动资源的标签，例如Pod或者Deployment，如果修改后，再进行部署，该修改的标签无效。

labels实践：

```
// Pod新增label
kubectl label pod busybox state=CN

// Pod删除标签，标签名称后跟上"-"号
kubectl label pod busybox state-

// 根据标签筛选Pod
kubectl get pod -l app=busybox

// 修改已有的标签信息，使用--overwrite
kubectl label pod busybox app=busybox-2 --overwrite

```

selector实践：

```
// 查看所有pod的labels
kubectl get pod -A --show-labels

// 多条件查找，查找带有k8s-app=metrics-server和k8s-app=dashboard-metrics-scraper的pod
kubectl get pod -A -l "k8s-app in (metrics-server,dashboard-metrics-scraper)"

// 设置pod的label为version=1.0
kubectl label pod web-0 version=1.0

// 筛选pod，label中version不等于1.0且app为nginx的pod
kubectl get pod -l version!=1.0,app=nginx

```

2.11 Service


2.11.1 南北流量和东西流量

在Service Mesh微服务架构中，我们常常会听到东西流量和南北流量两个术语。

南北流量（NORTH-SOUTH traffic）和东西流量（EAST-WEST traffic）是数据中心环境中的网络流量模式。下面我们通过一个例子来理解这两个术语。

假设我们尝试通过浏览器访问某些Web应用。Web应用部署在位于某个数据中心的应用服务器中。在多层体系结构中，典型的数据中心不仅包含应用服务器，还包含其他服务器，如负载均衡器、数据库等，以及路由器和交换机等网络组件。假设应用服务器是负载均衡器的前端。

当我们访问web应用时，会发生以下类型的网络流量：一个是客户端（位于数据中心一侧的浏览器）与负载均衡器（位于数据中心）之间的网络流量；另一个是负载均衡器、应用服务器、数据库等之间的网络流量，它们都位于数据中心。

- 南北流量

在这个例子中，前者即即客户端和服务器之间的流量被称为南北流量。简而言之，南北流量是server-client流量。

- 东西流量

不同服务器之间的流量与数据中心或不同数据中心之间的网络流被称为东西流量。简而言之，东西流量是server-server流量。

当下，东西流量远超南北流量，尤其是在当今的大数据生态系统中，比如Hadoop生态系统（大量server驻留在数据中心中，用map reduce处理），server-server流量远大于server-client流量。

大家可能会好奇，东西南北，为什么这么命名。

该命名来自于绘制典型network diagrams的习惯。在图表中，通常核心网络组件绘制在顶部（NORTH），客户端绘制在底部（SOUTH），而数据中心内的不同服务器水平（EAST-WEST）绘制。

2.11.2 传统架构服务访问方式和k8s中服务的访问方式

传统架构：

* nginx反向代理，upstream
* Spring Cloud 注册中心，路由表

k8s：

服务之间访问，通过Service层面进行访问，利用Service匹配后端Pod的label，来决定流量走向。外部访问整个系统，通过ingress代理到Service层面，再由Service层面匹配后向的Pod对外提供服务。

其实本质都是通过路由表的方式确定服务的访问，只是路由表的表达不同。

2.11.3 什么是Service

为什么不通过ip地址的方式访问pod？Pod被删除后，ip地址就会发生变化，重建后的pod就会更新ip地址，原有的访问ip就失效了。

Service可以简单的理解为逻辑上的一组Pod，一种可以访问Pod的策略，而且其它Pod可以通过Service访问到这个Service代理的Pod。相对于Pod而言，它会有一个固定的名称，一旦创建就固定不变。对比Pod而言更稳定。

常用命令：

```
kubectl get svc -A // 获取集群中所有的Service信息

```

当Service创建后，会创建一个同名的Endpoint，该Endpoint会指向Pod的ip地址信息。

2.11.3 定义一个Service

```yaml

apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-svc
  name: nginx-svc
spec:
  ports:
  - name: dns   # Service端口的名称，最好使用端口用途区分
    port: 80    # Service自己的端口, serviceA-->serviceB    http://serviceB
    protocol: TCP  # UDP、TCP、STCP，默认TCP
    targetPort: 80 # 后端应用的端口，两个端口可以不一致，但是此处的端口必须保证填写正确
  - name: https
    port: 443
    protocol: TCP
    targetPort: 443
  selector:       # 通过这个Selector来过滤后向的Pod信息
    app: nginx
  sessionAffinity: None
  type: ClusterIP

```

创建Service：

```
kubectl create -f Service.yaml
```

查看pod的日志信息
```
kubectl logs -f --tail=100 nginx-7f785d94f5-p9qxp

```

注意：访问Service时，如果不在同一个namespace中，需要在访问时添加namespace名称，例如：http://nginx-svc.default。

应用之间的调用，尽量不要使用跨namespace的调用。

Service创建之后，无需关心后端应用有什么变化，直接通过Service访问即可。

2.11.4 通过Service代理k8s外部应用

使用场景：

* 希望在生产环境中使用某个固定的名称而非IP地址访问外部的中间件服务。
* 希望Service执行另一个namespace中或者其他集群中的服务。
* 某个项目正在迁移至k8s集群，但一部分服务仍然在集群外部，此时可以使用Service代理至k8s集群外部的服务。

代理外部服务的Service

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-svc-external
  name: nginx-svc-external
spec:
  ports:
  - name: http   
    port: 80    
    protocol: TCP  
    targetPort: 80 
  # 未使用selector，需要自行创建endpoint
  sessionAffinity: None
  type: ClusterIP

```

创建外部地址的endpoint，以代理百度首页为例：

```yaml

apiVersion: v1
kind: Endpoints
metadata:
  labels:
    app: nginx-svc-external
  name: nginx-svc-external  # endpoint的名称必须和Service的名称一致，否则建立不了连接
  namespace: default
subsets:
- addresses:
  - ip: 220.181.38.148 # 此处为外部服务的IP地址
  # 此处端口信息，必须与Service中设置的端口一致，协议一致，名称一致
  ports:
  - name: http
    port: 80      # 此处为外部服务的端口号
    protocol: TCP

```

创建Service和Endpoint：

```
kubectl create -f service-external.yaml
kubectl create -f service-external-endpoint.yaml

```

如果后续发生了地址变更，直接修改Endpoint的地址，不需要修改Service的配置。

注意：apply = replace + create

反向代理外部域名

```yaml

apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-svc-external-address
  name: nginx-svc-external-address
spec:
  type: ExternalName
  externalName: www.baidu.com

```

少用，出现了跨域的情况，访问http://nginx-svc-external-address，由于先解析到nginx-svc-external-address这个地址，再解析到百度官网地址，导致的跨域。

2.11.5 Service类型

* ClusterIP，在集群内部使用，也是默认值。集群内部可以进行访问，集群外部无法访问。
* ExternalName，通过返回定义的CNAME别名，用的少
* NodePort: 在所有安装了kube-proxy的节点上开启一个端口，此端口可以代理至后端Pod，然后集群外部可以使用节点的IP地址和NodePort端口号访问到集群Pod的服务。默认端口范围为：30000至32767。限制性使用，对外有限暴露端口号。常见于Redis、RabbitMQ等工具集群化部署后，对外提供服务。（NodePort性能有限）。
* LoadBalancer：使用云提供商的负载均衡器公开服务。

2.12 Ingress

为何少使用NodePort？在Service比较多的时候，NodePort的性能会急剧下降；过多的Pod带来端口管理的问题。

2.12.1 Ingress概念

Ingress用于实现用域名的方式访问k8s的内部应用。

服务发布的方式：负载均衡设施 --> Ingress --> Service --> Deployment/Pod/DaemonSet/StatefulSet

番外：什么是eBPF？

2.12.2 helm安装Ingress

安装helm3：

```
// 下载helm 3.5.0 的压缩包
wget https://get.helm.sh/helm-v3.5.0-linux-amd64.tar.gz

// 解压缩
tar -zxvf helm-v3.5.0-linux-amd64.tar.gz

// 将运行文件传入/usr/local/bin/中
mv linux-amd64/helm /usr/local/bin/helm

// 最后执行测试
helm help

```

安装ingress

```
// 添加ingress仓库
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

// 查看已添加仓库
helm repo list

// 搜索ingress安装包
helm search repo ingress-nginx

// 下载安装包
helm pull ingress-nginx/ingress-nginx

// 解压安装包
tar -zxvf ingress-nginx-3.20.1.tgz

```

解压完成后需要修改values.yaml文件，修改后如下：

```yaml

## nginx configuration
## Ref: https://github.com/kubernetes/ingress-nginx/blob/master/controllers/nginx/configuration.md
##

## Overrides for generated resource names
# See templates/_helpers.tpl
# nameOverride:
# fullnameOverride:

controller:
  name: controller
  image:
    # repository: google_containers/ingress-nginx/controller
    repository: pollyduan/ingress-nginx-controller
    tag: "v0.43.0"
    # digest: sha256:9bba603b99bf25f6d117cf1235b6598c16033ad027b143c90fa5b3cc583c5713
    pullPolicy: IfNotPresent
    # www-data -> uid 101
    runAsUser: 101
    allowPrivilegeEscalation: true

  # Configures the ports the nginx-controller listens on
  containerPort:
    http: 80
    https: 443

  # Will add custom configuration options to Nginx https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/
  config: {}

  ## Annotations to be added to the controller config configuration configmap
  ##
  configAnnotations: {}

  # Will add custom headers before sending traffic to backends according to https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/customization/custom-headers
  proxySetHeaders: {}

  # Will add custom headers before sending response traffic to the client according to: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#add-headers
  addHeaders: {}

  # Optionally customize the pod dnsConfig.
  dnsConfig: {}

  # Optionally change this to ClusterFirstWithHostNet in case you have 'hostNetwork: true'.
  # By default, while using host network, name resolution uses the host's DNS. If you wish nginx-controller
  # to keep resolving names inside the k8s network, use ClusterFirstWithHostNet.
  # 使用hostNetwork需要修改为这个值
  dnsPolicy: ClusterFirstWithHostNet

  # Bare-metal considerations via the host network https://kubernetes.github.io/ingress-nginx/deploy/baremetal/#via-the-host-network
  # Ingress status was blank because there is no Service exposing the NGINX Ingress controller in a configuration using the host network, the default --publish-service flag used in standard cloud setups does not apply
  reportNodeInternalIp: false

  # Required for use with CNI based kubernetes installations (such as ones set up by kubeadm),
  # since CNI and hostport don't mix yet. Can be deprecated once https://github.com/kubernetes/kubernetes/issues/23920
  # is merged
  # 推荐使用hostNetwork方式去部署，直接使用宿主机的端口号，性能更好。
  # 可以通过hostNetwork的方式部署到指定的节点上
  hostNetwork: true

  ## Use host ports 80 and 443
  ## Disabled by default
  ##
  hostPort:
    enabled: false
    ports:
      http: 80
      https: 443

  ## Election ID to use for status update
  ##
  electionID: ingress-controller-leader

  ## Name of the ingress class to route through this controller
  ##
  ingressClass: nginx

  # labels to add to the pod container metadata
  podLabels: {}
  #  key: value

  ## Security Context policies for controller pods
  ##
  podSecurityContext: {}

  ## See https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/ for
  ## notes on enabling and using sysctls
  ###
  sysctls: {}
  # sysctls:
  #   "net.core.somaxconn": "8192"

  ## Allows customization of the source of the IP address or FQDN to report
  ## in the ingress status field. By default, it reads the information provided
  ## by the service. If disable, the status field reports the IP address of the
  ## node or nodes where an ingress controller pod is running.
  publishService:
    enabled: true
    ## Allows overriding of the publish service to bind to
    ## Must be <namespace>/<service_name>
    ##
    pathOverride: ""

  ## Limit the scope of the controller
  ##
  scope:
    enabled: false
    namespace: ""   # defaults to .Release.Namespace

  ## Allows customization of the configmap / nginx-configmap namespace
  ##
  configMapNamespace: ""   # defaults to .Release.Namespace

  ## Allows customization of the tcp-services-configmap
  ##
  tcp:
    configMapNamespace: ""   # defaults to .Release.Namespace
    ## Annotations to be added to the tcp config configmap
    annotations: {}

  ## Allows customization of the udp-services-configmap
  ##
  udp:
    configMapNamespace: ""   # defaults to .Release.Namespace
    ## Annotations to be added to the udp config configmap
    annotations: {}

  # Maxmind license key to download GeoLite2 Databases
  # https://blog.maxmind.com/2019/12/18/significant-changes-to-accessing-and-using-geolite2-databases
  maxmindLicenseKey: ""

  ## Additional command line arguments to pass to nginx-ingress-controller
  ## E.g. to specify the default SSL certificate you can use
  ## extraArgs:
  ##   default-ssl-certificate: "<namespace>/<secret_name>"
  extraArgs: {}

  ## Additional environment variables to set
  extraEnvs: []
  # extraEnvs:
  #   - name: FOO
  #     valueFrom:
  #       secretKeyRef:
  #         key: FOO
  #         name: secret-resource

  ## DaemonSet or Deployment
  ##
  # 使用DaemonSet方式部署
  kind: DaemonSet

  ## Annotations to be added to the controller Deployment or DaemonSet
  ##
  annotations: {}
  #  keel.sh/pollSchedule: "@every 60m"

  ## Labels to be added to the controller Deployment or DaemonSet
  ##
  labels: {}
  #  keel.sh/policy: patch
  #  keel.sh/trigger: poll


  # The update strategy to apply to the Deployment or DaemonSet
  ##
  updateStrategy: {}
  #  rollingUpdate:
  #    maxUnavailable: 1
  #  type: RollingUpdate

  # minReadySeconds to avoid killing pods before we are ready
  ##
  minReadySeconds: 0


  ## Node tolerations for server scheduling to nodes with taints
  ## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
  ##
  tolerations: []
  #  - key: "key"
  #    operator: "Equal|Exists"
  #    value: "value"
  #    effect: "NoSchedule|PreferNoSchedule|NoExecute(1.6 only)"

  ## Affinity and anti-affinity
  ## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity
  ##
  affinity: {}
    # # An example of preferred pod anti-affinity, weight is in the range 1-100
    # podAntiAffinity:
    #   preferredDuringSchedulingIgnoredDuringExecution:
    #   - weight: 100
    #     podAffinityTerm:
    #       labelSelector:
    #         matchExpressions:
    #         - key: app.kubernetes.io/name
    #           operator: In
    #           values:
    #           - ingress-nginx
    #         - key: app.kubernetes.io/instance
    #           operator: In
    #           values:
    #           - ingress-nginx
    #         - key: app.kubernetes.io/component
    #           operator: In
    #           values:
    #           - controller
    #       topologyKey: kubernetes.io/hostname

    # # An example of required pod anti-affinity
    # podAntiAffinity:
    #   requiredDuringSchedulingIgnoredDuringExecution:
    #   - labelSelector:
    #       matchExpressions:
    #       - key: app.kubernetes.io/name
    #         operator: In
    #         values:
    #         - ingress-nginx
    #       - key: app.kubernetes.io/instance
    #         operator: In
    #         values:
    #         - ingress-nginx
    #       - key: app.kubernetes.io/component
    #         operator: In
    #         values:
    #         - controller
    #     topologyKey: "kubernetes.io/hostname"

  ## Topology spread constraints rely on node labels to identify the topology domain(s) that each Node is in.
  ## Ref: https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/
  ##
  topologySpreadConstraints: []
    # - maxSkew: 1
    #   topologyKey: failure-domain.beta.kubernetes.io/zone
    #   whenUnsatisfiable: DoNotSchedule
    #   labelSelector:
    #     matchLabels:
    #       app.kubernetes.io/instance: ingress-nginx-internal

  ## terminationGracePeriodSeconds
  ## wait up to five minutes for the drain of connections
  ##
  terminationGracePeriodSeconds: 300

  ## Node labels for controller pod assignment
  ## Ref: https://kubernetes.io/docs/user-guide/node-selection/
  ##
  nodeSelector:
    kubernetes.io/os: linux
    # 指定部署的节点所含有的标签
    ingress: "true"

  ## Liveness and readiness probe values
  ## Ref: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes
  ##
  livenessProbe:
    failureThreshold: 5
    initialDelaySeconds: 10
    periodSeconds: 10
    successThreshold: 1
    timeoutSeconds: 1
    port: 10254
  readinessProbe:
    failureThreshold: 3
    initialDelaySeconds: 10
    periodSeconds: 10
    successThreshold: 1
    timeoutSeconds: 1
    port: 10254

  # Path of the health check endpoint. All requests received on the port defined by
  # the healthz-port parameter are forwarded internally to this path.
  healthCheckPath: "/healthz"

  ## Annotations to be added to controller pods
  ##
  podAnnotations: {}

  replicaCount: 1

  minAvailable: 1

  # Define requests resources to avoid probe issues due to CPU utilization in busy nodes
  # ref: https://github.com/kubernetes/ingress-nginx/issues/4735#issuecomment-551204903
  # Ideally, there should be no limits.
  # https://engineering.indeedblog.com/blog/2019/12/cpu-throttling-regression-fix/
  # 临时使用默认的配置，在生产环境中需要重新设置
  # 如果是专有节点，需要尽可能配置的更大
  resources:
  #  limits:
  #    cpu: 100m
  #    memory: 90Mi
    requests:
      cpu: 100m
      memory: 90Mi

  # Mutually exclusive with keda autoscaling
  autoscaling:
    enabled: false
    minReplicas: 1
    maxReplicas: 11
    targetCPUUtilizationPercentage: 50
    targetMemoryUtilizationPercentage: 50

  autoscalingTemplate: []
  # Custom or additional autoscaling metrics
  # ref: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-custom-metrics
  # - type: Pods
  #   pods:
  #     metric:
  #       name: nginx_ingress_controller_nginx_process_requests_total
  #     target:
  #       type: AverageValue
  #       averageValue: 10000m

  # Mutually exclusive with hpa autoscaling
  keda:
    apiVersion: "keda.sh/v1alpha1"
  # apiVersion changes with keda 1.x vs 2.x
  # 2.x = keda.sh/v1alpha1
  # 1.x = keda.k8s.io/v1alpha1
    enabled: false
    minReplicas: 1
    maxReplicas: 11
    pollingInterval: 30
    cooldownPeriod: 300
    restoreToOriginalReplicaCount: false
    triggers: []
 #     - type: prometheus
 #       metadata:
 #         serverAddress: http://<prometheus-host>:9090
 #         metricName: http_requests_total
 #         threshold: '100'
 #         query: sum(rate(http_requests_total{deployment="my-deployment"}[2m]))

    behavior: {}
 #     scaleDown:
 #       stabilizationWindowSeconds: 300
 #       policies:
 #       - type: Pods
 #         value: 1
 #         periodSeconds: 180
 #     scaleUp:
 #       stabilizationWindowSeconds: 300
 #       policies:
 #       - type: Pods
 #         value: 2
 #         periodSeconds: 60

  ## Enable mimalloc as a drop-in replacement for malloc.
  ## ref: https://github.com/microsoft/mimalloc
  ##
  enableMimalloc: true

  ## Override NGINX template
  customTemplate:
    configMapName: ""
    configMapKey: ""

  service:
    enabled: true

    annotations: {}
    labels: {}
    # clusterIP: ""

    ## List of IP addresses at which the controller services are available
    ## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
    ##
    externalIPs: []

    # loadBalancerIP: ""
    loadBalancerSourceRanges: []

    enableHttp: true
    enableHttps: true

    ## Set external traffic policy to: "Local" to preserve source IP on
    ## providers supporting it
    ## Ref: https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-typeloadbalancer
    # externalTrafficPolicy: ""

    # Must be either "None" or "ClientIP" if set. Kubernetes will default to "None".
    # Ref: https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies
    # sessionAffinity: ""

    # specifies the health check node port (numeric port number) for the service. If healthCheckNodePort isn’t specified,
    # the service controller allocates a port from your cluster’s NodePort range.
    # Ref: https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip
    # healthCheckNodePort: 0

    ports:
      http: 80
      https: 443

    targetPorts:
      http: http
      https: https
    # 自有资源部署，不使用LoadBalancer，使用clusterIP进行部署
    type: ClusterIP

    # type: NodePort
    # nodePorts:
    #   http: 32080
    #   https: 32443
    #   tcp:
    #     8080: 32808
    nodePorts:
      http: ""
      https: ""
      tcp: {}
      udp: {}

    ## Enables an additional internal load balancer (besides the external one).
    ## Annotations are mandatory for the load balancer to come up. Varies with the cloud service.
    internal:
      enabled: false
      annotations: {}

      # loadBalancerIP: ""

      ## Restrict access For LoadBalancer service. Defaults to 0.0.0.0/0.
      loadBalancerSourceRanges: []

      ## Set external traffic policy to: "Local" to preserve source IP on
      ## providers supporting it
      ## Ref: https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-typeloadbalancer
      # externalTrafficPolicy: ""

  extraContainers: []
  ## Additional containers to be added to the controller pod.
  ## See https://github.com/lemonldap-ng-controller/lemonldap-ng-controller as example.
  #  - name: my-sidecar
  #    image: nginx:latest
  #  - name: lemonldap-ng-controller
  #    image: lemonldapng/lemonldap-ng-controller:0.2.0
  #    args:
  #      - /lemonldap-ng-controller
  #      - --alsologtostderr
  #      - --configmap=$(POD_NAMESPACE)/lemonldap-ng-configuration
  #    env:
  #      - name: POD_NAME
  #        valueFrom:
  #          fieldRef:
  #            fieldPath: metadata.name
  #      - name: POD_NAMESPACE
  #        valueFrom:
  #          fieldRef:
  #            fieldPath: metadata.namespace
  #    volumeMounts:
  #    - name: copy-portal-skins
  #      mountPath: /srv/var/lib/lemonldap-ng/portal/skins

  extraVolumeMounts: []
  ## Additional volumeMounts to the controller main container.
  #  - name: copy-portal-skins
  #   mountPath: /var/lib/lemonldap-ng/portal/skins

  extraVolumes: []
  ## Additional volumes to the controller pod.
  #  - name: copy-portal-skins
  #    emptyDir: {}

  extraInitContainers: []
  ## Containers, which are run before the app containers are started.
  # - name: init-myservice
  #   image: busybox
  #   command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']

  # 准入控制配置
  # 0.40以后可用
  admissionWebhooks:
    annotations: {}
    enabled: true
    failurePolicy: Fail
    # timeoutSeconds: 10
    port: 8443
    certificate: "/usr/local/certificates/cert"
    key: "/usr/local/certificates/key"
    namespaceSelector: {}
    objectSelector: {}

    service:
      annotations: {}
      # clusterIP: ""
      externalIPs: []
      # loadBalancerIP: ""
      loadBalancerSourceRanges: []
      servicePort: 443
      type: ClusterIP

    patch:
      enabled: true
      image:
        # repository: docker.io/jettech/kube-webhook-certgen
        repository: jettech/kube-webhook-certgen
        tag: v1.5.0
        pullPolicy: IfNotPresent
      ## Provide a priority class name to the webhook patching job
      ##
      priorityClassName: ""
      podAnnotations: {}
      nodeSelector: {}
      tolerations: []
      runAsUser: 2000

  metrics:
    port: 10254
    # if this port is changed, change healthz-port: in extraArgs: accordingly
    enabled: false

    service:
      annotations: {}
      # prometheus.io/scrape: "true"
      # prometheus.io/port: "10254"

      # clusterIP: ""

      ## List of IP addresses at which the stats-exporter service is available
      ## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
      ##
      externalIPs: []

      # loadBalancerIP: ""
      loadBalancerSourceRanges: []
      servicePort: 9913
      type: ClusterIP
      # externalTrafficPolicy: ""
      # nodePort: ""

    serviceMonitor:
      enabled: false
      additionalLabels: {}
      namespace: ""
      namespaceSelector: {}
      # Default: scrape .Release.Namespace only
      # To scrape all, use the following:
      # namespaceSelector:
      #   any: true
      scrapeInterval: 30s
      # honorLabels: true
      targetLabels: []
      metricRelabelings: []

    prometheusRule:
      enabled: false
      additionalLabels: {}
      # namespace: ""
      rules: []
        # # These are just examples rules, please adapt them to your needs
        # - alert: NGINXConfigFailed
        #   expr: count(nginx_ingress_controller_config_last_reload_successful == 0) > 0
        #   for: 1s
        #   labels:
        #     severity: critical
        #   annotations:
        #     description: bad ingress config - nginx config test failed
        #     summary: uninstall the latest ingress changes to allow config reloads to resume
        # - alert: NGINXCertificateExpiry
        #   expr: (avg(nginx_ingress_controller_ssl_expire_time_seconds) by (host) - time()) < 604800
        #   for: 1s
        #   labels:
        #     severity: critical
        #   annotations:
        #     description: ssl certificate(s) will expire in less then a week
        #     summary: renew expiring certificates to avoid downtime
        # - alert: NGINXTooMany500s
        #   expr: 100 * ( sum( nginx_ingress_controller_requests{status=~"5.+"} ) / sum(nginx_ingress_controller_requests) ) > 5
        #   for: 1m
        #   labels:
        #     severity: warning
        #   annotations:
        #     description: Too many 5XXs
        #     summary: More than 5% of all requests returned 5XX, this requires your attention
        # - alert: NGINXTooMany400s
        #   expr: 100 * ( sum( nginx_ingress_controller_requests{status=~"4.+"} ) / sum(nginx_ingress_controller_requests) ) > 5
        #   for: 1m
        #   labels:
        #     severity: warning
        #   annotations:
        #     description: Too many 4XXs
        #     summary: More than 5% of all requests returned 4XX, this requires your attention

  ## Improve connection draining when ingress controller pod is deleted using a lifecycle hook:
  ## With this new hook, we increased the default terminationGracePeriodSeconds from 30 seconds
  ## to 300, allowing the draining of connections up to five minutes.
  ## If the active connections end before that, the pod will terminate gracefully at that time.
  ## To effectively take advantage of this feature, the Configmap feature
  ## worker-shutdown-timeout new value is 240s instead of 10s.
  ##
  lifecycle:
    preStop:
      exec:
        command:
          - /wait-shutdown

  priorityClassName: ""

## Rollback limit
##
revisionHistoryLimit: 10

## Default 404 backend
##
defaultBackend:
  ##
  enabled: false

  name: defaultbackend
  # 去除外部连接，通过搜索dockerhub寻找同版本替代品，使用mirrorgooglecontainers替代原来的地址信息
  image:
    # repository: k8s.gcr.io/defaultbackend-amd64
    repository: mirrorgooglecontainers/defaultbackend-amd64
    tag: "1.5"
    pullPolicy: IfNotPresent
    # nobody user -> uid 65534
    runAsUser: 65534
    runAsNonRoot: true
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false

  extraArgs: {}

  serviceAccount:
    create: true
    name:
  ## Additional environment variables to set for defaultBackend pods
  extraEnvs: []

  port: 8080

  ## Readiness and liveness probes for default backend
  ## Ref: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
  ##
  livenessProbe:
    failureThreshold: 3
    initialDelaySeconds: 30
    periodSeconds: 10
    successThreshold: 1
    timeoutSeconds: 5
  readinessProbe:
    failureThreshold: 6
    initialDelaySeconds: 0
    periodSeconds: 5
    successThreshold: 1
    timeoutSeconds: 5

  ## Node tolerations for server scheduling to nodes with taints
  ## Ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
  ##
  tolerations: []
  #  - key: "key"
  #    operator: "Equal|Exists"
  #    value: "value"
  #    effect: "NoSchedule|PreferNoSchedule|NoExecute(1.6 only)"

  affinity: {}

  ## Security Context policies for controller pods
  ## See https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/ for
  ## notes on enabling and using sysctls
  ##
  podSecurityContext: {}

  # labels to add to the pod container metadata
  podLabels: {}
  #  key: value

  ## Node labels for default backend pod assignment
  ## Ref: https://kubernetes.io/docs/user-guide/node-selection/
  ##
  nodeSelector: {}

  ## Annotations to be added to default backend pods
  ##
  podAnnotations: {}

  replicaCount: 1

  minAvailable: 1


  resources: {}
  # limits:
  #   cpu: 10m
  #   memory: 20Mi
  #  requests:
  #    cpu: 100m
  #    memory: 120Mi

  autoscaling:
    enabled: false
    minReplicas: 1
    maxReplicas: 2
    targetCPUUtilizationPercentage: 50
    targetMemoryUtilizationPercentage: 50

  service:
    annotations: {}

    # clusterIP: ""

    ## List of IP addresses at which the default backend service is available
    ## Ref: https://kubernetes.io/docs/user-guide/services/#external-ips
    ##
    externalIPs: []

    #loadBalancerIP: ""
    loadBalancerSourceRanges: []
    servicePort: 80
    type: ClusterIP

  priorityClassName: ""

## Enable RBAC as per https://github.com/kubernetes/ingress/tree/master/examples/rbac/nginx and https://github.com/kubernetes/ingress/issues/266
rbac:
  create: true
  scope: false

# If true, create & use Pod Security Policy resources
# https://kubernetes.io/docs/concepts/policy/pod-security-policy/
podSecurityPolicy:
  enabled: false

serviceAccount:
  create: true
  name:

## Optional array of imagePullSecrets containing private registry credentials
## Ref: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
imagePullSecrets: []
# - name: secretName

# TCP service key:value pairs
# Ref: https://github.com/kubernetes/contrib/tree/master/ingress/controllers/nginx/examples/tcp
##
tcp: {}
#  8080: "default/example-tcp-svc:9000"

# UDP service key:value pairs
# Ref: https://github.com/kubernetes/contrib/tree/master/ingress/controllers/nginx/examples/udp
##
udp: {}
#  53: "kube-system/kube-dns:53"

```

需要修改的位置：

a) Controller和admissionWebhook的镜像地址，需要将公网镜像同步至公司内网镜像仓库 
b) hostNetwork 设置为 true
c) dnsPolicy设置为 ClusterFirstWithHostNet
d) NodeSelector添加ingress: "true"部署至指定节点
e) 类型更改为 kind: DaemonSet

注意：在yaml中写入true时一定要加引号，例如 nginx: "true"。

最后使用命令安装ingress-nginx：

```
// 创建命名空间
kubectl create ns ingress-nginx

// 对节点打标签，将目标ingress部署到该节点
kubectl label node k8s-master-03 ingress=true

// 安装
helm install ingress-nginx -n ingress-nginx .

```

2.12.3 ingress的扩容缩容

对节点打标签，进行ingress的扩容操作

```
kubectl label node k8s-master-02 ingress=true
```

DaemonSet会监听各个节点机器的变化，如果有新的节点添加了*ingress=true*这个标签，DaemonSet会控制在该节点上新部署一个ingress的实例。

查看ingress信息：

```
kubectl get pod -n ingress-nginx -o wide

NAME                             READY   STATUS    RESTARTS   AGE     IP               NODE            NOMINATED NODE   READINESS GATES
ingress-nginx-controller-549t5   1/1     Running   0          4h33m   192.168.229.53   k8s-master-03   <none>           <none>
ingress-nginx-controller-ktx9r   1/1     Running   0          5m4s    192.168.229.52   k8s-master-02   <none>           <none>


```

在节点上删除*ingress=true*这个标签，DaemonSet将会删除在该节点上部署的ingress实例，如下：

```
kubectl label node k8s-master-02 ingress-
node/k8s-master-02 labeled

kubectl get pod -n ingress-nginx -o wide
NAME                             READY   STATUS        RESTARTS   AGE     IP               NODE            NOMINATED NODE   READINESS GATES
ingress-nginx-controller-549t5   1/1     Running       0          4h35m   192.168.229.53   k8s-master-03   <none>           <none>
ingress-nginx-controller-ktx9r   1/1     Terminating   0          7m24s   192.168.229.52   k8s-master-02   <none>           <none>

```

**注意：**如果在外部存在负载均衡器，例如硬件F5，软件上云服务厂商的SLB，在添加时需要先启动新的ingress实例，然后在负载均衡器中添加该节点的访问信息；在删除时，需要先在负载均衡器中删除该ingress配置的访问信息，再停止该实例，避免出现服务宕机的情况。

ingress实质是在宿主机上使用了宿主机的端口和网络，并不是使用kube-proxy进行代理。

2.12.4 配置ingress进行服务发布

通过读取配置文件的annotations节点下的配置信息，可以通过ingress-controller动态生成nginx的配置文件，达成站点部署的操作。

rewrite示例，配置文件如下:

```yaml

apiVersion: networking.k8s.io/v1beta1   # 1.19以后建议使用networking.k8s.io/v1，但是ingress-controller有可能不支持v1，extensions/v1beta1不要再写
kind: Ingress
metadata:
  annotations:
    # ingress的配置写在annotations中 
    kubernetes.io/ingress.class: "nginx"  # 在创建ingress的时候通过ingressClass指定了该名称
  name: example
spec:
  rules:  # 一个ingress可以配置多个rule
  - host: foo.bar.com  # 域名配置，可以不写，匹配*, 例如*.bar.com
    http:
      paths:  # 相当于nginx的location配置，同一个host可以配置多个path
      - backend:
          serviceName: nginx-svc
          servicePort: 80
        path: /
      # - backend:
      #     serviceName: http-svc-abc
      #     servicePort: 80
      #   path: /abc

```

通过配置域名的方式反向代理到已部署的nginx上。使上述配置生效，如下：

```
kubectl create -f ingress.yaml

```

**重大问题：访问504、502错误码的问题**



注意：多域名配置时，在annotion中设置rewrite时，需要rewrite方式转发的域名写在一个配置文件中，不需要的写在另一个文件中。

2.12.5 ingress排错

安装kubectl工具的krew插件管理工具，以此为基础，安装ingress的调试工具。

首先安装krew，执行下面的脚本内容

````bash
(
  set -x; cd "$(mktemp -d)" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/krew.tar.gz" &&
  tar zxvf krew.tar.gz &&
  KREW=./krew-"$(uname | tr '[:upper:]' '[:lower:]')_$(uname -m | sed -e 's/x86_64/amd64/' -e 's/arm.*$/arm/')" &&
  "$KREW" install krew
)

// （无科学上网的情况下）或者考虑提前下载好krew的安装包krew.tar.gz，以及配置文件krew.yaml，执行下面的命令

(
  set -x; cd "$(mktemp -d)" &&
  cp /root/k8s-practice/ingress/krew.tar.gz ./krew.tar.gz && 
  cp /root/k8s-practice/ingress/krew.yaml ./krew.yaml && 
  tar zxvf krew.tar.gz &&
  KREW=./krew-"$(uname | tr '[:upper:]' '[:lower:]')_$(uname -m | sed -e 's/x86_64/amd64/' -e 's/arm.*$/arm/')" &&
  "$KREW" install --archive=krew.tar.gz --manifest=krew.yaml && 
  "$KREW" update 
)

// 输出日志如下
Installing plugin: krew
Installed plugin: krew
\
 | Use this plugin:
 |      kubectl krew
 | Documentation:
 |      https://krew.sigs.k8s.io/
 | Caveats:
 | \
 |  | krew is now installed! To start using kubectl plugins, you need to add
 |  | krew's installation directory to your PATH:
 |  |
 |  |   * macOS/Linux:
 |  |     - Add the following to your ~/.bashrc or ~/.zshrc:
 |  |         export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
 |  |     - Restart your shell.
 |  |
 |  |   * Windows: Add %USERPROFILE%\.krew\bin to your PATH environment variable
 |  |
 |  | To list krew commands and to get help, run:
 |  |   $ kubectl krew
 |  | For a full list of available plugins, run:
 |  |   $ kubectl krew search
 |  |
 |  | You can find documentation at
 |  |   https://krew.sigs.k8s.io/docs/user-guide/quickstart/.
 | /
/


````

安装后需要进行配置环境变量

```
$ cat >>EOF< ~/.bashrc
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
EOF

```

安装krew完成后，需要退出终端重新进行登录。这时候使用krew安装ingress排查工具ingress-nginx，如下：

```
kubectl krew update
kubectl krew install ingress-nginx

// 或者直接通过压缩包进行安装，需要先下载该压缩包
$ wget https://github.com/kubernetes/ingress-nginx/releases/download/controller-0.31.0/kubectl-ingress_nginx-linux-amd64.tar.gz

// 还要下载yaml配置文件
$ wget https://github.com/kubernetes-sigs/krew-index/blob/master/plugins/ingress-nginx.yaml

// 通过yaml文件同压缩包一起安装
$ kubectl krew install --manifest=ingress-nginx-plugin.yaml --archive=kubectl-ingress_nginx-linux-amd64.tar.gz
Installing plugin: ingress-nginx
Installed plugin: ingress-nginx
\
 | Use this plugin:
 |      kubectl ingress-nginx
 | Documentation:
 |      https://kubernetes.github.io/ingress-nginx/kubectl-plugin/
/

// 执行命令进行验证
kubectl ingress-nginx --help

```

下面就可以进行问题排查了。


**参考文章**

* https://segmentfault.com/a/1190000023088442
* https://segmentfault.com/a/1190000021784761

2.12.6 ingress替代品：traefik

需要解决的问题是，如何让traefik使用ClusterIP的方式部署在node节点，通过80端口访问。

需要验证的问题是，如果ingress-nginx使用默认部署的方式，通过port-forward是否可以正常进行访问？


2.13 ConfigMap && Secret

2.13.1 ConfigMap

ConfigMap管理配置文件或者一些大量的环境变量信息。ConfigMap将配置和Pod分离，更易于配置文件的更改和管理。

Secret更倾向于存储和共享敏感、加密的配置信息。

2.13.2 创建ConfigMap

通过本地目录下的多个配置文件创建配置信息

```
mkdir -p configure-pod-container/configmap

touch game.properties && touch user-interface.properties

cat <<EOF> game.properties
enemy.types=aliens,monsters
player.maximum-lives=5 
EOF

cat <<EOF></EOF> user-interface.properties
color.good=purple
color.bad=yellow
allow.textmode=true
EOF

// 从文件中创建配置信息
kubectl create configmap game-config --from-file=configure-pod-container/configmap/

kubectl describe cm game-config

// 通过yaml文件的方式创建
kubectl get cm game-config -o yaml
```

**注意：**在实际使用过程中，尽可能使用配置文件进行创建，减少直接在命令行中编写配置信息的方式。

通过本地配置文件创建配置信息

```

kubectl create configmap game-config-ui --from-file=configure-pod-container/configmap/user-interface.properties

// 更改配置文件名称，在--from-file=后指定名称
kubectl create configmap game-config-ui-rename --from-file=ui-pro=configure-pod-container/configmap/user-interface.properties

```

创建key-value形式的配置文件，不展示配置文件名称。

```
touch configure-pod-container/configmap/user-env-interface.properties

cat <<EOF> configure-pod-container/configmap/user-env-interface.properties
color.good=red
color.bad=yellow
color.middle=blue
allow.textmode=true
EOF

kubectl create configmap game-config-env --from-env-file=configure-pod-container/configmap/user-env-interface.properties

// 以key-value形式展示，不包含文件名称信息
# kubectl describe configmap game-config-env
Name:         game-config-env
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
allow.textmode:
----
true
color.bad:
----
yellow
color.good:
----
red
color.middle:
----
blue
Events:  <none>


// 查看生成的配置信息
kubectl get configmap game-config-env -o yaml
```

可以通过上述方式生成配置文件信息，利用yaml格式进行创建。

**注意：** 如果指定了多个--from-env-file后，只有最后一个指定的生效，前面指定的配置信息将会被覆盖掉。


从一个字段创建ConfigMap

```
kubectl create cm bite-config --from-literal=special.how=very --from-literal=special.types=vtec --from-literal=special.who=JIM

```

2.13.3 ConfigMap用途

2.13.3.1 作为环境变量的方式

在Pod中读取ConfigMap的信息，实现将上面创建的ConfigMap中的值作为环境变量在Pod中输出。Pod的配置文件如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: test-container
    image: busybox
    command: ["/bin/sh", "-c", "env"]
    env:
    - name: SPECIAL_LEVEL_KEY
      valueFrom: 
        configMapKeyRef:
          name: bite-config
          key: special.how
  restartPolicy: Never

```

创建上面的Pod信息并查看输出的环境变量：

```
kubectl create -f test-pod.yaml

kubectl logs -f mypod | grep SPECIAL_LEVEL_KEY
SPECIAL_LEVEL_KEY=very


// 查看bite-config的key-value信息
kubectl get cm bite-config -o yaml
```

上述方式用的少，尽量使用一个ConfigMap就把所有环境变量包含进去，不要一个个设置。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: default
data:
  SPECIAL_LEVEL: very
  SPECIAL_TYPE: charm

---
apiVersion: v1
kind: Pod
metadata:
  name: mypod-multi
  namespace: default
spec:
  containers:
  - name: test-container-multi
    image: busybox
    command: ["/bin/sh", "-c", "env"]
    envFrom:
    - configMapRef:
        name: env-config # name should be indented under the secretRef or configMapRef fields
    # 手动添加
    env:
    - name: test1
      value: test1-value
    - name: port
      value: "3306"  # value部分只能是字符串
  restartPolicy: Never

```

以*envFrom*这种形式设置环境变量最常用。

2.13.3.2 作为配置文件的方式

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: mypod-volume
  namespace: default
spec:
  containers:
  - name: test-container-volume
    image: busybox
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: config-volume
      mountPath: /mnt/1
    - name: config-volume-file
      mountPath: /mnt/2
  volumes:
  - name: config-volume
    configMap:
      name: env-config
  - name: config-volume-file
    configMap:
      name: game-config-ui
  restartPolicy: Never


```

通过延时来保证Pod运行，然后进入镜像中查看配置信息，如下：

```
# kubectl exec -it mypod-volume -- sh
/ # ls
bin   dev   etc   home  mnt   proc  root  sys   tmp   usr   var
/ # cd /mnt/
/mnt # ls
1  2
/mnt # cd 1/
/mnt/1 # ls
SPECIAL_LEVEL  SPECIAL_TYPE
/mnt/1 # cat SPECIAL_LEVEL
very/mnt/1 # cd ..
/mnt # ls
1  2
/mnt # cd 2/
/mnt/2 # ls
user-interface.properties
/mnt/2 # cat user-interface.properties
color.good=purple
color.bad=yellow
allow.textmode=true

```

key-value的形式一半用作环境变量，文件的形式一般通过挂载的方式作为配置文件使用。

2.13.4 Secret

Secret：用来保存敏感信息的，比如密码、令牌或者key值，例如Redis、MySQL的密码等。

2.13.4.1 Secret创建

通过配置文件来创建Secret

```

echo "admin" > ./admin.txt

echo "1234567" > ./password.txt

kubectl create secret generic db-user-pass --from-file=admin.txt --from-file=password.txt

kubectl get secret db-user-pass -o yaml
apiVersion: v1
data:
  admin.txt: YWRtaW4K
  password.txt: MTIzNDU2Nwo=
kind: Secret
metadata:
  creationTimestamp: "2021-01-26T14:28:41Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:admin.txt: {}
        f:password.txt: {}
      f:type: {}
    manager: kubectl-create
    operation: Update
    time: "2021-01-26T14:28:41Z"
  name: db-user-pass
  namespace: default
  resourceVersion: "798263"
  uid: df3f2dd3-3f5d-4d90-993b-39a5c0ce7845


echo "MTIzNDU2Nwo=" | base64 --decode

```

**注意：**在Secret存储时，使用base64进行了加密。

直接通过命令行创建，但是需要注意的是，在双引号字符串中，使用特殊字符的，必须使用命令行进行转译，例如$，\，*，!，否则无法创建。如果含有特殊字符时使用单引号，则特殊字符不需要转义。

```

kubectl create secret generic db-user --from-literal=username=user001 --from-literal=passwd="\!\\\*QAZxsw2"

// 等同于

kubectl create secret generic db-user-2 --from-literal=username=user001 --from-literal=passwd='!\*QAZxsw2'

```

可以通过配置文件的方式进行创建，但是创建之前需要注意，在data下所使用的字符串必须经过base64加密，而在stringData下使用的字符串则不需要加密，如下：

```yaml

apiVersion: v1
kind: Secret
metadata:
  name: db-user-3
  namespace: default
type: Opaque
data:
  passwd: XCFcXCpRQVp4c3cy
  username: dXNlcjAwMQ==
stringData:
  center: daioafiah
  username: myname

```

执行命令创建

```
kubectl create -f secret.yaml

```

**注意：**stringData优先级高于data，如果stringData中存在和data中同样key值的字符串，以stringData中的为准。

默认类型为Opaque，


2.13.4.2 Secret使用

配置secretName的方式进行读写。

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: mypod-secret
  namespace: default
spec:
  containers:
  - name: test-container-volume
    image: busybox
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: secret-volume
      mountPath: /mnt/1
      readOnly: true  # 只读模式
  volumes:
  - name: secret-volume
    secret:
      secretName: db-user-3
  restartPolicy: Never

```

```
kubectl create -f pod-secret.yaml

```

**注意：**将信息挂载到镜像后，在对应路径的文件中，会以明文的形式保存。

典型用法：ImagePullSecret，Pod拉取私有镜像仓库时使用的账号密码，里面的账号信息会传递给kubelet，然后kubelet就可以拉取有用户名密码设置的镜像仓库中的镜像。

创建一个docker registry类型的secret
```

kubectl create secret docker-registry docker-secret --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL

kubectl get secret docker-secret -o yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJET0NLRVJfUkVHSVNUUllfU0VSVkVSIjp7InVzZXJuYW1lIjoiRE9DS0VSX1VTRVIiLCJwYXNzd29yZCI6IkRPQ0tFUl9QQVNTV09SRCIsImVtYWlsIjoiRE9DS0VSX0VNQUlMIiwiYXV0aCI6IlJFOURTMFZTWDFWVFJWSTZSRTlEUzBWU1gxQkJVMU5YVDFKRSJ9fX0=
kind: Secret
metadata:
  creationTimestamp: "2021-01-27T14:19:59Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:.dockerconfigjson: {}
      f:type: {}
    manager: kubectl-create
    operation: Update
    time: "2021-01-27T14:19:59Z"
  name: docker-secret
  namespace: default
  resourceVersion: "811770"
  uid: 80501ce1-ead7-4b7f-af33-5df46b8674fa
type: kubernetes.io/dockerconfigjson


```

下面将该secret挂载到创建容器的配置文件上，如下：

```yaml

apiVersion: v1
kind: Pod
metadata:
  labels:
    name: tomcat
spec:
  imagePullSecrets:
  - name: docker-secret
  containers:
  - image: tomcat:8.0.51-alpine
    imagePullPolicy: IfNotPresent
    name: tomcat
    ports:
    - containerPort: 8080
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-fzgbs
      readOnly: true
  dnsPolicy: ClusterFirst
  

```

最后执行生效，pod将会从指定的镜像地址中，通过验证信息确定拉取镜像！

也可以将Secret放在ServiceAccount中，pod挂载ServiceAccount来实现私有仓库拉取时的验证！

**补充：** ServiceAccount

2.13.4.3 SubPath使用

由于将配置文件或者Secret挂载到容器内的某个目录时，会覆盖该目录下的所有文件，可能会引起容器工作不正常。利用Subpath来解决目录覆盖的问题。

以nginx容器作为示例，首先导出nginx容器的配置：

```
kubectl exec -it nginx-app-76b5cd66f5-mvc5d -- cat /etc/nginx/nginx.conf > nginx.conf

```

然后通过该配置文件创建ConfigMap，

```
kubectl create configmap nginx-conf --from-file=nginx.conf

```

修改deployment为nginx-app的配置信息，挂载该配置文件，如下：

```
kubectl edit deploy nginx-app
// 修改配置

      name: nginx
      // 3. 修改启动命令
      command: ["sh","-c","sleep 3600"]
      ......
      ......
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        // 2. 添加volumeMounts挂载到容器的目录下
        volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx
      ......
      ......    
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      // 1. 添加volume信息
      volumes:
      - name: config-volume
        configMap:
          name: nginx-conf


```

最后来查看效果，会发现，原来nginx容器/etc/nginx目录下会被覆盖掉。

```
kubectl exec -it       nginx-app-6dcd794b48-hx784 -- ls /etc/nginx
// 只有一条结果，其它的配置文件不存在
nginx.conf

```

使用subpath来解决这个问题，如下：

```
kubectl edit deploy nginx-app
// 修改配置

      name: nginx
      command: ["sh","-c","sleep 3600"]
      ......
      ......
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        volumeMounts:
        - name: config-volume
          // 1. 首先修改mountPath信息，加入文件的完整路径
          mountPath: /etc/nginx/nginx.conf
          // 3. 添加subPath
          // 该处subPath路径和下面编写的items下的路径相同
          subPath：etc/nginx/nginx.conf
      ......
      ......    
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: config-volume
        configMap:
          name: nginx-conf
          // 2. 添加items
          items:
          - key: nginx.conf
            // 注意：path的位置不能以“/”开头
            path: etc/nginx/nginx.conf

```

这样配置完成后，在subPath中以文件形式挂载，就能正常在nginx配置中生效了，下面查看该configMap的信息如下：

```
kubectl get cm nginx-conf -oyaml
// 所有配置信息如下
apiVersion: v1
data:
  nginx.conf: "\r\nuser  nginx;\r\nworker_processes  1;\r\n\r\nerror_log  /var/log/nginx/error.log
    warn;\r\npid        /var/run/nginx.pid;\r\n\r\n\r\nevents {\r\n    worker_connections
    \ 1024;\r\n}\r\n\r\n\r\nhttp {\r\n    include       /etc/nginx/mime.types;\r\n
    \   default_type  application/octet-stream;\r\n\r\n    log_format  main  '$remote_addr
    - $remote_user [$time_local] \"$request\" '\r\n                      '$status
    $body_bytes_sent \"$http_referer\" '\r\n                      '\"$http_user_agent\"
    \"$http_x_forwarded_for\"';\r\n\r\n    access_log  /var/log/nginx/access.log  main;\r\n\r\n
    \   sendfile        on;\r\n    #tcp_nopush     on;\r\n\r\n    keepalive_timeout
    \ 65;\r\n\r\n    #gzip  on;\r\n\r\n    include /etc/nginx/conf.d/*.conf;\r\n}\r\n"
kind: ConfigMap
metadata:
  creationTimestamp: "2021-02-09T02:24:49Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:nginx.conf: {}
    manager: kubectl-create
    operation: Update
    time: "2021-02-09T02:24:49Z"
  name: nginx-conf
  namespace: default
  resourceVersion: "835513"
  uid: 87638177-e623-40e9-96bd-eb39fb84c5c7

// 查看该目录下的所有文件信息，发现目录下的文件不再被覆盖
kubectl exec -it nginx-app-5cbd79f89c-2qwdf -- ls /etc/nginx

conf.d  fastcgi_params  koi-utf  koi-win  mime.types  modules  nginx.conf  scgi_params  uwsgi_params  win-utf

```

subPath也可以用在动态PV中，在一个目录下挂载两个path。

**补充：** 以图形化的方式编写k8s的配置文件。

2.13.4.4 配置信息的热更新

如果ConfigMap和Secret如果是以subPath的形式挂载的，那么Pod是不会感知到ConfigMap和Secret的更新。也就是无法将配置信息同步到Pod中。

如果Pod的变量来自于ConfigMap和Secret中定义的内容，那么ConfigMap和Secret更新后，也不会更新Pod中的变量。

在上一节操作的基础上，添加一组对照的内容，如下：

```
kubectl edit deploy nginx-app
// 修改配置信息
    ......
    ......
    spec:
      containers:
      - command:
        - sh
        - -c
        - sleep 3600
        image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        // 2. 添加新的挂载点
        volumeMounts:
        - mountPath: /etc/nginx/nginx.conf
          name: config-volume
          subPath: etc/nginx/nginx.conf
        // 添加mnt挂载新的路径
        - mountPath: /mnt/
          name: config-volume-non-subpath
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          defaultMode: 420
          items:
          - key: nginx.conf
            path: etc/nginx/nginx.conf
          name: nginx-conf
        name: config-volume
      // 1. 添加新的配置信息
      - configMap:
          defaultMode: 420
          name: nginx-conf
        name: config-volume-non-subpath
        ......
        ......
```

查看最终结果信息，如下：

```

kubectl exec -it nginx-app-d87589bdc-jsc9r -- cat /etc/nginx/nginx.conf
// 输出结果如下
user  nginx;
worker_processes  2;

kubectl exec -it nginx-app-d87589bdc-jsc9r -- cat /mnt/nginx.conf
// 输出结果如下
user  nginx;
worker_processes  2;

```

修改ConfigMap，将work_processes的值改为3，然后再进行查看，如下：

```

kubectl edit cm nginx-conf
// 修改配置信息
worker_processes 3;

// 等待2分钟左右会在Pod中生效

// 查看如下：
kubectl exec -it nginx-app-d87589bdc-jsc9r -- cat /etc/nginx/nginx.conf
// 输出结果如下
user  nginx;
worker_processes  2;

kubectl exec -it nginx-app-d87589bdc-jsc9r -- cat /mnt/nginx.conf
// 输出结果如下
user  nginx;
worker_processes  3;

```

由上述结果对比，subPath的方式不能进行配置信息的热更新。

在热更新修改ConfigMap的时候，有以下三种方式，修改比较完善：

1. （**待补充**）图形化资源管理平台进行修改--Ratel
2. kubectl edit cm ConfigMap名称，在线进行修改
3. 修改配置文件后，使用create + --dry-run命令进行运行，如下：

```
kubectl create cm nginx-conf --from-file=nginx.conf --dry-run=client -oyaml | kubectl replace -f-

```

*--dry-run*:--dry-run='none': Must be "none", "server", or "client". If client strategy, only print the object that would be sent, without sending it. If server strategy, submit server-side request without persisting the resource.

分两步区别的话，通过--dry-run命令生成配置文件的yaml信息，然后使用replace来进行替代。

热更新Secret时，将上述命令中的ConfigMap更换为Secret即可。


**注意：**不可变的ConfigMap和Secret，immutable ConfigMap和immutable Secret。

在yaml配置文件中，添加**immutable: true**这一句配置信息。k8s 1.18版本以后可用。

2.14 存储卷和存储--Volumes

2.14.1 Volumes

解决问题：

1. Container中的磁盘文件是短暂的，但容器崩溃时，kubelet会重新启动容器，但最初的文件会丢失。通过Volumes来保存这些文件。
2. 当一个Pod运行多个Container时，各个容器可能需要共享一些文件，通过Volumes来进行共享文件的存储。同时可以支持多个Pod之间的文件共享。

总结：一些需要持久化数据的程序或者需要共享数据的容器需要使用Volumes。例如：Redis-Cluster中的配置文件nodes.conf，日志收集的需求，需要在应用程序的容器里面加一个SideCar，这个容器专门负责收集日志，比如filebeat，它通过volumes共享应用程序的日志文件目录。

其它：nfs作为存储来说效率太低。使用其它的后端存储或者云存储代替。

2.14.2 HostPath模式

hostPath：将节点上的文件和目录挂载到Pod上，如果Pod被删除，将会hostPath指定的路径将会不变。

不推荐使用，无法保证Pod总是部署在特定的节点上，如果更换了节点，将会导致对应的文件找不到，导致运行失败的情况。

演示：通过hostPath挂载timezone文件信息

```yaml

# vim nginx-volumes.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.16.1
        imagePullPolicy: IfNotPresent
        name: nginx
        ports:
        - name: enter
          containerPort: 80
        # 2. 挂载该volume文件信息 
        volumeMounts:
        - mountPath: /etc/timezone
          name: timezone
      # 1. 添加该Volume信息，挂载/etc/timezone文件
      volumes:
      - name: timezone
        hostPath:
          path: /etc/timezone
          type: File

```

使配置信息生效，并且查看时区信息，如下：

```
// 使配置信息生效
# kubectl apply -f nginx-volumes.yaml

# kubectl exec -it nginx-689f8fd586-mq57p -- cat /etc/timezone
Asia/Shanghai

# cat /etc/timezone
Asia/Shanghai

# kubectl exec -it nginx-app-d87589bdc-jsc9r -- cat /etc/timezone
Etc/UTC

```

hostPath支持的类型如下：

|   类型    |     解释         |
| -----------   | ------------- |
| 	| 空字符串（默认）用于向后兼容，这意味着在安装 hostPath 卷之前不会执行任何检查。|
|DirectoryOrCreate	|如果在给定路径上什么都不存在，那么将根据需要创建空目录，权限设置为 0755，具有与 kubelet 相同的组和属主信息。|
|Directory	|在给定路径上必须存在的目录。|
|FileOrCreate	|如果在给定路径上什么都不存在，那么将在那里根据需要创建空文件，权限设置为 0644，具有与 kubelet 相同的组和所有权。|
|File	|在给定路径上必须存在的文件。|
|Socket	|在给定路径上必须存在的 UNIX 套接字。|
|CharDevice	|在给定路径上必须存在的字符设备。|
|BlockDevice	|在给定路径上必须存在的块设备。|

当使用这种类型的卷时要小心，因为：

1. 具有相同配置（例如基于同一 PodTemplate 创建）的多个 Pod 会由于节点上文件的不同 而在不同节点上有不同的行为。
2. 下层主机上创建的文件或目录只能由 root 用户写入。你需要在 特权容器 中以 root 身份运行进程，或者修改主机上的文件权限以便容器能够写入 hostPath 卷。


2.14.3 emptyDir模式

共享目录使用该模式进行，可以被挂载到相同或者不同的卷上。如果删除Pod，emptyDir卷中的数据也将被删除。


默认情况下，emptyDir卷支持节点上的任何介质，可能是SSD、磁盘或网络存储，具体取决于自身的环境。可以将emptyDir.medium字段设置为Memory,让Kubernetes使用tmpfs （内存支持的文件系统），虽然tmpfs非常快，但是tmpfs在节点重启时，数据同样会被清除，并且设置的大小会被计入到Container的内存限制当中。

在上一个hostPath的基础上，进行修改：

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      # 3. 创建一个非同名容器，挂载到/opt目录下
      - image: nginx:1.16.1
        imagePullPolicy: IfNotPresent
        name: nginx
        ports:
        - name: enter
          containerPort: 80
        volumeMounts:
        - mountPath: /opt
          name: share-volume
      # 2. 创建一个非同名容器，挂载到/mnt目录下
      - image: nginx:1.16.1
        imagePullPolicy: IfNotPresent
        name: nginx-2
        ports:
        - name: enter
          containerPort: 80
        volumeMounts:
        - mountPath: /mnt
          name: share-volume

      volumes:
      # 1. 创建一个emptyDir路径
      - name: share-volume
        emptyDir: {}
          # 如果使用Memory的设置，需要去除上面的花括号
          #medium: Memory
      - name: timezone
        hostPath:
          path: /etc/timezone
          type: File

```

在一个Pod中启动多个容器，其网络空间是共享的，而两个nginx容器，均使用了80端口，在启动第二个容器时出现了端口被占用的问题。

临时的解决方式是，修改启动的command命令，例如将第二个pod重新配置command命令信息，如下：

```yaml

......
      - image: nginx:1.16.1
        imagePullPolicy: IfNotPresent
        name: nginx-2
        command: 
        - sh
        - -c
        - sleep 3600
        ports:
        - name: enter
          containerPort: 80
        volumeMounts:
        - mountPath: /mnt
          name: share-volume

......


```

彻底的解决办法是，尽可能不使用相同的pod进行部署，或者修改command启动命令，更换端口号信息。

这样修改完成后，在第一个名称为nginx的container中创建文件并添加内容如下：

```
# kubectl exec -it nginx-5d67cccf59-69fv8 -c nginx --  bash

root@nginx-5d67cccf59-69fv8:/# touch /mnt/123 && echo '111111' > 123

// exit退出该容器

```

第一个container中挂载的目录为/mnt，因此在/mnt目录下创建，而第二个container中挂载的目录为/opt，这样在第二个容器的opt目录中查看第一个容器中创建的文件信息

```

# kubectl exec -it nginx-5d67cccf59-69fv8 -c nginx2 --  bash

root@nginx-5d67cccf59-69fv8:/# cat /opt/123
// 输出内容如下
111111

```

这样就实现了文件的共享。需要注意的是，在含有多个容器的Pod中，指定要访问的容器信息，需要使用"-c"来进行指定。

2.14.4 nfs安装与挂载

在192.168.229.54这台机器上安装nfs，如下：

```

# yum install nfs-utils -y

# systemctl enable nfs-server && systemctl start nfs-server

```
这样就启动了nfs服务，查看当前nfs服务支持的nfs版本，如下：

```
# cat /proc/fs/nfsd/versions
-2 +3 +4 +4.1 +4.2

```

下面开始创建并配置共享目录，如下：

```
# mkdir -p /data/nfs

// 修改export配置文件，加入该目录

# vim /etc/exports
// 添加以下内容
/data/nfs 192.168.229.0/24(rw,sync,no_subtree_check,no_root_squash)

// :wq保存退出

// 配置完成后需要使配置生效
# exportfs -r

# systemctl restart nfs-server
```

这样nfs就安装完成了，重要的是，需要给剩下的每个节点装一下nfs-utils，否则会出现挂载的时候识别不了nfs存储。全部安装完成后，下面回到192.168.229.1这台机器上，尝试挂载nfs，操作如下：

```
# mount -t nfs 192.168.229.54:/data/nfs /mnt/

```

挂载完成后可以在nfs安装的机器上查看挂载信息，如下：

```
# showmount -e
Export list for k8s-node-01:
/data/nfs 192.168.229.0/24

```

这时回到192.168.229.51机器上，创建文件，测试nfs是否可用，如下：

```
# cd /mnt && touch 123 && echo "111111111111111111" > 123

```

在nginx中挂载nfs服务，首先在192.168.229.51所在机器上，卸载nfs服务所在的文件夹，然后再进行挂载。

```

# umount /mnt

// 找到之前的nginx配置文件，通过volumes挂载nfs服务
# cd ~/k8s-practice/volumes && vim nginx-volumes.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.16.1
        imagePullPolicy: IfNotPresent
        name: nginx
        ports:
        - name: enter
          containerPort: 80
        volumeMounts:
        - mountPath: /mnt
          name: share-volume

      - image: nginx:1.15.2
        imagePullPolicy: IfNotPresent
        name: nginx2
        command:
        - sh
        - -c
        - sleep 3600
        ports:
        - name: enter2
          containerPort: 81
        volumeMounts:
        - mountPath: /opt
          name: share-volume
        # 2. 挂载nfs服务
        - mountPath: /mnt
          name: nfs-volume          

      volumes:
      - name: share-volume
        emptyDir: {}
          # 如果使用Memory的设置，需要去除上面的花括号
          #medium: Memory
      - name: timezone
        hostPath:
          path: /etc/timezone
          type: File
      # 1. 添加nfs目录
      - name: nfs-volume
        nfs:
          server: 192.168.229.54
          # 0. 记得在nfs服务器上首先创建该目录，并且在该目录下创建名称为123的文件
          path: /data/nfs/test-dp

// :wq保存退出

// 使配置信息生效并进行查看
# kubectl replace -f nginx-deployment.yaml

// 查看结果
# kubectl exec -it nginx-64fcb85cd9-5gk8x -- ls -al /opt
drwxrwxrwx 2 root root 17 Feb 27 04:25 .
drwxr-xr-x 1 root root 28 Feb 27 04:28 ..
-rw-r--r-- 1 root root 19 Feb 27 04:25 123

```

**注意：**在生产环境中还是不推荐使用nfs存储的，由于nfs没有任何高可用的保障，难以实现高可用的架构。

一般情况下使用pv和pvc的方式挂载nfs。

2.14.5 持久化存储PV和PVC

Volume：NFS、Ceph、GFS

PV：PersistentVolume，NFS、Ceph、GFS，由k8s配置连接不同的存储，PV同样使集群的一类资源，可以用yaml进行定义，相当于一块存储。可以由管理员定义，由PV去连接后向的存储。

PVC：PersistentVolumeClaim，对PV的申请。将PVC挂载到容器中，由容器去使用。

Pod --> PVC --> PV --> Storage

PV/PVC用来管理k8s的存储，降低了复杂度和简化了使用。PV通过不同的配置，连接对应的后向存储，但是PVC连接PV的时候，使用相同的方式连接。PV用作申请存储，而PVC用作绑定存储。

PV又区分为动态存储和静态存储。

2.14.5.1 使用PV连接nfs存储

```yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  # PV的容量
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  # 回收策略
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2


```

回收策略persistentVolumeReclaimPolicy: Retain、Recycle和Delete

- Recycle：回收并销毁文件，rm -rf
- Retain: 保留
- Delete：删除了PVC，同时也会删除PV，这一类的PV需要支持删除的功能（动态存储支持）

Capacity：PV的容量

volumeMode： 挂载类型，FileSystem、block

accessModes：访问模式

- ReadWriteOnce： RWO，可以被单节点以读写模式挂载
- ReadWriteMany： RWX，可以被多节点以读写模式挂载
- ReadOnlyMany： ROX，可以被多个节点以只读模式挂载

storageClassName：PV的类（类名），PVC和PV的这个名字一样才能被绑定，名字不同，读写权限不同是不能进行绑定的。

mountOptions：挂载参数

**注意：**不需要记住所有的配置，只需要知道连接的原理即可。

实际操作，创建PV，配置文件如下：

```yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0001
spec:
  # PV的容量
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  # 回收策略
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs-slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /data/nfs/test-dp
    server: 192.168.229.54



```

注意：PV没有namespace的限制，直接创建即可。但PVC有namespace限制！

```
// 创建pv
# kubectl create -f nginx-pv.yaml

// 查看pv
# kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv0001   1Gi        RWX            Recycle          Available           nfs-slow                9s

```

PV的状态：

- Available：空闲的，没有被任何pvc绑定
- Bound: 已经被pvc绑定
- Released： pvc被删除，但是资源未被重新使用
- Failed：自动回收失败

创建一个pvc去申请使用pv，配置文件如下：

```yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  # 访问策略要和pv中配置的权限一致
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      # 存储大小不能超过pv创建时配置的文件大小
      # 如果超过pv创建时配置的存储大小，将会导致找不到对应的pv
      storage: 1Gi
  # 这个位置名称要和pv中配置的名称一致
  storageClassName: nfs-slow

```

创建pv信息如下：

```
# kubectl create -f nginx-pvc.yaml

# kubectl get pvc
NAME     STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc   Bound    pv0001   1Gi        RWX            nfs-slow       19s

# kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   REASON   AGE
pv0001   1Gi        RWX            Recycle          Bound    default/my-pvc   nfs-slow                12m

```

创建成功后，可以看到pv和pvc的状态，均为bound的状态，pv的claim下也显示了哪个pvc绑定了该pv。

**注意：**PVC不允许使用kubectl edit命令进行更改

最后在deployment中使用这个pvc。

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.16.1
        imagePullPolicy: IfNotPresent
        name: nginx
        ports:
        - name: enter
          containerPort: 80
        volumeMounts:
        - mountPath: /mnt
          name: share-volume
        - mountPath: /opt
          name: nfs-volume

      - image: nginx:1.15.2
        imagePullPolicy: IfNotPresent
        name: nginx2
        command:
        - sh
        - -c
        - sleep 3600
        ports:
        - name: enter2
          containerPort: 81
        volumeMounts:
        # 2. 挂载pvc所在的目录
        - mountPath: /tmp/pvc
          name: pvc-test

      volumes:
      - name: share-volume
        emptyDir: {}
          # 如果使用Memory的设置，需要去除上面的花括号
          #medium: Memory
      - name: timezone
        hostPath:
          path: /etc/timezone
          type: File
      # 1. 新增pvc的引用
      - name: pvc-test
        persistentVolumeClaim:
          # 注意：claimName对应的是pvc的名称
          claimName: my-pvc

```

创建pv的配置文件信息可能有不同，但是对于pvc和volume的挂载，使用方式是一致的。

最后使其生效，然后查看结果：

```
# kubectl create -f nginx-pv-pvc.yaml

# kubectl get pod
NAME                          READY   STATUS      RESTARTS   AGE
nginx-6976f57754-75chk        2/2     Running     0          4m42s
nginx-6976f57754-8fjzp        2/2     Running     0          4m42s
nginx-6976f57754-rp6p5        2/2     Running     0          4m42s

# kubectl exec -it nginx-6976f57754-rp6p5 -c nginx2 -- df -Th
Filesystem                       Type     Size  Used Avail Use% Mounted on
overlay                          overlay   17G  5.0G   13G  30% /
tmpfs                            tmpfs     64M     0   64M   0% /dev
tmpfs                            tmpfs    2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/centos-root          xfs       17G  5.0G   13G  30% /opt
192.168.229.54:/data/nfs/test-dp nfs4      17G  5.0G   13G  30% /tmp/pvc
shm                              tmpfs     64M     0   64M   0% /dev/shm
tmpfs                            tmpfs    2.0G   12K  2.0G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                            tmpfs    2.0G     0  2.0G   0% /proc/acpi
tmpfs                            tmpfs    2.0G     0  2.0G   0% /proc/scsi
tmpfs                            tmpfs    2.0G     0  2.0G   0% /sys/firmware

```

可以看到*192.168.229.54:/data/nfs/test-dp nfs4      17G  5.0G   13G  30% /tmp/pvc*记录已经显示pvc挂载到该容器中。

这时候，可以进入到容器中对应的/tmp/pvc目录下，创建文件并添加内容，最终可以在nfs存储所在的机器上同时看到该文件。

2.14.5.2 常见问题

- 创建PVC之后一致绑定不上PV

1. PVC的空间申请大小大于PV的大小
2. PVC的StorageClassName没有和PV中设定的一致
3. PVC的访问模式accessModes和PV中设定的不一致

- 创建挂在了PVC的Pod之后，一直处于Pending状态

1. PVC没有创建成功，或者被创建
2. PVC和Pod不在同一个namespace下

删除时，必须先删除PV再删除PVC，如果先删除PVC，PV会一直处于Terminal状态，导致容器删不掉的情况。

如果要删除PVC，必须将所有使用PVC的容器删掉。

PVC有namespace的区分，PV则是全局存在的。

Recycle模式下，回收PV时，会主动创建一个回收镜像来进行回收操作。删除PVC以后，k8s或创建一个用于回收的Pod，根据PV的回收策略进行PV的回收，回收完以后，PV的状态就会变成可被绑定的状态，也就是空闲状态，其它Pending状态的PVC如果匹配到这个PV，就可以和这个PV进行绑定。

PV支持selector标签挂载。