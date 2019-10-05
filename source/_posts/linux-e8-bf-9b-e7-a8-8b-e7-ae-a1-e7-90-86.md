---
title: Linux进程管理
tags:
  - Linux
url: 2595.html
id: 2595
categories:
  - L&amp;H Site
  - Linux进程管理
date: 2019-09-15 09:01:02
---

回顾下操作系统概念：现代计算机往往都是“同时”运行多个任务。系统若只有一个处理器，那么给定时刻只可能有一个任务在执行。而操作系统通过进程管理和调度，切换正在执行的任务，是用户在感官上认为计算机是并行执行多个任务。当然，若是多处理器系统，真正同时执行的任务可以达到处理器的数目。

内核进行进程管理的主要解决的问题：

*   任务有轻重缓急之分，需要可以根据任务的紧急程度给予任务不同的执行优先级和时间。同时尽可能保证任务执行的公平性。

### 进程状态

在Linux系统中，我们所讲的任务即进程。进程调度，就是根据当前系统的运行状况，对进程状态的切换。

Linux中进程主要有如下状态：

*   **运行**：该进程此刻正在执行。
*   **等待**：进程能够运行，但没有得到许可，因为CPU分配给另一个进程。调度器可以在下一次任务切换时选择该进程。
*   **睡眠**：进程正在睡眠无法运行，因为它在等待一个外部事件（或某种资源）。调度器无法在下一次任务切换时选择该进程。

以上状态可以相互转换（等待-->睡眠转换除外），转换的条件主要有：

*   进程时间片用完或轮转到该进程
*   进程阻塞等待某种资源/某种资源准备好了

### 进程表示

进程用task_struct结构体来表示，定义在include/linux/sched.h如下（省略部分成员）。重点结构体成员意义注释在代码中：

struct task_struct {
#ifdef CONFIG\_THREAD\_INFO\_IN\_TASK
	struct thread\_info thread\_info;
#endif
/*进程当前状态，由sched.h中的宏定义TASK\_RUNNING~TASK\_STATE_MAX表示。
 TASK_RUNNING意味着进程处于可运行状态。这并不意味着已经实际分配了CPU。进程可能会一直等到调度器选中它。该状态确保进程可以立即运行，而无需等待外部事件。
 TASK\_INTERRUPTIBLE是针对等待某事件或其他资源的睡眠进程设置的。在内核发送信号给该进程表明事件已经发生时，进程状态变为TASK\_RUNNING，它只要调度器选中该进程即可恢复执行。
 TASK_UNINTERRUPTIBLE用于因内核指示而停用的睡眠进程。它们不能由外部信号唤醒，只能由内核亲自唤醒。
 TASK_STOPPED表示进程特意停止运行，例如，由调试器暂停。
 TASK_TRACED本来不是进程状态，用于从停止的进程中，将当前被调试的那些（使用ptrace机制）与常规的进程区分开来。
 EXIT_ZOMBIE为僵尸状态，表示进程结束时父进程未调用wait调用的进程
 EXIT_DEAD状态则是指wait系统调用已经发出，而进程完全从系统移除之前的状态。只有多个线程对同一个进程发出wait调用时，该状态才有意义。
*/
	volatile long state;
	void *stack; //进程栈指针
	atomic_t usage; //进程描述符使用计数，被置为2时，表示进程描述符正在被使用而且其相应的进程处于活动状态
	unsigned int flags; //进程标志	
	unsigned int ptrace;//进程调试跟踪相关标记
	int on_rq; //CPU可能有多个运行队列，可能会在运行队列中移动。标记表示进程目前在运行队列位置状态。
	int prio, static\_prio, normal\_prio;//静态优先级动态优先级和一般优先级，调度器使用
	unsigned int rt_priority; //实时进程运行优先级
	const struct sched\_class *sched\_class; //指向进程的调度器(完全公平调度器？实时调度器等)
	struct sched_entity se; //调度实体，每个进程就是一个调度实体。调度实体也可以是用户进程组
	struct sched\_rt\_entity rt; //实时进程调度实体
#ifdef CONFIG\_CGROUP\_SCHED
	struct task\_group *sched\_task_group;
#endif
	struct sched\_dl\_entity dl; //Deadline调度实体
#ifdef CONFIG\_PREEMPT\_NOTIFIERS
	struct hlist\_head preempt\_notifiers;//存放进程被抢占的通知函数链表
#endif
	unsigned int policy; //调度策略
	int nr\_cpus\_allowed; //允许调度的CPU数量
	cpumask\_t cpus\_allowed;//掩码图表示允许调度的CPU

