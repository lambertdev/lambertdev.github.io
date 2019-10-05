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
\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-
  +---------------------------+
  |     VFS 虚 拟 文 件 系 统  |
  \+\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\+ 内
  \+\-\-\-\-\-\-\+ \+\-\-\-\-\-\-\+ +---------+
  |File  | |File  | | YAFFS   | 核
  |Sys-1 | |Sys-2 | |         |
  \+\-\-\-\-\-\-\+ \+\-\-\-\-\-\-\+ \+\-\-\-\-\-\-\-\-\-\+ 态
  \+\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\+ +---------+
  |               | | MTD     |
  |               | | Sub SYS |
  | ............  | +---------+
  |               | +---------+
  |               | | Nand or |
  |               | | NOR DrV |
  \+\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\+ +---------+
\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-
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
*   文件的唯一标示id(也叫obj\_id)位于OOB区域，紧跟chunk\_id标示这个是文件的第几块数据，Chunk_id 0表示该page为文件头信息，不含文件内容
*   真正的文件内容根据文件大小散落在的size/page_size+1个页内。若文件大小正好不是页大小的整数倍，那么最后一段文件内容后会填充全0xFF放在一个页内，Page对应的OOB区域
*   因为文件可能会被YAFFS管理（例如，上层应用对文件改写）的关系，同一个文件的内容不一定会摆放在相邻的Flash 页内（刚刚使用mkyaffs创建的文件系统除外）。同时，对同一个文件，可能会存在多个相同obj\_id肯chunk\_id的文件页，此时有另外的标示seq number来决定选择哪个page

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

module\_init(init\_yaffs_fs)
static int \_\_init init\_yaffs_fs(void)
{
	int error = 0;
	struct file\_system\_to_install *fsinst;

	mutex\_init(&yaffs\_context_lock);        

	error = yaffs\_procfs\_init();                        ----1
	if (error)
		return error;

	fsinst = fs\_to\_install;

	while (fsinst->fst && !error) {
		error = register_filesystem(fsinst->fst);   ----2
		if (!error)
			fsinst->installed = 1;
		fsinst++;
	}
	/\* Any errors? uninstall  */                        ----3
	if (error) {
		fsinst = fs\_to\_install;
		while (fsinst->fst) {
			if (fsinst->installed) {
				unregister_filesystem(fsinst->fst);
				fsinst->installed = 0;
			}
			fsinst++;
	}}
	return error;
}
static struct file\_system\_to\_install fs\_to_install\[\] = {
	{&yaffs\_fs\_type, 0},
	{&yaffs2\_fs\_type, 0},
	{NULL, 0}
};
static struct file\_system\_type yaffs2\_fs\_type = {
	.owner = THIS_MODULE,
	.name = "yaffs2",
	.mount = yaffs2_mount,
	.kill\_sb = kill\_block_super,
	.fs\_flags = FS\_REQUIRES_DEV,
};

*   第1段初始化系统的procfs
*   第2段向Kernel注册yaffs文件系统，这里不光注册yaffs2也会注册yaffs1，所以可以看到是一个while循环
*   第3段：如果注册文件系统发生问题，解注册文件系统，此处不做过多分析

数据结构
----

被成功注册到内核后，YAFFS利用内存中如下重要的数据结构来对文件系统进行管理：

*   _**yaffs_dev：**_YAFFS Partition或者挂载点的相关信息
*   _**yaffs\_block\_info**：_Nand Flash 块的信息，一个yaffs partition会有多个block info
*   _**yaffs_obj：**_记录文件系统中的文件或者目录相关信息
*   _**yaffs_tnode：**_记录文件系统的目录结构，以及文件chunk位置相关信息

此外，YAFFS会使用少量的内存缓存（Cache）来提升访问性能。

yaffs_dev定义如下：

struct yaffs_dev {
	struct yaffs_param param;          //Yaffs相关参数
	struct yaffs_driver drv;           //驱动处理相关函数
	struct yaffs\_tags\_handler tagger;  //Yaffs标签处理相关函数
        //操作系统相关参数
	void *os_context;
	void *driver_context;

	struct list\_head dev\_list;
	int ll_init;
	u32 data\_bytes\_per_chunk;

