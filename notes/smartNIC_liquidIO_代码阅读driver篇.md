- [打开调试打印](#打开调试打印)
- [代码里随处可见的spin_lock和atomic变量](#代码里随处可见的spin_lock和atomic变量)
- [几种模式的组合](#几种模式的组合)
- [一些主要脉络](#一些主要脉络)
- [pci base driver模块](#pci-base-driver模块)
	- [A. pci驱动](#a-pci驱动)
		- [A1. octeon的pci probe函数](#a1-octeon的pci-probe函数)
			- [A11. 如何osi? 到底哪些是osi的? 哪些又不是?](#a11-如何osi-到底哪些是osi的-哪些又不是)
			- [A12.cn66xx的配置表](#a12cn66xx的配置表)
			- [A13. oct_poll_module_starter](#a13-oct_poll_module_starter)
			- [A14. check_db_timeout  "Doorbell Timeout", 周期1个tick](#a14-check_db_timeout--doorbell-timeout-周期1个tick)
				- [A141. octeon_pci_map_single和octeon_pci_unmap_single](#a141-octeon_pci_map_single和octeon_pci_unmap_single)
				- [A142. 内存申请及释放](#a142-内存申请及释放)
			- [A15. oct_poll_req_completion, 1个tick](#a15-oct_poll_req_completion-1个tick)
			- [A16. oct_poll_check_unordered_list](#a16-oct_poll_check_unordered_list)
			- [A17. check_droq_refill, 每个q一个](#a17-check_droq_refill-每个q一个)
				- [A171. octeon_droq_refill](#a171-octeon_droq_refill)
			- [A18. oct_droq_thread, 收包线程, 每个output q一个](#a18-oct_droq_thread-收包线程-每个output-q一个)
				- [A181. octeon_register_dispatch_fn() 注册收包对应的opcode函数](#a181-octeon_register_dispatch_fn-注册收包对应的opcode函数)
				- [A182. octeon_create_recv_info(), 从驱动层到分发层, 每个来的报文都会对应一个](#a182-octeon_create_recv_info-从驱动层到分发层-每个来的报文都会对应一个)
			- [A19. octeon_droq_bh](#a19-octeon_droq_bh)
	- [B. oct_poll_thread, 所有设备和回调共享这一个线程](#b-oct_poll_thread-所有设备和回调共享这一个线程)
	- [C. octeon_fops: 这里面主要是ioctl](#c-octeon_fops-这里面主要是ioctl)
		- [C1. octeon_ioctl](#c1-octeon_ioctl)
			- [C11.oct_poll_module_starter()](#c11oct_poll_module_starter)
			- [C12.octeon_ioctl_send_request](#c12octeon_ioctl_send_request)
				- [C121. octeon_copy_input_dma_buffers : 新申请内核态的input buffer, 并把用户态提供的data拷进来](#c121-octeon_copy_input_dma_buffers--新申请内核态的input-buffer-并把用户态提供的data拷进来)
				- [C122. octeon_create_output_dma_buffers : 将用户态提供的output buffer在内核态也申请一份](#c122-octeon_create_output_dma_buffers--将用户态提供的output-buffer在内核态也申请一份)
				- [C123. octeon_create_data_buf](#c123-octeon_create_data_buf)
					- [C1231. octeon_create_sg_list](#c1231-octeon_create_sg_list)
				- [C124. 主要的查询接口](#c124-主要的查询接口)
	- [中断处理 :就是调用具体器件的中断处理函数](#中断处理-就是调用具体器件的中断处理函数)
		- [INT1. octeon_droq_bh, 见A19](#int1-octeon_droq_bh-见a19)
		- [INT2. octeon_request_completion_bh, 和A15几乎一样](#int2-octeon_request_completion_bh-和a15几乎一样)

# 打开调试打印
定义`CAVIUM_DEBUG PRINT_FLOW`  
或`CAVIUM_DEBUG PRINT_ALL`  
还可以用这个函数打印droq信息:
`oct_dump_droq_state(octeon_droq_t   *oq)`

# 代码里随处可见的spin_lock和atomic变量
几乎每个结构体都有`spin_lock`保护, 比如pending list就有  
`cavium_spin_lock_init(&oct->plist->lock);`
接着还有atomic变量  
`cavium_atomic_set(&oct->plist->instr_count, 0);`

# 几种模式的组合
```
resp_order == OCTEON_RESP_ORDERED
	: resp_list = OCTEON_ORDERED_LIST
	: 由process_ordered_list()处理, 见A15

resp_order == OCTEON_RESP_UNORDERED && resp_mode == OCTEON_RESP_BLOCKING
	: resp_list = OCTEON_UNORDERED_BLOCKING_LIST
	: 由check_unordered_blocking_list()处理, 见A15

resp_order == OCTEON_RESP_UNORDERED && resp_mode == OCTEON_RESP_NON_BLOCKING
	: resp_list = OCTEON_UNORDERED_NONBLOCKING_LIST
	: 由oct_poll_check_unordered_list()处理, 但被注释掉了?, 见A16
```

# 一些主要脉络
```c
octeon_ioctl_send_request
	__do_request_processing()
		__do_instruction_processing();

oct_test_thread()
	send_test_packet()
		octeon_process_request()
			__do_request_processing()
				__do_instruction_processing();

perf_test_loop()
	octeon_process_request()
		__do_request_processing()
			__do_instruction_processing();

cn56xx_send_peer_to_peer_map()----注意: nic模块也会调用下面这个函数
	octeon_process_instruction()
		__do_instruction_processing();

octeon_destroy_resources()
	octeon_send_short_command()
		__do_instruction_processing();

octeon_hot_reset()
	octeon_send_short_command()
		__do_instruction_processing();

octeon_shutdown_output_queue()
	octeon_send_short_command()
		__do_instruction_processing();

octeon_restart_output_queue()
	octeon_send_short_command()
		__do_instruction_processing();
```

# pci base driver模块
`components/driver/host/driver/linux/octeon_linux.c`
```c
module_init(octeon_base_init_module);
	//打印一些系统特征信息, 大小端, HZ, 编译宏开关等, 这是很好的架构设计习惯
	octeon_state = OCT_DRV_DEVICE_INIT_START;
	//清零octeon device数组, 共4个
	octeon_init_device_list();
	//清零octmodhandlers数组, 3个
	octeon_init_module_handler_list(); //见PCI-NIC 代码阅读之driver篇(结构体)之octeon_module_handler_t
	//注册pci驱动, 见A
	ret = pci_register_driver(&octeon_pci_driver); //简单设置了driver的bus等属性后, 还是调用driver_register(&drv->driver)
	octeon_state = OCT_DRV_DEVICE_INIT_DONE;
	//初始化poll进程
	octeon_init_poll_thread()
		//表示kthread的结构体见PCI-NIC 代码阅读之driver篇(结构体)G2
		//进程主体是oct_poll_thread, 见B
		cavium_kthread_setup(&oct_poll_id, oct_poll_thread, NULL, "Oct Poll Thread", 1);
		cavium_kthread_create()
			t->id = kthread_create(t->fn, t->fn_arg, t->fn_string);
			if(t->exec_on_create)
				wake_up_process(t->id);
	octeon_state = OCT_DRV_POLL_INIT_DONE;
	//注册字符设备/dev/octeon_device, 见C
	ret = register_chrdev(OCTEON_DEVICE_MAJOR, DRIVER_NAME,&octeon_fops);
	octeon_state = OCT_DRV_REGISTER_DONE;
```

## A. pci驱动
```c
static struct pci_driver octeon_pci_driver = {
	.name     = "Octeon",
	.id_table = octeon_pci_tbl, //包括了6XXX全系列的PCI ID
	.probe    = octeon_probe, //见A1
	.remove   = __devexit_p(octeon_remove),
};
```

### A1. octeon的pci probe函数
`components/driver/host/driver/linux/octeon_main.c`
```c
octeon_probe(struct pci_dev *pdev, const struct pci_device_id *ent)
	//首先要hold一个octeon_device_t, 见主结构体
	octeon_device_t  *oct_dev=NULL;
	/*这里先打印(uint32_t)pdev->vendor, (uint32_t)pdev->device
	**这是linux驱动core层匹配好pci设备后传入的
	**如果要改为用户态,参考remote-pci的方式找octeon设备
	*/
	/*申请全局变量octeon_device[4]的内存
	**这是个osi的接口, 但最后怎么调到linux的?见A11
	oct_dev = octeon_allocate_device(pdev->device);
		oct = octeon_allocate_device_mem(pci_id);
			/*实际分配的内存包括 
			**主结构体:sizeof(octeon_device_t)+见结构体:sizeof(octeon_cn6xxx_t)+见结构体J:sizeof(octeon_dispatch_t)*64
			**可能是为了省事, 把内存一次全部申请了, 后面也会分开用
			*/
			//这里怎么做到osi的?见A11
			buf  = cavium_alloc_virt(size); //虚拟地址
			oct        = (octeon_device_t *)buf;
			oct->chip  = (void *)(buf + octdevsize);
			oct->dispatch.dlist = (octeon_dispatch_t *)(buf + octdevsize + configsize); //见A18里面的使用
		octeon_device[oct_idx] = oct;
		//不同的octeon的name不一样, "Octeon%d"
	//把oct_dev当作驱动的priv成员
	pci_set_drvdata(pdev, oct_dev);
	//反过来指
	oct_dev->pci_dev = (void *)pdev;
	//具体硬件要在这里配?
	octeon_device_init(oct_dev)
		/*这里面有个函数指针, 原型是
		**typedef   oct_poll_fn_status_t (* octeon_poll_fn_t)(void *, unsigned long);
		*/
		octeon_poll_ops_t   poll_ops;
		//这里面一共做了三件事, 在pci-remote里面有对应的
		octeon_pci_os_setup(octeon_dev)
			pci_enable_device(octeon_dev->pci_dev)
			//使能64bit访问, dev可以访问64bit host空间?
			pci_set_dma_mask(octeon_dev->pci_dev, PCI_DMA_64BIT)
			//使能device的master位
			pci_set_master(octeon_dev->pci_dev)
		//使用驱动函数pci_read_config_dword来获得ID,并调用相应函数
		octeon_chip_specific_setup(octeon_dev)
			case OCTEON_CN66XX_PCIID:
				/*设置主结构体:C
				**用来适配不同型号的octeon的钩子函数
				**这些函数是和硬件最相关的操作, 寄存器和中断
				**注意这是个osi的函数
				**函数实现在components/driver/host/driver/osi/cn6xxx_common.c
				*/
				setup_cn66xx_octeon_device(oct)
					//其实chip是个void*, 还记得前面多申请的size吗? sizeof(octeon_cn6xxx_t)
					octeon_cn6xxx_t *cn6xxx = (octeon_cn6xxx_t *)oct->chip;
					//map bar0地址到oct->mmio[0].start
					octeon_map_pci_barx(oct, 0, 0)
						//reserve资源
						pci_request_region(oct->pci_dev, baridx*2, DRIVER_NAME)
					   	oct->mmio[baridx].start = pci_resource_start(oct->pci_dev, baridx*2);
   						oct->mmio[baridx].len   = pci_resource_len(oct->pci_dev, baridx*2);
						//地址需要ioremap?, 把调试打印打开!
						oct->mmio[baridx].hw_addr    = ioremap(oct->mmio[baridx].start, mapped_len);
					//map bar1
					octeon_map_pci_barx(oct, 1, MAX_BAR1_IOREMAP_SIZE))
					cn6xxx->conf  = (cn6xxx_config_t  *)oct_get_config_info(oct);
						/*聪明!根据chip id, 直接赋值配置表
						**所有的配置表在components/driver/host/include/oct_config_data.h
						**这里按照octeon_cn6xxx_t来配的, 见A12
						*/
						return (void *)&default_cn66xx_conf;
					/*设置oct->fn_list
					**包括setup_iq_regs setup_oq_regs setup_device_regs update_iq_read_idx
					**bar1_idx_setup bar1_idx_write     
					**interrupt_handler enable_interrupt enable_io_queues
					**等等, 详见主结构体:C
					**中断处理函数cn6xxx_interrupt_handler见下:中断处理
					*/
					//下面这个函数设置bar0的window寄存器相关的地址
					cn6xxx_setup_reg_address()
		//map完成了, 配置也完成了, 但还没有写进去
		cavium_atomic_set(&octeon_dev->status, OCT_DEV_PCI_MAP_DONE);
		//先复位一下
		octeon_dev->fn_list.soft_reset(octeon_dev)
		//清空dispatch链表
		octeon_init_dispatch_list(octeon_dev)
		//初始化poll fn list到oct->poll_list, 里面共64个poll函数
		octeon_setup_poll_fn_list(octeon_dev);
		/*注册一个驱动自己用的poll_ops:"Module Starter" 
		**函数是oct_poll_module_starter, 1秒一次, 见A13
		**注册到上面的oct->poll_list
		*/
		octeon_register_poll_fn(octeon_dev->octeon_id, &poll_ops);
		//至此初始化完成? 但没有写任何的octeon的寄存器???
		cavium_atomic_set(&octeon_dev->status, OCT_DEV_DISPATCH_INIT_DONE);
		//如果使能了buffer pool(USE_BUFFER_POOL), 搜索关键字: HUGE_BUFFER_CHUNKS
		/*注意这也是个osi的函数
		**基本上是调用cavium_malloc_dma来预先分配内存, 按size大小分为6类,32k,16k,依次
		**最后是调用kmalloc ----需要适配
		**默认是关闭的, ----我准备打开这个功能
		**但从get_buffer_from_pool()这个函数来看,不能把要申请的size分片
		**比如超过比如32K, 则会调用cavium_malloc_dma分配内存
		**也就是说, 这个驱动里面的packet都是假定物理内存连续的 ----这个问题要解决--要结合后面的pci_map_single()
		*/
		octeon_init_buffer_pool(octeon_dev, &bufpool_config)
		//先关闭input ring和output ring
		octeon_set_io_queues_off(octeon_dev);
		//初始化input ring
		octeon_setup_instr_queues(octeon_dev)
			//对32个ring, 或者说queue
				//注意这个结构体不需要物理地址连续, 见主结构体D
				oct->instr_queue[i] = cavium_alloc_virt(sizeof(octeon_instr_queue_t));
				//这里才是真正的input ring的内存申请的地方
				octeon_init_instr_queue(oct, i)
					iq = oct->instr_queue[iq_no];
					q_size = conf->instr_type * conf->num_descs; //这个input queue的深度
					//这个函数底层是调用linux内核的pci_alloc_consistent ----想好怎么替代了吗? 要保证硬件和软件看到的东西同步哦!
					iq->base_addr = octeon_pci_alloc_consistent(oct->pci_dev, q_size, &iq->base_addr_dma); //见A141, DMA内存
					//no response list, 申请虚拟连续的内存, 是结构体D里面的一个成员
					octeon_init_nr_free_list(iq, iq->max_count)
					//命令字的32bit或64bit是根据配置来的
					//真正开始干活, 调用钩子函数设置iput ring
					oct->fn_list.setup_iq_regs(oct, iq_no);
					//注册poll函数 check_db_timeout, "Doorbell Timeout", 周期1个tick??会不会太快了?见A14
					octeon_register_poll_fn(oct->octeon_id, &poll_ops);
		//到这里input ring完工
		cavium_atomic_set(&octeon_dev->status, OCT_DEV_INSTR_QUEUE_INIT_DONE);

		//pending list是用来处理需要response的request的, 见主结构体E, 很霸道的那个结构体
		octeon_init_pending_list(octeon_dev)
			//count是从那个配置表里来的
			count = (CHIP_FIELD(oct, cn6xxx, conf))->c.pending_list_size;
			oct->pend_list_size = count;
			//给主结构体E分配空间, 虚拟地址连续就行
			oct->plist = cavium_alloc_virt(sizeof(octeon_pending_list_t));
			//这个list是E1,是个链表, 有count个node
			oct->plist->list = (octeon_pending_entry_t *) cavium_alloc_virt(OCT_PENDING_ENTRY_SIZE * count);
			//free_list是个数组, 有count个uint32
			oct->plist->free_list = (uint32_t *) cavium_alloc_virt(sizeof(uint32_t) * count);
			oct->plist->entries = count;
		cavium_atomic_set(&octeon_dev->status, OCT_DEV_PEND_LIST_INIT_DONE);
        //现在是应答链表
		octeon_setup_response_list(octeon_dev)
			//对三种类型, 初始化链表头, 实际上, 以下三类的response, 由链表头引领
			//1 ordered, 1 unordered-blocking, 1 unordered-nonblocking
			//对全局变量noresp_buf_free_fn赋NULL, 这是个函数指针的二维数组
			noresp_buf_free_fn[oct->octeon_id][i] = NULL;
			//注册poll函数, oct_poll_req_completion, 1个tick, 见A15
			ret = octeon_register_poll_fn(oct->octeon_id, &poll_ops);
			//注意: 这个函数被注释掉了! 注册poll函数, oct_poll_check_unordered_list, 一秒10次, 见A16
			/*ret = octeon_register_poll_fn(oct->octeon_id, &poll_ops);*/

		cavium_atomic_set(&octeon_dev->status, OCT_DEV_RESP_LIST_INIT_DONE);

		//开始初始化output queue, 也叫dpoq?
		octeon_setup_output_queues(octeon_dev)
			//给32个droq分配空间
			for 32个
                //droq见主结构体G
				oct->droq[i] = cavium_alloc_virt(sizeof(octeon_droq_t));
				octeon_init_droq(oct, i)
					//先从config表中查配置信息
					//然后给真正的硬件output ring分配空间
					//必须注意以下的申请内存的函数
					droq->desc_ring = octeon_pci_alloc_consistent() //DMA内存
                    //底下调用__get_free_pages(),8字节对齐
					droq->info_list = cavium_alloc_aligned_memory() //一个新的页?
					droq->recv_buf_list = cavium_alloc_virt() //见主结构体G5

					octeon_droq_setup_ring_buffers(oct, droq))
						//A1i1. 对每个描述符, 申请buffer, 真正和octeon硬件对应的buffer, 另外一处在refill, 见A17
                        for(i = 0; i < droq->max_count; i++)
						    //注意这里的get_new_recv_buffer是osi的, 但实现是linux的skbuf； 这里就已经把收包bufer申请好了
						    buf = get_new_recv_buffer(droq->buffer_size); //用skb申请buffer?为什么?
						    //recv_buf_list是个G5的数组
                            droq->recv_buf_list[i].buffer  = buf;
						    droq->recv_buf_list[i].data    = get_recv_buffer_data(buf);
						    //下面关键是, 简写, 实际上用octeon_pci_map_single转为IOMMU地址, 说明也是DMA内存
						    desc_ring[i].info_ptr   = octeon_pci_map_single(&droq->info_list[i],...)
						    desc_ring[i].buffer_ptr = octeon_pci_map_single(droq->recv_buf_list[i].data,...)
	                    octeon_droq_reset_indices(droq);
                        	droq->host_read_index     = 0;
                        	droq->octeon_write_index  = 0;
                        	droq->host_refill_index   = 0;
	                        droq->refill_count        = 0;
                        	cavium_atomic_set(&droq->pkts_pending, 0);
					//调用真正的设硬件的函数
					oct->fn_list.setup_oq_regs(oct, q_no);
					poll_ops.fn     = check_droq_refill;
					//注册check_droq_refill函数, 见A17, 每个q一个
					octeon_register_poll_fn(oct->octeon_id, &poll_ops);
					//如果打开了USE_DROQ_THREADS, 但好像没开, 收包内核线程
					cavium_init_wait_channel(&droq->wc); //底下是init_waitqueue_head()
					//droq内核线程, 运行oct_droq_thread, 见A18, 每个q一个
					cavium_kthread_setup(&droq->thread, oct_droq_thread, droq,"Oct DROQ Thread", 0)
					cavium_kthread_create(&droq->thread) //底下是kthread_create()
					//绑定到cpu, q和cpu个数求余droq->q_no % cavium_get_cpu_count()
					cavium_kthread_set_cpu_affinity() //底下是kthread_bind()
					cavium_kthread_run(&droq->thread); //底下是wake_up_process()
		//至此output q初始化结束
		cavium_atomic_set(&octeon_dev->status, OCT_DEV_DROQ_INIT_DONE);
		//根据注释, 调用其他的device相关的初始化寄存器函数
		ret = octeon_dev->fn_list.setup_device_regs(octeon_dev);
		//这部分注册/proc/octeon
		cavium_init_proc(octeon_dev);
			root = proc_mkdir(octeon_dev->device_name, NULL);
			octeon_dev->proc_root_dir = (void *)root;
			node = create_proc_entry("debug_level", 0644, root);
			node->read_proc = proc_read_debug_level;
			node->write_proc = proc_write_debug_level;
			//这里面的create_proc_read_entry()在新的内核里应该是proc_create_data() and seq_file
			node = create_proc_read_entry("stats",0444, root, proc_read_stats,octeon_dev)
			//类似的, 创建"configreg" "csrreg" "npireg", 挂的是不同的读写函数

		//如果没有打开USE_DROQ_THREADS, 则用tasklet来处理output q, 默认是走这
		//octeon_droq_bh见A19, 由中断触发, 见中断处理
		cavium_tasklet_init(&octeon_dev->droq_tasklet, octeon_droq_bh, (unsigned long)octeon_dev);
		//comp_tasklet用来加速ORDERED的请求的处理, 这个下半部函数主要是调用process_ordered_list(), 可以参考A15
		cavium_tasklet_init(&octeon_dev->comp_tasklet, octeon_request_completion_bh, (unsigned long)octeon_dev);
		//注册PCIE中断, 见中断处理
		octeon_setup_interrupt(octeon_dev);
			//打开msi中断
			pci_enable_msi(oct->pci_dev)
			irqret = request_irq(oct->pci_dev->irq, octeon_intr_handler, CVM_SHARED_INTR, "octeon", oct);
		//现在开始使能中断
	   	octeon_dev->fn_list.enable_interrupt(octeon_dev->chip);
		//使能input output q
		octeon_dev->fn_list.enable_io_queues(octeon_dev);
		//send credit是什么意思?
		for(j = 0; j < octeon_dev->num_oqs; j++)  
			OCTEON_WRITE32(octeon_dev->droq[j]->pkts_credit_reg, octeon_dev->droq[j]->max_count);

		/* Packets can start arriving on the output queues from this point. */
		octeon_dev->app_mode = CVM_DRV_INVALID_APP;
		cavium_atomic_set(&octeon_dev->status, OCT_DEV_HOST_OK);
    //注册dispath函数, 见A181
    octeon_setup_driver_dispatches(oct_dev->octeon_id)
    //要求oct_dev->app_mode不是CVM_DRV_BASE_APP或CVM_DRV_INVALID_APP
    octeon_start_module(oct_dev->app_mode, oct_dev->octeon_id)
        //里面调的函数是octeon_register_module_handler()注册的
        __octeon_module_action(app_type, OCTEON_START_MODULE, octeon_id)
    octeon_state = OCT_DRV_ACTIVE;
```

#### A11. 如何osi? 到底哪些是osi的? 哪些又不是?
`components/driver/host/driver/osi/octeon_device.c`
`buf  = cavium_alloc_virt(size)`;实际调的是`vmalloc`, 这是linux kernel的函数, 这里已经不是osi了

os相关的定义在  
`components/driver/host/driver/linux/cvm_linux_types.h`
```c
#define   cavium_alloc_virt(size)        vmalloc((size))
```

这个头文件被
`components/driver/host/driver/linux/linux_sysdep.h` 引用  
进而被
`components/driver/host/include/cavium_sysdep.h` 引用

```c
#ifdef linux
#include "../driver/linux/linux_sysdep.h" //现在是这个分支
#elif defined(__FreeBSD__)
#include "../driver/freebsd/freebsd_sysdep.h"
#elif defined (_WIN32)
#include "..\driver\windows\windows_sysdep.h"
#else /* CUSTOM_OS */
#include "custom_sysdep.h"
#endif
```

#### A12.cn66xx的配置表
```c
cn66xx_config_t  default_cn66xx_conf =
{
	/* num_iqs; num_oqs; pending_list_size; */
	{
	4,
	4,
	4096,
	},

	/* Input Queue configuration */
	/* num_descs; instr_type; db_min; db_timeout; */
	{
	{1024, 32, 1, 1} ,
	{1024, 32, 1, 1}  ,
	{1024, 32, 1, 1}  ,
	{1024, 32, 1, 1}
	},

	/* Output Queue configuration */
	/* num_descs; info_ptr; bufsize, pkts_per_intr; refill_threshold*/
	{
	{1024, 1, 1536, 128, 128} ,
	{1024,  1, 1536, 128, 128} ,
	{1024,  1, 1536, 128, 128} ,
	{1024,  1, 1536, 128, 128} ,
	},

	/* oq_intr_pkt; oq_intr_time; */
	64,
	100,

#ifdef CVMCS_DMA_IC
	/* dma_intr_pkt; dma_intr_time; */
	64,
	1000,
#endif

};
```

#### A13. oct_poll_module_starter
这个poll的作用是等待core程序运行成功, 并报告它的app type
```c
oct_poll_module_starter(void  *octptr, unsigned long arg)
        if(cavium_atomic_read(&oct->status) == OCT_DEV_RUNNING) {
		return OCT_POLL_FN_FINISHED;
	}

	/* If the status of the device is CORE_OK, the core
	   application has reported its application type. Call
	   any registered handlers now and move to the RUNNING
	   state. */
	if(cavium_atomic_read(&oct->status) != OCT_DEV_CORE_OK)
		return OCT_POLL_FN_CONTINUE;

	cavium_atomic_set(&oct->status,OCT_DEV_RUNNING);
```

#### A14. check_db_timeout  "Doorbell Timeout", 周期1个tick
```c
/* Called by the Poll thread at regular intervals to check the instruction
 * queue for commands to be posted and for commands that were fetched by Octeon.
 */
check_db_timeout(void  *octptr, unsigned long  iq_no)
	//先获取对应的input queue
	iq = oct->instr_queue[iq_no];
	//计算是否超时, 这里面的cavium_jiffies是内核变量jiffies
    //没超时就啥也不干
	If cavium_jiffies - last_db_time < db_timeout
        return OCT_POLL_FN_CONTINUE;
	iq->last_db_time = cavium_jiffies; //记录上次的cavium_jiffies
	//这里用spin_lock_softirqsave是为了阻止中断(tasklet)--哪个中断也要锁iq->lock呢?
	cavium_spin_lock_softirqsave(&iq->lock);
	//fill_cnt表示等待发到octeon的instructions个数, 超时了还有值, 则按门铃
	if(iq->fill_cnt != 0)
		ring_doorbell(iq);
			//OCTEON_WRITE32也是个osi的接口, 底下是writel
			OCTEON_WRITE32(iq->doorbell_reg, iq->fill_cnt);
			iq->fill_cnt     = 0;
			iq->last_db_time = cavium_jiffies;
	cavium_spin_unlock_softirqrestore(&iq->lock);
	if(cavium_atomic_read(&iq->instr_pending))
		/** 这个注释真好!
		* Check for commands that were fetched by Octeon. If they were NORESPONSE
		* requests, move the requests from the per-queue pending list to the
		* per-device noresponse completion list.
		*/
		flush_instr_queue(oct, iq);
			update_iq_indices(oct, iq);
				//根据octeon已经读取的commands数更新, octeon_read_index表示octeon已经执行完的index
				iq->octeon_read_index = oct->fn_list.update_iq_read_idx(iq);
				/*把flush_index到octeon_read_index之间的
				**NORESPONSE的请求移动到completion list(每个device一个)
				*/
				if(iq->flush_index != iq->octeon_read_index)
					inst_processed = __process_iq_noresponse_list(oct, iq);
						//在循环里把flush_index到octeon_read_index之间的instruction移到freelist
						//从iq->nrlist移到iq->nr_free.q
						iq->nr_free.q[put_idx].buf     = iq->nrlist[old].buf;
						iq->nr_free.q[put_idx].buftype = iq->nrlist[old].buftype;
						iq->nrlist[old].buf = 0; iq->nrlist[old].buftype = 0;
					//这里表示noresponse的已经处理了inst_processed个
					if(inst_processed)
						cavium_atomic_sub(inst_processed, &iq->instr_pending);
						iq->stats.instr_processed += inst_processed ;
			//对iq->nr_free进行释放, 还是irqsave锁iq->lock
			process_noresponse_list(oct, iq);
				//在get_idx和put_idx之间
				while(get_idx != iq->nr_free.put_idx)
					switch(iq->nr_free.q[get_idx].buftype) {
					case NORESP_BUFTYPE_INSTR:
						//instr是octeon_soft_instruction_t, 见结构体E11
						instr = (octeon_soft_instruction_t *)iq->nr_free.q[get_idx].buf;
						//instr_list是个局部变量, 表示一个链表
						cavium_list_add_tail(&instr->list, &instr_list);
					case NORESP_BUFTYPE_NET:
					case NORESP_BUFTYPE_NET_SG:
					case NORESP_BUFTYPE_OHSM_SEND:
						//这里释放buf
						noresp_buf_free_fn[oct->octeon_id][iq->nr_free.q[get_idx].buftype](iq->nr_free.q[get_idx].buf);
				//接下来处理刚生成的instr_list链表
				release_soft_instr(oct, instr, instr->req_info.status); //标记A14i1
					if(soft_instr->dptr)
						//DPI_INST_HDR[G]=0时
						if(!soft_instr->ih.gather)
							octeon_pci_unmap_single() //response_manager.c
						//DPI_INST_HDR[G]=1时, DPTR指向一个链表
						else
							octeon_pci_unmap_sg_list()
						//注: 以上两个又是osi的接口, 见A141
					if(soft_instr->rptr)
						if(!soft_instr->irh.scatter)
							octeon_pci_unmap_single()
						else
							octeon_pci_unmap_sg_list()
					//调用callback,是SET_REQ_INFO_CALLBACK()注册的, 见C12里面的调用
					soft_instr->req_info.callback()
					if soft_instr->alloc_flags
						delete_soft_instr_buffers(octeon_dev, soft_instr); //见A142
							cavium_free_buffer(octeon_dev,(uint8_t *)((uint64_t *)soft_instr->status_word - 1));
							cavium_free_buffer(octeon_dev, (uint8_t*)soft_instr->scatter_ptr);
							cavium_free_buffer(octeon_dev, (uint8_t*)soft_instr->gather_ptr);
							cavium_free_buffer(octeon_dev, (uint8_t*)soft_instr->dptr);
							cavium_free_buffer(octeon_dev, (uint8_t*)soft_instr);
```

##### A141. octeon_pci_map_single和octeon_pci_unmap_single
```c
#define USE_PCIMAP_CALLS //这里默认是打开的
octeon_pci_map_single(cavium_pci_device_t *pci_dev, void *virt_addr,uint32_t  size, int direction)
#if defined(USE_PCIMAP_CALLS)
	physaddr = pci_map_single(pci_dev, virt_addr, size, direction);
#else
	physaddr = virt_to_phys(virt_addr);
#endif

octeon_pci_unmap_single(cavium_pci_device_t  *pci_dev, unsigned long dma_addr,
                        uint32_t  size, int direction)
#if defined(USE_PCIMAP_CALLS)
	pci_unmap_single(pci_dev, (dma_addr_t)dma_addr, size, direction);
#endif
```
注:`octeon_pci_map_single`用来map已经申请的内存
```c
desc_ring[i].info_ptr
desc_ring[i].buffer_ptr
cmd->dptr
cmd->rptr
sg_list[i].ptr[k]都会用到
```
为什么说只是个map呢? 因为这个bufer已经申请好了. 比如在
```c
octeon_droq_setup_ring_buffers中
	buf = get_new_recv_buffer(droq->buffer_size);
		dev_alloc_skb(size + SKB_ADJUST) //实际底下使用skbuf申请的
	droq->recv_buf_list[i].data    = get_recv_buffer_data(buf);
	desc_ring[i].buffer_ptr = (uint64_t)octeon_pci_map_single(oct->pci_dev, droq->recv_buf_list[i].data, droq->buffer_size, CAVIUM_PCI_DMA_FROMDEVICE );
```
补充:`pci_map_single`是干啥的?见`linux/include/asm-generic/pci-dma-compat.h`
```c
static inline dma_addr_t
pci_map_single(struct pci_dev *hwdev, void *ptr, size_t size, int direction)
	dma_map_single(hwdev == NULL ? NULL : &hwdev->dev, ptr, size, (enum dma_data_direction)direction);
		struct dma_map_ops *ops = get_dma_ops(dev);
		addr = ops->map_page(dev, virt_to_page(ptr),
			     (unsigned long)ptr & ~PAGE_MASK, size,
			     dir, attrs);
		return addr;
```

这里需要强调的是:
实际上这里用到了IOMMU, 把ptr转成了IOVA(IO Virtual address)  
驱动在发DMA的command之前, 需要调用pci_map_*()去把地址转成IO虚拟地址  
IOVA是分domain的, 每个pcie的device都有自己的domain, 在一个p2p桥下面的所有devices共享虚拟地址空间

再补充:`octeon_pci_alloc_consistent`这个函数被用来申请`droq->desc_ring和iq->base_addr`的内存
这个函数是否保证了内存一致性???
```c
octeon_pci_alloc_consistent
	pci_alloc_consistent(pci_dev, size, (dma_addr_t *)dma_addr_ptr);
		dma_alloc_coherent(hwdev == NULL ? NULL : &hwdev->dev, size, dma_handle, GFP_ATOMIC);
			struct dma_map_ops *ops = platform_dma_get_ops(dev);
			caddr = ops->alloc(dev, size, daddr, gfp, attrs);
			return caddr;
```

##### A142. 内存申请及释放
在`delete_soft_instr_buffers`函数中, 调用了很多`cavium_free_buffer()`函数,  
这个函数和`cavium_alloc_buffer()`一起,管理像  
```
soft_instr->dptr 
soft_instr 
soft_instr->gather_ptr
soft_instr->scatter_ptr
```
这样的buffer  
这些是osi的接口
```c
/*
 *  Macros that switch between using the buffer pool and the system-dependent
 *  routines based on compilation flags used.
 */
static __inline void *
cavium_alloc_buffer(octeon_device_t *octeon_dev, uint32_t size)
{
#ifdef USE_BUFFER_POOL
    //从预先分配的几个pool中分配, 32k, 16k, 8k, 4k, 2k...
    //超过32k的size还是用cavium_malloc_dma来分配
    return get_buffer_from_pool(octeon_dev, size);
#else
    //其底层实现是kmalloc, 在cvm_linux_types.h
    return cavium_malloc_dma(size, __CAVIUM_MEM_ATOMIC);
#endif
}

static __inline void
cavium_free_buffer(octeon_device_t  *octeon_dev, void *buf)
{
#ifdef USE_BUFFER_POOL
     put_buffer_in_pool(octeon_dev, (uint8_t *)buf);
#else
     cavium_free_dma(buf);
#endif
}
```

#### A15. oct_poll_req_completion, 1个tick
和`octeon_request_completion_bh`类似
对ordered list, 由链表头  
`&octeon_dev->response_list[OCTEON_ORDERED_LIST]`带领
下面链表的元素是  
`octeon_pending_entry_t`, 见结构体E1 

```c
oct_poll_req_completion(void *octptr, unsigned long  arg)
	process_ordered_list(oct);
		//这个变量是一次处理的上限值
		uint32_t resp_to_process = 4096;
		/*注意, 如果定义了CVMCS_DMA_IC
		**如果使能了DMA中断(def CVMCS_DMA_IC), 就可以确切的使用dma_cnt_to_process
		**dma_cnt_to_process表示已经有多少个被DMA了
		*/
		resp_to_process = cavium_atomic_read(&octeon_dev->dma_cnt_to_process);
		//见结构体F, 三个response list中的一个头
		response_list =  &octeon_dev->response_list[OCTEON_ORDERED_LIST];
		while < resp_to_process
			//pending_entry是octeon_pending_entry_t, 见E1
			//从response_list取节点依次处理
            pending_entry =  get_list_head(octeon_dev, response_list);
			soft_instr = pending_entry->instr;
			//这里的soft_instr->status_word是个地址, 应该是octeon写回status用的
			uint64_t   status64 = *(soft_instr->status_word); 
			status = (octeon_req_status_t) (status64 & 0xffffffffULL);
			//已经发送ok了
			if(status != OCTEON_REQUEST_PENDING)
				//从这个response list里面删除节点
				release_from_response_list(octeon_dev, pending_entry);
                //注意这里会调callback, 也就是会把response从内核态考到用户态
				release_soft_instr(octeon_dev, soft_instr, status); //见标记A14i1
				//从pending list删除节点
                release_from_pending_list(octeon_dev,pending_entry);
					oct->plist->free_index--;
					oct->plist->free_list[oct->plist->free_index] = pe->request_id;
					pe->status = OCTEON_PENDING_ENTRY_FREE;
			else
				//相当于退出循环了, 这里也正是order模式的精髓! 因为这个模式下的pending list是根据请求的顺序加的, 而处理的结果从status得出, 如果前面的entry没完成, 后面也是不会完成的.所以就此退出
	//unordered blocking的ioctl会阻塞, 
	check_unordered_blocking_list(oct);
		//见结构体F, 三个response list中的一个头
		response_list = (octeon_response_list_t *)&(oct->response_list[OCTEON_UNORDERED_BLOCKING_LIST]);
		//对每个链表节点
        //注:这里是unordered, 就不像上面, 遇到个未处理的就退出了. 这里是一直遍历这个链表: 但有个人为的最多16个的限制.
		cavium_list_for_each_safe(curr, tmp, &response_list->head)
			pending_entry = (octeon_pending_entry_t *)curr;
			soft_instr = pending_entry->instr;
			if *(soft_instr->status_word) != COMPLETION_WORD_INIT
				uint64_t   status64 = *(soft_instr->status_word);
			else
				status = OCTEON_REQUEST_TIMEOUT;
			//不pending表示已经ok了?
			if(status != OCTEON_REQUEST_PENDING)
				//这里只调个callback?那么释放资源呢? --是SET_REQ_INFO_CALLBACK()注册的, 见C12里面的调用
				soft_instr->req_info.callback()
```

#### A16. oct_poll_check_unordered_list
这里的注释写的很好, 这个函数是给UNORDERED NONBLOCKING LIST收尸的;
这个类型的response是要app去查询完成状态的:
app调用api octeon_query_request_status(query.octeon_id,&query)来查询.
但如果app意外退出了, 有些pending entry就永远不会释放了.

#### A17. check_droq_refill, 每个q一个
```c
check_droq_refill(void  *octptr, unsigned long q_no)
	droq = oct->droq[q_no];
	if (droq->refill_count >= droq->refill_threshold) 
		//no dispatch时, buffer会保留, 否则还要申请新的recv buffer
		//并把droq->refill_count--;见A171
		desc_refilled = octeon_droq_refill(oct, droq);
		//这可能是个刷cache的函数, 重要!怎么在用户态刷cache呢?
		cavium_flush_write();
		OCTEON_WRITE32(droq->pkts_credit_reg, desc_refilled);
```

##### A171. octeon_droq_refill
这个函数很多地方都在调
```c
octeon_droq_refill(octeon_device_t  *octeon_dev, octeon_droq_t   *droq)
   desc_ring = droq->desc_ring;
   while(droq->refill_count && (desc_refilled < droq->max_count)) {
		/* If a valid buffer exists (happens if there is no dispatch), reuse 
		   the buffer, else allocate. */
		if(droq->recv_buf_list[droq->host_refill_index].buffer == 0) 
            //buffer为NULL说明已经dispatch了? 此时重新申请一个buffer --参考A182
			buf = get_new_recv_buffer(droq->buffer_size); //见A141
			/* If a buffer could not be allocated, no point in continuing */
			if(!buf) 
				break;
			droq->recv_buf_list[droq->host_refill_index].buffer = buf;
			data = get_recv_buffer_data(buf);
		else
			data = get_recv_buffer_data(droq->recv_buf_list[droq->host_refill_index].buffer);
#if defined(ETHERPCI)
		data += 8;
#endif
		droq->recv_buf_list[droq->host_refill_index].data = data;
		desc_ring[droq->host_refill_index].buffer_ptr = (uint64_t)octeon_pci_map_single(octeon_dev->pci_dev, data, droq->buffer_size, CAVIUM_PCI_DMA_FROMDEVICE);

		/* Reset any previous values in the length field. */
		droq->info_list[droq->host_refill_index].length = 0;

		INCR_INDEX_BY1(droq->host_refill_index, droq->max_count);
		desc_refilled++;
		droq->refill_count--;
```

#### A18. oct_droq_thread, 收包线程, 每个output q一个
但好像没打开; 和A19收包bh二选一
```c
oct_droq_thread(void* arg)
	//传入的是q号
	//打印跑在smp_processor_id()这个核上, 初始化时绑定过的
	//如果没有stop, 没有signal, 底下是用kthread_should_stop()检查signal的
	//全部的干活主体在这个循环里
	while(!droq->stop_thread && !cavium_kthread_signalled())
		while(cavium_atomic_read(&droq->pkts_pending))
			//干活的主体
			octeon_droq_process_packets(droq->oct_dev, droq);
			    //pkts_pending就是待处理的报文数
                pkt_count = cavium_atomic_read(&droq->pkts_pending);	
                //何为快转? 好像是调了octeon_register_droq_ops()注册的, base里面没人调; nic模块里面用到了
				if droq->fastpath_on
					pkts_processed = octeon_droq_fast_process_packets(oct, droq, pkt_count); //要不要关注一下?
				else
				//如果定义了ETHERPCI, 应该没有吧?
					octeon_droq_drop_packets(droq, pkt_count);
				//应该是下面的分支
					pkts_processed = octeon_droq_slow_process_packets(oct, droq, pkt_count);
						for(pkt = 0; pkt < pkt_count; pkt++) 
							info = &(droq->info_list[droq->host_read_index]); //从上次的地方开始, 见主结构体G4
							resp_hdr = (octeon_resp_hdr_t *)&info->resp_hdr; //见主结构体G41
							octeon_swap_8B_data((uint64_t *)info, 2);
                            //这里有dispatch
							buf_cnt = octeon_droq_dispatch_pkt(oct, droq, resp_hdr, info);
								//根据buffer_size得到buffer count, 因为每个recvbuf都是固定大小, 而buffer size可以很大
								cnt = octeon_droq_get_bufcount(droq->buffer_size, info->length);
								//根据opcode得到分发函数, 是octeon_register_dispatch_fn()注册的, 见A181
								disp_fn = octeon_get_dispatch(oct, (uint16_t)resp_hdr->opcode);
                                        //在octeon_dev->dispatch.dlist[idx]里面(注意这可能是个链表), 依次匹配opcode;见主结构体J
								//注意这里只添加droq->dispatch_list节点, 见A182, 真正处理这个链表在下面
                                //这里实际上已经从"底层驱动"收包层, 上交到分发层处理了
								rinfo = octeon_create_recv_info(oct, droq, cnt, droq->host_read_index);
								rdisp->rinfo    = rinfo;
								rdisp->disp_fn  = disp_fn;
								*((uint64_t *)&rinfo->recv_pkt->resp_hdr) = *((uint64_t *)resp_hdr);
								cavium_list_add_tail(&rdisp->list, &droq->dispatch_list);
							/*上面已经dispatch了, 不管是不是有人在处理这些package, 将来总是会被处理和回收
							**这里把refill_count加上去, 说明该申请新的buf了
							*/
							droq->refill_count += buf_cnt;
							//下面判断是否pkt太多, 要丢弃, 和droq->pkts_per_intr配置有关
							if((droq->ops.drop_on_max) && (pkts_to_process - pkt))
								octeon_droq_drop_packets(droq, (pkts_to_process - pkt));
								droq->stats.dropped_toomany += (pkts_to_process - pkt); 
				//减掉已处理的
				cavium_atomic_sub(pkts_processed, &droq->pkts_pending);
				if(droq->refill_count >= droq->refill_threshold)
					//和A17类似, 那么如果有USE_DROQ_THREADS, 那可能不需要A17了
					desc_refilled = octeon_droq_refill(oct, droq); 
				/*接下来遍历droq->dispatch_list链表, 其中每个元素是, 调用里面的disp_fn
				**调用完即删除
				**struct __dispatch {
				**	cavium_list_t          list;
				**	octeon_recv_info_t    *rinfo;
				**	octeon_dispatch_fn_t   disp_fn;
				**};
				*/
				disp_fn(); //从这就断了???????????????怎么再往上面走?????????, 跟scatter的DMA又怎么联系起来?--不用联系起来
		//到这里内层while刚结束, 睡眠等待新的pkts, 中断处理函数会唤醒
		cavium_sleep_atomic_cond(&droq->wc, &droq->pkts_pending);
			//以下是一个常见的wait在wait_queue的操作, 等待droq->pkts_pending不为0
			cavium_wait_entry      we;
			cavium_init_wait_entry(&we, current);
			cavium_add_to_waitq(waitq, &we);
			set_current_state(TASK_INTERRUPTIBLE);
			while(!cavium_atomic_read(pcond))
				schedule();
			set_current_state(TASK_RUNNING); 
			cavium_remove_from_waitq(waitq, &we);
```

##### A181. octeon_register_dispatch_fn() 注册收包对应的opcode函数
所有的opcode在这里`components/driver/common/octeon-drv-opcodes.h`
在这里调用
```c
octeon_setup_driver_dispatches(oct_dev->octeon_id);
        //注册的CORE_DRV_ACTIVE_OP处理函数, 主要作用是收到core OK的报文后, 置oct->status为OCT_DEV_CORE_OK
	octeon_register_dispatch_fn(oct_id, CORE_DRV_ACTIVE_OP, octeon_core_drv_init,
	                            get_octeon_device_ptr(oct_id));
```
另外在`components/driver/host/test/kernel/droq_test/octeon_droq_test.c` 内核测试代码里面也有注册
```c
	if(octeon_register_dispatch_fn(OCTEON_ID, DROQ_PKT_OP1, droq_test_dispatch, NULL)) {
	if(octeon_register_dispatch_fn(OCTEON_ID, DROQ_PKT_OP2, droq_test_dispatch, NULL)) {
```

##### A182. octeon_create_recv_info(), 从驱动层到分发层, 每个来的报文都会对应一个
```c
/** Receive Packet format used when dispatching output queue packets
    with non-raw opcodes.
    The received packet will be sent to the upper layers using this
    structure which is passed as a parameter to the dispatch function 
*/
typedef struct {
  /**  Number of buffers in this received packet */
   uint16_t               buffer_count; 
  /** Id of the device that is sending the packet up */
   uint8_t                octeon_id;
  /** The type of buffer contained in the buffer ptr's for this recv pkt. */
   uint8_t                buf_type;
  /** Length of data in the packet buffer */
   uint32_t               length;
  /** The response header */
   octeon_resp_hdr_t      resp_hdr;
  /** Pointer to the OS-specific packet buffer */
   void                  *buffer_ptr[MAX_RECV_BUFS]; 
  /** Size of the buffers pointed to by ptr's in buffer_ptr */
   uint32_t               buffer_size[MAX_RECV_BUFS];
} octeon_recv_pkt_t;

/** The first parameter of a dispatch function.
    For a raw mode opcode, the driver dispatches with the device 
    pointer in this structure. 
    For non-raw mode opcode, the driver dispatches the recv_pkt_t
    created to contain the buffers with data received from Octeon.
    ---------------------
    |     *recv_pkt ----|---
    |-------------------|   |
    | 0 or more bytes   |   |
    | reserved by driver|   |
    |-------------------|<-/
    | octeon_recv_pkt_t |
    |                   |
    |___________________|
*/
typedef struct {
	void                     *rsvd;
	octeon_recv_pkt_t        *recv_pkt;
} octeon_recv_info_t;

struct __dispatch {
	cavium_list_t          list;
	octeon_recv_info_t    *rinfo;
	octeon_dispatch_fn_t   disp_fn;
};
```
```c
octeon_create_recv_info(octeon_device_t  *octeon_dev,
                        octeon_droq_t    *droq,
                        uint32_t          buf_cnt,
                        uint32_t          idx)
   //见G4
   octeon_droq_info_t    *info;
   octeon_recv_pkt_t     *recv_pkt; //见上
   octeon_recv_info_t    *recv_info; //见上
   uint32_t               i, bytes_left;

   info = &droq->info_list[idx];
   recv_info = octeon_alloc_recv_info(sizeof(struct __dispatch));
	   //以上两个结构体的size加起来再加extra_bytes, 为什么这里要用DMA的内存?????
	   buf = cavium_malloc_dma(OCT_RECV_PKT_SIZE + OCT_RECV_INFO_SIZE + extra_bytes, __CAVIUM_MEM_ATOMIC);
	   recv_info = (octeon_recv_info_t *)buf;
	   recv_info->recv_pkt = (octeon_recv_pkt_t *)(buf + OCT_RECV_INFO_SIZE);
	   recv_info->rsvd = NULL;
	   if(extra_bytes)
		   recv_info->rsvd = buf + OCT_RECV_INFO_SIZE + OCT_RECV_PKT_SIZE;
   recv_pkt = recv_info->recv_pkt;

   recv_pkt->resp_hdr      =  info->resp_hdr;
   recv_pkt->length        =  info->length;
   recv_pkt->buffer_count  =  (uint16_t)buf_cnt;
   recv_pkt->octeon_id     =  (uint16_t)octeon_dev->octeon_id;
   recv_pkt->buf_type      = OCT_RECV_BUF_TYPE_2;

   i = 0;
   bytes_left = info->length;
   //一个大报文可以有多个buf_cnt
   while(buf_cnt)
      octeon_pci_unmap_single(octeon_dev->pci_dev, (unsigned long)droq->desc_ring[idx].buffer_ptr, droq->buffer_size, CAVIUM_PCI_DMA_FROMDEVICE);

      recv_pkt->buffer_size[i] =
      (bytes_left >= droq->buffer_size)?droq->buffer_size: bytes_left;
      //这里不是把buffer拷贝过去, 而只是填指针, 为什么要把buffer挪个位置呢? 
      //其实这里是个分层, droq->recv_buf_list[idx].buffer是下层的buffer --就是用skb申请的buffer, 对应硬件DMA的output buffer
      //recv_pkt->buffer_ptr[i]是上层的buffer, 但从底层到上层不是buffer拷贝, 而只是指针转移
      //那么此时, 底层的buffer就可以重新申请了.
      recv_pkt->buffer_ptr[i]  = droq->recv_buf_list[idx].buffer;
      //底层的buffer已经转移给上层了; 这里赋值0, 在output buffer refill环节重新申请buffer,参考A171
      droq->recv_buf_list[idx].buffer = 0;

      INCR_INDEX_BY1(idx, droq->max_count);
      bytes_left -= droq->buffer_size;
      i++; buf_cnt--;

   return recv_info;
```

#### A19. octeon_droq_bh
```c
octeon_droq_bh(unsigned long  pdev)
	for(q_no = 0; q_no < oct->num_oqs; q_no++)
		reschedule |= octeon_droq_process_packets(oct, oct->droq[q_no]); //参考A18
	if(reschedule)
		//这个函数用来触发tasklet, 一般会在中断处理函数调
		cavium_tasklet_schedule(&oct->droq_tasklet);
```

## B. oct_poll_thread, 所有设备和回调共享这一个线程
```c
oct_poll_thread(void* arg)
	while !cavium_kthread_signalled()
		//对每个octeon设备
		oct_process_poll_list(get_octeon_device(i));
			poll_list = (octeon_poll_fn_node_t *)oct->poll_list;
			//对poll_list里面最多64个fn依次调用
			//这里的函数由octeon_register_poll_fn()注册
			//根据前面分析, 已经注册的函数有:A13 A14 A15 A16 A17
			n = (octeon_poll_fn_node_t *)&poll_list[i];
			ret = n->fn((void *)oct, n->fn_arg);

		cavium_sleep_timeout(1); //1个jiffy
			set_current_state(TASK_INTERRUPTIBLE);  
			schedule_timeout(timeout);              
			set_current_state(TASK_RUNNING);
```

## C. octeon_fops: 这里面主要是ioctl
```c
static struct file_operations octeon_fops =
{
   open:      octeon_open,
   release:   octeon_release,
   read:      NULL,
   write:     NULL,
   ioctl:     octeon_ioctl, //见C1
#if LINUX_VERSION_CODE > KERNEL_VERSION(2,6,11)
   unlocked_ioctl: octeon_unlocked_ioctl,
   compat_ioctl:  octeon_compat_ioctl,
#endif
   mmap:      NULL
};
```

### C1. octeon_ioctl
```c
int octeon_ioctl (struct inode   *inode,
                  struct file    *file, 
                  unsigned int   cmd,
                  unsigned long  arg)
   switch(cmd)  {
      case IOCTL_OCTEON_HOT_RESET:
           retval = octeon_ioctl_hot_reset(cmd, (void *)arg);
		//arg是octeon_rw_reg_buf_t 
		//先copy_from_user
		cavium_copy_in(&rw_buf, arg, OCTEON_RW_REG_BUF_SIZE)
		//得到主结构体
		oct_dev = get_octeon_device(rw_buf.oct_id);
		octeon_hot_reset(oct_dev);
			/*我觉得一个典型的重启架构应该是重启函数设置一个全局变量
			**真正干活的任务或资源主体在主循环里主动判断是否已经被stop了
			**从而干净的清除资源
			**这样的好处是功能代码集中在一起使用和释放
			**而不用把释放资源的函数全部集中在发起stop的函数里执行
			*/
			//这里面也是, 根据当前的状态字来干活.
			//总体上是发stop命令给octeon,等待一些活干完, 复位一些变量
			//最后注册oct_poll_module_starter()函数到poll进程, 见C11

           break;

      case IOCTL_OCTEON_SEND_REQUEST:
           retval = octeon_ioctl_send_request(cmd, (void *)arg); //见C12
           break;

      case IOCTL_OCTEON_QUERY_REQUEST:
           retval = octeon_ioctl_query_request(cmd, (void *)arg); 
		       //查询, 结果到octeon_query_request_t   
           break;

      case IOCTL_OCTEON_STATS:
           retval = octeon_ioctl_stats(cmd, (void *)arg);
           break;

      case IOCTL_OCTEON_READ32:
      case IOCTL_OCTEON_READ16:
      case IOCTL_OCTEON_READ8:
      case IOCTL_OCTEON_READ_PCI_CONFIG:
      case IOCTL_OCTEON_WIN_READ:
           retval = octeon_ioctl_read(cmd, (void *)arg);
           break;


      case IOCTL_OCTEON_WRITE32:
      case IOCTL_OCTEON_WRITE16:
      case IOCTL_OCTEON_WRITE8:
      case IOCTL_OCTEON_WRITE_PCI_CONFIG:
      case IOCTL_OCTEON_WIN_WRITE:
           retval = octeon_ioctl_write(cmd, (void *)arg);
           break;

      case IOCTL_OCTEON_CORE_MEM_READ:
           retval = octeon_ioctl_read_core_mem(cmd, (void *)arg);
           break;

      case IOCTL_OCTEON_CORE_MEM_WRITE:
           retval = octeon_ioctl_write_core_mem(cmd, (void *)arg);
           break;

      case IOCTL_OCTEON_GET_DEV_COUNT:
           retval = octeon_ioctl_get_dev_count(cmd, (void *)arg);
           break;

      case IOCTL_OCTEON_GET_MAPPING_INFO: 
           retval = octeon_ioctl_get_mapping_info(cmd, (void *)arg);
           break;

      default:
           cavium_error("octeon_ioctl: Unknown ioctl command\n");
           retval = -ENOTTY;
           break;
   }  /* switch */
```

#### C11.oct_poll_module_starter()
#### C12.octeon_ioctl_send_request
```c
typedef struct {
        /** wait channel head for this request. */
        cavium_wait_channel   wait_head;
        /** completion condition */
        int                   condition;
        /** Status of request. Used in NORESPONSE completion. */
        octeon_req_status_t   status;
} octeon_user_req_complete_t;
```
```c
copy buffer 既管input又管ouput, 这些是在内核空间的拷贝
/**  This structure is used to copy the response for a request
     from kernel space to user space.
*/
typedef struct {
  /** The kernel Input buffer. */
  uint8_t                    *kern_inptr;
  /** Size of the kernel output buffer. */
   uint32_t                   kern_outsize;
  /** Address of the kernel output buffer. */
   uint8_t                   *kern_outptr;
  /** Number of user space buffers */
   uint32_t                   user_bufcnt;
  /** Address of user-space buffers. */
   uint8_t                   *user_ptr[MAX_BUFCNT];
  /** Size of each user-space buffer. */
   uint32_t                   user_size[MAX_BUFCNT];
  /** The octeon device from which response is awaited. */
  octeon_device_t            *octeon_dev;
  /** Wait queue and completion flag for user request. */
  octeon_user_req_complete_t *comp;
}octeon_copy_buffer_t;
```
```c
int octeon_ioctl_send_request(unsigned int   cmd,
                              void          *arg)
	//从内核新申请octeon_soft_request_t结构, 这个结构见app篇 重要结构体
	soft_req = (octeon_soft_request_t *)cavium_alloc_buffer(octeon_dev, OCT_SOFT_REQUEST_SIZE);
	//把用户态的这个结构拷进去
	cavium_copy_in(soft_req, (void *)arg, OCT_SOFT_REQUEST_SIZE)
	//同样的方法把用户的octeon_request_info_t拷进去
	req_info = (octeon_request_info_t *)cavium_alloc_buffer(octeon_dev, OCT_REQ_INFO_SIZE);
	cavium_copy_in((void *)req_info, (void *)user_req_info, OCT_USER_REQ_INFO_SIZE)

	resp_order = GET_SOFT_REQ_RESP_ORDER(soft_req);
	resp_mode  = GET_SOFT_REQ_RESP_MODE(soft_req);
	dma_mode   = GET_SOFT_REQ_DMA_MODE(soft_req);
	//获得主结构体
	octeon_dev = get_octeon_device(GET_REQ_INFO_OCTEON_ID(req_info));

	if (resp_mode == OCTEON_RESP_BLOCKING) 
		//因为block的请求要wait在一个等待队列里, 在linux下是wait_queue_head_t
        //comp是octeon_user_req_complete_t, 见上面结构体
		comp = octeon_alloc_user_req_complete(octeon_dev);

	//开始拷贝input数据的buf, 总的原则是用户空间的buf考到内核连续空间的buf
	//并替换相应的指针地址(从指向user的地址变为指向新申请的内核地址)
	if(soft_req->inbuf.cnt) {
		if((dma_mode == OCTEON_DMA_DIRECT) || (dma_mode == OCTEON_DMA_SCATTER))
			//gather模式下, 用inbuf.ptr[(idx)].addr这个数组
			//非gather模式下, 只用inbuf.ptr[0]
			retval = octeon_copy_input_dma_buffers(octeon_dev, soft_req, 0); //见C121
		else
			retval = octeon_copy_input_dma_buffers(octeon_dev, soft_req, 1);
		if(retval)
			goto free_req_info;
	} else {
		soft_req->inbuf.cnt = 0;
		soft_req->inbuf.size[0] = 0;
		SOFT_REQ_INBUF(soft_req, 0)  = NULL;
	}

	//申请octeon_copy_buffer_t的buffer, 用来把response拷回user空间
    //copy_buf是octeon_copy_buffer_t, 见上面结构体
	copy_buf = cavium_alloc_buffer(octeon_dev, sizeof(octeon_copy_buffer_t));
    //这里只用0是因为从用户态拷贝到内核态变成了一个整的buffer
	copy_buf->kern_inptr = SOFT_REQ_INBUF(soft_req, 0);
	copy_buf->octeon_dev = octeon_dev;
	copy_buf->comp       = comp;
	//noresponse模式下, 不用申请output buf
		soft_req->outbuf.cnt     = 0;
		soft_req->outbuf.size[0] = 0;
		SOFT_REQ_OUTBUF(soft_req, 0)  = NULL;
		copy_buf->user_bufcnt    = 0;
	//以下是response模式, 申请在内核态用的response buffer
		retval = octeon_create_output_dma_buffers(octeon_dev, soft_req,...); //见C122
	//完事还得拷回用户态, 问题是谁调了这个函数呢? --被注释掉的A16和release_soft_instr(), A14和A15都会调
	SET_REQ_INFO_CALLBACK(req_info, octeon_copy_user_buffer, copy_buf);
		            //下面是octeon_copy_user_buffer()做的事情
		            //1. 这个request是阻塞的
		            if(copy_buf->comp)
			            copy_buf->comp->condition = 1;
			            copy_buf->comp->status    = status;
			            //唤醒阻塞的进程
			            cavium_wakeup(&(copy_buf->comp->wait_head));
		            2. 非阻塞的, 在这里拷贝
		            else
			        /* For non-blocking there is no process sleeping. So do copy
			           to user here and free buffers here. */
			        if(copy_buf->kern_inptr) {
				        cavium_free_buffer(octeon_dev, copy_buf->kern_inptr);

			        if(copy_buf->user_bufcnt) {
				        //这里就是把copy_buf->kern_outptr + 8的buf考到用户态
				        //看了这个函数的实现就知道为什么内核态申请的buf都是一整块的了
				        octeon_user_req_copyout_response(copy_buf, status);

			        if(copy_buf->kern_outptr) {
				        cavium_free_buffer(octeon_dev, copy_buf->kern_outptr);

			        cavium_free_buffer(octeon_dev, copy_buf);

	//这个重要了
	status = __do_request_processing(octeon_dev, soft_req);
		//先申请重要结构体octeon_soft_instruction_t si, 见E11
		si = cavium_alloc_buffer()
		//开始从soft_req往si里捯饬
		cavium_memcpy(&si->exhdr_info, &sr->exhdr_info, OCT_EXHDR_INFO_SIZE);
		cavium_memcpy(&si->req_info, SOFT_REQ_INFO(sr), OCT_REQ_INFO_SIZE);
		si->ih   = sr->ih;
		si->irh  = sr->irh;
		//这个函数关键了! 带着问题, 前面已经把内核态的input和output的buffer都搞定了呀(C121, C122)? 而且更底层的output buffer也OK了(A1i1和A17), 这里搞啥? --dptr和rptr
		octeon_create_data_buf(oct, si, sr) //见C123	
		//调这个函数之前, input output bufer要准备好, dptr和rptr也要准备好； --所有buffer都要准备好
		retval = __do_instruction_processing(oct, si, sr);
            irh = (uint64_t*)&si->irh;
            ih  = (uint64_t*)&si->ih;
			si->irh.pcie_port = oct->pcie_port;
			resp_order = SOFT_INSTR_RESP_ORDER(si);
			resp_mode  = SOFT_INSTR_RESP_MODE(si);
			iq_no      = SOFT_INSTR_IQ_NO(si);
			//得到iq号
			iq = oct->instr_queue[iq_no];
			cmd = &si->command; //见主结构体E114
			//注意: si->dptr是虚拟地址, 这里用pci_map*函数把这个虚拟地址转为iommu地址, 转到cmd->dptr里
            if(si->dptr)	
				//gather模式下, 硬件看到的是octeon_sg_entry_t的数组, 见C1231
				cmd->dptr = (uint64_t)octeon_pci_map_single(si->dptr,...)
                //非gather模式下, 是一个bufer地址
                cmd->dptr = (uint64_t)octeon_pci_map_single(si->dptr,...)
			if(si->rptr)
				//和上面差不多
			//设超时时间
			si->timeout = cavium_jiffies + SOFT_INSTR_TIMEOUT(si);
			if(resp_order != OCTEON_RESP_NORESPONSE) 
				//加到pending list, 这是等待链表 在A15里面处理
				pending_entry =  add_to_pending_list(oct, si);
					//从oct->plist->list[]数组里找个空位置, 把这个command加到list
                    //此时free index不能超过总的size
					if (oct->plist->free_index == oct->pend_list_size))
						return ERROR;
					pl_index = oct->plist->free_list[oct->plist->free_index];
					//这个index对应的node应该是free的
					if(oct->plist->list[pl_index].status != OCTEON_PENDING_ENTRY_FREE)
						return ERROR;
					oct->plist->free_index++;
					oct->plist->list[pl_index].instr      = si;
					oct->plist->list[pl_index].request_id = pl_index;
					oct->plist->list[pl_index].status     = OCTEON_PENDING_ENTRY_USED;
					oct->plist->list[pl_index].iq_no      = SOFT_INSTR_IQ_NO(si);
					//instr_count表示pending的命令数
					cavium_atomic_inc(&oct->plist->instr_count);
					//更新一些status
					si->req_info.status     = OCTEON_REQUEST_PENDING;
					si->irh.rid             = pl_index;
					si->req_info.request_id = pl_index;

			cmd->ih   = *ih;
			cmd->irh  = *irh;
			//返回值是   index = iq->host_write_index;
			q_index = post_command64B(oct, iq,...) //实质就是ring_doorbell(iq);
				__post_command(oct, iq, force_db, (uint8_t *)cmd64);
					//把cmd拷到iq中去
					__copy_cmd_into_iq(iq, cmd);
						iqptr = iq->base_addr + (cmdsize * iq->host_write_index);
						cavium_memcpy(iqptr, cmd, cmdsize);
					//更新host_write_index
					index = iq->host_write_index;
					INCR_INDEX_BY1(iq->host_write_index, iq->max_count);
					iq->fill_cnt++;
					/* Flush the command into memory. */
					cavium_flush_write(); //wmb()
					//超过fill_threshold才按门铃
					if(iq->fill_cnt >= iq->fill_threshold || force_db)
						ring_doorbell(iq);
							OCTEON_WRITE32(iq->doorbell_reg, iq->fill_cnt);
							iq->fill_cnt     = 0;
							iq->last_db_time = cavium_jiffies;
					cavium_atomic_inc(&iq->instr_pending);

			if(q_index >= 0)
				if(resp_order == OCTEON_RESP_NORESPONSE) 
					//谁来处理? --A14
					__add_to_nrlist(iq, q_index, si, NORESP_BUFTYPE_INSTR);
				else 
					pending_entry->queue_index = q_index;
                //至此这个command已经发送到iq里面了, 然后就是octeon取command执行

				retval.s.request_id = si->req_info.request_id;
				retval.s.error 	  = 0;
				retval.s.status     = si->req_info.status;
				if(sr)
					SOFT_REQ_INFO(sr)->request_id = retval.s.request_id;
					SOFT_REQ_INFO(sr)->status     = retval.s.status;

				if(resp_order != OCTEON_RESP_NORESPONSE)
					uint32_t   resp_list;
					//三选一, OCTEON_ORDERED_LIST OCTEON_UNORDERED_NONBLOCKING_LIST OCTEON_UNORDERED_BLOCKING_LIST
					GET_RESPONSE_LIST(resp_order, resp_mode, resp_list);
					//加到相应的链表里, 这是应答链表. 谁来处理?--难道又是A15?????
                    //每个发送的command都会加到应答链表(response模式下)
                    //这个链表见主结构体F, 是个"纯的"链表, 其实它的节点是pending_entry
                    //pending_entry从等待链表来, pending_entry见主结构体E1
					push_response_list(&oct->response_list[resp_list], pending_entry);

			if(cavium_atomic_read(&iq->instr_pending) >= (iq->max_count/2))
				flush_instr_queue(oct, iq); //参考A14

	//接下来就是等着octeon干活了
	//如果是阻塞模式, 注:这个模式不支持ordered
	if (resp_mode == OCTEON_RESP_BLOCKING) {
		//就直接在循环里查询等着, 后面等完了以后就地释放资源
		do {
			if(!SOFT_REQ_IGNORE_SIGNAL(soft_req) && signal_pending(current))
				query.status = OCTEON_REQUEST_INTERRUPTED;

			retval = octeon_query_request_status(query.octeon_id,&query);//见C124
	
			/* comp->condition would be set in the callback octeon_copy_user_buffer()
			   called from the request completion tasklet */
			if(comp->condition == 0)
				cavium_sleep_timeout_cond(&comp->wait_head,&comp->condition,1);
			else
				query.status = comp->status;
		} while((query.status == OCTEON_REQUEST_PENDING) && (!retval));
		/*### Copy query status to req_info here ###*/
		SOFT_REQ_INFO(soft_req)->status = query.status;
		octeon_user_req_copyout_response(copy_buf, query.status);
	//把req info考到用户态
	cavium_copy_out( user_req_info, SOFT_REQ_INFO(soft_req), OCT_USER_REQ_INFO_SIZE)
```

##### C121. octeon_copy_input_dma_buffers : 新申请内核态的input buffer, 并把用户态提供的data拷进来
```c
octeon_copy_input_dma_buffers(octeon_device_t        *octeon_dev,
                              octeon_soft_request_t  *soft_req,
                              uint32_t                gather)
	//计算出态total size
	for (i = 0,total_size = 0; i < soft_req->inbuf.cnt; i++)  
		total_size += soft_req->inbuf.size[i];
	//gather模式最大支持64K, direct模式支持16K
	//新申请整个buf
	mem = cavium_alloc_buffer(octeon_dev, total_size);
    memtmp = mem;
	//把每个inbuf从用户态拷到刚申请的内核态buf里, 虽然拷成一个整体, 但还是会保留每个inbuf
	for (i = 0; i < soft_req->inbuf.cnt; i++)  {
		if(cavium_copy_in((void *)memtmp, (void *)SOFT_REQ_INBUF(soft_req, i), soft_req->inbuf.size[i])) {
			cavium_error("OCTEON: copy in failed for inbuf\n");
			cavium_free_buffer(octeon_dev, mem);
			soft_req->inbuf.cnt    = 0;
			SOFT_REQ_INBUF(soft_req, 0) = NULL;
			return -EFAULT;
		}
		if(gather)
			//gather模式下设置每个inbuf的指针
			SOFT_REQ_INBUF(soft_req, i)  = memtmp;
		memtmp += soft_req->inbuf.size[i];
	}
	
	if(!gather) {
		soft_req->inbuf.cnt = 1;
		soft_req->inbuf.size[0]      = total_size;
		SOFT_REQ_INBUF(soft_req, 0)  = mem;
	}
```

##### C122. octeon_create_output_dma_buffers : 将用户态提供的output buffer在内核态也申请一份
```c
octeon_create_output_dma_buffers(octeon_device_t        *octeon_dev,
                                 octeon_soft_request_t  *soft_req,
                                 octeon_copy_buffer_t   *copy_buf,
                                 uint32_t                scatter)
	//首先计算所有的outbuf的total size, 这个size最大16K, 或14*8K(scatter)
	cnt = soft_req->outbuf.cnt;
	for (i = 0, total_size=0; i < cnt; i++)  
		total_size += soft_req->outbuf.size[i];
	//total size还要加上sizeof(octeon_resp_hdr_t)再加8, 见主结构体G41
	total_size += OCT_RESP_HDR_SIZE + 8;
    //这个就是在内核空间的output buffer, 但要和refill的那个更底层的buffer区别开
	mem = cavium_alloc_buffer(octeon_dev, total_size);
	//为copy会用户态做的准备  
	copy_buf->kern_outsize = total_size - OCT_RESP_HDR_SIZE - 8;
	copy_buf->kern_outptr  = mem;
	copy_buf->user_bufcnt  = cnt;
	for (i = 0; i < cnt; i++)   {
        //保留用户态buffer的地址, 往回拷贝要用
		copy_buf->user_ptr[i]  = SOFT_REQ_OUTBUF(soft_req, i);
		copy_buf->user_size[i] = soft_req->outbuf.size[i];
	}  
	//填soft_req->outbuf
	while (size < total_size)
        //把outbuf这个数组里面的指针更新为内核态地址
		SOFT_REQ_OUTBUF(soft_req, i) = mem + size;
		if( (size + OCT_MAX_USER_BUF_SIZE) > total_size)
			buf_size = (total_size - size);	
		else
			buf_size = OCT_MAX_USER_BUF_SIZE;
		soft_req->outbuf.size[i] = buf_size; 
		i++;

	soft_req->outbuf.cnt = i;
```

##### C123. octeon_create_data_buf
根据DMA的方式, 填soft_instr->dptr(往octeon去)和soft_instr->rptr(从octeon来), 问题, 在哪里用的呢?????

以上两个ptr可以是直接一个指针地址, 也可以是octeon_sg_entry_t 的链表
```c
static  uint32_t 
octeon_create_data_buf(octeon_device_t            *oct,
                       octeon_soft_instruction_t  *soft_instr,
                       octeon_soft_request_t      *soft_req)
	dma_mode   = GET_SOFT_REQ_DMA_MODE(soft_req);
	resp_order = GET_SOFT_REQ_RESP_ORDER(soft_req);
	//确认buffer count不大于MAX_BUFCNT, 16
	//DMA类型
	//第一种: input和output都是direct
	OCTEON_DMA_DIRECT :soft_req->outbuf.cnt必须为0；最大支持16K的buffer;
	可以没有input, 但必须有header的buffer
		soft_instr->ih.gather   = 0;
		soft_instr->irh.scatter = 0;
		//申请input buf, 这里好像罗嗦了, 应该不用再申请一次buffer
		soft_instr->dptr =  octeon_process_request_inbuf(oct,
                                                soft_instr, soft_req);
			//是哪个iq
			iq_no = SOFT_INSTR_IQ_NO(soft_instr);
			exhdr_size = SOFT_INSTR_EXHDR_COUNT(soft_instr) * 8;

			//这里在直接DMA模式下, 如果app提供了多个inbuf, 就合并到新的大buf里
			if (soft_req->inbuf.cnt > 1)
				for i in all inbuf.cnt:
					total_size += soft_req->inbuf.size[i];
				//总的size还要加上extra header
				total_size += exhdr_size; /* Add any extra header bytes present */
				//重新申请buf, 底下要么是从buffer池来, 要么是kmalloc(可dma)
				buf = cavium_alloc_buffer(octeon_dev, total_size);
				//先拷extra header
				if(exhdr_size) {
				   cavium_memcpy(tmpbuf, soft_instr->exhdr, exhdr_size);
				   tmpbuf += exhdr_size;
				}
				//再拷每个inbuf的内容, 注意这里是dada的第二次拷贝, 第一次发生在从用户态拷到内核态, 见C121
				for(i = 0 ; i < soft_req->inbuf.cnt; i++) {
				  cavium_memcpy(tmpbuf, SOFT_REQ_INBUF(soft_req, i),soft_req->inbuf.size[i]);
				  tmpbuf += soft_req->inbuf.size[i];
				}

				soft_instr->dptr       = buf;
				soft_instr->ih.dlengsz = total_size;
				SET_SOFT_INSTR_ALLOCFLAGS(soft_instr, OCTEON_DPTR_COALESCED);
			//只有一个inbuf
			else
				//只有一个buf就不用拷贝了
				soft_instr->dptr        = SOFT_REQ_INBUF(soft_req, 0);
				soft_instr->ih.dlengsz  = soft_req->inbuf.size[0];

		//填rptr, 因为在直接DMA模式下, soft_req->outbuf.cnt必须为0
		if((soft_instr->ih.raw) && (resp_order != OCTEON_RESP_NORESPONSE))
			soft_instr->rptr = SOFT_REQ_OUTBUF(soft_req, 0);
			soft_instr->irh.rlenssz  = soft_req->outbuf.size[0];
			soft_instr->status_word = outbuf的最后8个字节
	//第二种:input是gather, output是direct
	OCTEON_DMA_GATHER :soft_req->outbuf.cnt必须为0
		soft_instr->ih.gather   = 1;
		//因为这个模式下output是直接的
		soft_instr->rptr         = SOFT_REQ_OUTBUF(soft_req, 0);
		soft_instr->irh.rlenssz  = soft_req->outbuf.size[0];
		soft_instr->status_word =
			 (uint64_t *)((uint8_t *)SOFT_REQ_OUTBUF(soft_req, 0)
				    + soft_req->outbuf.size[0] - 8);
		//上面搞定了rptr, 现在来搞dptr, 见C1231
		soft_instr->dptr = octeon_create_sg_list(oct, &soft_req->inbuf
				         , soft_instr, OCTEON_DPTR_GATHER);
	//第三种:input是direct, output是scatter
	OCTEON_DMA_SCATTER : 此DMA不支持noresponse, 很简单, 如果不用output,还要scatter干啥?
		soft_instr->irh.scatter = 1;
		//这个模式下, 要求inbuf.cnt必须为1???????和上面第一种DMA模式为什么有差别呢?
		//那么只有一个inbuf, 就直接用作dptr了
		soft_instr->dptr        = SOFT_REQ_INBUF(soft_req, 0);
		soft_instr->ih.dlengsz  = soft_req->inbuf.size[0];
		//上面算是搞定了dptr, 接写来搞rptr, 见C1231
		soft_instr->rptr = octeon_create_sg_list(oct, &soft_req->outbuf,
				              soft_instr, OCTEON_RPTR_SCATTER);
		if(resp_order != OCTEON_RESP_NORESPONSE) {
				soft_instr->status_word = (uint64_t *)
				  ((uint8_t *)SOFT_REQ_OUTBUF(soft_req, (soft_req->outbuf.cnt-1))
				   + soft_req->outbuf.size[soft_req->outbuf.cnt-1] - 8);

	//第四种:input是gather, output是scatter
	OCTEON_DMA_SCATTER_GATHER :是以上两个模式的综合, 同上, 不支持noresponse

	//input和output的buf都搞定以后
	if(soft_instr->ih.raw)
		if(IQ_INSTR_MODE_32B(oct, SOFT_INSTR_IQ_NO(soft_instr)))
			soft_instr->ih.fsz = 16;
		else 
			soft_instr->ih.fsz = 16 + (SOFT_INSTR_EXHDR_COUNT(soft_instr) * 8);

	if(soft_instr->status_word)
		//status置为COMPLETION_WORD_INIT
		*(soft_instr->status_word) = COMPLETION_WORD_INIT;
```

###### C1231. octeon_create_sg_list
```c
/** The Scatter-Gather List Entry. The scatter or gather component used with
    a Octeon input instruction has this format. */
typedef struct { 

	/** The first 64 bit gives the size of data in each dptr.*/
	union {
		uint16_t  size[4];
		uint64_t  size64;
	} u;
	
	/** The 4 dptr pointers for this entry. */
	uint64_t     ptr[4];

}octeon_sg_entry_t;
```

```c
//这个函数并不拷贝data, 只是申请sg_count个上面的结构体, 然后把里面的4个ptr指向outbuf
octeon_create_sg_list(octeon_device_t            *oct,
                      octeon_buffer_t            *buf, 
                      octeon_soft_instruction_t  *soft_instr,
                      uint32_t                    flags)
	//cnt就是buf的个数
	//gather链表
		//也是先算total size, 加上一些extra header, 最大64K
		soft_instr->gather_bytes = total_size;
		soft_instr->ih.dlengsz   = cnt;
	//scatter链表
		//同样先算total size, 加上一些extra header, 最大14*8K
		soft_instr->irh.rlenssz   = cnt;
		soft_instr->scatter_bytes = total_size;
	//把cnt按4个一组
	sg_count = ROUNDUP4(cnt) >> 2;
	sg_list = octeon_alloc_sglist(oct, soft_instr, sg_count, flags);
		//结构体见上面octeon_sg_entry_t, sg_list需要8字节对齐
		sg_list = cavium_alloc_buffer(octeon_dev,(sg_count*OCT_SG_ENTRY_SIZE)+ 7);
		if(flags & OCTEON_DPTR_GATHER)
			soft_instr->gather_ptr = (void *)sg_list;
		else
			soft_instr->scatter_ptr = (void *)sg_list;

	//向sg_list[i].ptr[k]填物理地址(或经过IOMMU的地址)
	for (i = 0, j = 0; i < sg_count; i++)  {
		for(k = tmpcnt; ((k < 4) && (j < cnt)); j++, k++)   {
		    size = buf->size[j];
		    /* Gather list data is swapped and interpreted in Hardware.
		       Scatter data is interpreted by core software. We need to do
		       swapping of scatter list manually. */
		    if(flags == OCTEON_RPTR_SCATTER) {
			    octeon_swap_2B_data(&size, 1);
			    sg_list[i].u.size[3-k] = size;
		    } else {
			    CAVIUM_ADD_SG_SIZE(&sg_list[i], size, k);
		    }

		    if(flags == OCTEON_DPTR_GATHER) {
			    sg_list[i].ptr[k] = octeon_pci_map_single(oct->pci_dev, OCT_SOFT_REQ_BUFPTR(buf, j), buf->size[j], CAVIUM_PCI_DMA_TODEVICE);
				} else {
			    sg_list[i].ptr[k] = octeon_pci_map_single(oct->pci_dev, OCT_SOFT_REQ_BUFPTR(buf, j), buf->size[j], CAVIUM_PCI_DMA_FROMDEVICE);
			octeon_swap_8B_data(&sg_list[i].ptr[k], 1);
		    }
		}
		tmpcnt = 0;
	}
    return sg_list; //被当作rptr或dptr
```

##### C124. 主要的查询接口
```c
octeon_query_request_status(uint32_t                 octeon_id,
                            octeon_query_request_t  *query)
	octeon_device_t  *octeon_dev = get_octeon_device(query->octeon_id);
	process_unordered_poll(octeon_dev, query))
        //有request_id就可以找到pending entry
		pending_entry  = &octeon_dev->plist->list[query->request_id];
		soft_instr = pending_entry->instr;
		//COMPLETION_WORD_INIT是最开始的状态, 不是init说明已经在干活了?
		if(*(soft_instr->status_word) != COMPLETION_WORD_INIT)
			//得到新的status
			status = (octeon_req_status_t) (status64 & 0x00000000ffffffffULL);
		else
			//是否超时了?
			if(cavium_check_timeout(cavium_jiffies, soft_instr->timeout)) 
				status = OCTEON_REQUEST_TIMEOUT;
		//不是pending说明用完了
		if(status != OCTEON_REQUEST_PENDING)
			if(SOFT_INSTR_RESP_MODE(soft_instr) == OCTEON_RESP_BLOCKING)
				response_list = &octeon_dev->response_list[OCTEON_UNORDERED_BLOCKING_LIST];
			else
				response_list = &octeon_dev->response_list[OCTEON_UNORDERED_NONBLOCKING_LIST];
			release_from_response_list(octeon_dev, pending_entry);
			release_from_pending_list(octeon_dev,pending_entry);
			release_soft_instr(octeon_dev, soft_instr, status);
		//本次查询的结果, 一般开始会是OCTEON_REQUEST_PENDING, 最后是完成
		query->status = status;
```

## 中断处理 :就是调用具体器件的中断处理函数
```c
octeon_intr_handler(int irq, void *dev)
		return  oct->fn_list.interrupt_handler(oct);
```
比如
```c
cn6xxx_interrupt_handler
cn6xxx_interrupt_handler(void  *dev)
	//通过bar0读INT_SUM寄存器, 没有中断则马上退出
	intr64 = OCTEON_READ64(cn6xxx->intr_sum_reg64);
    if !intr64 
        return

	//马上关本中断
	oct->fn_list.disable_interrupt(oct->chip);
	if(intr64 & CN63XX_INTR_ERR)
		cn6xxx_handle_pcie_error_intr(oct, intr64);
	if(intr64 & CN63XX_INTR_PKT_DATA) //这里是从octeon网口来的报文, 和response没有关系
		cn6xxx_droq_intr_handler(oct);
                     droq_time_mask = octeon_read_csr(oct, CN63XX_SLI_PKT_TIME_INT);
			//对每个q
            for (oq_no = 0; oq_no < oct->num_oqs; oq_no++)
                //先检查是不是在mask里
                if ( !(droq_mask & (1 << oq_no)) )
                    continue;
                droq = oct->droq[oq_no];
                //一共收了多少包?
			    pkt_count = octeon_droq_check_hw_for_pkts(oct, droq);
                    pkt_count = OCTEON_READ32(droq->pkts_sent_reg)
                    //为什么是加在pkts_pending上呢? 是指收到的还没处理的报文数?
		            cavium_atomic_add(pkt_count, &droq->pkts_pending);
                    //这里为什么还要写回这个寄存器呢?
			        OCTEON_WRITE32(droq->pkts_sent_reg, pkt_count);
                //还有个poll模式???? 好像没打开
			    if(droq->ops.poll_mode) {
				    droq->ops.napi_fn(oct->octeon_id, oq_no, POLL_EVENT_INTR_ARRIVED);
			    else
				    //打开了USE_DROQ_THREADS才会调, 也就是说, 即使是用内核线程收包, 也需要中断触发
				    cavium_wakeup(&droq->wc); //这个任务见A18
			//不用USE_DROQ_THREADS时使用tasklet, 见INT1
			cavium_tasklet_schedule(&oct->droq_tasklet);

	if(intr64 & (CN63XX_INTR_DMA0_FORCE|CN63XX_INTR_DMA1_FORCE))
		//这里开始让comp_tasklet干活, 见INT2
		cavium_tasklet_schedule(&oct->comp_tasklet);
    //好像没有搜到任何dma_ops下挂的函数
	if((intr64 & CN63XX_INTR_DMA_DATA) && (oct->dma_ops.intr_handler))
		oct->dma_ops.intr_handler((void *)oct, intr64);
	/* Clear the current interrupts */
	OCTEON_WRITE64(cn6xxx->intr_sum_reg64, intr64);
	/* Re-enable our interrupts  */
	oct->fn_list.enable_interrupt(oct->chip);
```

### INT1. octeon_droq_bh, 见A19
* 这里的收包和发送命令字里面的response应该没有关系, 只是收octeon主动发过来的包, 然后分发; 现在走的应该是slow path, 而且没有真正的"上层"处理函数注册, 见A181
* 关于这里使用的tasklet:
  1. tasklet工作在软中断上下文
  2. tasklet在多CPU之间是串行化执行的, 多CPU不会同时执行一个tasklet----而同是软中断的softirq可以同时执行
  3. tasklet始终运行在被初始提交的同一处理器上，workqueue不一定 ----待确认

综上, 即使中断来的很快, `tasklet_schedule(&oct->droq_tasklet)`不断的被调用, 那么droq_tasklet也还是会被串行执行.
而线程收包模式可以利用多CPU一起收包, 只要他们不是处理同一个Q. --最多32个q

### INT2. octeon_request_completion_bh, 和A15几乎一样
```c
void
octeon_request_completion_bh(unsigned long pdev)
{
	octeon_device_t    *octeon_dev = (octeon_device_t *)pdev;

	octeon_dev->stats.comp_tasklet_count++;

	/* process_ordered_list returns 1 if list is empty. */
#ifdef CVMCS_DMA_IC
	/* No need to schedule this again, if too many pending,
 	 * then we got DMA_COUNT interrupt immediately */
	process_ordered_list(octeon_dev);
#else
	if(!process_ordered_list(octeon_dev))
		cavium_tasklet_schedule(&octeon_dev->comp_tasklet);
#endif
	check_unordered_blocking_list(octeon_dev);
}
```
