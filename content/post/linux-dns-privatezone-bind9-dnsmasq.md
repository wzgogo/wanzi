---
title: "阿里云PrivateZone+Bind9+Dnsmasq实现内部DNS"
date: 2021-07-10T17:22:42+08:00
lastmod: 2021-07-10T17:22:42+08:00
draft: false
description: "阿里云PrivateZone+Bind9+Dnsmasq实现内部DNS"
tags: ["bind9","dnsmasq","内部dns","阿里云", "aliyun"]
categories: ["linux"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---


> 需求：
> - 阿里云集群能够解析内部域名
> - 办公网解析内部域名+办公网上网解析

解决方法：
- 对于第一个问题，直接使用阿里云PrivateZone解析即可
- 对于第二个问题，采用在PrivateZone配置内部域名zone，然后通过阿里云同步工具同步到办公网bind9服务器；
对于办公网DNS解析入口，使用Dnsmasq处理，对于公网解析直接Forward到公网DNS，内部域名直接转发到bind9处理。

这里可能有人疑问，为什么不用bind直接实现所有内部解析呢？
这里主要原因在实际使用中发现bind9的forward多个dns的时候并发有性能问题，偶尔会有超时现象，这一点dnsmasq做的相对出色很多。

## 一、阿里云PrivateZone配置
参考：https://help.aliyun.com/document_detail/64627.html

## 二、同步阿里云zone到bind9
### 1、docker-compose搭建bind9
```yaml
version: '2'

services:
  bind:
    restart: always
    image: sameersbn/bind:9.16.1-20200524
    environment:
    - ROOT_PASSWORD=DNS2021#
    - WEBMIN_ENABLED=true
    - WEBMIN_INIT_SSL_ENABLED=false
    ports:
    - "15353:53/tcp"
    - "15353:53/udp"
    - "11953:953/tcp"
    - "10000:10000/tcp" #webmin管理
    volumes:
    - ./data:/data
    networks:
      - bind9

networks:
  bind9:
    ipam:
      config:
      - subnet: 10.220.0.0/16
        gateway: 10.220.0.1
```

### 2、修改bind配置文件
> data/bind/etc/named.conf
```yaml
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";
key "rndc-key" {
    algorithm hmac-sha256;
    secret "tREasaE2Jal1GfwfL5iii3a88eRGKWui41l5h3v89OM=";
};
controls {
	inet 127.0.0.1 port 953 allow { 127.0.0.1; } keys { rndc-key; };
	};
logging {
    channel query_log {
        file "query.log" versions 10 size 50M;
        severity info;
        print-category yes;
        print-severity yes;
        print-time yes;
    };
    category queries {
        query_log;
    };
};
```
> data/bind/etc/named.conf.options
```yaml
options {
	directory "/var/cache/bind";
    dnssec-validation no;
    dnssec-enable no;
    recursion yes;
    allow-recursion { any;};
    allow-transfer { any; };
    allow-query-cache { any; };
    listen-on-v6 { any; };
    listen-on port 53 { any; };
    forward first;
    forwarders {
        192.168.1.211 port 53;
        192.168.1.212 port 53;
        };
    transfer-format many-answers;
    transfers-per-ns 500;
    recursive-clients 100000;
    max-transfer-time-in 5;
    transfers-in 300;
    transfers-out 300;
    querylog yes;
};
```
> data/bind/etc/named.conf.local
```yaml
zone "sd.com" {
	type master;
	file "/etc/bind/zones/sd.com.zone";
    allow-update { 127.0.0.1; };
	};
zone "bgt.sdi" {
	type master;
	file "/etc/bind/zones/bgt.sdi.zone";
    allow-update { 127.0.0.1; };
	};
zone "con.sdi" {
	type master;
	file "/etc/bind/zones/con.sdi.zone";
    allow-update { 127.0.0.1; };
	};
```

3、同步域名zone配置到bind9

参考：https://help.aliyun.com/document_detail/102718.html

这里写入到shell，编写任务计划，批量执行同步即可。

```yaml
*/5 * * * * /bin/bash /opt/bind9/update.sh
```

update.sh

```shell
#!/bin/bash
cd /opt/bind9/tools
./Zone_file_sync config.json
chown 101.101 /opt/bind9/data/bind/etc/zones/*
docker exec bind9_bind_1 bash -c "rndc -c /etc/bind/rndc.conf freeze;rndc -c /etc/bind/rndc.conf reload;rndc -c /etc/bind/rndc.conf thaw"
```

配置文件tools/config.json
```yaml
{
  "accessKeyId": "LIDD5ssssmGExzGsdfsY6sJrSqo",
  "accessKeySecret": "C5N1TTESt74KhSTsswSSSWiz2",
  "zone": [
    {
      "zoneName": "sd.com",
      "zoneId": "2a4dc4e0sdsfa5d36a3b88ab6482saf",
      "filePath": "/opt/bind9/data/bind/etc/zones/sd.com.zone"
    },
    {
      "zoneName": "bgt.sdi",
      "zoneId": "f842ca07ccsd6f35d9e294d55a0c900",
      "filePath": "/opt/bind9/data/bind/etc/zones/bgt.sdi.zone"
    },
    {
      "zoneName": "con.sdi",
      "zoneId": "beb4d911addsf2bd86425ds280e7bbf2",
      "filePath": "/opt/bind9/data/bind/etc/zones/con.sdi.zone"
    }
  ]
}
```

## 三、dnsmasq部署
### 1、安装配置dnsmasq

```shell
# yum -y install dnsmasq
# cat >> /etc/dnsmasq.conf << Tag
port=53
proxy-dnssec
no-hosts  #不加载本地/etc/hosts
no-negcache
dns-forward-max=2000
server=114.114.114.114 #指定上游dns服务器
server=223.5.5.5 #指定上游dns服务器
server=/sd.com/127.0.0.1#15353  #转发到指定dns指定端口
server=/bgt.sdi/127.0.0.1#15353
server=/con.sdi/127.0.0.1#15353
log-queries  #记录dns查询日志
log-facility=/var/log/dnsmasq/dnsmasq.log #指定日志路径
log-async=50
cache-size=100000
Tag
# systemctl  start dnsmasq
# systemctl  enable dnsmasq
```
### 2、配置dnsmasq日志轮训策略：

```shell
#cat >> /etc/logrotate.d/dnsmasq Tag
/var/log/dnsmasq/dnsmasq.log {
notifempty
daily
dateext
rotate 15
sharedscripts
postrotate
[ ! -f /var/run/dnsmasq.pid ] || kill -USR2 `cat /var/run/dnsmasq.pid`
endscript
}
Tag
```

测试执行

```shell
logrotate -vf  /etc/logrotate.conf
```

## 四、新增一个新zone，需要做哪些操作？

bind和dnsmasq需要如下变更:

例如：新增test.com

1、/etc/dnsmasq.conf增加server=/test.com/127.0.0.1#15353

2、/opt/bind9/data/bind/etc/named.conf.local新增如下：

```yaml
zone "test.com" {
    type master;
    file "/etc/bind/zones/test.com.zone";
    allow-update { 127.0.0.1; };
    forwarders {};
    };
```

## 五、k8s集群内部如何接入内部DNS
1、修改集群节点/etc/resolv.conf为内部DNS
```yaml
options timeout:1 attempts:1 rotate
nameserver 192.168.1.211
nameserver 192.168.1.212
```
2、coredns对于无法解析的直接forward走
```yaml
Corefile: |
  .:53 {
      errors
      health
      kubernetes cluster.local in-addr.arpa ip6.arpa {
         pods insecure
         upstream
         fallthrough in-addr.arpa ip6.arpa
      }
      prometheus :9153
      forward . /etc/resolv.conf
      cache 30
      loop
      reload
      loadbalance
  }
```  

至此，整个内部DNS解析实现完成。
