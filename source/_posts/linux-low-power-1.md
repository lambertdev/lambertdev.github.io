---
title: Linux电源管理介绍-1
tags:
  - Linux
  - 电源管理
url: 1069.html
id: 1069
categories:
  - Linux
  - Linux电源管理
date: 2018-10-24 21:46:19
---

前言
==

为了节省功耗，需要让系统在不需要工作的时候尽可能的休眠（甚至断电）。同时，为了不影响到系统功能的正常运作，又必须保证休眠不能打断正在进行的重要事务，且能在适时还原回来。因此，一个高效运行的Low Power框架必不可少。 ![Linux电源管理介绍-1](http://pic.www.l2h.site/l2hsiteLinux-low-power-1.png "Linux电源管理介绍-1") 本文以Linux 4.4内核为基础，介绍Linux的Low Power框架。

架构
==

Linux的电源管理架构如下图。首先内核态导出了一些sysfs节点，用来给用户空间态作电源管理控制：

1.  /sys/power/autosleep : 主要用于控制是否开启内核自动休眠，之后会有章节介绍
2.  /sys/power/state：直接控制进入休眠（有多种state，后边会看到）
3.  /sys/power/wake_lock&wake_unlock：拿Wake lock和释放Wake lock，用于避免正在处理重要事务时系统进入休眠
```
                                                      +-----------+
                                                      |应 用 程 序|
             +------------------+                     +-----+-----+
             |电 源 管 理 程 序  +---------+                 |
             +-----+------------+         |                 |
                   |                      |                 |
                   |                      |                 |
用 户 空 间 态      v                      v                 v
            /sys/power/autosleep   /sys/power/state  /sys/power/wake_lock&wake_unlock
+------------------+----------------------+-----------------+---------------------------
                   |                      |                 |
内 核 态           |                      |                 |
                   v                      v                 v 
             +-----+----------------------+-----------------+---+
             |                   sysfs                          |
             +--------------------------------------------------+
           kernel/power/main.c                              |
               +                                            |
   +--------------------------------------------------------|-------------------------+
   |           |                                            v        电 源 管 理 框 架 |
   |           |                   +--------------------------------+                 |
   |           v                   |                                |                 |
   |  +----------+                 | Wakeup Source                  |                 |
   |  |autosleep | <---------------+                                |                 |
   |  +----------+                 +----------------------------^---+                 |
   |kernel/power/autosleep.c                                    |                     |
   |        |                                             +-----+-----------+         |
   |        |                                             |                 |         |
   |        |-------------------------------------------->|  驱 动 程 序     |         |
   |        v                                             |                 |         |
   |  +-----+--------+                                    +-----------------+         |
   |  | 平 台 相 关   |                                                                |
   |  | 睡 眠 代 码   |                                                                |
   |  +--------------+                                                                |
   |                                                                                  |
   +----------------------------------------------------------------------------------+
```

内核态是实现电源管理的核心代码，主要分为如下几大功能模块：

1.  提供用户空间态操作的SYSFS节点操作
2.  记录系统中拿睡眠锁的Wake Source
3.  内核自动休眠例程Autosleep
4.  平台相关睡眠（及唤醒）代码
5.  各个硬件驱动程序：
    1.  注册硬件相关的睡眠及唤醒代码，保证经过一次断电恢复后的硬件仍能正常工作。
    2.  根据应用需要，拿Wake Lock和释放Wake Lock，保证硬件驱动可以正常工作。

数据和变量
=====

在对电源管理的系统流程进行介绍之前，先对电源管理的一些必要的数据结构和关键变量进行分析。

wakeup_source/wakeup_sources
------------------------------

wakeup_source名字具有一定迷惑性，乍一看像是触发系统从睡眠中唤醒的唤醒源，实则不然。这个结构体是用来描述所有“阻挡”系统进入休眠的对象，即向电源管理模块拿wakelock用的。而wakeup_sources则是记录系统中所有wakeup_source的链表。
```C
struct wakeup_source {
	const char 		*name;            //名称
	struct list_head	entry;             //挂入wakeup_sources链表的表头
	spinlock_t		lock;              //自旋锁保护该wakeup_source内容的修改
	struct wake_irq		*wakeirq;
	struct timer_list	timer;             //wakeup source相关的定时器
	unsigned long		timer_expires;  //超时时间，也就是wake_lock_timeout()里面的时间参数，超时后会执行deactivate函数
	ktime_t total_time;                  //该wakeup source被激活的总时间
	ktime_t max_time;                  
	ktime_t last_time;                  //该wakeup source上次被访问的时间
	ktime_t start_prevent_time;           //上次因wakeup source导致系统无法休眠的时间
	ktime_t prevent_sleep_time;          //’阻止系统休眠的时长
	unsigned long		event_count;   //产生事件数
	unsigned long		active_count;   //激活次数
	unsigned long		relax_count;    //释放次数
	unsigned long		expire_count;   //Wakeup source的超时次数
	unsigned long		wakeup_count;  //该source阻止系统休眠的次数
	bool			active:1;               //是否激活状态
	bool			autosleep_enabled:1;    //系统是否开启autosleep
};
```
suspend_state_t
-----------------

描述系统休眠几种不同状态，按照睡眠从浅入深分别是：

1.  On：未休眠
2.  Freeze：轻量级的休眠状态。
3.  Standby：中等休眠状态，CPU没有挂起（Memory和CPU带电）
4.  Mem：深度休眠，系统及设备状态挂起到Memory（Memory仍然带电），CPU不再工作（少数外围设备工作）。
```C
typedefint __bitwise suspend_state_t;

#define PM_SUSPEND_ON         ((__force suspend_state_t) 0)

#define PM_SUSPEND_FREEZE   ((__force suspend_state_t) 1)

#define PM_SUSPEND_STANDBY      ((__force suspend_state_t) 2)

#define PM_SUSPEND_MEM             ((__force suspend_state_t) 3)

#define PM_SUSPEND_MIN        PM_SUSPEND_FREEZE

#define PM_SUSPEND_MAX              ((__force suspend_state_t) 4)
```
combined_event_count
----------------------

用于记录系统中已注册的wakeup事件数量和正在生效的wakeup事件数量。这两个变量需要一起更新的，所以用combined_event_count一个变量来记录。高16 bit存放正在生效的wakeup事件数量，低16bit存放wakeup事件总数量。

后续章节预告
======

> Autosleep及power状态设置流程 Autosleep代码分析 Wakelock拿锁与释放锁流程分析