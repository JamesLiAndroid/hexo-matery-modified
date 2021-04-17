---
title: 两个gitlab之间的仓库同步问题解决
top: false
cover: false
toc: true
mathjax: true
date: 2021-04-16 17:26:34
password:
summary:
tags:
categories:
---

# 两个gitlab之间的仓库同步问题解决

## 问题场景

由于我们在用户现场进行开发，而公司本部这边存放有我们自己的基础平台代码，而两边开发的过程中，不可避免的存在代码同步的问题。中间还存在VPN的连接问题，于是在这样的环境下实现代码同步，而且是尽可能的自动同步。

经过了解，查找了各项资料，发现一个问题，就是绝大部分都是自有或者本地gitlab和github之间进行的同步，不好找gitlab和gitlab之间的同步方式。虽然理论上应该是一致的，但实际操作上还有不同。

根据我英明领导的要求，要实现全双工通信，也就是说，在两个gitlab上创建互为镜像的项目，这两个项目可以定期的互相同步各自的代码。现在我只能实现单向通信，就是只有一侧进行同步，实现的是一种主从模式，如果有朋友能实现互相同步的，请联系我。但是我想了下，两个项目间互为镜像，是不是会造成先有鸡后有蛋的问题？就是必须先有一个项目，才能将另一个项目作为该项目的镜像？

## 实现方案介绍

### 环境以及必备工具介绍

* 两个gitlab

  - 公司本部gitlab版本：12.2.5，ip地址（内网）：192.168.166.202:8181。
  - 客户现场gitlab版本：12.4.2，ip地址（内网）：192.168.229.52:8181。

操作时均使用root账号操作，这一步比较危险，最好控制下权限，尽可能使用该项目的developer角色或者reporter角色的用户来进行操作。

在公司本部创建项目，studentmanagement2，以该项目进行测试。

* 两个开源项目

