---
title: "argocd部署deployment出现: no space left on device"
date: 2020-05-18T10:22:42+08:00
lastmod: 2020-05-18T17:22:42+08:00
draft: false
description: "argocd 部署deployment出现: no space left on devic"
tags: ["argocd", "k8s", "cicd"]
categories: ["cicd"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---
# 故障现象

上午通过argocd部署几个业务应用，部署了2个以后，第三方死活部署不成功，相同的配置，知识集群不一样，怎么会出现这样的问题呢？

于是查看了下日志,如下：

```yaml
  Warning  Failed     1m                kubelet, 172.16.25.13  Error: Error response from daemon: error creating overlay mount to /var/lib/docker/overlay2/ba37165607862efb350093e5e287207e2547759fd81dc4e5e356a86ac5e28324-init/merged: no space left on device
  Warning  Failed     1m                kubelet, 172.16.25.13  Error: Error response from daemon: error creating overlay mount to /var/lib/docker/overlay2/f69b62f360fc2a94487aca041b08d0929810beab0602e0ec8b90c94b2e893337-init/merged: no space left on device
  Warning  Failed     48s               kubelet, 172.16.25.13  Error: Error response from daemon: error creating overlay mount to /var/lib/docker/overlay2/a8d20a44183b39ae989eee8a442960124ff23844482f726ea7ab39a292aecbb3-init/merged: no space left on device
```

# 解决方法

1. 排查磁盘空间,发现没有满
```shell
root@gpu613:~# df -Th /
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sda2      ext4  1.8T  359G  1.3T  22% /
```
2. 经过谷歌，发现可能是inotify watch 耗尽导致

```shell
#cat /proc/sys/fs/inotify/max_user_watches
8192
```
尝试修改fd watch的目录数量
```shell
echo "fs.inotify.max_user_watches=100000" >> /etc/sysctl.conf
sysctl -p
```
重新发送argocd sync同步应用，发现这次成功创建了deployment,果真是这货的原因.
