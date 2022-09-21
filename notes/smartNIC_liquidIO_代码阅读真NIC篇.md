这个驱动会把所有octeon的接口注册为host上的网口

- [结构体](#结构体)
  - [octdev_props_t](#octdev_props_t)
  - [oct_link_status_resp_t](#oct_link_status_resp_t)
  - [被用来做net_dev的priv](#被用来做net_dev的priv)
  - [glist](#glist)
- [A. init_module](#a-init_module)
  - [A1. octnet_init_nic_module](#a1-octnet_init_nic_module)
    - [A11. octnic_free_netbuf和octnic_free_netsgbuf](#a11-octnic_free_netbuf和octnic_free_netsgbuf)
    - [A12. octnet_setup_nic_device](#a12-octnet_setup_nic_device)
      - [A121. octnet_push_packet](#a121-octnet_push_packet)
- [B. octnetdevops](#b-octnetdevops)
  - [B1. octnet_open](#b1-octnet_open)
  - [B2. octnet_xmit 网口发送函数](#b2-octnet_xmit-网口发送函数)

# 结构体
## octdev_props_t
```c
/** Octeon device properties to be used by the NIC module.
    Each octeon device in the system will be represented
    by this structure in the NIC module. */
struct  octdev_props_t {
	/** Number of interfaces detected in this octeon device. */
	int                        ifcount;
	/* Link status sent by core app is stored in a buffer at this
	   address. */
	oct_link_status_resp_t     *ls;
	/** Pointer to pre-allocated soft instr used to send link status
	    request to Octeon app. */
	octeon_soft_instruction_t  *si_link_status;
	/** Flag to indicate if a link status instruction is currently
	    being processed. */
	cavium_atomic_t             ls_flag;
	/** The last tick at which the link status was checked. The
	    status is checked every second. */
	unsigned long               last_check;
	/** Each interface in the Octeon device has a network
	   device pointer (used for OS specific calls). */
	octnet_os_devptr_t         *pndev[MAX_OCTEON_LINKS]; //是linux net_device 
};
```
## oct_link_status_resp_t
```c
typedef struct {
	struct {
		int                   octeon_id;
		cavium_wait_channel   wc;
		int                   cond;
	} s;
	uint64_t          resp_hdr;
	uint64_t          link_count;
	//一个link_info就是octeon的一个接口, 将来会被注册为host的网口
	oct_link_info_t   link_info[MAX_OCTEON_LINKS];
	uint64_t          status;
} oct_link_status_resp_t;
```

## 被用来做net_dev的priv
```c
/** Octeon per-interface Network Private Data */
typedef  struct {
	cavium_spinlock_t             lock;
	/** State of the interface. Rx/Tx happens only in the RUNNING state.  */
	atomic_t                      ifstate;
	/** Octeon Interface index number. This device will be represented as
	    oct<ifidx> in the system. */
	int                           ifidx;
	/** Octeon Input queue to use to transmit for this network interface. */
	int                           txq; //只有一个txq说明octeon的么个interface只能用一个q
	/** Octeon Output queue from which pkts arrive for this network interface.*/
	int                           rxq; //只有一个rxq说明octeon的么个interface只能用一个q
	/** Linked list of gather components */
	cavium_list_t                 glist; //比如q的最大深度是n, 则每个格子都一个glist --gather list
	/** Pointer to the NIC properties for the Octeon device this network
	    interface is associated with. */
	struct  octdev_props_t       *octprops;
	/** Pointer to the octeon device structure. */
	void                         *oct_dev;
	octnet_os_devptr_t           *pndev;
#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,24)
	struct napi_struct            napi;
#endif
	/** Link information sent by the core application for this interface. */
	oct_link_info_t               linfo;
	/** Statistics for this interface. */
	struct net_device_stats       stats;
	/** Size of Tx queue for this octeon device. */
	uint32_t                      tx_qsize;
	/** Size of Rx queue for this octeon device. */
	uint32_t                      rx_qsize;
	/** Copy of netdevice flags. */
	uint32_t                      pndev_flags;
	/* Copy of the flags managed by core app & NIC module. */
	octnet_ifflags_t              core_flags;
} octnet_priv_t;
#define OCTNET_PRIV_SIZE   (sizeof(octnet_priv_t))
```

## glist
```c
/** Structure of a node in list of gather components maintained by
	NIC driver for each network device. */
struct octnic_gather {
	/** List manipulation. Next and prev pointers. */
	cavium_list_t        list;
	/** Size of the gather component at sg in bytes. */
	int                  sg_size;
	/** Number of bytes that sg was adjusted to make it 8B-aligned. */
	int                  adjust;
	/** Gather component that can accomodate max sized fragment list
	    received from the IP layer. */
	octeon_sg_entry_t   *sg;
};
```

# A. init_module
```c
init_module()
	octeon_module_handler_t   nethandler;

	/* Register handlers with the BASE driver. For each octeon device that
	   runs the NIC core app, the BASE driver would call the functions
	   below for initialization, reset and shutdown operations. */
	//会被base调用, 比如在设备重启后
	nethandler.startptr = octnet_init_nic_module; //见A1
	nethandler.resetptr = octnet_reset_nic_module;
	nethandler.stopptr  = octnet_stop_nic_module;
	nethandler.app_type = CVM_DRV_NIC_APP;
	octeon_register_module_handler(&nethandler)
```

## A1. octnet_init_nic_module
```c
octnet_init_nic_module(int octeon_id, void *octeon_dev)

	oct_link_status_resp_t     *ls = NULL;
	octeon_soft_instruction_t  *si = NULL;
	int                         ifidx, retval = 0;

	//见上面结构体octdev_props_t
	octprops[octeon_id] = cavium_alloc_virt(sizeof(struct octdev_props_t));

	/* Allocate a buffer to collect link status from the core app. */
	ls = cavium_malloc_dma(sizeof(oct_link_status_resp_t),
		                    __CAVIUM_MEM_GENERAL);
	octprops[octeon_id]->ls = ls;

	/* Allocate a soft instruction to be used to send link status requests
	   to the core app. */
	si = (octeon_soft_instruction_t *)
	      cavium_alloc_buffer(octeon_dev, OCT_SOFT_INSTR_SIZE);

	octprops[octeon_id]->si_link_status = si;
	//用于link status的命令字
	octnet_prepare_ls_soft_instr(octeon_dev, si);
		cavium_memset(si, 0, OCT_SOFT_INSTR_SIZE);
		si->ih.fsz     = 16;
		si->ih.tagtype = ORDERED_TAG;
		si->ih.tag     = 0x11111111;
		si->ih.raw     = 1;
		si->irh.opcode = HOST_NW_INFO_OP;
		si->irh.param  = 32;
		SET_SOFT_INSTR_DMA_MODE(si, OCTEON_DMA_DIRECT);
		SET_SOFT_INSTR_RESP_ORDER(si, OCTEON_RESP_ORDERED);
		SET_SOFT_INSTR_RESP_MODE(si, OCTEON_RESP_NON_BLOCKING);
		SET_SOFT_INSTR_IQ_NO(si, 0);
		SET_SOFT_INSTR_TIMEOUT(si, 100);

		/* Since this instruction is sent in the poll thread context, if the
		   doorbell coalescing is > 1, the doorbell will never be rung for
		   this instruction (this call has to return for poll thread to hit
		   the doorbell). So enforce the doorbell ring. */
		SET_SOFT_INSTR_ALLOCFLAGS(si, OCTEON_SOFT_INSTR_DB_NOW);
		si->dptr        = NULL;
		si->ih.dlengsz  = 0;

	/* Send an instruction to get the link status information from core. */
	//这个函数会阻塞等待, 注意: 这里是典型的调用发送接口的例子
	octnet_get_inittime_link_status(octeon_dev, octprops[octeon_id])
		struct octdev_props_t       *props;
		octeon_soft_instruction_t   *si;
		oct_link_status_resp_t      *ls;
		octeon_instr_status_t        retval;

		props = (struct octdev_props_t  *)props_ptr;

		/* Use the link status soft instruction pre-allocated
		   for this octeon device. */
		si = props->si_link_status;

		/* Reset the link status buffer in props for this octeon device. */
		ls = props->ls;
		cavium_memset(ls, 0, OCT_LINK_STATUS_RESP_SIZE);

		cavium_init_wait_channel(&ls->s.wc);
		//注意: 这里的rptr是ls的一个成员的地址, ls是上面用cavium_malloc_dma申请的
		si->rptr           = &(ls->resp_hdr);
		//这里的size是octeon回复的size, 从ls->resp_hdr地址开始.
		//是oct_link_status_resp_t去掉s的大小
		si->irh.rlenssz    = ( OCT_LINK_STATUS_RESP_SIZE - sizeof(ls->s) );
		//octeon返回的status
		si->status_word    = (uint64_t *)&(ls->status);
		*(si->status_word) = COMPLETION_WORD_INIT;
		ls->s.cond         = 0;
		ls->s.octeon_id    = get_octeon_device_id(oct);
		SET_SOFT_INSTR_OCTEONID(si, ls->s.octeon_id);
		//这个callback是收包(准确的说是收到response)以后的回调
		//作用就是把	ls->s.cond = 1;	
		SET_SOFT_INSTR_CALLBACK(si, octnet_inittime_ls_callback);
		SET_SOFT_INSTR_CALLBACK_ARG(si, (void *)ls);
		//发送命令字
		retval = octeon_process_instruction(oct, si, NULL);
		//等待ls->s.cond = 1
		/* Sleep on a wait queue till the cond flag indicates that the
		   response arrived or timed-out. */
		cavium_sleep_timeout_cond(&ls->s.wc, (int *)&ls->s.cond, 1000);

		return(ls->status);

	/* The link count should be swapped on little endian systems. */
	octeon_swap_8B_data(&(ls->link_count), 1);

	octeon_swap_8B_data((uint64_t *)ls->link_info,
		(ls->link_count * (OCT_LINK_INFO_SIZE >> 3)));

	//这里是打印从octeon获取到的接口信息
	for(ifidx = 0; ifidx < ls->link_count; ifidx++) {
		printk("OCTNIC: if%d rxq: %d txq: %d gmx: %d hw_addr: 0x%llx\n",
	             ifidx, ls->link_info[ifidx].rxpciq, ls->link_info[ifidx].txpciq, ls->link_info[ifidx].gmxport, CVM_CAST64(ls->link_info[ifidx].hw_addr));
	}

	octprops[octeon_id]->ifcount = ls->link_count;
	//注册noresponse的释放buf函数, 见driver篇A14
	//这两个free的回调函数见A11
	octeon_register_noresp_buf_free_fn(octeon_id, NORESP_BUFTYPE_NET,
		octnic_free_netbuf);
	octeon_register_noresp_buf_free_fn(octeon_id, NORESP_BUFTYPE_NET_SG,
		octnic_free_netsgbuf);

	//注册host上的网口, 见A12
	for(ifidx = 0; ifidx < ls->link_count; ifidx++)
		     octnet_setup_nic_device(octeon_id, &ls->link_info[ifidx], ifidx);

	cavium_atomic_set(&octprops[octeon_id]->ls_flag, LINK_STATUS_FETCHED);
	octprops[octeon_id]->last_check = cavium_jiffies;

	/* Register a poll function to run every second to collect and update
	   link status. */
	octeon_poll_ops_t           poll_ops;
	poll_ops.fn     = octnet_get_runtime_link_status;
	poll_ops.fn_arg = (unsigned long)octprops[octeon_id];
	poll_ops.ticks  = CAVIUM_TICKS_PER_SEC;
	strcpy(poll_ops.name, "NIC Link Status");
	octeon_register_poll_fn(octeon_id, &poll_ops);

	cavium_print_msg("OCTNIC: Network interfaces ready for Octeon %d\n",
	                 octeon_id);
```

### A11. octnic_free_netbuf和octnic_free_netsgbuf
```c
octnic_free_netbuf(void *buf)
	struct sk_buff               *skb;
	struct octnet_buf_free_info  *finfo;
	octnet_priv_t                *priv;
	//其实是sk_buff的一个成员cb
	finfo = (struct octnet_buf_free_info  *)buf;
	skb   = finfo->skb;
	priv  = finfo->priv;

	octeon_unmap_single_buffer(get_octeon_device_id(priv->oct_dev),
	                           finfo->dptr, skb->len,
	                           CAVIUM_PCI_DMA_TODEVICE);
	free_recv_buffer((cavium_netbuf_t *)skb);

	__check_txq_state(priv);
```

```c
octnic_free_netsgbuf(void *buf)
	struct octnet_buf_free_info  *finfo;
	struct sk_buff               *skb;
	octnet_priv_t                *priv;
	struct octnic_gather         *g;
	int                           i, frags;

	finfo    = (struct octnet_buf_free_info *)buf;
	skb      = finfo->skb;
	priv     = finfo->priv;
	g        = finfo->g;
	frags    = skb_shinfo(skb)->nr_frags;

	octeon_unmap_single_buffer(get_octeon_device_id(priv->oct_dev),
	                           g->sg[0].ptr[0], (skb->len - skb->data_len),
	                           CAVIUM_PCI_DMA_TODEVICE);
	i = 1;
	while(frags--) {
		struct skb_frag_struct  *frag = &skb_shinfo(skb)->frags[i-1];

		octeon_unmap_page(get_octeon_device_id(priv->oct_dev),
		                  g->sg[(i >> 2)].ptr[(i&3)],
		                  frag->size, CAVIUM_PCI_DMA_TODEVICE);
		i++;
	}

	octeon_unmap_single_buffer(get_octeon_device_id(priv->oct_dev),
	                           finfo->dptr, g->sg_size,
	                           CAVIUM_PCI_DMA_TODEVICE);

	cavium_spin_lock(&priv->lock);
	//把这个g回收再利用
	cavium_list_add_tail(&g->list, &priv->glist);
	cavium_spin_unlock(&priv->lock);

	free_recv_buffer((cavium_netbuf_t *)skb);

	__check_txq_state(priv);
```

### A12. octnet_setup_nic_device
```c
octnet_setup_nic_device(int octeon_id, oct_link_info_t  *link_info, int ifidx)
	octnet_priv_t       *priv;
	octnet_os_devptr_t  *pndev;
	uint8_t              macaddr[6], i;

	pndev = octnet_alloc_netdev(OCTNET_PRIV_SIZE);

	octprops[octeon_id]->pndev[ifidx] = pndev;

	/* Associate the routines that will handle different netdev tasks. */
	//这个网口的操作函数, 见B
	pndev->netdev_ops          = &octnetdevops;
	/* Can checksum all the packets. */  
	pndev->features  = NETIF_F_HW_CSUM;
	/* Scatter/gather IO. */
	pndev->features |=  NETIF_F_SG;

	priv            = GET_NETDEV_PRIV(pndev);
	cavium_memset(priv, 0, sizeof(octnet_priv_t));

	priv->ifidx     = ifidx;

	/* Point to the  properties for octeon device to which this interface
	   belongs. */
	priv->oct_dev   = get_octeon_device_ptr(octeon_id);
	priv->octprops  = octprops[octeon_id];
	priv->pndev     = pndev;
	cavium_spin_lock_init(&(priv->lock));

	/* Record the ethernet port number on the Octeon target for this
       interface. */
	priv->linfo.gmxport = link_info->gmxport;

	/* Record the pci port that the core app will send and receive packets
	   from host for this interface. */
	priv->linfo.ifidx   = link_info->ifidx;
	priv->linfo.hw_addr = link_info->hw_addr;
	priv->linfo.txpciq  = link_info->txpciq;
	priv->linfo.rxpciq  = link_info->rxpciq;

	//默认是关的, napi见wiz笔记<数据包接收系列 — NAPI的原理和实现>
	if(OCT_NIC_USE_NAPI)
		octnet_setup_napi(priv);
			netif_napi_add(priv->pndev, &priv->napi, octnet_napi_poll, 64);


	cavium_print(PRINT_DEBUG, "OCTNIC: if%d gmx: %d hw_addr: 0x%llx\n",
	             ifidx, priv->linfo.gmxport, CVM_CAST64(priv->linfo.hw_addr));

	/* 64-bit swap required on LE machines */
	octeon_swap_8B_data(&priv->linfo.hw_addr, 1);
	for(i = 0; i < 6; i++)
		macaddr[i] = *((uint8_t *)(((uint8_t *)&priv->linfo.hw_addr) + 2 + i));

	/* Copy MAC Address to OS network device structure */
	cavium_memcpy(pndev->dev_addr, &macaddr, ETH_ALEN);

	priv->linfo.link.u64 = link_info->link.u64;

	//这里的size是指q的深度
	priv->tx_qsize = octeon_get_tx_qsize(octeon_id, priv->txq);
	priv->rx_qsize = octeon_get_rx_qsize(octeon_id, priv->rxq);

	octnet_setup_glist(priv)
		int                    i;
		struct octnic_gather  *g;

		CAVIUM_INIT_LIST_HEAD(&priv->glist);
		//这个q的每个格子都有个glist
		for(i = 0; i < priv->tx_qsize; i++)
			g = cavium_malloc_dma(sizeof(struct octnic_gather),
				               __CAVIUM_MEM_GENERAL);
			//OCTNIC_MAX_SG一般是kernel的MAX_SKB_FRAGS, 此值一般为18
			//sg entry参考driver篇C1231
			g->sg_size = ((ROUNDUP4(OCTNIC_MAX_SG) >> 2)* OCT_SG_ENTRY_SIZE);
			//为sg entry数组分配空间
			g->sg = cavium_malloc_dma(g->sg_size + 8, __CAVIUM_MEM_GENERAL);

			/* The gather component should be aligned on a 64-bit boundary. */
			if( ((unsigned long)g->sg) & 7)
				g->adjust = 8 - ( ((unsigned long)g->sg) & 7);
				g->sg = (octeon_sg_entry_t *)((unsigned long)g->sg + g->adjust);
			//以g的链表元素list为节点, 加到glist链表
			cavium_list_add_tail(&g->list, &priv->glist);

	OCTNET_IFSTATE_SET(priv, OCT_NIC_IFSTATE_DROQ_OPS);

	/* Register the network device with the OS */
	register_netdev(pndev)

	netif_carrier_off(pndev);

	if(priv->linfo.link.s.status)
		netif_carrier_on(pndev);
		octnet_start_txqueue(pndev);
	else
		netif_carrier_off(pndev);

	/* Register the fast path function pointers after the network device
	   related activities are completed. We should be ready for Rx at this
	   point. */
	/* By default all interfaces on a single Octeon uses the same tx and rx
	   queues */
	priv->txq = priv->linfo.txpciq;
	priv->rxq = priv->linfo.rxpciq;
	octnet_setup_net_queues(octeon_id, priv)
		octeon_droq_ops_t    droq_ops;

		memset(&droq_ops, 0, sizeof(octeon_droq_ops_t));

		droq_ops.fptr        = octnet_push_packet; //收包函数, 见A121

		if(OCT_NIC_USE_NAPI)
			droq_ops.poll_mode  = 1;
			droq_ops.napi_fn    = octnet_napi_drv_callback;
		else
			droq_ops.drop_on_max = 1;

		/* Register the droq ops structure so that we can start handling packets
		 * received on the Octeon interfaces. */
		//注册fast path, 在收包线程和收包bh里面用到；见A18和A19
		octeon_register_droq_ops(octeon_id, priv->rxq, &droq_ops)

	if(OCT_NIC_USE_NAPI)
		octnet_napi_enable(priv);

	OCTNET_IFSTATE_SET(priv, OCT_NIC_IFSTATE_REGISTERED);

	octnet_send_rx_ctrl_cmd(priv, 1);
		octnic_ctrl_pkt_t    nctrl;

		memset(&nctrl, 0, sizeof(octnic_ctrl_pkt_t));

		nctrl.ncmd.s.cmd    = OCTNET_CMD_RX_CTL;
		nctrl.ncmd.s.param1 = priv->linfo.ifidx;
		nctrl.ncmd.s.param2 = start_stop;
		nctrl.netpndev      = (unsigned long)priv->pndev;

		octnet_send_nic_ctrl_pkt(priv->oct_dev, &nctrl)
			si = octnic_alloc_ctrl_pkt_si(oct, nctrl);
			retval = octeon_process_instruction(oct, si, NULL);

	octnet_print_link_info(pndev);
```

#### A121. octnet_push_packet
在收包线程或收包bh里面被调用, 负责把报文向上传递给协议栈
```c
octnet_push_packet(int                  octeon_id,
                   void                *skbuff,
                   uint32_t             len,
                   octeon_resp_hdr_t   *resp_hdr)

	struct sk_buff     *skb   = (struct sk_buff *)skbuff;
	octnet_os_devptr_t *pndev = (octnet_os_devptr_t *)octprops[octeon_id]->pndev[resp_hdr->dest_qport];

	if(pndev)
		octnet_priv_t  *priv  = GET_NETDEV_PRIV(pndev);
	
		/* Do not proceed if the interface is not in RUNNING state. */
		if( !(cavium_atomic_read(&priv->ifstate) & OCT_NIC_IFSTATE_RUNNING)) {
			free_recv_buffer(skb);
			priv->stats.rx_dropped++;
			return;
		}

		skb->dev       = pndev;
		skb->protocol  = eth_type_trans(skb, skb->dev);
		skb->ip_summed = CHECKSUM_NONE;
		//用linux收包接口netif_rx()交到协议栈
		if(netif_rx(skb) != NET_RX_DROP)
			priv->stats.rx_bytes += len;
			priv->stats.rx_packets++;
			pndev->last_rx  = jiffies;
		else
			priv->stats.rx_dropped++;
```

# B. octnetdevops
```c
const static struct net_device_ops  octnetdevops = {
	.ndo_open                = octnet_open, //见B1
	.ndo_stop                = octnet_stop,
	.ndo_start_xmit          = octnet_xmit, //见B2
	.ndo_get_stats           = octnet_stats,
	.ndo_set_mac_address     = octnet_set_mac,
	.ndo_set_multicast_list  = octnet_set_mcast_list,
	.ndo_tx_timeout          = octnet_tx_timeout,
	.ndo_change_mtu          = octnet_change_mtu,
};
```

## B1. octnet_open
```c
octnet_open(struct net_device *pndev)
{
	octnet_priv_t      *priv = GET_NETDEV_PRIV(pndev);
	//告诉上层, 可以往驱动层发包了
	netif_start_queue(pndev);
	//现在是running态
	OCTNET_IFSTATE_SET(priv, OCT_NIC_IFSTATE_RUNNING);
	//注册一个poll函数, octnet_poll_check_txq_status
	__setup_tx_poll_fn(pndev);
		octeon_poll_ops_t    poll_ops;
		octnet_priv_t       *priv = GET_NETDEV_PRIV(pndev);

		poll_ops.fn     = octnet_poll_check_txq_status;
		poll_ops.fn_arg = (unsigned long)priv;
		poll_ops.ticks  = 1;
		poll_ops.rsvd   = 0xff;
		octeon_register_poll_fn(get_octeon_device_id(priv->oct_dev), &poll_ops);

	CVM_MOD_INC_USE_COUNT;

	return 0;
}
```

## B2. octnet_xmit 网口发送函数
```c
/** This structure is used by NIC driver to store information required
	to free the sk_buff when the packet has been fetched by Octeon.
	Bytes offset below assume worst-case of a 64-bit system. */
struct octnet_buf_free_info {

	/** Bytes 1-8.  Pointer to network device private structure. */
	octnet_priv_t        *priv;

	/** Bytes 9-16.  Pointer to sk_buff. */
	struct sk_buff       *skb;

	/** Bytes 17-24.  Pointer to gather list. */
	struct octnic_gather *g;

	/** Bytes 25-32. Physical address of skb->data or gather list. */
    uint64_t              dptr;
};
```

```c
octnet_xmit(struct sk_buff *skb, struct net_device *pndev)
	octnet_priv_t                 *priv;
	struct octnet_buf_free_info   *finfo;
	octnic_cmd_setup_t             cmdsetup;
	octnic_data_pkt_t              ndata;
	int                            status = 0;

	priv = GET_NETDEV_PRIV(pndev);
	priv->stats.tx_packets++;

	if(!OCTNET_IFSTATE_CHECK(priv, OCT_NIC_IFSTATE_TXENABLED))
		return OCT_NIC_TX_BUSY;

	//检查条件
	if( !(cavium_atomic_read(&priv->ifstate) &  OCT_NIC_IFSTATE_RUNNING)
		|| (!priv->linfo.link.s.status)
		|| (octnet_iq_is_full(priv->oct_dev, priv->txq))
		|| (skb->len <= 0) ) {
		goto oct_xmit_failed;
	}

	/* Use space in skb->cb to store info used to unmap and free the buffers. */
	//这里看不懂; 为什么skb->cb就能是octnet_buf_free_info?? 协议栈按理说不应该知道这个结构体啊????
	//解答: 在sk_buff的定义里cb是个48个字节的自由使用区 char cb[48] __aligned(8);
	//octnet_buf_free_info的定义见上面, 可见它只用了32个字节---标记:B2i1
	finfo       = (struct octnet_buf_free_info *)skb->cb;
	finfo->priv = priv;
	finfo->skb  = skb;

	/* Prepare the attributes for the data to be passed to OSI. */
	ndata.buf       = (void *)finfo;
	ndata.q_no      = priv->txq;
	ndata.datasize  = skb->len;

	cmdsetup.u64       = 0;
	cmdsetup.s.ifidx = priv->linfo.ifidx;


	if( (is_ipv4(skb) && !is_ip_fragmented(skb) && is_tcpudp(skb)) ||
		(is_ipv6(skb) && is_wo_extn_hdr(skb)) ) {
			cmdsetup.s.cksum_offset = sizeof(struct ethhdr) + 1;
	}

	/*重要!:不管有没有frags, 这里都做到了直接使用skb做为DMA的内存*/
	//没有frags, 即只有一个buffer
	if(skb_shinfo(skb)->nr_frags == 0)
		cmdsetup.s.u.datasize = skb->len;
		octnet_prepare_pci_cmd(&(ndata.cmd), &cmdsetup);
			volatile octeon_instr_ih_t     *ih;
			volatile octeon_instr_irh_t    *irh;

			cmd->ih      = 0;
			ih           = (octeon_instr_ih_t *)&cmd->ih;

			ih->fsz      = 16;
			ih->tagtype  = ORDERED_TAG;
			ih->grp      = OCTNET_POW_GRP;
			ih->tag      = 0x11111111 + setup->s.ifidx;
			ih->raw      = 1;

			if(!setup->s.gather)
				ih->dlengsz  = setup->s.u.datasize;
			else
				ih->gather   = 1;
				ih->dlengsz  = setup->s.u.gatherptrs;
			//rptr为0说明是noresponse
			cmd->rptr    = 0;
			cmd->irh     = 0;
			irh          = (octeon_instr_irh_t *)&cmd->irh;

			if(setup->s.cksum_offset)
				irh->rlenssz = setup->s.cksum_offset;

			irh->opcode  = OCT_NW_PKT_OP;
			irh->param   = setup->s.ifidx;


		/* Offload checksum calculation for TCP/UDP packets */
		ndata.cmd.dptr =
		          octeon_map_single_buffer(get_octeon_device_id(priv->oct_dev),
		                          skb->data, skb->len, CAVIUM_PCI_DMA_TODEVICE);
		finfo->dptr     = ndata.cmd.dptr;
		ndata.buftype   = NORESP_BUFTYPE_NET;

	//这中情况下是多buffer, 要搞gather链
	else
		int                     i, frags;
		struct skb_frag_struct *frag;
		struct octnic_gather   *g;

		cavium_spin_lock(&priv->lock);
		//从链表头摘一个元素, 发送完毕会再申请一个glist, 见A11
		g = (struct octnic_gather *)cavium_list_delete_head(&priv->glist);
		cavium_spin_unlock(&priv->lock);

		cmdsetup.s.gather       = 1;
		cmdsetup.s.u.gatherptrs = (skb_shinfo(skb)->nr_frags + 1);
		octnet_prepare_pci_cmd(&(ndata.cmd), &cmdsetup); //见上面

		memset(g->sg, 0, g->sg_size);
		//建立gather list
		g->sg[0].ptr[0] =
		          octeon_map_single_buffer(get_octeon_device_id(priv->oct_dev),
 		                             skb->data, (skb->len - skb->data_len),
		                             CAVIUM_PCI_DMA_TODEVICE);
		CAVIUM_ADD_SG_SIZE(&(g->sg[0]), (skb->len - skb->data_len), 0);
		//一共有多少个frags
		frags = skb_shinfo(skb)->nr_frags;

		while(frags--) {
			frag = &skb_shinfo(skb)->frags[i-1];
			//建立剩下的gather list
			g->sg[(i >> 2)].ptr[(i&3)] =
					 octeon_map_page(get_octeon_device_id(priv->oct_dev), frag->page, frag->page_offset,
			                         frag->size,CAVIUM_PCI_DMA_TODEVICE);
			CAVIUM_ADD_SG_SIZE(&(g->sg[(i >> 2)]), frag->size, (i&3));
			i++;
		}
		//这个cmd的dptr是给octeon DMA的
		ndata.cmd.dptr =
		          octeon_map_single_buffer(get_octeon_device_id(priv->oct_dev),
		                           g->sg, g->sg_size, CAVIUM_PCI_DMA_TODEVICE);

		finfo->dptr      = ndata.cmd.dptr;
		finfo->g         = g;

		ndata.buftype   = NORESP_BUFTYPE_NET_SG;
	//发包, 重要!
	status = octnet_send_nic_data_pkt(priv->oct_dev, &ndata);
		//这个发送函数更底层, 直接就发送命令字, 不搞什么buffer; buffer在上面已经搞好了
		return octeon_send_noresponse_command(oct, ndata->q_no, 1, &ndata->cmd,
	                         ndata->buf, ndata->datasize, ndata->buftype);

	pndev->trans_start = jiffies;

	return OCT_NIC_TX_OK;
```
