+++
title = '一年的坚守与回顾 - 我与 Linux CAN 通信框架的故事'
date = 2025-11-25T11:26:53+08:00
draft = false
description = "一年的坚守与回顾 - 我与 Linux CAN 通信框架的故事"
slug = ""
image = "1.png"
categories = ["编程相关","Linux"]
tags = ["Linux", "C/C++", "CAN", "HyCAN", "驱动", "开发"]
weight = 1
keywords = ["Linux", "C/C++", "CAN", "HyCAN", "驱动", "开发", "开源"]
readingTime = true
+++

## 序

今年年底，我把我打磨很久的 CAN 通信框架开源并且发布在了各大论坛上。作为 Linux 初学者和大学生，我非常感谢网络上各位前辈对我作品的认可和指正。在这里，我想把我自己写这个框架的一些心路历程记录下来，为这段 Linux 开发的旅程做一个简单的总结。

## 动机

我和 Linux CAN 的故事说来话长。大一的夏天，工作室的学长给了一台 `Manifold 2-G`。这是 `DJI` 出品的一台嵌入式 Linux 设备，上面直接附带两个原生 `CAN` 口。当时我们计划把它作为我们 `RoboMaster 工程机器人` 的上位机，通过 CAN 直接控制机械臂电机。

![正在给妙算刷机](manifold.jpg)

这里我就不得不吐槽一下 DJI，好好一个机器发布完以后一点都不维护，到 2023 年跑的系统还是 `Ubuntu 16.04`，手动给这个机器刷机和维护 ROS2、Docker 简直是要老命了……

抛开机器本身的问题不谈，理论上来说，用 CAN 控制电机应该是一个很简单的任务。你只需要发送控制帧，获取反馈帧，把反馈数据同步给 ros2_control 就行。

![理想下的模型](graph-1.jpg)

问题是，对于 `DJI M3508` 和一些其它的类似的电机来说，它们的电调只接受控制电流数据，不接收位置/速度之类的控制指令。那么怎样做到位置/速度控制呢？电调会在 CAN 总线上以 1000Hz 的频率发送角度和速度反馈帧，我们需要根据反馈帧进行 PID 运算得到实际的控制电流，最后再发送给电调。于是整个流程就变成了这样：

![实际的情况](graph-2.jpg)

在这种情况下，如果采用直接 `read()` CAN 总线的方法，我们妙算里 Jetson Nano 的那颗 ARM Cortex-A57 肯定是撑不住的。于是，我开始研究 Linux 平台上怎样采取更高效的方式来读取总线，降低 CPU 占用率。

## 初次尝试

