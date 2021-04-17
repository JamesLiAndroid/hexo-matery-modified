---
title: Linux日常运维操作
top: false
cover: false
toc: true
mathjax: true
date: 2020-01-31 11:17:55
password:
summary:
tags: DevOps, Linux 运维手册
categories: DevOps
---

# Linux系统初始化以及部署

## 系统版本

1. 系统版本：CentOS 7.6
2. 内核版本：3.10.0-957.27.2.el7.x86_64
3. 虚拟机信息：Vmware Exsi 6.5

## 当前进度

* 未完成消息队列的编写
* 未完成Redis集群搭建的编写
* 未完成mongodb集群搭建的编写

## 系统初始化

首先进行镜像的安装，使用CentOS 7.6的镜像进行安装，采用最小化安装的方式（无图形化界面、不联网更新）。安装完成后进行系统初始化之前，确保机器可以配置外网连接，如果需要内网使用，请选择基本系统安装方式（安装方式说明见附录）。

注意：

    1. “#” 代表在root用户下执行命令，“$” 代表在常规用户下执行命令。

    2. 如果处于“$”常规用户下，可以使用sudo命令进行提权操作。
      
    3. 所有命令使用man查看帮助，例如 man ls 查看ls命令的使用方式。某些命令还可以使用--help项查看帮助信息，例如git --help，查看git命令的相关使用方式。

    4. 文中所有的服务器ip均为假ip信息，需要根据自己的需要进行更换。

### 0.0 查看基础配置信息

* cpu查看

* 内存查看 

* 硬盘查看

* 系统信息获取

### 0. 网络配置

* 设置公网ip或者是内网可访问的地址

        // 1. 先查看机器的网卡信息
        # ip addr
        1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        ...
        2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
        ...

        // 2. 修改网卡的配置文件，对应上方的ens-192
        # vi /etc/sysconfig/network-scripts/ifcfg-ens192 
        // 按照以下配置修改
        TYPE=Ethernet
        PROXY_METHOD=none
        BROWSER_ONLY=no
        BOOTPROTO=static                // 配置为静态ip
        DEFROUTE=yes
        IPV4_FAILURE_FATAL=no
        IPV6INIT=yes
        IPV6_AUTOCONF=yes
        IPV6_DEFROUTE=yes
        IPV6_FAILURE_FATAL=no
        IPV6_ADDR_GEN_MODE=stable-privacy
        NAME=ens192
        UUID=1df6ac77-3294-42c8-b2af-fd81ce6bb5fc
        DEVICE=ens192
        ONBOOT=yes                      // 配置开机启动
        IPADDR=10.0.11.156              // 配置ip地址
        NETMASK=255.255.255.0           // 配置子网掩码
        GATEWAY=10.0.11.254             // 配置网关
        DNS=202.102.152.3               // 配置DNS信息
        ZONE=public
    
        // 3. 重启网络服务
        # systemctl restart network

        // 4. 查看修改结果
        # ip addr
        1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        ...
        2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
        link/ether 00:0c:29:73:f8:de brd ff:ff:ff:ff:ff:ff
        inet 10.0.11.156/24 brd 10.0.11.255 scope global noprefixroute ens192
        valid_lft forever preferred_lft forever

        // 此时表示网络设置完成

如果网络依旧不能使用，例如ping baidu.com不通，但是ping 114.114.114.114成功，这时需要设置DNS服务器地址。

        // 编辑/etc/resolv.conf，如果是新安装的机器，这个文件的内容可能为空
        # vim /etc/resolv.conf

        // 添加下面的内容：
        nameserver 114.114.114.114 // (电信的DNS)

        nameserver 8.8.8.8 //（googel的DNS）
 
        nameserver 1.1.1.1
    
        nameserver 223.5.5.5 // 阿里的公共DNS
        nameserver 223.6.6.6

        nameserver 202.102.152.3 // 山东网通的dns

        // :wq保存文件

        // 重启network服务
        # systemctl restart network


### 1. 软件源配置

* 更新源

        // 0. 转到软件路径下
        # cd /etc/yum.repos.d/

        // 1. 备份一下当前的repo文件
        # mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak

        // 2. 下载yum源，目前这里使用163的
        # wget http://mirrors.163.com/.help/CentOS7-Base-163.repo

        // 可以使用aliyun的yum源，不需要执行下面修改名称的操作
        # wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

        // 3. 执行替换操作
        # mv CentOS7-Base-163.repo CentOS7-Base.repo

        // 4. 将服务器上的软件包信息 现在本地缓存,以提高 搜索 安装软件的速度
        # yum makecache

        // 等待更新完成即可进行更新系统操作

        // 5. 安装epel源---常用工具安装必备
        # yum install -y epel-release
        
        // 6. 安装sysstat---排查工具
        # yum install -y sysstat

        // 7. 安装httpd---添加压测命令ab
        # yum install httpd-tools -y

* 更新系统

        // 使用update命令更新
        // 总的来说更新（update）和升级（upgrade）是相同的，除了事实上 升级 = 更新 + 更新时进行废弃处理。
        # yum update && yum upgrade

* 包管理工具常用命令

        // 搜索软件包
        # yum search 软件包   
        // 安装软件包
        # yum install 软件包  
        // 移除软件包
        # yum remove 软件包
        //更新系统
        # yum update  

* 常用工具安装

        // 1. 安装 Wget
        # yum install -y wget

        // 2. 安装git
        # yum install -y git

        // 3. 安装unzip
        # yum install -y unzip

        // 4. 网络工具箱
        # yum install -y net-tools 

        // 5. curl工具
        # yum install -y curl

        // 6. vim编辑器，比vi工具更现代
        # yum install -y vim 

        // 7. telnet工具
        # yum install -y telnet 

        // 8. 安装pstree工具集
        // psmisc包含三个帮助管理/proc目录的程序。
        // fuser 显示使用指定文件或者文件系统的进程的PID。
        // killall 杀死某个名字的进程，它向运行指定命令的所有进程发出信号。
        // pstree 树型显示当前运行的进程。
        # yum install -y psmisc

        // 9. 安装更直观的htop来替代top命令
        # yum install -y htop

        // 10. 安装sshpass用于shell脚本中传递密码信息
        # yum install -y sshpass

        // 11. 安装tree实现文件夹下批量展示
        # yum install -y tree

        // 12. 安装jq可以对json数据进行分片、过滤、映射和转换，
        # yum install -y jq

        // 13. 安装yum工具
        # yum install -y yum-utils

    注意：“-y”代表同意安装该程序，无需在安装时确定

    一条命令安装如下：

        # yum install -y wget git unzip net-tools curl vim telnet psmisc  tree sshpass jq

