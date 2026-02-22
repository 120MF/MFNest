+++
title = 'Linux io_uring 开发与实践'
date = 2026-02-22T01:54:00+08:00
draft = false
description = "从 epoll 的性能瓶颈出发，深入讲解 io_uring 的工作原理，并实现一个完整的 Context 类来管理异步 I/O 操作。"
slug = ""
image = ""
categories = ["Linux", "编程相关"]
tags = ["io_uring", "异步IO", "liburing", "C++", "高性能"]
weight = 1
keywords = ["io_uring", "epoll", "异步IO", "内核编程", "liburing"]
readingTime = true
+++

## 引入：为什么需要 io_uring

### 传统 socket 读取方法对比

在 Linux 网络编程中，处理多个连接的 socket 读取主要有三种方式：

1. **阻塞 I/O**：简单直观，但线程阻塞等待数据，吞吐量低，无法应对高并发。
2. **select/poll**：可监听多个文件描述符，但受限于 1024 个 fd 上限（select），且每次调用需传递整个 fd 集合，性能随 fd 数量线性下降。
3. **epoll**：利用红黑树存储监听的 fd，通过内核通知（callback）向用户程序注入事件。相比 select/poll，epoll 避免了重复遍历，事件通知效率高，成为现代高性能服务器的标准选择。

尽管 epoll 已被广泛应用，但它仍有性能瓶颈。

### epoll 的性能瓶颈

epoll 的工作模式是 **事件驱动**：内核检测到事件后调用回调函数通知用户程序，用户程序再发起系统调用执行 I/O。这个过程存在以下问题：

1. **系统调用开销**：每次 I/O 都需要从用户态切换到内核态（syscall），然后返回用户态。高并发场景下，这种上下文切换带来的开销累积显著。
2. **内存拷贝开销**：数据从内核缓冲区拷贝到用户空间需要额外开销。
3. **缓冲区管理复杂**：应用需要预先分配缓冲区，或在运行时动态管理缓冲区，增加复杂性和内存碎片。
4. **原子性问题**：事件通知和 I/O 操作不是原子的，在高并发下可能丢失事件或重复处理。

### io_uring 解决的痛点

io_uring（I/O ring）是 Linux 5.1+ 引入的新异步 I/O 接口，通过以下设计彻底改进了上述问题：

1. **无锁环形队列**：
   - **Submission Queue (SQ)**：用户程序将 I/O 请求写入环形队列，无需 syscall 即可提交。
   - **Completion Queue (CQ)**：内核将完成的事件写入另一个环形队列，用户程序轮询读取结果。
2. **Batch processing**：用户可在多个请求后一次性调用 `submit_and_wait()`，减少系统调用次数。

3. **内核态驻留**（可选）：通过 `IORING_SETUP_SQPOLL` 标志，可让专用内核线程持续监听 SQ，用户程序甚至可以实现无阻塞的异步 I/O。

4. **灵活的通知机制**：支持事件驱动（eventfd）、轮询或混合模式，适应不同场景。

5. **零拷贝支持**：内核可直接访问用户预先注册的内存缓冲区，减少拷贝。

综合来看，io_uring 通过减少系统调用、批量提交和完整的异步设计，在高并发场景中性能远超 epoll。

---

## 代码实现：IoUringContext 设计

接下来我们实现一个完整的 `IoUringContext` 类，为用户提供简洁的阻塞式事件循环和灵活的异步 I/O 提交接口。

### 基础设施：IoOperation 基类

首先定义一个虚基类 `IoOperation`，所有 I/O 操作都派生自它。当 I/O 完成时，内核会回调 `complete()` 方法，用户可在派生类中实现具体的完成逻辑。

```cpp
// IoOperation.hpp
namespace hyio::core {

struct IoOperation {
    /// @brief 当 I/O 操作完成时由事件循环调用
    /// @param res 操作结果（正值为字节数，负值为错误码）
    /// @param flag CQE 标志位，用于调试或特殊处理
    virtual void complete(int res, uint32_t flag) noexcept = 0;

    virtual ~IoOperation();
};

}
```

虚基类的设计使得用户可以灵活地派生出不同的 I/O 操作类，在 `complete()` 中实现各自的处理逻辑（如更新状态、触发回调等）。

### IoUringContext 类定义

