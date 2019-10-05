---
title: Linux中断(2)
tags:
  - Linux
  - 中断
url: 1563.html
id: 1563
categories:
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

1.  设定early\_boot\_irq_disabled标志为真，表示系统正在初始化过程中，需要关闭CPU中断。
2.  关闭CPU中断，local\_irq\_disable()函数根据CPU Arch来执行不同CPU体系结构的关终端代码。这里特别说明一下，另外一个函数local\_irq\_save也是做同样的事情，只是关闭中断前会保存一下中断的标志状态，供打开中断时恢复。
3.  初始化中断向量表。对ARM来讲直接返回。
4.  初始化中断描述符, **_early\_irq\_init()_**
5.  初始化中断控制器, **_init_IRQ()_**
6.  初始化软中断，会在之后的文章中介绍
7.  开CPU中断

#### early\_irq\_init()

early\_irq\_init有两种定义，本质上差异不大，根据Linux编译选项**_CONFIG\_SPARSE\_IRQ_**定义与否来决定走哪条路。前文中有简单带过，该编译选项的主要作用是区分中断描述符具体是使用基树还是使用线性管理。本文以基树管理方式来对early\_irq\_init进行介绍，代码如下：

int \_\_init early\_irq_init(void) {
	int i, initcnt, node = first\_online\_node;
	struct irq_desc *desc;

	init\_irq\_default_affinity();               --- 1
	initcnt = arch\_probe\_nr_irqs();            --- 2
	printk(KERN\_INFO "NR\_IRQS:%d nr\_irqs:%d %d\\n", NR\_IRQS, nr_irqs, initcnt);

	if (WARN\_ON(nr\_irqs > IRQ\_BITMAP\_BITS))
		nr\_irqs = IRQ\_BITMAP_BITS;
	if (WARN\_ON(initcnt > IRQ\_BITMAP_BITS))
		initcnt = IRQ\_BITMAP\_BITS;         ---3
	if (initcnt > nr_irqs)
		nr_irqs = initcnt;                 ---4
	for (i = 0; i < initcnt; i++) {            
		desc = alloc_desc(i, node, NULL);  ---5
		set\_bit(i, allocated\_irqs);        
		irq\_insert\_desc(i, desc);          ---6
	}
	return arch\_early\_irq_init();
}

1\. 初始化IRQ描述符默认分配的IRQ Affinity（对多核生效），之后初始化IRQ描述符时，对应affinity的默认值从该值拷贝

static void \_\_init init\_irq\_default\_affinity(void){
	alloc\_cpumask\_var(&irq\_default\_affinity, GFP_NOWAIT);
	cpumask\_setall(irq\_default_affinity);
}
bool alloc\_cpumask\_var(cpumask\_var\_t *mask, gfp_t flags)
{
	return alloc\_cpumask\_var\_node(mask, flags, NUMA\_NO_NODE);
}
bool alloc\_cpumask\_var\_node(cpumask\_var\_t *mask, gfp\_t flags, int node)
{
	*mask = kmalloc\_node(cpumask\_size(), flags, node);
#ifdef CONFIG\_DEBUG\_PER\_CPU\_MAPS
	if (!*mask) {
		printk(KERN\_ERR "=> alloc\_cpumask_var: failed!\\n");
		dump_stack();
	}
#endif
	return *mask != NULL;
}

