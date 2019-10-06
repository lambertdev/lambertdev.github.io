---
title: YAFFS文件系统(4)– 垃圾回收(GC)机制
tags:
  - Linux
  - YAFFS
  - 文件系统
url: 2268.html
id: 2268
categories:
  - Linux
  - Linux文件系统
date: 2019-05-18 10:02:59
---

垃圾回收(Garbage Collection。 注：其实“垃圾回收”比较像是直译，叫“块回收”似乎比较恰当 )机制的主要作用是为了回收空闲或接近空闲的Nand块（Block），供文件系统后续使用（例如，创建文件、扩展文件大小、修改文件内容）。第一章《[概述](https://www.l2h.site/2019/04/30/yaffs-1/)》启动代码分析 yaffs_bg_start函数，其作用就是启用Linux内核线程，专门用来做垃圾回收。因为是内核线程做垃圾回收，后台工作找到可以回收的Flash块就用内核空闲的时间做回收，因此较大程度提升了Yaffs的工作性能。

“垃圾回收”的核心思想为：

*   先找到可以或者说值得回收的Block。注意：这里值得回收并非指的是Block内所有的Chunk（或者说Page）都无用了。Yaffs的回收器会根据一定的算法（本章之后小节会介绍）来找到一个可回收的Block。
*   对该Block内所有的Chunk进行遍历。若当前Chunk正在使用中，会将该Chunk复制一份到另外的Block内，并删除该Chunk。同时要更新文件Tnode、Block使用等内存中的信息。
    *   所以同一个文件存储到Flash上后，就算不被改写，存放位置也不一定是一成不变的。
    *   另外，因此我们也会看到Flash上可能会存储同一个文件的同一个Chunk在不同的位置（当然也有可能是文件被改写造成的）。Yaffs用seq_number位来标记使用哪一个chunk作为当前文件真正要使用的chunk（seq_number大者为较新）。
*   当Block内所有的Chunk都顺利删除后，则可以对Block进行擦除，供后续使用。

Yaffs的垃圾回收有两种模式，取决于当前Nand Flash空间的使用状况。当Flash中仍有许多可用Block时，回收机制会相对缓和地回收（实际程序执行上，Yaffs后台程序会每次回收较少数量的chunk而不用回收完一个完整的Block），称为Passive（被动）模式，此时需要尽快释放出一个完整可用的block。否则称为Aggressive（主动）模式。两种模式的判断依据为，Flash所剩可用的Block数量是否小于预留Block数量加上Checkpoint所需Block数量。

以下是代码分析。首先看入口函数：
```C
static int yaffs_bg_thread_fn(void *data)
{
.......
	while (context->bg_running) {
//此循环为后台垃圾回收线程主循环
		yaffs_trace(YAFFS_TRACE_BACKGROUND, "yaffs_background");
		if (kthread_should_stop()) break;
//访问yaffs临界区数据做保护
		yaffs_gross_lock(dev);

		now = jiffies;
//若目前已经到脏目录更新时间，则更新脏目录信息。
//脏目录：目录内的文件被添加、修改或删除，则需要更新目录的访问时间等信息。Yaffs将此操作延后到垃圾回收时间，主要是性能考虑。例：若某段时间内该目录下有多个文件被修改，可以最后做一次更新而不用每次修改都更新该目录的访问时间
		if (time_after(now, next_dir_update) && yaffs_bg_enable) {
			yaffs_update_dirty_dirs(dev);
			next_dir_update = now + HZ;
		}
//若目前已到垃圾回收时间，则开始垃圾回收，并计算下次垃圾回收时间
		if (time_after(now, next_gc) && yaffs_bg_enable) {
			if (!dev->is_checkpointed) {
				urgency = yaffs_bg_gc_urgency(dev);
//执行垃圾回收
				gc_result = yaffs_bg_gc(dev, urgency);
//根据紧急度来计算下次垃圾回收时间
				if (urgency > 1)
					next_gc = now + HZ / 20 + 1;
				else if (urgency > 0)
					next_gc = now + HZ / 10 + 1;
				else
					next_gc = now + HZ * 2;
			} else	{
				next_gc = next_dir_update;
                        }
		}
		yaffs_gross_unlock(dev);
//计算下次唤醒该线程的时间
		expires = next_dir_update;
		if (time_before(next_gc, expires))
			expires = next_gc;
		if (time_before(expires, now))
			expires = now + HZ;
//将垃圾回收线程（也就是正在执行的线程自己）让出调度，同时增加内核定时器到时唤醒此线程
		Y_INIT_TIMER(&timer.timer, yaffs_background_waker);
		timer.timer.expires = expires + 1;
		timer.task = current;
		set_current_state(TASK_INTERRUPTIBLE);
		add_timer(&timer);
		schedule();
		del_timer_sync(&timer);
	}
	return 0;
}
```
先看Yaffs如何进行脏目录更新：
```C
void yaffs_update_dirty_dirs(struct yaffs_dev *dev)
{
........
	while (!list_empty(&dev->dirty_dirs)) {
		link = dev->dirty_dirs.next;
		list_del_init(link);

		d_s = list_entry(link, struct yaffs_dir_var, dirty);
		o_v = list_entry(d_s, union yaffs_obj_var, dir_variant);
		obj = list_entry(o_v, struct yaffs_obj, variant);
		if (obj->dirty)
//执行更新目录的操作
			yaffs_update_oh(obj, NULL, 0, 0, 0, NULL);
	}
}
int yaffs_update_oh(struct yaffs_obj \*in, const YCHAR \*name, int force,int is_shrink, int shadows, struct yaffs_xattr_mod *xmod)
{
	strcpy(old_name, _Y("silly old name"));

	if (in->fake && in != dev->root_dir && !force && !xmod)
		return ret_val;
//垃圾回收代码，后续分析
	yaffs_check_gc(dev, 0);
	yaffs_check_obj_details_loaded(in);
	buffer = yaffs_get_temp_buffer(in->my_dev);
	oh = (struct yaffs_obj_hdr *)buffer;
	prev_chunk_id = in->hdr_chunk;

	if (prev_chunk_id > 0) {
//从Flash读出文件或者目录的标签（主要是读出文件名）
		result = yaffs_rd_chunk_tags_nand(dev, prev_chunk_id,buffer, &old_tags);
		if (result == YAFFS_OK) {
//对新分配的目录对象赋值
			yaffs_verify_oh(in, oh, &old_tags, 0);
			memcpy(old_name, oh->name, sizeof(oh->name));
			memset(oh, 0xff, sizeof(*oh));
		}
	} else {
		memset(buffer, 0xff, dev->data_bytes_per_chunk);
	}

	oh->type = in->variant_type;
	oh->yst_mode = in->yst_mode;
	oh->shadows_obj = oh->inband_shadowed_obj_id = shadows;
//文件属性赋值
	yaffs_load_attribs_oh(oh, in);
//父目录赋值
	if (in->parent)
		oh->parent_obj_id = in->parent->obj_id;
	else oh->parent_obj_id = 0;
//文件名赋值
	if (name && *name) {
		memset(oh->name, 0, sizeof(oh->name));
		yaffs_load_oh_from_name(dev, oh->name, name);
	} else if (prev_chunk_id > 0) {
		memcpy(oh->name, old_name, sizeof(oh->name));
	} else {
		memset(oh->name, 0, sizeof(oh->name));
	}
	oh->is_shrink = is_shrink;
	switch (in->variant_type) {
	case YAFFS_OBJECT_TYPE_UNKNOWN:
		break;
	case YAFFS_OBJECT_TYPE_FILE:
		if (oh->parent_obj_id != YAFFS_OBJECTID_DELETED &&
		    oh->parent_obj_id != YAFFS_OBJECTID_UNLINKED)
			file_size = in->variant.file_variant.stored_size;
		yaffs_oh_size_load(dev, oh, file_size, 0);
		break;
	case YAFFS_OBJECT_TYPE_HARDLINK:
		oh->equiv_id = in->variant.hardlink_variant.equiv_id;
		break;
	case YAFFS_OBJECT_TYPE_SPECIAL:
		/\* Do nothing */
		break;
	case YAFFS_OBJECT_TYPE_DIRECTORY:
		/\* Do nothing */
		break;
	case YAFFS_OBJECT_TYPE_SYMLINK:
		alias = in->variant.symlink_variant.alias;
		if (!alias) alias = _Y("no alias");
		strncpy(oh->alias, alias, YAFFS_MAX_ALIAS_LENGTH);
		oh->alias[YAFFS_MAX_ALIAS_LENGTH] = 0;
		break;
	}
//赋值过程
	if (xmod)
		yaffs_apply_xattrib_mod(in, (char *)buffer, xmod);
	memset(&new_tags, 0, sizeof(new_tags));
	in->serial++;
	new_tags.chunk_id = 0;
	new_tags.obj_id = in->obj_id;
	new_tags.serial_number = in->serial;
	new_tags.extra_available = 1;
	new_tags.extra_parent_id = oh->parent_obj_id;
	new_tags.extra_file_size = file_size;
	new_tags.extra_is_shrink = oh->is_shrink;
	new_tags.extra_equiv_id = oh->equiv_id;
	new_tags.extra_shadows = (oh->shadows_obj > 0) ? 1 : 0;
	new_tags.extra_obj_type = in->variant_type;
	yaffs_do_endian_oh(dev, oh);
//将新文件信息写回Nand Flash
	yaffs_verify_oh(in, oh, &new_tags, 1);
	new_chunk_id =  yaffs_write_new_chunk(dev, buffer, &new_tags,(prev_chunk_id > 0) ? 1 : 0);

	if (buffer) yaffs_release_temp_buffer(dev, buffer);

	if (new_chunk_id < 0) return new_chunk_id;

	in->hdr_chunk = new_chunk_id;
//删除原有的文件chunk（因为我们已经有替代者了）。注意这里删除只是简单的对文件所在block的bitmap做一下标记
	if (prev_chunk_id > 0) yaffs_chunk_del(dev, prev_chunk_id, 1, __LINE__);
	if (!yaffs_obj_cache_dirty(in))
		in->dirty = 0;
	if (is_shrink) {
		bi = yaffs_get_block_info(in->my_dev, new_chunk_id / in->my_dev->param.chunks_per_block);
		bi->has_shrink_hdr = 1;
	}
	return new_chunk_id;
}

紧急程度计算方法见yaffs_bg_gc_urgency函数如下：

static unsigned yaffs_bg_gc_urgency(struct yaffs_dev *dev)
{
	unsigned erased_chunks = dev->n_erased_blocks * dev->param.chunks_per_block;
	struct yaffs_linux_context *context = yaffs_dev_to_lc(dev);
	unsigned scattered = 0;	/* Free chunks not in an erased block */
//scattered为计算除空闲block外，整个flash剩余的可用chunk数量
	if (erased_chunks < dev->n_free_chunks) scattered = (dev->n_free_chunks - erased_chunks);
//若非后台运行，则直接返回urgency0
	if (!context->bg_running)
		return 0;
//若数量小于2个block，表示暂时不需要开辟新block来做chunk分配
	else if (scattered < (dev->param.chunks_per_block * 2))
		return 0;
//当可用非空block的空闲chunk已经不足2个block。则比较空闲block数量占所有空闲chunk比重。若占比较大，则回收紧急程度不高，否则紧急程度较高
	else if (erased_chunks > dev->n_free_chunks / 2)
		return 0;
	else if (erased_chunks > dev->n_free_chunks / 4)
		return 1;
	else
		return 2;
}
```
最核心的垃圾回收代码yaffs_bg_gc如下：
```C
int yaffs_bg_gc(struct yaffs_dev *dev, unsigned urgency)
{
	int erased_chunks = dev->n_erased_blocks * dev->param.chunks_per_block;
//垃圾回收
	yaffs_check_gc(dev, 1);
	return erased_chunks > dev->n_free_chunks / 2;
}
static int yaffs_check_gc(struct yaffs_dev *dev, int background)
{
	if (dev->param.gc_control_fn && (dev->param.gc_control_fn(dev) & 1) == 0)
		return YAFFS_OK;
	if (dev->gc_disable) return YAFFS_OK;
//执行垃圾回收
	do {
		max_tries++;
//计算checkpoint所需的Flash Block数量
		checkpt_block_adjust = yaffs_calc_checkpt_blocks_required(dev);
//计算最少需要的块数量
		min_erased = dev->param.n_reserved_blocks + checkpt_block_adjust + 1;
		erased_chunks = dev->n_erased_blocks * dev->param.chunks_per_block;
//若空闲Block数量小于最小需求，则需要进行主动回收
		if (dev->n_erased_blocks < min_erased)
			aggressive = 1;
		else {
//若非后台回收，且空闲的block对应chunk数量大于总空闲chunk数量（含已分配Block中未使用的chunk）的四分之一则退出循环
			if (!background && erased_chunks > (dev->n_free_chunks / 4)) break;
			if (dev->gc_skip > 20) dev->gc_skip = 20;
			if (erased_chunks < dev->n_free_chunks / 2 || dev->gc_skip < 1 || background) aggressive = 0;
			else {
				dev->gc_skip--;
				break;
			}
		}

		dev->gc_skip = 5;
//若目前还未指定到要回收的Block且执行被动模式，则尝试找到一个seq最老的需要刷新的block。
		if (dev->gc_block < 1 && !aggressive) {
			dev->gc_block = yaffs2_find_refresh_block(dev);
			dev->gc_chunk = 0;
			dev->n_clean_ups = 0;
		}
//若仍未找到可回收块。则查找最脏的块进行回收
		if (dev->gc_block < 1) {
			dev->gc_block = yaffs_find_gc_block(dev, aggressive, background);
			dev->gc_chunk = 0;
			dev->n_clean_ups = 0;
		}
//进行回收
		if (dev->gc_block > 0) {
			dev->all_gcs++;
			if (!aggressive) dev->passive_gc_count++;
			gc_ok = yaffs_gc_block(dev, dev->gc_block, aggressive);
		}

		if (dev->n_erased_blocks < (int)dev->param.n_reserved_blocks && dev->gc_block > 0) {
			yaffs_trace(YAFFS_TRACE_GC, "yaffs: GC !!!no reclaim!!! n_erased_blocks %d after try %d block %d", dev->n_erased_blocks, max_tries, dev->gc_block);
		}
	} while ((dev->n_erased_blocks < (int)dev->param.n_reserved_blocks) &&  (dev->gc_block > 0) && (max_tries < 2));

	return aggressive ? gc_ok : YAFFS_OK;
}
```
yaffs_gc_block代码如下：
```C
static int yaffs_gc_block(struct yaffs_dev *dev, int block, int whole_block)
{
.......
	int chunks_before = yaffs_get_erased_chunks(dev);
	struct yaffs_block_info *bi = yaffs_get_block_info(dev, block);
	is_checkpt_block = (bi->block_state == YAFFS_BLOCK_STATE_CHECKPOINT);
//改变Block状态为正在回收
	if (bi->block_state == YAFFS_BLOCK_STATE_FULL) bi->block_state = YAFFS_BLOCK_STATE_COLLECTING;

	bi->has_shrink_hdr = 0;	
//因为正在做回收，暂时关闭回收
	dev->gc_disable = 1;
//回收summary chunk
	yaffs_summary_gc(dev, block);
//若是checkpoint block或者block没有正在使用的chunk，直接回收
	if (is_checkpt_block || !yaffs_still_some_chunks(dev, block)) {
		yaffs_block_became_dirty(dev, block);
	} else {
//否则，复制block内的使用中chunk到其他block并回收block
		u8 *buffer = yaffs_get_temp_buffer(dev);
		yaffs_verify_blk(dev, bi, block);
		max_copies = (whole_block) ? dev->param.chunks_per_block : 5;
		old_chunk = block * dev->param.chunks_per_block + dev->gc_chunk;

		for (/* init already done */ ;
		     ret_val == YAFFS_OK &&
		     dev->gc_chunk < dev->param.chunks_per_block &&
		     (bi->block_state == YAFFS_BLOCK_STATE_COLLECTING) &&
		     max_copies > 0;  dev->gc_chunk++, old_chunk++) {
			if (yaffs_check_chunk_bit(dev, block, dev->gc_chunk)) { 
				max_copies--;
//对每个chunk进行复制和回收
				ret_val = yaffs_gc_process_chunk(dev, bi,
							old_chunk,buffer);
			}
		}
		yaffs_release_temp_buffer(dev, buffer);
	}

	yaffs_verify_collected_blk(dev, bi, block);

	if (bi->block_state == YAFFS_BLOCK_STATE_COLLECTING) {
		bi->block_state = YAFFS_BLOCK_STATE_FULL;
	} else {
		for (i = 0; i < dev->n_clean_ups; i++) {
//对所有的object，进行一次性统一删除
			struct yaffs_obj *object = yaffs_find_by_number(dev, dev->gc_cleanup_list[i]);
			if (object) {
				yaffs_free_tnode(dev,  object->variant.file_variant.top);
				object->variant.file_variant.top = NULL;
				yaffs_generic_obj_del(object);
				object->my_dev->n_deleted_files--;
			}

		}
		chunks_after = yaffs_get_erased_chunks(dev);
		if (chunks_before >= chunks_after)
			yaffs_trace(YAFFS_TRACE_GC, "gc did not increase free chunks before %d after %d", chunks_before, chunks_after);
		dev->gc_block = 0;
		dev->gc_chunk = 0;
		dev->n_clean_ups = 0;
	}

	dev->gc_disable = 0;

	return ret_val;
}
```
以上，即为Yaffs垃圾回收机制的分析。至此，Yaffs文件系统代码分析结束。若有不恰当之处，也请大家多交流指出。