```cpp
// IoUringContext.hpp
#include "IoOperation.hpp"
#include <liburing.h>

namespace hyio {

class IoUringContext {
public:
    /// @brief 初始化 io_uring，预分配指定数量的环形队列条目
    /// @param entries SQ/CQ 的条目数，默认 1024
    explicit IoUringContext(unsigned entries = 1024);

    ~IoUringContext();

    // 禁用拷贝和移动语义，确保单一所有权
    IoUringContext(const IoUringContext&) = delete;
    IoUringContext& operator=(const IoUringContext&) = delete;
    IoUringContext(IoUringContext&&) = delete;
    IoUringContext& operator=(IoUringContext&&) = delete;

    /// @brief 启动阻塞式事件循环，处理 I/O 完成事件直至 stop() 被调用
    void run();

    /// @brief 通过 eventfd 信号优雅停止事件循环
    void stop();

    // 异步 I/O 提交接口
    void submit_read(int fd, void* buf, unsigned len, void* user_data);
    void submit_write(int fd, const void* buf, unsigned len, void* user_data);
    void submit_sendmsg(int fd, const msghdr* msg, void* user_data);
    void submit_recvmsg(int fd, msghdr* msg, void* user_data);

private:
    /// @brief 从 SQ 获取一个空闲 SQE，SQ 满时自动 flush
    io_uring_sqe* get_sqe();

    /// @brief 注册 eventfd 用于接收 stop 信号
    void add_eventfd_poll();

    io_uring m_ring{};           ///< io_uring 结构体
    int m_event_fd;              ///< eventfd 文件描述符
    bool m_running = false;       ///< 事件循环运行标志
};

}
```

### 实现细节

#### 初始化

```cpp
// IoUringContext.cpp
namespace hyio {

IoUringContext::IoUringContext(unsigned entries) {
    // 初始化 io_uring 环形队列
    // IORING_SETUP_SQPOLL：启用 SQ polling 模式（可选，需内核支持）
    // 该模式下，专用内核线程持续监听 SQ，用户无需频繁 syscall
    int ret = io_uring_queue_init(entries, &m_ring, IORING_SETUP_SQPOLL);
    if (ret < 0) {
        throw std::system_error(-ret, std::system_category(),
                                "io_uring_queue_init failed");
    }

    // 创建 eventfd 用于优雅 stop
    // EFD_NONBLOCK：非阻塞读，防止阻塞事件循环
    // EFD_CLOEXEC：exec() 时关闭，避免 fd 泄漏
    m_event_fd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    if (m_event_fd < 0) {
        io_uring_queue_exit(&m_ring);
        throw std::system_error(errno, std::system_category(),
                                "eventfd creation failed");
    }
}

IoUringContext::~IoUringContext() {
    if (m_running) {
        stop();
    }
    ::close(m_event_fd);
    io_uring_queue_exit(&m_ring);
}

}
```

**关键点：**

- `io_uring_queue_init()` 初始化环形队列，传入 `IORING_SETUP_SQPOLL` 标志启用 SQ polling 模式。
- `eventfd()` 用于线程间通信，stop() 会向其写入信号，唤醒 run() 中的等待。

#### 事件循环核心

```cpp
void IoUringContext::run() {
    m_running = true;
    add_eventfd_poll();  // 提前注册 eventfd poll，以便接收 stop 信号

    while (m_running) {
        // submit_and_wait：
        // 1. 将 SQ 中的请求 flush 到内核
        // 2. 阻塞等待至少 1 个 CQE 完成
        // 若没有待处理 CQE，则阻塞；若已有 CQE，则立即返回
        int ret = io_uring_submit_and_wait(&m_ring, 1);
        if (ret < 0 && ret != -EINTR) {
            std::cerr << "io_uring_submit_and_wait error: " << -ret << "\n";
            break;
        }

        // 批量处理所有完成的 CQE
        io_uring_cqe* cqe;
        unsigned head;
        unsigned count = 0;

        io_uring_for_each_cqe(&m_ring, head, cqe) {
            ++count;
            void* user_data = io_uring_cqe_get_data(cqe);

            if (user_data == nullptr) {
                // user_data == nullptr 表示这是 eventfd poll 事件
                if (!m_running) {
                    break;  // stop 信号已接收，退出循环
                }
                // 不是 stop，说是正常的 eventfd 可读事件，清空并重新注册
                uint64_t val;
                ::read(m_event_fd, &val, sizeof(val));
                add_eventfd_poll();
            } else {
                // 这是用户提交的 I/O 操作完成事件
                auto* op = reinterpret_cast<core::IoOperation*>(user_data);
                // 调用 complete()，用户在派生类中实现具体逻辑
                op->complete(cqe->res, cqe->flags);
            }
        }

        // 向内核通知已处理的 CQE 个数，推进 CQ 的尾指针
        io_uring_cq_advance(&m_ring, count);
    }
}

void IoUringContext::stop() {
    m_running = false;
    // 向 eventfd 写入非零值以触发可读事件，唤醒阻塞的 run()
    constexpr uint64_t val = 1;
    ::write(m_event_fd, &val, sizeof(val));
}
```

