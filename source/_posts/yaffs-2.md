---
title: Yaffs文件系统(2)--块及Chunk管理
tags:
  - Linux
  - YAFFS
  - 文件系统
url: 2194.html
id: 2194
categories:
  - Linux
  - Linux文件系统
date: 2019-05-14 14:12:01
---

前言
--

[Yaffs文件系统(1)-概述](https://www.l2h.site/2019/04/30/yaffs-1/)对文件系统的基本数据结构和初始化流程进行了介绍，本节着重介绍Yaffs如何对Block和Chunk进行管理的。

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

![](http://pic.l2h.site/yaffs-2-1.png)

Block和Chunk分配
-------------

Yaffs分配原则如下：

*   Yaffs保存目前正在使用的Block编号，优先从该block上分配Chunk
*   当Block已经分配完毕（进入FULL状态），则选取另外一个未分配Block（即EMPTY Block）来分配
*   分配Chunk时，更新该Block对应的分配位图

代码如下：
```C
static int yaffs_alloc_chunk(struct yaffs_dev *dev, int use_reserver, struct yaffs_block_info **block_ptr) {
	int ret_val;
	struct yaffs_block_info *bi;
        //若当前用于分配的Block为0，则找到一个EMPTY块分配
	if (dev->alloc_block < 0) {
		dev->alloc_block = yaffs_find_alloc_block(dev);
		dev->alloc_page = 0;
	}
        ..........
	if (dev->alloc_block >= 0) {
                //取得当前Block信息
		bi = yaffs_get_block_info(dev, dev->alloc_block);
                //得到Chunk ID
		ret_val = (dev->alloc_block * dev->param.chunks_per_block) + dev->alloc_page;
		bi->pages_in_use++;
                //设定当前Block的Chunk位图，更新剩余chunk的id，以及当前Block已分配page数量
		yaffs_set_chunk_bit(dev, dev->alloc_block, dev->alloc_page);
		dev->alloc_page++;
		dev->n_free_chunks--;
                //若已经分配数量大于Block支持最大的数量，则标记block状态为FULL，同时设置alloc_block为-1，表示重新查找一个EMPTY block进行分配
		if (dev->alloc_page >= dev->param.chunks_per_block)          {
			bi->block_state = YAFFS_BLOCK_STATE_FULL;
			dev->alloc_block = -1;
		}
		if (block_ptr) *block_ptr = bi;
		return ret_val;
	}
        .......
}
static int yaffs_find_alloc_block(struct yaffs_dev *dev)
{
        ......//若剩余可用Block不足，则不分配
	if (dev->n_erased_blocks < 1)  return -1;
        //以下为遍历查找一个可用block
	for (i = dev->internal_start_block; i <= dev->internal_end_block; i++) {
		dev->alloc_block_finder++;
		if (dev->alloc_block_finder < (int)dev->internal_start_block || dev->alloc_block_finder > (int)dev->internal_end_block) {
			dev->alloc_block_finder = dev->internal_start_block;
		}

		bi = yaffs_get_block_info(dev, dev->alloc_block_finder);
		if (bi->block_state == YAFFS_BLOCK_STATE_EMPTY) {
//查找到一个EMPLY Block，则使用之，使用前更新状态为ALLOCATING
			bi->block_state = YAFFS_BLOCK_STATE_ALLOCATING;
//序列号，用于标记文件chunk的新旧，每分配一个block就+1
			dev->seq_number++;
			bi->seq_number = dev->seq_number;
			dev->n_erased_blocks--;
			return dev->alloc_block_finder;
		}
	}
..........
}
static inline struct yaffs_block_info *yaffs_get_block_info(struct yaffs_dev *dev, int blk)
{
..........
       //返回当前Block的info
	return &dev->block_info[blk - dev->internal_start_block];
}
```
Block的扫描
--------

扫描的目的是Mount时建立起Yaffs的文件结构，代码实现在yaffs_guts_initialise（如果您还有印象，即mount过程中yaffs_internal_read_super会调用到的），如下：
```C
int yaffs_guts_initialise(struct yaffs_dev *dev)
{
......//yaffs的Low Level初始化，主要为初始化一些基本参数
	if(yaffs_guts_ll_init(dev) != YAFFS_OK) return YAFFS_FAIL; 
......
	dev->is_mounted = 1;
......//初始化块管理相关一些基本参数
	if (!init_failed && !yaffs_init_blocks(dev)) init_failed = 1;
      //初始化tnode对象树
	yaffs_init_tnodes_and_objs(dev);
      //初始化根目录、lost and found等特殊目录
	if (!init_failed && !yaffs_create_initial_dir(dev)) init_failed = 1;
      //summary init（Yaffs2新引入的机制，利用block的空白chunk区域存储一些有效信息来帮助加速加载）
	if (!init_failed && dev->param.is_yaffs2 && !dev->param.disable_summary && !yaffs_summary_init(dev))
		init_failed = 1;

	if (!init_failed) {
		if (dev->param.is_yaffs2) {
      //尝试从checkpoint中恢复mount
			if (yaffs2_checkpt_restore(dev)) {
				yaffs_check_obj_details_loaded(dev->root_dir);
......
			} else {
      //如果checkpoint加载失败，则重新初始化block和tnode树（因为checkpoint加载过程中可能已经使用了相关结构做存储，而我们这里要重新建立）
				yaffs_deinit_blocks(dev);
				yaffs_deinit_tnodes_and_objs(dev);
......
				if (!init_failed && yaffs_init_blocks(dev)) init_failed = 1;
				yaffs_init_tnodes_and_objs(dev);
				if (!init_failed && !yaffs_create_initial_dir(dev)) init_failed = 1;
      //执行Block扫描过程
				if (!init_failed && !yaffs2_scan_backwards(dev)) init_failed = 1;
			}
		} else if (!yaffs1_scan(dev)) {
			init_failed = 1;
		}

		yaffs_strip_deleted_objs(dev);
		yaffs_fix_hanging_objs(dev);
		if (dev->param.empty_lost_n_found)
			yaffs_empty_l_n_f(dev);
	}
......
	return YAFFS_OK;
}
```
本文暂不讨论summary以及checkpoint等情况，那么整个block加载过程的核心代码即为：**yaffs2_scan_backwards**。详细如下：
```C
int yaffs2_scan_backwards(struct yaffs_dev *dev)
{
......
	dev->seq_number = YAFFS_LOWEST_SEQUENCE_NUMBER;
//为block信息存储分配空间
	block_index = kmalloc(n_blocks * sizeof(struct yaffs_block_index), GFP_NOFS);
	if (!block_index) {
		block_index =  vmalloc(n_blocks * sizeof(struct yaffs_block_index)); alt_block_index = 1;
	}
......
	dev->blocks_in_checkpt = 0;

	chunk_data = yaffs_get_temp_buffer(dev);
//以下这段代码看着很长，但其实比较容易理解，详见注释
	bi = dev->block_info;
	for (blk = dev->internal_start_block; blk <= dev->internal_end_block; blk++) {
//清除block的chunk使用标记
		yaffs_clear_chunk_bits(dev, blk);
		bi->pages_in_use = 0;
		bi->soft_del_pages = 0;
//得到block初始状态以及序列号，即执行yaffs_tags_marshall_query_block得到当前block里chunk是否有正在使用，如果有则需要扫描
		yaffs_query_init_block_state(dev, blk, &state, &seq_number);
		bi->block_state = state;
		bi->seq_number = seq_number;
		if (bi->seq_number == YAFFS_SEQUENCE_CHECKPOINT_DATA)
			bi->block_state = YAFFS_BLOCK_STATE_CHECKPOINT;
		if (bi->seq_number == YAFFS_SEQUENCE_BAD_BLOCK)
			bi->block_state = YAFFS_BLOCK_STATE_DEAD;
......
		if (bi->block_state == YAFFS_BLOCK_STATE_CHECKPOINT) {
			dev->blocks_in_checkpt++;
		} else if (bi->block_state == YAFFS_BLOCK_STATE_DEAD) {
			yaffs_trace(YAFFS_TRACE_BAD_BLOCKS, "block %d is bad", blk);
		} else if (bi->block_state == YAFFS_BLOCK_STATE_EMPTY) {
			yaffs_trace(YAFFS_TRACE_SCAN_DEBUG, "Block empty ");
			dev->n_erased_blocks++;
			dev->n_free_chunks += dev->param.chunks_per_block;
		} else if (bi->block_state == YAFFS_BLOCK_STATE_NEEDS_SCAN) {
//如果block状态为needs_scan，则标记后续对block进行scan
			if (seq_number >= YAFFS_LOWEST_SEQUENCE_NUMBER && seq_number < YAFFS_HIGHEST_SEQUENCE_NUMBER) {
				block_index[n_to_scan].seq = seq_number;
				block_index[n_to_scan].block = blk;
				n_to_scan++;
				if (seq_number >= dev->seq_number)
					dev->seq_number = seq_number;
			} else {
				yaffs_trace(YAFFS_TRACE_SCAN, "Block scanning block %d has bad sequence number %d", blk, seq_number);
			}
		}
		bi++;
	}
...... //对block进行排序，排序的依据是按照block的序列号。会这样做原因是yaffs在管理文件，采用就是逐步增加的序列号。同一个文件同一个chunk id，较大的序列号为最新。这也是函数scan_backwards的由来。
	sort(block_index, n_to_scan, sizeof(struct yaffs_block_index),
		   yaffs2_ybicmp, NULL);
//真正开始进行扫描
	start_iter = 0;
	end_iter = n_to_scan - 1;
	for (block_iter = end_iter;!alloc_failed && block_iter >= start_iter;block_iter--) {
......
//对每个需要扫描的block进行扫描
		blk = block_index[block_iter].block;
		bi = yaffs_get_block_info(dev, blk);
		deleted = 0;
//summary读取，本文不做详述
		summary_available = yaffs_summary_read(dev, dev->sum_tags, blk);
		found_chunks = 0;
		if (summary_available)
			c = dev->chunks_per_summary - 1;
		else
			c = dev->param.chunks_per_block - 1;
//对block中的每一个chunk进行扫描
		for (  !alloc_failed && c >= 0 &&
		     (bi->block_state == YAFFS_BLOCK_STATE_NEEDS_SCAN ||
		      bi->block_state == YAFFS_BLOCK_STATE_ALLOCATING);
		      c--) {
			if (yaffs2_scan_chunk(dev, bi, blk, c, &found_chunks, chunk_data, &hard_list, summary_available) == YAFFS_FAIL) alloc_failed = 1;
		}

		if (bi->block_state == YAFFS_BLOCK_STATE_NEEDS_SCAN) {
			bi->block_state = YAFFS_BLOCK_STATE_FULL;
		}

		if (bi->pages_in_use == 0 &&  !bi->has_shrink_hdr &&
		    bi->block_state == YAFFS_BLOCK_STATE_FULL) {
			yaffs_block_became_dirty(dev, blk);
		}
	}

	yaffs_skip_rest_of_block(dev);

	if (alt_block_index) vfree(block_index);
	else kfree(block_index);
        //处理文件Hard Link
	yaffs_link_fixup(dev, &hard_list);
        //释放分配的临时缓存
	yaffs_release_temp_buffer(dev, chunk_data);

	if (alloc_failed)
		return YAFFS_FAIL;

	return YAFFS_OK;
}
```
从代码注释可以看出，yaffs2_scan_backwards核心代码为yaffs2_scan_chunk,代码如下：
```C
static inline int yaffs2_scan_chunk(struct yaffs_dev *dev,
		struct yaffs_block_info *bi,
		int blk, int chunk_in_block,
		int *found_chunks,
		u8 *chunk_data,
		struct list_head *hard_list,
		int summary_available)
{
......//如果存在summary则读取summary
	int chunk = blk * dev->param.chunks_per_block + chunk_in_block
	if (summary_available) {
		result = yaffs_summary_fetch(dev, &tags, chunk_in_block);
		tags.seq_number = bi->seq_number;
	}
//从chunk中读取对应tag信息
	if (!summary_available || tags.obj_id == 0) {
		result = yaffs_rd_chunk_tags_nand(dev, chunk, NULL, &tags);
		dev->tags_used++;
	} else {
		dev->summary_used++;
	}

	if (!tags.chunk_used) {
......
		if (*found_chunks) {
			/* This is a chunk that was skipped due
			 * to failing the erased check */
		} else if (chunk_in_block == 0) {
			//如果为block第一个chunk且未使用，则整个block为空
			bi->block_state = YAFFS_BLOCK_STATE_EMPTY;
			dev->n_erased_blocks++;
		} else {
//若扫到block末尾，则进行相应标记
			if (bi->block_state == YAFFS_BLOCK_STATE_NEEDS_SCAN ||  bi->block_state == YAFFS_BLOCK_STATE_ALLOCATING) {
				if (dev->seq_number == bi->seq_number) {
					bi->block_state = YAFFS_BLOCK_STATE_ALLOCATING;
					dev->alloc_block = blk;
					dev->alloc_page = chunk_in_block;
					dev->alloc_block_finder = blk;
				} else {
//从打印可以看出该block之前没有写完整，导致block内seq_number不相同
					yaffs_trace(YAFFS_TRACE_SCAN,
						"Partially written block %d detected. gc will fix this.", blk);
				}
			}
		}
		dev->n_free_chunks++;
	} else if (tags.ecc_result ==
		YAFFS_ECC_RESULT_UNFIXED) {
//出现ECC校验失败
                yaffs_trace(YAFFS_TRACE_SCAN, " Unfixed ECC in chunk(%d:%d), chunk ignored",blk, chunk_in_block);
		dev->n_free_chunks++;
	} else if (tags.obj_id > YAFFS_MAX_OBJECT_ID ||
		   tags.chunk_id > YAFFS_MAX_CHUNK_ID ||
		   tags.obj_id == YAFFS_OBJECTID_SUMMARY ||
		   (tags.chunk_id > 0 &&  tags.n_bytes > dev->data_bytes_per_chunk) || tags.seq_number != bi->seq_number) {
                //对一些异常chunk做处理
		yaffs_trace(YAFFS_TRACE_SCAN, "Chunk (%d:%d) with bad tags:obj = %d, chunk_id = %d, n_bytes = %d, ignored", blk, chunk_in_block, tags.obj_id, tags.chunk_id, tags.n_bytes);
		dev->n_free_chunks++;
	} else if (tags.chunk_id > 0) {
        //所有的tag标记都正常，则开始进行后续处理
		/* chunk_id > 0 so it is a data chunk... */
		loff_t endpos;
		loff_t chunk_base = (tags.chunk_id - 1) *
					dev->data_bytes_per_chunk;

		*found_chunks = 1;
        //标记block中的chunk使用状况
		yaffs_set_chunk_bit(dev, blk, chunk_in_block);
		bi->pages_in_use++;
        //找到文件句柄。若没有，表示第一次扫到该文件，创建之
		in = yaffs_find_or_create_by_number(dev,tags.obj_id, YAFFS_OBJECT_TYPE_FILE);
		if (!in)  alloc_failed = 1;
		if (in &&  in->variant_type == YAFFS_OBJECT_TYPE_FILE &&
		    chunk_base < in->variant.file_variant.shrink_size) {
			if (!yaffs_put_chunk_in_file(in, tags.chunk_id, chunk, -1)) alloc_failed = 1;
//将该文件部分加入tnode tree
			endpos = chunk_base + tags.n_bytes;
			if (!in->valid && in->variant.file_variant.scanned_size < endpos) {
				in->variant.file_variant.
				    scanned_size = endpos;
				in->variant.file_variant.
				    file_size = endpos;
			}
		} else if (in) {
//该文件超出了文件shrink大小，表示这是一个被resize过的文件，则删除超出文件大小的部分
			yaffs_chunk_del(dev, chunk, 1, __LINE__);
		}
	} else {
//存放文件头部（chunk id为0）的chunk进行处理
		*found_chunks = 1;
		yaffs_set_chunk_bit(dev, blk, chunk_in_block);
		bi->pages_in_use++;
		oh = NULL;
		in = NULL;
		if (tags.extra_available) {
			in = yaffs_find_or_create_by_number(dev,
					tags.obj_id,
					tags.extra_obj_type);
			if (!in) alloc_failed = 1;
		}

		if (!in || (!in->valid && dev->param.disable_lazy_load) || tags.extra_shadows || (!in->valid && (tags.obj_id == YAFFS_OBJECTID_ROOT || tags.obj_id == YAFFS_OBJECTID_LOSTNFOUND))) {
//从头部中读取文件信息
			result = yaffs_rd_chunk_tags_nand(dev, chunk,  chunk_data,  NULL);
//文件头部的信息存放在chunk的数据起始位置
			oh = (struct yaffs_obj_hdr *)chunk_data;
//使用Inband tag的特殊处理
			if (dev->param.inband_tags) {
				oh->shadows_obj = oh->inband_shadowed_obj_id;
				oh->is_shrink = oh->inband_is_shrink;
			}
//创建文件obj
			if (!in) {
				in = yaffs_find_or_create_by_number(dev, tags.obj_id, oh->type);
				if (!in) alloc_failed = 1;
			}
		}
..........
		if (in->valid) {
//找到过同一文件。删除该chunk之前进行必要清理工作
			if ((in->variant_type == YAFFS_OBJECT_TYPE_FILE) && ((oh && oh->type == YAFFS_OBJECT_TYPE_FILE) || (tags.extra_available &&  tags.extra_obj_type == YAFFS_OBJECT_TYPE_FILE) )) {
				loff_t this_size = (oh) ? yaffs_oh_to_size(oh) : tags.extra_file_size;
				u32 parent_obj_id = (oh) ? oh->parent_obj_id : tags.extra_parent_id;
				is_shrink = (oh) ? oh->is_shrink : tags.extra_is_shrink;
//文件的父id若为特殊标记，则文件大小为0
				if (parent_obj_id == AFFS_OBJECTID_DELETED ||parent_obj_id == YAFFS_OBJECTID_UNLINKED) {
					this_size = 0;
					is_shrink = 1;
				}

				if (is_shrink &&  in->variant.file_variant.shrink_size > this_size)
					in->variant.file_variant.shrink_size = this_size;
				if (is_shrink) bi->has_shrink_hdr = 1;
			}
			/* Use existing - destroy this one. */
			yaffs_chunk_del(dev, chunk, 1, __LINE__);
		}

		if (!in->valid && in->variant_type !=
		    (oh ? oh->type : tags.extra_obj_type)) {
			yaffs_trace(YAFFS_TRACE_ERROR,
				"yaffs tragedy: Bad type, %d != %d, for object %d at chunk %d during scan",
				oh ? oh->type : tags.extra_obj_type,
				in->variant_type, tags.obj_id,
				chunk);
			in = yaffs_retype_obj(in, oh ? oh->type : tags.extra_obj_type);
		}
//Root目录和lostfound目录处理
		if (!in->valid &&  (tags.obj_id == YAFFS_OBJECTID_ROOT || tags.obj_id == YAFFS_OBJECTID_LOSTNFOUND)) {
			in->valid = 1;
			if (oh) {
				in->yst_mode = oh->yst_mode;
				yaffs_load_attribs(in, oh);
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
				in->yst_mode = oh->yst_mode;
				yaffs_load_attribs(in, oh);

				if (oh->shadows_obj > 0) yaffs_handle_shadowed_obj(dev,  oh->shadows_obj, 1);

				yaffs_set_obj_name_from_oh(in, oh);
				parent = yaffs_find_or_create_by_number(dev, oh->parent_obj_id, YAFFS_OBJECT_TYPE_DIRECTORY);
				file_size = yaffs_oh_to_size(oh);
				is_shrink = oh->is_shrink;
				equiv_id = oh->equiv_id;
			} else {
				in->variant_type = tags.extra_obj_type;
				parent = yaffs_find_or_create_by_number(dev, tags.extra_parent_id, YAFFS_OBJECT_TYPE_DIRECTORY);
				file_size = tags.extra_file_size;
				is_shrink = tags.extra_is_shrink;
				equiv_id = tags.extra_equiv_id;
				in->lazy_loaded = 1;
			}
			in->dirty = 0;
			if (!parent)
				alloc_failed = 1;
			if (parent &&
			    parent->variant_type == YAFFS_OBJECT_TYPE_UNKNOWN) {
				parent->variant_type =
					YAFFS_OBJECT_TYPE_DIRECTORY;
				INIT_LIST_HEAD(&parent->variant.dir_variant.children);
			} else if (!parent || parent->variant_type != YAFFS_OBJECT_TYPE_DIRECTORY) {
//文件若找不到父目录放到lost and found
				yaffs_trace(YAFFS_TRACE_ERROR,
					"yaffs tragedy: attempting to use non-directory as a directory in scan. Put in lost+found."
					);
				parent = dev->lost_n_found;
			}
			yaffs_add_obj_to_dir(parent, in);

			is_unlinked = (parent == dev->del_dir) || (parent == dev->unlinked_dir);

			if (is_shrink)
				bi->has_shrink_hdr = 1;

			switch (in->variant_type) {
			case YAFFS_OBJECT_TYPE_UNKNOWN:
				break;
			case YAFFS_OBJECT_TYPE_FILE:
				file_var = &in->variant.file_variant;
				if (file_var->scanned_size < file_size) {
				//文件扫到的大小小于文件chunk头部表示的大小
					file_var->file_size = file_size;
					file_var->scanned_size = file_size;
				}

				if (file_var->shrink_size > file_size)
					file_var->shrink_size = file_size;  break;
			case YAFFS_OBJECT_TYPE_HARDLINK:
				hl_var = &in->variant.hardlink_variant;
				if (!is_unlinked) {
					hl_var->equiv_id = equiv_id;
					list_add(&in->hard_links, hard_list); } break;
			case YAFFS_OBJECT_TYPE_DIRECTORY:
				/* Do nothing */
				break;
			case YAFFS_OBJECT_TYPE_SPECIAL:
				/* Do nothing */
				break;
			case YAFFS_OBJECT_TYPE_SYMLINK:
				sl_var = &in->variant.symlink_variant;
				if (oh) {
					sl_var->alias =
					    yaffs_clone_str(oh->alias);
					if (!sl_var->alias)
						alloc_failed = 1;
				}
				break;
			}
		}
	}
	return alloc_failed ? YAFFS_FAIL : YAFFS_OK;
}
```
以上，就完成了扫描过程。

Block分配
-------

执行Block分配一般是由写操作触发的，最后的三个函数调用顺序为yaffs_write_new_chunk-->yaffs_alloc_chunk-->yaffs_find_alloc_block。代码分析如下：
```C
static int yaffs_write_new_chunk(struct yaffs_dev *dev,const u8 *data, struct yaffs_ext_tags *tags, int use_reserver)
{
......//因为写信chunk，表示Flash上内容已经改变了，将Checkpoint的数据设置为无效
	yaffs2_checkpt_invalidate(dev);

	do {
......//分配chunk空间
		chunk = yaffs_alloc_chunk(dev, use_reserver, &bi);
		if (chunk < 0) {/* no space */break;}
		attempts++;

		if (dev->param.always_check_erased)
			bi->skip_erased_check = 0;

		if (!bi->skip_erased_check) {
//确认chunk是否已经擦除
			erased_ok = yaffs_check_chunk_erased(dev, chunk);
			if (erased_ok != YAFFS_OK) {
				yaffs_trace(YAFFS_TRACE_ERROR,
				  "**>\> yaffs chunk %d was not erased", chunk);
//如果chunk没有擦除，则删除该chunk，跳到下个可用block寻找chunk
				yaffs_chunk_del(dev, chunk, 1, __LINE__);
				yaffs_skip_rest_of_block(dev);
				continue;
			}
		}
//写数据和tag到chunk内
		write_ok = yaffs_wr_chunk_tags_nand(dev, chunk, data, tags);
		if (!bi->skip_erased_check)
			write_ok =  yaffs_verify_chunk_written(dev, chunk, data, tags);

		if (write_ok != YAFFS_OK) {
//执行写错的回调
			yaffs_handle_chunk_wr_error(dev, chunk, erased_ok);
			continue;
		}

		bi->skip_erased_check = 1;
//执行成功的回调
		yaffs_handle_chunk_wr_ok(dev, chunk, data, tags);

	} while (write_ok != YAFFS_OK && (yaffs_wr_attempts == 0 || attempts <= yaffs_wr_attempts));

	if (!write_ok) chunk = -1;
	if (attempts > 1) {
		yaffs_trace(YAFFS_TRACE_ERROR,"**>> yaffs write required %d attempts",attempts);
		dev->n_retried_writes += (attempts - 1);
	}
	return chunk;
}

static int yaffs_alloc_chunk(struct yaffs_dev *dev, int use_reserver, struct yaffs_block_info **block_ptr)
{
	if (dev->alloc_block < 0) {
//若当前无可分配的块，则分配块
		dev->alloc_block = yaffs_find_alloc_block(dev);
		dev->alloc_page = 0;
	}
	if (!use_reserver && !yaffs_check_alloc_available(dev, 1)) {
//不使用保留块也找不到可用块
		return -1;
	}

	if (dev->n_erased_blocks < (int)dev->param.n_reserved_blocks && dev->alloc_page == 0)
		yaffs_trace(YAFFS_TRACE_ALLOCATE, "Allocating reserve");
	if (dev->alloc_block >= 0) {
		bi = yaffs_get_block_info(dev, dev->alloc_block);
		ret_val = (dev->alloc_block * dev->param.chunks_per_block) +dev->alloc_page;
		bi->pages_in_use++;
		yaffs_set_chunk_bit(dev, dev->alloc_block, dev->alloc_page);
		dev->alloc_page++;
		dev->n_free_chunks--;

//若块已满，则标记状态为已满
		if (dev->alloc_page >= dev->param.chunks_per_block) {
			bi->block_state = YAFFS_BLOCK_STATE_FULL;
			dev->alloc_block = -1;
		}

		if (block_ptr)
			*block_ptr = bi;

		return ret_val;
	}

	yaffs_trace(YAFFS_TRACE_ERROR,
		"!!!!!!!!! Allocator out !!!!!!!!!!!!!!!!!");

	return -1;
}

static int yaffs_find_alloc_block(struct yaffs_dev *dev)
{
	u32 i;
	struct yaffs_block_info *bi;
//没有可使用的块则返回-1
	if (dev->n_erased_blocks < 1) {
	        yaffs_trace(YAFFS_TRACE_ERROR,"yaffs tragedy: no more erased blocks");
		return -1;
	}
//否则，遍历找到一个可使用的块
	for (i = dev->internal_start_block; i <= dev->internal_end_block; i++) {
		dev->alloc_block_finder++;
		if (dev->alloc_block_finder < (int)dev->internal_start_block || dev->alloc_block_finder > (int)dev->internal_end_block) {
			dev->alloc_block_finder = dev->internal_start_block;
		}

		bi = yaffs_get_block_info(dev, dev->alloc_block_finder);

		if (bi->block_state == YAFFS_BLOCK_STATE_EMPTY) {
//找到块的条件是该块的状态为EMPTY，此时返回块的编号			
bi->block_state = YAFFS_BLOCK_STATE_ALLOCATING;
			dev->seq_number++;
			bi->seq_number = dev->seq_number;
			dev->n_erased_blocks--;
			yaffs_trace(YAFFS_TRACE_ALLOCATE,
			  "Allocated block %d, seq  %d, %d left" ,
			   dev->alloc_block_finder, dev->seq_number,
			   dev->n_erased_blocks);
			return dev->alloc_block_finder;
		}
	}

	yaffs_trace(YAFFS_TRACE_ALWAYS,"yaffs tragedy: no more erased blocks, but there should have been %d",dev->n_erased_blocks);

	return -1;
}
```
Block擦除
-------

Block擦除的条件是：

*   遇到坏块，且无法正确执行坏块标记
*   脏数据回收
*   上层下Format Flash指令

其中脏数据回收是GC（Garbage Collection）机制在执行，后续章节会再对GC进行介绍。

擦除块执行的函数也很简单，代码如下：
```C
int yaffs_erase_block(struct yaffs_dev *dev, int block_no)
{
	block_no -= dev->block_offset;
	dev->n_erasures++;
	result = dev->drv.drv_erase_fn(dev, block_no);
	return result;
}

其中drv_erase_fn回调函数在Linux的系统下最后MTD块系统的mtd_erase函数：

int nandmtd_erase_block(struct yaffs_dev *dev, int block_no)
{
	struct mtd_info *mtd = yaffs_dev_to_mtd(dev);
	u32 addr = ((loff_t) block_no) * dev->param.total_bytes_per_chunk *dev->param.chunks_per_block;
	struct erase_info ei;
	int retval = 0;
........
	retval = mtd_erase(mtd, &ei);
.......
	if (retval == 0)return YAFFS_OK;
	return YAFFS_FAIL;
}
```
总结
--

以上，块初始化及管理大致过程分析结束，下一章节介绍Checkpoint机制