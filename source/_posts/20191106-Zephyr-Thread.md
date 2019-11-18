---
title: Zephyr线程管理 - 概念
s: 20191106-Zephyr-Thread
date: 2019-11-16 20:56:38
tags:
categories:
  - RTOS
---

## 前言

一直想写一些RTOS的技术资料，算作对自己之前一些相关技术调研的总结。无奈懒癌发作，一拖再拖。然今日灌上鸡血，笃定主意，从最基本的调度相关内容开始。  

简单讲，Zephyr是一个开源实时操作系统。相较Linux，其对系统资源的使用量更小，当然也牺牲了许多复杂且完善的功能（如，系统Debug易用性，线程的堆栈保护）。与此同时，因其是开源社区开发，也多少继承和保留了许多Linux系统的优秀思想和功能（例如，Workqueue、设备树等）。其定位为万物互联时代各种各样的嵌入式设备，目光长远。

Zephyr里没有管程、进程、线程之分。除了中断响应例程，其余所有可执行调度单位皆是线程。系统中应用程序可以定义一个或者多个线程，这种情况下，每个线程也都有自己独立的调度信息和线程ID。

虽然也有区分用户空间和内核空间态，Zephyr线程却没有自己独立的地址转换表，皆使用相同的地址空间，这是由Zephyr的内存管理方式决定的。

## 线程状态

线程状态即转换关系如下图所示：  
![https://docs.zephyrproject.org/latest/_images/thread_states.svg](https://docs.zephyrproject.org/latest/_images/thread_states.svg)
可以看出，线程分为如下状态：  

+ New：表示线程新创建
+ Ready: 线程就绪状态
+ Waiting: 线程等待某种IO资源
+ Running: 线程执行中
+ Suspended: 线程挂起状态，非等待资源原因被停止执行
+ Terminated: 线程退出

以上线程状态转换关系见图，后续进行代码分析将进一步介绍。

## 线程优先级

线程的优先级使用数字表示，数字越小，线程的优先级越高。Zephyr系统的线程可以分为两类:  

+ Cooperative线程：一旦开始执行，除非中断或者线程自行让出CPU，将会一直执行
+ 可抢占线程: 普通线程，可被更高优先级的线程抢占执行

Zephyr默认给Cooperative线程分配小于0的优先级数值，而可抢占线程分配为正值。用户可修改编译选项来更改这两种线程的优先级区间，当然要保证Cooperative线程的优先级数值区间小于可抢占线程。线程运行过程中，其优先级可以被更改。一张图表示线程优先级关系：

![https://docs.zephyrproject.org/latest/_images/priorities.svg](https://docs.zephyrproject.org/latest/_images/priorities.svg)

## 线程属性

线程有一列属性，根据这些不同的属性，线程的执行方式也会有相应的差异。一些主要的属性如下:  

属性|说明|
-|-|
K_ESSENTIAL|表示线程是核心线程，该类线程若有退出或者取消是系统不允许的，系统会断言严重错误|
K_SSE_REGS|X86独有属性，表示是否使用CPU的SSE功能|
K_FP_REGS|表示是否使用CPU的浮点计算寄存器|
K_USER|表示用户空间态线程，只有当系统编译选项CONFIG_USERSPACE打开时才有效|
K_INHERIT_PERMS|对USERSPACE线程有效，表示是否继承父进程的权限属性|

## 线程自定义数据

线程可以自定义数据，使得应用可以对线程功能做一定程度的扩展。

## 特殊线程

### 系统线程

Zephyr内核初始化会创建的一些初始线程。主要分为主线程和空闲线程。

### Workqueue

与Linux系统的Workqueue相似，主要用于中断下半部使用。

## 总结

本文为Zephyr系统，线程的基本概念。主要介绍了线程的状态、分类、优先级等基础思想。