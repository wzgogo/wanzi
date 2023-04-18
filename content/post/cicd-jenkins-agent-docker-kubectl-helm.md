---
title: "自定义Jenkins Agent集成docker和kubectl工具"
date: 2023-03-22T18:22:42+08:00
lastmod: 2023-03-22T18:22:42+08:00
draft: false
description: "jenkins搭建"
tags: ["jenkins", "docker", "kubectl", "helm"]
categories: ["cicd"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

由于我们Jenkins构建的时候，官方提供的jenkins-agent没有我们使用的工具，比如helm，kubectl，curl，argocd等，因此，我们需要集成进来。


> 注意官方镜像名称有变化：
> jenkins/agent镜像，原名称为jenkins/slave，从4.3-2 开始更名为jenkins/agent
> jenkins/inbound-agent 镜像：原名称为 jenkins/jnlp-slave ，从 4.3-2 开始更名为 jenkins/inbound-agent

## Dockerfile

```yaml
FROM jenkins/inbound-agent:4.11-1-alpine-jdk11
 
USER root
 
ADD docker/docker /usr/bin/docker
ADD kubectl /usr/bin/kubectl
ADD helm /usr/bin/helm
 
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
RUN chmod +x /usr/bin/docker /usr/bin/kubectl /usr/bin/helm
RUN apk add curl
 
ENTRYPOINT ["/usr/local/bin/jenkins-agent"]
```

## 构建

```shell
docker build -t harbor.test.com/tools/jnlp-docker:4.11-1-alpine-jdk11 -f Dockerfile  .
```
