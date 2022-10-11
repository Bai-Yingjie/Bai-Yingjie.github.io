在驱动中使用工作队列过程如下:

# 在设备相关结构体内添加work
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

# work的处理函数
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

# 在初始化的时候开始工作队列
```c
cpld_probe
    /* Start a worker to check martini status */
    INIT_DELAYED_WORK(&platdata->dwork, cpld_dworker);
    schedule_delayed_work(&platdata->dwork, HZ);
```

# 在rmmod时删除这个work
```c
cpld_remove
    cancel_delayed_work(&platdata->dwork);
```