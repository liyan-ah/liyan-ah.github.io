---
title: lisp-do循环
author: liyan
tags:
  - lisp
categories:
  - code
date: 2019-06-03 22:42:00
---
lisp中，do循环形象如下:  
```
(do (variable-definition*)
    (end-test-form result-form*)
    statement*);
```
<!--more-->
其中，<code>(variable-definition*)</code>是一些行日<code>(var init next)</code>的赋值结构。在<code>do</code>开始时，<code>var</code>会被赋值为<code>init</code>。并且在一次循环结束后，<code>var</code>会被赋值为<code>next</code>所表示的内容。  
形如：  
```
(do ((n 0 (+ 1 n))
     (cur 0 next)
     (next 1 (+ cur next)))
    ((= 10 n) (format t "|end ~d" cur))
  (format t "~d|" cur));
```
输出：  
```
0|1|1|2|3|5|8|13|21|34||end 55
```
类似于python中的：  
```
cur = 0
next = 1
for i in range(10):
    print("%d|" % cur)
    cur, next = next, cur + next
print("|end %d" % cur)
```
