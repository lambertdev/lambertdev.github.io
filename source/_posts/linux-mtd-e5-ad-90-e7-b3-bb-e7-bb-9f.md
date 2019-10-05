---
title: Linux MTD子系统
tags:
  - Linux
  - MTD
url: 2555.html
id: 2555
categories:
  - Linux文件系统
date: 2019-08-02 19:27:46
---

### 为什么需要MTD子系统

嵌入式系统使用Flash作为存储设备，Flash类别有Nand、Nor等。Flash的上层是文件系统。直觉上，系统中使用这些Flash时，我们需要为每种Flash编写驱动。同时在调用Flash的文件系统做接口对接。这样，每使用一种新的Flash类型甚至型号，都得修改文件系统的编码来做适配。显然，这会造成代码的爆炸，同时也不方便大家各司其职（例如：厂商A做Flash，添加一个新的Flash需要厂商A的驱动开发人员去改写所有文件系统的接口，这显然不现实）。

几乎所有的现代操作系统都不会允许以上事情的发生。通用的做法是，抽象出上下层对接的方式，厂商驱动开发人员只需要按照接口进行匹配节课。

MTD（Memory Technology Devices）便是Linux系统下处理以上问题的方式。

### 架构与代码目录

MTD在系统中的结构如下图。

*   构成MTD的部分有MTD核心、MTD字符设备和MTD块设备层。成为文件系统和底层硬件驱动的沟通桥梁
*   MTD核心建立在Flash驱动（位于drivers/mtd/）之上，为Flash驱动提供一系列的API抽象
*   MTD为上层提供统一的操作抽象接口如`dev/mtd0`, `/dev/mtd1`（例如擦除、读写等），同时提供 `/proc/mtd`供上层读取MTD系统相关信息
*   MTD提供一系列的API，为基于Flash的文件系统提供控制操作

             \+\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\+
             | 使用文件系统的应用 |
             \+\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\+
   \+\-\-\-\-\-\-\-\-\-\-\-\-\+ +------------+
   |Char Dev节点 | |Block Dev节点| 用户空间层
   \+\-\-\-\-\-\-\-\-\-\-\-\-\+ +------------+
+---------------------------------+
   +---------------+-----------+
   |  MTD字 符 设 备| MTD块 设 备|
   +---------------+-----------+
   +---------------------------+
   |      MTD Core             | 内核层
   +---------------------------+
   +---------------------------+
   |       Flash驱 动           |
   +---------------------------+
+---------------------------------+
   +---------------------------+
   |       各 种 Flash          | 硬件层
   +---------------------------+

MTD代码在Linux内核源码树的位置：

*   _include/linux/mtd/_：定义所有MTD相关头文件
*   _drivers/mtd_: 定义MTD核心，以及Flash驱动

### 数据结构

MTD的核心结构定义在内核源码树的_include/linux/mtd/mtd.h_。先看核心数据结构mtd_info：

struct mtd_info {
	u_char type;
	uint32_t flags;
	uint64_t size;	 // MTD总大小
	uint32_t erasesize; //擦除大小
	uint32_t writesize; //最小写单位
	uint32_t writebufsize; //写缓冲大小，提升写效率
	uint32_t oobsize;   // 每个Flash 块的OOB数量
	uint32_t oobavail;  // 每个Flash块的OOB大小
	unsigned int erasesize_shift;
	unsigned int writesize_shift;
	unsigned int erasesize_mask;
	unsigned int writesize_mask; //shift和mask在Linux源码中常见，主要用于MTD大小相关的位运算
	unsigned int bitflip_threshold; //位翻转阈值，即最大允许位翻转个数，超出后，读写返回“-EUCLEAN”

	const char *name; //MTD名称
	int index;  //MTD索引
	const struct mtd\_ooblayout\_ops *ooblayout; //OOB layout操作
	const struct mtd\_pairing\_scheme *pairing;//MLC/TLC NANDs Flash 颗粒的配对策略
	unsigned int ecc\_step\_size; //ecc步长
	unsigned int ecc_strength; //ecc最大可纠正错误bit数

	int numeraseregions;
	struct mtd\_erase\_region_info *eraseregions; //擦除区域信息

