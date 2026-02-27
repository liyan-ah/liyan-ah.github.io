
### 部分Linux指令缺失
<code>Mac OS</code>并没有实现所有的<code>Linux</code>下的指令，如<code>realpath</code>这里就需要单独安装一些扩展包了：
> brew install coreutils
<!--more-->

### 终端Basic颜色不友好
可以单独下载一些配色方案。从尽量使用原生配色的角度来说，只需要在<code>.bashrc</code>下面作如下配置即可：
> \# for color  
export CLICOLOR=1  
\# grep  
alias grep='grep --color=always'

此外，配合<code>.vimrc</code>中配色变更食用更佳：
> " 设置搜索高亮  
set hlsearch  
" 设置语法高亮  
syntax on 

### 启动shell配置不同
<code>Mac OS</code>默认使用<code>zsh</code>作为登陆<code>shell</code>。所以设置<code>.bashrc</code>作为启动配置时，需要在<code>~/.zshrc</code>中进行配置：
> [ -f ~/.bashrc ] && source ~/.bashrc

后面碰到再补充吧。