+++
title = 'STM32探险之旅 第4章 STM32串口通信'
date = 2023-10-10T07:24:43+08:00
draft = false
description = ""
slug = "STM32探险之旅-第4章-STM32串口通信"
image = "/media/image-22-1024x341.png"
categories = ["编程相关","嵌入式"]
tags = ["C/C++","STM32","串口","配置","HAL"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
keywords = ["C/C++","STM32","串口","配置","HAL"]
readingTime = true
+++

通过前面几章的学习，我们现在可以编写代码和程序，将其编译到学习板上并运行。

ST-Link确实是一种方便的调试接口。然而实际应用中，我们肯定需要比这更快速且更方便的方法来和STM32通信。

设想有这样一个需求：红灯每隔1秒翻转亮灭，STM32从外界接收指令。若收到1则绿色小灯亮起，0则熄灭。

我们先来通过串口通信实现接受指令并控制小灯亮灭。

## 一、初识串口

查询开发板原理图，可以找到我们开发板上的串口。

<img src="/media/image-22-1024x341.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>自带USB转换</em></p>

在这幅图中有一些待解释的关键名词。

首先，RX代表接收端（Receive），TX代表发送端（Transmit）。两设备分别对应的发送和接收端是相反的。

其次，USART代表了"_Universal Synchronous/Asynchronous Receiver/Transmitter_"（通用同步/异步接收/发送器）。USART有许多标准，比如RS-232、RS-485、TTL（使用COM接口），通用串行总线（使用USB接口）。在我们板子上的这一部分是使用TTL标准的。

<img src="/media/dc011a70fdf8969f1ca7d4ab350d2729_720-768x1024.jpg" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>板子上的串口</em></p>

啊？可是这是Type-C的接口啊？

你说得对，但是这是因为开发商为了方便我们使用，所以直接将TTL的四针转出了USB。我们只需要将电脑与这个USB口相连，就可以在电脑上收发数据了。

<img src="/media/image-23.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>TTL信号一般都走这种COM串口，你猜厂商为什么没给你做这样的口？</em></p>

传输数据的过程中，两设备在TX-RX上按照约定好的频率发送-接受高低电平（1/0），就可以完成数据的发送和接收。
<img src="/media/image-25-1024x458.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

有时，两设备之间也会连接一根地线，这是为了方便规定零电势，避免可能产生的高低电平识别障碍。

<img src="/media/image-26-1024x634" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

## 二、串口轮询模式

进入STM32CubeIDE的ioc文件，选择选项卡中的Connectivity，找到USART1并将其开启。
<img src="/media/image-24.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>将模式调为异步(Asynchronous)</em></p>

在下方可以调整一些基本的参数。
<img src="/media/image-110.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
Baud Rate（波特率）指的是每秒发送的字节数。对于TTL协议来说，每次发送的内容都是1Byte(8bit)，即一字节，加上开头的0和结束的1共10bit。也就是说115200 Bits/s的波特率对应下来就是11520 Byte/s，即11.52 KB/s。两设备必须使用相同的波特率才能正常通信。这也是异步通信的特点之一。

下面分别是字节长度、校验位、停止位，保持默认即可。

按下Ctrl+S生成代码，可以发现IDE帮我们生成了USART的初始化函数。

那么接下来就让我们试着让STM32向我们每秒发送一次信息吧！

### 1、发送

首先在自定义变量区定义一个用来储存发送内容的数组。

<img src="/media/image-29.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>PV→Private Variable</em></p>

为什么用uint8_t数据类型？因为这种类型可以存放多种内容，且长度均为8字节。

我们接下来使用的函数也将这种变量作为形参。

<img src="/media/image-111.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>别忘了在开头引用string.h库</em></p>

第一个参数所使用的指针（代表使用的接口）已经在程序开头被自动定义。最后一个参数代表是等待时间。HAL_MAX_DELAY是其最大值。

<img src="/media/image-112.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

刷板完成后，我们用USB数据线连接UART接口到电脑USB-A口上，再到串口调试软件处接收数据。这边我们使用了Bilibili UP主 Keysking开发的串口助手。（[https://serial.keysking.com/](https://serial.keysking.com/)）

{{< video src="/media/QQ20231010-221654.mp4" style="display: block; margin-left: auto; margin-right: auto;">}}

这下我们就成功实现了STM32向电脑传输信息的功能。

### 2、接收

像刚才一样，定义一个用于接收信息的数组。

用于接收的函数和发送的函数相似，如图所示：

<img src="/media/image-113.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>一直等待输入，读取到输入之后就输出读取到的输入值</em></p>

刷板运行，效果如图：

{{< video src="/media/QQ20231010-222629.mp4" style="display: block; margin-left: auto; margin-right: auto;">}}

通过这个视频，我们也可以看出，超过传输长度的数据会被堵塞在通道中，一直到下一次才会被读取。在这个过程中，实际上CPU一直密切监督着寄存器进行数据的发送和储存，不会进行其它的工作。我们把这种串口的通信方式称为“轮询”。

回到我们一开始的需求：若收到1则绿色小灯亮起，0则熄灭。那么我们需要根据收到的内容来控制小灯的亮灭。我们可以写如下代码：

<img src="/media/image-114.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

这里我们使用了GPIO_PinState类型来初始化state，这样我们就可以把state作为WritePin函数的形参来传递了。

刷板运行如下：

{{< video src="/media/4657491d03cccabe60857eddc73dc469.mp4" style="display: block; margin-left: auto; margin-right: auto;">}}

成功运行！

## 二、串口中断模式

如果我们想让红灯在串口读取信息的过程中不断闪烁，又该怎么办呢？

我们必然不可能直接在while循环中加上控制红灯闪烁的代码——因为我们的CPU被串口通信的任务占用，没有办法不停地执行红灯闪烁。

<img src="/media/image-36-1024x359.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>一般称这种状态为“堵塞”</em></p>

——上面这段话有没有让你想起什么？没错，我们又要使用中断来解决这一问题。

进入ioc文件，在NVIC栏勾选上UART1，使其可以触发中断。

<img src="/media/image-35.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

串口的中断输入输出函数只需要在之前的函数名后加上_IT后缀即可。

<img src="/media/image-115.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>区别是没有等待时间，因为中断模式不阻塞CPU</em></p>

有了这个函数，我们可以大致构建出一个程序的模型：

1. 在程序开始就接收数据；

2. 接收到数据之后触发中断，在中断函数内部执行绿灯亮灭操作；

3. while循环内控制红灯亮灭。

<img src="/media/image-39.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<img src="/media/image-38.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>中断函数内部</em></p>

很合理的逻辑，不是吗？然而有一些问题：

- 接收的数据可能并不完整：

一旦接收到了字节，CPU就会处理中断函数。若是此时数据还没有被完整接收，就会使处理出现问题；

- 中断触发源有多种：

<img src="/media/image-40.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

可以看到，有多种事件均能触发该中断函数。

总之，我们需要确定产生中断的原因，并根据这个原因来触发我们控制绿灯亮灭的代码。在我们的实例中，这个原因应该是中断事件中的“接收数据就绪可读”。当STM32触发了这个事件，就会调用具体的函数来处理这个事件。在STM32中，对应的函数是一个回调函数：`HAL_UART_RxCpltCallback`。

<img src="/media/image-41.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>STMCubeIDE自动生成的回调函数，位于stm32f1xx_hal_uart.c</em></p>

<details>
    <summary>回调函数</summary>
<p>回调函数简单来说就是将函数作为参数来调用。还记得c语言中的qsort函数吗？这个函数接收一个自定义的比较函数作为其参数之一。我们把这个自定义函数称为“回调函数”。</p>

<p>函数 F1 调用函数 F2 的时候，函数 F1 通过参数给 函数 F2 传递了另外一个函数 F3 的指针，在函数 F2 执行的过程中，函数F2 调用了函数 F3，这个动作就叫做回调（Callback）。</p>

<p>奇怪的是，为什么我们要使用回调函数呢？</p>

<p>还记得吗？你曾经写过用qsort来排序字符串二维数组和结构体的程序！</p>

<img src="/media/c0c320bd2ccb1cd5d0413d07e969bce2.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>用qsort来排序一个二维数组</em></p>

<p>开发者不可能在.h库文件中将所有比较方法都列举一遍。如果使用回调函数，就能允许用户或其它开发者自定义地调用函数，大大提高了函数的灵活性和可用性。我们把使用回调函数得到的好处称为“解耦”。</p>
</details>

函数最开始的"__weak"代表了弱定义，也就是说，我们可以在程序的其它位置重新定义这个函数。只要接收完成，该函数就会被触发，接着NVIC触发中断，CPU转而执行函数内的内容。

现在我们可以构建一个新的程序的模型：

在程序开始就接收数据；

接收数据完成，触发HAL_UART_RxCpltCallback()函数，在该函数内部执行绿灯亮灭操作；

while循环内控制红灯亮灭。

进入main.c文件，编写代码。

<img src="/media/image-44.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>每次回调函数结束后都会再次执行接收函数，保证一直能够读入数据</em></p>

刷板运行一下：

{{< video src="/media/581edc94835b138ff593580f3c940747.mp4" style="display: block; margin-left: auto; margin-right: auto;">}}

成功运行！

## 三、串口DMA模式

无论是轮询和中断模式，我们总是看到CPU在勤勤恳恳地执行任务，在接收数据和正常任务之间来回辗转。有没有一种办法可以让信道之间的信息传输绕开CPU处理，从而减轻CPU的负担呢？还真有，那就是DMA，Direct Memory Access，直接内存读取技术。

<img src="/media/image-46.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/image-45.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

进行DMA传输，需要四个关键参数：

- 数据的源地址
- 数据的目标地址
- 数据传输量
- 传输模式（正常（一次）or循环）

有了这几个参数，我们只需要在两个设备之间建立一条（双向就要两条）DMA通道。当信息传输完成时就触发DMA中断回调函数，接着就执行该中断函数中的代码。这样一来，CPU的工作量就被大大减少了。

<img src="/media/image-47-1024x290.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

要建立DMA通道，在ioc文件中的DMA选项卡（属于System Core）中点击Add新增一条，CubeMX会自动帮我们填写参数并建立DMA通道。也可以在USART（属于Connectivity）配置中新增。

<img src="/media/image-50.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>新建RX和TX的DMA通道</em></p>

使用DMA传输，仍然会触发中断。因此我们只需要将原来的收发函数改写成_DMA后缀即可。

<img src="/media/image-51.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

刷板运行，和之前效果一致。