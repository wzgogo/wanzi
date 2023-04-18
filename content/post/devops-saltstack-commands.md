---
title: "saltstack之salt命令大全"
date: 2018-02-10T16:22:42+08:00
lastmod: 2018-02-10T16:22:42+08:00
draft: false
description: "saltstack常用命令"
tags: ["saltstack", "salt"]
categories: ["devops"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

## 常用命令
```shell
salt -N 'ceph' test.ping     #测试连通性
salt -E '^server10*' test.ping           #正则匹配连通性
salt -S 192.168.150.101 test.ping      #根据agent ip地址匹配执行
salt -S 192.168.150.0/24 test.ping     #根据agent ip地址匹配执行
salt -N 'ceph' cmd.run 'df -Th' #按组查看磁盘使用
salt -N 'ceph' cmd.exec_code python 'import os; print os.system("df -Th")' #python代码执行
salt -N 'ceph' cmd.exec_code perl 'print scalar localtime' #perl代码执行
salt -G 'osrelease:6.3' cmd.run 'python -V' #根据grans信息过滤后执行
salt '*'  smbios.get system-serial-number #获取服务器硬件序列号
```
## 远程执行脚本：
```shell
salt -N 'ceph' salt://scripts/runme.sh  #远程执行脚本
```

## 远程拷贝文件：
```shell
salt -N 'ceph' cp.get_file salt://files/opencdn/a.txt /tmp/a.txt #agent端从master拉取文件到/tmp下
salt -N 'ceph' cp.get_file salt://files/opencdn/dir1/  /tmp   #agent端从master拉取目录到/tmp下
salt -N 'ceph' cp.get_url http://www.baidu.com  /tmp/index.html #下载url
salt -N 'ceph' cp.push /etc/fstab   #提取指定节点文件到master,默认在/var/cache/salt/master/minions/minion-id/files目录下
salt -N 'ceph' cp.push_dir /etc/modprobe.d/ glob='*.conf' #提取节点指定目录的指定配置文件
salt-cp '*' fstab /etc/fstab   #拷贝当前目录下fstab文件到节点/etc/fstab
salt-cp -E 'gpu0[1-2][1-5]' fstab /etc/fstab    #正则匹配拷贝文件
salt-cp -G 'os:CentOS*' fstab /etc/fstab   #grains匹配拷贝文件    
```

## 获取磁盘信息
```shell
salt -N 'ceph' disk.percent /data01  #获取磁盘使用率
salt -N 'ceph' disk.nodeusage /data01  #获取磁盘inode使用
salt -N 'ceph' xfs.info /data01 #获取xfs分区信息
```
## 文件操作
```shell
salt -N 'ceph' file.chown /etc/passwd root root   #修改文件权限
salt -N 'ceph' file.copy  /etc/passwd  /tmp/passwd #拷贝文件
salt -N 'ceph' file.directory_exists /etc/passwd  #判断文件是否存在
```
## 用户信息操作
```shell
salt -N 'ceph' user.list_users  #列出系统所有系统账号
salt -N 'ceph' user.info  lianghaiqiang  #列出用户信息
salt -N 'ceph' user.delete lianghaiqiang remove=True force=True #强制删除用户
```
## 时区、重启、任务计划
```shell
salt -N 'ceph' timezone.get_zone  #获取时区
salt -N 'ceph' timezone.set_zone #设置时区
salt -N 'ceph' system.reboot #重启系统
salt -N 'ceph' cron.list_tab root  #列出root计划任务
salt -N 'ceph' cron.raw_cron root  #列出root计划任务，文本格式
```
## 文件解压缩：
```shell
salt -N 'ceph' archive.cmd_unzip template=jinja /tmp/zipfile.zip /tmp/{{grains.id}}/ excludes=file_1,file_2
salt -N 'ceph' archive.cmd_unzip /tmp/zipfile.zip /home/strongbad/ excludes=file_1,file_2
salt -N 'ceph' archive.cmd_zip template=jinja /tmp/zipfile.zip /tmp/sourcefile1,/tmp/{{grains.id}}.txt
```
## 网络
```shell
salt -N 'ceph' network.dig www.baidu.com  #dig解析
salt -N 'ceph' network.get_hostname  #获取主机名
salt -N 'ceph' network.hw_addr eth0  #获取MAC地址
salt -N 'ceph' network.ip_addrs  #获取IP地址
salt -N 'ceph' network.ping www.baidu.com  timeout=3 #ping信息
salt -N 'ceph' pkg.install  zlib   #安装软件包
```

## 获取对象帮助信息：

如：grains/disk/pillar/pip/pkg
```shell
salt -N 'ceph' sys.doc  sys  #获取sys系统帮助信息
salt 'gpu054' sys.list_functions grains  #查询grains属性用法
salt 'gpu054' sys.doc grains    #查询grains用法帮助
salt 'gpu054' sys.doc disk    #查询disk用法帮助

salt 'gpu054' sys.list_functions  pillar #查询pillar用法
salt 'gpu054' sys.doc  pillar #查询pillar用法帮助
salt 'gpu054' pillar.item  #查询pillar支持数据源

salt 'gpu054' sys.list_modules #查询minion支持模块
salt 'gpu054' sys.list_functions  pip #查询pip模块支持方法
salt 'gpu054' sys.doc  pip #查看pip模块帮助

salt 'gpu054' sys.list_state_modules #查询所有states列表
salt 'gpu054' sys.list_state_functions pkg #查询pkg方法
salt 'gpu054' sys.state_doc pkg #查询pkg帮助
```
## Jobs管理：
```shell
salt-run jobs.active   #查看所有minion当前正在运行的jobs(在所有minions上运⾏saltutil.running)
salt-run jobs.lookup_jid <jid>        #从master jobs cache中查询指定jid的运行结果
salt-run jobs.list_jobs    #列出当前master jobs cache中的所有job
salt 'xxxx' saltutil.kill_job  <jid>  #master上kill掉某个job
```
