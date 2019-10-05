---
title: Yaffs文件系统(2)--块及Chunk管理
tags:
  - Linux
  - YAFFS
  - 文件系统
url: 2194.html
id: 2194
categories:
  - Linux文件系统
date: 2019-05-14 14:12:01
---

前言
--

[Yaffs文件系统(1)-概述](https://l2h.site/2019/04/30/yaffs-1/)对文件系统的基本数据结构和初始化流程进行了介绍，本节着重介绍Yaffs如何对Block和Chunk进行管理的。

Block分类
-------

UNKNOWN

区块状态未知

NEEDS_SCANNING

预扫描时确定该Block需要被扫描

SCANNING

Block正在扫描

EMPTY

Block为空，表示已经被擦除

ALLOCATING

Block被Chunk分配器使用中

FULL

Block已经分配，且至少有一个chunk被删除

DIRTY

Block全部chunk被删可被擦除

CHECKPOINT

Block中包含checkpoint数据

COLLECTING

Block正在被做垃圾回收

DEAD

Block为坏块

以上Block状态转化图如下：

![](https://l2h.site/wp-content/uploads/2019/05/yaffs-2-1.png)

Block和Chunk分配
-------------

Yaffs分配原则如下：

*   Yaffs保存目前正在使用的Block编号，优先从该block上分配Chunk
*   当Block已经分配完毕（进入FULL状态），则选取另外一个未分配Block（即EMPTY Block）来分配
*   分配Chunk时，更新该Block对应的分配位图

代码如下：

static int yaffs\_alloc\_chunk(struct yaffs\_dev \*dev, int use\_reserver, struct yaffs\_block\_info \*\*block_ptr) {
	int ret_val;
	struct yaffs\_block\_info *bi;
        //若当前用于分配的Block为0，则找到一个EMPTY块分配
	if (dev->alloc_block < 0) {
		dev->alloc\_block = yaffs\_find\_alloc\_block(dev);
		dev->alloc_page = 0;
	}
        ..........
	if (dev->alloc_block >= 0) {
                //取得当前Block信息
		bi = yaffs\_get\_block\_info(dev, dev->alloc\_block);
                //得到Chunk ID
		ret\_val = (dev->alloc\_block * dev->param.chunks\_per\_block) + dev->alloc_page;
		bi->pages\_in\_use++;
                //设定当前Block的Chunk位图，更新剩余chunk的id，以及当前Block已分配page数量
		yaffs\_set\_chunk\_bit(dev, dev->alloc\_block, dev->alloc_page);
		dev->alloc_page++;
		dev->n\_free\_chunks--;
                //若已经分配数量大于Block支持最大的数量，则标记block状态为FULL，同时设置alloc_block为-1，表示重新查找一个EMPTY block进行分配
		if (dev->alloc\_page >= dev->param.chunks\_per_block)          {
			bi->block\_state = YAFFS\_BLOCK\_STATE\_FULL;
			dev->alloc_block = -1;
		}
		if (block\_ptr) *block\_ptr = bi;
		return ret_val;
	}
        .......
}
static int yaffs\_find\_alloc\_block(struct yaffs\_dev *dev)
{
        ......//若剩余可用Block不足，则不分配
	if (dev->n\_erased\_blocks < 1)  return -1;
        //以下为遍历查找一个可用block
	for (i = dev->internal\_start\_block; i <= dev->internal\_end\_block; i++) {
		dev->alloc\_block\_finder++;
		if (dev->alloc\_block\_finder < (int)dev->internal\_start\_block || dev->alloc\_block\_finder > (int)dev->internal\_end\_block) {
			dev->alloc\_block\_finder = dev->internal\_start\_block;
		}

		bi = yaffs\_get\_block\_info(dev, dev->alloc\_block_finder);
		if (bi->block\_state == YAFFS\_BLOCK\_STATE\_EMPTY) {
//查找到一个EMPLY Block，则使用之，使用前更新状态为ALLOCATING
			bi->block\_state = YAFFS\_BLOCK\_STATE\_ALLOCATING;
//序列号，用于标记文件chunk的新旧，每分配一个block就+1
			dev->seq_number++;
			bi->seq\_number = dev->seq\_number;
			dev->n\_erased\_blocks--;
			return dev->alloc\_block\_finder;
		}
	}
..........
}
static inline struct yaffs\_block\_info \*yaffs\_get\_block\_info(struct yaffs\_dev \*dev, int blk)
{
..........
       //返回当前Block的info
	return &dev->block\_info\[blk - dev->internal\_start_block\];
}

Block的扫描
--------

扫描的目的是Mount时建立起Yaffs的文件结构，代码实现在yaffs\_guts\_initialise（如果您还有印象，即mount过程中yaffs\_internal\_read_super会调用到的），如下：

int yaffs\_guts\_initialise(struct yaffs_dev *dev)
{
......//yaffs的Low Level初始化，主要为初始化一些基本参数
	if(yaffs\_guts\_ll\_init(dev) != YAFFS\_OK) return YAFFS_FAIL; 
......
	dev->is_mounted = 1;
......//初始化块管理相关一些基本参数
	if (!init\_failed && !yaffs\_init\_blocks(dev)) init\_failed = 1;
      //初始化tnode对象树
	yaffs\_init\_tnodes\_and\_objs(dev);
      //初始化根目录、lost and found等特殊目录
	if (!init\_failed && !yaffs\_create\_initial\_dir(dev)) init_failed = 1;
      //summary init（Yaffs2新引入的机制，利用block的空白chunk区域存储一些有效信息来帮助加速加载）
	if (!init\_failed && dev->param.is\_yaffs2 && !dev->param.disable\_summary && !yaffs\_summary_init(dev))
		init_failed = 1;

	if (!init_failed) {
		if (dev->param.is_yaffs2) {
      //尝试从checkpoint中恢复mount
			if (yaffs2\_checkpt\_restore(dev)) {
				yaffs\_check\_obj\_details\_loaded(dev->root_dir);
......
			} else {
      //如果checkpoint加载失败，则重新初始化block和tnode树（因为checkpoint加载过程中可能已经使用了相关结构做存储，而我们这里要重新建立）
				yaffs\_deinit\_blocks(dev);
				yaffs\_deinit\_tnodes\_and\_objs(dev);
......
				if (!init\_failed && yaffs\_init\_blocks(dev)) init\_failed = 1;
				yaffs\_init\_tnodes\_and\_objs(dev);
				if (!init\_failed && !yaffs\_create\_initial\_dir(dev)) init_failed = 1;
      //执行Block扫描过程
				if (!init\_failed && !yaffs2\_scan\_backwards(dev)) init\_failed = 1;
			}
		} else if (!yaffs1_scan(dev)) {
			init_failed = 1;
		}

		yaffs\_strip\_deleted_objs(dev);
		yaffs\_fix\_hanging_objs(dev);
		if (dev->param.empty\_lost\_n_found)
			yaffs\_empty\_l\_n\_f(dev);
	}
......
	return YAFFS_OK;
}

本文暂不讨论summary以及checkpoint等情况，那么整个block加载过程的核心代码即为：**yaffs2\_scan\_backwards**。详细如下：

int yaffs2\_scan\_backwards(struct yaffs_dev *dev)
{
......
	dev->seq\_number = YAFFS\_LOWEST\_SEQUENCE\_NUMBER;
//为block信息存储分配空间
	block\_index = kmalloc(n\_blocks * sizeof(struct yaffs\_block\_index), GFP_NOFS);
	if (!block_index) {
		block\_index =  vmalloc(n\_blocks * sizeof(struct yaffs\_block\_index)); alt\_block\_index = 1;
	}
......
	dev->blocks\_in\_checkpt = 0;

	chunk\_data = yaffs\_get\_temp\_buffer(dev);
//以下这段代码看着很长，但其实比较容易理解，详见注释
	bi = dev->block_info;
	for (blk = dev->internal\_start\_block; blk <= dev->internal\_end\_block; blk++) {
//清除block的chunk使用标记
		yaffs\_clear\_chunk_bits(dev, blk);
		bi->pages\_in\_use = 0;
		bi->soft\_del\_pages = 0;
//得到block初始状态以及序列号，即执行yaffs\_tags\_marshall\_query\_block得到当前block里chunk是否有正在使用，如果有则需要扫描
		yaffs\_query\_init\_block\_state(dev, blk, &state, &seq_number);
		bi->block_state = state;
		bi->seq\_number = seq\_number;
		if (bi->seq\_number == YAFFS\_SEQUENCE\_CHECKPOINT\_DATA)
			bi->block\_state = YAFFS\_BLOCK\_STATE\_CHECKPOINT;
		if (bi->seq\_number == YAFFS\_SEQUENCE\_BAD\_BLOCK)
			bi->block\_state = YAFFS\_BLOCK\_STATE\_DEAD;
......
		if (bi->block\_state == YAFFS\_BLOCK\_STATE\_CHECKPOINT) {
			dev->blocks\_in\_checkpt++;
		} else if (bi->block\_state == YAFFS\_BLOCK\_STATE\_DEAD) {
			yaffs\_trace(YAFFS\_TRACE\_BAD\_BLOCKS, "block %d is bad", blk);
		} else if (bi->block\_state == YAFFS\_BLOCK\_STATE\_EMPTY) {
			yaffs\_trace(YAFFS\_TRACE\_SCAN\_DEBUG, "Block empty ");
			dev->n\_erased\_blocks++;
			dev->n\_free\_chunks += dev->param.chunks\_per\_block;
		} else if (bi->block\_state == YAFFS\_BLOCK\_STATE\_NEEDS_SCAN) {
//如果block状态为needs_scan，则标记后续对block进行scan
			if (seq\_number >= YAFFS\_LOWEST\_SEQUENCE\_NUMBER && seq\_number < YAFFS\_HIGHEST\_SEQUENCE\_NUMBER) {
				block\_index\[n\_to\_scan\].seq = seq\_number;
				block\_index\[n\_to_scan\].block = blk;
				n\_to\_scan++;
				if (seq\_number >= dev->seq\_number)
					dev->seq\_number = seq\_number;
			} else {
				yaffs\_trace(YAFFS\_TRACE\_SCAN, "Block scanning block %d has bad sequence number %d", blk, seq\_number);
			}
		}
		bi++;
	}
...... //对block进行排序，排序的依据是按照block的序列号。会这样做原因是yaffs在管理文件，采用就是逐步增加的序列号。同一个文件同一个chunk id，较大的序列号为最新。这也是函数scan_backwards的由来。
	sort(block\_index, n\_to\_scan, sizeof(struct yaffs\_block_index),
		   yaffs2_ybicmp, NULL);
//真正开始进行扫描
	start_iter = 0;
	end\_iter = n\_to_scan - 1;
	for (block\_iter = end\_iter;!alloc\_failed && block\_iter >= start\_iter;block\_iter--) {
......
//对每个需要扫描的block进行扫描
		blk = block\_index\[block\_iter\].block;
		bi = yaffs\_get\_block_info(dev, blk);
		deleted = 0;
//summary读取，本文不做详述
		summary\_available = yaffs\_summary\_read(dev, dev->sum\_tags, blk);
		found_chunks = 0;
		if (summary_available)
			c = dev->chunks\_per\_summary - 1;
		else
			c = dev->param.chunks\_per\_block - 1;
//对block中的每一个chunk进行扫描
		for (  !alloc_failed && c >= 0 &&
		     (bi->block\_state == YAFFS\_BLOCK\_STATE\_NEEDS_SCAN ||
		      bi->block\_state == YAFFS\_BLOCK\_STATE\_ALLOCATING);
		      c--) {
			if (yaffs2\_scan\_chunk(dev, bi, blk, c, &found\_chunks, chunk\_data, &hard\_list, summary\_available) == YAFFS\_FAIL) alloc\_failed = 1;
		}

		if (bi->block\_state == YAFFS\_BLOCK\_STATE\_NEEDS_SCAN) {
			bi->block\_state = YAFFS\_BLOCK\_STATE\_FULL;
		}

		if (bi->pages\_in\_use == 0 &&  !bi->has\_shrink\_hdr &&
		    bi->block\_state == YAFFS\_BLOCK\_STATE\_FULL) {
			yaffs\_block\_became_dirty(dev, blk);
		}
	}

	yaffs\_skip\_rest\_of\_block(dev);

	if (alt\_block\_index) vfree(block_index);
	else kfree(block_index);
        //处理文件Hard Link
	yaffs\_link\_fixup(dev, &hard_list);
        //释放分配的临时缓存
	yaffs\_release\_temp\_buffer(dev, chunk\_data);

	if (alloc_failed)
		return YAFFS_FAIL;

	return YAFFS_OK;
}

从代码注释可以看出，yaffs2\_scan\_backwards核心代码为yaffs2\_scan\_chunk,代码如下：

static inline int yaffs2\_scan\_chunk(struct yaffs_dev *dev,
		struct yaffs\_block\_info *bi,
		int blk, int chunk\_in\_block,
		int *found_chunks,
		u8 *chunk_data,
		struct list\_head *hard\_list,
		int summary_available)
{
......//如果存在summary则读取summary
	int chunk = blk * dev->param.chunks\_per\_block + chunk\_in\_block
	if (summary_available) {
		result = yaffs\_summary\_fetch(dev, &tags, chunk\_in\_block);
		tags.seq\_number = bi->seq\_number;
	}
//从chunk中读取对应tag信息
	if (!summary\_available || tags.obj\_id == 0) {
		result = yaffs\_rd\_chunk\_tags\_nand(dev, chunk, NULL, &tags);
		dev->tags_used++;
	} else {
		dev->summary_used++;
	}

	if (!tags.chunk_used) {
......
		if (*found_chunks) {
			/\* This is a chunk that was skipped due
			 \* to failing the erased check */
		} else if (chunk\_in\_block == 0) {
			//如果为block第一个chunk且未使用，则整个block为空
			bi->block\_state = YAFFS\_BLOCK\_STATE\_EMPTY;
			dev->n\_erased\_blocks++;
		} else {
//若扫到block末尾，则进行相应标记
			if (bi->block\_state == YAFFS\_BLOCK\_STATE\_NEEDS\_SCAN ||  bi->block\_state == YAFFS\_BLOCK\_STATE_ALLOCATING) {
				if (dev->seq\_number == bi->seq\_number) {
					bi->block\_state = YAFFS\_BLOCK\_STATE\_ALLOCATING;
					dev->alloc_block = blk;
					dev->alloc\_page = chunk\_in_block;
					dev->alloc\_block\_finder = blk;
				} else {
//从打印可以看出该block之前没有写完整，导致block内seq_number不相同
					yaffs\_trace(YAFFS\_TRACE_SCAN,
						"Partially written block %d detected. gc will fix this.", blk);
				}
			}
		}
		dev->n\_free\_chunks++;
	} else if (tags.ecc_result ==
		YAFFS\_ECC\_RESULT_UNFIXED) {
//出现ECC校验失败
                yaffs\_trace(YAFFS\_TRACE\_SCAN, " Unfixed ECC in chunk(%d:%d), chunk ignored",blk, chunk\_in_block);
		dev->n\_free\_chunks++;
	} else if (tags.obj\_id > YAFFS\_MAX\_OBJECT\_ID ||
		   tags.chunk\_id > YAFFS\_MAX\_CHUNK\_ID ||
		   tags.obj\_id == YAFFS\_OBJECTID_SUMMARY ||
		   (tags.chunk\_id > 0 &&  tags.n\_bytes > dev->data\_bytes\_per\_chunk) || tags.seq\_number != bi->seq_number) {
                //对一些异常chunk做处理
		yaffs\_trace(YAFFS\_TRACE\_SCAN, "Chunk (%d:%d) with bad tags:obj = %d, chunk\_id = %d, n\_bytes = %d, ignored", blk, chunk\_in\_block, tags.obj\_id, tags.chunk\_id, tags.n\_bytes);
		dev->n\_free\_chunks++;
	} else if (tags.chunk_id > 0) {
        //所有的tag标记都正常，则开始进行后续处理
		/\* chunk_id > 0 so it is a data chunk... */
		loff_t endpos;
		loff\_t chunk\_base = (tags.chunk_id - 1) *
					dev->data\_bytes\_per_chunk;

		*found_chunks = 1;
        //标记block中的chunk使用状况
		yaffs\_set\_chunk\_bit(dev, blk, chunk\_in_block);
		bi->pages\_in\_use++;
        //找到文件句柄。若没有，表示第一次扫到该文件，创建之
		in = yaffs\_find\_or\_create\_by\_number(dev,tags.obj\_id, YAFFS\_OBJECT\_TYPE_FILE);
		if (!in)  alloc_failed = 1;
		if (in &&  in->variant\_type == YAFFS\_OBJECT\_TYPE\_FILE &&
		    chunk\_base < in->variant.file\_variant.shrink_size) {
			if (!yaffs\_put\_chunk\_in\_file(in, tags.chunk\_id, chunk, -1)) alloc\_failed = 1;
//将该文件部分加入tnode tree
			endpos = chunk\_base + tags.n\_bytes;
			if (!in->valid && in->variant.file\_variant.scanned\_size < endpos) {
				in->variant.file_variant.
				    scanned_size = endpos;
				in->variant.file_variant.
				    file_size = endpos;
			}
		} else if (in) {
//该文件超出了文件shrink大小，表示这是一个被resize过的文件，则删除超出文件大小的部分
			yaffs\_chunk\_del(dev, chunk, 1, \_\_LINE\_\_);
		}
	} else {
//存放文件头部（chunk id为0）的chunk进行处理
		*found_chunks = 1;
		yaffs\_set\_chunk\_bit(dev, blk, chunk\_in_block);
		bi->pages\_in\_use++;
		oh = NULL;
		in = NULL;
		if (tags.extra_available) {
			in = yaffs\_find\_or\_create\_by_number(dev,
					tags.obj_id,
					tags.extra\_obj\_type);
			if (!in) alloc_failed = 1;
		}

		if (!in || (!in->valid && dev->param.disable\_lazy\_load) || tags.extra\_shadows || (!in->valid && (tags.obj\_id == YAFFS\_OBJECTID\_ROOT || tags.obj\_id == YAFFS\_OBJECTID_LOSTNFOUND))) {
//从头部中读取文件信息
			result = yaffs\_rd\_chunk\_tags\_nand(dev, chunk,  chunk_data,  NULL);
//文件头部的信息存放在chunk的数据起始位置
			oh = (struct yaffs\_obj\_hdr *)chunk_data;
//使用Inband tag的特殊处理
			if (dev->param.inband_tags) {
				oh->shadows\_obj = oh->inband\_shadowed\_obj\_id;
				oh->is\_shrink = oh->inband\_is_shrink;
			}
//创建文件obj
			if (!in) {
				in = yaffs\_find\_or\_create\_by\_number(dev, tags.obj\_id, oh->type);
				if (!in) alloc_failed = 1;
			}
		}
..........
		if (in->valid) {
//找到过同一文件。删除该chunk之前进行必要清理工作
			if ((in->variant\_type == YAFFS\_OBJECT\_TYPE\_FILE) && ((oh && oh->type == YAFFS\_OBJECT\_TYPE\_FILE) || (tags.extra\_available &&  tags.extra\_obj\_type == YAFFS\_OBJECT\_TYPE_FILE) )) {
				loff\_t this\_size = (oh) ? yaffs\_oh\_to\_size(oh) : tags.extra\_file_size;
				u32 parent\_obj\_id = (oh) ? oh->parent\_obj\_id : tags.extra\_parent\_id;
				is\_shrink = (oh) ? oh->is\_shrink : tags.extra\_is\_shrink;
//文件的父id若为特殊标记，则文件大小为0
				if (parent\_obj\_id == AFFS\_OBJECTID\_DELETED ||parent\_obj\_id == YAFFS\_OBJECTID\_UNLINKED) {
					this_size = 0;
					is_shrink = 1;
				}

				if (is\_shrink &&  in->variant.file\_variant.shrink\_size > this\_size)
					in->variant.file\_variant.shrink\_size = this_size;
				if (is\_shrink) bi->has\_shrink_hdr = 1;
			}
			/\* Use existing - destroy this one. */
			yaffs\_chunk\_del(dev, chunk, 1, \_\_LINE\_\_);
		}

		if (!in->valid && in->variant_type !=
		    (oh ? oh->type : tags.extra\_obj\_type)) {
			yaffs\_trace(YAFFS\_TRACE_ERROR,
				"yaffs tragedy: Bad type, %d != %d, for object %d at chunk %d during scan",
				oh ? oh->type : tags.extra\_obj\_type,
				in->variant\_type, tags.obj\_id,
				chunk);
			in = yaffs\_retype\_obj(in, oh ? oh->type : tags.extra\_obj\_type);
		}
//Root目录和lostfound目录处理
		if (!in->valid &&  (tags.obj\_id == YAFFS\_OBJECTID\_ROOT || tags.obj\_id == YAFFS\_OBJECTID\_LOSTNFOUND)) {
			in->valid = 1;
			if (oh) {
				in->yst\_mode = oh->yst\_mode;
				yaffs\_load\_attribs(in, oh);
				in->lazy_loaded = 0;
			} else {
				in->lazy_loaded = 1;
			}
			in->hdr_chunk = chunk;

		} else if (!in->valid) {
//第一次找到文件头部
			in->valid = 1;
			in->hdr_chunk = chunk;
			if (oh) {
				in->variant_type = oh->type;
				in->yst\_mode = oh->yst\_mode;
				yaffs\_load\_attribs(in, oh);

				if (oh->shadows\_obj > 0) yaffs\_handle\_shadowed\_obj(dev,  oh->shadows_obj, 1);

				yaffs\_set\_obj\_name\_from_oh(in, oh);
				parent = yaffs\_find\_or\_create\_by\_number(dev, oh->parent\_obj\_id, YAFFS\_OBJECT\_TYPE\_DIRECTORY);
				file\_size = yaffs\_oh\_to\_size(oh);
				is\_shrink = oh->is\_shrink;
				equiv\_id = oh->equiv\_id;
			} else {
				in->variant\_type = tags.extra\_obj_type;
				parent = yaffs\_find\_or\_create\_by\_number(dev, tags.extra\_parent\_id, YAFFS\_OBJECT\_TYPE\_DIRECTORY);
				file\_size = tags.extra\_file_size;
				is\_shrink = tags.extra\_is_shrink;
				equiv\_id = tags.extra\_equiv_id;
				in->lazy_loaded = 1;
			}
			in->dirty = 0;
			if (!parent)
				alloc_failed = 1;
			if (parent &&
			    parent->variant\_type == YAFFS\_OBJECT\_TYPE\_UNKNOWN) {
				parent->variant_type =
					YAFFS\_OBJECT\_TYPE_DIRECTORY;
				INIT\_LIST\_HEAD(&parent->variant.dir_variant.children);
			} else if (!parent || parent->variant\_type != YAFFS\_OBJECT\_TYPE\_DIRECTORY) {
//文件若找不到父目录放到lost and found
				yaffs\_trace(YAFFS\_TRACE_ERROR,
					"yaffs tragedy: attempting to use non-directory as a directory in scan. Put in lost+found."
					);
				parent = dev->lost\_n\_found;
			}
			yaffs\_add\_obj\_to\_dir(parent, in);

			is\_unlinked = (parent == dev->del\_dir) || (parent == dev->unlinked_dir);

			if (is_shrink)
				bi->has\_shrink\_hdr = 1;

			switch (in->variant_type) {
			case YAFFS\_OBJECT\_TYPE_UNKNOWN:
				break;
			case YAFFS\_OBJECT\_TYPE_FILE:
				file\_var = &in->variant.file\_variant;
				if (file\_var->scanned\_size < file_size) {
				//文件扫到的大小小于文件chunk头部表示的大小
					file\_var->file\_size = file_size;
					file\_var->scanned\_size = file_size;
				}

				if (file\_var->shrink\_size > file_size)
					file\_var->shrink\_size = file_size;  break;
			case YAFFS\_OBJECT\_TYPE_HARDLINK:
				hl\_var = &in->variant.hardlink\_variant;
				if (!is_unlinked) {
					hl\_var->equiv\_id = equiv_id;
					list\_add(&in->hard\_links, hard_list); } break;
			case YAFFS\_OBJECT\_TYPE_DIRECTORY:
				/\* Do nothing */
				break;
			case YAFFS\_OBJECT\_TYPE_SPECIAL:
				/\* Do nothing */
				break;
			case YAFFS\_OBJECT\_TYPE_SYMLINK:
				sl\_var = &in->variant.symlink\_variant;
				if (oh) {
					sl_var->alias =
					    yaffs\_clone\_str(oh->alias);
					if (!sl_var->alias)
						alloc_failed = 1;
				}
				break;
			}
		}
	}
	return alloc\_failed ? YAFFS\_FAIL : YAFFS_OK;
}

以上，就完成了扫描过程。

Block分配
-------

执行Block分配一般是由写操作触发的，最后的三个函数调用顺序为yaffs\_write\_new\_chunk-->yaffs\_alloc\_chunk-->yaffs\_find\_alloc\_block。代码分析如下：

static int yaffs\_write\_new\_chunk(struct yaffs\_dev \*dev,const u8 \*data, struct yaffs\_ext\_tags *tags, int use_reserver)
{
......//因为写信chunk，表示Flash上内容已经改变了，将Checkpoint的数据设置为无效
	yaffs2\_checkpt\_invalidate(dev);

	do {
......//分配chunk空间
		chunk = yaffs\_alloc\_chunk(dev, use_reserver, &bi);
		if (chunk < 0) {/* no space */break;}
		attempts++;

		if (dev->param.always\_check\_erased)
			bi->skip\_erased\_check = 0;

		if (!bi->skip\_erased\_check) {
//确认chunk是否已经擦除
			erased\_ok = yaffs\_check\_chunk\_erased(dev, chunk);
			if (erased\_ok != YAFFS\_OK) {
				yaffs\_trace(YAFFS\_TRACE_ERROR,
				  "**>\> yaffs chunk %d was not erased", chunk);
//如果chunk没有擦除，则删除该chunk，跳到下个可用block寻找chunk
				yaffs\_chunk\_del(dev, chunk, 1, \_\_LINE\_\_);
				yaffs\_skip\_rest\_of\_block(dev);
				continue;
			}
		}
//写数据和tag到chunk内
		write\_ok = yaffs\_wr\_chunk\_tags_nand(dev, chunk, data, tags);
		if (!bi->skip\_erased\_check)
			write\_ok =  yaffs\_verify\_chunk\_written(dev, chunk, data, tags);

		if (write\_ok != YAFFS\_OK) {
//执行写错的回调
			yaffs\_handle\_chunk\_wr\_error(dev, chunk, erased_ok);
			continue;
		}

		bi->skip\_erased\_check = 1;
//执行成功的回调
		yaffs\_handle\_chunk\_wr\_ok(dev, chunk, data, tags);

	} while (write\_ok != YAFFS\_OK && (yaffs\_wr\_attempts == 0 || attempts <= yaffs\_wr\_attempts));

	if (!write_ok) chunk = -1;
	if (attempts > 1) {
		yaffs\_trace(YAFFS\_TRACE_ERROR,"**>> yaffs write required %d attempts",attempts);
		dev->n\_retried\_writes += (attempts - 1);
	}
	return chunk;
}

static int yaffs\_alloc\_chunk(struct yaffs\_dev \*dev, int use\_reserver, struct yaffs\_block\_info \*\*block_ptr)
{
	if (dev->alloc_block < 0) {
//若当前无可分配的块，则分配块
		dev->alloc\_block = yaffs\_find\_alloc\_block(dev);
		dev->alloc_page = 0;
	}
	if (!use\_reserver && !yaffs\_check\_alloc\_available(dev, 1)) {
//不使用保留块也找不到可用块
		return -1;
	}

	if (dev->n\_erased\_blocks < (int)dev->param.n\_reserved\_blocks && dev->alloc_page == 0)
		yaffs\_trace(YAFFS\_TRACE_ALLOCATE, "Allocating reserve");
	if (dev->alloc_block >= 0) {
		bi = yaffs\_get\_block\_info(dev, dev->alloc\_block);
		ret\_val = (dev->alloc\_block * dev->param.chunks\_per\_block) +dev->alloc_page;
		bi->pages\_in\_use++;
		yaffs\_set\_chunk\_bit(dev, dev->alloc\_block, dev->alloc_page);
		dev->alloc_page++;
		dev->n\_free\_chunks--;

//若块已满，则标记状态为已满
		if (dev->alloc\_page >= dev->param.chunks\_per_block) {
			bi->block\_state = YAFFS\_BLOCK\_STATE\_FULL;
			dev->alloc_block = -1;
		}

		if (block_ptr)
			*block_ptr = bi;

		return ret_val;
	}

	yaffs\_trace(YAFFS\_TRACE_ERROR,
		"!!!!!!!!! Allocator out !!!!!!!!!!!!!!!!!");

	return -1;
}

static int yaffs\_find\_alloc\_block(struct yaffs\_dev *dev)
{
	u32 i;
	struct yaffs\_block\_info *bi;
//没有可使用的块则返回-1
	if (dev->n\_erased\_blocks < 1) {
	        yaffs\_trace(YAFFS\_TRACE_ERROR,"yaffs tragedy: no more erased blocks");
		return -1;
	}
//否则，遍历找到一个可使用的块
	for (i = dev->internal\_start\_block; i <= dev->internal\_end\_block; i++) {
		dev->alloc\_block\_finder++;
		if (dev->alloc\_block\_finder < (int)dev->internal\_start\_block || dev->alloc\_block\_finder > (int)dev->internal\_end\_block) {
			dev->alloc\_block\_finder = dev->internal\_start\_block;
		}

		bi = yaffs\_get\_block\_info(dev, dev->alloc\_block_finder);

		if (bi->block\_state == YAFFS\_BLOCK\_STATE\_EMPTY) {
//找到块的条件是该块的状态为EMPTY，此时返回块的编号			
bi->block\_state = YAFFS\_BLOCK\_STATE\_ALLOCATING;
			dev->seq_number++;
			bi->seq\_number = dev->seq\_number;
			dev->n\_erased\_blocks--;
			yaffs\_trace(YAFFS\_TRACE_ALLOCATE,
			  "Allocated block %d, seq  %d, %d left" ,
			   dev->alloc\_block\_finder, dev->seq_number,
			   dev->n\_erased\_blocks);
			return dev->alloc\_block\_finder;
		}
	}

	yaffs\_trace(YAFFS\_TRACE\_ALWAYS,"yaffs tragedy: no more erased blocks, but there should have been %d",dev->n\_erased_blocks);

	return -1;
}

Block擦除
-------

Block擦除的条件是：

*   遇到坏块，且无法正确执行坏块标记
*   脏数据回收
*   上层下Format Flash指令

其中脏数据回收是GC（Garbage Collection）机制在执行，后续章节会再对GC进行介绍。

擦除块执行的函数也很简单，代码如下：

int yaffs\_erase\_block(struct yaffs\_dev *dev, int block\_no)
{
	block\_no -= dev->block\_offset;
	dev->n_erasures++;
	result = dev->drv.drv\_erase\_fn(dev, block_no);
	return result;
}

其中drv\_erase\_fn回调函数在Linux的系统下最后MTD块系统的mtd_erase函数：

int nandmtd\_erase\_block(struct yaffs\_dev *dev, int block\_no)
{
	struct mtd\_info *mtd = yaffs\_dev\_to\_mtd(dev);
	u32 addr = ((loff\_t) block\_no) * dev->param.total\_bytes\_per\_chunk *dev->param.chunks\_per_block;
	struct erase_info ei;
	int retval = 0;
........
	retval = mtd_erase(mtd, &ei);
.......
	if (retval == 0)return YAFFS_OK;
	return YAFFS_FAIL;
}

总结
--

以上，块初始化及管理大致过程分析结束，下一章节介绍Checkpoint机制