	//以下为Flash操作的回调函数，不同flash有不同实现
	int (\*\_erase) (struct mtd\_info \*mtd, struct erase_info *instr);
	int (\*\_point) (struct mtd\_info \*mtd, loff\_t from, size\_t len, size\_t \*retlen, void \*\*virt, resource\_size_t *phys);
	int (\*\_unpoint) (struct mtd\_info \*mtd, loff\_t from, size\_t len);
	unsigned long (\*\_get\_unmapped\_area) (struct mtd\_info \*mtd,
					     unsigned long len,
					     unsigned long offset,
					     unsigned long flags);
	int (\*\_read) (struct mtd\_info \*mtd, loff\_t from, size\_t len,
		      size\_t \*retlen, u\_char \*buf);
	int (\*\_write) (struct mtd\_info \*mtd, loff\_t to, size\_t len,
		       size\_t \*retlen, const u\_char \*buf);
	int (\*\_panic\_write) (struct mtd\_info \*mtd, loff\_t to, size\_t len, size\_t \*retlen, const u_char \*buf);
	int (\*\_read\_oob) (struct mtd\_info \*mtd, loff\_t from,
			  struct mtd\_oob\_ops *ops);
	int (\*\_write\_oob) (struct mtd\_info \*mtd, loff\_t to,
			   struct mtd\_oob\_ops *ops);
	int (\*\_get\_fact\_prot\_info) (struct mtd\_info \*mtd, size\_t len, size\_t \*retlen, struct otp\_info \*buf);
	int (\*\_read\_fact\_prot\_reg) (struct mtd\_info \*mtd, loff\_t from, size\_t len, size\_t \*retlen, u_char \*buf);
	int (\*\_get\_user\_prot\_info) (struct mtd\_info \*mtd, size\_t len,  size\_t \*retlen, struct otp\_info \*buf);
	int (\*\_read\_user\_prot\_reg) (struct mtd\_info \*mtd, loff\_t from, size\_t len, size\_t \*retlen, u_char \*buf);
	int (\*\_write\_user\_prot\_reg) (struct mtd\_info \*mtd, loff\_t to, size\_t len, size\_t \*retlen, u_char \*buf);
	int (\*\_lock\_user\_prot\_reg) (struct mtd\_info \*mtd, loff\_t from,  size_t len);
	int (\*\_writev) (struct mtd\_info \*mtd, const struct kvec \*vecs, unsigned long count, loff\_t to, size\_t \*retlen);
	void (\*\_sync) (struct mtd\_info \*mtd);
	int (\*\_lock) (struct mtd\_info \*mtd, loff\_t ofs, uint64\_t len);
	int (\*\_unlock) (struct mtd\_info \*mtd, loff\_t ofs, uint64\_t len);
	int (\*\_is\_locked) (struct mtd\_info \*mtd, loff\_t ofs, uint64_t len);
	int (\*\_block\_isreserved) (struct mtd\_info \*mtd, loff\_t ofs);
	int (\*\_block\_isbad) (struct mtd\_info \*mtd, loff\_t ofs);
	int (\*\_block\_markbad) (struct mtd\_info \*mtd, loff\_t ofs);
	int (\*\_suspend) (struct mtd\_info \*mtd);
	void (\*\_resume) (struct mtd\_info \*mtd);
	void (\*\_reboot) (struct mtd\_info \*mtd);
	int (\*\_get\_device) (struct mtd_info \*mtd);
	void (\*\_put\_device) (struct mtd_info \*mtd);
	struct backing\_dev\_info *backing\_dev\_info;

	struct notifier\_block reboot\_notifier;  //重启通知
	struct mtd\_ecc\_stats ecc_stats; //ECC统计数据
	int subpage_sft;
	void *priv;
	struct module *owner;
	struct device dev;
	int usecount;
};

另外一个比较重要的数据结构为mtd_partition，顾名思义，对一块flash进行分区:

struct mtd_partition {
	const char *name; //分区名称
	uint64_t size;	//大小
	uint64_t offset; //在MTD设备的偏移
	uint32\_t mask\_flags; //MTD主设备的掩码
}

通常每种类型的Flash芯片定义了这样一个数据结构，用于对Flash进行操作。可参考drivers/mtd/devices下相关使用（例如lpddr2_nvm.c）

mtdcore.h/mtdcore.c下定义了一系列使用MTD设备的API：

struct mtd\_info *\_\_mtd\_next\_device(int i);
int add\_mtd\_device(struct mtd_info *mtd); //注册MTD设备
int del\_mtd\_device(struct mtd_info *mtd); //注销
int add\_mtd\_partitions(struct mtd\_info *, const struct mtd\_partition *, int); 
int del\_mtd\_partitions(struct mtd_info *); //添加、删除MTD分区

