---
title: Zabbix+Grafana运维监控平台搭建
top: false
cover: false
toc: true
mathjax: true
date: 2019-08-15 22:19:47
password:
summary:
tags: DevOps Zabbix Grafana
categories: DevOps
---

# zabbix+grafana进行服务监控和参数图形化展示

## 运行环境

### Linux
CentOS7.6，内核版本3.10，承载Nginx服务和Zabbix+Grafana服务。需要注意的是，Docker 目前支持 CentOS 7 及以后的版本，内核要求至少为 3.10。

### WinServer

- 四台IIS服务
WinServer2012

- 四台SQLServer服务
WinServer2016 + SQLServer2016

- 一台FileServer
WinServer2016

### 监控版本说明

* zabbix 4.2
* grafana 6.1


## 监控信息--利用zabbix进行服务监控

- zabbix安装和部署--利用docker进行安装部署
    - zabbix版本：
    zabbix server和zabbix agent版本均采用v4.2.1版本的程序进行安装部署。
    - CentOS安装docker
    
        注意：Docker 需要用到 `centos-extra` 这个源，如果您关闭了，需要重启启用，可以参考 [Available Repositories for CentOS](https://wiki.centos.org/AdditionalResources/Repositories)。
    
         - 卸载旧版本
        	    		
           ```
           $ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
               docker-logrotate \
        	       docker-engine
        	```
        	
        	旧版本的内容在 /var/lib/docker 下，目录中的镜像(images), 容器(containers), 存储卷(volumes), 和 网络配置（networks）都可以保留。
        	
        	Docker CE 包，目前的包名为 docker-ce。Docker现在分为两个版本，Docker CE和Docker EE
        	其中Docker CE为开源版，Docker EE为企业版
    
   		 - 安装准备
      		 为了方便添加软件源，支持 devicemapper 存储类型，安装如下软件包
            ```
      		$ sudo yum update
            $ sudo yum install -y yum-utils \
                    device-mapper-persistent-data \
                    lvm2
            ```
   		 - 添加 yum 软件源
            添加 Docker 稳定版本的 yum 软件源
            ```
            $ sudo yum-config-manager \
                    --add-repo \
                    https://download.docker.com/linux/centos/docker-ce.repo
            ```
        - 安装 Docker
            更新一下 yum 软件源的缓存，并安装 Docker。
            ```
            $ sudo yum update
            $ sudo yum install docker-ce
            ```
            如果弹出 GPG key 的接收提示，请确认是否为 **060a 61c5 1b55 8a7f 742b 77aa c52f eb6b 621e 9f35**，如果是，可以接受并继续安装。   
        至此，Docker 已经安装完成了，Docker 服务是没有启动的，操作系统里的 docker 组被创建，但是没有用户在这个组里。
        **注意：**
            默认的 docker 组是没有用户的（也就是说需要使用 sudo 才能使用 docker 命令）。
            您可以将用户添加到 docker 组中（此用户就可以直接使用 docker 命令了）。
        加入 docker 用户组命令:
            ```
            $ sudo usermod -aG docker USER_NAME
            ```
            用户组更新信息后，重新登录系统生效。
        - 安装指定版本
        如果想安装指定版本的 Docker，可以查看一下版本并安装。
            ```
            $ yum list docker-ce --showduplicates | sort -r

                docker-ce.x86_64  3:18.09.1-3.el7                     docker-ce-stable
                docker-ce.x86_64  3:18.09.0-3.el7                     docker-ce-stable
                docker-ce.x86_64  18.06.1.ce-3.el7                    docker-ce-stable
                docker-ce.x86_64  18.06.0.ce-3.el7                    docker-ce-stable
            ```
            可以指定版本安装,版本号可以忽略 _:_ 和 _el7_，如 docker-ce-18.09.1
            ```
            $ sudo yum install docker-ce-<VERSION STRING>
            ```
            至此，指定版本的 Docker 也安装完成，同样，操作系统内 docker 服务没有启动，只创建了 docker 组，而且组里没有用户。

        - 启动docker
            如果想添加到开机启动
            ```
            $ sudo systemctl enable docker
            ```
            启动 docker 服务
            ```
            $ sudo systemctl start docker
            ```
        - 验证安装
        验证 Docker CE 安装是否正确，可以运行 hello-world 镜像
            ```
            $ sudo docker run hello-world
            ```
        - 更新和卸载Docker CE
            ```
            # 更新 Docker CE
            $ sudo yum update docker-ce
            # 卸载 Docker CE
            $ sudo yum remove docker-ce
            ```
        - 删除本地文件
        注意，docker 的本地文件，包括镜像(images), 容器(containers), 存储卷(volumes)等，都需要手工删除。默认目录存储在 /var/lib/docker。
            ```
            $ sudo rm -rf /var/lib/docker
            ```
    
    - 通过docker安装zabbix服务
        首先需要把镜像下载下来，分为以下四个
        - mysql
        - zabbix-java-gateway
        - zabbix-server-mysql
        - zabbix-web-apache-mysql

        ```
        # 下载镜像
        $sudo docker pull mysql:5.7
        $sudo docker pull zabbix/zabbix-java-gateway:centos-4.2-latest
        $sudo docker pull zabbix/zabbix-server-mysql:centos-4.2-latest
        $sudo docker pull zabbix/zabbix-web-apache-mysql:centos-4.2-latest
        ```
        然后挨个进行启动

        ```
        # mysql
        $ sudo docker run --name mysql-server -t \
                -e MYSQL_DATABASE="zabbix" \
                -e MYSQL_USER="zabbix" \
                -e MYSQL_PASSWORD="zabbix" \
                -e MYSQL_ROOT_PASSWORD="zabbix" \
                -p 127.0.0.1:3306:3306 \
                -d mysql:5.7 \
                --character-set-server=utf8 \ --collation-server=utf8_bin 
        # zabbix-java-gateway
        $sudo docker run --name zabbix-java-gateway -t \
                -d zabbix/zabbix-java-gateway:latest
        
        # zabbix-server-mysql
        $sudo docker run --name zabbix-server-mysql -t \
                --link mysql-server:mysql \
                -e DB_SERVER_HOST="mysql-server" \
                -e MYSQL_DATABASE="zabbix" \
                -e MYSQL_USER="zabbix" \
                -e MYSQL_PASSWORD="zabbix" \
                -e MYSQL_ROOT_PASSWORD="zabbix" \
                -p 10051:10051 \
                -d \
                zabbix/zabbix-server-mysql:centos-4.2-latest

        # zabbix-web-apache-mysql
        $sudo docker run --name zabbix-web-apache-mysql -t \
            --link mysql-server:mysql \
            --link zabbix-server-mysql:zabbix-server \
            -e DB_SERVER_HOST="mysql-server" \
            -e MYSQL_DATABASE="zabbix" \
            -e MYSQL_USER="zabbix" \
            -e MYSQL_PASSWORD="zabbix" \
            -e MYSQL_ROOT_PASSWORD="zabbix" \
            -e PHP_TZ="Asia/Shanghai" \
            -p 80:8088 \     # 前一个端口为docker镜像端口，后一个端口为服务器本地端口,映射时
            -d \
            zabbix/zabbix-web-apache-mysql:centos-4.2-latest
        ```
        **注意：**
        1. 需要注意端口号占用的问题， 灵活调整
        2. 按顺序进行启动

至此，zabbix服务端就部署完毕了，可以通过浏览器访问localhost:8088进行访问了，输入用户名Admin，密码zabbix，即可进入。

- 补充：Docker镜像开机启动命令，在所有命令后面添加**--restart=always**

- 补充：Docker镜像添加*--restart=always*后，对于zabbix-web-apache-mysql所在镜像，需要及时检查其启动状态。如果一直处在restart状态，需要停止运行、rm，重新启动

- 补充：关于时钟问题，当服务器时钟不准确时，需要去掉"-e PHP_TZ="Asia/Shanghai" "的配置选项！
    
- 补充：Docker常用命令

```
# 查看当前系统 Docker 信息 
docker info

# 拉取 docker 镜像 
docker pull image_name
# 从 Docker hub 上下载某个镜像 
docker pull centos:latest

# 查看宿主机上的镜像，Docker 镜像保存在 / var/lib/docker 目录下:
docker images

REPOSITORY                      TAG                 IMAGE ID            CREATED             SIZE
mysql                           5.7                 1b30b36ae96a        8 days ago          372MB
zabbix/zabbix-web-nginx-mysql   centos-4.0-latest   8be5f91b2fa1        3 weeks ago         415MB
zabbix/zabbix-server-mysql      centos-4.0-latest   8e5becf45c4e        3 weeks ago         326MB

# 删除镜像 
docker rmi image_name/image_id
docker rmi zabbix/zabbix-web-nginx-mysql:centos-4.0-latest 或者 docker rmi 8be5f91b2fa1

# 查看当前有哪些容器正在运行 
docker ps
# 查看所有容器 
docker ps -a

CONTAINER ID        IMAGE                                             COMMAND                  CREATED             STATUS                       PORTS                           NAMES
b30307ad65be        zabbix/zabbix-web-nginx-mysql:centos-4.0-latest   "docker-entrypoint.sh"   7 days ago          Exited (255) 8 minutes ago   443/tcp, 0.0.0.0:8080->80/tcp   zabbix-web-nginx-mysql
0ad822cd52b7        zabbix/zabbix-server-mysql:centos-4.0-latest      "docker-entrypoint.sh"   7 days ago          Exited (255) 8 minutes ago   0.0.0.0:10051->10051/tcp        zabbix-server-mysql
d01c89a112f7        mysql:5.7                                         "docker-entrypoint.s…"   7 days ago          Exited (255) 8 minutes ago   3306/tcp, 33060/tcp             mysql-server

# 启动、停止、重启容器命令：
docker start container_name/container_id
docker stop container_name/container_id
docker restart container_name/container_id

# 后台启动一个容器后，如果想进入到这个容器，可以使用 attach
docker attach container_name/container_id

# 删除容器 
docker rm container_name/container_id

# 查看容器日志 
docker logs -f container_name/container_id

# 查看容器 IP 地址 
docker inspect container_name/container_id

# 进入容器 
docker exec -it container_name/container_id bash

# 从 Docker 容器与宿主机相互传输文件 
[root@localhost tmp]# docker cp --help

Usage:	docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH|-
	docker cp [OPTIONS] SRC_PATH|- CONTAINER:DEST_PATH

Copy files/folders between a container and the local filesystem

docker cp zabbix_config.sql mysql-server:/tmp
docker cp mysql-server:/tmp/zabbix_config.sql /tmp


# 批量删除所有已经退出的容器 
docker rm -v $(docker ps -aq -f status=exited)


# docker help
[root@centos7lined1 wangao]# docker help

Usage:	docker [OPTIONS] COMMAND

A self-sufficient runtime for containers

Options:
      --config string      Location of client config files (default "/root/.docker")
  -D, --debug              Enable debug mode
  -H, --host list          Daemon socket(s) to connect to
  -l, --log-level string   Set the logging level ("debug"|"info"|"warn"|"error"|"fatal") (default "info")
      --tls                Use TLS; implied by --tlsverify
      --tlscacert string   Trust certs signed only by this CA (default "/root/.docker/ca.pem")
      --tlscert string     Path to TLS certificate file (default "/root/.docker/cert.pem")
      --tlskey string      Path to TLS key file (default "/root/.docker/key.pem")
      --tlsverify          Use TLS and verify the remote
  -v, --version            Print version information and quit

Management Commands:
  config      Manage Docker configs
  container   Manage containers
  image       Manage images
  network     Manage networks
  node        Manage Swarm nodes
  plugin      Manage plugins
  secret      Manage Docker secrets
  service     Manage services
  stack       Manage Docker stacks
  swarm       Manage Swarm
  system      Manage Docker
  trust       Manage trust on Docker images
  volume      Manage volumes

Commands:
  attach      Attach local standard input, output, and error streams to a running container
  build       Build an image from a Dockerfile
  commit      Create a new image from a container's changes
  cp          Copy files/folders between a container and the local filesystem
  create      Create a new container
  diff        Inspect changes to files or directories on a container's filesystem
  events      Get real time events from the server
  exec        Run a command in a running container
  export      Export a container's filesystem as a tar archive
  history     Show the history of an image
  images      List images
  import      Import the contents from a tarball to create a filesystem image
  info        Display system-wide information
  inspect     Return low-level information on Docker objects
  kill        Kill one or more running containers
  load        Load an image from a tar archive or STDIN
  login       Log in to a Docker registry
  logout      Log out from a Docker registry
  logs        Fetch the logs of a container
  pause       Pause all processes within one or more containers
  port        List port mappings or a specific mapping for the container
  ps          List containers
  pull        Pull an image or a repository from a registry
  push        Push an image or a repository to a registry
  rename      Rename a container
  restart     Restart one or more containers
  rm          Remove one or more containers
  rmi         Remove one or more images
  run         Run a command in a new container
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  search      Search the Docker Hub for images
  start       Start one or more stopped containers
  stats       Display a live stream of container(s) resource usage statistics
  stop        Stop one or more running containers
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
  top         Display the running processes of a container
  unpause     Unpause all processes within one or more containers
  update      Update configuration of one or more containers
  version     Show the Docker version information
  wait        Block until one or more containers stop, then print their exit codes

````

- zabbix agent安装
    - LVS
        1. 直接安装，从软件源安装二进制包：
        ```
        # 安装客户端
        $ sudo yum install zabbix-agent
        ```

        当客户端找不到的时候，需要先进行添加安装的rpm
        ```
        rpm -ivh http://repo.zabbix.com/zabbix/4.2/rhel/7/x86_64/zabbix-agent-4.2.0-1.el7.x86_64.rpm
        ```
        安装完成之后，进行启动：
        ```
        $ sudo service zabbix-agent start

        #或者执行以下命令：
        $ sudo systemctl start zabbix-agent

        # 停止、重启、查看状态，则需要执行以下命令：

        $ sudo service zabbix-agent stop
        $ sudo service zabbix-agent restart
        $ sudo service zabbix-agent status
        ```

        如果你的zabbix-agent和zabbix-server安装在同一台机器上，就不需要做额外配置，否则就需要进行服务器日志等的配置，配置参考win端的zabbix agent配置。

        但是注意，如果不做配置的话，需要将你zabbix所在的linux服务端名称配置为默认的Zabbix Server。

        2. 编译安装(暂时不写)

    - WinServer

        1. 解压包安装
    
        先从[下载地址](https://www.zabbix.com/download_agents#tab:42)下载windows的解压包，我们这里设定解压路径为*C:\ZabbixAgent*。下载图片中标记的客户端。

        然后我们需要将其配置为系统服务并进行启动：

        ```shell

        // cmd需要以管理员身份运行
        // 安装
        C:\ZabbixAgent\bin\zabbix_agentd.exe -c c:\ZabbixAgent\conf\zabbix_agentd.win.conf -i
        // 启动
        C:\ZabbixAgent\bin\zabbix_agentd.exe -c C:\ZabbixAgent\conf\zabbix_agentd.win.conf -s

        // 停止
         C:\ZabbixAgent\bin\zabbix_agentd.exe -c C:\ZabbixAgent\conf\zabbix_agentd.win.conf -x
        // 卸载
        C:\ZabbixAgent\bin\zabbix_agentd.exe -c C:\ZabbixAgent\conf\zabbix_agentd.win.conf -d   

        // 命令解析
        -c ：指定配置文件所有位置
        -i ：安装客户端
        -s ：启动客户端
        -x ：停止客户端
        -d ：卸载客户端

        ```

        2. 安装包安装(暂时不写)

    操作完成之后需要从任务管理器里面查看是否存在ZabbixAgent服务，如果存在说明启动成功。

- 数据监控
    
    - 针对win server
    
    如果只是针对WinServer进行监控，不需要对Zabbix agent进行脚本的配置更改。
    
        1. Zabbix agent端进行配置
    
        首先找到我们解压的ZabbixAgent所在路径，为**C:\ZabbixAgent\**，然后找到conf文件夹下的zabbix_agentd.win.conf文件，进行修改：

        ```
        #日志文件存储位置
        LogFile=c:\ZabbixAgent\zabbix_agentd.log
        #zabbix主控端ip地址，也就是Server的地址
        Server=192.168.1.132
        #本机名，也可以在cmd下使用hostname命令获得，可以进行自定义
        Hostname=WinServer
        #zabbix主控端ip地址
        ServerActive=192.168.1.132
        ```
        然后安装再重启即可。

        2. 导入WinServer监控模板

        由于Zabbix中自带WinServer监控模板，因此无需导入。
        
        3. 创建WinServer主机

        步骤如下：
        
        ![zabbix-创建主机.PNG](zabbix-创建主机.PNG)
        
        ![zabbix-创建主机.PNG](zabbix-winServer配置名称.PNG)
        
        ![zabbix-创建主机.PNG](zabbix-winserver-选择模板.PNG)
        
        ![zabbix-创建主机.PNG](zabbix-WinServer-添加模板.PNG)
        
        4. 查看上报数据
        这样通过监测->图形，选择自己要监控的主机和监控数据类型，即可看到监控数据上报。如下图所示：
        
        ![zabbix-创建主机.PNG](zabbix-winserver监控数据.PNG)

        5. 参数指标详解

    - 针对linux server
    
    如果只是针对WinServer进行监控，不需要对Zabbix agent进行脚本的配置更改。
    
        1. Zabbix agent端进行配置
    
            - 安装Zabbix Agent
    
            注意：需要在VPS管理页面开放10051端口的访问。
    
        2. 导入Linux监控模板
    
        3. 创建Linux主机
    
        4. 查看上报的数据
    
        5. 参数指标详解
    
        - 补充：针对linux server遇到：failed to accept an incoming connection: connection from “one-ip” rejected, allowed hosts:“two-ip”的问题，需要：

            针对配置文件中的Server字段，多配置几个ip，示例如下：Server=127.0.0.1,192.168.100.10,172.73.1.3

    - 针对nginx监控
    
    针对Nginx的监控，需要开启Nginx监控功能，然后对Zabbix agent端添加脚本并修改配置文件，然后进行导入模板的操作。Nginx部分接
    [上一篇文章](https://www.zybuluo.com/lsyAndroid/note/1466869)
        
        1. 配置文件开启监控功能
        
        切换到nginx安装路径的conf文件夹下，例如/usr/local/nginx/conf目录，然后进行操作：
        
        ```
        # 创建配置文件
        $ sudo touch nginx-status.conf
        $ vim nginx-status.conf
        server {
            listen 8090; # 内网访问端口
            server_name abc.test.com;
            location /nginx_stat {
                # 开启状态可访问
                stub_status on;
                access_log off;
            }
        }

        # 在nginx.conf文件中引用该文件
        $ sudo vim nginx.conf
        ......
        
        include nginx-status.conf;
        
        # 检查配置文件没有问题
        $ sudo nginx -t
        nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
        nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful

        # 重启nignx
        $ sudo nginx -s reload
 
        # 检查之后，可以通过curl来访问测试
        $  curl http://127.0.0.1:8090/nginx_stat
        Active connections: 2
        server accepts handled requests
            55342 55342 74613
        Reading: 0 Writing: 1 Waiting: 1

        ```
        参数指标解释：

        Active connections –当前活跃的连接数量  
        
        server accepts handled requests — 总共处理了756072922个连接 , 成功创建 756072922次握手, 总共处理了1136799890个请求
        
        reading — 读取客户端的连接数.
        
        writing — 响应数据到客户端的数量
        
        waiting — 开启 keep-alive 的情况下,这个值等于 active – (reading+writing), 意思就是 Nginx 已经处理完正在等候下一次请求指令的驻留连接.

        这样就配置完成了Nginx的监控状态。
        
        2. 配置Nginx监控执行脚本
        
        在Nginx安装目录下，添加scripts文件夹，存放我们的监控脚本。
        
        ```
        $ sudo vim scripts/nginx-performance.sh
        #!/bin/bash
        HOST="139.159.199.40"     #这里的地址要写自己的
        PORT="8090"               #端口号和配置文件中的nginx_stat

        function ping {
            num=$(/sbin/pidof nginx |wc -l)
        }
        function active {
            num=$(/usr/bin/curl -s "http://$HOST:$PORT/nginx_stat" |grep 'Active' |awk '{print $NF}')
        }
        function reading {
            num=$(/usr/bin/curl -s "http://$HOST:$PORT/nginx_stat" |grep 'Reading' |awk '{print $2}')
        }
        function writing {
            num=$(/usr/bin/curl -s "http://$HOST:$PORT/nginx_stat" |grep 'Writing' |awk '{print $4}')
        }
        function waiting {
            num=$(/usr/bin/curl -s "http://$HOST:$PORT/nginx_stat" |grep 'Waiting' |awk '{print $6}')
        }
        function accepts {
            num=$(/usr/bin/curl -s "http://$HOST:$PORT/nginx_stat" |awk NR==3 |awk '{print $1}')
        }
        function handled {
            num=$(/usr/bin/curl -s "http://$HOST:$PORT/nginx_stat" |awk NR==3 |awk '{print $2}')
        }
        function requests {
            num=$(/usr/bin/curl -s "http://$HOST:$PORT/nginx_stat" |awk NR==3 |awk '{print $3}')
        }

        $1
        echo ${num:-0}

        ```
        脚本很简单，分表对应上节讲的各项指标。
        
        3. Zabbix agent进行配置
        
        配置文件配置如下：
        
        ```
        $ sudo vim /etc/zabbix/zabbix_agentd.conf
        # 守护进程文件
        PidFile=/var/run/zabbix/zabbix_agentd.pid
        # 日志文件
        LogFile=/var/log/zabbix/zabbix_agentd.log
        # 日志文件初始大小
        LogFileSize=0
        # zabbix server所在ip地址
        Server=127.0.0.1 # 更换为你自己的地址
        # 同上配置即可
        ServerActive=127.0.0.1
        # 主机名称，和你在zabbix中配置的主机名称一致，否则无法进行监控数据的采集
        Hostname=HT-LVS-Nginx-0001
        # 配置文件
        Include=/etc/zabbix/zabbix_agentd.d/*.conf
        # 用户自定义参数，传入脚本用
        UnsafeUserParameters=1
        # 脚本配置，*,*以前是你在zabbix服务端进行配置的参数名称，*， *后是你要执行的脚本以及传入的参数
        UserParameter=nginx.status[*],/usr/local/nginx/scripts/nginx-performance.sh $1

        ```
        配置完成后，需要重启zabbix-agent进程
        
        ```
        $ sudo systemctl restart zabbix_agentd

        # 或者执行下面的代码
        $ sudo /usr/sbin/zabbix_agentd -c /etc/zabbix/zabbix_agentd.conf

        ```
        这样就可以进行对Nginx的监控数据采集了。

        此时可以在Server端执行:
        
        ```
        # 在zabbix服务端（server）进行测试。
        $ zabbix_get -s 10.253.17.20 -p 10050 -k "nginx[reading]"
        0
        ```
        
        4. 导入Nginx监控模板
        
        [下载模板](zabbix_monitor_nginx_template_ttlsa_com)，然后进行导入。
        
        首先，登录zabbix界面，依次点击：配置（configuration）---模板（template）---导入（import）
        
        ![](zabbix_nginx模板导入.png)
        
        其次配置主机，注意这里配置的主机名称为：**HT-LVS-Nginx-0001**，然后是添加模板
        
        ![](zabbix_nginx配置主机.png)
        
        最后，查看nginx监控的最新数据：监控中---图形---选择相应的监控类型。
        
        ![](zabbix_nginx_监控1.png)
        
        ![](zabbix_nginx-参数.png)
        
        5. 参数指标详解

        | 名称 | 描述 | 指标类型 |
        |------|------|---------|
        |Accepts（接受）| NGINX 所接受的客户端连接数| 资源: 功能|
        |Handled（已处理）| 成功的客户端连接数| 资源: 功能|
        |Active（活跃）| 当前活跃的客户端连接数 | 资源: 功能 |
        |Dropped（已丢弃，计算得出）| 丢弃的连接数（接受 - 已处理）| 工作：错误*|
        | Requests（请求数） | 客户端请求数 |  工作：吞吐量 |

    - 针对SQL Server监控
        
        1. 导入SQLServer配置模板
        
        SQLServer模板很多，推荐使用[这个模板](https://share.zabbix.com/template-windows-sql-server)，下载后解压先把Template文件夹下面的**Template - Windows LLD MSSQL - 32.xml**的模板导入到zabbix中即可。
        注意模板是葡萄牙语的，如果有能力请更换为英语或者简体中文。

        汉化文章在此：https://blog.51cto.com/ygqygq2/2073139
        
        2. 配置SQLServer监控执行脚本
        
        在SQLServer所在机器安装zabbix agent，这点操作同WinServer，安装完成并配置服务进行启动。
        然后再WinServer中配置监控执行脚本。在ZabbixAgent安装路径下，创建**scripts**文件夹，将上面下载的**discovery.mssql.server.ps1**拷贝到该文件夹下面。最后需要把**discovery.mssql.server.conf**这个配置文件拷贝到ZabbixAgent安装目录下的conf文件夹中，并且修改同级目录下的
        
        **zabbix_agentd.win.conf**文件，如下:
        
        ```
        LogFile=c:\ZabbixAgent\zabbix_agentd.log

        Server=127.0.0.1  # 配置你自己的ZabbixServer所在地址

        ServerActive=127.0.0.1 # 配置你自己的ZabbixServer所在地址

        Hostname=HT-WinServer2016-db-0002-SQLServer-25

        Timeout=30

        Include=C:\ZabbixAgent\conf\discovery.mssql.server.conf # 引用这个配置文件
        
        UnsafeUserParameters=1
        ```
        discovery.mssql.server.conf文件内容如下：
        ```
        # 要注意修改discovery.mssql.server.ps1脚本所在路径
        UserParameter=discovery.mssql.databases,powershell.exe -noprofile -executionpolicy bypass -File C:\ZabbixAgent\scripts\discovery.mssql.server.ps1 JSONDB
        UserParameter=discovery.mssql.jobs,powershell.exe -noprofile -executionpolicy bypass -File C:\ZabbixAgent\scripts\discovery.mssql.server.ps1 JSONJOB
        UserParameter=discovery.mssql.data[*],powershell.exe -noprofile -executionpolicy bypass -File C:\ZabbixAgent\scripts\discovery.mssql.server.ps1 $1 "$2"

        ```
        
        3. 创建SQLServer主机
        
        创建SQLServer所在主机，并且配置我们刚刚导入的监控模板。
        
        4. 查看上报的数据
        
        上报数据示例如下图：
        
        ![](SQLServer监控数据.png)
        
        5. 参数指标详解

**注意：**
1. agent部分所在服务器要开放10050和10051端口
2. agent部分所在服务器的配置文件内的**Hostname**，同创建主机时**主机名称**一致，如果不一致，无法看到主机的监控数据


- 配置报警
    
    - 邮件
    
    实现邮件报警有两种思路，一个是通过平台的报警媒介配置实现，另一个是通过通过配置可执行脚本来发送邮件，这里演示前一种，因为后一种在Linux端配置的时候，**遇到了邮件发送超时的问题**，无法配置成功。
    
        - 配置报警媒介类型
    
        首先通过下面路径前往：管理->报警媒介类型->创建媒体类型，到以下页面:
       
        ![zabbix配置邮件报警1](zabbix配置邮件报警1.PNG)
        
        ![zabbix配置邮件报警2](zabbix配置邮件报警2.PNG)
       
        **注意：**
        1. 在创建报警媒介类型时，建议使用企业邮箱进行创建，不建议使用普通的QQ邮箱或者163邮箱进行，个人原因在配置的时候这两个邮箱均不可用，浪费了大量时间，特此订正。我自己用的是腾讯企业邮。
        2. 配置邮件报警后，回到前一个页面，可以点击测试，测试邮件是否能够发送成功。
        
        然后再切换到选项页面，如下图设置：
       
        ![zabbix配置邮件报警3](zabbix配置邮件报警3.PNG)

        设置完成后点击添加/更新即可。这样配置完成了媒介类型。
        
	- 将媒介类型绑定用户
    
        切换到用户页面，可以先创建用户，也可以在原有用户上进行添加。根据下图操作，切换到Admin用户所在页面：

        ![zabbix配置邮件报警4](zabbix配置邮件报警4.PNG)
        
        ![zabbix配置邮件报警5](zabbix配置邮件报警5.PNG)

        切换到报警媒介页面，点击添加。这里可以添加多个收件人。**添加完成后需要返回点击更新，以免添加无效！**下面为添加页面：

        ![zabbix配置邮件报警6](zabbix配置邮件报警6.PNG)

    - 添加报警动作
    
        切换到配置->动作->创建动作页面

        ![zabbix配置邮件报警7](zabbix配置邮件报警7.PNG)

        ![zabbix配置邮件报警8](zabbix配置邮件报警8.PNG)

        添加名称，然后添加触发器：

        ![zabbix配置邮件报警9](zabbix配置邮件报警9.PNG)
        
        通过选择群组和模板，选择要报警的类别，在这里选择**Zabbix agent on Template OS Linux is unreachable for 5 minutes**这一项，不要忘了点击“新的出发条件”一栏，下面的**添加**按钮。将监控项添加到里面。
        - 配置邮件内容-操作、恢复操作、更新操作
        以操作为例，进行添加，如下图点击**新的**超链接：
        
        ![zabbix配置邮件报警10](zabbix配置邮件报警10.PNG)
        
        到达：
       
        ![zabbix配置邮件报警11](zabbix配置邮件报警11.PNG)
       
        在上面页面中，添加**发送到用户群组**和**发送到用户**，这里发送到Admin用户下配置的邮箱。恢复操作和更新操作与上面雷同，不再说明。这里可以编辑默认标题和消息内容，更改邮件发送的内容格式。
        - 测试
        前提是你要测试的VPS已经在系统中添加并且配置了和前面页面选择模板时选择的相同的模板。这样在你的VPS中关闭ZabbixAgent系统服务，5分钟左右就会收到邮件，类似如下：
        
        ![zabbix配置邮件报警12](zabbix配置邮件报警12.PNG)
        
        至此，配置完成。
    
    - 企业微信

    - 钉钉

## grafana抓取zabbix数据

注意：这里操作均在root用户下操作。

- 安装 

    - 安装Grafana

    ```
    yum install -y initscripts fontconfig    
    rpm -ivh ./package/grafana-4.0.2-1481203731.x86_64.rpm   
    yum install -y fontconfig    
    yum install -y freetype*    
    yum install -y urw-fonts    
    #显示安装的文件    
    rpm -qc grafana    
    #二进制文件 /usr/sbin/grafana-server    
    #服务管理脚本 /etc/init.d/grafana-server    
    #安装默认文件 /etc/sysconfig/grafana-server    
    #配置文件 /etc/grafana/grafana.ini
    ```
    这样安装完成后就可以启动了，启动的命令如下：
    ```
    systemctl enable grafana-server

    systemctl start grafana-server   

    这样grafana在开机时会自动启动
    ```
    - 安装插件
    
        - zabbix插件
    
        ```
        #下载模板等信息    
        cd ~    
        git clone https://github.com/hqh546020152/grafana.git    
        cd grafana
        #获取可用插件列表    
        grafana-cli plugins list-remote
        #安装Zabbix插件    
        grafana-cli plugins install alexanderzobnin-zabbix-app
        #安装硬盘监控插件,用于danyi.json模板   
        # grafana-cli plugins install grafana-piechart-panel    
        # 重启服务生效
        systemctl restart grafana-server
        ```
    
    - 安装图库（饼图、折线图、仪表盘等）

- Grafana和Zabbix联通

* 在浏览器地址栏输入 http://IP:3000就可以看到Grafana的登陆页面了。输入默认的用户名admin 密码admin登陆
点击左侧的齿轮按钮，弹出选项框中，选择Data Sourses进入Data Source子页面，点击add data source进行添加。

![Grafana添加数据源](Grafana添加数据源.PNG)

* Type下拉框中选择Zabbix

![Grafana添加zabbix数据源](Grafana添加zabbix数据源.PNG)
    
* Name 可以自由发挥~~
* HTTP-->Url 填入http://zabbix-server-ip/zabbix/api_jsonrpc.php 这里填入的是Zabbix API接口 （例如：http://10.0.93.126:8181/api_jsonrpc.php）
* Http-->Access 选择 Server(default) 使用直接访问的方式
* Zabbix API details-->User 填入Admin
* Zabbix API details-->Password 填入 zabbix
* 点Save&Test按钮保存并且测试API配置是否正确。出现：Zabbix API version: 4.2.1 信息，说明配置成功

![Grafana配置Zabbix数据源](Grafana配置Zabbix数据源.PNG)

- 导入模板

以LVS监控为例子来进行讲解，首先我们先导入Linux监控的模板，从[这里](https://grafana.com/dashboards)挑选模板进行下载。由于我们使用zabbix做数据监控，这样我们可以使用[Dashboard Servers Linux](https://grafana.com/dashboards/8677)模板进行，如图所示：

![grafana下载模板](grafana下载模板.PNG)


然后复制ID，转到我们添加模板的页面，如图所示：
![grafana导入模板](grafana导入模板.PNG)

将ID粘贴到Grafana.com Dashboard这个下的输入框中，点击下边的load按钮，即可进入模板数据源绑定页面，如下所示：
![grafana导入模板配置数据源](grafana导入模板配置数据源.PNG)

在数据源绑定部分，我们选择zabbix数据源，然后稍加修改，就可以导入到grafana中了。如下图所示：
![grafana导入模板-配置数据源完成](grafana导入模板-配置数据源完成.PNG)

这样我们需要修改一些地方，才能使模板工作。因为我目前只有一台Linux服务器，所以我们在这个模板中固定下服务器的参数即可。首先去掉预设的参数，操作如下：

![grafana模板修改](grafana模板修改.PNG)

![grafana删除自定义的参数](grafana删除自定义的参数.PNG)

修改完成之后点击Save进行保存。然后我们回到dashboard页面，随便点开一个图表，进入图标的设计页面，如图：

![grafana图表配置进入之前](grafana图表配置进入之前.PNG)

最后，我们对这个图表进行调整，第一步，修改Group和Host数据为下面的内容：

![grafana-修改Group-Host](grafana-修改Group-Host.PNG)

第二步，数据出现后，我们修改图表名称，切换到下一个页面中，在如下图所示

![grafana修改图表名字](grafana修改图表名字.PNG)

这样这个图表就修复完成了。

最后的最后，经过调整，我们会得到这样的图表：

![grafana最终效果](grafana最终效果.PNG)

- 自定义创建图表

- 添加分组信息

针对多台相同类型的服务器进行统一管理

- 数据绑定和处理

- 自带报警（对zabbix无效）

## 参考文章

- [CentOS 7 下 yum 安装 Docker CE](https://qizhanming.com/blog/2019/01/25/how-to-install-docker-ce-on-centos-7)

- [使用 Docker 安装 Zabbix 实践](https://wsgzao.github.io/post/zabbix-docker/)

- [Docker 从入门到实践](https://wsgzao.github.io/post/docker/)

- [Zabbix监控nginx性能](http://www.ttlsa.com/zabbix/zabbix-monitor-nginx-performance/)
