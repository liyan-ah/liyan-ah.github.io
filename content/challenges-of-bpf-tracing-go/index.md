+++
title = "challenges of bpf tracing go"
author = "李岩"
date = 2022-04-19T14:33:00+08:00

[taxonomies]
tags = ["code", "bpf", "go"]
categories = ["程序人生"]
+++
> goroutine 开销为 2KB（最少），对比线程 2MB 的开销，有明显的优势。当goroutine 栈资源不足时，runtime 会将整个 goroutine stack 拷贝、重新分配空间。


> Instead of using a thread for every goroutine, Go multiplexes goroutines across multiple threads ("M:N scheduling"). So instead of each thread having a default 2MB stack, each goroutine has a tiny 2KB stack that's managed by the runtime instead of the operating system. When the program needs to grow the stack for a goroutine and there's not enough room, the runtime copies the entire goroutine's stack to another place in memory where it has enough room to expand.  


[Challenges of BPF Tracing Go](https://blog.0x74696d.com/posts/challenges-of-bpf-tracing-go/)