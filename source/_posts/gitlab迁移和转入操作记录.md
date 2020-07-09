---
title: gitlab迁移和转入操作记录
top: false
cover: false
toc: true
mathjax: true
date: 2020-06-09 16:21:25
password:
summary:
tags:
categories:
---

# gitlab迁移和转入的操作指南

## 前言

目前涉及到之前平台的gitlab需要进行停机，转出所使用的的机器，引发了gitlab的迁移操作，最终决定迁移到云平台这边的gitlab中。

下面介绍下基本环境

* 待迁入机器

ubuntu 16.04，gitlab 10.x版本，使用普通安装方式，完整安装在操作系统中

* 目标迁入机器

centos 7.6，gitlab 12.2.x版本，使用docker进行安装

## 执行过程

### 1. 导出用户

根据配置文件，定位相关信息

```
# cat /var/opt/gitlab/gitlab-rails/etc/database.yml

// 找到gitlab-psql用户信息
 

// 查看Gitlab对应的系统用户，确定是否存在gitlab-psql用户
# cat /etc/passwd | grep gitlab

// 根据信息登陆数据库
// 切换用户
# su - gitlab-psql

// 登陆数据库（-h指定host，-d指定数据库）
-sh-4.2$ psql -h /var/opt/gitlab/postgresql -d gitlabhq_production

// 查看帮助信息
gitlabhq_production=# \h

// 查看数据库
gitlabhq_production=# \l

// 查看库中的表（执行命令后，按回车键显示更多表信息）
gitlabhq_production=# \dt

// 通过筛查，可在库中找到users表，相关用户信息都记录在表中！
// 查看users表结构
gitlabhq_production=# \d users

// 查看表信息
gitlabhq_production=# SELECT * FROM users;

// 查看users表中的name字段
gitlabhq_production=# SELECT name FROM users;

// 登出数据库
gitlabhq_production=# \q

// 确定表表users中的 username , email , state , name 字段是需要提取的信息，进行导出操作
-sh-4.2$  echo 'select username,email,state,name from users;' | psql -h /var/opt/gitlab/postgresql -d gitlabhq_production > userinfo.txt

// 退出用户，回到同一个目录下，查看该文件
-sh-4.2$ exit

$ ls 
userinfo.txt

```

将导出的信息重新整理，整理后的userinfo.txt如下：

```
zhongyanyan123 zhongyanyan     zhongyanyan@hoteamsoft.com     active  zhongyanyan
zongquanxiang123 zongquanxiang   zongquanxiang@hoteamsoft.com   active  zongquanxiang
yangjiajie123 yangjiajie      yangjiajie@hoteamsoft.com      active  yangjiajie
sujingfang123 sujingfang      sujingfang@hoteamsoft.com      active  sujingfang
yangshuai123 yangshuai       yangshuai@hoteamsoft.com       active  yangshuai
wangchao123 wangchao        wangchao@hoteamsoft.com        active  wangchao
zhanglianjiang123 zhanglianjiang  zhanglianjiang@hoteamsoft.com  active  zhanglianjiang
liulily123 liulily         liulili@hoteamsoft.com         active  liulily

```

### 2. 导入用户

编写脚本信息，通过api调用的方式导入用户，首先需要在gitlab中用管理员用户，调用gitlab的api，申请对应的token信息，步骤如下：

![](调用api信息.png)

然后编写插入脚本，注意在private_token参数位置填写gitlab中申请的token信息，如下：

```
$ vim gitlab_import_user.sh

// 插入以下内容
#!/bin/bash
# 导入gitlab用户信息
userinfo="userinfo.txt"
while read line
do
    password=`echo $line | awk '{print $1}'`
    username=`echo $line | awk '{print $2}'`
    mail=`echo $line | awk '{print $3}'`
    name=`echo $line | awk '{print $5}'`
    # 这里private_token使用你上面生成的token，只能在24小时内有效
    curl -d "password=$password&email=$mail&username=$username&name=$name&private_token=m6UTzKvksmfnaA4Nx957" "http://192.168.88.159:8181/api/v4/users"

done <$userinfo

// :wq保存退出

```
最后，将上述备份的userinfo.txt导入到目标服务器中，执行导入用户的脚本，如下：

