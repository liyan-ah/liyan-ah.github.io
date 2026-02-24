+++
title = "golang proto3 使用"
author = "李岩"
date = 2022-05-19T17:53:00+08:00

[taxonomies]
tags = ["code", "protobuf"]
categories = ["code"]
+++
> 一直都比较赞赏[`protocol buffer`](https://developers.google.com/protocol-buffers/docs/proto3)。由于其表现性强、压缩比高，可以把很多结构都写到`proto`文件中，同时添加很多的注释。当需要进行进行数据存储时，使用`proto`序列化结果替代`json`，可以省去很多的冗余字段。本篇找了一些`golang`中`protocol buffer`的使用示例，以及`protocol`对象与`json`对象互相转换的示例。  

<!--more-->

# 依赖环境
这部分主要参照[官网教程](https://developers.google.com/protocol-buffers/docs/gotutorial)来：
* `protoc` 安装：  
[github-protobuf-releases](https://github.com/protocolbuffers/protobuf/releases) 下载对应平台的 `protoc` 编译器即可；
* `protoc-gen-go` 安装：  
`go install google.golang.org/protobuf/cmd/protoc-gen-go@latest` 需要能够安装对应语言的插件，`proto` 文件才被翻译为对应语言可调用的模块； 

# 示例代码
比较推荐将`proto`文件单独放入一个仓库。`proto`一般定义的是需要服务/模块间共享的，所以单独放在一个仓库里便于调用及约定的维护。
```
.
├── main.go
└── proto
    └── user.proto
```

```
// main.go
package main

import (
	"bytes"
	"fmt"
	"log"
	"net/http"
	"proto/message"

	"github.com/gin-gonic/gin"
	"github.com/gogo/protobuf/jsonpb"
	jsoniter "github.com/json-iterator/go"
	"google.golang.org/protobuf/proto"
)

func main() {
	msg := message.UserInfo{UserList: []*message.UserInfo_User{{Username: "test"}}}
	msg.UserList = append(msg.UserList, &message.UserInfo_User{Username: "test1"})

	// go message 可以直接序列化为 json byte
	byt, err := jsoniter.Marshal(&msg)
	if err != nil {
		log.Fatal("cannot parse to json")
	}
	fmt.Println("json result: ", string(byt))

	// 可以将 json 对象反序列化为 go message 对象
	msg1 := &message.UserInfo{}
	err = jsonpb.Unmarshal(bytes.NewReader(byt), msg1)
	if err != nil {
		log.Fatal("parse failed, ", err)
	}
	fmt.Printf("parsed: %+v
", msg1)

	// protobuf 本身的字符串表征
	msg1Str := msg1.String()
	fmt.Println("msg1 string, ", msg1Str)
    // protobuf 序列化
	out, err := proto.Marshal(msg1)
	fmt.Println("msg1 marshal result is, ", string(out))
	msg2 := message.UserInfo{}
    // 将序列化后的结果，反序列化为 message 对象
	proto.Unmarshal(out, &msg2)
	fmt.Printf("unmarshal result msg2 is: %+v
", &msg2)

	engine := gin.Default()
	engine.GET("check", func(c *gin.Context) {
    	// message 对象可以直接用来作为接口的返回值
		c.JSON(http.StatusOK, &msg1)
	})

	srv := &http.Server{}
	srv.Addr = "0.0.0.0:9988"
	srv.Handler = engine
	srv.ListenAndServe()
}

// proto/user.proto
syntax = "proto3";

package user_info;

// 对于 golang 的使用说，这里的 go_package 是必须的。表述的是编译后的模块名
option go_package = "./message";

message UserInfo{
  message User{
    string username = 1;
    uint32 age = 2;
    string graduate = 3;
  }

  repeated User user_list = 1;
}
```

进行编译:`protoc -I./proto user.proto --go_out=./`，
```
.
├── go.mod
├── go.sum
├── main.go
├── message
│   └── user.pb.go
└── proto
    └── user.proto

2 directories, 5 files
```

执行 `go run main.go`。  
```
go run main.go
json result:  {"user_list":[{"username":"test"},{"username":"test1"}]}
parsed: user_list:{username:"test"}  user_list:{username:"test1"}
msg1 string,  user_list:{username:"test"}  user_list:{username:"test1"}
msg1 marshal result is,

test

test1
unmarshal result msg2 is: user_list:{username:"test"}  user_list:{username:"test1"}
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:	export GIN_MODE=release
 - using code:	gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /check_must               --> main.main.func1 (3 handlers)
```

# 总结
`protocol buffer` 在大多数场景下，都能兼容`json`对象的使用场景。其劣势为序列化相关操作时额外的性能开销。对于与外部进行交互、不会进行频繁序列化、反序列化的数据，可以考虑优先使用`protocol buffer`。