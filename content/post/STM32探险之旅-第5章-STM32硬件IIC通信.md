+++
title = 'STM32探险之旅 第5章 STM32硬件IIC通信'
date = 2023-10-17T19:40:45+08:00
draft = false
description = ""
slug = "STM32探险之旅-第5章-STM32硬件IIC通信"
image = "/media/image-54-1024x424.png"
categories = ["编程相关","嵌入式"]
tags = ["C/C++","STM32","IIC","I2C","HAL"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
keywords = ["C/C++","STM32","IIC","I2C","HAL"]
readingTime = true
+++

## 前言

家人们，我现在真的有点相似了。

你说得对，但是IIC是飞利浦自主研发的一款通信协议，目前被大量工控硬件广泛采用。

正因为其应用之广泛，我们板子上的那块IIC寄存器，并不包含在Keysking教程之内。我们必须自己通过板子自带的教程研究出来寄存器的使用方法。

然后一研究就用了两天。

板子自带的教程是用软件控制GPIO口的电平变化，从而模拟了IIC通信的实现。据说这是因为ST公司的硬件IIC实现有缺陷。但是据Keysking所述，此缺陷伴随着软件更新已经被修复得差强人意。而且有还不用不是很蠢吗？所以我最后还是决定用硬件实现IIC通信。以下就是我们这次的“探险”过程。

## IIC通信的原理

说到通信，我们肯定会想到之前学到的串口通信。串口通信的原理是这样的：

<img src="/media/image-52-1024x345.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>串口通信</em></p>

数据来往走两条车道，可以同时双向进行。我们将其称为“全双工”通信。

IIC通信的原理与串口不同，它只有一条信道可以传输数据。正因如此它采用了主-从模式，即由主机发送信号，收到从机的应答后再继续进行下一步操作。我们称其为“半双工”通信。

容易发现，在上述的应答过程中，可以存在多个从机和主机的通信，也就是说一个IIC通道可以连接很多个IIC硬件（想想USB）。我们把这种支持多设备连接的协议称为总线协议。

<img src="/media/image-54-1024x424.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>IIC通信</em></p>

IIC另一个与串口不同的点在于，串口是异步通信，而IIC是同步通信。

我们都知道，在数据传输的过程中，存在“发送”和“读取”两个指令。对于异步通信来说，数据发送读取操作时机由两设备各自的时钟决定；而同步通信的发送读取操作时机依靠SCL通道上发出的时钟脉冲确定。

这又是怎么做到的呢？

简单来说，SCL会按照一定频率发送高低电平信号。接着主从机按照下面的顺序执行通信：

SCL低电平：主机修改SDA的值（0或1）；

SCL高电平：从机读取SDA的值。

如此循环就可以完成一轮通信。

还有些比较复杂的原理，可以参考下面的视频。

{{< bilibili BV1QN411D7ak>}}

## IIC通信的实现

在我们的开发板上，有一块EEPROM。它是一种早期开发的特殊的闪存，目前多被用于烧录主板的BIOS。查看开发手册可以看到其与STM32的连接方式为IIC通信。

<img src="/media/image-55-1024x405.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

因为PB6、7引脚也是GPIO引脚，我们可以使用软件模拟的方式来实现IIC通信。不过HAL库函数已经为我们准备好了IIC通信的相关代码，我们这里就采用HAL库来直接实现IIC通信吧。

首先新建工程，并且在ioc文件中进行相应的调整。

<img src="/media/image-56.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>打开IIC通信</em></p>

<img src="/media/image-57-1024x291.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>勾选上“为每个驱动生成相应的.c/.h文件对”</em></p>

由于我们要通过串口得知数据是否写入成功，因此我们也打开USART1。

保存生成代码之后可以发现，在Src和Inc文件夹中多出了各个设备对应的驱动文件。

我们也来为我们的EEPROM新建驱动文件，就叫24c02.h/24c02.c吧。

下面是我们创建一个驱动链的过程：

<img src="/media/image-59.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>在24c02.c中包含24c02.h</em></p>

<img src="/media/image-60.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>在24c02.h中包含i2c.h</em></p>

<img src="/media/image-61.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>在main.c中包含24c02.h</em></p>

要想在这块EEPROM上实现IIC通信，我们首先要了解其对应的标准通信方法。查阅资料如下：
<img src="/media/image-62.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

简单来说，主设备每次发送信息之后都需要得到从机的ACK信号，也就是说，我们得不断先发送、再接收、发送、接收......

这也太麻烦了吧？？

你说得对，所以HAL库帮我们简化了这个流程。不难发现，在上述的过程中，从机的所有回复都是ACK，就连终止信号也是主机发出。从机在这个过程中处于一个“啊对对对”的状态，我们其实并不必费力对其回复做出判断，只需要专注需要发送的内容即可。使用这个HAL库函数：

<img src="/media/image-64.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

第一个参数是iic接口定义的地址；第二个参数是写从机的地址；第三个参数是要发送的数据的地址；第四个参数是写的发送数据的长度；第五个参数是超时时间。

怎么样，和串口通信的Transmit函数是不是很像？

就这张图的而言，运行这个函数会自动帮我们处理两次从机的ACK应答，并且自动完成传输的操作。实际上，这个函数还有返回值，类型为HAL_StatusTypeDef。简单说，如果返回值是其类型之一的"HAL_OK"，就代表本次操作顺利完成。如果是"HAL_ERROR"或者"HAL_BUSY"，就说明本次操作失败，我们可以用switch-case对其进行对应的错误处理。

接下来，我们不妨更进一步完善我们EEPROM的传入函数。

<img src="/media/image-67.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<img src="/media/image-74.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>为了方便调试引入返回值</em></p>

这样一来我们就一次性传输了所写入的地址和写入的数据。

在主函数中，我们可以用for循环来对EEPROM写入对应的数据：

<img src="/media/image-68.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<img src="/media/image-69.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

不过我们现在只是“如写”——只通过调用HAL库向一个地址写入了数据，我们并不知道该数据有没有被放到我们想要的地方。那么接下来，我们不妨用HAL的IIC读取函数将刚才写入的数据读取到另一个数组中吧。

首先我们来看看这块EEPROM的读取通信。

<img src="/media/image-71.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

HAL库中的IIC读取函数如下：

<img src="/media/image-70.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>这和Transmit函数类似</em></p>

于是，我们可以分析出应该先进行I2CTransmit函数，向设备写入读取的内存地址，然后进行I2CReceive函数读取变量。构建的函数如下：

<img src="/media/image-73.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">
<p style="text-align: center"><em>为了方便调试引入返回值</em></p>

回到main.c中，利用这两个函数进行一系列操作。
```c
//start program,waiting for huart connection.
HAL_GPIO_WritePin(GreenLight_GPIO_Port, GreenLight_Pin, GPIO_PIN_RESET);
HAL_Delay(2500);
//printing the origin of "Receive"
HAL_UART_Transmit(&huart1, Receive, strlen(Receive), HAL_MAX_DELAY);
//writing date to EEPROM
HAL_GPIO_WritePin(GreenLight_GPIO_Port, GreenLight_Pin, GPIO_PIN_SET);
HAL_GPIO_WritePin(RedLight_GPIO_Port, RedLight_Pin, GPIO_PIN_RESET);
for(uint8_t i = 0; i < sizeof(Transmit); i++){
    if(epr_Transmit_Byte(Transmit[i], i) == HAL_OK){
        HAL_UART_Transmit(&huart1, "OK1\n", 4, HAL_MAX_DELAY);
    }
    else HAL_UART_Transmit(&huart1, "Fail1\n", 6, HAL_MAX_DELAY);
    HAL_Delay(300);
}
//reading EEPROM data to "Receive"
for(uint8_t i = 0; i < sizeof(Transmit); i++){
    if(epr_Receive_Byte(&Receive[i], i) == HAL_OK){
        HAL_UART_Transmit(&huart1, "OK2\n", 4, HAL_MAX_DELAY);
    }
    else HAL_UART_Transmit(&huart1, "Fail2\n", 6, HAL_MAX_DELAY);
    HAL_Delay(300);
}
//printing changed "Receive"
HAL_GPIO_WritePin(RedLight_GPIO_Port, RedLight_Pin, GPIO_PIN_SET);
HAL_UART_Transmit(&huart1, Receive, strlen(Receive), HAL_MAX_DELAY);
HAL_Delay(2000);
```

刷板运行看看效果！

{{< video src="/media/16c187126ae81f3b6fa962eb7bbc383b.mp4" style="display: block; margin-left: auto; margin-right: auto;">}}

完美实现了预期目标！