---
title: 黑魔法--用 ebpf 构建用户空间数据的桥梁
author: 李岩
tags:
  - bpf
  - code
categories:
  - BPF
date: 2022-11-04 18:00:00
---
在之前的示例中，仅涉及到`ebpf`对用户空间数据的读取。工程性较强的如：[ebpf采集mysql请求信息及ebpf对应用安全的思考](https://liyan-ah.github.io/2022/10/21/ebpf%E9%87%87%E9%9B%86mysql%E8%AF%B7%E6%B1%82%E4%BF%A1%E6%81%AF%E5%8F%8Aebpf%E5%AF%B9%E5%BA%94%E7%94%A8%E5%AE%89%E5%85%A8%E7%9A%84%E6%80%9D%E8%80%83/)也仅是通过`urpobe`采集用户空间的数据。本文介绍点`ebpf`的“黑魔法”：将用户空间数据的读取、用户空间数据的写入结合起来，成为用户空间数据交互的桥梁。
<!--more-->

# 运行效果
在看运行效果之前，需要先看下目标示例的代码以便更好的理解本文介绍的功能：

``` 
// user/obj/obj.go
package main

import (
	"fmt"
)

var Len = 5

// 预设的 uprobe
//go:noinline
func Set(info []byte) {
	fmt.Println("info: ", string(info))
	return
}

// 预设的 uprobe
//go:noinline
func Get(info []byte) []byte {
	fmt.Printf("info addr: %p
", info)
	fmt.Println(string(info))         // 请注意这里的输出操作
	return info
}

type Third struct {
	Info string
}

func (t *Third) SetSomething(info string) {
	infoByt := []byte(info)
	// 这里假设是个约束
	infoByt = infoByt[:Len]
	Set(infoByt)  // 在这里调用预设的处理函数
}

func (t *Third) GetSomething() string {
	infoByt := make([]byte, Len, Len)
	infoByt = Get(infoByt) // 在这里调用预设的处理函数
	return string(infoByt)
}

func main() {
	third1 := Third{}
	info := "12345"
	third1.SetSomething(info)  // 请注意，这里进行写入的对象

	/* very long handle logic, many goroutines or proces happend here */

	third2 := Third{}
    // 请注意，这里读取的对象和上述执行写入的对象是完全没有关系的
	infoGet := third2.GetSomething()
	fmt.Printf("after getInfo, [%s]
", infoGet)
}
``` 

这段代码非常简单，下面进行了两次执行来说明`ebpf`达到的效果：

``` 
$ ./obj  // 第一次，没有使用 ebpf 生效。代码的正常输出结果
info:  12345
info addr: 0xc0000180f0
                         // 请注意这里
after getInfo, []

$ ./obj  // 第二次，开始执行前开启 ebpf 监听
info:  12345
info addr: 0xc0000180f0
12345                    // 请注意这里
after getInfo, [12345]
```

请关注上述示例里的注释。通过`ebpf`的`attach`，实现了数据从用户空间->`ebpf`空间->用户空间，这个过程并不关心用户代码里发生了什么，`ebpf`只关注预设的`uprobe`是怎么被调用的。

# 应用及思考
`ebpf`的这个功能显然具有很广泛的应用，但是具体的应用就需要结合业务的应用来说明了（颇有一些拿着锤子找钉子的感觉），比如：结合调用了特定埋点`sdk`的使用，能够用来对`traceId`信息的补全。  
事物自然都有两面性，`ebpf`提供了变更用户空间数据的潜力，自然就会带来风险：代码里的逻辑似乎不再靠谱了。而且，想象下将代码里的读操作，变更为删除操作，将会对用户空间的安全造成很大的破坏。

# ebpf 逻辑
之前一直是使用`bpftrace`来进行示例演示的，但是本文涉及的功能需要使用`long bpf_probe_write_user(void *dst, const void *src, u32 len)`这个`bpf-helper`函数。笔者没有找到`bpftrace`里的调用方式，因此采用`cilium-ebpf`来进行示例演示。其中涉及的主要`bpf`代码附在下面，基本表述了相对原生的`bpf-helper`的调用方法。

``` 
struct{
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(key_size, sizeof(u32));
    __uint(value_size, sizeof(u8)*5);
    __uint(max_entries, 100);
} info_map SEC(".maps");

struct event{
    u64 pid_tgid;

    u8 info[5];  // 这里的成员长度，请结合 obj.go 来看
    uintptr_t addr;
    long res;
};
typedef struct event event_t;
// Force emitting struct event into the ELF.
const struct event *unused __attribute__((unused));

struct {
  __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
} events SEC(".maps");

SEC("uprobe/main_set")
int uprobe__main_set(struct pt_regs *ctx){
    uintptr_t info_p = 0;
    u8 info[5];
    u64 pid_tgid = bpf_get_current_pid_tgid();

    SARG(ctx, 0, info_p);

    bpf_probe_read(&info, sizeof(info), (const void*)info_p);
    u32 tgid = (u32)(pid_tgid >> 32);
    bpf_map_update_elem(&info_map, &tgid, &info, BPF_ANY);

    event_t event = {};
    event.pid_tgid = pid_tgid;
    memcpy(event.info, info, sizeof(info));
    bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &event, sizeof(event));
    
    return 0;
}

SEC("uprobe/main_get")
int uprobe__main_get(struct pt_regs *ctx){
    uintptr_t info_p = 0;
    u64 pid_tgid = bpf_get_current_pid_tgid();

    void* r_info_p = NULL;
    u32 tgid = (u32)(pid_tgid >> 32);
    r_info_p = bpf_map_lookup_elem(&info_map, &tgid);
    if (r_info_p == NULL){
        return 0;
    }
    event_t event = {};
    event.pid_tgid = pid_tgid;

    SARG(ctx, 0, info_p);

    u8 info[5];
    memcpy(info, r_info_p, sizeof(info));

    memcpy(event.info, info, sizeof(event.info));

    event.res = bpf_probe_write_user((u8*)info_p, info, sizeof(info));
    event.addr = info_p;

    bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &event, sizeof(event));
    return 0;
}
```

以上，周末愉快。
