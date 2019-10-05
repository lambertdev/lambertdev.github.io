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
```
   +-----------------+
  | 使用文件系统的应用 |
   +-----------------+
   +------------+ +------------+
   |Char Dev节点 | |Block Dev节点| 用户空间层
   +------------+ +------------+
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
```
MTD代码在Linux内核源码树的位置：

*   _include/linux/mtd/_：定义所有MTD相关头文件
*   _drivers/mtd_: 定义MTD核心，以及Flash驱动

### 数据结构

MTD的核心结构定义在内核源码树的_include/linux/mtd/mtd.h_。先看核心数据结构mtd_info：
```C
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
	const struct mtd_ooblayout_ops *ooblayout; //OOB layout操作
	const struct mtd_pairing_scheme *pairing;//MLC/TLC NANDs Flash 颗粒的配对策略
	unsigned int ecc_step_size; //ecc步长
	unsigned int ecc_strength; //ecc最大可纠正错误bit数

	int numeraseregions;
	struct mtd_erase_region_info *eraseregions; //擦除区域信息

	//以下为Flash操作的回调函数，不同flash有不同实现
	int (*_erase) (struct mtd_info *mtd, struct erase_info *instr);
	int (*_point) (struct mtd_info *mtd, loff_t from, size_t len, size_t *retlen, void **virt, resource_size_t *phys);
	int (*_unpoint) (struct mtd_info *mtd, loff_t from, size_t len);
	unsigned long (*_get_unmapped_area) (struct mtd_info *mtd,
					     unsigned long len,
					     unsigned long offset,
					     unsigned long flags);
	int (*_read) (struct mtd_info *mtd, loff_t from, size_t len,
		      size_t *retlen, u_char *buf);
	int (*_write) (struct mtd_info *mtd, loff_t to, size_t len,
		       size_t *retlen, const u_char *buf);
	int (*_panic_write) (struct mtd_info *mtd, loff_t to, size_t len, size_t *retlen, const u_char *buf);
	int (*_read_oob) (struct mtd_info *mtd, loff_t from,
			  struct mtd_oob_ops *ops);
	int (*_write_oob) (struct mtd_info *mtd, loff_t to,
			   struct mtd_oob_ops *ops);
	int (*_get_fact_prot_info) (struct mtd_info *mtd, size_t len, size_t *retlen, struct otp_info *buf);
	int (*_read_fact_prot_reg) (struct mtd_info *mtd, loff_t from, size_t len, size_t *retlen, u_char *buf);
	int (*_get_user_prot_info) (struct mtd_info *mtd, size_t len,  size_t *retlen, struct otp_info *buf);
	int (*_read_user_prot_reg) (struct mtd_info *mtd, loff_t from, size_t len, size_t *retlen, u_char *buf);
	int (*_write_user_prot_reg) (struct mtd_info *mtd, loff_t to, size_t len, size_t *retlen, u_char *buf);
	int (*_lock_user_prot_reg) (struct mtd_info *mtd, loff_t from,  size_t len);
	int (*_writev) (struct mtd_info *mtd, const struct kvec *vecs, unsigned long count, loff_t to, size_t *retlen);
	void (*_sync) (struct mtd_info *mtd);
	int (*_lock) (struct mtd_info *mtd, loff_t ofs, uint64_t len);
	int (*_unlock) (struct mtd_info *mtd, loff_t ofs, uint64_t len);
	int (*_is_locked) (struct mtd_info *mtd, loff_t ofs, uint64_t len);
	int (*_block_isreserved) (struct mtd_info *mtd, loff_t ofs);
	int (*_block_isbad) (struct mtd_info *mtd, loff_t ofs);
	int (*_block_markbad) (struct mtd_info *mtd, loff_t ofs);
	int (*_suspend) (struct mtd_info *mtd);
	void (*_resume) (struct mtd_info *mtd);
	void (*_reboot) (struct mtd_info *mtd);
	int (*_get_device) (struct mtd_info *mtd);
	void (*_put_device) (struct mtd_info *mtd);
	struct backing_dev_info *backing_dev_info;

	struct notifier_block reboot_notifier;  //重启通知
	struct mtd_ecc_stats ecc_stats; //ECC统计数据
	int subpage_sft;
	void *priv;
	struct module *owner;
	struct device dev;
	int usecount;
};
```
另外一个比较重要的数据结构为mtd_partition，顾名思义，对一块flash进行分区:
```C
struct mtd_partition {
	const char *name; //分区名称
	uint64_t size;	//大小
	uint64_t offset; //在MTD设备的偏移
	uint32_t mask_flags; //MTD主设备的掩码
}
```
通常每种类型的Flash芯片定义了这样一个数据结构，用于对Flash进行操作。可参考drivers/mtd/devices下相关使用（例如lpddr2_nvm.c）