	u16 chunk\_grp\_bits;
	u16 chunk\_grp\_size;
        //TNode相关参数
	struct yaffs\_tnode *tn\_swap_buffer;
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
	u32 internal\_start\_block;
	u32 internal\_end\_block;
	int block_offset;
	int chunk_offset;
        //检查点相关参数
	int checkpt\_page\_seq;
        ..............
	//块管理相关参数
	struct yaffs\_block\_info *block_info;
	u8 \*chunk_bits;		/\* bitmap of chunks in use */
	u8 block\_info\_alt:1;	/* allocated using alternative alloc */
	u8 chunk\_bits\_alt:1;	/* allocated using alternative alloc */
	int chunk\_bit\_stride;	
	int n\_erased\_blocks;
	int alloc_block;	
	u32 alloc_page;
	int alloc\_block\_finder;	

	/\* Tnode或者Object内存分配相关参数 */
	void *allocator;
	int n_obj;
	int n_tnodes;
	int n_hardlinks;

	struct yaffs\_obj\_bucket obj\_bucket\[YAFFS\_NOBJECT_BUCKETS\];
	u32 bucket_finder;
	int n\_free\_chunks; 
        //垃圾回收相关参数
	u32 *gc\_cleanup\_list;
       ................
	//根目录及lost and found目录
	struct yaffs\_obj *root\_dir;
	struct yaffs\_obj *lost\_n_found;

	int buffered_block;	/* Which block is buffered here? */
	int doing\_buffered\_block_rewrite;
        //Yaffs缓存相关
	struct yaffs_cache *cache;
	int cache\_last\_use;

	/\* 后台删除或者unlink文件相关参数 */
	struct yaffs\_obj *unlinked\_dir;	
	struct yaffs\_obj *del\_dir;	
	struct yaffs\_obj *unlinked\_deletion;
	int n\_deleted\_files;	
	int n\_unlinked\_files;	/* Count of unlinked files. */
	int n\_bg\_deletions;	/* Count of background deletions. */
	/\* 临时缓存相关 */
	struct yaffs\_buffer temp\_buffer\[YAFFS\_N\_TEMP_BUFFERS\];
	int max_temp;
	int temp\_in\_use;
	int unmanaged\_buffer\_allocs;
	int unmanaged\_buffer\_deallocs;

	unsigned seq_number;	
	unsigned oldest\_dirty\_seq;
	unsigned oldest\_dirty\_block;

	int refresh_skip;	
	struct list\_head dirty\_dirs;	
	int chunks\_per\_summary;
	struct yaffs\_summary\_tags *sum_tags;

	/\* 统计数据*/
	u32 n\_page\_writes;
        .......
};

yaffs_param 主要包含文件系统相关参数。

struct yaffs_param {
	const YCHAR *name;      //文件系统对应的MTD设备名称
	int inband_tags;	/* Use unband tags */
	u32 total\_bytes\_per_chunk;//每个chunk的大小（字节为单位）
	u32 chunks\_per\_block; //每个Flash块的chunk数量
	u32 spare\_bytes\_per_chunk;//每个chunk后的spare区域大小（字节为单位）
	u32 start_block; //第一个可以使用的块	
        u32 end_block;	//最后一个可使用的块
	u32 n\_reserved\_blocks; //保留的块数量，主要为垃圾回收机制使用	
	u32 n_caches;//Yaffs高速缓存的数量。Flash颗粒的写擦除是有次数上限的，为了避免读写端数据时Flash的频繁访问影响flash寿命，且避免访问Flash的性能下降，Yaffs定义了一些磁盘高速缓存在内存中
	int cache\_bypass\_aligned; //若值为1：当读写是从chunk align的位置开始，便不使用cache
	int use\_nand\_ecc;	//使用Nand Driver的ECC方法
	int tags_9bytes;	/* Use 9 byte tags */
	int no\_tags\_ecc;	//是否对Tag使用ECC
	int is_yaffs2;		//是否是Yaffs2
	int empty\_lost\_n_found;	
	int refresh_period;	//检查Block的时间间隔
	u8 skip\_checkpt\_rd;     //是否跳过checkpoint检查
	u8 skip\_checkpt\_wr;
	int enable_xattr;	/* Enable xattribs */
	int max_objects;	//最大的yaffs object数量
	int hide\_lost\_n_found;  //是否隐藏lost and found目录
	int stored_endian;      //大小端
	void (\*remove\_obj\_fn) (struct yaffs_obj \*obj); //移除object的函数回调
	void (\*sb\_dirty\_fn) (struct yaffs_dev \*dev);//超级块标记脏函数
	unsigned (\*gc\_control\_fn) (struct yaffs_dev \*dev);//垃圾回收管理的回调函数
        //以下为debug相关标记
	int use\_header\_file_size;	
	int disable\_lazy\_load;	
	int wide\_tnodes\_disabled;	
	int disable\_soft\_del;	
	int defered\_dir\_update;	

	int auto_unicode;
	int always\_check\_erased;
	int disable_summary;
	int disable\_bad\_block_marking;
};