### 2. 用户配置

* 添加用户
    
        // 创建用户的命令, -m表示创建home目录信息，centos表示常规使用的用户
        # useradd -m centos 
        // 设定密码
        # passwd centos

        // 创建用户的命令, -m表示创建home目录信息，ftpuser表示常规使用的用户
        # useradd -m ftpuser 
        // 设定密码
        # passwd ftpuser


* 用户权限（非测试环境不允许启用root权限）

        // 使用vim修改文件
        # vim /etc/sudoers

        ##
        ## The COMMANDS section may have other options added to it.
        ##
        ## Allow root to run any commands anywhere
        root    ALL=(ALL)       ALL
        #centos  ALL=NOPASSWD:/usr/libexec/openssh/sftp-server
        centos  ALL=(ALL)       ALL    # 添加该行信息

* 修改主机名称

        // 变更服务器hostname
        # hostnamectl set-hostname your-own-machine-name

* **注意**：一定不要在生产环境启用root权限，原则上不允许在线上环境切换到root用户执行命令。

### 3. 时钟同步（可访问外网的情况下启用）

使用ntpdate+crontab来完成时钟同步。

        // 0. 查看时钟是否正确
        $ date

        // 1. 首先安装ntpdate
        # yum install -y ntpdate

        // 2. 调用ntpdate命令进行时钟修正，确保该命令可用
        # ntpdate -u pool.ntp.org

        // 3. 最后，集成定时任务
        # crontab -e
        
        // 类vim编辑页面，输入i添加下面的内容，表示每10分钟执行一次定时任务，同步时钟信息。
        */10 * * * * /usr/sbin/ntpdate pool.ntp.org > /dev/null 2>&1

        // 点击Esc按钮，退出编辑，输入:wq进行保存。此时定时任务已经设置完成。

        // 4. 重启定时任务服务
        # systemctl restart crond

        // 5. 查看定时任务执行情况，参考下面命令：
        tail -f /var/log/cron

参考链接：https://www.cnblogs.com/frankdeng/p/9005691.html


### 4. 防火墙设置（默认不允许关闭）

此处的防火墙为CentOS系统默认的防火墙服务firewalld，使用firewall-cmd来进行设置端口的开放和关闭。系统初始使用时，默认只开放21（ftp服务）、22（ssh）、80（http web服务）、443（https）等端口。

        // 1. 首先查看firewalld服务是否已经打开
        # systemctl status firewalld

        // 2. 查看目前已选的区信息
        # firewall-cmd --get-default-zone 

        // 一般已启用的区信息为public

        // 3. 查看目前已经开放的端口信息
        # firewall-cmd --list-all 

        // 或者使用下面的命令
        
        # firewall-cmd --zone=public --list-ports

        // 4. 设置要开放的端口，--permanent代表永久生效
        # firewall-cmd --zone=public --add-port=5000/tcp --permanent

        // 5. 设置要开放的端口段，例如19000-19999
        # firewall-cmd --zone=public --add-port=4990-4999/tcp --permanent

        // 6. 关闭已经设置的端口
        # firewall-cmd --zone=public --remove-port=5000/tcp --permanent

        // 7. 重新加载，当设置完端口后，使用该命令生效
        # firewall-cmd --reload
        
        // 8. 重启firewalld服务
        # systemctl restart firewalld

        // 9. 禁用firewalld服务
        # systemctl disable firewalld

        // 10. 启用firewalld服务
        # systemctl start firewalld

