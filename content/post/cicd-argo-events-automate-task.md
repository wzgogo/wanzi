---
title: "Argo Events入门实践"
date: 2021-07-07T16:22:42+08:00
lastmod: 2021-07-07T16:22:42+08:00
draft: false
description: "Argo Events入门实践"
tags: ["argo", "argoevents", "workflow"]
categories: ["cicd"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---



前面我们介绍了Argo Workflow如何安装与触发任务，这一篇主要介绍一个新工具：

## ArgoEvents是什么？
Argo Events是一个事件驱动的 Kubernetes 工作流自动化框架。它支持20 多种不同的事件（例如 webhook、S3 drop、cronjob、消息队列-例如 Kafka、GCP PubSub、SNS、 SQS等）

### 特性：
- 支持来自[20多个事件源](https://argoproj.github.io/argo-events/concepts/event_source/)和10多个[触发器的事件](https://argoproj.github.io/argo-events/concepts/trigger/)。
- 能够为工作流自动化定制业务级约束逻辑。
- 管理从简单、线性、实时到复杂、多源事件的所有内容。
- 符合[CloudEvents](https://cloudevents.io/)。

### 组件：
- EventSource(类似Gateway,把消息发送给eventbus)
- EventBus(事件消息队列，基于高性能分布式消息中间件NATS实现，不过看NATS官网到2023年后不再维护了，估计后期架构也会调整)
- EventSensor(订阅消息队列，事件参数化并对事件过滤)

## ArgoEvents部署安装

### argo-events部署：

```shell
kubectl create ns argo-events
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/v1.2.3/manifests/install.yaml
```

### argo-eventbus部署:

```shell
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/eventbus/native.yaml
```


## RBAC账户授权
### 创建operate-workflow-sa账户
授权operate-workflow-sa可以在argo-events namespace下创建argo workflow任务，这个对于后边EventSensor自动化创建workflow会用到。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: argo-events
  name: operate-workflow-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: operate-workflow-role
  namespace: argo-events
rules:
  - apiGroups:
      - argoproj.io
    verbs:
      - "*"
    resources:
      - workflows
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: operate-workflow-role-binding
  namespace: argo-events
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: operate-workflow-role
subjects:
  - kind: ServiceAccount
    name: operate-workflow-sa
    namespace: argo-events
```
### 创建workflow-pods-sa账户
授权workflow-pods-sa可以在argo-events下通过workflow自动创建pod
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: argo-events
  name: workflow-pods-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: workflow-pods-role
rules:
  - apiGroups:
      - ""
    verbs:
      - "*"
    resources:
      - pods
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: workflow-pods-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: workflow-pods-role
subjects:
  - kind: ServiceAccount
    name: workflow-pods-sa
    namespace: argo-events
```

## ArgoEvents自动化触发任务

### 启动一个event-sources接受请求：

```shell
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/event-sources/webhook.yaml
```

注意：这里event-sources里name为example，对于实际生产环境下边创建sensor需要指定这里的名称

### 创建 webhook方式sensor消费请求：
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: webhook
spec:
  template:
    serviceAccountName: operate-workflow-sa
  dependencies:
    - name: test-dep
      eventSourceName: webhook
      eventName: example
  triggers:
    - template:
        name: webhook-workflow-trigger
        k8s:
          group: argoproj.io
          version: v1alpha1
          resource: workflows
          operation: create
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: webhook-
              spec:
                serviceAccountName: workflow-pods-sa
                entrypoint: whalesay
                arguments:
                  parameters:
                  - name: message
                    # the value will get overridden by event payload from test-dep
                    value: hello world
                templates:
                - name: whalesay
                  inputs:
                    parameters:
                    - name: message
                  container:
                    image: docker/whalesay:latest
                    command: [cowsay]
                    args: ["{{inputs.parameters.message}}"]
          parameters:
            - src:
                dependencyName: test-dep
              dest: spec.arguments.parameters.0.value
```
参考：https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/sensors/webhook.yaml

### forward本地请求到远端：
```shell
kubectl -n argo-events port-forward <name-of-event-source-pod>  <local port>:12000
```
### 往event-sources筛数据：
```shell
curl -d '{"message":"ok"}' -H "Content-Type: application/json" -X POST http://localhost:12000/example
```

至此，通过ArgoEvents就可以自动化创建workflow任务了

