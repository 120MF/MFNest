+++
title = 'Linux系统下CAN总线的开发指南'
date = 2024-05-18T21:13:16+08:00
draft = false
description = ""
slug = "Linux系统下CAN总线的开发指南"
image = ""
categories = ["编程相关","嵌入式"]
tags = ["Linux","Socket","C/C++","CAN","多线程"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
keywords = ["Linux","Socket","C/C++","CAN","多线程"]
readingTime = true
+++

## 前言

本文根据个人开发经验撰写，简述了Linux下C/C++与CAN总线交互的一些方法。
欢迎查看我开发的Linux[电机通用驱动库All in one](https://gitee.com/dlmu-cone/rm-linux-motor-driver)方案！

## 接口初始化

```cpp
#include <sys/socket.h>
#include <sys/select.h>
#include <linux/can.h>
#include <linux/can/raw.h>
#include <net/if.h>
#include <sys/ioctl.h>
#include <cstring>
#include <unistd.h>
#include <fcntl.h>

bool MotorDrivers::CANSocket::socket_Init(){
    struct sockaddr_can addr;
    struct ifreq ifr;
    /// Configure Can Socket 配置Can口 ///
    socket_ = socket(PF_CAN, SOCK_RAW, CAN_RAW);
    strcpy(ifr.ifr_name, socketName_.c_str());
    ioctl(socket_, SIOCGIFINDEX, &ifr);
    addr.can_family = AF_CAN;
    addr.can_ifindex = ifr.ifr_ifindex;
    /// Set Can Filter 设置Can滤波器 ///
    rfilter_[0].can_id   = recvID_;
    rfilter_[0].can_mask = CAN_SFF_MASK;
    setsockopt(socket_, SOL_CAN_RAW, CAN_RAW_FILTER, &rfilter_, sizeof(rfilter_));
    if (setsockopt(socket_, SOL_SOCKET, SO_RCVBUF, &MotorDrivers::BUFFER_SIZE, sizeof(MotorDrivers::BUFFER_SIZE)) == -1) {
            perror("setsockopt error");
            return false;
        }
    fcntl(socket_, F_SETFL, O_NONBLOCK);

    /// Export Socket 导出Can接口 ///
    if(bind(socket_, (struct sockaddr *)&addr, sizeof(addr)) < 0){
        return false;
    }
    else
    {
        readThread = std::thread(&MotorDrivers::CANSocket::socket_ReadingThread, this);
        return true;
    }
}
```

我们将分块解析这些代码中关键的部分。
```cpp
    struct sockaddr_can addr;
    struct ifreq ifr;
    /// Configure Can Socket 配置Can口 ///
    socket_ = socket(PF_CAN, SOCK_RAW, CAN_RAW);
    strcpy(ifr.ifr_name, socketName_.c_str());
    ioctl(socket_, SIOCGIFINDEX, &ifr);
    addr.can_family = AF_CAN;
    addr.can_ifindex = ifr.ifr_ifindex;
```
首先，我们明确socket_是MotorDrivers::CANSocket类下的成员变量。它储存了初始化中的一切信息并在之后被调用。

我们新建了一些局部变量并将传递作为socket_的参数。这之所以可行是因为socket_本质上是一个int32型的数字，在初始化之后它本身并不需要再获取这些局部变量的引用。
```cpp
/// Set Can Filter 设置Can滤波器 ///
    rfilter_[0].can_id   = recvID_;
    rfilter_[0].can_mask = CAN_SFF_MASK;
    setsockopt(socket_, SOL_CAN_RAW, CAN_RAW_FILTER, &rfilter_, sizeof(rfilter_));
    if (setsockopt(socket_, SOL_SOCKET, SO_RCVBUF, &MotorDrivers::BUFFER_SIZE, sizeof(MotorDrivers::BUFFER_SIZE)) == -1) {
            perror("setsockopt error");
            return false;
        }
    fcntl(socket_, F_SETFL, O_NONBLOCK);
```
CAN滤波器是CAN总线使用过程中一个颇为重要的主题。我的方法是定义成员变量can_filter rfilter_[1];，并在初始化过程中将其can_id属性设置为需要接收的ID。如果有多个需要接收的ID，自然可以通过增强can_filter数组的长度来实现。

在第5行我们还设置了缓冲区的长度。这会影响到CAN总线的读取，我们会在稍后提及。

最后，我们将CAN总线的读取设置为非阻塞，避免其阻碍整个线程的运行。

## CAN总线的读取

我们先来看最简单的实现方法。

### 单线程读取
```cpp
struct can_frame frame_;
socket_SendEmpty();//发送一帧空信息使电机吐出CAN总线消息
if(read(socket_, &frame_, sizeof(struct can_frame)) < 0){
            perror("socket busy");
}
uint8_t p_int = (frame_.data[1] << 8) | frame_.data[0];
uint8_t v_int = (frame_.data[3] << 4) | (frame_.data[4] >> 4);
uint8_t t_int = (frame_.data[4] &0xF) << 8 | frame_.data[5];
//......
```
我们首先新建用来存储读取消息的数据结构frame_，接着使用read()函数来从初始化完成的socket_中读取can_frame长度的信息。若读取失败则会输出错误信息。

读取完成后，程序将通过frame_中的data来计算获取电机的相关信息。

这样的读取方式是十分简单易懂的。只要CAN总线上不主动更新消息，它就能起到很好的效果。

### 多线程读取

想象一下一个电机/电调以1000Hz的频率在你的CAN总线上发布自己的位置/速度/扭矩消息。如果你是Linux上`<sys/socket.h>`的开发者，你会怎么样保存这些信息以供读取呢？

不妨先看看STM32芯片的做法。在带有CAN外设的STM32芯片中，我们有3个CAN信箱用于储存CAN信息。所有的CAN总线上的消息都会被顺序推入这三个CAN信箱以供读取。当CAN信箱全被塞满之后，STM32将自动清空所有信箱。通过这样的机制，我们总能轻松在STM32程序中读入目前最新的CAN总线消息。

在Linux上，CAN信箱被一种名为“缓冲区”的队列（先进先出）数据结构替代了。CAN总线上所有的CAN消息也会被顺序推入缓冲区。上文提到的read()函数，本质上就是将缓冲区尾部的数据弹出并复制给我们指定的frame_变量。

因此，在CAN总线上的数据高速更新的情况下，若我们仍然使用上文的单线程读取，主函数中调用read()的速度跟不上CAN总线上消息更新的速度，就会造成缓冲区内的数据淤积。我们将无法获取CAN总线上电机发来的最新数据。

在这种情况下，我们多线程地读取CAN总线上的消息。

```cpp
std::atomic<bool> pause {false};
void MotorDrivers::CANSocket::socket_ReadingThread(){
    fd_set readfds;
    struct timeval tv;
    int retval;
    while(true){

        while(!pause.load()) {

            FD_ZERO(&readfds);
            FD_SET(socket_, &readfds);

            tv.tv_usec = 10;

            retval = select(socket_ + 1, &readfds, nullptr, nullptr, &tv);

            if(retval == -1) perror("select error");
            else if(retval) {
                read(socket_, &frame.can_frame_, sizeof(can_frame));
                CANLog();
            }
        }
    }
}
```

上面的代码是我们的读取线程函数。当线程同步变量pause为false时，我们将非阻塞地（使用select）读取CAN总线上的消息，并将值赋给成员变量frame。

值得一提的是，std::atomic变量是一种多线程读写安全的变量。它的所有操作都是“原子”的，这样在多线程竞争使用pause变量时能避免可能发生的错误。在我们的程序中，我们仅仅使用pause变量就实现了多线程的资源安全。

```cpp
void MotorDrivers::CANSocket::socket_PauseThread(){
    pause = true;
}

void MotorDrivers::CANSocket::socket_ResumeThread(){
    pause = false;
}
```

这是两个封装了pause操作的函数。每次我们要对frame进行操作时，我们都先暂停，之后再继续线程，避免可能出现的资源竞争。

```cpp
MotorDrivers::SocketFrame MotorDrivers::CANSocket::socket_Read(){
    socket_PauseThread();
    SocketFrame frame_ = frame;
    socket_ResumeThread();
    return frame_;
}
```

外部想要读取最新一帧的CAN总线消息，我们只需要将多线程中读取的frame复制并返回即可。

```cpp
//……
// in class private: std::thread readThread
//in socket_Init():
{
    readThread = std::thread(&MotorDrivers::CANSocket::socket_ReadingThread, this);
    return true;
}
```

别忘了在`socket_Init()`完成之后运行该线程。

### 获取突变值

有些时候，我们可能想监测某几位CAN消息值发生的突变。例如，当大疆M2006的CAN消息第一位从0x1F突变到0x00时，说明电机顺时针旋转了一圈。我们可以利用这种突变记录下电机的连续位置变化，进而实现更多的电机功能，例如实现位置/速度的串级PID控制，让电机以一定速度转至目标位置。

毫无疑问，若要实现这种突变值的检测，我们就需要记录下多帧的数据。使用与缓冲区类似的双段队列 std::deque可以实现这一点。

```cpp
std::deque<SocketFrame> data_buffer_;
void MotorDrivers::CANSocket::socket_ReadingThread(){
    //......
            else if(retval) {
                read(socket_, &frame.can_frame_, sizeof(can_frame));
                socket_DetectMutation();
                frame.timestamp_ = getTimeStamp();
                
                data_buffer_.push_back(frame);
                if(data_buffer_.size() > MAX_BUFFER_SIZE){
                    data_buffer_.pop_front();
                }
                CANLog();
            }
        }
    }
}
```

如你所见，我们在每次`read()`之后都会将frame复制一份并推入这个缓冲区队列，并在缓冲区队列满后自动清除老消息。

接下来是`socket_DetectMutation()`方法：

```cpp
int16_t mutation[16];
void MotorDrivers::CANSocket::socket_DetectMutation(){
    if(data_buffer_.size() < 2) return;

    SocketFrame prv_socket_frame = data_buffer_.back();

    can_frame crt_frame = frame.can_frame_;
    can_frame prv_frame = prv_socket_frame.can_frame_;
    data_buffer_.pop_back();
    can_frame prv_prv_frame = data_buffer_.back().can_frame_;
    data_buffer_.push_back(prv_socket_frame);

    for(size_t i = 0; i < 8; i++){
        
        if(crt_frame.data[i] < prv_frame.data[i] && prv_frame.data[i] >= prv_prv_frame.data[i]
        && crt_frame.data[i] - prv_frame.data[i] < -5 && !mutation[i+8] )
        {
            mutation[i] ++;
            mutation[i+8] = 1;
            logCAN.logWrite("Mutation Detected: Data[" + 
            std::to_string(i) + "] ++" + std::to_string(mutation[i]));
        }
        else if(crt_frame.data[i] > prv_frame.data[i] && prv_frame.data[i] <= prv_prv_frame.data[i]
        && crt_frame.data[i] - prv_frame.data[i] > 5 && !mutation[i+8])
        {
            mutation[i]--;
            mutation[i+8] = 1;
            logCAN.logWrite("Mutation Detected: Data[" + 
            std::to_string(i) + "] --" + std::to_string(mutation[i]));
        }
        else mutation[i+8] = 0;
    }

}
```
在这个案例中，`mutation[16]`的前八位用来存储突变的数量，后八位用来存储上一次发生了何种突变。

我们仅使用了三帧数据来判断突变的产生，这在1000Hz的总线数据量下基本是可行的。之所以不用滑动窗口之类的检测方式，是因为检测的帧越多，Read线程的执行速度就会被拖得越慢；若新开线程，也要不断地暂停Read线程来保证数据竞态安全，最后都会导致读取不到CAN总线上最新的数据，得不偿失。

之后，要在程序的位置读取突变的数量，我们只需要使用socket_ReadMutation()方法：

```cpp
int16_t MotorDrivers::CANSocket::socket_ReadMutation(uint8_t index){
    int16_t mutation_value;
    socket_PauseThread();
    mutation_value = mutation[index];
    mutation[index] = 0;
    socket_ResumeThread();
    return mutation_value;
}
```

## CAN总线的发送

相比于读取操作，CAN总线的操作并不需要考虑太多。

```cpp
bool MotorDrivers::CANSocket::socket_Send(uint8_t* frame_data){
    socket_PauseThread();
    std::copy(frame_data, frame_data + frame.can_frame_.len, frame.can_frame_.data);
    if(write(socket_, &frame.can_frame_, sizeof(can_frame)) < 0){
        perror("socket send error");
        socket_ResumeThread();
        return false;
    }
    socket_ResumeThread();
    return true;
}
```

`socket_Send()`接收一个待发送的数组，将其值复制并使用`write()`函数发送之。

## 尾声

做这篇文章主要也是因为目前中文互联网上没有相关方面讲得比较细致的文章，在开发的过程中我也因此走了很多弯路。在这里祝愿各位都能顺利开发出自己想要的功能！