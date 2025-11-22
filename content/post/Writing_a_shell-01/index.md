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

## 仓库

[wsh](https://github.com/120MF/wsh)