int parse\_mtd\_partitions(struct mtd\_info \*master, const char \* const \*types, struct mtd\_partitions \*pparts, struct mtd\_part\_parser_data *data); //从MTD设备查找MTD分区

void mtd\_part\_parser\_cleanup(struct mtd\_partitions *parts);//清除parse\_mtd\_partitions得到的MTD分区数据

int \_\_init init\_mtdchar(void);
void \_\_exit cleanup\_mtdchar(void); //注册、清除MTD字符设备

void register\_mtd\_user (struct mtd_notifier *new)
int unregister\_mtd\_user (struct mtd_notifier *old) //注册、去注册MTD使用者（当有MTD变动时通知）

### MTD字符设备

MTD字符设备定义在mtdchar.c，其中定义了字符设备的相关操作：

static const struct file\_operations mtd\_fops = {
	.owner		= THIS_MODULE,
	.llseek		= mtdchar_lseek,
	.read		= mtdchar_read,
	.write		= mtdchar_write,
	.unlocked\_ioctl	= mtdchar\_unlocked_ioctl,
#ifdef CONFIG_COMPAT
	.compat\_ioctl	= mtdchar\_compat_ioctl,
#endif
	.open		= mtdchar_open,
	.release	= mtdchar_close,
	.mmap		= mtdchar_mmap,
#ifndef CONFIG_MMU
	.get\_unmapped\_area = mtdchar\_get\_unmapped_area,
	.mmap\_capabilities = mtdchar\_mmap_capabilities,
#endif
};

### MTD块设备

MTD块设备定义在mtdblock.c/mtdblock_ro.c，定义了块设备相关操作：

static struct mtd\_blktrans\_ops mtdblock_tr = {
	.name		= "mtdblock",
	.major		= MTD\_BLOCK\_MAJOR,
	.part_bits	= 0,
	.blksize 	= 512,
	.open		= mtdblock_open,
	.flush		= mtdblock_flush,
	.release	= mtdblock_release,
	.readsect	= mtdblock_readsect,
	.writesect	= mtdblock_writesect,
	.add\_mtd	= mtdblock\_add_mtd,
	.remove\_dev	= mtdblock\_remove_dev,
	.owner		= THIS_MODULE,
};
static struct mtd\_blktrans\_ops mtdblock_tr = {
	.name		= "mtdblock",
	.major		= MTD\_BLOCK\_MAJOR,
	.part_bits	= 0,
	.blksize 	= 512,
	.readsect	= mtdblock_readsect,
	.writesect	= mtdblock_writesect,
	.add\_mtd	= mtdblock\_add_mtd,
	.remove\_dev	= mtdblock\_remove_dev,
	.owner		= THIS_MODULE,
};

### 分区

在嵌入式设备，一个flash往往会被划分做不同的功能分区。例如，升级分区、文件系统分区、bootloader分区、配置分区、备份分区等等。代码树中的drivers/mtd/mtdpart.c实现了分区的相关操作：

以下全局变量定义了MTD分区的链表：

static LIST\_HEAD(mtd\_partitions);
static DEFINE\_MUTEX(mtd\_partitions_mutex);

链表元素为：

struct mtd_part {
	struct mtd_info mtd;      //MTD信息
	struct mtd_info *master;  //MTD 设备指针
	uint64_t offset;          //主设备偏移
	struct list_head list;
};

相关操作为：

static struct mtd\_part \*allocate\_partition(struct mtd\_info \*master, const struct mtd\_partition *part, int partno, uint64\_t cur\_offset)
//分配分区
static int mtd\_add\_partition\_attrs(struct mtd\_part *new)
int mtd\_add\_partition(struct mtd_info \*master, const char \*name, long long offset, long long length)
int mtd\_del\_partition(struct mtd_info *master, int partno)
//添加删除分区
int add\_mtd\_partitions(struct mtd\_info \*master, const struct mtd\_partition \*parts, int nbparts)
//根据分区表添加分区
int \_\_register\_mtd\_parser(struct mtd\_part_parser \*p, struct module \*owner)
void deregister\_mtd\_parser(struct mtd\_part\_parser *p)
//向内核注册和去注册回调函数用于MTD分区时执行
int parse\_mtd\_partitions(struct mtd\_info \*master, const char \*const \*types, struct mtd\_partitions \*pparts, struct mtd\_part\_parser_data *data)

### 参考文献

1.  《[General MTD documentation](http://www.linux-mtd.infradead.org/doc/general.html)》