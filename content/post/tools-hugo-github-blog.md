---
title: "Hugo+Github搭建个人博客"
date: 2020-03-10T16:22:42+08:00
lastmod: 2020-03-10T17:22:42+08:00
draft: false
description: "hugo+github搭建个人博客"
tags: ["hugo", "github", "blog"]
categories: ["tools"]              
author: "wanzi"                 

comment: false   # 关闭评论
toc: false       # 关闭文章目录
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true	 # 关闭打赏
mathjax: true    # 打开 mathjax
---
# Hugo介绍

之前博客一直使用hexo搭建,随着用golang越来越多，一直想把博客也迁移到hugo,hugo就不用多说了go语言编写的静态网站生成器,简单、易用、高效、易扩展、快速部署.

# 安装hugo

这里以mac环境为例:
```shell
brew install hugo
hugo new site wanzi
cd wanzi
git clone https://github.com/xianmin/hugo-theme-jane.git --depth=1 themes/jane
cp -r themes/jane/exampleSite/content ./
cp themes/jane/exampleSite/config.toml ./
```
修改config.toml信息为你自己博客信息

我网站目录结构如下:
```yaml
.
├── LICENSE
├── archetypes #存放default.md，头文件格式
│   └── default.md
├── config.toml
├── content #整个网站项目全局配置
│   ├── about.md
│   └── post
├── data #存放数据或配置,可以是json|toml|yaml
├── layouts #存放的是网站的模板文件
├── public #hugo编译后生成的静态文件
│   ├── 404.html
│   ├── about
│   ├── atom.xml
│   ├── categories
│   ├── css
│   ├── favicon.ico
│   ├── fonts
│   ├── icons
│   ├── index.html
│   ├── js
│   ├── manifest.json
│   ├── mark
│   ├── posts
│   ├── robots.txt
│   ├── rss.xml
│   ├── sitemap.xml
│   └── tags
├── resources
│   └── _gen
├── static #存放图片,css,jss静态资源
└── themes #存放网站主题
    └── jane
```

# 编写文章
```shell
hugo new content/posts/git-commands-base.md
```

# 推送到github

```shell
cd wanzi/public
git init 
git remote add origin https://github.com/iwz2099/wanzi
echo "wnote.com" > CNAME
git  add -A
git commit -m "initialization"
git push -u origin master
```
如果使用个人域名,只需要在github仓库下创建CNAME文件写入自己的域名即可，这样访问wnote.com就可以访问自己博客了。
