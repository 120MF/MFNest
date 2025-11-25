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

今年年底，我把我打磨很久的 CAN 通信框架开源并且发布在了各大论坛上。作为 Linux 初学者和大学生，我非常感谢网络上各位前辈对我作品的认可和斧正。在这里，我想把我自己写这个框架的一些心路历程记录下来，为这段 Linux 开发的旅程做一个简单的总结。

## 动机

我和 Linux CAN 的故事说来说来话长。大一的夏天，工作室的学长给了一台`Manifold 2-G`。这是`DJI`出品的一台嵌入式 Linux 设备，上面直接附带两个原生`CAN`口。当时我们计划把它作为我们`RoboMaster 工程机器人`的上位机，通过 CAN 直接控制机械臂电机。

> ![manifold](manifold.jpg)
> 正在给妙算刷机

这里我就不得不骂一下 DJI，好好一个机器发布完以后一点都不维护，到 2023 年跑的系统还是`Ubuntu 16.04`，手动给这个机器刷机和维护 ROS2、Docker 简直是要老命了……

抛开机器本身的问题不谈，理论上来说，用 CAN 控制电机应该是一个很简单的任务。你只需要发送控制帧，获取反馈帧，把反馈数据同步给 ros2_control 就行。

> ![graph1](graph-1.jpg)
> 理想下的模型

问题是，对于`DJI M3508`和一些其它的类似的电机来说，

> ![graph2](graph-2.jpg)
> 实际的情况
