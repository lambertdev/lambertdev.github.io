---
title: Linux中断学习笔记(1)
tags:
  - Linux
  - 中断
url: 460.html
id: 460
categories:
  - Linux
  - Linux中断
date: 2018-07-15 00:56:44
---

什么是中断
=====

CPU获取外设状态变化有两种方式:

*   Polling：不断跟外设询问它的状态
*   当外设状态变化后主动通知CPU

CPU要负责处理系统中各种各样的业务，如果频繁地轮询外设状态，必然会对整个系统的吞吐量产生影响，影响操作系统的正常运作。 中断便是外设通知CPU其状态变化的一种机制。CPU会有中断线，由中断控制器的输出线连接。中断控制器作用：

*   中断优先级处理
*   接收中断控制器ACK
*   分发中断

中断控制器连接到MCU的中断输入引脚（在ARM就是IRQ-Interrupt Request or FIQ-Fast Interrupt Request信号线）。中断控制器驱动负责Kernel中断号与物理中断号的Mapping。 当外设没有变化的时候，CPU可以专心地处理其他事务。外设准备好资料时，通过中断控制器通知CPU，CPU再对其进行相应的处理。

Linux的中断处理
==========

顶半部（Top Half）和底半部（Bottom Half）
------------------------------

中断会将正在运行的程序给打断（操作系统会保留相应的执行数据以便返回）。外设中断有时要处理的数据量很大，为了避免中断处理长期占用CPU导致系统中其他事务无法正常进行，Linux将中断处理分为顶半部和底半部。顶半部主要负责响应中断，处理该外设一些必须的业务，同时将需要复杂处理的事务放到底半部执行。 Linux中中断的顶半部是不能被Block的，原因： _操作系统并没有为中断分配相应的进程管理单元，当中断服务例程因为发生Block而被系统Schedule出去后，便无法再返回到该中断服务例程。而因为中断被调度出的进程，被schedule后却不能得到调度。_ 实现底半部有多种方式，常用的是Tasklet（特殊的软中断）和Workqueue（内核线程）

### 软中断和Tasklet

当中断上半部执行结束后，操作系统会调度执行软中断。具体实现以linux-4.x为例：

./kernel/softirq.c
```C
void arch_do_IRQ(unsigned int irq, struct pt_regs *regs)                //中断向量表调用相应的IRQ处理函数
{
    struct pt_regs *old_regs = set_irq_regs(regs);                      //保留中断现场

    irq_enter();                                                        //进入中断处理前的必要处理，如关闭硬件中断,关闭抢占（preempt），禁止ksoftirqd启动处理（softirq）-- 因为稍后会看到，中断结束后会做处理
    generic_handle_irq(irq);                                            //执行外设驱动注册的irq handler
    irq_exit();                                                         //进入到irq_exit
    set_irq_regs(old_regs);
}
void irq_exit(void)
{
#ifndef __ARCH_IRQ_EXIT_IRQS_DISABLED
    local_irq_disable();
#else
    WARN_ON_ONCE(!irqs_disabled());
#endif
    account_irq_exit_time(current);
    preempt_count_sub(HARDIRQ_OFFSET);                                 //打开抢占
    if (!in_interrupt() && local_softirq_pending())                    //如果有软中断pending，执行softirq
        invoke_softirq();
    tick_irq_exit();
    rcu_irq_exit();
    trace_hardirq_exit(); 
}
```
可以看到，软中断是在中断处理进程退出后，立即执行。软中断处理函数，会检查当前系统中有哪些软中断，然后循环执行。因为系统仍有很多任务要处理，包括之前被抢占的任务，因此kernel不会无限度的将时间片都留给软中断。它设置了一个软中断循环执行的最大次数。当超过这个次数后，就退出软中断执行，并将内核线程ksoftirqd叫起。 ksoftirqd是内核线程，他与系统中其他线程包括用户态进程一起接受系统调度。因此保证了未完成的软中断也可再后来继续执行完毕。 Tasklet是一种特殊的软中断。Workqueue是一种特殊的内核线程。它们都是实现中断下半部的方式。具体用法可在有使用需求时查看。

irq 在多处理器系统的分发
--------------

*   中断亲和力：多处理系统中，操作系统会按照一定策略把中断分配到各CPU Core来处理。中断亲和力高的CPU Core，操作系统会倾向于把中断交给其处理
*   通过修改/proc/irq/smp_affinity 可以改变Linux的中断亲和力
*   内核线程kirqd：周期性执行do_irq_balance()。Track最近时间间隔每个Core的接收中断次数，动态做Balance