	struct list_head tasks;//进程链表
	struct mm\_struct \*mm, \*active\_mm;//进程内存管理结构体
	u64 vmacache_seqnum;//虚拟地址区间缓存序列号
	struct vm\_area\_struct *vmacache\[VMACACHE_SIZE\];//虚拟地址区间缓存
	int exit_state; //以下记录进程的退出状态信息
	int exit\_code, exit\_signal;
	/\* 调度器相关标记 */
	unsigned sched\_reset\_on_fork:1;
	unsigned sched\_contributes\_to_load:1;
	unsigned sched_migrated:1;
	unsigned sched\_remote\_wakeup:1;
	unsigned :0; /* force alignment to the next boundary */
	unsigned in_execve:1; /* bit to tell LSMs we're in execve */
	unsigned in_iowait:1;
#if !defined(TIF\_RESTORE\_SIGMASK)
	unsigned restore_sigmask:1;
#endif
#ifdef CONFIG_MEMCG
	unsigned memcg\_may\_oom:1;
#ifndef CONFIG_SLOB
	unsigned memcg\_kmem\_skip_account:1;
#endif
#endif
#ifdef CONFIG\_COMPAT\_BRK
	unsigned brk_randomized:1;
#endif
#ifdef CONFIG_CGROUPS
	struct restart\_block restart\_block;
	pid_t pid; //进程全局PID
	pid_t tgid; //进程组内PID

	struct task\_struct \_\_rcu *real_parent; //真正父进程
	struct task\_struct \_\_rcu *parent; //父进程（为什么会有real_parent和parent，因为线程的存在，其父进程应为创建它的进程的父进程）
	struct list_head children;	//子进程列表
	struct list_head sibling;	//兄弟进程
	struct task\_struct *group\_leader;//对多线程程序有用，指向线程组组长
	struct list_head ptraced; //ptrace跟踪的进程链表
	struct list\_head ptrace\_entry; //所在父进程的ptrace链表

	struct pid\_link pids\[PIDTYPE\_MAX\];
	struct list\_head thread\_group; //线程组链表
	struct list\_head thread\_node; //线程节点链表
	struct completion \*vfork_done;		/\* for vfork() */
	int \_\_user *set\_child_tid;//与创建新进程相关，传回用户空间
	int \_\_user *clear\_child_tid;//与创建新进程相关，传回用户空间
//进程运行时间相关参数
	cputime_t utime, stime, utimescaled, stimescaled;
	cputime_t gtime;
	struct prev\_cputime prev\_cputime;
#ifdef CONFIG\_VIRT\_CPU\_ACCOUNTING\_GEN
	seqcount\_t vtime\_seqcount;
	unsigned long long vtime_snap;
	enum {
		VTIME_INACTIVE = 0,
		VTIME_USER,
		VTIME_SYS,
	} vtime\_snap\_whence;
#endif

#ifdef CONFIG\_NO\_HZ_FULL
	atomic\_t tick\_dep_mask;
#endif
	unsigned long nvcsw, nivcsw; /* context switch counts */
	u64 start_time;		
	u64 real\_start\_time;	//启动时间
	unsigned long min\_flt, maj\_flt;
	struct task\_cputime cputime\_expires;
	struct list\_head cpu\_timers\[3\];

//进程身份相关参数
	const struct cred \_\_rcu \*ptracer\_cred; /\* Tracer's credentials at attach */
	const struct cred \_\_rcu \*real\_cred; /\* objective and real subjective task credentials (COW) */
	const struct cred __rcu *cred;	
	char comm\[TASK\_COMM\_LEN\]; /* executable name excluding path
				     \- access with \[gs\]et\_task\_comm (which lockit with task_lock())
				     \- initialized normally by setup\_new\_exec */
	struct nameidata *nameidata; //路径相关信息
#ifdef CONFIG_SYSVIPC
//IPC参数
	struct sysv_sem sysvsem;//信号量
	struct sysv_shm sysvshm;//共享内存
#endif
#ifdef CONFIG\_DETECT\_HUNG_TASK
	unsigned long last\_switch\_count; //切换次数，用于检验进程挂起
#endif
	struct fs_struct *fs; // 进程文件系统信息 
	struct files_struct *files; // 打开文件的信息 
	struct nsproxy *nsproxy; // 命名空间 
	struct signal_struct *signal; //信号
	struct sighand_struct *sighand; //信号处理

