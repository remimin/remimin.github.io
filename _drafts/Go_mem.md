---
layout:     post
title:      "Golang内存管理"
subtitle:   ""
date:       2018-05-11
author:     "min"
header-img: "img/post-bg-2015.jpg"
tags:
    - Golang, memory management
---

# Golang内存管理

程序运行内存会有三部分组成堆(heap)栈(stack)和数据区(data)：
- 堆存储动态内存分配区
- 栈存储临时变量
- 数据区全局变量和静态变量

    