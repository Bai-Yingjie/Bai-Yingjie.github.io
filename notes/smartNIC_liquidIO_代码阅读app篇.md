- [编译](#编译)
- [用户态使用](#用户态使用)
  - [load](#load)
- [代码梳理](#代码梳理)
  - [共享内存里面保存的结构体](#共享内存里面保存的结构体)
  - [每个进程一个的test_stats](#每个进程一个的test_stats)
  - [处理SIGINT的handler](#处理sigint的handler)
  - [主函数流程](#主函数流程)
  - [重要结构体](#重要结构体)
  - [这个测试程序的默认配置](#这个测试程序的默认配置)
  - [子进程](#子进程)

# 编译
* host
```
cd OCTEON-SDK/components/driver
make
```
* core
```
cd OCTEON-SDK/applications/pci-core-app/base
make
```
* test
```
cd OCTEON-SDK-2.3.0/components/driver/host/test
make
```

# 用户态使用

## load
```
cd /home/yingjie/repos/OCTEON-SDK-2.3.0/target/bin
OCTEON_REMOTE_DEBUG=1 oct-pci-boot u-boot-octeon_nic10e_66.bin
OCTEON_REMOTE_DEBUG=1 oct-pci-load 0 ../../components/driver/bin/cvmcs.strip
OCTEON_REMOTE_DEBUG=1 oct-pci-bootcmd "bootoct 0 coremask=f"

cd /home/yingjie/repos/OCTEON-SDK-2.3.0/components/driver/bin
insmod octeon_drv.ko
mknod /dev/octeon_device c 127 0
./oct_req 0 -ubsy
```

对应代码`OCTEON-SDK/components/driver/host/test/oct_req.c`
```
[root@cvmx bin]# ./oct_req 0 -ubsy
Octeon Test utility version: PCI BASE RELEASE 2.3.0 build 84


Starting operation in silent mode with
Octeon id: 0
  response order =  UNORDERED (1)
  response mode  =  BLOCKING (0)
  dma mode       =  DIRECT (0)
  Max in bufs  = 1
  Max out bufs  = 1
  Inbuf size: 1024  Outbuf size: 1024
Shared memory id is 851990
 --- Press Ctrl-C to stop the test ---

Main thread pid: 20804
Thread (index: 0 pid: 20805) starting execution
Request Failed with status 4: 0x105d4a--C-

Test Thread 0 (pid: 20805)stopping now....

Main thread stopping... 

Test completed: 19770 requests sent.
Verification: 19769 passed 0 failed 
Tested with Input buffers [buffer count/requests sent]
[ 1/19770 ] 
Tested with Output buffers [buffer count/requests sent]
[ 1/19770 ] 
Max data sent: 1024 bytes
Max data received: 1024 bytes
```

# 代码梳理
## 共享内存里面保存的结构体
```c
struct test {
	volatile int            ok_to_send;
	pid_t                   main_pid;
	OCTEON_RESPONSE_ORDER   resp_order;
	OCTEON_RESPONSE_MODE    resp_mode;
	OCTEON_DMA_MODE         dma_mode;
	//每个子进程都有一个test_stats	
	struct test_stats       perthread[TEST_THREAD_COUNT];
	struct test_stats       total;
	time_t                  start, end;
};
```

## 每个进程一个的test_stats
```c
struct test_stats {
	void                  *sh_mem;
	struct request_list   *nbreqs;
	pid_t                  pid;
	volatile int           running;
	volatile int           reqs_pending;
	unsigned long          incntfreq[MAX_INBUFS+1];
	unsigned long          outcntfreq[MAX_OUTBUFS+1];
	unsigned long          maxdatasent;
	unsigned long          maxdatareceived;
	unsigned long          request_count;
	unsigned long          verify_failed;
	unsigned long          verify_passed;
};
```

## 处理SIGINT的handler
注意SIGINT会向前台进程组里面的所有进程发signal  
所以这个函数里面会判断只有主进程才干活  
在这里是置一个标志, 各个子进程会把当前的活干完, 才根据这个标志退出
```c
void signal_handler(int x)
	pid_t   my_pid = getpid();

	/* Just clear the ok_to_send flag. When the signal handler returns in
	   the main process, it will wait for the children to complete its
	   processing.
	*/
	if(t_main && (my_pid == t_main->main_pid) ) {
		t_main->ok_to_send = 0;
		return;
	}
}
```

## 主函数流程
```c
main
  octeon_initialize
    //打开字符设备
    oct_dev_handle = open("/dev/octeon_device", 0)
  print_test_setup
    //打印req的信息
  //共享内存, 用于多进程
  shmid = shmget(0, sizeof(struct test), IPC_CREAT | IPC_EXCL);
  //attach共享内存, 得到地址
  t_main = (struct test *)shmat(shmid, NULL, 0);
  //安装signal处理SIGINT
  prev_sig_handler = signal(signal_to_catch, signal_handler);
  //起多线程 
  for TEST_THREAD_COUNT 个:
    pid = fork()
    //子进程
    case 0:
      oct_request_thread(q_no, count, i);
    //fork失败
    case -1:
      //对已经成功fork的子进程kill, 发SIGINT
      kill(t_main->perthread[i].pid, signal_to_catch);
    //父进程
      //记录子进程的pid
      t_main->perthread[i].pid = pid
  //开始
  t_main->ok_to_send = 1;
  t_main->main_pid = getpid();
  //等待子进程完工
  wait_for_thread_completion()
    //在循环里sleep(1)等待所有子进程的t->perthread[i].running全为0
    //为什么不用waitpid呢?
  //统计并打印信息
  add_thread_stats(t_main);
  print_test_stats(t_main);
  //detach共享内存
  shmdt(t_main);
  //恢复signal
  signal(signal_to_catch, prev_sig_handler );
```

## 重要结构体
`components/driver/host/include/cavium_defs.h`
```c
//这个共用体的好处是不用强转地址了
/** Use this type to pass buffer address to the driver in ioctls. Use the 
	addr field to copy your buffer's address. */
typedef  union {
	uint64_t   addr64;
	uint8_t   *addr;
} cavium_ptr_t;
```
```c
//这里面MAX_BUFCNT是16
/**
  Structure for passing input and output buffers in the request structure.
*/
typedef struct {
  /** number of buffers */
   uint32_t        cnt;
   uint32_t        rsvd;
  /** buffer pointers */
   cavium_ptr_t   ptr[MAX_BUFCNT];
  /** their data sizes*/
   uint32_t       size[MAX_BUFCNT];
} octeon_buffer_t;
```
```c
/** Information about the request sent to driver. This structure
 *  points to the input data buffer(s), to the output buffer(s) (if any)
 *  if a response is expected. It also keep information about the type of
 *  DMA, mode of operation (response order, mode etc).
 *  Several MACROS are defined to help access the fields.
 */
typedef struct  {
  /** The input buffers and their sizes. */
    octeon_buffer_t             inbuf;
  /** The output buffer pointers and the size allocated at each pointer. */
    octeon_buffer_t             outbuf;
  /** The instruction header to be sent with this request to Octeon. */
  //和手册里面input ring里面DPI_INST_HDR一致
    octeon_instr_ih_t           ih;
  /** The Input Request Header to be sent with the request to Octeon. */
    octeon_instr_irh_t          irh;
  /** The extra headers (upto 4 64-bit words) for a 64-bytes instruction. */
    uint64_t                    exhdr[4];
  /** Information about the formatting to be done to each extra header. */
    octeon_exhdr_info_t         exhdr_info;
  /** Additional information required for processing this request. Also
      driver returns an id identifying the request and the current status
      of the request in its fields.*/
    union {
         uint64_t               addr64;
         octeon_request_info_t *ptr;
    } req_info;
} octeon_soft_request_t;
```

## 这个测试程序的默认配置
```c
//以下决定了往哪个dev上发
#define OCTEON_ID    0
#define REQ_IQ_NO    0
//一共用几个进程
#define TEST_THREAD_COUNT  1
//buffer数
#define MAX_INBUFS 1
#define MAX_OUTBUFS 1
//buffer大小
#define INBUF_SIZE    (1 * 1024)
#define OUTBUF_SIZE   (1 * 1024)
#define REQUEST_TIMEOUT   500
#define MAX_NB_REQUESTS   256

/*DMA可以是以下几种:
**OCTEON_DMA_DIRECT; OCTEON_DMA_GATHER; 
**OCTEON_DMA_SCATTER; OCTEON_DMA_SCATTER_GATHER.
*/
OCTEON_DMA_MODE        TEST_DMA_MODE  =   OCTEON_DMA_DIRECT;

//以下模式可以通过命令行传入

/*response的oder类型, 可以是:
**OCTEON_RESP_NORESPONSE; OCTEON_RESP_ORDERED; OCTEON_RESP_UNORDERED;
**用户态app不支持ORDERED模式
*/
OCTEON_RESPONSE_ORDER   TEST_RESP_ORDER  =  OCTEON_RESP_NORESPONSE;

/*response的阻塞模式, 可以是:
**OCTEON_RESP_NON_BLOCKING; OCTEON_RESP_BLOCKING;
*/
OCTEON_RESPONSE_ORDER   TEST_RESP_MODE  =  OCTEON_RESP_NON_BLOCKING;
```

## 子进程
```c
oct_request_thread(int  q_no, int count, int  tidx)
  //注: google以后发现子进程应该会继承shmat的segment, 但这里为什么又自己attach一遍呢?
  t = (struct test *)shmat(shmid, NULL, 0)
  s = (struct test_stats *)&(t->perthread[tidx]);
  s->sh_mem  = t;
  /*nbregs是个数组, 元素是request_list 
  struct request_list {
	octeon_soft_request_t  *sr;
	int                     status;
	uint32_t                outsize;
	uint32_t                verify_size;
  };
  */
  //MAX_NB_REQUESTS是256
  s->nbreqs = malloc(sizeof(struct request_list) * MAX_NB_REQUESTS)
  //等着主进程发开始
  do { sleep(1); } while(!t->ok_to_send);
  time(&t1);
  srandom(t1);
  /*主体循环, 条件是上次发送成功&&t_main->ok_to_send = 1
  **这个循环没有什么sleep操作, 全速运转
  */
  /*response的order类型, 好像用户态不支持ORDERED
    OCTEON_RESP_ORDERED=0,
    OCTEON_RESP_UNORDERED=1,
    OCTEON_RESP_NORESPONSE=2
  response的block类型
    OCTEON_RESP_BLOCKING=0,
    OCTEON_RESP_NON_BLOCKING
  */
  //所以req分几种:
    //不需要response的应该最简单, 这里传入的tag是固定的0x101011
    req_status = noresponse_request(q_no, tag, s, t->dma_mode);
      //随机生成incnt outcnt insize outsize
      generate_data_sizes()
      //根据DMA mode, 创建no response, 非阻塞的请求
      soft_req = create_soft_request()
        //注意: 这里第二个malloc写错了!
        //给以下两个结构malloc空间
        octeon_soft_request_t   *soft_req=malloc()
        octeon_request_info_t   *req_info=malloc()

        //raw为0则其后面的位会被忽略
        soft_req->ih.raw     = 1;
        soft_req->ih.qos     = 0;
        soft_req->ih.grp     = 0;
        soft_req->ih.rs      = 0;
        soft_req->ih.tagtype = 1;
        soft_req->ih.tag = tag;
        //DMA模式里面有gather则置1
        soft_req->ih.gather  = 1;
        //irh是instruction response header
        soft_req->irh.opcode = CVMCS_REQRESP_OP;
        soft_req->irh.param  = 0x10;
        soft_req->irh.dport  = 32;
        //根据DMA置1
        soft_req->irh.scatter  = 1;
        //这个req_info是给谁看的?
        req_info->octeon_id  = 0;
        req_info->request_id = 0xff;
        req_info->req_mask.dma_mode   = dma_mode;
        req_info->req_mask.resp_mode  = resp_mode;
        req_info->req_mask.resp_order = resp_order;
        req_info->req_mask.iq_no = q_no;
        req_info->timeout             =  REQUEST_TIMEOUT;

        //这里要申请(malloc)真正的inbuf outbuf, 因为上面的结构体里面只有buffer指针和大小
        set_buffers(soft_req, inbuf_cnt, outbuf_cnt)
        //这里的status 3是什么意思?
        SOFT_REQ_INFO(soft_req)->status = 3;
        return soft_req;

      req_status = send_request(oct_id, s, soft_req);
        /*octeon*是用户态下的api, 基本上都是对ioctl的封装
        **这部分代码在components/driver/host/api/octeon_user.c
        */
        retval = octeon_send_request(oct_id, sr);
          //raw模式下不支持ORDERED模式
          //非raw模式不支持response
          ioctl(oct_dev_handle, IOCTL_OCTEON_SEND_REQUEST, soft_req)
        //发送成功则各种count++
      //最后free这个soft_req
      free_soft_request(soft_req);

    /*上面详细看了noresponse的发送
    **剩下是unordered阻塞, 这里采用轮询的策略, 调octeon_query_request()来查询
    */
    req_status = unordered_blocking_request(q_no, tag, s, t->dma_mode);
      //也是调这两个函数, 但阻塞在ioctl里面?
      create_soft_request()
      send_request()
      //因为是阻塞方式, 现在就可以free这个soft_req了
      free_soft_request(soft_req);
    /*再剩下是unordered非阻塞
    **这种情况下, 要看nbreqs[nbidx].status
    **可以有三种:REQ_NONE  REQ_PEND  REQ_DONE
    */
    switch nbreqs[nbidx].status: //这里的status是下面置的
      //发出去的报文有回应, 但还没有处理
      case REQ_PEND:
        r = check_req_status(nbreqs[nbidx].sr);
          //调用api
          octeon_query_request()
        /*如果此时r为OCTEON_REQUEST_PENDING
        **说明respond还在queue里, 那本次不处理
        */
        /*其他情况要么respond done, 要么出错了*/
        if r == OCTEON_REQUEST_DONE
          verify_output()
        //释放buffer, 传入octeon_soft_request_t
        free_soft_request(nbreqs[nbidx].sr);
          //释放input buffer, output buffer和这个soft_req本身
        nbreqs[nbidx].status  = REQ_NONE;
      case REQ_NONE:
        /*上面看了阻塞式的带response的发送
        **这个是非阻塞有response的发送, 入参多了一个参数struct request_list *nb
        **还记得吗, struct request_list *nb是个数组, 里面有256项
        **上面的case说的是已经发完了请求
        **这里的case是说要开始发送
        */
        req_status = unordered_nonblocking_request(q_no, tag,s,&nbreqs[nbidx], t->dma_mode);
          //同样是
          create_soft_request()
          send_request()
          //因为是非阻塞, 所以不能在这里free, 因此在这里做REQ_PEND标记, 在上面的case用
          nb->status      = REQ_PEND;
          nb->outsize     = outsize;
          nb->verify_size = verify_size;
          nb->sr          = soft_req;

      if(nbreqs[nbidx].status == REQ_PEND)
        nbidx++;//简化, 实际会处理回环
  //主循环结束,如果还有pending
  if(s->reqs_pending)
    wait_for_unordered_requests(s); //清理每个s->nbreqs[i]
    free(s->nbreqs);
  //dettach共享内存
  shmdt(t);
```
