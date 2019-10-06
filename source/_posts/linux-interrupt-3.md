---
title: Linux中断(1)
tags:
  - Linux
  - 中断
url: 1490.html
id: 1490
categories:
  - Linux
  - Linux中断
date: 2019-01-01 00:12:17
---

前言
--

之前[LINUX中断学习笔记(1)](https://l2h.site/2018/07/15/linux-interrupt-1/)和[LINUX中断学习笔记(2)](https://l2h.site/2018/07/15/linux-interrupt-2)介绍了Linux中断的一些基础知识，但是不够深入。最近公司所在团队逐渐在往内核进一步深入，仅有前边文章的浅显知识已不足以覆盖工作的需求，也无法和部门同事做深入的技术讨论。本文是在工作之余，进行代码的研究和资料的查找，希望可以将中断子系统尽量整理清楚。本文内容会不定期补充更新，也希望能对访问到本站的朋友有所帮助。

![](https://l2h.site/wp-content/uploads/2018/12/Linux-Interrupt.png)

架构(Architecture)
----------------

### 硬件连接
```
                                   +---------+
                    +-------+      |         |
                    |Device1+------+         |
                    +-------+      |         |   +---------+
                                   |         +---+  CPU1   |
                    +-------+      |         |   |         |
                    |Device2+------+         |   +---------+
                    +-------+      |         |
                                   |  IRQ    |   +---------+
                    +-------+      |  Ctrl   |   |  CPU2   |
                    |Device3+------+  A      +---+         |
                    +-------+      |         |   +---------+
                                   |         |
                    +-------+      |         |   +---------+
+------+    +----+  |Device4+------+         |   |  CPU3   |
|Dev5  +----+    |  +-------+      |         +---+         |
+------+    |    |                 |         |   +---------+
            |    +-----------------+         |
+------+    |IRQ |                 +---------+
|Dev6  +----+Ctrl|
+------+    |B   |
+------+    |    |
|Dev7  +----+    |
+------+    |    |
            +----+
```
*   中断控制器（IRQ Controller）：负责对硬件中断进行管理。例如缓冲、优先级判断等。
*   硬件设备（Devices）：中断接入中断控制器

如上图，系统中可能存在多个中断控制器（IRQ Controller）。中断控制器或者直接与CPU连接，或者通过级联方式接入上一级中断控制器，最后接入CPU。中断控制器可以与系统多个核心都有连接，根据一定方法选择响应该中断的CPU（即设定中断Affinity）

### 软件架构

如下图。Linux中断子系统软件架构主要分为三个层次，从上到下依次是：

*   设备驱动层：设备驱动，主要负责向系统注册真正的中断处理函数
*   硬件无关中断处理层：Linux中断子系统的核心代码
*   CPU架构相关中断控制器：与CPU架构体系相关的中断处理代码，以及与特定中断控制器相关的中断处理代码

可以看出，Linux的中断处理架构非常清晰。硬件无关中断处理层，统一了中断处理的接口：避免了设备驱动实现者必须要关心不同CPU体系以及不同中断处理器的不同特性。
```
+-------------------------------------------------+
| +---------------------------------------------+ |
| |Device    ||Device   || Device   ||  Device  | |  设 备 驱 动 层
| |DriVer    ||DriVer   || DriVer   ||  DriVer  | |
| +---------------------------------------------+ |
+-------------------------------------------------+

+-------------------------------------------------+
| +---------------------------------------------+ |
| |  Linux Interrupt Handle FWK                 | |  Linux硬 件 无 关 中 断 处 理 层
| +---------------------------------------------+ |
+-------------------------------------------------+

+-------------------------+-----------------------+
|------------+ +--------+ | +--------+ +----------|
|| CPU ARCH 1| |CPU ARCH| | |IRQCHIP1| |IRQCHIP2 ||  CPU体 系 相 关 中 断 处 理 层
|------------+ +--------+ | +--------+ +----------|
+-------------------------+-----------------------+
```

### 内核目录结构

IRQ相关的Linux内核目录结构如下图所示：
```
Linux/
| 
+-->include/
|   |
|   |-->irqchip/             中断控制器相关API定义
|   |-->linux/               Linux硬件无关中断处理层API定义
|       |
|       +-->irqdesc.h
|       |-->irqhandler.h
|       |-->irqdomain.h
|       |-->irqflags.h
|       +-->irq.h
+-->kernel/
|   |
|   +-->irq/                 Linux硬件无关中断处理层实现
|       |
|       +-->irqdesc.c/irqdomain.c/.../etc.
+-->driver/               
    |
    +-->irqchip/             中断控制器的相关实现
```
内核数据结构以及API
-----------

为了实现上述的软件架构，将中断使用者和硬件相关的代码隔离开。内核定义了一系列了数据结构以及相应的操作函数。本节对其中重点的数据结构进行分析：

### 中断域（struct irq_domain）

在介绍irq_domain之前，先介绍什么是IRQ Domain。早期的系统，只有一个中断控制器，接入中断控制器的物理中断号都是不同的。但是随着计算机系统的发展，系统中可以挂接更多的中断控制器。特别是嵌入式系统的出现，类似GPIO这种也可以视作一种中断控制器。每个中断控制器都有自己中断线的物理编号，且这些物理编号会有重复。此时，Linux Kernel发展出了IRQ Domain的概念，来区分这些相同的物理中断编号。

IRQ Domain，顾名思义，即中断控制域。每个中断控制器都有自己的struct irq_domain结构体，以及自己下属的物理中断号，也不用担心物理中断号重复无法区分的问题。以“硬件连接”一节的图片为例，IRQ Controller 1及其下属的几根中断输入线作为一个中断控制域，而IRQ Controller 2作为另外一个中断控制域。

![Linux中断控制域](http://pic.l2h.site/l2hsiteLinux-interrupt-3-irq-domain.png)

**Figure 1. Linux中断控制域图例**

可能会有朋友问了，“记得向Linux Kernel注册中断处理函数是唯一的中断号，也没有传入Domain ID之类的参数，这如何解释？”。原因很简单，因为这个唯一的中断号是Linux Kernel分配的全局唯一**虚拟**中断号，Linux通过一定方式将其和中断控制域的物理中断号关联。对每个中断控制域来讲，这种映射/关联主要有以下三类：

*   线性映射(Linear)：固定大小的数组映射（物理中断号为数组索引）。当该中断控制域的物理中断号较连续且数量不大时使用。
*   基树映射(Radix Tree): 与线性映射相反，物理中断号比较大，或者不太连续时使用。
*   直接映射(No Map/Diret): 有些中断控制器支持物理中断号编程，其被分配到的虚拟中断号可以被直接设定到中断控制器。
*   Legacy映射(传统映射？): 特殊的一种映射模式。当需要固定分配一部分中断编号范围时使用。设备驱动可以直接使用约定的虚拟中断号来进行IRQ操作。

#### 数据结构

中断域数据结构的定义在内核目录include/linux/irq.h，其所有成员以及相应解释如下：
```C
struct irq_domain {
	struct list_head link;            //Linux全局irq_domain链表的成员        
	const char *name;                 //中断控制域的名称
	const struct irq_domain_ops *ops; //中断控制域的操作集合
	void *host_data;                  //该中断控制域的私有数据指针
	unsigned int flags;               //该中断控制域的标识
 
	struct fwnode_handle *fwnode;     //待补充
	enum irq_domain_bus_token bus_token;//中断域的总线类型，见irq_domain_bus_token的定义
	struct irq_domain_chip_generic *gc; //IRQ Domain使用的中断芯片数据结构
#ifdef	CONFIG_IRQ_DOMAIN_HIERARCHY
	struct irq_domain *parent;       //如果支持中断控制域层次结构，指向该控制域的父级
#endif
	irq_hw_number_t hwirq_max;          //该域中的最大物理中断号
	unsigned int revmap_direct_max_irq; //直接映射的最大中断号
	unsigned int revmap_size;           //虚拟/物理线性映射表的大小
	struct radix_tree_root revmap_tree; //虚拟/物理映射基树（当该IRQ Domain使用基树方式映射时）
	unsigned int linear_revmap[];       //虚拟/物理线性映射表
};
enum irq_domain_bus_token {
	DOMAIN_BUS_ANY		= 0,
	DOMAIN_BUS_PCI_MSI,
	DOMAIN_BUS_PLATFORM_MSI,
	DOMAIN_BUS_NEXUS,
};
```
#### 中断域处理函数

中断域处理函数主要有以下几类：

1.  中断域添加/删除函数irq_domain_add_*()/remove()系列，例：
```C
//线性映射添加
static inline struct irq_domain *irq_domain_add_linear(struct device_node *of_node,
					 unsigned int size,
					 const struct irq_domain_ops *ops,
					 void *host_data)
//直接映射添加
static inline struct irq_domain *irq_domain_add_nomap(struct device_node *of_node,
					 unsigned int max_irq,
					 const struct irq_domain_ops *ops,
					 void *host_data)
{
	return __irq_domain_add(of_node_to_fwnode(of_node), 0, max_irq, max_irq, ops, host_data);
}
//Legacy映射添加
static inline struct irq_domain *irq_domain_add_legacy_isa(
				struct device_node *of_node,
				const struct irq_domain_ops *ops,
				void *host_data)
{
	return irq_domain_add_legacy(of_node, NUM_ISA_INTERRUPTS, 0, 0, ops,
				     host_data);
}
//基树映射添加
static inline struct irq_domain *irq_domain_add_tree(struct device_node *of_node,
					 const struct irq_domain_ops *ops,
					 void *host_data)
{
	return __irq_domain_add(of_node_to_fwnode(of_node), 0, ~0, 0, ops, host_data);
}
//移除Domain
extern void irq_domain_remove(struct irq_domain *host);
```
2. 创建中断号映射的irq_create_XXX()系列
```C
//创建HWIRQ映射，返回对应的虚拟中断号
extern unsigned int irq_create_mapping(struct irq_domain *host,
				       irq_hw_number_t hwirq);
//根据Firmware Spec创建映射，一般与设备树DTS解析的信息有关
extern unsigned int irq_create_fwspec_mapping(struct irq_fwspec *fwspec);
//创建Direct Mapping
extern unsigned int irq_create_direct_mapping(struct irq_domain *host);
//创建固定映射
extern int irq_create_strict_mappings(struct irq_domain *domain,
				      unsigned int irq_base,
				      irq_hw_number_t hwirq_base,
                                      int count);
```
3. IRQ Domain Callback函数，供IRQ Chip Driver定义。而该中断域中的设备驱动在
```C
struct irq_domain_ops {
	int (*match)(struct irq_domain *d, struct device_node *node,
		     enum irq_domain_bus_token bus_token); //
	int (*map)(struct irq_domain *d, unsigned int virq, irq_hw_number_t hw);                                       // 创建或者更新虚拟中断号及中断域物理中断号的映射
	void (*unmap)(struct irq_domain *d, unsigned int virq);//删除映射
	int (*xlate)(struct irq_domain *d, struct device_node *node,
		     const u32 *intspec, unsigned int intsize,
		     unsigned long *out_hwirq, unsigned int *out_type);//根据DTS节点以及对应的interrupt描述符，解析出物理中断号和中断出发方式
#ifdef	CONFIG_IRQ_DOMAIN_HIERARCHY
//以下为IRQ Domain 层次相关回调函数
	int (*alloc)(struct irq_domain *d, unsigned int virq,
		     unsigned int nr_irqs, void *arg);
	void (*free)(struct irq_domain *d, unsigned int virq,
		     unsigned int nr_irqs);
	void (*activate)(struct irq_domain *d, struct irq_data *irq_data);
	void (*deactivate)(struct irq_domain *d, struct irq_data *irq_data);
	int (*translate)(struct irq_domain *d, struct irq_fwspec *fwspec,
			 unsigned long *out_hwirq, unsigned int *out_type);
#endif
};
```
我们在后边的介绍中再回过头看这些回调函数的定义。

### 中断描述符 (struct irq_desc)

接着我们介绍中断描述符。中断描述符与Kernel中全局唯一的虚拟中断号关联。中断描述符或者采取固定分配，例如：

> struct irq_desc irq_desc[NR_IRQS];

或者采用分散管理方式（管理上使用基树-Radix Tree做管理）：

> #ifdef CONFIG_SPARSE_IRQ  
> static RADIX_TREE(irq_desc_tree, GFP_KERNEL);
> 
> #endif
```C
//分配cnt数量的连续中断号
unsigned int irq_alloc_hwirqs(int cnt, int node)
{
	int i, irq = __irq_alloc_descs(-1, 0, cnt, node, NULL);

	if (irq < 0)
		return 0;
	for (i = irq; cnt > 0; i++, cnt--) {
		if (arch_setup_hwirq(i, node))
			goto err;
		irq_clear_status_flags(i, _IRQ_NOREQUEST);
	}
	return irq;
err: //如果执行失败，则释放之前已经分配的中断
	for (i--; i >= irq; i--) {
		irq_set_status_flags(i, _IRQ_NOREQUEST | _IRQ_NOPROBE);
		arch_teardown_hwirq(i);
	}
	irq_free_descs(irq, cnt);
	return 0;
}
//释放已经分配的中断描述符
void irq_free_descs(unsigned int from, unsigned int cnt)
{
	int i;
	if (from >= nr_irqs || (from + cnt) > nr_irqs)
		return;
	for (i = 0; i < cnt; i++)
		free_desc(from + i);
	mutex_lock(&sparse_irq_lock);
	bitmap_clear(allocated_irqs, from, cnt);
	mutex_unlock(&sparse_irq_lock);
}
```

而中断描述符的数据结构以及其与中断子系统其他数据结构的关系如下（本图以线性管理方式为例）：
```
+-------+-------+-------+------+-------+-------+
|irq_   | irq_  | irq_  | irq_ | ....  | irq_  |
|desc 0 | desc 1| desc 2| desc3|       | desc n|
+---+---+------------------------------+-------+
    |
    |
+---v------+
|handle_irq|
+----------+
|lock      |
+----------+    +---------------+
|irq_data  +--->+struct irq_data|
+----------+    +------+--------+
| ....     |           |
+----------+           v
|*action   |    +--------------+
+----------+    |    irq        |
                +---------------+
                |    hw_irq     |
                +---------------+
                |struct irq_chip|
                +---------------|
                |struct irq_domain
                +---------------+
                |struct irq_data|
                +---------------+
                | *chip_data    |
                +---------------+
```
其中：

*   **handle_irq**: High-level IRQ处理函数，一般在中断控制器初始化时定义
*   **irq_data**: 中断相关数据
    *   中断号
    *   物理中断号
    *   IRQ中断域
    *   IRQ控制器芯片相关信息
    *   IRQ控制器芯片独有的数据
*   **action**: 设备驱动注册的中断处理函数（即request_irq传入的Handler）

### 中断芯片 (struct irq_chip)
```C
struct irq_chip {
	const char	*name; //中断芯片名称，会呈现在/proc/interrupts内
	unsigned int	(*irq_startup)(struct irq_data *data); //中断开启（默认指向enable回调函数）
	void		(*irq_shutdown)(struct irq_data *data); //中断关闭（默认指向disable回调）
	void		(*irq_enable)(struct irq_data *data);  //开中断 （如果未定义，默认执行irq_mask）
	void		(*irq_disable)(struct irq_data *data);  //关中断
	void		(*irq_ack)(struct irq_data *data); //？？
	void		(*irq_mask)(struct irq_data *data); //屏蔽中断
	void		(*irq_mask_ack)(struct irq_data *data); //??
	void		(*irq_unmask)(struct irq_data *data); //中断去除屏蔽
	void		(*irq_eoi)(struct irq_data *data); //中断结束
 
	int		(*irq_set_affinity)(struct irq_data *data, const struct cpumask *dest, bool force); //设置中断的CPU affinity
	int		(*irq_retrigger)(struct irq_data *data); //重新发送IRQ到CPU
	int		(*irq_set_type)(struct irq_data *data, unsigned int flow_type); //设定IRQ类型（边沿触发，电平触发）
	int		(*irq_set_wake)(struct irq_data *data, unsigned int on); //电源管理休眠或唤醒时使用，若该中断需要在CPU休眠时能触发，则函数需要实现对应功能
	void		(*irq_bus_lock)(struct irq_data *data);
//IRQ BUS锁
	void		(*irq_bus_sync_unlock)(struct irq_data *data);  //??
	void		(*irq_cpu_online)(struct irq_data *data);//配置第二CPU的中断源
	void		(*irq_cpu_offline)(struct irq_data *data);//配置第二CPU的中断源
	void		(*irq_suspend)(struct irq_data *data); // 进入休眠状态前电源管理系统调用
	void		(*irq_resume)(struct irq_data *data); //唤醒后电源管理系统调用
	void		(*irq_pm_shutdown)(struct irq_data *data);//电源管理相关，当关机前（Suspend to disk前应该也会调用），内核调用
	void		(*irq_calc_mask)(struct irq_data *data);//??
	void		(*irq_print_chip)(struct irq_data *data, struct seq_file *p);//当用户调用显示中断时，印出中断芯片的一些特别字段
	int		(*irq_request_resources)(struct irq_data *data);
	void		(*irq_release_resources)(struct irq_data *data); //调用其他callback前请求或者释放资源（可选）
	void		(*irq_compose_msi_msg)(struct irq_data *data, struct msi_msg *msg);//对MSI中断类型有效
	void		(*irq_write_msi_msg)(struct irq_data *data, struct msi_msg *msg);//对MSI中断类型有效
	int		(*irq_get_irqchip_state)(struct irq_data *data, enum irqchip_irq_state which, bool *state); //返回IRQ中断芯片状态
	int		(*irq_set_irqchip_state)(struct irq_data *data, enum irqchip_irq_state which, bool state);//设定IRQ中断芯片状态
	int		(*irq_set_vcpu_affinity)(struct irq_data *data, void *vcpu_info); //虚拟机中的CPU Affinity
	unsigned long	flags; //中断芯片相关字段
};
```
待续
--

[Linux中断(2) -- 流程](https://l2h.site/2019/01/01/linux-interrupt-4)