	sigset\_t blocked, real\_blocked;
	sigset\_t saved\_sigmask;	/* restored if set\_restore\_sigmask() was used */
	struct sigpending pending;

	unsigned long sas\_ss\_sp;
	size\_t sas\_ss_size;
	unsigned sas\_ss\_flags;

	struct callback\_head *task\_works;

	struct audit\_context *audit\_context;
#ifdef CONFIG_AUDITSYSCALL
	kuid_t loginuid;
	unsigned int sessionid;
#endif
	struct seccomp seccomp;
//线程组信息
   	u32 parent\_exec\_id;
   	u32 self\_exec\_id;
//进程资源锁
	spinlock\_t alloc\_lock;
	raw\_spinlock\_t pi_lock;
	struct wake\_q\_node wake_q;
//虚拟内存状态信息
	struct reclaim\_state *reclaim\_state;
	struct backing\_dev\_info *backing\_dev\_info;
	struct io\_context *io\_context;
	unsigned long ptrace_message;
	siginfo\_t \*last\_siginfo; /\* For ptrace use.  */
	struct task\_io\_accounting ioac;
#if defined(CONFIG\_TASK\_XACCT)
	u64 acct\_rss\_mem1;	/* accumulated rss usage */
	u64 acct\_vm\_mem1;	/* accumulated virtual memory usage */
	cputime\_t acct\_timexpd;	/* stime + utime since last update */
#endif
#ifdef CONFIG_CPUSETS
	nodemask\_t mems\_allowed;	/* Protected by alloc_lock */
	seqcount\_t mems\_allowed_seq;	/* Seqence no to catch updates */
	int cpuset\_mem\_spread_rotor;
	int cpuset\_slab\_spread_rotor;
#endif
#ifdef CONFIG_CGROUPS
	/\* Control Group info protected by css\_set\_lock */
	struct css\_set \_\_rcu *cgroups;
	/\* cg\_list protected by css\_set\_lock and tsk->alloc\_lock */
	struct list\_head cg\_list;
#endif
#ifdef CONFIG_FUTEX
	struct robust\_list\_head \_\_user *robust\_list;
#ifdef CONFIG_COMPAT
	struct compat\_robust\_list\_head \_\_user *compat\_robust\_list;
#endif
	struct list\_head pi\_state_list;
	struct futex\_pi\_state *pi\_state\_cache;
#endif
#ifdef CONFIG\_PERF\_EVENTS
	struct perf\_event\_context *perf\_event\_ctxp\[perf\_nr\_task_contexts\];
	struct mutex perf\_event\_mutex;
	struct list\_head perf\_event_list;
#endif
#ifdef CONFIG\_DEBUG\_PREEMPT
	unsigned long preempt\_disable\_ip;
#endif
	struct thread_struct thread;
........
};

因为要支持各种各样的功能，task_struct已经变得非常大。不过总体上，结构体可以被划分为如下部分：

