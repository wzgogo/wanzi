---
title: "kubernetes集群部署traefik2.1"
date: 2019-12-17T10:22:42+08:00
lastmod: 2019-12-31T17:22:42+08:00
draft: false
description: "kubernetes集群部署traefik2.1"
tags: ["traefik", "k8s", "ingress"]
categories: ["kubernetes"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

## 架构&概念


![traefik v2.1 router](/images/2019/routers.png) 

Traefik2.x版本相比1.7.x架构有很大变化，正如上边这张架构图，最主要的功能是支持了TCP协议、增加了Router概念。

这里我们采用在kubernetes集群部署Traefik2.1，业务访问通过haproxy请求到traefik Ingress，下边是搭建过程涉及到一些概念：

* EntryPoints：Traefik的网络入口，定义请求接受的端口(不分http或tcp)
* CRD：Kubernetes API的扩展
* IngressRouter：将传入请求转发到可以处理请求的服务，另外转发请求之前可以通过Middlewares动态更新请求
* Middlewares：请求到达服务之前进行动态处理请求参数，比如header或转发规则等等。
* TraefikService：如果果CRD定义了了这种类型，IngressRouter可以直接引用，处在IngressRouter和服务之间,类似于Maesh架构，更适合较为复杂场景,一般情况可以不使用。

## kubernetes配置

### 配置SSL证书

因为业务服务使用https,这里先配置下SSL证书:
```shell
kubectl -n cattle-system create secret tls tls-rancher-ingress --cert=test.cn.pem  --key=test.cn.key
```

### 配置集群访问控制(RBAC)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - traefik.containo.us
    resources:
      - middlewares
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - traefik.containo.us
    resources:
      - ingressroutes
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - traefik.containo.us
    resources:
      - ingressroutetcps
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - traefik.containo.us
    resources:
      - tlsoptions
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - traefik.containo.us
    resources:
      - traefikservices
    verbs:
      - get
      - list
      - watch

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: kube-system
```

### TLS参数配置
默认配置TLS1.2,当然也可以使用TLS1.3

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: TLSOption
metadata:
  name: mytlsoption
  namespace: kube-system

spec:
  minversion: VersionTLS12
  snistrict: true
  ciphersuites:
    - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    - TLS_RSA_WITH_AES_256_GCM_SHA384
    - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
    - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
    - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
    - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

## CRD配置
这里定义了IngressRoute、Middleware、TLSOption、IngressRouteTCP、TraefikService,其中TraefikService是在2.1版本新增加的CRD

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutes.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRoute
    plural: ingressroutes
    singular: ingressroute
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: middlewares.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: Middleware
    plural: middlewares
    singular: middleware
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsoptions.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSOption
    plural: tlsoptions
    singular: tlsoption
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutetcps.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteTCP
    plural: ingressroutetcps
    singular: ingressroutetcp
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: traefikservices.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TraefikService
    plural: traefikservices
    singular: traefikservice
  scope: Namespaced
```
## TraefikService配置
TraefikService有点类似Maesh解决服务之间的调用逻辑，只是Maesh依赖coredns；另外traefik service还可以设置后端服务权重，配置服务的流量镜像。

这里我们配置traefik dashboard和rancher的traefikservice类型服务，其他服务配置可以参考这里rancher的traefik service, traefik service会转发请求到kubernetes 的服务类型(上一章节我们已经通过helm3创建了rancher服务)，这里只是举例说明：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: TraefikService
metadata:
  name: traefik-webui-traefikservice
  namespace: kube-system

spec:
  weighted:
    services:
      - name: traefik-ingress-service
        weight: 1
        port: 8080

---
apiVersion: traefik.containo.us/v1alpha1
kind: TraefikService
metadata:t
  name: rancher-traefikservice
  namespace: cattle-system

spec:
  weighted:
    services:
      - name: rancher
        weight: 1
        port: 80
```

## Deployment配置

* 配置k8s标准service服务 
* 创建traefik configmap,并配置entrypoints和默认SSL证书
* Deployment方式部署traefik ingress controller

```yaml
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      nodePort: 23456
      name: http
    - protocol: TCP
      port: 443
      nodePort: 23457
      name: https
  type: NodePort

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: traefik-conf
  namespace: kube-system
data:
  traefik.toml: |
    [global]
      checkNewVersion = false
      sendAnonymousUsage = false
    [log]
      level = "DEBUG"
    [api]
      dashboard = true
    [metrics.prometheus]
      buckets = [0.1,0.3,1.2,5.0]
      entryPoint = "metrics"
    [entryPoints]
      [entryPoints.http]
        address = ":80"
      [entryPoints.https]
        address = ":443"
    [tls.stores]
      [tls.stores.default]
        [tls.stores.default.defaultCertificate]
          certFile = "/config/tls/test.cn.crt"
          keyFile  = "/config/tls/test.cn.key"
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: traefik
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      nodeSelector:
        node-role.kubernetes.io/traefik: "true"
      volumes:
      - name: ssl
        secret:
          secretName: traefik-cert
      - name: config
        configMap:
          name: traefik-conf
      containers:
      - image: traefik:v2.1.1
        name: traefik-ingress-lb
        volumeMounts:
        - mountPath: "/config"
          name: "config"
        - mountPath: "/config/tls"
          name: "ssl"
        resources:
          limits:
            cpu: 1000m
            memory: 800Mi
          requests:
            cpu: 500m
            memory: 600Mi
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: https
          containerPort: 443
          hostPort: 443
        args:
        - --entrypoints.http.Address=:80
        - --entrypoints.https.Address=:443
        - --api
        - --accesslog
        - --providers.file.directory=/config/
        - --providers.file.watch=true
        - --ping=true
        - --providers.kubernetescrd
```

## IngressRouter配置

* 定义middleware给rancher服务配置X-Forwarded-Proto的头信息
* 配置rancher的ingressroute，并使用默认证书

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: kube-system
spec:
  redirectScheme:
    scheme: https

---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: http-default-router
  namespace: kube-system
spec:
  entryPoints:
    - http
  routes:
  - match: HostRegexp(`{host:.+}`)
    kind: Rule
    services:
    - name: traefik-ingress-service
      kind: Service
      namespaces: kube-system
      port: 80
    middlewares:
    - name: redirect-https
  tls:
    options:
      name: mytlsoption
      namespaces: kube-system
    certResolver: default

---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-webui
  namespace: kube-system
spec:
  entryPoints:
    - https
  routes:
  - match: Host(`traefik.test.cn`)
    kind: Rule
    services:
    - name: api@internal
      kind: TraefikService
      namespaces: kube-system
```

## 配置Rancher的IngressRoute

由于我这里使用rancher管理集群，实际环境可以根据你自己需求配置
* 定义middleware给rancher服务配置X-Forwarded-Proto的头信息
* 配置rancher的ingressroute，并使用默认证书

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: rancher-https-headers
  namespace: cattle-system
spec:
  headers:
    customRequestHeaders:
      X-Forwarded-Proto: "https"
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: rancher-tls
  namespace: cattle-system
spec:
  entryPoints:
  - https
  routes:
  - match: Host(`rancher.test.cn`)
    kind: Rule
    services:
    - name: rancher
      kind: Service
      namespaces: cattle-system
      port: 80
    middlewares:
    - name: rancher-https-headers
      namespaces: cattle-system

  tls:
    certResolver: default
```
## 统一部署traefik2.1

```shell
kubectl apply -f 01-crd.yaml
kubectl apply -f 02-rbac.yaml
kubectl apply -f 03-tlsoption.yaml
kubectl apply -f 04-traefikservices.yaml #非必须
kubectl apply -f 05-traefik.yaml
kubectl apply -f 06-ingressrouter.yaml
kubectl apply -f 06-ingressrouter-rancher.yaml
```

更多配置信息，请移步我的github仓库：https://github.com/iwz2099/kubecase/tree/master/traefik/v2
