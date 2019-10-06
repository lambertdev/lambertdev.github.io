---
title: Linux虚拟文件系统（4）-- 路径名查找
tags:
  - Linux
  - VFS
url: 928.html
id: 928
categories:
  - Linux
  - Linux文件系统
date: 2018-10-14 11:41:44
---

前言
==

几乎所有Linux的文件操作，例，read、write、mkdir等都会涉及到路径名查找操作。而文件查找对Linux内核来说，主要指的是找到文件路径对应的Dentry节点。其主要过程就是对路径字符串进行一级级解析（以路径名中的.. , . , /等字符作为解析依据），找到路径的最后一级目录。若传入的路径字符串是以/开始的，那么查找会从系统根目录开始。否则，从当前工作目录开始查找。 ![Linux虚拟文件系统（4）-- 路径名查找](http://pic.l2h.site/l2hsitevfs-4-find.jpeg "Linux虚拟文件系统（4）-- 路径名查找") 然而，查找过程并非仅仅是对路径名一级级解析和匹配，其中还要考虑到如下细节：

*   用户可能对要查找的路径没有访问权限
*   路径名查找是系统中频繁且常见的操作，如何保证查找的效率
*   多个进程 or 用户会同时操作到同一个路径，查找过程中需要做必要的保护
*   查找过程中可能经过符号链接，同时还要考虑避免符号链接的循环引用，造成无限查找
*   查找过程可能会跨越多个文件系统类别
*   …….

引用内核文档对VFS路径查找的介绍：

> The most obvious aspect of pathname lookup, which very little exploration is needed to discover, is that it is complex.  There are many rules, special cases, and implementation alternatives that all combine to confuse the unwary reader.

[上篇文章](http://l2h.site/linux-vfs-3/)中介绍了Mount系统调用，其中do_mount函数会进行文件系统挂载点的查找，代码如下：

long do_mount(const char *dev_name, const char __user *dir_name,
const char *type_page, unsigned long flags, void *data_page)
{
........
/* ... and get the mountpoint */
retval = user_path(dir_name, &path)
........
}

user_path()函数传入要查找的路径名，返回struct path结构体类型供后续使用。user_path()最终会调用到filename_lookup(),执行文件查找工作。本章就对filename_lookup()函数进行深入剖析（以Linux 4.4内核为基础）。

数据结构
====

与路径查找相关的数据结构是struct nameidata，其具体字段及主要作用如下：
```C
struct nameidata {
    struct path path;              //记录路径查找的结果
    struct qstr last;                //路径名最后一个分量
    struct path root;              //路径查找的根目录信息，可能会在查找开始时由调用者传入
    struct inode    *inode; /* path.dentry.d_inode */
    unsigned int    flags;       //路径名查找标志
    unsigned    seq, m_seq; //
    int     last_type;               //记录当前查找到目录的类别Normal/Dot/DotDot/Root/Bind
    unsigned    depth;          //查找过程中跨越的符号链接深度
    int     total_link_count;    //查找过程中经过的符号链接总数
    struct saved {
        struct path link;
        void *cookie;
        const char *name;
        struct inode *inode;
        unsigned seq; 
    } *stack, internal[EMBEDDED_LEVELS]; //用来记录查找过程中碰到的符号链接
    struct filename *name;    //
    struct nameidata *saved;//
    unsigned    root_seq;      //
    int     dfd;                        //
};
```
filename_lookup
===============

函数filename_lookup解析如下：
```C
static int filename_lookup(int dfd, struct filename *name, unsigned flags,
               struct path *path, struct path *root)
{
    int retval;
    struct nameidata nd;
    if (IS_ERR(name))
        return PTR_ERR(name);
    if (unlikely(root)) {
        nd.root = *root;
        flags |= LOOKUP_ROOT;
    }
    set_nameidata(&nd, dfd, name);
    retval = path_lookupat(&nd, flags | LOOKUP_RCU, path);
    if (unlikely(retval == -ECHILD))
        retval = path_lookupat(&nd, flags, path);
    if (unlikely(retval == -ESTALE))
        retval = path_lookupat(&nd, flags | LOOKUP_REVAL, path);

    if (likely(!retval))
        audit_inode(name, path->dentry, flags & LOOKUP_PARENT);
    restore_nameidata();
    putname(name);
    return retval;
}
```
Kernel路径名查找的方式有两种：

*   REF-Walk方式：指的是路径查找过程中，使用Spinlock（dentryàd_lock）并发使用或者修改目录项，来保证系统最终目录项内容的正确性。但是Spinlock因为会引发阻塞，所以效率会低于RCU-Walk.Kernel路径名查找的方式有两种：
*   RCU-Walk方式：采用RCU锁的方式进行查找，并发查找过程中并不会因为等待spinlock而阻塞，因此速度相对更快。但是它并不能保证所有情况下都能查找成功

filename_lookup函数首先初始化上文介绍的nameidata结构体，接着进行RCU方式查找指定路径。若RCU查找失败，则退回传统的查找方式（REF-Walk）。正因为Linux路径查找穿插了两种查找方式的代码，所以读起来比较困难。本文接下来试图将两种查找方式分开进行介绍。

REF-Walk
========

path_lookupat主要代码如下：
```C
static int path_lookupat(struct nameidata *nd, unsigned flags, struct path *path)
{
    const char *s = path_init(nd, flags);

    while (!(err = link_path_walk(s, nd))
        && ((err = lookup_last(nd)) > 0)) {
        s = trailing_symlink(nd);
        if (IS_ERR(s)) {
            err = PTR_ERR(s);
            break;
        }
    }
    if (!err)
        err = complete_walk(nd);

    if (!err && nd->flags & LOOKUP_DIRECTORY)
        if (!d_can_lookup(nd->path.dentry))
            err = -ENOTDIR;
    if (!err) {
        *path = nd->path;
        nd->path.mnt = NULL;
        nd->path.dentry = NULL;
    }
    terminate_walk(nd);
    return err;
}
```
首先path_init对nameidata做必要的初始赋值并返回路径字符串:

*   如果执行路径查找函数的函数传入了root参数，那么会置起LOOKUP_ROOT标志，此时会对当前用户的对应root目录访问权限进行检查，若不允许，则返回错误。
*   如果未置起LOOKUP_ROOT，则判断路径名查找从根目录，当前目录或者对应文件描述符对应的目录开始查找，并修改nameidata的path字段，最后修改nameidata的inode字段作为查找起始点的inode。

其次循环执行link_path_walk()，做真正的路径查找，直到碰到查找错误或者查找结束。接下来一节深入剖析。 之后执行complete_walk()，主要为再次确认要访问的dentry是否仍然有效（这取决于对应dentry的文件系统的d_weak_revalidate函数，一般情况下为NULL，且对REF-WALK模式来讲不会被调用到）。 最后执行terminate_walk()前，将nameidata的查找结果（即path）赋值给调用者传入的参数。而terminate_walk()则会将对路径的引用释放，同时将查找过程中跨越的符号链接引用释放掉并将nameidata的深度置为0。

link_path_walk
----------------

link_path_walk()首先跳过路径名中的斜杠/，接下来执行其核心：一个for循环。主要做如下事情：

1.  检查当前要查找目录是否有查找的权限（当前目录对应inode是否有EXEC权限），若没有则退出查找。
2.  对当前查找目录计算其Hash长度
3.  若当前查找目录为”..”，则置起来LOOKUP_JUMP标记，表示查找跳过该目录。否则开始路径查找walk_component()
4.  若第三步查找返回为0，则进行下一路径分量的查找（Walk componet过程中会修改nameidata的path以及last等字段）
5.  若第三步查找返回不为0且不为负数，则表示查找过程中碰到符号链接，修改符号链接对应节点的访问时间等信息，并将相关信息记录在nameidata的stack字段。
6.  重新开始执行for循环，直到路径遍历结束（关键变量name为NULL）

walk_component
--------------

walk_component主要做如下事情：

1.  处理点符号（即.或者..），这里主要处理的是..返回上一级目录，这里要特别处理的是上级目录可能与当前在两个不同的系统挂载点上。
2.  若当前要查找的路径是普通路径，则进行路径的快查找lookup_fast，其会执行dentry = __d_lookup(parent, &nd->last)，即从dentry高速缓存中查找。
3.  若第2步查找失败则进行慢查找lookup_slow()，其调用__lookup_hash进行查找，这会执行lookup_real到对应文件系统的i_op到磁盘上去进行查找。
4.  判断查找到的目录分量是否是符号链接，若是符号链接，则看当前遍历的符号链接数量是否超过系统中允许的最大数量（避免循环遍历），如超过则返回错误。否则，在nameidata中的stack字段为当前遍历到的符号链接分配空间并做相应赋值。注意，nameidata字段默认已经预留了储存符号链接的stack空间，只有当遍历过程经过的符号链接超过数量时，才需要重新分配。
5.  将查找的目录分量信息记录在nameidata里，供下个查找循环使用

RCU-Walk
========

待添加