+++
title = 'CS61A学习笔记 迭代器和生成器'
date = 2023-10-21T16:48:22+08:00
draft = false
description = ""
slug = "CS61A学习笔记-迭代器和生成器"
image = "/media/image-118.png"
categories = ["编程相关"]
tags = ["CS61A","Python","迭代"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
keywords = [""]
readingTime = true
+++

## 迭代器

我们在之前学习过Python中的可迭代变量，比如：

<img src="/media/image-92.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>可以对这些变量进行for...in迭代循环操作</em></p>

<img src="/media/image-94.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>可以判断变量是否可迭代</em></p>

迭代器和迭代变量不同。迭代器是一种对象，能够逐个访问可迭代对象的值。我们用iter()来新建一个迭代器，用next()对迭代器进行迭代。

<img src="/media/image-93.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<img src="/media/image-96.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>对字符串进行迭代操作</em></p>

当next()迭代到最后一个元素的下一个元素时，就会返回StopIteration。

<img src="/media/image-95.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

可以用try/except语句来处理StopIteration异常。

<img src="/media/image-97.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

那么为什么不用For循环来进行这些操作呢？

## For循环和迭代器

python中的for循环本质上是一种语法糖，执行的就是刚才的迭代过程。

<img src="/media/image-98.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>For循环的幕后</em></p>

<img src="/media/image-116.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>iter和next实际上都是可迭代变量的方法之一</em></p>

实际上，Python优化了For循环，使其在运行速度和效率上都比直接写迭代器的方法要优秀。

<img src="/media/image-116.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

所以正常情况下我们应该使用for循环，但是了解for循环和iter、next、StopIteration机制也是非常重要的。

## 迭代器类型（Iterator）

有很多返回迭代器的函数。比如下面这个reversed函数。

<img src="/media/image-118.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/image-119.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>可以将返回的迭代器转换为链表，避免迭代器的追踪引发的错误</em></p>

还有之前提到的zip函数。

<img src="/media/image-120.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/image-121.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>对多个元素进行zip</em></p>

以及map和filter。

<img src="/media/image-122.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>可以替代之前学过的方法</em></p>

生成器

你可以把它理解成函数中带return功能的断点。

<img src="/media/image-123.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

使用yield会使函数返回一个迭代器类型。你可以对这个迭代器进行next()操作，这样函数就会从yield的位置再次开始执行。

<img src="/media/image-124.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>return的值会被放在StopIteration里</em></p>

<img src="/media/image-125.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>进行各种各样的yield</em></p>

当我们需要处理很多的输入或者输出时，使用yield可以按需生成，避免一次性开出过大的内存。

比如，我们可以用yield来实现斐波那契数列。

```python
import sys
 
def fibonacci(n): # 生成器函数 - 斐波那契
    a, b, counter = 0, 1, 0
    while True:
        if (counter > n): 
            return
        yield a
        a, b = b, a + b
        counter += 1
f = fibonacci(10) # f 是一个迭代器，由生成器返回生成
 
while True:
    try:
        print (next(f), end=" ")
    except StopIteration:
        sys.exit()
```

在一些情况下我们可以使用语法糖：yield from语句。
<img src="/media/image-126.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

Fun Fact
<img src="/media/image-91.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">