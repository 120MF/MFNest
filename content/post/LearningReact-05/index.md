+++
title = '学习React第5章-性能优化'
date = 2024-10-17T21:00:50+08:00
draft = false
description = ""
slug = "学习React第5章-性能优化"
image = ""
categories = ["编程相关","前端"]
tags = ["JavaScript","React","JSX","前端","学习笔记","性能优化"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
tags = ["JavaScript","React","JSX","前端","学习笔记","性能优化"]
readingTime = true
+++

React开发过程中，我们有些时候会遇到一些大型组件产生性能问题，造成网页卡顿现象。这时，我们可以针对性地对这些问题进行优化。基本上，我们会从这几个方面考虑性能优化：

- 防止无必要的重渲染；
- 提升应用响应速度；
- 减少打包大小；

![性能优化及其工具](image.png)

## 重渲染优化

有时，我们构造组件的方式会导致无效的重渲染。如果部分组件的渲染过程缓慢，这部分组件又被反复无用地重渲染，那么应用的相应速度就会被拖慢。

![Archive组件拖慢了SearchBar](image-2.png)

### 使用Profiler工具

在实际开发中，可以使用`React Developer Tool`插件中的`Profiler`工具来检测网站性能。

![Profiler工具](image-1.png)

在选项卡中，你可以清楚的看到哪些组件触发了重渲染，以及重渲染的触发源。

### 组合组件避免重渲染

一些情况下，我们可以简单把缓慢组件作为`children`道具传入父组件。这样就能避免缓慢组件导致父组件中的其它子组件重新渲染。

![使用children](image-3.png)

