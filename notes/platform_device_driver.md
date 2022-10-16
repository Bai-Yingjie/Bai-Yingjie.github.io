- [关于软中断](#关于软中断)
- [关于各种地址](#关于各种地址)
- [关于smp_processor_id()](#关于smp_processor_id)
  - [问题现象](#问题现象)
  - [分析](#分析)
- [关于initrd](#关于initrd)
- [关于struct file](#关于struct-file)
  - [A. inode, inode也有fop指针(struct file_operations)](#a-inode-inode也有fop指针struct-file_operations)
    - [A1. inode_operations](#a1-inode_operations)
    - [B. file_operations](#b-file_operations)
- [关于vm, 内存管理](#关于vm-内存管理)
  - [A. vm_operations_struct](#a-vm_operations_struct)
    - [A1. vm_fault](#a1-vm_fault)
      - [A11. 物理页框](#a11-物理页框)
- [物理页与线性地址](#物理页与线性地址)
  - [arm / arm64](#arm--arm64)
  - [ia64](#ia64)
  - [x86](#x86)
  - [历史](#历史)
  - [分配页框](#分配页框)
  - [使用](#使用)
- [关于module](#关于module)
  - [头文件](#头文件)
  - [MODULE_AUTHOR MODULE_DESCRIPTION宏](#module_author-module_description宏)
  - [导出符号](#导出符号)
  - [声明模块参数](#声明模块参数)
  - [如何修改参数](#如何修改参数)
  - [为初始化函数建立别名init_module](#为初始化函数建立别名init_module)
  - [如何编译module](#如何编译module)
  - [如何编译module完全版](#如何编译module完全版)
- [reboot相关](#reboot相关)
- [CPLD之平台驱动](#cpld之平台驱动)
  - [platform相关驱动](#platform相关驱动)
  - [cpld_probe](#cpld_probe)
  - [platform_driver核心结构](#platform_driver核心结构)
- [关于uio](#关于uio)
  - [注册](#注册)
  - [uio_device](#uio_device)
  - [uio info相关](#uio-info相关)
  - [resource 结构体](#resource-结构体)
  - [uio内核操作](#uio内核操作)
  - [用户态程序读是读中断, 读写uio用mmap](#用户态程序读是读中断-读写uio用mmap)
  - [uio注册设备, 在cpld_probe里面有注册uio设备](#uio注册设备-在cpld_probe里面有注册uio设备)
  - [用户态uio lib](#用户态uio-lib)
    - [用户态使用uio](#用户态使用uio)
  - [UIO总结:](#uio总结)
- [平台设备初始化](#平台设备初始化)
- [设备驱动如何注册的?](#设备驱动如何注册的)
  - [现在的问题是, dev哪里来的?](#现在的问题是-dev哪里来的)
- [关于device_add和sysfs](#关于device_add和sysfs)
  - [很多地方都会调device_add(), 比如:](#很多地方都会调device_add-比如)
  - [那么device_add()干了什么?](#那么device_add干了什么)
  - [octeon-platform.c](#octeon-platformc)
  - [dev都是在这里生成的?](#dev都是在这里生成的)
- [驱动相关](#驱动相关)
  - [driver的初始化位置](#driver的初始化位置)
  - [platform驱动注册流程](#platform驱动注册流程)
  - [I2C是个字符设备](#i2c是个字符设备)
    - [i2c设备初始化](#i2c设备初始化)
    - [很多地方都会调用i2c_add_driver()](#很多地方都会调用i2c_add_driver)
- [关于device_create](#关于device_create)
  - [原型及主要流程](#原型及主要流程)
  - [i2c中的使用例子](#i2c中的使用例子)
  - [uio中的使用例子](#uio中的使用例子)
- [如何创建设备?](#如何创建设备)
  - [方法1: 在父节点调用](#方法1-在父节点调用)
  - [方法2: 直接在dts里声明](#方法2-直接在dts里声明)
- [mips kernel](#mips-kernel)
  - [mips 64bit空间](#mips-64bit空间)
  - [mips ioremap](#mips-ioremap)
  - [ioread8的入参应该是个CPU地址, 而不是物理地址](#ioread8的入参应该是个cpu地址-而不是物理地址)
  - [编译时的条件检测，条件为真则导致编译错误](#编译时的条件检测条件为真则导致编译错误)
  - [运行时的条件检测，条件为真则触发运行时exception](#运行时的条件检测条件为真则触发运行时exception)
  - [jiffies jiffies_64和HZ](#jiffies-jiffies_64和hz)
  - [local_irq_disable()  关中断](#local_irq_disable--关中断)
  - [current 表示当前进程](#current-表示当前进程)
  - [preempt_disable() 禁止抢占](#preempt_disable-禁止抢占)
  - [自旋锁spin_lock()](#自旋锁spin_lock)
  - [ARRAY_SIZE(arr)](#array_sizearr)
- [各种内核编译宏, 见compiler.h](#各种内核编译宏-见compilerh)
- [dump_stack() 驱动打印调用栈](#dump_stack-驱动打印调用栈)
- [linux中断](#linux中断)
  - [中断注册request_threaded_irq()](#中断注册request_threaded_irq)
  - [关于中断号](#关于中断号)
- [信号量](#信号量)
- [debugfs 在/sys/kernel/debug/创建文件](#debugfs-在syskerneldebug创建文件)
- [关于notify_chain](#关于notify_chain)
- [创建proc的entry, 并绑定相关的文件操作](#创建proc的entry-并绑定相关的文件操作)
- [在sys目录下创建个class](#在sys目录下创建个class)
- [驱动中使用工作队列轮询](#驱动中使用工作队列轮询)
  - [在设备相关结构体内添加work](#在设备相关结构体内添加work)
  - [work的处理函数](#work的处理函数)
  - [在初始化的时候开始工作队列](#在初始化的时候开始工作队列)
  - [在rmmod时删除这个work](#在rmmod时删除这个work)

# 关于软中断
硬件上也有软中断: 也叫做编程异常, 就是一个指令可以触发的异常

linux里面的软中断: 这其实是个软件的概念, 也叫做可延迟函数, 下面说的软中断特指linux软中断.

软中断是静态分配的, 能够并发的运行在多个CPU上, 必须用可重入的函数, 必须用自旋锁保护.

而同类型的tasklet是串行执行的, 不可能出现在两个CPU上同时执行一个tasklet的情况.

[深入理解linux内核](https://github.com/lancetw/ebook-1/tree/master/03_operating_system)里面说, 触发和执行软中断都是在同一个CPU上做的. --是否RPS CPU解决了这个问题?

软中断的触发: `raise_softirq()`, 本质上是调用`wakeup_softirq()`去唤醒本地CPU的ksoftirqd内核线程.

内核在几个固定点上, 会主动的调用`do_softiq()`去处理一些已经在等待处理的软中断, 那里处理不过来的, 再交给ksoftirqd线程.

* `local_bh_enable()`激活软中断时
* `do_IRQ()`的`irq_exit()`时
* 多处理器的核间中断

每个CPU都有自己的ksoftirqd线程, 用来处理那些需要频繁激活自己的软中断, 比如网络. 

# 关于各种地址
有好几种地址: 

1. CPU虚拟地址: `kmalloc()`, `vmalloc()`得到的地址, 可以表示为`void *`

2. CPU物理地址: 不能直接使用, 表示为`phys_addr_t`或`resource_size_t`, 需要用`ioremap()`转成虚拟地址后使用
在`/proc/iomem`里面有体现.  

3. bus地址: 从外设角度来看的地址, 主要对DMA来说的

```
               CPU                  CPU                  Bus
             Virtual              Physical             Address
             Address              Address               Space
              Space                Space
            +-------+             +------+             +------+
            |       |             |MMIO  |   Offset    |      |
            |       |  Virtual    |Space |   applied   |      |
          C +-------+ --------> B +------+ ----------> +------+ A
            |       |  mapping    |      |   by host   |      |
  +-----+   |       |             |      |   bridge    |      |   +--------+
  |     |   |       |             +------+             |      |   |        |
  | CPU |   |       |             | RAM  |             |      |   | Device |
  |     |   |       |             |      |             |      |   |        |
  +-----+   +-------+             +------+             +------+   +--------+
            |       |  Virtual    |Buffer|   Mapping   |      |
          X +-------+ --------> Y +------+ <---------- +------+ Z
            |       |  mapping    | RAM  |   by IOMMU
            |       |             |      |
            |       |             |      |
            +-------+             +------+
```
在初始化阶段, kernel知道一个PCI设备的bar地址`(A)`, 然后转成物理地址`(B)`, 并做为资源`struct resource`
保存在`/proc/iomem`. 驱动使用`ioremap()`获得`(B)`的虚拟地址`(C)`, 然后就可以用`ioread32(C)`来访问总线地址`(A)`.

对一个支持DMA的设备来说, 驱动使用`kmalloc()`或类似的接口获得虚拟地址`(X)`, 通过TLB对应到物理地址`(Y)`. 

驱动可以直接使用地址`(X)`来操作这个buffer, 但设备不可以. 设备的DMA不认CPU的虚拟地址.

在一些简单的系统下面, 设备可以使用物理地址`(Y)`, 这些系统没有IOMMU, 设备看到的地址和CPU的物理地址是一致的;

但其他系统下, 这么做是不行的--这里面设备访问的地址需要经过IOMMU, 比如把总线地址`(Z)`(其实也就是这个设备看到的地址), 转换成物理地址`(Y)`. 这个过程是`dma_map_single()`完成的, 它把输入的虚拟地址`(X)`映射到总线地址`(Z)`, 填在一个表里, 这个表由IOMMU来解析. 使用完了用dma_unmap_single()来取消映射.

那么什么地址是可以DMA的呢? 我理解首先要在物理上地址连续, 比如`kmalloc()`可以DMA, 而`vmalloc()`就不行

# 关于smp_processor_id()
这个函数的作用是获得当前CPU ID.

## 问题现象
但在调LSI驱动的时候, 经常打印:
```
BUG: using smp_processor_id() in preemptible [00000000] code: systemd-udevd/1318
Call trace:

或

BUG: using smp_processor_id() in preemptible [00000000] code: mount/1973
Call trace:
```
每个对这个RAID卡操作的命令都会有很多这样的打印, 但基本功能还OK, 只是有这些打印

## 分析
在打开了CONFIG_DEBUG_PREEMPT选项之后, `smp_processor_id`是个宏, 实际调用`debug_smp_processor_id()`
```c
#ifdef CONFIG_DEBUG_PREEMPT
  extern unsigned int debug_smp_processor_id(void);
# define smp_processor_id() debug_smp_processor_id()
```
而`debug_smp_processor_id()`会做一些检查, 检查什么呢? 调用smp_processor_id是否安全.

首先, 在以下场景下是安全的:
* 当前这个内核thread(或者说thread这个进程在内核态执行)不可抢占, 即`current_thread_info()->preempt_count`不为0时.  
--为什么pteempt count不为0就不可抢占了呢? 肯定有地方把这个值++了, 比如一些中断或中断下半部调了什么禁止抢占的函数
* 我们明确知道这个thread只绑定在当前这个CPU上, 即这个thread不会被调度到其他CPU上

为什么要在不能抢占的情况下调用呢?

整个系统干的活可以认为是以(thread+核)为单位的, 比如在一个多核系统下, 一个核C1正运行thread A, 此时如果发生抢占, 即运行任务A的核C1被安排去干其他活了, 比如去搞thread B了, 等B运行完了回来可就不一定是A了, 也许A被安排到其他core上去了.

结合smp_processor_id()这个函数, 比如是thread A调的, 如果调的时候还是core C1, 但中间发生内核抢占, 结果是C1去干其他事了, 现在是C2来接手, 但返回C1的CPU ID, 不就乱了吗?

# 关于initrd
在init/initramfs.c中，`populate_rootfs()`, 会首先调用`unpack_to_rootfs(__initramfs_start, __initramfs_size)`来从kernel内置的rootfs解压.

其次，会从initrd_start这个变量地址解压，此时需要两个变量，一个就是前面的initrd_start，还有就是initrd_end。

那么initrd_start和initrd_end是哪里设的呢?

# 关于struct file
内核中文件表示为`struct file`结构体, 有fop, 也保存了inode指针
```c
struct file {
    /*
    * fu_list becomes invalid after file_free is called and queued via
    * fu_rcuhead for RCU freeing
    */
    union {
        struct list_head    fu_list;
        struct rcu_head     fu_rcuhead;
    } f_u;
    struct path         f_path;
#define f_dentry        f_path.dentry
    
    //指向inode的指针
    struct inode        *f_inode;       /* cached value */ //见A
    const struct file_operations    *f_op; //见B
    /*
    * Protects f_ep_links, f_flags, f_pos vs i_size in lseek SEEK_CUR.
    * Must not be taken from IRQ context.
    */
    spinlock_t          f_lock;
#ifdef CONFIG_SMP
    int                 f_sb_list_cpu;
#endif
    atomic_long_t       f_count;
    unsigned int        f_flags;
    fmode_t             f_mode;
    loff_t              f_pos;
    struct fown_struct  f_owner;
    const struct cred   *f_cred;
    struct file_ra_state    f_ra;
    u64                 f_version;
#ifdef CONFIG_SECURITY
    void                *f_security;
#endif
    /* needed for tty driver, and maybe others */
    void                *private_data;
#ifdef CONFIG_EPOLL
    /* Used by fs/eventpoll.c to link all the hooks to this file */
    struct list_head    f_ep_links;
    struct list_head    f_tfile_llink;
#endif /* #ifdef CONFIG_EPOLL */
    struct address_space    *f_mapping;
#ifdef CONFIG_DEBUG_WRITECOUNT
    unsigned long       f_mnt_write_state;
#endif
#ifdef CONFIG_FUMOUNT
    atomic_t            f_getcount;
    struct list_head    fumount_list;
#endif
};
```

## A. inode, inode也有fop指针(struct file_operations)
inode 处理"实体"
```c
/*
 * Keep mostly read-only and often accessed (especially for
 * the RCU path lookup and 'stat' data) fields at the beginning
 * of the 'struct inode'
 */
struct inode {
    umode_t             i_mode;
    unsigned short      i_opflags;
    kuid_t              i_uid;
    kgid_t              i_gid;
    unsigned int        i_flags;
#ifdef CONFIG_FS_POSIX_ACL
    struct posix_acl	*i_acl;
    struct posix_acl	*i_default_acl;
#endif
    const struct inode_operations   *i_op; //见A1
    struct super_block              *i_sb;
    struct address_space            *i_mapping;
#ifdef CONFIG_SECURITY
    void                *i_security;
#endif
    /* Stat data, not accessed from path walking */
    unsigned long       i_ino;
    /*
    * Filesystems may only read i_nlink directly.  They shall use the
    * following functions for modification:
    *
    *    (set|clear|inc|drop)_nlink
    *    inode_(inc|dec)_link_count
    */
    union {
        const unsigned int  i_nlink;
        unsigned int        __i_nlink;
    };
    dev_t               i_rdev;
    loff_t              i_size;
    struct timespec     i_atime;
    struct timespec     i_mtime;
    struct timespec     i_ctime;
    spinlock_           i_lock;     /* i_blocks, i_bytes, maybe i_size */
    unsigned short      i_bytes;
    unsigned int        i_blkbits;
    blkcnt_t            i_blocks;
#ifdef __NEED_I_SIZE_ORDERED
    seqcount_t          i_size_seqcount;
#endif
    /* Misc */
    unsigned long       i_state;
    struct mutex        i_mutex;
    unsigned long       dirtied_when;   /* jiffies of first dirtying */
    struct hlist_node   i_hash;
    struct list_head    i_wb_list;      /* backing dev IO list */
    struct list_head    i_lru;          /* inode LRU list */
    struct list_head    i_sb_list;
    union {
        struct hlist_head	i_dentry;
        struct rcu_head		i_rcu;
    };
    u64             i_version;
    atomic_t        i_count;
    atomic_t        i_dio_count;
    atomic_t        i_writecount;
    //重要!!! 指向fop的指针.
    const struct file_operations    *i_fop;	/* former ->i_op->default_file_ops */
    struct file_lock                *i_flock;
    struct address_space            i_data;
#ifdef CONFIG_QUOTA
    struct dquot                    *i_dquot[MAXQUOTAS];
#endif
    struct list_head                i_devices;
    union {
        struct pipe_inode_info  *i_pipe;
        struct block_device     *i_bdev;
        struct cdev             *i_cdev;
    };
    __u32           i_generation;
#ifdef CONFIG_FSNOTIFY
    __u32           i_fsnotify_mask; /* all events this inode cares about */
    struct hlist_head   i_fsnotify_marks;
#endif
#ifdef CONFIG_IMA
    atomic_t        i_readcount; /* struct files open RO */
#endif
    void            *i_private; /* fs or device private pointer */
};
```

### A1. inode_operations
inode的操作主要针对"实体存在"的操作, 比如创建, 删除, 重命名等
```c
struct inode_operations {
    struct dentry * (*lookup) (struct inode *,struct dentry *, unsigned int);
    void * (*follow_link) (struct dentry *, struct nameidata *);
    int (*permission) (struct inode *, int);
    struct posix_acl * (*get_acl)(struct inode *, int);
    int (*readlink) (struct dentry *, char __user *,int);
    void (*put_link) (struct dentry *, struct nameidata *, void *);
    int (*create) (struct inode *,struct dentry *, umode_t, bool);
    int (*link) (struct dentry *,struct inode *,struct dentry *);
    int (*unlink) (struct inode *,struct dentry *);
    int (*symlink) (struct inode *,struct dentry *,const char *);
    int (*mkdir) (struct inode *,struct dentry *,umode_t);
    int (*rmdir) (struct inode *,struct dentry *);
    int (*mknod) (struct inode *,struct dentry *,umode_t,dev_t);
    int (*rename) (struct inode *, struct dentry *,
    struct inode *, struct dentry *);
    int (*setattr) (struct dentry *, struct iattr *);
    int (*getattr) (struct vfsmount *mnt, struct dentry *, struct kstat *);
    int (*setxattr) (struct dentry *, const char *,const void *,size_t,int);
    ssize_t (*getxattr) (struct dentry *, const char *, void *, size_t);
    ssize_t (*listxattr) (struct dentry *, char *, size_t);
    int (*removexattr) (struct dentry *, const char *);
    int (*fiemap)(struct inode *, struct fiemap_extent_info *, u64 start, u64 len);
    int (*update_time)(struct inode *, struct timespec *, int);
    int (*atomic_open)(struct inode *, struct dentry *,
                struct file *, unsigned open_flag,
                umode_t create_mode, int *opened);
} ____cacheline_aligned;
```

### B. file_operations
file_operations表述的是系统和"实体存在"之间的交互, 比如读写, mmap, 锁等
```c
    struct file_operations {
    struct module *owner;
    loff_t (*llseek) (struct file *, loff_t, int);
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*aio_read) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
    ssize_t (*aio_write) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
    int (*readdir) (struct file *, void *, filldir_t);
    unsigned int (*poll) (struct file *, struct poll_table_struct *);
    long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
    long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
    int (*mmap) (struct file *, struct vm_area_struct *); //见下节, 内存管理
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
    int (*setlease)(struct file *, long, struct file_lock **);
    long (*fallocate)(struct file *file, int mode, loff_t offset,
                loff_t len);
    int (*show_fdinfo)(struct seq_file *m, struct file *f);
} __do_const;
```

# 关于vm, 内存管理
一个进程可以有很多`vm_area_struct`, 它们用红黑树来管理  
`include/linux/mm_types.h`
```c
/*
 * This struct defines a memory VMM memory area. There is one of these
 * per VM-area/task.  A VM area is any part of the process virtual memory
 * space that has a special rule for the page-fault handlers (ie a shared
 * library, the executable area etc).
 */
struct vm_area_struct {
    /* The first cache line has the info for VMA tree walking. */
    unsigned long vm_start;		/* Our start address within vm_mm. */
    unsigned long vm_end;		/* The first byte after our end address within vm_mm. */
    /* linked list of VM areas per task, sorted by address */
    struct vm_area_struct *vm_next, *vm_prev;
    struct rb_node vm_rb;
    /*
    * Largest free memory gap in bytes to the left of this VMA.
    * Either between this VMA and vma->vm_prev, or between one of the
    * VMAs below us in the VMA rbtree and its ->vm_prev. This helps
    * get_unmapped_area find a free area of the right size.
    */
    unsigned long rb_subtree_gap;
    /* Second cache line starts here. */
    struct mm_struct *vm_mm;	/* The address space we belong to. */
    pgprot_t vm_page_prot;		/* Access permissions of this VMA. */
    unsigned long vm_flags;		/* Flags, see mm.h. */
    /*
    * For areas with an address space and backing store,
    * linkage into the address_space->i_mmap interval tree, or
    * linkage of vma in the address_space->i_mmap_nonlinear list.
    */
    union {
        struct {
            struct rb_node rb;
            unsigned long rb_subtree_last;
        } linear;
    struct list_head nonlinear;
    } shared;
    /*
    * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
    * list, after a COW of one of the file pages.	A MAP_SHARED vma
    * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
    * or brk vma (with NULL file) can only be in an anon_vma list.
    */
    struct list_head anon_vma_chain; /* Serialized by mmap_sem & * page_table_lock */
    struct anon_vma *anon_vma;	/* Serialized by page_table_lock */
    /* Function pointers to deal with this struct. */
    const struct vm_operations_struct *vm_ops; //见A
    /* Information about our backing store: */
    unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE units, *not* PAGE_CACHE_SIZE */
    struct file * vm_file;		/* File we map to (can be NULL). */
    void * vm_private_data;		/* was vm_pte (shared mem) */
#ifndef CONFIG_MMU
    struct vm_region *vm_region;	/* NOMMU mapping region */
#endif
#ifdef CONFIG_NUMA
    struct mempolicy *vm_policy;	/* NUMA policy for the VMA */
#endif
    struct vm_area_struct *vm_mirror;/* PaX: mirror vma or NULL */
};
```

## A. vm_operations_struct
`include/linux/mm.h`
```c
/*
 * These are the virtual MM functions - opening of an area, closing and
 * unmapping it (needed to keep files on disk up-to-date etc), pointer
 * to the functions called when a no-page or a wp-page exception occurs. 
 */
struct vm_operations_struct {
    void (*open)(struct vm_area_struct * area);
    void (*close)(struct vm_area_struct * area);
    int (*fault)(struct vm_area_struct *vma, struct vm_fault *vmf); //见A1
    /* notification that a previously read-only page is about to become
    * writable, if an error is returned it will cause a SIGBUS 
    */
    int (*page_mkwrite)(struct vm_area_struct *vma, struct vm_fault *vmf);
    /* called by access_process_vm when get_user_pages() fails, typically
    * for use by special VMAs that can switch between memory and hardware
    */
    ssize_t (*access)(struct vm_area_struct *vma, unsigned long addr, void *buf, size_t len, int write);
#ifdef CONFIG_NUMA
    /*
    * set_policy() op must add a reference to any non-NULL @new mempolicy
    * to hold the policy upon return.  Caller should pass NULL @new to
    * remove a policy and fall back to surrounding context--i.e. do not
    * install a MPOL_DEFAULT policy, nor the task or system default
    * mempolicy.
    */
    int (*set_policy)(struct vm_area_struct *vma, struct mempolicy *new);
    /*
    * get_policy() op must add reference [mpol_get()] to any policy at
    * (vma,addr) marked as MPOL_SHARED.  The shared policy infrastructure
    * in mm/mempolicy.c will do this automatically.
    * get_policy() must NOT add a ref if the policy at (vma,addr) is not
    * marked as MPOL_SHARED. vma policies are protected by the mmap_sem.
    * If no [shared/vma] mempolicy exists at the addr, get_policy() op
    * must return NULL--i.e., do not "fallback" to task or system default
    * policy.
    */
    struct mempolicy *(*get_policy)(struct vm_area_struct *vma,
unsigned long addr);
    int (*migrate)(struct vm_area_struct *vma, const nodemask_t *from, const nodemask_t *to, unsigned long flags);
#endif
    /* called by sys_remap_file_pages() to populate non-linear mapping */
    int (*remap_pages)(struct vm_area_struct *vma, unsigned long addr, unsigned long size, pgoff_t pgoff);
};
typedef struct vm_operations_struct __no_const vm_operations_struct_no_const;
```

### A1. vm_fault
```c
/*
 * vm_fault is filled by the the pagefault handler and passed to the vma's
 * ->fault function. The vma's ->fault is responsible for returning a bitmask
 * of VM_FAULT_xxx flags that give details about how the fault was handled.
 *
 * pgoff should be used in favour of virtual_address, if possible. If pgoff
 * is used, one may implement ->remap_pages to get nonlinear mapping support.
 */
struct vm_fault {
    unsigned int flags;		/* FAULT_FLAG_xxx flags */
    pgoff_t pgoff;			/* Logical page offset based on vma */
    void __user *virtual_address;	/* Faulting virtual address */
    struct page *page; //见A11
    /* ->fault handlers should return a 
    * page here, unless VM_FAULT_NOPAGE
    * is set (which is also implied by
    * VM_FAULT_ERROR).
    */
};
```

#### A11. 物理页框
```c
/*
 * Each physical page in the system has a struct page associated with
 * it to keep track of whatever it is we are using the page for at the
 * moment. Note that we have no way to track which tasks are using
 * a page, though if it is a pagecache page, rmap structures can tell us
 * who is mapping it.
 *
 * The objects in struct page are organized in double word blocks in
 * order to allows us to use atomic double word operations on portions
 * of struct page. That is currently only used by slub but the arrangement
 * allows the use of atomic double word operations on the flags/mapping
 * and lru list pointers also.
 */
struct page {
    /* First double word block */
    unsigned long flags;        /* Atomic flags, some possibly
                                * updated asynchronously */
    struct address_space *mapping;  /* If low bit clear, points to
                                    * inode address_space, or NULL.
                                    * If page mapped as anonymous
                                    * memory, low bit is set, and
                                    * it points to anon_vma object:
                                    * see PAGE_MAPPING_ANON below.
                                    */
    /* Second double word */
    struct {
        union {
            pgoff_t index;		/* Our offset within mapping. */
            void *freelist;		/* slub/slob first free object */
            bool pfmemalloc;    /* If set by the page allocator,
                                * ALLOC_NO_WATERMARKS was set
                                * and the low watermark was not
                                * met implying that the system
                                * is under some pressure. The
                                * caller should try ensure
                                * this page is only used to
                                * free other pages.
                                */
        };
        union {
#if defined(CONFIG_HAVE_CMPXCHG_DOUBLE) && 
	defined(CONFIG_HAVE_ALIGNED_STRUCT_PAGE)
            /* Used for cmpxchg_double in slub */
            unsigned long counters;
#else
            /*
            * Keep _count separate from slub cmpxchg_double data.
            * As the rest of the double word is protected by
            * slab_lock but _count is not.
            */
            unsigned counters;
#endif
            struct {
                union {
                    /*
                    * Count of ptes mapped in
                    * mms, to show when page is
                    * mapped & limit reverse map
                    * searches.
                    *
                    * Used also for tail pages
                    * refcounting instead of
                    * _count. Tail pages cannot
                    * be mapped and keeping the
                    * tail page _count zero at
                    * all times guarantees
                    * get_page_unless_zero() will
                    * never succeed on tail
                    * pages.
                    */
                    atomic_t _mapcount;
                    struct { /* SLUB */
                        unsigned inuse:16;
                        unsigned objects:15;
                        unsigned frozen:1;
                    };
                    int units;	/* SLOB */
                };
                atomic_t _count;		/* Usage count, see below. */
            };
        };
    };
    /* Third double word block */
    union {
        struct list_head lru;	/* Pageout list, eg. active_list
                                * protected by zone->lru_lock !
                                */
        struct {		/* slub per cpu partial pages */
            struct page *next;	/* Next partial slab */
#ifdef CONFIG_64BIT
            int pages;	/* Nr of partial slabs left */
            int pobjects;	/* Approximate # of objects */
#else
            short int pages;
            short int pobjects;
#endif
        };
        struct list_head list;	/* slobs list of pages */
        struct slab *slab_page; /* slab fields */
    };
    /* Remainder is not double word aligned */
    union {
        unsigned long private;  /* Mapping-private opaque data:
                                * usually used for buffer_heads
                                * if PagePrivate set; used for
                                * swp_entry_t if PageSwapCache;
                                * indicates order in the buddy
                                * system if PG_buddy is set.
                                */
#if USE_SPLIT_PTLOCKS
# ifndef CONFIG_PREEMPT_RT_FULL
        spinlock_t ptl;
# else
        spinlock_t *ptl;
# endif
#endif
        struct kmem_cache *slab_cache;	/* SL[AU]B: Pointer to slab */
        struct page *first_page;	/* Compound tail pages */
    };
    /*
    * On machines where all RAM is mapped into kernel address space,
    * we can simply calculate the virtual address. On machines with
    * highmem some memory is mapped into kernel virtual memory
    * dynamically, so we need a place to store that address.
    * Note that this field could be 16 bits on x86 ... ;)
    *
    * Architectures with slow multiplication can define
    * WANT_PAGE_VIRTUAL in asm/page.h
    */
#if defined(WANT_PAGE_VIRTUAL)
    void *virtual;      /* Kernel virtual address (NULL if not kmapped, ie. highmem) */
#endif /* WANT_PAGE_VIRTUAL */
#ifdef CONFIG_WANT_PAGE_DEBUG_FLAGS
    unsigned long debug_flags;	/* Use atomic bitops on this */
#endif
#ifdef CONFIG_KMEMCHECK
    /*
    * kmemcheck wants to track the status of each byte in a page; this
    * is a pointer to such a status block. NULL if not tracked.
    */
    void *shadow;
#endif
#ifdef LAST_NID_NOT_IN_PAGE_FLAGS
    int _last_nid;
#endif
}
```

# 物理页与线性地址
## arm / arm64
```c
arch/arm/include/asm/memory.h
#define PAGE_OFFSET		UL(CONFIG_PAGE_OFFSET)
#define __virt_to_phys(x)	((x) - PAGE_OFFSET + PHYS_OFFSET)
#define __phys_to_virt(x)	((x) - PHYS_OFFSET + PAGE_OFFSET)

arch/arm64/include/asm/memory.h
#define PAGE_OFFSET        UL(0xffffc00000000000)
//这里的PHYS_OFFSET应该是内存的起始物理地址
#define __virt_to_phys(x)    (((phys_addr_t)(x) - PAGE_OFFSET + PHYS_OFFSET))
#define __phys_to_virt(x)    ((unsigned long)((x) - PHYS_OFFSET + PAGE_OFFSET))

/*
 * These are *only* valid on the kernel direct mapped RAM memory.
 * Note: Drivers should NOT use these.  They are the wrong
 * translation for translating DMA addresses.  Use the driver
 * DMA support - see dma-mapping.h.
 */
static inline phys_addr_t virt_to_phys(const volatile void *x)
{
    return __virt_to_phys((unsigned long)(x));
}
```

## ia64
```c
/*
 * The top three bits of an IA64 address are its Region Number.
 * Different regions are assigned to different purposes.
 */
#define RGN_SHIFT	(61)
#define RGN_BASE(r)	(__IA64_UL_CONST(r)<<RGN_SHIFT)
#define RGN_BITS	(RGN_BASE(-1))
#define RGN_KERNEL	7	/* Identity mapped region */
#define RGN_UNCACHED    6	/* Identity mapped I/O region */
#define RGN_GATE	5	/* Gate page, Kernel text, etc */
#define RGN_HPAGE	4	/* For Huge TLB pages */

#define PAGE_OFFSET			RGN_BASE(RGN_KERNEL)

/*
 * Change virtual addresses to physical addresses and vv.
 */
static inline unsigned long
virt_to_phys (volatile void *address)
{
    return (unsigned long) address - PAGE_OFFSET;
}
```

## x86
```c
/*
 * This handles the memory map.
 *
 * A __PAGE_OFFSET of 0xC0000000 means that the kernel has
 * a virtual address space of one gigabyte, which limits the
 * amount of physical memory you can use to about 950MB.
 *
 * If you want more physical memory than this then see the CONFIG_HIGHMEM4G
 * and CONFIG_HIGHMEM64G options in the kernel configuration.
 */
#define __PAGE_OFFSET		_AC(CONFIG_PAGE_OFFSET, UL)
```

## 历史
x86历史上, 内存被划分为:
* ZONE_DMA: 物理内存低16M; 受限于ISA总线
* ZONE_NORMAL: 物理内存高于16M但低于896M; 被线性的映射到"第四个GB"; 内核可以直接访问; "内核页表"
* ZONE_HIGHMEM: 物理内存高于896M; 内核不能直接使用; 在64位模式下总是空的.

## 分配页框
内核函数alloc_page*()或__get_free_page*()用来获得物理页框, 可以接受一些mask

`include/linux/gfp.h`
```c
#define __GFP_DMA	((__force gfp_t)___GFP_DMA) //从DMA区分页框
#define __GFP_HIGHMEM	((__force gfp_t)___GFP_HIGHMEM) //从HIGHMEM分页框
```

如果没有以上标记, 则从ZONE_NORMAL分配页框

另外还有一些标记:
```c
#define __GFP_WAIT	((__force gfp_t)___GFP_WAIT)    /* Can wait and reschedule? */
#define __GFP_HIGH	((__force gfp_t)___GFP_HIGH)    /* Should access emergency pools? */
#define __GFP_IO	((__force gfp_t)___GFP_IO)      /* Can start physical IO? */
#define __GFP_FS	((__force gfp_t)___GFP_FS)      /* Can call down to low-level FS? */
#define __GFP_ZERO	((__force gfp_t)___GFP_ZERO)    /* Return zeroed page on success */
```

组合标记类型:
```c
#define GFP_ATOMIC(__GFP_HIGH)
#define GFP_NOIO	(__GFP_WAIT)
#define GFP_NOFS	(__GFP_WAIT | __GFP_IO)
#define GFP_KERNEL	(__GFP_WAIT | __GFP_IO | __GFP_FS) //一般使用这个标记的是从NORMAL区分的
#define GFP_TEMPORARY	(__GFP_WAIT | __GFP_IO | __GFP_FS | 
			 __GFP_RECLAIMABLE)
#define GFP_USER	(__GFP_WAIT | __GFP_IO | __GFP_FS | __GFP_HARDWALL)
#define GFP_HIGHUSER	(__GFP_WAIT | __GFP_IO | __GFP_FS | __GFP_HARDWALL | 
			 __GFP_HIGHMEM)
#define GFP_HIGHUSER_MOVABLE	(__GFP_WAIT | __GFP_IO | __GFP_FS | 
				 __GFP_HARDWALL | __GFP_HIGHMEM | 
				 __GFP_MOVABLE)
#define GFP_IOFS	(__GFP_IO | __GFP_FS)
#define GFP_TRANSHUGE	(GFP_HIGHUSER_MOVABLE | __GFP_COMP | 
			 __GFP_NOMEMALLOC | __GFP_NORETRY | __GFP_NOWARN | 
			 __GFP_NO_KSWAPD)
```

## 使用
```c
# define __pa(x)		((x) - PAGE_OFFSET) 
推荐使用
static inline unsigned long virt_to_phys (volatile void *address)
# define __va(x)		((x) + PAGE_OFFSET) 
推荐使用
static inline void* phys_to_virt (unsigned long address)

#define page_to_phys(page)	(page_to_pfn(page) << PAGE_SHIFT)
//返回一个	struct page 见A11
#define virt_to_page(kaddr)	pfn_to_page(__pa(kaddr) >> PAGE_SHIFT)
#define pfn_to_kaddr(pfn)	__va((pfn) << PAGE_SHIFT)
```

# 关于module
## 头文件
```c
#include <linux/module.h>   /* Needed by all modules */
#include <linux/kernel.h>   /* Needed for KERN_INFO */
```

## MODULE_AUTHOR MODULE_DESCRIPTION宏
类似的宏, 会编译在特殊的段".modinfo"
```c
MODULE_LICENSE("GPL");
    MODULE_INFO(license, "GPL")
    #ifdef MODULE
        static const char __mod_license__LINE__[] __attribute__((section(".modinfo"),unused)) = "license=GPL"
    #else
        空
#define MODULE_PARM_DESC(_parm, desc) \
    __MODULE_INFO(parm, _parm, #_parm ":" desc)
```

## 导出符号
在`__ksymtab`段, 定义一个{地址, 名字}的结点
```c
EXPORT_SYMBOL(sym)
    __EXPORT_SYMBOL(sym, sec)
    __EXPORT_SYMBOL(sym, "")
        extern typeof(sym) sym;
        static const char __kstrtab_##sym[] __attribute__((section("__ksymtab_strings"), aligned(1))) = #sym
        static const struct kernel_symbol __ksymtab_##sym __attribute__((section("__ksymtab" sec), unused)) = { (unsigned long)&sym, __kstrtab_##sym }
其中,
struct kernel_symbol
{
    unsigned long value;
    const char *name;
};
```

## 声明模块参数
参数类型`byte`, `short`, `ushort`, `int`, `uint`, `long`, `ulong`, `charp`, `bool` or `invbool`  
也可以用`module_param_array`声明数组

在段`__param`保存一个下面的结构体
```c
struct kernel_param {
    const char *name;
    u16 perm;
    u16 flags;
    param_set_fn set;
    param_get_fn get;
    union {
        void *arg;
        const struct kparam_string *str;
        const struct kparam_array *arr;
    };
};
module_param(name, type, perm)
module_param(panic_counter, ulong, 0644);
    module_param_named(name, name, type, perm)
        module_param_call(name, param_set_##type, param_get_##type, &value, perm);
            static struct kernel_param __moduleparam_const __param_##name __attribute__ ((unused,__section__ ("__param"),aligned(sizeof(void *)))) \
            = { __param_str_##name, perm, isbool ? KPARAM_ISBOOL : 0, set, get, { arg } }
        __MODULE_INFO(parmtype, name##type, #name ":" _type)
```

参数的权限如下:
```c
#define S_IRWXU 00700
#define S_IRUSR 00400
#define S_IWUSR 00200
#define S_IXUSR 00100
#define S_IRWXG 00070
#define S_IRGRP 00040
#define S_IWGRP 00020
#define S_IXGRP 00010
#define S_IRWXO 00007
#define S_IROTH 00004
#define S_IWOTH 00002
#define S_IXOTH 00001
```

## 如何修改参数
* 加载module时: 
  * 例1: `insmod hello.ko msg_buf=veryCD n_arr=1,2,3,4,5,6`
  * 例2: `modprobe usbcore blinkenlights=1`
* 模块加载以后, 还可以使用sysfs文件系统动态修改:
  * 例: `/sys/module/modulexxx/parameters/xxx`
* 模块编进内核时, 在启动linux时加选项:
  * 例: `usbcore.blinkenlights=1`

## 为初始化函数建立别名init_module
可能系统调用要用到这个函数, SYSCALL_DEFINE3(init_module ...), 前提是这个module已经被insmod进内核空间?  
--不是的, SYSCALL_DEFINE3只是定义一个系统调用sys_init_module.

那这个init_module的别名有什么用?
```c
#include <linux/init.h>
module_init(reboot_helper_init);
module_init(initfn)
    static inline initcall_t __inittest(void) {return initfn;}       
    int init_module(void) __attribute__((alias(#initfn)));
```

## 如何编译module
```c
obj-m := hello.o
kDIR := /lib/modules/2.6.18-53.el5/build
make –C $(KDIR)  M=$(PWD) modules
```

```c
obj-m  := megaraid_sas.o

megaraid_sas-objs := megaraid_sas_base.o megaraid_sas_fusion.o megaraid_sas_fp.o megaraid_sas_raptor.o
```
## 如何编译module完全版

https://www.kernel.org/doc/Documentation/kbuild/modules.txt

# reboot相关
```c
reboot_helper_init()
    request_mem_region(mem_address, mem_size, "reset_safe_reboot_info");
    reboot_info = ioremap(mem_address,mem_size);
    node = of_find_node_by_name(NULL, "cpld");
    prop = of_get_address(node, 0, &cpld_size, NULL);
    cpld_start = of_translate_address(node, prop);
    of_node_put(node);
    request_mem_region(cpld_start, cpld_size, "cpld");
    _cpld = ioremap(cpld_start, cpld_size);
    atomic_notifier_chain_register(&panic_notifier_list, &panic_helper_notifier_block);
    register_reboot_notifier(&reboot_helper_notifier_block);
    board_specific_init(reboot_info);

reboot_helper_exit()
    iounmap(_cpld);
    release_mem_region(cpld_start, cpld_size);
    unregister_reboot_notifier(&reboot_helper_notifier_block);
    atomic_notifier_chain_unregister(&panic_notifier_list, &panic_helper_notifier_block);
    iounmap(reboot_info);
    release_mem_region(mem_address, mem_size);
```

# CPLD之平台驱动
驱动常用函数
```c
//可移植性好的io读, 需#include <asm/io.h>
ioread8(off_reg)

// irq
disable_irq(dev_info->irq)
enable_irq(dev_info->irq);  //#include <linux/interrupt.h>

// of与dts对应的处理函数
of_get_property()           //#include <linux/of.h>

//驱动错误输出
dev_warn() dev_dbg() dev_err()
dev_warn(&pdev->dev, "Device node %s has missing or invalid "
                       "cell-index property. Using 0.\n", pdev->dev.of_node->full_name);
// 驱动申请内存
platdata = devm_kzalloc(&pdev->dev,sizeof(*platdata), GFP_KERNEL);
```

## platform相关驱动
```c
struct platform_device {
    const char * name;
    int id;
    struct device dev;
    u32 num_resources;
    struct resource * resource;
    struct platform_device_id *id_entry;
    /* arch specific additions */
    struct pdev_archdata archdata;
};
struct platform_driver {
    int (*probe)(struct platform_device *);
    int (*remove)(struct platform_device *);
    void (*shutdown)(struct platform_device *);
    int (*suspend)(struct platform_device *, pm_message_t state);
    int (*resume)(struct platform_device *);
    struct device_driver driver;
    struct platform_device_id *id_table;
};
```


## cpld_probe
```c
static int __devinit cpld_probe(struct platform_device *pdev)
    //获得cpld序号, 参见dts
    indexp = of_get_property(pdev->dev.of_node, "cell-index", &len); 
    index = be32_to_cpup(indexp);
    //申请数据结构内存,
    platdata = devm_kzalloc(&pdev->dev,sizeof(*platdata), GFP_KERNEL);
    uioinfo_nmi = devm_kzalloc(&pdev->dev,sizeof(*uioinfo_nmi), GFP_KERNEL);
    uioinfo_com = devm_kzalloc(&pdev->dev,sizeof(*uioinfo_com), GFP_KERNEL);
    uioinfo_fqm = devm_kzalloc(&pdev->dev,sizeof(*uioinfo_fqm), GFP_KERNEL);
    //申请name内存
    name = devm_kzalloc(&pdev->dev,UIO_NAME_SIZE, GFP_KERNEL);
    name_nmi = devm_kzalloc(&pdev->dev,UIO_NAME_SIZE, GFP_KERNEL);
    name_fqm = devm_kzalloc(&pdev->dev,UIO_NAME_SIZE, GFP_KERNEL);
    //填结构体
    uioinfo_com->name = "cpld0";
    uioinfo_com->version = "0.01";
    uioinfo_com->irq = UIO_IRQ_NONE;
    //memory map, 是给usr看的吗?
    of_address_to_resource(pdev->dev.of_node, 0, &res);
    struct uio_mem *uiomem = &uioinfo_com->mem[0];
    uiomem->name = "cpld memory map";
    uiomem->memtype = UIO_MEM_PHYS;
    uiomem->addr = res.start;
    uiomem->size = res.end - res.start + 1;
    //内部调用ioremap, 映射到内核空间
    uiomem->internal_addr = of_iomap(pdev->dev.of_node, 0);
    //中断
    int irq = of_irq_to_resource(pdev->dev.of_node, cpld_com_irq, NULL);
    uioinfo_com->irq = irq;
    uioinfo_com->handler = cpld_com_irq_handler;
    uioinfo_com->irq_flags = IRQF_SHARED; /* Good practice to support sharing interrupt lines */
    //注册platform_device的私有成员, 就是本驱动的核心结构platdata
    platdata->uioinfo_nmi = uioinfo_nmi;
    platdata->uioinfo_com = uioinfo_com;
    platdata->uioinfo_fqm = uioinfo_fqm;
    platform_set_drvdata(pdev, platdata);
    //注册UIO
    uio_register_device(&pdev->dev, uioinfo_nmi);
    uio_register_device(&pdev->dev, uioinfo_com);
    uio_register_device(&pdev->dev, uioinfo_fqm);
```
```c
// 和dts对应的
static const struct of_device_id cpld_ids[] = {
    {
        .compatible = "alu,cpld",
    },
    {},
};
```

## platform_driver核心结构
```c
static struct platform_driver cpld_driver = {
    .driver = {
        .owner = THIS_MODULE,
        .name = "cpld",
        .of_match_table = cpld_ids,
    },
    .probe = cpld_probe,
    .remove = __devexit_p(cpld_remove),
};
```

# 关于uio
uio是个字符设备, 设备文件挂在devtmpfs下面, read和write是针对中断来说的. 而mmap用于对地址空间的访问.

## 注册
`uio_register_device(&pdev->dev, uioinfo_com)`
```c
uio_major = register_chrdev(0, "uio", &uio_fops)
static const struct file_operations uio_fops = {
    .owner = THIS_MODULE,
    .open = uio_open,
    .release = uio_release,
    .read = uio_read,
    .write = uio_write,
    .mmap = uio_mmap,
    .poll = uio_poll,
    .fasync = uio_fasync,
};
```

## uio_device
```c
struct uio_device {
    struct module *owner;
    struct device *dev;
    int minor;
    atomic_t event;
    struct fasync_struct *async_queue;
    wait_queue_head_t wait;
    int vma_count;
    struct uio_info *info;
    struct kobject *map_dir;
    struct kobject *portio_dir;
};
```

## uio info相关
```c
struct uio_info {
    struct uio_device *uio_dev;
    const char *name;
    const char *version;
    struct uio_mem mem[MAX_UIO_MAPS];
    struct uio_port port[MAX_UIO_PORT_REGIONS];
    long irq;
    unsigned long irq_flags;
    void *priv;
    irqreturn_t (*handler)(int irq, struct uio_info *dev_info);
    int (*mmap)(struct uio_info *info, struct vm_area_struct *vma);
    int (*open)(struct uio_info *info, struct inode *inode);
    int (*release)(struct uio_info *info, struct inode *inode);
    int (*irqcontrol)(struct uio_info *info, s32 irq_on);
};


struct uio_mem {
    const char *name;
    phys_addr_t addr;
    unsigned long size;
    int memtype;
    void __iomem *internal_addr;
    struct uio_map *map;
};
```

## resource 结构体
```c
struct resource {
    resource_size_t start;
    resource_size_t end;
    const char *name;
    unsigned long flags;
    struct resource *parent, *sibling, *child;
};
```

## uio内核操作
```c
static int uio_open(struct inode *inode, struct file *filep)
    //调用device自己的open
    ret = idev->info->open(idev->info, inode);
 
// 这个read阻塞的. 读出中断个数?
static ssize_t uio_read(struct file *filep, char __user *buf, size_t count, loff_t *ppos)
    DECLARE_WAITQUEUE(wait, current);
    add_wait_queue(&idev->wait, &wait);
    do while 1 循环:
        set_current_state(TASK_INTERRUPTIBLE);
        if 某个条件 //先从idev->event里面读出evnet个数, 再拷贝给用户空间的buffer
            copy_to_user(buf, &event_count, count)
        schedule()
    __set_current_state(TASK_RUNNING)
    remove_wait_queue(&idev->wait, &wait)

//read用了waitqueue, 那么肯定有地方调用wake_up函数, 见下:
static irqreturn_t uio_interrupt(int irq, void *dev_id)
    //调用device自己的handler
    ret = idev->info->handler(irq, idev->info)
    uio_event_notify(idev->info) //对atomic_unchecked_t event这个变量增1
        wake_up_interruptible(&idev->wait)
 
static unsigned int uio_poll(struct file *filep, poll_table *wait)
    poll_wait(filep, &idev->wait, wait)
        poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
            p->_qproc(filp, wait_address, p)
    if (listener->event_count != atomic_read_unchecked(&idev->event))
        return POLLIN | POLLRDNORM;
    return 0
 
static int uio_mmap(struct file *filep, struct vm_area_struct *vma)
    case UIO_MEM_PHYS:
        uio_mmap_physical(vma)
            remap_pfn_range()
```

## 用户态程序读是读中断, 读写uio用mmap
上面的uio_interrupt函数, 被注册到irq上, irq号由dts得来

`request_irq(info->irq, uio_interrupt, info->irq_flags, info->name, idev);`

`uio_interrupt()`会调用uio设备注册时提供的handler(), 只有该handler()返回`IRQ_HANDLED`时, 才会更新uio文件, 此时用户态会发现这个uio文件可读, 有中断来了.

所以, uio中断有两步, 先是uio.ko的内核态中断, 然后是uio用户态从uio设备文件"读"到的中断.

## uio注册设备, 在cpld_probe里面有注册uio设备
```c
uio_register_device(struct device *parent, struct uio_info *info)
    __uio_register_device(THIS_MODULE, parent, info)
        struct uio_device *idev
        init_uio_class()
        //申请uio_device结点
        idev = kzalloc(sizeof(*idev), GFP_KERNEL)
        idev->owner = owner;
        idev->info = info;
        init_waitqueue_head(&idev->wait);
        atomic_set(&idev->event, 0);
        //申请新的minor号
        uio_get_minor(idev)
        //创建设备文件
        idev->dev = device_create(uio_class->class, parent,
                  MKDEV(uio_major, idev->minor), idev,
                  "uio%d", idev->minor)
            dev = kzalloc(sizeof(*dev), GFP_KERNEL)
            //一些初始化结构变量的填写
            //创建"dev" 文件
            device_register(dev)
        uio_dev_add_attributes(idev)
        info->uio_dev = idev
        if (idev->info->irq >= 0)
            request_irq(idev->info->irq, uio_interrupt,
                  idev->info->irq_flags, idev->info->name, idev)
```
uio设备注册的时候, 有个irq_handler, 在中断上下文中执行的.


## 用户态uio lib
提供了从名字找info, 以及用户态open等操作(open时自动mmap)

### 用户态使用uio
```c
//获得设备首地址
void *uiodrv_get_base(DeviceNbr dev)
//真正的首地址是从mmap而来, 详见libuio.c
void *uio_mmap(struct uio_info_t* info, int map_num)
//drv_cpld.c
uio_hww_cpld_early_init(void)
    for (i = 0; i < cpld_num; i++)
        uiodrv_register_device(DEVICE_CPLD0 + i, "cpld", i)
        *(cpld_base + i) = (unsigned long)uiodrv_get_base(DEVICE_CPLD0 + i)
    _cpld_board_specific_early_init(cpld_base)
        //cpld为全局变量, 是个结构体指针, 定义在板子相关的bsp目录下
        cpld = (CLIP_CPLD *)(*(base + clip_cpld0))
        clip_cpld_init((void *)cpld)
//以后访问cpld就通过全局结构体指针加偏移量来访问.
```
在uio hww初始化时, 会起一个进程, 处理uio中断

```c
uiodrv_start_interrupt_task(253)
    xt_create ("UIOI", prio, 0x4000, 0, 0, &task_id);
    xt_start (task_id, T_PREEMPT | T_NOASR | T_TSLICE, interrupt_task_body, args);
        while (1)
            uiodrv_process_devices();
                for each dev:
                    中断速率限制, 在currSet中, FD_CLR中断太频繁的dev
                do {
                    ret = select(maxSocket + 1, &currSet, NULL, NULL, timeout);
                    //在select收到EINTR时重试
                } while ((ret < 0) && (errno == EINTR));
                
                for each dev:
                    如果该dev->fd在select列表内, 则handle=1
                    uiodrv_handle_fd(inst, timestamp, handle);
                        read(inst->fd)
                        inst->handler(inst->device_nr, inst->user_arg);
                        //向这个文件fd写0是禁止中断, 写1是使能中断
                        如果设置了auto unmask标记, 则向fd写1
```

## UIO总结:
在cpld内核驱动probe里面创建UIO设备, 此时在`/dev`下面会有`uio%d`的字符设备;

在用户态open的时候mmap, 并保存在`info->maps [i].map`, 后可通过`uio_get_mem_map()`获得

# 平台设备初始化
```c
cpld_init(void)
    platform_driver_register(&cpld_driver);
```
# 设备驱动如何注册的?
```c
int platform_driver_register(struct platform_driver *drv)
    drv->driver.bus = &platform_bus_type;
    drv->driver.probe = platform_drv_probe;
    drv->driver.remove = platform_drv_remove;
    drv->driver.shutdown = platform_drv_shutdown;
    //这里注册用的是device_driver, 后面可以用to_platform_driver(drv)把platform_driver找回来
    driver_register(&drv->driver);
        bus_add_driver(drv);
            driver_attach(drv)
                bus_for_each_dev(drv->bus, NULL, drv, __driver_attach);
                //以下是__driver_attach干的活, 优先从fdt里面match, 然后依次是ACPI, id table, name
                platform_match(struct device *dev, struct device_driver *drv)
                //如果match
                driver_probe_device(drv, dev)
                    really_probe(dev, drv)
                        //暂时把dev的driver指针置为当前drv
                        dev->driver = drv
                        driver_sysfs_add(dev)
                        //platform的bus没有probe方法, 不调用dev->bus->probe(dev)
                        drv->probe(dev) //在这里就是platform_drv_probe(struct device *_dev)
                            //把platform的drv和dev通过指针偏移找回来
                            drv = to_platform_driver(_dev->driver);
                            dev = to_platform_device(_dev);
                            //调用具体drv的probe, 比如CPLD, cpld_probe(struct platform_device *pdev)
                            drv->probe(dev);
                                //执行过程见上面的cpld_probe()
                        driver_bound(dev)
                            //把dev的p->knode_driver加到drv的p->klist_devices链表尾部
                            klist_add_tail(&dev->p->knode_driver, &dev->driver->p->klist_devices);
```

## 现在的问题是, dev哪里来的?
可能是通过调用`of_device_add(dev)`, 在解析dtb时得来.

# 关于device_add和sysfs
`device_add()`就会生成`sysfs`和`/dev`下面的文件


## 很多地方都会调device_add(), 比如:
```c
device_register()
    device_initialize(dev);
    device_add(dev);
pci_bus_add_device()
of_device_add()
    dev_name(&ofdev->dev)
    set_dev_node()
    device_add(&ofdev->dev)
platform_device_add(struct platform_device *pdev)
    pdev->dev.parent = &platform_bus
    pdev->dev.bus = &platform_bus_type
    device_add(&pdev->dev)
spi_add_device()
```

## 那么device_add()干了什么?
```c
device_private_init(dev)
  setup_parent(dev, parent)
  //加到/sys?--yes
  kobject_add(&dev->kobj, dev->kobj.parent, NULL)
  platform_notify(dev)
  device_create_file(dev, &uevent_attr)
  if (MAJOR(dev->devt))
      device_create_file(dev, &devt_attr)
      device_create_sys_dev_entry(dev)
      //通过devtmpfs创建设备结点
      devtmpfs_create_node(dev)
          //查找/dev?
          vfs_path_lookup(dev_mnt->mnt_root, dev_mnt, nodename, LOOKUP_PARENT, &nd)
          dentry = lookup_create(&nd, 0)
          //mknod???
          vfs_mknod(nd.path.dentry->d_inode, dentry, mode, dev->devt)
  device_add_class_symlinks(dev)
  device_add_attrs(dev)
  //把dev加到bus的dev链表里
  bus_add_device(dev)
  dpm_sysfs_add(dev)
  device_pm_add(dev)
  blocking_notifier_call_chain()
  //通知用户态有ADD事件, mdev会处理, 并创建设备文件
  kobject_uevent(&dev->kobj, KOBJ_ADD)
  bus_probe_device(dev)
      device_attach(dev)
          if (dev->driver)
              device_bind_driver(dev)
          else
              //遍历驱动
              bus_for_each_drv(dev->bus, NULL, dev, __device_attach)
                  driver_probe_device(drv, dev)
```


## octeon-platform.c
```c
static struct of_device_id __initdata octeon_ids[] = {
    { .compatible = "simple-bus", },
    { .compatible = "cavium,octeon-6335-uctl", },
    { .compatible = "cavium,octeon-3860-bootbus", },
    { .compatible = "cavium,mdio-mux", },
    { .compatible = "gpio-leds", },
    {},
};
```

## dev都是在这里生成的?
```c
device_initcall(octeon_publish_devices)
    octeon_publish_devices()
        of_platform_bus_probe(NULL, octeon_ids, NULL)
            of_platform_bus_create(child, matches, parent)
                of_platform_device_create(bus, NULL, parent);
                for_each_child_of_node(bus, child)
                    of_platform_bus_create(child, matches, &dev->dev)
                        of_device_add(dev)
```

# 驱动相关
## driver的初始化位置
```c
start_kernel
    ...最后
    rest_init()
        kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND)
            //新内核线程
            do_basic_setup()
                init_workqueues();
                cpuset_init_smp();
                usermodehelper_init();
                init_tmpfs();
                driver_init();
                    devtmpfs_init();
                    devices_init();
                    buses_init();
                    classes_init();
                    firmware_init();
                    hypervisor_init();
                    platform_bus_init();
                    system_bus_init();
                    cpu_dev_init();
                    memory_dev_init();
                init_irq_proc();
                do_ctors();
                do_initcalls();
        cpu_idle()
```

i2c驱动 spi驱动  platform驱动都会调用driver_register()

## platform驱动注册流程

`kernel_init()`中`do_basic_setup()->driver_init()->platform_bus_init()->...`初始化platform bus(虚拟总线)

设备向内核注册的时候`platform_device_register()->platform_device_add()->...`内核把设备挂在虚拟的platform bus下

驱动注册的时候`platform_driver_register()->driver_register()->bus_add_driver()->driver_attach()->bus_for_each_dev()`

对每个挂在虚拟的platform bus的设备做`__driver_attach()->driver_probe_device()->drv->bus->match()==platform_match()`比较`strncmp(pdev->name, drv->name, BUS_ID_SIZE)`，如果相符就调用`platform_drv_probe()->driver->probe()`，如果probe成功则绑定该设备到该驱动.


## I2C是个字符设备
```c
ssize_t i2cdev_write (struct file *file, const char __user *buf, size_t count, loff_t *offset)
    struct i2c_client *client = (struct i2c_client *)file->private_data
    copy_from_user()
    i2c_master_send(client,tmp,count)
        struct i2c_adapter *adap=client->adapter;
        struct i2c_msg msg;
        msg.addr = client->addr;
        msg.flags = client->flags & I2C_M_TEN;
        msg.len = count;
        msg.buf = (char *)buf;
        i2c_transfer(adap, &msg, 1)
            adap->algo->master_xfer(adap, msgs, num)
```
```c
static const struct file_operations i2cdev_fops = {
    .owner = THIS_MODULE,
    .llseek = no_llseek,
    .read = i2cdev_read,
    .write = i2cdev_write,
    .unlocked_ioctl = i2cdev_ioctl,
    .open = i2cdev_open,
    .release = i2cdev_release,
};
 
static struct i2c_driver i2cdev_driver = {
    .driver = {
        .name = "dev_driver",
    },
    .attach_adapter = i2cdev_attach_adapter,
    .detach_adapter = i2cdev_detach_adapter,
};
```


### i2c设备初始化
```c
i2c_dev_init(void)
    register_chrdev(I2C_MAJOR, "i2c", &i2cdev_fops)
    i2c_dev_class = class_create(THIS_MODULE, "i2c-dev");
    i2c_add_driver(&i2cdev_driver);
        i2c_register_driver(THIS_MODULE, driver)
            driver->driver.bus = &i2c_bus_type;
            driver_register(&driver->driver);
            bus_for_each_dev(&i2c_bus_type, NULL, driver, __attach_adapter);
```

### 很多地方都会调用i2c_add_driver()
```c
i2c_init(void)
    i2c_add_driver(&dummy_driver);
i2c_dev_init(void)
    i2c_add_driver(&i2cdev_driver);
tmp421_init(void)
    i2c_add_driver(&tmp421_driver);
eeprom_init(void)
    i2c_add_driver(&eeprom_driver);
```

# 关于device_create
## 原型及主要流程
```c
struct device *device_create(struct class *class, struct device *parent, dev_t devt, void *drvdata, const char *fmt, ...)
    device_register(dev)
        device_initialize(dev)
            kobject kset 互斥锁 自旋锁 链表初始化
        device_add(dev)
            ...
            //sysfs添加文件
            device_create_file(dev, &uevent_attr);
            device_create_file(dev, &devt_attr)
            device_create_sys_dev_entry(dev);
            devtmpfs_create_node(dev);
                init_completion()
                // 向内核进程devtmpfsd发送request, 创建设备结点
                wake_up_process(devtmpfsd)
                wait_for_completion()
            device_add_class_symlinks(dev);
            device_add_attrs(dev);
            bus_add_device(dev);
            dpm_sysfs_add(dev);
            device_pm_add(dev);
            blocking_notifier_call_chain(&dev->bus->p->bus_notifier, BUS_NOTIFY_ADD_DEVICE, dev);
            kobject_uevent(&dev->kobj, KOBJ_ADD);
            bus_probe_device(dev);
```

## i2c中的使用例子
```c
i2c_dev->dev = device_create(i2c_dev_class, &adap->dev,
                 MKDEV(I2C_MAJOR, adap->nr), NULL,
                 "i2c-%d", adap->nr);
```

## uio中的使用例子
```c
idev->dev = device_create(&uio_class, parent,
              MKDEV(uio_major, idev->minor), idev,
              "uio%d", idev->minor);
```

# 如何创建设备?
有两种方法:

## 方法1: 在父节点调用
```c
of_platform_bus_probe(pdev->dev.of_node, cpld_child_ids, &pdev->dev)
```
这个函数是在cpld probe阶段调用的, 会根据cpld_child_ids来匹配其子节点, 并为匹配上的节点及其子节点创建device.
```c
static struct of_device_id cpld_child_ids[] = {
	{ .compatible = "cpld-leds", },
	{},
};
```
创建好device以后, 对应的driver的probe函数才能被调用.

比如在cpld这一级调用`of_platform_bus_probe()`, 那么cpld的子节点leds的driver才能找到leds的device

在leds的driver里面调用  
```c
led_classdev_register(&pdev->dev, &led_dat->cdev)
```
来创建led.

成功后在`/sys/class/leds`里能看到led.

## 方法2: 直接在dts里声明
把simple-bus加进cpld的compatible 
```
compatible = "alu,cpld", "simple-bus";
```
原理:

还是在`arch/mips/cavium-octeon/octeon-platform.c`里面  
有个`octeon_ids`列表, 里面有
```c
static struct of_device_id __initdata octeon_ids[] = { 
    { .compatible = "simple-bus", },
    { .compatible = "cavium,octeon-6335-uctl", },
    { .compatible = "cavium,octeon-5750-usbn", },
    { .compatible = "cavium,octeon-3860-bootbus", },
    { .compatible = "cavium,mdio-mux", },
    { .compatible = "gpio-leds", },
    { .compatible = "cavium,octeon-7130-usb-uctl", },
    { .compatible = "cavium,octeon-7130-sata-uctl", },
    {}, 
};
```
并在`device_initcall`的时候调用, 负责创建全部的of_device
```c
static int __init octeon_publish_devices(void)
{
    return of_platform_bus_probe(NULL, octeon_ids, NULL);
}
device_initcall(octeon_publish_devices);
```

# mips kernel
## mips 64bit空间
见`arch/mips/include/asm/mach-cavium-octeon/spaces.h`
```c
#ifdef CONFIG_64BIT
/* They are all the same and some OCTEON II cores cannot handle 0xa8.. */
#define CAC_BASE        _AC(0x8000000000000000, UL)
#define UNCAC_BASE      _AC(0x8000000000000000, UL)
#define IO_BASE         _AC(0x8000000000000000, UL)
#endif /* CONFIG_64BIT */
```

## mips ioremap
```c
/*
 * ioremap     -   map bus memory into CPU space
 * @offset:    bus address of the memory
 * @size:      size of the resource to map
 *
 * ioremap performs a platform specific sequence of operations to
 * make bus memory CPU accessible via the readb/readw/readl/writeb/
 * writew/writel functions and the other mmio helpers. The returned
 * address is not guaranteed to be usable directly as a virtual
 * address.
 */
#define ioremap(offset, size)                       \
    __ioremap_mode((offset), (size), _CACHE_UNCACHED)
        plat_ioremap(offset, size, flags)
            base = (u64) IO_BASE
            return (void __iomem *) (unsigned long) (base + offset)
```

综上: 相当于把物理地址最高位或上1
使用时  
```c
flash_map.virt = ioremap(flash_map.phys, flash_map.size);
```

## ioread8的入参应该是个CPU地址, 而不是物理地址
```c
ioread8()
    readb(addr)
        return *(const volatile u8 __force *) addr
```

## 编译时的条件检测，条件为真则导致编译错误
```c
#include linux/kernel.h
BUILD_BUG_ON(sizeof(struct lock_class_key) > sizeof(struct lockdep_map));
```
原理：
```c
#define BUILD_BUG_ON(condition) ((void)sizeof(char[1 - 2*!!(condition)]))
```

## 运行时的条件检测，条件为真则触发运行时exception
```c
#include asm/bug.h
BUG_ON(i != pos);
```
或直接调用`BUG()`

原理：利用mips断点和自陷指令break和tne做运行时检测
```c
__asm__ __volatile__("tne $0, %0, %1"
                 : : "r" (condition), "i" (BRK_BUG));
__asm__ __volatile__("break %0" : : "i" (BRK_BUG));
```

## jiffies jiffies_64和HZ
在jiffies.h中有 jiffies和jiffies_64声明
```c
extern u64 __jiffy_data jiffies_64;
extern unsigned long volatile __jiffy_data jiffies;
```
linux中，用jiffies记录系统启动以来的时钟滴答数。  
HZ是一秒钟的时钟滴答数，被定义为CONFIG_HZ，这是menuconfig时指定的值。
所以，HZ为100时，jiffies一秒钟增大100，32位机下，497天溢出。

为了防止jiffies溢出导致问题，linux定义了jiffies_64，这样能保证几百年都不溢出。

`jiffies_64`是在`/kernel/timer.c`中定义的
```
u64 jiffies_64 __cacheline_aligned_in_smp = INITIAL_JIFFIES;
```

而jiffies并没有定义，只有个声明。但代码里却大量使用它，那么这个变量到底在哪里呢？

在链接脚本`vmlinux.lds.S`中，有
```
jiffies = jiffies_64 + 4;
```
注意，链接脚本的`=`号并不是赋值，而是"赋地址"，在这里，`jiffies`的地址是`jiffies_64`地址向后偏移`4`字节，在大端CPU上，就是`jiffies_64`的低32位。
所以，`jiffies`就是`jiffies_64`的低32位，几乎可以认为是同一个变量。

每次时钟中断到来，会调用do_timer函数，jiffies就会增加了。
```c
void do_timer(unsigned long ticks)
{
    jiffies_64 += ticks;
    update_times(ticks);
}
```

与jiffies有关的宏：用于确定时间关系，常和另外一个宏`cpu_relax()`一起，用于忙等待。
```c
time_after(a,b)
time_before(a,b)
```

## local_irq_disable()  关中断
关中断，并保存SR到flags
```c
#include linux/irqflags.h
local_irq_save(flags)
```

原理："di"，汇编指令，关中断，相当于`SR(IE) = 0` 

## current 表示当前进程
mips用`$28(gp)`保存当前进程的`thread_info`  
而current宏返回当前进程的进程描述符
```c
current = $28->task 
```
代码：
```c
register struct thread_info *__current_thread_info __asm__("$28");
#define current_thread_info()  __current_thread_info

static inline struct task_struct * get_current(void)
{
    return current_thread_info()->task;
}

#define current        get_current()
```

## preempt_disable() 禁止抢占
```c
#include linux/preempt.h
```

原理：每个thread都有个`preempt_count`，只有为0的时候才能抢占
```c
current_thread_info()->preempt_count += 1
```

## 自旋锁spin_lock()
```c
#include linux/spinlock.h
```
原型：
```c
typedef struct {
    raw_spinlock_t raw_lock;
} spinlock_t;
```
定义
```c
static DEFINE_SPINLOCK(logbuf_lock); //默认初值为unlock
```
初始化
```c
spin_lock_init(&logbuf_lock); //初始化为unlock
```

使用时配对
```c
spin_lock()/spin_unlock()                      //禁止抢占，用于只和其他cpu互斥
spin_lock_irq()/spin_unlock_irq()，            //禁止抢占，关中断，用于和其他cpu互斥+和中断互斥
spin_lock_irqsave/spin_unlock_irqrestore()     //禁止抢占，关中断，并保存SR寄存器，用于和其他cpu互斥+和中断互斥
spin_lock_bh()/spin_unlock_bh()                //禁止抢占，关下半部，用于和其他cpu互斥+和bh互斥
```

原理：  
linux提供spin_lock用于SMP环境下保护临界区，换句话说，用于多CPU互斥。在没有配置CONFIG_SMP时，没有真正的锁操作，只是调用了`preempt_disable()`。
SMP时，`spin_lock()`会先调用`preempt_disable()`来禁止抢占，最后会调用`__raw_spin_lock(raw_spinlock_t *lock)`来完成真正的获取锁操作。

注意：
* spin_lock()只是禁止抢占，并没有关中断
* 持有锁期间不能主动让出cpu
* 锁被持有时, 持有者不允许再次尝试获取该锁.
* 在必须获取多个锁时, 始终以相同的顺序获得.

## ARRAY_SIZE(arr)
```c
#include linux/kernel.h
```

# 各种内核编译宏, 见compiler.h
```c
#ifdef __CHECKER__
# define __user         __attribute__((noderef, address_space(1)))
# define __force_user   __force __user
# define __kernel       __attribute__((address_space(0)))
# define __force_kernel __force __kernel
# define __safe         __attribute__((safe))
# define __force        __attribute__((force))
# define __nocast       __attribute__((nocast))
# define __iomem        __attribute__((noderef, address_space(2)))
# define __force_iomem  __force __iomem
# define __must_hold(x) __attribute__((context(x,1,1)))
# define __acquires(x)  __attribute__((context(x,0,1)))
# define __releases(x)  __attribute__((context(x,1,0)))
# define __acquire(x)   __context__(x,1)
# define __release(x)   __context__(x,-1)
# define __cond_lock(x,c)   ((c) ? ({ __acquire(x); 1; }) : 0)
# define __percpu       __attribute__((noderef, address_space(3)))
# define __force_percpu __force __percpu
#ifdef CONFIG_SPARSE_RCU_POINTER
# define __rcu          __attribute__((noderef, address_space(4)))
# define __force_rcu    __force __rcu
#else
# define __rcu
# define __force_rcu
#endif
```

# dump_stack() 驱动打印调用栈
比如, `ubifs在ubi_io_read()`中, 调用底层读出错时, 
```c
ubi_err("error %d%s while reading %d bytes from PEB %d:%d, read %zd bytes",
    err, errstr, len, pnum, offset, read);
dump_stack();
```

# linux中断

## 中断注册request_threaded_irq()
```c
//这个函数用来由硬件中断号获取linux irq号.
root->irq = irq_create_mapping(NULL, root->hwint)
rv = request_threaded_irq(root->irq, NULL,octeon_hw_status_irq, IRQF_ONESHOT,"octeon-hw-status", root)
/*
 * request_threaded_irq()用来注册中断.
 * 第2个参数为handler, 在中断上下文执行, 第3个参数为thread_fn, 在内核线程执行, 相当于下半部.
 * 如果这两个参数同时存在, 则在do_IRQ()中, 先调用handler(), 然后根据handler()的返回值, 判断是否再起内核进程调用thread_fn()
 * handler()可能为, IRQ_WAKE_THREAD或IRQ_HANDLED.
 * 如果入参handler为NULL, 则默认用 irq_default_primary_handler(), 这个函数直接return IRQ_WAKE_THREAD
 *
 * thread_fn()是在内核进程执行的, 在request_threaded_irq()里面, 会创建内核进程
 */
t = kthread_create(irq_thread, new, "irq/%d-%s", irq, new->name);
```

## 关于中断号
中断号是个软件概念, 一般都是芯片相关的, 不同的芯片中断号也不一样.
```c
//下面的函数用来从硬件中断号找linux中断号
unsigned int irq_find_mapping(struct irq_domain *domain,irq_hw_number_t hwirq)
 
//只有linux irq号是不够的, 这时就需要转为irq描述符
struct irq_desc *desc = irq_to_desc(irq)
struct irq_data *irq_data = irq_desc_get_irq_data(desc);
```

# 信号量
```c
#include <linux/semaphore.h>
// 定义信号量
static DEFINE_SEMAPHORE(hwstat_sem);

// 获取信号量
down(&hwstat_sem);

// 释放信号量
up(&hwstat_sem);
```

# debugfs 在/sys/kernel/debug/创建文件

* debugfs有宏开关 #ifdef CONFIG_DEBUG_FS
* debugfs一般在/sys/kernel/debug/
* 创建一个"文件", 需要传入文件操作集,支持.open .read .llseek .release  
`debugfs_create_file("hwstat", S_IFREG | S_IRUGO, NULL, NULL, &hwstat_operations);`

# 关于notify_chain
内核用notify机制来提供注册-通知机制, 内部用带优先级的链表实现.
有四种notify方式, 下面以raw方式为例
1. 先声明链表头, 以后的操作都通过这个链表头来访问.  
`static RAW_NOTIFIER_HEAD(octeon_hw_status_notifiers);`
2. 在初始化或者适当的地方注册一个call-back, 或者叫notifier_block  
`raw_notifier_chain_register(&octeon_hw_status_notifiers, nb);`  
在实现这个nb时, 如果返回NOTIFY_STOP, 则链表后面的nb都不执行.  
比如`static int octeon_error_tree_hw_status(struct notifier_block *nb, unsigned long val, void *v)`  
先判断事件类型, 这个类型一般保存在val里面. 然后干活, 成功就返回`NOTIFY_DONE`, 失败返回`NOTIFY_STOP`
3. 在事件产生时, 调用这个链表, 一般后两个参数是事件类型和一个指针  
`raw_notifier_call_chain(&octeon_hw_status_notifiers,OCTEON_HW_STATUS_SOURCE_ASSERTED, &ohsd);`

# 创建proc的entry, 并绑定相关的文件操作
比如logbuffer_operations
```c
static const struct file_operations logbuffer_operations = {
    .open           = logbuffer_open,
    .read           = seq_read,
    .llseek         = seq_lseek,
    .release        = single_release,
};
//seq_read, single_open等函数在fs/seq_file.c

entry = create_proc_entry("logbuffer", 0, NULL);
    if (entry)
        entry->proc_fops = &logbuffer_operations;
```

# 在sys目录下创建个class
```c
reborn_class = class_create(THIS_MODULE, "reborn");
```

# 驱动中使用工作队列轮询
在驱动中使用工作队列过程如下:

## 在设备相关结构体内添加work
```c
struct platdata {
    struct uio_info *uioinfo_nmi;
    spinlock_t nmi_lock;
    unsigned long nmi_flags;
    struct uio_info *uioinfo_com;
    struct mtd_writeprotection_handler writeprotect_handler;
    void __iomem *cpld_base;
    struct delayed_work dwork;
};
```

## work的处理函数
在最后再次注册自身, 达到周期调用的效果. 这里使用内核的默认工作队列. HZ是1秒一次
```c
void cpld_dworker(struct work_struct *work)
{
    int i = 0;
    unsigned char val = 0;
    static unsigned char last_martini_status[2] = {0, 0};
    struct delayed_work *dwork = to_delayed_work(work);
    struct platdata *platdata = container_of(dwork, struct platdata, dwork);

    //printk("Get cpld base:0x%llx\n", (unsigned long long)platdata->cpld_base);

    for (i = 0; i < 2; i++) {
        if (i == 0)
            rd_cpld_field(platdata->cpld_base, MAR1_UP, &val);
        else
            rd_cpld_field(platdata->cpld_base, MAR2_UP, &val);

        if (val ^ last_martini_status[i]) {
            if (val) {
                printk("Martini%d status changed up. Opening SHPI bus.\n", i);
            } else {
                printk(KERN_ERR "Martini%d status changed down! Closing SHPI bus.\n", i);
            }

            if (i == 0)
                wr_cpld_field(platdata->cpld_base, MAR1_SHPI, val);
            else
                wr_cpld_field(platdata->cpld_base, MAR2_SHPI, val);

            last_martini_status[i] = val;
        }
    }
 
    schedule_delayed_work(dwork, HZ);
}
```

## 在初始化的时候开始工作队列
```c
cpld_probe
    /* Start a worker to check martini status */
    INIT_DELAYED_WORK(&platdata->dwork, cpld_dworker);
    schedule_delayed_work(&platdata->dwork, HZ);
```

## 在rmmod时删除这个work
```c
cpld_remove
    cancel_delayed_work(&platdata->dwork);
```