**关键点：**

- `io_uring_submit_and_wait()`：原子地提交 SQ 中的请求并阻塞等待至少 1 个完成事件。这样可以批量提交多个请求，减少系统调用。
- `io_uring_for_each_cqe()`：安全遍历 CQ 中的所有完成事件，避免手工管理索引。
- `user_data == nullptr` 约定：用于内部控制事件（如 eventfd），区别于用户 I/O 操作。

#### 优雅的停止机制

```cpp
void IoUringContext::add_eventfd_poll() {
    io_uring_sqe* sqe = get_sqe();

    // 准备一个 poll_add 请求，监听 eventfd 上的可读事件（POLLIN）
    io_uring_prep_poll_add(sqe, m_event_fd, POLLIN);

    // 设置 user_data 为 nullptr，作为特殊标记
    // 当 eventfd 有数据可读时，完成的 CQE 会包含这个 nullptr user_data
    io_uring_sqe_set_data(sqe, nullptr);
}
```

这种设计避免了直接检查 `m_running` 标志带来的竞态条件。eventfd poll 完成时会自动唤醒 `submit_and_wait()`，确保 stop 请求被及时响应。

#### I/O 提交接口

```cpp
void IoUringContext::submit_read(int fd, void* buf, unsigned len, void* user_data) {
    io_uring_sqe* sqe = get_sqe();
    io_uring_prep_read(sqe, fd, buf, len, 0);  // offset = 0
    io_uring_sqe_set_data(sqe, user_data);
}

void IoUringContext::submit_write(int fd, const void* buf, unsigned len, void* user_data) {
    io_uring_sqe* sqe = get_sqe();
    io_uring_prep_write(sqe, fd, buf, len, 0);
    io_uring_sqe_set_data(sqe, user_data);
}

void IoUringContext::submit_sendmsg(int fd, const msghdr* msg, void* user_data) {
    io_uring_sqe* sqe = get_sqe();
    io_uring_prep_sendmsg(sqe, fd, msg, 0);
    io_uring_sqe_set_data(sqe, user_data);
}

void IoUringContext::submit_recvmsg(int fd, msghdr* msg, void* user_data) {
    io_uring_sqe* sqe = get_sqe();
    io_uring_prep_recvmsg(sqe, fd, msg, 0);
    io_uring_sqe_set_data(sqe, user_data);
}

io_uring_sqe* IoUringContext::get_sqe() {
    io_uring_sqe* sqe = io_uring_get_sqe(&m_ring);

    if (!sqe) {
        // SQ 已满，需要 flush 当前请求到内核
        io_uring_submit(&m_ring);

        // flush 后重新尝试获取
        sqe = io_uring_get_sqe(&m_ring);
        if (!sqe) {
            // 仍然获取失败（极少见），说明 SQ 规模不足
            throw std::runtime_error("io_uring Submission Queue is fully exhausted!");
        }
    }

    return sqe;
}
```

**提交的关键流程：**

1. 调用 `get_sqe()` 获取一个空闲 SQE（如果 SQ 满，自动 flush）。
2. 使用 `io_uring_prep_*()` 宏填充 SQE，设置 fd、缓冲区和操作类型。
3. 用 `io_uring_sqe_set_data()` 关联 user_data（通常是 `IoOperation*`）。
4. 请求不会立即执行，而是在 `submit_and_wait()` 时 flush 到内核。

---

## 代码讲解与设计思想

### 工作原理：环形队列的 Batching

