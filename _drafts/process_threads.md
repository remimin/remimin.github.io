--
layout:     post
title:      "Linux进程和线程小结"
subtitle:   ""
date:       2018-06-11
author:     "min"
header-img: "img/post-bg-2015.jpg"
tags:
    - Linux, OS, thread, process
---

# Linux进程和线程

> 写在前面的废话，作为一名窃以为比较自身的coder，对进程和线程理解不会再有什么需要思考的地方，
就是直接套取自己的经验值选择进程或者线程即可，奈何在被问了几个问题之后才知自己太想当然，或者
时自己的弊端活在自己的逻辑里，BUT，知不足就要补起来。对进程和线程理解，相信部分童鞋跟我一样
局限于大学课本上的几个描述，并未真正去探究过。

## 进程是什么

进程从直观的角度看，就是一个正在运行的程序，包含了可执行code从外设读取数据加载到内存，由cpu调用执行。
从操作系统角度看进程包含了地址空间，内存映射，读写操作，进程状态切换和线程管理等等。

