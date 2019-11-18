---
title: Zephyr线程管理 - 数据结构与API
s: 20191117-Zephyr-Thread-2
date: 2019-11-16 22:16:35
tags:
categories:
  - RTOS
---

本文介绍和Zephyr线程的数据结构，及相应的API。  

## 数据结构

Zephyr数据结构使用k_thread定义，如下所示：

```C
struct k_thread {
	struct _thread_base base;
	struct _callee_saved callee_saved;
	void *init_data;
	void (*fn_abort)(void);

#if defined(CONFIG_THREAD_MONITOR)
	struct __thread_entry entry;
	struct k_thread *next_thread;
#endif
#if defined(CONFIG_THREAD_NAME)
	char name[CONFIG_THREAD_MAX_NAME_LEN];
#endif
#ifdef CONFIG_THREAD_CUSTOM_DATA
	void *custom_data;
#endif
#ifdef CONFIG_THREAD_USERSPACE_LOCAL_DATA
	struct _thread_userspace_local_data *userspace_local_data;
#endif
#ifdef CONFIG_ERRNO
#ifndef CONFIG_USERSPACE
	int errno_var;
#endif
#endif

#if defined(CONFIG_THREAD_STACK_INFO)
	struct _thread_stack_info stack_info;
#endif /* CONFIG_THREAD_STACK_INFO */
#if defined(CONFIG_USERSPACE)
	struct _mem_domain_info mem_domain_info;
	k_thread_stack_t *stack_obj;
#endif /* CONFIG_USERSPACE */
#if defined(CONFIG_USE_SWITCH)
	int swap_retval;
	void *switch_handle;
#endif
	struct k_mem_pool *resource_pool;
	struct _thread_arch arch;
};
```

参数说明：

+ **base**： 存储Thread的基础调度信息结构体。具体如下：
  + **qnode_dlist/qnode_rb**: 指向线程在等待/就绪队列的位置
  + **pended_on**:指向线程所在的等待队列（仅对红黑树等待队列有效）
  + **user_options**:线程参数
  + **thread_state**:线程状态
  + **union(prio/sched_locked/preemt)**: 线程的抢占优先级相关参数
  + **order_key**:被调度器使用来比对优先级
  + SMP相关参数先忽略
  + **timeout**:指向超时队列中该线程的位置

```C
struct _thread_base {
	union {
		sys_dnode_t qnode_dlist;
		struct rbnode qnode_rb;
	};
	_wait_q_t *pended_on;
	u8_t user_options;
	u8_t thread_state;
	union {
		struct {
#if __BYTE_ORDER__ == __ORDER_BIG_ENDIAN__
			u8_t sched_locked;
			s8_t prio;
#else /* LITTLE and PDP */
			s8_t prio;
			u8_t sched_locked;
#endif
		};
		u16_t preempt;
	};
#ifdef CONFIG_SCHED_DEADLINE
	int prio_deadline;
#endif
	u32_t order_key;
/*SMP 相关*/
	void *swap_data;
#ifdef CONFIG_SYS_CLOCK_EXISTS
	struct _timeout timeout;
#endif
};
```

+ **_callee_saved**: 与平台相关的一些参数，主要保存一些线程相关寄存器信息。
+ **init_data**: 线程静态初始化数据
+ **fn_abort**: 线程取消回调函数
+ **entry**:Thread入口函数
+ **next_thread**: Thread链表中下一个线程指针
+ **name**: Thread名称
+ **custom_data**: Thread自定义数据
+ **userspace_local_data**:用户空间态数据
+ **errno_var**: 错误编号（类似Linux的Errno）
+ **stack_info**: 栈信息，主要记录栈的开始地址与大小
+ **mem_domain_info**: 线程内存域信息（用户空间态线程使用）
+ **stack_obj**: 线程栈地址
+ **swap_retval**: TBD
+ **switch_handle**: TBD
+ **resource_pool**:线程所占用的资源池
+ **arch**: 体系结构相关的线程数据。因为体系结构不同，所占大小也不相同，为保证其他参数的位置相同，Zephyr将此参数置于线程末尾

## API

Zephyr定义了一系列的API来使用线程：

### 线程创建

线程创建的函数定义如下：

