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
```
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

