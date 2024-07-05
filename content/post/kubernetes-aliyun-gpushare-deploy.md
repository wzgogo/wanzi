---
title: "利用阿里云开源方案实现单GPU卡多人使用"
date: 2023-06-30T18:10:42+08:00
lastmod: 2023-06-30T18:10:42+08:00
draft: false
description: "利用阿里云开源方案实现单GPU卡多人使用"
tags: ["gpu", "k8s", "aliyun", "gpushare-device-plugin"]
categories: ["kubernetes"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---


最近新上了一个AI项目，主要是给大学提供AI在线实验，项目也采购了GPU服务器，但是只有一张Nvidia Tesla T4卡，需要支持多个学生同时在线做实验呢。

在线实验系统目前运行在Kubernetes上，因此，需要考虑k8s环境下，GPU共享，之前也测试过阿里云GPU卡共享方案，这里就直接记录使用步骤即可：

> kubernetes集群版本：1.23.1 

## 调整K8S调度器

从1.23.+起，由于kube-scheduler调度策略有调整，所以之前的部署方式就不好时了。

参考这里：https://kubernetes.io/docs/reference/scheduling/policies/

```shell
git clone https://github.com/AliyunContainerService/gpushare-scheduler-extender.git
cp config/scheduler-policy-config.yaml /etc/kubernetes/
```

由于我这里是通过kubeasz安装的k8s，所以Kubeconfig放在这里/etc/kubernetes/kubelet.kubeconfig，需要更新 /etc/kubernetes/scheduler-policy-config.yaml如下：

```yaml
---
apiVersion: kubescheduler.config.k8s.io/v1beta2
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: /etc/kubernetes/kube-scheduler.kubeconfig
extenders:
- urlPrefix: "http://192.168.233.101:32766/gpushare-scheduler"
  filterVerb: filter
  bindVerb: bind
  enableHTTPS: false
  nodeCacheCapable: true
  managedResources:
  - name: aliyun.com/gpu-mem
    ignoredByScheduler: false
  ignorable: false
```

修改Kube-scheduler启动启动参数，/etc/systemd/system/kube-scheduler.service

```yaml
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kube/bin/kube-scheduler \
  --authentication-kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \
  --authorization-kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \
  --bind-address=0.0.0.0 \
  --kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \
  --config=/etc/kubernetes/scheduler-policy-config.yaml \
  --leader-elect=true \
  --v=2
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
```shell
systemctl  daemon-reload
systemctl  restart  kube-scheduler.service
```

## 安装调度器扩展器

给GPU机器打上label:

```
kubectl label node mynode gpushare=true
```

```shell
cd gpushare-scheduler-extender/config
kubectl apply -f gpushare-schd-extender.yaml 
```

## 安装插件

```shell
kubectl create -f https://raw.githubusercontent.com/AliyunContainerService/gpushare-device-plugin/master/device-plugin-rbac.yaml
kubectl create -f https://raw.githubusercontent.com/AliyunContainerService/gpushare-device-plugin/master/device-plugin-ds.yaml
```
安装完成以后如下

```shell
# kubectl get pod   -n kube-system |grep gpushare
gpushare-device-plugin-ds-j8blj              1/1     Running   0                30h
gpushare-schd-extender-74796c5f64-7g4bl      1/1     Running   0                29h

```



## 测试GPU
修改samples/1.yaml如下：
```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: binpack-1
  labels:
    app: binpack-1

spec:
  replicas: 1

  selector: # define how the deployment finds the pods it mangages
    matchLabels:
      app: binpack-1

  template: # define the pods specifications
    metadata:
      labels:
        app: binpack-1

    spec:
      containers:
      - name: binpack-1
        image: cheyang/gpu-player:v2
        resources:
          limits:
            # GiB
            aliyun.com/gpu-count: 1
            aliyun.com/gpu-mem: 5
```

```shell
# kubectl apply -f  samples/1.yaml 
deployment.apps/binpack-1 created
# kubectl get pod  |grep binpack
binpack-1-9995bdf69-pk2d4                 1/1     Running   0             12s
# kubectl logs -f binpack-1-9995bdf69-pk2d4 
ALIYUN_COM_GPU_MEM_DEV=14
ALIYUN_COM_GPU_MEM_CONTAINER=5
2023-06-30 09:40:50.890296: I tensorflow/core/platform/cpu_feature_guard.cc:137] Your CPU supports instructions that this TensorFlow binary was not compiled to use: SSE4.1 SSE4.2 AVX AVX2 FMA
2023-06-30 09:40:50.976283: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1030] Found device 0 with properties: 
name: Tesla T4 major: 7 minor: 5 memoryClockRate(GHz): 1.59
pciBusID: 0000:03:00.0
totalMemory: 14.75GiB freeMemory: 14.66GiB
2023-06-30 09:40:50.976313: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1120] Creating TensorFlow device (/device:GPU:0) -> (device: 0, name: Tesla T4, pci bus id: 0000:03:00.0, compute capability: 7.5)
```
通过日志看，GPU共享已经成功，且申请了5G显存。

当然你也可以通过kubectl扩展工具kubectl-inspect-gpushare来查看:

```
# cd /usr/bin/
# wget https://github.com/AliyunContainerService/gpushare-device-plugin/releases/download/v0.3.0/kubectl-inspect-gpushare
# chmod u+x /usr/bin/kubectl-inspect-gpushare
# kubectl-inspect-gpushare 
NAME             IPADDRESS        GPU0(Allocated/Total)  GPU Memory(GiB)
192.168.233.101  192.168.233.101  9/14                   9/14
--------------------------------------------------------------
Allocated/Total GPU Memory In Cluster:
9/14 (64%) 
```

至此，GPU卡共享已经完成，剩下就是通过tensorflow来调度我的GPU卡了。

参考文档：

https://github.com/AliyunContainerService/gpushare-scheduler-extender/blob/master/docs/install.md

