+++
title = 'WeaveShell 01 - 重定向与管道'
date = 2025-11-22T00:21:12+08:00
draft = false
description = "C/C++ 自制 Unix(Linux) Shell，实现管道与重定向。"
slug = ""
image = "1.png"
categories = ["编程相关","Linux"]
tags = ["Linux", "Shell", "进程"]
weight = 1
keywords = ["Linux", "Shell", "进程", "WeaveStar", "MoonFeather"]
readingTime = true
+++

## 序

这次我们为 WeaveShell 实现重定向与管道功能！

## 核心概念

在 Linux 中，进程启动时默认会打开三个文件描述符：

- 0 (STDIN_FILENO): 标准输入（通常是键盘/终端）
- 1 (STDOUT_FILENO): 标准输出（通常是屏幕/终端）
- 2 (STDERR_FILENO): 标准错误（通常是屏幕/终端）

重定向和管道的本质，就是把这三个数字（0, 1, 2）背后指向的“文件”换成别的东西。

## 重定向

实现以下功能：

- ls > out.txt: 不把结果打印到屏幕（FD 1），而是写进 out.txt。
- grep hello < in.txt: 不从键盘（FD 0）读数据，而是从 in.txt 读。

### 语法解析

#### 单符号

符号（>, <, >>）右边的第一个 token 必定被视为目标文件路径，而剩下的部分（无论是在左边还是远处）被视为程序及其参数。

在程序解析时，通常将运算符后紧跟着的一个单词作为文件名，然后将该重定向操作从命令参数中剔除。剩下所有单词重新拼接起来，作为传递给`execvp`。

例如：

```bash
> output.txt ls -l
ls > output.txt -l
```

都能正常工作！ 结果都是把 ls -l 的内容写入 output.txt。

#### 多符号

Shell 从左往右解析语句，最后的那个重定向语句作为最终结果。

例如：

```bash
grep "ls" < 2.txt < 1.txt # only input from 1.txt
ls > 1.txt > 2.txt # only output to 2.txt
```

当然，我们也可以输入多种/多个重定向符号，这种情况下，输入/输出重定向符号分别代表了它们各自的流，这些流各自取最后的语句作为最终结果。

例如：

```bash
echo "hello world" > in.txt
cat < in.txt > out.txt
cat out.txt
# hello world
cat < out.txt < in.txt > in.txt > out.txt
# same as cat < in.txt > out.txt
```

补充一个我个人觉得很有趣的 zsh 下的行为：`MULTIOS`

```bash
ls > 1.txt > 2.txt # 两个文件都会被输出
grep "hello" < 1.txt < 2.txt # 两个文件的内容会被叠加到输入流中
```

### 重构解析器

之前，我们的解析器返回用户输入的分词；现在我们在解析器中解析可能存在的重定向符号，将其与文件名从用户输入中剔除并返回。这样还避免了改动原来的子进程逻辑。

```cppif (fd == -1) {
        // ...
      }
      int ret = dup2(fd, STDIN_FILENO);
      if (ret == -1) {
        // ...
      }
//Vector内数组分别代表重定向符和文件名
using Redirects = std::vector<std::array<std::string, 2>>;
// 解析结果
struct ParseResult {
  std::vector<std::string> words;
  Redirects redirects;
};
```

接下来，在原先的解析函数中加入新的剔除重定向逻辑

```cpp
ParseResult parse_line(std::string str) {
  ParseResult res;
  auto &words = res.words;
  std::stringstream s(str);
  std::string word;
  while (s >> word) {
    if (word == ">" || word == "<" || word == ">>") {
      std::array<std::string, 2> tmp;
      tmp[0] = word;
      s >> tmp[1];
      res.redirects.push_back(std::move(tmp));
    } else {
      words.push_back(word);
    }
  }
  return std::move(res);
}
```

这样，我们就能在子进程中使用解析到的重定向符和文件名了。

### 实现重定向

#### 文件描述符

要实现输出、输入的重定向，我们需要理解在 Linux 中，输出和输入代表了什么。

> 在 Linux 中，文件描述符（File Descriptor）是一个非负整数，它是内核为了高效管理已被打开的文件所创建的索引。
>
> 每一个进程都有一个文件描述符表，指向打开的文件、Socket、管道等。默认情况下：
>
> - **标准输入 (stdin)** 对应文件描述符 `0`
> - **标准输出 (stdout)** 对应文件描述符 `1`
> - **标准错误 (stderr)** 对应文件描述符 `2`

程序的输入输出操作本质上是对这些文件描述符的读写。重定向操作的核心，就是通过系统调用（如 `dup2`）修改文件描述符表中 `0`、`1` 或 `2` 所指向的具体文件对象。

