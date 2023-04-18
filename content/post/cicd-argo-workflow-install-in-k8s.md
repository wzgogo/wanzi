---
title: "Argo Worflow实践一安装部署"
date: 2021-05-07T16:22:42+08:00
lastmod: 2021-05-07T16:22:42+08:00
draft: false
description: "Argo Worflow实践一安装部署"
tags: ["argo", "argo workflow", "workflow"]
categories: ["cicd"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

## 简介&架构
Argo Workflows是一个开源容器级别工作流引擎，用于在Kubernetes上协调并行作业。 Argo Workflows通过抽象Kubernetes CRD（自定义资源定义）来实现整个架构功能，比如Workflow Template、Workflow、Cron Workflow。

Argo workflow能做什么？
- 定义工作流，工作流中的每个步骤都是一个容器。
- 将多步骤工作流建模为一系列任务，或者使用有向无环图（DAG）捕获任务之间的依存关系。
- 使用Kubernetes上的ArgoWorkflow，可以在短时间内轻松运行用于计算机学习或数据处理的计算密集型作业。
- 无需配置复杂的软件开发产品，即可在Kubernetes上本地运行CI / CD管道。

Argo workflow有哪些功能?
- Workflow ：调用多个工作流模版进行任务编排，通过不同顺序来执行，
- Workflow Template：Workflow的模版，是对workflow的一种定义，因此workflow template内部或者集群其他workflow和workflow template都可以调用。
- Cluster Workflow Template：集群级别Workflow Template，通过clusterrole角色授权可以访问集群所有namespace
- Cron Wrokflow：任务计划类型工作流，相当于高级版的k8s cronjob。

## 安装配置
### 安装argo workflow
这里我们安装的稳定版本2.12.10，整个安装过程会配置service account、role、ClusterRole、deployment等

```shell
kubectl create ns argo
kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo/v2.12.10/manifests/install.yaml
```


### 设置Ingress集群外部访问
由于我们的集群环境ingress controller采用的是traefik，而argo workflow默认内部访问的方式是通过https访问，因此我这里只有添加相关注解(annotations)才能转发请求到argo-server

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/redirect-entry-point: https
  name: argo-server
  namespace: argo
spec:
  rules:
  - host: argo.test.cn
    http:
      paths:
      - backend:
          serviceName: argo-server
          servicePort: web
        path: /
status:
  loadBalancer: {}
```

集群外部访问：https://argo.test.cn 

## workflow简单测试
这里我们以hello world为例子测试：
```shell
# argo submit -n argo --watch https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml
Name:                hello-world-4mffd
Namespace:           argo
ServiceAccount:      default
Status:              Pending
Created:             Fri May 07 16:11:25 +0800 (now)
Name:                hello-world-4mffd
Namespace:           argo
ServiceAccount:      default
Status:              Pending
Created:             Fri May 07 16:11:25 +0800 (now)
Name:                hello-world-4mffd
Namespace:           argo
ServiceAccount:      default
Status:              Running
Created:             Fri May 07 16:11:25 +0800 (now)
Started:             Fri May 07 16:11:25 +0800 (now)
Duration:            0 seconds

STEP                  TEMPLATE  PODNAME            DURATION  MESSAGE
 ◷ hello-world-4mffd  whalesay  hello-world-4mffd  0s
Name:                hello-world-4mffd
Namespace:           argo
ServiceAccount:      default
Status:              Running
Created:             Fri May 07 16:11:25 +0800 (10 seconds ago)
Started:             Fri May 07 16:11:25 +0800 (10 seconds ago)
Duration:            10 seconds

STEP                  TEMPLATE  PODNAME            DURATION  MESSAGE
 ◷ hello-world-4mffd  whalesay  hello-world-4mffd  10s       ContainerCreating
Name:                hello-world-4mffd
Namespace:           argo
ServiceAccount:      default
Status:              Succeeded
Conditions:
 Completed           True
Created:             Fri May 07 16:11:25 +0800 (2 minutes ago)
Started:             Fri May 07 16:11:25 +0800 (2 minutes ago)
Finished:            Fri May 07 16:14:18 +0800 (now)
Duration:            2 minutes 53 seconds
ResourcesDuration:   1m18s*cpu,1m18s*memory

STEP                  TEMPLATE  PODNAME            DURATION  MESSAGE
 ✔ hello-world-4mffd  whalesay  hello-world-4mffd  2m

# argo list -n argo
NAME                STATUS      AGE   DURATION   PRIORITY
hello-world-4mffd   Succeeded   3m    2m         0

# argo logs -f hello-world-4mffd -n argo
hello-world-4mffd:  _____________
hello-world-4mffd: < hello world >
hello-world-4mffd:  -------------
hello-world-4mffd:     \
hello-world-4mffd:      \
hello-world-4mffd:       \
hello-world-4mffd:                     ##        .
hello-world-4mffd:               ## ## ##       ==
hello-world-4mffd:            ## ## ## ##      ===
hello-world-4mffd:        /""""""""""""""""___/ ===
hello-world-4mffd:   ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
hello-world-4mffd:        \______ o          __/
hello-world-4mffd:         \    \        __/
hello-world-4mffd:           \____\______/
```
