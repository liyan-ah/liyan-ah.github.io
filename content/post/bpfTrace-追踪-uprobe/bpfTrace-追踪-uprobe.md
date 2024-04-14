---
title: bpftrace 无侵入遍历golang链表
author: 李岩
tags:
  - ebpf
  - golang
  - bpftrace
  - uprobe
categories:
  - 程序人生
date: 2022-07-22 21:48:00
---
> `bpftrace` 基于 `bcc` 进行开发的工具，语法简洁、功能强大。用其分析`Linux` 环境下的程序会很方便。本文构造了一个入参为链表头节点的函数使用场景，通过使用`bpftrace`无侵入遍历链表成员的方式，介绍`bpftrace` `attach uprobe` 的使用。更多使用说明见:[bpftrace官网使用文档](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md)

<!--more-->

# 执行结果
下面直接给出执行结果。可以看到，通过`bpftrace`脚本输出的结果与代码中实际遍历的结果相同。
```
sudo ./handle.bt  // 先启动监听
Attaching 1 probe...  // 启动后停止在这里
=== enter main.handle.  // 目标程序执行后输出
name: Alice
age : 10
name: Bob
age : 11
name: Claire
age : 12
=== total node: 3

./demo  // 再执行目标程序
cur name: Alice, cur aget: 10
cur name: Bob, cur aget: 11
cur name: Claire, cur aget: 12
```

# 示例说明
系统环境如下： 
```
Linux 4.18.0-193.6.3.el8_2.v1.2.x86_64
bpftrace v0.14.0-72-g6761-dirty
go version go1.16.15 linux/amd64
```

示例环境目录：
```
.
├── demo
├── go.mod
├── handle.bt
└── main.go
```

其中：

```
// main.go
package main

import (
    "context"
    "fmt"
)

type Student struct {
    Name string
    Age  int64
    // Comment [600]Byte 这样会使得这个问题变得很麻烦，hhh
    Next *Student
}

// 添加如下配置以防止函数被编译优化掉
//go:noinline
func handle(ctx context.Context, student *Student) {
    for cur := student; cur != nil; cur = cur.Next {
        fmt.Printf("cur name: %s, cur aget: %d
", cur.Name, cur.Age)
    }
    return
}

func main() {
    first := &Student{
        Name: "Alice",
        Age:  10,
        Next: &Student{
            Name: "Bob",
            Age:  11,
            Next: &Student{
                Name: "Claire",
                Age:  13,
                Next: nil,
            }}}
    handle(context.Background(), first)
}
```

```
// handle.bt
#!/bin/bpftrace

// 当目标结构体较小时(使得整体栈开销 < 512Byte)，可以直接构造使用
struct student{
    u64 name_ptr;
    u64 name_length;
    long  age;
    struct student *next;
};

uprobe:./demo:"main.handle"
{
    printf("=== enter main.handle.
");

    $cur = (struct student *)sarg2;
    if ($cur == 0){
        printf("input param is nil.
");
        return;
    }
    $node_count = 1;
    unroll(10){  // 这里定义的最大节点数量为10
        printf("name: %s
", str($cur->name_ptr, $cur->name_length));
        printf("age : %d
", $cur->age);
        $cur = $cur->next;
        if ($cur == 0){
            printf("=== total node: %d
", $node_count);
            return;
        }
        $node_count += 1;
    }

    printf("==== meet max
");
    return;
}
```
在编译完成`main.go`后，通过`sudo bpftrace -l "uprobe:./demo:*" > uprobe.info`的方式，可以获取`demo`中所有可以`attach`的`uprobe`信息。这里说明下取`student*`指针值时，为什么取`sarg3`。`bpftrace`中对内存传参的参数支持是`sarg0, sarg1, sarg2...`，且每个参数实际只对应`8Byte`大小的空间。对于`func handle(ctx context.Context, student *Student)`函数来说，由于`context.Context`实际占用`2*8Byte`的空间（见[golang常见类型字节数](https://blacker1230.github.io/2022/06/06/golang-%E5%B8%B8%E8%A7%81%E7%B1%BB%E5%9E%8B%E5%AD%97%E8%8A%82%E6%95%B0/))，因此需要使用`sarg2`来取`student`的值，而非直觉上的`sarg1`。  
整个过程比较简单、明了。只要拥有`root`权限，基本上可以对系统内的任何进程进行详细的分析。