#### 获取文件描述符

我们可以使用`open()`函数打开文件并获取其文件描述符`fd`。

```plaintext
man 2 open

NAME
       open, openat, creat - open and possibly create a file

SYNOPSIS
       #include <fcntl.h>

       int open(const char *pathname, int flags, ...
                  /* mode_t mode */ );


DESCRIPTION
       The open() system call opens the file specified by pathname.  If the specified file does not exist, it may optionally (if O_CREAT is specified in flags) be
       created by open().

       The return value of open() is a file descriptor, a small, nonnegative integer that is an index to an entry in the process's table of open file descriptors.
       The  file  descriptor is used in subsequent system calls (read(2), write(2), lseek(2), fcntl(2), etc.)  to refer to the open file.  The file descriptor re‐
       turned by a successful call will be the lowest-numbered file descriptor not currently open for the process.

```

在 REPL 循环中，我们先获取最后一次出现的重定向文件名，并使用 open()函数打开它。

```cpp
if (!out_file.empty()) {
  // 覆盖/追加 | 创建文件 | 读写权限 , 权限
    int fd = open(out_file.data(), out_flag | O_CREAT | O_WRONLY, 0644);
}
```

```plaintext
使用到的 open 标记

       O_TRUNC
              If the file already exists and is a regular file and the access mode allows writing (i.e., is O_RDWR or O_WRONLY) it will be truncated to length  0.
              If the file is a FIFO or terminal device file, the O_TRUNC flag is ignored.  Otherwise, the effect of O_TRUNC is unspecified.


       O_APPEND
              The file is opened in append mode.  Before each write(2), the file offset is positioned at the end of the file, as if with lseek(2).  The  modifica‐
              tion of the file offset and the write operation are performed as a single atomic step.

              O_APPEND  may lead to corrupted files on NFS filesystems if more than one process appends data to a file at once.  This is because NFS does not sup‐
              port appending to a file, so the client kernel has to simulate it, which can't be done without a race condition.

       O_CREAT
              If pathname does not exist, create it as a regular file.

......
```

#### 重定向描述符

我们可以使用 `dup()` 或 `dup2()` 函数来复制文件描述符，从而实现重定向。

这两个函数的主要功能是创建一个新的文件描述符，让它指向与旧文件描述符（`oldfd`）相同的内核文件表项（File Description）。这意味着，两个描述符共享同一个文件偏移量（File Offset）和文件状态标志。

- `int dup(int oldfd);`

  - 复制 `oldfd`，返回一个新的文件描述符。
  - 系统保证返回的新描述符是当前未使用的最小整数。

- `int dup2(int oldfd, int newfd);`
  - 复制 `oldfd` 到 `newfd`。
  - 如果 `newfd` 已经被打开，`dup2` 会先自动关闭它，然后进行复制。
  - 如果 `oldfd` 和 `newfd` 相等，则直接返回 `newfd`，不做任何操作。

以 `ls > out.txt` 为例：

1.  Shell 打开 `out.txt`，得到一个新的文件描述符，假设是 `fd = 3`。
2.  此时，进程有 `0, 1, 2` 指向终端，`3` 指向 `out.txt`。
3.  调用 `dup2(3, 1)`：
    - `dup2` 会先关闭 `1`（断开与终端的连接）。
    - 然后让 `1` 指向 `3` 所指向的文件（即 `out.txt`）。
4.  此时，向 `1`（标准输出）写入数据，实际上就是写入 `out.txt`。
5.  `ls` 程序并不知道发生了什么，它只是照常向 `fd 1` 写入结果，但数据被“重定向”到了文件。

> 具体细节可参考 man 2 dup

我们现在只需要将刚刚`open()`得到的 fd 使用`dup2()`复制并替换标准输入/输出流即可。

```cpp
    if (!out_file.empty()) {
      int fd = open(out_file.data(), out_flag | O_RDWR);
      if (fd == -1) {
        perror("open");
        std::exit(EXIT_FAILURE);
      }
      dup2(fd, STDOUT_FILENO);
    }
```

> 注意：子操作完成后，不要忘记`close()`刚才`open()`的`fd`！

```cpp
//成功创建后
fds.push_back(fd);
// 批量关闭
for (auto fd : fds) {
    close(fd);
}
```

#### 实现代码

