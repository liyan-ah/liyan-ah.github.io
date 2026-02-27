
# 信息
这里列出基本类型及其作为参数传递时，占用的空间大小如下表。  

| 类型      | 长度 | 说明                                             |
|-----------|------|--------------------------------------------------|
| 指针      | 8B   | 64位机为 8Byte, 32位机位4Byte                    |
| context   | 16B  | interface 类型。其中，前8B是类型信息，后8B是对象的指针信息         |
| interface | 16B  | 2 个指针，详见[draveness-go-interface](https://draveness.me/golang/docs/part2-foundation/ch04-basic/golang-interface/)，或者 runtime/runtime2.go iface/eface 定义                                                |
| int64     | 8B   | -                                                |
| int       | 8B   | 64位机为 8Byte, 32位机位4Byte                    |
| string    | 16B  | 8B 地址 + 8B string长度                          |
| slice     | 24B  | 8B地址 + 8B slice 成员数量 + 8B slice capability |
| func      | 8B   | func 作为函数参数时，传递的是 func 的地址        |


需要注意的是，作为函数参数传递时，golang会对参数按照 8B 进行对齐。

<!--more-->


# 验证示例
```
// main.go
package main

import "context"

type A struct {
	p1 int64
	a  byte
	b  int64
}

type FuncPt func(A)

type InterA interface {
	Echo(A)
}

func CheckPointer(a *A)            {}
func CheckCtx(ctx context.Context) {}
func CheckInterface(inter InterA)  {}
func CheckString(s string)         {}
func CheckSlice(arr []string)      {}
func CheckFunc(fn FuncPt)          {}
func CheckAlign(a byte)            {}
func CheckStruct(a A)              {}
func main()                        {}

```

对该代码进行汇编:  
`go build -gcflags "-S" . > main.s` 
可以得到汇编后的结果，并验证上述类型所占大小的描述。这里推荐下曹大的[plan9 汇编入门](https://xargin.com/plan9-assembly/)，里面对`golang`汇编后的`plan9`进行了介绍。由其介绍可知，汇编后函数签名后的`$x-y`指代的是该函数的栈空间以及参数大小（入参+返回参数，`go-1.17`及之后的版本未验证）。 
``` 
# command-line-arguments
"".CheckPointer STEXT size=16 args=0x8 locals=0x0 funcid=0x0 leaf
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:17)	TEXT	"".CheckPointer(SB), LEAF|NOFRAME|ABIInternal, $0-8
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:17)	FUNCDATA	ZR, gclocals·2a5305abe05176240e61b8620e19a815(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:17)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:17)	RET	(R30)
	0x0000 c0 03 5f d6 00 00 00 00 00 00 00 00 00 00 00 00  .._.............
"".CheckCtx STEXT size=16 args=0x10 locals=0x0 funcid=0x0 leaf
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:18)	TEXT	"".CheckCtx(SB), LEAF|NOFRAME|ABIInternal, $0-16
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:18)	FUNCDATA	ZR, gclocals·f207267fbf96a0178e8758c6e3e0ce28(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:18)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:18)	RET	(R30)
	0x0000 c0 03 5f d6 00 00 00 00 00 00 00 00 00 00 00 00  .._.............
"".CheckInterface STEXT size=16 args=0x10 locals=0x0 funcid=0x0 leaf
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:19)	TEXT	"".CheckInterface(SB), LEAF|NOFRAME|ABIInternal, $0-16
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:19)	FUNCDATA	ZR, gclocals·f207267fbf96a0178e8758c6e3e0ce28(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:19)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:19)	RET	(R30)
	0x0000 c0 03 5f d6 00 00 00 00 00 00 00 00 00 00 00 00  .._.............
"".CheckString STEXT size=16 args=0x10 locals=0x0 funcid=0x0 leaf
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:20)	TEXT	"".CheckString(SB), LEAF|NOFRAME|ABIInternal, $0-16
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:20)	FUNCDATA	ZR, gclocals·2a5305abe05176240e61b8620e19a815(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:20)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:20)	RET	(R30)
	0x0000 c0 03 5f d6 00 00 00 00 00 00 00 00 00 00 00 00  .._.............
"".CheckSlice STEXT size=16 args=0x18 locals=0x0 funcid=0x0 leaf
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:21)	TEXT	"".CheckSlice(SB), LEAF|NOFRAME|ABIInternal, $0-24
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:21)	FUNCDATA	ZR, gclocals·2a5305abe05176240e61b8620e19a815(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:21)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:21)	RET	(R30)
	0x0000 c0 03 5f d6 00 00 00 00 00 00 00 00 00 00 00 00  .._.............
"".CheckFunc STEXT size=16 args=0x8 locals=0x0 funcid=0x0 leaf
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:22)	TEXT	"".CheckFunc(SB), LEAF|NOFRAME|ABIInternal, $0-8
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:22)	FUNCDATA	ZR, gclocals·2a5305abe05176240e61b8620e19a815(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:22)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:22)	RET	(R30)
	0x0000 c0 03 5f d6 00 00 00 00 00 00 00 00 00 00 00 00  .._.............
"".CheckAlign STEXT size=16 args=0x8 locals=0x0 funcid=0x0 leaf
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:23)	TEXT	"".CheckAlign(SB), LEAF|NOFRAME|ABIInternal, $0-8
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:23)	FUNCDATA	ZR, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:23)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:23)	RET	(R30)
	0x0000 c0 03 5f d6 00 00 00 00 00 00 00 00 00 00 00 00  .._.............
"".CheckStruct STEXT size=16 args=0x18 locals=0x0 funcid=0x0 leaf
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:24)	TEXT	"".CheckStruct(SB), LEAF|NOFRAME|ABIInternal, $0-24
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:24)	FUNCDATA	ZR, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:24)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:24)	RET	(R30)
	0x0000 c0 03 5f d6 00 00 00 00 00 00 00 00 00 00 00 00  .._.............
"".main STEXT size=16 args=0x0 locals=0x0 funcid=0x0 leaf
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:26)	TEXT	"".main(SB), LEAF|NOFRAME|ABIInternal, $0-0
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:26)	FUNCDATA	ZR, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:26)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (/Users/liyan/zone/go_learn/param/main.go:28)	RET	(R30)
	0x0000 c0 03 5f d6 00 00 00 00 00 00 00 00 00 00 00 00  .._.............

```