---
title: "kind构建轻量级kubernetes集群"
date: 2023-04-13T18:22:42+08:00
lastmod: 2023-04-13T18:22:42+08:00
draft: false
description: "kind构建轻量级kubernetes集群"
tags: ["kind", "k8s", "docker"]
categories: ["kubernetes"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

最近在复习Golang，写了一个web应用，本地跑通以后想在k8s集群测试一番，苦于机器配置原因，
搭建一套完整的k8s集群还是有一些力不从心，记得之前有朋友说过docker里也可以跑k8s，于是这就折腾了一番。

今天的主角是kind,那么kind是什么呢？kind可以用来干什么呢？

## 一、kind介绍

kind是Kubernetes In Docker缩写，一个使用docker容器节点来运行本地的kubernetes集群工具，kind主要用于测试kubernetes本身，
适用于本地开发或者CI。

对于kind目前几乎支持kubernetes官方所有版本，对于运行时目前仅支持docker，未来会逐步支持CRI里常用运行时，
相信未来一些日子，会慢慢支持conterned。

其实，kind管理集群过程，是在使用kubeadm、kustomize等开源组件来进行工作。

废话不多说，下面我们就快速安装使用kind。

## 二、kind安装
由于我系统是mac osx所以，我这里直接使用brew安装：
```Shell
# brew install kind
==> Downloading https://mirrors.ustc.edu.cn/homebrew-bottles/kind-0.17.0.arm64_monterey.bottle.tar.gz
Already downloaded: /Users/wanzi/Library/Caches/Homebrew/downloads/bcd419997297730492f5cebc36be86fb51f21061a6ddb2e066e1e4d8ad33ddf3--kind-0.17.0.arm64_monterey.bottle.tar.gz
==> Pouring kind-0.17.0.arm64_monterey.bottle.tar.gz
==> Caveats
zsh completions have been installed to:
  /opt/homebrew/share/zsh/site-functions
==> Summary
🍺  /opt/homebrew/Cellar/kind/0.17.0: 8 files, 8.7MB
==> Running `brew cleanup kind`...
Disable this behaviour by setting HOMEBREW_NO_INSTALL_CLEANUP.
Hide these hints with HOMEBREW_NO_ENV_HINTS (see `man brew`).
# kind version
kind v0.17.0 go1.19.2 darwin/arm64
```

对于linux安装：
```Shell
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

当然，如果你是windows，也可以使用Chocolatey安装；或者使用二进制方式，这些官方都支持。

## 三、创建第一个集群
在操作之前，docker和kubectl是我们需要提前准备，具体安装我这里就先忽略。

```Shell
# kind help
kind creates and manages local Kubernetes clusters using Docker container 'nodes'

Usage:
  kind [command]

Available Commands:
  build       Build one of [node-image]
  completion  Output shell completion code for the specified shell (bash, zsh or fish)
  create      Creates one of [cluster]
  delete      Deletes one of [cluster]
  export      Exports one of [kubeconfig, logs]
  get         Gets one of [clusters, nodes, kubeconfig]
  help        Help about any command
  load        Loads images into nodes
  version     Prints the kind CLI version

Flags:
  -h, --help              help for kind
      --loglevel string   DEPRECATED: see -v instead
  -q, --quiet             silence all stderr output
  -v, --verbosity int32   info log verbosity, higher value produces more output

Use "kind [command] --help" for more information about a command.
```

对于kind help命令，我们知道kind支持build,create,delete,get,load等操作，
对于子命令可以通过-h获取更多讯息。

kind创建一个集群，
在创建集群之前我们需要确保~/.kube目录存在，因为创建过程中，kind会将集群API地址、证书等信息写入kubeconfig。
```Shell
# kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a nice day! 👋

# kubectl get node
NAME                 STATUS   ROLES           AGE     VERSION
kind-control-plane   Ready    control-plane   9m21s   v1.25.3
# kubectl get pods -A
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
kube-system          coredns-565d847f94-kp6l7                     1/1     Running   0          9m15s
kube-system          coredns-565d847f94-ml28b                     1/1     Running   0          9m15s
kube-system          etcd-kind-control-plane                      1/1     Running   0          9m32s
kube-system          kindnet-hfsc8                                1/1     Running   0          9m15s
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          9m31s
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          9m33s
kube-system          kube-proxy-m9zp5                             1/1     Running   0          9m15s
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          9m33s
local-path-storage   local-path-provisioner-684f458cdd-d7f29      1/1     Running   0          9m15s
```
这样我们第一个集群就用kind创建完成。


## 四、集群操作：
### 1、kind自定义集群名称：
```Shell
# kind create cluster --name ci-cluster
Creating cluster "ci-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-ci-cluster"
You can now use your cluster with:
```

### 2、切换集群：
```Shell
# kubectl cluster-info --context kind-ci-cluster
Kubernetes control plane is running at https://127.0.0.1:56527
CoreDNS is running at https://127.0.0.1:56527/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

Have a nice day! 👋
```

### 3、查看kind所有集群
```Shell
# kind get clusters
ci-cluster
kind
```

### 4、导入镜像

默认kind node节点是无法读取宿主机镜像，需要手动导入到kind 

node节点：

```Shell
# kind load docker-image --name ci-cluster --nodes  ci-cluster-control-plane  traefik:v2.9.5
# docker exec -it ci-cluster-control-plane bash
root@ci-cluster-control-plane:/# crictl images  |grep traefik
docker.io/library/traefik                  v2.9.5               a1252ce6bfaaa       132MB
```

## 五、集群配置：

### 1、多节点集群
对于前面，我们介绍了kind创建集群，默认集群只有一个节点，下面介绍如何创建多个节点。

创建config.yaml内容如下：
```Shell
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

重新创建集群
```Shell
# kind create cluster --config config.yaml
# kubectl get node
NAME                 STATUS   ROLES           AGE    VERSION
kind-control-plane   Ready    control-plane   110s   v1.25.3
kind-worker          Ready    <none>          74s    v1.25.3
kind-worker2         Ready    <none>          88s    v1.25.3

kind本身也可以支持多master高可用部署，这里我主要测试就不简单介绍。

### 2、自定义kubernetes集群版本

```Yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
  images: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
```

### 3、挂载宿主机目录到节点容器持久化存储

```Yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraMounts:
  - hostPath: /Users/wanzi/tools/kind/files
    containerPath: /files
  - hostPath: /Users/wanzi/tools/kind/other-files/
    containerPath: /other-files
    readOnly: true
    selinuxRelabel: false
    propagation: None
```

### 4、端口映射

宿主机端口80映射到nodePort 30950，kind节点containerPort需要和Service的nodePort端口相同
```Yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30950
    hostPort: 80

kind: Pod
apiVersion: v1
metadata:
  name: foo
  labels:
    app: foo
spec:
  containers:
  - name: foo
    image: hashicorp/http-echo:0.2.3
    args:
    - "-text=foo"
    ports:
    - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: foo
spec:
  type: NodePort
  ports:
  - name: http
    nodePort: 30950
    port: 5678
  selector:
    app: foo
```

### 5、自定义label
```Yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
  extraPortMappings:
  - containerPort: 30950
    hostPort: 80
  labels:
    tier: frontend
- role: worker
  labels:
    tier: backend
```

### 6、kubeadm自定义配置：

kind用kubeadm配置集群，他会在第一个控制平面节点上运行kubeadm init,我们可以对它进行自定义：

```Yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "my-label=true"
```

如果你想做更多的定制，有四种配置类型可供选择kubeadm init

- InitConfiguration
- ClusterConfiguration 
- KubeProxyConfiguration 
- KubeletConfiguration


例如，我们可以使用 kubeadm ClusterConfiguration ( spec ) 覆盖 apiserver 标志:

```Yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
        extraArgs:
          enable-admission-plugins: NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook
```

在 KIND管理HA的集群中配置的每个附加节点上，KIND运行kubeadm join可以使用JoinConfiguration自定义配置：

```Yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "my-label2=true"
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "my-label3=true"
```

### 7、配置ingress
允许本地主机通过端口80/443向Ingress控制器发出请求，并配置控制器特定label
```Yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

```Shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```


## 六、完整集群配置

```Yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
#增加多结点集群,并自定义node的集群版本
- role: control-plane
  images: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
  images: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
  labels:
    app: front
  extraMounts:
  - hostPath: /Users/wanzi/tools/kind/wwwroot
    containerPath: /wwwroot
- role: worker
  images: kindest/node:v1.24.7@sha256:577c630ce8e509131eab1aea12c022190978dd2f745aac5eb1fe65c0807eb315
  labels:
    app: backend
  extraMounts:
  - hostPath: /Users/wanzi/tools/kind/wwwroot
    containerPath: /wwwroot
networking:
  #自定义配置APIServer和网络
  apiServerAddress: "127.0.0.1"
  apiServerPort: 6443
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  #disableDefaultCNI: true #默认CNI插件为kindnetd，也可以禁用使用其他插件比如calico
  kubeProxyMode: "ipvs" #设置kubeproxy模式为lvs,如果要禁用可以设置none
```

至此，使用kind来快速构建集群算是告一段落，平常就可以用它跑测试，验证功能了。

参考文档：
https://kind.sigs.k8s.io
https://github.com/kubernetes-sigs/kind
