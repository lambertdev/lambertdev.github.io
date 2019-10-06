---
title: Linux虚拟文件系统（2）-- 初始化流程
tags:
  - Linux
  - VFS
  - 文件系统
url: 527.html
id: 527
categories:
  - Linux
  - Linux文件系统
date: 2018-08-10 20:36:02
---

本文介绍Linux虚拟文件系统的初始化过程。如下图:

+-----> start_kernel()
|
|
+-----> vfs_caches_init_early()
|
|
+-----> vfs_caches_init()
|                        +--->new thread(kernel_init)
|                        |       +
|                        |       |
+-----> rest_init()+------       v
                           kernel_init_freeable
                                 + 
                                 | 
                                 v
                             do_basic_setup
                                 +
                                 |
                                 v
                           prepare_namespace

在Linux kernel的入口函数start_kernel中，系统调用vfs_caches_init_early和vfs_caches_init进行必要的初始化。

vfs_caches_init_early
=======================

Dentry和inode cache是linux为了加速对虚拟文件系统管理，在内存中分配的的hash Cache，用来存放最近访问的inode或dentry节点。 一般而言，最近访问的节点也是最容易被再次访问的。当要访问inode或dentry节点时，先到内存中的cache查找（Hash链表）。如果可以找到，便可以直接使用内存中的节点，否则才到文件系统中去找。
```C
static void __init dcache_init_early(void)
{
  unsigned int loop;

  /* If hashes are distributed across NUMA nodes, defer
   * hash allocation until vmalloc space is available.
   */
  if (hashdist)
    return;

  dentry_hashtable =
    alloc_large_system_hash("Dentry cache",
          sizeof(struct hlist_bl_head),
          dhash_entries,
          13,
          HASH_EARLY,
          &d_hash_shift,
          &d_hash_mask,
          0,
          0);

  for (loop = 0; loop < (1U << d_hash_shift); loop++)
    INIT_HLIST_BL_HEAD(dentry_hashtable + loop);
}
void __init inode_init_early(void)
{
	unsigned int loop;
	if (hashdist)
		return;
	inode_hashtable =
		alloc_large_system_hash("Inode-cache",
					sizeof(struct hlist_head),
					ihash_entries,
					14,
					HASH_EARLY,
					&i_hash_shift,
					&i_hash_mask,
					0,
					0);

	for (loop = 0; loop < (1U << i_hash_shift); loop++)
		INIT_HLIST_HEAD(&inode_hashtable[loop]);
}
```C
上述代码可以看出，vfs_caches_init_early创建了inode和dentry cache的hash链表，但是这里并没有分配dentry和inode的cache（vfs_caches_init中才会创建）。Hash 链表的结构如下图所示。

