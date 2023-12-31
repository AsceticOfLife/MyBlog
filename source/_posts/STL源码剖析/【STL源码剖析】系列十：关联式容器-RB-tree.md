---
title: 【STL源码剖析】系列十：关联式容器--RB-tree
keywords: 'STL源码剖析, RB-tree, 红黑树'
categories:
  - STL源码剖析
tags:
  - STL
  - 读书笔记
typora-root-url: 【STL源码剖析】系列十：关联式容器-RB-tree
abbrlink: 1344270053
date: 2023-11-09 10:01:56
---

## 前言

本篇主要介绍STL中的RB-tree(红黑树)结构，主要内容包括：

1.在了解红黑树之前，需要对于二叉搜索树和AVL树进行初步认识，红黑树作为一种二叉搜索树，其本身就是为了提高查找效率(O(logn))，此外还需要熟悉AVL树的左旋和右旋操作；

2.红黑树的性质，以及当插入时如何通过一些调整措施保证红黑树的性质不被破坏；

3.在STL中对于红黑树的实现，以及常见操作。

<!-- more -->

## 树与AVL树

### 二叉搜索树

**二叉搜索树**也称为二叉排序树、二叉查找树，定义为具有以下性质的二叉树：

- 若左子树不空，那么左子树上所有节点的值均小于根节点的值；
- 若右子树不空，那么右子树上所有节点的值均大于根节点的值；
- 左右子树也均为二叉搜索树

**二叉搜索树的查找操作：**

从根节点开始逐个与目标值进行比较，如果当前根节点的值大于目标值，就递归从左子树查找；如果当前根节点的值小于目标值，就递归从右子树查找。

**二叉搜索树的插入操作：**

插入一个新元素时，从根节点开始，遇到值较大就向左子树，遇到值小就向右子树，直到尾端就是插入位置。

**二叉搜索树的删除操作：**

如果要删除一个节点A，分为两种情况：（如果A是叶子节点可以直接删除）

（1）A只有一个子节点，直接将A的子节点连接到A的父节点，然后删除A；

（2）A有两个子节点，以A的右子树的最小值取代A，也就是A的右子树的最左叶子节点。

查找速度取决于二叉排序的层数，但是二叉排序树的形状是不确定的。比如：

<img src="5-1-1.png" alt="image-20230411164120634" style="zoom:67%;" />

<img src="5-1-2.png" alt="image-20230411164136146" style="zoom:50%;" />

### 平衡二叉搜索树--AVL树

**树的一些相关定义：**

树的根节点到任何节点之间有一条唯一的路径，路径所包含的边数称为路径长度。

一个节点的深度指的是根节点到该节点的路径长度。

一个节点的高度指的是该节点到其最深子节点（叶子节点）的路径长度。

由于二叉搜索树的查找效率取决于节点所在的层数，因此所有节点的深度均不会太大就会使得总体的查找效率比较低。这也就是**平衡二叉树**的思想。

**AVL-tree的平衡条件**是任何节点的左右子树的高度差最多为1。左子树深度减去右子树深度称为平衡因子，平衡树的平衡因子只有可能是1（左子树比右子树深）、0（左子树和右子树一样深）、-1（右子树比左子树深）

这里需要引入一个最小不平衡子树的概念：距离插入节点最近的，并且以平衡因子的绝对值大于1的节点为根的子树，称为最小不平衡子树。如果插入新节点导致不平衡，只需调整最小不平衡子树即可。

**AVL-tree的单旋转与双旋转**

- 当最小不平衡子树根节点的BF大于1时，右旋；
- 当最小不平衡子树根节点的BF小于-1时，左旋。

（左旋和右旋均只涉及一些指针操作，比较简单，以右旋为例：

<img src="5-1-5.png" alt="image-20230411165107253" style="zoom:67%;" />

<img src="5-1-4.png" alt="image-20230411165131157" style="zoom:67%;" />

同时需要注意，当插入结点之后，最小不平衡子树根节点的BF与它的子树的BF符号相反时，需要先对**子树**进行旋转使得子树与根的BF符号相同之后，再对**根节点**进行一次**反向旋转**之后才能平衡。

注1：什么时候最小不平衡子树根节点的BF与它的子树的BF符号相反？什么时候相同？
左左（插入点位于最小不平衡子树根节点的左子节点的左子树）和右右（插入点位于最小不平衡子树根节点的右子节点的右子树）BF符号相同：以左左为例，根的BF为2，左子节点的BF为1；

