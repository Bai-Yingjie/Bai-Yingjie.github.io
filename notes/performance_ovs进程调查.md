- [OVS进程调查](#ovs进程调查)
  - [pmap 查看地址空间](#pmap-查看地址空间)
  - [lsof 查看打开的文件](#lsof-查看打开的文件)
  - [strace 调查程序动态系统调用](#strace-调查程序动态系统调用)
  - [pstack 抓调用栈](#pstack-抓调用栈)
  - [perf top到hot spot](#perf-top到hot-spot)
  - [情况有变化](#情况有变化)
- [后记](#后记)

# OVS进程调查
10.64.16.21机器上，41320号进程一直100%CPU，这个线程是：
`ovs-vswitchd --pidfile --detach --log-file`
ovs-vswitchd是由systemd启动的一组线程
下面我想查看这个线程在干什么，导致cpu占用100%

## pmap 查看地址空间
```sh
$ sudo pmap -p 41246 -x
Address           Kbytes     RSS   Dirty Mode  Mapping
0000000000400000    1856    1856       0 r-x-- /usr/local/sbin/ovs-vswitchd
00000000005d0000     256     256     256 rw--- /usr/local/sbin/ovs-vswitchd
0000fff780000000  524288       0       0 rw-s- /dev/hugepages/rtemap_0
0000fff7a0000000  524288       0       0 rw-s- /dev/hugepages/rtemap_1
0000fff7c0000000  524288       0       0 rw-s- /dev/hugepages/rtemap_127
0000fff7e0000000  524288       0       0 rw-s- /dev/hugepages/rtemap_126
0000ffff983d0000      64       0       0 -w-s- /dev/infiniband/uverbs1
0000ffff983e0000      64       0       0 -w-s- /dev/infiniband/uverbs0
0000ffff998d0000      64      64      64 r---- /usr/lib64/libmlx5.so.1.3.16.0
0000ffff99b00000      64      64      64 r---- /usr/lib64/libc-2.17.so
0000ffff99f50000      64      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_vhost.so.4.1 (deleted)
0000ffff99f60000     128     128     128 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_vhost.so.4.1 (deleted)
0000ffff99f80000      64      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_timer.so.1.1 (deleted)
0000ffff99f90000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_timer.so.1.1 (deleted)
0000ffff99fa0000     128      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_table.so.3.1 (deleted)
0000ffff99fc0000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_table.so.3.1 (deleted)
0000ffff99fd0000      64      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_security.so.1.1 (deleted)
0000ffff99fe0000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_security.so.1.1 (deleted)
0000ffff99ff0000      64      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_sched.so.1.1 (deleted)
0000ffff9a000000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_sched.so.1.1 (deleted)
0000ffff9a010000      64      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_ring.so.1.1 (deleted)
0000ffff9a020000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_ring.so.1.1 (deleted)
0000ffff9a030000      64      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_reorder.so.1.1 (deleted)
0000ffff9a040000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_reorder.so.1.1 (deleted)
0000ffff9a050000      64      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_power.so.1.1 (deleted)
0000ffff9a060000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_power.so.1.1 (deleted)
0000ffff9a080000     128      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_port.so.3.1 (deleted)
0000ffff9a0a0000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_port.so.3.1 (deleted)
0000ffff9a0b0000      64      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_vmxnet3_uio.so.1.1 (deleted)
0000ffff9a0c0000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_vmxnet3_uio.so.1.1 (deleted)
0000ffff9a0d0000     128     128       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_virtio.so.1.1 (deleted)
0000ffff9a0f0000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_virtio.so.1.1 (deleted)
0000ffff9a100000      64      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_vhost.so.2.1 (deleted)
0000ffff9a110000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_vhost.so.2.1 (deleted)
0000ffff9a120000     128     128       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_thunderx_nicvf.so.1.1 (deleted)
0000ffff9a140000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_thunderx_nicvf.so.1.1 (deleted)
0000ffff9a150000      64      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_tap.so.1.1 (deleted)
0000ffff9a160000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_tap.so.1.1 (deleted)
0000ffff9a170000      64      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_sw_event.so.1.1 (deleted)
0000ffff9a180000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_sw_event.so.1.1 (deleted)
0000ffff9a190000      64      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_softnic.so.1.1 (deleted)
0000ffff9a1a0000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_softnic.so.1.1 (deleted)
0000ffff9a1b0000      64      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_skeleton_event.so.1.1 (deleted)
0000ffff9a1c0000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_skeleton_event.so.1.1 (deleted)
0000ffff9a1d0000      64      64       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_ring.so.2.1 (deleted)
0000ffff9a1e0000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_ring.so.2.1 (deleted)
0000ffff9a1f0000     320     128       0 r-x-- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_qede.so.1.1 (deleted)
0000ffff9a240000      64      64      64 rw--- /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_pmd_qede.so.1.1 (deleted)
```
注：mmap的file并不是把文件内容复制到内存里，而是把文件映射到进程的虚拟地址空间，进程对这块地址空间读写的时候，OS会以page为单位，
根据情况来选择真正读文件/写文件的时机，比如你写了一段数据，这段数据所在的page在后面的某个时刻，OS决定换出，那么它会被OS真正写入
这个文件，而这个写入的动作可能又经历了文件系统 -> page caching -> IO驱动 -> 物理硬件的复杂过程。

## lsof 查看打开的文件
```sh
$ sudo lsof -p 41246
ovs-vswit 41246 root  txt       REG                8,5  14113280 3527027728 /usr/local/sbin/ovs-vswitchd
ovs-vswit 41246 root  mem-R     REG               0,40 536870912     968169 /dev/hugepages/rtemap_0
ovs-vswit 41246 root  mem-R     REG               0,40 536870912     968170 /dev/hugepages/rtemap_1
ovs-vswit 41246 root  mem-R     REG               0,40 536870912     968296 /dev/hugepages/rtemap_127
ovs-vswit 41246 root  mem-R     REG               0,40 536870912     968295 /dev/hugepages/rtemap_126
ovs-vswit 41246 root  mem       CHR            231,193                27211 /dev/infiniband/uverbs1
ovs-vswit 41246 root  mem       CHR            231,192                27210 /dev/infiniband/uverbs0
ovs-vswit 41246 root  mem-w     REG               0,23    208420     789242 /run/.rte_config
ovs-vswit 41246 root    3w      REG                8,5    283688 3527027762 /usr/local/var/log/openvswitch/ovs-vswitchd.log
ovs-vswit 41246 root   15u      CHR             10,196       0t0       2155 /dev/vfio/vfio
ovs-vswit 41246 root   12u  netlink                          0t0     968157 ROUTE
ovs-vswit 41246 root   21u  a_inode               0,13         0       9112 [eventpoll]
ovs-vswit 41246 root   36u      CHR             10,200       0t0       2153 /dev/net/tun
ovs-vswit 41246 root   47u     unix 0xffff8017d901f300       0t0     946033 /usr/local/var/run/openvswitch/ovs-br0.snoop
ovs-vswit 41246 root   51u     unix 0xffff8017d6540000       0t0     953826 /usr/local/var/run/openvswitch/ovs-br0.mgmt
ovs-vswit 41246 root   53u     unix 0xffff8017d6540d80       0t0     953887 /usr/local/var/run/openvswitch/vhost-user0
ovs-vswit 41246 root   58u     unix 0xffff8017d901f780       0t0     946094 /usr/local/var/run/openvswitch/vhost-user1
ovs-vswit 41246 root   59u     unix 0xffff8017d6ee9480       0t0     953199 /usr/local/var/run/openvswitch/vhost-user2
ovs-vswit 41246 root   60u     unix 0xffff8017d6ee5e80       0t0     953211 /usr/local/var/run/openvswitch/vhost-user3
```
注：
* 打开的有普通文件，内核设备，管道，unix socket，netlink socket，
* 这里看到有个log文件，这是调查的好入口

## strace 调查程序动态系统调用
```
$ sudo strace -p 41246
strace: Process 41246 attached
ppoll([{fd=11, events=POLLIN}, {fd=49, events=POLLIN}, {fd=10, events=POLLIN}, {fd=9, events=POLLIN}, {fd=12, events=POLLIN}, {fd=51, events=POLLIN}, {fd=36, events=POLLIN}, {fd=20, events=POLLIN}, {fd=7, eve
nts=POLLIN}, {fd=30, events=POLLIN}, {fd=47, events=POLLIN}], 11, {3, 310381239}, NULL, 0) = 1 ([{fd=30, revents=POLLIN}], left {3, 247386583})
getrusage(0x1 /* RUSAGE_??? */, {ru_utime={41, 121831}, ru_stime={45, 446739}, ...}) = 0
read(30, "\0", 512)                     = 1
recvfrom(11, 0x2409328, 264, 0, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
recvmsg(10, 0xffffee67ea18, MSG_DONTWAIT) = -1 EAGAIN (Resource temporarily unavailable)
read(36, 0x24fbe36, 1518)               = -1 EAGAIN (Resource temporarily unavailable)
read(49, 0x24fbe36, 1518)               = -1 EAGAIN (Resource temporarily unavailable)
accept(51, 0xffffee68f7d0, 0xffffee68f7cc) = -1 EAGAIN (Resource temporarily unavailable)
accept(47, 0xffffee68f7d0, 0xffffee68f7cc) = -1 EAGAIN (Resource temporarily unavailable)
accept(9, 0xffffee68fac0, 0xffffee68fabc) = -1 EAGAIN (Resource temporarily unavailable)
recvmsg(20, 0xffffee67ea68, MSG_DONTWAIT) = -1 EAGAIN (Resource temporarily unavailable)
recvmsg(12, 0xffffee67ea58, MSG_DONTWAIT) = -1 EAGAIN (Resource temporarily unavailable)
recvmsg(12, 0xffffee67ea58, MSG_DONTWAIT) = -1 EAGAIN (Resource temporarily unavailable)
recvmsg(12, 0xffffee67ea58, MSG_DONTWAIT) = -1 EAGAIN (Resource temporarily unavailable)
ppoll([{fd=11, events=POLLIN}, {fd=49, events=POLLIN}, {fd=10, events=POLLIN}, {fd=9, events=POLLIN}, {fd=12, events=POLLIN}, {fd=51, events=POLLIN}, {fd=36, events=POLLIN}, {fd=20, events=POLLIN}, {fd=7, eve
nts=POLLIN}, {fd=30, events=POLLIN}, {fd=47, events=POLLIN}], 11, {3, 247000000}, NULL, 0) = 1 ([{fd=30, revents=POLLIN}], left {2, 746806149})
getrusage(0x1 /* RUSAGE_??? */, {ru_utime={41, 122010}, ru_stime={45, 446739}, ...}) = 0
read(30, "\0", 512)                     = 1
recvfrom(11, 0x2409328, 264, 0, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
recvmsg(10, 0xffffee67ea18, MSG_DONTWAIT) = -1 EAGAIN (Resource temporarily unavailable)
read(36, 0x24fbe36, 1518)               = -1 EAGAIN (Resource temporarily unavailable)
read(49, 0x24fbe36, 1518)               = -1 EAGAIN (Resource temporarily unavailable)
accept(51, 0xffffee68f7d0, 0xffffee68f7cc) = -1 EAGAIN (Resource temporarily unavailable)
accept(47, 0xffffee68f7d0, 0xffffee68f7cc) = -1 EAGAIN (Resource temporarily unavailable)
accept(9, 0xffffee68fac0, 0xffffee68fabc) = -1 EAGAIN (Resource temporarily unavailable)
recvmsg(20, 0xffffee67ea68, MSG_DONTWAIT) = -1 EAGAIN (Resource temporarily unavailable)
recvmsg(12, 0xffffee67ea58, MSG_DONTWAIT) = -1 EAGAIN (Resource temporarily unavailable)
recvmsg(12, 0xffffee67ea58, MSG_DONTWAIT) = -1 EAGAIN (Resource temporarily unavailable)
recvmsg(12, 0xffffee67ea58, MSG_DONTWAIT) = -1 EAGAIN (Resource temporarily unavailable)
ppoll([{fd=11, events=POLLIN}, {fd=49, events=POLLIN}, {fd=10, events=POLLIN}, {fd=9, events=POLLIN}, {fd=12, events=POLLIN}, {fd=51, events=POLLIN}, {fd=36, events=POLLIN}, {fd=20, events=POLLIN}, {fd=7, eve
nts=POLLIN}, {fd=30, events=POLLIN}, {fd=47, events=POLLIN}], 11, {2, 746000000}, NULL, 0) = 1 ([{fd=30, revents=POLLIN}], left {2, 246780821})
getrusage(0x1 /* RUSAGE_??? */, {ru_utime={41, 122186}, ru_stime={45, 446739}, ...}) = 0
read(30, "\0", 512)                     = 1
recvfrom(11, 0x2409328, 264, 0, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
recvmsg(10, 0xffffee67ea18, MSG_DONTWAIT) = -1 EAGAIN (Resource temporarily unavailable)
read(36, 0x24fbe36, 1518)               = -1 EAGAIN (Resource temporarily unavailable)
read(49, 0x24fbe36, 1518)               = -1 EAGAIN (Resource temporarily unavailable)
accept(51, 0xffffee68f7d0, 0xffffee68f7cc) = -1 EAGAIN (Resource temporarily unavailable)
accept(47, 0xffffee68f7d0, 0xffffee68f7cc) = -1 EAGAIN (Resource temporarily unavailable)
accept(9, 0xffffee68fac0, 0xffffee68fabc) = -1 EAGAIN (Resource temporarily unavailable)
```
注：
* ppoll和pselect差不多，所谓pselect就是不被信号打断的select
* read recvfrom recvmsg都差不多，后两者多了些控制flag
* 这里不正常的地方在于，ppoll返回的fd，按说都应该能有数据，但是后面的read/recv等函数都读不到东西。--这就是100%的原因？
  似乎不对，应该是用户态收发包才对呀？

## pstack 抓调用栈
```
$ sudo pstack 41320
#0  0x000000000053e91c in netdev_dpdk_vhost_rxq_recv (rxq=<optimized out>, batch=<optimized out>) at lib/netdev-dpdk.c:1943
#1  0x0000000000491d98 in netdev_rxq_recv (rx=<optimized out>, batch=0xffff767fe2c0, batch@entry=0xffff767fe300) at lib/netdev.c:701
#2  0x000000000046dd0c in dp_netdev_process_rxq_port (pmd=pmd@entry=0xffff94180010, rxq=0x24f4620, port_no=4) at lib/dpif-netdev.c:3279
#3  0x000000000046e0a8 in pmd_thread_main (f_=0xffff94180010) at lib/dpif-netdev.c:4146
#4  0x00000000004e452c in ovsthread_wrapper (aux_=<optimized out>) at lib/ovs-thread.c:348
#5  0x0000ffff99c47bb0 in start_thread () from /lib64/libpthread.so.0
#6  0x0000ffff99a6b4c0 in thread_start () from /lib64/libc.so.6
```
注：没看出啥来...

## perf top到hot spot
```
$ sudo perf top
Samples: 360K of event 'cycles:ppp', Event count (approx.): 42373494756
Overhead  Shared Object            Symbol
  40.02%  ovs-vswitchd             [.] dp_netdev_process_rxq_port
  25.65%  [vdso]                   [.] monotonic
   9.31%  ovs-vswitchd             [.] netdev_dpdk_vhost_rxq_recv
   6.33%  ovs-vswitchd             [.] pmd_thread_main
   4.65%  ovs-vswitchd             [.] netdev_rxq_recv
   3.47%  ovs-vswitchd             [.] time_timespec__
   2.21%  ovs-vswitchd             [.] time_usec
   1.67%  libpthread-2.17.so       [.] __pthread_once
   1.65%  libc-2.17.so             [.] __clock_gettime
   1.15%  [vdso]                   [.] __kernel_clock_gettime
```
```sh
#9382是这个线程的pid
#进程可能重启了，变成了9382
$ sudo perf top -p 9382
Samples: 193K of event 'cycles:ppp', Event count (approx.): 38607322931
Overhead  Shared Object        Symbol
  55.02%  ovs-vswitchd         [.] dp_netdev_process_rxq_port
  21.43%  librte_vhost.so.4.1  [.] rte_vhost_dequeue_burst
  10.78%  ovs-vswitchd         [.] netdev_dpdk_vhost_rxq_recv
   3.43%  ovs-vswitchd         [.] pmd_thread_main
   2.92%  librte_vhost.so.4.1  [.] get_device
   2.82%  [vdso]               [.] monotonic
   1.94%  ovs-vswitchd         [.] netdev_rxq_recv
   0.41%  ovs-vswitchd         [.] rte_vhost_dequeue_burst@plt
   0.31%  ovs-vswitchd         [.] time_timespec__
   0.26%  ovs-vswitchd         [.] time_usec
   0.21%  libc-2.17.so         [.] __clock_gettime
   0.12%  libpthread-2.17.so   [.] __pthread_once
   0.11%  [vdso]               [.] __kernel_clock_gettime
   0.03%  ovs-vswitchd         [.] clock_gettime@plt
   0.02%  ovs-vswitchd         [.] pthread_once@plt
   0.02%  [kernel]             [k] _raw_spin_unlock_irqrestore
   0.01%  libpthread-2.17.so   [.] pthread_mutex_unlock
   0.01%  [kernel]             [k] sys_futex
   0.01%  [kernel]             [k] futex_wake
   0.01%  libc-2.17.so         [.] _int_free
   0.01%  ovs-vswitchd         [.] single_threaded
   0.01%  libc-2.17.so         [.] __poll
```
注：以上两个，函数dp_netdev_process_rxq_port都出现在第一位
看一下汇编如下：
```sh
Percent│       sub    x2, x2, x3   
       │       add    x2, x2, x1    
       │       str    x2, [x0]  
       │ cc:   ldr    x2, [x29,#88]     
       │       str    x0, [x19,#520]        
       │       add    x0, x20, #0x20 
       │       sub    x1, x1, x2 
       │     → bl     non_atomic_ullong_add
       │       ldr    w0, [x19,#312]
       │     ↓ cbnz   w0, 1d0
  0.19 │ e8:   str    xzr, [x19,#248]  
       │       mov    w0, w23  
       │       ldp    x19, x20, [sp,#16]
  0.72 │       ldp    x21, x22, [sp,#32]   
  0.08 │       ldp    x23, x24, [sp,#48]  
       │       ldr    x25, [sp,#64]     
  0.77 │       ldp    x29, x30, [sp],#384   
       │     ← ret      
  0.09 │108:   ldr    x0, [x23,#16] 
       │       cmp    x0, x22    
       │     ↓ b.ne   1b0       
 39.26 │       mrs    x2, cntvct_el0  
  0.08 │       ldr    x0, [x29,#104]   
  3.68 │       str    x2, [x19,#512]     
       │     ↓ cbz    x0, 134       
       │       ldp    x1, x3, [x0]   
       │       sub    x1, x1, x3    
       │       add    x1, x1, x2 
       │       str    x1, [x0]    
  0.14 │134:   str    x0, [x19,#520]  
  7.96 │       cmp    w21, #0x5f 
       │       mov    w23, #0x0                       // #0  
       │       ccmp   w21, #0xb, #0x4, ne   
 36.43 │     ↑ b.eq   e8     
       │       adrp   x22, 5dc000 <rl.7549+0x20>     
       │       add    x22, x22, #0x138    
       │       ldr    w0, [x22,#36]     
       │       cmp    w0, #0x1  
       │     ↑ b.ls   e8    
       │       ldr    x0, [x20,#8]  
       │     → bl     netdev_rxq_get_name   
       │       mov    x20, x0   
       │       mov    w0, w21    
       │     → bl     ovs_strerror 
       │       mov    x4, x20    
       │       mov    x5, x0                  
       │       add    x2, x22, #0x160         
       │       mov    x0, x22            
       │       mov    w1, #0x2                        // #2       
```
注意以下两条汇编占用很高
mrs    x2, cntvct_el0
b.eq   e8
根据ARM手册， cntvct_el0是个64bit的Virtual Timer Count register，只读的  
可参考https://patchwork.kernel.org/patch/9290801/
```sh
$ sudo perf stat -e cycles,stalled-cycles-frontend,stalled-cycles-backend,branch-misses,cache-references,cache-misses -p 9382
^C
 Performance counter stats for process id '9382':
     9,670,958,313      cycles
       324,264,597      stalled-cycles-frontend   #    3.35% frontend cycles idle
     6,355,584,509      stalled-cycles-backend    #   65.72% backend cycles idle
         5,210,609      branch-misses
     4,424,834,814      cache-references
        48,121,592      cache-misses              #    1.088 % of all cache refs
       3.867804140 seconds time elapsed
```
这里stalled-cycles-backend很高，可能是data cache问题。复习一下data cache：
* cache有两种写策略：
	* write through： 写透 写cache，同时更新下个level的mem。
	* write back： 写回 暂时不更新到下个level的mem，等到这个block被换出时一起更新。这样复杂点，但性能高
* 写也有cache miss，此时需要allocate一个cache line。
  有人问写cache miss的时候，要先allocate cacheline，那么还要先把数据读到这个cache line里吗？
  答：需要。正因为cache的操作单位是cache line，比如64字节，但通常写个data，不会把这个64字节都更新，
  那么就需要先读出这64字节，更新其中的一部分，再一起写回。

## 情况有变化
ovs-vswitchd的所有线程都100%，昨天还只有一个线程是...
* strace出来大部分都是futex系统调用
* ifconfig也没看到有流量，用`ip -s addr`
```
$ sudo pstack 27680
Thread 1 (process 27680):
#0  0x0000ffffa29a048c in monotonic () at arch/arm64/kernel/vdso/gettimeofday.S:241
#1  0x0000ffffa15eeb28 in clock_gettime () from /lib64/libc.so.6
#2  0x000000000050f99c in xclock_gettime (ts=0xffff7969e3c0, id=<optimized out>) at lib/timeval.c:503
#3  time_timespec__ (c=0x690e70 <monotonic_clock>, ts=0xffff7969e3c0) at lib/timeval.c:155
#4  0x000000000050fbf4 in time_usec__ (c=0x690e70 <monotonic_clock>) at lib/timeval.c:246
#5  time_usec () at lib/timeval.c:247
#6  0x000000000046e1dc in pmd_thread_ctx_time_update (pmd=0xffff796a0010) at lib/dpif-netdev.c:777
#7  pmd_thread_main (f_=0xffff796a0010) at lib/dpif-netdev.c:4156
#8  0x00000000004e452c in ovsthread_wrapper (aux_=<optimized out>) at lib/ovs-thread.c:348
#9  0x0000ffffa17b7bb0 in start_thread () from /lib64/libpthread.so.0
#10 0x0000ffffa15db4c0 in thread_start () from /lib64/libc.so.6
bai@CentOS-21 ~/tmp
$ sudo pstack 27680
Thread 1 (process 27680):
#0  0x0000000000491d90 in netdev_rxq_recv (rx=0xfff7ed43a100, batch=0xffff7969e2c0, batch@entry=0xffff7969e300) at lib/netdev.c:701
#1  0x000000000046dd0c in dp_netdev_process_rxq_port (pmd=pmd@entry=0xffff796a0010, rxq=0x1ea85bd0, port_no=4) at lib/dpif-netdev.c:3279
#2  0x000000000046e0a8 in pmd_thread_main (f_=0xffff796a0010) at lib/dpif-netdev.c:4146
#3  0x00000000004e452c in ovsthread_wrapper (aux_=<optimized out>) at lib/ovs-thread.c:348
#4  0x0000ffffa17b7bb0 in start_thread () from /lib64/libpthread.so.0
#5  0x0000ffffa15db4c0 in thread_start () from /lib64/libc.so.6
bai@CentOS-21 ~/tmp
$ sudo pstack 27680
Thread 1 (process 27680):
#0  0x0000ffffa1ad5420 in get_device () from /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_vhost.so.4.1
#1  0x0000ffffa1adab60 in rte_vhost_dequeue_burst () from /usr/src/dpdk-stable-17.11.3/arm64-armv8a-linuxapp-gcc/lib/librte_vhost.so.4.1
#2  0x000000000053e740 in netdev_dpdk_vhost_rxq_recv (rxq=<optimized out>, batch=0xffff7969e2c0) at lib/netdev-dpdk.c:1918
#3  0x0000000000491d98 in netdev_rxq_recv (rx=<optimized out>, batch=0xffff7969e2c0, batch@entry=0xffff7969e300) at lib/netdev.c:701
#4  0x000000000046dd0c in dp_netdev_process_rxq_port (pmd=pmd@entry=0xffff796a0010, rxq=0x1eb3c0e0, port_no=5) at lib/dpif-netdev.c:3279
#5  0x000000000046e0a8 in pmd_thread_main (f_=0xffff796a0010) at lib/dpif-netdev.c:4146
#6  0x00000000004e452c in ovsthread_wrapper (aux_=<optimized out>) at lib/ovs-thread.c:348
#7  0x0000ffffa17b7bb0 in start_thread () from /lib64/libpthread.so.0
#8  0x0000ffffa15db4c0 in thread_start () from /lib64/libc.so.6
```

# 后记
后来了解了一下ovs dpdk才知道, dpdk的pmd线程因为使用了轮询模式, 就是100% CPU占用的,.