```C
K_THREAD_DEFINE(my_tid, MY_STACK_SIZE, my_entry_point, NULL, NULL, NULL, MY_PRIORITY, 0, K_NO_WAIT);
k_tid_t k_thread_create(structk_thread *new_thread, k_thread_stack_t *stack, size_t stack_size, k_thread_entry_t entry, void *p1, void *p2, void *p3, int prio, u32_t options, s32_t delay)
```

前者主要用于静态定义线程，而后者主要用于运行时创建线程。线程的静态定义主要就是定义如下结构体（参数明确，不做一一解释）：

```C
struct _static_thread_data {
	struct k_thread *init_thread;
	k_thread_stack_t *init_stack;
	unsigned int init_stack_size;
	k_thread_entry_t init_entry;
	void *init_p1;
	void *init_p2;
	void *init_p3;
	int init_prio;
	u32_t init_options;
	s32_t init_delay;
	void (*init_abort)(void);
	const char *init_name;
};
```

k_thread_create的源码如下：

```C
k_tid_t z_impl_k_thread_create(struct k_thread *new_thread,      //代码注解1
			      k_thread_stack_t *stack,
			      size_t stack_size, k_thread_entry_t entry,
			      void *p1, void *p2, void *p3,
			      int prio, u32_t options, s32_t delay)
{
	__ASSERT(!z_arch_is_in_isr(), "Threads may not be created in ISRs"); //代码注解2

	z_setup_new_thread(new_thread, stack, stack_size, entry, p1, p2, p3, prio, options, NULL); //代码注解3

	if (delay != K_FOREVER) {
		schedule_new_thread(new_thread, delay);  //代码注解4
	}

	return new_thread;
}
```

其中：

+ **代码注解1**： 这里函数名为何是z_impl_k_thread_create而不是k_thread_create。因为Zephyr在编译时使用了脚本来生成系统中API调用。所以在搜索源码时大部分时间只能搜索到z_impl_xxx形式的系统函数定义。不过编译连接前，是可以看得到不带*z_impl_*形式的API，最终调用的即是z_impl_k_thread_create
+ **代码注解2**：中断上下文不能创建线程，因为创建线程会需要申请系统资源，也会引起重新调度，带来不可预料的后果
+ **代码注解3**：创建新线程
+ **代码注解4**：将线程加入调度

z_setup_new_thread定义如下(为方便分析，代码做一定简化)：

```C
void z_setup_new_thread(struct k_thread *new_thread, k_thread_stack_t *stack, size_t stack_size, k_thread_entry_t entry, void *p1, void *p2, void *p3, int prio, u32_t options, const char *name)
{
	stack_size = adjust_stack_size(stack_size); //代码注解1

	z_arch_new_thread(new_thread, stack, stack_size, entry, p1, p2, p3, prio, options); //代码注解2

#ifdef CONFIG_THREAD_MONITOR
	new_thread->entry.pEntry = entry;
	new_thread->entry.parameter1 = p1;
	new_thread->entry.parameter2 = p2;
	new_thread->entry.parameter3 = p3;

	k_spinlock_key_t key = k_spin_lock(&lock);

	new_thread->next_thread = _kernel.threads;
	_kernel.threads = new_thread;
	k_spin_unlock(&lock, key); //代码注解3
#endif
#ifdef CONFIG_THREAD_NAME
	if (name != NULL) {
		strncpy(new_thread->name, name,
			CONFIG_THREAD_MAX_NAME_LEN - 1);
		new_thread->name[CONFIG_THREAD_MAX_NAME_LEN - 1] = '\0'; //代码注解4
	}
#endif

#ifdef CONFIG_ARCH_HAS_CUSTOM_SWAP_TO_MAIN
	if (!_current) {
		new_thread->resource_pool = NULL;
		return;
	}
#endif
	new_thread->resource_pool = _current->resource_pool; //代码注解5
	sys_trace_thread_create(new_thread);
}

```

+ **代码注解1**: 修正栈大小，主要为线程栈地址随机化功能使用（算是简单的地址随机化方案，感兴趣大家可以搜索一下ASLR相关资料），目的为的是防止破解攻击
+ **代码注解2**: 执行体系结构相关的线程创建过程（本文接下来以ARM Cortex-M为例介绍）
+ **代码注解3/代码注解4**: 线程一些公共参数的初始化，用于系统监视器监视线程状态。
+ **代码注解5**: 继承父线程的资源池。