左右（插入点位于最小不平衡子树根节点的左子节点的右子树）和右左（插入点位于最小不平衡子树根节点的右子节点的左子树）BF符号不同：以左右为例，根的BF为2，左子节点的BF为-1.

注2：以左右为例说明**双旋转**的过程：根据根节点的BF为2判断应该进行**右旋**，但是检查左子节点的BF为-1，所以应该先对左子节点进行左旋（因为左子节点的BF是负数），再对根节点进行**右旋**。

<img src="5-1-3.png" alt="image-20231108094714363" style="zoom:67%;" />



## RB-tree（红黑树）

RB-tree的定义，不仅是一个二叉搜索树，还必须满足以下规则：

1.每个节点不是红色就是黑色；

2.根节点为黑色；

3.如果节点为红，子节点必须为黑色；

4.任一节点到NULL（树尾端）的所有路径，所包含的黑节点的数目必须相同。

默认NULL节点为黑色。

根据规则4，新增节点必须为红：如果新增一个黑色节点，那么其父节点经过这个新增节点到达NULL的黑节点数目一定大于经过其它路径到达NULL的黑节点数目。

根据规则3，新增节点的父节点一定是黑色：因为新增节点为红色，因此父节点不能为红（父子不能同时为红）

当新节点根据二叉搜索树的规则到达插入点，却不能符合上面的规则时，需要调整颜色和旋转树形。

**补充内容：**

红黑树能保证：其最长路径中节点个数不会超过最短路径节点个数的两倍。因为最短路径就是全黑，最长路径就是红黑节点交替，因为每条路径的黑色节点数目都相同，所以最长路径就刚好是最短路径的两倍。

也就是说，相比于AVL树，RB树并不严格追求绝对平衡。

### 红黑树的插入操作

红黑树是在二叉搜索的基础上进行平衡，因此红黑树的插入步骤可以分为两步：1.按照二叉搜索树的规则找到插入点；2.判断新节点插入之后是否破坏红黑树的条件。

新节点插入时默认是红色，如果其父节点是黑色，那么无需做任何调整，因为没有违反红黑树的性质；如果**父节点是红色**，就违反了性质3，需要进行调整，调整分为以下几种情况：

**父亲为祖父的左儿子：**

情况一：父亲和叔叔都是红色，此时无论X是父节点的左孩子还是右孩子，均进行以下调整：
（1）父亲和叔叔变为黑色，满足规则3：父子不能同时为红；叔叔需要变成黑色是因为需要保证叔叔和父亲的路径上黑色数目需要相同
（2）祖父变成红色，满足规则4：因为父亲和叔叔都是黑色，黑色的高度发生了变化，如果祖父还是黑色，会导致这条路径上黑色数目比原来增加；
（3）从祖父开始，继续进行调整，这是为了防止曾祖父为红色，出现祖父和曾祖父同时为红色的情况。

<img src="5-2-1.png" alt="image-20231108153803573" style="zoom: 80%;" />

情况二：叔叔是黑色（如果叔叔不存在，即为NULL节点，也默认为黑色），自己是父亲的左孩子：
（1）父亲变成黑色，祖父变成红色。此时右子树的黑色高度减少；
（2）对祖父进行**右旋**，令父节点称为新的祖父，这样右子树的黑色高度得以恢复.

<img src="5-2-2.png" alt="image-20231108154129372" style="zoom:80%;" />

情况三：叔叔是黑色（如果叔叔不存在，即为NULL节点，也默认为黑色），自己是父亲的右孩子：
（1）对父亲进行左旋操作，使得父亲成为新的X，此时称为上面情况二；
（2）按照情况二进行处理。

<img src="5-2-3.png" alt="image-20231108154555941" style="zoom:80%;" />

**父亲为祖父的右儿子：**

情况一：父亲和叔叔都是红色，此时无论X是父亲的左孩子还是右孩子，均进行以下处理：
（1）父亲和叔叔都变成黑色，保证父亲不会与孩子X同时为红色。叔叔也需要变成黑色是因为需要保证叔叔和父亲的路径上黑色数目需要相同；
（2）祖父变成红色，满足规则4：因为父亲和叔叔都是黑色，黑色的高度发生了变化，如果祖父还是黑色，会导致这条路径上黑色数目比原来增加；
（3）从祖父开始继续调整，这是为了防止曾祖父为红色，出现祖父和曾祖父同时为红色的情况。

<img src="5-2-4.png" alt="image-20231108155505164" style="zoom:80%;" />

情况二：叔叔是黑色，自己是父亲的右孩子：
（1）父亲变成黑色，祖父变成红色，此时左子树的黑色高度降低；
（2）对祖父进行左旋操作，恢复左子树的黑色高度。

