- [查看调用栈](#查看调用栈)
  - [内核态](#内核态)
  - [用户态](#用户态)
- [ping收到杂包问题](#ping收到杂包问题)
  - [现象](#现象)
  - [ping的基本流程](#ping的基本流程)
  - [ping都收到了什么报文?](#ping都收到了什么报文)
  - [为什么会收到127.0.0.1的ICMP报文?](#为什么会收到127001的icmp报文)
  - [结论](#结论)

# 查看调用栈
当板子上程序跑死的时候,看调用栈
## 内核态
```
/proc/1080 # cat stack
[<ffffffffc009ced8>] do_signal_stop+0x140/0x200
[<ffffffffc009e1d8>] get_signal+0x270/0x5d8
[<ffffffffc006fcec>] do_signal+0x24/0x210
[<ffffffffc0070988>] do_notify_resume+0x58/0x70
[<ffffffffc006b920>] work_notifysig+0x10/0x18
```

## 用户态
使用ptrace库和libunwind库
```c
#include <sys/ptrace.h>
#include <sys/user.h>
#include <libunwind.h>
#include <libunwind-ptrace.h>
```
流程
```c
//unwind创建space
unw_create_addr_space(&_UPT_accessors, 0);
//用ptrace attach到目标tid上
ptrace(PTRACE_ATTACH, tracee_tid, 0, 0)
//等待pid
ret_pid = waitpid(tracee_tid, &status, __WALL);
//用libunwind解析栈
unw_init_remote(&cursor, as, ui);
unw_get_reg(&cursor, UNW_REG_IP, &ip)
unw_step(&cursor)
//打印栈

//销毁space
unw_destroy_addr_space(as);
```

用户态也可以用pstack, 但需要板子上有gdb.

# ping收到杂包问题
## 现象
板子的一个脚本里面, 一直运行一个ping命令: 
`ping 169.254.1.253 -W 1 -c 3600 -p 42 -s 16`
超时1秒, 3600次, 填充0x42, 报文size 16
不打流时, ping进程的CPU占用非常低, 几乎没有; 但打流的时候, ping进程的CPU占用高到10%以上.

## ping的基本流程
用strace看ping的流程, 基本上
1. 先用`socket(AF_INET, SOCK_RAW, IPPROTO_ICMP) = 3`创建socket
2. ping会查dns, 方法是
```c
socket(AF_INET, SOCK_DGRAM, IPPROTO_IP) = 5
connect(5, {sa_family=AF_INET, sin_port=htons(1025), sin_addr=inet_addr("169.254.1.253")}, 16) = 0
getsockname(5, {sa_family=AF_INET, sin_port=htons(37650), sin_addr=inet_addr("192.168.0.14")}, [16]) = 0
```
3. 设置socket属性
```c
setsockopt(3, SOL_RAW, ICMP_FILTER, ~(1<<ICMP_ECHOREPLY|1<<ICMP_DEST_UNREACH|1<<ICMP_SOURCE_QUENCH|1<<ICMP_REDIRECT|1<<ICMP_TIME_EXCEEDED|1<<ICMP_PARAMETERPROB), 4) = 0
setsockopt(3, SOL_IP, IP_RECVERR, [1], 4) = 0
setsockopt(3, SOL_SOCKET, SO_SNDBUF, [284], 4) = 0
setsockopt(3, SOL_SOCKET, SO_RCVBUF, [65536], 4) = 0
getsockopt(3, SOL_SOCKET, SO_RCVBUF, [131072], [4]) = 0
write(1, "PING 169.254.1.253 (169.254.1.25"..., 57PING 169.254.1.253 (169.254.1.253) 16(44) bytes of data.
) = 57
setsockopt(3, SOL_SOCKET, SO_TIMESTAMP, [1], 4) = 0
setsockopt(3, SOL_SOCKET, SO_SNDTIMEO, "\1\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0", 16) = 0
setsockopt(3, SOL_SOCKET, SO_RCVTIMEO, "\1\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0", 16) = 0
```
4. 发送ICMP请求
```c
sendto(3, "\10\0a\27g\326\0\1&\220G_\0\0\0\0\273!\6\0\0\0\0\0", 24, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("169.254.1.253")}, 16) = 24
# 注意, 这里收到了"192.168.0.1"的ICMP报文?
recvmsg(3, {msg_name={sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.168.0.1")}, msg_namelen=128->16, msg_iov=[{iov_base="E\0\0,\351\332\0\0@\1\17\227\300\250\0\1\300\250\0\16\0\0L\252Z\374\f &\220G_"..., iov_len=152}], msg_iovlen=1, msg_control=[{cmsg_len=32, cmsg_level=SOL_SOCKET, cmsg_type=SCM_TIMESTAMP, cmsg_data={tv_sec=1598525478, tv_usec=674988}}], msg_controllen=32, msg_flags=0}, 0) = 44
```
5. set filter
```c
setsockopt(3, SOL_SOCKET, SO_ATTACH_FILTER, {len=8, filter=0x5588bb473020}, 16) = 0
```

6. 收到ICMP响应
```c
poll([{fd=3, events=POLLIN|POLLERR}], 1, 513) = 0 (Timeout)
sendto(3, "\10\0\333\311i\0\0\2J\221G_\0\0\0\0\31C\10\0\0\0\0\0", 24, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("169.254.1.253")}, 16) = 24
recvmsg(3, {msg_namelen=128}, 0)        = -1 EAGAIN (Resource temporarily unavailable)
sendto(3, "\10\0\252Xi\0\0\3K\221G_\0\0\0\0I\263\10\0\0\0\0\0", 24, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("169.254.1.253")}, 16) = 24
recvmsg(3, {msg_namelen=128}, 0)        = -1 EAGAIN (Resource temporarily unavailable)
```

## ping都收到了什么报文?
用strace看ping进程的系统调用`strace -p 18060 -s 256 -x`, 发现正常情况下能收到`169.254.1.253`的ICMP响应报文
```c
--- SIGALRM {si_signo=SIGALRM, si_code=SI_KERNEL} ---
clock_gettime(CLOCK_MONOTONIC, {tv_sec=64787, tv_nsec=218738867}) = 0
sendto(0, "\x08\x42\x81\xa4\x46\x8c\x0b\x2f\x15\x9e\x81\x32\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42", 24, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("169.254.1.253")}, 28) = 24
rt_sigaction(SIGALRM, {sa_handler=0x10021800, sa_mask=[ALRM], sa_flags=SA_RESTART}, {sa_handler=0x10021800, sa_mask=[ALRM], sa_flags=SA_RESTART}, 16) = 0
setitimer(ITIMER_REAL, {it_interval={tv_sec=0, tv_usec=0}, it_value={tv_sec=1, tv_usec=0}}, NULL) = 0
rt_sigreturn({mask=[]})                 = 6044
recvfrom(0, "\x45\x00\x00\x2c\x50\xf3\x00\x00\x40\x01\xd2\xdf\xa9\xfe\x01\xfd\xa9\xfe\x01\x05\x00\x42\x89\xa4\x46\x8c\x0b\x2f\x15\x9e\x81\x32\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42\x42", 152, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("169.254.1.253")}, [16]) = 44
clock_gettime(CLOCK_MONOTONIC, {tv_sec=64787, tv_nsec=221353769}) = 0
write(1, "24 bytes from 169.254.1.253: seq=2863 ttl=64 time=2.615 ms\n", 59) = 59
recvfrom(0, 0x1041f1b0, 152, 0, 0x7f979a80, [16]) = ? ERESTARTSYS (To be restarted if SA_RESTART is set)
```

打流的情况下, 收到很多`127.0.0.1`的报文.
```
recvfrom(0, "\x45\xc0\x00\xf2\x92\x27\x00\x00\x40\x01\xe9\x21\x7f\x00\x00\x01\x7f\x00\x00\x01\x03\x03\x1a\x69\x00\x00\x00\x00\x45\x00\x00\xd6\x58\x4f\x40\x00\x40\x11\xe3\xc5\x7f\x00\x00\x01\x7f\x00\x00\x01\x9e\x21\x02\x02\x00\xc2\xfe\xd5\x3c\x31\x35\x3e\x41\x50\x50\x5f\x4e\x41\x4d\x45\x3a\x64\x68\x63\x70\x76\x34\x5f\x72\x65\x6c\x61\x79\x5f\x6c\x6f\x67\x69\x63\x5f\x61\x70\x70\x2c\x41\x50\x50\x5f\x56\x45\x52\x53\x49\x4f\x4e\x3a\x32\x30\x30\x39\x2e\x32\x35\x36\x2c\x4d\x4f\x44\x55\x4c\x45\x5f\x4e\x41\x4d\x45\x3a\x64\x68\x63\x70\x5f\x72\x65\x6c\x61\x79\x5f\x70\x72\x6f\x74\x6f\x63\x6f\x6c\x2c\x41\x50\x50\x5f\x50\x48\x41", 152, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("127.0.0.1")}, [16]) = 152
recvfrom(0, "\x45\xc0\x00\xf2\x92\x28\x00\x00\x40\x01\xe9\x20\x7f\x00\x00\x01\x7f\x00\x00\x01\x03\x03\x1a\x69\x00\x00\x00\x00\x45\x00\x00\xd6\x58\x50\x40\x00\x40\x11\xe3\xc4\x7f\x00\x00\x01\x7f\x00\x00\x01\x9e\x21\x02\x02\x00\xc2\xfe\xd5\x3c\x31\x35\x3e\x41\x50\x50\x5f\x4e\x41\x4d\x45\x3a\x64\x68\x63\x70\x76\x34\x5f\x72\x65\x6c\x61\x79\x5f\x6c\x6f\x67\x69\x63\x5f\x61\x70\x70\x2c\x41\x50\x50\x5f\x56\x45\x52\x53\x49\x4f\x4e\x3a\x32\x30\x30\x39\x2e\x32\x35\x36\x2c\x4d\x4f\x44\x55\x4c\x45\x5f\x4e\x41\x4d\x45\x3a\x64\x68\x63\x70\x5f\x72\x65\x6c\x61\x79\x5f\x70\x72\x6f\x74\x6f\x63\x6f\x6c\x2c\x41\x50\x50\x5f\x50\x48\x41", 152, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("127.0.0.1")}, [16]) = 152
```

* 这些报文都是ICMP报文, 因为socket指定了`socket(AF_INET, SOCK_RAW, IPPROTO_ICMP) = 3`

> 额外冲上来的报文的确也都是ICMP的，但是是板内自己的ICMP报文。
应该是当时pppoe_ia_app产生了大量syslog想报给514端口的syslog server，但是server没有开，内核回复了端口不可达的ICMP报文。

* 报文是从环回口上来的, man recvfrom, src_addr是协议栈填上来的.
```c
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                        struct sockaddr *src_addr, socklen_t *addrlen);
```

* ICMP的reply报文, 如果是不可达, 似乎内核会把之前的payload报文封装到不可达报文里.

## 为什么会收到127.0.0.1的ICMP报文?
答案是本来就要收到.因为`socket(AF_INET, SOCK_RAW, IPPROTO_ICMP)`的意思就是说IPPROTO_ICMP协议的SOCK_RAW都收上来.
busybox版本的ping, 没有实现上面的第5步. 没有filter, 所以ICMP报文都上来了.  
验证方法也简单, 在`ping 169.254.1.253`的时候, 再`ping 127.0.0.1`, 发现第一个ping进程就会收到`127.0.0.1`的ICMP reply
而`iputils`的ping, 会set filter. 所以, PC上的ping不会有这个现象

## 结论
* buxybox的ping是精简版, 会收说有的ICMP报文, 除非指定接口, 比如(-I eth0)
* 通用版本的ping本来就调用了`setsockopt(3, SOL_SOCKET, SO_ATTACH_FILTER, {len=8, filter=0x5588bb473020}, 16)`接口, 过滤了别的地址的IMCP报文.
* 用strace抓数据, 放到wireshark里面协议分析是好方法.

参考[详细的ping的协议栈流程解析](https://jgsun.github.io/2018/12/21/linux-ping/)