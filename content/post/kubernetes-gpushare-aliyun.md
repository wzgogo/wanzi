---
title: "阿里云共享GPU方案测试"
date: 2021-08-31T14:22:42+08:00
lastmod: 2021-08-31T14:22:42+08:00
draft: false
description: "阿里云共享GPU方案测试"
tags: ["gpushare", "aliyun", "gpu"]
categories: ["kubernetes"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---


## 一、k8s部署GPU共享插件

部署之前需要确保k8s节点上已安装nvidia-driver和nvidia-docker，同时已将docker默认运行时设置为nvidia

```shell
# cat /etc/docker/daemon.json
{
  "runtimes": {
    "nvidia": {
      "path": "/usr/bin/nvidia-container-runtime",
        "runtimeArgs": []
      }
  },
  "default-runtime": "nvidia",
}
```


### 1、helm安装gpushare-device-plugin

```shell
$ git clone https://github.com/AliyunContainerService/gpushare-scheduler-extender.git
$ cd gpushare-scheduler-extender/deployer/chart
$ helm install --name gpushare --namespace kube-system  --set masterCount=3 gpushare-installer
```


### 2、打Label，标记GPU节点

```shell
$ kubectl label node sd-cluster-04 gpushare=true
$ kubectl label node sd-cluster-05 gpushare=true
```

### 3、安装kubectl-inspect-gpushare

需要提前安装kubectl，这里就省略
```shell
$ cd /usr/bin/
$ wget https://github.com/AliyunContainerService/gpushare-device-plugin/releases/download/v0.3.0/kubectl-inspect-gpushare
$ chmod u+x /usr/bin/kubectl-inspect-gpushare
```
查看当前k8s集群GPU资源使用情况
```shell
$ kubectl inspect gpushare
NAME           IPADDRESS      GPU0(Allocated/Total)  GPU Memory(GiB)
sd-cluster-04  192.168.1.214  0/14                   0/14
sd-cluster-05  192.168.1.215  8/14                   8/14
----------------------------------------------------------
Allocated/Total GPU Memory In Cluster:
8/28 (28%)
```

当然要禁用GPU节点共享GPU资源，直接设置gpushare=false即可
```shell
$ kubectl label node sd-cluster-04 gpushare=false
$ kubectl label node sd-cluster-05 gpushare=false
```

## 二、验证测试

### 1、部署第一个应用
申请2G GPU显存，测试应用，应该分配到一张卡上

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
            aliyun.com/gpu-mem: 2
```

```shell
$ kubectl apply -f 1.yaml -n test
$ kubectl inspect gpushare
NAME           IPADDRESS      GPU0(Allocated/Total)  GPU Memory(GiB)
sd-cluster-04  192.168.1.214  0/14                   0/14
sd-cluster-05  192.168.1.215  2/14                   2/14
----------------------------------------------------------
Allocated/Total GPU Memory In Cluster:
2/28 (7%)
$ kubectl get pod -n test
NAME                               READY   STATUS    RESTARTS   AGE
binpack-1-6d6955c487-j4c4b         1/1     Running   0          28m
$ kubectl logs -f binpack-1-6d6955c487-j4c4b -n test
ALIYUN_COM_GPU_MEM_DEV=14
ALIYUN_COM_GPU_MEM_CONTAINER=2
2021-08-13 02:47:22.395557: I tensorflow/core/platform/cpu_feature_guard.cc:137] Your CPU supports instructions that this TensorFlow binary was not compiled to use: SSE4.1 SSE4.2 AVX AVX2 AVX512F FMA
2021-08-13 02:47:22.552831: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1030] Found device 0 with properties:
name: Tesla T4 major: 7 minor: 5 memoryClockRate(GHz): 1.59
pciBusID: 0000:af:00.0
totalMemory: 14.75GiB freeMemory: 14.65GiB
2021-08-13 02:47:22.552873: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1120] Creating TensorFlow device (/device:GPU:0) -> (device: 0, name: Tesla T4, pci bus id: 0000:af:00.0, compute capability: 7.5)
```

### 2、部署第二个应用

现在申请8G内存，2个实例，总共16G内存
```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: binpack-2
  labels:
    app: binpack-2

