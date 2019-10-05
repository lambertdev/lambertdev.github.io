---
title: Linux设备驱动模型-- 数据结构Kset/KObject
tags:
  - KOBJ
  - KSET
  - Linux
  - 设备驱动
url: 2048.html
id: 2048
categories:
  - Linux
  - Linux设备驱动
date: 2019-03-06 21:20:08
---

前言
--

Kset和kobject是Linux设备驱动模型中的核心数据结构，其主要作用是将系统中的设备抽象出来，以树状结构组织，方便系统统一管理。

而这个统一管理的地方，就是sysfs，先放一张示例图，阐明一下kset和kobj的关系:

*   kset指向的是设备驱动模型中sysfs中的目录
*   kobject指向的是目录中的子节点（注意：目录本身也是节点）
*   sysfs子目录每一项kobj是使用该级目录对应kset的下的链表连接

```
   kset(/sys/bus)
                    +--------+
                    | kobj   |
                    |        |
                    +--------+
                    | list   |
       +------------+--------+
       |
       |
       |                      /sys/bus/pci
       v                       +--------+
/sys/bus/term1   /sys/bus/term2|        |
  +--------+     +--------+    |  kobj  |
  |        |     |        +--->+        |
  | kobj   +---->+ kobj   |    +--------+
  |        |     |        |    | list   |
  +--------+     +--------+    +---+----+
                                   |
                                   v
                        /sys/bus/pci/term3
                               +--------+
                               | kobj   |
                               +--------+
```

数据结构
----

kobject的数据结构如下：
```C
struct kobject {
	const char		*name;
	struct list_head	entry;
	struct kobject		*parent;
	struct kset		*kset;
	struct kobj_type	*ktype;
	struct kernfs_node	*sd; /* sysfs directory entry */
	struct kref		kref;
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
	struct delayed_work	release;
#endif
	unsigned int state_initialized:1;
	unsigned int state_in_sysfs:1;
	unsigned int state_add_uevent_sent:1;
	unsigned int state_remove_uevent_sent:1;
	unsigned int uevent_suppress:1;
};
```
解释如下：

*   name: 在sysfs下的名称
*   entry: kobject链表项，将该kobject串入链表
*   parent: 父级kobject
*   kset: 所属的kset
*   ktype: kobject对应的属性
*   sd: sysfs对应的目录项
*   kref: 原子操作引用计数
*   state_initialized : 是否初始化
*   state_in_sysfs: 该Kobject是否已在sysfs中呈现
*   state_add_uevent_sent/state_remove_uevent_sent: 记录是否已经向用户空间发送ADD uevent，如果有，且没有发送remove uevent，则在自动注销时，补发REMOVE uevent，以便让用户空间正确处理
*   uevent_suppress: 是否 忽略所有上报的uevent事件

kobj_type也定义在kobject.h
```C
struct kobj_type {
	void (*release)(struct kobject *kobj);
	const struct sysfs_ops *sysfs_ops;
	struct attribute **default_attrs;
	const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
	const void *(*namespace)(struct kobject *kobj);
};
```
*   release: 释放kobject所使用资源的回调函数
*   sysfs_ops: 顾名思义，与sysfs相关的操作
*   default_attrs: 保存类型属性的列表
*   chind_ns_types/namespace: 文件系统命名空间相关参数

而kset的数据结构如下：
```C
struct kset {
	struct list_head list;
	spinlock_t list_lock;
	struct kobject kobj;
	const struct kset_uevent_ops *uevent_ops;
};
```
解释如下：

*   list: 该kset下所有的kobject的链表
*   list_lock: 链表访问的自旋锁
*   kobj: kset对应的kobject
*   uevent_ops: uevent事件上报相关操作

kobject和kset操作
--------------

### kobject操作

kobject相关操作定义在 include/linux/kobject.h中，代码如下：
```C
extern void kobject_init(struct kobject *kobj, struct kobj_type *ktype);

int kobject_add(struct kobject *kobj, struct kobject *parent,
		const char *fmt, ...);

int kobject_init_and_add(struct kobject *kobj,
			 struct kobj_type *ktype, struct kobject *parent,
			 const char *fmt, ...);

extern void kobject_del(struct kobject *kobj);

extern struct kobject * __must_check kobject_create(void);
extern struct kobject * __must_check kobject_create_and_add(const char *name,struct kobject *parent);

extern int __must_check kobject_rename(struct kobject *, const char *new_name);
extern int __must_check kobject_move(struct kobject *, struct kobject *);
```
*   kobject_init： 已经分配好kobject的前提下，将kobj内的相关字段初始化。ktype为kobj的属性
*   kobject_add: 将该kobject加入到 sysfs树里
*   kobject_init_and_add： 同时调用kobject_init 和 kobject_add
*   kobject_del：将kobject从 sysfs树里去掉
*   kobject_create/kobject_create_and_add：向内核申请kboject空间，ktype使用系统全局的dynamic_kobj_ktype
*   kobject_rename: 修改kobject在sysfs树的名称

### kset操作

kset相关操作如下（同样定义在 include/linux/kobject.h 中）：
```C
extern void kset_init(struct kset *kset);
extern int __must_check kset_register(struct kset *kset);
extern void kset_unregister(struct kset *kset);
extern struct kset * __must_check kset_create_and_add(const char *name,
const struct kset_uevent_ops *u,struct kobject *parent_kobj);
static inline struct kset *to_kset(struct kobject *kobj)
static inline struct kset *kset_get(struct kset *k);
static inline void kset_put(struct kset *k);
static inline struct kobj_type *get_ktype(struct kobject *kobj);
extern struct kobject *kset_find_obj(struct kset *, const char *);
```
*   kset_init: 与kobject类似，将已经分配空间的kset初始化
*   kset_register: 初始化一个kset，加入到sysfs树里
*   kset_unregister: 将kset从sysfs树里移除，并将对应kobject删除
*   kset_create_and_add: 创建kset并加入sysfs树里
*   to_kset：从kobject找到对应kset
*   kset_get: 增加kset对应kobject引用记数
*   kset_put: 释放对kset对应kobject的 引用记数
*   get_ktype: 从kobject得到ktype
*   kset_find_obj: 从set下找到对应名称的kboject

实例
--

待添加