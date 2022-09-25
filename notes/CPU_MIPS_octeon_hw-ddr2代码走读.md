- [bootoct 0x20000000 coremask=0x0f endbootargs -ecc](#bootoct-0x20000000-coremask0x0f-endbootargs--ecc)
- [hw-ddr2](#hw-ddr2)
- [用CVMX_SHARED声明的变量是共享的, 其他核直接用](#用cvmx_shared声明的变量是共享的-其他核直接用)
- [但怎么调到这个函数来的? 但肯定在main之前](#但怎么调到这个函数来的-但肯定在main之前)
- [A真正的中断处理函数](#a真正的中断处理函数)
- [然后是main](#然后是main)

# bootoct 0x20000000 coremask=0x0f endbootargs -ecc
```c
    do_bootocteon()
        setup_se_app_boot_info()
        load_elf(addr, argc, argv, stack_size, heap_size, &core_mask, -1);
            read_elf_info(ei, elf_addr);
            为每个core建立tlb, 建立堆和栈
            拷贝只读的segment, 一份
            拷贝共享的segment, 一份
            拷贝可写的segment, 每个core一份
            octeon_setup_boot_desc_block()
            octeon_setup_boot_vector(coremask) //把start_app函数地址写入boot vector
            octeon_bootloader_shutdown() //这里是关闭u-boot
            start_cores(coremask)
                //把不在coremask里面的core的boot vector写为idle_core_start_app(), 这最后是个while(1)
                octeon_setup_boot_vector(0, &invert_coremask); 
                //给其他核发NMI, 其他核要么执行start_app(), 要么执行idle_core_start_app()
                cvmx_write_csr(CVMX_CIU_NMI, -2ull & cvmx_coremask_get64(&avail_coremask));
                //如果core 0不是第一个core, 也就是说mask从其他core开始, 那么core 0就在这里等
                while (!start_core0)
                ((void (*)(void)) (((boot_init_vector_t *) (BOOT_VECTOR_BASE))[0].app_start_func_addr)) ();
                其实就是上面的start_app(), 注意:这个函数每个在mask里的核都会跑
                    set_except_base_addr(boot_info_ptr->exception_base);
                    /*这个函数有个性, 所有核都跑这里, 都判断同一个变量, 
                    **因为所有SE的内存布局几乎都是1:1的, 
                    **所以这个变量地址落在同一个物理地址上, 只有core0会改这个地址
                    */
                    octeon_sync_cores() 
                    //最后在汇编里跳到entry_point
                    octeon_setup_crt0_tlb((uint32_t) & stack_top, (uint32_t) & entry_point, uint32_t) & desc_vaddr);
```

# hw-ddr2
mipsisa64-octeon-elf-readelf -e hw-ddr2 找到入口地址Entry point address:               0x10000d68  
这个地址是__start, 是个是编译器里面的东西, 编译器里面还有libc的库
```sh
$ mipsisa64-octeon-elf-nm -Al hw-ddr2 | grep 10000d68 
hw-ddr2:10000d68 T __start      /usr/local/Cavium_Networks/octsw/toolchain/scripts/../src/newlib/libc/sys/octeon/crt0.S:31
```
反汇编hw-ddr2
```
mipsisa64-octeon-elf-objdump -d hw-ddr2 > objdump.log
10000d68 <__start>:
10000e88:	0c045bbc 	jal	10116ef0 <__octeon_app_init> //这是个库函数
__octeon_app_init
    octeon_os_set_console
    octeon_os_set_uart_num
    __octeon_clock_get_rate
    __octeon_bootmem_init
```

# 用CVMX_SHARED声明的变量是共享的, 其他核直接用
```c
static CVMX_SHARED struct cvmx_interrupt_cpu cpu_ip2 = {
	.ip = 2
};
static CVMX_SHARED struct cvmx_interrupt_cpu cpu_ip3 = {
	.ip = 3
};
static CVMX_SHARED struct cvmx_interrupt *cvmx_ipx_irq[8];
static CVMX_SHARED union cvmx_ciux_to_irq cvmx_ciux_to_irq;
```

# 但怎么调到这个函数来的? 但肯定在main之前
```c
__cvmx_app_init(uint64_t app_desc_addr)
    core = cvmx_get_core_num();
    if ( sys_info_ptr->init_core == core )
        cvmx_bootmem_init(bootinfo->phy_mem_desc_addr); 
        first_core_initialized = 1;
    //所有的核都会等这个变量
    while (!first_core_initialized);
    cvmx_interrupt_initialize();
        //78是ciu3, 68是ciu2
        cvmx_interrupt_ciu_initialize(sys_info_ptr);
            //写0到CVMX_CIU_INTX_EN0/1,先把中断全部禁止
            //下面是注册两个钩子函数
            if (cvmx_is_init_core())
                cvmx_interrupt_register = cvmx_interrupt_register_ciu;
                cvmx_interrupt_map = cvmx_interrupt_map_ciu;
        if (cvmx_is_init_core())
            //__cvmx_interrupt_ciu见A
            cpu_ip2.irq.handler = __cvmx_interrupt_ciu;
            cpu_ip3.irq.handler = __cvmx_interrupt_ciu;
            //这里面的cpu_ip2/3都是全局的share变量, 那个map函数会转换irq号
            cvmx_interrupt_register(cvmx_interrupt_map(CVMX_IRQ_MIPS2),&cpu_ip2.irq);
                //这里cvmx_ipx_irq是个全局变量数组, 也是share的
                int bit = hw_irq_number & 0x3f;
                info->data = bit;
                info->mask = __cvmx_interrupt_cpu_mask_irq;
                info->unmask = __cvmx_interrupt_cpu_unmask_irq;
                cvmx_ipx_irq[bit] = info;
            cvmx_interrupt_register(cvmx_interrupt_map(CVMX_IRQ_MIPS3),&cpu_ip3.irq);
            cvmx_interrupt_register(cvmx_interrupt_map(CVMX_IRQ_MIPS6), &__cvmx_interrupt_perf);
            //所有的异常处理函数(非中断)
            cvmx_interrupt_state.exception_handler = __cvmx_interrupt_default_exception_handler;
            low_level_loc = CASTPTR(void, CVMX_ADD_SEG32(CVMX_MIPS32_SPACE_KSEG0, sys_info_ptr->exception_base_addr));
            memcpy(low_level_loc + 0x80, (void *)cvmx_interrupt_stage1, 0x80);
            memcpy(low_level_loc + 0x100, (void *)cvmx_interrupt_cache_error_v3, 0x80);
            memcpy(low_level_loc + 0x180, (void *)cvmx_interrupt_stage1, 0x80);
            memcpy(low_level_loc + 0x200, (void *)cvmx_interrupt_stage1, 0x80);
            //这里面传了个flag为0, 但据我分析应该是加上CVMX_ERROR_FLAGS_ECC_SINGLE_BIT
            cvmx_error_initialize(0);
                cvmx_error_initialize_cn70xx()
                    //用cvmx_error_add()把各种error加进去
                __cvmx_error_custom_initialize()
                    //这里面把CVMX_CIU_CIB_LMCX_RAWX的sec_err ded_err 替换成了__cvmx_cn6xxx_lmc_ecc_error_display
                    //但前提是1.CIU_CIB_L2C_EN(0)的相应位要为1; 2. LMC(0)_INT_EN也要为1(目前为0), 这条不需要
                    //__cvmx_cn6xxx_lmc_ecc_error_display()里面会调cvmx_safe_printf()来打印
                    cvmx_error_change_handler(...__cvmx_cn6xxx_lmc_ecc_error_display)
                cvmx_error_enable_group(CVMX_ERROR_GROUP_INTERNAL, 0);
                //这里应该是把LMC相应的东东使能了, 但我觉得L2C也应该加上
                cvmx_error_enable_group(CVMX_ERROR_GROUP_LMC, 0);
                cvmx_error_poll()
            //等等
            REGISTER_AND_UNMASK_ERR_IRQ(CVMX_IRQ_L2C);
                //简写, 声明一个_irq的静态共享变量_irq, 并把handler弄成__cvmx_interrupt_ecc
                static CVMX_SHARED struct cvmx_interrupt _irq.handler = __cvmx_interrupt_ecc
                //这个宏也会调cvmx_interrupt_register_ciu(), 但应该是走的另外分支
                    info->data = hw_irq_number;
                    info->mask = __cvmx_interrupt_ciu_mask_irq;
                    info->unmask =__cvmx_interrupt_ciu_unmask_irq;
                    //这个东西是个全局的share变量, 估计在中断里用
                    cvmx_ciu_to_irq[reg][bit] = info;
            REGISTER_AND_UNMASK_ERR_IRQ(CVMX_IRQ_RST);
            REGISTER_AND_UNMASK_ERR_IRQ(CVMX_IRQ_MIO);
            cpu_ip2.irq.unmask(&cpu_ip2.irq);
            cpu_ip3.irq.unmask(&cpu_ip3.irq);
    好了,现在开始让SR.BEV为1, 可以用在ram里面的中断向量了, 但地址在哪里? 应该是ebase地址吧?
    cvmx_debug_init();
    cvmx_coredump_init();
```

# A真正的中断处理函数
```c
__cvmx_interrupt_ciu
    //这里面大约是调每个irq的handler
    //而这个irq号就是cvmx_ciux_to_irq.en0/1_to_irq[]表里面的号
    //也就是用REGISTER_AND_UNMASK_ERR_IRQ注册的东西
    irq->handler(irq, registers);
        //大多数都是下面这个, 这个是前面注册的
        __cvmx_interrupt_ecc()
            //下面这个函数应该是遍历在cvmx_error_initialize_cn70xx()add进去的东西, 并调相应的handler
            cvmx_error_poll()
```

# 然后是main
```c
main()
    cvmx_user_app_init();
        //先检查一些BIST
        //如果是CVMX_USE_1_TO_1_TLB_MAPPINGS = 1, 这是在makefile里面写的, 这里是1
        //设1:1TLB, 应该是所有内存, 注意这里是每个核都跑的
        cvmx_bootmem_init(sys_info_ptr->phy_mem_desc_addr);
        if (cvmx_is_init_core())
            cvmx_qlm_init();
    hw_cons_open(); //这里面注册了串口的中断函数
    if (cvmx_get_core_num() == initialCore)
        //随机数, patten会用
        cvmx_rng_enable ();
        //把cache弄小到一个line
        hw_ddr_useOneCacheWay ();
        //两个性能计数
        cvmx_l2c_config_perf (0, CVMX_L2C_EVENT_WRITE_REQUEST, 0);
        cvmx_l2c_config_perf (1, CVMX_L2C_EVENT_READ_REQUEST,  0);
        //默认禁止ECC
        //然后就是开始跑了
        run_tests (&ppMemoryMap[cvmx_get_core_num()].range[0], 3, -1);
```
