---
title: "基于K8S部署gitlab-runner"
date: 2019-11-14T17:22:42+08:00
lastmod: 2019-11-14T17:22:42+08:00
draft: false
description: "k8s下部署gitlab-runner"
tags: ["gitlab-runner", "gitlab", "k8s"]
categories: ["cicd"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

## 部署gitlab-runner

这里基于helm部署，参考：https://gitlab.com/gitlab-org/charts/gitlab-runner.git

```shell
helm install --namespace gitlab-managed-apps --name k8s-gitlab-runner -f  values.yaml 
```
注意：values.yaml文件需要设置privileged: true

## 构建基础镜像(docker in docker)

Dockerfile文件内容：
```yaml
FROM docker:19.03.1-dind
WORKDIR /opt
RUN echo "nameserver 114.114.114.114" >> /etc/resolv.conf
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
RUN apk update
RUN apk upgrade
RUN apk add g++ gcc make docker docker-compose  git
```
构建镜像并推送到harbor
```shell
docker build -t registry.test.cn/devops/docker-tool:19.03.1  .
docker push  registry.test.cn/devops/docker-tool:19.03.1
```

## Gitlab-CI测试

.gitlab-ci.yml文件内容如下:
```yaml
image: registry.test.cn/devops/docker-tool:19.03.1

variables:
  REPO_NAME: gitlab.test.cn/xxx/xxxx
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""
  
before_script:
  - mkdir -p $GOPATH/src/$(dirname $REPO_NAME)
  - ln -svf $CI_PROJECT_DIR $GOPATH/src/$REPO_NAME
  - cd $GOPATH/src/$REPO_NAME

stages:
  - deploy

deploy:
  tags:
    - k8s-gitlab-runner #指定runner
  only:
    - tags
  stage: deploy
  services:
    - registry.test.cn/devops/docker-tool:19.03.1
  script:
    - export DOCKER_HOST='tcp://localhost:2375'
    - docker login -u "$Harbor_bce_user" -p "$Harbor_ecs_passwd" $Harbor_ecs_address #gitlab后台设置统一变量
    - make deploy VERSION=$CI_COMMIT_REF_NAME #根据git tag构建镜像
```
