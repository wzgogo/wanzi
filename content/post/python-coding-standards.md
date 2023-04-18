---
title: "python编码规范"
date: 2018-01-20T10:22:42+08:00
lastmod: 2018-01-20T10:22:42+08:00
draft: false
tags: ["编码规范", "PEP8"]
categories: ["python"]              
author: "wanzi"                 

comment: false   # 关闭评论
toc: false       # 关闭文章目录
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true	 # 关闭打赏
mathjax: true    # 打开 mathjax
---

# 编码声明

默认不设置为ASCII,可设置为
```python
# coding=<encoding name>
```
或
```python
#!/usr/bin/python
# -*- coding: <encoding name> -*-
```
或
```python
#!/usr/bin/python
# coding: utf-8
```

# 代码编排

* 四个空格一个缩进，不建议使用tab，更不建议混合使用tab和空格
* 单行最长79字符，换行可使用反斜杠，最好使用括号
* 类和同级别函数之间空2行，类中方法之间空一行
* 模块书写顺序: 模块说明和docstring，import，globals，constants，其他；其中import又按标准、第三方、自己编写顺序摆放，同时不建议书写在一行
* 操作符左右各加一个空格，函数默认参数使用的赋值符左右省略空格，各种右括号前不要加空格;
* 注释必须使用英文，最好是完整的句子，首字母大写，句后要有结束符，结束符后跟两个空格，开始下一句。如果是短语，可以省略结束符。
* 所有的共有模块、函数、类、方法写docstrings；非共有的没有必要，但是可以写注释（在def的下一行）

# 命名规范
* 尽量单独使用小写字母‘l’，大写字母‘O’等容易混淆的字母。
* 模块命名尽量短小，使用全部小写的方式，可以使用下划线。
* 包命名尽量短小，使用全部小写的方式，不可以使用下划线。
* 类的命名使用CapWords的方式，模块内部使用的类采用_CapWords的方式。
* 异常命名使用CapWords+Error后缀的方式。
* 全局变量尽量只在模块内有效，类似C语言中的static。实现方法有两种，一是__all__机制;二是前缀一个下划线。
* 函数命名使用全部小写的方式，可以使用下划线。
* 常量命名使用全部大写的方式，可以使用下划线。
* 类的属性（方法和变量）命名使用全部小写的方式，可以使用下划线。
* 类的属性有3种作用域public、non-public和subclass API，可以理解成C++中的public、private、protected，non-public属性前，前缀一条下划线。
* 类的属性若与关键字名字冲突，后缀一下划线，尽量不要使用缩略等其他方式。
* 为避免与子类属性命名冲突，在类的一些属性前，前缀两条下划线。比如：类Foo中声明__a,访问时，只能通过Foo._Foo__a，避免歧义。如果子类也叫Foo，那就无能为力了。
* 类的方法第一个参数必须是self，而静态方法第一个参数必须是cls

# 代码检查工具
```shell
pip install flake8
```
安装flake8成功后，打开VScode，文件->首选项->用户设置，在settings.json文件中输入"python.linting.flake8Enabled": true

# 自动格式化工具
```shell
pip install yapf
```
安装yapf成功后，打开VScode，文件->首选项->用户设置，在settings.json文件中输入"python.formatting.provider": "yapf"

# 项目开发注意事项：

在开始一个空白项目的时候需要注意如下几条：

* README.md这里写你项目的简介，quick start等信息，虽然distutils要求这个文件没有后缀名，但github上如果后缀是.md的话可以直接转换成html显示。
* ChangeLog.txt该文件存放程序各版本的变更信息，也有一定的格式，参考web.py的ChangeLog.txt
* LICENES.txt 这里存放你项目使用的协议，不要编写自己的协议。
* requirements.txt 如果你的项目需要依赖其它的python第三方库，在这里一行一个写出来，可能pip install的时候能自动帮你安装
* setup.py安装脚本，后面详细介绍
* docs 里面存放你的项目文档，如概要设计，详细设计，维护文档，pydoc自动生成的文档等，强烈推荐大家使用MarkDown格式编写文档
* src 这个目录里存放项目模块的主要代码，尽量不要把模块目录直接放到根目录，模块代码目录可以在setup.py里指定的
* tests 这个目录存放所有单元测试，性能测试脚本，单元测试的文件确保以test_做前缀，这样distutils会自动打包这些文件，并且用python -m unittest discover -s ./ -p 'test_*.py' -v 可以直接执行这些测试

参考：

https://www.python.org/dev/peps/pep-0008/
https://wiki.woodpecker.org.cn/moin/PythonCodingRule