### 5. ftp服务设置（可选）

        // 使用yum安装vsftpd
        # yum install -y vsftpd 

        // 启动 FTP 服务
        # systemctl start vsftpd

        // 对ftp服务进行配置，配置信息位于/etc/vsftpd/目录下
        # vim /etc/vsftpd/vsftpd.conf
        
        // 配置文件中找到下面两行
        // 禁用匿名用户  12 YES 改为NO
        anonymous_enable=NO

        // 禁止切换根目录 101 行 删除#
        chroot_local_user=YES 

        // 更换登录端口
        listen=NO
        listen_port=2231

        // :wq保存配置信息

        // 重启服务
        # systemctl restart vsftpd

        // 防火墙开启2231端口对外访问
        # firewall-cmd --zone=public --add-port=2231/tcp --permanent
        # firewall-cmd --reload

        // 添加ftp用户
        # useradd ftpuser

        // 设置密码，如果需要自己设定请替换引号内的信息
        # echo "testftp" | passwd ftpuser --stdin

        // 限制ftpuser只能通过ftp服务访问
        # usermod -s /sbin/nologin ftpuser

        // 为用户分配主目录
        // /home/ftpuser/ 为主目录, 该目录不可上传文件
        // /home/ftpuser/pub 文件只能上传到该目录下（目前设定的是，直接传输到/home/ftpuser/目录下）

        
        // 创建目录结构
        # mkdir -p /home/ftpuser/pub

        // 设置访问权限
        # chmod a-w /home/ftpuser && chmod 777 -R /home/ftpuser/pub

        // 设置为用户主目录
        # usermod -d /home/ftpuser ftpuser

        // 重启下ftp服务
        # systemctl restart vsftpd

    这样ftp服务就设置完成了，可以使用[WinSCP](https://winscp.net/eng/docs/lang:chs)登录或者使用[FileZilla](https://filezilla-project.org/)登录。


### 6. ssh服务设置

        // 首先，确保服务器已安装openssh-server，命令如下：

        // 1. 判断ssh服务已安装
        # yum list installed | grep openssh-server
        // 如果有以下输出，说明已经安装相关组件
        openssh-server.x86_64                7.4p1-21.el7                   @base  

        // 2. 若未安装，首先安装openssh-server
        # yum install -y openssh-server

        // 3. 找到sshd的配置文件
        # cd /etc/ssh
        # vim ./sshd_config

        // 3.1 修改登录端口为15555, 允许任意ip远程登录，如果是内网机器，则不要设置任意ip远程登录
        Port 15555
    
        ListenAddress 0.0.0.0
        ListenAddress ::

        // 3.2 设置不允许root用户远程登录
        PermitRootLogin no

        // 3.3 开启使用用户名密码来作为连接验证
        PasswordAuthentication yes

        // 3.4 加速ssh连接速度
        GSSAPIAuthentication no
        // 如果这句没有需要自己添加，如果被注释请放开注释
        UseDNS no

        // 设置完成后, :wq 退出编辑
    
        // 4. 重启ssh服务，重启网络，并设置开机启动sshd，依次执行下面的命令
        # systemctl restart sshd
        # systemctl restart network
        # systemctl enable sshd

        // 5. 防火墙开启访问信息
        # firewall-cmd --zone=public --add-port=15555/tcp --permanent
        # firewall-cmd --reload
    
    这样ssh服务就设置完毕了，我们在前面已经创建了**centos**用户，因此在客户机上尝试远程连接服务器。以Windows系统为例，使用[Cmder](https://cmder.net/)作为终端工具。打开Cmder终端工具，输入以下信息：

        ssh -p 15555 centos@10.0.11.11

    其中**-p*表示指定端口号，*centos*就是我们在前面设置的用户名，**10.0.11.11*需要替换为你自己的真实ip地址。在出现输入密码的位置，输入用户对应的密码信息，回车即可登录服务器。

    另外，在重启sssh服务的时候，可能会报错，如下：

        # systemctl restart sshd
        Job for sshd.service failed because the control process exited with error code. See "systemctl status sshd.service" and "journalctl -xe" for details.

        // 执行后无法启动，查看报错信息
        
        # journalctl -xe
        
        error: Bind to port 522 on 0.0.0.0 failed: Permission denied.
        error: Bind to port 522 on :: failed: Permission denied.
        fatal: Cannot bind any address.

        // 当端口设置完成后，需要在selinux设置允许该端口作为ssh的连接端口
        
        // 首先安装管理工具
        # yum install -y policycoreutils-python
        
        // 开放该端口使用
        # semanage port -a -t ssh_port_t -p tcp 522

        // 防火墙开启访问信息
        # firewall-cmd --zone=public --add-port=552/tcp --permanent
        # firewall-cmd --reload

    上边552代表你在/etc/ssh/sshd_config中设置的端口号。

    参考地址：https://wildwolf.name/centos-7-how-to-change-ssh-port/

### 7. sshd进程提升优先级

建议使用 **nice** 命令或者**renice**命令将 sshd 进程的优先级调高，这样当系统内存紧张时，还能勉强登陆服务器进行调试，然后分析故障。操作如下：

    // 找到该进程信息，注意查找真正的进程，根据/usr/sbin/sshd -D获取
    $ ps -aux | grep sshd
    // 展示下面的信息
    root      2567  0.0  0.0 158936  5684 ?        Ss   08:34   0:00 sshd: centos [priv]
    centos    2601  0.0  0.0 158936  2452 ?        S    08:35   0:00 sshd: centos@pts/0
    root      2994  0.0  0.0 158936  5684 ?        Ss   08:51   0:00 sshd: centos [priv]
    centos    3000  0.0  0.0 158936  2452 ?        S    08:51   0:00 sshd: centos@pts/1
    centos    4474  0.0  0.0 112712   964 pts/1    S+   09:36   0:00 grep --color=auto sshd
    root     23773  0.0  0.0 112920   460 ?        Ss   Mar11   0:00 /usr/sbin/sshd -D

    // 查看进程优先级
    $  ps ax -o pid,nice,comm | grep sshd
     2567   0 sshd
     2601   0 sshd
     2994   0 sshd
     3000   0 sshd
    23773   0 sshd

    // 修改第一个进程的id信息，将优先级提升到最高
    // 修改的是/usr/sbin/sshd -D启动命令所在的进程
    $ sudo renice -20 -p 23773

    // 查看进程优先级
    $  ps ax -o pid,nice,comm | grep sshd
     2567   0 sshd
     2601   0 sshd
     2994   0 sshd
     3000   0 sshd
    23773 -20 sshd

    // 退出后重新登录，再查看
    $ ps aux | grep sshd
    root      2567  0.0  0.0 158936  5684 ?        Ss   08:34   0:00 sshd: centos [priv]
    centos    2601  0.0  0.0 158936  2452 ?        S    08:35   0:00 sshd: centos@pts/0
    root      4944  0.5  0.0 158936  5684 ?        S<s  09:40   0:00 sshd: centos [priv]
    centos    4959  0.0  0.0 158936  2300 ?        S<   09:41   0:00 sshd: centos@pts/1
    centos    4997  0.0  0.0 112716   960 pts/1    S<+  09:41   0:00 grep --color=auto sshd
    root     23773  0.0  0.0 112920   460 ?        S<s  Mar11   0:00 /usr/sbin/sshd -D

    $ ps ax -o pid,nice,comm | grep sshd
    2567   0 sshd
    2601   0 sshd
    4944 -20 sshd
    4959 -20 sshd
    23773 -20 sshd

可以看到的是sshd进程中，出现**S<s**，"<"代表高优先级进程标志，说明测试成功。

参考链接：

* 关于进程相关命令的说明：https://www.cnblogs.com/alongdidi/p/linux_process.html

### 8.内核升级和优化（非必选）

以内核升级到4.19版本为例，进行操作，需要在root用户下操作。

首先更新并重启机器，需要排除内核信息

```
# yum update -y --exclude=kernel* && reboot

```

在生产环境中必须进行内核升级，如果需要安装docker，强烈建议将CentOS 7的内核版本升级到4.18+。 这里使用4.19版本

```
# cd /root
# wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm
# wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm

```

可以先下载后再上传到服务器上，如果有多个服务器，可以使用下面的示例命令，传输到各个服务器上。

```

// 将内核文件传输到各个机器上

# for i in k8s-master-02 k8s-master-03 k8s-node-01 k8s-node-02; do scp kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm $i:/root/ ;done

```

开始安装内核信息，在服务器上执行以下命令，如下：

```
# cd /root/ && yum localinstall kernel-ml* -y

```

安装完成后，需要更改内核启动顺序，以便启动时自动选择最新的内核版本信息。

```
# grub2-set-default 0 && grub2-mkconfig -o /etc/grub2.cfg
# grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"

```

最后检查内核版本信息。

```
# grubby --default-kernel
// 输出以下版本信息
/boot/vmlinuz-4.19.12-1.el7.elrepo.x86_64

```

所有节点重启，再检查内核信息。


## 软件安装

前提信息：区分使用原生安装还是docker安装

1. 基础设施

    * JDK安装--OpenJDK、OracleJDK

        首先介绍OracleJDK的安装，生产环境中使用的是OracleJDK，版本为1.8.0_202。预设前提是，已经将OracleJDK的安装包下载完毕了。第一步需要判断服务器上是否安装过jdk，执行下面的命令：

            # yum list installed | grep openjdk

        如果已经安装过jdk，例如镜像内已经含有openjdk，需要先移除已安装的jdk。执行下面的命令：

            # yum remove java-1.8.0-openjdk

        第二步，通过ftp的方式将OracleJDK安装文件传入服务器，可以使用图形化工具，例如[WinSCP](https://winscp.net/eng/index.php)、[FileZilla](https://filezilla-project.org/)，这里使用命令行的形式传输，打开Cmder工具，输入以下命令：

            scp -P 15555 jdk-8u202-linux-x64.tar.gz ftpuser@10.0.11.11:/home/ftpuser

        这样就将OracleJDK的安装包传入到服务器了。第三部开始解压，并创建相关目录。

            // 将安装文件解压
            $ tar -zxvf jdk-8u202-linux-x64.tar.gz
            // 创建安装目录
            $ sudo mkdir /usr/local/java/
            // 将解压后的文件夹拷贝到安装目录
            $ sudo cp -r jdk1.8.0_202/ /usr/local/java/

        第四步，开始配置环境变量，用vim打开/etc/profile，进行设置

            $ sudo vim /etc/profile

            // 在文档末尾添加
            # JAVA env
            JAVA_HOME=/usr/local/java/
            JRE_HOME=$JAVA_HOME/jre
            CLASS_PATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
            PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
            export JAVA_HOME JRE_HOME CLASS_PATH PATH

            // 最后输入:wq保存文档

            // 使环境变量生效
            $ sudo source /etc/profile

            // 添加软链接
            $ sudo ln -s /usr/local/java/jdk1.8.0_202/bin/java /usr/bin/java
            $ sudo ln -s /usr/local/java/jdk1.8.0_202/bin/javac /usr/bin/javac
        
        这样OracleJDK就安装完成了，可以执行以下命令进行检查。

            $ java -version
            $ javac -version
        
        如果均能输出java的版本信息，说明安装成功。

        如果是要安装openjdk的话，直接使用下面的命令安装即可：

            $ sudo yum install -y java-1.8.0-openjdk.x86_64

        如果是多版本jdk存在，使用alternatives命令进行管理和切换。[参考下面的链接](https://blog.csdn.net/waplys/article/details/98478247)。
        
    

* docker、docker-compose安装

    安装的docker版本为19.0.3，安装的docker-compose版本为1.24.1。依据下面的命令来安装docker：

        // 确定自己服务器的内核版本
        # uname -r

    确定

        // 更新当前的软件源
        # yum update

        // 卸载旧版本（如果以前安装过旧版本），并清除之前的docker存档(如果需要的话)
        # yum remove docker docker-common docker-selinux docker-engine
        # rm -r /var/lib/docker

        // 安装需要的软件包依赖
        # yum install -y yum-utils \
                    device-mapper-persistent-data \
                    lvm2
        
        // 添加repo信息
        # yum-config-manager \
                --add-repo \
                https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

        // 更新软件源缓存
        # yum makecache fast

        // 安装最新的稳定版本的docker或者安装指定版本的docker，执行一条命令即可
        # yum install -y docker-ce

        // 配置docker服务以及设置开机启动
        # systemctl start docker
        # systemctl enable docker

        // 拉取hello-world镜像测试docker命令是否可用
        # docker pull hello-world
        # docker run hello-world
    
    如果能够正常输出hello-world，则说明docker安装成功。

    安装完成后，需要配置当前用户在不需要root权限的情况下可以正常使用docker的各项命令。例如我们添加centos用户到docker可执行的组中。配置命令如下：

        $ sudo groupadd docker
        $ sudo usermod -aG docker ${USER}
        (或者执行 sudo gpasswd -a ${USER} docker)
        $ sudo systemctl restart docker

        // 建议这时候退出重新登录下，断开ssh连接或者重启下服务器
        // 否则可能在执行docker pull命令的时候报错：Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post http://%2Fvar%2Frun%2Fdocker.sock/v1.40/images/create?fromImage=hello-world&tag=latest: dial unix /var/run/docker.sock: connect: permission denied
        # reboot

        // 这时候也可以尝试，退出ssh，重新使用ssh登录服务器

        // 如果不重启服务器，可以执行下面的命令解决
        $ sudo setfacl --modify user:centos:rw /var/run/docker.sock
        // 更新用户组
        $ newgrp docker

    这样docker就全部安装完成了。可以在用户centos环境下，测试

        $ docker run hello-world

    如果可以正常输出hello-world，则说明成功。

    另外，可能因为某些原因无意间执行了yum update或者apt-get -y upgrade;导致Docker版本升级。为了避免此类问题发生，建议在安装好Docker后对Docker软件进行锁定，防止Docker意外更新。


        // 安装yum-plugin-versionlock插件
        # yum install yum-plugin-versionlock -y

        // 锁定软件包
        # yum versionlock add docker-ce docker-ce-cli

        // 查看已锁定的软件包
        # yum versionlock list

        // 解锁指定的软件包
        # yum versionlock delete <软件包名称>

        // 解锁所有的软件包
        # yum versionlock clear

    最后，安装docker-compose，确保前面已经安装好docker了！

        $ sudo curl -L https://github.com/docker/compose/releases/download/1.24.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

        $ sudo chmod +x /usr/local/bin/docker-compose   

        $ sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

2. 数据库

* MySQL安装
    
    MySQL版本选择5.7.28。分别演示使用rpm安装和docker安装。安装之前，首先需要检查系统中是否存在MariaDB，并删除MariaDB。

        # yum list installed | grep mariadb
        mariadb-libs-x86_64                  ........................................

        // 删除MariaDB
        # yum remove -y mariadb*
    
    由于是测试环境，所以先使用docker安装MySQL镜像。

        // 拉取docker镜像
        $ docker pull mysql:5.7.28

        // 启动docker镜像
        $ docker run --restart=always --name mysql5.7 -v /data/mysql5.7-data:/var/lib/mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456  -d mysql:5.7.28

    当MySQL的docker镜像启动后，登录的用户名默认为root，密码设定为123456。这样启动后，需要打开3306端口，允许外部连接。执行如下：

        # firewall-cmd --zone=public --add-port=15555/tcp --permanent
        # firewall-cmd --reload

    这样就可以通过[Navicat](https://www.navicat.com.cn/)等工具访问MySQL数据库了。
    
    如果选择使用软件源的方式安装，需要进行以下操作：

        // 下载MySQL的YUM源
        $ wget https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm

        // 安装 MySQL 的 YUM 源
        #  rpm -ivh mysql57-community-release-el7-11.noarch.rpm

        // 检查 MySQL 的 YUM 源是否安装成功
        # yum repolist enabled | grep "mysql.*-community.*"

        // 加入缓存信息
        # yum makecache fast

        // 如果结果中出现mysql57-community/x86_64的信息，说明安装成功

        // 查看MySQL版本
        # yum repolist all | grep mysql

        // 安装MySQL，这样安装并不是安装5.7.25版本，有可能是5.7.31这样的版本
        # yum install -y mysql-community-server 

        // 启动MySQL服务
        # systemctl start mysqld

        // 远程访问 MySQL，需要开放 3306 端口：
        # firewall-cmd --permanent --zone=public --add-port=3306/tcp
        # firewall-cmd --permanent --zone=public --add-port=3306/udp
        # firewall-cmd --reload

    由此MySQL就安装完成了，这时候需要对MySQL进行连接。

        $ mysql -u root -p 
    
注意事项:

0. 关于MySQL在安装后修改初始化密码出现的问题解决：

```
$ mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.25

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> UPDATE mysql.user SET authentication_string=PASSWORD('htMySQL789') where USER='root';
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.
mysql> show databases;
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.
mysql> ALTER USER USER() IDENTIFIED BY 'htMySQL789';
ERROR 1819 (HY000): Your password does not satisfy the current policy requirements
mysql> SHOW VARIABLES LIKE 'validate_password%';
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.
mysql> set global validate_password_policy=LOW;
Query OK, 0 rows affected (0.00 sec)

mysql> ALTER USER USER() IDENTIFIED BY 'htMySQL789';
Query OK, 0 rows affected (0.00 sec)

mysql> exit
Bye

```

1. 针对rpm+yum安装，刚安装的 MySQL 是没有密码的，这时如果出现：

    ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)，解决如下：

        // ① 停止 MySQL 服务
        # systemctl stop mysqld 

        // ② 以不检查权限的方式启动 MySQL
        # mysqld --user=root --skip-grant-tables &

        // ③ 再次输入 
        # mysql -u root 或者 mysql

        // ④ 更新密码：
        // MySQL 5.7 以下版本：
        mysql> UPDATE mysql.user SET Password=PASSWORD('123456') where USER='root';

        // MySQL 5.7 版本：
        mysql> UPDATE mysql.user SET authentication_string=PASSWORD('123456') where USER='root';

        // ⑤ 刷新，使之前的修改生效：
        mysql> flush privileges;

        // ⑥ 退出：
        mysql> exit;

设置完之后，输入 mysql -u root -p，这时输入刚设置的密码，就可以登进数据库了。

2.  针对rpm+yum安装，需要设置允许远程访问

默认情况下 MySQL 是不允许远程连接的，所以在 Java 项目或者 MySQLWorkbench 等数据库连接工具连接服务器上的 MySQL 服务的时候会报 "Host 'x.x.x.x' is not allowed to connect to this MySQL server"。可以通过下面的设置解决。详细可以参考之前写的一篇文章 XXX is not allowed to connect to this MySQL server。

    // 授权操作, '0'需要改为之前设置的MySQL密码
    mysql> grant all privileges on *.* to root@"%" identified by '0';

    mysql> flush privileges;

3. 错误信息：ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.

首先登录MySQL，然后进行修改密码的操作，常见于第一次登录的时候，如下：

```
$ mysql -u root -p 
Enter password:

mysql>  alter user 'root'@'localhost' identified by 'jy@MySQL246';

mysql>  flush privileges;

```

* MyCat安装以及高可用

* MongoDB安装
        
通过docker的方式来安装MongoDB测试服务器，版本为3.4。操作如下:

    // 拉取docker镜像
    $ docker pull mongo:3.4
    // 运行镜像
    $ docker run --name mongod -p 27017:27017 -d mongo:3.4 --auth
    // 查看已有的镜像信息
    $ docker ps -a 
    // 进入mongoDB设置登录用户信息
    $ docker exec -it <镜像的md5信息> mongo admin
    // 在下面的>后填写以下命令创建用户
    > db.createUser({user:"root",pwd:"root",roles:[{role:'root',db:'admin'}]})
        
这时MongoDB就部署完成了。

如果是通过软件源安装，这里配置的是4.2版本的mongodb。步骤如下：

    // 添加软件源
    $ sudo vim /etc/yum.repos.d/mongodb-org-4.2.repo

    // 文件中输入以下信息
    [mongodb-org-4.2]
    name=MongoDB Repository
    baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.2/x86_64/
    gpgcheck=1
    enabled=1
    gpgkey=https://www.mongodb.org/static/pgp/server-4.2.asc

    // :wq 保存文件
    // 更新源信息
    $  sudo yum update -y
    // 开始安装mongodb
    $  sudo yum install -y mongodb-org

    // 启动已经安装的mongo服务
    $ sudo systemctl start mongod
    $ sudo systemctl enable mongod

    // 针对mongodb进行设置
    // 1. 进入mongodb的shell操作空间
    $ mongo

    // 2. 切换数据库
    > use admin

    // 3. 添加管理员用户
    > db.createUser({ 
            user: "admin",  
            customData：{description:"superuser"},
            pwd: "admin",  
            roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]  
        })  

    // 4. 退出，以管理员身份重新登录
    > quit()
    $ mongo -u admin -p --authenticationDatabase admin
    // 输入密码

    // 5. 使用指定的库
    > use db001

    // 6. 对需要操作的库进行授权
    > db.createUser({
            user:"user001",
            pwd:"123456",
            roles:[
                {role:"readWrite",db:"db001"},
                'read'// 对其他数据库有只读权限，对db001、db002是读写权限
            ]
        })
    // 6. 退出
    > quit()

这样用户权限就授权完毕了，现在需要对mongodb的配置文件进行修改。

    // 修改配置文件
    $ vim /etc/mongod.conf
        # mongod.conf

        # for documentation of all options, see:
        #   http://docs.mongodb.org/manual/reference/configuration-options/

        # where to write logging data.
        systemLog:
            destination: file
            logAppend: true
            path: /var/log/mongodb/mongod.log

        journal:
            enabled: true
        #  engine:
        #  wiredTiger:

        # how the process runs
        processManagement:
            fork: true  # fork and run in background
            pidFilePath: /var/run/mongodb/mongod.pid  # location of pidfile
            timeZoneInfo: /usr/share/zoneinfo

        # network interfaces
        net:
            port: 27017
            # 修改为所有ip均可访问
            bindIp: 0.0.0.0  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.

        # 添加授权登录的配置信息
        security:
            authorization: enabled
    
    // :wq 保存退出
    // 这时重启mongodb服务  
    $  sudo systemctl restart mongod

这样配置就完成了，随后通过NoSQLBooster for MongoDB工具或者Studio 3T工具进行连接。需要注意的是在javaweb开发时，在配置文件中写入完整的连接字符串，如下：

    data:
        mongodb:
            uri: mongodb://mongo_ex:123456@10.0.11.12:27017/test-visualmodel  

* MongoDB集群搭建

3. 缓存Redis

* Redis单体安装

Redis版本选择Redis 4.0以上版本，因为4.0以上版本可以支持官方的cluster集群模式构建缓存集群，这里使用4.0.14版本。安装步骤如下：

    // 下载Redis安装包
    $ wget http://download.redis.io/releases/redis-4.0.14.tar.gz
    // 解压
    $ tar -zxvf redis-4.0.14.tar.gz
    // 进入文件夹准备安装
    $ cd redis-4.0.14
    // 安装依赖包
    $ sudo yum -y install gcc gcc-c++ kernel-devel
    // 执行编译
    $ make
    // 进行安装，确定安装目录到/usr/local/redis
    $ make PREFIX=/usr/local/redis install
    // copy配置文件redis.conf到/usr/local/redis目录下
    $ cp redis.conf /usr/local/redis
    // 修改Redis的配置文件
    $ vim /usr/local/redis/redis.conf

        # 修改一下配置
        # redis以守护进程的方式运行
        # no表示不以守护进程的方式运行(会占用一个终端)  
        daemonize yes

        # 客户端闲置多长时间后断开连接，默认为0关闭此功能  
        timeout 300

        # 设置redis日志级别，默认级别：notice                    
        loglevel verbose

        # 设置日志文件的输出方式,如果以守护进程的方式运行redis 默认:"" 
        # 并且日志输出设置为stdout,那么日志信息就输出到/dev/null里面去了 
        logfile stdout

        # 设置密码授权
        requirepass <设置密码>
        
        # 监听ip，允许外网连接
            bind 0.0.0.0
    
    // 输入:wq保存并退出编辑

    // 配置环境变量
    # vim /etc/profile
    
    // 在文件末尾进行追加
    export REDIS_HOME=/usr/local/redis
    export PATH=$PATH:$REDIS_HOME/bin

    // 输入:wq保存并退出编辑
    // 使环境变量生效
    # source /etc/profile

    // 需要配置开机启动
    # cd /etc/init.d
    # vim redis

        #!/bin/bash
        #chkconfig: 2345 80 90
        # Simple Redis init.d script conceived to work on Linux systems
        # as it does use of the /proc filesystem.

        PATH=/usr/local/bin:/sbin:/usr/bin:/bin
        REDISPORT=6379
        EXEC=/usr/local/redis/bin/redis-server
        REDIS_CLI=/usr/local/redis/bin/redis-cli
        
        PIDFILE=/var/run/redis.pid
        CONF="/usr/local/redis/etc/redis.conf"
        
        case "$1" in
            start)
                if [ -f $PIDFILE ]
                then
                        echo "$PIDFILE exists, process is already running or crashed"
                else
                        echo "Starting Redis server..."
                        $EXEC $CONF
                fi
                if [ "$?"="0" ] 
                then
                    echo "Redis is running..."
                fi
                ;;
            stop)
                if [ ! -f $PIDFILE ]
                then
                        echo "$PIDFILE does not exist, process is not running"
                else
                        PID=$(cat $PIDFILE)
                        echo "Stopping ..."
                        $REDIS_CLI -p $REDISPORT SHUTDOWN
                        while [ -x ${PIDFILE} ]
                    do
                            echo "Waiting for Redis to shutdown ..."
                            sleep 1
                        done
                        echo "Redis stopped"
                fi
                ;;
        restart|force-reload)
                ${0} stop
                ${0} start
                ;;
        *)
            echo "Usage: /etc/init.d/redis {start|stop|restart|force-reload}" >&2
                exit 1
        esac

    // 输入:wq保存并退出编辑
    // 给脚本增加运行权限
    # chmod +x /etc/init.d/redis
    // 查看服务列表
    # chkconfig --list

    // 添加服务
    # chkconfig --add redis

    // 配置启动级别
    # chkconfig --level 2345 redis on

    // 使配置生效
    # systemctl daemon-reload

    // 启动测试
    # systemctl start redis   #或者 /etc/init.d/redis start  
    # systemctl stop redis   #或者 /etc/init.d/redis stop

    // 添加redis开放端口
    # firewall-cmd --permanent --zone=public --add-port=6379/tcp
    # firewall-cmd --permanent --zone=public --add-port=6379/udp
    # firewall-cmd --reload


