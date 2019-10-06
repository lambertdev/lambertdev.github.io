---
title: Linux虚拟文件系统（1）
tags:
  - Linux
  - VFS
  - 文件系统
url: 497.html
id: 497
categories:
  - Linux
  - Linux文件系统
date: 2018-07-28 23:21:45
---

VFS简介
=====

说明：本系列文章均以Linux 4.4为原型进行分析 Linux将系统中很多资源都抽象成文件，如Socket、设备节点、以及内存。可以如此做，归功于Linux操作系统的虚拟文件系统（VFS）。 有了VFS，无论底层文件系统格式是FAT、ext格式甚至是内存，（一般情况下）上层应用无论关心底层这些文件系统的实现细节，可以按照统一的方式对文件进行操作。 ![Linux虚拟文件系统（1）](http://pic.l2h.site/l2hsite屏幕快照 2018-07-28 下午10.54.04.png "Linux虚拟文件系统（1）") 如上图，用户态应用通过glibc提供的API对文件进行操作，如open()、read()、 write()等。Glibc将这些函数转换成Linux提供的系统调用，操作系统根据系统调用号执行对应的操作函数，如__syscall_open/__syscall_read/__syscall_write等。这样就走到了Linux的虚拟文件系统层（上图VFS）。 

Linux文件系统模型
===========

Linux VFS最重要的数据结构包括

*   **SuperBlock**：存储与已加载（Mount）的文件系统相关的信息
*   **inode**：与系统中文件关联的相关信息
*   **dentry**：文件系统目录描述符，用来关联文件系统中的目录
*   **file**：与特定进程锁打开文件相关的结构体

下图（引自<[Understanding the Linux Kernel](http://johnchukwuma.com/training/UnderstandingTheLinuxKernel3rdEdition.pdf)>）描述了这些数据结构间的关系。三个进程打开同一个文件，其中进程3通过不同的文件硬链接访问这个文件。可以看出进程打开一个文件后，通过对应的文件描述符关联了到文件对象。相同的硬链接文件，它对应的dentry对象也是相同的（虽然是同一个文件，不同硬链接的dentry对象也是不同的）。注：f_dentry目前已经由struct path **f_path**成员取代。不同的dentry对象最终指向的是同一个在某个超级块上的inode节点，最终指向磁盘文件。 注意到其中process 1访问对应dentry有一个dentry cache，这是Linux的一种缓存机制。可以将最近加载或者使用的dentry节点缓存到内存中，加速访问。 ![Linux虚拟文件系统（1）](http://pic.l2h.site/l2hsite图片 1.png "Linux虚拟文件系统（1）") 下边看一下具体的数据结构。

Super Block
-----------

系统中定义了一个已经挂载的超级块的链表super_blocks（双向循环链表），用锁sb_lock保护。
```C
// . fs/super.c
static LIST_HEAD(super_blocks);     //系统中超级块全局链表
static DEFINE_SPINLOCK(sb_lock);    //超级块访问保护自旋锁

// . include/linux/fs.h
struct super_block {
  struct list_head	s_list;		/* 超级块链表指针 */
  dev_t			s_dev;		/* search index; _not_ kdev_t */
  unsigned char		s_blocksize_bits;  //块大小所占用的bit数
  unsigned long		s_blocksize;       //块大小（字节为单位）
  loff_t			s_maxbytes;	/* 最大文件大小 */
  struct file_system_type	*s_type;   //文件系统类型
  const struct super_operations	*s_op;     //超级块操作
  const struct dquot_operations	*dq_op;
  const struct quotactl_ops	*s_qcop;
  const struct export_operations *s_export_op;
  unsigned long		s_flags;
  unsigned long		s_iflags;	/* internal SB_I_* flags */
  unsigned long		s_magic;
  struct dentry		*s_root;       //文件系统根目录对应的Dentry
  struct rw_semaphore	s_umount;
  int			s_count;
  atomic_t		s_active;
#ifdef CONFIG_SECURITY
  void                    *s_security;
#endif
  const struct xattr_handler **s_xattr;

  struct hlist_bl_head	s_anon;		/* anonymous dentries for (nfs) exporting */
  struct list_head	s_mounts;	/* list of mounts; _not_ for fs use */
  //省略部分变量。。。。。
  int s_stack_depth;
  /* s_inode_list_lock protects s_inodes */
  spinlock_t		s_inode_list_lock ____cacheline_aligned_in_smp;
  struct list_head	s_inodes;	/* all inodes */
};
struct super_operations {
   struct inode *(*alloc_inode)(struct super_block *sb);
   void (*destroy_inode)(struct inode *);

   void (*dirty_inode) (struct inode *, int flags);
//省略大部分变量
};
```
dentry
------

与目录相关的文件结构
```C
// include/fs/dcache.h
struct dentry {
  /* RCU lookup touched fields */
  unsigned int d_flags;		/* 目录项高速缓存标记protected by d_lock */
  seqcount_t d_seq;		/* per dentry seqlock */
  struct hlist_bl_node d_hash;	/* lookup hash list */
  struct dentry *d_parent;	/* 指向父目录的denetry */
  struct qstr d_name;           /* 目录名 */
  struct inode *d_inode;		/* 目录inode节点 */
  unsigned char d_iname[DNAME_INLINE_LEN];	/* small names */
  /* Ref lookup also touches following */
  struct lockref d_lockref;	/* per-dentry lock and refcount */
  const struct dentry_operations *d_op;  /*存在与dentry相关的操作函数*/
  struct super_block *d_sb;	/* The root of the dentry tree */
  unsigned long d_time;		/* used by d_revalidate */
  void *d_fsdata;			/* fs-specific data */

  struct list_head d_lru;		/* LRU list */
  struct list_head d_child;	/* child of parent list */
  struct list_head d_subdirs;	/* our children */
  /*
   * d_alias and d_rcu can share memory
   */
  union {
    struct hlist_node d_alias;	/* inode alias list */
   	struct rcu_head d_rcu;
  } d_u;
};

enum dentry_d_lock_class
{
  DENTRY_D_LOCK_NORMAL, 
  DENTRY_D_LOCK_NESTED
};

struct dentry_operations {
  int (*d_revalidate)(struct dentry *, unsigned int);
  int (*d_weak_revalidate)(struct dentry *, unsigned int);
  int (*d_hash)(const struct dentry *, struct qstr *);
  int (*d_compare)(const struct dentry *, const struct dentry *,
      unsigned int, const char *, const struct qstr *);
  int (*d_delete)(const struct dentry *);
  void (*d_release)(struct dentry *);
  void (*d_prune)(struct dentry *);
  void (*d_iput)(struct dentry *, struct inode *);
  char *(*d_dname)(struct dentry *, char *, int);
  struct vfsmount *(*d_automount)(struct path *);
  int (*d_manage)(struct dentry *, bool);
  struct inode *(*d_select_inode)(struct dentry *, unsigned);
  struct dentry *(*d_real)(struct dentry *, struct inode *);
} ____cacheline_aligned;
```
inode
-----
```C
// include/linux/fs.h

struct inode {
  umode_t			i_mode;     //文件类型与访问权限
  unsigned short		i_opflags; 
  kuid_t			i_uid; //所有者标识
  kgid_t			i_gid; //组标识
  unsigned int		i_flags;
#ifdef CONFIG_FS_POSIX_ACL
  struct posix_acl	*i_acl;
  struct posix_acl	*i_default_acl;
#endif
  const struct inode_operations	*i_op;
  struct super_block	*i_sb;
  struct address_space	*i_mapping;
#ifdef CONFIG_SECURITY
  void			*i_security;
#endif
  unsigned long		i_ino;    //索引节点号
  union {                         //Hard Link数量
    const unsigned int i_nlink;
    unsigned int __i_nlink;
  };
  dev_t			i_rdev;
  loff_t			i_size;
  struct timespec		i_atime;i_mtime; i_ctime; //文件的访问时间等
  spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */
  unsigned short          i_bytes;
  unsigned int		i_blkbits;
  blkcnt_t		i_blocks;
#ifdef __NEED_I_SIZE_ORDERED
  seqcount_t		i_size_seqcount;
#endif
  /* Misc */
  unsigned long		i_state;
  struct mutex		i_mutex;
  unsigned long		dirtied_when;	/* jiffies of first dirtying */
  unsigned long		dirtied_time_when;
  struct hlist_node	i_hash;           //Inode Hash指针
  struct list_head	i_io_list;	/* backing dev IO list */
#ifdef CONFIG_CGROUP_WRITEBACK
  struct bdi_writeback	*i_wb;		/* the associated cgroup wb */

  /* foreign inode detection, see wbc_detach_inode() */
  int			i_wb_frn_winner;
  u16			i_wb_frn_avg_time;
  u16			i_wb_frn_history;
#endif
  struct list_head	i_lru;		/* inode LRU list */
  struct list_head	i_sb_list;
  union {
    struct hlist_head	i_dentry;       //引用索引节点的dentry链表头
    struct rcu_head		i_rcu;
  };
  u64			i_version;
  atomic_t		i_count;
  atomic_t		i_dio_count;
  atomic_t		i_writecount;
#ifdef CONFIG_IMA
  atomic_t		i_readcount; /* struct files open RO */
#endif
  const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
  struct file_lock_context	*i_flctx;
  struct address_space	i_data;
  struct list_head	i_devices;
  union {
    struct pipe_inode_info	*i_pipe;
    struct block_device	*i_bdev;
    struct cdev		*i_cdev;
    char			*i_link;
  };
  __u32			i_generation;  //索引节点版本号
#ifdef CONFIG_FSNOTIFY
  __u32			i_fsnotify_mask; /* all events this inode cares about */
  struct hlist_head	i_fsnotify_marks;
#endif

  void			*i_private; /* fs or device private pointer */ 
};
struct inode_operations {
	struct dentry * (*lookup) (struct inode *,struct dentry *, unsigned int);
	const char * (*follow_link) (struct dentry *, void **);
	int (*permission) (struct inode *, int);
	struct posix_acl * (*get_acl)(struct inode *, int);

	int (*readlink) (struct dentry *, char __user *,int);
	void (*put_link) (struct inode *, void *);

	int (*create) (struct inode *,struct dentry *, umode_t, bool);
	int (*link) (struct dentry *,struct inode *,struct dentry *);
	int (*unlink) (struct inode *,struct dentry *);
	int (*symlink) (struct inode *,struct dentry *,const char *);
	int (*mkdir) (struct inode *,struct dentry *,umode_t);
	int (*rmdir) (struct inode *,struct dentry *);
	int (*mknod) (struct inode *,struct dentry *,umode_t,dev_t);
	int (*rename) (struct inode *, struct dentry *,
			struct inode *, struct dentry *);
	int (*rename2) (struct inode *, struct dentry *,
			struct inode *, struct dentry *, unsigned int);
	int (*setattr) (struct dentry *, struct iattr *);
	int (*getattr) (struct vfsmount *mnt, struct dentry *, struct kstat *);
	int (*setxattr) (struct dentry *, const char *,const void *,size_t,int);
	ssize_t (*getxattr) (struct dentry *, const char *, void *, size_t);
	ssize_t (*listxattr) (struct dentry *, char *, size_t);
	int (*removexattr) (struct dentry *, const char *);
	int (*fiemap)(struct inode *, struct fiemap_extent_info *, u64 start,
		      u64 len);
	int (*update_time)(struct inode *, struct timespec *, int);
	int (*atomic_open)(struct inode *, struct dentry *,
			   struct file *, unsigned open_flag,
			   umode_t create_mode, int *opened);
	int (*tmpfile) (struct inode *, struct dentry *, umode_t);
	int (*set_acl)(struct inode *, struct posix_acl *, int);
} ____cacheline_aligned;
```
file
----
```C
// include/linux/fs.h

struct file {
  union {
    struct llist_node	fu_llist;
    struct rcu_head 	fu_rcuhead;
  } f_u;
  struct path		f_path;
  struct inode		*f_inode;	/* cached value */
  const struct file_operations	*f_op;

  spinlock_t		f_lock;
  atomic_long_t		f_count;
  unsigned int 		f_flags;
  fmode_t			f_mode;
  struct mutex		f_pos_lock;
  loff_t			f_pos;
  struct fown_struct	f_owner;
  const struct cred	*f_cred;
  struct file_ra_state	f_ra;

  u64			f_version;
#ifdef CONFIG_SECURITY
  void			*f_security;
#endif
  /* needed for tty driver, and maybe others */
  void			*private_data;

#ifdef CONFIG_EPOLL
  /* Used by fs/eventpoll.c to link all the hooks to this file */
  struct list_head	f_ep_links;
  struct list_head	f_tfile_llink;
#endif /* #ifdef CONFIG_EPOLL */
  struct address_space	*f_mapping;
} __attribute__((aligned(4)));	/* lest something weird decides that 2 is OK */
struct path {
	struct vfsmount *mnt;
	struct dentry *dentry;
};

struct file_handle {
  __u32 handle_bytes;
  int handle_type;
  /* file identifier */
  unsigned char f_handle[0];
};
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
	int (*iterate) (struct file *, struct dir_context *);
	unsigned int (*poll) (struct file *, struct poll_table_struct *);
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);
	int (*aio_fsync) (struct kiocb *, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
	int (*setlease)(struct file *, long, struct file_lock **, void **);
	long (*fallocate)(struct file *file, int mode, loff_t offset,
			  loff_t len);
	void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
	unsigned (*mmap_capabilities)(struct file *);
#endif
};
```
一张图说明这几个VFS重要结构体的指向关系 ![Linux虚拟文件系统（1）](http://pic.l2h.site/l2hsitevfs_struct_relation2.png "Linux虚拟文件系统（1）")