![Linux虚拟文件系统（2）-- 初始化流程](http://pic.l2h.site/l2hsitehlist.png "Linux虚拟文件系统（2）-- 初始化流程")vfs_caches_init
=========================================================================================================

相较于vfs_caches_init_early, vfs_caches_init则会做更多的事情。如下代码
```C
void __init vfs_caches_init(void)
{
  names_cachep = kmem_cache_create("names_cache", PATH_MAX, 0,
      SLAB_HWCACHE_ALIGN|SLAB_PANIC, NULL);

  dcache_init();   //初始化dentry cache，此时才会创建cache
  inode_init();    //类上
  files_init();    //初始化文件cache
  files_maxfiles_init(); //初始化系统最大文件数
  mnt_init();      // 初始化文件系统挂载相关资源并挂载特殊文件系统
  bdev_cache_init(); //初始化块设备管理伪文件系统
  chrdev_init();   //初始化字符设备相关
}
```C
_dcache_init/inode_init_: 初始化dentry和inode cache，如果没有创建对应的哈希链表，就创建之（前边vfs_caches_init_early已经创建） _files_init_: 根据当前系统内存计算最大文件数

n = ((totalram_pages - memreserve) * (PAGE_SIZE / 1024)) / 10;
files_stat.max_files = max_t(unsigned long, n, NR_FILE);

_mnt_init_：与文件系统mount相关的初始化。细节流程如下：
```
+----------------------+
| Create MNT cache and |
| MNT cache Hash table |
+-----------+----------+
            |
+-----------v----------+    +---------------------+
| Create Mount Point   |    | init mount tree     |
| Hash Table           |    |                     |
+-----------+----------+    +----------^----------+
            |                          |
+-----------v----------+    +----------+----------+
| kernfs_init()        |    | init_rootfs()       |
|                      |    |                     |
+-----------+----------+    +----------^----------+
            |                          |
+-----------v----------+    +----------+----------+
| sysfs_init()         +--> | Register FS folder  |
|                      |    | to sysfs            |
+----------------------+    +---------------------+
```
首先，创建struct mount cache及哈希链表，之后创建文件系统加载点（struct mountpoint）Cache Hash。 其中前者为系统中已经Mount的VFS相关信息（包含不同VFS Mount的相互关系），后者主要存放的是系统加载点Dentry。 _kernfs_init_：主要创建kernfs_node的cache。kernfs_node主要为创建kernfs结构的结构体。kernfs的作用引用wiki如下：

> In the Linux kernel, **kernfs** is a set of functions that contain the functionality required for creating pseudo file systems used internally by various kernel subsystems. The creation of kernfs resulted from splitting off part of the internal logic used by sysfs, which provides a set of virtual files by exporting information about hardware devices and associated device drivers from the kernel's device model to user space, into an independent and reusable functionality so other kernel subsystems can implement their own pseudo file systems more easily and consistently

_sysfs_init_：初始化sysfs，并向系统中注册该文件系统。sysfs.txt(Documentation/zh_CN/filesystems/sysfs.txt)对sysfs的简介如下：

> sysfs 是一个最初基于 ramfs 且位于内存的文件系统。它提供导出内核 数据结构及其属性，以及它们之间的关联到用户空间的方法。sysfs 目录的安排显示了内核数据结构之间的关系。顶层 sysfs 目录如下: block/ bus/ class/ dev/ devices/ firmware/ net/ fs/

紧接着注册fs目录到sysfs(fs_kobj = kobject_create_and_add("fs", NULL);)，即构成sysfs顶级目录的fs目录。 _init_rootfs()_: 主要向系统中注册rootfs与tmpfs文件系统，以及执行Mount
```C
static struct file_system_type rootfs_fs_type = {
  .name		= "rootfs",
  .mount		= rootfs_mount,
  .kill_sb	= kill_litter_super,
};
int __init init_rootfs(void)
{
  int err = register_filesystem(&rootfs_fs_type);//注册rootfs到文件系统。注意如果kernel启动的cmd line如果传入了 "root=xxx"，此处便执行不到
  if (err) return err;
  if (IS_ENABLED(CONFIG_TMPFS) && !saved_root_name[0] &&  //如果kernel config打开CONFIG_TMPFS
    (!root_fs_names || strstr(root_fs_names, "tmpfs"))) { // 如果cmd line没有传入rootfs的类型，便执行shmem 的初始化
    err = shmem_init();      //执行shmem_init
    is_tmpfs = true;
  } else {
    err = init_ramfs_fs();  //注册ramfs
  }
  if (err) unregister_filesystem(&rootfs_fs_type);
  return err;
}
static struct file_system_type shmem_fs_type = {
  .owner		= THIS_MODULE,
  .name		= "tmpfs",
  .mount		= shmem_mount,  
  .kill_sb	= kill_litter_super,
  .fs_flags	= FS_USERNS_MOUNT,
};
int __init shmem_init(void)
{
  int error;
  /* If rootfs called this, don't re-init */
  if (shmem_inode_cachep) return 0;
  error = shmem_init_inodecache();  //初始化shmem的inode cache
  if (error) goto out3;

  error = register_filesystem(&shmem_fs_type); //注册tmpfs文件系统
  if (error) {
    printk(KERN_ERR "Could not register tmpfs\\n");
    goto out2;
  }
  shm_mnt = kern_mount(&shmem_fs_type);  //执行shmem的mount调用
  if (IS_ERR(shm_mnt)) {
    error = PTR_ERR(shm_mnt);
    printk(KERN_ERR "Could not kern_mount tmpfs\\n");
    goto out1;
  }
  return 0;
out1:
  unregister_filesystem(&shmem_fs_type);
out2:
  shmem_destroy_inodecache();
out3:
  shm_mnt = ERR_PTR(error);
  return error;
}
```
之后的文章会介绍shmem的mount，shmem_mount _init_mount_tree()_: 如其名，初始化文件系统挂载树。代码如下：
```C
static void __init init_mount_tree(void)
{
  struct vfsmount *mnt;
  struct mnt_namespace *ns;
  struct path root;
  struct file_system_type *type;

  type = get_fs_type("rootfs"); //获取已注册到kernel中的rootfs
  if (!type)
    panic("Can't find rootfs type"); //前方init_rootfs已经向系统注册rootfs，若此处找不到，视为异常
  mnt = vfs_kern_mount(type, 0, "rootfs", NULL); //挂载rootfs，其中会执行对应rootfs的mount callback，具体下一篇文章介绍
  put_filesystem(type);
  if (IS_ERR(mnt))
    panic("Can't create rootfs");

  ns = create_mnt_ns(mnt);     //创建mnt命名空间，并将rootfs加入到对应命名空间，作为命名空间的root
  if (IS_ERR(ns))
    panic("Can't allocate initial namespace");

  init_task.nsproxy->mnt_ns = ns;
  get_mnt_ns(ns);
  root.mnt = mnt;
  root.dentry = mnt->mnt_root;
  mnt->mnt_flags |= MNT_LOCKED;
  set_fs_pwd(current->fs, &root);   //设定init进程的当前目录为挂载点的主目录
  set_fs_root(current->fs, &root);  //设定init进程的root为挂载点的主目录
}
```
至此，vfs_caches_init初始化流程结束

do_basic_setup
================

在rest_init函数中，kernel创建kernel_init线程，该线程会执行do_basic_setup。其执行do_initcalls-->rootfs_initcall(populate_rootfs);-->rootfs_initcall( default_rootfs);    populate_rootfs主要是将打包在initramfs中的cpio档填充到rootfs，而default_rootfs主要创建/dev，/dev/console和/root

prepare_namespace
=================

当/sbin/init 或者 init不存在时，便执行prepare_namespace。其主要作用也能够便是挂载MTD device或UBI device到root。这里引用[IBM developworkers](https://www.ibm.com/developerworks/cn/linux/l-initrd.html)的说明对prepare_namespace作介绍如下：

> 在内核和 initrd 映像被解压并拷贝到内存中之后，内核就会被调用了。它会执行不同的初始化操作，最终您会发现自己到了 `init/main.c:init()`（subdir/file:function）函数中。这个函数执行了大量的子系统初始化操作。此处会执行一个对 `init/do_mounts.c:prepare_namespace()` 的调用，这个函数用来准备名称空间（挂载 dev 文件系统、RAID 或 md、设备以及最后的 initrd）。加载 initrd 是通过调用 `init/do_mounts_initrd.c:initrd_load()` 实现的。 `initrd_load()` 函数调用了 `init/do_mounts_rd.c:rd_load_image()`，它通过调用 `init/do_mounts_rd.c:identify_ramdisk_image()` 来确定要加载哪个 RAM 磁盘。这个函数会检查映像文件的 magic 号来确定它是 minux、etc2、romfs、cramfs 或 gzip 格式。在返回到 `initrd_load_image` 之前，它还会调用 `init/do_mounts_rd:crd_load()`。这个函数负责为 RAM 磁盘分配空间，并计算循环冗余校验码（CRC），然后对 RAM 磁盘映像进行解压，并将其加载到内存中。现在，我们在一个适合挂载的块设备中就有了这个 initrd 映像。 现在使用一个 `init/do_mounts.c:mount_root()` 调用将这个块设备挂载到根文件系统上。它会创建根设备，并调用 `init/do_mounts.c:mount_block_root()`。在这里调用 `init/do_mounts.c:do_mount_root()`，后者又会调用 `fs/namespace.c:sys_mount()` 来真正挂载根文件系统，然后 `chdir` 到这个文件系统中。这就是我们在清单 6 中所看到的熟悉消息 `VFS: Mounted root (ext2 file system).` 的地方。 最后，返回到 `init` 函数中，并调用 `init/main.c:run_init_process`。这会导致调用 `execve` 来启动 init 进程（在本例中是 `/linuxrc`）。linuxrc 可以是一个可执行程序，也可以是一个脚本（条件是它有脚本解释器可用）。这些函数的调用层次结构如下边清单所示。尽管此处并没有列出拷贝和挂载初始 RAM 磁盘所涉及的所有函数，但是这足以为我们提供一个整体流程的粗略框架
> 
> init/main.c:init()
>   init/do_mounts.c:prepare_namespace()
>     init/do_mounts_initrd.c:initrd_load()
>       init/do_mounts_rd.c:rd_load_image()
>         init/do_mounts_rd.c:identify_ramdisk_image()
>         init/do_mounts_rd.c:crd_load()
>           lib/inflate.c:gunzip()
>     init/do_mounts.c:mount_root()
>       init/do_mounts.c:mount_block_root()
>          init/do_mounts.c:do_mount_root()
>            fs/namespace.c:sys_mount()
>   init/main.c:run_init_process()
>     execve

引用Kernel中的README The kernel has currently 3 ways to mount the root filesystem: a) all required device and filesystem drivers compiled into the kernel, no initrd. init/main.c:init() will call prepare_namespace() to mount the final root filesystem, based on the root= option and optional init= to run some other init binary than listed at the end of init/main.c:init(). b) some device and filesystem drivers built as modules and stored in an initrd. The initrd must contain a binary '/linuxrc' which is supposed to load these driver modules. It is also possible to mount the final root filesystem via linuxrc and use the pivot_root syscall. The initrd is mounted and executed via prepare_namespace(). c) using initramfs. The call to prepare_namespace() must be skipped. This means that a binary must do all the work. Said binary can be stored into initramfs either via modifying usr/gen_init_cpio.c or via the new initrd format, an cpio archive. It must be called "/init". This binary is responsible to do all the things prepare_namespace() would do. To maintain backwards compatibility, the /init binary will only run if it comes via an initramfs cpio archive. If this is not the case, init/main.c:init() will run prepare_namespace() to mount the final root and exec one of the predefined init binaries.