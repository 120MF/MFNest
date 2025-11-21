+++
title = '自制Shell 00 - REPL与子进程'
date = 2025-11-15T21:00:50+08:00
draft = false
description = ""
slug = ""
image = ""
categories = ["编程相关","Linux"]
tags = ["Linux", "Shell", "进程"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
keywords = []
readingTime = true
+++

## REPL 循环

Shell 的运行逻辑是典型的 REPL（`Read-Evaluate-Print`）。我们首先在程序中构建这个循环。

### IO

```cpp
void loop() {
  std::string line;
  while (true) {
    std::print("> ");
    if (!std::getline(std::cin, line)) {
      break;
    }
    std::println("{}", line);
  }
}
```

采用`std::print`和`std::getline`进行控制台输出/输入

### 解析用户指令

使用 `std::stringstream` 将用户输入按空格分离。

```cpp
std::vector<std::string> parse_line(std::string str) {
  std::vector<std::string> res;
  std::stringstream s(str);
  std::string word;
  while (s >> word) {
    res.push_back(word);
  }
  return std::move(res);
}
```

接下来，要实现执行用户指定的操作，我们需要在终端中**新建进程**，在进程中**执行文件**，在终端进程中**等待进程结束**。

## 关键实现

### fork()

man 文档：

```plaintext

fork() creates a new process by duplicating the calling process.  The new process is referred to as the child process.  The calling process is referred to as the parent process.

The  child  process and the parent process run in separate memory spaces.  At the time of fork() both memory spaces have the same content.  Memory writes, file mappings (mmap(2)), and unmappings (munmap(2)) performed by one of the processes do not affect the other.

...

RETURN VALUE
       On  success,  the PID of the child process is returned in the parent, and 0 is returned in the child.  On failure, -1 is returned in the parent, no child process is created, and errno
       is set to indicate the error.

ERRORS
       EAGAIN A system-imposed limit on the number of threads was encountered.

```

fork 出的子进程是当前进程的拷贝。要使用这个 fork 出的进程，只需要在当前代码的位置中继续往下执行就可以。

```c
// man 2 fork example
#include <signal.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

int main(void)
{
    pid_t pid;

    if (signal(SIGCHLD, SIG_IGN) == SIG_ERR) {
        perror("signal");
        exit(EXIT_FAILURE);
    }
    pid = fork(); // fork process begins here
    /*
    * 'the PID of the child process is returned in the parent, and 0 is returned in the child.'
    */
    switch (pid) {
    case -1:
        perror("fork");
        exit(EXIT_FAILURE);
    case 0:
    // Child process
        puts("Child exiting.");
        exit(EXIT_SUCCESS);
    default:
    // Parent process
        printf("Child is PID %jd\n", (intmax_t) pid);
        puts("Parent exiting.");
        exit(EXIT_SUCCESS);
    }
}
```

### execvp()

```plaintext

NAME
       execl, execlp, execle, execv, execvp, execvpe - execute a file

       int execvp(const char *file, char *const argv[]);

       v - execv(), execvp(), execvpe()
       The char *const argv[] argument is an array of pointers to null-terminated strings that represent the argument list available to the new program.  The first argument,  by  convention,
       should point to the filename associated with the file being executed.  The array of pointers must be terminated by a null pointer.
```

我们可以在子进程中将用户输入的分割传入该函数执行。

```cpp
      auto &path = s.front();
      char *argv[s.size()];
      for (int i = 0; i < s.size() - 1; i++) {
        argv[i] = s[i + 1].data();
      }
      // 注意，因为execvp没有argc限制参数数量，因此需要将数组最后一位置为nullptr。
      argv[s.size()] = nullptr;
      auto ret = execvp(path.c_str(), argv);
      if (ret == -1) {
        perror("execvp");
      }
      std::exit(EXIT_SUCCESS);
```

### waitpid()

```plaintext
NAME
       wait, waitpid, waitid - wait for process to change state

SYNOPSIS
       #include <sys/wait.h>

       pid_t waitpid(pid_t pid, int *_Nullable wstatus, int options);

DESCRIPTION
       All of these system calls are used to wait for state changes in a child of the calling process, and obtain information about the child whose state has changed.  A state change is considered  to  be: the child terminated; the child was stopped by a signal; or the child was resumed by a signal.  In the case of a terminated child, performing a wait allows the system
       to release the resources associated with the child; if a wait is not performed, then the terminated child remains in a "zombie" state (see NOTES below).

       If a child has already changed state, then these calls return immediately.  Otherwise, they block until either a child changes state or a signal handler interrupts the call  (assuming
       that  system  calls  are  not automatically restarted using the SA_RESTART flag of sigaction(2)).  In the remainder of this page, a child whose state has changed and which has not yet
       been waited upon by one of these system calls is termed waitable.
```

对给定的`pid`执行`waitpid()`后，`waitpid()`会根据 options 参数决定返回值：

```plaintext
       WNOHANG
              return immediately if no child has exited.

       WUNTRACED
              also return if a child has stopped (but not traced via ptrace(2)).  Status for traced children which have stopped is provided even if this option is not specified.

       WCONTINUED (since Linux 2.6.10)
              also return if a stopped child has been resumed by delivery of SIGCONT.
```

各种 option 参数的执行情况如下：

- 默认 option(0)的情况下，waitpid 阻塞等待进程结束，返回进程 PID。
- 使用`WNOHANG`，waitpid 非阻塞；如果子进程结束，返回进程 PID；如果没有子进程结束，立即返回 0。
- 使用`WUNTRACED`，waitpid 阻塞，如果子进程进入停止态，waitpid()立即返回 0。这可以解决默认情况下 waitpid 在子进程进入停止态时无限等待的行为
- 使用`WCONTINUED`， waitpid 阻塞，如果停止的子进程被恢复就立即返回

在当前简单的 REPL 实现里我们直接使用 0 阻塞等待即可。

```cpp
//主线程
auto ret = waitpid(pid, &wstatus, 0);
if (ret == -1) {
  perror("waitpid");
  std::exit(EXIT_FAILURE);
}
```

### Debug

运行时会有错误：

```plaintext
> ls --help
--help: 无法访问 'H'$'\211\302''H'$'\203\352\001''H'$'\211''U'$'\320''H'$'\215\024\305': 没有那个文件或目录
```

两个问题：

- size()大小访问数组导致溢出

```cpp
argv[s.size()] = nullptr;
```

- argv[0]必须要是程序名称。

修改完的代码如下：

```cpp
void loop() {
  std::string line;
  while (true) {
    std::print("> ");
    if (!std::getline(std::cin, line)) {
      break;
    }
    int wstatus{};
    auto s = parse_line(line);
    for (auto &str : s) {
      std::println("{}", str);
    }
    auto pid = fork();
    switch (pid) {
    case -1:
      perror("fork");
      std::exit(EXIT_FAILURE);
    case 0: {
      auto &path = s.front();
      std::vector<char *> argv(s.size() + 1);
      for (size_t i = 0; i < s.size(); ++i) {
        argv[i] = s[i].data();
      }
      argv[s.size()] = nullptr;
      auto ret = execvp(path.c_str(), argv.data());
      if (ret == -1) {
        perror("execvp");
        std::exit(EXIT_FAILURE);
      }
      std::exit(EXIT_SUCCESS);
    }
    default: {
      auto ret = waitpid(pid, &wstatus, 0);
      if (ret == -1) {
        perror("waitpid");
        std::exit(EXIT_FAILURE);
      }
    }
    }
  }
}

```
