---
title: go-simplejson 插入数组
author: 李岩
tags:
  - golang
  - simplejson
categories:
  - code
date: 2021-07-22 17:58:00
---
> [go-simplejson](https://github.com/bitly/go-simplejson)是go lang语言中操作json非常方便的开源库。最近使用simplejson进行数据插入操作时遇到了问题，经过排查后最终解决。现记录如下。

<!--more-->
# 问题描述
```
// 创建了一个json对象J，需要从其他地方获取剩余json信息后，插入到J中的data字段中。初始版本如下：
import (
	"fmt"

	simplejson "github.com/bitly/go-simplejson"
)

func main() {
	js, _ := simplejson.NewJson([]byte(`
{
"errno": 0,
"errmsg": "test"
}`))

	var js_2 = new(simplejson.Json)
	*js_2 = *js
	jsArr := []*simplejson.Json{}
	js1, _ := simplejson.NewJson([]byte(`{"num": 1}`))
	js2, _ := simplejson.NewJson([]byte(`{"num": 2}`))
	jsArr = append(jsArr, js1)
	jsArr = append(jsArr, js2)
	js.Set("data", jsArr)
	js.Get("data").GetIndex(0).Set("test", 1)
	jsB, _ := js.MarshalJSON()
	fmt.Println(string(jsB)) 
}
// % go run js_check.go
// {"data":[{"num":1},{"num":2}],"errmsg":"test","errno":0}
```

# 问题排查
经过`dlv`逐行调试，实际问题出在`js.Get("data").GetIndex(0).Set("test", 1)`中。跳转至定义，该操作实际做如下转换:
```
arr, ok := js.Get("data").data.([]interface{})
if ok {
    &simplejson.Json(arr[index]).Set("test", 1)
}
```
这里由于`jsArr`是`[]*simplejson.Json`，类型断言为`[]interface{}`失败。所以无法正常设置值。查看`simplejson.go`，其中的`Json`对象结构如下：
```
type Json struct {
    data interface{}
}
```
其实可以通过`js.Interface()`获取其中的真实数据。

# 解决方案
变更为如下代码即可：
```
package main

import (
	"fmt"

	simplejson "github.com/bitly/go-simplejson"
)

func main() {
	js, _ := simplejson.NewJson([]byte(`
{
"errno": 0,
"errmsg": "test"
}`))

	var js_2 = new(simplejson.Json)
	*js_2 = *js
	jsArr := []interface{}{}
	js1, _ := simplejson.NewJson([]byte(`{"num": 1}`))
	js2, _ := simplejson.NewJson([]byte(`{"num": 2}`))
	jsArr = append(jsArr, js1.Interface())
	jsArr = append(jsArr, js2.Interface())
	js.Set("data", jsArr)
	js.Get("data").GetIndex(0).Set("test", 1)
	jsB, _ := js.MarshalJSON()
	fmt.Println(string(jsB))
}
// % go run js_check.go
// {"data":[{"num":1,"test":1},{"num":2}],"errmsg":"test","errno":0}
```
