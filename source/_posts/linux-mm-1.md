---
title: Linux内核内存管理简析(1)
tags:
  - Linux
  - 内存管理
url: 2580.html
id: 2580
categories:
  - Linux
  - Linux内存管理
date: 2019-08-13 08:49:07
---

内存管理是Linux内核最为复杂且最为重要的部分，本文从原理及代码角度对Linux内存管理机制进行分析。

内存的划分
-----

Linux将内存从大到小依次划分为Node（节点）->Zone（内存域）->Page（页）：

*   节点：在大型结算及系统中，内存有不同的簇，依据对处理器距离的不同，访问这些簇有不同的代价。而这些簇就可以成为节点。例：在PC系统中可以理解为实际挂载的物理内存；在嵌入式系统中，有两块内存芯片A和B，分别代表一个节点。
*   内存域：内存域并不是物理存在的概念，是Linux系统对每个内存节点进行管理的单位，每个节点的内存域表示的是对该节点不同地址范围的划分。一般内存域有三种，分别为Normal、DMA和HighMem。
*   页：在每个内存域中，内存被划分为大小固定的块（32位系统一般为4K大小），为内核进行内存分配的基本单位（当然内核内存管理机制其实更为复杂，“基本单位”不代表每次分配内存最小就要分到4K。后边可以看到，当需要获取小于4K大小的内存时，内核有Slab分配器来满足要求）

一张图说明Node、Zone和Page的关系如下：
```
                  Node 1            Node 2           Node 3
                       +----------+     +----------+     +----------+
                       |          |     |          |     |          |
                       |Zone_High |     |          |     |          |
                       |          |     |          |     |          |
                       +----------+     |          |     |          |
                       |          |     |          |     |          |
                       |          |     |          |     |          |
                       |Zone_Norm |     |          |     |          |
                       |          |     |          |     |          |
                       |          |     |          |     |          |
                       +----------+     |          |     |          |
                       |          |     |          |     |          |
     page  page        |Zone_DMA  |     |          |     |          |
+-+--+--+--+--+        |          |     |          |     |          |
| |  |  |  |  |  <--------+       |     |          |     |          |
+-+--+--+--+--+        +----------+     +----------+     +----------+
```
数据结构
----

构成上述三个内存划分的数据结构如下：

### 内存节点

Node对应的结构为pglist_data_t，定义如下（为方便理解，省略部分结构体成员）：
```C
typedef struct pglist_data {
    struct zone node_zones_MAX_NR_ZONES];
    struct zonelist node_zonelists_MAX_ZONELISTS];
    int nr_zones;
#ifdef CONFIG_FLAT_NODE_MEM_MAP /* means !SPARSEMEM */
    struct page *node_mem_map;
#ifdef CONFIG_PAGE_EXTENSION
    struct page_ext *node_page_ext;
#endif
#endif
#ifndef CONFIG_NO_BOOTMEM
    struct bootmem_data *bdata;
#endif
#ifdef CONFIG_MEMORY_HOTPLUG
    spinlock_t node_size_lock;
#endif
    unsigned long node_start_pfn;
    unsigned long node_present_pages; /* total number of physical pages */
    unsigned long node_spanned_pages; /* total size of physical page  range, including holes */
    int node_id;
    wait_queue_head_t kswapd_wait;
    wait_queue_head_t pfmemalloc_wait;
    struct task_struct *kswapd; /* Protected by mem_hotplug_begin/end() */
    int kswapd_order;
    enum zone_type kswapd_classzone_idx;
    int kswapd_failures;        /* Number of 'reclaimed == 0' runs */
#ifdef CONFIG_COMPACTION
    int kcompactd_max_order;
    enum zone_type kcompactd_classzone_idx;
    wait_queue_head_t kcompactd_wait;
    struct task_struct *kcompactd;
#endif
#ifdef CONFIG_NUMA_BALANCING
    /* Lock serializing the migrate rate limiting window */
    spinlock_t numabalancing_migrate_lock;
    /* Rate limiting time interval */
    unsigned long numabalancing_migrate_next_window;
    /* Number of pages migrated during the rate limiting time interval */
    unsigned long numabalancing_migrate_nr_pages;
#endif
    unsigned long       totalreserve_pages;
#ifdef CONFIG_NUMA
    /*
     * zone reclaim becomes active if more unmapped pages exist.
     */
    unsigned long       min_unmapped_pages;
    unsigned long       min_slab_pages;
#endif /* CONFIG_NUMA */

    /* Write-intensive fields used by page reclaim */
    ZONE_PADDING(_pad1_)
    spinlock_t      lru_lock;

#ifdef CONFIG_DEFERRED_STRUCT_PAGE_INIT
    /*
     * If memory initialisation on large machines is deferred then this
     * is the first PFN that needs to be initialised.
     */
    unsigned long first_deferred_pfn;
    /* Number of non-deferred pages */
    unsigned long static_init_pgcnt;
#endif /* CONFIG_DEFERRED_STRUCT_PAGE_INIT */

#ifdef CONFIG_TRANSPARENT_HUGEPAGE
    spinlock_t split_queue_lock;
    struct list_head split_queue;
    unsigned long split_queue_len;
#endif
    unsigned int inactive_ratio;
    unsigned long       flags;
    ZONE_PADDING(_pad2_)
    /* Per-node vmstats */
    struct per_cpu_nodestat __percpu *per_cpu_nodestats;
    atomic_long_t       vm_stat_NR_VM_NODE_STAT_ITEMS];
} pg_data_t;
```
*   **node_zones**: 内存节点上的内存域，分别为 ZONE_HIGHMEM, ZONE_NORMAL, ZONE_DMA。新版Linux还增加了ZONE_MOVABLE和ZONE_DEVICE。
*   **node_zonelists:** 对内存域进行类别指定的优先级顺序。例，当ZONE_HIGHMEM分配失败时，会u退到ZONE_DMA类型后ZONE_NORMAL类型
*   **nr_zones:** 该节点上的内存域数量
*   **node_mem_map:** 节点中页面的映射图
*   **bdata:** 与内核初始化内存分配器相关数据
*   **node_size_lock**： 与内存热拔插相关
*   **node_start_pfn:** 内存节点的起始页。
*   **node_present_pages:** 物理页面数量**.**
*   **node_spanned_pages:** 内存节点物理页面的大小
*   **node_id:** 节点编号
*   **kswapd_wait**/**pfmemalloc_wait**/**kswapd**/**kswapd_order**/**kswapd_classzone_idx/kswapd_failures:** kswapd内核线程相关参数
*   **........**

