---
title: golang GMP
author: 李岩
date: 2021-04-15 21:39:29
tags:
  - golang
  - GMP
categories:
  - code
---
> 写点东西还是难，果然还是搬运工来的轻松些。今天搬运点Golang的`GMP`模型看看。最近在准备一篇`Golang`的`GC`实践。慢慢搞吧。

# 前言
`Golang`作为语言层面支持并发的语言，使用`go`可以让搬砖体验飞起。但是从直觉来说，事情并没有这么简单：从操作系统层面来说，进程和线程是操作系统认可的并行机制。协程以及`Golang`的所谓*纤程*是期望一堆程序员期望将操作系统的工作拿过来，以满足一些优化的效果。所以诸如`Python`的协程以及`Golang`的纤程，总是能够对应到操作系统认可的执行单元上。对于`Python`的协程还好理解一些，是严格运行在自己的线程里的，只是语言层面实现了线程内的上下文切换优化。所以对于`CPU`密集型的操作，仅使用协程是无法达到优化效果的：这种场景下`Python`会推荐多进程。相比起来，`Golang`的`go`野心更大一些：期望给用户以`go`作为接口，在语言内实现与操作系统调度单元的交互。`Golang`里实际的调度模型是`GMP`。  
<!--more-->
# 搬运
这里搬运一些文章，介绍`GMP`。  
[[典藏版] Golang 调度器 GMP 原理与调度全分析](https://learnku.com/articles/41728) 从单进程开始介绍，后面的调试部分能学到一些东西  
[Go语言学习 - GMP模型](https://juejin.cn/post/6844904034449489933) `G`调度这块说的比较详细，可以看看  
[6.5 调度器 #](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/) 日常膜拜  
# 思考
* `goroutine`还是运行在一个进程里的。多线程想对比多进程，稳定性上会差一些：如果线程内出现了coredump等异常，整个进程可能就退出了。所以`goroutine`运行在一个进程内，会不会一个`g`出现了crash，整个程序崩溃？
* `Python`的进程及线程，解释器层面分别使用了`C`的`fork`以及`pthread`(Linux)进行实现。`g`的实现是怎么样的。
