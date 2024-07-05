---
title: "给博客自动续签免费SSL证书"
date: 2022-11-12T16:22:42+08:00
lastmod: 2022-11-12T17:22:42+08:00
draft: false
description: "ACME给博客自动续签免费SSL证书"
tags: ["SSL证书", "acme", "阿里云"]
categories: ["tools"]              
author: "wanzi"                 

comment: true   # 关闭评论
toc: true       # 文章目录
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true	 # 关闭打赏
mathjax: true    # 打开 mathjax
---

## 背景

如果你拥有一个网站或者独立博客，你每年都需要关心自己网站SSL证书过期问题，最近wnote.com证书也快过期了；

申请SSL证书，既让网站支持https访问，市面上有免费的也有收费的；

对于免费证书，国内主流云厂商比如阿里、腾讯、ucloud等平台都有免费SSL证书渠道，一般有效期为1年，需要手动申请，当然如果你的用户主要是海外，可以考虑cloudflare家CDN安全防护，默认提供了免费SSL证书。

当然喜欢倒腾的也可以使用Let"s Engypt家免费证书，可以移步[https://letsencrypt.org/](https://letsencrypt.org)，官方一般推荐Certbot客户端来申请，也支持ACME协议这里我就不过多介绍；


今天我们主要使用acme.sh来申请免费证书,一方面原因是acme.sh完全支持ACME协议，另外一个有点就是支持泛域名证书且可以自动续签。

