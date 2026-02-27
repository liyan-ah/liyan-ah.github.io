参照《实用common lisp编程》：
```
;按照 min max, 步长step为参数的fn计算的长度输出 *
(defun plot (fn min max step)
  (loop for i from min to max by step do
        (loop repeat (funcall fn i) do (format t "*"))
        (format t "~%")))

(plot #'exp 0 4 1/2);
(plot #'(lambda (x) (* 2 x)) 0 10 1);
```
<!--more-->
输出：
```
*
**
***
*****
********
*************
*********************
**********************************
*******************************************************

**
****
******
********
**********
************
**************
****************
******************
********************
```
<code>#'</code>为lisp语言的语法糖，展开表示为<code>function</code>。后者将会把一个函数生成为一个函数对象，后者可以通过<code>funcall</code>调用。  
interesting。