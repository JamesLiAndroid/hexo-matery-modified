---
title: 关于多module下代码结构的gitlab构建问题解决
top: false
cover: false
toc: true
mathjax: true
date: 2021-01-14 20:21:23
password:
summary:
tags:
categories:
---

# 关于多module代码结构的gitlab构建问题解决

## 前言

目前项目结构中，存在一个项目多个module的情况，也就是一个git项目中，多个微服务代码放在一个文件夹下的场景，而多个微服务对应在jenkins中创建了多个构建任务。这种场景下带来的问题是，当该文件夹下某一个微服务的代码修改后，会触发该文件夹下所有微服务进行构建。所以需要避免这个问题。

## 面临问题

如何提取git记录信息，该记录信息能够明确表示提交目录，用来判断是否输入该目录下的文件修改。

## 解决方式

在gitlab中配置了每个项目的webhook链接用以触发构建，于是从webhook下手，利用webhook中的请求信息进行区分。下面是一次webhook发送时的请求体信息：

```json
{
  "object_kind": "push",
  "event_name": "push",
  "before": "5c28233547357212cde8f51ca3608a87439c274a",
  "after": "e6e94eaff904bd377448a7b795b9a7b79687d42f",
  "ref": "refs/heads/master",

  .....

  "project": {
   ......
  },
  "commits": [
    {
      "id": "5c28233547357212cde8f51ca3608a87439c274a",
      "message": "build(project): 修改忽略文件\n\n1. 修改忽略文件\n",
      "timestamp": "2021-01-08T02:24:51Z",
      "url": "/commit/5c28233547357212cde8f51ca3608a87439c274a",

      .......

      "added": [

      ],
      "modified": [
        "test-micro-service-bda-dataanalysis-management/.gitignore",
        "test-micro-service-bda-dataset-management/.gitignore",
        "test-micro-service-bda-datasource-management/.gitignore",
        "test-micro-service-bda-datavisualization-management/.gitignore",
        "test-micro-service-psl-equipment-management/.gitignore"
      ],
      "removed": [

      ]
    }
  ],
  "total_commits_count": 3,
  "push_options": {
  },
  "repository": {
    ......
  }
}

```

如下可以看到在*commit*节点中，主要的提交信息分为added、modified、removed这三部分，其内分为其它不同的文件，需要匹配其修改的文件所在路径。

通过gitlab中的Generic Webhook Trigger插件实现添加两个信息，一个是原有的判断分支的参数*ref*，另一个是需要进行匹配内容的范围，也就是*commit*节点中的修改内容。通过正则表达式先获取这两个内容，在最后的*Optional filter*这个部分进行最终判断。最后jenkins任务示例设置如下：

![](多参数判断条件设置.png)

首先在*Post content parameters*添加两个参数，如下表

|Variable |Expression（type） | value filter |
|---------|-------------------|--------------|
| ref | $.ref（JSONPath） | ^(refs/heads/|refs/remotes/origin/) |
| changed_files | $.commits[*].['modified','added','removed'][*] （JSONPath）| |

然后添加触发的token信息，最后在Optional filter部分添加匹配的内容，如下：

* 匹配文本（Text）

```R

$ref $changed_files，

```

中间有空格分隔，表示两个参数信息，获取的内容组合为一个字符串

* 正则表达式（Expression）

```R

develop\s(.*"(test-micro-service-app-mng/)[^"]+?"){1,}

```

匹配特定分支，加上文件夹路径信息，和上面的匹配文本进行对比。如果满足则启动构建，否则不启动。

这样就能区分在特定分支下提交的内容能够匹配并触发构建。
