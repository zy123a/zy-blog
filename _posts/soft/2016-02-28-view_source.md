---
layout: post
title: vi和它的插件们
categories: unix
---

*   [vi分屏](#vi_split)
*   [ctags](#ctags)
*   [cscope](#cscope)

### vi分屏 {#vi_split}

***1.直接用终端打开多个文件***

a.横向分屏
    
    vi -o file1 file2
    
b.纵向分屏
    
    vi -O file1 file2

***2.如果已经用vi打开了一个文件,想要在窗口中同时打开另一个文件***

a.横向分屏 vs或vsplit
    
    :vs filename
    
b.纵向分屏 sp 或split
    
    :sp filename

如果文件名不存在,会新建文件并打开

***3.关闭窗口***

a.关闭光标所在窗口
    
    :q 或 :close
    
b.关闭除光标所在的窗口之外的其他窗口

    :only

c.关闭所有窗口
    
    :qa

***4.切换窗口***

ctrl + w w 

或ctrl + w + 方向键

### ctags 

***1.安装*** 

ubuntu: `sudo apt-get install ctags`
mac: 自带的ctags 不支持-R递归,所以想要-R的需要自己安装

1.  http://ctags.sourceforge.net/  下载源码
2.  tar zxvf ctags-5.8.tar.gz
3.  sudo ./configure && make all && sudo make install

***2.使用*** 

进入源码根目录,创建tags `ctags -R *`

|快捷键|作用|
|---|---|
|ctrl ] |跳到光标下方的tag|
|ctrl t  xx|调回tag栈|
|:ts <tag> <RET> |跳到指定的tag|
|:tn| 跳到下一个tag的定义|
|:tp| 跳到上一个tag的定义|
|:ts| 列举上一个tag所有的定义|

### cscope

***1.安装***
ubuntu `sudo atp-get install cscope`
mac `brew install cscope`

***2.使用***

首先需要为你的代码生成一个cscope数据库。生成数据库很简单，在你的项目根目录运行下面的命令： 
cscope -Rbq 
这个命令会生成三个文件：cscope.out, cscope.in.out, cscope.po.out。其中cscope.out是基本的符号索引，后两个文件是使用”-q"选项生成的，可以加快cscope的索引速度。 
具体的参数的含义可以通过:man cscope 得到。-R: 在生成索引文件时，搜索子目录树中的代码

-b: 只生成索引文件，不进入cscope的界面

-q: 生成cscope.in.out和cscope.po.out文件，加快cscope的索引速度

-k: 在生成索引文件时，不搜索/usr/include目录

-i: 如果保存文件列表的文件名不是cscope.files时，需要加此选项告诉cscope到哪儿去找源文件列表。可以使用“-”，表示由标准输入获得文件列表。

-I dir: 在-I选项指出的目录中查找头文件

-u: 扫描所有文件，重新生成交叉索引文件

***3.在vim中使用cscope***

1 、用vim编辑的时候 ： 
vim filename.c 

2 把生成的cscope文件导入到vim中来 

:cs add /路径/cscope.out 代码所在目录 
我习惯现切换到代码所在目录再操作，所以直接使用： 
：cs add cscope.out 就可以了。

3 查看是否已经连接到对应数据库 
:cs s

4 `cs f s xxxx` 查找xxxx出现的地方，它能详细列出哪些文件的哪行的哪个函数引用，以及该行的内容. 
，首先调用“cscope add”命令添加一个cscope数据库，然后就可以调用“cscope find”命令进行查找了。VIM支持8种cscope的查询功能，如下：例如，我们想在代码中查找调用work()函数的函数，我们可以输入：“:cs find c work”，回车后发现没有找到匹配的功能，可能并没有函数调用work()。我们再输入“:cs find s work”，查找这个符号出现的位置，现在vim列出了这个符号出现的所有位置。我们还可以进行字符串查找，它会双引号或单引号括起来的内容中查找。还可 以输入一个正则表达式，这类似于egrep程序的功能。

s: 查找C语言符号，即查找函数名、宏、枚举值等出现的地方

g: 查找函数、宏、枚举等定义的位置。

    cs f g main 光标跳到main函数的位置
d: 查找本函数调用的函数

    cs f d main 列出main函数调用的函数，输入 函数所对应的数字，就可以调到相应掉用的函数
c: 查找调用本函数的函数

    cs f c main 会输出找到没有匹配的结果，因为没有函数调用main函数
t: 查找指定的字符串

    cs f t string
f: 查找并打开文件

    cs f f filename 将会打开文件名为FIANME的文件


### 参考