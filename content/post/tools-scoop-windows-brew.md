---
title: "推荐一个Windows包管理神器"
date: 2023-07-10T16:22:42+08:00
lastmod: 2023-07-10T16:23:42+08:00
draft: false
description: "Scoop安装Windows包，windows下scoop开源工具"
tags: ["scoop", "windows", "brew"]
categories: ["tools"]              
author: "wanzi"                 

comment: true   # 关闭评论
toc: true       # 文章目录
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true	 # 关闭打赏
mathjax: true    # 打开 mathjax
---

由于一直用Mac来办公，习惯了Mac下brew安装软件包的便利，最近公司电脑安装了windows系统，想着也折腾一番，如何才能像MAC下安装软件包那么丝滑呢，当然是今天推荐的主角Scoop了。

Scoop是一个开源项目，主要通过命令来安装windows软件包，可以很好避免权限弹窗，隐藏了GUI向导式安装，可以自动查找安装依赖，自动执行安装步骤。

官网地址：https://scoop.sh/

开源项目地址：https://github.com/ScoopInstaller/Scoop

另外,scoop包可以作为git仓库的一部分，通常叫buckets，可以在https://scoop.sh/#/buckets 这里查看到github上buckets，当然你也可以很方便创建自己的buckets。

## 环境准备
- Windows 10 专业版 19044.1586
- PowerShell最新版本或Windows PowerShell 5.1
- PowerShell执行策略必须是以下之一：Unrestricted，RemoteSigned或ByPass执行安装程序。

## 安装Scoop

从非管理员PowerShell运行此命令以使用默认配置安装 scoop，scoop 将安装到C:\Users\<YOUR USERNAME>\scoop

这里安装时已经提示，让更改策略：
```shell
PS C:\Users\wnote> iwr -useb get.scoop.sh | iex
Initializing...
PowerShell requires an execution policy in [Unrestricted, RemoteSigned, ByPass] to run Scoop. For example, to set the execution policy to 'RemoteSigned' please run 'Set-ExecutionPolicy RemoteSigned -Scope CurrentUser'.
Abort.
PS C:\Users\wnote> Set-ExecutionPolicy RemoteSigned -Scope CurrentUser                                                 
执行策略更改
执行策略可帮助你防止执行不信任的脚本。更改执行策略可能会产生安全风险，如 https:/go.microsoft.com/fwlink/?LinkID=135170
中的 about_Execution_Policies 帮助主题所述。是否要更改执行策略?
[Y] 是(Y)  [A] 全是(A)  [N] 否(N)  [L] 全否(L)  [S] 暂停(S)  [?] 帮助 (默认值为“N”): Y
PS C:\Users\tarena> iwr -useb get.scoop.sh | iex
Initializing...
Downloading ...
Creating shim...
Adding ~\scoop\shims to your path.
Scoop was installed successfully!
Type 'scoop help' for instructions.

PS C:\Users\tarena> scoop update
Updating Scoop...
Updating 'main' bucket...
Scoop was updated successfully!
```

当然，如果你本身网络无法连接官方地址，可以增加代理方式：
```shell
irm get.scoop.sh -Proxy 'http://<ip:port>' | iex
```
另外，对于不想安装默认路径的，可以通过如下方式指定,install.ps1参考https://github.com/ScoopInstaller/Install/blob/master/install.ps1：

```shell
.\install.ps1 -ScoopDir 'D:\Applications\Scoop' -ScoopGlobalDir 'F:\GlobalScoopApps' -NoProxy
```

## 更换国内源
在网上发现几个国内的源可以使用：
https://gitee.com/squallliu/scoop
https://gitee.com/glsnames/scoop-installer
```shell
iwr -useb https://gitee.com/glsnames/scoop-installer/raw/master/bin/install.ps1 | iex
scoop config SCOOP_REPO https://gitee.com/glsnames/scoop-installer
scoop update
```

## 安装软件包
```shell
scoop install kubectl 
scoop install helm 
scoop install wget 
scoop install curl 
scoop install 7zip 
scoop install aria2 
scoop install sudo 
scoop install -g extras/windows-terminal
```

如果安装卡住，建议配置代理
```shell
scoop config proxy 127.0.0.1:18080 #设置http代理
scoop config rm proxy #删除代理
```

## 配置多线程下载神器aria2
```shell
scoop config aria2-enabled false
scoop config aria2-warning-enabled false
scoop config aria2-retry-wait 4
scoop config aria2-split 16
scoop config aria2-max-connection-per-server 16
scoop config aria2-min-split-size 4M
```

## scoop常用命令

```shell
scoop install "app name" #安装软件包
scoop list #查看已经安装软件
scoop status #查看更新状态
scoop config #查看配置
```

参考：https://github.com/ScoopInstaller/Install