2\. 获取CPU arch的 irq数量。此处是从默认的struct machine\_desc（对应嵌入式板级操作集合）的nr\_irq字段得到 -- 嵌入式设备的机器设定大多各不相同，因此使用者需要定义自己的  
machine\_desc来描述一些设备的基本信息或者回调函数，这个结构体在更早的初始化过程中建立起来。关于machine\_desc的描述可以到[百度搜索](https://www.baidu.com/s?wd=struct%20machine_desc)。

3\. 确认IRQ数量，如果超过IRQ Bitmap的位数，则发出Warning。

4\. nr_irqs 设置为initcnt

5\. 根据irq数据初始化中断描述符，并将它们初始化

static struct irq\_desc \*alloc\_desc(int irq, int node, struct module \*owner)
{
	struct irq_desc *desc;
	gfp\_t gfp = GFP\_KERNEL;
	desc = kzalloc_node(sizeof(*desc), gfp, node); //分配中断描述符
	if (!desc)
		return NULL;
	/\* allocate based on nr\_cpu\_ids */
	desc->kstat\_irqs = alloc\_percpu(unsigned int);
	if (!desc->kstat_irqs)
		goto err_desc;
	if (alloc_masks(desc, gfp, node))   //分配该中断的affinity，并初始化设定为第一步所设置的affinity
		goto err_kstat;

	raw\_spin\_lock_init(&desc->lock); //初始化中断描述符Spinlock用于并发保护
	lockdep\_set\_class(&desc->lock, &irq\_desc\_lock_class);
	desc\_set\_defaults(irq, desc, node, owner); //初始化中断描述符其他字段
	return desc;
err_kstat:
	free\_percpu(desc->kstat\_irqs);
err_desc:
	kfree(desc);
	return NULL;
}
static void desc\_set\_defaults(unsigned int irq, struct irq_desc \*desc, int node, struct module \*owner)
{
	int cpu;
	desc->irq\_common\_data.handler_data = NULL;
	desc->irq\_common\_data.msi_desc = NULL;
	desc->irq\_data.common = &desc->irq\_common_data;
	desc->irq_data.irq = irq;
........
	desc->owner = owner;
	for\_each\_possible_cpu(cpu)
		*per\_cpu\_ptr(desc->kstat_irqs, cpu) = 0;
	desc\_smp\_init(desc, node);
}

6\. 将中断描述符插入到中断描述符基树 。 注意early\_irq\_init()函数执行前start\_kernel做了基树使用的必要初始化radix\_tree_init() 。基树可以在中断值离散时能极大地节约空间，且查找效率也比较高。更多基树的介绍见[Wikipedia](https://en.wikipedia.org/wiki/Radix_tree)。

static RADIX\_TREE(irq\_desc\_tree, GFP\_KERNEL);
static void irq\_insert\_desc(unsigned int irq, struct irq_desc *desc)
{
	radix\_tree\_insert(&irq\_desc\_tree, irq, desc);
}

#### init_IRQ()

init_IRQ为执行与所使用CPU架构对应的中断初始化流程。我们可以查看Linux源码树，arch/***/kernel/irq.c中均对此函数有做了定义。本文以arch/arm64/kernel/irq.c中的定义为例进行介绍。代码如下：

//in arch/arm64/kernel/irq.c
void \_\_init init\_IRQ(void)
{
	irqchip_init();
        ...
}
//in drivers/irqchip/irqchip.c
extern struct of\_device\_id \_\_irqchip\_of_table\[\];
void \_\_init irqchip\_init(void)
{
	of\_irq\_init(\_\_irqchip\_of_table);                          --- 1
	acpi\_probe\_device_table(irqchip);
}

上面代码要作用为根据DTS定义，去Linux IRQ Chip数组里匹配IRQ Chip，并初始化IRQ CHIP。这里内核使用了链接上的一些小技巧，将\_\_irqchip\_of_table定义为kernel elf的段(见arch/arm64/kernel/vmlinux.lds)。而Linux编译过程中各个IRQ chip将自己声明在该段中。以gic为例：

//in drivers/irqchip/irq-gic-v3.c
IRQCHIP\_DECLARE(gic\_v3, "arm,gic-v3", gic\_of\_init);
//逐步展开
#define IRQCHIP\_DECLARE(name, compat, fn) OF\_DECLARE_2(irqchip, name, compat, fn)
#define OF\_DECLARE\_2(table, name, compat, fn) \
		\_OF\_DECLARE(table, name, compat, fn, of\_init\_fn_2)

#define \_OF\_DECLARE(table, name, compat, fn, fn_type)			\
	static const struct of\_device\_id \_\_of\_table_##name		\
		\_\_used \_\_section(__##table##\_of\_table)			\
		 = { .compatible = compat,				\
		     .data = (fn == (fn_type)NULL) ? fn : fn  }

上述宏定义逐步展开得到：

static const struct of\_device\_id \_\_of\_table\_gic\_v3 \_\_used \_\_section(\_\_irqchip\_of_table) =
{ .compatible= "arm,gic-v3"
  .data =  gic\_of\_init
}

展开后可以清晰地看到，其实IRQCHIP\_DECLARE就是将gic\_v3这颗IRQ中断控制器的一些基本信息定义到 \_\_irqchip\_of\_table中。这样在of\_irq\_init过程中就能进行匹配查找。我们回过头来继续分析of\_irq_init()：

void \_\_init of\_irq\_init(const struct of\_device_id *matches)
{
	struct device_node \*np, \*parent = NULL;
	struct of\_intc\_desc \*desc, \*temp_desc;
	struct list\_head intc\_desc\_list, intc\_parent_list;

	INIT\_LIST\_HEAD(&intc\_desc\_list);
	INIT\_LIST\_HEAD(&intc\_parent\_list);

	for\_each\_matching_node(np, matches) { -----------1
		if (!of\_find\_property(np, "interrupt-controller", NULL) ||
				!of\_device\_is_available(np))
			continue;
		desc = kzalloc(sizeof(*desc), GFP_KERNEL);
		if (WARN_ON(!desc)) {
			of\_node\_put(np);
			goto err;
		}

		desc->dev = of\_node\_get(np);
		desc->interrupt\_parent = of\_irq\_find\_parent(np);
		if (desc->interrupt_parent == np)
			desc->interrupt_parent = NULL;
		list\_add\_tail(&desc->list, &intc\_desc\_list);
	}
        
	while (!list\_empty(&intc\_desc_list)) { -----------2
		list\_for\_each\_entry\_safe(desc, temp\_desc, &intc\_desc_list, list) {
			const struct of\_device\_id *match;
			int ret;
			of\_irq\_init\_cb\_t irq\_init\_cb;

			if (desc->interrupt_parent != parent)
				continue;

			list_del(&desc->list);
			match = of\_match\_node(matches, desc->dev);
			if (WARN(!match->data,
			    "of\_irq\_init: no init function for %s\\n",
			    match->compatible)) {
				kfree(desc);
				continue;
			}

			pr\_debug("of\_irq_init: init %s @ %p, parent %p\\n",
				 match->compatible,
				 desc->dev, desc->interrupt_parent);
			irq\_init\_cb = (of\_irq\_init\_cb\_t)match->data;
			ret = irq\_init\_cb(desc->dev, desc->interrupt_parent);                                  ------ 3
			if (ret) {
				kfree(desc);
				continue;
			}
			list\_add\_tail(&desc->list, &intc\_parent\_list);
		}

		desc = list\_first\_entry\_or\_null(&intc\_parent\_list,
						typeof(*desc), list);
		if (!desc) {
			pr\_err("of\_irq_init: children remain, but no parents\\n");
			break;
		}
		list_del(&desc->list);
		parent = desc->dev;
		kfree(desc);
	}
           .........
}

1.上述函数标号为1的循环，主要是将DTS中定义的所有中断控制器加到intc\_desc\_list。 可以这样做的前提是，start_kernel初期内核已经做了DTS的初步解析。

2\. 循环2，则是将所有DTS定义的中断控制器，去和编译时 **_\_\_irqchip\_of_table_**里的定义中断控制器设备做匹配。

3\. 如果匹配到，则执行对应irqchip项的**data**字段指向的回调函数。对前面的例子，gic_v3来讲，就是执行_**gic\_of\_init**_。关于_**gic\_of\_init**_, 本文不做深入解析。大家可以到drivers/irqchip/irq-gic-v3.c去找到对应函数。不过值得一提的是，如[Linux中断(1)](https://l2h.site/2019/01/01/linux-interrupt-3/)所讲，每个中断控制器属于一个中断控制域。GIC也不例外，初始化过程中，会去向系统注册中断控制域(irq\_domain)。另外也会向如下代码中对handle\_arch_irq赋值。

static int \_\_init gic\_of\_init(struct device\_node \*node, struct device_node \*parent)
{
	......
	set\_handle\_irq(gic\_handle\_irq);
        ......
}
void \_\_init set\_handle\_irq(void (\*handle\_irq)(struct pt_regs \*))
{
	if (handle\_arch\_irq)
		return;
	handle\_arch\_irq = handle_irq;
}

### 设备驱动初始化

前边所有提到的初始化部分，是为了设备驱动可以简单有效使用中断做准备。事实上，设备驱动使用中断也的确蛮简单。只需要request_irq注册中断处理函数即可。Request IRQ定义如下：

static inline int \_\_must\_check
request\_irq(unsigned int irq, irq\_handler_t handler, unsigned long flags,
	    const char \*name, void \*dev)
{
	return request\_threaded\_irq(irq, handler, NULL, flags, name, dev);
}

int request\_threaded\_irq(unsigned int irq, irq\_handler\_t handler,
			 irq\_handler\_t thread_fn, unsigned long irqflags,
			 const char \*devname, void \*dev_id)
{
	struct irqaction *action;
	struct irq_desc *desc;
	int retval;

       ..........................

	desc = irq\_to\_desc(irq);                                -----1
	if (!desc)
		return -EINVAL;

	if (!irq\_settings\_can_request(desc) ||
	    WARN\_ON(irq\_settings\_is\_per\_cpu\_devid(desc)))
		return -EINVAL;

	if (!handler) {
		if (!thread_fn)
			return -EINVAL;
		handler = irq\_default\_primary_handler;
	}

	action = kzalloc(sizeof(struct irqaction), GFP_KERNEL); -----2 start
	if (!action)
		return -ENOMEM;
	action->handler = handler;
	action->thread\_fn = thread\_fn;
	action->flags = irqflags;
	action->name = devname;
	action->dev\_id = dev\_id;                                -----2 end

	chip\_bus\_lock(desc); //对该中断描述符设置做访问保护
	retval = \_\_setup\_irq(irq, desc, action);                -----3
	chip\_bus\_sync_unlock(desc); //解除访问保护

	if (retval) {
		kfree(action->secondary);
		kfree(action);
	}
        .......
	return retval;
}

1\. 通过IRQ编号找到对应的中断描述符

2\. 分配IRQ Action, 并将对应的结构体字段赋值（由函数传入）。

3\. 将 驱动的中断处理函数挂给该IRQ描述符

根据内核对request\_threaded\_irq的注释：当设备驱动希望内核启用一个内核线程来响应中断时，需要同时提供handler和thread\_fn。尽管会使用thread\_fn来响应中断，handler仍然会被调用，保证可以在中断上下文来屏蔽该设备中断。handler调用过后需要返回 IRQ\_WAKE\_THREAD ，这样内核才能接着呼叫thread_fn。  

中断触发流程
------

当中断子系统初始化成功，且设备驱动都注册好中断处理函数后，内核就能正常对中断进行处理了。

以ARM架构为例：当发生中断后，汇编码会调用到handle\_arch\_irq，即gic的gic\_handle\_irq函数。

static void \_\_exception\_irq\_entry gic\_handle\_irq(struct pt\_regs *regs)
{
	u32 irqstat, irqnr;
	struct gic\_chip\_data *gic = &gic_data\[0\];
	void \_\_iomem *cpu\_base = gic\_data\_cpu_base(gic);

	do {
		irqstat = readl\_relaxed(cpu\_base + GIC\_CPU\_INTACK); 
		irqnr = irqstat & GICC\_IAR\_INT\_ID\_MASK;             -----1

		if (likely(irqnr > 15 && irqnr < 1021)) {
			if (static\_key\_true(&supports_deactivate))
				writel\_relaxed(irqstat, cpu\_base + GIC\_CPU\_EOI);
			handle\_domain\_irq(gic->domain, irqnr, regs);-----2
			continue;
		}
		if (irqnr < 16) {                                   -----3
			writel\_relaxed(irqstat, cpu\_base + GIC\_CPU\_EOI);
			if (static\_key\_true(&supports_deactivate))
				writel\_relaxed(irqstat, cpu\_base + GIC\_CPU\_DEACTIVATE);
#ifdef CONFIG_SMP
			smp_rmb();
			handle_IPI(irqnr, regs);
#endif
			continue;
		}
		break;
	} while (1);
}

1\. 从中断控制器芯片中读取物理中断号

2\. 如果大于15，执行handle\_domain\_irq

int \_\_handle\_domain\_irq(struct irq\_domain *domain, unsigned int hwirq,
			bool lookup, struct pt_regs *regs)
{
	struct pt\_regs *old\_regs = set\_irq\_regs(regs);
	unsigned int irq = hwirq;
	int ret = 0;

	irq_enter();                                      //进入中断处理前准备

#ifdef CONFIG\_IRQ\_DOMAIN
	if (lookup)
		irq = irq\_find\_mapping(domain, hwirq);    //返回虚拟中断号
#endif

	if (unlikely(!irq || irq >= nr_irqs)) {
		ack\_bad\_irq(irq);
		ret = -EINVAL;
	} else {
		generic\_handle\_irq(irq);                  
	}

	irq_exit();                                       //退出IRQ处理
	set\_irq\_regs(old_regs);
	return ret;
}
int generic\_handle\_irq(unsigned int irq)
{
	struct irq\_desc *desc = irq\_to_desc(irq);
	if (!desc)
		return -EINVAL;
	generic\_handle\_irq_desc(desc);
	return 0;
}
static inline void generic\_handle\_irq\_desc(struct irq\_desc *desc)
{
	desc->handle\_irq(desc);                          //执行中断描述符的handle\_irq回调函数
}

中断描述符对应的handle\_irq函数在GIC Chip初始化时有设定，位于gic\_irq\_domain\_map函数内：

static int gic\_irq\_domain\_map(struct irq\_domain *d, unsigned int irq,
			      irq\_hw\_number_t hw)
{
................
	if (hw < 32) {
................
		irq\_domain\_set\_info(d, irq, hw, chip, d->host\_data,
				    handle\_percpu\_devid_irq, NULL, NULL);
................
	}
	if (hw >= 32 && hw < gic\_data.irq\_nr) {
		irq\_domain\_set\_info(d, irq, hw, chip, d->host\_data,
				    handle\_fasteoi\_irq, NULL, NULL);
................
	if (hw >= 8192 && hw < GIC\_ID\_NR) {
................
		irq\_domain\_set\_info(d, irq, hw, chip, d->host\_data,
				    handle\_fasteoi\_irq, NULL, NULL);
	}
................
	return 0;
}

以handle\_fasteoi\_irq来进行分析：

void handle\_fasteoi\_irq(struct irq_desc *desc)
{
	.........
	if (desc->istate & IRQS_ONESHOT)
		mask\_irq(desc);                   //屏蔽中断,即调用对应中断芯片的irq\_mask函数

	preflow_handler(desc);
	handle\_irq\_event(desc);                  //处理中断
        .........
}
irqreturn\_t handle\_irq\_event\_percpu(struct irq_desc *desc)
{
        .........
	struct irqaction *action = desc->action;
	while (action) {
		irqreturn_t res;

		trace\_irq\_handler_entry(irq, action);
		res = action->handler(irq, action->dev_id);
		trace\_irq\_handler_exit(irq, action, res);

		switch (res) {
		case IRQ\_WAKE\_THREAD:
			if (unlikely(!action->thread_fn)) {
				warn\_no\_thread(irq, action);
				break;
			}

			\_\_irq\_wake_thread(desc, action);
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

对同一个中断来讲，是可以注册多个中断处理函数的。即request\_irq对同一个中断号执行多次。这些handler会被挂到对应中断描述的action链表中。上述函数即依次执行对应该中断号的irqaction中handler（即设备request\_irq时注册的handler）。

3\. 0-15中断号是ARM多核之间用来相互沟通的SGI消息。本文不做分析。

### irq\_enter()与irq\_exit()

irq\_enter和irq\_exit是中断处理的重要函数，前一节的handle\_domain\_irq函数中可以看到他们分别在执行对应中断描述符的handler前后被调用。本节对这两个函数做重点分析：

void irq_enter(void)
{
	rcu\_irq\_enter();                               
	if (is\_idle\_task(current) && !in_interrupt()) {-----1
h_disable();
		local\_bh\_disable();
                tick\_irq\_enter();    
		\_local\_bh_enable();                    
	}

	\_\_irq\_enter();              
}

#define \_\_irq\_enter()					\  
	do {						\
		account\_irq\_enter_time(current);	\
		preempt\_count\_add(HARDIRQ_OFFSET);	\ ----2
		trace\_hardirq\_enter();			\  
	} while (0)

void irq_exit(void)
{
#ifndef \_\_ARCH\_IRQ\_EXIT\_IRQS_DISABLED
	local\_irq\_disable();
#else
	WARN\_ON\_ONCE(!irqs_disabled());
#endif

	account\_irq\_exit_time(current);
	preempt\_count\_sub(HARDIRQ_OFFSET);
	if (!in\_interrupt() && local\_softirq_pending())
		invoke_softirq();                   ------3

	tick\_irq\_exit();
	rcu\_irq\_exit();
	trace\_hardirq\_exit(); /* must be last! */
}

1\. 如果当前是在idle状态，且不在中断上下文中，则通知中断状态下发生了idle

2\. irq\_enter() 最重要的步骤就是\_\_irq_enter(),本函数的主要作用就是关闭抢占，避免执行中断处理函数时，CPU被调度到执行其他事情。

3\. 在执行完设备驱动的中断handler，开始执行irq\_exit()，该函数主要的作用就是查看系统中是否有未执行的软中断，若有则呼叫invoke\_softirq执行系统中的软中断。此时中断的上半部执行结束，开始执行中断下半部。

待续
--

下一篇文章，将介绍中断下半部的主要的实现方式：

*   软中断（Soft IRQ）
*   Workqueue