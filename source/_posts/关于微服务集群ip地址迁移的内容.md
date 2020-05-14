---
title: 项目迁移到k8s的配置修改和问题记录
top: false
cover: false
toc: true
mathjax: true
date: 2020-05-11 13:51:42
password:
summary:
tags:
categories:
---

# 关于ip地址迁移的内容

## 涉及到修改IP地址的内容

1. 代码的硬编码中包含的ip地址信息

包括目前在代码中的硬编码的ip地址

2. 关于nacos配置文件中的配置的ip地址信息

包括所有服务涉及的ip地址信息，例如数据库、缓存等连接地址

3. 各个工具内部设置的ip地址

包括：

  - jenkins中配置的ip地址信息，包括各个服务配置中的ip地址、总体configure配置中的ip地址
  - gitlab中配置的ip地址信息，包括webhook配置的URL
  - 其它待补充（nexus、registry、sonarqube等等）

4. Linux服务器中涉及的IP地址

包括：

  - /etc/hosts中的ip地址
  - docker配置，/etc/docker/daemon.json
  - mysql、redis、mongodb中的ip配置
  - 其它待补充

5. 待增加

## 问题列表

1. 备份信息

* gitlab备份，代码+镜像
* jenkins备份，插件列表+项目配置
* 数据库备份

2. 重建k8s集群

IP地址更改导致证书以及集群环境的变化，导致k8s必须重新进行部署

3. 是否能够分批进行地址迁移？
