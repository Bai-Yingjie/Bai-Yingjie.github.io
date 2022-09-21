- [insmod log](#insmod-log)
- [主结构体`octeon_device_t`](#主结构体octeon_device_t)
- [A.描述PCIE空间的结构体](#a描述pcie空间的结构体)
- [B.间接访问寄存器, SLI_WIN*](#b间接访问寄存器-sli_win)
- [C.用来适配不同的octeon型号](#c用来适配不同的octeon型号)
- [D.表示一个input ring](#d表示一个input-ring)
- [E.对应主结构体的plist域](#e对应主结构体的plist域)
  - [E1.octeon_pending_entry_t](#e1octeon_pending_entry_t)
    - [E11.这个就是和硬件最接近的结构?](#e11这个就是和硬件最接近的结构)
      - [E111.octeon_request_info_t](#e111octeon_request_info_t)
      - [E112.对应到硬件寄存器DPI_INST_HDR](#e112对应到硬件寄存器dpi_inst_hdr)
      - [E113.Input Request Header](#e113input-request-header)
      - [E114.这个就是input ring的一个instruction](#e114这个就是input-ring的一个instruction)
      - [E115.extra header](#e115extra-header)
- [F.头指针, 表示一个response链表](#f头指针-表示一个response链表)
- [G.表示output ring描述符](#g表示output-ring描述符)
  - [G1.octeon_droq_ops_t](#g1octeon_droq_ops_t)
  - [G2.cvm_kthread_t](#g2cvm_kthread_t)
  - [G3.真正的符合硬件的定义的结构体](#g3真正的符合硬件的定义的结构体)
  - [G4.octeon_droq_info_t](#g4octeon_droq_info_t)
    - [G41.octeon_resp_hdr_t](#g41octeon_resp_hdr_t)
  - [G5.驱动用这个来表示buffer, 用的是虚拟地址](#g5驱动用这个来表示buffer-用的是虚拟地址)
  - [G6. oct_droq_stats_t](#g6-oct_droq_stats_t)
- [H. octeon_dma_ops_t](#h-octeon_dma_ops_t)
- [I. octeon_range_table_t](#i-octeon_range_table_t)
- [J.收到报文后按opcode分发](#j收到报文后按opcode分发)
- [K. buffer pool](#k-buffer-pool)
- [L. buffer pool的fragment](#l-buffer-pool的fragment)
- [M.USE_DDOQ_THREADS](#muse_ddoq_threads)
- [octeon_module_handler_t](#octeon_module_handler_t)
- [octeon_cn6xxx_t](#octeon_cn6xxx_t)
  - [2A.cn6xxx_config_t](#2acn6xxx_config_t)

# insmod log
```
octeon_drv: module license 'Cavium Networks' taints kernel.
Disabling lock debugging due to kernel taint
-- OCTEON: Loading Octeon PCI driver (base module)
OCTEON: Driver Version: PCI BASE RELEASE 2.3.0 build 84
OCTEON: System is Little endian (1000 ticks/sec)
OCTEON: PCI Driver compile options:  DEBUG 


OCTEON: Found device 177d:92..Initializing...
OCTEON: Setting up Octeon device 0 
Octeon 0000:08:00.0: PCI INT A -> GSI 16 (level, low) -> IRQ 16
Octeon 0000:08:00.0: setting latency timer to 64
OCTEON[0]: CN66XX PASS1.2
OCTEON[0]: Using PCIE Port 0
OCTEON[0]: BIST enabled for soft reset
OCTEON[0]: Reset completed
OCTEON[0] Poll Function (Module Starter arg: 0x0) registered
OCTEON[0]: Enabling PCI-E error reporting..
OCTEON: Initializing droq tasklet
OCTEON: Initializing completion on interrupt tasklet
  alloc irq_desc for 146 on node 0
  alloc kstat_irqs on node 0
alloc irq_2_iommu on node 0
Octeon 0000:08:00.0: irq 146 for MSI/MSI-X
OCTEON[0]: MSI enabled
OCTEON: Octeon device 0 is ready

-- OCTEON: Octeon Poll Thread starting execution now!
-- OCTEON: Octeon PCI driver (base module) is ready!
```

# 主结构体`octeon_device_t`
在`components/driver/host/driver/osi/octeon_device.c`中  
定义了`octeon_device[]`数组, 最多支持`MAX_OCTEON_DEVICES(4)`同时存在
```c
octeon_device_t *octeon_device[MAX_OCTEON_DEVICES];
typedef struct _OCTEON_DEVICE octeon_device_t;
```
每个octeon device被抽象为一个`octeon_device_t`结构体, 注意这个主结构是osi的

```c
/** The Octeon device. 
 *  Each Octeon device has this structure to represent all its
 *  components.
 */
struct _OCTEON_DEVICE {
   /** Lock for this Octeon device */
   cavium_spinlock_t        oct_lock; //实际是spinlock_t
   /** OS dependent PCI device pointer */ 
   cavium_pci_device_t     *pci_dev; //linux下面是struct pci_dev
   /** Chip specific information. */
   void                    *chip;
   /** Octeon Chip type. */
   uint16_t                 chip_id;
   uint16_t                 rev_id;
   /** This device's id - set by the driver. */
   uint16_t                 octeon_id;
   /** This device's PCIe port used for traffic. */
   uint16_t                 pcie_port;
   /** The state of this device */
   cavium_atomic_t          status; //是atomic_t
   /** memory mapped io range */
   octeon_mmio              mmio[OCT_MEM_REGIONS]; //3个, 结构体定义见A
   struct octeon_reg_list   reg_list; //见B
   struct octeon_fn_list    fn_list; //见C
   cavium_atomic_t          interrupts;
   cavium_atomic_t          in_interrupt;
   int                      num_iqs;
   /** The 4 input instruction queues */
   octeon_instr_queue_t     *instr_queue[MAX_OCTEON_INSTR_QUEUES]; //32个, 结构体定义在D,很关键
   int                       pend_list_size;
   octeon_pending_list_t    *plist; //见E
   /** The doubly-linked list of instruction response */
   octeon_response_list_t    response_list[MAX_RESPONSE_LISTS]; //3个, 一个ordered, 一个unordered-blocking, 一个unordered-nonblocking, 结构体见F
   int                        num_oqs;
   /** The 4 output queues  */
   octeon_droq_t            *droq[MAX_OCTEON_OUTPUT_QUEUES]; //32个, droq是Descriptor Ring Output Queue, 结构体见G
#if  !defined(USE_DROQ_THREADS)
   /** Tasklet structures for this device. */
   struct tasklet_struct     droq_tasklet;
#endif
   struct tasklet_struct     comp_tasklet;
   uint32_t                  napi_mask;
   void                     *poll_list;
   cavium_spinlock_t         poll_lock;
   struct tasklet_struct     cntq_tasklet;
   uint32_t                  cntq_ready;
   /* The 2 Octeon DMA Counter Queues */
   void                     *cntq[MAX_OCTEON_DMA_QUEUES]; //2个
   /* The DDOQ lookup table */
   void                     *ddoq_list;
   /** Operations on the DMA queues */
   octeon_dma_ops_t          dma_ops; //见H
   /** A table maintaining maps of core-addr to BAR1 mapped address. */
   octeon_range_table_t      range_table[MAX_OCTEON_MAPS]; //最多32个map, 见I
   /** Total number of core-address ranges mapped (Upto 32). */
   uint32_t                  map_count;
   octeon_io_enable_t        io_qmask; //三个uint32_t, iq, oq, iq64B
   /** List of dispatch functions */
   octeon_dispatch_list_t    dispatch; //驱动收到报文后,会依次调这个链表里面的函数, 注意里面的opcode, 见J
#ifdef USE_BUFFER_POOL //如何打开buffer pool???
   /** The buffer pool implementation */
   cavium_buffer_t           buf[BUF_POOLS]; //6个pool, 分别是32k,16k,8k,4k,2k,1k大小的pool, 见K
   cavium_frag_buf_t         fragments[MAX_BUFFER_CHUNKS]; //1500个, 见L
   uint16_t                  fragment_free_list[MAX_BUFFER_CHUNKS];
   uint16_t                  fragment_free_list_index;
   cavium_spinlock_t         fragment_lock;
#endif
   /** The /proc file entries */
   void                     *proc_root_dir;
   /** Statistics for this octeon device. Does not include IQ, DROQ stats */
   oct_dev_stats_t           stats; //统计信息, 都是64位, interrupts poll_count comp_tasklet_count droq_tasklet_count cntq_tasklet_count
   /** IRQ assigned to this device. */
   int                       irq;
   int                       msi_on;
   /** The core application is running in this mode. See octeon-drv-opcodes.h
       for values. */
   int                       app_mode;
#ifdef CVMCS_DMA_IC
   /* When DMA interrupt raised we have these many packets DMAed by Octeon */
   cavium_atomic_t          dma_cnt_to_process;
#endif
#ifdef USE_DDOQ_THREADS
   cvm_ddoq_thread_t	     ddoq_thread[CVM_MAX_DDOQ_THREADS]; //8个或16个, 见M
#endif
   /** The name given to this device. */
   char                      device_name[32];
};
```

# A.描述PCIE空间的结构体
```c
/** PCI address space mapping information.
 *  Each of the 3 address spaces given by BAR0, BAR2 and BAR4 of
 *  Octeon gets mapped to different physical address spaces in
 *  the kernel. 
 */
typedef struct  {
  /** PCI address to which the BAR is mapped. */
   unsigned long   start;
  /** Length of this PCI address space. */
   unsigned long   len;
  /** Length that has been mapped to phys. address space. */
   unsigned long   mapped_len;
  /** The physical address to which the PCI address space is mapped. */
   void           *hw_addr;
  /** Flag indicating the mapping was successful. */
   int             done;
}octeon_mmio;
```

# B.间接访问寄存器, SLI_WIN*
```
struct octeon_reg_list {

	uint32_t    *pci_win_wr_addr_hi;
	uint32_t    *pci_win_wr_addr_lo;
	uint64_t    *pci_win_wr_addr;

	uint32_t    *pci_win_rd_addr_hi;
	uint32_t    *pci_win_rd_addr_lo;
	uint64_t    *pci_win_rd_addr;

	uint32_t    *pci_win_wr_data_hi;
	uint32_t    *pci_win_wr_data_lo;
	uint64_t    *pci_win_wr_data;

	uint32_t    *pci_win_rd_data_hi;
	uint32_t    *pci_win_rd_data_lo;
	uint64_t    *pci_win_rd_data;
};
```

# C.用来适配不同的octeon型号
最基本的设置寄存器的接口
```c
struct octeon_fn_list {

	void                (* setup_iq_regs)(struct _OCTEON_DEVICE *, int);
	void                (* setup_oq_regs)(struct _OCTEON_DEVICE *, int);

	cvm_intr_return_t   (* interrupt_handler)(void *);
	int                 (* soft_reset)(struct _OCTEON_DEVICE *);
	int                 (* setup_device_regs)(struct _OCTEON_DEVICE *);
	void                (* reinit_regs)(struct _OCTEON_DEVICE *);
	void                (* bar1_idx_setup)(struct _OCTEON_DEVICE *, uint64_t, int, int);
	void                (* bar1_idx_write)(struct _OCTEON_DEVICE *, int, uint32_t);
	uint32_t            (* bar1_idx_read)(struct _OCTEON_DEVICE *, int);
	uint32_t            (* update_iq_read_idx)(octeon_instr_queue_t  *);

	void                (* enable_oq_pkt_time_intr)(octeon_device_t *, int );
	void                (* disable_oq_pkt_time_intr)(octeon_device_t *, int );

	void                (* enable_interrupt)(void  *);
	void                (* disable_interrupt)(void  *);

	void                (* enable_io_queues)(struct _OCTEON_DEVICE  *);
	void                (* disable_io_queues)(struct _OCTEON_DEVICE  *);
};
```

# D.表示一个input ring
```c
/** The instruction (input) queue. 
    The input queue is used to post raw (instruction) mode data or packet
    data to Octeon device from the host. Each input queue (upto 4) for
    a Octeon device has one such structure to represent it.
*/
typedef struct   {
  /** A spinlock to protect access to the input ring.  */
   cavium_spinlock_t     lock;
  /** Flag that indicates if the queue uses 64 byte commands. */
  uint32_t               iqcmd_64B:1;
  /** Queue Number. */
  uint32_t               iq_no:5;
  uint32_t               rsvd:18;
  uint32_t               status:8;
  /** Maximum no. of instructions in this queue. */
   uint32_t              max_count;
#if !defined(DISABLE_PCIE14425_ERRATAFIX)
   /** Count of packets that were not sent due to backpressure.  */
   uint32_t              bp_hits;
#endif
  /** Index in input ring where the driver should write the next packet. */
   uint32_t              host_write_index;
  /** Index in input ring where Octeon is expected to read the next packet. */
   uint32_t              octeon_read_index;
  /** This index aids in finding the window in the queue where Octeon 
      has read the commands. */
   uint32_t              flush_index;
  /** This field keeps track of the instructions pending in this queue. */
   cavium_atomic_t       instr_pending;
   uint32_t              reset_instr_cnt;
  /** Pointer to the Virtual Base addr of the input ring. */
   uint8_t              *base_addr;
   octeon_noresponse_list_t    *nrlist; //int buftype, void * buf, 表示NORESPONSE的请求被octeon执行了, 但驱动还没有把资源释放掉
   struct oct_noresp_free_list nr_free; //octeon_noresponse_list_t *q, int put_idx, get_idx;表示需要free的NORESPONSE list?
  /** Octeon doorbell register for the ring. */
   void                 *doorbell_reg;
  /** Octeon instruction count register for this ring. */
   void                 *inst_cnt_reg;
  /** Number of instructions pending to be posted to Octeon. */
   uint32_t              fill_cnt;
  /** The max. number of instructions that can be held pending by the driver. */
   uint32_t              fill_threshold;
  /** The last time that the doorbell was rung. The unit is OS-dependent. */
   unsigned long         last_db_time;
  /** The doorbell timeout. If the doorbell was not rung for this time and 
      fill_cnt is non-zero, ring the doorbell again. */
   unsigned long         db_timeout;
  /** Statistics for this input queue. */
  oct_iq_stats_t         stats;
  /** DMA mapped base address of the input descriptor ring. */
   unsigned long          base_addr_dma;
}  octeon_instr_queue_t;
```

# E.对应主结构体的plist域
pending list是用来处理需要response的request的, 和output ring有关吗? --not only, but also input ring
```c
/** Pending list implementation for each Octeon device. */
typedef  struct  {
   /** Pending list for input instructions */
   octeon_pending_entry_t   *list; //见E1
   /** A list which indicates which entry in the pending_list above is free */
   uint32_t                 *free_list; //free_list是个数组, 有count个uint32
   /** The next location in pending_free_list where an index into pending_list
      can be saved */
   uint32_t                  free_index;
   /** Number of pending list entries. */
   uint32_t                  entries;
   /** Count of pending instructions */
   cavium_atomic_t           instr_count;
   /** A lock to control access to the pending list */
   cavium_spinlock_t         lock; 
} octeon_pending_list_t;
```

## E1.octeon_pending_entry_t
```c
/** Structure of an entry in pending list.
 */
typedef struct  {
  /** Used to add/delete this entry to one of the 3 response lists. */
   cavium_list_t                    list; //双向链表
  /** Index in the input queue where this request was posted */
   uint16_t                         queue_index;
  /** Queue into which request was posted. */
   uint16_t                         iq_no;
  /** Index into pending_list that is returned to the user (for polling) */
   uint32_t                         request_id;
  /** Status of this entry */
   OCTEON_PENDING_ENTRY_STATUS      status; //有4种状态, FREE USED TIMEOUT REMOVE
  /** The instruction itself (not in the format that Octeon sees it)*/
   octeon_soft_instruction_t       *instr; //见E11, 非常重要
}octeon_pending_entry_t;
```

### E11.这个就是和硬件最接近的结构?
--no, 还只是个开始
```c
/** Format of a instruction presented to the driver. This structure has the
    values that get posted to Octeon in addition to other fields that are
    used by the driver to keep track of the instruction's progress.
*/
typedef struct {
	cavium_list_t                    list;
#define COMPLETION_WORD_INIT    0xffffffffffffffffULL
	/** Pointer to the completion status word */
	volatile uint64_t               *status_word;
	/** The timestamp (in ticks) till we wait for a response for this
        instruction. */
	unsigned long                    timeout;
	/**How the response for the instruction should be handled.*/
	octeon_request_info_t            req_info; //见E111
	/** Input data pointer. It is either pointing directly to input data
	    or to a gather list which is a list of addresses where data is present. */
	void                            *dptr; //可以是直接一个指针地址, 也可以是octeon_sg_entry_t 的链表
	/** Response from Octeon comes at this address. It is either pointing to 
	    output data buffer directly or to a scatter list which in turn points 
	    to output data buffers. */
	void                            *rptr; //可以是soft_instr->rptr = SOFT_REQ_OUTBUF(soft_req, 0), 也可以是octeon_sg_entry_t 的链表
	/** The instruction header. All input commands have this field. */
	octeon_instr_ih_t                ih; //见E112
	/** Input request header. */
	octeon_instr_irh_t               irh; //见E113
	/** The PCI instruction to be sent to Octeon. This is stored in the instr
	    to retrieve the physical address of buffers when instr is freed. */
	octeon_instr_64B_t               command; //见E114, 重要! input ring 的64字节命令字
	/** These headers are used to create a 64-byte instruction  */
	uint64_t                         exhdr[4];
	/** Information about the extra headers. */
	octeon_exhdr_info_t              exhdr_info; //见E115
	/** Flags to indicate memory allocated for this instruction. Used by driver
	    when freeing the soft instruction.  */
	uint32_t                         alloc_flags;
	/** If a gather list was allocated, this ptr points to the buffer used for
	    the gather list. The gather list has to be 8B aligned, so this value
	    may be different from dptr.
	*/
	void                            *gather_ptr;
	/** Total data bytes transferred in the gather mode request. */
	uint32_t                         gather_bytes;
	/** If a scatter list was allocated, this ptr points to the buffer used for
	    the scatter list. The scatter list has to be 8B aligned, so this value
	    may be different from rptr.
	*/
	void                            *scatter_ptr;
	/** Total data bytes to be received in the scatter mode request. */
	uint32_t                         scatter_bytes;
}octeon_soft_instruction_t;
```

#### E111.octeon_request_info_t
```c
/** Information about the request sent to driver by kernel mode applications. */
typedef struct  {
  /** The Octeon device to use for this request */
   uint32_t                  octeon_id;
  /** The request mask */
   octeon_request_mask_t     req_mask; //一共32bit, resp_mode:2,dma_mode:2,resp_order:2,ignore_signal:2,iq_no:5,rsvd:19
  /** timeout for this request */
   uint32_t                  timeout;
  /** Status of this request */
   octeon_req_status_t       status; //一个u32
  /** The request id assigned by driver to this request */
   uint32_t                  request_id;
  /** The callback function to call after request completion */
   instr_callback_t          callback; //原型是void(* instr_callback_t)(octeon_req_status_t, void *); 
  /** Argument passed to callback */
   void                     *callback_arg;
} octeon_request_info_t;
```

#### E112.对应到硬件寄存器DPI_INST_HDR
好像是小端模式
```c
typedef struct  {
  /** Tag Value */
  uint64_t     tag:32;
  /** Tag type */
  uint64_t     tagtype:2;
  /** Short Raw Packet Indicator 1=short raw pkt */
  uint64_t     rs:1;
  /** Core group selection (1 of 16) */
  uint64_t     grp:4;
  /** Packet Order / Work Unit selection (1 of 8)*/
  uint64_t     qos:3;
  /** Front Data size */
  uint64_t     fsz:6;
  /** Data length OR no. of entries in gather list */
  uint64_t     dlengsz:14;
  /** Gather indicator 1=gather*/
  uint64_t     gather:1;
  /** Raw mode indicator 1 = RAW */ 
  uint64_t     raw:1;
}octeon_instr_ih_t;
```

#### E113.Input Request Header
```c
/** Input Request Header in LITTLE ENDIAN format */

typedef struct  {
  /** Request ID  */
  uint64_t     rid:16;
  /** PCIe port to use for response */
  uint64_t     pcie_port:3;
  /** Scatter indicator  1=scatter */
  uint64_t     scatter:1;
  /** Size of Expected result OR no. of entries in scatter list */
  uint64_t     rlenssz:14;
  /** Desired destination port for result */
  uint64_t     dport:6;
  /** Opcode Specific parameters */
  uint64_t     param:8;
  /** Opcode for the return packet  */
  uint64_t     opcode:16;
} octeon_instr_irh_t;
```

#### E114.这个就是input ring的一个instruction 
```c
/** 64-byte instruction format.
    Format of instruction for a 64-byte mode input queue.
*/
typedef struct  {
  /** Pointer where the input data is available. */
  uint64_t             dptr;
  /** Instruction Header. */
  uint64_t             ih;
  /** Pointer where the response for a RAW mode packet will be written
      by Octeon. */
  uint64_t             rptr;
  /** Input Request Header. */
  uint64_t             irh;
  /** Additional headers available in a 64-byte instruction. */
  uint64_t             exhdr[4];
}octeon_instr_64B_t;
```

#### E115.extra header
```c
/** Information about each of the extra headers added for a 64-byte
    instruction. */
typedef struct {
   /** The number of 64-bit extra header words in this request. */
   uint64_t    exhdr_count:4;
   /** Use a value of type OCTEON_EXHDR_FMT */
   uint64_t    exhdr1_op:2;
   uint64_t    exhdr2_op:2;
   uint64_t    exhdr3_op:2;
   uint64_t    exhdr4_op:2;
   uint64_t    rsvd:52;
} octeon_exhdr_info_t;
```

# F.头指针, 表示一个response链表
```c
typedef struct {
  /** List structure to add delete pending entries to */
   cavium_list_t head;
  /** A lock for this response list */
   cavium_spinlock_t lock;
} octeon_response_list_t;
```

# G.表示output ring描述符
```c
/** The Descriptor Ring Output Queue structure.
    This structure has all the information required to implement a 
    Octeon DROQ.
*/
typedef struct  {
  /** A spinlock to protect access to this ring. */
   cavium_spinlock_t        lock;
   uint32_t                 q_no;
   uint32_t                 fastpath_on;
   octeon_droq_ops_t        ops; //见G1
   octeon_device_t         *oct_dev; //指向主结构体的指针
#ifdef  USE_DROQ_THREADS
   cvm_kthread_t            thread; //见G2
   cavium_wait_channel      wc; //就是wait_queue_head_t
   int                      stop_thread;
   cavium_atomic_t          thread_active;
#endif
  /** The 8B aligned descriptor ring starts at this address. */
   octeon_droq_desc_t      *desc_ring; //见G3, 这个才是硬件寄存器
  /** Index in the ring where the driver should read the next packet */
   uint32_t                 host_read_index;
  /** Index in the ring where Octeon will write the next packet */
   uint32_t                 octeon_write_index;
  /** Index in the ring where the driver will refill the descriptor's buffer */
   uint32_t                 host_refill_index;
  /** Packets pending to be processed - tasklet implementation */
   cavium_atomic_t          pkts_pending;
  /** Number of  descriptors in this ring. */
   uint32_t                 max_count;
  /** The number of descriptors pending refill. */
   uint32_t                 refill_count;
   uint32_t                 pkts_per_intr;
   uint32_t                 refill_threshold;
  /** The max number of descriptors in DROQ without a buffer.
      This field is used to keep track of empty space threshold. If the
      refill_count reaches this value, the DROQ cannot accept a max-sized
      (64K) packet. */
   uint32_t                 max_empty_descs;
   /** The 8B aligned info ptrs begin from this address. */
   octeon_droq_info_t      *info_list; //见G4
  /** The receive buffer list. This list has the virtual addresses of the
      buffers.  */
   octeon_recv_buffer_t    *recv_buf_list; //见G5, 驱动用来保存虚拟buffer地址
  /** The size of each buffer pointed by the buffer pointer. */
   uint32_t                 buffer_size;
  /** Pointer to the mapped packet credit register.
       Host writes number of info/buffer ptrs available to this register */
   void                    *pkts_credit_reg;
  /** Pointer to the mapped packet sent register.
      Octeon writes the number of packets DMA'ed to host memory
      in this register. */
   void                    *pkts_sent_reg;
#if  defined(ENABLE_PCIE_2G4G_FIX)
   void                    *buffer_block;
#endif
   cavium_list_t            dispatch_list; //双向链表
  /** Statistics for this DROQ. */
   oct_droq_stats_t         stats; //见G6
  /** DMA mapped address of the DROQ descriptor ring. */
   unsigned long            desc_ring_dma;
  /** Info ptr list are allocated at this virtual address. */
   unsigned long            info_base_addr;
  /** Allocated size of info list. */
   uint32_t                 info_alloc_size;
}octeon_droq_t;
```

## G1.octeon_droq_ops_t
```c
/** Used by NIC module to register packet handler and to get device
  * information for each octeon device.
  */
typedef struct {
	/** This registered function will be called by the driver with
	    the octeon id, pointer to buffer from droq and length of 
	    data in the buffer. The response header gives the port 
	    number to the caller.  Function pointer is set by caller.  */
	void       (*fptr)(int, void *, uint32_t, octeon_resp_hdr_t *);
	/* This function will be called by the driver for all NAPI related
	   events. The first param is the octeon id. The second param is the
	   output queue number. The third is the NAPI event that occurred. */
	void       (*napi_fn)(int, int, int );
	int        poll_mode;
	/** Flag indicating if the DROQ handler should drop packets that
	    it cannot handle in one iteration. Set by caller. */
	int        drop_on_max;
	uint16_t   op_mask;
	uint16_t   op_major;
} octeon_droq_ops_t;
```

## G2.cvm_kthread_t
```c
typedef  struct {
#if LINUX_VERSION_CODE < KERNEL_VERSION(2,6,18)
	cavium_pid_t      id; //就是pid_t               
#else
	cavium_ostask_t  *id; //就是struct task_struct
#endif
	int               (*fn)(void *);
	void              *fn_arg;
	char               fn_string[80];
	int                exec_on_create;
} cvm_kthread_t;
```

## G3.真正的符合硬件的定义的结构体
```c
/** Octeon descriptor format.
    The descriptor ring is made of descriptors which have 2 64-bit values:
    -# Physical (bus) address of the data buffer.
    -# Physical (bus) address of a octeon_droq_info_t structure.
    The Octeon device DMA's incoming packets and its information at the address
    given by these descriptor fields.
 */
typedef struct  {
  /** The buffer pointer */
   uint64_t        buffer_ptr;
  /** The Info pointer */
   uint64_t        info_ptr; //octeon_droq_info_t的地址, 见G4
}octeon_droq_desc_t;
```

## G4.octeon_droq_info_t
```c
/** Information about packet DMA'ed by Octeon.
    The format of the information available at Info Pointer after Octeon 
    has posted a packet. Not all descriptors have valid information. Only
    the Info field of the first descriptor for a packet has information
    about the packet. */
typedef struct {
  /** The Output Response Header. */
   octeon_resp_hdr_t    resp_hdr; //见G41
  /** The Length of the packet. */
   uint64_t             length;
}octeon_droq_info_t;
```

### G41.octeon_resp_hdr_t
```c
/** Response Header in LITTLE ENDIAN format */
typedef struct {
  /** The request id for a packet thats in response to pkt sent by host. */
   uint64_t        request_id:16;
  /** Reserved. */
   uint64_t        reserved:4;
  /** The destination Queue port. */
   uint64_t        dest_qport:22;
  /** The source port for a packet thats in response to pkt sent by host. */
   uint64_t        src_port:6;
  /** Opcode for this packet. */
   uint64_t        opcode:16;
} octeon_resp_hdr_t;
```

## G5.驱动用这个来表示buffer, 用的是虚拟地址
```c
/** Pointer to data buffer.
    Driver keeps a pointer to the data buffer that it made available to 
    the Octeon device. Since the descriptor ring keeps physical (bus)
    addresses, this field is required for the driver to keep track of
    the virtual address pointers. The fields are operated by
    OS-dependent routines.
*/
typedef struct  {
  /** Pointer to the packet buffer. Hidden by void * to make it OS independent.
         */
     void        *buffer;
  /** Pointer to the data in the packet buffer.
      This could be different or same as the buffer pointer depending
      on the OS for which the code is compiled. */
     uint8_t        *data;
} octeon_recv_buffer_t;
```

## G6. oct_droq_stats_t
```c
/** Output Queue statistics. Each output queue has four stats fields. */
typedef struct {
  uint64_t   pkts_received; /**< Number of packets received in this queue. */
  uint64_t   bytes_received;/**< Bytes received by this queue. */
  uint64_t   dropped_nodispatch; /**< Packets dropped due to no dispatch function. */
  uint64_t   dropped_nomem; /**< Packets dropped due to no memory available. */
  uint64_t   dropped_toomany; /**< Packets dropped due to large number of pkts to process. */
} oct_droq_stats_t;
```

# H. octeon_dma_ops_t
```c
/** Used by CNTQ module to register DMA queue interrupt handler, tasklets
  * and statistics routines for each octeon device.
  */
typedef struct {
	/** Tasklet to be scheduled for CNTQ bottom half processing. */
	void      (*bh)(unsigned long);
	/** Interrupt Handler for DMA Queue interrupts. */
	int       (*intr_handler)(void  *, uint64_t);
	/** Read DMA Counter Queue and DDOQ list statistics into a structure */
	int       (* read_statsb)(int, oct_stats_t *);
	/** Format and print DMA Counter Queue and DDOQ list statistics into a buffer */
	int       (* read_stats)(int, char *);
} octeon_dma_ops_t;
```

# I. octeon_range_table_t
```c
/** Map of Octeon core memory address to Octeon BAR1 indexed space. */
typedef  struct  {
  /** Starting Core address mapped */
  uint64_t      core_addr;
  /** Physical address (of the BAR1 mapped space) 
      corressponding to core_addr. */
  void       *mapped_addr;
  /** Indicator that the mapping is valid. */
  int         valid;
} octeon_range_table_t;
```

# J.收到报文后按opcode分发
```c
/** The dispatch list entry.
 *  The driver keeps a record of functions registered for each 
 *  response header opcode in this structure. Since the opcode is
 *  hashed to index into the driver's list, more than one opcode
 *  can hash to the same entry, in which case the list field points
 *  to a linked list with the other entries.
 */
typedef struct {
  /** List head for this entry */
    cavium_list_t             list;
  /** The opcode for which the above dispatch function & arg should be
      used */
    octeon_opcode_t           opcode;
  /** The function to be called for a packet received by the driver */
    octeon_dispatch_fn_t      dispatch_fn;
  /** The application specified argument to be passed to the above
      function along with the received packet */
    void                      *arg;
} octeon_dispatch_t;


/** The dispatch list structure. */
typedef struct {
	cavium_spinlock_t      lock;
	/** Count of dispatch functions currently registered */
	uint32_t               count;
	/** The list of dispatch functions */
	octeon_dispatch_t     *dlist;
} octeon_dispatch_list_t;
```

# K. buffer pool
```c
/** Each buffer pool is represented by this structure.
 */
typedef struct {
  /** Lock for this pool. */
   cavium_spinlock_t buffer_lock;
  /** Number of chunks in this pool. */
   int chunks;
  /** Size of each chunk available for use after allocation. */
   int chunk_size;
  /** Actual size of each chunk. (includes size of buffer tag)*/
   int real_size;
   uint8_t *base;
  /** Address of each chunk. */
   uint8_t *address[MAX_BUFFER_CHUNKS];
  /** Address of usable space in chunk ( chunk - buffer tag) */
   uint8_t *address_trans[MAX_BUFFER_CHUNKS];
  /** Free list for this pool. */
   uint16_t free_list[MAX_BUFFER_CHUNKS];
  /** The next location in free list where a buffer is available. */
   int free_list_index;
  /** Start of head for this pool's fragment list.  */
   cavium_list_t frags_list;
} cavium_buffer_t;
```

# L. buffer pool的fragment
```c
/** List to keep track of fragmented buffers in the buffer pool. */
typedef struct {
	cavium_list_t list;
	cavium_list_t alloc_list;
	uint8_t *big_buf;
	int frags_count;
	int index;
	OCTEON_BUFPOOL p;
	uint16_t free_list[MAX_FRAGMENTS];
	uint8_t *address[MAX_FRAGMENTS];
	int free_list_index;
	int not_allocated;
} cavium_frag_buf_t;
```

# M.USE_DDOQ_THREADS
```c
typedef struct cvm_ddoq_thread_info {
	int ddoq_id;
	int req_id;
	int num_pkts;
} octeon_ddoq_thread_info_t;

typedef struct cvm_ddoq_thread {
   octeon_device_t	     *oct_dev; //主结构体的指针
   cvm_kthread_t	     thread; //见G2
   cavium_wait_channel	 wc; //还是wait_queue_head_t
   int 			     stop_thread;
   cavium_atomic_t	     thread_active;
   cavium_atomic_t      ddoq_pkts_queued;

   cavium_spinlock_t    th_lock;
   int                  th_read_idx;
   int                  th_write_idx;
   
   /* On each interrupt we can handle at most */
   octeon_ddoq_thread_info_t th_info[CVM_DDOQ_MAX_THREAD_PKTS];
} cvm_ddoq_thread_t;
```

# octeon_module_handler_t
当octeon device被初始化 被复位 被停止时调用的附加函数
```c
/** Structure passed by kernel application when registering a module with
	the driver. */
typedef struct {

	/* Application type for which handler is being registered. */
	uint32_t   app_type;

	/* Call this routine to perform add-on module related setup activities
	   when a octeon device is being initialized.
	 */
	int   (*startptr)(int, void *);

	/* Call this routine to perform add-on module related reset activities
	   when a octeon device is being reset.
	 */
	int   (*resetptr)(int, void *);

	/* Call this routine to perform add-on module related shutdown 
	   activities when a octeon device is being removed or the driver is 
	   being unloaded.
         */
	int   (*stopptr)(int, void *);

} octeon_module_handler_t;
```

# octeon_cn6xxx_t
```c
/* Register address and configuration for a CN6XXX devices. */
/* If device specific changes need to be made then add a struct to include
   device specific fields as shown in the commented section */
typedef struct {

	/** PCI interrupt summary register */
	uint8_t            *intr_sum_reg64;

	/** PCI interrupt enable register */
	uint8_t            *intr_enb_reg64;

	/** The PCI interrupt mask used by interrupt handler */
	uint64_t            intr_mask64;

	cn6xxx_config_t    *conf; //见2A

	/* Example additional fields - not used currently
	struct {
	}cn6xyz;
	*/

} octeon_cn6xxx_t;
```

## 2A.cn6xxx_config_t
```c
/** Structure to define the configuration for CN61XX,CN63XX,CN66XX & CN68XX Octeon processors. */
typedef  struct {
	/** Common attributes. */
	octeon_common_config_t  c; //num_iqs num_oqs pending_list_size
	/** Input Queue attributes. */
	/*num_descs: 每个ring有多少个命令
	**instr_type: 是32bit还是64bit格式的
	**db_min: 发doorbell之前需要准备好多少个command
	**db_timeout: 在查询pending的command之前的超时时间
	*/
	octeon_iq_config_t      iq[CN6XXX_MAX_INPUT_QUEUES];

	/** Some of the Output Queue attributes. */
	/*num_descs: 每个ring多少个描述符
	**info_ptr: 默认是1, 表示使用info指针模式
	**buf_size: buffer大小
	**pkts_per_intr: 每次调用driver的tasklet要处理多少个packets
	**refill_threshold: 小于此值, 驱动需要充填(replenish)
	*/
	cn6xxx_oq_config_t      oq[CN6XXX_MAX_OUTPUT_QUEUES];
	/** Interrupt Coalescing (Packet Count). Octeon will interrupt the host
	    only if it sent as many packets as specified by this field. The driver
	    usually does not use packet count interrupt coalescing. 
	    All output queues have the same packet count setting. */
	//两种中断机制之一, 时间间隔
	uint32_t                oq_intr_pkt;
	/** Interrupt Coalescing (Time Interval). Octeon will interrupt the host
	    if atleast one packet was sent in the time interval specified by this
	    field. The driver uses time interval interrupt coalescing by default.
	    The time is specified in microseconds.
	    All output queues have the same time interval setting. */
	uint32_t                oq_intr_time;
#ifdef CVMCS_DMA_IC
	/** Interrupt Coalescing (Packet Count). Octeon will interrupt the host
	    only if it DMAed as many packets as specified by this field. */
	//两种中断机制之二, 已经DMA的报文数
	uint32_t                dma_intr_pkt;
	/** Interrupt Coalescing (Time Interval). Octeon will interrupt the host
	    if atleast one packet was DMAed in the time interval specified by this
	    field. */ 
	uint32_t                dma_intr_time;
#endif
} cn6xxx_config_t;
```
