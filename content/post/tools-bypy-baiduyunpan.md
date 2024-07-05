---
title: "Linux下操作百度云盘的利器"
date: 2023-12-04T16:22:42+08:00
lastmod: 2023-12-04T16:23:42+08:00
draft: false
description: "Linux下操作百度云盘的利器bypy"
tags: ["bypy", "工具", "python"]
categories: ["tools"]              
author: "wanzi"                 

comment: true   # 关闭评论
toc: true       # 文章目录
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true	 # 关闭打赏
mathjax: true    # 打开 mathjax
---

最近在给公司SAAS产品在客户侧进行私有化部署，由于客户现场网络无法上网，只能通过Linux跳板机将数据传输过去，除了k8s离线部署程序、镜像外，还有几百G已经切好的视频数据，所以只能通过百度网盘传输了，试想一下能不能通过命令行同步百度云盘数据呢，谷歌一搜还真有，下面就简单介绍一下bypy的使用。

## bypy介绍
bypy是一个百度云/百度网盘的Python客户端，主要用于linux下操作百度云盘，提供文件列表、下载、上传、比较、向上同步、向下同步等操作，主要特点：支持Unicode/中文；失败重试；递归上传/下载；目录比较; 哈希缓存。

另外，由于百度PCS API权限限制，程序只能存取百度云端/apps/bypy目录下面的文件和目录。

## 安装bypy

安装pip
```
curl -O https://bootstrap.pypa.io/pip/2.7/get-pip.py
curl -O https://bootstrap.pypa.io/pip/3.6/get-pip.py
python get-pip.py 
```
安装bypy
```
pip install bypy
```

```
python -m bypy info #生成授权码，然后打开浏览器， 进行授权，并赋值授权码
```
将授权码复制粘贴到终端，回车，即完成。


## bypy命令详解
```
$ bypy help command #查看命令帮助
$ bypy info  #查看百度云盘空间
Quota: 13.294TB
Used: 1.917TB
$ bypy list #查看百度云盘“我的应用数据”
/apps/bypy ($t $f $s $m $d):
D course 0 2023-12-04, 15:21:41
D k8s 0 2023-12-04, 15:47:47

$ bypy downdir k8s #直接下载bypy目录下k8s
$ bypy syncup #把当前目录同步到云盘
$ bypy upload #把当前目录同步到云盘
$ bypy syncdown #把云盘内容同步到当前目录
$ bypy downdir / #把云盘内容同步到当前目录

$ bypy compare #比较本地当前目录和云盘（程序的）根目录
$ bypy -v #运行时添加-v参数，会显示进度详情。
$ bypy -d #运行时添加-d，会显示一些调试信息
$ bypy -ddd #显示更多http通讯信息

```
## 总结
这次对于使用bypy算告一段落，其实在调研的过程中也有很多基于baidupcs实现的工具，不过后来这些项目都不怎么维护了，所以也希望bypy能够坚持下去。

参考：https://github.com/houtianze/bypy
