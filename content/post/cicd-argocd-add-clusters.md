---
title: "ArgoCD添加多集群"
date: 2020-05-05T15:22:42+08:00
lastmod: 2020-05-05T17:22:42+08:00
draft: false
description: "ArgoCD添加多集群"
tags: ["argocd", "k8s"]
categories: ["cicd"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---


## 生成argocd管理用户token
登陆dashboard，settings-->Accounts-->admin-->Generate New
生成后，请记录下token信息，类似如下：

```yaml
fyJhbGciOiJ3UzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiI2OWI0M2M0Mi01MmZiLTRlZmItODIxOC0yOWU3NGM5MWI0NDIiLCJpYXQiOjE1OTUzMTEx3zQsImlzcyI6ImFyZ29jZCIsIm5iZiI6MTU5NTMxMTE3NCwic3ViIjoib3duZXIifQ.9u4XzArEeaz7G2Q2TWusnTkakEmq9BYDAUHr3dC6wG5
```

## 配置argocd config
对于开启了https认证的argocd在添加集群的时候比较鸡肋，需要登陆到server端POD里进行配置，具体如下：

```shell
# cat ~/.argocd/config
contexts:
- name: argocd-server.argocd
  server: qacd.test.cn
  user: argocd-server.argocd
current-context: argocd-server.argocd
servers:
- grpc-web-root-path: ""
  insecure: true
  server: qacd.test.cn
users:
- auth-token: xxxxxx #这里就是第一步生成token信息
  name: argocd-server.argocd
```

## 配置kubeconfig

具体配置这里忽略，请参考以往文档，前提要能访问集群并且是集群管理员，这里配置CONTEXT为idc-bj-k8s

## 添加集群

```shell
# argocd  --grpc-web cluster  add  idc-bj-k8s  --kubeconfig ~/.kube/config
INFO[0000] ServiceAccount "argocd-manager" already exists in namespace "kube-system"
INFO[0000] ClusterRole "argocd-manager-role" updated
INFO[0000] ClusterRoleBinding "argocd-manager-role-binding" updated
Cluster 'https://172.16.16.250:8443' added
# argocd --grpc-web cluster list
SERVER                          NAME        VERSION  STATUS      MESSAGE
https://172.16.16.250:8443      idc-bj-k8s  1.14     Successful
https://kubernetes.default.svc              1.14     Successful
```

目前看北京idc集群已经添加到argocd里,后边就可以往集群里部署应用啦啦

## 删除集群

```
# argocd --grpc-web  cluster rm https://172.16.16.250:8443
# argocd --grpc-web cluster list
SERVER                          NAME        VERSION  STATUS      MESSAGE
https://kubernetes.default.svc              1.14     Successful
```
