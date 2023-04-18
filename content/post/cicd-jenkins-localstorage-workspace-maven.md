---
title: "Jenkins Workspace和Maven仓库本地持久化存储"
date: 2023-03-22T16:22:42+08:00
lastmod: 2023-03-22T16:22:42+08:00
draft: false
description: "Jenkins Workspace和Maven仓库本地持久化存储"
tags: ["jenkins", "k8s", "maven"]
categories: ["cicd"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---


## 创建本地存储

由于是测试集群，我们这里直接使用本地volume存储即可；

由于这里jenkins server里运行用户为jenkins，且jenkins的uid为1000，我们需要在node1上提前将/opt/jenkins_agent/和/opt/jenkins_maven/权限授权给jenkins

```shell
chown 1000.1000 -R  /opt/jenkins_agent -R
chown 1000.1000 -R  /opt/jenkins_maven/ -R
```


本地存储：agent-pv-pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-agent-pv
spec:
  storageClassName: local # Local PV
  capacity:
    storage: 30Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  local:
    path: /opt/jenkins_agent
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node01
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-agent-pvc
  namespace: kube-ops
spec:
  storageClassName: local
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
```
本地存储：maven-pv-pvc.yaml
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-maven-pv
spec:
  storageClassName: local # Local PV
  capacity:
    storage: 30Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  local:
    path: /opt/jenkins_maven
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node01
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-maven-pvc
  namespace: kube-ops
spec:
  storageClassName: local
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
```

## 创建PV和PVC资源

```shell
# kubectl apply -f  agent-pv-pvc.yaml
persistentvolume/jenkins-agent-pv created
persistentvolumeclaim/jenkins-agent-pvc created

# kubectl apply -f  maven-pv-pvc.yaml
persistentvolume/jenkins-maven-pv created
persistentvolumeclaim/jenkins-maven-pvc created
```
