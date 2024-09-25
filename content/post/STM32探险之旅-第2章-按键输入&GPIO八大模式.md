+++
title = 'STM32探险之旅 第2章 按键输入&GPIO八大模式'
date = 2023-09-22T14:59:19+08:00
draft = false
description = ""
slug = "STM32探险之旅-第2章-按键输入&GPIO八大模式"
image = "/media/image-32.png"
categories = ["编程相关","嵌入式"]
tags = ["GPIO","STM32"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
keywords = ["GPIO","STM32"]
readingTime = true
+++

## 一、按键输入

和上次一样，我们还是先从原理图上找到按键对应的引脚。

<img src="/media/image-31.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

### 1、浮空输入模式

先从KEY_UP开始。我们只需要设定PA0为“浮空输入模式”，就能让引脚内部处于高电阻态。当我们按下按钮，此时电路连通，在PA0处几乎没有多少压降，读取到的电压也就是3.3V的高电平；松开按钮，PA0就又会回到0V。我们只需要用函数来读取这个地方的电平即可进行更多操作。

<img src="/media/image-32.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>浮空输入模式</em></p>

### 2、上拉/下拉输入

接下来看到KEY0和KEY1。对于这两枚按键，我们明显不能只使用浮空输入模式。这个时候就需要用到“上拉/下拉输入”模式。

<img src="/media/image-33.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

我们在原本的电路上加一个上拉电阻和一个VDD输入。当按键松开，PB12处为3.3V（芯片内部高阻态）；当按键按下，PB12处为0V，这样就达到了我们的目的。

<img src="/media/image-34.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>有上拉，也就有下拉，原理相同，效果相反</em></p>

然而我们的原理图上并没有什么上拉和下拉电阻，因此我们就需要用到程序自带的上/下拉电阻。
<img src="/media/image-36.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>效果和刚才的电路图一致</em></p>

至于为什么能做到和外加上拉电阻一样的效果，我们会在本章的结尾解释。

## 二、控制小灯

在这部分，我们使用Key1和2来对红色和绿色小灯进行控制。

我们的目标是：按下Key1翻转红色小灯亮灭；按住Key2绿色小灯亮起，松开Key2绿色小灯熄灭。

### 1、翻转亮灭

根据前文所述，我们首先需要用函数读取PE4和PE3的电平状态。于是我们使用HAL_GPIO_ReadPin函数。

<img src="/media/image.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>两个参数分别对应引脚组和引脚，返回值是高/低电平</em></p>

对于翻转亮灭来说，我们可以用if...else...来分别给PB5引脚赋值高电平或低电平。不过我们有更方便的函数来直接翻转特定引脚的电平。

<img src="/media/image-0.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/image-99.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>当读取到Key2（PE3）低电平时，翻转红色小灯引脚电平</em></p>

将这一语句放进while循环中，并刷入学习板。
{{< video src="/media/8388914d6e1c4e2c0501ba12e15e9fa6.mp4" style="display: block; margin-left: auto; margin-right: auto;">}}
我们发现小灯的亮灭似乎不怎么受控制。

实际上，在按键按下和松开的时候，由于人手指和电路板的抖动，引脚PE4电路的状态并不是完全闭合，而是在闭合和断路之间不断切换。这时芯片程序便会反复进入if语句并且触发HAL_GPIO_TogglePin函数，最终结果的体现就是小灯的亮灭不受控制。

我们可以使用软件消抖来解决这个问题。

<img src="/media/image-100.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

在读取到按下Key1后，首先延时10ms，这时再读取到的电平就应该是稳定的电平。

接下来翻转亮灭，并且开始死等，直到读取到稳定的Key1松开的电平（SET），再结束这个if语句。这样就避免了多次进入if语句并触发TogglePin函数。

{{< video src="/media/dcd3d626dd8c6d3a644f0ec2d02d4c16.mp4" style="display: block; margin-left: auto; margin-right: auto;">}}
<p style="text-align: center"><em>稳定的翻转控制</em></p>

实际上，除了上述的软件消抖策略外也有硬件消抖的策略。下图是一种硬件消抖的实现：
<img src="/media/image-101.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

2、按住按键

这部分的逻辑也很简单。检测到Key2按下，就将PE5写成低电平；检测到Key2松开，就将PE5写成高电平。结合刚才提到的软件消抖，我们可以写出这样的代码并且成功编译刷板运行。

<img src="/media/image-102.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

三、GPIO八大模式详解

{{< bilibili BV1zG4y1K78S>}}

