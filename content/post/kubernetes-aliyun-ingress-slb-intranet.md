---
title: "阿里云ACK同时支持公网和内网SLB"
date: 2021-10-21T17:22:42+08:00
lastmod: 2021-10-21T17:22:42+08:00
draft: false
description: "阿里云ACK同时支持公网和内网SLB"
tags: ["aliyun", "ingress", "k8s", "slb"]
categories: ["kubernetes"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---



## 一、背景
- 有一套ACK集群
- 成功部署Nginx ingress controller并绑定了公网SLB

注意：通过阿里云容器服务管理控制台创建的Kubernetes集群在初始化时会自动部署一套Nginx Ingress Controller，默认其挂载在公网SLB实例上.

## 三、配置
### 1、创建内网LB
阿里云控制台，创建内网SLB并绑定VPC

### 2、配置Nginx ingress controller

```yaml
# my-nginx-ingress-slb-intranet.yaml
# intranet nginx ingress slb service
apiVersion: v1
kind: Service
metadata:
  # 这里服务取名为nginx-ingress-lb-intranet。
  name: nginx-ingress-lb-intranet
  namespace: kube-system
  labels:
    app: nginx-ingress-lb-intranet
  annotations:
    # 指明SLB实例地址类型为私网类型。
    service.beta.kubernetes.io/alicloud-loadbalancer-address-type: intranet
    # 修改为您的私网SLB实例ID。
    service.beta.kubernetes.io/alicloud-loadbalancer-id: <YOUR_INTRANET_SLB_ID>
    # 是否自动创建SLB端口监听（会覆写已有端口监听），也可手动创建端口监听。
    #service.beta.kubernetes.io/alicloud-loadbalancer-force-override-listeners: 'false'
spec:
  type: LoadBalancer
  # route traffic to other nodes
  externalTrafficPolicy: "Cluster"
  ports:
  - port: 80
    name: http
    targetPort: 80
  - port: 443
    name: https
    targetPort: 443
  selector:
    # select app=ingress-nginx pods
    app: ingress-nginx
```

创建service资源：
```shell
kubectl apply -f my-nginx-ingress-slb-intranet.yaml
```
获取service资源：
```shell
# kubectl -n kube-system get svc | grep nginx-ingress-lb
nginx-ingress-lb            LoadBalancer   172.21.15.148   39.107.xxx.xxx   80:32076/TCP,443:30803/TCP     433d
nginx-ingress-lb-intranet   LoadBalancer   172.21.5.0      172.17.193.181   80:32282/TCP,443:30507/TCP     1d
```
通过Ingress对外暴露服务时，即可以通过公网SLB来访问该服务，同一个VPC下的其他服务也可以直接通过私网SLB来访问该服务。

3、创建ingress服务
配置完ingress controller，ingress创建无须特殊配置，和以前创建ingress一摸一样，如下例子：
```yaml
# Source: prometheusalert/templates/ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: RELEASE-NAME-prometheusalert
  labels:
    app.kubernetes.io/name: prometheusalert
    helm.sh/chart: prometheusalert-1.0.0
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/version: "1.2.0"
    app.kubernetes.io/managed-by: Helm
spec:
  rules:
    - host: "palert.con.sdi"
      http:
        paths:
          - path: /
            backend:
              serviceName: RELEASE-NAME-prometheusalert
              servicePort: 8080
```
对于内网域名直接解析到内网SLB地址（这里172.17.193.181）即可。

参考：[阿里云文档](https://help.aliyun.com/document_detail/151506.htm)
