---
title: lisp-let变量声明
author: Blacker
tags:
  - lisp
categories:
  - code
date: 2019-05-30 22:51:00
---
lisp声明、使用变量的一种方法，是使用<code>let</code>语句。  
形如：  
```
;(let ((variable declare1) (variable declare2) (...))
;  (varaible used here));
(defun foo(x)
  (format t "Parameter: ~a~%" x)
  (let ((x 2))
    (format t "Outer LET: ~a~%" x)
    (let ((x 3))
      (format t "Inner LET: ~a~%" x))
    (format t "Outer LET: ~a~%" x))
  (format t "Parameter: ~a~%" x));

(foo 10);
```
<!--more-->
声明的作用域，和C语言很相似，存在覆盖的特点。输出：
```
Parameter: 10
Outer LET: 2
Inner LET: 3
Outer LET: 2
Parameter: 10
```
使用<code>let</code>声明时，变量声明域内，无法使用前一个在本声明域内声明的变量：
```
(defun year-day(y)
  (let ((m (* y 12)) (d (* m 30)))
    (format t "Year:~d~%Month:~d~%Day:~d~%" y m d)));
(year-day 1);

*** - LET: variable M has no value
```
使用<code>let*</code>可以进行如此操作：
```
(defun year-day(y)
  (let* ((m (* y 12)) (d (* m 30)))
    (format t "Year:~d~%Month:~d~%Day:~d~%" y m d)));
(year-day 1);
Year:1
Month:12
Day:360
```
just like this.