以下为yaffs标签相关处理函数，在yaffs\_tags\_marshall_install函数初始化。源码如下：

struct yaffs\_tags\_handler {
	int (\*write\_chunk\_tags\_fn) (struct yaffs\_dev \*dev,int nand\_chunk, const u8 \*data,const struct yaffs\_ext_tags \*tags);
	int (\*read\_chunk\_tags\_fn) (struct yaffs\_dev \*dev, int nand\_chunk, u8 \*data,struct yaffs\_ext_tags \*tags);
	int (\*query\_block\_fn) (struct yaffs\_dev \*dev, int block\_no,enum yaffs\_block\_state \*state, u32 \*seq_number);
	int (\*mark\_bad\_fn) (struct yaffs\_dev \*dev, int block\_no);
};
void yaffs\_tags\_marshall\_install(struct yaffs\_dev *dev)
{
	if (!dev->param.is_yaffs2)
		return;
	if (!dev->tagger.write\_chunk\_tags_fn)
		dev->tagger.write\_chunk\_tags\_fn = yaffs\_tags\_marshall\_write;
	if (!dev->tagger.read\_chunk\_tags_fn)
		dev->tagger.read\_chunk\_tags\_fn = yaffs\_tags\_marshall\_read;
	if (!dev->tagger.query\_block\_fn)
		dev->tagger.query\_block\_fn = yaffs\_tags\_marshall\_query\_block;
	if (!dev->tagger.mark\_bad\_fn)
		dev->tagger.mark\_bad\_fn = yaffs\_tags\_marshall\_mark\_bad;
}

yaffs\_driver定义了Yaffs驱动相关，读写擦除，坏块检查等操作MTD设备的接口函数，其初始化位于yaffs\_mtd\_drv\_install函数。代码如下：

struct yaffs_driver {
	int (\*drv\_write\_chunk\_fn) (struct yaffs\_dev \*dev, int nand\_chunk, const u8 \*data, int data\_len, const u8 \*oob, int oob_len);
	int (\*drv\_read\_chunk\_fn) (struct yaffs\_dev \*dev, int nand\_chunk, u8 \*data, int data\_len,u8 \*oob, int oob\_len,enum yaffs\_ecc\_result *ecc\_result);
	int (\*drv\_erase\_fn) (struct yaffs\_dev \*dev, int block\_no);
	int (\*drv\_mark\_bad\_fn) (struct yaffs\_dev \*dev, int block_no);
	int (\*drv\_check\_bad\_fn) (struct yaffs\_dev \*dev, int block_no);
	int (\*drv\_initialise\_fn) (struct yaffs_dev \*dev);
	int (\*drv\_deinitialise\_fn) (struct yaffs_dev \*dev);
};
void yaffs\_mtd\_drv\_install(struct yaffs\_dev *dev)
{
	struct yaffs_driver *drv = &dev->drv;

	drv->drv\_write\_chunk\_fn = yaffs\_mtd_write;
	drv->drv\_read\_chunk\_fn = yaffs\_mtd_read;
	drv->drv\_erase\_fn = yaffs\_mtd\_erase;
	drv->drv\_mark\_bad\_fn = yaffs\_mtd\_mark\_bad;
	drv->drv\_check\_bad\_fn = yaffs\_mtd\_check\_bad;
	drv->drv\_initialise\_fn = yaffs\_mtd\_initialise;
	drv->drv\_deinitialise\_fn = yaffs\_mtd\_deinitialise;
}

以下为Yaffs选项，为yaffs系统挂载（Mount）时配置，最后被传给yaffs_dev对应栏位。

struct yaffs_options {
	int inband_tags;         
	int skip\_checkpoint\_read;
	int skip\_checkpoint\_write;
	int no_cache;
	int tags\_ecc\_on;
	int tags\_ecc\_overridden;
	int lazy\_loading\_enabled;
	int lazy\_loading\_overridden;
	int empty\_lost\_and_found;
	int empty\_lost\_and\_found\_overridden;
};

yaffs对应Linux对应上下文：

struct yaffs\_linux\_context {
	struct list\_head context\_list;	
	struct yaffs_dev *dev; //指向Yaffs Dev的回调
	struct super_block *super; //指向超级块指针
	struct task\_struct *bg\_thread;	//做GC的后台线程
	int bg_running; //后台线程启动标记
	struct mutex gross_lock; //大粒度锁，用来保护文件系统关键字段
	u8 *spare_buffer;	//Spare缓冲区
	struct list\_head search\_contexts; //目录查找上下文链表
	struct task\_struct *readdir\_process; //标记当前目录读取的task
	unsigned mount_id; //yaffs2挂载的id号，对每个yaffs文件系统来讲标号唯一
	int dirty;
};

