+++
title = "C++ 协程：学习与探索"
date = 2026-02-22T00:20:00+08:00
draft = false
description = "基于实践与实现细节，带你理解 C++20 协程的核心概念、内存与控制流，以及在 HyIO（HyCAN 下一代）中如何用最小运行时实现 Task/Promise。"
slug = "cpp-coroutine-learning"
categories = ["编程相关","C++"]
tags = ["C++","coroutine","协程","异步","设计"]
weight = 1
keywords = ["C++协程","Task","promise_type","awaitable"]
readingTime = true
+++

## 前言

本文以“实现一个用 C++20 写的最小 Task/Promise”为主线，边实现边解释：当我们介绍概念（如 promise_type 的 get_return_object）时，会把对应的代码片段搬出来并逐行说明。目标是让有 C++/Linux 背景的读者能把抽象概念和现实代码直接对应起来。

## 契机

1. 为 HyCAN 设计 HyIO —— 一个以协程为基础的异步 I/O 层，让用户能像写同步代码一样组合异步操作。
2. 把学到的概念工程化：实现一个易扩展的 Task/Promise 基座，便于后续加入内存池、io_uring Awaitable 等。

---

## 1 设计决策：为什么 C++ 选择“Stackless”？

在深入代码之前，先理解设计决策能让后续实现更有意义。

有两类协程设计：有栈（stackful）与无栈（stackless）。

- Stackful（有栈）：每个协程有独立的调用栈，切换时保存/恢复整个栈指针；实现上类似用户态线程（green thread）。优点是实现直观，局部变量与调用链都保存在栈上；缺点是每个协程栈开销大，且在某些平台下切换代价较高。

- Stackless（无栈）：编译器把协程转换为状态机，把需要跨越挂起点的局部变量与状态保存在“协程帧（coroutine frame）”中，通常放在堆或自定义分配器管理的内存上。恢复时用一个轻量的句柄（coroutine_handle）跳回执行点。

为什么 C++ 选择 Stackless？从工程与互操作角度考虑：

1. 内存与调度可控：用户/库可以决定协程帧如何分配（堆、线程局部池、预分配区域），适配不同负载与平台。对于短生命周期的高频任务，自定义池能显著提升性能；对于长期挂起的任务，把帧放在堆上更可靠。
2. 与线程栈语义解耦：协程不等同于线程。Stackless 允许协程在单一线程中被调度，实现轻量级协作式并发，而不需要重建完整线程栈模型。
3. 优化机会更多：编译器能在严格嵌套场景下做帧消除（frame elision），将堆分配优化为栈或内联，从而获得类似 stackful 的效率；同时 promise_type 提供了插入自定义分配器的钩子。
4. 与现有运行时/内核更好集成：像 io_uring、epoll 这类基于事件的IO模型天生适合把句柄或指针（而不是整段栈）传递给内核或事件循环。

举例对比（伪代码）：

Stackful（概念）：
```
协程A栈: [frameA variables...]
协程B栈: [frameB variables...]
切换: 保存A栈指针 -> 恢复B栈指针
```

Stackless（概念）：
```
协程A帧(obj on heap): { locals that cross await points }
协程B帧(obj on heap): { ... }
切换: resume(coroutine_handle to frame B)
```

由此可以看到，Stackless 更容易与事件循环、内存池等工程化组件结合。

---

## 2 promise_type：协程帧的“大管家”

在 C++ 协程中，promise_type 决定了：如何构造返回对象、在初始/结束时是否挂起、如何存储返回值和异常等。

下面先解释 promise 在设计中的位置：编译器在生成协程帧时会在帧中放置一个 promise 对象，promise 提供了协程帧的构造与生命周期钩子（包括 operator new/delete 的重载点），因此 promise 是实现 Stackless 策略的主要切入点——它让用户控制帧的分配和资源处理。

接下来通过代码分步说明 promise 的关键接口与语义。

---

## 3 设计目标与最小接口

