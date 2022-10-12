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


