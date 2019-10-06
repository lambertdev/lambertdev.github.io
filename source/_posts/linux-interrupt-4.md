---
title: Linux中断(2)
tags:
  - Linux
  - 中断
url: 1563.html
id: 1563
categories:
  - Linux
  - Linux中断
date: 2019-01-05 23:50:57
---

前言
--

[前文](https://l2h.site/2019/01/01/linux-interrupt-3/)介绍了Linux中断的软件架构，以及其中的重要数据结构。本文介绍中断子系统的一些基本流程。本文组织如下：

1.  初始化流程
2.  中断处理流程

![](http://pic.l2h.site/l2hsiteLinux Interrupt-2.png)

初始化流程
-----

Linux中断的初始化分为以下几个部分：

*   系统初始化
*   设备驱动初始化

### 中断子系统初始化

![](http://pic.l2h.site/l2hsiteirq_start_kernel3.jpg)

中断初始化流程如上图，初始化从start_kernel()开始：

1.  设定early_boot_irq_disabled标志为真，表示系统正在初始化过程中，需要关闭CPU中断。
2.  关闭CPU中断，local_irq_disable()函数根据CPU Arch来执行不同CPU体系结构的关终端代码。这里特别说明一下，另外一个函数local_irq_save也是做同样的事情，只是关闭中断前会保存一下中断的标志状态，供打开中断时恢复。
3.  初始化中断向量表。对ARM来讲直接返回。
4.  初始化中断描述符, **_early_irq_init()_**
5.  初始化中断控制器, **_init_IRQ()_**
6.  初始化软中断，会在之后的文章中介绍
7.  开CPU中断

#### early_irq_init()

early_irq_init有两种定义，本质上差异不大，根据Linux编译选项**_CONFIG_SPARSE_IRQ_**定义与否来决定走哪条路。前文中有简单带过，该编译选项的主要作用是区分中断描述符具体是使用基树还是使用线性管理。本文以基树管理方式来对early_irq_init进行介绍，代码如下：
```C
int __init early_irq_init(void) {
	int i, initcnt, node = first_online_node;
	struct irq_desc *desc;

	init_irq_default_affinity();               --- 1
	initcnt = arch_probe_nr_irqs();            --- 2
	printk(KERN_INFO "NR_IRQS:%d nr_irqs:%d %d\\n", NR_IRQS, nr_irqs, initcnt);

	if (WARN_ON(nr_irqs > IRQ_BITMAP_BITS))
		nr_irqs = IRQ_BITMAP_BITS;
	if (WARN_ON(initcnt > IRQ_BITMAP_BITS))
		initcnt = IRQ_BITMAP_BITS;         ---3
	if (initcnt > nr_irqs)
		nr_irqs = initcnt;                 ---4
	for (i = 0; i < initcnt; i++) {            
		desc = alloc_desc(i, node, NULL);  ---5
		set_bit(i, allocated_irqs);        
		irq_insert_desc(i, desc);          ---6
	}
	return arch_early_irq_init();
}
```
1. 初始化IRQ描述符默认分配的IRQ Affinity（对多核生效），之后初始化IRQ描述符时，对应affinity的默认值从该值拷贝
```C
static void __init init_irq_default_affinity(void){
	alloc_cpumask_var(&irq_default_affinity, GFP_NOWAIT);
	cpumask_setall(irq_default_affinity);
}
bool alloc_cpumask_var(cpumask_var_t *mask, gfp_t flags)
{
	return alloc_cpumask_var_node(mask, flags, NUMA_NO_NODE);
}
bool alloc_cpumask_var_node(cpumask_var_t *mask, gfp_t flags, int node)
{
	*mask = kmalloc_node(cpumask_size(), flags, node);
#ifdef CONFIG_DEBUG_PER_CPU_MAPS
	if (!*mask) {
		printk(KERN_ERR "=> alloc_cpumask_var: failed!\\n");
		dump_stack();
	}
#endif
	return *mask != NULL;
}
```
2. 获取CPU arch的 irq数量。此处是从默认的struct machine_desc（对应嵌入式板级操作集合）的nr_irq字段得到 -- 嵌入式设备的机器设定大多各不相同，因此使用者需要定义自己的  
machine_desc来描述一些设备的基本信息或者回调函数，这个结构体在更早的初始化过程中建立起来。关于machine_desc的描述可以到[百度搜索](https://www.baidu.com/s?wd=struct%20machine_desc)。

3. 确认IRQ数量，如果超过IRQ Bitmap的位数，则发出Warning。

4. nr_irqs 设置为initcnt

5. 根据irq数据初始化中断描述符，并将它们初始化
```C
static struct irq_desc *alloc_desc(int irq, int node, struct module *owner)
{
	struct irq_desc *desc;
	gfp_t gfp = GFP_KERNEL;
	desc = kzalloc_node(sizeof(*desc), gfp, node); //分配中断描述符
	if (!desc)
		return NULL;
	/* allocate based on nr_cpu_ids */
	desc->kstat_irqs = alloc_percpu(unsigned int);
	if (!desc->kstat_irqs)
		goto err_desc;
	if (alloc_masks(desc, gfp, node))   //分配该中断的affinity，并初始化设定为第一步所设置的affinity
		goto err_kstat;

	raw_spin_lock_init(&desc->lock); //初始化中断描述符Spinlock用于并发保护
	lockdep_set_class(&desc->lock, &irq_desc_lock_class);
	desc_set_defaults(irq, desc, node, owner); //初始化中断描述符其他字段
	return desc;
err_kstat:
	free_percpu(desc->kstat_irqs);
err_desc:
	kfree(desc);
	return NULL;
}
static void desc_set_defaults(unsigned int irq, struct irq_desc *desc, int node, struct module *owner)
{
	int cpu;
	desc->irq_common_data.handler_data = NULL;
	desc->irq_common_data.msi_desc = NULL;
	desc->irq_data.common = &desc->irq_common_data;
	desc->irq_data.irq = irq;
........
	desc->owner = owner;
	for_each_possible_cpu(cpu)
		*per_cpu_ptr(desc->kstat_irqs, cpu) = 0;
	desc_smp_init(desc, node);
}
```
6. 将中断描述符插入到中断描述符基树 。 注意early_irq_init()函数执行前start_kernel做了基树使用的必要初始化radix_tree_init() 。基树可以在中断值离散时能极大地节约空间，且查找效率也比较高。更多基树的介绍见[Wikipedia](https://en.wikipedia.org/wiki/Radix_tree)。
```C
static RADIX_TREE(irq_desc_tree, GFP_KERNEL);
static void irq_insert_desc(unsigned int irq, struct irq_desc *desc)
{
	radix_tree_insert(&irq_desc_tree, irq, desc);
}
```
#### init_IRQ()

init_IRQ为执行与所使用CPU架构对应的中断初始化流程。我们可以查看Linux源码树，arch/***/kernel/irq.c中均对此函数有做了定义。本文以arch/arm64/kernel/irq.c中的定义为例进行介绍。代码如下：
```C
//in arch/arm64/kernel/irq.c
void __init init_IRQ(void)
{
	irqchip_init();
        ...
}
//in drivers/irqchip/irqchip.c
extern struct of_device_id __irqchip_of_table[];
void __init irqchip_init(void)
{
	of_irq_init(__irqchip_of_table);                          --- 1
	acpi_probe_device_table(irqchip);
}
```
上面代码要作用为根据DTS定义，去Linux IRQ Chip数组里匹配IRQ Chip，并初始化IRQ CHIP。这里内核使用了链接上的一些小技巧，将__irqchip_of_table定义为kernel elf的段(见arch/arm64/kernel/vmlinux.lds)。而Linux编译过程中各个IRQ chip将自己声明在该段中。以gic为例：
```C
//in drivers/irqchip/irq-gic-v3.c
IRQCHIP_DECLARE(gic_v3, "arm,gic-v3", gic_of_init);
//逐步展开
#define IRQCHIP_DECLARE(name, compat, fn) OF_DECLARE_2(irqchip, name, compat, fn)
#define OF_DECLARE_2(table, name, compat, fn) \
		_OF_DECLARE(table, name, compat, fn, of_init_fn_2)

#define _OF_DECLARE(table, name, compat, fn, fn_type)			\
	static const struct of_device_id __of_table_##name		\
		__used __section(__##table##_of_table)			\
		 = { .compatible = compat,				\
		     .data = (fn == (fn_type)NULL) ? fn : fn  }
```
上述宏定义逐步展开得到：
```C
static const struct of_device_id __of_table_gic_v3 __used __section(__irqchip_of_table) =
{ .compatible= "arm,gic-v3"
  .data =  gic_of_init
}
```
展开后可以清晰地看到，其实IRQCHIP_DECLARE就是将gic_v3这颗IRQ中断控制器的一些基本信息定义到 __irqchip_of_table中。这样在of_irq_init过程中就能进行匹配查找。我们回过头来继续分析of_irq_init()：
```C
void __init of_irq_init(const struct of_device_id *matches)
{
	struct device_node *np, *parent = NULL;
	struct of_intc_desc *desc, *temp_desc;
	struct list_head intc_desc_list, intc_parent_list;

	INIT_LIST_HEAD(&intc_desc_list);
	INIT_LIST_HEAD(&intc_parent_list);

	for_each_matching_node(np, matches) { -----------1
		if (!of_find_property(np, "interrupt-controller", NULL) ||
				!of_device_is_available(np))
			continue;
		desc = kzalloc(sizeof(*desc), GFP_KERNEL);
		if (WARN_ON(!desc)) {
			of_node_put(np);
			goto err;
		}

		desc->dev = of_node_get(np);
		desc->interrupt_parent = of_irq_find_parent(np);
		if (desc->interrupt_parent == np)
			desc->interrupt_parent = NULL;
		list_add_tail(&desc->list, &intc_desc_list);
	}
        
	while (!list_empty(&intc_desc_list)) { -----------2
		list_for_each_entry_safe(desc, temp_desc, &intc_desc_list, list) {
			const struct of_device_id *match;
			int ret;
			of_irq_init_cb_t irq_init_cb;

			if (desc->interrupt_parent != parent)
				continue;

			list_del(&desc->list);
			match = of_match_node(matches, desc->dev);
			if (WARN(!match->data,
			    "of_irq_init: no init function for %s\\n",
			    match->compatible)) {
				kfree(desc);
				continue;
			}

			pr_debug("of_irq_init: init %s @ %p, parent %p\\n",
				 match->compatible,
				 desc->dev, desc->interrupt_parent);
			irq_init_cb = (of_irq_init_cb_t)match->data;
			ret = irq_init_cb(desc->dev, desc->interrupt_parent);                                  ------ 3
			if (ret) {
				kfree(desc);
				continue;
			}
			list_add_tail(&desc->list, &intc_parent_list);
		}

		desc = list_first_entry_or_null(&intc_parent_list,
						typeof(*desc), list);
		if (!desc) {
			pr_err("of_irq_init: children remain, but no parents\\n");
			break;
		}
		list_del(&desc->list);
		parent = desc->dev;
		kfree(desc);
	}
           .........
}
```
1.上述函数标号为1的循环，主要是将DTS中定义的所有中断控制器加到intc_desc_list。 可以这样做的前提是，start_kernel初期内核已经做了DTS的初步解析。

2. 循环2，则是将所有DTS定义的中断控制器，去和编译时 **___irqchip_of_table_**里的定义中断控制器设备做匹配。

3. 如果匹配到，则执行对应irqchip项的**data**字段指向的回调函数。对前面的例子，gic_v3来讲，就是执行_**gic_of_init**_。关于_**gic_of_init**_, 本文不做深入解析。大家可以到drivers/irqchip/irq-gic-v3.c去找到对应函数。不过值得一提的是，如[Linux中断(1)](https://l2h.site/2019/01/01/linux-interrupt-3/)所讲，每个中断控制器属于一个中断控制域。GIC也不例外，初始化过程中，会去向系统注册中断控制域(irq_domain)。另外也会向如下代码中对handle_arch_irq赋值。
```C
static int __init gic_of_init(struct device_node *node, struct device_node *parent)
{
	......
	set_handle_irq(gic_handle_irq);
        ......
}
void __init set_handle_irq(void (*handle_irq)(struct pt_regs *))
{
	if (handle_arch_irq)
		return;
	handle_arch_irq = handle_irq;
}
```
### 设备驱动初始化

前边所有提到的初始化部分，是为了设备驱动可以简单有效使用中断做准备。事实上，设备驱动使用中断也的确蛮简单。只需要request_irq注册中断处理函数即可。Request IRQ定义如下：
```C
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
	    const char *name, void *dev)
{
	return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}

int request_threaded_irq(unsigned int irq, irq_handler_t handler,
			 irq_handler_t thread_fn, unsigned long irqflags,
			 const char *devname, void *dev_id)
{
	struct irqaction *action;
	struct irq_desc *desc;
	int retval;

       ..........................

	desc = irq_to_desc(irq);                                -----1
	if (!desc)
		return -EINVAL;

	if (!irq_settings_can_request(desc) ||
	    WARN_ON(irq_settings_is_per_cpu_devid(desc)))
		return -EINVAL;

	if (!handler) {
		if (!thread_fn)
			return -EINVAL;
		handler = irq_default_primary_handler;
	}

	action = kzalloc(sizeof(struct irqaction), GFP_KERNEL); -----2 start
	if (!action)
		return -ENOMEM;
	action->handler = handler;
	action->thread_fn = thread_fn;
	action->flags = irqflags;
	action->name = devname;
	action->dev_id = dev_id;                                -----2 end

	chip_bus_lock(desc); //对该中断描述符设置做访问保护
	retval = __setup_irq(irq, desc, action);                -----3
	chip_bus_sync_unlock(desc); //解除访问保护

	if (retval) {
		kfree(action->secondary);
		kfree(action);
	}
        .......
	return retval;
}
```
1. 通过IRQ编号找到对应的中断描述符

2. 分配IRQ Action, 并将对应的结构体字段赋值（由函数传入）。

3. 将 驱动的中断处理函数挂给该IRQ描述符

根据内核对request_threaded_irq的注释：当设备驱动希望内核启用一个内核线程来响应中断时，需要同时提供handler和thread_fn。尽管会使用thread_fn来响应中断，handler仍然会被调用，保证可以在中断上下文来屏蔽该设备中断。handler调用过后需要返回 IRQ_WAKE_THREAD ，这样内核才能接着呼叫thread_fn。  

中断触发流程
------

当中断子系统初始化成功，且设备驱动都注册好中断处理函数后，内核就能正常对中断进行处理了。

以ARM架构为例：当发生中断后，汇编码会调用到handle_arch_irq，即gic的gic_handle_irq函数。
```C
static void __exception_irq_entry gic_handle_irq(struct pt_regs *regs)
{
	u32 irqstat, irqnr;
	struct gic_chip_data *gic = &gic_data[0];
	void __iomem *cpu_base = gic_data_cpu_base(gic);

	do {
		irqstat = readl_relaxed(cpu_base + GIC_CPU_INTACK); 
		irqnr = irqstat & GICC_IAR_INT_ID_MASK;             -----1

		if (likely(irqnr > 15 && irqnr < 1021)) {
			if (static_key_true(&supports_deactivate))
				writel_relaxed(irqstat, cpu_base + GIC_CPU_EOI);
			handle_domain_irq(gic->domain, irqnr, regs);-----2
			continue;
		}
		if (irqnr < 16) {                                   -----3
			writel_relaxed(irqstat, cpu_base + GIC_CPU_EOI);
			if (static_key_true(&supports_deactivate))
				writel_relaxed(irqstat, cpu_base + GIC_CPU_DEACTIVATE);
#ifdef CONFIG_SMP
			smp_rmb();
			handle_IPI(irqnr, regs);
#endif
			continue;
		}
		break;
	} while (1);
}
```
1. 从中断控制器芯片中读取物理中断号

2. 如果大于15，执行handle_domain_irq
```C
int __handle_domain_irq(struct irq_domain *domain, unsigned int hwirq,
			bool lookup, struct pt_regs *regs)
{
	struct pt_regs *old_regs = set_irq_regs(regs);
	unsigned int irq = hwirq;
	int ret = 0;

	irq_enter();                                      //进入中断处理前准备

#ifdef CONFIG_IRQ_DOMAIN
	if (lookup)
		irq = irq_find_mapping(domain, hwirq);    //返回虚拟中断号
#endif

	if (unlikely(!irq || irq >= nr_irqs)) {
		ack_bad_irq(irq);
		ret = -EINVAL;
	} else {
		generic_handle_irq(irq);                  
	}

	irq_exit();                                       //退出IRQ处理
	set_irq_regs(old_regs);
	return ret;
}
int generic_handle_irq(unsigned int irq)
{
	struct irq_desc *desc = irq_to_desc(irq);
	if (!desc)
		return -EINVAL;
	generic_handle_irq_desc(desc);
	return 0;
}
static inline void generic_handle_irq_desc(struct irq_desc *desc)
{
	desc->handle_irq(desc);                          //执行中断描述符的handle_irq回调函数
}
```
中断描述符对应的handle_irq函数在GIC Chip初始化时有设定，位于gic_irq_domain_map函数内：
```C
static int gic_irq_domain_map(struct irq_domain *d, unsigned int irq,
			      irq_hw_number_t hw)
{
................
	if (hw < 32) {
................
		irq_domain_set_info(d, irq, hw, chip, d->host_data,
				    handle_percpu_devid_irq, NULL, NULL);
................
	}
	if (hw >= 32 && hw < gic_data.irq_nr) {
		irq_domain_set_info(d, irq, hw, chip, d->host_data,
				    handle_fasteoi_irq, NULL, NULL);
................
	if (hw >= 8192 && hw < GIC_ID_NR) {
................
		irq_domain_set_info(d, irq, hw, chip, d->host_data,
				    handle_fasteoi_irq, NULL, NULL);
	}
................
	return 0;
}
```
以handle_fasteoi_irq来进行分析：
```C
void handle_fasteoi_irq(struct irq_desc *desc)
{
	.........
	if (desc->istate & IRQS_ONESHOT)
		mask_irq(desc);                   //屏蔽中断,即调用对应中断芯片的irq_mask函数

	preflow_handler(desc);
	handle_irq_event(desc);                  //处理中断
        .........
}
irqreturn_t handle_irq_event_percpu(struct irq_desc *desc)
{
        .........
	struct irqaction *action = desc->action;
	while (action) {
		irqreturn_t res;

		trace_irq_handler_entry(irq, action);
		res = action->handler(irq, action->dev_id);
		trace_irq_handler_exit(irq, action, res);

		switch (res) {
		case IRQ_WAKE_THREAD:
			if (unlikely(!action->thread_fn)) {
				warn_no_thread(irq, action);
				break;
			}

			__irq_wake_thread(desc, action);
		case IRQ_HANDLED:
			flags |= action->flags;
			break;
		default:
			break;
		}

		retval |= res;
		action = action->next;
	}
        .........
	return retval;
}
```
对同一个中断来讲，是可以注册多个中断处理函数的。即request_irq对同一个中断号执行多次。这些handler会被挂到对应中断描述的action链表中。上述函数即依次执行对应该中断号的irqaction中handler（即设备request_irq时注册的handler）。

3. 0-15中断号是ARM多核之间用来相互沟通的SGI消息。本文不做分析。

### irq_enter()与irq_exit()

irq_enter和irq_exit是中断处理的重要函数，前一节的handle_domain_irq函数中可以看到他们分别在执行对应中断描述符的handler前后被调用。本节对这两个函数做重点分析：
```C
void irq_enter(void)
{
	rcu_irq_enter();                               
	if (is_idle_task(current) && !in_interrupt()) {-----1
h_disable();
		local_bh_disable();
                tick_irq_enter();    
		_local_bh_enable();                    
	}

	__irq_enter();              
}

#define __irq_enter()					\  
	do {						\
		account_irq_enter_time(current);	\
		preempt_count_add(HARDIRQ_OFFSET);	\ ----2
		trace_hardirq_enter();			\  
	} while (0)

void irq_exit(void)
{
#ifndef __ARCH_IRQ_EXIT_IRQS_DISABLED
	local_irq_disable();
#else
	WARN_ON_ONCE(!irqs_disabled());
#endif

	account_irq_exit_time(current);
	preempt_count_sub(HARDIRQ_OFFSET);
	if (!in_interrupt() && local_softirq_pending())
		invoke_softirq();                   ------3

	tick_irq_exit();
	rcu_irq_exit();
	trace_hardirq_exit(); /* must be last! */
}
```
1. 如果当前是在idle状态，且不在中断上下文中，则通知中断状态下发生了idle

2. irq_enter() 最重要的步骤就是__irq_enter(),本函数的主要作用就是关闭抢占，避免执行中断处理函数时，CPU被调度到执行其他事情。

3. 在执行完设备驱动的中断handler，开始执行irq_exit()，该函数主要的作用就是查看系统中是否有未执行的软中断，若有则呼叫invoke_softirq执行系统中的软中断。此时中断的上半部执行结束，开始执行中断下半部。

待续
--

下一篇文章，将介绍中断下半部的主要的实现方式：

*   软中断（Soft IRQ）
*   Workqueue