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

        // 所属用户组转换
        // 不做转换容易导致权限问题，无法进行备份
        root@10:/# chown git.root /var/opt/gitlab/backups/1579400282_2020_01_19_12.2.5_gitlab_backup.tar

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

## 补充：使用rpm的方式安装gitlab

```
$ wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-12.2.5-ce.0.el7.x86_64.rpm

$ sudo yum install -y policycoreutils-python

$ sudo rpm -i gitlab-ce-12.2.5-ce.0.el7.x86_64.rpm
warning: gitlab-ce-12.2.5-ce.0.el7.x86_64.rpm: Header V4 RSA/SHA1 Signature, key ID f27eab47: NOKEY
It looks like GitLab has not been configured yet; skipping the upgrade script.

       *.                  *.
      ***                 ***
     *****               *****
    .******             *******
    ********            ********
   ,,,,,,,,,***********,,,,,,,,,
  ,,,,,,,,,,,*********,,,,,,,,,,,
  .,,,,,,,,,,,*******,,,,,,,,,,,,
      ,,,,,,,,,*****,,,,,,,,,.
         ,,,,,,,****,,,,,,
            .,,,***,,,,
                ,*,.



     _______ __  __          __
    / ____(_) /_/ /   ____ _/ /_
   / / __/ / __/ /   / __ `/ __ \
  / /_/ / / /_/ /___/ /_/ / /_/ /
  \____/_/\__/_____/\__,_/_.___/


Thank you for installing GitLab!
GitLab was unable to detect a valid hostname for your instance.
Please configure a URL for your GitLab instance by setting `external_url`
configuration in /etc/gitlab/gitlab.rb file.
Then, you can start your GitLab instance by running the following command:
  sudo gitlab-ctl reconfigure

For a comprehensive list of configuration options please see the Omnibus GitLab readme
https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/README.md

$ sudo vim /etc/gitlab/gitlab.rb
// 修改external-url如下：
external_url 'http://192.168.1.198:8181'

// 配置刷新
$ sudo gitlab-ctl reconfigure

// 重启
$ sudo gitlab-ctl restart

```

启动完成后，访问http://192.168.1.198:8181，重新设置管理员密码，即可进行登录！

## 定时备份并发送备份文件到服务器

脚本信息编写如下：

```bash

#!/bin/bash
# 参数信息
remote_user='root'
remote_host='192.168.1.62'
remote_passwd='123456'
# 备份docker中的gitlab信息
docker exec -t gitlab gitlab-rake gitlab:backup:create
# 判断文件是否存在
bak_file_tar=`ls /home/centos/gitlab/data/backups/ | grep gitlab_backup`
if [ ! -f "$bak_file_tar" ]; then
    echo "备份文件 $bak_file_tar 存在，开始传输"
    sshpass -p $remote_passwd scp -r /home/centos/gitlab/data/backups/ $remote_user@$remote_host:/home/backup/gitlab/
    # sshpass -p $remote_passwd scp -r /home/centos/gitlab/data/backups $remote_user@$remote_host:~/
    echo "=============传输完毕================="
fi

```

主要是在docker中进行备份，由于我们的gitlab做了外部路径的映射，可以在外部路径直接传输备份文件。

在操作之前需要先安装sshpass工具，安装完成后需要先执行以下scp命令连接一下陌生的主机，获取主机指纹信息，获取完成后sshpass工具才能正常运行，如下：

```

# yum install sshpass -y

# scp workflow.txt root@192.168.1.62:~/
The authenticity of host '192.168.1.62 (192.168.1.62)' can't be established.
ECDSA key fingerprint is SHA256:rFFMBntJrMa/7ljJbRngqNmqhA/eFuuEZ2qmPqRh34k.
ECDSA key fingerprint is MD5:59:df:64:7c:48:3c:af:67:96:55:54:93:f1:5c:cb:c5.
Are you sure you want to continue connecting (yes/no)?yes
Warning: Permanently added '192.168.1.62' (ECDSA) to the list of known hosts.
root@192.168.1.62's password:

// ctrl+c终止执行

// 执行上面编写的脚本
# sh +x send.sh

```

## 说明

* 文件权限问题，所属

必须将文件所有者修改，同容器所在卷所有者信息一致。否则容易导致恢复不成功！

* gitlab停机问题

不停机就往里复制备份文件，容易出现不可知的问题。

## 参考文章

* https://www.jianshu.com/p/e7c056d273b6

* https://zhuanlan.zhihu.com/p/56108334
