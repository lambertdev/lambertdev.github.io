---
title: LINUX电源管理介绍-2
tags:
  - Linux
  - 电源管理
url: 1080.html
id: 1080
categories:
  - Linux
  - Linux电源管理
date: 2018-10-29 12:57:22
---

前篇文章
====

http://www.l2h.site/2018/10/24/linux-low-power-1/ ![LINUX电源管理介绍-2](http://pic.www.l2h.site/l2hsiteLinux-low-power-2.png "LINUX电源管理介绍-2")

电源管理相关状态设置流程
============

通过用户空间态访问内核开出的sysfs节点autosleep/state，可以直接控制Linux内核的休眠。例：

> echo “mem” > /sys/power/state cat /sys/power/autosleep

/sys/power/state
----------------

/sys/power/state节点的处理代码定义在kernel/power/main.c。首先，节点定义如下：

power_attr(state);          //定义state sysfs节点（struct kobj_attribute）

当从用户态写该节点时（例，前文提到的echo命令），内核中响应函数如下(忽略一些非核心代码)：
```C
static ssize_t state_store(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t n)

{
       error = pm_autosleep_lock();
       state = decode_state(buf, n);            //将用户空间态传入的state字符串，转化为suspend_state_t，目前只接收"mem", "standby", "freeze"三种输入
       if (state < PM_SUSPEND_MAX)
              error = pm_suspend(state);         //调用pm_suspend，根据解析出的电源state进入不同的休眠等级
       else if (state == PM_SUSPEND_MAX)
              error = hibernate();                //否则调用hibernate，suspend to disk
       else
              error = -EINVAL;
 out:
       pm_autosleep_unlock();
       return error ? error : n;
}
```
而读取这个sysfs节点的响应函数则很简单了，如下：
```C
power_attr(autosleep);

static ssize_t state_show(struct kobject *kobj, struct kobj_attribute *attr,char *buf)
{
       char *s = buf;
#ifdef CONFIG_SUSPEND
       suspend_state_t i;
       for (i = PM_SUSPEND_MIN; i < PM_SUSPEND_MAX; i++) //循环将所有电源管理状态加入到要返回的字符串
              if (pm_states\[i\])
                     s += sprintf(s,"%s ", pm_states\[i\]);  
#endif
       if (hibernation_available())
              s += sprintf(s, "disk "); //如果系统支持Hibernate，则将disk加入要返回的字符串
       if (s != buf)
              *(s-1) = '\\n';
       return (s - buf);
}
```
/sys/power/autosleep
--------------------

autosleep节点同样定义在kernel/power/main.c，其基本结构与state基本相似。当从用户空间态读取该节点时，返回全局变量autosleep_state的值对应的字符串（"mem", "standby", "freeze"或者”disk”）。 而写该节点时，处理与/sys/power/state的处理则稍有不同：
```C
static ssize_t autosleep_store(struct kobject *kobj,struct kobj_attribute *attr,const char *buf, size_t n)
{
       suspend_state_t state = decode_state(buf, n);
       int error;
       if (state == PM_SUSPEND_ON && strcmp(buf, "off") && strcmp(buf, "off\\n"))
              return -EINVAL;
       error = pm_autosleep_set_state(state); //将suspend_state_t类型的状态传入供处理
       return error ? error : n;
}

int pm_autosleep_set_state(suspend_state_t state)
{
       __pm_stay_awake(autosleep_ws);  // 避免系统进入休眠状态（这里传入全局变量，autosleep_ws，表明是autosleep wakeup source）
       mutex_lock(&autosleep_lock);
       autosleep_state = state;         // 将传入的状态赋值给全局变量autosleep_state
       __pm_relax(autosleep_ws);       // 释放休眠锁
       if (state > PM_SUSPEND_ON) {
              pm_wakep_autosleep_enabled(true); //对系统中注册的所有wakeup source，均设置其中的autosleep成员为true，并更新start_prevent_time
              queue_up_suspend_work(); //将autosleep例程加入到kernel的workqueue进行调度
       } else {
              pm_wakep_autosleep_enabled(false); //对系统中注册的所有wakeup source，均设置其中的autosleep成员为false
       }
       mutex_unlock(&autosleep_lock);
       return 0;
}
```
/sys/power/wake_lock & wake_unlock
------------------------------------

