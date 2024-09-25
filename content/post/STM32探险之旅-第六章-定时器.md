+++
title = 'STM32探险之旅 第六章 定时器'
date = 2023-11-06T18:56:59+08:00
draft = false
description = ""
slug = "STM32探险之旅-第六章-定时器"
image = "/media/image-15 (1).png"
categories = ["编程相关","嵌入式"]
tags = ["C/C++","STM32","定时器","HAL"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
keywords = ["C/C++","STM32","定时器","HAL"]
readingTime = true
math = true
+++

## 青春版定时器：WatchDog

混沌邪恶：看门狗也是定时器！

但是你说得对，看门狗本质上就是一个定时器。之所以将其称之为狗，是因为其存在两种状态：饿了和饱了。饿了它就要触发中断，饱了就啥事不干。从饱到饿消耗的时间，不就是我们想定的时么？而且这么一想，如果只是单纯地需要定时，看门狗的操作其实比真正的定时器来得更方便。

STM32中存在两种看门狗：IWDG（独立看门狗）和WWDG（窗口看门狗）。

- 独立看门狗（IWDG）由专用的低速时钟（LSI）驱动（40kHz），即使主时钟发生故障它仍有效。独立看门狗适合应用于需要看门狗作为一个在主程序之外 能够完全独立工作，并且对时间精度要求低的场合。

- 窗口看门狗由从APB1时钟（36MHz）分频后得到时钟驱动。通过可配置的时间窗口来检测应用程序非正常的过迟或过早操作。 窗口看门狗最适合那些要求看门狗在精确计时窗口起作用的程序。

<img src="/media/image (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>看门狗的分类及使用</em></p>

下面来配置STM32的IWDG和WWDG。

## IWDG

<img src="/media/image-1 (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

**溢出时间（ms）=((4×2^PR) ×RLR)/LSI时钟频率**

<img src="/media/image-2-edited.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>PR与预分频系数的关系</em></p>

**可以发现一个重要规律：预分频系数越高，溢出时间就越长。这条对之后的计时器同样适用。**

按照我们的参数，该IWDG的溢出时间在300ms左右。

当程序开始时，我们先初始化IWDG，其开始计时并使寄存器内的变量自减；

<img src="/media/image-3.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

程序正常执行后喂狗，如果不正常运行的话就会触发复位。

<img src="/media/image-4 (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

## WWDG

WWDG的“窗口”含义在于：你只能在其中一段窗口时间内喂狗。

<img src="/media/image-5 (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

我们可以设置2，也就是上限时间，但不能设置下限时间，也不能设置1到3的总时间。（不过可以调节分频系数来变相改变）当CNT减少到0x3f时，就会先触发WWDG中断（需要激活NVIC），并复位。我们可以在中断内喂狗，并进行自定义的操作。

<img src="/media/image-6 (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

- prescaler：预分频系数
- window value：刷新窗口上限
- free-running downcounter value：计时器时间（过程1-3总时间），无法更改。
- Early wakeup interrupt：若Enable，则中断将在CNT减到0x40时触发。

设置好参数后，我们仿照刚才的IWDG来初始化WWDG，并在main函数中加入wwdg的中断回调函数。

<img src="/media/image-7 (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

也就是当wwdg刷新时就翻转小灯亮灭。

最后的效果：

{{< video src="/media/aa9373b4f89bbf34fa6431273e2f79c1.mp4" style="display: block; margin-left: auto; margin-right: auto;">}}

<p style="text-align: center"><em>分频系数：8</em></p>

{{< video src="/media/4c6a9a581c0fb4c39987087080c7be7b.mp4" style="display: block; margin-left: auto; margin-right: auto;">}}

<p style="text-align: center"><em>分频系数：4 闪得太快摄像机拍不出来了</em></p>

## TIM定时器

STM32一共有三种不同类的定时器，F103板上共有八个。

<img src="/media/image-8 (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

需要补充的是，基本定时器在APB1<sup><a href="#note1">1</a></sup>上，而通用定时器和高级定时器都在APB2上。

“计数模式”指的是寄存器中CNT变量的变化方式，如下图所示：

<img src="/media/image-9 (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

我们之前使用的WDG，使用的就是向下计数。

### 基本定时器与定时器中断

基本定时器的功能和使用方法和WDG类似。计时器的CNT变量以特定方式变化，到达临界值后溢出产生中断，我们通过编写中断回调函数来实现自定义功能。

首先在Cube中启用TIM6基本定时器。

<img src="/media/image-10 (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

我们需要Enable auto-reload preload，以此在更新事件(UEV)产生后再开始新的计数，**保护原来的计数周期不受影响。**

我们可以用下面的公式来计算计时器的溢出时间。若使PSC为7199，则溢出时间就等于重装载值/10。

$$
    T = \frac{(arr + 1) * (psc + 1)} {Tclk}
$$

<p style="text-align: center"><em>psc：预分频器（系数）；arr：AutoReloadRegister，自动装载寄存器；Tclk：APB1或2上的时钟频率</em></p>

<details>
<summary>为什么？</summary>

----------

事实上，定时器除了指定CNT的临界值之外，还需要指定它的分频器PSC（Prescaler）。这是因为：

- 时钟频率太快。16位CNT的临界值范围在0～65536，而对于72MHz的时钟频率来说，跑完65536只需要0.9ms。

- 计算不便。如果不分频，72MHz的时钟对应每周期1/72us，十分不利于计算。

PSC的原理是：时钟源每Tick一次，PSCCNT++，直到达到PSCCNT的临界值，下一次Tick会使PSCCNT归零，CNT++。可以简单把这个原理理解成给CNT又加了一层循环嵌套。

对于基本定时器，时钟脉冲的来源是HCLK/1 - APB1总线/2 - APB1计时器*2 = 72MHz，所以只需要指定PSC为7199即可达到分频的目的。

----------

</details>

首先，和WDG类似，我们需要先初始化定时器。

<img src="/media/image-14.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>初始化函数别写错了！</em></p>

接着找到定时器中断相关的回调函数，在main.c中重新定义它。

<img src="/media/image-13-1024x377.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>需要对传入的定时器进行判断</em></p>

刷版运行。

{{< video src="/media/0caa7f8d26e989806fe845845f7bc696.mp4" style="display: block; margin-left: auto; margin-right: auto;">}}
<p style="text-align: center"><em>ARR=4999</em></p>

{{< video src="/media/32f490159c58d7b88ca57aa337bb48bd.mp4" style="display: block; margin-left: auto; margin-right: auto;">}}
<p style="text-align: center"><em>ARR=9999</em></p>

通用计时器的中断溢出的操作方法与基本定时器基本一致，故不再赘述。

## 通用定时器输出PWM脉冲

<img src="/media/image-15 (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>定时器与PWM</em></p>

所谓的PWM，就是图下方的那一条方波。图上方的部分是我们的通用定时器中的CNT变量。（向上计数）我们设定一个值CCRx，当CNT<CCRx时，PWM输出0；反之输出1。一个PWM周期的时长就是计时器的溢出时长。我们把PWM输出1的时间占定时器占一个PWM周期的比例称为占空比。

我们可以用PWM来做什么呢？一个很简单的例子是PWM呼吸灯。（或者使用PWM控制小灯亮度）

众所周知，要控制小灯的亮度，只需要改变电压的大小。我们只需要将PWM复用输出到GPIO_OUT，就能使得PWM控制模拟电压的大小。

什么，还是听不懂？那么回想一下高中物理的知识。对于方波交流电路，其电压的有效值可以通过下面的公式来算出：

<img src="/media/image-16 (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

可以看出，我们只需要改变t和T的比值，也就是占空比，就能改变电压的有效值。进一步，我们就能控制小灯的亮度。

那么接下来，我们就来尝试使用PWM控制GPIO口模拟电压输出，并且通过改变PWM占空比来改变小灯的亮度。

首先在ioc文件中将PA5改为复用输出TIM3_CH2；

<img src="/media/image-17 (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

接着在TIM3属性设置中启用CH2的PWM输出；

<img src="/media/image-18 (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

然后更改TIM3的PWM参数；

<img src="/media/image-19.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

如图所示，根据刚才基本定时器的计算公式，我们可以得到该定时器的溢出时间（PWM周期）是0.5ms。对我们本次的例程来说，更小的PWM周期意味着更精确的电压控制。

对于CH Polarity参数，由于我们的小灯是低电平亮，所以将其设置为Low。

目前我们初始化了PWM的周期，接下来我们只要在程序中给定CCRx，确定占空比，即可改变小灯的亮度。

接下来，我们就来试试通过按键来控制小灯亮度。

首先配置Key0和Key1的中断。

<img src="/media/image-21 (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

和之前类似的，我们也要先初始化PWM输出。我们还要定义PWM占空比。

首先定义PWM周期：

<img src="/media/image-24 (1).png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

然后计算PWM周期，初始化PWM占空比。

<img src="/media/image-25-1024x699.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

当Key0按下时，亮度减少：

<img src="/media/image-26-1024x536.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<footer>
  <p id="note1">1. APB（Advanced Peripheral Bus，先进外设总线），挂载在AHB（Advanced High Performance Bus，先进高性能总线）上。</p>
</footer>