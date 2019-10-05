---
title: Linux虚拟文件系统（3）-- VFS系统调用之Mount
tags:
  - Linux
  - VFS
  - 文件系统
url: 561.html
id: 561
categories:
  - Linux文件系统
date: 2018-10-08 09:15:35
---

前言
--

前2章分别介绍了VFS的基本数据结构和初始化流程。本章介绍VFS文件系统的使用。文件系统使用大多是从应用层系统调用开始的，下表是对文件系统系统调用的一个整理。

![Linux虚拟文件系统（3）-- VFS系统调用](http://pic.l2h.site/l2hsitevfs-syscalls.png "Linux虚拟文件系统（3）-- VFS系统调用")
===================================================================================================

MOUNT
=====

文件系统可以使用的前提是系统挂载(Mount)成功。初始化过程中，Linux会预先挂载一些支撑系统运行的特殊文件系统，例如，sysfs、rootfs、tmpfs等。本节先介绍初始化过程中Kernel对rootfs和sysfs文件系统的直接挂载，接着以一个典型的文件系统类型（待补充）介绍在用户空间态做挂载后内核的执行流程。

初始化过程中文件系统挂载
------------

### rootfs 挂载

前一篇文章中我们看到系统初始化过程中挂载了rootfs，即根文件系统(/)。这里我们看一下rootfs挂载的细节。

static void \_\_init init\_mount_tree(void)
{
//......
  mnt = vfs\_kern\_mount(type, 0, "rootfs", NULL); //挂载rootfs，其中会执行对应rootfs的mount callback
  put_filesystem(type);
  if (IS_ERR(mnt))
    panic("Can't create rootfs");

  ns = create\_mnt\_ns(mnt);     //创建mnt命名空间，并将rootfs加入到对应命名空间，作为命名空间的root
//......
}

struct vfsmount *
vfs\_kern\_mount(struct file\_system\_type \*type, int flags, const char \*name, void *data)
{
//.......
  mnt = alloc_vfsmnt(name);                         //分配VFS Mount
  if (!mnt)
    return ERR_PTR(-ENOMEM);

  if (flags & MS\_KERNMOUNT)                         //如果是kern\_mount，做内部Mount标记
    mnt->mnt.mnt\_flags = MNT\_INTERNAL;

  root = mount_fs(type, flags, name, data);        //真正执行fs的挂载
  if (IS_ERR(root)) {
    mnt\_free\_id(mnt);free\_vfsmnt(mnt); return ERR\_CAST(root); //如果Mount失败，就作清理
  }
  mnt->mnt.mnt\_root = root;mnt->mnt.mnt\_sb = root->d\_sb;mnt->mnt\_mountpoint = mnt->mnt.mnt\_root;mnt->mnt\_parent = mnt;  //挂载成功后，对vfsmount数据结构进行必要的赋值
  
  lock\_mount\_hash(); list\_add\_tail(&mnt->mnt\_instance, &root->d\_sb->s\_mounts);   unlock\_mount_hash(); //将新分配的VFS mount加入到root的mount列表
  return &mnt->mnt;
}

struct dentry *
mount\_fs(struct file\_system_type \*type, int flags, const char \*name, void *data)
{
  struct dentry \*root;struct super_block \*sb; 
        char *secdata = NULL;int error = -ENOMEM;
//...
  root = type->mount(type, flags, name, data); //执行mount函数进行mount，这里执行的是rootfs_mount
  if (IS\_ERR(root)) {error = PTR\_ERR(root); goto out\_free\_secdata; }
  sb = root->d\_sb;  BUG\_ON(!sb);  WARN\_ON(!sb->s\_bdi);
  sb->s\_flags |= MS\_BORN;
  
   error = security\_sb\_kern_mount(sb, flags, secdata);
  if (error) goto out_sb;

//....
}

static struct dentry \*rootfs\_mount(struct file\_system\_type \*fs\_type, int flags, const char \*dev_name, void \*data)
{
	static unsigned long once;
	void *fill = ramfs\_fill\_super;
//....
	if (IS\_ENABLED(CONFIG\_TMPFS) && is\_tmpfs) //当CONFIG\_TMPFS打开，使用shmem相关函数进行挂载和超级块填充，否则使用ramfs相关函数进行挂载和超级块填充
		fill = shmem\_fill\_super;
	return mount\_nodev(fs\_type, flags, data, fill);
}

struct dentry \*mount\_nodev(struct file\_system\_type \*fs\_type,
	int flags, void *data,
	int (\*fill\_super)(struct super\_block \*, void *, int))
{
	int error;
       //根据filesystem 类别获取超级块
	struct super\_block *s = sget(fs\_type, NULL, set\_anon\_super, flags, NULL);

	if (IS_ERR(s))
		return ERR_CAST(s);

	error = fill\_super(s, data, flags & MS\_SILENT ? 1 : 0); //执行fill\_super函数，即ramfs\_fill\_super或者shmem\_fill_super
	if (error) {
		deactivate\_locked\_super(s);
		return ERR_PTR(error);
	}
	s->s\_flags |= MS\_ACTIVE;
	return dget(s->s_root);
}

这里以shmem\_fill\_super为例，会为rootfs分配shmem 类型超级块的私有结构并进行填充，同时分配Inode以及Dentry，用来构建所有文件系统树的根目录(即'/')

int shmem\_fill\_super(struct super_block \*sb, void \*data, int silent)
{
 
  sbinfo = kzalloc(max((int)sizeof(struct shmem\_sb\_info),L1\_CACHE\_BYTES), GFP_KERNEL); 
  sbinfo->mode = S\_IRWXUGO | S\_ISVTX;
  sbinfo->uid = current_fsuid();
  sbinfo->gid = current_fsgid();
  sb->s\_fs\_info = sbinfo;
  //以上分配shmem 超级块的私有信息结构并做必要初始化
  if (!(sb->s\_flags & MS\_KERNMOUNT)) {
    

  spin\_lock\_init(&sbinfo->stat_lock);
  if (percpu\_counter\_init(&sbinfo->used\_blocks, 0, GFP\_KERNEL))
    goto failed;
  sbinfo->free\_inodes = sbinfo->max\_inodes;
  sb->s\_maxbytes = MAX\_LFS_FILESIZE;
  sb->s\_blocksize = PAGE\_CACHE_SIZE;
  sb->s\_blocksize\_bits = PAGE\_CACHE\_SHIFT;
  sb->s\_magic = TMPFS\_MAGIC;
  sb->s\_op = &shmem\_ops;
  sb->s\_time\_gran = 1;
  //超级块初始化

  //从shmem初始化的cache中获取inode（作为/对应的inode）
  inode = shmem\_get\_inode(sb, NULL, S\_IFDIR | sbinfo->mode, 0, VM\_NORESERVE); 
  inode->i_uid = sbinfo->uid;
  inode->i_gid = sbinfo->gid;
  //设定超级块的root dentry（作为/对应的dentry）
  sb->s\_root = d\_make_root(inode);
  return 0;
}

shmem\_fill\_super可以直接分配shmem\_inode cache，是因为初始化流程init\_rootfs()中，已经执行了如下shmem的初始化（参考[Linux虚拟文件系统（2）– 初始化流程](http://l2h.site/linux-vfs-2/) ）。

shmem_init()
    |
    +-->shmem\_init\_inodecache()
    |
    +-->register\_filesystem(&shmem\_fs_type)
    |
    +-->kern\_mount(&shmem\_fs_type)
         |
         +----->vfs\_kern\_mount()
                  |
                  +--->alloc_vfsmnt(name)
                  |
                  +--->mount_fs(type, flags, name, data);
                          |
                          +---->type->mount(type, flags, name, data)
                                    |
                                    v
                               shmem_mount()

### sysfs挂载

static struct file\_system\_type sysfs\_fs\_type = {
  .name		= "sysfs",
  .mount		= sysfs_mount,
  .kill\_sb	= sysfs\_kill_sb,
  .fs\_flags	= FS\_USERNS\_VISIBLE | FS\_USERNS_MOUNT,
};

int \_\_init sysfs\_init(void)
{
  int err;
  sysfs\_root = kernfs\_create\_root(NULL, KERNFS\_ROOT\_EXTRA\_OPEN\_PERM\_CHECK,NULL); //创建sysfs的根目录
  sysfs\_root\_kn = sysfs_root->kn;
  err = register\_filesystem(&sysfs\_fs_type);//向系统注册sysfs
}

以上过程在rootfs\_init初始化过程中执行，主要为向系统中注册sysfs sysfs\_mount主要做以下事情：

1.  > 分配kernfs\_kern\_info
    
2.  > 为sysfs分配superblock
    
3.  > 调用kernfs\_fill\_super来填充超级块
    

用户空间态一般文件系统挂载
-------------

在用户空间态执行系统挂载的命令如下：

> mount \[-t vfstype\] \[-o options\] device dir

其中-t参数表示您要挂载的文件系统类型，-o为可选参数，device代表您要挂载的设备（可以是一个设备文件，也可以一个二级制文件系统包）， dir表示文件系统要挂载到的目录。 假设用户（注意必须是root组用户才有权限执行）在根目录/执行如下挂载命令

> mount -t yaffs rawfs.bin ./test

系统会执行系统调用，从用户空间态切换到内核态，并执行mount命令对应的系统调用sys\_mount。从sys\_mount开始，调用执行如下：

SYSCALL\_DEFINE5(mount, char \_\_user *, dev\_name, char \_\_user *, dir\_name, char \_\_user *, type, unsigned long, flags, void __user *, data)
-->kernel\_type =copy\_mount_string(type); //从user space拷贝mount的vfs 类型
-->kernel\_dev =copy\_mount\_string(dev\_name); //从user space拷贝设备名
-->copy\_mount\_options(data, &data_page); //从user space拷贝mount参数
-->do\_mount(kernel\_dev, dir\_name, kernel\_type, flags,(void *) data_page); //执行mount
   --\> //查找挂载点，这里会涉及到VFS操作时最常见的path_walk，之后章节会重点介绍
   --\> //执行必要的挂载flag设定
   --\> //检查挂载类型，根据挂载类型决定执行do\_remount(),do\_new\_mount()，do\_move_mount()或者其他

因为上边的挂载命令是执行新的挂载，因此，这里执行到的是do\_new\_mount

do\_new\_mount
-->type = get\_fs\_type(fstype); //根据传入的文件系统名，从已经加载到系统的文件系统链表中查找到对应的文件系统struct file\_system\_type，如果对应的文件系统模块没有加载到系统，此处会尝试加载。
-->mnt = vfs\_kern\_mount(type, flags, name, data); //执行vfs\_kern\_mount,参见上一节“rootfs挂载”的介绍。

vfs\_kern\_mount的流程与rootfs挂载中执行类似。此处差异只在于yaffs的type->mount(type, flags, name, data)执行的是yaffs的do_mount函数，**之后会开专门的章节介绍yaffs**。