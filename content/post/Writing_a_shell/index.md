+++
title = '自制Shell'
date = 2025-11-15T21:00:50+08:00
draft = false
description = ""
slug = ""
image = ""
categories = ["编程相关","Linux"]
tags = ["Linux"]
weight = 1       # You can add weight to some posts to override the default sorting (date descending)
keywords = []
readingTime = true
+++

## 实现基本循环

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

## 解析用户指令

