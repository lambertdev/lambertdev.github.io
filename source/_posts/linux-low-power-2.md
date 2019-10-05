---
title: LINUX电源管理介绍-2
tags:
  - Linux
  - 电源管理
url: 1080.html
id: 1080
categories:
  - Linux电源管理
date: 2018-10-29 12:57:22
---

前篇文章
====

http://l2h.site/2018/10/24/linux-low-power-1/ ![LINUX电源管理介绍-2](http://pic.l2h.site/l2hsiteLinux-low-power-2.png "LINUX电源管理介绍-2")

电源管理相关状态设置流程
============

通过用户空间态访问内核开出的sysfs节点autosleep/state，可以直接控制Linux内核的休眠。例：

> echo “mem” > /sys/power/state cat /sys/power/autosleep

/sys/power/state
----------------

/sys/power/state节点的处理代码定义在kernel/power/main.c。首先，节点定义如下：

power\_attr(state);          //定义state sysfs节点（struct kobj\_attribute）

当从用户态写该节点时（例，前文提到的echo命令），内核中响应函数如下(忽略一些非核心代码)：

static ssize\_t state\_store(struct kobject \*kobj, struct kobj\_attribute \*attr, const char *buf, size\_t n)

{
       error = pm\_autosleep\_lock();
       state = decode\_state(buf, n);            //将用户空间态传入的state字符串，转化为suspend\_state_t，目前只接收"mem", "standby", "freeze"三种输入
       if (state < PM\_SUSPEND\_MAX)
              error = pm\_suspend(state);         //调用pm\_suspend，根据解析出的电源state进入不同的休眠等级
       else if (state == PM\_SUSPEND\_MAX)
              error = hibernate();                //否则调用hibernate，suspend to disk
       else
              error = -EINVAL;
 out:
       pm\_autosleep\_unlock();
       return error ? error : n;
}

而读取这个sysfs节点的响应函数则很简单了，如下：

power_attr(autosleep);

static ssize\_t state\_show(struct kobject \*kobj, struct kobj_attribute \*attr,char *buf)
{
       char *s = buf;
#ifdef CONFIG_SUSPEND
       suspend\_state\_t i;
       for (i = PM\_SUSPEND\_MIN; i < PM\_SUSPEND\_MAX; i++) //循环将所有电源管理状态加入到要返回的字符串
              if (pm_states\[i\])
                     s += sprintf(s,"%s ", pm_states\[i\]);  
#endif
       if (hibernation_available())
              s += sprintf(s, "disk "); //如果系统支持Hibernate，则将disk加入要返回的字符串
       if (s != buf)
              *(s-1) = '\\n';
       return (s - buf);
}

/sys/power/autosleep
--------------------

autosleep节点同样定义在kernel/power/main.c，其基本结构与state基本相似。当从用户空间态读取该节点时，返回全局变量autosleep_state的值对应的字符串（"mem", "standby", "freeze"或者”disk”）。 而写该节点时，处理与/sys/power/state的处理则稍有不同：

static ssize\_t autosleep\_store(struct kobject \*kobj,struct kobj\_attribute \*attr,const char *buf, size\_t n)
{
       suspend\_state\_t state = decode_state(buf, n);
       int error;
       if (state == PM\_SUSPEND\_ON && strcmp(buf, "off") && strcmp(buf, "off\\n"))
              return -EINVAL;
       error = pm\_autosleep\_set\_state(state); //将suspend\_state_t类型的状态传入供处理
       return error ? error : n;
}

int pm\_autosleep\_set\_state(suspend\_state_t state)
{
       \_\_pm\_stay\_awake(autosleep\_ws);  // 避免系统进入休眠状态（这里传入全局变量，autosleep_ws，表明是autosleep wakeup source）
       mutex\_lock(&autosleep\_lock);
       autosleep\_state = state;         // 将传入的状态赋值给全局变量autosleep\_state
       \_\_pm\_relax(autosleep_ws);       // 释放休眠锁
       if (state > PM\_SUSPEND\_ON) {
              pm\_wakep\_autosleep\_enabled(true); //对系统中注册的所有wakeup source，均设置其中的autosleep成员为true，并更新start\_prevent_time
              queue\_up\_suspend_work(); //将autosleep例程加入到kernel的workqueue进行调度
       } else {
              pm\_wakep\_autosleep_enabled(false); //对系统中注册的所有wakeup source，均设置其中的autosleep成员为false
       }
       mutex\_unlock(&autosleep\_lock);
       return 0;
}