* Redis集群--哨兵模式搭建

（进程内缓存使用--caffine--框架进行开发，在编码时使用）

4. 消息队列

    * RocketMQ搭建

5. 依赖管理，仓库和镜像

请参考{% post_link 持续集成实践大纲 %}

6. CI/CD工具链

请参考{% post_link 持续集成实践大纲 %}

7. Zookeeper

* Zookeeper安装和集群搭建

- 使用安装文件进行发布

- 所在目录：/home/centos/zookeeper-3.4.14

- 发布地址：
    10.0.11.11:21810
    10.0.11.12:21810
    10.0.11.13:21810

- 相关命令：

1. 配置环境变量

* 在每一台部署了zookeeper的服务器上

        #修改环境变量文件
        vi /etc/profile

        #增加以下内容
        export ZOOKEEPER_HOME=/usr/zookeeper/zookeeper-3.4.11
        export PATH=$ZOOKEEPER_HOME/bin:$PATH

        #使环境变量生效
        source /etc/profile

        #查看配置结果
        echo $ZOOKEEPER_HOME


2. 配置节点标识


        #针对zk01，也就是11服务器：

        echo "1" > /zookeeper/data/myid
        
        #针对zk02，也就是12服务器：

        echo "2" > /zookeeper/data/myid
        
        #针对zk03，也就是13服务器：

        echo "3" > /zookeeper/data/myid

