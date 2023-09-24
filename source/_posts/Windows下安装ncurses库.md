---
title: Windows下安装ncurses库
date: 2023-09-23 15:24:31
categories: 
- Tetris
tags:
- ncurses库
---

# 前言

ncurses是一个管理应用程序在字符终端显示的函数库。它提供了移动光标，建立窗口，产生颜色，处理鼠标操作等功能。
ncurses提供的是字符用户界面，而非图形用户界面。

# ncurses库下载

ncurses库在2023年之前不支持windows平台，在2023年之后更新了windows下MinGW编译器版本链接库。

到[Index of /archives/ncurses (invisible-mirror.net)](https://invisible-mirror.net/archives/ncurses/)下载win32版本（64位同样可用），解压之后目录如下：
{% asset_img ncurses文件夹.png Ncurses库原始文件夹 %}

其中bin目录下是动态链接库，实际编译时应该与需要生成程序在同级目录下；
lib目录下是动态链接库的导引库，可以选择放在工作目录下（推荐）或者MinGW编译器的lib目录中（不推荐）；
include目录是头文件，可以选择放在工作目录下（推荐）或者MinGW编译器的include目录中（不推荐）。

以一个项目文件为例：

{% asset_img 文件夹结构.png 项目文件夹结构 %}

这里lib、ncursesw以及四个dll文件均是压缩文件解压出来的文件，Mytetris文件夹是我的类文件，code.cpp是主文件，code.exe是生成文件，如果想要运行code.exe，那么就需要动态链接库dll的支持。

# 库文件修改和编译

文件夹按照上述安排之后，还存在一个问题：ncurses文件夹里面的头文件会出现包含错误。

这是因为这些头文件编写的目的是希望放到编译器的include文件夹中，也就是在编译时的根文件夹。但是我们这里把它放在了工作目录下，因此需要把<>形式的include指令替换成""形式的相对路径。比如：

```
#include <ncursesw/ncurses_dll.h>

// 修改为
#include "ncurses_dll.h"
```

(并不需要将所有的<>形式都修改，知识需要修改一下其它文件对于ncurses_dll.h头文件的包含。)

最后如果想要使用库，需要以下步骤：

1.在自己的文件中包含curses.h文件

2.编译命令应该是：

```
g++ code.cpp  -L ./lib -lncursesw -o code.exe
```

-L：表示要链接的库所在的目录。

-l：指定链接时需要的动态库。