```

$ chmod +x gitlab_import_user.sh

$ ./gitlab_import_user.sh

```

当用户导入完毕后，由于我们没有配置邮箱信息，需要使用管理员用户，登录到新的gitlab页面，用管理员账户确认每一个用户（Confirm user），使这些用户可用。操作单个用户，示意如下：

![](新增用户.png)

![](确认用户可用.png)


用户导入后，对用户进行分组，这里设置CMMP和ICP_Mobile两个分组，将各自用户导入即可。

### 3. 导出项目

直接备份目标服务器中的源码信息，进入gitlab所在的服务器中，定位到/var/opt/gitlab/git-data/repository路径下，对该路径下的所有文件夹进行打包。

```

# cd /var/opt/gitlab/git-data/repository/

# tar -zxvf gitlab-code.tar.gz .

```

将gitlab-code.tar.gz拷贝出来，使用scp传输到目标迁入机器上。

```

$ scp -P 15555 gitlab-code.tar.gz centos@192.168.88.159:~/

```

这样导出项目就完成了。

### 4. 导入项目

首先将压缩过的代码在目标迁入机器上进行解压，分批传入镜像中进行导入。

```

$ mkdir old_code && cd old_code

$ tar -zxvf gitlab-code.tar.gz ./

```

以test-user文件夹下的代码为例子进行导入，内部拥有gac-wechat这一个项目。首先对test-user文件夹进行压缩，转为test-user.tar.gz。

```
$ tar -zcvf test-user.tar.gz ./test-user

```

然后传入docker镜像中,

```
$ docker cp test-user.tar.gz gitlab:/

```

最后创建要导入的文件夹，和repositories同级，把压缩包解压到这个文件夹中，执行导入的操作。

```
$ docker exec -it gitlab /bin/bash

# cd /var/opt/gitlab/git-data/

// 创建要导入的文件夹
// 这里test-instance_mobile对应我们在gitlab中创建的分组信息
// 包含对应的用户
# mkdir -p repository-import-2020-05-25/test-instance_mobile

// 压缩包解压
# cd repository-import-2020-05-25/test-instance_mobile && tar zxvf /test-user.tar.gz .

// 修改所有者权限，一定注意修改，否则将导致项目无法导入
# cd ../.. && chown git.git -R repository-import-2020-05-25

// 执行导入命令
# cd && gitlab-rake gitlab:import:repos['/var/opt/gitlab/git-data/repository-import-2020-05-25/']

```

导入完成后，在gitlab页面中查看该项目，然后将新添加的用户分配给该项目，这样项目的迁移就完成了。其它项目照此办理，逐步进行导入即可。

![](添加用户到群组并关联项目.png)

### 5. 现有项目修改远程连接地址

项目迁移完成后，之前配置的远程url已经不能使用了，而且之前的ssh密钥同样需要进行更换。

#### 5.1 添加、更换ssh密钥

不更换ssh密钥，需要输入用户名密码进行登录操作。

#### 5.2 设置远程地址

在你本地开发机上进行操作，设置远程地址，命令行如下：

```
$ git remote -v

$ git remote set-url origin http://192.168.88.159:8181/test-instance_mobile/test-user/gac-wechat.git

```

### 6. 测试

在本地开发机测试拉取代码、提交代码等操作，即可。

### 7. 问题

停止已迁移的gitlab运行，直接使用docker stop命令停止。在重启该docker镜像的时候，打开页面出现502的返回码，运行出现了问题，查看日志如下：

