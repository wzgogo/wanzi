---
title: "开源Waf安全防护解决方案"
date: 2024-06-16T16:22:42+08:00
lastmod: 2024-06-16T16:23:42+08:00
draft: false
description: "开源Waf安全防护解决方案"
tags: ["safeline", "雷池", "waf", "cdn"]
categories: ["tools"]              
author: "wanzi"                 

comment: true   # 关闭评论
toc: true       # 文章目录
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true	 # 关闭打赏
mathjax: true    # 打开 mathjax
---

最近网站加了CDN后，凌晨总是莫名其妙来一堆垃圾请求，有些是扫描、有些是大模型的UserAgent、有些是黑蜘蛛。

为了节省CDN费用，同时防止各种注入攻击，所以开始调研开源的Waf方案(自己的小破站够用就行)，如果是企业使用，还是建议使用商业版本，比如阿里云的DCDN、腾讯的EdgeOne或者海外的Cloudflare(海外业务首选)。

## 1、功能介绍

雷池（SafeLine）是长亭科技开源一个Waf解决方案，核心检测能力由智能语义分析算法驱动。

主要功能为网络安全网关，侧重于Waf，可以防御所有的 Web 攻击，例如 sql注入、代码注入、os命令注入、CRLF注入、ldap注入、xpath注入、rce、xss、xxe、ssrf、路径遍历、后门、暴力破解、http 洪水、机器人滥用等等。

大概看了下项目源码，API采用golang，核心waf功能采用tengine+Lua，类似openresty。

项目地址：https://github.com/chaitin/safeline

官网地址：https://waf-ce.chaitin.cn/

https://waf.chaitin.com/


> 备注：其他流行网关，比如Apisix，Kong，Openresty等

## 2、安装

### 下载离线镜像

由于前几天，很多国内的dockerhub镜像加速已经不可用了,所以这里只能离线了。

```shell
# mkdir /opt/safeline
# cd /opt/safeline
# wget https://demo.waf-ce.chaitin.cn/image.tar.gz
# cat image.tar.gz | gzip -d | docker load
# docker ps 
REPOSITORY                                                           TAG       IMAGE ID       CREATED         SIZE
chaitin/safeline-fvm                                                 latest    d9589d01be57   3 days ago      175MB
swr.cn-east-3.myhuaweicloud.com/chaitin-safeline/safeline-fvm        latest    d9589d01be57   3 days ago      175MB
chaitin/safeline-mgt                                                 latest    56ef6365e4c3   3 days ago      81.6MB
swr.cn-east-3.myhuaweicloud.com/chaitin-safeline/safeline-mgt        latest    56ef6365e4c3   3 days ago      81.6MB
chaitin/safeline-luigi                                               latest    1423e012eef0   3 days ago      31.8MB
swr.cn-east-3.myhuaweicloud.com/chaitin-safeline/safeline-luigi      latest    1423e012eef0   3 days ago      31.8MB
chaitin/safeline-mario                                               latest    758fd16c3d30   3 days ago      201MB
swr.cn-east-3.myhuaweicloud.com/chaitin-safeline/safeline-mario      latest    758fd16c3d30   3 days ago      201MB
chaitin/safeline-bridge                                              latest    94aaed2a1b5a   3 days ago      17.9MB
swr.cn-east-3.myhuaweicloud.com/chaitin-safeline/safeline-bridge     latest    94aaed2a1b5a   3 days ago      17.9MB
chaitin/safeline-tengine                                             latest    c67893333287   3 days ago      140MB
swr.cn-east-3.myhuaweicloud.com/chaitin-safeline/safeline-tengine    latest    c67893333287   3 days ago      140MB
chaitin/safeline-detector                                            latest    f5cff5cc7e35   3 days ago      179MB
swr.cn-east-3.myhuaweicloud.com/chaitin-safeline/safeline-detector   latest    f5cff5cc7e35   3 days ago      179MB
chaitin/safeline-chaos                                               latest    71b05b47e3fd   3 days ago      118MB
swr.cn-east-3.myhuaweicloud.com/chaitin-safeline/safeline-chaos      latest    71b05b47e3fd   3 days ago      118MB
chaitin/safeline-postgres                                            15.2      bf700010ce28   13 months ago   379MB
swr.cn-east-3.myhuaweicloud.com/chaitin-safeline/safeline-postgres   15.2      bf700010ce28   13 months ago   379MB
```

