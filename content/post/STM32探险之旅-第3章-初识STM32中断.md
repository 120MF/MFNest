+++
title = 'STM32探险之旅 第3章 初识STM32中断'
date = 2023-10-06T14:35:48+08:00
draft = false
description = ""
slug = "STM32探险之旅-第3章-初识STM32中断"
image = "media/image-14-1024x180.png"
categories = ["编程相关","嵌入式"]
tags = ["C/C++","STM32","中断"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
keywords = ["C/C++","STM32","中断"]
readingTime = true
+++

## 一、引入

设想有这样一个需求：红色小灯以2s为周期循环亮灭闪烁，并且当检测到Key1按下后翻转绿色小灯亮灭。

我们新建一个工程，分配引脚，并且写以下代码刷板运行。

<img src="/media/image-103.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

可以看到板子运行的结果很奇怪。除非长按Key1，绿色小灯几乎不受控制。

{{< video src="/media/adf26ec4829843e42150f5d9b2055eb4.mp4" style="display: block; margin-left: auto; margin-right: auto;">}}

这实际上是因为在while(1)循环的过程中，大部分时间都在HAL_Delay(2000)，真正给到Key1的判断时间很短。

<img src="/media/image-8-1024x195.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

那么有没有办法让STM32即使正在执行Delay这种耗时的任务，也能快速相应按键按下这种突发状况呢？这就需要我们用到“中断”了。

## 二、外部中断

在IOC文件中更改原先的PE3引脚类型为“EXTI_13"。"EXTI"是"External Interrupt"，即外部中断。

<img src="/media/image-104.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

再移动到旁边的GPIO选项卡，可以看到PE3对应的GPIO mode也发生了变化。

<img src="/media/image-10-1024x192.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

所谓上升沿就是某个GPIO口低电平到高电平的变化，下降沿就是高电平到低电平的变化。

我们选择下降沿触发中断，并选择下面的上拉输入，这样当Key1按下，PE3处的电平就会由高电平变为低电平，从而进入中断任务。

接着，我们在NVIC选项卡中勾选上新建的中断向量。

<img src="/media/image-105.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

然后我们保存并生成代码，在下图的目录位置中找到控制中断相应的文件。

<img src="/media/image-12.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>it代表interrupt相关</em></p>

进入文件，在该文件最下方的"xxxx_IRQHandler"就是CubeMX自动生成的中断控制函数。

<img src="/media/image-13.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

STM32的运行过程现在可以用下图表示：

<img src="/media/image-14-1024x180.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

从图中也能看出来，我们需要保证中断函数的过程尽可能短，这样才不会影响正常的执行流程。

那么，我们在第一个注释对中写下控制小灯翻转亮灭的代码。

<img src="/media/image-106.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>进行了软件消抖</em></p>

编译刷板，结果如下：

{{< video src="/media/198bedd671b45784a9dca80f7a0f9ee2.mp4" style="display: block; margin-left: auto; margin-right: auto;">}}

正常运行的小灯在按下Key1之后直接停止运行，这是怎么回事呢？

其实，这是因为HAL_Delay()函数依赖于System tick timer（系统滴答）时钟中断。在NVIC选项卡中，可以发现该中断优先级比我们的EXTI中断优先级更低，这也就导致了HAL_Delay()函数无法在我们的EXTI中断函数中运行，程序也就卡死在这行代码上了。

知道了这一点，我们只需要调整优先级，让系统滴答优先级高于EXTI优先级，就能使程序正常运行了。

<img src="/media/image-107.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

（编译刷板）

{{< video src="/media/62a80b67c557b4ed8af196c358fc6606.mp4" style="display: block; margin-left: auto; margin-right: auto;">}}

## 三、详细原理

这次实战中我们接触到了EXIT、NVIC、中断向量、中断优先级等等STM32的概念。通过下面的视频，我们可以了解到这些概念在STM32内部实现的原理。

{{< bilibili BV1M24y1473t>}}