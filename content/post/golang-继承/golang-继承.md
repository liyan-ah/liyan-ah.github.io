---
title: golang 封装
author: 李岩
tags:
  - golang
categories: []
date: 2021-04-08 21:16:00
---
> golang作为一种高级语言，实现了面向对象语言的封装、继承、多态的特性。本文简要介绍下golang面向对象的这些特性。
<!-- more-->
# 封装
## 访问权限控制
`Golang`采用首字母大小写的方式控制访问权限。举个例子：

```  
// lib/pub.go
package learn

type A struct{  // 定义对象
    private int
    Public  string
}

func NewA(private int, public string) A{
    return A{private: private, Public: public}
}

func (a A) PrintInf(){ // 可通过a.PrintInf() 访问该函数
	print(a.private, a.Public)
}

var (
    private = 1
    Public = "aa"
)

// main.go
package main
import (
    "go_learn/lib"
)


func main(){
    a := learn.NewA(1, "aa")
    print(a.Public, a.private) // a.private不可包外访问，编译报错

    print(learn.private, learn.Public) // learn.private不可包外访问，编译报错
}  

```
和 `C++/Java/Python`等常见的面向对象语言不同，`Golang`的结构体中不支持函数的定义。某个结构体的函数，可以通过函数名前的归属生命来表示。
## 访问结构体私有成员
这是个很有意思的话题。`C++`和`Python`都是有方法可以越过结构体的访问限制的，`Golang`通过`unsafe.Pointer`类型的转换也可以达到相同的目的。举个例子：

```
// learn/pub.go
package learn

type A struct{
    private int
    Public  string
}

func (a A) PrintInf(){
    print(a.private, a.Public)
    return
}

func NewA(private int, public string) A{
    return A{private: private, Public: public}
}

var (
    private = 1
    Public = "aa"
)

// main.go
/* 测试golang封装、继承、多态特性
 */
package main
import (
    "unsafe"

    "go_learn/lib"
)

type AA struct{
    Private int
    Public  string
}

func main(){
    a := learn.NewA(1, "aa")
    //print(a.Public, a.private)

    //print(learn.Public, learn.private)
    p := unsafe.Pointer(&a)
    aa := &AA{}
    aa = (*AA)(p) // golang unsafe.Pointer 更加接近 C/C++中指针的用法，编译器进行的校验较少；
    print(aa.Private, aa.Public) // 可以正常运行。
}
```

# 总结
简单备注了下`Golang`封装的特性。后续再备注下继承、多态的使用吧。由于`Golang`采用鸭子式的继承检查思想，继承和多态的特性使用会相对较繁琐。
