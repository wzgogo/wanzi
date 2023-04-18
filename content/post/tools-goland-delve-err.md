---
title: "Delve版本过低无法适配Goland"
date: 2023-03-15T16:22:42+08:00
lastmod: 2023-03-15T17:22:42+08:00
draft: false
description: "Delve版本过低无法适配Goland"
tags: ["delve", "goland", "go"]
categories: ["tools"]              
author: "wanzi"                 

comment: true   # 关闭评论
toc: true       # 文章目录
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true	 # 关闭打赏
mathjax: true    # 打开 mathjax
---

## 问题

Goland下最近调试代码，总是报如下错误：

```yaml
WARNING: undefined behavior - version of Delve is too old for Go version 1.19.3 (maximum supported version 1.18)
```


## 解决方法

问题的大概的意思是Delve版本过低，无法满足当前的Golang版本


更新delve，由于我这里使用brew安装，默认官网安装文档里没有对brew过多说明，升级方法也没有，那么我们就直接安装即可。

```shell
git clone https://github.com/go-delve/delve
cd delve
go install github.com/go-delve/delve/cmd/dlv
```
或者按照指定版本：
```shell
# Install the latest release
go install github.com/go-delve/delve/cmd/dlv@latest
# Install 1.20.1
go install github.com/go-delve/delve/cmd/dlv@1.20.1
```

默认go install是安装在GOPATH下
```shell
# go env GOPATH
/Users/wanzi/go
# ./go/bin/dlv version
Delve Debugger
Version: 1.20.1
Build: $Id: 96e65b6c615845d42e0e31d903f6475b0e4ece6e $
```
卸载默认delve，并更新当前zsh里PATH：
```shell
brew uninstall delve
```

vim ~/.zshrc 更新如下：
```shell
export PATH="${PATH}:${HOME}/.krew/bin/:$HOME/go/bin"
```

参考：https://github.com/go-delve/delve/tree/master/Documentation/installation

