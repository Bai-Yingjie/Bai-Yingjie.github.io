# 现象
```
02-10 01:10:24  [   81.726789] Kernel bug detected[#1]:
02-10 01:10:24  [   81.743065] CPU: 1 PID: 699 Comm: watchdog_monito Tainted: G           O 3.10.20-rt14-Cavium-Octeon #6
02-10 01:10:24  [   81.765061] task: 800000008c476af0 ti: 8000000088b28000 task.ti: 8000000088b28000
02-10 01:10:24  [   81.785229] $ 0   : 0000000000000000 ffffffff8054af84 0000000000000004 0000000000000008
02-10 01:10:24  [   81.869389] $ 4   : 0000000000000000 800000008c01ce88 00000000ffffff01 00000000ffffff02
02-10 01:10:24  [   81.953547] $ 8   : 800000008c01cea8 ffffffff80710000 0000000000000001 0000000000000000
02-10 01:10:24  [   82.037705] $12   : 8000000088b2bc98 800000008c211f18 0000000000000000 0000000000000000
02-10 01:10:24  [   82.121863] $16   : 800000008c20b340 800000008c07f000 0000000000000009 800000008c01dd80
02-10 01:10:24  [   82.206021] $20   : 00000000000080d0 0000000000000000 0000000000000009 ffffffff812c0000
02-10 01:10:24  [   82.290179] $24   : 00000000100c60c8 00000000774e4060                                 
02-10 01:10:25  [   82.374337] $28   : 8000000088b28000 8000000088b2bcb0 800000008c01ce80 ffffffff802698ac
02-10 01:10:25  [   82.458496] Hi    : 0000000000000001
02-10 01:10:25  [   82.474754] Lo    : 0000000000000000
02-10 01:10:25  [   82.491022] epc   : ffffffff8026997c cache_alloc_refill+0x174/0x7e0
02-10 01:10:25  [   82.509975]     Tainted: G           O
02-10 01:10:25  [   82.526410] ra    : ffffffff802698ac cache_alloc_refill+0xa4/0x7e0
02-10 01:10:25  [   82.545276] Status: 14009ce2 KX SX UX KERNEL EXL
02-10 01:10:25  [   82.638828] Cause : 00800034
02-10 01:10:25  [   82.654385] PrId  : 000d9602 (Cavium Octeon III)
02-10 01:10:25  [   82.671685]  Modules linked in: hxdrv(O) uio_generic_driver(O) led_cpld(O)  reborn_macfilter(O) logbuffer(O) fglt_b_reboot_helper(O) ramoops  reed_solomon fglt_b_cpld(O) reborn_class(O) generic_access(O)  spi_oak_island(O)
02-10 01:10:25  [   82.868798]  Process watchdog_monito (pid: 699, threadinfo=8000000088b28000,  task=800000008c476af0, tls=00000000776404a0)
02-10 01:10:25  [   82.892353] Stack : 0000000000000001 ffffffff8054abfc 800000008c073100 0000000000000000
02-10 01:10:25            800000008c211c90 ffffffff801a4e1c 800000008c01dd80 00000000000080d0
02-10 01:10:25            00000000000080d0 0000000000000001 ffffffff8016d260 00000000000080d0
02-10 01:10:25            0000000000000000 800000008c2874a0 0000000000000000 ffffffff8026974c
02-10 01:10:25            800000008c3fba00 0000000001200012 0000000000000000 800000008c3fba00
02-10 01:10:25            0000000000000000 ffffffff81230000 0000000077639068 ffffffff8016d260
02-10 01:10:25            fffffff48c073100 ffffffff80295cd8 0000000000000000 0000000000000000
02-10 01:10:26            800000008c287660 8000000088b18000 00000000000002bb 000000007f6b24e0
02-10 01:10:26            0000000001200012 0000000000000002 0000000000000000 0000000000000000
02-10 01:10:26            00000000100cb3e8 00000000100c0000 000000007f6b2500 ffffffff8016dae0
02-10 01:10:26            ...
02-10 01:10:26  [   83.630192] Call Trace:
02-10 01:10:26  [   83.645325] [<ffffffff8026997c>] cache_alloc_refill+0x174/0x7e0
02-10 01:10:26  [   83.663932] [<ffffffff8026974c>] kmem_cache_alloc+0x154/0x210
02-10 01:10:26  [   83.682371] [<ffffffff8016d260>] copy_process.part.53+0x760/0xea0
02-10 01:10:26  [   83.701153] [<ffffffff8016dae0>] do_fork+0xa8/0x2c8
02-10 01:10:26  [   83.718719] [<ffffffff80159a94>] handle_sys+0x134/0x160
02-10 01:10:26  [   83.736630]
02-10 01:10:26  [   83.750804]
02-10 01:10:26  Code: 8e640018  0044202b  2c8a0001 <000a0336> 10800023  26c4ffff  12c0006c  0080902d  0809a668
02-10 01:10:26  [   83.900378] [sched_delayed] sched: RT throttling activated
02-10 01:10:26  [   83.918593] ---[ end trace a91730027ab35f25 ]---
02-10 01:10:28  [   83.937180]  Fatal exception: panic in 5 seconds[   85.896096] EDAC MC0: 1 CE DIMM 1  rank 0 bank 4 row 63553 col 1436 on any memory ( page:0x0 offset:0x0  grain:0 syndrome:0x0)
02-10 01:10:28  [   85.920145]  EDAC MC0: 1 UE DIMM 1 rank 0 bank 4 row 63553 col 1436 on any memory (  page:0x0 offset:0x0 grain:0)
02-10 01:10:31 
02-10 01:10:31  [   88.965613] Kernel panic - not syncing: Fatal exception
02-10 01:10:31  [   88.997694] reboot_helper: stored panic_counter = 1
02-10 01:10:31  [   89.005599] [sched_delayed] process 1366 (TICK) no longer affine to cpu0
02-10 01:10:31  [   89.015598] [sched_delayed] process 1520 (shl1) no longer affine to cpu0
02-10 01:10:31  [   89.054033] Extra info: CPLD: 10=05 12=3e 92=00 e0=00
02-10 01:10:31  [   89.065598] [sched_delayed] process 1521 (shl1) no longer affine to cpu0
02-10 01:10:31  [   89.091240] reboot_helper: isam_reboot_type='warm'
02-10 01:10:31  [   89.108715] reboot-helper: Enabling preserved ram
02-10 01:10:31  [   89.126103] flush l2 cache.
02-10 01:10:31  [   89.141690] reboot_helper: continuing standard linux reboot
02-10 01:10:37  [   89.159957] Rebooting in 5 seconds..
```