io_uring 相比 epoll 的核心优势在于 **batching**：

```
用户程序：
  loop:
    submit_read(fd1, buf1, user_op1)
    submit_write(fd2, buf2, user_op2)
    submit_read(fd3, buf3, user_op3)
    ... (多个提交，不触发 syscall)
    io_uring_submit_and_wait()  ← 单次 syscall，批量进入内核

内核：
  处理所有 SQE，并行执行 I/O
  完成时写入 CQE

用户程序：
  io_uring_for_each_cqe():
    处理 CQE1, CQE2, CQE3, ...
```

在高并发场景（数千个并发连接）下，batching 可将系统调用次数从 "每个 I/O 一次" 减少到 "每次事件循环一次"，性能提升 10-100 倍。

### eventfd 优雅停止设计

```cpp
stop() 流程：
  1. 设置 m_running = false
  2. 向 eventfd 写入信号
  ↓
run() 循环中的 submit_and_wait() 被唤醒
  ↓
  CQE 包含 eventfd poll 完成事件（user_data == nullptr）
  ↓
  若 m_running == false，退出循环
```

这种设计的优点：

- **非竞态的**：即使 stop() 在 submit_and_wait() 阻塞时调用，eventfd poll CQE 也会唯一清晰地通知循环。
- **不使用信号**：避免 POSIX 信号的复杂性和不可靠性。
- **响应快速**：eventfd 的触发是内核原生支持，无延迟。

### IoOperation 虚基类的灵活性

用户可派生 `IoOperation`，实现自定义的完成处理：

```cpp
// 示例 1：Socket 读操作
struct SocketReadOp : public core::IoOperation {
    Socket* socket_ptr;
    std::vector<uint8_t> buffer;

    void complete(int res, uint32_t flag) noexcept override {
        if (res < 0) {
            std::cerr << "Read failed: " << -res << "\n";
            return;
        }
        // 处理接收到的数据
        buffer.resize(res);
        socket_ptr->on_data_received(buffer);
    }
};

// 示例 2：文件写操作
struct FileWriteOp : public core::IoOperation {
    std::promise<void> write_done;

    void complete(int res, uint32_t flag) noexcept override {
        if (res < 0) {
            write_done.set_exception(
                std::make_exception_ptr(std::system_error(-res, std::system_category()))
            );
        } else {
            write_done.set_value();
        }
    }
};

// 使用方式
auto ctx = IoUringContext(4096);

auto* sock_op = new SocketReadOp{socket_ptr, buffer};
ctx.submit_read(sock_fd, sock_op->buffer.data(), 4096, sock_op);

// 在 ctx.run() 的某个循环中，完成事件触发：
// sock_op->complete(字节数, flags) 自动调用 → on_data_received() 执行
```

这种设计提供了：

1. **类型安全**：通过虚函数避免 void 指针的不安全转型。
2. **灵活性**：每个操作可自定义完成逻辑（回调、Promise、状态机等）。
3. **可扩展性**：新增 I/O 类型（如 `sendfile`, `splice` 等）只需派生新类。

---

## 总结

io_uring 相比传统 epoll 在以下方面提供了显著改进：

1. **减少系统调用开销**：通过环形队列的 batching，多个 I/O 请求可在一次 syscall 中提交，大幅降低上下文切换成本。

2. **灵活的完成通知**：支持阻塞式 `submit_and_wait()`、轮询或结合 eventfd 的事件驱动模式，适应各种应用场景。

3. **内核态驻留**（可选）：SQPOLL 模式下，专用内核线程持续监听 SQ，用户态程序可实现完全无阻塞的异步 I/O。

4. **设计简洁**：我们实现的 `IoUringContext` 提供了清晰的事件循环和虚基类接口，用户可灵活派生实现各种 I/O 操作，代码复用性高。

5. **零拷贝潜力**：io_uring 支持内存注册和 `splice`/`sendfile` 等零拷贝操作，后续可进一步优化。

在生产环境中，io_uring 已被 nginx、Redis 等高性能软件采用，是 Linux 网络编程的未来方向。

---

**延伸阅读：**

- [liburing GitHub](https://github.com/axboe/liburing)
- [io_uring 内核文档](https://kernel.org/doc/html/latest/userspace-api/io_uring/index.html)
- Jens Axboe 关于 io_uring 的演讲与白皮书
