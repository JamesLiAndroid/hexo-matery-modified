---
title: gitlab备份
top: false
cover: false
toc: true
mathjax: true
date: 2020-01-18 12:12:02
password:
summary:
tags: git gitlab DevOps
categories: DevOps
---

# 关于gitlab的备份和导入

## 背景

* 操作环境：CentOS 7.6，Docker version 19.03.5

* 待备份对象：192.168.1.199:8181下的gitlab，备份该gitlab镜像

* 启动时已经指定了镜像名称gitlab

* 备份后目标恢复的服务器：192.168.1.198

## docker中gitlab 的备份

首先登录192.168.1.199服务器，找到gitlab所在容器，对该容器的当前状态进行备份。

        // 找到gitlab在运行的容器信息
        $ docker ps -a | grep gitlab 

        ffce609db97c        gitlab/gitlab-ce:latest     "/assets/wrapper"        2 months ago        Up 11 days (healthy)       0.0.0.0:2222->22/tcp, 0.0.0.0:8181->80/tcp, 0.0.0.0:8443->443/tcp   gitlab

        // 进入该容器
        $ docker exec -it ffce /bin/bash

        // 执行内部gitlab自身的备份命令
        root@10:/#  gitlab-rake gitlab:backup:create

        // 输入exit退出容器

        // 将容器中以及备份的内容拷贝出来
        $ docker cp gitlab:/var/opt/gitlab/backups/1579400282_2020_01_19_12.2.5_gitlab_backup.tar .

        // 执行备份容器的命令

        // 提交保存旧容器（容器名：gitlab）
        $ docker commit gitlab gitlab/gitlab-ce:12.2.5

        // 从旧docker导出gitlab容器镜像
        $ docker export gitlab > gitlab-ce-12.2.5.img

完成后，已经存在docker容器的备份信息，以及gitlab的备份信息了。随后将其拷贝出来，准备到恢复环境中运行。


## gitlab恢复

首先通过ftp将文件传输到192.168.1.198服务器上，然后进行恢复操作。

        // 导入镜像
        $ docker import - gitlab/gitlab-ce:12.2.5 < gitlab-ce-12.2.5.img

        // 查看镜像是否被导入
        $ docker images

        REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
        gitlab/gitlab-ce    12.2.5              a574a1869b14        3 hours ago         1.75GB

        // 创建文件夹
        $ mkdir /home/centos/gitlab_all/logs

        $ mkdir /home/centos/gitlab_all/data

        $ mkdir /home/centos/gitlab_all/config

        // 启动导入的镜像
        $ docker run  \
            -d \
            -h gitlab \
            -p 2222:22 \
            -p 8181:80 \
            -p 8443:443 \
            -v /home/centos/gitlab_all/config:/etc/gitlab \
            -v /home/centos/gitlab_all/logs:/var/log/gitlab \
            -v /home/centos/gitlab_all/data:/var/opt/gitlab \
            --restart always \
            --name gitlab \
            gitlab/gitlab-ce:12.2.5  /assets/wrapper
        
        // 复制之前的备份信息放入backup目录
        // 停止gitlab服务
        $ docker stop gitlab

        // 修改用户和用户组权限
        $ sudo chown root.root 1579400282_2020_01_19_12.2.5_gitlab_backup.tar
        // 拷贝文件
        $ cp 1579400282_2020_01_19_12.2.5_gitlab_backup.tar /home/centos/gitlab_all/data/backups/
        // 启动gitlab
        $ docker start gitlab

        // 进入gitlab容器开始执行恢复命令

        $ docker exec -it gitlab /bin/bash

        // 停止数据服务
        root@10:/# gitlab-ctl stop unicorn
        root@10:/# gitlab-ctl stop sidekiq

        // 检查关闭状态
        root@10:/# gitlab-ctl status

        // 开始进行数据恢复
        root@10:/# gitlab-rake gitlab:backup:restore BACKUP=1579400282_2020_01_19_12.2.5

        // 恢复主要分两部分，第一部分是数据库的变更，需要确定是否删除之前的数据库表
        // 第二部分是仓库的变更
        
        // 重启gitlab
        root@10:/# gitlab-ctl restart

        // 验证是否恢复成功
        root@10:/# gitlab-rake gitlab:check SANITIZE=true

        // 输入exit退出

这时访问192.168.1.198:8181，可以访问到新的gitlab，用之前的用户名密码进行登录即可。但是可能之前的ssh_key会不可用。

## 说明

* 文件权限问题，所属

必须将文件所有者修改，同容器所在卷所有者信息一致。否则容易导致恢复不成功！

* gitlab停机问题

不停机就往里复制备份文件，容易出现不可知的问题。


## 参考文章

* https://www.jianshu.com/p/e7c056d273b6

* https://zhuanlan.zhihu.com/p/56108334