spec:
  replicas: 2

  selector: # define how the deployment finds the pods it mangages
    matchLabels:
      app: binpack-2

  template: # define the pods specifications
    metadata:
      labels:
        app: binpack-2

    spec:
      containers:
      - name: binpack-2
        image: cheyang/gpu-player:v2
        resources:
          limits:
            aliyun.com/gpu-mem: 8
```

```shell
$ kubectl apply -f 2.yaml -n test
$ kubectl inspect gpushare
NAME           IPADDRESS      GPU0(Allocated/Total)  GPU Memory(GiB)
sd-cluster-04  192.168.1.214  8/14                   8/14
sd-cluster-05  192.168.1.215  10/14                  10/14
----------------------------------------------------------
Allocated/Total GPU Memory In Cluster:
18/28 (64%)
$ kubectl get pod -n test
NAME                               READY   STATUS    RESTARTS   AGE
binpack-1-6d6955c487-j4c4b         1/1     Running   0          28m
binpack-2-58579b95f7-4wpbl         1/1     Running   0          27m
$ kubectl logs -f binpack-2-58579b95f7-4wpbl -n test
ALIYUN_COM_GPU_MEM_DEV=14
ALIYUN_COM_GPU_MEM_CONTAINER=8
2021-08-13 02:48:41.246585: I tensorflow/core/platform/cpu_feature_guard.cc:137] Your CPU supports instructions that this TensorFlow binary was not compiled to use: SSE4.1 SSE4.2 AVX AVX2 AVX512F FMA
2021-08-13 02:48:41.338992: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1030] Found device 0 with properties:
name: Tesla T4 major: 7 minor: 5 memoryClockRate(GHz): 1.59
pciBusID: 0000:af:00.0
totalMemory: 14.75GiB freeMemory: 13.07GiB
2021-08-13 02:48:41.339031: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1120] Creating TensorFlow device (/device:GPU:0) -> (device: 0, name: Tesla T4, pci bus id: 0000:af:00.0, compute capability: 7.5)
```

通过资源占用可以看到，第二个应用，已经分别使用了2张卡的8G内存

### 3、部署第三个应用

申请2G显存
```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: binpack-3
  labels:
    app: binpack-3

spec:
  replicas: 1

  selector: # define how the deployment finds the pods it mangages
    matchLabels:
      app: binpack-3

  template: # define the pods specifications
    metadata:
      labels:
        app: binpack-3

    spec:
      containers:
      - name: binpack-3
        image: cheyang/gpu-player:v2
        resources:
          limits:
            aliyun.com/gpu-mem: 2
```

```shell
$ kubectl apply -f 3.yaml -n test
$ kubectl inspect gpushare
NAME           IPADDRESS      GPU0(Allocated/Total)  GPU Memory(GiB)
sd-cluster-04  192.168.1.214  8/14                   8/14
sd-cluster-05  192.168.1.215  12/14                  12/14
----------------------------------------------------------
Allocated/Total GPU Memory In Cluster:
20/28 (71%)
$ kubectl get pod -n test
NAME                               READY   STATUS    RESTARTS   AGE
binpack-1-6d6955c487-j4c4b         1/1     Running   0          28m
binpack-2-58579b95f7-4wpbl         1/1     Running   0          27m
binpack-2-58579b95f7-sjhwt         1/1     Running   0          27m
binpack-3-556bbd84f9-9xqg7         1/1     Running   0          14m
$ kubectl logs -f binpack-3-556bbd84f9-9xqg7 -n test
ALIYUN_COM_GPU_MEM_DEV=14
ALIYUN_COM_GPU_MEM_CONTAINER=2
2021-08-13 03:01:53.897423: I tensorflow/core/platform/cpu_feature_guard.cc:137] Your CPU supports instructions that this TensorFlow binary was not compiled to use: SSE4.1 SSE4.2 AVX AVX2 AVX512F FMA
2021-08-13 03:01:54.008665: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1030] Found device 0 with properties:
name: Tesla T4 major: 7 minor: 5 memoryClockRate(GHz): 1.59
pciBusID: 0000:af:00.0
totalMemory: 14.75GiB freeMemory: 7.08GiB
2021-08-13 03:01:54.008716: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1120] Creating TensorFlow device (/device:GPU:0) -> (device: 0, name: Tesla T4, pci bus id: 0000:af:00.0, compute capability: 7.5)

