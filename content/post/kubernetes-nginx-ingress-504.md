---
title: "解决k8s下nginx文件上传限制和504网关超时"
date: 2021-11-08T17:22:42+08:00
lastmod: 2021-11-08T17:22:42+08:00
draft: false
description: "解决k8s下nginx文件上传限制和504网关超时"
tags: ["aliyun", "ingress", "k8s", "nginx"]
categories: ["kubernetes"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---


最近业务使用k8s集群经常有两个问题需要解决，这里记录一下：
- 前端页面上传文件1M限制
- 前端页面发送POST请求到后端，出现504超时

第一个问题主要解决方法：nginx默认上传大小为1M，nginx配置文件http,server, location区域增加如下内容：

```yaml
client_max_body_size 800M;
```

第二个问题主要解决方法：nginx默认读取后端超时时间为1分钟，nginx配置文件http,server, location区域增加如下内容：

```yaml
proxy_connect_timeout 120;
proxy_read_timeout  900;
proxy_send_timeout 900;
```

最终，nginx配置如下：
```yaml
server {
    listen 80;
    server_name web_server;
    location / {
        root /usr/share/nginx/html/;
        index index.htm index.html;
        try_files $uri $uri/ /index.html;
    }
    location /business-operation {
        proxy_pass http://business-operat.common:5000;
        
        proxy_ignore_client_abort on; 
        keepalive_timeout 900s;
        client_body_timeout 900s;
        proxy_connect_timeout 120s;
        proxy_read_timeout  900s;
        proxy_send_timeout 900s;
        client_max_body_size 800m;
        proxy_pass_header Authorization;
        proxy_pass_header WWW-Authenticate;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
    }
}
```
最后，以上nginx配置调整以后，还需要在创建ingress的时候，注解里带上这些信息：
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "800M"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: '120'
    nginx.ingress.kubernetes.io/proxy-read-timeout: '900'
    nginx.ingress.kubernetes.io/proxy-send-timeout: '900'
  name: enterprise-mark
  namespace: frontend
spec:
  rules:
    - host: enterprise-marketing.xxxx.com
      http:
        paths:
          - backend:
              serviceName: enterprise-mark
              servicePort: 80
            path: /
status:
  loadBalancer:
    ingress:
      - ip: 39.107.xxx.xxx
```