### 配置环境
```shell
# cd  /opt/safeline
# vim .env
SAFELINE_DIR=/opt/safeline
IMAGE_TAG=latest
MGT_PORT=9443
POSTGRES_PASSWORD=Pgssf201Waf
SUBNET_PREFIX=172.22.222
IMAGE_PREFIX=chaitin
```

* SAFELINE_DIR: 雷池安装目录，这里配置/opt/safeline
* IMAGE_TAG: 要安装的雷池版本，保持默认的 latest 即可
* MGT_PORT: 雷池控制台的端口，保持默认的 9443 即可
* POSTGRES_PASSWORD: 雷池所需数据库的初始化密码，请随机生成一个
* SUBNET_PREFIX: 雷池内部网络的网段，保持默认的 172.22.222 即可
* IMAGE_PREFIX: 雷池镜像源的前缀，保持默认的 chaitin 即可

### docker-compose管理容器：

```shell
# cd "/opt/safeline"
# wget "https://waf-ce.chaitin.cn/release/latest/compose.yaml"  -O docker-compose.yaml
# docker-compose  up -d
# docker-compose ps 
Name                     Command                  State                            Ports
--------------------------------------------------------------------------------------------------------------------
safeline-bridge     /app/bridge serve -n unix  ...   Up
safeline-chaos      ./entrypoint.sh                  Up             9000/tcp
safeline-detector   /detector/entrypoint.sh          Up (healthy)   8000/tcp, 8001/tcp
safeline-fvm        ./fvm /app/config.yml            Up
safeline-luigi      /bin/sh -c /app/luigi            Up             80/tcp
safeline-mario      /mario/entrypoint.sh             Up (healthy)
safeline-mgt        /docker-entrypoint.sh /bin ...   Up (healthy)   0.0.0.0:9443->1443/tcp,:::9443->1443/tcp, 80/tcp
safeline-pg         docker-entrypoint.sh postg ...   Up (healthy)   5432/tcp
safeline-tengine    entrypoint.sh nginx -g dae ...   Up
```

第一次登录雷池需要初始化管理员账户，执行以下命令即可
```shell
docker exec safeline-mgt resetadmin
```

命令执行完成后会随机重置 admin 账户的密码，输出结果如下
```
[SafeLine] Initial username：admin
[SafeLine] Initial password：**********
[SafeLine] Done
```

到这里基本安装完成，只需要云端ACL开放9443端口访问即可

可以打开浏览器访问 https://<safeline-ip>:9443/ 来使用雷池控制台

## 3、网站配置

### SSL证书配置：

支持自定义上传证书和申请免费证书两种。

![添加证书](https://wnote.com/images/2024/safeline1002.png)

### 添加站点：

添加站点，另外这里默认已经开启人机验证、身份验证、动态防护;需要注意的是这里添加上游服务器时，回源比较模糊，比如回源端口配置、协议等

![添加站点](https://wnote.com/images/2024/safeline1001.png)


### 代理配置：

![代理配置](https://wnote.com/images/2024/safeline1003.png)


### 安全防护设置：
这里主要配置频率限制(类似nginx limit)、自定义规则(根据http头信息设置规则)、语意分析(注入相关)等

限制访问频率: 底层逻辑和nginx下limit_req、limit_conn、limit_rate 差不多

![频率限制](https://wnote.com/images/2024/safeline1004.png)

自定义规则: 主要根据http头部信息做一些策略，和常用CDN的访问控制类似，但是这个功能更灵活.

![自定义规则1](https://wnote.com/images/2024/safeline1005.png)

![自定义规则2](https://wnote.com/images/2024/safeline1007.png)

语义分析: 主要是识别各种注入、机器人检测等

![语义分析](https://wnote.com/images/2024/safeline1006.png)


## 4、总结

通过两天测试，雷池基本满足我这个小破站的需求，但是它也有很多无法满足的地方。

比如：申请免费证书不支持泛域名、上游服务器回源协议模糊等

最后，放一张效果图.

![雷池后台首页](https://wnote.com/images/2024/safeline1008.png)


参考：
https://docs.waf-ce.chaitin.cn/zh/home

