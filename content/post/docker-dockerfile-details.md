---
title: "Dockerfile语法详情"
date: 2018-06-21T16:22:42+08:00
lastmod: 2018-06-21T16:22:42+08:00
draft: false
description: "dockerfile文件如何编写,dockerfile编写配置"
tags: ["docker", "dockerfile"]
categories: ["linux"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

## FROM
指定构建镜像使用的基础镜像,FROM必须是Dockerfile中非注释行的第一个指令,如果本地没有指定的镜像，则会自动从Docker的公共库pull镜像下来。
例子：
```yaml
FROM ubuntu:14.04  #继承ubuntu:14.04
```

## MAINTAINER
指定创建者信息
```yaml
MAINTAINER wanzi "iwz2099@163.com"
```
## ENV 
设置环境变量，会被后续 RUN 指令使用，并在容器运行时保持。
ENV <key> <value>       # 只能设置一个变量
ENV <key>=<value> ...   # 允许一次设置多个变量
例子：
```shell
ENV LANG en_US.UTF-8
ENV LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
```

## RUN
RUN <command> 或 RUN ["executable", "param1", "param2"]。
例子：
```shell
RUN yum -y install bind-utils
RUN ["/bin/bash", "-c", "yum -y install bind-utils"]
```

前者将在shell终端中运行命令,即 /bin/sh -c yum -y install bind-utils；后者则使用exec执行。指定使用其它终端可以通过第二种方式实现。
每条RUN指令将在当前镜像基础上执行指定命令，并提交为新的镜像，后续的RUN都在之前RUN提交后的镜像为基础，镜像是分层的，可以通过一个镜像的任何一个历史提交点来创建，类似源码的版本控制。当命令较长时可以使用 \ 来换行。

## COPY:
COPY <src> <dest>
例子：
```shell
COPY script/ /build/script/
```
复制本地主机的<src>（为Dockerfile所在目录的相对路径）到容器中的 <dest>,当使用本地目录为源目录时，推荐使用 COPY。

## ADD
ADD <src> <dest>
例子：
```yaml
ADD https://www.baidu.com/index.html   /var/www/html/
ADD ubuntu-xenial-core-cloudimg-amd64-root.tar.gz /
```
该命令将复制指定的 <src> 到容器中的 <dest>。 其中<src>可以是Dockerfile所在目录的一个相对路径；也可以是一个 URL(自动下载后复制到容器)；还可以是一个tar文件（自动解压为目录）。

## VOLUME
```yaml
VOLUME [ "/data" ]
VOLUME [ "/var/lib/redis", "/var/log/redis" ]
```
声明一个数据卷, 可用于挂载, []里面是路径，创建一个可以从本地主机或其他容器挂载的挂载点，一般用来存放数据库和需要保持的数据等。

## USER
USER <uid>
镜像正在运行时设置的一个UID,RUN命令执行时的用户

## CMD
支持三种格式
CMD ["executable","param1","param2"] 使用 exec 执行，推荐方式；
CMD command param1 param2 在 /bin/sh 中执行，提供给需要交互的应用；
CMD ["param1","param2"] 提供给 ENTRYPOINT 的默认参数；
指定启动容器时执行的命令，每个 Dockerfile 只能有一条 CMD 命令。如果指定了多条命令，只有最后一条会被执行。

## WORKDIR
指定RUN、CMD与ENTRYPOINT命令的工作目录
例子:
```yaml
WORKDIR /opt/nodeapp
```
## ONBUILD
ONBUILD [INSTRUCTION]
配置当所创建的镜像作为其它新创建镜像的基础镜像时，所执行的操作指令。

例如:Dockerfile使用如下的内容创建了镜像image-A。
```yaml
ONBUILD ADD . /app/src
ONBUILD RUN /usr/local/bin/python-build --dir /app/src
```

例如:如果基于 image-A 创建新的镜像时，新的Dockerfile中使用 FROM image-A指定基础镜像时，会自动执行 ONBUILD 指令内容，等价于在后面添加了两条指令。
```yaml
FROM image-A
#Automatically run the following
ADD . /app/src
RUN /usr/local/bin/python-build --dir /app/src
```
使用 ONBUILD 指令的镜像，推荐在标签中注明，例如 ruby:1.9-onbuild。

## ENTRYPOINT
配置容器启动后执行的命令，并且不可被docker run提供的参数覆盖，而CMD是可以被覆盖的。如果需要覆盖，则可以使用docker run --entrypoint选项。每个Dockerfile中只能有一个ENTRYPOINT，当指定多个时，只有最后一个生效。
支持两种格式
ENTRYPOINT [ "nodejs", "server.js" ]
ENTRYPOINT command param1 param2（shell中执行）


## EXPOSE
EXPOSE <port>
告知服务器容器在运行时监听的端口,在启动容器时需要通过 -P，Docker 主机会自动分配一个端口转发到指定的端口。
例子：
```yaml
EXPOSE 3000
```

## 注意事项
* 尽量精简,不安装多余的软件包
* 编写.dockerignore 文件,排除一些目录和文件,语法类似于.gitignore
* 尽量选择 Docker官方提供镜像作为基础版本,减少镜像体积.
* Dockerfile开头几行的指令应当固定下来,不建议频繁更改,有效利用缓存.
* 多条RUN命令使用'\'连接,有利于理解且方便维护.
* COPY与ADD优先使用前者
* 通过-t标记构建镜像,有利于管理新创建的镜像.
* 不在Dockerfile中映射公有端口.
* Push前先在本地运行,确保构建的镜像无误.