<img src="5-2-5.png" alt="image-20231108155643934" style="zoom:80%;" />

情况三：叔叔是黑色，自己是父亲的左孩子：
（1）对父亲进行右旋操作，使得父亲成为新的X，构造上面情况二；
（2）按照情况二进行处理。

<img src="5-2-6.png" alt="image-20231108155801673" style="zoom: 80%;" />

**总结：**

当父亲为祖父的左孩子时：父叔同色，只进行变色；父叔异色，自己是左孩子，进行R操作；父叔异色，自己是右孩子，进行LR操作；

当父亲为祖父的右孩子时：父叔同色，只进行变色；父叔异色，自己是右孩子，进行L操作；父叔异色，自己是左孩子，进行RL操作。

### 针对父叔同色的改进

当父叔同色时，对祖父和父叔进行颜色修改之后，需要继续向上对曾祖父进行同样的处理。为了避免自下而上的处理，可以进行一个自上而下的处理：假设新增节点为A，那么验证从根节点到A的路径，只要发现**某个节点X**的两个子节点都是红色，就将X改为黑色，两个子节点改为黑色。

这样处理之后，如果这个**节点X**的父节点是红色（此时叔叔不可能是红色，因为上一步中已经把叔父同时为红的情况处理掉了），就需要将X节点视作新增节点，按照情况二（1次旋转）或三（2次旋转）进行处理。

<img src="5-2-7.png" alt="image-20231108195146250" style="zoom: 50%;" />

处理完所有X节点之后，就可以对**新增节点A**进行处理，此时要**么直接插入**（新增节点的父节点是黑色），要么插入后进行一次单旋转（新增节点的父节点是红色，叔叔一定是黑色）。
不理解为什么作者说一定是一次单旋转，上面的处理过程并不能保证父节点一定是祖父的左孩子还是右孩子，而插入位置也不能保证是父节点的左孩子还是右孩子，因此我认为有可能需要进行两次旋转。

### 红黑树的节点设计

STL红黑树的节点分为一个base类和一个派生类：

<img src="5-2-8.png" alt="image-20231108195547831" style="zoom:67%;" />

<img src="5-2-9.png" alt="image-20231108195757492" style="zoom:67%;" />

红黑树的节点示意图：

<img src="5-2-10.png" alt="image-20231108195857546" style="zoom:80%;" />

### 红黑树的迭代器

红黑树的迭代器同样采用继承的架构，分为base迭代器和派生迭代器，两层迭代器与节点之间的关系为：

<img src="5-2-11.png" alt="image-20231108201453628" style="zoom:67%;" />

红黑树的迭代器属于双向迭代器，其解引用和成员访问操作与list相似，比较特殊的是前进和后退操作。RB-tree迭代器的operator++和operator--操作分别调用基类迭代器的increment和decrement方法。

**基类迭代器实现：**

<img src="5-2-12.png" alt="image-20231108201751919" style="zoom:67%;" />

<img src="5-2-13.png" alt="image-20231108201908231" style="zoom:67%;" />

<img src="5-2-14.png" alt="image-20231108202927600" style="zoom:67%;" />

<img src="5-2-15.png" alt="image-20231108203539919" style="zoom:67%;" />

<img src="5-2-16.png" alt="image-20231108203606839" style="zoom:67%;" />

在实现时，会在树的根节点上多增加一个header节点，这个节点作为根节点的父节点，并且header的父节点就是根节点。header节点的左孩子是二叉排序树的第一个元素，右孩子是二叉排序树的最后一个元素。同时header也作为end()，左孩子是begin()。

<img src="5-2-19.png" alt="image-20231108210025242" style="zoom:67%;" />

increment函数中的情况4：当迭代器指向根节点时，并且根节点没有右孩子，也就是没有直接后继，此时node指向的是根节点，经过while循环处理之后node指向header，y指向根节点，因此node->right就等于y，因此node就是最终结果。

decrement中情况1：当迭代器指向end也就是heaher时，减一操作应该指向header的右孩子，也就是二叉搜索树的最后一个值。

**红黑树的迭代器：**

<img src="5-2-17.png" alt="image-20231108205828143" style="zoom:67%;" />

<img src="5-2-18.png" alt="image-20231108205844531" style="zoom:67%;" />

### 红黑树的数据结构

<img src="5-2-20.png" alt="image-20231108210901044" style="zoom:67%;" />

<img src="5-2-21.png" alt="image-20231108211223017" style="zoom:67%;" />

