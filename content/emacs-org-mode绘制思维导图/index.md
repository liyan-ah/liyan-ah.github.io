+++
title = "emacs org-mode 绘制思维导图"
author = "李岩"
date = 2023-02-10T19:46:21+08:00

[taxonomies]
tags = ["emacs"]
categories = ["程序人生"]
+++
> 工作中难免会搞一些思维导图，一些小的需求又不希望切换窗口到另外一个界面去特地绘制。使用 emacs 来整理思维导图可以提升一些的效率，在当前窗口（文本编辑器）里即可完成简单思维导图的绘制。同时可以便于对工作内容进行归档（比如把相关的文本都放到一起）。live in emacs.


# 依赖内容
* org-contrib 扩展文件。用来将 org-mode 格式的文本转换成 freemind mm 文件。
* freemind 软件。用来查看生成的 mm 文件。  

笔者试了一下，`Xmind思维导图`看起来无法打开`mm`文件，`freemind`工作正常。也可能是我操作有问题。  
此外，生成的思维导图展现样式肯定没有目前专业的思维导图工具丰富，如果有正式的使用需求，还是首先考虑下专业的思维导图工具。

<!--more-->

# org-contrib 安装
笔者使用的`emacs`发布版本默认没有`org-contrib`，需要自行安装。安装过程也比较简单，从`github`里把`org-contrib`拉下来，在`emacs init.el`里配置加载路径，然后主动加载需要的`ox-freemind.el`即可。  
`github org-contrib`地址为`git@github.com:emacsmirror/org-contrib.git`。目录地址可以视自己的需求确定。笔者的`emacs`配置都放到了`.emacs.d`里，`org-contrib`的本地目录也就放到了`~/.emacs.d/org-contrib`这里。扩展下载后，在`init.el`里做如下配置即可：

```
;; ox-fremind
;; 这里改成本地的 org-contrib 地址
(add-to-list 'load-path "~/.emacs.d/org-contrib/lisp")
;; 目前只需要 ox-freemind，因此仅加载这个插件。
(load-file "~/.emacs.d/org-contrib/lisp/ox-freemind.el")
```

安装结束后，需要重新加载一下`emacs`的配置文件，`ox-freemind`才能可用。

# 使用 org-mode 整理文档并转换
这里直接贴一个示例：

```
#+TITLE:  emacs org-mode 绘制思维导图
#+OPTIONS: H:1000

* org-contrib 安装
   org-contrib 可以直接从 github 下载，然后在 emacs 配置文件里加载。
** org-contrib github 地址
*** git@github.com:emacsmirror/org-contrib.git
** emacs 本地配置
*** (add-to-list 'load-path "~/.emacs.d/org-contrib/lisp")(load-file "~/.emacs.d/org-contrib/lisp/ox-freemind.el")
* org-mode 下文档编写
** org-mode 是 emacs 下的神器
*** 打开 freemind.org 文件，输入这个文本
** 转换文本文件到 freemind mm 文件
*** M-x org-freemind-export-to-freemind
* 查看 mm 文件
** 使用 freemind 查看生成的 freemind.mm
```

用`emacs`打开一个`freemind.org`，笔者这里直接触发了`org-mode`。如果没有触发`org-mode`的话，需要手动执行下`M-x org-mode`。然后执行`org-freemind-export-to-freemind`。如果没有这个函数，需要看下之前`org-contrib`的安装是否有问题，或者加载路径是否正常，加载是否有报错。如果函数执行异常，则需要查下原因。笔者安装后即可直接执行，因此没有报错处置的经验可供参考。  
# 使用freemind查看及导出
`mac`可以直接`brew install --cask freemind`。或者到其他下载源下载，如[freemind sourceforge 下载](https://freemind.sourceforge.net/wiki/index.php/Download)。   
最后使用`freemind`打开`freemind.org`同级目录生成的`freemind.mm`。展示效果如下：
![upload successful](freemind.png)
最后，可以使用`emacs`查看导出的`freemind.png`（🐶，笔者还在探索如何在不打开`freemind`的情况下把`mm`文件转换成`png`)。