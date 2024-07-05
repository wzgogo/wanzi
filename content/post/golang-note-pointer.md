---
title: "Go学习笔记 - 指针详解"
date: 2023-05-22T18:22:42+08:00
lastmod: 2023-05-22T18:22:42+08:00
draft: false
description: "Go学习笔记 - go指针详解"
tags: ["golang", "指针"]
categories: ["golang"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---


## 指针是什么？

在Go语言中，指针是一个变量，它存储了另一个变量的内存地址。因此，指针变量指向的是另一个变量的内存地址，而不是变量本身。

声明指针变量时，需要在变量名前加上 *，表示这是一个指针变量，例如：

```go
var p *int
```

这表示声明了一个名为 p 的指向整型变量的指针。使用 & 运算符可以取得一个变量的地址，例如：
```go
x := 10
p := &x
```

这表示取得变量 x 的地址，并将地址赋给指针变量 p。

使用指针访问变量时，需要使用 * 运算符，它表示解引用操作符，例如：
```go
x := 10
p := &x
fmt.Println(*p)
```

这表示打印出指针变量 p 指向的变量的值，即变量 x 的值。

除了声明指针变量和获取变量的地址之外，还可以使用 new 函数来创建指针变量。例如，创建一个指向整型变量的指针：
```go
p := new(int)
```

这将创建一个新的整型变量，并返回它的地址，然后将地址赋给指针变量 p。

指针还可以用于函数参数和返回值，以便在函数调用之间共享数据。

在使用指针时需要小心，因为如果指针指向一个无效的内存地址，程序可能会崩溃或产生不可预测的行为。因此，使用指针时需要确保指针指向的内存地址是有效的。

## 指针使用场景

### 通过指针函数传参

指针可以用于函数参数的传递，当一个函数需要修改实参的值时，可以将实参的地址作为形参传递给函数，通过操作指针来达到修改实参的值的目的。例如：
```go
func modify(s *string) {
    *s = "hello, world"
}

func main() {
    s := "hello"
    modify(&s)
    fmt.Println(s) // 输出：hello, world
}
```


### 通过指针访问结构体字段

通过指针可以更方便地访问结构体中的字段，特别是当结构体很大时，传递指针比较传递整个结构体更高效。例如：
```go
type Person struct {
    Name string
    Age  int
}

func main() {
    p := &Person{
        Name: "Alice",
        Age:  18,
    }
    p.Name = "Bob"
    fmt.Println(p) // 输出：&{Bob 18}
}
```

### 通过指针动态分配内存

通过指针可以在运行时动态地分配内存，比如使用 new() 函数创建一个新的变量并返回它的地址。例如：
```go
func main() {
    p := new(int)
    *p = 42
    fmt.Println(*p) // 输出：42
}
```

### 传递变长参数

在Golang中，可以使用指针传递变长参数。这是因为在使用变长参数时，传递的是一个切片（slice），而切片本身就是指向一个数组的指针。这样的操作，可以避免拷贝大量的数据。

例如，考虑以下函数，它接受一个变长参数并将其打印出来：
```go
func printArgs(args ...int) {
    for _, arg := range args {
        fmt.Println(arg)
    }
}
```

现在，如果要将切片作为参数传递给另一个函数，可以使用指针：
```go
func printArgsPtr(args *[]int) {
    for _, arg := range *args {
        fmt.Println(arg)
    }
}

func main() {
    args := []int{1, 2, 3}
    printArgsPtr(&args)
}
```
在这个示例中，我们定义了一个新函数printArgsPtr，它使用指针作为参数来接收切片。函数体内，我们使用*args来解引用指针，从而得到实际的切片，并遍历它以打印每个元素。在主函数中，我们创建了一个切片，并将其地址传
递给printArgsPtr函数。

需要注意的是，在使用指针传递变长参数时，如果切片为空，则不能传递nil指针，而应该传递一个空的切片。这是因为在Golang中，使用空的切片和nil指针的含义不同,后面我会介绍。


除了上述几个方面，指针在一些特定的场景下也非常有用，比如在处理数据结构时，通过指针可以更高效地操作链表、树等数据结构。但需要注意，过度使用指针可能会导致代码难以维护，需要谨慎使用。


## 常见空数据类型

在 Go 语言中，有一些空数据类型，包括：


* 空指针（nil pointer）：指向一个不存在地址的指针，一般用 nil 表示。
* 空切片（nil slice）：长度为 0 的切片，一般用 nil 表示。
* 空字典（nil map）：长度为 0 的字典，一般用 nil 表示。
* 空接口（nil interface）：类型和值均为 nil 的接口，一般用 nil 表示。
* 空通道（nil channel）：没有分配存储空间的通道，一般用 nil 表示。


nil表示一个空指针，即指针变量未指向任何地址，可以表示指针、接口、map、通道和函数类型的零值，而不能用来表示其他类型的零值，如int、float、bool等

这些空数据类型的定义均为对应类型的零值，其中指针、切片、字典、接口和通道的零值都是 nil，表示这些类型的默认值为 nil，即没有指向任何实际数据的指针、切片、字典、接口和通道。因此在实际使用这些数据类型时，可以
将其初始化为零值，也可以显式地赋值为 nil。


例如:
```go
var s []int = nil // 定义一个空切片
var p *int = nil  // 定义一个空指针
var m map[string]int = nil //定义一个空字典
var i interface{} = nil //定义一个空接口
var c chan int = nil //定一个空通道
```

需要注意，如下几点:
* nil不是关键字，而是预定义的标识符。在 Go 中，可以将 nil 赋给任何类型的指针、切片、字典、函数和通道等变量。
* 空切片和空字典在使用前需要初始化，否则会出现 panic 异常。
* 空通道需要使用 make 函数初始化后才能使用。
* 空指针可以直接使用，因为它的值为 nil。
* 空接口不需要初始化即可直接使用。因为空接口本质上就是一个类型为 interface{} 的变量，它可以接受任何类型的值，包括 nil。所以在使用空接口时，如果没有显式地赋值，它就是一个空接口，可以直接使用。

## 避免空指针异常

在使用指针类型的变量时，需要进行 nil 检查以避免出现空指针异常。

例如：

### 整型指针类型
```go
var p *int
fmt.Println(p == nil) // 输出 true
```
代码中，p 是一个整型指针类型的变量，因为没有被初始化，它的值为 nil。可以通过比较 p 是否为 nil 来检查它是否为空指针。


### 整型切片类型
```go
var s []int
fmt.Println(s == nil) // 输出 true
```
代码中，s 是一个整型切片类型的变量，因为没有被初始化，它的值为 nil。可以通过比较 s 是否为 nil 来检查它是否为空切片。

### 字典类型
```go
var m map[string]int
fmt.Println(m == nil) // 输出 true
```
代码中，m 是一个字典类型的变量，因为没有被初始化，它的值为 nil。可以通过比较 m 是否为 nil 来检查它是否为空字典。


### 函数类型
```go
var f func(int) int
fmt.Println(f == nil) // 输出 true
```
代码中，f 是一个函数类型的变量，因为没有被初始化，它的值为 nil。可以通过比较 f 是否为 nil 来检查它是否为空函数。


### 整型通道类型
```go
var ch chan int
fmt.Println(ch == nil) // 输出 true
```
代码中，ch 是一个整型通道类型的变量，因为没有被初始化，它的值为 nil。可以通过比较 ch 是否为 nil 来检查它是否为空通道。

最后，特别注意，空数组、空切片和空 map是可以使用 len() 函数求长度，结果都为 0。但是，空指针、空接口、空通道是不能使用 len() 函数求长度，否则会引发运行时错误。

这是因为，在 Go 中，len()函数返回一个值的长度，这个长度是预先定义好的，通常是由编译器计算得出的。空切片、空数组和空字典等类型变量的长度为0，因为它们在内存中已经有了一块空间，所以可以使用len()函数获取它们
的长度。

而对于空接口和空通道，它们是不存储数据的，所以它们的长度是没有意义的，因此在使用len()函数时会引发运行时错误。如果要判断它们是否为空，应该使用nil值进行判断。