3. 配置文件修改

* 在每一台部署了zookeeper的服务器上

        #进入ZooKeeper配置目录
        cd $ZOOKEEPER_HOME/conf

        #新建配置文件
        vim zoo.cfg

        #写入以下内容并保存

        tickTime=2000
        initLimit=10
        syncLimit=5
        dataDir=/home/centos/zookeeper-3.4.14/data
        dataLogDir=/home/centos/zookeeper-3.4.14/logs
        clientPort=21810
        server.1=10.0.11.11:28880:38880
        server.2=10.0.11.12:28880:38880
        server.3=10.0.11.13:28880:38880        

4. 开放端口信息

* 在每一台部署了zookeeper的服务器上

        sudo firewall-cmd --add-port=21810/tcp --permanent
        sudo firewall-cmd --add-port=28880/tcp --permanent
        sudo firewall-cmd --add-port=38880/tcp --permanent
        sudo firewall-cmd --reload

5. 启动

* 在每一台部署了zookeeper的服务器上

        #进入ZooKeeper bin目录
        cd $ZOOKEEPER_HOME/bin

        #启动
        nohup sh zkServer.sh start &



- 配置方式：使用ZooInspector访问zookeeper服务并进行配置

* [参考地址](https://juejin.im/post/5ba879ce6fb9a05d16588802)
    

8. ELK体系

请参考{% post_link ELK单机日志系统搭建 %}

9. Docker+kubernetes体系


10. API管理工具--YApi安装

启动方式，先启动MongoDB，再通过pm2 启动yapi项目

请参考[YApi官方](http://yapi.demo.qunar.com/)安装方式。


11. JIRA/Confluence的安装部署

* 前提：需要安装docker和docker-compose，下载docker镜像包

* 首先启动数据库

        docker pull mysql:5.7

        # 启动jira的数据库
        docker run --name mysql-jira --restart always -p 46001:3306 -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_DATABASE=jira -e MYSQL_USER=jira -e MYSQL_PASSWORD=jira -d mysql:5.7 --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci

        # 针对jira的数据库设置

        vim /etc/mysql/conf.d/mysql.cnf

        添加下面的内容：

        [mysql]
        default-character-set=utf8
        [client]
        default-character-set=uft8

        # 启动confluence的数据库
        docker run --name mysql-confluence --restart always -p 46003:3306 -e MYSQL_ROOT_PASSWORD=123456 -e MYSQL_DATABASE=confluence -e MYSQL_USER=confluence -e MYSQL_PASSWORD=confluence -d mysql:5.7 --character-set-server=utf8 --collation-server=utf8_bin
        
        # 针对confluence的数据库的设置

        vim /etc/mysql/conf.d/mysql.cnf

        添加下面的内容：
        
        [mysql]
        default-character-set=utf8

        [client]
        default-character-set=utf8

        [mysqld]
        character-set-server=utf8
        collation-server=utf8_bin

* 下载需要的jira和Confluence镜像信息

        git clone https://github.com/zhangguanzhang/Dockerfile.git

        cd ./Dockerfile/atlassian-jira


* 构建jira镜像

        # 不要忘记后面的“.”，表示当前目录下进行构建
        docker build -t ht-jira:v7 . 

        # 等待一段时间进行构建

        # 查看已经构建完成的镜像
        docker images 

        REPOSITORY                              TAG                 IMAGE ID            CREATED             SIZE
ht-jira                                 v1                  c4a747622b73        45 seconds ago      532MB

        # 启动镜像

        docker run --restart always --detach --link mysql-jira:mysql --publish 46002:8080 ht-jira:v7

* 构建confluence镜像

        # 不要忘记后面的“.”，表示当前目录下进行构建
        docker build -t ht-confluence:v2 . 

        # 等待一段时间进行构建

        # 查看已经构建完成的镜像
        docker images 

        REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
ht-confluence                      v2                  75d3834d330f        27 minutes ago      785MB

        # 启动镜像

        docker run --restart always --detach --link mysql-confluence:mysql --publish 46004:8080 ht-confluence:v2


* jira的用户信息

    1. 管理员信息

    管理员账户：HTCAdmin

    管理员邮箱：lisongyang@123.com

    用户名：HTCAdmin

    密码：123456

    2. 用户信息

    各个用户为名字的汉语拼音字母小写，密码为123456.

* confluence的用户信息

    1. 管理员信息

    全名：HTCAdmin

    管理员邮箱：lsy@123.com

    用户名：htcadmin

    密码：123456

    2. 用户信息

    各个用户为名字的汉语拼音字母小写，密码为123456。


* 访问地址

jira：10.0.11.11:46002
confluence：10.0.11.12.46004


* 参考地址：

        * https://zhangguanzhang.github.io/2019/02/19/jira-confluence/#jira%E6%95%B0%E6%8D%AE%E5%BA%93%E9%85%8D%E7%BD%AE

        * https://blog.csdn.net/weixin_38229356/article/details/84875205

        * https://github.com/zhangguanzhang/Dockerfile.git

        * confluence文档：https://www.cwiki.us/pages/viewpage.action?pageId=917513
        
        * jira文档：https://www.cwiki.us/pages/viewpage.action?pageId=2393502

## 关于服务器内核自动升级问题的解决

由于部分机器进行了

```
sudo yum update

```
操作，导致内核信息进行了升级，导致有一些机器在启动的时候出现了黑屏和无法启动的情况。

由于在启动时，默认选择了新升级的内核进行了启动，导致报错。如下：

![](新内核启动报错.png)

目前可用的稳定版内核为：3.10.0-1062.12.1.el7.x86_64，或者低于该版本的内核信息。

以192.168.229.221机器为例进行查看和修改，首先登陆192.168.229.221，查看已安装的内核信息：

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

## 注意事项

1. 硬盘扩容

参考地址：[参考链接](http://ttlop.com/2016/11/29/Centos-7-LVM-%E7%A3%81%E7%9B%98%E6%89%A9%E5%AE%B9/)

2. 修改运行中的docker配置文件

    首先停止所有的容器

        $ sudo docker stop $(docker ps -a | awk '{ print $1}' | tail -n +2)

    然后停止docker服务

        $ sudo systemctl stop docker

    备份容器的配置文件（需要管理员权限）

        # cd /var/lib/docker/containers/de9c6501cdd3(容器编号)
        # cp hostconfig.json hostconfig.json.bak
        # cp config.v2.json config.v2.json.bak

    修改配置文件，例如

        # vim config.v2.json

    重启docker服务

        $ sudo systemctl start docker

    重启docker镜像

        $ sudo docker start $(docker ps -a | awk '{ print $1}' | tail -n +2

3. CentOS 7 安装方式选择？

* Desktop ：基本的桌面系统，包括常用的桌面软件，如文档查看工具。

* Minimal Desktop：基本的桌面系统，包含的软件更少。

* Minimal：基本的系统，不含有任何可选的软件包。

* Basic Server ：安装的基本系统的平台支持，不包含桌面。

* Database Server：基本系统平台，加上MySQL和PostgreSQL数据库，无桌面。

* Web Server：基本系统平台，加上PHP，Web server，还有MySQL和PostgreSQL数据库的客户端，无桌面。

* Virtual Host：基本系统加虚拟平台。

* Software Development Workstation：包含软件包较多，基本系统，虚拟化平台，桌面环境，开发工具。

而安装Linux基本是用来构建服务器的，所以基本上选择Basic Server即可。如果业务环境无法连接外网，尽量选择比minimal安装更高级别的安装方式，例如Software Development Workstation方式。

4. 选取Linux版本原则

尽量选择LTS（长期支持版）版本，5年期支持，社区有保障。第二选择就是选择次新版本，例如当前发行版本为CentOS 7.6，可以选择CentOS 7.4版本。尽量不要选择CentOS 6系列，因为长期维护支持马上到期。

对照表：

|发行版本 |	全力支持到期 |	维护支持到期|
|-------|-----------|----------|
|CentOS 6 |	2016 |	2020-11 |
|CentOS 7 |	2019 |	2024-06 |

5. 利用scp命令上传下载文件夹

使用scp命令，远程上传下载文件/文件夹
1、从服务器下载文件
scp username@servername:/path/filename /local/path
例如: scp ubuntu@117.50.20.56:/ygf/data/data.txt /desktop/ygf   把117.50.20.56上的/ygf/data/data.txt 的文件下载到/desktop/ygf目录中

 

2、上传本地文件到服务器
scp /local/path/local_filename username@servername:/path
例如: scp /ygf/learning/deeplearning.doc  ubuntu@117.50.20.56:/ygf/learning    把本机/ygf/learning/目录下的deeplearning.doc文件上传到117.50.20.56这台服务器上的/ygf/learning目录中

 

3、从服务器下载整个目录
scp -r username@servername:/path /path
例如: scp  -r  ubuntu@117.50.20.56:/home/ygf/data  /local/local_dir    “-r”命令是文件夹目录，把当前/home/ygf/data目录下所有文件下载到本地/local/local_dir目录中


4、上传目录到服务器
scp  -r  /path  username@servername:/path
例如: scp -r  /ygf/test  ubuntu@117.50.20.56:/ygf/tx     “-r”命令是文件夹目录，把当前/ygf/test目录下所有文件上传到服务器的/ygf/tx/目录中

参考自：https://www.cnblogs.com/tectal/p/9478326.html

6. 关于shell脚本编写问题

执行 shell 脚本 \r 问题解决
一看应该是 windows 的回车换行跟 linux 换行差异，百度了一下，的确有很多类似问题，在 windows 下编辑 shell 文件，输入的回车是 "\r\n" ，导致在 linux 下执行 shell 脚本时报这个 \r 的错。
怎么办？想办法解决。
让开发或测试自己想办法转化或者在 linux 环境编辑这个 build.sh 很明显不现实，在服务器安装 dos2unix 来转化 build.sh 我又要在主从节点分别重新构建镜像，麻烦。想了想用替换的方式看能否实现，百度一下，使用 sed -i 's/\r//' 来处理 build.sh 后再执行，即 sed -i 's/\r//' build.sh && bash build.sh，完美解决问题，大功告成！！！

```
$ sed -i 's/\r//' init.sh
$ sed -i 's/\r//' ssh-keygen-send.sh
```

7. 使用yumdownloader对依赖库进行下载

```
// 创建下载的文件夹
$ sudo mkdir -p /opt/download-package/

// 如果未安装yum-utils，可以先安装
$ sudo yum install yum-utils -y

// 查看 yum-utils 软件包有没有 yumdownloader，如果有输出代表可用
$ sudo rpm -ql yum-utils | grep yumdownloader
/usr/bin/yumdownloader
/usr/share/man/man1/yumdownloader.1.gz

// 下载，--resolve：下载依赖，--destdir：指定下载目录， --donwloadonly：只下载不更新
$ sudo yumdownloader java-1.8.0-openjdk.x86_64 --resolve --destdir=/opt/download-package/ --donwloadonly

```

**注意：**如果openjdk已经安装了，容易导致下载时无法下载相关的依赖信息。