*   状态和执行信息，如待决信号、使用的二进制格式（和其他系统二进制格式的任何仿真信息）、进程ID号（ pid）、到父进程及其他有关进程的指针、优先级和程序执行有关的时间信息（例如CPU时间）。
*   有关已经分配的虚拟内存的信息。
*   进程身份凭据，如用户ID、组ID以及权限①等。可使用系统调用查询（或修改）这些数据。
*   使用的文件包含程序代码的二进制文件，以及进程所处理的所有文件的文件系统信息，这些都必须保存下来。
*   进程信息记录该进程特定于CPU的运行时间数据（该结构的其余字段与所使用的硬件无关）。
*   在与其他应用程序协作时所需的进程间通信有关的信息。
*   该进程所用的信号处理程序，用于响应到来的信号。

### 进程ID号

在task_struct结构体里，我们看到了很多进程ID相关字段，初看会很容易混淆。本节介绍Linux进程ID管理相关思想，帮助理解这些字段的含义。

乍一看进程ID管理应该比较简单：内核只需要保证分配的id不唯一，释放掉的id可以被其他新创建的进程id使用即可。但是事实并非如此，内核需要做如下考量：

*   内核有[命名空间](http://man7.org/linux/man-pages/man7/namespaces.7.html)的概念，一个进程可以出现在多个命名空间，它在不同的命名空间的id是不同的。
*   同一个进程可以有多个线程，这些线程（其实也是task_struct）共享同一个线程组id (TGID)
*   进程可以合并为进程组，而进程组又可以合并为会话（Session）。组或者会话里的进程共享相同的组id或会话id。

PID分配需要在特定的命名空间内保证id的唯一性。用来表示PID的命名空间定义为：

struct pid_namespace {
	struct kref kref;
	struct idr idr;
	struct rcu_head rcu;
	unsigned int pid_allocated;
	struct task\_struct *child\_reaper; //对应命名空间0号进程
	struct kmem\_cache *pid\_cachep;
	unsigned int level; //该命名空间的层级
	struct pid_namespace *parent; //命名空间上级的指针
#ifdef CONFIG\_PROC\_FS
	struct vfsmount *proc_mnt;
	struct dentry *proc_self;
	struct dentry *proc\_thread\_self;
#endif
#ifdef CONFIG\_BSD\_PROCESS_ACCT
	struct fs_pin *bacct;
#endif
	struct user\_namespace *user\_ns;
	struct ucounts *ucounts;
	struct work\_struct proc\_work;
	kgid\_t pid\_gid;
	int hide_pid;
	int reboot;	/* group exit code if this pidns was rebooted */
	struct ns_common ns;
} \_\_randomize\_layout

而内核管理命名空间内的pid也主要围绕两个数据结构：

struct upid {
	int nr; //id
	struct pid_namespace *ns; //指向所在命名空间
};
struct pid
{
	atomic_t count;  //pid使用数量
	unsigned int level; //层级数量
	struct hlist\_head tasks\[PIDTYPE\_MAX\]; //对应每种类别命名空间进程的指针
	struct rcu_head rcu;
	struct upid numbers\[1\]; //每级的pid
};

其中PIDTYPE_MAX为pid类别的枚举类型最大值，具体该枚举类型定义如下：

enum pid_type
{
	PIDTYPE_PID,
	PIDTYPE_TGID,
	PIDTYPE_PGID,
	PIDTYPE_SID,
	PIDTYPE_MAX,
};

除此之外，task_struct还保留了两个pid，分别为：

*   pid: 初使命名空间（即init进程所在空间）中该进程的全局ID号
*   tgid：初使命名空间中该进程的线程组ID号，若该进程非多线程进程，则值与pid相同。

一张图表示task_struct中进程id的相互关联：

![](https://i0.wp.com/l2h.site/wp-content/uploads/2019/09/1.png?fit=810%2C467&ssl=1)

pid数据结构关系图（引用自《深入理解linux内核架构》）

注意，上图结构为2.6版内核中数据结构。新版内核（截至目前应该是5.）对结构会有部分调整，但总体管理方式和数据结构间关联未变。

### 进程间关系

进程可以有子进程，其子进程链表用task\_struct的children元素表示。一个子进程和父进程的其他子进程互为兄弟进程，通过task\_struct的sibling元素相互关联。

### 总结

本文介绍了linux内核管理的基本概念，以及相应的数据结构。后续介绍会包含进程调度基本架构和思想。