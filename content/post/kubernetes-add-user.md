---
title: "kubernetes集群添加用户"
date: 2019-12-31T10:22:42+08:00
lastmod: 2019-12-31T17:22:42+08:00
draft: false
description: "kubernetes集群添加用户"
tags: ["k8s", "cfssl"]
categories: ["kubernetes"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

之前通过ansible搭建了kubernetes集群环境,这里需求主要是添加一个用户进行日常管理，并限制到指定的namespace，接下来进行操作：

# kubernetes中用户

> K8S中有两种用户(User)——服务账号(ServiceAccount)和普通意义上的用户(User), ServiceAccount是由K8S管理的，而User通常是在外部管理，K8S不存储用户列表——也就是说，添加/编辑/删除用户都是在外部进行，无需与K8S API交互，虽然K8S并不管理用户，但是在K8S接收API请求时，是可以认知到发出请求的用户的，实际上，所有对K8S的API请求都需要绑定身份信息(User或者ServiceAccount)，这意味着，可以为User配置K8S集群中的请求权限

对于kubernetes接受用户请求的时候通常采取客户端证书、静态token文件、静态密码文件三种方式，这里我们只介绍证书验证。

# 生成用户证书

准备证书生成文件csr,通过kubernetes的ca签发证书,通常k8s api server的ca文件路径为/etc/kubernetes/pki/,这里会生成cicd-admin-key.pem(私钥)和cicd-admin.pem(证书), 具体csr如何生成可以参考：https://wnote.com/post/linux-openssl-issue-private-certificate/

这里的csr文件传递的用户名为cicd-admin
```yaml
# vim cicd-admin-csr.json
{
  "CN": "cicd-admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
# cd /etc/kubernetes/pki/ 
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  cicd-admin-csr.json | cfssljson -bare cicd-admin
# ls -l cicd-admin*
-rw-r--r-- 1 root root 1001 Dec 19 16:51 cicd-admin.csr
-rw-r--r-- 1 root root  224 Dec 19 16:50 cicd-admin-csr.json
-rw------- 1 root root 1675 Dec 19 16:51 cicd-admin-key.pem
-rw-r--r-- 1 root root 1387 Dec 19 16:51 cicd-admin.pem
```

## 用户权限控制(RBAC)

### 创建角色

k8s中rbac的角色主要有两种,普通角色(Role)和集群角色(ClusterRole)，ClusterRole是特殊的Role 
* Role属于某个命名空间; 而ClusterRole属于整个集群，包括所有的命名空间
* ClusterRole能够授予集群范围的权限，比如node资源的管理，可以请求全命名空间的资源(通过指定--all-namespaces), 默认情况下, K8S内置了一个名为admin的ClusterRole，实际使用中无需创建admin Role

在cicd的namespace下创建一个role为cicd-admin:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: cicd
  name: cicd-admin
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["*"]
```

### 绑定用户到指定角色

> subjects指定账户类型可以是User也可以是service account，这里指定的是用户cicd-admin, roleRef指定RoleBinding引用角色

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: cicd-admin-binding
  namespace: cicd
subjects:
- kind: User
  name: cicd-admin
  apiGroup: ""
roleRef:
  kind: Role
  name: admin
  apiGroup: ""
```
或者通过kubectl进行绑定也可以:
```shell
kubectl create rolebinding cicd-admin-binding --role=admin --user=cicd-admin --namespace=cicd
```
或者给用户绑定到默认ClusterRole的admin角色,而这里cicd-admin如果绑定了admin，其权限也只会被限制在cicd的namespace
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: cicd-admin-binding
  namespace: cicd
subjects:
- kind: User
  name: cicd-admin
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: ""
```


## 客户端配置

### 设置集群参数

```shell
export KUBE_APISERVER="https://cicd-k8s.test.cn:8443"
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/ssl/ca.pem \
--embed-certs=true \
--server=${KUBE_APISERVER} \
--kubeconfig=cicd-admin.kubeconfig
```

### 设置客户端认证参数

```shell
kubectl config set-credentials cicd-admin \
--client-certificate=/etc/kubernetes/ssl/cicd-admin.pem \
--client-key=/etc/kubernetes/ssl/cicd-admin-key.pem \
--embed-certs=true \
--kubeconfig=cicd-admin.kubeconfig
```

### 设置上下文参数
```shell
kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=cicd-admin \
--namespace=cicd \
--kubeconfig=cicd-admin.kubeconfig
```

### 设置默认上下文

```shell
kubectl config use-context kubernetes --kubeconfig=cicd-admin.kubeconfig
```

### 客户端使用
将kubeconfig文件覆盖~/.kube/config
```shell
cp cicd-admin.kubeconfig ~/.kube/config
```
至此，kubernetes添加用户并进行授权访问k8s集群的介绍就告一段落。