先说明我们要做什么：提供一个可以被 co_await 的 Task<T>，允许外部协程通过 co_await 等待其完成，且在 Task 析构时自动清理协程帧。核心契约：不可拷贝、可移动、awaitable。

下面是分步实现。在每步中我都会贴出该步骤对应的代码片段并解释。

### 3.1 Task 的骨架（句柄与所有权）

```cpp
// Task 的核心：持有 std::coroutine_handle<promise_type>
template <typename T = void>
class Task {
public:
    using promise_type = TaskPromise<T>;
    explicit Task(std::coroutine_handle<promise_type> handle) noexcept : m_handle(handle) {}
    Task(const Task&) = delete; // 单所有权
    Task(Task&& other) noexcept : m_handle(std::exchange(other.m_handle, nullptr)) {}
    ~Task(){ if(m_handle) m_handle.destroy(); }

    // await 接口（后文解释 await_suspend 的细节）
    bool await_ready() const noexcept { return false; }
    std::coroutine_handle<> await_suspend(std::coroutine_handle<> waiting_coro) noexcept { 
        m_handle.promise().continuation = waiting_coro;
        return m_handle; // 对称转移：立即切换执行 Task 对应的协程
    }
    T await_resume();
private:
    std::coroutine_handle<promise_type> m_handle;
};
```

解释：Task 只是一个持有 coroutine handle 的轻量对象。它不可拷贝以避免 double-destroy，支持移动以传递所有权。await_suspend 将调用者协程保存为 promise.continuation，然后返回 m_handle 表示把控制权对称地转给被等待的协程（Symmetric Transfer）。

---

## 4 promise_type：实现与例子

在下文的控制流示意图中，展示了调用者协程 co_await Task 时的主要步骤（先保存 continuation、再对称转移到被等待协程、被等待协程执行并在 final_suspend 中 resume continuation）。图示：

![协程控制流](./diagram-controlflow.svg)


### await_suspend 的返回策略（示意图）

下图给出 await_suspend 不同返回值对控制流的影响（简化示意）：

![await_suspend 返回策略](./diagram-await-strategies.svg)




在 C++ 协程中，promise_type 决定了：如何构造返回对象、在初始/结束时是否挂起、如何存储返回值和异常等。

下面是我们基类的一部分实现（简化）以及逐行说明：

```cpp
struct TaskPromiseBase {
    std::coroutine_handle<> continuation = nullptr; // 保存等待者的句柄（如果存在）

    std::suspend_always initial_suspend() noexcept { return {}; } // 创建后立即挂起，交由外部调度

    struct FinalAwaiter {
        bool await_ready() noexcept { return false; }
        std::coroutine_handle<> await_suspend(std::coroutine_handle<> h) noexcept {
            auto base_handle = std::coroutine_handle<TaskPromiseBase>::from_address(h.address());
            const auto& promise = base_handle.promise();
            if (promise.continuation) return promise.continuation; // 唤醒等待者
            return std::noop_coroutine(); // 没有人等待则返回 noop
        }
        void await_resume() noexcept {}
    };

    FinalAwaiter final_suspend() noexcept { return {}; }

    void unhandled_exception() { m_exception = std::current_exception(); }
    std::exception_ptr m_exception;
};
```

解释要点：
- initial_suspend 返回 std::suspend_always 表示协程创建后不立即运行，而是由外部决定何时 resume（适用于事件/调度驱动场景）。
- final_suspend 返回一个 FinalAwaiter，它会尝试把之前保存在 promise.continuation 的等待者唤醒（如果存在）。这就是 Task/await 的闭环。
- unhandled_exception 把异常保存为 std::exception_ptr，以便在 await_resume 中跨线程/协程重新抛出。

下面展示如何把返回值存入 promise：

```cpp
template <typename T>
struct TaskPromise : TaskPromiseBase {
    std::variant<std::monostate, T, std::exception_ptr> m_result;
    Task<T> get_return_object() { return Task<T>{std::coroutine_handle<TaskPromise<T>>::from_promise(*this)}; }
    void return_value(T value) { m_result.template emplace<1>(std::move(value)); }
    void unhandled_exception(){ m_result.template emplace<2>(std::current_exception()); }
    T result(){ if(m_result.index()==2) std::rethrow_exception(std::get<2>(m_result)); return std::move(std::get<1>(m_result)); }
};
```

