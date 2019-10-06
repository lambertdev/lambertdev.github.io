---
title: Linux中断学习笔记(2) -- 嵌入式设备中断
tags:
  - Linux
  - 中断
url: 476.html
id: 476
categories:
  - Linux
  - Linux中断
date: 2018-07-21 20:44:47
---

在[Linux中断学习笔记(1)](http://www.l2h.site/linux-interrupt-1/)提到，外设通过中断控制器连接到CPU的中断线。嵌入式系统也不例外。

ARM嵌入式系统GIC架构
-------------

[ARM官网](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dai0176c/ar01s03s01.html)所举图为例：ARM的中断控制器GIC（General Interrupt Controller）将从外设输入的中断通过CPU的IRQ信号线（ARM中主要为FIQ和IRQ）连接到系统中各CPU。

图1. GIC 简单结构图

![Linux中断学习笔记(2) -- 嵌入式设备中断](http://pic.www.l2h.site/l2hsiteImage%203.png "Linux中断学习笔记(2) -- 嵌入式设备中断") 

中断控制器允许级联，一个有中断级联的终端流程如下图所示。次级GIC将中断信号通知到主GIC后，主GIC再通知CPU，CPU读各级中断控制器的Ack Register得到中断号，并开始执行相应的中断例程。执行完后，**直接**写次级中断控制器的寄存器标记中断服务结束。

图2. GIC中断服务级联

![Linux中断学习笔记(2) -- 嵌入式设备中断](http://pic.www.l2h.site/l2hsiteImage%204.png "Linux中断学习笔记(2) -- 嵌入式设备中断") 
参见MT6577 GIC中断控制器的DTS声明（以arch/arm/boot/dts/mt6592.dtsi为例）

```C
sysirq: interrupt-controller@10200220 {
    compatible = "mediatek,mt6592-sysirq", "mediatek,mt6577-sysirq";
    interrupt-controller;
    #interrupt-cells = <3>;
    interrupt-parent = <&gic>;
    reg = <0x10200220 0x1c>;
};

gic: interrupt-controller@10211000 {
    compatible = "arm,cortex-a7-gic";
    interrupt-controller;
    #interrupt-cells = <3>;
    interrupt-parent = <&gic>;
    reg = <0x10211000 0x1000>,
          <0x10212000 0x1000>;
};
```C
Linux操作系统通过加载DTS将GIC的硬件信息装载到特定内存位置，GIC的驱动程序运行时通过DTS的API读取到这些硬件信息（例如寄存器地址）来控制中断的处理。

物理中断号的映射
--------

GIC驱动程序初始化时，会向系统申请中断描述符。中断描述符是全局变量，外设驱动request_irq传入的第一个参数便是中断描述符的索引。外设根据DTS中对应的物理中断号和其所在的中断Domain，便可以得到外设的虚拟中断id（即中断描述符的索引）
```C
static int __init
gic_of_init(struct device_node *node, struct device_node *parent)
{
    void __iomem *cpu_base;
    void __iomem *dist_base;
    u32 percpu_offset;
    int irq;

    if (WARN_ON(!node))
        return -ENODEV;

    dist_base = of_iomap(node, 0);
    WARN(!dist_base, "unable to map gic dist registers\\n");

    cpu_base = of_iomap(node, 1);
    WARN(!cpu_base, "unable to map gic cpu registers\\n");

    /*
     * Disable split EOI/Deactivate if either HYP is not available
     * or the CPU interface is too small.
     */
    if (gic_cnt == 0 && !gic_check_eoimode(node, &cpu_base))
        static_key_slow_dec(&supports_deactivate);

    if (of_property_read_u32(node, "cpu-offset", &percpu_offset))
        percpu_offset = 0;

    __gic_init_bases(gic_cnt, -1, dist_base, cpu_base, percpu_offset,
             &node->fwnode);       //执行GIC相关初始化
    //.......
}
static void __init __gic_init_bases(unsigned int gic_nr, int irq_start,
			   void __iomem *dist_base, void __iomem *cpu_base,
			   u32 percpu_offset, struct fwnode_handle *handle)
{
	//...................
	/*
	 * 计算GIC支持的IRQ数量
	 */
	gic_irqs = readl_relaxed(gic_data_dist_base(gic) + GIC_DIST_CTR) & 0x1f;
	gic_irqs = (gic_irqs + 1) * 32;
	if (gic_irqs > 1020)
		gic_irqs = 1020;
	gic->gic_irqs = gic_irqs;

	if (handle) {		/* DT/ACPI */
		gic->domain = irq_domain_create_linear(handle, gic_irqs,
						       &gic_irq_domain_hierarchy_ops,
						       gic);
	} else {		/* Legacy support */
		/*
		 * For primary GICs, skip over SGIs.
		 * For secondary GICs, skip over PPIs, too.
		 */
		if (gic_nr == 0 && (irq_start & 31) > 0) {
			hwirq_base = 16;
			if (irq_start != -1)
				irq_start = (irq_start & ~31) + 16;
		} else {
			hwirq_base = 32;
		}

		gic_irqs -= hwirq_base; /* calculate # of irqs to allocate */

		irq_base = irq_alloc_descs(irq_start, 16, gic_irqs,
					   numa_node_id());
                //为中断控制器分配gic_irqs个中断描述符（数量如前计算）
		if (IS_ERR_VALUE(irq_base)) {
			WARN(1, "Cannot allocate irq_descs @ IRQ%d, assuming pre-allocated\\n",
			     irq_start);
			irq_base = irq_start;
		}

		gic->domain = irq_domain_add_legacy(NULL, gic_irqs, irq_base,
					hwirq_base, &gic_irq_domain_ops, gic);
	}

}
```
在Kernel初始化时，会去枚举DTS，根据每一个设备的中断domain以及DTS中描述的物理中断号来与系统中的唯一irq编号virq做映射（在DTS枚举时，会根据设备的中断描述来申请系统中唯一的中断描述符）