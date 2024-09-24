+++
title = 'CS61A学习笔记 基础序列《抽象》构建数据结构'
date = 2023-09-21T18:37:48+08:00
draft = false
description = ""
slug = "CS61A学习笔记 基础序列《抽象》构建数据结构"
image = "/media/image-24-1024x744.png"
categories = ["编程相关"]
tags = ["CS61A","Python","学习笔记","数据结构"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
keywords = ["CS61A","Python","学习笔记","数据结构"]
+++

## 一、字典

一个字典的声明：

<img src="/media/image-11.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

字典中的参数有二：键（key）和键值（value）。在上图中，"CA"是键，"California"是键值。

### 1、字典的操作

下面是一些操作字典的例子和方法：

<img src="/media/image-12-1024x253.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/image-13-1024x553.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/image-14-1024x439.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

### 2、字典的特性

- ①不存在多个相同的键

<img src="/media/image-15.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<p style="text-align: center"><em>会输出'a': 'agua"</em></p>

- ②键的引用必须是一个常量。例如：数字、字符串。

- ③键值的引用可以是任意种类。例如：

<img src="/media/image-16.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>在字典中引用了字典</em></p>

### 3、字典的迭代

一个简单的字典迭代如下图



这个迭代返回的是字典中的键，我们再在之后的代码中根据键来索引键值。

那么问题来了，索引到的键的排列顺序是怎样的呢？

答案是按照键的插入先后来排序。

<img src="/media/image-18.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

也就是说，在python3中，目前我们没有办法“获取某字典中第n个键值”。当我们需要这个功能时，应该考虑使用列表。

## 二、复合数据结构

### 1、一些可能的复合数据结构

<img src="/media/image-19-1024x365.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

### 2、构建矩阵（martix）

<img src="/media/image-20.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>一种可能的矩阵表示</em></p>

一种构建方式是使用列表+列表。

<img src="/media/image-21.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>行主序的矩阵表示</em></p>

- 这行代码是很有意思的，值得认真体会

```python
a = [ [1 for i in range(4)] for j in range(4)]
```


<img src="/media/image-22.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>列主序的矩阵表示</em></p>

值得注意的是，无论你采用的是行主序还是列主序，使用你程序的人都不会知道这个细节。

“程序员可以有自己的观点。我们可以用我们的观点来指导我们的实现，只需要为使用这个功能的人提供一个好的抽象。”

### 3、构建树

<img src="/media/image-24-1024x744.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>树的定义</em></p>

这一次，我们需要实现下面四种抽象封装：

<img src="/media/image-25-1024x250.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">


<img src="/media/image-27.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>列表+列表构建树</em></p>


<img src="/media/image-28-1024x673.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>元组+列表构建树，这个看起来舒服多了！</em></p>

### 4、树与递归

从上述的代码和表述中不难看出，树是一种递归结构。（我们也学过树递归）当我们需要递归时就可以用到它。

<img src="/media/image-29-1024x474.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

尝试写一个函数，输入一个树，返回该树的叶节点数量，可以有如下代码：

<img src="/media/image-30.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

从中我们可以窥见树递归的几个关键特性：

- ①边界情况往往是递归到叶结点，因为此时已经”无路可走“；

- ②进行下一步递归，只需要对子节点执行函数即可，用for很方便。

可以发现，我们构建出的树结构其实是一种非常便于递归的数据结构。

## 破坏与非破坏性函数

尝试写一个函数，输入一个树，将该树的每个节点的标签都翻倍。可以有如下代码：
<img src="/media/image-75.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

也可以将其简化成下面的代码：
<img src="/media/image-76.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<p style="text-align: center"><em>如果没有children就不会触发下一次double函数，故不会无限递归</em></p>

在这个过程中，我们实际上新建了一棵树来返回，并没有破坏原来的树。我们将这种操作理解成“非破坏性”的。

<img src="/media/image-77.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

类似的，变量也有可变和不可变之分：

<img src="/media/image-78.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

我们之前所的构建树操作和数据结构可以看作是非破坏性和不可变类型。我们也可以将其改变，比如将数据结构改为List嵌套List，就可以实现破坏性地操作树和构建树。

在对List的操作中，我们也可以见到一些破坏性/非破坏性的操作。值得注意的是非破坏性的操作会改变对内存的指向和引用，而破坏性的操作会直接修改原链表的内存本身。

<img src="/media/image-79.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/image-80.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<p style="text-align: center"><em>值得注意的是，这样操作会使b指向新的链表，但是+=不会，+=在链表中不是语法糖</em></p>

<img src="/media/image-81.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>切片操作是破坏性的</em></p>

<img src="/media/image-82.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>使用method来操作链表，破坏性</em></p>

## 相等（Equality）和同一（Identity）

<img src="/media/image-83.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/image-85.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

也就是说，同一代表了两对象完全相同，相等只代表了两对象的值相同。

我们可以用is 和 == 来分别判断同一和相等。一般来说，我们会在判断变量类型的时候使用is。其他时候，用is来代替==是比较危险的操作。

明显地，如果is返回True，那么==也会返回True。

<img src="/media/image-86.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>也可以写成这样的语法糖</em></p>

## 作用域（Scoop）

看看下面的例子：

<img src="/media/image-88.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

在第一个例子中，我们只是修改了链表对象，并没有进行赋值操作；在第二个例子中，我们实际上是在局部先定义了一个current变量，然后再进行current++，而这个全局变量还未被赋值就被引用，这就会引发UnboundLocalError。

解决这个问题的一个方法是添加global语句。这样在python执行到“current=”的赋值操作时，就会自动使用全局变量中的current，而不是在本地新建一个current变量。

<img src="/media/image-89.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

对于高阶函数，我们可以写nonlocal来查找并使用父框架的变量。

<img src="/media/image-90.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

我们应该尽量避免写global和nonlocal，因为这会使抽象函数充满不确定性，代码中的某个全局变量也很有可能在不经意间被修改。我们只需要将某个全局变量作为参数传入函数就好了。

## Fun fact

1

<img src="/media/屏幕截图 2024-09-24 222801.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

2

<img src="/media/image-26-1024x398.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

3

<img src="/media/910e51ffbf346d6f1f02c420c73610f6-1024x518.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