```cpp
void child_process(ParseResult &result) {
  // Parse exe
  // ...
  // Parse redirection
  // 这里保留在子进程中解析vector的方法，未来可以实现MULTIOS
  if (!result.redirects.empty()) {
    std::string_view out_file{}, in_file{};
    int out_flag{};
    for (auto &arr : result.redirects) {
      if (arr[0] == ">") {
        out_file = arr[1];
        out_flag = O_TRUNC;
      } else if (arr[0] == ">>") {
        out_file = arr[1];
        out_flag = O_APPEND;
      } else if (arr[0] == "<") {
        in_file = arr[1];
      }
    }
    std::vector<int> fds;
    if (!out_file.empty()) {
      int fd = open(out_file.data(), out_flag | O_CREAT | O_WRONLY, 0644);
      if (fd == -1) {
        perror("open");
        std::exit(EXIT_FAILURE);
      }
      int ret = dup2(fd, STDOUT_FILENO);
      if (ret == -1) {
        perror("dup2");
        std::exit(EXIT_FAILURE);
      }
      fds.push_back(fd);
    }
    if (!in_file.empty()) {
      int fd = open(in_file.data(), O_RDONLY);
      if (fd == -1) {/* ... */}
      int ret = dup2(fd, STDIN_FILENO);
      if (ret == -1) {/* ... */ }
      fds.push_back(fd);
    }
    for (auto fd : fds) {
      close(fd);
    }
  }
  // Execute
  // ...
}

```

## 管道

实现以下功能：

- ls | wc -l
- ls /dev | grep tty

### 语法解析

在 Shell 中，管道（`|`）的作用是把左侧命令的标准输出连接到右侧命令的标准输入。语法解析的目标是把用户输入拆分成有顺序的“命令段（segment）”，每个命令段独立解析为程序名与参数，同时保留每段内可能的重定向信息。

- 管道将多个命令按顺序串联：A | B | C 意味着 A 的 stdout -> B 的 stdin，B 的 stdout -> C 的 stdin。
- 重定向作用域限制在“命令段”内部：例如 `cmd1 > out | cmd2` 将输出重定向到文件后再通过管道传递（依解析策略而定）；更常见的是 `cmd1 | cmd2 > out`，重定向只影响 cmd2。
- 语法解析应遵循从左到右的顺序，但生成的“段”用于后续创建管道并 fork/exec 子进程。

### 实现管道

#### 重构运行流程

不难看出，管道这一功能需要在我们的 Shell 中同时运行多个命令，这意味着我们必须在`loop`中创建`管道数量`个子进程。因此，我们有必要对当前的解析和循环流程进行重构。

可以发现，对管道中每个子命令来说，解析的规则和流程和之前相比没有发生变化；那么，我们可以：

- 保留当前的解析逻辑，作为`子命令`的解析方法；
- 在 Parser 中，我们进一步把`用户输入`通过`|`管道符号切片成`子命令`，并通过 vector 来储存；
- 在 REPL 中，我们遍历每个子命令，为每个子命令 fork 一个子进程，并按照之前的逻辑进行执行。

```cpp
// Parser重构
using Words = std::vector<std::string>;
using Redirects = std::vector<std::array<std::string, 2>>;

struct ParseResult {
  struct Process {
    Words words;
    Redirects redirects;
  };
  std::vector<Process> processes;
};

ParseResult parse_line(std::string str) {
  ParseResult res;
  ParseResult::Process process{};
  std::stringstream s(str);
  std::string word;
  while (s >> word) {
    if (word == "|") {
      // 储存当前已经解析到的命令
      res.processes.push_back(std::move(process));
      // 新建下一个子命令
      process = ParseResult::Process{};
    }
    // 剩下的逻辑不变
    else if (word == ">" || word == "<" || word == ">>") {
      std::array<std::string, 2> tmp;
      tmp[0] = word;
      s >> tmp[1];
      process.redirects.push_back(std::move(tmp));
    } else {
      process.words.push_back(word);
    }
  }
  // 将剩余的命令推入
  if (!process.words.empty() || !process.redirects.empty()) {
    res.processes.push_back(process);
  }
  return std::move(res);
}

```

```cpp
// loop重构
for (auto &process : res.processes) {
  // Start new process
  auto pid = fork();
  switch (pid) {
  case -1:
    // On fork error
    perror("fork");
    std::exit(EXIT_FAILURE);
  case 0:
    // Child process
    child_process(process);
  default:
    // 父进程将创建的PID储存起来
    pids.push_back(pid);
  }
}
for (auto pid : pids) {
  // 统一等待
  auto ret = waitpid(pid, nullptr, 0);
  if (ret == -1) {
    perror("waitpid");
    std::exit(EXIT_FAILURE);
  }
}

```

#### 连接输入输出流

## 仓库

[wsh](https://github.com/120MF/wsh)
