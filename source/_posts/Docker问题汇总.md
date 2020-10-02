---
title: Docker问题汇总
top: false
cover: false
toc: true
mathjax: true
date: 2020-10-02 09:30:11
password:
summary:
tags:
categories:
---

## 关于Docker进程错误的问题解决

错误日志如下：

```
docker: Error response from daemon: driver failed programming external connectivity on endpoint server-register-k8s (c2d9369369d37d9ce02df690ae39787903fa3fc9b470ad1763c072575f8a8088):  (iptables failed: iptables --wait -t nat -A DOCKER -p tcp -d 0/0 --dport 19013 -j DNAT --to-destination 172.17.0.2:19013 ! -i docker0: iptables: No chain/target/match by that name.
18:37:04  (exit status 1)).

```

解决方案：

```
$ sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

$ sudo systemctl restart docker

```

## 解决docker因为polkit进程挂掉无法启动的问题

```
$ docker ps -a
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.40/containers/json?all=1: dial unix /var/run/docker.sock: connect: permission denied

$ sudo systemctl restart docker
Authorization not available. Check if polkit service is running or see debug message for more information.
Failed to restart docker.service: Connection timed out
See system logs and 'systemctl status docker.service' for details.


$ sudo systemctl status polkit
[sudo] password for centos:
● polkit.service - Authorization Manager
   Loaded: loaded (/usr/lib/systemd/system/polkit.service; static; vendor preset: enabled)
   Active: failed (Result: timeout) since Wed 2020-07-22 00:16:04 CST; 1 months 6 days ago
     Docs: man:polkit(8)
  Process: 801 ExecStart=/usr/lib/polkit-1/polkitd --no-debug (code=killed, signal=TERM)
 Main PID: 801 (code=killed, signal=TERM)

Warning: Journal has been rotated since unit was started. Log output is incomplete or unavailable.




$ sudo systemctl restart polkit
Authorization not available. Check if polkit service is running or see debug message for more information.
Failed to restart polkit.service: Connection timed out
See system logs and 'systemctl status polkit.service' for details.



$ sudo yum install -y setroubleshoot setools

$ sudo systemctl restart polkit

$ sudo systemctl restart docker

$ docker ps -a
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.40/containers/json?all=1: dial unix /var/run/docker.sock: connect: permission denied

$ sudo gpasswd -a ${USER} docker

$ newgrp docker

$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS               NAMES
222ca0539f7b        hello-world         "/hello"            11 months ago       Exited (0) 11 months ago                       recursing_nobel


```

参考链接：https://www.cnblogs.com/dream397/p/13294875.html?utm_source=tuicool