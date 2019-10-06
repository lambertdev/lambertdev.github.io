---
title: Linux虚拟文件系统（3）-- VFS系统调用之Mount
tags:
  - Linux
  - VFS
  - 文件系统
url: 561.html
id: 561
categories:
  - Linux
  - Linux文件系统
date: 2018-10-08 09:15:35
---

前言
--

前2章分别介绍了VFS的基本数据结构和初始化流程。本章介绍VFS文件系统的使用。文件系统使用大多是从应用层系统调用开始的，下表是对文件系统系统调用的一个整理。

![Linux虚拟文件系统（3）-- VFS系统调用](http://pic.www.l2h.site/l2hsitevfs-syscalls.png "Linux虚拟文件系统（3）-- VFS系统调用")
===================================================================================================

MOUNT
=====

文件系统可以使用的前提是系统挂载(Mount)成功。初始化过程中，Linux会预先挂载一些支撑系统运行的特殊文件系统，例如，sysfs、rootfs、tmpfs等。本节先介绍初始化过程中Kernel对rootfs和sysfs文件系统的直接挂载，接着以一个典型的文件系统类型（待补充）介绍在用户空间态做挂载后内核的执行流程。

初始化过程中文件系统挂载
------------

### rootfs 挂载

前一篇文章中我们看到系统初始化过程中挂载了rootfs，即根文件系统(/)。这里我们看一下rootfs挂载的细节。
```C
static void __init init_mount_tree(void)
{
//......
  mnt = vfs_kern_mount(type, 0, "rootfs", NULL); //挂载rootfs，其中会执行对应rootfs的mount callback
  put_filesystem(type);
  if (IS_ERR(mnt))
    panic("Can't create rootfs");

  ns = create_mnt_ns(mnt);     //创建mnt命名空间，并将rootfs加入到对应命名空间，作为命名空间的root
//......
}

struct vfsmount *
vfs_kern_mount(struct file_system_type *type, int flags, const char *name, void *data)
{
//.......
  mnt = alloc_vfsmnt(name);                         //分配VFS Mount
  if (!mnt)
    return ERR_PTR(-ENOMEM);

  if (flags & MS_KERNMOUNT)                         //如果是kern_mount，做内部Mount标记
    mnt->mnt.mnt_flags = MNT_INTERNAL;

  root = mount_fs(type, flags, name, data);        //真正执行fs的挂载
  if (IS_ERR(root)) {
    mnt_free_id(mnt);free_vfsmnt(mnt); return ERR_CAST(root); //如果Mount失败，就作清理
  }
  mnt->mnt.mnt_root = root;mnt->mnt.mnt_sb = root->d_sb;mnt->mnt_mountpoint = mnt->mnt.mnt_root;mnt->mnt_parent = mnt;  //挂载成功后，对vfsmount数据结构进行必要的赋值
  
  lock_mount_hash(); list_add_tail(&mnt->mnt_instance, &root->d_sb->s_mounts);   unlock_mount_hash(); //将新分配的VFS mount加入到root的mount列表
  return &mnt->mnt;
}

struct dentry *
mount_fs(struct file_system_type *type, int flags, const char *name, void *data)
{
  struct dentry *root;struct super_block *sb; 
        char *secdata = NULL;int error = -ENOMEM;
//...
  root = type->mount(type, flags, name, data); //执行mount函数进行mount，这里执行的是rootfs_mount
  if (IS_ERR(root)) {error = PTR_ERR(root); goto out_free_secdata; }
  sb = root->d_sb;  BUG_ON(!sb);  WARN_ON(!sb->s_bdi);
  sb->s_flags |= MS_BORN;
  
   error = security_sb_kern_mount(sb, flags, secdata);
  if (error) goto out_sb;

//....
}

static struct dentry *rootfs_mount(struct file_system_type *fs_type, int flags, const char *dev_name, void *data)
{
	static unsigned long once;
	void *fill = ramfs_fill_super;
//....
	if (IS_ENABLED(CONFIG_TMPFS) && is_tmpfs) //当CONFIG_TMPFS打开，使用shmem相关函数进行挂载和超级块填充，否则使用ramfs相关函数进行挂载和超级块填充
		fill = shmem_fill_super;
	return mount_nodev(fs_type, flags, data, fill);
}

struct dentry *mount_nodev(struct file_system_type *fs_type,
	int flags, void *data,
	int (*fill_super)(struct super_block *, void *, int))
{
	int error;
       //根据filesystem 类别获取超级块
	struct super_block *s = sget(fs_type, NULL, set_anon_super, flags, NULL);

	if (IS_ERR(s))
		return ERR_CAST(s);

	error = fill_super(s, data, flags & MS_SILENT ? 1 : 0); //执行fill_super函数，即ramfs_fill_super或者shmem_fill_super
	if (error) {
		deactivate_locked_super(s);
		return ERR_PTR(error);
	}
	s->s_flags |= MS_ACTIVE;
	return dget(s->s_root);
}
```
这里以shmem_fill_super为例，会为rootfs分配shmem 类型超级块的私有结构并进行填充，同时分配Inode以及Dentry，用来构建所有文件系统树的根目录(即'/')
```C
int shmem_fill_super(struct super_block *sb, void *data, int silent)
{
 
  sbinfo = kzalloc(max((int)sizeof(struct shmem_sb_info),L1_CACHE_BYTES), GFP_KERNEL); 
  sbinfo->mode = S_IRWXUGO | S_ISVTX;
  sbinfo->uid = current_fsuid();
  sbinfo->gid = current_fsgid();
  sb->s_fs_info = sbinfo;
  //以上分配shmem 超级块的私有信息结构并做必要初始化
  if (!(sb->s_flags & MS_KERNMOUNT)) {
    

  spin_lock_init(&sbinfo->stat_lock);
  if (percpu_counter_init(&sbinfo->used_blocks, 0, GFP_KERNEL))
    goto failed;
  sbinfo->free_inodes = sbinfo->max_inodes;
  sb->s_maxbytes = MAX_LFS_FILESIZE;
  sb->s_blocksize = PAGE_CACHE_SIZE;
  sb->s_blocksize_bits = PAGE_CACHE_SHIFT;
  sb->s_magic = TMPFS_MAGIC;
  sb->s_op = &shmem_ops;
  sb->s_time_gran = 1;
  //超级块初始化

  //从shmem初始化的cache中获取inode（作为/对应的inode）
  inode = shmem_get_inode(sb, NULL, S_IFDIR | sbinfo->mode, 0, VM_NORESERVE); 
  inode->i_uid = sbinfo->uid;
  inode->i_gid = sbinfo->gid;
  //设定超级块的root dentry（作为/对应的dentry）
  sb->s_root = d_make_root(inode);
  return 0;
}
```
shmem_fill_super可以直接分配shmem_inode cache，是因为初始化流程init_rootfs()中，已经执行了如下shmem的初始化（参考[Linux虚拟文件系统（2）– 初始化流程](http://www.l2h.site/linux-vfs-2/) ）。
```
shmem_init()
    |
    +-->shmem_init_inodecache()
    |
    +-->register_filesystem(&shmem_fs_type)
    |
    +-->kern_mount(&shmem_fs_type)
         |
         +----->vfs_kern_mount()
                  |
                  +--->alloc_vfsmnt(name)
                  |
                  +--->mount_fs(type, flags, name, data);
                          |
                          +---->type->mount(type, flags, name, data)
                                    |
                                    v
                               shmem_mount()
```
### sysfs挂载
```C
static struct file_system_type sysfs_fs_type = {
  .name		= "sysfs",
  .mount		= sysfs_mount,
  .kill_sb	= sysfs_kill_sb,
  .fs_flags	= FS_USERNS_VISIBLE | FS_USERNS_MOUNT,
};

int __init sysfs_init(void)
{
  int err;
  sysfs_root = kernfs_create_root(NULL, KERNFS_ROOT_EXTRA_OPEN_PERM_CHECK,NULL); //创建sysfs的根目录
  sysfs_root_kn = sysfs_root->kn;
  err = register_filesystem(&sysfs_fs_type);//向系统注册sysfs
}
```
以上过程在rootfs_init初始化过程中执行，主要为向系统中注册sysfs sysfs_mount主要做以下事情：

1.  > 分配kernfs_kern_info
    
2.  > 为sysfs分配superblock
    
3.  > 调用kernfs_fill_super来填充超级块
    

用户空间态一般文件系统挂载
-------------

在用户空间态执行系统挂载的命令如下：

> mount [-t vfstype] [-o options] device dir

其中-t参数表示您要挂载的文件系统类型，-o为可选参数，device代表您要挂载的设备（可以是一个设备文件，也可以一个二级制文件系统包）， dir表示文件系统要挂载到的目录。 假设用户（注意必须是root组用户才有权限执行）在根目录/执行如下挂载命令

> mount -t yaffs rawfs.bin ./test

系统会执行系统调用，从用户空间态切换到内核态，并执行mount命令对应的系统调用sys_mount。从sys_mount开始，调用执行如下：
```C
SYSCALL_DEFINE5(mount, char __user *, dev_name, char __user *, dir_name, char __user *, type, unsigned long, flags, void __user *, data)
-->kernel_type =copy_mount_string(type); //从user space拷贝mount的vfs 类型
-->kernel_dev =copy_mount_string(dev_name); //从user space拷贝设备名
-->copy_mount_options(data, &data_page); //从user space拷贝mount参数
-->do_mount(kernel_dev, dir_name, kernel_type, flags,(void *) data_page); //执行mount
   --> //查找挂载点，这里会涉及到VFS操作时最常见的path_walk，之后章节会重点介绍
   --> //执行必要的挂载flag设定
   --> //检查挂载类型，根据挂载类型决定执行do_remount(),do_new_mount()，do_move_mount()或者其他
```
因为上边的挂载命令是执行新的挂载，因此，这里执行到的是do_new_mount
```C
do_new_mount
-->type = get_fs_type(fstype); //根据传入的文件系统名，从已经加载到系统的文件系统链表中查找到对应的文件系统struct file_system_type，如果对应的文件系统模块没有加载到系统，此处会尝试加载。
-->mnt = vfs_kern_mount(type, flags, name, data); //执行vfs_kern_mount,参见上一节“rootfs挂载”的介绍。
```
vfs_kern_mount的流程与rootfs挂载中执行类似。此处差异只在于yaffs的type->mount(type, flags, name, data)执行的是yaffs的do_mount函数，**之后会开专门的章节介绍yaffs**。