其中z_arch_new_thread代码如下(省去部分代码)：

```C
void z_arch_new_thread(struct k_thread *thread, k_thread_stack_t *stack,  size_t stackSize, k_thread_entry_t pEntry,  void *parameter1, void *parameter2, void *parameter3, int priority, unsigned int options)
{
	char *pStackMem = Z_THREAD_STACK_BUFFER(stack);
	char *stackEnd;

	u32_t top_of_stack_offset = 0U;

	Z_ASSERT_VALID_PRIO(priority, pEntry);


#if defined(CONFIG_MPU_REQUIRES_POWER_OF_TWO_ALIGNMENT) \
	&& defined(CONFIG_USERSPACE)
	stackSize -= MPU_GUARD_ALIGN_AND_SIZE;
#endif

#if defined(CONFIG_FLOAT) && defined(CONFIG_FP_SHARING) \
	&& defined(CONFIG_MPU_STACK_GUARD)
	if ((options & K_FP_REGS) != 0) {
		pStackMem += MPU_GUARD_ALIGN_AND_SIZE_FLOAT
			- MPU_GUARD_ALIGN_AND_SIZE;
		stackSize -= MPU_GUARD_ALIGN_AND_SIZE_FLOAT
			- MPU_GUARD_ALIGN_AND_SIZE;
	}
#endif
	stackEnd = pStackMem + stackSize;

	struct __esf *pInitCtx;

	z_new_thread_init(thread, pStackMem, stackSize, priority,  options);

	pInitCtx = (struct __esf *)(STACK_ROUND_DOWN(stackEnd -
		(char *)top_of_stack_offset - sizeof(struct __basic_sf)));
        
	pInitCtx->basic.pc = (u32_t)z_thread_entry;

#if defined(CONFIG_CPU_CORTEX_M)
	pInitCtx->basic.pc &= 0xfffffffe;
#endif
	pInitCtx->basic.a1 = (u32_t)pEntry;
	pInitCtx->basic.a2 = (u32_t)parameter1;
	pInitCtx->basic.a3 = (u32_t)parameter2;
	pInitCtx->basic.a4 = (u32_t)parameter3;
	pInitCtx->basic.xpsr = 0x01000000UL; 

	thread->callee_saved.psp = (u32_t)pInitCtx;
	thread->arch.basepri = 0;

#if defined(CONFIG_USERSPACE) || defined(CONFIG_FP_SHARING)
	thread->arch.mode = 0;
#endif
}

static ALWAYS_INLINE void z_new_thread_init(struct k_thread *thread, char *pStack, size_t stackSize, int prio, unsigned int options)
{
#ifdef CONFIG_INIT_STACKS
	memset(pStack, 0xaa, stackSize);
#endif
#ifdef CONFIG_STACK_SENTINEL
	*((u32_t *)pStack) = STACK_SENTINEL;
#endif /* CONFIG_STACK_SENTINEL */
	z_init_thread_base(&thread->base, prio, _THREAD_PRESTART, options);
	thread->init_data = NULL;
	thread->fn_abort = NULL;
#ifdef CONFIG_THREAD_CUSTOM_DATA
	thread->custom_data = NULL;
#endif
#ifdef CONFIG_THREAD_NAME
	thread->name[0] = '\0';
#endif
#if defined(CONFIG_THREAD_STACK_INFO)
	thread->stack_info.start = (uintptr_t)pStack;
	thread->stack_info.size = (u32_t)stackSize;
#endif /* CONFIG_THREAD_STACK_INFO */
}
```

以上初始化过程执行后，线程的栈空间如图所示：

