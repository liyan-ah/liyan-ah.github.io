---
title: python调试方法（其一）
author: Blacker
tags:
  - python
categories:
  - code
date: 2019-07-17 22:51:00
---
<一> 这里记录一些python调试的方法：
```
# coding=UTF-8
''' python debug method 1
use print function to get output informatino
'''
DEBUG = True

def _debug_(*args, **kwds):
	''' depends on DEBUG value, print some function '''
    global DEBUG
    if DEBUG:
    	print(args, kwds)

if __name__ == "__main__":
	_debug_("this is a test")
```
最常见的调试方法了。<code>print</code>可以依据需求调整为其他的方式（<code>logging</code>输出日志或者直接输出到文件中均可）。  
<!--more-->
```
# 输出结果如下：
(('this is a test',), {})
```
<二>然后就是更直接一些的调试方法了
```
# coding=UTF-8
import pdb
def test_function():
	''' regard it as a test funcion '''
    try:
    	a = 1
        b = 0
    	c = a / b
    except Exception, e:
    	pdb.set_trace()
    return
    
if __name__ == "__main__":
	test_function()
```
直接一点了，直接在代码中显式设置断点。这样，在异常发生时，就可以直接中断调试了。  
<code>python</code>中的<code>pdb</code>应该可以认为是一种阉割版的<code>gdb</code>了。仅对<code>list</code><code>print</code>及其他的<code>python</code>的内置函数有较好的支持。相互配合来看的话，也能发现很多问题。
```
# 输出如下：
> test_debug.py(11)test_function()
-> return
(Pdb) list
  6             a = 1
  7             b = 0
  8             c = a / b
  9         except Exception, e:
 10             pdb.set_trace()
 11  ->     return
 12
 13     if __name__ == "__main__":
 14         test_function()
[EOF]
(Pdb) print(e)
integer division or modulo by zero
(Pdb) print(a, b, c)
*** NameError: name 'c' is not defined
(Pdb) print(a, b)
(1, 0)
(Pdb) quit()
```
唔，先这样吧。可以考虑收集一些<code>python</code>的内置解析包来配合调试了。