/sys/power/wake\_lock & wake\_unlock
------------------------------------

wake\_lock和wake\_unlock为用户空间态拿睡眠锁与释放睡眠锁的节点。不出意料，它们还是定义在kernel/power/main.c。Kernel对这两个节点的读取处理几乎一致，以wake_lock的读取为例，对它的读取打印系统中所有激活的wakeup source。

static ssize\_t wake\_lock\_show(struct kobject \*kobj,struct kobj\_attribute \*attr,char *buf)
{
       return pm\_show\_wakelocks(buf, false);
}

当从用户空间态写这两个节点时，内核处理则不相同。写wake\_lock和wake\_unlock的处理分别如下：

static ssize\_t wake\_lock\_store(struct kobject \*kobj,struct kobj\_attribute \*attr,const char *buf, size_t n)
{
       int error = pm\_wake\_lock(buf); //buf内为拿休眠锁的wakeup\_source的名字，将该wakeup\_source注册到内核中
       return error ? error : n;
}

static ssize\_t wake\_unlock\_store(struct kobject \*kobj,struct kobj\_attribute \*attr,const char *buf, size_t n)
{
       int error = pm\_wake\_unlock(buf); //buf内为释放休眠锁的wakeup\_source的名字，告诉内核释放该wakeup\_source拿的休眠锁
       return error ? error : n;
}

Autosleep代码分析
=============

Autosleep其实为一个内核的一个workqueue，其定义在kernel/power/autosleep.c，核心处理函数为try\_to\_suspend。代码如下：

static DECLARE\_WORK(suspend\_work, try\_to\_suspend);
static void try\_to\_suspend(struct work_struct *work)
{
       unsigned int initial\_count, final\_count;
       if (!pm\_get\_wakeup\_count(&initial\_count, true))
              goto out;
       mutex\_lock(&autosleep\_lock);
       if (!pm\_save\_wakeup\_count(initial\_count) ||
              system\_state != SYSTEM\_RUNNING) {
              mutex\_unlock(&autosleep\_lock);
              goto out;
       }
       if (autosleep\_state == PM\_SUSPEND_ON) {
              mutex\_unlock(&autosleep\_lock);
              return;
       }
       if (autosleep\_state >= PM\_SUSPEND_MAX)
              hibernate();
       else
              pm\_suspend(autosleep\_state);

       mutex\_unlock(&autosleep\_lock);
       if (!pm\_get\_wakeup\_count(&final\_count, false))
              goto out;
       /\*
        \* If the wakeup occured for an unknown reason, wait to prevent the
        \* system from trying to suspend and waking up in a tight loop.
        */

       if (final\_count == initial\_count)
              schedule\_timeout\_uninterruptible(HZ / 2);
 out:
       queue\_up\_suspend_work();

}

**第5,6行(pm\_get\_wakeup_count)**，读取wakeup source的总数到initial_count供函数后续使用。同时判断若当前激活的wakeup source的数量大于0（表示有人持锁），则跳出函数执行。 **第8-12行(pm\_save\_wakeup_count)**，再次检查，如果执行期间wakeup source数量有变化、或者持锁的wakeup source数量大于0，或者当前系统已经不在休眠状态，则跳出执行。 **第13-16行(autosleep\_state == PM\_SUSPEND_ON)**，若自动休眠状态的等级为非休眠，则跳出执行。 **第17-20行**，进入到系统休眠。 **第22-31行**，从休眠跳出后，马上检查系统的wakeup\_source数量。若wakeup\_source总数量没有变化，会让出该线程执行，让内核重新调度。从注释的解释分析，这里的作用主要是因为从休眠中跳出，但是没有新的wakeup source产生，为了避免系统再次进入到休眠。所以让出系统调度，让造成休眠跳出的系统模块得以注册新的wakeup source。 **第33行**，将try\_to\_suspend加入workqueue，等待下次调度。