```
==> /var/log/gitlab/unicorn/unicorn_stdout.log <==
bundler: failed to load command: unicorn (/opt/gitlab/embedded/bin/unicorn)
bundler: failed to load command: unicorn (/opt/gitlab/embedded/bin/unicorn)
bundler: failed to load command: unicorn (/opt/gitlab/embedded/bin/unicorn)
bundler: failed to load command: unicorn (/opt/gitlab/embedded/bin/unicorn)
bundler: failed to load command: unicorn (/opt/gitlab/embedded/bin/unicorn)
bundler: failed to load command: unicorn (/opt/gitlab/embedded/bin/unicorn)
bundler: failed to load command: unicorn (/opt/gitlab/embedded/bin/unicorn)
bundler: failed to load command: unicorn (/opt/gitlab/embedded/bin/unicorn)
bundler: failed to load command: unicorn (/opt/gitlab/embedded/bin/unicorn)
bundler: failed to load command: unicorn (/opt/gitlab/embedded/bin/unicorn)

==> /var/log/gitlab/unicorn/current <==
2020-06-12_02:22:48.15146 starting new unicorn master
2020-06-12_02:22:49.28312 master failed to start, check stderr log for details
2020-06-12_02:22:50.31663 failed to start a new unicorn master
2020-06-12_02:22:50.38006 starting new unicorn master
2020-06-12_02:22:51.45315 master failed to start, check stderr log for details
2020-06-12_02:22:52.47585 failed to start a new unicorn master
2020-06-12_02:22:52.52884 starting new unicorn master
2020-06-12_02:22:53.67915 master failed to start, check stderr log for details
2020-06-12_02:22:54.69961 failed to start a new unicorn master
2020-06-12_02:22:54.75306 starting new unicorn master

==> /var/log/gitlab/unicorn/unicorn_stdout.log <==
bundler: failed to load command: unicorn (/opt/gitlab/embedded/bin/unicorn)

==> /var/log/gitlab/unicorn/unicorn_stderr.log <==
ArgumentError: Already running on PID:436 (or pid=/opt/gitlab/var/unicorn/unicorn.pid is stale)
  /opt/gitlab/embedded/lib/ruby/gems/2.6.0/gems/unicorn-5.4.1/lib/unicorn/http_server.rb:205:in `pid='
  /opt/gitlab/embedded/lib/ruby/gems/2.6.0/gems/unicorn-5.4.1/lib/unicorn/http_server.rb:137:in `start'
  /opt/gitlab/embedded/lib/ruby/gems/2.6.0/gems/unicorn-5.4.1/bin/unicorn:126:in `<top (required)>'
  /opt/gitlab/embedded/bin/unicorn:23:in `load'
  /opt/gitlab/embedded/bin/unicorn:23:in `<top (required)>'

```

解决方式：

进入docker镜像，删除unicorn.pid文件，退出后再重启镜像，操作如下：

```
$ docker exec -it gitlab /bin/bash

root@gitlab:/# rm -rf /opt/gitlab/var/unicorn/unicorn.pid
root@gitlab:/# exit
exit

$ docker restart gitlab
gitlab

```

## 总结



## 参考地址

* https://www.cnblogs.com/kazihuo/p/11200585.html
* https://blog.csdn.net/hnmpf/article/details/80531444
* 单个项目导出：https://www.jianshu.com/p/848711844704
* https://tsov.net/uupee/25512/


## docker registry的备份和迁移

docker registry的启动：

docker run --name=registry --volume="/usr/local/registry:/var/lib/registry" -p 5000:5000 --restart=always registry:2 



备份docker registry镜像信息

主要是整体迁移，blob文件夹，以及打好的镜像信息。

安装前端页面

docker run \
  -d \
  -e ENV_DOCKER_REGISTRY_HOST=192.168.88.159 \
  -e ENV_DOCKER_REGISTRY_PORT=5000 \
  -p 5001:80 \
  konradkleine/docker-registry-frontend:v2


是否保留之前的构建产物？保留规则是什么？

直接拷贝/usr/local/registry路径下的所有文件，打压缩包进行拷贝。

## nexus3 的备份和迁移

备份nexus镜像

docker run -d --name=nexus3 --volume="/kichun/nexus3/nexus-data:/var/nexus-data" --privileged -p 8081:8081 --restart=always sonatype/nexus3

docker cp nexus3:/nexus-data/ ./nexus-data/

直接拷贝/nexus-data/路径下的所有文件，打压缩包进行拷贝。
