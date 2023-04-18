---
title: "ArgoCD安部部署"
date: 2020-05-01T15:22:42+08:00
lastmod: 2020-05-01T17:22:42+08:00
draft: false
description: "ArgoCD安部部署"
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


## 安装部署
ArgoCD的部署非常简单，安装官方的部署方法(HA模式：

```shell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v1.5.2/manifests/ha/install.yaml
```

可以按照需求调整部署文件，待pod顺利启动后

```shell
# kubectl  -n argocd get pod
NAME                                             READY   STATUS    RESTARTS   AGE
argocd-application-controller-66fbf66657-ghf2c   1/1     Running   0          6d17h
argocd-application-controller-66fbf66657-gpm7d   1/1     Running   0          6d17h
argocd-application-controller-66fbf66657-tr5kd   1/1     Running   0          6d17h
argocd-dex-server-5c5f986596-c8ftv               1/1     Running   0          9d
argocd-redis-ha-haproxy-69c6df79c6-2fxd6         1/1     Running   0          9d
argocd-redis-ha-haproxy-69c6df79c6-mksg2         1/1     Running   0          9d
argocd-redis-ha-haproxy-69c6df79c6-wq57f         1/1     Running   0          9d
argocd-redis-ha-server-0                         2/2     Running   0          9d
argocd-redis-ha-server-1                         2/2     Running   0          9d
argocd-redis-ha-server-2                         2/2     Running   0          9d
argocd-repo-server-76bbb56cc7-d8fp5              1/1     Running   0          7d
argocd-repo-server-76bbb56cc7-qvl5z              1/1     Running   0          7d
argocd-repo-server-76bbb56cc7-xqrfn              1/1     Running   0          7d
argocd-server-6464c7bcd-fgktr                    1/1     Running   0          6d19h
argocd-server-6464c7bcd-jkqdb                    1/1     Running   0          6d19h
argocd-server-6464c7bcd-nfdwn                    1/1     Running   0          6d19h
```

## 配置ingress访问argocd

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/redirect-entry-point: https
spec:
  rules:
    - host: cd.testcn
      http:
        paths:
        - backend:
            serviceName: argocd-server
            servicePort: https
          path: /
```

通过https://cd.test.cn/访问argocd，默认本地用户名是admin，密码是其中一个pod的name，获取密码使用如下方法：

```shell
# kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```
![argocd](/images/2020/argocd-install-001.jpeg)


## 用户管理
argocd默认使用本地用户，可以支持ldap。本地用户管理使用修改argocd-cm这个configmap方式，新增用户如下：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-cm
  namespace: argocd
data:
  accounts.chlai: apiKey,login  #允许用户login和生成aipkey
  users.anonymous.enabled: "true"  #允许匿名用户登陆。
```

修改完保存立即生效。

系统默认的role有readonly和admin两个，授予用户admin的权限方法是修改argocd-rbac-cm这个configmap：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/name: argocd-rbac-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |-
    g, chlai, role:admin
  policy.default: role:readonly  #匿名登陆默认使用readonly角色。
```

生成登陆密码需要使用admin用户cli方式login server，通过argocd account update-password命令修改：


```shell
argocd login <argocd-server>
argocd account list
argocd account update-password \
  --account <name> \
  --current-password <current-admin> \
  --new-password <new-user-password>
```

使用新的用户密码登陆argocd web ui。

![argocd](/images/2020/argocd-install-002.png)


## ArgoCD UI界面来创建应用
点击“+ NEW APP”按钮创建应用；
![argocd](/images/2020/argocd-install-003.png)

填写应用名称：guestbook；项目：default；同步策略：手动；

![argocd](/images/2020/argocd-install-004.png)


配置来源。这里配置的是Git ，代码仓库的URL配置为 Github上的项目地址为：https://github.com/argoproj/argocd-example-apps.git；Revision选择：HEAD；项目路径选择：guestbook；

![argocd](/images/2020/argocd-install-005.png)


选择应用部署的目标集群：https://kubernetes.default.svc ，因为此次的Argo CD部署在Kubernetes集群当中，默认Argo CD已经帮我们添加好当前所在的Kubernetes集群，直接使用即可。Namespace选择：my-app。Namespcae可以在Kubernetes集群上使用# kubectl create namespace my-app 命令来创建；

![argocd](/images/2020/argocd-install-006.png)


填写完成后，点击 “CREATE” 按钮进行创建；

![argocd](/images/2020/argocd-install-007.png)


由于尚未部署应用程序，并且尚未创建Kubernetes资源，所以Status还是OutOfSync状态，因此我们还需要点击 “SYNC”按钮进行同步（部署）。同时也可以安装argocd客户端，使用Argo CD CLI进行同步：# argocd app sync guestbook

![argocd](/images/2020/argocd-install-008.png)


应用创建完成，处于“未同步”状态，手动同步应用，开始部署应用
![argocd](/images/2020/argocd-install-009.png)


等待应用创建完成
![argocd](/images/2020/argocd-install-010.png)


在Kubernetes集群中查看应用

![argocd](/images/2020/argocd-install-011.png)

