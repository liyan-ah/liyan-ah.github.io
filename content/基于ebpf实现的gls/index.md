+++
title = "基于ebpf实现的gls"
author = "李岩"
date = 2022-11-25T01:14:00+08:00

[taxonomies]
tags = ["code", "bpf", "golang", "gls"]
categories = ["BPF"]
+++
> 虽然`golang`并不推荐使用`goid`来构建`gls`(`goroutine local storage`)，仍然有着很多的实现`gls`并使用的尝试。[github-gls](https://github.com/jtolio/gls)这里是一个常见的实现，基本表述了`golang`里`gls`的实现思路：获取`goid`，基于`goid`构建一个存储。本文中笔者尝试基于`ebpf`来构建一个`golang`的`gls`。

# 基本功能
本文中基于`ebpf`实现的`gls`具有如下功能：  
* 基于`goid`的存储。即`map[goid]=value`；
* 基于`goroutine`派生关系设置的`value`缺省值。即`map[goid=1]=121`，且`goid=1`派生`goid=2`，则`map[goid=2]=map[goid=1]=121`；  
本文建议参照[黑魔法-ebpf-对用户空间数据的写入](https://liyan-ah.github.io/2022/11/04/%E9%BB%91%E9%AD%94%E6%B3%95-ebpf-%E5%AF%B9%E7%94%A8%E6%88%B7%E7%A9%BA%E9%97%B4%E6%95%B0%E6%8D%AE%E7%9A%84%E5%86%99%E5%85%A5/#more)进行理解。  
<!--more-->

# 用户态代码及效果
```
package main

import (
	"context"
	"fmt"
	"strconv"
	"sync"
)

var Len = 5

// 当 info 为空时，使用父 goid 设置的值，否则存入 info
func Go1(ctx context.Context, info string, wg *sync.WaitGroup) {
	defer wg.Done()
	third := Third{}
	if info != "" {
		third.Store(info)
	}

	/* 诸多其他的逻辑 */
    
	info1 := third.Get()
	fmt.Printf("raw info: [%s], info get: [%s]
", info, info1)
}

//go:noinline
func Set(info []byte) {
	if len(info) > Len {
		info = info[:Len]
	}
	if len(info) < Len {
		tmp := make([]byte, Len-len(info))
		info = append(tmp, info...)
	}
	fmt.Println("info: ", string(info))

	return
}

//go:noinline
func Get(info []byte) []byte {
	// alalalala, magic come
	return info
}

type Third struct {
	Info string
}

func (t *Third) Store(info string) {
	infoByt := []byte(info)
	// 这里假设是个约束
	infoByt = infoByt[:Len]
	Set(infoByt)
}

func (t *Third) Get() string {
	infoByt := make([]byte, Len, Len)
	infoByt = Get(infoByt)
	return string(infoByt)
}

func main() {
	third1 := Third{}
	info := "12345"
	third1.Store(info)
	wg := &sync.WaitGroup{}
	ctx := context.Background()
	wg.Add(1)
	go Go1(ctx, "", wg) // 写入空数据，预期使用父 goid 数据，即 12345
	for i := 1125; i < 1130; i++ {
		wg.Add(1)
		v := strconv.Itoa(i)
		if i%10 == 0 {
			v = ""
		}
		go Go1(ctx, v, wg)
	}
	wg.Wait()

	/* very long handle logic*/

	third2 := Third{}
	infoGet := third2.Get()
	fmt.Printf("in main, getInfo, [%s]
", infoGet)
}
``` 

执行结果为：
```
// 未开启 bpf 
info:  12345
info:  1129
raw info: [1129], info get: []
info:  1126
raw info: [1126], info get: []
info:  1128
raw info: [1128], info get: []
raw info: [], info get: []
info:  1125
info:  1127
raw info: [1125], info get: []
raw info: [1127], info get: []
in main, getInfo, []

// 开启 bpf
info:  12345
info:  1129
raw info: [], info get: [12345]  // 传入空值，使用父 goid 存入数据
raw info: [1129], info get: [1129]
info:  1126
raw info: [1126], info get: [1126]
info:  1127
raw info: [1127], info get: [1127]
info:  1125
raw info: [1125], info get: [1125]
info:  1128
raw info: [1128], info get: [1128]
in main, getInfo, [12345]
```
上述示例对比了开启`bpf`前后的用户态代码输出。可以看到，当子`goroutine`缺少某个信息时，可以获取父`goroutine`的数据作为缺省值。  
# 应用
意味着我们可以在父`goroutine`中存入我们需要的数据，而后无论是否创建新的`goroutine`，均能获取该信息。维护了`goroutine session`的数据。  
# ebpf 逻辑
这里仍然附上了`ebpf`的主要逻辑以便说明实现过程。除了之前文章中涉及的`ebpf`向用户态写入数据，本文使用了`golang`创建`goroutine`相关的`uprobe`来维护`goroutine session`状态。  
```  
struct {
  __uint(type, BPF_MAP_TYPE_LRU_HASH);
  __uint(key_size, sizeof(u64));   // pid_tgid
  __uint(value_size, sizeof(u64)); // parent goid
  __uint(max_entries, MAX_ENTRIES);
} go_goid_map SEC(".maps");  // 用来获取 goid 状态

struct {
  __uint(type, BPF_MAP_TYPE_HASH);
  __uint(key_size, sizeof(u64));
  __uint(value_size, sizeof(u8) * 5);
  __uint(max_entries, 100);
} info_map SEC(".maps");  // 用来存储 goid->info

struct {
  __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
} events SEC(".maps");

static __always_inline u64 get_goid(u32 tgid, u32 pid) {
  unsigned long task_addr = (unsigned long)bpf_get_current_task();
  unsigned long fsbase    = 0;
  unsigned long g         = 0;
  u64           goid      = 0;
  // 直接基于 偏移量进行处理了
  // offset(task_struct, thread) = 4992
  // offset(thread_struct, fsbase) = 40
  bpf_probe_read(&fsbase, sizeof(fsbase),
                 (void *)(task_addr + OFF_TASK_THRD + OFF_THRD_FSBASE));
  bpf_probe_read(&g, sizeof(g), (void *)(fsbase - 8));
  bpf_probe_read(&goid, sizeof(goid), (void *)(g + GOID_OFFSET));
  return goid;
}

SEC("uprobe/main_set")
int uprobe__main_set(struct pt_regs *ctx) {
  uintptr_t info_p = 0;
  u8        info[5];
  u64       pid_tgid = bpf_get_current_pid_tgid();
  u32       pid      = (u32)(pid_tgid & 0x00FF);
  u32       tgid     = (u32)(pid_tgid >> 32);
  u64       goid     = 0;

  goid = get_goid(tgid, pid);
  SARG(ctx, 0, info_p);

  bpf_probe_read(&info, sizeof(info), (const void *)info_p);
  bpf_map_update_elem(&info_map, &goid, &info, BPF_ANY);

  event_t event  = {};
  event.pid_tgid = pid_tgid;
  memcpy(event.info, info, sizeof(info));
  bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &event, sizeof(event));

  return 0;
}

SEC("uprobe/main_get")
int uprobe__main_get(struct pt_regs *ctx) {
  uintptr_t info_p   = 0;
  u64       pid_tgid = bpf_get_current_pid_tgid();
  u32       pid      = (u32)(pid_tgid & 0x00FF);
  u32       tgid     = (u32)(pid_tgid >> 32);
  u64       goid     = 0;

  goid = get_goid(tgid, pid);

  void *r_info_p = NULL;
  r_info_p       = bpf_map_lookup_elem(&info_map, &goid);
  if (r_info_p == NULL) {
    return 0;
  }
  event_t event  = {};
  event.pid_tgid = pid_tgid;

  SARG(ctx, 0, info_p);

  u8 info[5];
  memcpy(info, r_info_p, sizeof(info));

  memcpy(event.info, info, sizeof(event.info));

  event.res  = bpf_probe_write_user((u8 *)info_p, info, sizeof(info));
  event.addr = info_p;

  bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &event, sizeof(event));
  return 0;
}

/* golang_runtime_newproc1
   func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g,
 callerpc uintptr) *g {
*/
SEC("uprobe/golang_runtime_newproc1")
int uprobe__golang_runtime_newproc1(struct pt_regs *ctx) {
  u64       pid_tgid = bpf_get_current_pid_tgid();
  uintptr_t g_addr   = 0;
  u64       cur_goid = 0;

  SARG(ctx, 3, g_addr);
  bpf_probe_read(&cur_goid, sizeof(cur_goid), (void *)(g_addr + GOID_OFFSET));
  bpf_map_update_elem(&go_goid_map, &pid_tgid, &cur_goid, BPF_ANY);
  return 0;
}

SEC("uprobe/golang_runtime_runqput")
int uprobe__golang_runtime_runqput(struct pt_regs *ctx) {
  u64       pid_tgid    = bpf_get_current_pid_tgid();
  uintptr_t g_addr      = 0;
  u64      *parent_goid = NULL;
  u64       child_goid  = 0;
  void     *v_p         = NULL;

  parent_goid = bpf_map_lookup_elem(&go_goid_map, &pid_tgid);
  if (parent_goid == NULL) {
    return 0;
  }

  // 1. 获取新 goroutine 的 goid
  SARG(ctx, 1, g_addr);
  bpf_probe_read(&child_goid, sizeof(child_goid),
                 (void *)(g_addr + GOID_OFFSET));

  // 2. 设置新 goid 绑定的 caller 信息
  v_p = bpf_map_lookup_elem(&info_map, parent_goid);
  if (v_p == NULL) {
    return 0;
  }
  // 设置子 goid 绑定 caller 为 父 goid 信息
  bpf_map_update_elem(&info_map, &child_goid, v_p, BPF_ANY);

  bpf_map_delete_elem(&go_goid_map, &pid_tgid);
  return 0;
}
```  
以上。