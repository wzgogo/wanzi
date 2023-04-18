---
title: "Dockerfile多阶段构建"
date: 2018-07-02T16:22:42+08:00
lastmod: 2018-07-02T16:22:42+08:00
draft: false
description: "docker多阶段构建"
tags: ["docker", "dockerfile"]
categories: ["linux"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

Docker多阶段构建理解:

* 构建镜像需要有一个基础镜像,后续操作就会基于该基础镜像构建
* docker镜像文件里有层级概念,每执行一次RUN指令,镜像就会多一层,所以通过减少层级来减少镜像大小
* 多个from的时候,只有最后一个from的镜像才是镜像的根镜像

自己项目部署中多阶段构建示例, 这里基于golang基础镜像编译以后的二进制直接copy到基于alpine构建的最小镜像里:
```yaml
FROM golang:1.12.7 as build
MAINTAINER wanzi <iwz2099@163.com>
 
# 编译配置相关
ARG NAME=gaia
ARG FLAGS=-tags=jsoniter
ARG GOOS=linux
ARG GOARCH=amd64
ARG PORT_TO_EXPOSE=10020
 
ENV GOPROXY https://mirrors.aliyun.com/goproxy/
ENV GO111MODULE on
 
WORKDIR /opt/gaia
COPY . .
RUN GOOS=$GOOS GOARCH=$GOARCH go build -mod vendor -ldflags="-s -w" -o $NAME $FLAGS
 
FROM alpine
WORKDIR /opt/gaia
COPY --from=build /opt/gaia/gaia .
RUN mkdir -p /opt/gaia/conf
VOLUME ["/opt/gaia/conf"]
 
CMD ["./gaia"]
EXPOSE $PORT_TO_EXPOSE
```
