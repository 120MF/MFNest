+++
title = 'CS61A学习笔记 序列与for循环'
date = 2023-09-18T14:10:58+08:00
draft = false
description = ""
slug = "CS61A学习笔记-序列与for循环"
image = "/media/屏幕截图-2023-09-18-140426.png"
categories = ["编程相关"]
tags = ["CS61A","Python","学习笔记","数据结构"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
keywords = ["CS61A","Python","学习笔记","数据结构"]
+++

包含Lecture 10 & 11

## 一、序列

一般序列可能具有的特性：

有限/无限性、可变/不可变性、可索引/不可索引性、迭代/不可迭代性

<img src="/media/屏幕截图-2023-09-18-140426.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<p style="text-align: center"><em>Python中的一些序列</em></p>

_PS：range的括号内变量取法和切片操作的变量取法是一致的。_

序列之间也可以相互转换。例如：

<img src="/media/屏幕截图-2023-09-18-190036.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

## 二、字符串

### 1、引号

下面是字符串的一些使用方法（单引号、双引号、三引号）

<img src="/media/屏幕截图-2023-09-18-140923.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

### 2、切片

字符串切片的语法是：`str[<start>:<end>:<step>]`

`<start>`是字符串的开始，`<end>`是字符串的结尾，`<step>`是截取的步长。

这三个参数都有一个对应的默认值，分别是0、-1、1。当step为-1时，start和end的默认值相反。（因为是反向切片，所以要从尾端往前切）

<img src="/media/屏幕截图-2023-09-18-185652.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<p style="text-align: center"><em>字符串中索引的表示</em></p>

<img src="/media/屏幕截图-2023-09-18-185102.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<p style="text-align: center"><em>一些字符串切片的示例</em></p>

值得注意的是，在python中，`[1:4]`表示从1到4，包含首端，不包含尾端。类似的有：`range(1,4)` = `(1,2,3)`

这在python或其他编程语言里算一种不成文的规定。

当然，切片不仅限于字符串，其他序列也可以使用切片功能。

## 三、使用序列迭代：For循环

### 1、for与while

for循环的本质是语法糖，以一种方便的方法省略了while循环中计数器的存在。

```python
s = (1,2,3,4,5)
n = 0
while(n<=5):
  print(s[n])
  n++
# which is equal to
for i in s:
  print(i)
```

如果要实现不同的步进的话，只需要更改in后序列的切片即可，非常方便！

```python
for i in s[0:5:2]:
  print(i)
#>>>1 3 5
```

### 2、多变量赋值

<img src="/media/屏幕截图-2023-09-18-191018-1.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

也就是x,y = (1,9)的形式。这意味着等号右边一定是一个序列，且序列内的元素个数与左边变量个数相等。

### 3、通过切片来更改序列

<img src="/media/屏幕截图-2023-09-18-194154.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

### 4、使用if迭代for循环

<img src="/media/屏幕截图-2023-09-18-195322.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>其实如果你不加if，expression也能正常运行。这样你就拥有了一个一行内的for循环。</em></p>

### 5、循环的嵌套（同一行）

<img src="/media/屏幕截图-2023-09-18-195447.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>可以看出python真是很自由的语言</em></p>

## 小技巧

<img src="/media/屏幕截图-2023-09-18-194038.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>用zip函数结合两个序列</em></p>