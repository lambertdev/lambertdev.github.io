---
title: YAFFS文件系统(4)– 垃圾回收(GC)机制
tags:
  - Linux
  - YAFFS
  - 文件系统
url: 2268.html
id: 2268
categories:
  - Linux文件系统
date: 2019-05-18 10:02:59
---

垃圾回收(Garbage Collection。 注：其实“垃圾回收”比较像是直译，叫“块回收”似乎比较恰当 )机制的主要作用是为了回收空闲或接近空闲的Nand块（Block），供文件系统后续使用（例如，创建文件、扩展文件大小、修改文件内容）。第一章《[概述](https://l2h.site/2019/04/30/yaffs-1/)》启动代码分析 yaffs\_bg\_start函数，其作用就是启用Linux内核线程，专门用来做垃圾回收。因为是内核线程做垃圾回收，后台工作找到可以回收的Flash块就用内核空闲的时间做回收，因此较大程度提升了Yaffs的工作性能。

“垃圾回收”的核心思想为：

*   先找到可以或者说值得回收的Block。注意：这里值得回收并非指的是Block内所有的Chunk（或者说Page）都无用了。Yaffs的回收器会根据一定的算法（本章之后小节会介绍）来找到一个可回收的Block。
*   对该Block内所有的Chunk进行遍历。若当前Chunk正在使用中，会将该Chunk复制一份到另外的Block内，并删除该Chunk。同时要更新文件Tnode、Block使用等内存中的信息。
    *   所以同一个文件存储到Flash上后，就算不被改写，存放位置也不一定是一成不变的。
    *   另外，因此我们也会看到Flash上可能会存储同一个文件的同一个Chunk在不同的位置（当然也有可能是文件被改写造成的）。Yaffs用seq\_number位来标记使用哪一个chunk作为当前文件真正要使用的chunk（seq\_number大者为较新）。
*   当Block内所有的Chunk都顺利删除后，则可以对Block进行擦除，供后续使用。

Yaffs的垃圾回收有两种模式，取决于当前Nand Flash空间的使用状况。当Flash中仍有许多可用Block时，回收机制会相对缓和地回收（实际程序执行上，Yaffs后台程序会每次回收较少数量的chunk而不用回收完一个完整的Block），称为Passive（被动）模式，此时需要尽快释放出一个完整可用的block。否则称为Aggressive（主动）模式。两种模式的判断依据为，Flash所剩可用的Block数量是否小于预留Block数量加上Checkpoint所需Block数量。

以下是代码分析。首先看入口函数：

static int yaffs\_bg\_thread_fn(void *data)
{
.......
	while (context->bg_running) {
//此循环为后台垃圾回收线程主循环
		yaffs\_trace(YAFFS\_TRACE\_BACKGROUND, "yaffs\_background");
		if (kthread\_should\_stop()) break;
//访问yaffs临界区数据做保护
		yaffs\_gross\_lock(dev);

		now = jiffies;
//若目前已经到脏目录更新时间，则更新脏目录信息。
//脏目录：目录内的文件被添加、修改或删除，则需要更新目录的访问时间等信息。Yaffs将此操作延后到垃圾回收时间，主要是性能考虑。例：若某段时间内该目录下有多个文件被修改，可以最后做一次更新而不用每次修改都更新该目录的访问时间
		if (time\_after(now, next\_dir\_update) && yaffs\_bg_enable) {
			yaffs\_update\_dirty_dirs(dev);
			next\_dir\_update = now + HZ;
		}
//若目前已到垃圾回收时间，则开始垃圾回收，并计算下次垃圾回收时间
		if (time\_after(now, next\_gc) && yaffs\_bg\_enable) {
			if (!dev->is_checkpointed) {
				urgency = yaffs\_bg\_gc_urgency(dev);
//执行垃圾回收
				gc\_result = yaffs\_bg_gc(dev, urgency);
//根据紧急度来计算下次垃圾回收时间
				if (urgency > 1)
					next_gc = now + HZ / 20 + 1;
				else if (urgency > 0)
					next_gc = now + HZ / 10 + 1;
				else
					next_gc = now + HZ * 2;
			} else	{
				next\_gc = next\_dir_update;
                        }
		}
		yaffs\_gross\_unlock(dev);
//计算下次唤醒该线程的时间
		expires = next\_dir\_update;
		if (time\_before(next\_gc, expires))
			expires = next_gc;
		if (time_before(expires, now))
			expires = now + HZ;
//将垃圾回收线程（也就是正在执行的线程自己）让出调度，同时增加内核定时器到时唤醒此线程
		Y\_INIT\_TIMER(&timer.timer, yaffs\_background\_waker);
		timer.timer.expires = expires + 1;
		timer.task = current;
		set\_current\_state(TASK_INTERRUPTIBLE);
		add_timer(&timer);
		schedule();
		del\_timer\_sync(&timer);
	}
	return 0;
}

先看Yaffs如何进行脏目录更新：

void yaffs\_update\_dirty\_dirs(struct yaffs\_dev *dev)
{
........
	while (!list\_empty(&dev->dirty\_dirs)) {
		link = dev->dirty_dirs.next;
		list\_del\_init(link);

		d\_s = list\_entry(link, struct yaffs\_dir\_var, dirty);
		o\_v = list\_entry(d\_s, union yaffs\_obj\_var, dir\_variant);
		obj = list\_entry(o\_v, struct yaffs_obj, variant);
		if (obj->dirty)
//执行更新目录的操作
			yaffs\_update\_oh(obj, NULL, 0, 0, 0, NULL);
	}
}
int yaffs\_update\_oh(struct yaffs\_obj \*in, const YCHAR \*name, int force,int is\_shrink, int shadows, struct yaffs\_xattr\_mod *xmod)
{
	strcpy(old\_name, \_Y("silly old name"));

	if (in->fake && in != dev->root_dir && !force && !xmod)
		return ret_val;
//垃圾回收代码，后续分析
	yaffs\_check\_gc(dev, 0);
	yaffs\_check\_obj\_details\_loaded(in);
	buffer = yaffs\_get\_temp\_buffer(in->my\_dev);
	oh = (struct yaffs\_obj\_hdr *)buffer;
	prev\_chunk\_id = in->hdr_chunk;

	if (prev\_chunk\_id > 0) {
//从Flash读出文件或者目录的标签（主要是读出文件名）
		result = yaffs\_rd\_chunk\_tags\_nand(dev, prev\_chunk\_id,buffer, &old_tags);
		if (result == YAFFS_OK) {
//对新分配的目录对象赋值
			yaffs\_verify\_oh(in, oh, &old_tags, 0);
			memcpy(old_name, oh->name, sizeof(oh->name));
			memset(oh, 0xff, sizeof(*oh));
		}
	} else {
		memset(buffer, 0xff, dev->data\_bytes\_per_chunk);
	}

	oh->type = in->variant_type;
	oh->yst\_mode = in->yst\_mode;
	oh->shadows\_obj = oh->inband\_shadowed\_obj\_id = shadows;
//文件属性赋值
	yaffs\_load\_attribs_oh(oh, in);
//父目录赋值
	if (in->parent)
		oh->parent\_obj\_id = in->parent->obj_id;
	else oh->parent\_obj\_id = 0;
//文件名赋值
	if (name && *name) {
		memset(oh->name, 0, sizeof(oh->name));
		yaffs\_load\_oh\_from\_name(dev, oh->name, name);
	} else if (prev\_chunk\_id > 0) {
		memcpy(oh->name, old_name, sizeof(oh->name));
	} else {
		memset(oh->name, 0, sizeof(oh->name));
	}
	oh->is\_shrink = is\_shrink;
	switch (in->variant_type) {
	case YAFFS\_OBJECT\_TYPE_UNKNOWN:
		break;
	case YAFFS\_OBJECT\_TYPE_FILE:
		if (oh->parent\_obj\_id != YAFFS\_OBJECTID\_DELETED &&
		    oh->parent\_obj\_id != YAFFS\_OBJECTID\_UNLINKED)
			file\_size = in->variant.file\_variant.stored_size;
		yaffs\_oh\_size\_load(dev, oh, file\_size, 0);
		break;
	case YAFFS\_OBJECT\_TYPE_HARDLINK:
		oh->equiv\_id = in->variant.hardlink\_variant.equiv_id;
		break;
	case YAFFS\_OBJECT\_TYPE_SPECIAL:
		/\* Do nothing */
		break;
	case YAFFS\_OBJECT\_TYPE_DIRECTORY:
		/\* Do nothing */
		break;
	case YAFFS\_OBJECT\_TYPE_SYMLINK:
		alias = in->variant.symlink_variant.alias;
		if (!alias) alias = _Y("no alias");
		strncpy(oh->alias, alias, YAFFS\_MAX\_ALIAS_LENGTH);
		oh->alias\[YAFFS\_MAX\_ALIAS_LENGTH\] = 0;
		break;
	}
//赋值过程
	if (xmod)
		yaffs\_apply\_xattrib_mod(in, (char *)buffer, xmod);
	memset(&new\_tags, 0, sizeof(new\_tags));
	in->serial++;
	new\_tags.chunk\_id = 0;
	new\_tags.obj\_id = in->obj_id;
	new\_tags.serial\_number = in->serial;
	new\_tags.extra\_available = 1;
	new\_tags.extra\_parent\_id = oh->parent\_obj_id;
	new\_tags.extra\_file\_size = file\_size;
	new\_tags.extra\_is\_shrink = oh->is\_shrink;
	new\_tags.extra\_equiv\_id = oh->equiv\_id;
	new\_tags.extra\_shadows = (oh->shadows_obj > 0) ? 1 : 0;
	new\_tags.extra\_obj\_type = in->variant\_type;
	yaffs\_do\_endian_oh(dev, oh);
//将新文件信息写回Nand Flash
	yaffs\_verify\_oh(in, oh, &new_tags, 1);
	new\_chunk\_id =  yaffs\_write\_new\_chunk(dev, buffer, &new\_tags,(prev\_chunk\_id > 0) ? 1 : 0);

	if (buffer) yaffs\_release\_temp_buffer(dev, buffer);

	if (new\_chunk\_id < 0) return new\_chunk\_id;

	in->hdr\_chunk = new\_chunk_id;
//删除原有的文件chunk（因为我们已经有替代者了）。注意这里删除只是简单的对文件所在block的bitmap做一下标记
	if (prev\_chunk\_id > 0) yaffs\_chunk\_del(dev, prev\_chunk\_id, 1, \_\_LINE\_\_);
	if (!yaffs\_obj\_cache_dirty(in))
		in->dirty = 0;
	if (is_shrink) {
		bi = yaffs\_get\_block\_info(in->my\_dev, new\_chunk\_id / in->my\_dev->param.chunks\_per_block);
		bi->has\_shrink\_hdr = 1;
	}
	return new\_chunk\_id;
}

紧急程度计算方法见yaffs\_bg\_gc_urgency函数如下：

static unsigned yaffs\_bg\_gc\_urgency(struct yaffs\_dev *dev)
{
	unsigned erased\_chunks = dev->n\_erased\_blocks * dev->param.chunks\_per_block;
	struct yaffs\_linux\_context *context = yaffs\_dev\_to_lc(dev);
	unsigned scattered = 0;	/* Free chunks not in an erased block */
//scattered为计算除空闲block外，整个flash剩余的可用chunk数量
	if (erased\_chunks < dev->n\_free\_chunks) scattered = (dev->n\_free\_chunks - erased\_chunks);
//若非后台运行，则直接返回urgency0
	if (!context->bg_running)
		return 0;
//若数量小于2个block，表示暂时不需要开辟新block来做chunk分配
	else if (scattered < (dev->param.chunks\_per\_block * 2))
		return 0;
//当可用非空block的空闲chunk已经不足2个block。则比较空闲block数量占所有空闲chunk比重。若占比较大，则回收紧急程度不高，否则紧急程度较高
	else if (erased\_chunks > dev->n\_free_chunks / 2)
		return 0;
	else if (erased\_chunks > dev->n\_free_chunks / 4)
		return 1;
	else
		return 2;
}

最核心的垃圾回收代码yaffs\_bg\_gc如下：

int yaffs\_bg\_gc(struct yaffs_dev *dev, unsigned urgency)
{
	int erased\_chunks = dev->n\_erased\_blocks * dev->param.chunks\_per_block;
//垃圾回收
	yaffs\_check\_gc(dev, 1);
	return erased\_chunks > dev->n\_free_chunks / 2;
}
static int yaffs\_check\_gc(struct yaffs_dev *dev, int background)
{
	if (dev->param.gc\_control\_fn && (dev->param.gc\_control\_fn(dev) & 1) == 0)
		return YAFFS_OK;
	if (dev->gc\_disable) return YAFFS\_OK;
//执行垃圾回收
	do {
		max_tries++;
//计算checkpoint所需的Flash Block数量
		checkpt\_block\_adjust = yaffs\_calc\_checkpt\_blocks\_required(dev);
//计算最少需要的块数量
		min\_erased = dev->param.n\_reserved\_blocks + checkpt\_block_adjust + 1;
		erased\_chunks = dev->n\_erased\_blocks * dev->param.chunks\_per_block;
//若空闲Block数量小于最小需求，则需要进行主动回收
		if (dev->n\_erased\_blocks < min_erased)
			aggressive = 1;
		else {
//若非后台回收，且空闲的block对应chunk数量大于总空闲chunk数量（含已分配Block中未使用的chunk）的四分之一则退出循环
			if (!background && erased\_chunks > (dev->n\_free_chunks / 4)) break;
			if (dev->gc\_skip > 20) dev->gc\_skip = 20;
			if (erased\_chunks < dev->n\_free\_chunks / 2 || dev->gc\_skip < 1 || background) aggressive = 0;
			else {
				dev->gc_skip--;
				break;
			}
		}

		dev->gc_skip = 5;
//若目前还未指定到要回收的Block且执行被动模式，则尝试找到一个seq最老的需要刷新的block。
		if (dev->gc_block < 1 && !aggressive) {
			dev->gc\_block = yaffs2\_find\_refresh\_block(dev);
			dev->gc_chunk = 0;
			dev->n\_clean\_ups = 0;
		}
//若仍未找到可回收块。则查找最脏的块进行回收
		if (dev->gc_block < 1) {
			dev->gc\_block = yaffs\_find\_gc\_block(dev, aggressive, background);
			dev->gc_chunk = 0;
			dev->n\_clean\_ups = 0;
		}
//进行回收
		if (dev->gc_block > 0) {
			dev->all_gcs++;
			if (!aggressive) dev->passive\_gc\_count++;
			gc\_ok = yaffs\_gc\_block(dev, dev->gc\_block, aggressive);
		}

		if (dev->n\_erased\_blocks < (int)dev->param.n\_reserved\_blocks && dev->gc_block > 0) {
			yaffs\_trace(YAFFS\_TRACE\_GC, "yaffs: GC !!!no reclaim!!! n\_erased\_blocks %d after try %d block %d", dev->n\_erased\_blocks, max\_tries, dev->gc_block);
		}
	} while ((dev->n\_erased\_blocks < (int)dev->param.n\_reserved\_blocks) &&  (dev->gc\_block > 0) && (max\_tries < 2));

	return aggressive ? gc\_ok : YAFFS\_OK;
}

yaffs\_gc\_block代码如下：

static int yaffs\_gc\_block(struct yaffs\_dev *dev, int block, int whole\_block)
{
.......
	int chunks\_before = yaffs\_get\_erased\_chunks(dev);
	struct yaffs\_block\_info *bi = yaffs\_get\_block_info(dev, block);
	is\_checkpt\_block = (bi->block\_state == YAFFS\_BLOCK\_STATE\_CHECKPOINT);
//改变Block状态为正在回收
	if (bi->block\_state == YAFFS\_BLOCK\_STATE\_FULL) bi->block\_state = YAFFS\_BLOCK\_STATE\_COLLECTING;

	bi->has\_shrink\_hdr = 0;	
//因为正在做回收，暂时关闭回收
	dev->gc_disable = 1;
//回收summary chunk
	yaffs\_summary\_gc(dev, block);
//若是checkpoint block或者block没有正在使用的chunk，直接回收
	if (is\_checkpt\_block || !yaffs\_still\_some_chunks(dev, block)) {
		yaffs\_block\_became_dirty(dev, block);
	} else {
//否则，复制block内的使用中chunk到其他block并回收block
		u8 *buffer = yaffs\_get\_temp_buffer(dev);
		yaffs\_verify\_blk(dev, bi, block);
		max\_copies = (whole\_block) ? dev->param.chunks\_per\_block : 5;
		old\_chunk = block * dev->param.chunks\_per\_block + dev->gc\_chunk;

		for (/* init already done */ ;
		     ret\_val == YAFFS\_OK &&
		     dev->gc\_chunk < dev->param.chunks\_per_block &&
		     (bi->block\_state == YAFFS\_BLOCK\_STATE\_COLLECTING) &&
		     max\_copies > 0;  dev->gc\_chunk++, old_chunk++) {
			if (yaffs\_check\_chunk\_bit(dev, block, dev->gc\_chunk)) { 
				max_copies--;
//对每个chunk进行复制和回收
				ret\_val = yaffs\_gc\_process\_chunk(dev, bi,
							old_chunk,buffer);
			}
		}
		yaffs\_release\_temp_buffer(dev, buffer);
	}

	yaffs\_verify\_collected_blk(dev, bi, block);

	if (bi->block\_state == YAFFS\_BLOCK\_STATE\_COLLECTING) {
		bi->block\_state = YAFFS\_BLOCK\_STATE\_FULL;
	} else {
		for (i = 0; i < dev->n\_clean\_ups; i++) {
//对所有的object，进行一次性统一删除
			struct yaffs\_obj *object = yaffs\_find\_by\_number(dev, dev->gc\_cleanup\_list\[i\]);
			if (object) {
				yaffs\_free\_tnode(dev,  object->variant.file_variant.top);
				object->variant.file_variant.top = NULL;
				yaffs\_generic\_obj_del(object);
				object->my\_dev->n\_deleted_files--;
			}

		}
		chunks\_after = yaffs\_get\_erased\_chunks(dev);
		if (chunks\_before >= chunks\_after)
			yaffs\_trace(YAFFS\_TRACE\_GC, "gc did not increase free chunks before %d after %d", chunks\_before, chunks_after);
		dev->gc_block = 0;
		dev->gc_chunk = 0;
		dev->n\_clean\_ups = 0;
	}

	dev->gc_disable = 0;

	return ret_val;
}

以上，即为Yaffs垃圾回收机制的分析。至此，Yaffs文件系统代码分析结束。若有不恰当之处，也请大家多交流指出。