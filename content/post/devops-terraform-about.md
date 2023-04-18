---
title: "自动化编排工具Terraform介绍"
date: 2021-02-24T17:22:42+08:00
lastmod: 2021-02-24T17:22:42+08:00
draft: false
description: "初识Terraform介绍"
tags: ["terraform", "aliyun"]
categories: ["devops"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---


## Terraform是什么？：
Terraform是由HashiCorp公司在2014年左右推出的开源资源编排工具, 目前几乎所有的主流云服务商都支持Terraform，包括阿里云、腾讯云、华为云、AWS、Azure、百度云等等。目前很多公司都基于terraform构建自己的基础架构。

> 诞生背景：
传统运维模式下，业务上线需经过设备采购，机器上架，网络环境搭建和系统安装等准备阶段。随着云计算的兴起，各大公有云厂商均提供了非常友好的交互界面，用户借助一个浏览器就可以按需采购各种云资源，快速实现业务架构的搭建。然而，随着业务架构的不断扩展，云资源采购的规模和种类也在持续增加。当用户需要快速采购大量不同类型的云资源时，云管理页面间大量的交互操作反而降低了云资源的采购效率。在阿里云控制台上初始化一个经典的VPC网络架构，从创建VPC、交换机VSwitch到创建Nat网关、弹性IP再到配置路由等工作，大概要花费20分钟甚至更久。同时，工作成果的不可复制性，带来的是跨Region和跨云平台场景下的重复劳动。

事实上，对业务运维人员而言，只关心对资源的配置，无需关心这些资源的创建步骤。如同喝咖啡，只需要告诉服务员喝什么，加不加冰等就够了。如果有一份完整的云资源采购清单，这张清单清楚的记录了所需要购买的云资源的种类，规格，数量以及各云资源之间的关系，然后一键完成购买，并且当业务需求发生变化时，只需要变更清单就可以实现对云资源的快速变更，那么效率就会提高很多。在云计算中这被称作资源编排，目前很多云平台也提供了资源编排的能力，如阿里云的ROS，AWS的CloudFormation等。

> 将云资源、服务或者操作步骤以代码的形式定义在模板中，借助编排引擎，实现资源的自动化管理，这就是基础设施即代码（Infrastructure as Code，简称IaC），也是资源编排最高效的实现模式。然而，多种云编排服务带来的是高昂的学习成本、低效的代码复用率和复杂的多云协同工作流程。每一种服务仅限于管理自家的单一云平台上，无法满足对多个云平台，多种层级（如IaaS，PaaS）资源的统一管理。如何解决如上问题，是否可以使用统一的编排工具，共用一套语法实现对包括阿里云在内的多云的统一管理呢？所以这个时候就诞生Terraform，来解决这些问题。

## Terrafrom功能和作用：
### 功能点
- IaC：infrastructure as code，用代码管理基础设施
- 执行计划：显示terraform apply时执行的操作
- 资源图：构建所有资源的图形
- 变更自动化：基于执行计划和资源图，可以清晰知道要变更的内容和顺序
总结：terraform用于各类基础设施资源初始化，支持多种云平台，支持第三方服务对接

### 作用
- 使用不同provider的API，包装抽象成Terraform的标准代码结构
- 用户不需要了解每个云计算厂商的API细节，降低了部署难度

## Terraform架构
Terraform本身是基于插件的架构，可扩展性很强，可以方便程序员对Terraform进行扩展。Terraform从逻辑上可以分为两层，核心层（Terraform Core）和插件层（Terraform Provider）。

### 核心层
核心层其实就是terraform的命令行工具，它是用go语言开发的，它负责：
1. 读取.tf配置，进行变量替换
2. 资源状态文件管理
3. 分析资源关系，绘制图谱
4. 依赖关系图谱，创建资源
根据依赖关系，创建资源；对于没有依赖关系的资源，会并行进行创建(缺省10个并行进程），这也是Terraform能够高效快速管理云资源的原因。
5. 用RPC调用插件层

### 插件层
插件层也是由go语言开发的，Terraform有超过250个不同的插件，它们负责：
  - 接受核心层的RPC调用
  - 具体提供某一项服务的执行

插件层又有两种：
#### Provider
Provider，负责与外界API的集成，比如阿里云Provider就提供了在阿里云创建、修改、删除云资源的功能。这个插件负责和阿里云云API的接口交互，并提供一层抽象，这样程序员可以在不了解API细节的情况下，通过terraform来编排资源。它负责：

- 始化以及外界API通信
- 外界API的认证
- 定义云资源与外界服务的关系

比如常见provider:
```
阿里云： https://github.com/aliyun/terraform-provider-alicloud
百度云：https://github.com/baidubce/terraform-provider-baiducloud
腾讯云：https://github.com/tencentcloudstack/terraform-provider-tencentcloud
华为云：https://github.com/huaweicloud/terraform-provider-huaweicloud
ucloud：https://github.com/ucloud/terraform-provider-ucloud
qingcloud：https://github.com/yunify/terraform-provider-qingcloud
AWS：https://github.com/hashicorp/terraform-provider-aws
Azure：https://github.com/terraform-providers/terraform-provider-azurerm
GoogleCloud：https://github.com/hashicorp/terraform-provider-google
```
#### Provisioner
Provisioner，负责在资源创建或者删除完成后，执行一些脚本。比如Puppet Provisioner就可以在云虚拟机资源创建完成后，在该资源上下载、安装、配置Puppet agent。

为了方便理解,网络上找了一个组件架构图，简单说明各个组件位置：

![terraform架构图](/images/2021/terraform-about.png)

对于terraform日常操作，我画了一个基本workflow流程图如下：
![terraform操作流程图](/images/2021/terraform-workflow.png)

## terraform关键字解释：
### 声明式语言（HCL）：
Terraform是通过HashiCorp Configuration Language来编写代码的，HCL是声明式的，也就是说，程序员用HCL来描述整个基础架构应该是什么样的，然后把具体的实施工作交给Terraform就可以了，程序员不需要了解实施的具体步骤和细节，不需要了解terraform如何与云服务商的API进行对接。Terraform会根据代码，自动下载相应的Provider和Provisioner来负责具体步骤和细节。于声明式对应的是命令式。命令式语言是按照步骤执行的，先后顺序很重要，对固定输入执行命令式语言会得到固定的输出。声明式和命令式并无高下之分，只是在云资源编排这一领域，声明式会比较方便实现。我们日常见到的云资源编排工具都是声明式的，包括AWS CloudFormation、Azure Resource Template、Google Cloud Deoplyment Manager。大家如果通过调用腾讯云API来在腾讯云上实施资源编排，那通常就是命令式的。
### 资源状态文件(state)
Terraform初始化以后，会生成一个状态文件，该状态文件记录了最近一次操作的时间、各资源的相关属性、各变量的当前值、状态文件的版本、等等。

下一次再操作的时候，terraform首先会把当前状态文件与云服务商上的状态进行一次更新，找出是否后有被删除或者更改了的资源，然后再根据.tf文件，决定那些资源需要删除、更新、创建。操作完成后，会重新生成一个状态文件。

### Terraform后台(backend)
资源状态文件的完整性比较重要，对于这些文件我们至少需要做到在操作开始时自动加锁，直到操作结束，这样别人无法更改；另外还需要对资源版本变更进行跟踪；对资源文件里敏感信息进行访问控制。

因此backend跟资源状态文件如何读取、存储、锁定，以及terraform apply如何执行严密相关。

terraform缺省使用本地后台，也就是说，状态文件会存放在当前目录下，terraform代码的执行也在本地虚拟机运行。这对一个人管理的云资源是没有问题的，但当团队人员数目加多以后，大家可能都有自己的工作台，但是需要一个共有的地方来存储资源状态文件。这是后就可以用到远程存储。目前terraform支持多种远程存储后台，包括AWS s3,Hashicorp Consul,etcd，Terraform云，以及terraform企业版等等，这些远程后台都提供在远程存储、锁定状态文件。其中terraform企业版提供远程运行terraform，以及其他一些企业级特性。

### Terraform模块(module)
Terraform模块就是把一些高度可重用的代码写成模块，方便其他人使用。模块由输入参数、输出参数以及主逻辑组成。这就跟传统编程语言里的函数很像。Terraform提供了公开的模块注册器，模块编写完成以后，只要符合规范，就可以发布到模块注册器中让大家使用。https://registry.terraform.io/

