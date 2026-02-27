<!--more-->

# rust
这里使用 `rust-analyzer` 作为 `rust` 的语言服务器，在安装 `rust-mode`后，通过绑定语言服务器信息，即可在打开由 `cargo` 创建的工程时，顺利进入 `lsp-mode`。需要关注的是，在非`cargo`创建的项目中，笔者的`lsp-mode`使用体验很差，甚至缺少代码补充、语法提示等功能。可能是`rust-analyzer`主要是针对`cargo`项目进行的设置，也可能是笔者设置的问题。
```
;; config for rust-lsp for emacs
;; rls install address: https://github.com/rust-lang-nursery/rls
(unless (package-installed-p 'rust-mode)
  (w-install 'rust-mode))
(add-to-list 'auto-mode-alist '("\.rs\'" . rust-mode))
(add-hook 'rust-mode-hook 'lsp)
(unless (package-installed-p 'rustic)
  (w-install 'rustic))
(unless (package-installed-p 'cargo)
  (w-install 'cargo))
(use-package rustic) ; lsp-compatible rust mode
(add-hook 'rust-mode-hook 'rustic-mode)
(add-hook 'rustic-mode-hook
  (lambda ()
    (setq rustic-lsp-server 'rust-analyzer) ; not rls
    (setq lsp-rust-analyzer-server-command '("/opt/homebrew/bin/rust-analyzer"))
                                        ;(setq rustic-format-on-save t) ; has annoying bug move point to other buffer bug
  (setq rustic-indent-offset 4)
    (setq rustic-match-angle-brackets nil)
    ;; thought this would be better, was wrong.
    ;(setq rustic-compile-display-method 'popwin:display-buffer-1) ; display if possible in popup-win
    ))
(use-package cargo)
(setq lsp-rust-server 'rust-analyzer)
```
# golang
`golang`作为谷歌的亲儿子，是拥有官方维护的语言服务器的。而且`gopls`的使用体验非常好，完全不逊色与目前用户较多的`goland`及`vscode`。配合`dlv-mode`使用，在调试上笔者认为能够更加的贴合`unix`风格，也更加方便。
```
;; Go - lsp-mode
;; Set up before-save hooks to format buffer and add/delete imports.
;; go install github.com/golang/tools/cmd/gopls
;;(require 'lsp-mode)
(setq lsp-ui-mode nil)
(defun lsp-go-install-save-hooks ()
  (add-hook 'before-save-hook #'lsp-format-buffer t t)
  (add-hook 'before-save-hook #'lsp-organize-imports t t))
(add-hook 'go-mode-hook #'lsp-go-install-save-hooks)

;; Start LSP Mode and YASnippet mode
(add-hook 'go-mode-hook #'lsp-deferred)
(add-hook 'go-mode-hook #'yas-minor-mode)
```
# python
`python`的语言服务器，笔者目前使用的是`lsp-python-ms`进行配置的。这个插件解决了很多`python lsp`的问题（实际上，在碰到这个插件之前，笔者一度要放弃安装`python lsp`）。由于`python`是解释型语言，对象的成员都较为灵活，一般编码阶段很难确认对象的成员及其确切的类型。所以在`pylsp`使用过程中，往往会碰到无法有效提示的情况。满足一般提示需求吧。
```
;;; set env for python
;; copied from
;; https://gitee.com/nutora-emacs/lsp-python-ms
;; python lsp-server use python-lsp-server
;; install as: pip3 install python-lsp-server
(ensure-package-installed 'lsp-python-ms)
(require 'lsp-python-ms)
(setq lsp-python-ms-auto-install-server t)
(add-hook 'python-mode-hook #'lsp)
```
# c/c++
实际上，笔者很喜欢`c/c++`的语言服务器，简单、方便，安装时无比的顺畅。完全符合笔者对`c`语言简单、强大、靠谱的印象。
```
;; set up lsp-mode for c/c++
;; brew install llvm
;; https://clangd.llvm.org/installation
(unless (package-installed-p 'eglot)
  (w-install 'eglot))
(require 'eglot)
(add-to-list 'eglot-server-programs '((c++-mode c-mode) "clangd"))
(add-hook 'c-mode-hook 'eglot-ensure)
(add-hook 'c++-mode-hook 'eglot-ensure)
```
# 使用的一点备注
这里唠叨一点
## lsp 的管理单位是文件目录
这里对于`golang`及`rust`尤为明显。在使用`emacs`打开一个关联了有效语言服务器的文件时，底部会提示为当前文件选择一个工作目录。尤其是，当路径`A`已经设为工作目录时，再将`A/B`设为工作目录，`A/B`的打开状态是会出现异常的。所以尽量保持每个工作目录的独特。  
这里附上一些`emacs lsp-mode`中笔者常用的函数： 

|指令|说明|
|---|---|
|`lsp-workspace-folders-remove`| 将工作目录移除  |
|`lsp-workspace-folders-add` |添加工作目录|
|`lsp-workspace-restart`| 重启工作目录|
## 其他备注
当安装了一个语言的`lsp`服务及对应的`emacs`客户端配置时，如果打开对应语言的文件发现`lsp`没有生效，且打开`toggle-debug-on-error`设置开启也没有发现任何报错，笔者建议重启`emacs`。似乎`emacs`热加载功能往往不会如人所愿。

# 参考文献
[emacs lsp mode](https://emacs-lsp.github.io/lsp-mode/)  
及其他网络文献