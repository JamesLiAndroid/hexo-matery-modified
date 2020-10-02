---
title: Harbor安装以及docker registry的迁移
top: false
cover: false
toc: true
mathjax: true
date: 2020-09-22 09:34:06
password:
summary:
tags:
categories:
---

## 环境信息

* 操作系统：CentOS 7.6 1804
* Linux 内核版本：3.10.0-862.el7.x86_64
* Docker版本：19.03.12
* docker-compose版本：1.24.1
* 安装地址：10.0.93.105

参考[Linux系统初始化以及部署]()文档进行配置。

## 安装准备

选择[harbor-offline-installer-v1.10.1](https://github.com/goharbor/harbor/releases/download/v1.10.3/harbor-offline-installer-v1.10.1.tgz)离线安装包，下载完成后，解压到/opt/harbor目录下。

## Harbor搭建以及配置

```
$ cd /opt/harbor/

$ cp harbor.yml.tmpl harbor.yml

$ vim harbor.yml
// 主要操作在于注释掉https相关设置以及添加hostname
# Configuration file of Harbor

# The IP address or hostname to access admin UI and registry service.
# DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
# hostname重新改写
hostname: 10.0.93.105

# http related config
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 80

# https related config
# 注释掉https相关设置
#https:
  # https port for harbor, default is 443
#  port: 443
  # The path of cert and key files for nginx
#  certificate: /your/certificate/path
#  private_key: /your/private/key/path

# Uncomment external_url if you want to enable external proxy
# And when it enabled the hostname will no longer used
# external_url: https://reg.mydomain.com:8433

# The initial password of Harbor admin
# It only works in first time to install harbor
# Remember Change the admin password from UI after launching Harbor.
harbor_admin_password: Harbor12345

# Harbor DB configuration
database:
  # The password for the root user of Harbor DB. Change this before any production use.
  password: root123
  # The maximum number of connections in the idle connection pool. If it <=0, no idle connections are retained.
  max_idle_conns: 50
  # The maximum number of open connections to the database. If it <= 0, then there is no limit on the number of open connections.
  # Note: the default number of connections is 100 for postgres.
  max_open_conns: 100

# The default data volume
data_volume: /data

# Harbor Storage settings by default is using /data dir on local filesystem
# Uncomment storage_service setting If you want to using external storage
# storage_service:
#   # ca_bundle is the path to the custom root ca certificate, which will be injected into the truststore
#   # of registry's and chart repository's containers.  This is usually needed when the user hosts a internal storage with self signed certificate.
#   ca_bundle:

#   # storage backend, default is filesystem, options include filesystem, azure, gcs, s3, swift and oss
#   # for more info about this configuration please refer https://docs.docker.com/registry/configuration/
#   filesystem:
#     maxthreads: 100
#   # set disable to true when you want to disable registry redirect
#   redirect:
#     disabled: false

# Clair configuration
clair:
  # The interval of clair updaters, the unit is hour, set to 0 to disable the updaters.
  updaters_interval: 12

jobservice:
  # Maximum number of job workers in job service
  max_job_workers: 10

notification:
  # Maximum retry count for webhook job
  webhook_job_max_retry: 10

chart:
  # Change the value of absolute_url to enabled can enable absolute url in chart
  absolute_url: disabled

# Log configurations
log:
  # options are debug, info, warning, error, fatal
  level: info
  # configs for logs in local storage
  local:
    # Log files are rotated log_rotate_count times before being removed. If count is 0, old versions are removed rather than rotated.
    rotate_count: 50
    # Log files are rotated only if they grow bigger than log_rotate_size bytes. If size is followed by k, the size is assumed to be in kilobytes.
    # If the M is used, the size is in megabytes, and if G is used, the size is in gigabytes. So size 100, size 100k, size 100M and size 100G
    # are all valid.
    rotate_size: 200M
    # The directory on your host that store log
    location: /var/log/harbor

  # Uncomment following lines to enable external syslog endpoint.
  # external_endpoint:
  #   # protocol used to transmit log to external endpoint, options is tcp or udp
  #   protocol: tcp
  #   # The host of external endpoint
  #   host: localhost
  #   # Port of external endpoint
  #   port: 5140

#This attribute is for migrator to detect the version of the .cfg file, DO NOT MODIFY!
_version: 1.10.0

# Uncomment external_database if using external database.
# external_database:
#   harbor:
#     host: harbor_db_host
#     port: harbor_db_port
#     db_name: harbor_db_name
#     username: harbor_db_username
#     password: harbor_db_password
#     ssl_mode: disable
#     max_idle_conns: 2
#     max_open_conns: 0
#   clair:
#     host: clair_db_host
#     port: clair_db_port
#     db_name: clair_db_name
#     username: clair_db_username
#     password: clair_db_password
#     ssl_mode: disable
#   notary_signer:
#     host: notary_signer_db_host
#     port: notary_signer_db_port
#     db_name: notary_signer_db_name
#     username: notary_signer_db_username
#     password: notary_signer_db_password
#     ssl_mode: disable
#   notary_server:
#     host: notary_server_db_host
#     port: notary_server_db_port
#     db_name: notary_server_db_name
#     username: notary_server_db_username
#     password: notary_server_db_password
#     ssl_mode: disable

# Uncomment external_redis if using external Redis server
# external_redis:
#   host: redis
#   port: 6379
#   password:
#   # db_index 0 is for core, it's unchangeable
#   registry_db_index: 1
#   jobservice_db_index: 2
#   chartmuseum_db_index: 3
#   clair_db_index: 4

# Uncomment uaa for trusting the certificate of uaa instance that is hosted via self-signed cert.
# uaa:
#   ca_file: /path/to/ca

# Global proxy
# Config http proxy for components, e.g. http://my.proxy.com:3128
# Components doesn't need to connect to each others via http proxy.
# Remove component from `components` array if want disable proxy
# for it. If you want use proxy for replication, MUST enable proxy
# for core and jobservice, and set `http_proxy` and `https_proxy`.
# Add domain to the `no_proxy` field, when you want disable proxy
# for some special registry.
proxy:
  http_proxy:
  https_proxy:
  # no_proxy endpoints will appended to 127.0.0.1,localhost,.local,.internal,log,db,redis,nginx,core,portal,postgresql,jobservice,registry,registryctl,clair,chartmuseum,notary-server
  no_proxy:
  components:
    - core
    - jobservice
    - clair

// :wq保存退出

```

执行下面的命令，开始安装：

```
$ sudo ./prepare.sh

$ sudo ./install.sh

```

在/opt/harbor目录下，会生成docker-compose.yml文件，如下：

```
$ vim docker-compose.yml

version: '2.3'
services:
  log:
    image: goharbor/harbor-log:v1.10.1
    container_name: harbor-log
    restart: always
    dns_search: .
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - SETGID
      - SETUID
    volumes:
      - /var/log/harbor/:/var/log/docker/:z
      - ./common/config/log/logrotate.conf:/etc/logrotate.d/logrotate.conf:z
      - ./common/config/log/rsyslog_docker.conf:/etc/rsyslog.d/rsyslog_docker.conf:z
    ports:
      - 127.0.0.1:1514:10514
    networks:
      - harbor
  registry:
    image: goharbor/registry-photon:v2.7.1-patch-2819-2553-v1.10.1
    container_name: registry
    restart: always
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
    volumes:
      - /data/registry:/storage:z
      - ./common/config/registry/:/etc/registry/:z
      - type: bind
        source: /data/secret/registry/root.crt
        target: /etc/registry/root.crt
    networks:
      - harbor
    dns_search: .
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "registry"
  registryctl:
    image: goharbor/harbor-registryctl:v1.10.1
    container_name: registryctl
    env_file:
      - ./common/config/registryctl/env
    restart: always
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
    volumes:
      - /data/registry:/storage:z
      - ./common/config/registry/:/etc/registry/:z
      - type: bind
        source: ./common/config/registryctl/config.yml
        target: /etc/registryctl/config.yml
    networks:
      - harbor
    dns_search: .
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "registryctl"
  postgresql:
    image: goharbor/harbor-db:v1.10.1
    container_name: harbor-db
    restart: always
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - SETGID
      - SETUID
    volumes:
      - /data/database:/var/lib/postgresql/data:z
    networks:
      harbor:
    dns_search: .
    env_file:
      - ./common/config/db/env
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "postgresql"
  core:
    image: goharbor/harbor-core:v1.10.1
    container_name: harbor-core
    env_file:
      - ./common/config/core/env
    restart: always
    cap_drop:
      - ALL
    cap_add:
      - SETGID
      - SETUID
    volumes:
      - /data/ca_download/:/etc/core/ca/:z
      - /data/psc/:/etc/core/token/:z
      - /data/:/data/:z
      - ./common/config/core/certificates/:/etc/core/certificates/:z
      - type: bind
        source: ./common/config/core/app.conf
        target: /etc/core/app.conf
      - type: bind
        source: /data/secret/core/private_key.pem
        target: /etc/core/private_key.pem
      - type: bind
        source: /data/secret/keys/secretkey
        target: /etc/core/key
    networks:
      harbor:
    dns_search: .
    depends_on:
      - log
      - registry
      - redis
      - postgresql
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "core"
  portal:
    image: goharbor/harbor-portal:v1.10.1
    container_name: harbor-portal
    restart: always
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
      - NET_BIND_SERVICE
    networks:
      - harbor
    dns_search: .
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "portal"

  jobservice:
    image: goharbor/harbor-jobservice:v1.10.1
    container_name: harbor-jobservice
    env_file:
      - ./common/config/jobservice/env
    restart: always
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
    volumes:
      - /data/job_logs:/var/log/jobs:z
      - type: bind
        source: ./common/config/jobservice/config.yml
        target: /etc/jobservice/config.yml
    networks:
      - harbor
    dns_search: .
    depends_on:
      - core
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "jobservice"
  redis:
    image: goharbor/redis-photon:v1.10.1
    container_name: redis
    restart: always
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
    volumes:
      - /data/redis:/var/lib/redis
    networks:
      harbor:
    dns_search: .
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "redis"
  proxy:
    image: goharbor/nginx-photon:v1.10.1
    container_name: nginx
    restart: always
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
      - NET_BIND_SERVICE
    volumes:
      - ./common/config/nginx:/etc/nginx:z
    networks:
      - harbor
    dns_search: .
    ports:
      - 80:8080
    depends_on:
      - registry
      - core
      - portal
      - log
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "proxy"
networks:
  harbor:
    external: false


``` 

配置完成后，可以进行Harbor的启动，如下：

```
$ docker-compose up -d

```

最后直接访问*http://10.0.93.105/*，用户名/密码为：admin/Harbor12345。

这样Harbor搭建完成！

## docker registry迁移到Harbor

### 介绍

docker registry 地址：192.168.163.202:5000，从这个地址迁移到10.0.93.105上的Harbor中。

下面的操作在10.0.93.107上完成，下面开始设置。

### 迁移机器配置

设置docker所属的daemon.json文件，保证可以同时对两个docker镜像仓库进行拉取和推送，设置如下：

```
$ sudo vim /etc/docker/daemon.json
// 添加insecure-regsitry
{
    "registry-mirrors": [
        "https://docker.mirrors.ustc.edu.cn",
        "https://registry.cn-hangzhou.aliyuncs.com",
        "https://registry.docker-cn.com"
    ],
    "insecure-registries": [
        "192.168.163.202:5000",
        "10.0.93.105"
    ],
    "live-restore": true,
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "50m",
        "max-file": "3"
    }
}

// :wq保存退出

$ sudo systemctl daemon-reload

$ sudo systemctl restart docker

```

docker设置完成后，需要登录到harbor所在的镜像仓库中，操作如下：

```
$ docker login -u admin -p Harbor12345 10.0.93.105

```

登录后测试一下，能否向Harbor中推送镜像。

```
$ docker images
REPOSITORY                       TAG                 IMAGE ID            CREATED             SIZE
prom/node-exporter               v1.0.1              0e0218889c33        3 months ago        26.4MB

$ docker tag 0e0218889c33 10.0.93.105/library/prom/node-exporter:v1.0.11

$ docker push 10.0.93.105/library/prom/node-exporter:v1.0.11
The push refers to repository [10.0.93.105/library/prom/node-exporter]
53fe7fa9c07c: Pushed
bb208cc3e926: Pushed
bef00f7ac5a9: Pushed
v1.0.11: digest: sha256:f5b6df287cc3a87e8feb00e3dbbfc630eb411ca6dc3f61abfefe623022fa6927 size: 949

```

Harbor默认是使用的library仓库，tag操作的时候注意下路径。推送完成，说明当前Harbor可用。

### 迁移脚本编写

先在Harbor中创建新项目，如图：

![](创建Harbor项目.png)

然后在107所在的机器上编写迁移脚本如下：

```
$ vim to_harbor.sh

#!/bin/bash
# docker registry 迁移脚本
images=`curl -s 192.168.163.202:5000/v2/_catalog | jq .repositories[] | tr -d '"'`

#测试部分
#list="base_backend base_frontend"
#for image in $list;
for image in $images;
do
    tags=`curl -s 192.168.163.202:5000/v2/$image/tags/list | jq .tags[] | tr -d '"'`
    for tag in $tags;
    do
        docker pull 192.168.163.202:5000/$image:$tag
        docker tag 192.168.163.202:5000/$image:$tag 10.0.93.105/test/$image:$tag
        docker push 10.0.93.105/test/$image:$tag
        if [ $? -eq 0 ];then
            echo "###############################$image:$tag pull complete!###############################"
        else
            echo "$image:$tag pull failure!"
        fi
    done
done

echo "最后删除所有无用的镜像信息"
docker rmi -f $(docker images | grep -v "prom" |  awk '{print $3}')

// :wq保存并退出

```

我们现存的镜像比较多，迁移非常耗时，而且迁移涉及大量的磁盘io，会对当前服务器造成影响。因此使用定时任务执行，如下：

```
$ sudo touch /tmp/tmp.log

$ sudo chown centos:centos /tmp/tmp.log

$ crontab -e
// 输入以下内容，配置九月23日晚上九点以后执行
0 0 21 23 9 ? sh +x to_harbor.sh > /tmp/tmp.log 2>&1 

// :wq保存退出

```

### 测试和执行

测试一下脚本是否能正常执行，然后在脚本中修改以下内容：

```
list="base_backend base_frontend"
for image in $list;
#for image in $images;

```

执行脚本测试拉取基础镜像的内容，如下：

```
$ sh +x to_harbor.sh

```

查看Harbor仓库中已经迁移了基础镜像仓库了。

最后恢复下文件，等待定时任务启动。

由于第二天休息，第三天回公司看结果，发现在昨天早上7点多完成的迁移，这样在Harbor中已经看到所有的镜像了。

## CI/CD流程中的变更

### 主机docker服务配置调整

这里使用spug运维平台批量执行脚本，脚本信息如下：

```
echo '' | sudo -S sed -i 's/192.168.163.202:5000/10.0.93.105\/test\//g' /etc/docker/daemon.json

echo '' | sudo systemctl daemon-reload

echo '' | sudo systemctl restart docker

```

在echo后的引号内，写入密码信息，批量修改完成后，需要对docker镜像进行重启即可。

### jenkins调整

对jenkins的调整主要涉及两个方面：

1. 构建任务中镜像名称中的url的替换
2. docker registry的地址重新配置为harbor的地址，并且需要对harbor添加用户名密码信息。 

这样需要批量修改构建任务中对应的config.xml配置文件，使用sed命令进行替换：

```

#!/bin/bash
cd jobs/

list=`find ./*/ -maxdepth 1 -type f -name config.xml`

for configFile in $list;
do
    sed -i 's/192.168.163.202:5000/10.0.93.105\/test/g' $configFile
    echo $configFile 'changed'
    echo '=================================================='
done

```

但是修改完成后，需要对每个构建任务添加认证信息，也就是添加Harbor的用户名密码作为登录信息，相当于在jenkins中进行了*docker login*操作。

### Dockerfile配置修改

对Dockerfile的修改主要涉及到基础镜像的地址需要进行变更，但是由于jenkins所在的机器中，本地docker镜像有缓存，应该是jenkins所在机器上已经留存了所有的基础镜像，目前临时不需要进行修改。

## 问题

## 总结

## 参考文章

* 迁移参考：https://soulchild.cn/691.html
* https://www.cnblogs.com/iXiAo9/p/13665547.html