<img src="5-2-22.png" alt="image-20231108211342864" style="zoom:67%;" />

<img src="5-2-23.png" alt="image-20231108211553220" style="zoom:67%;" />

<img src="5-2-24.png" alt="image-20231108211801641" style="zoom:67%;" />

<img src="5-2-25.png" alt="image-20231108211931545" style="zoom:67%;" />

### 红黑树的构造与内存管理

通过上面的源代码可以看出，红黑树的构造分为两种，一种是以现有的红黑树赋值一个新的红黑树，另一种就是产生一棵空树。但是这个空树会存在一个header节点：

<img src="5-2-26.png" alt="image-20231108212304824" style="zoom:67%;" />

因为增加了一个header节点，所以在进行插入新结点时，不仅需要根据红黑树的规则进行调整，还必须维护header的正确性，令其父节点是根节点，左孩子是最小节点，右孩子是最大节点。

### 红黑树的元素操作

这里主要介绍红黑树的插入和查找操作。

#### 元素的插入操作

红黑树提供两种插入操作，一种是插入节点的键值（key）必须在整棵树中独一无二，即insert_unique()，如果已经存在相同的键值，那么插入就不会进行；另一种是插入节点的键值可以在树中重复，即insert_equal()。

下面以最简单的版本（只接受一个实值value表示被插入节点）说明，虽然只制定了value，但是通过KeyOfValue仿函数可以从value中获取key。

**insert_euqal：**

<img src="5-2-27.png" alt="image-20231109082641967" style="zoom:67%;" />

**insert_unique：**

<img src="5-2-28.png" alt="image-20231109084635562" style="zoom:67%;" />

<img src="5-2-29.png" alt="image-20231109084701450" style="zoom:67%;" />

下面以comp标准为“小于”，说明while循环结束之后的一段代码。

当while循环结束之后，此时x指向插入节点，y指向插入节点的父节点，并且y一定是叶子节点。（这是因为二叉搜索树寻找插入点的特性）此时y与x只有两种可能，要么x是y的左孩子，要么x是y的右孩子，下面分别讨论：

1.x是y的右孩子：
此时comp一定是false，因此x大于等于y才会是右孩子。因此跳过第一段if语句；
在第二段if语句中判断 j （此时与y相同）是否小于x，若是，则经过上面的条件(1)$y \le x$；和此时的条件(2)$y \lt x$可以得到y与x不同，因此可以正常插入；若否，则说明y与x相同。

2.x是y的左孩子：
此时comp一定是true，因此需要进入第一段if语句。首先判断当前的父节点是否为最左节点begin()，如果是，那么直接插入；否则就令 j 指向 y 节点的前一个节点；
接着判断 y 节点的前一个结点与 x 之间的关系。如果 j < x，结合上一步的条件 x < y，并且 j 是 y 的直接前驱，因此 x 不会发生重复；但是如果$j \ge x$，结合之前对于插入点的搜索过程，因为插入点在 j 的后面，所以$x \ge j$，此时就可以说明 j 与 x结点键值重复。

**真正的插入程序：__insert**

<img src="5-2-30.png" alt="image-20231109091217740" style="zoom:67%;" />

<img src="5-2-31.png" alt="image-20231109092128794" style="zoom:67%;" />

<img src="5-2-32.png" alt="image-20231109092223023" style="zoom:67%;" />

在插入操作完成之后，需要进行调整操作，也就是上面的**__rb_tree_rebalance()函数**，使得整棵树满足红黑树的性质：

需要调整的情况在红黑树的插入操作已经分析过，下面这段代码不难理解：

<img src="5-2-33.png" alt="image-20231109093530776" style="zoom:67%;" />

<img src="5-2-34.png" alt="image-20231109093943193" style="zoom:67%;" />

<img src="5-2-35.png" alt="image-20231109094017545" style="zoom:67%;" />

注：作者在这里说明上面的程序就是“自上而下的程序”，但是我的理解中这个程序对于树的调整是从插入点开始自下而上，并没有用到前面提到的针对父叔同色的改进方法。

下面是左旋和右旋操作的函数：

**左旋：**

<img src="5-2-36.png" alt="image-20231109094514372" style="zoom:67%;" />

<img src="5-2-37.png" alt="image-20231109095022913" style="zoom:67%;" />

**右旋：**

<img src="5-2-38.png" alt="image-20231109095052266" style="zoom:67%;" />

#### 元素的查找操作

由于红黑树是一个二叉搜索树，因此对于元素的查找非常简单：

<img src="5-2-39.png" alt="image-20231109095322824" style="zoom:67%;" />

