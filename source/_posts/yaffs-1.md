---
title: Yaffs文件系统(1)-- 概述
tags:
  - Linux
  - YAFFS
  - 文件系统
url: 2143.html
id: 2143
categories:
  - Linux文件系统
date: 2019-04-30 22:12:19
---

[Yaffs](https://yaffs.net/)全称 Yet Another Flash File System，是一种嵌入式设备常用文件系统。它对基于Flash的存储设备（NAND 或NOR Flash）特别是NAND Flash有很好的支持，其具有速度快、鲁棒性强等特点。此外，其对各种操作系统的支持也很好。  
目前Yaffs已经更新到了版本2，而本文从源码角度对Yaffs2文件系统进行分析 。希望对知识进行巩固的同时，也能对大家有所帮助。其中理解若有不到位的地方，也麻烦大家指正。

概述
--

先看Yaffs在整个系统中的位置（注：本文无特别说明均以搭载Linux OS为例）：

*   YAFFS位于虚拟文件系统之下，为VFS提供调用接口
*   YAFFS使用Linux MTD 子系统提供的服务对Flash进行操作。
    *   为什么直接使用Nand或者Nor Flash 驱动直接操作？这就涉及到Linux内核的设计思想：尽可能的对架构进行抽象，避免上下层直接调用，产生N*N的组合，节省了内核驱动设计者的开发负担，同时降低系统的代码复杂度。

  +---------------------------+
  |   用 户 空 间 态           |
  +---------------------------+
--------------------------------------------
  +---------------------------+
  |     VFS 虚 拟 文 件 系 统  |
  +---------------------------+ 内
  +------+ +------+ +---------+
  |File  | |File  | | YAFFS   | 核
  |Sys-1 | |Sys-2 | |         |
  +------+ +------+ +---------+ 态
  +---------------+ +---------+
  |               | | MTD     |
  |               | | Sub SYS |
  | ............  | +---------+
  |               | +---------+
  |               | | Nand or |
  |               | | NOR DrV |
  +---------------+ +---------+
--------------------------------------------
  +---------------------------+
  |      硬   件              |
  +---------------------------+

Yaffs2主要有以下功能模块

*   Flash的块/Chunk分配管理
*   检查点机制
*   垃圾回收机制
*   缓存机制
*   Proc调试功能

YAFFS文件系统格式
-----------

如下图，按照由大到小，Flash一般如下组成：

*   Block（块）：Block一般为Flash擦除的基本单位。由多个页及对应OOB区域组成， 跟据Flash控制器和Flash颗粒的不同， 组成Block的page数量也不相同
*   Page（页）：Flash的页，用来存放真正数据的基本单位。在Yaffs文件系统管理里称作数据Chunk。
*   OOB（带外数据）：与页对应，紧跟在页的后边，一般存放ECC校验信息，Flash坏块标记等信息。

+--------------------+
| Flash 页           |----->存放文件内容
|                    |
|                    |
|                    |
+------------------------->Flash Chunk 1
| Flash Spare区域     |----->存放文件相关信息的Spare区域
+--------------------+
|                    |
|                    |
|                    |
|                    |
+------------------------->Flash Chunk 2
|                    |
+--------------------+
|                    |
|  ...............   +---->Flash Chunks
+--------------------+
|                    |
|                    |
|                    |
|                    |
+------------------------->Flash Chunk N
|                    |
+--------------------+

一个典型的普通文件在 YAFFS 文件系统格式Flash的存储形式如下图所示：

*   YAFFS将文件头部信息（包括文件类型，文件名，父目录的obj_id等）  
    单独 存放在page的起始位置。
*   文件的唯一标示id(也叫obj_id)位于OOB区域，紧跟chunk_id标示这个是文件的第几块数据，Chunk_id 0表示该page为文件头信息，不含文件内容
*   真正的文件内容根据文件大小散落在的size/page_size+1个页内。若文件大小正好不是页大小的整数倍，那么最后一段文件内容后会填充全0xFF放在一个页内，Page对应的OOB区域
*   因为文件可能会被YAFFS管理（例如，上层应用对文件改写）的关系，同一个文件的内容不一定会摆放在相邻的Flash 页内（刚刚使用mkyaffs创建的文件系统除外）。同时，对同一个文件，可能会存在多个相同obj_id肯chunk_id的文件页，此时有另外的标示seq number来决定选择哪个page

+----------------------------+
|类型 父目录id 文件名          |
|....                        |
|                            |
+----------------------------+
|id chunk_id(0) size   ecc   |
+----------------------------+
|                            |
|                            |
|                            |
|  ........................  |
|                            |
|                            |
|                            |
|                            |
+----------------------------+
|数 据                       |
|                            |
|                        数据|
+----------------------------+
|id chunk_id(1) size ecc     |
+----------------------------+
|数据                        |
|                            |
|                        数据|
+----------------------------+
|id chunk_id(2) size   ecc   |
+----------------------------+

源码结构
----

真正开始源码分析前，我们先看一下YAFFS的源码结构。Yaffs的核心代码如下：

yaffs_allocator.c

yaffs_obj和Tnode的分配器

yaffs_bitmap.c

管理块和Chunk的位图的代码  

yaffs_checkpointrw.c

读写Checkpoint数据的代码  

yaffs_ecc.c

Yaffs ECC相关代码（使用[Hamming ECC](https://en.wikipedia.org/wiki/Hamming_code)）  

yaffs_guts.c

Yaffs 主代码.

yaffs_nameval.c

处理Yaffs扩展标签的diamante (xattr).

yaffs_nand.c

Flash 接口抽象.

yaffs_packedtags1.c  
yaffs_packedtags2.c

打包Yaffs标签代码

yaffs_summary.c

处理Flash块Summary的代码.

yaffs_tagscompat.c

兼容Yaffs1模式的标签处理代码  

yaffs_tagsmarshall.c

标签组织代码

yaffs_verify.c

用于检查有效性的代码（例如，头部格式检查等）

yaffs_yaffs1.c

Yaffs1 模式特有代码.

yaffs_yaffs2.c

Yaffs2 模式特有代码.

yaffs_attribs.c  

文件属性处理相关代码

yaffs_error.c

存放Yaffs 错误代码

yaffsfs.c

Yaffs文件系统接口wrapper（支援多文件系统时用）

yaffs_hweight.c

用来检查Flash坏块标记的一些功能代码

yaffs_qsort.c

Yaffs使用的一些排序函数

初始化
---

YAFFS的初始化很简单，源代码如下：
```C
module_init(init_yaffs_fs)
static int __init init_yaffs_fs(void)
{
	int error = 0;
	struct file_system_to_install *fsinst;

	mutex_init(&yaffs_context_lock);        

	error = yaffs_procfs_init();                        ----1
	if (error)
		return error;

	fsinst = fs_to_install;

	while (fsinst->fst && !error) {
		error = register_filesystem(fsinst->fst);   ----2
		if (!error)
			fsinst->installed = 1;
		fsinst++;
	}
	/* Any errors? uninstall  */                        ----3
	if (error) {
		fsinst = fs_to_install;
		while (fsinst->fst) {
			if (fsinst->installed) {
				unregister_filesystem(fsinst->fst);
				fsinst->installed = 0;
			}
			fsinst++;
	}}
	return error;
}
static struct file_system_to_install fs_to_install\[\] = {
	{&yaffs_fs_type, 0},
	{&yaffs2_fs_type, 0},
	{NULL, 0}
};
static struct file_system_type yaffs2_fs_type = {
	.owner = THIS_MODULE,
	.name = "yaffs2",
	.mount = yaffs2_mount,
	.kill_sb = kill_block_super,
	.fs_flags = FS_REQUIRES_DEV,
};
```
*   第1段初始化系统的procfs
*   第2段向Kernel注册yaffs文件系统，这里不光注册yaffs2也会注册yaffs1，所以可以看到是一个while循环
*   第3段：如果注册文件系统发生问题，解注册文件系统，此处不做过多分析

数据结构
----

被成功注册到内核后，YAFFS利用内存中如下重要的数据结构来对文件系统进行管理：

*   _**yaffs_dev：**_YAFFS Partition或者挂载点的相关信息
*   _**yaffs_block_info**：_Nand Flash 块的信息，一个yaffs partition会有多个block info
*   _**yaffs_obj：**_记录文件系统中的文件或者目录相关信息
*   _**yaffs_tnode：**_记录文件系统的目录结构，以及文件chunk位置相关信息

此外，YAFFS会使用少量的内存缓存（Cache）来提升访问性能。

yaffs_dev定义如下：
```C
struct yaffs_dev {
	struct yaffs_param param;          //Yaffs相关参数
	struct yaffs_driver drv;           //驱动处理相关函数
	struct yaffs_tags_handler tagger;  //Yaffs标签处理相关函数
        //操作系统相关参数
	void *os_context;
	void *driver_context;

	struct list_head dev_list;
	int ll_init;
	u32 data_bytes_per_chunk;

	u16 chunk_grp_bits;
	u16 chunk_grp_size;
        //TNode相关参数
	struct yaffs_tnode *tn_swap_buffer;
	u32 tnode_width;
	u32 tnode_mask;
	u32 tnode_size;
        //Chunk大小相关参数
	u32 chunk_shift;	
	u32 chunk_div;		
	u32 chunk_mask;		
        //与文件系统挂载相关标识符
	int is_mounted;
	int read_only;
	int is_checkpointed;
	int swap_endian;	
        //块管理相关参数
	u32 internal_start_block;
	u32 internal_end_block;
	int block_offset;
	int chunk_offset;
        //检查点相关参数
	int checkpt_page_seq;
        ..............
	//块管理相关参数
	struct yaffs_block_info *block_info;
	u8 *chunk_bits;		/* bitmap of chunks in use */
	u8 block_info_alt:1;	/* allocated using alternative alloc */
	u8 chunk_bits_alt:1;	/* allocated using alternative alloc */
	int chunk_bit_stride;	
	int n_erased_blocks;
	int alloc_block;	
	u32 alloc_page;
	int alloc_block_finder;	

	/* Tnode或者Object内存分配相关参数 */
	void *allocator;
	int n_obj;
	int n_tnodes;
	int n_hardlinks;

	struct yaffs_obj_bucket obj_bucket\[YAFFS_NOBJECT_BUCKETS\];
	u32 bucket_finder;
	int n_free_chunks; 
        //垃圾回收相关参数
	u32 *gc_cleanup_list;
       ................
	//根目录及lost and found目录
	struct yaffs_obj *root_dir;
	struct yaffs_obj *lost_n_found;

	int buffered_block;	/* Which block is buffered here? */
	int doing_buffered_block_rewrite;
        //Yaffs缓存相关
	struct yaffs_cache *cache;
	int cache_last_use;

	/* 后台删除或者unlink文件相关参数 */
	struct yaffs_obj *unlinked_dir;	
	struct yaffs_obj *del_dir;	
	struct yaffs_obj *unlinked_deletion;
	int n_deleted_files;	
	int n_unlinked_files;	/* Count of unlinked files. */
	int n_bg_deletions;	/* Count of background deletions. */
	/* 临时缓存相关 */
	struct yaffs_buffer temp_buffer\[YAFFS_N_TEMP_BUFFERS\];
	int max_temp;
	int temp_in_use;
	int unmanaged_buffer_allocs;
	int unmanaged_buffer_deallocs;

	unsigned seq_number;	
	unsigned oldest_dirty_seq;
	unsigned oldest_dirty_block;

	int refresh_skip;	
	struct list_head dirty_dirs;	
	int chunks_per_summary;
	struct yaffs_summary_tags *sum_tags;

	/* 统计数据*/
	u32 n_page_writes;
        .......
};
```
yaffs_param 主要包含文件系统相关参数。
```C
struct yaffs_param {
	const YCHAR *name;      //文件系统对应的MTD设备名称
	int inband_tags;	/* Use unband tags */
	u32 total_bytes_per_chunk;//每个chunk的大小（字节为单位）
	u32 chunks_per_block; //每个Flash块的chunk数量
	u32 spare_bytes_per_chunk;//每个chunk后的spare区域大小（字节为单位）
	u32 start_block; //第一个可以使用的块	
        u32 end_block;	//最后一个可使用的块
	u32 n_reserved_blocks; //保留的块数量，主要为垃圾回收机制使用	
	u32 n_caches;//Yaffs高速缓存的数量。Flash颗粒的写擦除是有次数上限的，为了避免读写端数据时Flash的频繁访问影响flash寿命，且避免访问Flash的性能下降，Yaffs定义了一些磁盘高速缓存在内存中
	int cache_bypass_aligned; //若值为1：当读写是从chunk align的位置开始，便不使用cache
	int use_nand_ecc;	//使用Nand Driver的ECC方法
	int tags_9bytes;	/* Use 9 byte tags */
	int no_tags_ecc;	//是否对Tag使用ECC
	int is_yaffs2;		//是否是Yaffs2
	int empty_lost_n_found;	
	int refresh_period;	//检查Block的时间间隔
	u8 skip_checkpt_rd;     //是否跳过checkpoint检查
	u8 skip_checkpt_wr;
	int enable_xattr;	/* Enable xattribs */
	int max_objects;	//最大的yaffs object数量
	int hide_lost_n_found;  //是否隐藏lost and found目录
	int stored_endian;      //大小端
	void (*remove_obj_fn) (struct yaffs_obj *obj); //移除object的函数回调
	void (*sb_dirty_fn) (struct yaffs_dev *dev);//超级块标记脏函数
	unsigned (*gc_control_fn) (struct yaffs_dev *dev);//垃圾回收管理的回调函数
        //以下为debug相关标记
	int use_header_file_size;	
	int disable_lazy_load;	
	int wide_tnodes_disabled;	
	int disable_soft_del;	
	int defered_dir_update;	

	int auto_unicode;
	int always_check_erased;
	int disable_summary;
	int disable_bad_block_marking;
};
```
以下为yaffs标签相关处理函数，在yaffs_tags_marshall_install函数初始化。源码如下：
```C
struct yaffs_tags_handler {
	int (*write_chunk_tags_fn) (struct yaffs_dev *dev,int nand_chunk, const u8 *data,const struct yaffs_ext_tags *tags);
	int (*read_chunk_tags_fn) (struct yaffs_dev *dev, int nand_chunk, u8 *data,struct yaffs_ext_tags *tags);
	int (*query_block_fn) (struct yaffs_dev *dev, int block_no,enum yaffs_block_state *state, u32 *seq_number);
	int (*mark_bad_fn) (struct yaffs_dev *dev, int block_no);
};
void yaffs_tags_marshall_install(struct yaffs_dev *dev)
{
	if (!dev->param.is_yaffs2)
		return;
	if (!dev->tagger.write_chunk_tags_fn)
		dev->tagger.write_chunk_tags_fn = yaffs_tags_marshall_write;
	if (!dev->tagger.read_chunk_tags_fn)
		dev->tagger.read_chunk_tags_fn = yaffs_tags_marshall_read;
	if (!dev->tagger.query_block_fn)
		dev->tagger.query_block_fn = yaffs_tags_marshall_query_block;
	if (!dev->tagger.mark_bad_fn)
		dev->tagger.mark_bad_fn = yaffs_tags_marshall_mark_bad;
}
```
yaffs_driver定义了Yaffs驱动相关，读写擦除，坏块检查等操作MTD设备的接口函数，其初始化位于yaffs_mtd_drv_install函数。代码如下：
```C
struct yaffs_driver {
	int (*drv_write_chunk_fn) (struct yaffs_dev *dev, int nand_chunk, const u8 *data, int data_len, const u8 *oob, int oob_len);
	int (*drv_read_chunk_fn) (struct yaffs_dev *dev, int nand_chunk, u8 *data, int data_len,u8 *oob, int oob_len,enum yaffs_ecc_result *ecc_result);
	int (*drv_erase_fn) (struct yaffs_dev *dev, int block_no);
	int (*drv_mark_bad_fn) (struct yaffs_dev *dev, int block_no);
	int (*drv_check_bad_fn) (struct yaffs_dev *dev, int block_no);
	int (*drv_initialise_fn) (struct yaffs_dev *dev);
	int (*drv_deinitialise_fn) (struct yaffs_dev *dev);
};
void yaffs_mtd_drv_install(struct yaffs_dev *dev)
{
	struct yaffs_driver *drv = &dev->drv;

	drv->drv_write_chunk_fn = yaffs_mtd_write;
	drv->drv_read_chunk_fn = yaffs_mtd_read;
	drv->drv_erase_fn = yaffs_mtd_erase;
	drv->drv_mark_bad_fn = yaffs_mtd_mark_bad;
	drv->drv_check_bad_fn = yaffs_mtd_check_bad;
	drv->drv_initialise_fn = yaffs_mtd_initialise;
	drv->drv_deinitialise_fn = yaffs_mtd_deinitialise;
}
```
以下为Yaffs选项，为yaffs系统挂载（Mount）时配置，最后被传给yaffs_dev对应栏位。
```C
struct yaffs_options {
	int inband_tags;         
	int skip_checkpoint_read;
	int skip_checkpoint_write;
	int no_cache;
	int tags_ecc_on;
	int tags_ecc_overridden;
	int lazy_loading_enabled;
	int lazy_loading_overridden;
	int empty_lost_and_found;
	int empty_lost_and_found_overridden;
};
```
yaffs对应Linux对应上下文：
```C
struct yaffs_linux_context {
	struct list_head context_list;	
	struct yaffs_dev *dev; //指向Yaffs Dev的回调
	struct super_block *super; //指向超级块指针
	struct task_struct *bg_thread;	//做GC的后台线程
	int bg_running; //后台线程启动标记
	struct mutex gross_lock; //大粒度锁，用来保护文件系统关键字段
	u8 *spare_buffer;	//Spare缓冲区
	struct list_head search_contexts; //目录查找上下文链表
	struct task_struct *readdir_process; //标记当前目录读取的task
	unsigned mount_id; //yaffs2挂载的id号，对每个yaffs文件系统来讲标号唯一
	int dirty;
};
```
文件系统挂载
------

挂载后，整个文件系统才会为操作系统所用。YAFFS文件系统的挂载过程如下：
```C
//其实yaffs2挂载就是执行内核的mount_bdev函数，最后当然也会执行到传入的回调函数yaffs_internal_read_super
static struct dentry *yaffs2_mount(struct file_system_type *fs_type, int flags,const char *dev_name, void *data)
{
        return mount_bdev(fs_type, flags, dev_name, data, yaffs2_internal_read_super_mtd);   
}
static int yaffs2_internal_read_super_mtd(struct super_block *sb, void *data, int silent)
{
	return yaffs_internal_read_super(2, sb, data, silent) ? 0 : -EINVAL;
}

static struct super_block *yaffs_internal_read_super(int yaffs_version, struct super_block *sb,  void *data, int silent)
{
        .......
        //超级块的一些初使设置
	sb->s_magic = YAFFS_MAGIC;
	sb->s_op = &yaffs_super_ops;
	sb->s_flags |= MS_NOATIME;
	read_only = ((sb->s_flags & MS_RDONLY) != 0);
	sb->s_export_op = &yaffs_export_ops;
        .......
	if (yaffs_parse_options(&options, data_str))
		/* Option parsing failed */
		return NULL;
        .......
	sb->s_blocksize = PAGE_CACHE_SIZE;
	sb->s_blocksize_bits = PAGE_CACHE_SHIFT;
        .......
	mtd = get_mtd_device(NULL, MINOR(sb->s_dev));
        .......
        //若文件分区是MTD device类型的Nand Flash，开始着手建立整个yaffs_dev
	if (!read_only && !(mtd->flags & MTD_WRITEABLE)) {
		read_only = 1;
		sb->s_flags |= MS_RDONLY;
	}
	dev = kmalloc(sizeof(struct yaffs_dev), GFP_KERNEL);
	context = kmalloc(sizeof(struct yaffs_linux_context), GFP_KERNEL);
        .......
	memset(dev, 0, sizeof(struct yaffs_dev));
	param = &(dev->param);

	memset(context, 0, sizeof(struct yaffs_linux_context));
	dev->os_context = context;
	INIT_LIST_HEAD(&(context->context_list));
	context->dev = dev;
	context->super = sb;

	dev->read_only = read_only;
	sb->s_fs_info = dev;
	dev->driver_context = mtd;
        
        // 设定yaffs_param相关参数
        ...............

	mutex_lock(&yaffs_context_lock);

	for (mount_id = 0, found = 0; !found; mount_id++) {
		found = 1;
		list_for_each(l, &yaffs_context_list) {
			context_iterator =list_entry(l, struct yaffs_linux_context,context_list);
			if (context_iterator->mount_id == mount_id)
				found = 0;
		}
	}
	context->mount_id = mount_id;

	list_add_tail(&(yaffs_dev_to_lc(dev)->context_list),
		      &yaffs_context_list);
	mutex_unlock(&yaffs_context_lock);

	mutex_init(&(yaffs_dev_to_lc(dev)->gross_lock));
	yaffs_gross_lock(dev);
        //建立yaffs_dev参数，以及初始化Tnode
	err = yaffs_guts_initialise(dev);
        //执行yaffs的后台线程
	if (err == YAFFS_OK) yaffs_bg_start(dev);
	if (!context->bg_thread)
		param->defered_dir_update = 0;
	sb->s_maxbytes = yaffs_max_file_size(dev);
	yaffs_gross_unlock(dev);

	//创建文件系统根节点
	if (err == YAFFS_OK)
		inode = yaffs_get_inode(sb, S_IFDIR | 0755, 0, yaffs_root(dev));
        .......
	inode->i_op = &yaffs_dir_inode_operations;
	inode->i_fop = &yaffs_dir_operations;
	root = d_alloc_root(inode);
        .......
	sb->s_root = root;
	sb->s_dirt = !dev->is_checkpointed;
	return sb;
}
```
小结
--

本文介绍了Yaffs的架构、文件结构及初始化流程。之后章节介绍：

*   checkpoint机制
*   YAFFS参数调优及Debug
*   .......