### 内存区域

内存区域对应的结构体为struct zone，定义如下：
```C
struct zone {
    unsigned long watermark_NR_WMARK];
    unsigned long nr_reserved_highatomic;
    long lowmem_reserve_MAX_NR_ZONES];
#ifdef CONFIG_NUMA
    int node;
#endif
    struct pglist_data  *zone_pgdat;
    struct per_cpu_pageset __percpu *pageset;

#ifndef CONFIG_SPARSEMEM
    unsigned long       *pageblock_flags;
#endif /* CONFIG_SPARSEMEM */
    unsigned long       zone_start_pfn;
    unsigned long       managed_pages;
    unsigned long       spanned_pages;
    unsigned long       present_pages;

    const char      *name;
#ifdef CONFIG_MEMORY_ISOLATION
    unsigned long       nr_isolate_pageblock;
#endif

#ifdef CONFIG_MEMORY_HOTPLUG
    seqlock_t       span_seqlock;
#endif
    int initialized;
    ZONE_PADDING(_pad1_)
    struct free_area    free_area_MAX_ORDER];
    unsigned long       flags;
    spinlock_t      lock;
    ZONE_PADDING(_pad2_)
    unsigned long percpu_drift_mark;
    ........
    atomic_long_t       vm_stat_NR_VM_ZONE_STAT_ITEMS];
} ____cacheline_internodealigned_in_smp;
```
*   **watermark**: 内存域的水位
*   **nr_reserved_highatomic:** 紧急内存大小，
*   **lowmem_reserve**:内存域最少保留内存
*   **zone_pgdat**: 所在内存节点指针
*   **pageset**: 每个CPU维护的页面列表
*   **zone_start_pfn**:内存域第一个页的索引
*   **managed_pages**: 伙伴系统管理的所有页面数量
*   **spanned_pages**: 内存域所跨越所有内存页数量
*   **present_pages**: 内存域物理内存所有页数量(除去内存空洞后的部分)present_pages=spanned_pages-absent_pages
*   **name**: 区域名
*   **free_area**:所有空闲页面的数组
*   **flags**:内存域标识
*   **lock**:保护free_area的锁
*   **vm_stat**:虚拟内存统计信息

特别说明一下内存域的水位（Watermark），它表示几个阈值，用来管理内核线程kswapd唤起与休眠的。当域内可用内存水位较高时，kswapd不用起来工作，而水位较低时，kswapd需要唤起来回收内存。如下图（来自深入理解Linux虚拟内存管理）：

