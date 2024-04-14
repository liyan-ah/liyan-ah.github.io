---
title: elscreen标签背景颜色
author: 李岩
tags:
  - emacs
  - vim
  - elscreen
categories:
  - code
date: 2020-12-23 19:23:00
---
> 使用`emacs`过程中，配合`evil`使用，按照`tab`的划分，将编辑、浏览、`leetcode`等任务划分到不同的`tab`便于切换及管理。美中不足的是，模拟标签的`elscreen`默认将其他标签的颜色设置成`:background blue  :foreground black`的配色，每次切换任务时，都需要重复确认需要跳转到哪个标签，就比较麻烦了。查找了一下重置`face-attribute`的方法，备注下。

在初始文件的最后添加：
```
;; 选中标签设置为绿底黑字，其他标签为黄底黑字
(set-face-attribute 'elscreen-tab-other-screen-face nil
                    :background "yellow" :foreground "black")
(set-face-attribute 'elscreen-tab-current-screen-face nil
                    :background "green" :foreground "black")
```
<!-- more -->

备注下[`elscreen`](https://github.com/knu/elscreen)原始代码：  
```
(defface elscreen-tab-current-screen-face
  '((((class color))
     (:background "white" :foreground "black"))
    (t (:underline t)))
  "Face for current screen tab."
  :group 'elscreen)

(defface elscreen-tab-other-screen-face
  '((((type x w32 mac ns) (class color))
     :background "Gray85" :foreground "Gray50")
    (((class color))
     (:background "blue" :foreground "black" :underline t)))
  "Face for tabs other than current screen one."
  :group 'elscreen)
```
[emacs defface](https://www.gnu.org/software/emacs/manual/html_node/elisp/Defining-Faces.html)
