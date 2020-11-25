---
title: 关于微服务中Skywalking监控信息和ELK统一日志管理体系的对接
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-07 10:25:09
password:
summary:
tags:
categories:
---

## 当前场景

以Skywalking+ELK体系构建当前针对微服务的“链路+日志监控”的整个监控手段，主要监控微服务的运行层面的问题。目前在公司的应用场景下存在的严重问题是，目前无法将二者进行联动，也就是说，在skywalking中看到了请求异常，无法马上转到ELK中查看对应链路上的微服务的所有日志，导致排查问题效率不足，影响问题解决。

## 解决方案

目前考虑的是，从Skywalking的前端页面进行抓取，获取存在问题的服务链路，获取各个微服务名称，在skywalking和ELK之外，再写一个微服务，这个微服务承载从Skywalking和ELK之间的连接，称之为MicroBridge。然后从Skywalking获取的服务信息，传入MicroBridge，最后从MicroBridge根据对应的时间信息和微服务名称，从ES中查询出对应的日志信息，进行展示。

## 也许是更好的方案

[参考文章](https://cloud.tencent.com/developer/article/1696699)

直接把traceId写入ES中，通过traceId进行日志搜索，更加高效。

但是这样对于业务微服务的侵入较大，是否对性能造成影响还有待考虑，毕竟频繁打印日志。

