先看下官方的介绍
>Sed  is a stream editor. A stream editor is used to perform basic text transformations on an input stream (a file or input from a pipeline).   
>While in some ways similar to an editor which permits scripted edits (such as ed), sed works by making only one pass over the  input(s),  and   
>is consequently more efficient.  But it is sed’s ability to filter text in a pipeline which particularly distinguishes it from other types of   
>editors.

大概的意思，是面向流的文本编辑工具。一般用来对文件中的文本进行替换等操作。  
以下备注一些常用的操作方式了。  
<!--more-->
## 使用介绍
我们以上段文字为例，使用<code>sed</code>进行文本的操作。
```
sed -i "s#Sed#SED#g" text
使用 -i 才可以直接修改 text 里面的内容，否则无法修改（但是会将修改后的内容输出到标准输出）
这里使用#作为sed的限位符而非/，是因为一般文本中，/符号出现的频率要较#高。直接使用#就不需要频繁转义了。

sed -i '2,2 s#in#in_#g' text
将 行号 [2,2] 中的 in 全部替换为 in_，注意，input也会被替换为in_put

sed -i '/While.*/, /.*editors/ s#in#in_#g' text
将 While.* .*editors 之间的 in 全部替换为 in_

sed -i '2,+1 s#in#in_#g' text
将 [2, 2+1=3] 行内的 in 全部替换为 in_
```
基本上常用的一些 <code>sed</code>替换方式就是这些了。<code>man</code>文档中还有一些基于倍数的替换范围决定方式，这里就不说明了。使用的时候，还是尽量使用通俗易懂的方式。