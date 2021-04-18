---
title: jenkins批量复制job到新的view中的执行脚本
top: false
cover: false
toc: true
mathjax: true
date: 2020-11-02 17:54:56
password:
summary:
tags:
categories:
---

# jenkins批量复制job到新的view中的groovy脚本


```groovy

import hudson.model.*
  
def str_view = "testcloud-develop"
def str_new_view = "testcloud-212-develop-single"
def str_search = "testcloud-"
def str_replace = "testcloud-212-"
  
def view = Hudson.instance.getView(str_view)
  
//copy all projects of a view
for(item in view.getItems())
{
  if (!item.getName().contains(str_search)) {
      // 说明文字，表示跳过未匹配到的job，可加可不加
	  println("but $item.name ignore ")
      continue
  }
 
  //create the new project name
  newName = item.getName().replace(str_search, str_replace)
 
  
  // copy the job, disable and save it
  def job 
  try {
      job = Hudson.instance.copy(item, newName)
  }catch(IllegalArgumentException e) {
      // 重名的处理
      println("$newName job is exists")
      continue
  } catch(Exception e) {
      // 获取异常的处理
      println(e.toString())
      continue
  }
  job.disabled = true
  job.save()
   
  // update the workspace to avoid having two projects point to the same location
  //AbstractProject project = job
  //def new_workspace = project.getCustomWorkspace().replace(str_search, str_replace)
  //project.setCustomWorkspace(new_workspace)
  //project.save()
   
  Hudson.instance.getView(str_new_view).add(job)
  println(" $item.name copied as $newName")
 
}

```

注意，这里在使用之前，要先添加对应的View，添加时操作如下：

![](jenkins添加View的问题.png)
