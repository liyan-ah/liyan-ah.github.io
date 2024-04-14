---
title: lisp-hello world
author: Blacker
tags:
  - lisp
categories:
  - code
date: 2019-05-28 21:52:00
---
lisp语言的基本表达式为S-表达式。这与受[Algol](https://en.wikipedia.org/wiki/ALGOL)语言影响的C系语言有很大的不同。显然，这很有趣：
```
;the bellow is hello world function in lisp
  (defun hello-world()
    "hello world function in lisp"
    (format t "hello, world!"));```
由<code>()</code>所包围的内容，为*列表*，其余内容为原子。显然，lisp表达式有很多列表表示（List Processing)。
