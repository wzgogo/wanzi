---
title: "Git日常命令总结"
date: 2018-07-28T17:22:42+08:00
lastmod: 2018-07-28T17:22:42+08:00
draft: false
description: "git日常命令总结."
tags: ["git"]
categories: ["linux"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

## Git全局设置：
```shell
git config --global user.name "wanzi"
git config --global user.email "iwz2099@163.com"
```

## Git提交代码
```shell
git clone git@github.com:iwz2099/test.git
cd test
touch README.md
git add README.md
git commit -m "add README"
git push -u origin master
#上面命令将本地的master分支推送到远程origin分值，同时-u指定当前仓库的默认远程分支名为origin，后面就可以不加任何参数使用git push了

#包含本地分支都推送到origin主机
git push -u origin --all  

#如果本地是基于case_dev_wanzi分支开发,推送到远程case_dev分支
git push origin case_dev_wanzi:case_dev
```

## Git查询和清理
```shell
git status   #查询当前分支状态信息
git log      #查看当前分支commit信息
git log -n3  #查询最近三次提交commit 
git log -p  -2  #查询每次提交的内容差异，只显示最近2条提交记录，
git log --stat  #查询每次提交的简历统计信息
git log --pretty=oneline #将每次提交的信息一行显示,oneline可以是short,full,fuller
git log --pretty=format:"%h - %an, %ar : %s"
git log --pretty=format:"%h %s" --graph  #ASCII字符串来形象地展示你的分支、合并历史
git reflog #显示所有分支所有操作信息(包括commit,reset和已经删除的commit)
git reflog #显示最近10条日志
git grep -n   wanzi  #搜索提交历史和工作目录
git grep --count   wanzi  #搜索显示次数
git clean  #移除没有忽略的未跟踪文件
git clean -d  #清理工作目录
git clean -d -n  #-n测试清理工作目录，实际未清理
```

## Git分支操作：
```shell
git branch #查看当前分支
git branch -v #查看每个分支的最后一次提交
#基于远程master分支创建本地dev分支
git checkout -b dev  origin/master 

git branch --merged #查看哪些分支合并到当前分支
git branch --no-merged #查看哪些分支尚未合并到当前分支

#新开发一个功能，-b创建分支，并切换到新创建的分支上
git checkout -b dev-20180720-111111-wanzi
相当于：
git branch r-20180720-111111-wanzi
git checkout r-20180720-111111-wanzi

将开发后功能合并到Master主干
git checkout master
git merge  dev-20180720-111111-wanzi
```

## Git合并操作

> 基于远程master新建分支issue54,基于该分支进行开发项目
```shell
git remote add origin git@github.com:iwz2099/test.git
git  checkout -b issue54  origin/master
```

> 功能开发完后向把开发后的功能合并到远程分支
```shell
git  fetch origin    #拉去差异信息
git  merge origin/master  #合并远程分支到当前项目
git  push origin master  #推送代码到远程分支

git  log --no-merges issue54..origin/master  #查看master哪些信息没有合并到issue54分支里。
```

## Git tag操作
```shell
git tag   #列出当前项目tag信息
git tag -l 'wanzi-2018*'   #-l还可以只列出自己的分支
git tag  v1.8-20180720   #创建标签（一般临时用）
git tag -a  v1.8-20180720  -m 'wanzi version 1.8'   #-a创建tag，-m并添加说明，会存储tag信息
git  show  v1.8-20180720  #查看标签信息，这个时候可以对比下git tag -a和git tag差别
git push origin tag_name    #推送tag到远程仓库
git push origin  --tags      #一次性推送很多tag到远程仓库
git push origin :tag_name   #删除远程tag信息
git checkout -b   dev-20180720-111111-wanzi    v1.2.0  #基于某个分支新建一个tag版本
git branch -D  dev-20180720-123333-wanzi   #强制删除一个没有合并的分支
```
## Git多人协作：
```shell
#A码农基于远程master新建分支feature1
git checkout -b feature1 origin/master

#B码农基于远程master新建分支feature11
git checkout -b feature11 origin/master

#A码农先开发完，并推送代码到远程服务的feature1分支,B码农后开发完，想把改过信息合并到feature1
git fetch origin
git merge origin/feature1
git push -u origin feature11:feature1
```

## Git取消暂存区文件和取消文件修改
```shell
#代码git add到缓存区，并未commit提交，回滚代码
git reset HEAD .  或者
git reset HEAD a.txt
这个命令仅改变暂存区，并不改变工作区，这意味着在无任何其他操作的情况下，工作区中的实际文件同该命令运行之前无任何变化

git checkout --  .    #取消路径所有修改，回到上一个commit版本
git checkout -- a.txt #取消a.txt所有改动，回到上一个commit版本
```

## Git回滚到上次提交版本
```shell
git log
git reset --hard  <commit_id>  #回退一次提交的commit id
git reset --hard HEAD^  #回退到上一版本 
git reset --hard HEAD^^ #回退到倒数第二版
git reset --hard HEAD~4 #回退到倒数第四版

git push origin HEAD --force # 强制提交一次，之前错误的提交就从远程仓库删除(拥有项目管理权限的时候一定要慎重)
```

## Git回滚master代码到某个tag版本

- 基于release tag新建回滚临时分支，并将分支强制推送到远程master

```shell
git checkout -b rollback_20180720  r-20180720-111225-wanzi
```
- gitlab的项目setting里master项目管理员取消master保护

- 强制推送rollback_20180720分支到远程master主干(这个过程会强制回退到r-20180720-111225-wanzi这个版本)
```shell
git push origin rollback_20180720:master -f
```

## Git其他操作
- 提交信息写错，还没提交,修改提交信息
```shell
git commit --amend --only -m 'fix: 修复一个bug'  #修改提交信息
```
- 提交用户名和邮箱写错了，还没提交,修改提交信息
```shell
git commit --amend --author="wanzi <iwz2099@163.com>"
```