```
部署完第三个应用以后，目前可用最大显存为8G，但是实际我们部署应用的时候最多只能使用6G，因为同一个任务不能分布式的跑到不同的GPU卡上


### 4、部署第四个应用

申请5G显存，应该是调度到sd-cluster-04上
```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: binpack-4
  labels:
    app: binpack-4

spec:
  replicas: 1

  selector: # define how the deployment finds the pods it mangages
    matchLabels:
      app: binpack-4

  template: # define the pods specifications
    metadata:
      labels:
        app: binpack-4

    spec:
      containers:
      - name: binpack-4
        image: cheyang/gpu-player:v2
        resources:
          limits:
            aliyun.com/gpu-mem: 5
```

```shell
$ kubectl apply -f 4.yaml -n test
$ kubectl inspect gpushare
NAME           IPADDRESS      GPU0(Allocated/Total)  GPU Memory(GiB)
sd-cluster-04  192.168.1.214  13/14                  13/14
sd-cluster-05  192.168.1.215  12/14                  12/14
------------------------------------------------------------
Allocated/Total GPU Memory In Cluster:
25/28 (89%)
$ kubectl get pod -n test
NAME                               READY   STATUS    RESTARTS   AGE
binpack-1-6d6955c487-j4c4b         1/1     Running   0          26m
binpack-2-58579b95f7-4wpbl         1/1     Running   0          24m
binpack-2-58579b95f7-sjhwt         1/1     Running   0          24m
binpack-3-556bbd84f9-9xqg7         1/1     Running   0          11m
binpack-4-6956458f85-cv62j         1/1     Running   0          6s
$ kubectl logs -f binpack-4-6956458f85-cv62j -n test
ALIYUN_COM_GPU_MEM_DEV=14
ALIYUN_COM_GPU_MEM_CONTAINER=5
2021-08-13 03:13:20.208122: I tensorflow/core/platform/cpu_feature_guard.cc:137] Your CPU supports instructions that this TensorFlow binary was not compiled to use: SSE4.1 SSE4.2 AVX AVX2 AVX512F FMA
2021-08-13 03:13:20.361391: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1030] Found device 0 with properties:
name: Tesla T4 major: 7 minor: 5 memoryClockRate(GHz): 1.59
pciBusID: 0000:af:00.0
totalMemory: 14.75GiB freeMemory: 6.46GiB
2021-08-13 03:13:20.361481: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1120] Creating TensorFlow device (/device:GPU:0) -> (device: 0, name: Tesla T4, pci bus id: 0000:af:00.0, compute capability: 7.5)
```

## 三、总结
gpushare-device-plugin使用中有如下需求不能满足

- 同一个任务没法同时使用多台机器的GPU卡（共享）
- 同一张卡下没法通过GPU负载百分比来分配资源

不过对于算法团队模型测试已经完全够用了，对于GPU共享方案，其实还有另外两种，这里我就不介绍了，有需要的直接参考官方仓库配置即可。

https://github.com/tkestack/gpu-manager
https://github.com/vmware/bitfusion-with-kubernetes-integration

参考资料：
https://github.com/AliyunContainerService/gpushare-scheduler-extender/tree/master/deployer