文件系统挂载
------

挂载后，整个文件系统才会为操作系统所用。YAFFS文件系统的挂载过程如下：

//其实yaffs2挂载就是执行内核的mount\_bdev函数，最后当然也会执行到传入的回调函数yaffs\_internal\_read\_super
static struct dentry \*yaffs2\_mount(struct file\_system\_type \*fs\_type, int flags,const char \*dev_name, void \*data)
{
        return mount\_bdev(fs\_type, flags, dev\_name, data, yaffs2\_internal\_read\_super_mtd);   
}
static int yaffs2\_internal\_read\_super\_mtd(struct super_block \*sb, void \*data, int silent)
{
	return yaffs\_internal\_read_super(2, sb, data, silent) ? 0 : -EINVAL;
}

static struct super\_block \*yaffs\_internal\_read\_super(int yaffs\_version, struct super\_block \*sb,  void *data, int silent)
{
        .......
        //超级块的一些初使设置
	sb->s\_magic = YAFFS\_MAGIC;
	sb->s\_op = &yaffs\_super_ops;
	sb->s\_flags |= MS\_NOATIME;
	read\_only = ((sb->s\_flags & MS_RDONLY) != 0);
	sb->s\_export\_op = &yaffs\_export\_ops;
        .......
	if (yaffs\_parse\_options(&options, data_str))
		/\* Option parsing failed */
		return NULL;
        .......
	sb->s\_blocksize = PAGE\_CACHE_SIZE;
	sb->s\_blocksize\_bits = PAGE\_CACHE\_SHIFT;
        .......
	mtd = get\_mtd\_device(NULL, MINOR(sb->s_dev));
        .......
        //若文件分区是MTD device类型的Nand Flash，开始着手建立整个yaffs_dev
	if (!read\_only && !(mtd->flags & MTD\_WRITEABLE)) {
		read_only = 1;
		sb->s\_flags |= MS\_RDONLY;
	}
	dev = kmalloc(sizeof(struct yaffs\_dev), GFP\_KERNEL);
	context = kmalloc(sizeof(struct yaffs\_linux\_context), GFP_KERNEL);
        .......
	memset(dev, 0, sizeof(struct yaffs_dev));
	param = &(dev->param);

	memset(context, 0, sizeof(struct yaffs\_linux\_context));
	dev->os_context = context;
	INIT\_LIST\_HEAD(&(context->context_list));
	context->dev = dev;
	context->super = sb;

	dev->read\_only = read\_only;
	sb->s\_fs\_info = dev;
	dev->driver_context = mtd;
        
        // 设定yaffs_param相关参数
        ...............

	mutex\_lock(&yaffs\_context_lock);

	for (mount\_id = 0, found = 0; !found; mount\_id++) {
		found = 1;
		list\_for\_each(l, &yaffs\_context\_list) {
			context\_iterator =list\_entry(l, struct yaffs\_linux\_context,context_list);
			if (context\_iterator->mount\_id == mount_id)
				found = 0;
		}
	}
	context->mount\_id = mount\_id;

	list\_add\_tail(&(yaffs\_dev\_to\_lc(dev)->context\_list),
		      &yaffs\_context\_list);
	mutex\_unlock(&yaffs\_context_lock);

	mutex\_init(&(yaffs\_dev\_to\_lc(dev)->gross_lock));
	yaffs\_gross\_lock(dev);
        //建立yaffs_dev参数，以及初始化Tnode
	err = yaffs\_guts\_initialise(dev);
        //执行yaffs的后台线程
	if (err == YAFFS\_OK) yaffs\_bg_start(dev);
	if (!context->bg_thread)
		param->defered\_dir\_update = 0;
	sb->s\_maxbytes = yaffs\_max\_file\_size(dev);
	yaffs\_gross\_unlock(dev);

	//创建文件系统根节点
	if (err == YAFFS_OK)
		inode = yaffs\_get\_inode(sb, S\_IFDIR | 0755, 0, yaffs\_root(dev));
        .......
	inode->i\_op = &yaffs\_dir\_inode\_operations;
	inode->i\_fop = &yaffs\_dir_operations;
	root = d\_alloc\_root(inode);
        .......
	sb->s_root = root;
	sb->s\_dirt = !dev->is\_checkpointed;
	return sb;
}

小结
--

本文介绍了Yaffs的架构、文件结构及初始化流程。之后章节介绍：

*   checkpoint机制
*   YAFFS参数调优及Debug
*   .......