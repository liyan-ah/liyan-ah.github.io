---
title: find匹配文件名
author: 李岩
date: 2020-11-12 21:44:28
tags:
  - shell
categories:
  - code
---
目录内容：
```
text  text.bak
```
希望从中找到<code>text.bak</code>。使用<code>find</code>实现。
<!--more-->
错误操作：
```
>find -name *.bak* .
find: paths must precede expression: .
Usage: find [-H] [-L] [-P] [-Olevel] [-D help|tree|search|stat|rates|opt|exec] [path...] [expression]
```
<code>-name</code>会作为<code>EXPRESSIONS</code>存在。<code>find</code>要求的参数位置为：
```
find [-H] [-L] [-P] [-D debugopts] [-Olevel] [path...] [expression]
```
所以，正确格式为：
```
find . -name *.bak
./text.bak
```
关于正则中<code>.</code>会作为通配符，如需匹配<code>text.bak</code>需要对<code>.</code>进行转义的情况，也需要关注下。本例中就不涉及了。