![](http://pic.l2h.site/屏幕快照-2019-08-13-上午7.40.53.png)

### 页面

系统中每个物理页面都有数据结构struct page与其关联，用于管理页面的使用。结构如下：
```C
truct page {
    /* First double word block */
    unsigned long flags;       
    union {
        struct address_space *mapping; 
        void *s_mem;            /* slab first object */
        atomic_t compound_mapcount; /* first tail page */
    };

    /* Second double word */
    union {
        pgoff_t index;      /* Our offset within mapping. */
        void *freelist;     /* sl_aou]b first free object */
        /* page_deferred_list().prev    -- second tail page */
    };

    union {
#if defined(CONFIG_HAVE_CMPXCHG_DOUBLE) && \
    defined(CONFIG_HAVE_ALIGNED_STRUCT_PAGE)
        /* Used for cmpxchg_double in slub */
        unsigned long counters;
#else
        unsigned counters;
#endif
        struct {
            union {
                atomic_t _mapcount;
                unsigned int active;        /* SLAB */
                struct {            /* SLUB */
                    unsigned inuse:16;
                    unsigned objects:15;
                    unsigned frozen:1;
                };
                int units;          /* SLOB */
            };
            atomic_t _refcount;
        };
    };

    /*  Third double word block */
    union {
        struct list_head lru;   
        struct dev_pagemap *pgmap; 
        struct {        /* slub per cpu partial pages */
            struct page *next;  /* Next partial slab */
#ifdef CONFIG_64BIT
            int pages;  /* Nr of partial slabs left */
            int pobjects;   /* Approximate # of objects */
#else
            short int pages;
            short int pobjects;
#endif
        };

        struct rcu_head rcu_head;   
        struct {
            unsigned long compound_head; 
#ifdef CONFIG_64BIT
            unsigned int compound_dtor;
            unsigned int compound_order;
#else
            unsigned short int compound_dtor;
            unsigned short int compound_order;
#endif
        };

#if defined(CONFIG_TRANSPARENT_HUGEPAGE) && USE_SPLIT_PMD_PTLOCKS
        struct {
            unsigned long __pad;    
            pgtable_t pmd_huge_pte; /* protected by page->ptl */
        };
#endif
    };

    /* Remainder is not double word aligned */
    union {
        unsigned long private;      
#if USE_SPLIT_PTE_PTLOCKS
#if ALLOC_SPLIT_PTLOCKS
        spinlock_t *ptl;
#else
        spinlock_t ptl;
#endif
#endif
        struct kmem_cache *slab_cache;  /* SL_AU]B: Pointer to slab */
    };

#ifdef CONFIG_MEMCG
    struct mem_cgroup *mem_cgroup;
#endif
#if defined(WANT_PAGE_VIRTUAL)
    void *virtual;          /* Kernel virtual address (NULL if  not kmapped, ie. highmem) */
#endif /* WANT_PAGE_VIRTUAL */

#ifdef CONFIG_KMEMCHECK
    void *shadow;
#endif

#ifdef LAST_CPUPID_NOT_IN_PAGE_FLAGS
    int _last_cpupid;
#endif
}
```
页面结构体使用双字块来划分：

*   第一个双字
    
    *   flags: 页面状态，脏页、上锁等院子标记
    
    *   联合体
        
        *   mapping：指向inode address_space
        
        *   s_mem：slab首对象
        
        *   compound_mapcount：
*   第二个双字：
    
    *   联合体
        
        *   index：页面偏移
        
        *   freelist：slab/slob的首个可用对象
    
    *   联合体：slab/slub/slob相关的记数（取决于编译内核时选择的管理方式）
*   第三个双字：
    
    *   lru：换出页列表
    
    *   pgmap:
    
    *   rcu_head
    
    *   结构体，用于slub管理
    
    *   结构体，用于复合页管理
*   联合体(ptl/slab_cache): slab指针，或者PTE自旋锁
*   virtual: 内核虚拟地址。用于高端内存中的页，即无法直接映射到内核内存中的页

页表
--

Linux进行内存寻址时，往往不会直接内存物理地址，需要经过虚拟地址到物理地址的转化。使用虚拟地址的好处是可以避免进程与进程间互踩内存（除非特别指定共享内存），同时虚拟内存的换出换入使得进程使用超过物理内存大小的内存范围。

CPU中内存管理单元（MMU）作用就是根据内存中特定的转化表格（不错，页表本身也是需要内存存储的），将虚拟地址转化为真正的物理地址。而这个表格就是我们所讲的页表。

取决于体系结构，Linux采用三级或者四级页表机制：

*   PGD：Page Global Directory，全局页表目录
*   PUD：Page Upper Directory，上级页表目录
*   PMD：Page Middle Directory，中级页表目录
*   PTE：Page Table Entry，页表表项

每级表项所占位数，取决于我们编译内核时的选择。一般情况下，取决于寻址宽度，以及CPU体系结构每级页表所占位数是有约定俗成的。

![](http://pic.l2h.site/屏幕快照-2019-08-13-上午7.56.58.png)

内核在arch/xxx/include/asm/page.h（其中xxx表示CPU体系结构）定义了一系列的类型、函数和宏来方便对每级页表进行操作。

如上图我们看到的几个SHIFT宏定义，是为了方便通过位移操作来快速获取对应等级页表。

在IA64中用来表示以上各级页表目录的数据结构定义如下：
```C
  typedef struct { unsigned long pte; } pte_t;
  typedef struct { unsigned long pmd; } pmd_t;
#if CONFIG_PGTABLE_LEVELS == 4
  typedef struct { unsigned long pud; } pud_t;
#endif
  typedef struct { unsigned long pgd; } pgd_t;
```
与页表相关的宏或者函数定义有[pmd/pte/pgd_alloc](https://elixir.bootlin.com/linux/latest/ident/pmd_alloc)/free()等等，具体可以参考include/linux/mm.h。

结语
--

本文介绍了Linux内核内存管理的基本单位划分Node、Zone和Page及对应的数据结构，同时对页表的基本概念进行了介绍。将在下一文分析Linux初始化流程中对内存的管理。