借助两个开源项目实现，分别是：

  - [git-mirrors](https://github.com/samrocketman/gitlab-mirrors.git)  
  - [python-gitlab3](https://github.com/doctormo/python-gitlab3.git)

在python 2.7环境下运行，api兼容测试的gitlab版本。将这个中转程序放到客户现场的gitlab所在的机器上。

### 基本概念介绍

#### 1. git mirrors

引用开源中国的解释如下：

```
$ git clone --mirror $URL

$ git clone --bare $URL

等同于

$ (cd $(basename $URL) && git remote add --mirror=fetch origin $URL)

当前的man-page如何表达：

相比之下--bare，--mirror不仅将源的本地分支映射到目标的本地分支，它还映射所有引用（包括远程分支，注释等）并设置refspec配置，以便所有这些引用都被git remote update目标存储库中的a覆盖。

```

简单来说，--bare就是裸克隆，只克隆该分支的信息。而--mirror是完整的镜像克隆，克隆所有相关的提交信息，包含不同的分支！

#### 2. 整个实现流程介绍

核心是依靠git-mirrors项目进行中转，利用python-gitlab3依赖进行gitlab接口的调用。通过脚本利用git clone --mirror选项从公司本部的gitlab库，同步到客户现场的gitlab库中，并在中转机器上设置定时任务，进行定时同步。

示意图如下：

![](同步示意图.png)

### 实际操作流程

#### 0. 为客户现场中转主机进行环境配置

默认都在centos用户下安装，生成证书信息等操作。

安装git

```
$ sudo yum install -y git

```

已有python2.7的情况下，配置虚拟环境：

```
// 安装pip
$ curl https://bootstrap.pypa.io/pip/2.7/get-pip.py -o get-pip.py
$ sudo python get-pip.py

// 安装virtualenv
$ pip install virtualenv

// 生成虚拟环境
$ mkdir git_repository_sync && virtualenv venv --python=python2.7

```

安装python-gitlab

```
$ yum install python-setuptools

$ git clone https://github.com/alexvh/python-gitlab3.git

$ cd python-gitlab3

$ git checkout v0.5.8

$ python setup.py install

```

#### 1. 在客户现场的gitlab中配置账号

- 登录GitLab

- 创建一个用户

- 为该用户赋予管理员权限。简单起见，笔者使用root 这个GitLab的内置账户。

- 在GitLab创建一个Group，名字为test，该名称在下面的配置文件中会用到。


#### 2. 在虚拟机中配置用户并生成ssh key

这一步需要在root用户下执行

```
// 创建新用户
# adduser gitmirror

// 切换用户
# su - gitmirror

// 生成ssh密钥
$ ssh-keygen -t rsa -C

// 或者加入公司本部的gitlab所在的用户邮箱信息
$ ssh-keygen -t rsa -C "admin@example.com"

// 查看生成的密钥信息
$ more /home/gitmirror/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDUdvDGqmvboURWy1YV1rexD+vRhkt4eEDoQuUBbQ6ceH6Ta/NRDPK6o/BnBC9n0AoSrXtHem5PYPnYKkLkv0yQo3JwtN/p07lH4SoN8/v+aHzCP87A5j1wSHA
/6NB8CKefeZcR87llP/3P0v2gQukCcr3G1CaJeAhQ2lvLdimftgh83vvMv+28IJlt5UsJK0MqeQHw7uOA8//mgbEvZVHzzOGNpX/xkcj4lbCXhnLCyCr1/Q7LIKneJuxqZG1oyongIDU+yq3WFtNDGn8ORwIGVz
XvA7sfTQF0npSsgKIJJKRkmn5+6pQtSB8WAYfrHc10qBzieFRlV5mZ7KYtySql admin@example.com

```

**注意：**如果切换gitmirror用户后生成了密钥信息，而python的虚拟环境建在centos用户下，那么需要将生成的证书信息导入到centos用户下，否则在执行git clone操作时，会出现找不到证书的情况。拷贝证书的操作如下：

```
// 记得备份原有证书

// 拷贝证书操作
# cp /home/gitmirror/.ssh/id_rsa* /home/centos/.ssh/ 

```

#### 3. 为公司本部的gitlab设置免密码登录

将上面生成的密钥信息拷贝的公司本部的gitlab用户中，操作如下：

![](gitlab添加ssh-key信息.png)

同样将该密钥信息也添加到客户现场的gitlab中。

测试ssh访问情况

```
// 测试公司本部的gitlab访问
$ ssh -T -p 2222 git@192.168.166.202

// 这里gitlab使用docker部署，ssh端口为2222

// 测试客户现场的gitlab访问
$ ssh -T git@192.168.229.52

```

#### 4. 为客户现场的gitlab设置access_token（设置private token）

添加access_token的步骤如下：

![](添加access_token.png)

点击绿色按钮后将会生成一个新的token，复制该token（只会出现一次）。

![](复制access_token.png)

复制完成后，填写到客户现场中转服务器中，如下：

```
# su - gitmirror

$ touch private_token && echo '你复制的密钥信息' > private_token

```

#### 5. 创建本地仓库路径

GitLab Mirrors会将GitHub上的代码clone到本地，默认是~/repositories ，因此我们得创建该目录。

```

# su - gitmirror

$ mkdir ~/repositories

```

#### 6. 下载并配置gitlab-mirrors

切换到centos用户执行，如下：

```
# su - centos

$ git clone https://github.com/samrocketman/gitlab-mirrors.git

$ cd gitlab-mirrors

$ chmod 755 *.sh

// 添加配置文件
$ cp config.sh.SAMPLE config.sh

```

下面开始修改配置文件：

```

$ cd gitlab-mirrors && vim config.sh

#Environment file

#
# gitlab-mirrors settings
#

#The user git-mirrors will run as.
system_user="gitmirror"
#The home directory path of the $system_user
user_home="/home/${system_user}"
#The repository directory where gitlab-mirrors will contain copies of mirrored
#repositories before pushing them to gitlab.
repo_dir="${user_home}/repositories"
#colorize output of add_mirror.sh, update_mirror.sh, and git-mirrors.sh
#commands.
enable_colors=true
#These are additional options which should be passed to git-svn.  On the command
#line type "git help svn"
git_svn_additional_options="-s"
#Force gitlab-mirrors to not create the gitlab remote so a remote URL must be
#provided. (superceded by no_remote_set)
no_create_set=false
#Force gitlab-mirrors to only allow local remotes only.
no_remote_set=false
#Enable force fetching and pushing.  Will overwrite references if upstream
#forced pushed.  Applies to git projects only.
force_update=false
#This option is for pruning mirrors.  If a branch is deleted upstream then that
#change will propagate into your GitLab mirror.  Aplies to git projects only.
prune_mirrors=false

#
# Gitlab settings
#

// 注意这下面的内容
#This is the Gitlab group where all project mirrors will be grouped.
// 在客户现场gitlab中创建的group名称
gitlab_namespace="test"
#This is the base web url of your Gitlab server.
// 在客户现场gitlab地址
gitlab_url="http://192.168.229.52:8181"
#Special user you created in Gitlab whose only purpose is to update mirror sites
#and admin the $gitlab_namespace group.
// 在客户现场gitlab中使用的用户名
gitlab_user="root"
#Generate a token for your $gitlab_user and set it here.
// 默认的获取access_token的文件
gitlab_user_token_secret="$(head -n1 "${user_home}/private_token" 2> /dev/null || echo "")"
#Sets the Gitlab API version, either 3 or 4
gitlab_api_version=4
#Verify signed SSL certificates?
ssl_verify=false
#Push to GitLab over http?  Otherwise will push projects via SSH.
http_remote=false

#
# Gitlab new project default settings.  If a project needs to be created by
# gitlab-mirrors then it will assign the following values as defaults.
#

#values must be true or false
issues_enabled=false
wall_enabled=false
wiki_enabled=false
snippets_enabled=false
merge_requests_enabled=false
public=false

// :wq保存退出

```

#### 7. 添加仓库同步配置并开始同步

在centos用户下执行最后的操作，添加同步的公司本部仓库信息如下：

```

# su - centos

$ ./add_mirror.sh --git --project-name test --mirror ssh://git@192.168.166.202:2222/test-cloud/demo-business/express/studentmanage2.git

```

**注意：**这里使用的ssh格式的地址信息，而不是使用http链接的信息。

添加命令执行无错误，即可进行同步，执行如下：

```

$ cd ~/git_repository_sync/gitlab-mirrors && ./git_mirrors.sh

```

执行完成后，则可以看到客户现场的gitlab中有了我们自己的项目，如下：

![](同步项目截图.png)

如果需要定时同步，直接编写crontab规则即可，如下：

```

$ crontab -e

*/5 * * * * /home/centos/git_repository_sync/gitlab-mirrors/git_mirrors.sh

```

## 注意事项

## 总结

在上述操作下，可以实现两地间gitlab仓库的单向同步

## 参考连接

* http://www.itmuch.com/work/git-repo-sync-with-gitlab-mirrors/