mtdcore.h/mtdcore.c下定义了一系列使用MTD设备的API：
```C
struct mtd_info *__mtd_next_device(int i);
int add_mtd_device(struct mtd_info *mtd); //注册MTD设备
int del_mtd_device(struct mtd_info *mtd); //注销
int add_mtd_partitions(struct mtd_info *, const struct mtd_partition *, int); 
int del_mtd_partitions(struct mtd_info *); //添加、删除MTD分区

int parse_mtd_partitions(struct mtd_info *master, const char * const *types, struct mtd_partitions *pparts, struct mtd_part_parser_data *data); //从MTD设备查找MTD分区

void mtd_part_parser_cleanup(struct mtd_partitions *parts);//清除parse_mtd_partitions得到的MTD分区数据

int __init init_mtdchar(void);
void __exit cleanup_mtdchar(void); //注册、清除MTD字符设备

void register_mtd_user (struct mtd_notifier *new)
int unregister_mtd_user (struct mtd_notifier *old) //注册、去注册MTD使用者（当有MTD变动时通知）
```
### MTD字符设备

MTD字符设备定义在mtdchar.c，其中定义了字符设备的相关操作：
```C
static const struct file_operations mtd_fops = {
	.owner		= THIS_MODULE,
	.llseek		= mtdchar_lseek,
	.read		= mtdchar_read,
	.write		= mtdchar_write,
	.unlocked_ioctl	= mtdchar_unlocked_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl	= mtdchar_compat_ioctl,
#endif
	.open		= mtdchar_open,
	.release	= mtdchar_close,
	.mmap		= mtdchar_mmap,
#ifndef CONFIG_MMU
	.get_unmapped_area = mtdchar_get_unmapped_area,
	.mmap_capabilities = mtdchar_mmap_capabilities,
#endif
};
```
### MTD块设备

MTD块设备定义在mtdblock.c/mtdblock_ro.c，定义了块设备相关操作：
```C
static struct mtd_blktrans_ops mtdblock_tr = {
	.name		= "mtdblock",
	.major		= MTD_BLOCK_MAJOR,
	.part_bits	= 0,
	.blksize 	= 512,
	.open		= mtdblock_open,
	.flush		= mtdblock_flush,
	.release	= mtdblock_release,
	.readsect	= mtdblock_readsect,
	.writesect	= mtdblock_writesect,
	.add_mtd	= mtdblock_add_mtd,
	.remove_dev	= mtdblock_remove_dev,
	.owner		= THIS_MODULE,
};
static struct mtd_blktrans_ops mtdblock_tr = {
	.name		= "mtdblock",
	.major		= MTD_BLOCK_MAJOR,
	.part_bits	= 0,
	.blksize 	= 512,
	.readsect	= mtdblock_readsect,
	.writesect	= mtdblock_writesect,
	.add_mtd	= mtdblock_add_mtd,
	.remove_dev	= mtdblock_remove_dev,
	.owner		= THIS_MODULE,
};
```
### 分区

在嵌入式设备，一个flash往往会被划分做不同的功能分区。例如，升级分区、文件系统分区、bootloader分区、配置分区、备份分区等等。代码树中的drivers/mtd/mtdpart.c实现了分区的相关操作：

以下全局变量定义了MTD分区的链表：
```C
static LIST_HEAD(mtd_partitions);
static DEFINE_MUTEX(mtd_partitions_mutex);
```
链表元素为：
```C
struct mtd_part {
	struct mtd_info mtd;      //MTD信息
	struct mtd_info *master;  //MTD 设备指针
	uint64_t offset;          //主设备偏移
	struct list_head list;
};
```
相关操作为：
```C
static struct mtd_part *allocate_partition(struct mtd_info *master, const struct mtd_partition *part, int partno, uint64_t cur_offset)
//分配分区
static int mtd_add_partition_attrs(struct mtd_part *new)
int mtd_add_partition(struct mtd_info *master, const char *name, long long offset, long long length)
int mtd_del_partition(struct mtd_info *master, int partno)
//添加删除分区
int add_mtd_partitions(struct mtd_info *master, const struct mtd_partition *parts, int nbparts)
//根据分区表添加分区
int __register_mtd_parser(struct mtd_part_parser *p, struct module *owner)
void deregister_mtd_parser(struct mtd_part_parser *p)
//向内核注册和去注册回调函数用于MTD分区时执行
int parse_mtd_partitions(struct mtd_info *master, const char *const *types, struct mtd_partitions *pparts, struct mtd_part_parser_data *data)
```
### 参考文献

1.  《[General MTD documentation](http://www.linux-mtd.infradead.org/doc/general.html)》