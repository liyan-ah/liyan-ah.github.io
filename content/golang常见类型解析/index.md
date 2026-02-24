+++
title = "golang常见类型作为参数的eBPF解析"
author = "李岩"
date = 2022-12-30T19:24:18+08:00

[taxonomies]
tags = ["bpf", "golang"]
categories = ["BPF"]
+++
> 即将过去的2022年，笔者相当比例的精力都投入在了eBPF上。最初的时候，写了一篇[golang 常见类型字节数
](https://liyan-ah.github.io/2022/06/06/golang-%E5%B8%B8%E8%A7%81%E7%B1%BB%E5%9E%8B%E5%AD%97%E8%8A%82%E6%95%B0/)，开启了`eBPF+golang`的总结性工作。此后陆续整理了一些关于`ebpf`的使用文章，同时项目也在逐步的推进。`eBPF`的实际落地有很大的挑战，但是最终还是找到了一些落地的场景。年底了，结合最近的调研工作，笔者整理了这篇文章。既算是对之前文章的呼应，也是对今年整理内容的总结。  

`eBPF`能够提供一种切入服务细节的独特视角。本文即通过实例，对`golang`常见类型作为函数参数时进行解析，期望读者能够感受这一视角。需要说明的是，本文是基于`golang-1.16`来整理的。
<!--more-->

# 数值类型
目前`golang`支持的数值类型大概有`int`, `int8`, `int16`, `int32`, `int64`及相对应的无符号类型。无符号类型在传递时和对应的有符号类型是一致的，这里不再赘述。`int`在不同平台上大小会不同，`64`位操作系统时，`sizeof(int)=sizeof(int64)`。作为参数传递时，数值类型会直接传递值。  
一般来说，`int8`, `int16`, `int32`在作为参数传递时，基于内存对齐的原则，会使用`8B`的空间来传递。但是并不绝对，笔者在准备本篇文章时，就找到了这样的示例：

``` 
// type/main.go
package main

import "fmt"

// 如果严格按照"内存对齐"来推算，int_p 的参数大小应为 8B*5
//go:noinline
func int_p(a int8, b int16, c int32, d int64, e int) {
	fmt.Println(a, b, c, d, e)
}

func main() {
	int_p(1, 2, 3, 4, 5)
}

/* 
从输出结果来看，int_p 参数列表带大小为 3*8B，其中第一个 8B 的分布为：
 |int8|--|int16|int32|
 | 1B |1B| 2B  | 4B  |
 type/type.bt
*/
uprobe:./type:"main.int_p"
{
    printf("int8:  %d
", (int8)(sarg0));
    printf("int16: %d
", (int16)(sarg0>>16));
    printf("int32: %d
", (int32)(sarg0>>32));
    printf("int64: %d
", sarg1);
    printf("int:   %d
", sarg2);
}

// 运行结果
Attaching 1 probe...
int8:  1
int16: 2
int32: 3
int64: 4
int:   5
```

由此可知，当数值类型作为函数参数时，需要结合前后参数来判断是否触发了内存对齐，进而判断数值类型参数的具体位置。

# string  

`string`是由`8B addr + 8B length`来描述的。作为函数参数传递时，亦通过这样的方式来解析：

``` 
// type/main.go
package main

import "fmt"

//go:noinline
func string_p(name string) {
	fmt.Println(name)
}

func main() {
	name := string("didi")
	string_p(name)
}

// type/type.bt
uprobe:./type:"main.string_p"
{
    printf("addr:   %d
", sarg0);
    printf("length: %d
", sarg1);
    printf("name:   %s
", str(sarg0, sarg1));
}

bpftrace ./type.bt
Attaching 1 probe...
addr:   4958864        // 所谓的地址，就是这么个东西 @V@
length: 4
name:   didi
```

由于`string`的地址和长度都是`8B`，所以不会触发内存对齐。

# slice

`slice`是由`8B addr + 8B length + 8B cap`来描述的。作为函数值来传递时，需要关注地址及长度，以防止出现过解析的情况：

```
// type/main.go
package main

import "fmt"

//go:noinline
func slice_p(slices []int64) {
	fmt.Println(slices)
}

func main() {
	slices := []int64{1, 2, 3}
	slice_p(slices)
}

// type/type.bt
uprobe:./type:"main.slice_p"
{
    printf("addr:   %d
", sarg0);
    printf("length: %d
", sarg1);
    printf("cap:    %d
", sarg2);

    $pos = 0;
    $offset = 0;
    unroll(10){
        if ($pos >= sarg1){
            return;
        }
        $value = *(int64*)(sarg0+$offset);
        printf("%d: %d
", $pos, $value);

        $offset = $offset+8;
        $pos = $pos + 1;
    }
    return;
}

bpftrace ./type.bt
Attaching 1 probe...
addr:   1310720
length: 3
cap:    3
0: 1
1: 2
2: 3
```

由于`eBPF`对循环的长度是有限制的，所以通过循环来读取数据会麻烦。一般可以直接将所有的数据都读出来，记录长度并将其传递到用户空间处理。

# 定长数组

`golang`的定长数组在传递时，会直接将数据拷贝上去。所以一般是不建议使用定长数组作为函数参数。

```
// type/main.go
package main

import "fmt"

//go:noinline
func array_p(slices [4]int64) {
	fmt.Println(slices)
}

func main() {
	arrays := [4]int64{1, 2, 3, 4}
	array_p(arrays)
}

// type/type.bt
uprobe:./type:"main.array_p"
{
    printf("arr[0]: %d
", sarg0);
    printf("arr[1]: %d
", sarg1);
    printf("arr[2]: %d
", sarg2);
    printf("arr[3]: %d
", sarg3);
    return;
}

bpftrace ./type.bt
Attaching 1 probe...
arr[0]: 1
arr[1]: 2
arr[2]: 3
arr[3]: 4
```

需要注意的是，定长数组作为结构体参数时，也是直接将参数堆积的，而不是类似`slice`的由指针及长度组成。

# 结构体

`golang`结构体作为函数参数传递时，会直接将结构体内的成员逐个传递。

```
// type/main.go
package main

import (
	"fmt"
)

type S struct {
	X int64
	Y int64
	Z [3]int64
	A int64
}

// 请注意 other 参数，虽然其作为函数的第二个参数，但其在函数列表中的偏移量是 48B
//go:noinline
func struct_p(s S, other int64) {
	fmt.Println(s, other)
}

func main() {
	s := S{
		X: 1,
		Y: 2,
		Z: [3]int64{3, 4, 5},
		A: 6,
	}
	struct_p(s, 7)
}

// type/type.bt
uprobe:./type:"main.struct_p"
{
    printf("X: %d
", sarg0);
    printf("Y: %d
", sarg1);
    printf("A: %d
", sarg5);
    printf("other: %d
", sarg6);
    return;
}

bpftrace ./type.bt
Attaching 1 probe...
X: 1
Y: 2
A: 6
other: 7
```

结构体作为函数参数，往往会涉及到内存对齐的问题。需要逐个分析了。

# 指针
`golang`指针作为函数参数时，会直接传递指针数值。

```
// type/main.go
package main

import (
	"fmt"
)

type P struct {
	X int64
	Y int64
	Z [3]int64
	A int64
}

//go:noinline
func pointer_p(p *P) {
	fmt.Println(*p)
}

func main() {
	p := &P{
		X: 1,
		Y: 2,
		Z: [3]int64{3, 4, 5},
		A: 6,
	}
	pointer_p(p)
}

// type/type.bt
uprobe:./type:"main.pointer_p"
{
    $p = sarg0;
    printf("X: %d
", *(int64*)($p+0));
    printf("Y: %d
", *(int64*)($p+8));
    printf("A: %d
", *(int64*)($p+40));
    return;
}

bpftrace ./type.bt
Attaching 1 probe...
X: 1
Y: 2
A: 6
```

解析`golang`指针成员的时候，需要提前知晓指针结构体的内容。

# map
`golang`的`map`实现比较复杂，详细的介绍可以看下这篇文章：[golang map](https://golang.design/go-questions/map/principal/)。`map`作为参数传递时，实际上传递的是`hmap`的指针。  
由于`golang`的`map`实际的结构及具体结构体的大小会受到`map key, map value`的影响，这对使用`eBPF`来解析`golang map`带来了额外的挑战。所幸本文并不期望提供一个`golang map`解析的通用方法，我们可以提前定义所需要解析的`map`为`map[int64]int64`，在这个条件下，`bmap`的结构就为：

```
// sizeof(bmap) = 144B
type bmap struct {
    topbits  [8]uint8 // 8B
    keys     [8]int64 // 64B
    values   [8]int64 // 64B
    //pad      uintptr //不需要添加内存对齐参数
    overflow uintptr
}
```

确定了`bmap`的信息后，可以看到，`keys`及`values`的偏移信息就确定了，可以直接读取。但是由于`key`实际映射时是通过`hash`来决定其位置的，完整的读取`map`显然是很困难的。

```
// type/main.go
package main

import "fmt"

//go:noinline
func map_p(m map[int64]int64) {
	fmt.Println(m)
}

func main() {
	m := map[int64]int64{}
	for i := int64(1); i <= int64(10); i++ {
		m[i] = i
	}
	map_p(m)
}

// type/type.bt
uprobe:./type:"main.map_p"
{
    $hmap_addr = sarg0;
    $bucket_addr = *(uint64*)($hmap_addr+16);
    $bucket_offset = 0;
    unroll(2){ // 尝试读取两个 bmap
        $bmap_addr = $bucket_addr + $bucket_offset;
        $key_addr = $bmap_addr + 8;
        $value_addr = $bmap_addr + 72;

        $offset = 0;
        unroll(8){ // 读取每个 bmap 的所有 key-value
            $key = *(int64*)($key_addr+$offset);
            $value = *(int64*)($value_addr+$offset);
            printf("key: %d, value: %d
", $key, $value);
            $offset = $offset + 8;
        }
        $bucket_offset = $bucket_offset + 144;
    }
    return;
}

// 笔者的这次运行，把[1,10]所有的key都输出了
bpftrace ./type.bt
Attaching 1 probe...
key: 2, value: 2
key: 3, value: 3
key: 5, value: 5
key: 6, value: 6
key: 7, value: 7
key: 8, value: 8
key: 9, value: 9
key: 10, value: 10
key: 1, value: 1
key: 4, value: 4
key: 0, value: 0
key: 0, value: 0
key: 0, value: 0
key: 0, value: 0
key: 0, value: 0
key: 0, value: 0
```

使用`eBPF`来读取`map`，不得不预设一个确定的大小。至于是否能够读取所有的`map`值，就不好说了。

# interface
`golang`的`interface`是由`8B type pointer`+`8B struct pointer`组成的。当解析`interface`的时候，需要的往往是`struct pointer`。

```
// type/main.go
package main

import (
	"fmt"
)

type Inter interface{}

type S struct {
	X int64
	Y int64
	Z [3]int64
	A int64
}

//go:noinline
func struct_p(i Inter, other int64) {
	s, _ := i.(S)
	fmt.Println(s, other)
}

func main() {
	s := S{
		X: 1,
		Y: 2,
		Z: [3]int64{3, 4, 5},
		A: 6,
	}
	struct_p(s, 7)
}

// type/type.bt
uprobe:./type:"main.struct_p"
{
    $addr = sarg1;
    printf("X: %d
", *(int64*)($addr+0));
    printf("Y: %d
", *(int64*)($addr+8));
    printf("A: %d
", *(int64*)($addr+40));
    printf("other: %d
", sarg2);
    return;
}

bpftrace ./type.bt
Attaching 1 probe...
X: 1
Y: 2
A: 6
other: 7
```

从示例中可以看出，当解析`interface`时，`interface`具体的结构体成员对我们而言更加重要。在实际的工程里，往往会出现多个结构体实现同一个`interface`，并且均可以作为该`interface`来传递值的情况。这时就需要依据所实际期望解析的结构体来进行实际采集的过滤或者适配了。

# 写在最后
本文编写的文章超过了笔者的估计时间 :)。`anyway`，这篇文章最终整理完成了。期望有读者能够籍此对`eBPF`有直观的认识，同时体会到其观察`golang`的独特视角。  
周末愉快，新年快乐！