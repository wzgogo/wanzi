---
title: "Go学习笔记 - Go基础知识复习"
date: 2023-03-17T18:22:42+08:00
lastmod: 2023-03-17T18:22:42+08:00
draft: false
description: "Go学习笔记 - Go基础知识复习"
tags: ["golang", "go笔记"]
categories: ["golang"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---


## 一、基础知识
## 1、基本数据类型
### 值类型：
```yaml
bool（长度1Byte或1个字节）
byte(1个字节，uint8别名)
rune（4个字节，int32别名，-231~231）
int(32 or 64，长度4或8字节)
int8（长度1字节）、int16（长度2字节）、int32（长度4字节）、int64（长度8字节）
uint(32 or 64，长度4或8字节)
uint8(长度1字节)、uint16(长度4字节)、uint32(长度4字节)、uint64(长度8字节)
float32(长度4字节)、float64(长度8字节)
string     //一串固定长度字符连接起来的序列，底层为byte数组
complex64、complex128（默认）//复数类型
array   //数组（固定长度）
struct  //结构体
```

### 引用类型（指针类型）：
```yaml
slice  //序列数组，默认值nil
map //映射，默认值nil
chan // 管道，默认值nil
```

### 2、编码&&字符串
> golang默认编码是UTF-8

* Unicode：一种全球字符编码标准，可变长编码，实现方式有多种，其目的是为了统一所有文字编码，比如UTF-8、UTF-16等，长度范围为1-4字节，默认英文占1Byte，中文占2Byte；每个字符对应一个十六进制的数字，Unicode字符串长度使用utf8.RuneCountInString()函数获取。
* UTF-8：是可变长度字符编码，也是Unicode实现方式之一，英文占1Byte，中文占3Byte，emoji占4Byte。

GO中字符有两种，一种byte，一种rune

* byte：是uint8别名，常用来处理ASCII字符，表示ASCII码的一个字符串，[]byte(s)，将s转换为byte数组。
* rune：是int32别名，等价于int32类型，通常用于表示一个unicode字符的码点（code point），可以处理一切字符，使用方法为[]rune(s)，将s转换为unicode数组


> string：一串固定长度字符连接起来的序列，底层为byte数组，字符串是不可改变的，当字符串只有数字字母，可以转换为[]byte或者[]rune，当有汉字等多字节编码的就必须转换为[]rune，最通用的方法是转换为[]rune(s)

* 单引号：用来表示byte类型或rune类型，默认是rune类型，单引号里面内容为Ascii码字符，ASCII字符串长度使用len()函数获取；
* 双引号：表示可以解析字符串字面量，支持转义，但是不能用来引用多行；
* 反引号：用于创建原生的字符串字面量，不支持任何转义序列，支持多行，多用于书写多行消息、html、正则；


### 3、内置函数
```yaml
append  // 用于追加元素到数组、切片中，返回修改后的数组和切片
close   // 主要用于关闭channel
delete // 从map中删除key对应value
panic  // 停止常规goroutine(panic或recover：用于错误处理)
recover // 允许程序定义goroutine的panic动作
real  // 返回complex的实部(complex、real、imag用来创建和操作复数)
imag // 返回complex的虚部
make // 用来分配内存，返回Type本身（只能用于slice、map、channel）
new // 用来分配内存，主要用于分配值类型，比如int、struct，返回指向type的指针。
cap // 用来返回某个类型最大容量，只能用于切片和map
copy // 用来复制连接slice，返回复制的数目
len // 获取长度，比如string、array、slice、map、channel
print、println  // 底层打印函数
```


## 二、常用占位符：
```yaml
占位符     说明                           举例                   输出
%v      相应值的默认格式。            Printf("%v", people)   {zhangsan}，
%+v     打印结构体时，会添加字段名     Printf("%+v", people)  {Name:zhangsan}
%#v     相应值的Go语法表示            Printf("#v", people)   main.Human{Name:"zhangsan"}
%T      相应值的类型的Go语法表示       Printf("%T", people)   main.Human
%%      字面上的百分号，并非值的占位符  Printf("%%")            %
%t      布尔占位符  true 或 false。   Printf("%t", true)       true
%c      字符占位符，相应Unicode码点所表示的字符 Printf("%c", 0x4E2D)        中
%d      整形占位符，十进制表示               Printf("%d", 0x12)          18
%s      字符串占位符（string类型或[]byte)   Printf("%s", []byte("Go语言"))  Go语言
%p      指针占位符，十六进制表示，前缀 0x     Printf("%p", &people)             0x4f57f0  
```

## 三、数组&&切片&&map
### 1、数组
var 数组变量名 [元素数量]类型
数组长度必须是常量，一旦定义，长度不能变，数组中的元素是可以变化的
```golang
var  a[5]int  
```
声明一个长度为5，类型为int的数组
```golang
a := [8]int{0, 1, 2, 3, 4, 5, 6, 7}
b := a[3:6]
// 8 8
fmt.Println(len(a), cap(a))                        
//值：[3 4 5]，长度：3, 容量：5
fmt.Printf("值：%d，长度：%d, 容量：%d", b, len(b), cap(b)) 
```

### 2、切片
拥有相同类型元素的可变序列，它是基于数组做的一层封装，非常灵活，支持自动扩容，切片是一个引用类型，内部包含地址、长度(len)、容量(cap)

> 语法：var  切片变量名 []类型

```Golang
var a []string //声明一个长度为0，类型为字符串切片。
var a[]string{} //声明一个长度为0，类型为字符串，并初始化

var myslice = []string{"北京", "上海", "深圳"}

for i := 0; i < len(myslice); i++ {
   fmt.Println("for循环:", myslice[i])
}
```

### 3、map
一种无序，基于key/value的数据结构，go中map是一种引用类型，必须使用make初始化才能使用。

> 语法: map[keytype]valuetype

map默认初始值为nil，需要使用make()函数进行分配内存，其中cap表示map的容量，该参数虽然不是必须的，获取map容量不能用cap，cap返回的是数组切片分配的空间大小；获取map的容量，可以使用len函数。

```Golang
userInfo := map[string]string{
   "username": "xiaoli",
   "password": "1234556",
}
for k, v := range userInfo {
   fmt.Println(k, v, len(userInfo))
}
```

## 四、指针：
go语言中引用数据类型不仅要声明它，还要为它分配内存，否则我们的值就没发存储；而对于值类型的声明就不用分配内存，因为他们已经默认分配好内存。
```yaml
指针地址：&a
指针取值：*&a
指针类型：&a --> *int
&取值，*跟进地址取值
```

new：为类型申请一片内存空间，并返回指针。 比如对结构体实例化，返回的是结构体的指针地址

make：只用于chan、map以及、slice的内存创建，而且它的返回类型就是这三个类型本身，而不是他们的指针类型。

## 五、结构体

go语言里没有类的概念，也没有支持类的继承等面向对象的概念；go语言中主要通过结构体内嵌再配合接口比面向对象具有更高的扩展性和灵活性。

### 1、结构体实例化方法：

直接赋值(var)、new、键值对实例化


```golang
package main

import "fmt"

// 定义一个结构体
type member struct {
	name string
	city string
	age  int
}

func main() {
	var p1 member //直接赋值实例化结构体
	p1.name = "小明"
	p1.city = "beijing"
	p1.age = 32

	var p2 = new(member) // new关键字实例化
	p2.name = "小李"
	p2.city = "shanghai"
	p2.age = 20

	p3 := &member{
		name: "小张",
		city: "shenzhen",
		age:  18,
	} // 键值对实例化
	fmt.Printf("p1=%v \n", p1) // 输出值
	fmt.Printf("p1=%+v \n", p1) //输出结构体，并添加字段名称
	fmt.Printf("p2=%#v \n", p2) // 输出结构体字段并返回语法表示
	fmt.Printf("p3=%#v \n", p3)
}

```

输出结果：

```yaml
p1={小明 beijing 32} 
p1={name:小明 city:beijing age:32} 
p2=&main.member{name:"小李", city:"shanghai", age:20} 
p3=&main.member{name:"小张", city:"shenzhen", age:18} 

```


### 2、结构体方法和接收者

go语言的方法是一种特定类型变量的函数，这种特定类型叫做接收者，接收者的概念类似其他语言this,self等。

语法:
```golang
func (接受者变量，接受者类型)  方法名(参数列表) (返回值) {
}
```


```golang
package main

import "fmt"

// 定义一个结构体
type Person struct {
   name string
   age  int
   city string
}

// 值接收者
func (p Person) printInfo() {
   fmt.Printf("姓名：%s, 年龄：%d, 城市：%s \n", p.name, p.age, p.city)
}

// 指针接收者
func (p *Person) setInfo(name string, age int, city string) {
   p.name = name
   p.age = age
   p.city = city
}

func main() {
   // 实例化结构体
   p1 := Person{
      name: "xiaoming",
      age:  15,
      city: "tianjin",
   }
   p1.printInfo() //姓名：xiaoming, 年龄：15, 城市：tianjin
   p1.setInfo("tiantian", 18, "beijing")
   p1.printInfo() //姓名：tiantian, 年龄：18, 城市：beijing
}
```

