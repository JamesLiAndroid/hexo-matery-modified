---
title: DevOps介绍
top: false
cover: false
toc: true
mathjax: true
date: 2020-01-12 16:00:18
password:
summary:
tags: DevOps
categories: DevOps
---

# DevOps ppt讲解

1. 理念

* 是什么？并没有明确的定义！

    * 字面意思：Develop+Operations，开发运维一体化
    * 深层次含义：DevOps集文化理念、实践和工具于一身。可以提高组织交付应用程序和服务的能力。与使用传统软件和基础设施管理流程相比，能够帮助组织更快的发展和改进产品。这种速度使组织更好的服务其客户，并在市场上高效的参与竞争。（AWS）
    * 通俗来讲：为了帮助研发团队在保持质量的前提下提高交付效率的方法和方法论都隶属于 DevOps 的范畴。
    * 管理层面：全新的管理理念，打破职责屏障，减少研发、测试、运维的矛盾，统一思想，从业务需求出发，向着同一个目标前进

* 为什么？

企业内部的效率提升和研发成本的控制，保证高质量的同时减少发布流程的总耗时。

着眼在质量+效率上

继承自敏捷开发

* 核心理念

DevOps的核心实践理念统称为CALMS：文化（Culture）、自动化（Automation）、精益（Lean）、度量（Measurement） 、共享（Share）。

文化：统一思想，精简组织架构，强调协作和融合，标准化体系建设。

自动化：自动化工具的引入，灵活部署，解放双手，聚焦业务实现。

精益：代码质量、接口质量的提升，对于问题的快速响应和快速解决，事故复盘和学习。

度量：系统监控以及告警

共享：共同学习，共同成长，允许试错


2. 场景

将DevOps定义为一种强调通过高速、小型化和迭代步骤的应用程序开发、部署和反馈环方式，提供更主动的响应来满足客户需求的方法。它的特点是业务文化上的转变，将开发团队和运营团队作为一个团队，专注于实现业务价值。

特别是微服务场景下，天然适合DevOps理念。

强调合作性，运维与开发相互配合，协同工作。

3. 应用

开发云有项目管理、配置管理、代码检查、编译构建、测试、部署、发布、流水线这几大服务。以华为为例。

开源的应用工具链信息：

* 版本控制&协作开发：GitHub、GitLab、BitBucket、SubVersion、Coding、Bazaar
* 自动化构建和测试:Apache Ant、Maven 、Selenium、PyUnit、QUnit、JMeter、Gradle、PHPUnit
* 持续集成&交付:Jenkins、Capistrano、BuildBot、Fabric、Tinderbox、Travis CI、flow.ci Continuum、LuntBuild、CruiseControl、Integrity、Gump、Go
* 容器平台: Docker、Rocket、Ubuntu（LXC）、第三方厂商如（AWS/阿里云）
* 配置管理：Chef、Puppet、CFengine、Bash、Rudder、Powershell、RunDeck、Saltstack、Ansible
* 微服务平台：OpenShift、Cloud Foundry、Kubernetes、Mesosphere
* 服务开通：Puppet、Docker Swarm、Vagrant、Powershell、OpenStack Heat
* 日志管理：Logstash、CollectD、StatsD
* 监控，警告&分析：Nagios、Ganglia、Sensu、zabbix、ICINGA、Graphite、Kibana

重中之重：标准化体系的建设。

4. 困难/痛点

* 既有项目的复杂依赖

已有单体项目难以拆分的问题

* 自动化程度不足
    
    * 持续构建
    * 版本控制
    * 代码质量

* 项目交付与交付质量

交付速度和交付质量的平衡，利用持续集成来保障。

* 多运行环境的管理

多套运行环境，开发环境、测试环境、灰度环境、正式环境等等，各环境要求不同，配置不同，如何统一纳入管理，是问题。

* 开发人员身兼多职的问题

全才和专才的矛盾，全才干专才的事情，专才往全才转换，都是一种人才资源的浪费。

* 对公司：摸着石头过河

各公司要求不一致，发展程度、业务要求不一致，如何找到适合自己的方式。

5. 误区

* DevOps是银弹

没有银弹

* 站在运维角度看开发/站在开发角度看运维

都不对，应该站在架构或者更高的位置看，站在全局的位置看。

* 强调开发人员/运维人员的全栈能力？

强调的是协作和融合，发挥个人长处即可，打破协作的壁垒。

* 对开发：一定要敏捷？

如果团队实行敏捷开发的方式，承接敏捷开发，引入DevOps是正确的思路；如果不是敏捷开发的方式，那么可以利用DevOps引入自动化流水线。

6. 总结

DevOps是一种工程模式，本质上是一种分工，通过对开发、运维、测试，配管等角色职责的分工，实现工程效率最大化，进而满足业务的需求。

DevOps的核心是角色的分工，而不是组织架构变化，垂直化的组织架构不代表可以实现DevOps所需要的分工模式，横向的组织架构也不代表传统的分工模式。

DevOps的目标是工程效率最大化，它本身也只是一种方法论，是为了实现工程效率最大化的目标而存在的。

DevOps应该是一种有价值的试错，通过实践形成满足我们团队的开发运维模式。

7. 参考文章

* 具体实践：https://library.prof.wang/handbook/h/hdbk-MWnS99ThmLVDi7U5mVFrB9#toc314

* 理念：https://www.jianshu.com/p/645bb1283a77

* 腾讯实践：https://cloud.tencent.com/developer/article/1084454

* 工具集：https://www.cnblogs.com/SanMaoSpace/p/10110703.html

* 初学者指南：https://zhuanlan.zhihu.com/p/36905891

* 传统行业落地指南：https://dbaplus.cn/news-134-2068-1.html

* [IBM 的DevOps手册](ram14026-01-devops-for-dummies-3rd-ibm-editon-simplified-chinese_RAM14026CNZH.pdf)

* 极客时间《DevOps实战笔记》