意识到这可能是一个很关键的中间件，我把电机控制逻辑和 CAN 驱动封装在了一起，开发了一个叫做 [Linux-Motor-Drivers](https://gitee.com/dlmu-cone/rm-linux-motor-driver) 的库。

这也是我第一次设计 Linux C++ 程序。当时对 C++ 没有太多的了解，我只是感觉：“既然要学就学最难的，啃最硬的骨头”，然后就用 AI 辅助着学会了一些面向对象的皮毛，设计了一个依赖注入的模型。

![模块示意图](linux-motor-driver.png)

当时也不是很懂 Linux 和 Socket 模型，AI 也不是很发达，我也是探索了别人的开源库才明白，不能用阻塞轮询读取，应该用 `poll` 或者 `select`。

![以前写的文档](doc.png)

### 问题

第一次写的这版“驱动”在高负载下有时会出现 SIGSEGV 的报错，同时用 C++ 的 iostream 开发的日志系统的性能也比较羸弱。更主要的是，我在 C++ 的实践上采用了大量虚函数和类继承，但是不同电机的操作逻辑往往无法耦合在一起。比如达妙电机需要额外的使能/失能操作的同时不需要像大疆电机那样反复从总线上读取消息。在这样的情况下，各个电机之间能够复用的代码往往比较少，为每一个电机添加一个父类依赖反而有些鸡肋。因此我决定对这版代码进行重构。电机控制的部分演化成了 [OneMotor](https://github.com/RoboMaster-DLMU-CONE/OneMotor)，CAN 驱动的部分就演化成了 `HyCAN`。

## HyCAN 的开发经历

### 关键的 epoll

这个时候我意识到一个问题，异步模型下 `select` 的性能仍然不够强劲。要实现总线上挂载更多个电机的目标，我必须找到更高性能的通信方法。也是在这个时候，我了解了 Linux 的 Socket 模型，认识了 `epoll`。以下是在我的使用场景下 `epoll` 的使用方法：

```cpp
// create epoll_fd
epoll_fd = epoll_create(256);

// create event struct
ev = {
    .events = EPOLLIN,
    .data = {
        .fd = sock_fd
    }
};

// add the event you want to monitor ( IN, CAN Sock FD )
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sock_fd, &ev)

// process event in loop

const int nfds = epoll_wait(epoll_fd, events, MAX_EPOLL_EVENT, -1);
if (nfds == -1)
{
    // handle error
}

if (nfds == 0)
    // no event to read
    continue;

for (int i = 0; i < nfds; ++i)
{
    // real process event
}

```

epoll 的原理是：在内核中以红黑树的数据结构来存储所有待监听的文件描述符，以就绪链表（rdllist）来缓存已经就绪的事件。当文件描述符就绪时，内核通过回调函数将其从红黑树中移到就绪链表，然后 `epoll_wait()` 函数能高效地一次性将就绪的事件列表复制到用户空间。这避免了我们之前实现的 select/poll 的轮询开销。

### 再见 sudo

在 Linux 的 Netlink Socket 模型下，我们要对 CAN 接口进行一些操作（比如使用 `ip link set can0 up`）是需要有相应的权限的。在一般的发行版中，我们默认都要使用 `sudo` 来进行相关操作。前文有提到我们需要在 `ros2` 系统里进行实际的电机控制，这意味着我们的 CAN 驱动要被 colcon 编译进 ros2 构建系统的某个二进制文件中。也就意味着，如果不使用 root 权限来执行我们的 ros2 电机控制节点，我们就无法正常运行某些 CAN 操作（比如设置比特率和开关接口等）。而这对于 ros2 这样的大型系统来说几乎是不可能的。

常见的解决方法是，我们添加一个启动脚本，在这个脚本中添加 CAN 接口相关的操作，并在 systemd 中设置启动时以特权运行该脚本。这其实是一种很无奈妥协的解决办法，且存在安全隐患。如果启动脚本出现问题导致 CAN 接口无法正常开关，则整个业务程序运行都会受到干扰且难以 Debug；想要在业务程序运行过程中进行修改波特率、重启接口等操作也都成了不可能的事情。

还有一种解决方法是通过 `setcap` 命令为二进制程序赋予操作 Netlink Socket 的特权。但是问题是在开发环境中当前编译的二进制程序会被新的编译顶替，我们又需要额外维护一套在编译后运行 setcap 命令的脚本。同时，对 ros2 这种使用 colcon 构建的系统来说，你也很难去定位到你编译的二进制程序的位置。

在上述解决方案都很难消除痛点的情况下，我决定寻找一种更加优雅的方案。

首先，我们可以注意到，拥有特权的程序来开关 CAN Netlink Socket 是没有问题的。那么我们就可以构建这样一种 `Daemon-Client` 模型：后台进程以特权模式运行，负责进行 Netlink Socket 的操作；前台客户端（也就是用户链接到 HyCAN 库编译的程序）通过某种 IPC 机制和后台进程通信，向后台进程发送操作请求；后台进程完成前台客户端的请求操作之后向客户端返回操作结果；客户端处理这个结果并继续正常执行无需特权就能运行的 `socket`、`epoll` 操作。

于是，我基于 Linux 的 socket 和 UDS 封装了一套 IPC 实现，并且创建了一个 Daemon 可执行目标。它会在后台监听并接受前台 HyCAN Client 发来的请求并执行对应的操作。

```cpp
// Accept incoming connections with timeout
auto client_socket = main_socket_->accept(1000); // 1 second timeout
if (!client_socket)
{
    // No incoming connection within timeout, continue
    continue;
}

// Receive registration request
ClientRegisterRequest register_request;
const ssize_t bytes_received = client_socket->recv(&register_request, sizeof(register_request), 1000);

if (bytes_received == sizeof(ClientRegisterRequest))
{
    handle_client_registration(register_request, std::move(client_socket));
}
else if (bytes_received != 0)
{
    std::cerr << "Received invalid request size: " << bytes_received
                << " (expected: " << sizeof(ClientRegisterRequest) << ")" << std::endl;
}
```

将所有 Netlink 操作移动到 Daemon 端之后，HyCAN 库得以在所有情况下以非 root 的权限运行且拥有完整的控制 NetlinkSocket 的能力。这也是我个人在使用和部署过程中最满意的一点。

### 一些细节

除了换性能更强劲的 epoll，和处理 root 权限，开发过程中我还遇到很多值得注意的细节，会影响程序的正常运行。

#### socket 的回显问题

早期在设计这套通信框架的时候，我的想法是对于一个 CAN 接口来说就配一个 linux socket fd，发送就直接 write 这个 fd，读取就直接 epoll 监听这个 fd；但是这样做的结果是，当我往这个被 epoll 监控的 socket 上发送消息的时候，硬件上的 CAN 总线能够正常收到这条消息，但是被 epoll 监听的 socket 却接收不到这条消息。这在使用角度上通常没有问题，但是当我们想日志记录下总线上的情况的时候就不是很理想了。

产生这样结果的原因是，SocketCAN 默认情况下不会将发送的数据“回显”给发送者自己。虽然 `CAN_RAW_LOOPBACK` 选项默认是开启的，但它只负责将数据回环到本地网络栈，分发给**其他**绑定的 socket。对于发送者 socket 本身，除非显式开启 `CAN_RAW_RECV_OWN_MSGS` 选项，否则内核不会将自己发出的帧再送回来。这样的设计是故意为之，避免了对发送者多余的上下文切换和重复处理。

在当前的实现中，每个 CAN Interface 配备了两个 Linux socket，一个用来给 epoll 监控，一个用来往总线上发送消息。这样就能保证 epoll 能够监控到所有的消息（包括自己发出的）。后续也会增加可选的禁启用 loopback 的实现。

#### 线程终止处理

早期的开发过程中，我经常遇到 epoll 的循环线程退出不干净，导致程序终止时报 SIGSEGV 的问题。

为了解决这个问题，我采用了 C++ 的 `std::jthread` 线程。该线程支持 `stop_token`，能够在析构时尝试自动终止并 join 线程。

当然，仅仅是这样还不够。当总线上没有消息时，epoll 线程会被挂起，这时主线程尝试 join 这个线程时无用的。因此，我们有必要在创建 epoll 句柄时添加一个自定义的“线程终止”事件，并让 epoll 关注这个事件。这样，在我们进行 cleanup 操作的时候，就可以手动写入这个事件。这将强行唤醒 epoll 总线，我们可以在 epoll 总线中处理这个事件并最终优雅的退出循环。

```cpp
tl::expected<void, Error> Dispatcher::stop() noexcept
{
    if (reap_thread.joinable())
    {
        reap_thread.request_stop();
        constexpr uint64_t one = 1;
        if (const ssize_t result = write(thread_event_fd, &one, sizeof(one)); result == -1)
        {
            // handle error
        };
        reap_thread.join();
    }
    return {};
}


// use stop_token ref
void Dispatcher::reap_process(const std::stop_token& stop_token)
{
    //...
    for (int i = 0; i < nfds; ++i)
    {
        if (events[i].data.fd == thread_event_fd && stop_token.stop_requested()) return;
        // normal can bus event goes on...
    }
}
```

#### 性能优化

为了实现高性能 CAN 通信框架这一目标，我在这个库的关键位置都进行了性能优化。这主要集中在 epoll 循环附近，也是性能开销最大的地方。在 Linux 的 pthread 线程模型下，我们可以在 epoll 线程开始时进行 CPU 亲和和线程策略优化，提高 epoll 循环的实时性和稳定性（注：设置实时调度策略通常需要相应的系统权限）。

```cpp

inline tl::expected<void, Error> make_real_time()
{
    sched_param param{};
    param.sched_priority = 80;
    if (pthread_setschedparam(pthread_self(), SCHED_FIFO, &param) != 0)
    {
        return unexpected(Error(ThreadRealTimeError, format("Failed to make thread real time: {}", strerror(errno))));
    }
    return {};
}

inline tl::expected<void, Error> affinize_cpu(const uint8_t cpu)
{
    cpu_set_t cpu_set;
    CPU_ZERO(&cpu_set);
    CPU_SET(cpu, &cpu_set);

    if (pthread_setaffinity_np(pthread_self(), sizeof(cpu_set), &cpu_set) != 0)
    {
        return unexpected(Error(CPUAffinityError, format("Failed to set cpu affinity: {}", strerror(errno))));
    }
    return {};
}

```

## 下一步

开发 HyCAN 的过程中，我学习了很多 Linux Socket 以及 IPC 方面的知识，我也亲自实践了怎样开发一个现代的 CMake C++ 工程。目前这个通信库已经基本完善并且也经历了实战检验。但是，我还有更进一步的想法。CAN Socket 是如此，Serial、IIC、SPI 的通信是不是也可以使用这样的框架呢？我们能不能在 Linux 上有这样一个一应俱全的通信框架呢？

看来，我们的故事还没有结束。接下来，我计划开发一个支持多种通信协议的 Linux C++ 库，采用更加优化的 IPC 通信机制，配合“黑魔法” UDS 技术，彻底解决 socket 的权限问题。

## 总结

开发这个库的日子里，我的生活大概就是 `宿舍调程序` -> `去工作室部署` -> `回宿舍优化` 这样的两点一线。我想，可能就是这样看似无聊的过程的慢慢积累才能沉淀出一个能用、好用、稳定的库吧。

在这里也感谢工作室各位老人对我的支持，没有他们默默承受住工作室的其它方面的压力，也不会有这样一个能给我们自由探索技术的环境了。

最后希望 HyCAN 和我的经历能够帮助或者启发到阅读这篇文章的你。希望这些代码能够在你的机器上顺利运行！
