---
title: "Gitlab runner配置ceph s3"
date: 2021-03-26T17:22:42+08:00
lastmod: 2021-03-26T17:22:42+08:00
draft: false
description: "gitlab runner配置ceph s3"
tags: ["gitlab", "gitlab runner", "ceph"]
categories: ["cicd"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

> 对于前端项目Npm构建的时候，经常拉取前端库耗时比较长，另外不同的job之间复用也是一个问题，无论是artifacts或者cache最终我们需要持久化复用文件，这里我们以cache为例


注意：这里的Gitlab runner是采用helm chart方式部署到k8s集群，runner部署忽略；需要提前准备ceph s3的密钥对，用于配置accesskey和secretkey

## 创建k8s secret

给后边Gitlab Runner连接ceph s3使用。

```yaml
apiVersion: v1
data:
  accesskey: N1NMT0hIRzYxddfsgxVzVssddfsdY=
  secretkey: d25Uc0NDQVdsfsdUkssCQ1VsdEwxeUsdsNwb2R4TnRzZDliTG1DTUN6cQ==
kind: Secret
metadata:
  name: gitlab-runner-s3
  namespace: gitlab-managed-apps
type: Opaque
```


## helm部署gitlab runner

具体Gitlab helm chart参考：https://gitlab.com/gitlab-org/charts/gitlab-runner.git，修改helm chart目录下values.yaml内容如下：
```yaml
cache:
  ## General settings
  cacheType: s3
  cachePath: "devops" #指定ceph s3缓存路径，这里我们以部门来区分
  cacheShared: true
   
  ## S3 settings
  s3ServerAddress: "ops-rgw.test.cn"
  s3BucketName: "runners-cache"
  s3BucketLocation:
  s3CacheInsecure: true
  secretName: "gitlab-runner-s3"
```

更新helm配置
```shell
cd gitlab-runner
helm upgrade   runner-devops -f values.yaml -n gitlab-runner .
```


## 测试gitlab CI任务

配置gitlab Ci，修改.gitlab-ci.yaml，这里以前端项目构建为例：

```yaml
stages:
  - Build
   
build-and-deploy:
  image: registry.test.cn/devops/node:latest
  stage: Build
  cache:
    key: devops-vue
    paths:
      - node_modules/
      - .yarn
  tags:
    - devopstest
  script:
    - yarn config set registry https://r.cnpmjs.org
    - yarn config set @test:registry https://npm.test.cn/
    - yarn --pure-lockfile --cache-folder .yarn --network-timeout 600000
    - yarn build
  when: always
```
Gitlab CI触发构建任务以后，我们观察JOB构建任务实时情况，这时缓存文件已经上传到ceph s3，后期再构建编译就大大提高了效率！

```yaml
64 Creating cache devops-vue-4...
65 node_modules/: found 30710 matching files and directories
66 .yarn: found 34390 matching files and directories 
67 Uploading cache.zip to https://ops-rgw.test.cn/runners-cache/devops/project/4187/devops-vue-js-starter-4
68 Created cache
70Cleaning up file based variables
00:00
72 Job succeeded
```