说明：get_return_object 使用 from_promise 构造一个对应的 coroutine_handle，这正是编译器期望的流程：先在协程帧（包含 promise）上分配内存，再用 promise 返回外部句柄（Task）。这也回答了“为什么编译器先创建 promise”的问题：promise 居于协程帧中，控制帧的分配与生命周期。

---

## 3 Stackless 与内存（加示例）

下图对比了有栈（Stackful）与无栈（Stackless）的概念：有栈需交换完整栈指针，而无栈把跨挂起点的变量保存在堆上的协程帧中，通过句柄 resume 实现切换。图示：

![Stackless vs Stackful](./diagram-stackless.svg)


“Stackless” 是很多人困惑的地方。用一个小示例和注释说明：

示例：promise 中定制分配器使协程帧由自定义池分配

```cpp
struct MyPromise {
    void* operator new(size_t sz) {
        // 从线程本地/无锁池申请内存
        return my_pool_alloc(sz);
    }
    void operator delete(void* p, size_t) {
        my_pool_free(p, sz);
    }
    // 其它 promise 接口...
};
```

解释：上面展示了如何通过在 promise_type 中重载 operator new/operator delete 来控制协程帧的分配器——这正是 Stackless 的体现：编译器把协程帧作为一个可自定义分配的对象，而不是强制分配在调用栈上。

再举一个“类型擦除”的小片段：

```cpp
std::coroutine_handle<MyPromise> typed = /* ... */;
std::coroutine_handle<> erased = typed; // 隐式转换：丢弃具体 promise 类型信息
```

说明：类型擦除允许运行时只通过无类型句柄 resume 已分配的帧，而无需在恢复时知道具体的 promise 类型——因为底层恢复只关心程序计数器/寄存器等。

---

## 4 await_suspend 的返回策略（配以示例）

- 返回 void：标准的挂起（控制回到调用者/事件循环），例如：

```cpp
void await_suspend(std::coroutine_handle<> h) {
    queue_push(h); // 把句柄交给事件循环
}
```

- 返回 bool：决定是否挂起

```cpp
bool await_suspend(std::coroutine_handle<> h) {
    if (fast_path_available()) return false; // 不挂起，继续执行
    queue_push(h); return true; // 挂起并由事件驱动恢复
}
```

- 返回 std::coroutine_handle<>（对称转移）：立即切换到另一个协程执行

```cpp
std::coroutine_handle<> await_suspend(std::coroutine_handle<> h) {
    return other_coroutine_handle; // 马上切换执行 other_coroutine
}
```

说明：这些返回值决定了协程在 co_await 时的控制权流向，掌握它们能写出更高效或更确定性的调度逻辑。

---

## 5 性能建议（工程化）

- 对高频短生命周期协程使用自定义分配器（如线程本地对象池），通过在 promise_type 中重载 operator new/delete 实现。示例已在第3节给出。
- 尽量避免在 await_suspend 中做耗时阻塞工作。
- 使用 std::variant / union 来复用返回值与异常的内存，减少额外分配。

---

## 6 完整实现与附录

文中为了便于教学把实现拆分为多个片段。完整实现（可直接编译的 header）放在本节附录以便参考：

```cpp
// （此处可放回最开始的完整 HYIO_TASK_HPP 实现，便于读者复制）
```

---

## 小结

本文从工程设计决策出发（为何选择 Stackless），解析了 promise_type 在协程帧分配与生命周期中的角色，然后以分步实现的方式把 Task/Promise 的关键接口与运行时行为逐一对应：get_return_object、initial_suspend / final_suspend、await_suspend 的返回策略、以及异常/返回值的跨协程传播。最后给出了一些工程化建议（自定义分配器、避免阻塞的 await_suspend、异常传播策略等），供在 HyIO/HyCAN 的工程实现中采纳。

---
