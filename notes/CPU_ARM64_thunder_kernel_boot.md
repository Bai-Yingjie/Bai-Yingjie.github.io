# 启动流程
```
EFI stub: Booting Linux Kernel...
```
```c
start_kernel() in init/main.c
    lockdep_init()
    set_task_stack_end_magic(&init_task)
    smp_setup_processor_id()
    debug_objects_early_init()
    boot_init_stack_canary()
    cgroup_init_early()
```
```
[    0.000000] Initializing cgroup subsys cpuset
[    0.000000] Initializing cgroup subsys cpu
[    0.000000] Initializing cgroup subsys cpuacct
```
```c
    local_irq_disable()
    early_boot_irqs_disabled = true
    boot_cpu_init()
    page_address_init()
    pr_notice("%s", linux_banner)
```
```
[    0.000000] Linux version 3.18.0-gdcccbc1-dirty (root@Tfedora) (gcc version 4.9.2 20150212 (Red Hat 4.9.2-6) (GCC) ) #4 SMP Thu Apr 23 15:05:33 CST 2015
```
```c
    setup_arch(&command_line) in arch/arm64/kernel/setup.c
        setup_processor()
```
```
[    0.000000] CPU: AArch64 Processor [430f0a10] revision 0
```
```c
            cpuinfo_store_boot_cpu()
                cpuinfo_detect_icache_policy(info)
```
```
[    0.000000] Detected VIPT I-cache on CPU0
```
```c
        setup_machine_fdt(__fdt_pointer)    //这里的__fdt_pointer是uefi或grub传过来的
            early_init_dt_scan()
                early_init_dt_verify()
                early_init_dt_scan_nodes()
                    这里只做三件事: 用/chosen填boot_command_line, root, memory
                    of_scan_flat_dt(early_init_dt_scan_chosen, boot_command_line)
                    of_scan_flat_dt(early_init_dt_scan_root, NULL)
                    of_scan_flat_dt(early_init_dt_scan_memory, NULL);
        *cmdline_p = boot_command_line
        early_ioremap_init()
        disable_acpi()
        parse_early_param()    //此时cmdline已经从fdt里面获取到了
            //调用相应的声明early_param("earlycon", setup_of_earlycon)
```
```
[    0.000000] Early serial console at I/O port 0x0 (options '')
[    0.000000] bootconsole [uart0] enabled
```
```c
        local_async_enable()
        efi_init()
            efi_get_fdt_params(&params, uefi_debug)
```
```
[    0.000000] efi: Getting EFI parameters from FDT:
```
```c
            uefi_init()
```
```
[    0.000000] EFI v2.40 by Cavium Thunder cn88xx EFI Mar 20 2015 12:00:56
```
```c
                efi_config_init(NULL)
```
```
[    0.000000] efi:  ACPI=0xfffff000  ACPI 2.0=0xfffff014 
```
```c
        arm64_memblock_init() in arch/arm64/mm/init.c
        acpi_boot_table_init()
        paging_init()
            map_mem()
            bootmem_init()
                arm64_numa_init()
```
```c
[    0.000000] numa: Adding memblock 0 [0x1400000 - 0x800000000] on node 0 //20M ~ 32G(或64G)
[    0.000000] numa: Adding memblock 1 [0x10000400000 - 0x10800000000] on node 1 //1T(+4M ~ +32G(或64G))
```
```c
                    ...setup_node_data()
```
```c
[    0.000000] Initmem setup node 0 [mem 0x40000000-0x7ffffffff] //1G ~ 32G
[    0.000000]   NODE_DATA [mem 0x7ffb10000-0x7ffb1ffff] //顶端64K
[    0.000000] Initmem setup node 1 [mem 0x10040000000-0x107ffffffff] //1T(+1G ~ 32G)
[    0.000000]   NODE_DATA [mem 0x107ffe60000-0x107ffe6ffff] //顶端64K
```
```c
                arm64_memory_present()
                sparse_init()
                zone_sizes_init()
                    free_area_init_nodes()
```
```c
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x01400000-0xffffffff] //20M ~ 4G
[    0.000000]   Normal   [mem 0x100000000-0x107ffffffff] //4G ~ (1T+32G)
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x01400000-0x7ffffffff]
[    0.000000]   node   1: [mem 0x10000400000-0x107ffffffff]
[    0.000000] Initmem setup node 0 [mem 0x01400000-0x7ffffffff]
[    0.000000] Initmem setup node 1 [mem 0x10000400000-0x107ffffffff]
```
```c
        request_standard_resources()
        efi_idmap_init()
        //没用acpi
        if (acpi_disabled)
            unflatten_device_tree() in drivers/of/fdt.c    //根据firmware传过来的fdt创建设备树
            psci_dt_init() in arch/arm64/kernel/psci.c
```
```
[    0.000000] psci: probing for conduit method from DT.
```
```c
                psci_0_2_init()
```
```
[    0.000000] psci: PSCIv0.2 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
```
```c
            cpu_read_bootcpu_ops()
            of_smp_init_cpus()
        smp_build_mpidr_hash()
    //setup_arch()结束
    mm_init_cpumask()
    setup_command_line()
    setup_nr_cpu_ids()
    setup_per_cpu_areas()
        pcpu_embed_first_chunk()
```
```
[    0.000000] PERCPU: Embedded 2 pages/cpu @ffff8107fb830000 s82688 r8192 d40192 u131072
```
```c
    smp_prepare_boot_cpu()
    build_all_zonelists()
```
```
[    0.000000] Built 2 zonelists in Node order, mobility grouping on.  Total pages: 1047296
[    0.000000] Policy zone: Normal
```
```c
    page_alloc_init()
    pr_notice("Kernel command line: %s\n", boot_command_line)
```
```
[    0.000000] Kernel command line: BOOT_IMAGE=/boot/vmlinuz root=/dev/sda3 console=ttyAMA0,115200n8 earlycon=pl011,0x87e024000000 coherent_pool=16M rootwait rw transparent_hugepage=never
[    0.000000] log_buf_len individual max cpu contribution: 4096 bytes
[    0.000000] log_buf_len total cpu_extra contributions: 389120 bytes
[    0.000000] log_buf_len min size: 16384 bytes
[    0.000000] log_buf_len: 524288 bytes
[    0.000000] early log buf free: 12420(75%)
[    0.000000] PID hash table entries: 4096 (order: -1, 32768 bytes)
[    0.000000] Memory: 66972608K/67084288K available (7944K kernel code, 1075K rwdata, 5376K rodata, 832K init, 1322K bss, 111680K reserved)
[    0.000000] Virtual kernel memory layout:
[    0.000000]     vmalloc : 0xffff000000000000 - 0xffff77ffffff0000   (122879 GB)
[    0.000000]     vmemmap : 0xffff780000000000 - 0xffff7c0000000000   (  4096 GB maximum)
[    0.000000]               0xffff780000004600 - 0xffff780039c00000   (   923 MB actual)
[    0.000000]     PCI I/O : 0xffff7ffffa000000 - 0xffff7ffffb000000   (    16 MB)
[    0.000000]     fixed   : 0xffff7ffffbde0000 - 0xffff7ffffbdf0000   (    64 KB)
[    0.000000]     modules : 0xffff7ffffc000000 - 0xffff800000000000   (    64 MB)
[    0.000000]     memory  : 0xffff800000000000 - 0xffff8107fec00000   (1081324 MB)
[    0.000000]       .init : 0xffff800000d90000 - 0xffff800000e60000   (   832 KB)
[    0.000000]       .text : 0xffff800000080000 - 0xffff800000d82b04   ( 13323 KB)
[    0.000000]       .data : 0xffff800000e60000 - 0xffff800000f6cdb0   (  1076 KB)
[    0.000000] SLUB: HWalign=128, Order=0-3, MinObjects=0, CPUs=96, Nodes=2
[    0.000000] Hierarchical RCU implementation.
[    0.000000] 	RCU dyntick-idle grace-period acceleration is enabled.
[    0.000000] NR_IRQS:64 nr_irqs:64 0
[    0.000000] ITS: /soc/interrupt-controller@8010,00000000/gic-its@8010,00020000
[    0.000000] ITS: allocated 8192 Devices @7c0800000 (psz 64K, shr 3)
[    0.000000] ITS: using cache flushing for cmd queue
[    0.000000] ITS: /soc/interrupt-controller@8010,00000000/gic-its@9010,00020000
[    0.000000] ITS: allocated 8192 Devices @7c1000000 (psz 64K, shr 3)
[    0.000000] ITS: using cache flushing for cmd queue
[    0.000000] GIC: using LPI property table @0x00000007c0100000
[    0.000000] ITS: Allocated 32512 chunks for LPIs
[    0.000000] CPU0: found redistributor 0 region 0:0x0000801080000000
[    0.000000] CPU0: using LPI pending table @0x00000007c0110000
[    0.000000] GIC: using cache flushing for LPI property table
[    0.000000] Architected cp15 timer(s) running at 100.00MHz (phys).
[    0.000000] sched_clock: 56 bits at 100MHz, resolution 10ns, wraps every 2748779069440ns
[    0.000000] Console: colour dummy device 80x25
[    0.000000] allocated 16777216 bytes of page_cgroup
[    0.000000] please try 'cgroup_disable=memory' option if you don't want memory cgroups
[    0.030000] Calibrating delay loop (skipped), value calculated using timer frequency.. 200.00 BogoMIPS (lpj=1000000)
[    0.030006] pid_max: default: 98304 minimum: 768
[    0.036060] Security Framework initialized
[    0.040028] AppArmor: AppArmor initialized
[    0.044155] Yama: becoming mindful.
[    0.049169] Dentry cache hash table entries: 8388608 (order: 10, 67108864 bytes)
[    0.081894] Inode-cache hash table entries: 4194304 (order: 9, 33554432 bytes)
[    0.107257] Mount-cache hash table entries: 131072 (order: 4, 1048576 bytes)
[    0.110023] Mountpoint-cache hash table entries: 131072 (order: 4, 1048576 bytes)
[    0.119248] Initializing cgroup subsys memory
[    0.120035] Initializing cgroup subsys devices
[    0.124514] Initializing cgroup subsys freezer
[    0.128995] Initializing cgroup subsys net_cls
[    0.130005] Initializing cgroup subsys blkio
[    0.134308] Initializing cgroup subsys perf_event
[    0.139053] Initializing cgroup subsys net_prio
[    0.140005] Initializing cgroup subsys hugetlb
[    0.144672] ftrace: allocating 28292 entries in 7 pages
[    0.283823] hw perfevents: enabled with arm/armv8-pmuv3 PMU driver, 7 counters available
[    0.290049] Remapping and enabling EFI services.
[    0.294996] Freed 0xe40000 bytes of EFI boot services memory
[    0.030000] CPU1: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU1
[    0.030000] CPU1: found redistributor 1 region 0:0x0000801080020000
[    0.030000] CPU1: using LPI pending table @0x00000007c4ef0000
[    0.030000] CPU2: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU2
[    0.030000] CPU2: found redistributor 2 region 0:0x0000801080040000
[    0.030000] CPU2: using LPI pending table @0x00000007c4fa0000
[    0.030000] CPU3: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU3
[    0.030000] CPU3: found redistributor 3 region 0:0x0000801080060000
[    0.030000] CPU3: using LPI pending table @0x00000007c5060000
[    0.030000] CPU4: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU4
[    0.030000] CPU4: found redistributor 4 region 0:0x0000801080080000
[    0.030000] CPU4: using LPI pending table @0x00000007c51c0000
[    0.030000] CPU5: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU5
[    0.030000] CPU5: found redistributor 5 region 0:0x00008010800a0000
[    0.030000] CPU5: using LPI pending table @0x00000007c5260000
[    0.030000] CPU6: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU6
[    0.030000] CPU6: found redistributor 6 region 0:0x00008010800c0000
[    0.030000] CPU6: using LPI pending table @0x00000007c5300000
[    0.030000] CPU7: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU7
[    0.030000] CPU7: found redistributor 7 region 0:0x00008010800e0000
[    0.030000] CPU7: using LPI pending table @0x00000007c53a0000
[    0.030000] CPU8: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU8
[    0.030000] CPU8: found redistributor 8 region 0:0x0000801080100000
[    0.030000] CPU8: using LPI pending table @0x00000007c5460000
[    0.030000] CPU9: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU9
[    0.030000] CPU9: found redistributor 9 region 0:0x0000801080120000
[    0.030000] CPU9: using LPI pending table @0x00000007c5500000
[    0.030000] CPU10: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU10
[    0.030000] CPU10: found redistributor a region 0:0x0000801080140000
[    0.030000] CPU10: using LPI pending table @0x00000007c5620000
[    0.030000] CPU11: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU11
[    0.030000] CPU11: found redistributor b region 0:0x0000801080160000
[    0.030000] CPU11: using LPI pending table @0x00000007c56c0000
[    0.030000] CPU12: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU12
[    0.030000] CPU12: found redistributor c region 0:0x0000801080180000
[    0.030000] CPU12: using LPI pending table @0x00000007c57a0000
[    0.030000] CPU13: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU13
[    0.030000] CPU13: found redistributor d region 0:0x00008010801a0000
[    0.030000] CPU13: using LPI pending table @0x00000007c5850000
[    0.030000] CPU14: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU14
[    0.030000] CPU14: found redistributor e region 0:0x00008010801c0000
[    0.030000] CPU14: using LPI pending table @0x00000007c58f0000
[    0.030000] CPU15: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU15
[    0.030000] CPU15: found redistributor f region 0:0x00008010801e0000
[    0.030000] CPU15: using LPI pending table @0x00000007c5a10000
[    0.030000] CPU16: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU16
[    0.030000] CPU16: found redistributor 100 region 0:0x0000801080200000
[    0.030000] CPU16: using LPI pending table @0x00000007c5ad0000
[    0.030000] CPU17: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU17
[    0.030000] CPU17: found redistributor 101 region 0:0x0000801080220000
[    0.030000] CPU17: using LPI pending table @0x00000007c5ba0000
[    0.030000] CPU18: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU18
[    0.030000] CPU18: found redistributor 102 region 0:0x0000801080240000
[    0.030000] CPU18: using LPI pending table @0x00000007c5c40000
[    0.030000] CPU19: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU19
[    0.030000] CPU19: found redistributor 103 region 0:0x0000801080260000
[    0.030000] CPU19: using LPI pending table @0x00000007c5cf0000
[    0.030000] CPU20: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU20
[    0.030000] CPU20: found redistributor 104 region 0:0x0000801080280000
[    0.030000] CPU20: using LPI pending table @0x00000007c5e30000
[    0.030000] CPU21: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU21
[    0.030000] CPU21: found redistributor 105 region 0:0x00008010802a0000
[    0.030000] CPU21: using LPI pending table @0x00000007c5ed0000
[    0.030000] CPU22: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU22
[    0.030000] CPU22: found redistributor 106 region 0:0x00008010802c0000
[    0.030000] CPU22: using LPI pending table @0x00000007c5f70000
[    0.030000] CPU23: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU23
[    0.030000] CPU23: found redistributor 107 region 0:0x00008010802e0000
[    0.030000] CPU23: using LPI pending table @0x0000010003e40000
[    0.030000] CPU24: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU24
[    0.030000] CPU24: found redistributor 108 region 0:0x0000801080300000
[    0.030000] CPU24: using LPI pending table @0x0000010003e50000
[    0.030000] CPU25: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU25
[    0.030000] CPU25: found redistributor 109 region 0:0x0000801080320000
[    0.030000] CPU25: using LPI pending table @0x0000010003e60000
[    0.030000] CPU26: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU26
[    0.030000] CPU26: found redistributor 10a region 0:0x0000801080340000
[    0.030000] CPU26: using LPI pending table @0x0000010003e70000
[    0.030000] CPU27: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU27
[    0.030000] CPU27: found redistributor 10b region 0:0x0000801080360000
[    0.030000] CPU27: using LPI pending table @0x0000010003ea0000
[    0.030000] CPU28: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU28
[    0.030000] CPU28: found redistributor 10c region 0:0x0000801080380000
[    0.030000] CPU28: using LPI pending table @0x0000010003eb0000
[    0.030000] CPU29: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU29
[    0.030000] CPU29: found redistributor 10d region 0:0x00008010803a0000
[    0.030000] CPU29: using LPI pending table @0x0000010003ec0000
[    0.030000] CPU30: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU30
[    0.030000] CPU30: found redistributor 10e region 0:0x00008010803c0000
[    0.030000] CPU30: using LPI pending table @0x0000010003ed0000
[    0.030000] CPU31: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU31
[    0.030000] CPU31: found redistributor 10f region 0:0x00008010803e0000
[    0.030000] CPU31: using LPI pending table @0x0000010003f00000
[    0.030000] CPU32: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU32
[    0.030000] CPU32: found redistributor 200 region 0:0x0000801080400000
[    0.030000] CPU32: using LPI pending table @0x0000010003f10000
[    0.030000] CPU33: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU33
[    0.030000] CPU33: found redistributor 201 region 0:0x0000801080420000
[    0.030000] CPU33: using LPI pending table @0x0000010003f20000
[    0.030000] CPU34: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU34
[    0.030000] CPU34: found redistributor 202 region 0:0x0000801080440000
[    0.030000] CPU34: using LPI pending table @0x0000010003f30000
[    0.030000] CPU35: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU35
[    0.030000] CPU35: found redistributor 203 region 0:0x0000801080460000
[    0.030000] CPU35: using LPI pending table @0x0000010003f60000
[    0.030000] CPU36: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU36
[    0.030000] CPU36: found redistributor 204 region 0:0x0000801080480000
[    0.030000] CPU36: using LPI pending table @0x0000010003f70000
[    0.030000] CPU37: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU37
[    0.030000] CPU37: found redistributor 205 region 0:0x00008010804a0000
[    0.030000] CPU37: using LPI pending table @0x0000010003f80000
[    0.030000] CPU38: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU38
[    0.030000] CPU38: found redistributor 206 region 0:0x00008010804c0000
[    0.030000] CPU38: using LPI pending table @0x0000010003f90000
[    0.030000] CPU39: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU39
[    0.030000] CPU39: found redistributor 207 region 0:0x00008010804e0000
[    0.030000] CPU39: using LPI pending table @0x0000010003fc0000
[    0.030000] CPU40: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU40
[    0.030000] CPU40: found redistributor 208 region 0:0x0000801080500000
[    0.030000] CPU40: using LPI pending table @0x0000010003fd0000
[    0.030000] CPU41: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU41
[    0.030000] CPU41: found redistributor 209 region 0:0x0000801080520000
[    0.030000] CPU41: using LPI pending table @0x0000010003fe0000
[    0.030000] CPU42: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU42
[    0.030000] CPU42: found redistributor 20a region 0:0x0000801080540000
[    0.030000] CPU42: using LPI pending table @0x0000010003ff0000
[    0.030000] CPU43: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU43
[    0.030000] CPU43: found redistributor 20b region 0:0x0000801080560000
[    0.030000] CPU43: using LPI pending table @0x0000010004020000
[    0.030000] CPU44: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU44
[    0.030000] CPU44: found redistributor 20c region 0:0x0000801080580000
[    0.030000] CPU44: using LPI pending table @0x0000010004030000
[    0.030000] CPU45: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU45
[    0.030000] CPU45: found redistributor 20d region 0:0x00008010805a0000
[    0.030000] CPU45: using LPI pending table @0x0000010004040000
[    0.030000] CPU46: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU46
[    0.030000] CPU46: found redistributor 20e region 0:0x00008010805c0000
[    0.030000] CPU46: using LPI pending table @0x0000010004050000
[    0.030000] CPU47: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU47
[    0.030000] CPU47: found redistributor 20f region 0:0x00008010805e0000
[    0.030000] CPU47: using LPI pending table @0x0000010004060000
[    0.030000] CPU48: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU48
[    0.030000] CPU48: found redistributor 10000 region 1:0x0000901080000000
[    0.030000] CPU48: using LPI pending table @0x0000010004070000
[    0.030000] CPU49: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU49
[    0.030000] CPU49: found redistributor 10001 region 1:0x0000901080020000
[    0.030000] CPU49: using LPI pending table @0x00000100040a0000
[    0.030000] CPU50: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU50
[    0.030000] CPU50: found redistributor 10002 region 1:0x0000901080040000
[    0.030000] CPU50: using LPI pending table @0x00000100040b0000
[    0.030000] CPU51: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU51
[    0.030000] CPU51: found redistributor 10003 region 1:0x0000901080060000
[    0.030000] CPU51: using LPI pending table @0x00000100040c0000
[    0.030000] CPU52: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU52
[    0.030000] CPU52: found redistributor 10004 region 1:0x0000901080080000
[    0.030000] CPU52: using LPI pending table @0x00000100040d0000
[    0.030000] CPU53: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU53
[    0.030000] CPU53: found redistributor 10005 region 1:0x00009010800a0000
[    0.030000] CPU53: using LPI pending table @0x0000010004100000
[    0.030000] CPU54: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU54
[    0.030000] CPU54: found redistributor 10006 region 1:0x00009010800c0000
[    0.030000] CPU54: using LPI pending table @0x0000010004110000
[    0.030000] CPU55: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU55
[    0.030000] CPU55: found redistributor 10007 region 1:0x00009010800e0000
[    0.030000] CPU55: using LPI pending table @0x0000010004120000
[    0.030000] CPU56: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU56
[    0.030000] CPU56: found redistributor 10008 region 1:0x0000901080100000
[    0.030000] CPU56: using LPI pending table @0x0000010004130000
[    0.030000] CPU57: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU57
[    0.030000] CPU57: found redistributor 10009 region 1:0x0000901080120000
[    0.030000] CPU57: using LPI pending table @0x0000010004160000
[    0.030000] CPU58: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU58
[    0.030000] CPU58: found redistributor 1000a region 1:0x0000901080140000
[    0.030000] CPU58: using LPI pending table @0x0000010004170000
[    0.030000] CPU59: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU59
[    0.030000] CPU59: found redistributor 1000b region 1:0x0000901080160000
[    0.030000] CPU59: using LPI pending table @0x0000010004180000
[    0.030000] CPU60: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU60
[    0.030000] CPU60: found redistributor 1000c region 1:0x0000901080180000
[    0.030000] CPU60: using LPI pending table @0x0000010004190000
[    0.030000] CPU61: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU61
[    0.030000] CPU61: found redistributor 1000d region 1:0x00009010801a0000
[    0.030000] CPU61: using LPI pending table @0x00000100041c0000
[    0.030000] CPU62: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU62
[    0.030000] CPU62: found redistributor 1000e region 1:0x00009010801c0000
[    0.030000] CPU62: using LPI pending table @0x00000100041d0000
[    0.030000] CPU63: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU63
[    0.030000] CPU63: found redistributor 1000f region 1:0x00009010801e0000
[    0.030000] CPU63: using LPI pending table @0x00000100041e0000
[    0.030000] CPU64: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU64
[    0.030000] CPU64: found redistributor 10100 region 1:0x0000901080200000
[    0.030000] CPU64: using LPI pending table @0x00000100041f0000
[    0.030000] CPU65: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU65
[    0.030000] CPU65: found redistributor 10101 region 1:0x0000901080220000
[    0.030000] CPU65: using LPI pending table @0x0000010004220000
[    0.030000] CPU66: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU66
[    0.030000] CPU66: found redistributor 10102 region 1:0x0000901080240000
[    0.030000] CPU66: using LPI pending table @0x0000010004230000
[    0.030000] CPU67: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU67
[    0.030000] CPU67: found redistributor 10103 region 1:0x0000901080260000
[    0.030000] CPU67: using LPI pending table @0x0000010004240000
[    0.030000] CPU68: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU68
[    0.030000] CPU68: found redistributor 10104 region 1:0x0000901080280000
[    0.030000] CPU68: using LPI pending table @0x0000010004250000
[    0.030000] CPU69: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU69
[    0.030000] CPU69: found redistributor 10105 region 1:0x00009010802a0000
[    0.030000] CPU69: using LPI pending table @0x0000010004280000
[    0.030000] CPU70: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU70
[    0.030000] CPU70: found redistributor 10106 region 1:0x00009010802c0000
[    0.030000] CPU70: using LPI pending table @0x0000010004290000
[    0.030000] CPU71: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU71
[    0.030000] CPU71: found redistributor 10107 region 1:0x00009010802e0000
[    0.030000] CPU71: using LPI pending table @0x00000100042a0000
[    0.030000] CPU72: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU72
[    0.030000] CPU72: found redistributor 10108 region 1:0x0000901080300000
[    0.030000] CPU72: using LPI pending table @0x00000100042b0000
[    0.030000] CPU73: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU73
[    0.030000] CPU73: found redistributor 10109 region 1:0x0000901080320000
[    0.030000] CPU73: using LPI pending table @0x00000100042c0000
[    0.030000] CPU74: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU74
[    0.030000] CPU74: found redistributor 1010a region 1:0x0000901080340000
[    0.030000] CPU74: using LPI pending table @0x00000100042d0000
[    0.030000] CPU75: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU75
[    0.030000] CPU75: found redistributor 1010b region 1:0x0000901080360000
[    0.030000] CPU75: using LPI pending table @0x0000010004300000
[    0.030000] CPU76: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU76
[    0.030000] CPU76: found redistributor 1010c region 1:0x0000901080380000
[    0.030000] CPU76: using LPI pending table @0x0000010004310000
[    0.030000] CPU77: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU77
[    0.030000] CPU77: found redistributor 1010d region 1:0x00009010803a0000
[    0.030000] CPU77: using LPI pending table @0x0000010004320000
[    0.030000] CPU78: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU78
[    0.030000] CPU78: found redistributor 1010e region 1:0x00009010803c0000
[    0.030000] CPU78: using LPI pending table @0x0000010004330000
[    0.030000] CPU79: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU79
[    0.030000] CPU79: found redistributor 1010f region 1:0x00009010803e0000
[    0.030000] CPU79: using LPI pending table @0x0000010004360000
[    0.030000] CPU80: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU80
[    0.030000] CPU80: found redistributor 10200 region 1:0x0000901080400000
[    0.030000] CPU80: using LPI pending table @0x0000010004370000
[    0.030000] CPU81: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU81
[    0.030000] CPU81: found redistributor 10201 region 1:0x0000901080420000
[    0.030000] CPU81: using LPI pending table @0x0000010004380000
[    0.030000] CPU82: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU82
[    0.030000] CPU82: found redistributor 10202 region 1:0x0000901080440000
[    0.030000] CPU82: using LPI pending table @0x0000010004390000
[    0.030000] CPU83: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU83
[    0.030000] CPU83: found redistributor 10203 region 1:0x0000901080460000
[    0.030000] CPU83: using LPI pending table @0x00000100043c0000
[    0.030000] CPU84: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU84
[    0.030000] CPU84: found redistributor 10204 region 1:0x0000901080480000
[    0.030000] CPU84: using LPI pending table @0x00000100043d0000
[    0.030000] CPU85: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU85
[    0.030000] CPU85: found redistributor 10205 region 1:0x00009010804a0000
[    0.030000] CPU85: using LPI pending table @0x00000100043e0000
[    0.030000] CPU86: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU86
[    0.030000] CPU86: found redistributor 10206 region 1:0x00009010804c0000
[    0.030000] CPU86: using LPI pending table @0x00000100043f0000
[    0.030000] CPU87: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU87
[    0.030000] CPU87: found redistributor 10207 region 1:0x00009010804e0000
[    0.030000] CPU87: using LPI pending table @0x0000010004420000
[    0.030000] CPU88: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU88
[    0.030000] CPU88: found redistributor 10208 region 1:0x0000901080500000
[    0.030000] CPU88: using LPI pending table @0x0000010004430000
[    0.030000] CPU89: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU89
[    0.030000] CPU89: found redistributor 10209 region 1:0x0000901080520000
[    0.030000] CPU89: using LPI pending table @0x0000010004440000
[    0.030000] CPU90: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU90
[    0.030000] CPU90: found redistributor 1020a region 1:0x0000901080540000
[    0.030000] CPU90: using LPI pending table @0x0000010004450000
[    0.030000] CPU91: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU91
[    0.030000] CPU91: found redistributor 1020b region 1:0x0000901080560000
[    0.030000] CPU91: using LPI pending table @0x0000010004480000
[    0.030000] CPU92: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU92
[    0.030000] CPU92: found redistributor 1020c region 1:0x0000901080580000
[    0.030000] CPU92: using LPI pending table @0x0000010004490000
[    0.030000] CPU93: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU93
[    0.030000] CPU93: found redistributor 1020d region 1:0x00009010805a0000
[    0.030000] CPU93: using LPI pending table @0x00000100044a0000
[    0.030000] CPU94: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU94
[    0.030000] CPU94: found redistributor 1020e region 1:0x00009010805c0000
[    0.030000] CPU94: using LPI pending table @0x00000100044b0000
[    0.030000] CPU95: Booted secondary processor
[    0.030000] Detected VIPT I-cache on CPU95
[    0.030000] CPU95: found redistributor 1020f region 1:0x00009010805e0000
[    0.030000] CPU95: using LPI pending table @0x00000100044c0000
[    0.340309] Brought up 96 CPUs
[    2.238950] SMP: Total of 96 processors activated.
[    2.243500] devtmpfs: initialized
[    2.243640] evm: security.selinux
[    2.250013] evm: security.SMACK64
[    2.253349] evm: security.SMACK64EXEC
[    2.257034] evm: security.SMACK64TRANSMUTE
[    2.260003] evm: security.SMACK64MMAP
[    2.263691] evm: security.ima
[    2.266674] evm: security.capability
[    2.274053] regulator-dummy: no parameters
[    2.283325] NET: Registered protocol family 16
[    2.310008] cpuidle: using governor ladder
[    2.340007] cpuidle: using governor menu
[    2.344033] vdso: 2 pages (1 code @ ffff800000e80000, 1 data @ ffff800000e70000)
[    2.350090] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    2.358622] software IO TLB [mem 0x04000000-0x08000000] (64MB) mapped at [ffff800002c00000-ffff800006bfffff]
[    2.360649] DMA: preallocated 16384 KiB pool for atomic allocations
[    2.367062] Serial: AMBA PL011 UART driver
[    2.371019] uart-pl011 87e024000000.serial: ttyAMA0 at MMIO 0x87e024000000 (irq = 37, base_baud = 0) is a PL011 rev3
[    2.380016] console [ttyAMA0] enabled
[    2.380016] console [ttyAMA0] enabled
[    2.387352] bootconsole [uart0] disabled
[    2.387352] bootconsole [uart0] disabled
[    2.390317] uart-pl011 87e025000000.serial: ttyAMA1 at MMIO 0x87e025000000 (irq = 38, base_baud = 0) is a PL011 rev3
[    2.471267] vgaarb: loaded
[    2.471267] SCSI subsystem initialized
[    2.480157] usbcore: registered new interface driver usbfs
[    2.480157] usbcore: registered new interface driver hub
[    2.490118] usbcore: registered new device driver usb
[    2.492226] arm-smmu 830000000000.smmu0: probing hardware configuration...
[    2.499090] arm-smmu 830000000000.smmu0: SMMUv2 with:
[    2.510007] arm-smmu 830000000000.smmu0: 	stage 1 translation
[    2.515740] arm-smmu 830000000000.smmu0: 	stage 2 translation
[    2.520003] arm-smmu 830000000000.smmu0: 	nested translation
[    2.525650] arm-smmu 830000000000.smmu0: 	coherent table walk
[    2.530004] arm-smmu 830000000000.smmu0: 	stream matching with 128 register groups, mask 0x7fff
[    2.538689] arm-smmu 830000000000.smmu0: 	128 context banks (0 stage-2 only)
[    2.550006] arm-smmu 830000000000.smmu0: 	Stage-1: 48-bit VA -> 48-bit IPA
[    2.556868] arm-smmu 830000000000.smmu0: 	Stage-2: 48-bit IPA -> 48-bit PA
[    2.560028] arm-smmu 830000000000.smmu0: registered 1 master devices
[    2.568034] arm-smmu 831000000000.smmu1: probing hardware configuration...
[    2.580004] arm-smmu 831000000000.smmu1: SMMUv2 with:
[    2.585044] arm-smmu 831000000000.smmu1: 	stage 1 translation
[    2.590003] arm-smmu 831000000000.smmu1: 	stage 2 translation
[    2.595736] arm-smmu 831000000000.smmu1: 	nested translation
[    2.600003] arm-smmu 831000000000.smmu1: 	coherent table walk
[    2.605737] arm-smmu 831000000000.smmu1: 	stream matching with 128 register groups, mask 0x7fff
[    2.610003] arm-smmu 831000000000.smmu1: 	128 context banks (0 stage-2 only)
[    2.617039] arm-smmu 831000000000.smmu1: 	Stage-1: 48-bit VA -> 48-bit IPA
[    2.630003] arm-smmu 831000000000.smmu1: 	Stage-2: 48-bit IPA -> 48-bit PA
[    2.636888] arm-smmu 831000000000.smmu1: registered 1 master devices
[    2.641663] arm-smmu 832000000000.smmu2: probing hardware configuration...
[    2.648526] arm-smmu 832000000000.smmu2: SMMUv2 with:
[    2.650004] arm-smmu 832000000000.smmu2: 	stage 1 translation
[    2.655737] arm-smmu 832000000000.smmu2: 	stage 2 translation
[    2.660003] arm-smmu 832000000000.smmu2: 	nested translation
[    2.665649] arm-smmu 832000000000.smmu2: 	coherent table walk
[    2.680004] arm-smmu 832000000000.smmu2: 	stream matching with 128 register groups, mask 0x7fff
[    2.688688] arm-smmu 832000000000.smmu2: 	128 context banks (0 stage-2 only)
[    2.690003] arm-smmu 832000000000.smmu2: 	Stage-1: 48-bit VA -> 48-bit IPA
[    2.696865] arm-smmu 832000000000.smmu2: 	Stage-2: 48-bit IPA -> 48-bit PA
[    2.710027] arm-smmu 832000000000.smmu2: registered 1 master devices
[    2.718027] arm-smmu 833000000000.smmu3: probing hardware configuration...
[    2.720004] arm-smmu 833000000000.smmu3: SMMUv2 with:
[    2.725043] arm-smmu 833000000000.smmu3: 	stage 1 translation
[    2.730003] arm-smmu 833000000000.smmu3: 	stage 2 translation
[    2.735736] arm-smmu 833000000000.smmu3: 	nested translation
[    2.740003] arm-smmu 833000000000.smmu3: 	coherent table walk
[    2.745737] arm-smmu 833000000000.smmu3: 	stream matching with 128 register groups, mask 0x7fff
[    2.760003] arm-smmu 833000000000.smmu3: 	128 context banks (0 stage-2 only)
[    2.767039] arm-smmu 833000000000.smmu3: 	Stage-1: 48-bit VA -> 48-bit IPA
[    2.770004] arm-smmu 833000000000.smmu3: 	Stage-2: 48-bit IPA -> 48-bit PA
[    2.776890] arm-smmu 833000000000.smmu3: registered 1 master devices
[    2.781680] arm-smmu 930000000000.smmu4: probing hardware configuration...
[    2.788543] arm-smmu 930000000000.smmu4: SMMUv2 with:
[    2.800005] arm-smmu 930000000000.smmu4: 	stage 1 translation
[    2.805738] arm-smmu 930000000000.smmu4: 	stage 2 translation
[    2.810003] arm-smmu 930000000000.smmu4: 	nested translation
[    2.815649] arm-smmu 930000000000.smmu4: 	coherent table walk
[    2.820004] arm-smmu 930000000000.smmu4: 	stream matching with 128 register groups, mask 0x7fff
[    2.828689] arm-smmu 930000000000.smmu4: 	128 context banks (0 stage-2 only)
[    2.840004] arm-smmu 930000000000.smmu4: 	Stage-1: 48-bit VA -> 48-bit IPA
[    2.846866] arm-smmu 930000000000.smmu4: 	Stage-2: 48-bit IPA -> 48-bit PA
[    2.850028] arm-smmu 930000000000.smmu4: registered 1 master devices
[    2.858036] arm-smmu 931000000000.smmu5: probing hardware configuration...
[    2.860004] arm-smmu 931000000000.smmu5: SMMUv2 with:
[    2.865044] arm-smmu 931000000000.smmu5: 	stage 1 translation
[    2.880003] arm-smmu 931000000000.smmu5: 	stage 2 translation
[    2.885736] arm-smmu 931000000000.smmu5: 	nested translation
[    2.890003] arm-smmu 931000000000.smmu5: 	coherent table walk
[    2.895737] arm-smmu 931000000000.smmu5: 	stream matching with 128 register groups, mask 0x7fff
[    2.900004] arm-smmu 931000000000.smmu5: 	128 context banks (0 stage-2 only)
[    2.907040] arm-smmu 931000000000.smmu5: 	Stage-1: 48-bit VA -> 48-bit IPA
[    2.920003] arm-smmu 931000000000.smmu5: 	Stage-2: 48-bit IPA -> 48-bit PA
[    2.926890] arm-smmu 931000000000.smmu5: registered 1 master devices
[    2.931669] arm-smmu 932000000000.smmu6: probing hardware configuration...
[    2.938531] arm-smmu 932000000000.smmu6: SMMUv2 with:
[    2.940004] arm-smmu 932000000000.smmu6: 	stage 1 translation
[    2.945738] arm-smmu 932000000000.smmu6: 	stage 2 translation
[    2.950003] arm-smmu 932000000000.smmu6: 	nested translation
[    2.955650] arm-smmu 932000000000.smmu6: 	coherent table walk
[    2.970004] arm-smmu 932000000000.smmu6: 	stream matching with 128 register groups, mask 0x7fff
[    2.978688] arm-smmu 932000000000.smmu6: 	128 context banks (0 stage-2 only)
[    2.980004] arm-smmu 932000000000.smmu6: 	Stage-1: 48-bit VA -> 48-bit IPA
[    2.986866] arm-smmu 932000000000.smmu6: 	Stage-2: 48-bit IPA -> 48-bit PA
[    2.990029] arm-smmu 932000000000.smmu6: registered 1 master devices
[    2.998036] arm-smmu 933000000000.smmu7: probing hardware configuration...
[    3.010004] arm-smmu 933000000000.smmu7: SMMUv2 with:
[    3.015043] arm-smmu 933000000000.smmu7: 	stage 1 translation
[    3.020003] arm-smmu 933000000000.smmu7: 	stage 2 translation
[    3.025736] arm-smmu 933000000000.smmu7: 	nested translation
[    3.030003] arm-smmu 933000000000.smmu7: 	coherent table walk
[    3.035737] arm-smmu 933000000000.smmu7: 	stream matching with 128 register groups, mask 0x7fff
[    3.050003] arm-smmu 933000000000.smmu7: 	128 context banks (0 stage-2 only)
[    3.057039] arm-smmu 933000000000.smmu7: 	Stage-1: 48-bit VA -> 48-bit IPA
[    3.060003] arm-smmu 933000000000.smmu7: 	Stage-2: 48-bit IPA -> 48-bit PA
[    3.066891] arm-smmu 933000000000.smmu7: registered 1 master devices
[    3.070340] NetLabel: Initializing
[    3.073730] NetLabel:  domain hash size = 128
[    3.078074] NetLabel:  protocols = UNLABELED CIPSOv4
[    3.090036] NetLabel:  unlabeled traffic allowed by default
[    3.095631] Switched to clocksource arch_sys_counter
[    3.126141] AppArmor: AppArmor Filesystem Enabled
[    3.135798] NET: Registered protocol family 2
[    3.140916] TCP established hash table entries: 524288 (order: 6, 4194304 bytes)
[    3.150310] TCP bind hash table entries: 65536 (order: 4, 1048576 bytes)
[    3.157485] TCP: Hash tables configured (established 524288 bind 65536)
[    3.164139] TCP: reno registered
[    3.167386] UDP hash table entries: 32768 (order: 4, 1048576 bytes)
[    3.174329] UDP-Lite hash table entries: 32768 (order: 4, 1048576 bytes)
[    3.182140] NET: Registered protocol family 1
[    3.187696] RPC: Registered named UNIX socket transport module.
[    3.193621] RPC: Registered udp transport module.
[    3.198313] RPC: Registered tcp transport module.
[    3.203012] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    3.213858] futex hash table entries: 32768 (order: 6, 4194304 bytes)
[    3.220514] Initialise system trusted keyring
[    3.225077] audit: initializing netlink subsys (disabled)
[    3.230510] audit: type=2000 audit(3.230:1): initialized
[    3.239371] HugeTLB registered 512 MB page size, pre-allocated 0 pages
[    3.250251] zpool: loaded
[    3.252863] zbud: loaded
[    3.256397] VFS: Disk quotas dquot_6.5.2
[    3.260621] Dquot-cache hash table entries: 8192 (order 0, 65536 bytes)
[    3.272549] NFS: Registering the id_resolver key type
[    3.277612] Key type id_resolver registered
[    3.281799] Key type id_legacy registered
[    3.285810] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    3.292608] fuse init (API version 7.23)
[    3.297030] SGI XFS with ACLs, security attributes, realtime, no debug enabled
[    3.309280] msgmni has been set to 32768
[    3.313515] Key type big_key registered
[    3.318517] Key type asymmetric registered
[    3.322620] Asymmetric key parser 'x509' registered
[    3.327541] bounce: pool size: 64 pages
[    3.331609] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 252)
[    3.339594] io scheduler noop registered (default)
[    3.344392] io scheduler deadline registered
[    3.348680] io scheduler cfq registered
[    3.352884] pci_hotplug: PCI Hot Plug PCI Core version: 0.5
[    3.358458] pciehp: PCI Express Hot Plug Controller Driver version: 0.4
[    3.365622] thunder_pcie_probe: ECAM0 CFG BASE 0x848000000000 gser_base0:ffff00001d000000
[    3.373795] thunder_pcie_probe: ECAM0 CFG BASE 0x848000000000 gser_base1:ffff00002a800000
[    3.381963] PCI host bridge /pcie0@0x8480,00000000 ranges:
[    3.387455]   MEM 0x801000000000..0x807fffffffff -> 0x801000000000
[    3.393631]   MEM 0x830000000000..0x87ffffffffff -> 0x830000000000
[    3.399890] thunder-pcie 848000000000.pcie0: PCI host bridge to bus 0000:00
[    3.406854] pci_bus 0000:00: root bus resource [bus 00-ff]
[    3.412334] pci_bus 0000:00: root bus resource [mem 0x801000000000-0x807fffffffff]
[    3.419892] pci_bus 0000:00: root bus resource [mem 0x830000000000-0x87ffffffffff]
[    3.430713] pci 0000:01:00.0: disabling ASPM on pre-1.1 PCIe device.  You can enable it with 'pcie_aspm=force'
[    3.440945] pci 0000:02:00.0: disabling ASPM on pre-1.1 PCIe device.  You can enable it with 'pcie_aspm=force'
[    3.451159] pci 0000:03:00.0: disabling ASPM on pre-1.1 PCIe device.  You can enable it with 'pcie_aspm=force'
[    3.461372] pci 0000:04:00.0: disabling ASPM on pre-1.1 PCIe device.  You can enable it with 'pcie_aspm=force'
[    3.472119] thunder_pcie_probe: ECAM1 CFG BASE 0x849000000000 gser_base0:ffff00001d000000
[    3.480303] thunder_pcie_probe: ECAM1 CFG BASE 0x849000000000 gser_base1:ffff00002a800000
[    3.488467] PCI host bridge /pcie1@0x8490,00000000 ranges:
[    3.493954]   MEM 0x831000000000..0x83100fffffff -> 0x831000000000
[    3.500126]   MEM 0x810000000000..0x817fffffffff -> 0x810000000000
[    3.506386] thunder-pcie 849000000000.pcie1: PCI host bridge to bus 0001:00
[    3.513348] pci_bus 0001:00: root bus resource [bus 00-ff]
[    3.518822] pci_bus 0001:00: root bus resource [mem 0x831000000000-0x83100fffffff]
[    3.526386] pci_bus 0001:00: root bus resource [mem 0x810000000000-0x817fffffffff]
[    3.534873] thunder_pcie_probe: ECAM2 CFG BASE 0x84a000000000 gser_base0:ffff00001d000000
[    3.543050] thunder_pcie_probe: ECAM2 CFG BASE 0x84a000000000 gser_base1:ffff00002a800000
[    3.551219] PCI host bridge /pcie2@0x84a0,00000000 ranges:
[    3.556696]   MEM 0x832000000000..0x83200fffffff -> 0x832000000000
[    3.562871]   MEM 0x843000000000..0x8430ffffffff -> 0x843000000000
[    3.569115] thunder-pcie 84a000000000.pcie2: PCI host bridge to bus 0002:00
[    3.576078] pci_bus 0002:00: root bus resource [bus 00-ff]
[    3.581559] pci_bus 0002:00: root bus resource [mem 0x832000000000-0x83200fffffff]
[    3.589116] pci_bus 0002:00: root bus resource [mem 0x843000000000-0x8430ffffffff]
[    4.600163] pci 0002:01:00.0: disabling ASPM on pre-1.1 PCIe device.  You can enable it with 'pcie_aspm=force'
[    4.610203] pci 0002:00:03.0: can't claim BAR 0 [mem 0x842000000000-0x84200000ffff 64bit]: no compatible bridge window
[    4.620895] pci 0002:00:03.0: can't claim BAR 2 [mem 0x842040000000-0x84207fffffff 64bit]: no compatible bridge window
[    4.631580] pci 0002:00:03.0: can't claim BAR 4 [mem 0x842000f00000-0x842000ffffff 64bit]: no compatible bridge window
[    4.642431] thunder_pcie_probe: ECAM3 CFG BASE 0x84b000000000 gser_base0:ffff00001d000000
[    4.650602] thunder_pcie_probe: ECAM3 CFG BASE 0x84b000000000 gser_base1:ffff00002a800000
[    4.658765] PCI host bridge /pcie3@0x84b0,00000000 ranges:
[    4.664245]   MEM 0x833000000000..0x83300fffffff -> 0x833000000000
[    4.670418]   MEM 0x818000000000..0x81ffffffffff -> 0x818000000000
[    4.676662] thunder-pcie 84b000000000.pcie3: PCI host bridge to bus 0003:00
[    4.683620] pci_bus 0003:00: root bus resource [bus 00-ff]
[    4.689094] pci_bus 0003:00: root bus resource [mem 0x833000000000-0x83300fffffff]
[    4.696654] pci_bus 0003:00: root bus resource [mem 0x818000000000-0x81ffffffffff]
[    4.704547] thunder_pcie_probe: ECAM4 CFG BASE 0x948000000000 gser_base0:ffff00001d000000
[    4.712722] thunder_pcie_probe: ECAM4 CFG BASE 0x948000000000 gser_base1:ffff00002a800000
[    4.720888] PCI host bridge /pcie4@0x9480,00000000 ranges:
[    4.726365]   MEM 0x901000000000..0x907fffffffff -> 0x901000000000
[    4.732537]   MEM 0x930000000000..0x97ffffffffff -> 0x930000000000
[    4.738782] thunder-pcie 948000000000.pcie4: PCI host bridge to bus 0004:00
[    4.745741] pci_bus 0004:00: root bus resource [bus 00-ff]
[    4.751218] pci_bus 0004:00: root bus resource [mem 0x901000000000-0x907fffffffff]
[    4.758775] pci_bus 0004:00: root bus resource [mem 0x930000000000-0x97ffffffffff]
[    4.769273] pci 0004:01:00.0: disabling ASPM on pre-1.1 PCIe device.  You can enable it with 'pcie_aspm=force'
[    4.779800] thunder_pcie_probe: ECAM5 CFG BASE 0x949000000000 gser_base0:ffff00001d000000
[    4.787976] thunder_pcie_probe: ECAM5 CFG BASE 0x949000000000 gser_base1:ffff00002a800000
[    4.796143] PCI host bridge /pcie5@0x9490,00000000 ranges:
[    4.801623]   MEM 0x931000000000..0x93100fffffff -> 0x931000000000
[    4.807791]   MEM 0x910000000000..0x917fffffffff -> 0x910000000000
[    4.814039] thunder-pcie 949000000000.pcie5: PCI host bridge to bus 0005:00
[    4.820997] pci_bus 0005:00: root bus resource [bus 00-ff]
[    4.826471] pci_bus 0005:00: root bus resource [mem 0x931000000000-0x93100fffffff]
[    4.834032] pci_bus 0005:00: root bus resource [mem 0x910000000000-0x917fffffffff]
[    4.842588] thunder_pcie_probe: ECAM6 CFG BASE 0x94a000000000 gser_base0:ffff00001d000000
[    4.850763] thunder_pcie_probe: ECAM6 CFG BASE 0x94a000000000 gser_base1:ffff00002a800000
[    4.858925] PCI host bridge /pcie6@0x94a0,00000000 ranges:
[    4.864406]   MEM 0x932000000000..0x93200fffffff -> 0x932000000000
[    4.870578]   MEM 0x943000000000..0x9430ffffffff -> 0x943000000000
[    4.876823] thunder-pcie 94a000000000.pcie6: PCI host bridge to bus 0006:00
[    4.883782] pci_bus 0006:00: root bus resource [bus 00-ff]
[    4.889256] pci_bus 0006:00: root bus resource [mem 0x932000000000-0x93200fffffff]
[    4.896816] pci_bus 0006:00: root bus resource [mem 0x943000000000-0x9430ffffffff]
[    5.910186] pci 0006:01:00.0: disabling ASPM on pre-1.1 PCIe device.  You can enable it with 'pcie_aspm=force'
[    5.920387] thunder_pcie_probe: ECAM7 CFG BASE 0x94b000000000 gser_base0:ffff00001d000000
[    5.928552] thunder_pcie_probe: ECAM7 CFG BASE 0x94b000000000 gser_base1:ffff00002a800000
[    5.936725] PCI host bridge /pcie7@0x94b0,00000000 ranges:
[    5.942206]   MEM 0x933000000000..0x93300fffffff -> 0x933000000000
[    5.948374]   MEM 0x918000000000..0x91ffffffffff -> 0x918000000000
[    5.954624] thunder-pcie 94b000000000.pcie7: PCI host bridge to bus 0007:00
[    5.961582] pci_bus 0007:00: root bus resource [bus 00-ff]
[    5.967056] pci_bus 0007:00: root bus resource [mem 0x933000000000-0x93300fffffff]
[    5.974616] pci_bus 0007:00: root bus resource [mem 0x918000000000-0x91ffffffffff]
[    5.982530] thunder_pcie_probe: ECAM0 CFG BASE 0x87e0c0000000 gser_base0:ffff00001d000000
[    5.990705] thunder_pcie_probe: ECAM0 CFG BASE 0x87e0c0000000 gser_base1:ffff00002a800000
[    5.998868] PCI host bridge /pem0@0x87e0,c0000000 ranges:
[    6.004265]   No bus range found for /pem0@0x87e0,c0000000, using [bus 00-ff]
[    6.011395]   MEM 0x881008000000..0x88100fffffff -> 0x08000000
[    6.017217]   MEM 0x882010000000..0x882017ffffff -> 0x10000000
[    6.023042]    IO 0x883000000000..0x883007ffffff -> 0x00000000
[    6.028861] Requested IO range too big, new size set to 64K
[    6.034424] I/O range found for /pem0@0x87e0,c0000000. Please provide an io_base pointer to save CPU base address
[    6.044721] thunder-pcie: probe of 87e0c0000000.pem0 failed with error -22
[    6.051763] thunder_pcie_probe: ECAM0 CFG BASE 0x87e0c1000000 gser_base0:ffff00001d000000
[    6.059927] thunder_pcie_probe: ECAM0 CFG BASE 0x87e0c1000000 gser_base1:ffff00002a800000
[    6.068100] PCI host bridge /pem1@0x87e0,c1000000 ranges:
[    6.073496]   No bus range found for /pem1@0x87e0,c1000000, using [bus 00-ff]
[    6.080629]   MEM 0x885020000000..0x885027ffffff -> 0x20000000
[    6.086450]   MEM 0x886028000000..0x88602fffffff -> 0x28000000
[    6.092279]    IO 0x887000000000..0x887007ffffff -> 0x00000000
[    6.098098] Requested IO range too big, new size set to 64K
[    6.103665] I/O range found for /pem1@0x87e0,c1000000. Please provide an io_base pointer to save CPU base address
[    6.113963] thunder-pcie: probe of 87e0c1000000.pem1 failed with error -22
[    6.120998] thunder_pcie_probe: ECAM0 CFG BASE 0x87e0c2000000 gser_base0:ffff00001d000000
[    6.129161] thunder_pcie_probe: ECAM0 CFG BASE 0x87e0c2000000 gser_base1:ffff00002a800000
[    6.137334] PCI host bridge /pem2@0x87e0,c2000000 ranges:
[    6.142728]   No bus range found for /pem2@0x87e0,c2000000, using [bus 00-ff]
[    6.149853]   MEM 0x889030000000..0x889037ffffff -> 0x30000000
[    6.155683]   MEM 0x88a038000000..0x88a03fffffff -> 0x38000000
[    6.161512]    IO 0x88b000000000..0x88b007ffffff -> 0x00000000
[    6.167332] Requested IO range too big, new size set to 64K
[    6.172898] I/O range found for /pem2@0x87e0,c2000000. Please provide an io_base pointer to save CPU base address
[    6.183194] thunder-pcie: probe of 87e0c2000000.pem2 failed with error -22
[    6.190228] thunder_pcie_probe: ECAM0 CFG BASE 0x87e0c3000000 gser_base0:ffff00001d000000
[    6.198391] thunder_pcie_probe: ECAM0 CFG BASE 0x87e0c3000000 gser_base1:ffff00002a800000
[    6.206564] PCI host bridge /pem3@0x87e0,c3000000 ranges:
[    6.211959]   No bus range found for /pem3@0x87e0,c3000000, using [bus 00-ff]
[    6.219084]   MEM 0x891040000000..0x891047ffffff -> 0x40000000
[    6.224914]   MEM 0x892048000000..0x89204fffffff -> 0x48000000
[    6.230742]    IO 0x893000000000..0x893007ffffff -> 0x00000000
[    6.236561] Requested IO range too big, new size set to 64K
[    6.242128] I/O range found for /pem3@0x87e0,c3000000. Please provide an io_base pointer to save CPU base address
[    6.252423] thunder-pcie: probe of 87e0c3000000.pem3 failed with error -22
[    6.259442] thunder_pcie_probe: ECAM0 CFG BASE 0x87e0c4000000 gser_base0:ffff00001d000000
[    6.267621] thunder_pcie_probe: ECAM0 CFG BASE 0x87e0c4000000 gser_base1:ffff00002a800000
[    6.275792] PCI host bridge /pem4@0x87e0,c4000000 ranges:
[    6.281185]   No bus range found for /pem4@0x87e0,c4000000, using [bus 00-ff]
[    6.288311]   MEM 0x895050000000..0x895057ffffff -> 0x50000000
[    6.294140]   MEM 0x896058000000..0x89605fffffff -> 0x58000000
[    6.299961]    IO 0x897000000000..0x897007ffffff -> 0x00000000
[    6.305788] Requested IO range too big, new size set to 64K
[    6.311355] I/O range found for /pem4@0x87e0,c4000000. Please provide an io_base pointer to save CPU base address
[    6.321651] thunder-pcie: probe of 87e0c4000000.pem4 failed with error -22
[    6.328671] thunder_pcie_probe: ECAM0 CFG BASE 0x87e0c5000000 gser_base0:ffff00001d000000
[    6.336849] thunder_pcie_probe: ECAM0 CFG BASE 0x87e0c5000000 gser_base1:ffff00002a800000
[    6.345019] PCI host bridge /pem5@0x87e0,c5000000 ranges:
[    6.350413]   No bus range found for /pem5@0x87e0,c5000000, using [bus 00-ff]
[    6.357538]   MEM 0x899060000000..0x899067ffffff -> 0x60000000
[    6.363367]   MEM 0x89a068000000..0x89a06fffffff -> 0x68000000
[    6.369187]    IO 0x89b000000000..0x89b007ffffff -> 0x00000000
[    6.375014] Requested IO range too big, new size set to 64K
[    6.380580] I/O range found for /pem5@0x87e0,c5000000. Please provide an io_base pointer to save CPU base address
[    6.390876] thunder-pcie: probe of 87e0c5000000.pem5 failed with error -22
[    6.397895] thunder_pcie_probe: ECAM0 CFG BASE 0x97e0c0000000 gser_base0:ffff00001d000000
[    6.406074] thunder_pcie_probe: ECAM0 CFG BASE 0x97e0c0000000 gser_base1:ffff00002a800000
[    6.414245] PCI host bridge /pem6@0x97e0,c0000000 ranges:
[    6.419631]   No bus range found for /pem6@0x97e0,c0000000, using [bus 00-ff]
[    6.426764]   MEM 0x981008000000..0x98100fffffff -> 0x08000000
[    6.432593]   MEM 0x982010000000..0x982017ffffff -> 0x10000000
[    6.438414]    IO 0x983000000000..0x983007ffffff -> 0x00000000
[    6.444240] Requested IO range too big, new size set to 64K
[    6.449799] I/O range found for /pem6@0x97e0,c0000000. Please provide an io_base pointer to save CPU base address
[    6.460096] thunder-pcie: probe of 97e0c0000000.pem6 failed with error -22
[    6.467115] thunder_pcie_probe: ECAM0 CFG BASE 0x97e0c1000000 gser_base0:ffff00001d000000
[    6.475293] thunder_pcie_probe: ECAM0 CFG BASE 0x97e0c1000000 gser_base1:ffff00002a800000
[    6.483462] PCI host bridge /pem7@0x97e0,c1000000 ranges:
[    6.488849]   No bus range found for /pem7@0x97e0,c1000000, using [bus 00-ff]
[    6.495982]   MEM 0x985020000000..0x985027ffffff -> 0x20000000
[    6.501811]   MEM 0x986028000000..0x98602fffffff -> 0x28000000
[    6.507631]    IO 0x987000000000..0x987007ffffff -> 0x00000000
[    6.513459] Requested IO range too big, new size set to 64K
[    6.519019] I/O range found for /pem7@0x97e0,c1000000. Please provide an io_base pointer to save CPU base address
[    6.529315] thunder-pcie: probe of 97e0c1000000.pem7 failed with error -22
[    6.536349] thunder_pcie_probe: ECAM0 CFG BASE 0x97e0c2000000 gser_base0:ffff00001d000000
[    6.544522] thunder_pcie_probe: ECAM0 CFG BASE 0x97e0c2000000 gser_base1:ffff00002a800000
[    6.552692] PCI host bridge /pem8@0x97e0,c2000000 ranges:
[    6.558078]   No bus range found for /pem8@0x97e0,c2000000, using [bus 00-ff]
[    6.565212]   MEM 0x989030000000..0x989037ffffff -> 0x30000000
[    6.571041]   MEM 0x98a038000000..0x98a03fffffff -> 0x38000000
[    6.576862]    IO 0x98b000000000..0x98b007ffffff -> 0x00000000
[    6.582688] Requested IO range too big, new size set to 64K
[    6.588247] I/O range found for /pem8@0x97e0,c2000000. Please provide an io_base pointer to save CPU base address
[    6.598543] thunder-pcie: probe of 97e0c2000000.pem8 failed with error -22
[    6.605577] thunder_pcie_probe: ECAM0 CFG BASE 0x97e0c3000000 gser_base0:ffff00001d000000
[    6.613752] thunder_pcie_probe: ECAM0 CFG BASE 0x97e0c3000000 gser_base1:ffff00002a800000
[    6.621922] PCI host bridge /pem9@0x97e0,c3000000 ranges:
[    6.627308]   No bus range found for /pem9@0x97e0,c3000000, using [bus 00-ff]
[    6.634440]   MEM 0x991040000000..0x991047ffffff -> 0x40000000
[    6.640269]   MEM 0x992048000000..0x99204fffffff -> 0x48000000
[    6.646090]    IO 0x993000000000..0x993007ffffff -> 0x00000000
[    6.651917] Requested IO range too big, new size set to 64K
[    6.657476] I/O range found for /pem9@0x97e0,c3000000. Please provide an io_base pointer to save CPU base address
[    6.667773] thunder-pcie: probe of 97e0c3000000.pem9 failed with error -22
[    6.674806] thunder_pcie_probe: ECAM0 CFG BASE 0x97e0c4000000 gser_base0:ffff00001d000000
[    6.682979] thunder_pcie_probe: ECAM0 CFG BASE 0x97e0c4000000 gser_base1:ffff00002a800000
[    6.691149] PCI host bridge /pem10@0x97e0,c4000000 ranges:
[    6.696622]   No bus range found for /pem10@0x97e0,c4000000, using [bus 00-ff]
[    6.703842]   MEM 0x995050000000..0x995057ffffff -> 0x50000000
[    6.709663]   MEM 0x996058000000..0x99605fffffff -> 0x58000000
[    6.715492]    IO 0x997000000000..0x997007ffffff -> 0x00000000
[    6.721319] Requested IO range too big, new size set to 64K
[    6.726878] I/O range found for /pem10@0x97e0,c4000000. Please provide an io_base pointer to save CPU base address
[    6.737261] thunder-pcie: probe of 97e0c4000000.pem10 failed with error -22
[    6.744382] thunder_pcie_probe: ECAM0 CFG BASE 0x97e0c5000000 gser_base0:ffff00001d000000
[    6.752555] thunder_pcie_probe: ECAM0 CFG BASE 0x97e0c5000000 gser_base1:ffff00002a800000
[    6.760726] PCI host bridge /pem11@0x97e0,c5000000 ranges:
[    6.766199]   No bus range found for /pem11@0x97e0,c5000000, using [bus 00-ff]
[    6.773418]   MEM 0x999060000000..0x999067ffffff -> 0x60000000
[    6.779240]   MEM 0x99a068000000..0x99a06fffffff -> 0x68000000
[    6.785068]    IO 0x99b000000000..0x99b007ffffff -> 0x00000000
[    6.790896] Requested IO range too big, new size set to 64K
[    6.796455] I/O range found for /pem11@0x97e0,c5000000. Please provide an io_base pointer to save CPU base address
[    6.806838] thunder-pcie: probe of 97e0c5000000.pem11 failed with error -22
[    6.814142] gti: thunderx-gti, ver 1.0
[    6.818323] Serial: 8250/16550 driver, 32 ports, IRQ sharing enabled
[    6.828229] of_dma_request_slave_channel: dma-names property of node '/soc/serial@87e0,24000000' missing or empty
[    6.838492] uart-pl011 87e024000000.serial: no DMA platform data
[    6.844494] of_dma_request_slave_channel: dma-names property of node '/soc/serial@87e0,25000000' missing or empty
[    6.854746] uart-pl011 87e025000000.serial: no DMA platform data
[    6.865855] brd: module loaded
[    6.871395] loop: module loaded
[    6.875141] ahci 0001:00:08.0: SSS flag set, parallel bus scan disabled
[    6.881780] ahci 0001:00:08.0: AHCI 0001.0300 32 slots 1 ports 6 Gbps 0x1 impl SATA mode
[    6.889860] ahci 0001:00:08.0: flags: 64bit ncq sntf ilck stag pm led clo only pmp fbs pio slum part ccc apst 
[    6.899857] ahci 0001:00:08.0: port 0 is not capable of FBS
[    6.905930] scsi host0: ahci
[    6.908992] ata1: SATA max UDMA/133 abar m2097152@0x814000000000 port 0x814000000100 irq 5
[    6.917394] ahci 0001:00:09.0: SSS flag set, parallel bus scan disabled
[    6.924017] ahci 0001:00:09.0: AHCI 0001.0300 32 slots 1 ports 6 Gbps 0x1 impl SATA mode
[    6.932100] ahci 0001:00:09.0: flags: 64bit ncq sntf ilck stag pm led clo only pmp fbs pio slum part ccc apst 
[    6.942093] ahci 0001:00:09.0: port 0 is not capable of FBS
[    6.948105] scsi host1: ahci
[    6.951109] ata2: SATA max UDMA/133 abar m2097152@0x815000000000 port 0x815000000100 irq 32
[    6.959572] ahci 0001:00:0a.0: SSS flag set, parallel bus scan disabled
[    6.966196] ahci 0001:00:0a.0: AHCI 0001.0300 32 slots 1 ports 6 Gbps 0x1 impl SATA mode
[    6.974284] ahci 0001:00:0a.0: flags: 64bit ncq sntf ilck stag pm led clo only pmp fbs pio slum part ccc apst 
[    6.984279] ahci 0001:00:0a.0: port 0 is not capable of FBS
[    6.990290] scsi host2: ahci
[    6.993282] ata3: SATA max UDMA/133 abar m2097152@0x816000000000 port 0x816000000100 irq 6
[    7.001674] ahci 0001:00:0b.0: SSS flag set, parallel bus scan disabled
[    7.008284] ahci 0001:00:0b.0: AHCI 0001.0300 32 slots 1 ports 6 Gbps 0x1 impl SATA mode
[    7.016377] ahci 0001:00:0b.0: flags: 64bit ncq sntf ilck stag pm led clo only pmp fbs pio slum part ccc apst 
[    7.026370] ahci 0001:00:0b.0: port 0 is not capable of FBS
[    7.032373] scsi host3: ahci
[    7.035364] ata4: SATA max UDMA/133 abar m2097152@0x817000000000 port 0x817000000100 irq 33
[    7.043886] ahci 0005:00:08.0: SSS flag set, parallel bus scan disabled
[    7.050514] ahci 0005:00:08.0: AHCI 0001.0300 32 slots 1 ports 6 Gbps 0x1 impl SATA mode
[    7.058592] ahci 0005:00:08.0: flags: 64bit ncq sntf ilck stag pm led clo only pmp fbs pio slum part ccc apst 
[    7.068585] ahci 0005:00:08.0: port 0 is not capable of FBS
[    7.074607] scsi host4: ahci
[    7.077599] ata5: SATA max UDMA/133 abar m2097152@0x914000000000 port 0x914000000100 irq 7
[    7.086003] ahci 0005:00:09.0: SSS flag set, parallel bus scan disabled
[    7.092631] ahci 0005:00:09.0: AHCI 0001.0300 32 slots 1 ports 6 Gbps 0x1 impl SATA mode
[    7.100714] ahci 0005:00:09.0: flags: 64bit ncq sntf ilck stag pm led clo only pmp fbs pio slum part ccc apst 
[    7.110707] ahci 0005:00:09.0: port 0 is not capable of FBS
[    7.116714] scsi host5: ahci
[    7.119705] ata6: SATA max UDMA/133 abar m2097152@0x915000000000 port 0x915000000100 irq 34
[    7.128198] ahci 0005:00:0a.0: SSS flag set, parallel bus scan disabled
[    7.134826] ahci 0005:00:0a.0: AHCI 0001.0300 32 slots 1 ports 6 Gbps 0x1 impl SATA mode
[    7.142909] ahci 0005:00:0a.0: flags: 64bit ncq sntf ilck stag pm led clo only pmp fbs pio slum part ccc apst 
[    7.152902] ahci 0005:00:0a.0: port 0 is not capable of FBS
[    7.158903] scsi host6: ahci
[    7.161905] ata7: SATA max UDMA/133 abar m2097152@0x916000000000 port 0x916000000100 irq 8
[    7.170307] ahci 0005:00:0b.0: SSS flag set, parallel bus scan disabled
[    7.176922] ahci 0005:00:0b.0: AHCI 0001.0300 32 slots 1 ports 6 Gbps 0x1 impl SATA mode
[    7.185014] ahci 0005:00:0b.0: flags: 64bit ncq sntf ilck stag pm led clo only pmp fbs pio slum part ccc apst 
[    7.195006] ahci 0005:00:0b.0: port 0 is not capable of FBS
[    7.201017] scsi host7: ahci
[    7.204009] ata8: SATA max UDMA/133 abar m2097152@0x917000000000 port 0x917000000100 irq 35
[    7.212903] libphy: Fixed MDIO Bus: probed
[    7.217108] libphy: mdio-octeon: probed
[    7.221529] mdio-octeon 87e005003800.mdio: Version 1.0
[    7.226716] libphy: mdio-octeon: probed
[    7.230599] mdio-octeon 87e005003880.mdio: Version 1.0
[    7.235789] libphy: mdio-octeon: probed
[    7.239657] mdio-octeon 97e005003800.mdio: Version 1.0
[    7.244848] libphy: mdio-octeon: probed
[    7.248674] mdio-octeon 97e005003880.mdio: Version 1.0
[    7.253824] tun: Universal TUN/TAP device driver, 1.6
[    7.258864] tun: (C) 1999-2004 Max Krasnyansky <maxk@qualcomm.com>
[    7.260020] ata1: SATA link down (SStatus 0 SControl 300)
[    7.270522] thunder-BGX, ver 1.0
[    7.273812] thunder-BGX 0000:01:10.0: BGX0 QLM mode: XFI
[    7.279329] thunder-BGX 0000:01:10.1: BGX1 QLM mode: XLAUI
[    7.285044] thunder-BGX 0004:01:10.0: BGX2 QLM mode: XFI
[    7.290560] thunder-nic, ver 1.0
[    7.350035] ata3: SATA link down (SStatus 0 SControl 300)
[    7.390029] ata4: SATA link down (SStatus 0 SControl 300)
[    7.400407] thunder-nic 0002:01:00.0: SRIOV enabled, numer of VF available 2
[    7.430018] ata5: SATA link down (SStatus 0 SControl 300)
[    7.470017] ata6: SATA link down (SStatus 0 SControl 300)
[    7.500053] ata2: SATA link up 6.0 Gbps (SStatus 133 SControl 300)
[    7.507847] ata2.00: ATA-8: WDC WD4000FYYZ-01UL1B2, 01.01K03, max UDMA/133
[    7.514728] ata2.00: 7814037168 sectors, multi 0: LBA48 NCQ (depth 31/32), AA
[    7.520016] ata7: SATA link down (SStatus 0 SControl 300)
[    7.527441] thunder-nic 0006:01:00.0: SRIOV enabled, numer of VF available 1
[    7.534629] ata2.00: configured for UDMA/133
[    7.538988] thunder-nicvf, ver 1.0
[    7.539134] scsi 1:0:0:0: Direct-Access     ATA      WDC WD4000FYYZ-0 1K03 PQ: 0 ANSI: 5
[    7.539613] sd 1:0:0:0: [sda] 7814037168 512-byte logical blocks: (4.00 TB/3.63 TiB)
[    7.539728] sd 1:0:0:0: Attached scsi generic sg0 type 0
[    7.539917] sd 1:0:0:0: [sda] Write Protect is off
[    7.540066] sd 1:0:0:0: [sda] Write cache: enabled, read cache: enabled, doesn't support DPO or FUA
[    7.560024] ata8: SATA link down (SStatus 0 SControl 300)
[    7.570358]  sda: sda1 sda2 sda3
[    7.571189] sd 1:0:0:0: [sda] Attached SCSI disk
[    7.590644] thunder-nicvf 0002:01:00.1: enabling device (0004 -> 0006)
[    7.620523] thunder-nicvf 0002:01:00.2: enabling device (0004 -> 0006)
[    7.650501] thunder-nicvf 0006:01:00.1: enabling device (0004 -> 0006)
[    7.680539] PPP generic driver version 2.4.2
[    7.685044] VFIO - User Level meta-driver version: 0.3
[    7.690834] xhci_hcd 0000:00:10.0: xHCI Host Controller
[    7.696059] xhci_hcd 0000:00:10.0: new USB bus registered, assigned bus number 1
[    7.703915] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002
[    7.710704] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    7.717914] usb usb1: Product: xHCI Host Controller
[    7.722784] usb usb1: Manufacturer: Linux 3.18.0-gdcccbc1-dirty xhci-hcd
[    7.729472] usb usb1: SerialNumber: 0000:00:10.0
[    7.734399] hub 1-0:1.0: USB hub found
[    7.738170] hub 1-0:1.0: 1 port detected
[    7.742401] xhci_hcd 0000:00:10.0: xHCI Host Controller
[    7.747622] xhci_hcd 0000:00:10.0: new USB bus registered, assigned bus number 2
[    7.755393] usb usb2: New USB device found, idVendor=1d6b, idProduct=0003
[    7.762185] usb usb2: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    7.769395] usb usb2: Product: xHCI Host Controller
[    7.774270] usb usb2: Manufacturer: Linux 3.18.0-gdcccbc1-dirty xhci-hcd
[    7.780965] usb usb2: SerialNumber: 0000:00:10.0
[    7.785938] hub 2-0:1.0: USB hub found
[    7.789717] hub 2-0:1.0: 1 port detected
[    7.794000] xhci_hcd 0000:00:11.0: xHCI Host Controller
[    7.799225] xhci_hcd 0000:00:11.0: new USB bus registered, assigned bus number 3
[    7.807225] usb usb3: New USB device found, idVendor=1d6b, idProduct=0002
[    7.814015] usb usb3: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    7.821230] usb usb3: Product: xHCI Host Controller
[    7.826095] usb usb3: Manufacturer: Linux 3.18.0-gdcccbc1-dirty xhci-hcd
[    7.832787] usb usb3: SerialNumber: 0000:00:11.0
[    7.837712] hub 3-0:1.0: USB hub found
[    7.841491] hub 3-0:1.0: 1 port detected
[    7.845693] xhci_hcd 0000:00:11.0: xHCI Host Controller
[    7.850924] xhci_hcd 0000:00:11.0: new USB bus registered, assigned bus number 4
[    7.858523] usb usb4: New USB device found, idVendor=1d6b, idProduct=0003
[    7.865308] usb usb4: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    7.872521] usb usb4: Product: xHCI Host Controller
[    7.877387] usb usb4: Manufacturer: Linux 3.18.0-gdcccbc1-dirty xhci-hcd
[    7.884078] usb usb4: SerialNumber: 0000:00:11.0
[    7.888972] hub 4-0:1.0: USB hub found
[    7.892750] hub 4-0:1.0: 1 port detected
[    7.896917] xhci_hcd 0004:00:10.0: xHCI Host Controller
[    7.902150] xhci_hcd 0004:00:10.0: new USB bus registered, assigned bus number 5
[    7.909980] usb usb5: New USB device found, idVendor=1d6b, idProduct=0002
[    7.916767] usb usb5: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    7.923983] usb usb5: Product: xHCI Host Controller
[    7.928849] usb usb5: Manufacturer: Linux 3.18.0-gdcccbc1-dirty xhci-hcd
[    7.935541] usb usb5: SerialNumber: 0004:00:10.0
[    7.940445] hub 5-0:1.0: USB hub found
[    7.944214] hub 5-0:1.0: 1 port detected
[    7.948406] xhci_hcd 0004:00:10.0: xHCI Host Controller
[    7.953637] xhci_hcd 0004:00:10.0: new USB bus registered, assigned bus number 6
[    7.961207] usb usb6: New USB device found, idVendor=1d6b, idProduct=0003
[    7.967984] usb usb6: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    7.975201] usb usb6: Product: xHCI Host Controller
[    7.980072] usb usb6: Manufacturer: Linux 3.18.0-gdcccbc1-dirty xhci-hcd
[    7.986760] usb usb6: SerialNumber: 0004:00:10.0
[    7.991665] hub 6-0:1.0: USB hub found
[    7.995434] hub 6-0:1.0: 1 port detected
[    8.000361] xhci_hcd 0004:00:11.0: xHCI Host Controller
[    8.005586] xhci_hcd 0004:00:11.0: new USB bus registered, assigned bus number 7
[    8.013422] usb usb7: New USB device found, idVendor=1d6b, idProduct=0002
[    8.020211] usb usb7: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    8.027421] usb usb7: Product: xHCI Host Controller
[    8.032291] usb usb7: Manufacturer: Linux 3.18.0-gdcccbc1-dirty xhci-hcd
[    8.038979] usb usb7: SerialNumber: 0004:00:11.0
[    8.043878] hub 7-0:1.0: USB hub found
[    8.047642] hub 7-0:1.0: 1 port detected
[    8.051853] xhci_hcd 0004:00:11.0: xHCI Host Controller
[    8.057073] xhci_hcd 0004:00:11.0: new USB bus registered, assigned bus number 8
[    8.064654] usb usb8: New USB device found, idVendor=1d6b, idProduct=0003
[    8.071439] usb usb8: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    8.078649] usb usb8: Product: xHCI Host Controller
[    8.083520] usb usb8: Manufacturer: Linux 3.18.0-gdcccbc1-dirty xhci-hcd
[    8.090212] usb usb8: SerialNumber: 0004:00:11.0
[    8.095124] hub 8-0:1.0: USB hub found
[    8.098886] hub 8-0:1.0: 1 port detected
[    8.103813] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    8.110375] ehci-pci: EHCI PCI platform driver
[    8.114869] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    8.121054] ohci-pci: OHCI PCI platform driver
[    8.125544] uhci_hcd: USB Universal Host Controller Interface driver
[    8.132115] usbcore: registered new interface driver usb-storage
[    8.138294] mousedev: PS/2 mouse device common for all mice
[    8.145036] device-mapper: uevent: version 1.0.3
[    8.149963] device-mapper: ioctl: 4.28.0-ioctl (2014-09-17) initialised: dm-devel@redhat.com
[    8.159038] ledtrig-cpu: registered to indicate activity on CPUs
[    8.160050] usb 3-1: new high-speed USB device number 2 using xhci_hcd
[    8.171561] EFI Variables Facility v0.08 2004-May-17
[    8.176759] pstore: Registered efi as persistent store backend
[    8.182776] TCP: cubic registered
[    8.186091] NET: Registered protocol family 17
[    8.190595] Key type dns_resolver registered
[    8.195229] Loading compiled-in X.509 certificates
[    8.201682] Loaded X.509 cert 'Magrathea: Glacier signing key: 8f5d3440f3cec1b6a1a857b85f38455ec45a6b96'
[    8.211176] registered taskstats version 1
[    8.216213] Key type trusted registered
[    8.220831] Key type encrypted registered
[    8.225602] AppArmor: AppArmor sha1 policy hashing enabled
[    8.231093] ima: No TPM chip found, activating TPM-bypass!
[    8.236612] evm: HMAC attrs: 0x1
[    8.240828] drivers/rtc/hctosys.c: unable to open rtc device (rtc0)
[    8.247425] md: Waiting for all devices to be available before autodetect
[    8.254212] md: If you don't use raid, use raid=noautodetect
[    8.260309] md: Autodetecting RAID arrays.
[    8.264394] md: Scanned 0 and added 0 devices.
[    8.268824] md: autorun ...
[    8.271617] md: ... autorun DONE.
[    8.293522] EXT3-fs (sda3): error: couldn't mount because of unsupported optional features (240)
[    8.302591] EXT2-fs (sda3): error: couldn't mount because of unsupported optional features (240)
[    8.320943] usb 3-1: New USB device found, idVendor=0b95, idProduct=772b
[    8.327633] usb 3-1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[    8.334764] usb 3-1: Product: AX88772C
[    8.338502] usb 3-1: Manufacturer: ASIX            
[    8.343373] usb 3-1: SerialNumber: 0000C2
[    8.620550] EXT4-fs (sda3): mounted filesystem with ordered data mode. Opts: (null)
[    8.628210] VFS: Mounted root (ext4 filesystem) on device 8:3.
[    8.647294] devtmpfs: mounted
[    8.650304] Freeing unused kernel memory: 832K (ffff800000d90000 - ffff800000e60000)
[    9.010755] random: systemd urandom read with 36 bits of entropy available
[    9.021374] systemd[1]: systemd 216 running in system mode. (+PAM +AUDIT +SELINUX +IMA -APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ -LZ4 -SECCOMP +BLKID +ELFUTILS +KMOD +IDN)
[    9.039536] systemd[1]: Detected architecture 'arm64'.
Welcome to Fedora 21 (Twenty One)!
[    9.256111] NET: Registered protocol family 10
[    9.262111] systemd[1]: Inserted module 'ipv6'
[    9.266929] systemd[1]: Set hostname to <Tfedora>.
[   10.125066] systemd[1]: Configuration file /usr/lib/systemd/system/auditd.service is marked world-inaccessible. This has no effect as configuration data is accessible via APIs without restrictions. Proceeding anyway.
[   10.219054] systemd[1]: Expecting device dev-ttyAMA0.device...
         Expecting device dev-ttyAMA0.device...
[   10.240108] systemd[1]: Starting Forward Password Requests to Wall Directory Watch.
[   10.247876] systemd[1]: Started Forward Password Requests to Wall Directory Watch.
[   10.255514] systemd[1]: Starting Arbitrary Executable File Formats File System Automount Point.
[  OK  ] Set up automount Arbitrary Executab...ats File System Automount Point.
[   10.290080] systemd[1]: Set up automount Arbitrary Executable File Formats File System Automount Point.
[   10.299491] systemd[1]: Starting Swap.
[  OK  ] Reached target Swap.
[   10.320078] systemd[1]: Reached target Swap.
[   10.324358] systemd[1]: Starting Root Slice.
[  OK  ] Created slice Root Slice.
[   10.550088] systemd[1]: Created slice Root Slice.
[   10.554803] systemd[1]: Starting /dev/initctl Compatibility Named Pipe.
[  OK  ] Listening on /dev/initctl Compatibility Named Pipe.
[   10.580100] systemd[1]: Listening on /dev/initctl Compatibility Named Pipe.
[   10.587068] systemd[1]: Starting Delayed Shutdown Socket.
[  OK  ] Listening on Delayed Shutdown Socket.
[   10.610060] systemd[1]: Listening on Delayed Shutdown Socket.
[   10.615814] systemd[1]: Starting Device-mapper event daemon FIFOs.
[  OK  ] Listening on Device-mapper event daemon FIFOs.
[   10.640070] systemd[1]: Listening on Device-mapper event daemon FIFOs.
[   10.646606] systemd[1]: Starting LVM2 metadata daemon socket.
[  OK  ] Listening on LVM2 metadata daemon socket.
[   10.670081] systemd[1]: Listening on LVM2 metadata daemon socket.
[   10.676195] systemd[1]: Starting udev Control Socket.
[  OK  ] Listening on udev Control Socket.
[   10.700075] systemd[1]: Listening on udev Control Socket.
[   10.705495] systemd[1]: Starting udev Kernel Socket.
[  OK  ] Listening on udev Kernel Socket.
[   10.730061] systemd[1]: Listening on udev Kernel Socket.
[   10.735382] systemd[1]: Starting User and Session Slice.
[  OK  ] Created slice User and Session Slice.
[   10.760078] systemd[1]: Created slice User and Session Slice.
[   10.765841] systemd[1]: Starting Journal Socket.
[  OK  ] Listening on Journal Socket.
[   10.790070] systemd[1]: Listening on Journal Socket.
[   10.795075] systemd[1]: Starting System Slice.
[  OK  ] Created slice System Slice.
[   10.820075] systemd[1]: Created slice System Slice.
[   10.824974] systemd[1]: Starting Journal Socket (/dev/log).
[   10.831793] systemd[1]: Starting Monitoring of LVM2 mirrors, snapshots etc. using dmeventd or progress polling...
         Starting Monitoring of LVM2 mirrors... dmeventd or progress polling...
[   10.861608] systemd[1]: Started Device-Mapper Multipath Device Controller.
[   10.868663] systemd[1]: Mounting Debug File System...
         Mounting Debug File System...
[   10.891344] systemd[1]: Starting udev Coldplug all Devices...
         Starting udev Coldplug all Devices...
[   10.911484] systemd[1]: Mounting POSIX Message Queue File System...
         Mounting POSIX Message Queue File System...
[   10.931616] systemd[1]: Mounting Huge Pages File System...
         Mounting Huge Pages File System...
[   10.951613] systemd[1]: Starting Create list of required static device nodes for the current kernel...
         Starting Create list of required st... nodes for the current kernel...
[   10.981539] systemd[1]: Mounting RPC Pipe File System...
         Mounting RPC Pipe File System...
[   11.114963] systemd[1]: Starting system-getty.slice.
[  OK  ] Created slice system-getty.slice.
[   11.140107] systemd[1]: Created slice system-getty.slice.
[   11.145528] systemd[1]: Starting system-serial\x2dgetty.slice.
[  OK  ] Created slice system-serial\x2dgetty.slice.
[   11.170085] systemd[1]: Created slice system-serial\x2dgetty.slice.
[   11.176414] systemd[1]: Starting Collect Read-Ahead Data...
         Starting Collect Read-Ahead Data...
[   11.201539] systemd[1]: Starting Replay Read-Ahead Data...
         Starting Replay Read-Ahead Data...
[   11.221533] systemd[1]: Starting Slices.
[   11.224310] systemd-readahead[660]: Bumped block_nr parameter of 8:0 to 20480. This is a temporary hack and should be removed one day.
[  OK  ] Reached target Slices.
[   11.250108] systemd[1]: Reached target Slices.
[   11.262437] systemd[1]: Mounting Temporary Directory...
         Mounting Temporary Directory...
[   11.292712] systemd[1]: tmp.mount: Directory /tmp to mount over is not empty, mounting anyway.
[  OK  ] Mounted RPC Pipe File System.
[   11.320071] systemd[1]: Mounted RPC Pipe File System.
[  OK  ] Mounted Huge Pages File System.
[   11.340066] systemd[1]: Mounted Huge Pages File System.
[  OK  ] Mounted POSIX Message Queue File System.
[   11.360058] systemd[1]: Mounted POSIX Message Queue File System.
[  OK  ] Mounted Debug File System.
[   11.380067] systemd[1]: Mounted Debug File System.
[  OK  ] Mounted Temporary Directory.
[   11.400087] systemd[1]: Mounted Temporary Directory.
[  OK  ] Started Collect Read-Ahead Data.
[   11.420068] systemd[1]: Started Collect Read-Ahead Data.
[  OK  ] Started Replay Read-Ahead Data.
[   11.440058] systemd[1]: Started Replay Read-Ahead Data.
[  OK  ] Listening on Journal Socket (/dev/log).
[   11.460070] systemd[1]: Listening on Journal Socket (/dev/log).
[  OK  ] Started Create list of required sta...ce nodes for the current kernel.
[   11.490068] systemd[1]: Started Create list of required static device nodes for the current kernel.
[  OK  ] Started udev Coldplug all Devices.
[   11.530092] systemd[1]: Started udev Coldplug all Devices.
[   11.551726] systemd[1]: Starting udev Wait for Complete Device Initialization...
         Starting udev Wait for Complete Device Initialization...
[   11.581566] systemd[1]: Starting LVM2 metadata daemon...
         Starting LVM2 metadata daemon...
[  OK  ] Started LVM2 metadata daemon.
[   11.620072] systemd[1]: Started LVM2 metadata daemon.
[   11.625229] systemd[1]: Starting Journal Service...
         Starting Journal Service...
[   11.651539] systemd[1]: Starting Remount Root and Kernel File Systems...
         Starting Remount Root and Kernel File Systems...
[   11.681565] systemd[1]: Starting Setup Virtual Console...
         Starting Setup Virtual Console...
[   12.709128] systemd[1]: Started Set Up Additional Binary Formats.
[   12.729705] systemd[1]: Started Load legacy module configuration.
[   12.759745] systemd[1]: Started Load Kernel Modules.
[   12.764770] systemd[1]: Starting Apply Kernel Variables...
         Starting Apply Kernel Variables...
[   12.792215] systemd[1]: Mounted Configuration File System.
[   12.797765] systemd[1]: Mounting FUSE Control File System...
         Mounting FUSE Control File System...
[  OK  ] Started Remount Root and Kernel File Systems.
[   12.840090] systemd[1]: Started Remount Root and Kernel File Systems.
[  OK  ] Started Apply Kernel Variables.
[   12.870079] systemd[1]: Started Apply Kernel Variables.
[  OK  ] Mounted FUSE Control File System.
[   12.890142] systemd[1]: Mounted FUSE Control File System.
[   12.904405] systemd[1]: Started Import network configuration from initramfs.
[   12.911518] systemd[1]: Started First Boot Wizard.
[   12.916317] systemd[1]: Starting Load/Save Random Seed...
         Starting Load/Save Random Seed...
[   12.960670] systemd[1]: Started Rebuild Hardware Database.
[   12.966200] systemd[1]: Started Create System Users.
[   12.971299] systemd[1]: Starting Create Static Device Nodes in /dev...
         Starting Create Static Device Nodes in /dev...
[   12.991497] systemd[1]: Started Rebuild Dynamic Linker Cache.
[   12.997297] systemd[1]: Starting Configure read-only root support...
         Starting Configure read-only root support...
[  OK  ] Started Monitoring of LVM2 mirrors,...ng dmeventd or progress polling.
[   13.040112] systemd[1]: Started Monitoring of LVM2 mirrors, snapshots etc. using dmeventd or progress polling.
[  OK  ] Started Setup Virtual Console.
[   13.070074] systemd[1]: Started Setup Virtual Console.
[  OK  ] Started Load/Save Random Seed.
[   13.090069] systemd[1]: Started Load/Save Random Seed.
[  OK  ] Started Create Static Device Nodes in /dev.
[   13.110106] systemd[1]: Started Create Static Device Nodes in /dev.
[   13.127670] systemd[1]: Starting udev Kernel Device Manager...
         Starting udev Kernel Device Manager...
[   13.151606] systemd[1]: Starting Local File Systems (Pre).
[  OK  ] Reached target Local File Systems (Pre).
[   13.180071] systemd[1]: Reached target Local File Systems (Pre).
[   13.186152] systemd[1]: Mounting NFSD configuration filesystem...
         Mounting NFSD configuration filesystem...
[   13.211506] systemd[1]: Starting Show Plymouth Boot Screen...
         Starting Show Plymouth Boot Screen...
[   13.243976] Installing knfsd (copyright (C) 1996 okir@monad.swb.de).
[   13.252106] systemd[1]: Mounted NFSD configuration filesystem.
[   13.359144] systemd[1]: Started Configure read-only root support.
[   14.063882] systemd[1]: Started udev Kernel Device Manager.
[   14.347716] systemd[1]: Found device /dev/ttyAMA0.
[   14.543916] shpchp: Standard Hot Plug PCI Controller Driver version: 0.4
[   14.676676] thunder-nicvf 0002:01:00.2 enP2p1s0f2: renamed from eth1
[   14.870703] thunder-nicvf 0002:01:00.1 enP2p1s0f1: renamed from eth0
[   14.950492] thunder-nicvf 0006:01:00.1 enP6p1s0f1: renamed from eth2
[   15.884562] systemd[1]: Started Journal Service.
[   15.898456] systemd-journald[673]: Received request to flush runtime journal from PID 1
[   16.161623] random: nonblocking pool is initialized
[   17.441694] asix 3-1:1.0 eth0: register 'asix' at usb-0000:00:11.0-1, ASIX AX88772B USB 2.0 Ethernet, f0:1e:34:00:27:3e
[   17.452539] usbcore: registered new interface driver asix
[   17.477546] asix 3-1:1.0 enp0s17u1: renamed from eth0
ÿ[  OK  ] Mounted NFSD configuration filesystem.
[  OK  ] Started Configure read-only root support.
[  OK  ] Started udev Kernel Device Manager.
[  OK  ] Found device /dev/ttyAMA0.
G[  OK  ] Started Journal Service.
         Starting Flush Journal to Persistent Storage...
[  OK  ] Started Flush Journal to Persistent Storage.
[  OK  ] Started udev Wait for Complete Device Initialization.
         Starting Activation of DM RAID sets...
[  OK  ] Started Show Plymouth Boot Screen.
[  OK  ] Reached target Paths.
[  OK  ] Started Activation of DM RAID sets.
[  OK  ] Reached target Local File Systems.
         Starting Create Volatile Files and Directories...
         Starting Tell Plymouth To Write Out Runtime Data...
[  OK  ] Reached target Encrypted Volumes.
[  OK  ] Started Tell Plymouth To Write Out Runtime Data.
[  OK  ] Started Create Volatile Files and Directories.
         Starting Security Auditing Service...
[   17.790258] audit: type=1305 audit(17.790:2): audit_pid=957 old=0 auid=4294967295 ses=4294967295 res=1
[  OK  ] Started Security Auditing Service.
         Starting Update UTMP about System Boot/Shutdown...
[  OK  ] Started Update UTMP about System Boot/Shutdown.
[  OK  ] Reached target System Initialization.
[  OK  ] Listening on Cockpit Web Server Socket.
[  OK  ] Listening on RPCbind Server Activation Socket.
[  OK  ] Listening on PC/SC Smart Card Daemon Activation Socket.
[  OK  ] Reached target Timers.
[  OK  ] Listening on D-Bus System Message Bus Socket.
[  OK  ] Reached target Sockets.
[  OK  ] Reached target Basic System.
         Starting Hardware RNG Entropy Gatherer Daemon...
[  OK  ] Started Hardware RNG Entropy Gatherer Daemon.
         Starting Resets System Activity Logs...
         Starting ABRT Automated Bug Reporting Tool...
[  OK  ] Started ABRT Automated Bug Reporting Tool.
         Starting ABRT kernel log watcher...
[  OK  ] Started ABRT kernel log watcher.
         Starting Install ABRT coredump hook...
         Starting Network Manager...
         Starting Preprocess NFS configuration...
         Starting GSSAPI Proxy Daemon...
         Starting Self Monitoring and Reporting Technology (SMART) Daemon...
[  OK  ] Started Self Monitoring and Reporting Technology (SMART) Daemon.
         Starting irqbalance daemon...
[  OK  ] Started irqbalance daemon.
         Starting System Logging Service...
         Starting NTP client/server...
         Starting D-Bus System Message Bus...
[  OK  ] Started D-Bus System Message Bus.
         Starting Login Service...
[  OK  ] Started Resets System Activity Logs.
[  OK  ] Started Install ABRT coredump hook.
[  OK  ] Started Preprocess NFS configuration.
[  OK  ] Started GSSAPI Proxy Daemon.
[  OK  ] Started System Logging Service.
[  OK  ] Started Login Service.
[  OK  ] Started NTP client/server.
[  OK  ] Started Network Manager.
[  OK  ] Reached target Network.
         Starting rolekit - role server...
         Starting Notify NFS peers of a restart...
         Starting OpenSSH server daemon...
[  OK  ] Started OpenSSH server daemon.
[  OK  ] Started Notify NFS peers of a restart.
[  OK  ] Reached target NFS client services.
[  OK  ] Reached target Remote File Systems (Pre).
[  OK  ] Reached target Remote File Systems.
         Starting Permit User Sessions...
[  OK  ] Started Permit User Sessions.
         Starting Job spooling tools...
[  OK  ] Started Job spooling tools.
         Starting Terminate Plymouth Boot Screen...
         Starting Wait for Plymouth Boot Screen to Quit...
Fedora release 21 (Twenty One)
Kernel 3.18.0-gdcccbc1-dirty on an aarch64 (ttyAMA0)
Tfedora login: 
Fedora release 21 (Twenty One)
Kernel 3.18.0-gdcccbc1-dirty on an aarch64 (ttyAMA0)
```
