---
title: "基于Docker-compose搭建jenkins"
date: 2019-11-11T16:22:42+08:00
lastmod: 2019-11-11T16:22:42+08:00
draft: false
description: "jenkins搭建"
tags: ["jenkins", "ci"]
categories: ["cicd"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

## docker-compose配置

```yaml
version: '2'
 
services:
  jenkins:
    image: jenkins/jenkins:latest
    restart: always
    environment:
      JAVA_OPTS: "-Dorg.apache.commons.jelly.tags.fmt.timeZone=Asia/Shanghai -Djava.awt.headless=true -Dmail.smtp.starttls.enable=true"
    ports:
      - "80:8080"
      - "50000:50000"
    volumes:
      - '/ssd/jenkins:/var/jenkins_home'
      - '/var/run/docker.sock:/var/run/docker.sock'
      - '/etc/localtime:/etc/localtime:ro'
    dns: 223.5.5.5
    networks:
      - extnetwork
networks:
   extnetwork:
      ipam:
         config:
         - subnet: 172.255.0.0/16
```

## 启动服务
```shell
docker-compose  up -d 
```
