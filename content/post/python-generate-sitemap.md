---
title: "Python脚本生成一个sitemap网站地图"
date: 2024-03-10T10:22:42+08:00
lastmod: 2024-03-10T10:26:42+08:00
draft: false
description: "Python脚本生成一个sitemap网站地图"
tags: ["python", "sitemap"]
categories: ["python"]              
author: "wanzi"                 

comment: true   # 关闭评论
toc: true       # 文章目录
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true	 # 关闭打赏
mathjax: true    # 打开 mathjax
---

最近在帮朋友研究一个CMS站群，发现CMS没有sitemap功能，服务器上已经安装了python3，所以只能临时用脚本生成一个，这里记录一下。

## 脚本内容：

```python
#!/usr/bin/env python

import datetime
import mysql.connector
import xml.etree.cElementTree as ET
from lxml import etree

# 数据库连接参数
config = {
    'host': '10.80.0.3', # 例如：'192.168.1.100'
    'user': 'wnote_r',
    'password': 'Wnote#Pss2024',
    'database': 'wnote',
    'raise_on_warnings': True
}

#sitemap路径
sdir = '/opt/wwwroot/'

def mselect(site, n1, n2):
    # 连接数据库
    cnx = mysql.connector.connect(**config)
    cursor = cnx.cursor()

    # 获取文章ID
    article_ids = []
    tag_ids = []

    # 查询文章表中的所有记录
    query1 = "select id from article order by newstime desc  limit {};".format(n1)
    cursor.execute(query1)
    for row in cursor:
        article_ids.append(row[0])

    # 查询所有标签
    query2 = "select id from phome_ecms_book order by newstime desc limit {};".format(n2)
    cursor.execute(query2)
    for row in cursor:
        tag_ids.append(row[0])

    # 生成URL列表
    article_urls = [f"https://{site}/article/{id}.html" for id in article_ids]
    tag_urls = [f"https://{site}/tags/{id}.html" for id in tag_ids]

    # 合并URL列表
    urls = article_urls + tag_urls

    # 创建Sitemap的XML树结构
    root = ET.Element('urlset',
                  {'xmlns': 'http://www.sitemaps.org/schemas/sitemap/0.9',
                   'xmlns:xsi': 'http://www.w3.org/2001/XMLSchema-instance',
                   'xmlns:mobile': 'http://www.baidu.com/schemas/sitemap-mobile/1/',
                   'xsi:schemaLocation': 'http://www.sitemaps.org/schemas/sitemap/0.9 http://www.sitemaps.org/schemas/sitemap/0.9/sitemap.xsd'})

    # 生成sitemap文件,构建URL元素
    for url in urls:
        s_url = ET.SubElement(root, 'url')
        s_loc = ET.SubElement(s_url, 'loc')
        s_loc.text = url
        s_lastmod = ET.SubElement(s_url, 'lastmod')
        s_lastmod.text = datetime.now().strftime('%Y-%m-%dT%H:%M:%S+08:00')
        s_changefreq = ET.SubElement(s_url, 'changefreq')
        s_changefreq.text = 'always'
        s_priority = ET.SubElement(s_url, 'priority')
        s_priority.text = '0.95'


    # 生成XML字符串
    sitemap_str = ET.tostring(root, encoding='unicode')

    # 保存到文件
    with open(spfile, "w", encoding="utf-8") as f:
        f.write(sitemap_str)

    # 关闭数据库连接
    cursor.close()
    cnx.close()

with open('sites.list', 'r')  as files:
    for tmp in files:
        site = tmp.strip().split()[0]
        spfile = sdir + site + '/' + tmp.strip().split()[1]
        n1 = tmp.strip().split()[2]
        n2 = tmp.strip().split()[3]
        print(site, spfile)
        mselect(site, n1, n2)
```

其中sites.list内容为：
```shell
www.xxxxx.com 100 120
www.xxxxx.com 200 300
```

## 总结

其实，Python脚本写起来比较便捷，如果对性能有要求的话，还是用Golang比较好一些。

