---
title: "saltstack之saltapi"
date: 2018-02-15T16:22:42+08:00
lastmod: 2018-02-15T16:22:42+08:00
draft: false
description: "saltstack常用命令"
tags: ["saltstack", "saltapi"]
categories: ["devops"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

## 安装
```shell
yum -y install salt-api
```

## 配置
cat /etc/salt/master.d/api.conf  #配置证书和端口
```yaml
rest_cherrypy:
  port: 8888
  debug: True
  ssl_crt: /etc/pki/tls/certs/localhost.crt
  ssl_key: /etc/pki/tls/private/localhost_nopass.key
```
cat /etc/salt/master.d/eauth.conf  #设置权限
```yaml
external_auth:
  pam:
    saltapi:
      - .*
      - '@wheel'
      - '@runner'
```
## 添加账号
```shell
useradd -M -s /sbin/nologin saltapi
echo "saltapi_xxxxxx" | passwd saltapi --stdin
```
## 启动服务
```shell
systemctl enable salt-api
systemctl start salt-api
```

## 测试

curl测试并获取token信息：
```shell
curl -k https://manage-op.test.cn:8888/login -H "Accept: application/x-yaml" \
-d username='saltapi' \
-d password='saltapi_xxxxxxxx' \
-d eauth='pam'
```
如果使用python或者golang可以自己封装client
