---
title: 基于Ncurses库开发的俄罗斯方块小游戏
categories:
  - Tetris
tags:
  - ncurses库
  - tetris
typora-root-url: 基于Ncurses库开发的俄罗斯方块小游戏
abbrlink: 1833849217
date: 2023-09-25 09:49:13
---

# 前言

此项目是学习完《C++.Primer.Plus》之后的一个练手项目，将GitHub上的TinyTeris（现在搜不到原项目了）由C代码转换为C++代码，将基于方法的编程更换为基于对象的编程。

现在重新回顾之前的代码，优化了一些代码逻辑，并整理出类图和程序逻辑图。

时间比较赶，整体介绍可能过于简单（后续可能考虑详细介绍），详细注释和说明都在代码中。

<!--more-->

# 方块类设计

一共有7种类型的方块，每个方块的位置使用一个左上角的位置和宽度高度（右下角位置）确定，每个方块均是4个正方形。

每一种方块都有多种状态，通过**Rotate**可以旋转方块。

![](Abr_Chunk类图.png)



# 游戏类设计

游戏类维护背景板组件、方块组件等，利用ncurses库的函数将背景板刷新到屏幕上。

![](游戏类图.png)

# 游戏逻辑

游戏流程为：

![](游戏逻辑.png)

# 源码

github地址：https://github.com/AsceticOfLife/Tetris-BasedOnNCURSES















