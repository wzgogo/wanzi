---
title: "如何快速搭建一套Greenplum集群"
date: 2021-09-08T17:22:42+08:00
lastmod: 2021-09-08T17:22:42+08:00
draft: false
description: "如何快速搭建一套Greenplum集群"
tags: ["greenplum", "postgresql"]
categories: ["storage"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

最近内部项目支持大数据项目，需要模拟客户场景配置Greenplum(老版本4.2.2.4)，因此这里记录下greenplum集群搭建过程,其实对于高版本的GP搭建过程一样。

## 构建基础镜像

centos6环境Dockerfile：
```yaml
FROM centos:6

RUN mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
RUN curl -o /etc/yum.repos.d/CentOS-Base.repo https://www.xmpan.com/Centos-6-Vault-Aliyun.repo
RUN yum -y update; yum clean all
RUN yum install -y \
    net-tools \
    ntp \
    openssh-server \
    openssh-clients \
    less \
    iproute \
    lsof \
    wget \
    ed \
    which; yum clean all
RUN ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key -N ''
RUN groupadd gpadmin
RUN useradd gpadmin -g gpadmin
RUN echo gpadmin | passwd gpadmin --stdin
ENTRYPOINT ["/usr/sbin/sshd", "-D"]
```
构建镜像：
```shell
docker build -t harbor.test.com/sp/greenplum-base:centos6 .
```

centos7环境Dockerfile：
```yaml
FROM centos:7

RUN mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
RUN curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
RUN yum -y update; yum clean all
RUN yum install -y \
    net-tools \
    ntp \
    openssh-server \
    openssh-clients \
    less \
    iproute \
    lsof \
    wget \
    ed \
    which; yum clean all
RUN ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key -N ''
RUN groupadd gpadmin
RUN useradd gpadmin -g gpadmin
RUN echo gpadmin | passwd gpadmin --stdin
ENTRYPOINT ["/usr/sbin/sshd", "-D"]
```

```shell
docker build -t harbor.test.com/sp/greenplum-base:centos7 .
```

## Docker-compose管理GP

```yaml
version: "2"

services:
  mdw:
    #image: "harbor.test.com/sp/greenplum-base:centos7"
    image: "harbor.test.com/sp/greenplum-base:centos6"
    container_name: gpdb-mdw
    volumes:
    - greenplum-mdw:/home/gpadmin
    ports:
      - "2222:22"
      - "15432:5432"
    hostname: mdw
    tty: true
    networks:
      - greenplum

  sdw1:
    #image: "harbor.test.com/sp/greenplum-base:centos7"
    image: "harbor.test.com/sp/greenplum-base:centos6"
    container_name: gpdb-sdw1
    volumes:
    - greenplum-mdw:/home/gpadmin
    hostname: sdw1
    tty: true
    networks:
      - greenplum

  sdw2:
    #image: "harbor.test.com/sp/greenplum-base:centos7"
    image: "harbor.test.com/sp/greenplum-base:centos6"
    container_name: gpdb-sdw2
    volumes:
    - greenplum-mdw:/home/gpadmin
    hostname: sdw2
    tty: true
    networks:
      - greenplum

networks:
  greenplum:
    ipam:
      config:
      - subnet: 10.188.0.0/24
        gateway: 10.188.0.1

volumes:
 greenplum-mdw:
 greenplum-sdw1:
 greenplum-sdw2:
```

启动服务
```shell
docker-compose up -d 
```

## 初始化安装greenplum

### 将greenplum二进制copy到容器中进行安装
```shell
docker cp greenplum-db-4.2.2.4-build-1-CE-RHEL5-x86_64.bin gpdb-mdw:/home/gpadmin
docker exec -it gpdb-mdw /bin/bash
su - gpadmin
bash greenplum-db-4.2.2.4-build-1-CE-RHEL5-x86_64.bin
```
我这里安装目录为：/home/gpadmin/greenplum-db-new


### 初始化配置文件：

```shell
# cd /home/gpadmin/gpconfig/
# touch hostlist seglist
# cat hostlist
mdw
sdw1
sdw2
# cat seglist 
sdw1
sdw2
```
### 加载环境变量并配置ssh密钥信息
```shell
source ~/greenplum-db/greenplum_path.sh
gpssh-exkeys -f /home/gpadmin/gpconfig/hostlist
gpseginstall -f /home/gpadmin/gpconfig/seglist
gpssh -f ~/gpconfig/hostlist -e ls -l $GPHOME
```

### 分别在mdw、sdw1、sdw2主机下创建数据目录

```shell
mkdir -p ~/data/master
gpssh -f ~/gpconfig/seglist -e "mkdir -p ~/data/primary"
gpssh -f ~/gpconfig/seglist -e "mkdir -p ~/data/mirror"
```

### 修改初始化安装配置：
```shell
$ cp /home/gpadmin/greenplum-db/docs/cli_help/gpconfigs/gpinitsystem_config ~/gpconfig/gpinitsystem_config
$ egrep -v '^$|^#' gpinitsystem_config
ARRAY_NAME="EMC Greenplum DW"
SEG_PREFIX=gpseg
PORT_BASE=40000
MACHINE_LIST_FILE=/home/gpadmin/gpconfig/hostlist
declare -a DATA_DIRECTORY=(/home/gpadmin/data/primary)
MASTER_HOSTNAME=mdw
MASTER_DIRECTORY=/home/gpadmin/data/master
MASTER_PORT=5432
TRUSTED_SHELL=ssh
CHECK_POINT_SEGMENTS=8
ENCODING=UNICODE
```

### 初始化安装greenplum服务器
```shell
[gpadmin@mdw ~]$ gpinitsystem -c gpconfig/gpinitsystem_config -h gpconfig/seglist
20210723:08:04:39:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Checking configuration parameters, please wait...
/bin/mv: try to overwrite `gpconfig/gpinitsystem_config', overriding mode 0664 (rw-rw-r--)?
20210723:08:04:51:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Reading Greenplum configuration file gpconfig/gpinitsystem_config
20210723:08:04:51:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Locale has not been set in gpconfig/gpinitsystem_config, will set to default value
20210723:08:04:51:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Locale set to en_US.utf8
20210723:08:04:51:000282 gpinitsystem:mdw:gpadmin-[INFO]:-No DATABASE_NAME set, will exit following template1 updates
20210723:08:04:51:000282 gpinitsystem:mdw:gpadmin-[INFO]:-MASTER_MAX_CONNECT not set, will set to default value 250
20210723:08:04:52:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Checking configuration parameters, Completed
20210723:08:04:52:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Commencing multi-home checks, please wait...
..
20210723:08:04:52:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Configuring build for standard array
20210723:08:04:52:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Commencing multi-home checks, Completed
20210723:08:04:53:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Building primary segment instance array, please wait...
..
20210723:08:04:54:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Checking Master host
20210723:08:04:54:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Checking new segment hosts, please wait...
..
20210723:08:04:57:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Checking new segment hosts, Completed
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Greenplum Database Creation Parameters
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:---------------------------------------
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Master Configuration
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:---------------------------------------
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Master instance name       = EMC Greenplum DW
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Master hostname            = mdw
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Master port                = 5432
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Master instance dir        = /home/gpadmin/data/master/gpseg-1
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Master LOCALE              = en_US.utf8
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Greenplum segment prefix   = gpseg
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Master Database            =
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Master connections         = 250
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Master buffers             = 128000kB
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Segment connections        = 750
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Segment buffers            = 128000kB
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Checkpoint segments        = 8
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Encoding                   = UNICODE
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Postgres param file        = Off
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Initdb to be used          = /home/gpadmin/greenplum-db/./bin/initdb
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-GP_LIBRARY_PATH is         = /home/gpadmin/greenplum-db/./lib
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Ulimit check               = Passed
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Array host connect type    = Single hostname per node
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Master IP address [1]      = 10.188.0.2
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Standby Master             = Not Configured
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Primary segment #          = 1
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Total Database segments    = 2
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Trusted shell              = ssh
20210723:08:04:58:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Number segment hosts       = 2
20210723:08:04:59:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Mirroring config           = OFF
20210723:08:04:59:000282 gpinitsystem:mdw:gpadmin-[INFO]:----------------------------------------
20210723:08:04:59:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Greenplum Primary Segment Configuration
20210723:08:04:59:000282 gpinitsystem:mdw:gpadmin-[INFO]:----------------------------------------
20210723:08:04:59:000282 gpinitsystem:mdw:gpadmin-[INFO]:-sdw1 	/home/gpadmin/data/primary/gpseg0 	40000 	2 	0
20210723:08:04:59:000282 gpinitsystem:mdw:gpadmin-[INFO]:-sdw2 	/home/gpadmin/data/primary/gpseg1 	40000 	3 	1
Continue with Greenplum creation Yy/Nn>
Y
20210723:08:05:02:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Building the Master instance database, please wait...
20210723:08:05:13:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Starting the Master in admin mode
20210723:08:05:37:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Commencing parallel build of primary segment instances
20210723:08:05:37:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Spawning parallel processes    batch [1], please wait...
..
20210723:08:05:37:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Waiting for parallel processes batch [1], please wait...
.................
20210723:08:05:54:000282 gpinitsystem:mdw:gpadmin-[INFO]:------------------------------------------------
20210723:08:05:54:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Parallel process exit status
20210723:08:05:54:000282 gpinitsystem:mdw:gpadmin-[INFO]:------------------------------------------------
20210723:08:05:54:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Total processes marked as completed           = 2
20210723:08:05:54:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Total processes marked as killed              = 0
20210723:08:05:54:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Total processes marked as failed              = 0
20210723:08:05:54:000282 gpinitsystem:mdw:gpadmin-[INFO]:------------------------------------------------
20210723:08:05:55:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Deleting distributed backout files
20210723:08:05:55:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Removing back out file
20210723:08:05:55:000282 gpinitsystem:mdw:gpadmin-[INFO]:-No errors generated from parallel processes
20210723:08:05:55:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Restarting the Greenplum instance in production mode
20210723:08:05:55:010330 gpstop:mdw:gpadmin-[INFO]:-Starting gpstop with args: -a -i -m -d /home/gpadmin/data/master/gpseg-1
20210723:08:05:55:010330 gpstop:mdw:gpadmin-[INFO]:-Gathering information and validating the environment...
20210723:08:05:56:010330 gpstop:mdw:gpadmin-[INFO]:-Obtaining Greenplum Master catalog information
20210723:08:05:56:010330 gpstop:mdw:gpadmin-[INFO]:-Obtaining Segment details from master...
20210723:08:05:57:010330 gpstop:mdw:gpadmin-[INFO]:-Greenplum Version: 'postgres (Greenplum Database) 4.2.2.4 build 1 Community Edition'
20210723:08:05:57:010330 gpstop:mdw:gpadmin-[INFO]:-There are 0 connections to the database
20210723:08:05:57:010330 gpstop:mdw:gpadmin-[INFO]:-Commencing Master instance shutdown with mode='immediate'
20210723:08:05:57:010330 gpstop:mdw:gpadmin-[INFO]:-Master host=mdw
20210723:08:05:57:010330 gpstop:mdw:gpadmin-[INFO]:-Commencing Master instance shutdown with mode=immediate
20210723:08:05:57:010330 gpstop:mdw:gpadmin-[INFO]:-Master segment instance directory=/home/gpadmin/data/master/gpseg-1
20210723:08:05:58:010413 gpstart:mdw:gpadmin-[INFO]:-Starting gpstart with args: -a -d /home/gpadmin/data/master/gpseg-1
20210723:08:05:58:010413 gpstart:mdw:gpadmin-[INFO]:-Gathering information and validating the environment...
20210723:08:05:59:010413 gpstart:mdw:gpadmin-[INFO]:-Greenplum Binary Version: 'postgres (Greenplum Database) 4.2.2.4 build 1 Community Edition'
20210723:08:06:00:010413 gpstart:mdw:gpadmin-[INFO]:-Greenplum Catalog Version: '201109210'
20210723:08:06:01:010413 gpstart:mdw:gpadmin-[INFO]:-Starting Master instance in admin mode
20210723:08:06:02:010413 gpstart:mdw:gpadmin-[INFO]:-Obtaining Greenplum Master catalog information
20210723:08:06:02:010413 gpstart:mdw:gpadmin-[INFO]:-Obtaining Segment details from master...
20210723:08:06:03:010413 gpstart:mdw:gpadmin-[INFO]:-Setting new master era
20210723:08:06:03:010413 gpstart:mdw:gpadmin-[INFO]:-Master Started...
20210723:08:06:03:010413 gpstart:mdw:gpadmin-[INFO]:-Checking for filespace consistency
20210723:08:06:03:010413 gpstart:mdw:gpadmin-[INFO]:-Obtaining current filespace entries used by TRANSACTION_FILES
20210723:08:06:03:010413 gpstart:mdw:gpadmin-[INFO]:-TRANSACTION_FILES OIDs are consistent for pg_system filespace
20210723:08:06:04:010413 gpstart:mdw:gpadmin-[INFO]:-TRANSACTION_FILES entries are consistent for pg_system filespace
20210723:08:06:04:010413 gpstart:mdw:gpadmin-[INFO]:-Checking for filespace consistency
20210723:08:06:04:010413 gpstart:mdw:gpadmin-[INFO]:-Obtaining current filespace entries used by TEMPORARY_FILES
20210723:08:06:05:010413 gpstart:mdw:gpadmin-[INFO]:-TEMPORARY_FILES OIDs are consistent for pg_system filespace
20210723:08:06:06:010413 gpstart:mdw:gpadmin-[INFO]:-TEMPORARY_FILES entries are consistent for pg_system filespace
20210723:08:06:06:010413 gpstart:mdw:gpadmin-[INFO]:-Shutting down master
20210723:08:06:10:010413 gpstart:mdw:gpadmin-[INFO]:-No standby master configured.  skipping...
20210723:08:06:10:010413 gpstart:mdw:gpadmin-[INFO]:-Commencing parallel segment instance startup, please wait...
....
20210723:08:06:14:010413 gpstart:mdw:gpadmin-[INFO]:-Process results...
20210723:08:06:14:010413 gpstart:mdw:gpadmin-[INFO]:-----------------------------------------------------
20210723:08:06:14:010413 gpstart:mdw:gpadmin-[INFO]:-   Successful segment starts                                            = 2
20210723:08:06:14:010413 gpstart:mdw:gpadmin-[INFO]:-   Failed segment starts                                                = 0
20210723:08:06:14:010413 gpstart:mdw:gpadmin-[INFO]:-   Skipped segment starts (segments are marked down in configuration)   = 0
20210723:08:06:14:010413 gpstart:mdw:gpadmin-[INFO]:-----------------------------------------------------
20210723:08:06:14:010413 gpstart:mdw:gpadmin-[INFO]:-
20210723:08:06:14:010413 gpstart:mdw:gpadmin-[INFO]:-Successfully started 2 of 2 segment instances
20210723:08:06:14:010413 gpstart:mdw:gpadmin-[INFO]:-----------------------------------------------------
20210723:08:06:14:010413 gpstart:mdw:gpadmin-[INFO]:-Starting Master instance mdw directory /home/gpadmin/data/master/gpseg-1
20210723:08:06:16:010413 gpstart:mdw:gpadmin-[INFO]:-Command pg_ctl reports Master mdw instance active
20210723:08:06:17:010413 gpstart:mdw:gpadmin-[INFO]:-Database successfully started
20210723:08:06:17:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Completed restart of Greenplum instance in production mode
20210723:08:06:17:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Loading gp_toolkit...
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Scanning utility log file for any warning messages
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[WARN]:-*******************************************************
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[WARN]:-Scan of log file indicates that some warnings or errors
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[WARN]:-were generated during the array creation
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Please review contents of log file
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[INFO]:-/home/gpadmin/gpAdminLogs/gpinitsystem_20210723.log
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[INFO]:-To determine level of criticality
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[WARN]:-*******************************************************
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Greenplum Database instance successfully created
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[INFO]:-------------------------------------------------------
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[INFO]:-To complete the environment configuration, please
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[INFO]:-update gpadmin .bashrc file with the following
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[INFO]:-1. Ensure that the greenplum_path.sh file is sourced
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[INFO]:-2. Add "export MASTER_DATA_DIRECTORY=/home/gpadmin/data/master/gpseg-1"
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[INFO]:-   to access the Greenplum scripts for this instance:
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[INFO]:-   or, use -d /home/gpadmin/data/master/gpseg-1 option for the Greenplum scripts
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[INFO]:-   Example gpstate -d /home/gpadmin/data/master/gpseg-1
20210723:08:06:18:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Script log file = /home/gpadmin/gpAdminLogs/gpinitsystem_20210723.log
20210723:08:06:19:000282 gpinitsystem:mdw:gpadmin-[INFO]:-To remove instance, run gpdeletesystem utility
20210723:08:06:19:000282 gpinitsystem:mdw:gpadmin-[INFO]:-To initialize a Standby Master Segment for this Greenplum instance
20210723:08:06:19:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Review options for gpinitstandby
20210723:08:06:19:000282 gpinitsystem:mdw:gpadmin-[INFO]:-------------------------------------------------------
20210723:08:06:19:000282 gpinitsystem:mdw:gpadmin-[INFO]:-The Master /home/gpadmin/data/master/gpseg-1/pg_hba.conf post gpinitsystem
20210723:08:06:19:000282 gpinitsystem:mdw:gpadmin-[INFO]:-has been configured to allow all hosts within this new
20210723:08:06:19:000282 gpinitsystem:mdw:gpadmin-[INFO]:-array to intercommunicate. Any hosts external to this
20210723:08:06:19:000282 gpinitsystem:mdw:gpadmin-[INFO]:-new array must be explicitly added to this file
20210723:08:06:19:000282 gpinitsystem:mdw:gpadmin-[INFO]:-Refer to the Greenplum Admin support guide which is
20210723:08:06:19:000282 gpinitsystem:mdw:gpadmin-[INFO]:-located in the /home/gpadmin/greenplum-db/./docs directory
20210723:08:06:19:000282 gpinitsystem:mdw:gpadmin-[INFO]:-------------------------------------------------------
```

## Greenplum账户相关

创建用户admini，并设置能够创建数据库、登陆权限：

```sql
create role admini password 'SdAdm2021#' createdb login;
select rolname,oid from pg_roles;
```
对于密码忘记了，也可以修改密码信息来管理：

```sql
alter user gpadmin with password 'SdAdm2021#';
```

修改/home/gpadmin/data/master/gpseg-1/pg_hba.conf授权admini可以远程登陆：

```yaml
host     all         admini         127.0.0.1/32       md5
host     all         admini         10.188.0.2/32       md5
host     all         admini         0.0.0.0/0       md5
```

重新加载配置文件：

```shell
[gpadmin@mdw greenplum-db-new]$ pwd
/home/gpadmin/greenplum-db-new
[gpadmin@mdw greenplum-db-new]$ ./bin/gpstop  -u
20210728:01:50:22:112099 gpstop:mdw:gpadmin-[INFO]:-Starting gpstop with args: -u
20210728:01:50:22:112099 gpstop:mdw:gpadmin-[INFO]:-Gathering information and validating the environment...
20210728:01:50:22:112099 gpstop:mdw:gpadmin-[INFO]:-Obtaining Greenplum Master catalog information
20210728:01:50:22:112099 gpstop:mdw:gpadmin-[INFO]:-Obtaining Segment details from master...
20210728:01:50:23:112099 gpstop:mdw:gpadmin-[INFO]:-Greenplum Version: 'postgres (Greenplum Database) 4.2.2.4 build 1 Community Edition'
20210728:01:50:23:112099 gpstop:mdw:gpadmin-[INFO]:-Signalling all postmaster processes to reload
```

## GP服务器启动：
由于GP在容器内部手动安装，因此机器重启或者docker重启后需要手动启动GP


```shell
root@sd-cluster-01:/opt/greenplum# docker exec -it gpdb-mdw bash
[root@mdw /]# su - gpadmin
[gpadmin@mdw ~]$ gpstart  -a
通过master节点登陆到其他节点
[gpadmin@mdw ~]$ ssh sdw1
Last login: Tue Aug 10 10:19:29 2021 from gpdb-mdw.greenplum_greenplum
[gpadmin@sdw1 ~]$ exit
logout
Connection to sdw1 closed.
[gpadmin@mdw ~]$ ssh sdw2
Last login: Mon Aug  9 12:54:16 2021 from gpdb-mdw.greenplum_greenplum
[gpadmin@sdw2 ~]$ exit
logout
Connection to sdw2 closed.
```


## GP性能优化

由于GP底层使用的postgresql，因此这里的优化主要以postgresql为主,对于默认的配置，我这里主要调整几个参数：

登陆master的docker,修改：/home/gpadmin/data/master/gpseg-1/postgresql.conf配置参数

```yaml
max_connections = 800 #客户端最大会话连接数，通常防止max_connections * work_mem超出了实际内存大小
shared_buffers = 12GB    #更多的数据缓存在shared_buffers中,通常设置为实际RAM的10％
work_mem = 128MB #减少外部文件排序的可能，提高效率
maintenance_work_mem = 512M  #加速建立索引,一般CREATE INDEX, VACUUM等时用到
effective_cache_size = 20GB  #可用的系统内存
```