默认acme.sh申请的是[https://zerossl.com](https://zerossl.com)家的免费证书，没有数量限制，如果你直接跑到zerossl官网申请，数量上最多免费申请三个。


acme.sh项目地址:

[https://github.com/acmesh-official/acme.sh](https://github.com/acmesh-official/acme.sh)


## 安装ACME

正常安装完后，安装目录为：~/.acme.sh/，后边所有证书也将放置在此文件夹下。

```shell
git clone https://github.com/acmesh-official/acme.sh.git
cd ./acme.sh
./acme.sh --install -m my@example.com #这里修改成你自己邮箱
```

安装程序总共做了3个操作：

* 创建并复制acme.sh到您的主目录 ( $HOME): ~/.acme.sh/。
* 为：创建别名acme.sh=~/.acme.sh/acme.sh。
* 如果需要，创建每日 cron 作业以检查和更新证书。


## 申请证书

### http方式颁发证书

对于一个证书一个域名：
```shell
acme.sh --issue -d www.example.com -w /home/wwwroot/www.example.com
```
或者：
```shell
acme.sh --issue -d example.com -w /home/username/public_html
```
或者：
```shell
acme.sh --issue -d example.com -w /var/www/html

```
对于同一个证书包含多个域名。
```shell
acme.sh --issue -d example.com -d www.example.com -d cp.example.com -w /home/wwwroot/example.com
```

-d的参数，是为你颁发证书的域名，至少有一个域名，且和后边-w参数形成绑定关系；

-w的参数/home/wwwroot/example.com、/home/username/public_html、/var/www/html是您托管网站文件的 Web 根文件夹，你必须有可写权限，acme.sh会在网站根目录下创建.well-known目录，在其中生成验证文件，以此验证你是网站拥有者。


证书默认放置在~/.acme.sh/example.com/，证书将每60天自动更新一次。

### 自动DNS API颁发证书

如果你的DNS提供商支持API访问，就可以使用API自动颁发证书，无需手动执行任何操作！

支持的DNS供应商可以参考这里：[https://github.com/acmesh-official/acme.sh/wiki/dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi)

我这里wnote.com的域名在阿里云，正好在支持列表，所以这次申请证书就使用这种方式。

首先，我们需要到阿里云申请账户的密钥对，具体链接：[https://ram.console.aliyun.com/manage/ak](https://ram.console.aliyun.com/manage/ak)，申请成功后我们会得到AccessKey ID 和 AccessKey Secret，记录下来方便下边申请证书使用，如果后边成功申请完证书后，密钥对会记录在~/.acme.sh/account.conf文件中，方便后边cron自动续签证书。

```shell
root@VM-0-3-ubuntu:~# export Ali_Key="LTII5t8tE4IgMFGDK6iTcs2A"
root@VM-0-3-ubuntu:~# export Ali_Secret="Ku7q3lPJMISlJqZ9OomLOfzO8LFVff"
root@VM-0-3-ubuntu:~# acme.sh --issue --dns dns_ali -d wnote.com -d *.wnote.com #我这里申请主域证书和泛域名证书
```

申请等待日志如下,大约2分钟左右：
```yaml
[Fri 11 Nov 2022 10:40:31 PM CST] Using CA: https://acme.zerossl.com/v2/DV90
[Fri 11 Nov 2022 10:40:31 PM CST] Multi domain='DNS:wnote.com,DNS:*.wnote.com'
[Fri 11 Nov 2022 10:40:31 PM CST] Getting domain auth token for each domain
[Fri 11 Nov 2022 10:40:34 PM CST] Could not get nonce, let's try again.
[Fri 11 Nov 2022 10:40:41 PM CST] Could not get nonce, let's try again.
[Fri 11 Nov 2022 10:40:47 PM CST] Could not get nonce, let's try again.
[Fri 11 Nov 2022 10:41:38 PM CST] Getting webroot for domain='wnote.com'
[Fri 11 Nov 2022 10:41:38 PM CST] Getting webroot for domain='*.wnote.com'
[Fri 11 Nov 2022 10:41:38 PM CST] Adding txt value: 9QXV-Ve3eEI-JCLjJ7RkMMvPGNaTzV3YmlaXWtwrJVM for domain:  _acme-challenge.wnote.com
[Fri 11 Nov 2022 10:41:40 PM CST] The txt record is added: Success.
[Fri 11 Nov 2022 10:41:40 PM CST] Adding txt value: KXKFA3BChlz0c5NTHN4fmO8jo-e3DbMu0VybF-YohTw for domain:  _acme-challenge.wnote.com
[Fri 11 Nov 2022 10:41:42 PM CST] The txt record is added: Success.
[Fri 11 Nov 2022 10:41:42 PM CST] Let's check each DNS record now. Sleep 20 seconds first.
[Fri 11 Nov 2022 10:42:03 PM CST] You can use '--dnssleep' to disable public dns checks.
[Fri 11 Nov 2022 10:42:03 PM CST] See: https://github.com/acmesh-official/acme.sh/wiki/dnscheck
[Fri 11 Nov 2022 10:42:03 PM CST] Checking wnote.com for _acme-challenge.wnote.com
[Fri 11 Nov 2022 10:42:06 PM CST] Domain wnote.com '_acme-challenge.wnote.com' success.
[Fri 11 Nov 2022 10:42:06 PM CST] Checking wnote.com for _acme-challenge.wnote.com
[Fri 11 Nov 2022 10:42:07 PM CST] Domain wnote.com '_acme-challenge.wnote.com' success.
[Fri 11 Nov 2022 10:42:07 PM CST] All success, let's return
[Fri 11 Nov 2022 10:42:07 PM CST] Verifying: wnote.com
[Fri 11 Nov 2022 10:42:12 PM CST] Processing, The CA is processing your order, please just wait. (1/30)
[Fri 11 Nov 2022 10:42:20 PM CST] Processing, The CA is processing your order, please just wait. (2/30)
[Fri 11 Nov 2022 10:42:29 PM CST] Processing, The CA is processing your order, please just wait. (3/30)
[Fri 11 Nov 2022 10:42:38 PM CST] Processing, The CA is processing your order, please just wait. (4/30)
[Fri 11 Nov 2022 10:42:46 PM CST] Processing, The CA is processing your order, please just wait. (5/30)
[Fri 11 Nov 2022 10:42:55 PM CST] Processing, The CA is processing your order, please just wait. (6/30)
[Fri 11 Nov 2022 10:43:02 PM CST] Success
[Fri 11 Nov 2022 10:43:02 PM CST] Verifying: *.wnote.com
[Fri 11 Nov 2022 10:43:06 PM CST] Processing, The CA is processing your order, please just wait. (1/30)
[Fri 11 Nov 2022 10:43:13 PM CST] Processing, The CA is processing your order, please just wait. (2/30)
[Fri 11 Nov 2022 10:43:28 PM CST] Processing, The CA is processing your order, please just wait. (3/30)
[Fri 11 Nov 2022 10:43:41 PM CST] Processing, The CA is processing your order, please just wait. (4/30)
[Fri 11 Nov 2022 10:43:48 PM CST] Processing, The CA is processing your order, please just wait. (5/30)
[Fri 11 Nov 2022 10:43:52 PM CST] Processing, The CA is processing your order, please just wait. (6/30)
[Fri 11 Nov 2022 10:43:58 PM CST] Processing, The CA is processing your order, please just wait. (7/30)
[Fri 11 Nov 2022 10:44:10 PM CST] Processing, The CA is processing your order, please just wait. (8/30)
[Fri 11 Nov 2022 10:44:29 PM CST] Processing, The CA is processing your order, please just wait. (9/30)
[Fri 11 Nov 2022 10:44:59 PM CST] Success
[Fri 11 Nov 2022 10:44:59 PM CST] Removing DNS records.
[Fri 11 Nov 2022 10:44:59 PM CST] Removing txt: 9QXV-Ve3eEI-JCLmJ7RkLMvPGNaTzV3YmlaXWtwrJVM for domain: _acme-challenge.wnote.com
[Fri 11 Nov 2022 10:45:02 PM CST] Removed: Success
[Fri 11 Nov 2022 10:45:02 PM CST] Removing txt: KXKKA3BChlz0c5NOHN4fmk8jo-e3DMMu0VybF-YohTw for domain: _acme-challenge.wnote.com
[Fri 11 Nov 2022 10:45:06 PM CST] Removed: Success
[Fri 11 Nov 2022 10:45:06 PM CST] Verify finished, start to sign.
[Fri 11 Nov 2022 10:45:06 PM CST] Lets finalize the order.
[Fri 11 Nov 2022 10:45:06 PM CST] Le_OrderFinalize='https://acme.zerossl.com/v2/DV90/order/FYF3wr0jlkJarp3dFuRaKg/finalize'
[Fri 11 Nov 2022 10:45:33 PM CST] Order status is processing, lets sleep and retry.
[Fri 11 Nov 2022 10:45:33 PM CST] Retry after: 15
[Fri 11 Nov 2022 10:45:49 PM CST] Polling order status: https://acme.zerossl.com/v2/DV90/order/FYF3wr0jlkJarp3dFuRaKg
[Fri 11 Nov 2022 10:45:55 PM CST] Downloading cert.
[Fri 11 Nov 2022 10:45:55 PM CST] Le_LinkCert='https://acme.zerossl.com/v2/DV90/cert/RVtakD091ssrWC6HxgcWxQ'
[Fri 11 Nov 2022 10:46:02 PM CST] Cert success.
-----BEGIN CERTIFICATE-----
MIIGbjCCBFagAwIBAgIRAOee+mMTTeRu9hUkpcDXkQcwDQYJKoZIhvcNAQEMBQAw
SzELMAkGA1UEBhMCQVQxEDAOBgNVBAoTB1plcm9TU0wxKjAoBgNVBAMTIVplcm9T
U0wgUlNBIERvbWFpbiBTZWN1cmUgU2l0ZSBDQTAeFw0yMjExMTEwMDAwMDBaFw0y
......
......
fH91qX3O1eXUPQEku+ZLx8hfqKKUuI0l8t6qQkDnK3N+987XdsEXN2mRHnTOfyo4
4Z8RVsC8gXep8VxsVSQ/0urED3ghLBz7Ya5pLFl0inJeLXC/MD1HETryH1iSojv2
JHNsJhGlwrhqORg91jodBBl4
-----END CERTIFICATE-----
[Fri 11 Nov 2022 10:46:02 PM CST] Your cert is in: /root/.acme.sh/wnote.com/wnote.com.cer
[Fri 11 Nov 2022 10:46:02 PM CST] Your cert key is in: /root/.acme.sh/wnote.com/wnote.com.key
[Fri 11 Nov 2022 10:46:02 PM CST] The intermediate CA cert is in: /root/.acme.sh/wnote.com/ca.cer
[Fri 11 Nov 2022 10:46:02 PM CST] And the full chain certs is there: /root/.acme.sh/wnote.com/fullchain.cer
```

### 手动DNS颁发证书

如果DNS服务商不在ACME支持列表，手动申请即可：

在域名解析增加TXT记录，格式如下：

```yaml
域名:_acme-challenge.example.com
Txt记录值:9ihDbjYfTExAYeDs4DBUeuTo18KBzwvTEjUnSwd32-c

域名:_acme-challenge.www.example.com
Txt记录值:9ihDbjxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

申请证书：
```shell
acme.sh --issue --dns -d example.com -d www.example.com -d cp.example.com
```

## 后期更新CA证书
目前acme.sh支持四个正式环境CA, 分别是Let's Encrypt、Buypass、ZeroSSL和SSL.com，默认使用 ZeroSSL, 

如果需要更换可以使用如下命令:

```
acme.sh --set-default-ca --server letsencrypt

acme.sh --set-default-ca --server buypass

acme.sh --set-default-ca --server zerossl

acme.sh --set-default-ca --server ssl.com

acme.sh --set-default-ca --server google
```

注意：国内网站不建议使用google CA，因为很多域名无法访问，申请校验过不去

Google Public CA白嫖地址可以参考：https://cloud.google.com/certificate-manager/docs/public-ca-tutorial



## 更新Docker和nginx配置
申请完证书，我们需要更新下docker-compose和nginx配置，
```yaml
version: "3"
services:
  nginx:
    image: nginx:1.18.0
    volumes:
      - ./conf/nginx.conf:/etc/nginx/nginx.conf
      - /opt/wwwroot/wnote.com:/opt/wnote.com
      - /root/.acme.sh/wnote.com:/opt/ssl
      - ./logs:/var/log/nginx
    restart: always
    ports:
      - 443:443
      - 80:80
    networks:
      - my-nginx
networks:
  my-nginx:
    driver: bridge
```

```yaml
server {
    listen       443 ssl http2;
    server_name  wnote.com www.wnote.com;
    root         /opt/wnote.com;
    ssl_certificate      /opt/ssl/wnote.com.cer;
    ssl_certificate_key  /opt/ssl/wnote.com.key;
    ssl_session_timeout  5m;
    ssl_session_cache    shared:SSL:1m;
    ssl_ciphers          ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:aNULL:!MD5:!ADH:!RC4;
    ssl_protocols        TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers  on;

    location / {
        root   /opt/wnote.com;
        index  index.html index.htm;
    }
}
server {
    listen 80;
    server_name wnote.com;
    rewrite ^ https://$host$1 permanent;
}
```

至此，使用acme.sh申请免费证书、泛域名证书告一段落。
