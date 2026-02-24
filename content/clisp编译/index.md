+++
title = "clisp编译"
author = "Blacker"
date = 2019-05-28T22:16:00+08:00

[taxonomies]
tags = ["lisp"]
categories = ["code"]
+++
目前，lisp的开发环境基本上被[lispbox](https://common-lisp.net/project/lispbox/)所垄断。所以本文来说一[CLISP](https://clisp.sourceforge.io/)，C语言实现的LISP解释器的安装。
<!--more-->
```
wget "https://ftp.gnu.org/pub/gnu/clisp/latest/clisp-2.49.tar.gz"
tar -xvf clisp-2.49.tar.gz
cd clisp-2.49
./configure --prefix=LOCAL_PATH --ignore-absence-of-libsigsegv
cd src && make && make install
```
这样就可以将CLISP安装到<code>--prefix</code>指定的路径。  
然后是使用。
```
cd LOCAL_PATH/bin/
./clisp
```
就会出现欢迎界面：
```
  i i i i i i i       ooooo    o        ooooooo   ooooo   ooooo
  I I I I I I I      8     8   8           8     8     o  8    8
  I  \ `+' /  I      8         8           8     8        8    8
   \  `-+-'  /       8         8           8      ooooo   8oooo
    `-__|__-'        8         8           8           8  8
        |            8     o   8           8     o     8  8
  ------+------       ooooo    8oooooo  ooo8ooo   ooooo   8

Welcome to GNU CLISP 2.49 (2010-07-07) <http://clisp.cons.org/>

Copyright (c) Bruno Haible, Michael Stoll 1992, 1993
Copyright (c) Bruno Haible, Marcus Daniels 1994-1997
Copyright (c) Bruno Haible, Pierpaolo Bernardi, Sam Steingold 1998
Copyright (c) Bruno Haible, Sam Steingold 1999-2000
Copyright (c) Sam Steingold, Bruno Haible 2001-2010

Type :h and hit Enter for context help.
[1]>
```
尝试进行函数求值：
```
[1]> (defun sum(x y) (format t "~d" (+ x y)))
SUM
[2]> (sum 1 2)
3
NIL
[3]> (exit)
Bye.
```
或者，将以下内容写入```test.lisp```文件然后执行：
```
(defun sum(x y)
  (format t "~d" (+ x y)))
(sum 1 2)
```
执行```LOCAL_PATH/bin/clisp test.lisp```成功输出。