# 内核相关代码
首先要参考笔记 [octeon中断](CPU_MIPS_octeon中断.md)  
根据现场Cause : 00800034, 结合前面的笔记  
我们知道出现了13号trap异常, 触发中断向量`handle_tr`, 进而调用`do_tr()`

do_tr()出自下面  
arch/mips/kernel/traps.c  
异常打印出自  
```c
do_tr(struct pt_regs *regs)
    do_trap_or_bp(struct pt_regs *regs, unsigned int code, const char *str)
        switch (code)
        case BRK_BUG:
            //如果是kernel模式下, die;
            die_if_kernel("Kernel bug detected", regs);
                if (unlikely(!user_mode(regs)))
                    die(str, regs);
                        oops_enter();
                        printk("%s[#%d]:\n", str, ++die_counter);
                        show_registers(regs);
                            __show_regs(regs);
                            print_modules();
                            show_stacktrace(current, regs);
                            show_code((unsigned int __user *) regs->cp0_epc);
                        add_taint(TAINT_DIE, LOCKDEP_NOW_UNRELIABLE);
                        oops_exit();
                        if (in_interrupt())
                            panic("Fatal exception in interrupt");
                        if (panic_on_oops)
                            printk(KERN_EMERG "Fatal exception: panic in 5 seconds");
                            ssleep(5);
                            panic("Fatal exception");
            //能到这里说明是user代码出了问题, 发SIGTRAP信号
            force_sig(SIGTRAP, current);
                force_sig_info(sig, SEND_SIG_PRIV, p);
```