![http://pic.l2h.site/QQMail_0.png](http://pic.l2h.site/QQMail_0.png)

以上为线程的初始化过程

### 线程执行

线程开始执行调用*z_impl_k_thread_start*函数。该函数也会被k_thread_create-->schedule_new_thread间接调用，其源码如下：

```C
void z_impl_k_thread_start(struct k_thread *thread)
{
	k_spinlock_key_t key = k_spin_lock(&lock); 

	if (z_has_thread_started(thread)) {
		k_spin_unlock(&lock, key);
		return;
	}

	z_mark_thread_as_started(thread); //代码注解1
	z_ready_thread(thread);  //代码注解2
	z_reschedule(&lock, key); //代码注解3
}
```

+ **代码注解1**：仅仅将thread_state标记上_THREAD_PRESTART  
+ **代码注解2**：将Thread加入调度的ready Q，关于线程调度之后文章进行介绍
+ **代码注解3**：调用系统的reschedule函数触发系统调度。对Cortex-M3来讲，将会调用系统的PENDSV系统指令，悬起系统切换异常。下图为引用自《Cortex-M3权威指南》的一张PendSV使用的示例：

![PendSV](http://pic.l2h.site/PendSV.png)

### 线程挂起

线程挂起使用*z_impl_k_thread_suspend*，代码如下：

```C
void z_impl_k_thread_suspend(struct k_thread *thread)
{
	k_spinlock_key_t key = k_spin_lock(&lock);
	z_thread_single_suspend(thread);
	sys_trace_thread_suspend(thread);

	if (thread == _current) {
		z_reschedule(&lock, key); //代码注解3
	} else {
		k_spin_unlock(&lock, key);
	}
}
void z_thread_single_suspend(struct k_thread *thread)
{
	if (z_is_thread_ready(thread)) {
		z_remove_thread_from_ready_q(thread); //代码注解1
	}
	z_mark_thread_as_suspended(thread); //代码注解2
}
```

+ **代码注解3**：若该线程正在运行，调用reschedule重新调度
+ **代码注解1**：将Thread从调度的ready Q移除
+ **代码注解2**：对线程状态做挂起标记

### 线程继续（resume）

线程Resume仅做线程的标记，不再进行分析

### 线程取消（Abort）

线程取消使用API *z_impl_k_thread_abort*,代码如下：

```C
void z_impl_k_thread_abort(k_tid_t thread)
{
	unsigned int key;

	key = irq_lock();
	__ASSERT(!(thread->base.user_options & K_ESSENTIAL),
		 "essential thread aborted");

	z_thread_single_abort(thread);
	z_thread_monitor_exit(thread);

	if (_current == thread) {
		if ((SCB->ICSR & SCB_ICSR_VECTACTIVE_Msk) == 0) {
			(void)z_swap_irqlock(key);
			CODE_UNREACHABLE;   //代码注解7
		} else {
			SCB->ICSR |= SCB_ICSR_PENDSVSET_Msk; //代码注解7
		}
	}

	z_reschedule_irqlock(key);
}
void z_thread_single_abort(struct k_thread *thread)
{
	if (thread->fn_abort != NULL) {
		thread->fn_abort();     //代码注解1
	}
	if (IS_ENABLED(CONFIG_SMP)) {
		z_sched_abort(thread);  //代码注解2
	}
	if (z_is_thread_ready(thread)) {
		z_remove_thread_from_ready_q(thread); //代码注解3
	} else {
		if (z_is_thread_pending(thread)) {
			z_unpend_thread_no_timeout(thread); //代码注解4
		}
		if (z_is_thread_timeout_active(thread)) {
			(void)z_abort_thread_timeout(thread); //代码注解5
		}
	}
	thread->base.thread_state |= _THREAD_DEAD; //代码注解6
	sys_trace_thread_abort(thread);
}
```

+ **代码注解1**：先执行该线程的退出回调函数（供应用程序做特殊处理）
+ **代码注解2**：SMP相关通知，此处不做深入分析
+ **代码注解3**：将Thread从调度的ready Q移除
+ **代码注解4**：将Thread从Pending Q也移除
+ **代码注解5**：若Thread正在等待一定超时后调度，也将其从超时列表中移除
+ **代码注解6**：对线程状态做死亡（Dead）标记
+ **代码注解7**：待添加
+ **代码注解8**：置起PendSV，进行重新调度

## 结语

以上为线程使用相关结构体、API及对应的分析。可以看出，线程相关函数的实现足够简洁明了，这也正应了Zephyr的设计思想。有任何问题，欢迎留言讨论。
