---
title: Linux虚拟文件系统（1）
tags:
  - Linux
  - VFS
  - 文件系统
url: 497.html
id: 497
categories:
  - Linux文件系统
date: 2018-07-28 23:21:45
---

VFS简介
=====

说明：本系列文章均以Linux 4.4为原型进行分析 Linux将系统中很多资源都抽象成文件，如Socket、设备节点、以及内存。可以如此做，归功于Linux操作系统的虚拟文件系统（VFS）。 有了VFS，无论底层文件系统格式是FAT、ext格式甚至是内存，（一般情况下）上层应用无论关心底层这些文件系统的实现细节，可以按照统一的方式对文件进行操作。 ![Linux虚拟文件系统（1）](http://pic.l2h.site/l2hsite屏幕快照 2018-07-28 下午10.54.04.png "Linux虚拟文件系统（1）") 如上图，用户态应用通过glibc提供的API对文件进行操作，如open()、read()、 write()等。Glibc将这些函数转换成Linux提供的系统调用，操作系统根据系统调用号执行对应的操作函数，如\_\_syscall\_open/\_\_syscall\_read/\_\_syscall\_write等。这样就走到了Linux的虚拟文件系统层（上图VFS）。 

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

系统中定义了一个已经挂载的超级块的链表super\_blocks（双向循环链表），用锁sb\_lock保护。

// . fs/super.c
static LIST\_HEAD(super\_blocks);     //系统中超级块全局链表
static DEFINE\_SPINLOCK(sb\_lock);    //超级块访问保护自旋锁

// . include/linux/fs.h
struct super_block {
  struct list\_head	s\_list;		/* 超级块链表指针 */
  dev\_t			s\_dev;		/* search index; \_not\_ kdev_t */
  unsigned char		s\_blocksize\_bits;  //块大小所占用的bit数
  unsigned long		s_blocksize;       //块大小（字节为单位）
  loff\_t			s\_maxbytes;	/* 最大文件大小 */
  struct file\_system\_type	*s_type;   //文件系统类型
  const struct super\_operations	*s\_op;     //超级块操作
  const struct dquot\_operations	*dq\_op;
  const struct quotactl\_ops	*s\_qcop;
  const struct export\_operations *s\_export_op;
  unsigned long		s_flags;
  unsigned long		s\_iflags;	/* internal SB\_I_* flags */
  unsigned long		s_magic;
  struct dentry		*s_root;       //文件系统根目录对应的Dentry
  struct rw\_semaphore	s\_umount;
  int			s_count;
  atomic\_t		s\_active;
#ifdef CONFIG_SECURITY
  void                    *s_security;
#endif
  const struct xattr\_handler **s\_xattr;

  struct hlist\_bl\_head	s_anon;		/* anonymous dentries for (nfs) exporting */
  struct list\_head	s\_mounts;	/* list of mounts; \_not\_ for fs use */
  //省略部分变量。。。。。
  int s\_stack\_depth;
  /\* s\_inode\_list\_lock protects s\_inodes */
  spinlock\_t		s\_inode\_list\_lock \_\_\_\_cacheline\_aligned\_in\_smp;
  struct list\_head	s\_inodes;	/* all inodes */
};
struct super_operations {
   struct inode *(\*alloc\_inode)(struct super\_block \*sb);
   void (\*destroy_inode)(struct inode \*);

   void (\*dirty_inode) (struct inode \*, int flags);
//省略大部分变量
};

dentry
------

与目录相关的文件结构

// include/fs/dcache.h
struct dentry {
  /\* RCU lookup touched fields */
  unsigned int d\_flags;		/* 目录项高速缓存标记protected by d\_lock */
  seqcount\_t d\_seq;		/* per dentry seqlock */
  struct hlist\_bl\_node d_hash;	/* lookup hash list */
  struct dentry \*d_parent;	/\* 指向父目录的denetry */
  struct qstr d_name;           /* 目录名 */
  struct inode \*d_inode;		/\* 目录inode节点 */
  unsigned char d\_iname\[DNAME\_INLINE_LEN\];	/* small names */
  /\* Ref lookup also touches following */
  struct lockref d_lockref;	/* per-dentry lock and refcount */
  const struct dentry\_operations \*d\_op;  /\*存在与dentry相关的操作函数*/
  struct super\_block \*d\_sb;	/\* The root of the dentry tree */
  unsigned long d\_time;		/* used by d\_revalidate */
  void \*d_fsdata;			/\* fs-specific data */

  struct list\_head d\_lru;		/* LRU list */
  struct list\_head d\_child;	/* child of parent list */
  struct list\_head d\_subdirs;	/* our children */
  /\*
   \* d\_alias and d\_rcu can share memory
   */
  union {
    struct hlist\_node d\_alias;	/* inode alias list */
   	struct rcu\_head d\_rcu;
  } d_u;
};

enum dentry\_d\_lock_class
{
  DENTRY\_D\_LOCK_NORMAL, 
  DENTRY\_D\_LOCK_NESTED
};

struct dentry_operations {
  int (\*d_revalidate)(struct dentry \*, unsigned int);
  int (\*d\_weak\_revalidate)(struct dentry \*, unsigned int);
  int (\*d_hash)(const struct dentry \*, struct qstr *);
  int (\*d_compare)(const struct dentry \*, const struct dentry *,
      unsigned int, const char *, const struct qstr *);
  int (\*d_delete)(const struct dentry \*);
  void (\*d_release)(struct dentry \*);
  void (\*d_prune)(struct dentry \*);
  void (\*d_iput)(struct dentry \*, struct inode *);
  char *(\*d_dname)(struct dentry \*, char *, int);
  struct vfsmount *(\*d_automount)(struct path \*);
  int (\*d_manage)(struct dentry \*, bool);
  struct inode *(\*d\_select\_inode)(struct dentry \*, unsigned);
  struct dentry *(\*d_real)(struct dentry \*, struct inode *);
} \_\_\_\_cacheline\_aligned;

inode
-----

// include/linux/fs.h

struct inode {
  umode\_t			i\_mode;     //文件类型与访问权限
  unsigned short		i_opflags; 
  kuid\_t			i\_uid; //所有者标识
  kgid\_t			i\_gid; //组标识
  unsigned int		i_flags;
#ifdef CONFIG\_FS\_POSIX_ACL
  struct posix\_acl	*i\_acl;
  struct posix\_acl	*i\_default_acl;
#endif
  const struct inode\_operations	*i\_op;
  struct super\_block	*i\_sb;
  struct address\_space	*i\_mapping;
#ifdef CONFIG_SECURITY
  void			*i_security;
#endif
  unsigned long		i_ino;    //索引节点号
  union {                         //Hard Link数量
    const unsigned int i_nlink;
    unsigned int \_\_i\_nlink;
  };
  dev\_t			i\_rdev;
  loff\_t			i\_size;
  struct timespec		i\_atime;i\_mtime; i_ctime; //文件的访问时间等
  spinlock\_t		i\_lock;	/* i\_blocks, i\_bytes, maybe i_size */
  unsigned short          i_bytes;
  unsigned int		i_blkbits;
  blkcnt\_t		i\_blocks;
#ifdef \_\_NEED\_I\_SIZE\_ORDERED
  seqcount\_t		i\_size_seqcount;
#endif
  /\* Misc */
  unsigned long		i_state;
  struct mutex		i_mutex;
  unsigned long		dirtied_when;	/* jiffies of first dirtying */
  unsigned long		dirtied\_time\_when;
  struct hlist\_node	i\_hash;           //Inode Hash指针
  struct list\_head	i\_io_list;	/* backing dev IO list */
#ifdef CONFIG\_CGROUP\_WRITEBACK
  struct bdi\_writeback	\*i\_wb;		/\* the associated cgroup wb */

  /\* foreign inode detection, see wbc\_detach\_inode() */
  int			i\_wb\_frn_winner;
  u16			i\_wb\_frn\_avg\_time;
  u16			i\_wb\_frn_history;
#endif
  struct list\_head	i\_lru;		/* inode LRU list */
  struct list\_head	i\_sb_list;
  union {
    struct hlist\_head	i\_dentry;       //引用索引节点的dentry链表头
    struct rcu\_head		i\_rcu;
  };
  u64			i_version;
  atomic\_t		i\_count;
  atomic\_t		i\_dio_count;
  atomic\_t		i\_writecount;
#ifdef CONFIG_IMA
  atomic\_t		i\_readcount; /* struct files open RO */
#endif
  const struct file\_operations	\*i\_fop;	/\* former ->i\_op->default\_file_ops */
  struct file\_lock\_context	*i_flctx;
  struct address\_space	i\_data;
  struct list\_head	i\_devices;
  union {
    struct pipe\_inode\_info	*i_pipe;
    struct block\_device	*i\_bdev;
    struct cdev		*i_cdev;
    char			*i_link;
  };
  \_\_u32			i\_generation;  //索引节点版本号
#ifdef CONFIG_FSNOTIFY
  \_\_u32			i\_fsnotify_mask; /* all events this inode cares about */
  struct hlist\_head	i\_fsnotify_marks;
#endif

  void			\*i_private; /\* fs or device private pointer */ 
};
struct inode_operations {
	struct dentry * (\*lookup) (struct inode \*,struct dentry *, unsigned int);
	const char * (\*follow_link) (struct dentry \*, void **);
	int (\*permission) (struct inode \*, int);
	struct posix\_acl * (\*get\_acl)(struct inode \*, int);

	int (\*readlink) (struct dentry \*, char __user *,int);
	void (\*put_link) (struct inode \*, void *);

	int (\*create) (struct inode \*,struct dentry *, umode_t, bool);
	int (\*link) (struct dentry \*,struct inode *,struct dentry *);
	int (\*unlink) (struct inode \*,struct dentry *);
	int (\*symlink) (struct inode \*,struct dentry *,const char *);
	int (\*mkdir) (struct inode \*,struct dentry *,umode_t);
	int (\*rmdir) (struct inode \*,struct dentry *);
	int (\*mknod) (struct inode \*,struct dentry *,umode\_t,dev\_t);
	int (\*rename) (struct inode \*, struct dentry *,
			struct inode *, struct dentry *);
	int (\*rename2) (struct inode \*, struct dentry *,
			struct inode *, struct dentry *, unsigned int);
	int (\*setattr) (struct dentry \*, struct iattr *);
	int (\*getattr) (struct vfsmount \*mnt, struct dentry *, struct kstat *);
	int (\*setxattr) (struct dentry \*, const char *,const void *,size_t,int);
	ssize\_t (\*getxattr) (struct dentry \*, const char *, void *, size\_t);
	ssize\_t (\*listxattr) (struct dentry \*, char *, size\_t);
	int (\*removexattr) (struct dentry \*, const char *);
	int (\*fiemap)(struct inode \*, struct fiemap\_extent\_info *, u64 start,
		      u64 len);
	int (\*update_time)(struct inode \*, struct timespec *, int);
	int (\*atomic_open)(struct inode \*, struct dentry *,
			   struct file *, unsigned open_flag,
			   umode\_t create\_mode, int *opened);
	int (\*tmpfile) (struct inode \*, struct dentry *, umode_t);
	int (\*set\_acl)(struct inode \*, struct posix\_acl *, int);
} \_\_\_\_cacheline\_aligned;

file
----

// include/linux/fs.h

struct file {
  union {
    struct llist\_node	fu\_llist;
    struct rcu\_head 	fu\_rcuhead;
  } f_u;
  struct path		f_path;
  struct inode		\*f_inode;	/\* cached value */
  const struct file\_operations	*f\_op;

  spinlock\_t		f\_lock;
  atomic\_long\_t		f_count;
  unsigned int 		f_flags;
  fmode\_t			f\_mode;
  struct mutex		f\_pos\_lock;
  loff\_t			f\_pos;
  struct fown\_struct	f\_owner;
  const struct cred	*f_cred;
  struct file\_ra\_state	f_ra;

  u64			f_version;
#ifdef CONFIG_SECURITY
  void			*f_security;
#endif
  /\* needed for tty driver, and maybe others */
  void			*private_data;

#ifdef CONFIG_EPOLL
  /\* Used by fs/eventpoll.c to link all the hooks to this file */
  struct list\_head	f\_ep_links;
  struct list\_head	f\_tfile_llink;
#endif /* #ifdef CONFIG_EPOLL */
  struct address\_space	*f\_mapping;
} \_\_attribute\_\_((aligned(4)));	/* lest something weird decides that 2 is OK */
struct path {
	struct vfsmount *mnt;
	struct dentry *dentry;
};

struct file_handle {
  \_\_u32 handle\_bytes;
  int handle_type;
  /\* file identifier */
  unsigned char f_handle\[0\];
};
struct file_operations {
	struct module *owner;
	loff\_t (\*llseek) (struct file \*, loff\_t, int);
	ssize\_t (\*read) (struct file \*, char \_\_user *, size\_t, loff\_t *);
	ssize\_t (\*write) (struct file \*, const char \_\_user *, size\_t, loff\_t *);
	ssize\_t (\*read\_iter) (struct kiocb \*, struct iov_iter *);
	ssize\_t (\*write\_iter) (struct kiocb \*, struct iov_iter *);
	int (\*iterate) (struct file \*, struct dir_context *);
	unsigned int (\*poll) (struct file \*, struct poll\_table\_struct *);
	long (\*unlocked_ioctl) (struct file \*, unsigned int, unsigned long);
	long (\*compat_ioctl) (struct file \*, unsigned int, unsigned long);
	int (\*mmap) (struct file \*, struct vm\_area\_struct *);
	int (\*open) (struct inode \*, struct file *);
	int (\*flush) (struct file \*, fl\_owner\_t id);
	int (\*release) (struct inode \*, struct file *);
	int (\*fsync) (struct file \*, loff\_t, loff\_t, int datasync);
	int (\*aio_fsync) (struct kiocb \*, int datasync);
	int (\*fasync) (int, struct file \*, int);
	int (\*lock) (struct file \*, int, struct file_lock *);
	ssize\_t (\*sendpage) (struct file \*, struct page *, int, size\_t, loff_t *, int);
	unsigned long (\*get\_unmapped\_area)(struct file \*, unsigned long, unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (\*flock) (struct file \*, int, struct file_lock *);
	ssize\_t (\*splice\_write)(struct pipe\_inode\_info \*, struct file *, loff\_t *, size\_t, unsigned int);
	ssize\_t (\*splice\_read)(struct file \*, loff\_t *, struct pipe\_inode\_info *, size\_t, unsigned int);
	int (\*setlease)(struct file \*, long, struct file_lock **, void **);
	long (\*fallocate)(struct file \*file, int mode, loff_t offset,
			  loff_t len);
	void (\*show\_fdinfo)(struct seq\_file \*m, struct file *f);
#ifndef CONFIG_MMU
	unsigned (\*mmap_capabilities)(struct file \*);
#endif
};

一张图说明这几个VFS重要结构体的指向关系 ![Linux虚拟文件系统（1）](http://pic.l2h.site/l2hsitevfs_struct_relation2.png "Linux虚拟文件系统（1）")