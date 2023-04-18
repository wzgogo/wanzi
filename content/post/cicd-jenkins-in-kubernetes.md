---
title: "kubernetes环境下部署Jenkins"
date: 2023-03-16T16:22:42+08:00
lastmod: 2023-03-16T16:22:42+08:00
draft: false
description: "kubernetes环境下部署Jenkins"
tags: ["jenkins", "k8s"]
categories: ["cicd"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

## 一、安装部署Jenkins

### 1、手动安装

手动安装就很简单，提前将yaml配置准备好即可， 所有CICD资源jenkins-install.yaml内容如下：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
spec:
  storageClassName: local # Local PV
  capacity:
    storage: 30Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  local:
    path: /opt/jenkins
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node02
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: kube-ops
spec:
  storageClassName: local
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: kube-ops
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jenkins
rules:
  - apiGroups: ["extensions", "apps"]
    resources: ["deployments", "ingresses"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
  - apiGroups: [""]
    resources: ["pods/log", "events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins
  namespace: kube-ops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: kube-ops
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: kube-ops
spec:
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccount: jenkins
      initContainers:
        - name: fix-permissions
          image: busybox
          command: ["sh", "-c", "chown -R 1000:1000 /var/jenkins_home"]
          securityContext:
            privileged: true
          volumeMounts:
            - name: jenkinshome
              mountPath: /var/jenkins_home
      containers:
        - name: jenkins
          #image: jenkins/jenkins:lts
          image: jenkins/jenkins:2.375.3-lts #这里我们选用最新稳定版本
          imagePullPolicy: IfNotPresent
          env:
          - name: JAVA_OPTS
            value: -Dhudson.model.DownloadService.noSignatureCheck=true
          ports:
            - containerPort: 8080
              name: web
              protocol: TCP
            - containerPort: 50000
              name: agent
              protocol: TCP
          resources:
            limits:
              cpu: 1500m
              memory: 4096Mi
            requests:
              cpu: 1500m
              memory: 2048Mi
          readinessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 5
            failureThreshold: 12
          volumeMounts:
            - name: jenkinshome
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkinshome
          persistentVolumeClaim:
            claimName: jenkins-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: kube-ops
  labels:
    app: jenkins
spec:
  selector:
    app: jenkins
  ports:
    - name: web
      port: 8080
      targetPort: web
    - name: agent
      port: 50000
      targetPort: agent
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jenkins
  namespace: kube-ops
spec:
  ingressClassName: traefik
  rules:
  - host: jenkins.test.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jenkins
            port:
              number: 8080
```
安装Jenkins

```shell
kubectl apply -f jenkins-install.yaml
```

整个安装过程，牵涉如下资源配置：

* 本地存储：hostpath方式
* 权限认证相关：ServiceAccount、Clusterrole、ClusterRoleBinding
* 部署和服务：deployment、service、ingress

### 2、配置Jenkins 

这里需要注意，由于 jenkens 会对 update-center.json 做签名校验安全检查，这里我们需要先提前关闭，否则下面更改插件源可能会失败，通过配置环境变量 JAVA_OPTS=-Dhudson.model.DownloadService.noSignatureCheck=true 即可；

安装完成jenkins后，打开https://jenkins.test.com，会让你输入Jenkins管理员密码，通过如下方式获取即可：

```shell
kubectl exec -it jenkins-d995d6d4-srfr8 -n kube-ops -- cat /var/jenkins_home/secrets/initialAdminPassword
a571eb7a8ac54256ae638bb4a61fd31b
```


然后跳过插件安装，选择默认安装插件过程会非常慢（也可以选择安装推荐的插件），点击右上角关闭选择插件，等配置好插件中心国内镜像源后再选择安装一些插件。

打开：https://jenkins.test.com/chinese/ 配置国内镜像源地址，我这里选择阿里云（https://mirrors.aliyun.com/jenkins/updates/update-center.json）的：

![jenkins-202303001.png](/images/2023/jenkins-202303001.png)


###  3、安装插件

https://jenkins.test.com/manage/pluginManager/available

我这里主要安装kubernetes、中文语言包、pipeline等，具体安装步骤这里省略。

* Jenkins汉化：https://plugins.jenkins.io/localization-zh-cn/

* Build With Parameters：https://plugins.jenkins.io/build-with-parameters
* Kubernetes：https://plugins.jenkins.io/kubernetes/
* Pipeline：https://plugins.jenkins.io/workflow-aggregator/
* Git：https://plugins.jenkins.io/git/
* zentimestamp：https://plugins.jenkins.io/zentimestamp/


## 二、连接Kubernetes集群


### 1、k8s集群信息配置

打开https://jenkins.test.com/configureClouds/ 

直接新建集群，选择kubernetes，配置集群地址、namespace：

![jenkins-202303002.png](/images/2023/jenkins-202303002.png)

### 2、Jenkins地址配置

配置Jenkins在k8s集群调用地址，以及Jenkins通道地址:

![jenkins-202303003.jpg](/images/2023/jenkins-202303003.jpg)

### 3、Pod Template配置
主要配置命名空间、标签


![jenkins-202303004.png](/images/2023/jenkins-202303004.png)

### 4、Container Template配置
主要配置slave构建容器信息，注意容器名称必须是jnlp。

Docker镜像我这里是自己构建，集成了docker、kubectl等工具，另外，需要注意运行命令、命令参数留空即可，不然创建容器会失效。

![jenkins-202303005.png](/images/2023/jenkins-202303005.png)


配置卷，挂载docker socket文件，共享宿主机docker（Docker in docker）；挂载kubeconfig配置，主要方便kubectl、helm等命令可以操作k8s集群资源。

![jenkins-202303006.jpg](/images/2023/jenkins-202303006.jpg)

最后，配置下service account为Jenkins，这个账户是之前我们创建jenkins的时候的名称。

## 三、简单测试

### 1、创建一个自由风格Job任务

![jenkins-202303007.png](/images/2023/jenkins-202303007.png)


### 2、绑定Slave Pod

![jenkins-202303008.png](/images/2023/jenkins-202303008.png)


### 3、配置shell测试命令

![jenkins-202303009.png](/images/2023/jenkins-202303009.png)

![jenkins-202303010.png](/images/2023/jenkins-202303010.png)


### 4、执行一个构建任务
执行完成以后，我们查看任务构建历史记录发现，这个过程中会自动创建一个slave的pod，同时完成任务以后，该Pod也会自动销毁。


![jenkins-202303011.jpg](/images/2023/jenkins-202303011.jpg)


## 四、总结：
对于持续构建与发布是日常研发体系中必不可少的一步，目前大部分公司采用 Jenkins 集群搭建CICD环境，传统的Jenkins Slave 一主多从方式会存在一些痛点，比如：

* 主 Master发生单点故障，整个流程都不可用了
* 每个 Slave的配置环境不一样，来完成不同语言的编译打包等操作，但是这些差异化的配置导致管理起来非常不方便，维护起来也是比较费劲
* 资源分配不均衡，有的 Slave 要运行的 job 出现排队等待，而有的 Slave 处于空闲状态
* 资源有浪费，每台 Slave 可能是物理机或者虚拟机，当 Slave 处于空闲状态时，也不会完全释放掉资源。

正因为上面的这些种种痛点，我们渴望一种更高效更可靠的方式来完成这个 CI/CD 流程，而 Docker 虚拟化容器技术能很好的解决这个痛点，又特别是在 Kubernetes 集群环境下面能够更好来解决上面的问题，下图是基于 Kubernetes 搭建
Jenkins 集群的简单示意图：

![jenkins-202303012.png](/images/2023/jenkins-202303012.png)


从图上可以看到 Jenkins Master 和 Jenkins Slave 以 Pod 形式运行在 Kubernetes 集群的 Node 上，Master 运行在其中一个节点，并且将其配置数据存储到一个 Volume 上去，Slave 运行在各个节点上，并且它不是一直处于运行状态，它会按照需求动态的创建并自动删除。

这种方式的工作流程大致为：当 Jenkins Master 接受到 Build 请求时，会根据配置的 Label 动态创建一个运行在 Pod 中的 Jenkins Slave 并注册到 Master 上，当运行完 Job 后，这个 Slave 会被注销并且这个 Pod 也会自动删除，恢复到最初状态。

那么我们使用这种方式带来了哪些好处呢？

* 服务高可用，当 Jenkins Master 出现故障时，Kubernetes 会自动创建一个新的 Jenkins Master 容器，并且将 Volume 分配给新创建的容器，保证数据不丢失，从而达到集群服务高可用。
* 动态伸缩，合理使用资源，每次运行 Job 时，会自动创建一个 Jenkins Slave，Job 完成后，Slave 自动注销并删除容器，资源自动释放，而且 Kubernetes 会根据每个资源的使用情况，动态分配 Slave 到空闲的节点上创建，降低出现因某节点资源利用率高，还排队等待在该节点的情况。
* 扩展性好，当 Kubernetes 集群的资源严重不足而导致 Job 排队等待时，可以很容易的添加一个 Kubernetes Node 到集群中，从而实现扩展。 是不是以前我们面临的种种问题在 Kubernetes 

集群环境下面是不是都没有了啊？看上去非常完美。


最后，基于jenkins+kubernetes实现CICD容器化部署方案，无论在工作流程还是资源成本上，都有一定有优势。
