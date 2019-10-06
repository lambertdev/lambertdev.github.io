---
title: Yaffs文件系统(3)-- 检查点(Checkpoint)机制
tags:
  - Linux
  - YAFFS
  - 文件系统
url: 2248.html
id: 2248
categories:
  - Linux
  - Linux文件系统
date: 2019-05-15 22:16:46
---

前文对Yaffs文件系统[框架](https://l2h.site/2019/04/30/yaffs-1/)及[Block](https://l2h.site/2019/05/14/yaffs-2/)管理，本文介绍Yaffs的检查点机制。什么是检查点机制？文件系统将目录结构存储在Nand Flash中一个特殊Block内，在Mount时快速加载，加速文件系统的加载。同时，机制也可以一定程度上避免因为突然掉电等因素造成的文件系统破坏。

本章先介绍文件的Tnode Tree，接着介绍Yaffs的checkpoint格式。

Tnode Tree
----------

Tnode Tree是Yaffs文件系统用来存储某个文件与其所在Nand Flash位置关系的一个树状结构。其定义如下：
```C
struct yaffs_tnode {
	struct yaffs_tnode *internal\[YAFFS_NTNODES_INTERNAL\];
};
```
可以看出，tnode存储为简单的指针数组，内部存储为指向下一级tnodes的指针。其结构如下：

![](https://l2h.site/wp-content/uploads/2019/05/tnode.png)

Tnode树结构

可以看出，只有最后一级（即Level 0）存放的是该文件所在Nand Flash的位置。具体如何存储的此处不做过多说明，请大家自行分析，主要为一些位计算。

Checkpoint存储格式
--------------

Checkpoint的存储格式如下图：

![](https://l2h.site/wp-content/uploads/2019/05/checkpoint-2-1024x792.png)

在Checkpoint内部存储的内容依次是：

*   1字节有效性标记
*   36字节的设备信息相关变量，储存第一章介绍的[yaffs_dev结构体](https://l2h.site/2019/04/30/yaffs-1/#i-4)必须的内容
*   该分区所有Nand块的信息，包括块内的chunk是否被使用
*   所有文件或目录的信息，与运行时RAM里存储的内容类似
*   1字节尾部有效性标记
*   检查点所有内容的校验和

需要说明的是Yaffs的检查点信息，会使用专门的块来存储。其obj_id为特殊的0x21，且checkpoint不存在chunk id为0的Object（即 Header Object）。一个简单的checkpoint存储内容如下：
```
+----------------------------+
|数 据                       |
|                            |
|                        数据|
+----------------------------+
|0x21 chunk_id(1) size ecc   |
+----------------------------+
|数据                        |
|                            |
|                        数据|
+----------------------------+
|0x21 chunk_id(2) size ecc   |
+----------------------------+
|                            |
|                            |
|                            |
|  ........................  |
|                            |
|                            |
|                            |
+----------------------------+
```
Checkpoint读写擦除
--------------

Checkpoint是按照前一节所介绍存储结构的顺序进行读写。

*   读Checkpoint的函数入口是yaffs2_checkpt_restore()->yaffs2_rd_checkpt_data()，一般在Mount时进行
*   写Checkpoint的函数入口是yaffs_checkpoint_save()->yaffs2_wr_checkpt_data()
*   擦除Checkpoint的函数入口是yaffs2_checkpt_invalidate()->yaffs2_checkpt_invalidate_stream()-> yaffs_checkpt_erase()

代码如下（相对比较容易理解不做过多注释）：
```C
int yaffs2_checkpt_restore(struct yaffs_dev *dev)
{
	retval = yaffs2_rd_checkpt_data(dev);
	if (dev->is_checkpointed) {
		yaffs_verify_objects(dev);
		yaffs_verify_blocks(dev);
		yaffs_verify_free_chunks(dev);
	}
	yaffs_trace(YAFFS_TRACE_CHECKPOINT, "restore exit: is_checkpointed %d", dev->is_checkpointed);
	return retval;
}
static int yaffs2_rd_checkpt_data(struct yaffs_dev *dev)
{
	int ok = 1; 
	if (!dev->param.is_yaffs2) ok = 0;
	if (ok && dev->param.skip_checkpt_rd) {
		yaffs_trace(YAFFS_TRACE_CHECKPOINT, "skipping checkpoint read");
		ok = 0;
	}

	if (ok)
		ok = yaffs2_checkpt_open(dev, 0); /* open for read */

	if (ok) {
		yaffs_trace(YAFFS_TRACE_CHECKPOINT, "read checkpoint validity");
		ok = yaffs2_rd_checkpt_validity_marker(dev, 1);
	}
	if (ok) {
		yaffs_trace(YAFFS_TRACE_CHECKPOINT, "read checkpoint device");
		ok = yaffs2_rd_checkpt_dev(dev);
	}
	if (ok) {
		yaffs_trace(YAFFS_TRACE_CHECKPOINT, "read checkpoint objects");
		ok = yaffs2_rd_checkpt_objs(dev);
	}
	if (ok) {
		yaffs_trace(YAFFS_TRACE_CHECKPOINT,"read checkpoint validity");
		ok = yaffs2_rd_checkpt_validity_marker(dev, 0);
	}
	if (ok) {
		ok = yaffs2_rd_checkpt_sum(dev);
		yaffs_trace(YAFFS_TRACE_CHECKPOINT,"read checkpoint checksum %d", ok);
	}

	if (!yaffs_checkpt_close(dev))ok = 0;

	if (ok) dev->is_checkpointed = 1;
	else dev->is_checkpointed = 0;
	return ok ? 1 : 0;
}

int yaffs_checkpoint_save(struct yaffs_dev *dev)
{
	yaffs_trace(YAFFS_TRACE_CHECKPOINT, "save entry: is_checkpointed %d", dev->is_checkpointed);
	yaffs_verify_objects(dev);
	yaffs_verify_blocks(dev);
	yaffs_verify_free_chunks(dev);
	if (!dev->is_checkpointed) {
		yaffs2_checkpt_invalidate(dev);
		yaffs2_wr_checkpt_data(dev);
	}
	yaffs_trace(YAFFS_TRACE_CHECKPOINT | YAFFS_TRACE_MOUNT, "save exit: is_checkpointed %d", dev->is_checkpointed);

	return dev->is_checkpointed;
}
static int yaffs2_wr_checkpt_data(struct yaffs_dev *dev)
{
	if (!yaffs2_checkpt_required(dev)) {
		yaffs_trace(YAFFS_TRACE_CHECKPOINT,"skipping checkpoint write");ok = 0;
	}

	if (ok) ok = yaffs2_checkpt_open(dev, 1);

	if (ok) {
		yaffs_trace(YAFFS_TRACE_CHECKPOINT, "write checkpoint validity"); ok = yaffs2_wr_checkpt_validity_marker(dev, 1);
	}
	if (ok) {
		yaffs_trace(YAFFS_TRACE_CHECKPOINT, "write checkpoint device"); ok = yaffs2_wr_checkpt_dev(dev);
	}
	if (ok) {
		yaffs_trace(YAFFS_TRACE_CHECKPOINT, "write checkpoint objects"); ok = yaffs2_wr_checkpt_objs(dev);
	}
	if (ok) {
		yaffs_trace(YAFFS_TRACE_CHECKPOINT, "write checkpoint validity"); ok = yaffs2_wr_checkpt_validity_marker(dev, 0);
	}
	if (ok) ok = yaffs2_wr_checkpt_sum(dev);
	if (!yaffs_checkpt_close(dev)) ok = 0;

	if (ok) dev->is_checkpointed = 1;
	else dev->is_checkpointed = 0;

	return dev->is_checkpointed;
}
```
总结
--

以上内容即为Yaffs的Checkpoint机制，接下来将会介绍GC（Garbage Collection，垃圾回收）机制。