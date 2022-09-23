# 背景
有的板子kernel启动的时候有ecc错, kernel里面有个polling任务会打印
```
EDAC MC0: 1 CE DIMM 1 rank 1 bank 4 row 65056 col 1020 on any memory ( page:0x0 offset:0x0 grain:0 syndrome:0x0)
```
一个workaround思路是, 检测到ecc错的时候重启板子. 因为这个ecc错和重启相关.

# 尝试1
在init脚本里面检测
```
while true; do
    key="CE DIMM"
    if `dmesg | grep "$key" >/dev/null` ; then
        echo "i find:  $key"
        do_reboot "CE reset"
    fi
    key="UE DIMM"
    if `dmesg | grep "$key" >/dev/null` ; then
        echo "i find:  $key"
        do_reboot "UE reset"
    fi    

    sleep 1
done
```
这样做的问题是太晚了, 有的时候这个脚本根本没跑到kernel就打calltrace挂住了.

# 尝试2
在那个polling任务里监测, 还是太晚了. 这个任务可能都还没来得及运行, kernel就挂住了.

# 终极方案
挂中断  
这里先把代码放上来
在`~/repo/hg/xrepo/OCTEON-SDK-3.1/linux/kernel/linux/drivers/edac/octeon_edac-lmc.c`
最后加
```c
#if 1
#include <linux/interrupt.h>
static irqreturn_t interrupt_octeon_lmc(int irq, void *dev_id)
{
    cvmx_ciu_cib_lmcx_rawx_t cib_lmc_raw;
    union cvmx_lmcx_int int_reg;
    char msg[64];
    printk("%s:%d irq=%d", __FUNCTION__, __LINE__, irq);
    cib_lmc_raw.u64 = cvmx_read_csr(CVMX_CIU_CIB_LMCX_RAWX(0,0));
    //clear the source
    cvmx_write_csr(CVMX_CIU_CIB_LMCX_RAWX(0,0), cib_lmc_raw.u64);
    int_reg.u64 = cvmx_read_csr(CVMX_LMCX_INT(0));
    printk("DDR ecc error! LMC0_INT=0x%llx", int_reg.u64);
    if (int_reg.s.sec_err || int_reg.s.ded_err) {
        union cvmx_lmcx_fadr fadr;
        fadr.u64 = cvmx_read_csr(CVMX_LMCX_FADR(0));
        snprintf(msg, sizeof(msg),
             "DIMM %d rank %d bank %d row %d col %d",
             fadr.cn61xx.fdimm, fadr.cn61xx.fbunk,
             fadr.cn61xx.fbank, fadr.cn61xx.frow, fadr.cn61xx.fcol);
    }
    if (int_reg.s.sec_err) {
        printk("single error detected: %s", msg);
        int_reg.s.sec_err = -1; /* Done, re-arm */
    }
    if (int_reg.s.ded_err) {
        printk("double error detected: %s", msg);
        int_reg.s.ded_err = -1; /* Done, re-arm */
    }
    if (int_reg.s.nxm_wr_err) {
        snprintf(msg, sizeof(msg), "NXM_WR_ERR: Write to non-existent memory");
        printk("nxm error detected: %s", msg);
        int_reg.s.nxm_wr_err = -1;  /* Done, re-arm */
    }
    //clear INT
    cvmx_write_csr(CVMX_LMCX_INT(0), int_reg.u64);
    return IRQ_HANDLED;
}
static int octeon_setup_lmc_interrupt(void)
{
    unsigned int hwirq;
    int irq;
    cvmx_ciu_cib_lmcx_enx_t cib_lmc_en;
    if (!OCTEON_IS_MODEL(OCTEON_CN70XX))
        return 0;
    printk("ECC workaround for cn70xx!");
    hwirq = (1 << 6) + 52;
    irq = irq_create_mapping(NULL, hwirq);
    printk("map irq=%d for ecc error", irq);
    if (!irq)
        return -EINVAL;
    if (request_irq(irq, interrupt_octeon_lmc,
            0, "lmc-ecc-error", interrupt_octeon_lmc)) {
        panic("request_irq(%d) failed.", irq);
    }
    cib_lmc_en.u64 = 0;
    cib_lmc_en.s.int_ded_errx = -1;
    cib_lmc_en.s.int_sec_errx = -1;
    cib_lmc_en.s.int_nxm_wr_err = -1;
    //enable the interrupt
    cvmx_write_csr(CVMX_CIU_CIB_LMCX_ENX(0,0), cib_lmc_en.u64);
    printk("attach ecc handler done!");
    return 0;
}
core_initcall(octeon_setup_lmc_interrupt);
#endif
```
这里面有好几个点
* hwirq是硬件中断号, 和具体的中断控制器(可以认为一个中断控制器是一个domain)有关, 这个domain里面的map函数控制hwirq号到irq号的映射
* irq号是linux的一个概念, 对应这个irq的软件控制实例(也就是结构体)
* 要挂中断, 先要把hwirq map到irq, 这通过`irq_create_mapping()`实现, 这个函数会申请irq相关的结构体并返回irq号  
然后用request_irq()来挂中断处理函数, 最后一个参数会被传到handler里面
* handler里面一般要先清中断, 可以调printk打印(为什么?)
* core_initcall()是个系列函数中的一个, 可以在kernel启动的时候挂初始化函数, 位置比较靠前.

# 启动打印
```
03-19 06:56:05  [    0.000000] NR_IRQS:598
03-19 06:56:05  [   34.196761] Calibrating delay loop (skipped) preset value.. 2400.00 BogoMIPS (lpj=12000000)
03-19 06:56:05  [   34.204941] pid_max: default: 32768 minimum: 501
03-19 06:56:05  [   34.209775] Mount-cache hash table entries: 256
03-19 06:56:05  [   34.217025] Checking for the daddi bug... no.
03-19 06:56:05  [   34.221223] ftrace: allocating 14160 entries in 56 pages
03-19 06:56:05  [   34.248215] Performance counters: octeon PMU enabled, 4 64-bit counters available to each CPU, irq 7
03-19 06:56:05  [   34.308272] SMP: Booting CPU01 (CoreId  1)...
03-19 06:56:05  [   34.312471] CPU revision is: 000d9602 (Cavium Octeon III)
03-19 06:56:05  [   34.312475] FPU revision is: 00739600
03-19 06:56:05  [   34.312743] Brought up 2 CPUs
03-19 06:56:05  [   34.325172] devtmpfs: initialized
03-19 06:56:05  [   34.330228] ECC workaround for cn70xx!
03-19 06:56:05  [   34.333820] map irq=122 for ecc error
03-19 06:56:05  [   34.337512] attach ecc handler done!
03-19 06:56:05  [   34.341020] interrupt_octeon_lmc:229 irq=122
03-19 06:56:05  [   34.345272] DDR ecc error! LMC0_INT=0x0
03-19 06:56:05  [   34.349093] LCM0_INT_EN=0x0 LCM0_INT=0x0
03-19 06:56:05  [   34.353002] LCM0_FADR=0x2000000 LCM0_ECC_SYND=0xc500
03-19 06:56:05  [   34.358113] NET: Registered protocol family 16
03-19 06:56:05  [   34.363522] Installing handlers for error tree at: ffffffff80727130
```

# 后记
这个挂中断的方案后来也被证明还是太晚了, 最后还是因为ddr的参数不对导致的.  
但是在linux下面挂一个中断处理函数, 大体如此.