wake_lock和wake_unlock为用户空间态拿睡眠锁与释放睡眠锁的节点。不出意料，它们还是定义在kernel/power/main.c。Kernel对这两个节点的读取处理几乎一致，以wake_lock的读取为例，对它的读取打印系统中所有激活的wakeup source。
```C
static ssize_t wake_lock_show(struct kobject *kobj,struct kobj_attribute *attr,char *buf)
{
       return pm_show_wakelocks(buf, false);
}
```
当从用户空间态写这两个节点时，内核处理则不相同。写wake_lock和wake_unlock的处理分别如下：
```C
static ssize_t wake_lock_store(struct kobject *kobj,struct kobj_attribute *attr,const char *buf, size_t n)
{
       int error = pm_wake_lock(buf); //buf内为拿休眠锁的wakeup_source的名字，将该wakeup_source注册到内核中
       return error ? error : n;
}

static ssize_t wake_unlock_store(struct kobject *kobj,struct kobj_attribute *attr,const char *buf, size_t n)
{
       int error = pm_wake_unlock(buf); //buf内为释放休眠锁的wakeup_source的名字，告诉内核释放该wakeup_source拿的休眠锁
       return error ? error : n;
}
```
Autosleep代码分析
=============

Autosleep其实为一个内核的一个workqueue，其定义在kernel/power/autosleep.c，核心处理函数为try_to_suspend。代码如下：
```C
static DECLARE_WORK(suspend_work, try_to_suspend);
static void try_to_suspend(struct work_struct *work)
{
       unsigned int initial_count, final_count;
       if (!pm_get_wakeup_count(&initial_count, true))
              goto out;
       mutex_lock(&autosleep_lock);
       if (!pm_save_wakeup_count(initial_count) ||
              system_state != SYSTEM_RUNNING) {
              mutex_unlock(&autosleep_lock);
              goto out;
       }
       if (autosleep_state == PM_SUSPEND_ON) {
              mutex_unlock(&autosleep_lock);
              return;
       }
       if (autosleep_state >= PM_SUSPEND_MAX)
              hibernate();
       else
              pm_suspend(autosleep_state);

       mutex_unlock(&autosleep_lock);
       if (!pm_get_wakeup_count(&final_count, false))
              goto out;
       /*
        * If the wakeup occured for an unknown reason, wait to prevent the
        * system from trying to suspend and waking up in a tight loop.
        */

       if (final_count == initial_count)
              schedule_timeout_uninterruptible(HZ / 2);
 out:
       queue_up_suspend_work();

}
```
**第5,6行(pm_get_wakeup_count)**，读取wakeup source的总数到initial_count供函数后续使用。同时判断若当前激活的wakeup source的数量大于0（表示有人持锁），则跳出函数执行。 **第8-12行(pm_save_wakeup_count)**，再次检查，如果执行期间wakeup source数量有变化、或者持锁的wakeup source数量大于0，或者当前系统已经不在休眠状态，则跳出执行。 **第13-16行(autosleep_state == PM_SUSPEND_ON)**，若自动休眠状态的等级为非休眠，则跳出执行。 **第17-20行**，进入到系统休眠。 **第22-31行**，从休眠跳出后，马上检查系统的wakeup_source数量。若wakeup_source总数量没有变化，会让出该线程执行，让内核重新调度。从注释的解释分析，这里的作用主要是因为从休眠中跳出，但是没有新的wakeup source产生，为了避免系统再次进入到休眠。所以让出系统调度，让造成休眠跳出的系统模块得以注册新的wakeup source。 **第33行**，将try_to_suspend加入workqueue，等待下次调度。