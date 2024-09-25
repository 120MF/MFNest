+++
title = '仅需10分钟 最简单的VSCode&STM32开发教程 Windows篇'
date = 2024-04-14T18:08:17+08:00
draft = false
description = ""
slug = "仅需10分钟-最简单的VSCode&STM32开发教程-Windows篇"
image = "/media/屏幕截图-win1-screen0.webm-6-edited.png"
categories = ["编程相关","环境配置"]
tags = ["C/C++","STM32","VS Code","Makefile","STM32CubeMX","stm32forvscode"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
keywords = ["C/C++","STM32","VS Code","Makefile","STM32CubeMX","stm32forvscode"]
readingTime = true
+++

## 前言

相比于Keil，使用VSCode开发STM32十分方便迅捷。它的好处包括且不限于：

- 完全免费；
- 使用VS Code内置的git插件轻松进行团队开发；
- 编译速度是Keil5的10倍以上；
- 比Keil好用10000倍的代码补全；
- 丰富的主题、字体和扩展插件（~~daily waifu~~）；
- 多平台支持——x86/Arm，Windows、Linux，甚至是安卓手机

我从湖南大学的RM开源工程第一次了解到这种开发方式。然而，其复杂和低容错配置过程也实在是劝退了不少开发者。本次我将带你使用最便捷的方法，在10分钟内（网络条件良好）配置好VSCode和STM32的开发环境。

## 通过Scoop安装STM32开发环境

为了免去以后设置环境变量的麻烦，我们使用Scoop来安装STM32的开发环境。国内连接Scoop源的速度较慢，这里给出设置代理源的方法。

首先按Win+R，输入powershell并回车，运行powershell。

然后输入以下命令：

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
irm https://mirror.ghproxy.com/raw.githubusercontent.com/lzwme/scoop-proxy-cn/master/install.ps1 | iex
#换源
scoop config SCOOP_REPO https://mirror.ghproxy.com/github.com/ScoopInstaller/Scoop
scoop install git
scoop bucket add spc https://mirror.ghproxy.com/github.com/lzwme/scoop-proxy-cn
scoop bucket add extras
#安装环境
scoop install openocd make gcc-arm-none-eabi
```

## 安装VS Code及其插件

在官网下载并安装VSCode，打开VSCode的Extensions（插件）选项

确保安装：stm32-for-vscode、C/C++、Makefile Tools三个插件

<img src="/media/屏幕截图-win1-screen0.webm.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/屏幕截图-win1-screen0.webm-2.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

<img src="/media/屏幕截图-win1-screen0.webm-3.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

## 修改STM32CubeMX配置

在Project Manager选项卡中将Toolchain / IDE更改为Makefile。

<img src="/media/屏幕截图-win1-screen0.webm-4.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

小技巧：你可以在一个工作目录下同时生成Keil和Makefile工程文件，同时使能两种开发工具。

## 通过VS Code进行STM32开发

在VS Code内打开生成的工程文件夹。第一次打开时，右下角会弹出Makefile Tool的配置通知，选Yes后就能自动配置完成。

<img src="/media/屏幕截图-win1-screen0.webm-5.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

按下Ctrl+Shift+P，输入stm32，点击STM32: Open the STM32 for VSCode extension，之后VS Code窗口左侧会多处一个stm32-for-vscode插件的选项框，点击一下就能调出并进行各种操作。下面介绍各种操作的涵义：

<img src="/media/屏幕截图-win1-screen0.webm-6-edited.png" style="display: block; margin-left: auto; margin-right: auto;" alt="image">

- Build/Clean Build：编译/完全重新编译。编译的速度很快，重新编译也仅需3s左右。（如果Make报错，在工作目录下新建一个build文件夹即可）
- Flash STM32：将编译过的程序烧录进STM32。请在烧录前确认USB接口情况并选择烧录器类型。
- Debug STM32：调试STM32程序。
- Change Programmer：切换烧录器类型。可切换至任意Openocd支持的烧录器。

到此为止，恭喜你解锁了VS Code开发STM32的功能！