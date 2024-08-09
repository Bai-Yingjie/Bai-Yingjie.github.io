- [分离符号表](#分离符号表)
- [使用strace分析eoe_filter占用高的问题](#使用strace分析eoe_filter占用高的问题)
  - [统计](#统计)
- [使用其他版本的libc](#使用其他版本的libc)
  - [背景](#背景)
  - [直接使用是报错的](#直接使用是报错的)
  - [解决](#解决)
  - [LD_LIBRARY_PATH](#ld_library_path)
  - [LD_PRELOAD](#ld_preload)
  - [最终解决](#最终解决)
- [没有ldd看共享库](#没有ldd看共享库)
- [strace查看一个进程的open文件过程](#strace查看一个进程的open文件过程)
- [计算CPU load](#计算cpu-load)
  - [utime和cutime的关系](#utime和cutime的关系)
  - [/proc/stat](#procstat)
    - [上述所有相的和是CPU总的tick数](#上述所有相的和是cpu总的tick数)
  - [进程cpud利用率的计算](#进程cpud利用率的计算)
- [strace看系统调用时间](#strace看系统调用时间)
  - [I2C访问时间过长](#i2c访问时间过长)
- [在mips上使用perf和火焰图](#在mips上使用perf和火焰图)
  - [用perf record生成原始数据](#用perf-record生成原始数据)
  - [用perf script解析调用栈](#用perf-script解析调用栈)
  - [在mint虚拟机上, 生成火焰图](#在mint虚拟机上-生成火焰图)

# 分离符号表
用`objcopy --add-gnu-debuglink`的连接功能, 可以把strip后的可执行文件, 和只带debug信息的符号文件链接起来

```shell
#先做出只带debug符号的文件
objcopy --only-keep-debug "${tostripfile}" "${debugdir}/${debugfile}"
#把原始可执行文件strip
strip --strip-debug --strip-unneeded "${tostripfile}"
#设置debuglink
objcopy --add-gnu-debuglink="${debugdir}/${debugfile}" "${tostripfile}"
```

这样, strip后的文件能够单独运行; 在需要符号表的时候, 按照约定的路径`${debugdir}/${debugfile}`把debug文件放进去, 再运行strip file的时候, 符号就能找到

这里说的strip的符号指.debug相关的小节和.symtab .strtab
用`strip --strip-unneeded`命令后, 这些小节都会被strip掉. 而`--only-keep-debug`则只保留这些小节.
```shell
  [27] .debug_aranges    PROGBITS        00000000 3ed510 004f99 00   C  0   0  8   
  [28] .debug_info       PROGBITS        00000000 3f24a9 03b36a 00   C  0   0  1   
  [29] .debug_abbrev     PROGBITS        00000000 42d813 00324a 00   C  0   0  1   
  [30] .debug_line       PROGBITS        00000000 430a5d 054ab1 00   C  0   0  1   
  [31] .debug_str        PROGBITS        00000000 48550e 013d3f 01 MSC  0   0  1   
  [32] .debug_loc        PROGBITS        00000000 49924d 0097b3 00   C  0   0  1   
  [33] .debug_ranges     PROGBITS        00000000 4a2a00 0085c7 00   C  0   0  8   
  [34] .gnu.attributes   GNU_ATTRIBUTES  00000000 2e8a6f 000012 00      0   0  1   
  [35] .symtab           SYMTAB          00000000 2e8a84 092910 10     36 27035  4 
  [36] .strtab           STRTAB          00000000 37b394 07217a 00      0   0  1
```

# 使用strace分析eoe_filter占用高的问题
## 统计
```shell
/ # timeout 10 strace -c -p `pidof eoe_filter`
strace: Process 27238 attached
strace: Process 27238 detached
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 52.00    0.130000          15      8225           read
 48.00    0.120000          14      8225           write
------ ----------- ----------- --------- --------- ----------------
```
注strace的选项:
```
-c: Count time, calls, and errors for each system call and report a summary on program exit.
```
`timeout`也很实用, 带超时的执行一个命令.

# 使用其他版本的libc
## 背景
我想在板子上运行`netstat`, 但板子上的版本是busybox版本, 功能不全.
但是我几个月之前编译过完整版本的`netstat`, 想拷到板子上直接运行

## 直接使用是报错的
```shell
wget http://172.24.213.190:8088/bin/netstat
~ # ./netstat
./netstat: relocation error: ./netstat: symbol h_errno version GLIBC_PRIVATE not defined in file libc.so.6 with link time reference
```
说明libc版本不匹配

## 解决
## LD_LIBRARY_PATH
首先想到拷贝原libc.so版本, 全在/root目录下执行
报一样的错误
```shell
wget http://172.24.213.190:8088/lib/libc.so.6

~ # LD_LIBRARY_PATH=`pwd` ./netstat
./netstat: relocation error: ./netstat: symbol h_errno version GLIBC_PRIVATE not defined in file libc.so.6 with link time reference
```
## LD_PRELOAD
使用LD_PRELOAD报错不一样, 但还是不行
```shell
~ # LD_PRELOAD='/root/libc.so.6' ./netstat
ERROR: ld.so: object '/root/libc.so.6' from LD_PRELOAD cannot be preloaded (cannot open shared object file): ignored.
./netstat: relocation error: ./netstat: symbol h_errno version GLIBC_PRIVATE not defined in file libc.so.6 with link time reference
```

## 最终解决
使用ld.so来执行.
背景知识是: 所有可执行的文件实际上都是ld.so来执行的
```shell
wget http://172.24.213.190:8088/lib/ld-2.16.so

//注意使用--library-path执行so路径, 本文这里就包括了老版本的libc.so
./ld-2.16.so --library-path `pwd` ./netstat -apn
```

# 没有ldd看共享库
```shell
~ # LD_TRACE_LOADED_OBJECTS=1 ./htop
    linux-vdso.so.1 (0x773c9000)
    libncurses.so.6 => /usr/lib/libncurses.so.6 (0x77338000)
    libm.so.6 => /lib/libm.so.6 (0x77247000)
    libc.so.6 => /lib/libc.so.6 (0x77091000)
    /lib32-fp/ld.so.1 (0x7738b000)

~ # LD_TRACE_LOADED_OBJECTS=1 ./perf
        linux-vdso.so.1 (0x77141000)
        libunwind.so.8 => /usr/lib/libunwind.so.8 (0x770b6000)
        libunwind-mips.so.8 => /usr/lib/libunwind-mips.so.8 (0x77066000)
        libpthread.so.0 => /lib/libpthread.so.0 (0x77033000)
        librt.so.1 => /lib/librt.so.1 (0x77012000)
        libm.so.6 => /lib/libm.so.6 (0x76f21000)
        libdl.so.2 => /lib/libdl.so.2 (0x76f00000)
        libelf.so.1 => /usr/lib/libelf.so.1 (0x76ecf000)
        libdw.so.1 => /usr/lib/libdw.so.1 (0x76e6e000)
        libcrypto.so.1.0.0 => /usr/lib/libcrypto.so.1.0.0 (0x76ca4000)
        libz.so.1 => /usr/lib/libz.so.1 (0x76c7f000)
        libc.so.6 => /lib/libc.so.6 (0x76ac9000)
        /lib32-fp/ld.so.1 (0x77103000)
        libgcc_s.so.1 => /lib/libgcc_s.so.1 (0x76a9e000)
```
用`man ld.so`来看详细的ld程序信息.
我们知道linux启动一个elf程序, 实际上是先启动ld程序, 然后ld来启动elf.
ld.so也可以直接启动:
```shell
/lib/ld-linux.so.* [OPTIONS] [PROGRAM [ARGUMENTS]]
The programs ld.so and ld-linux.so* find and load the shared objects (shared libraries) needed by a program, prepare the program to run, and then run it.
```
ld会看一些环境变量:
```shell
LD_ASSUME_KERNEL
LD_BIND_NOW: ld在加载app的时候就解决所有的符号引用, 而不是当符号被引用的时候才解析.
LD_LIBRARY_PATH: 动态库搜索路径. 常用
LD_PRELOAD: 预加载的so列表, 作用在其他so之前
LD_TRACE_LOADED_OBJECTS: 后面跟任何值, 效果和ldd一样
LD_DEBUG=all: 输出ld的debug信息到stderr
LD_DEBUG_OUTPUT: LD_DEBUG的输出到指定位置
```

# strace查看一个进程的open文件过程
strace支持filter. 比如我想trace htop 16122的所有open文件的过程, 来看看它到底怎么得到进程树的.

```shell
strace -p 16122 -e trace=openat -o s.log

#其他举例
#open和openat都是打开文件, 但它们是不一样的系统调用
-e trace=open,stat
#支持正则
-e trace=/regex

#其他常用
//跟踪文件相关的: 类似 -e trace=open,stat,chmod,unlink,...
-e trace=%file

//跟踪进程管理相关的, 比如 fork, wait, and exec
-e trace=%process

//跟踪网络相关的
-e trace=%network

//跟踪信号相关的
-e trace=%signal

//IPC相关的
-e trace=%ipc

//文件描述符相关的
-e trace=%desc

//mmap相关的
-e trace=%memory
```

# 计算CPU load
参考: linux/Documentation/filesystems/proc.txt
参考: `man proc`搜索stat
```shell
(14) utime %lu
                        Amount of time that this process has been scheduled in user mode, measured in clock ticks (divide by sysconf(_SC_CLK_TCK)). This includes guest time,
                        guest_time (time spent running a virtual CPU, see below), so that applications that are not aware of the guest time field do not lose that time from their
                        calculations.
(15) stime %lu
                        Amount of time that this process has been scheduled in kernel mode, measured in clock ticks (divide by sysconf(_SC_CLK_TCK)).
(16) cutime %ld : 在wait返回后更新
                        Amount of time that this process's waited-for children have been scheduled in user mode, measured in clock ticks (divide by sysconf(_SC_CLK_TCK)). (See also
                        times(2).) This includes guest time, cguest_time (time spent running a virtual CPU, see below).
(17) cstime %ld
                        Amount of time that this process's waited-for children have been scheduled in kernel mode, measured in clock ticks (divide by sysconf(_SC_CLK_TCK)).
(24) rss %ld
                        Resident Set Size: number of pages the process has in real memory. This is just the pages which count toward text, data, or stack space. This does not
                        include pages which have not been demand-loaded in, or which are swapped out.
```
## utime和cutime的关系
比如一个bash进程, 只起了一个外部程序, 比如 `./tooManyTimer`:
对bash进程来说
* utime是自己用户态执行的时间
* cutime是`./tooManyTimer`的执行时间, 平时不更新, 只有子进程退出的时候, 父进程waitpid返回才更新

对子进程`./tooManyTimer`来说:
* utime是自己的执行时间, 包括所有线程
* 这个程序没有子进程, cutime一直是0

## /proc/stat
同样的`man proc`的`/proc/stat`小节有讲
```shell
/proc/stat
    kernel/system statistics. Varies with architecture. Common entries include:

    cpu 10132153 290696 3084719 46828483 16683 0 25195 0 175628 0
    cpu0 1393280 32966 572056 13343292 6130 0 17875 0 23933 0
        The amount of time, measured in units of USER_HZ (1/100ths of a second on most architectures, use sysconf(_SC_CLK_TCK) to obtain the right value), that the sys‐
        tem ("cpu" line) or the specific CPU ("cpuN" line) spent in various states:
        user (1) Time spent in user mode.
        nice (2) Time spent in user mode with low priority (nice).
        system (3) Time spent in system mode.
        idle (4) Time spent in the idle task. This value should be USER_HZ times the second entry in the /proc/uptime pseudo-file.
        iowait (since Linux 2.5.41)
            (5) Time waiting for I/O to complete. This value is not reliable, for the following reasons:
            1. The CPU will not wait for I/O to complete; iowait is the time that a task is waiting for I/O to complete. When a CPU goes into idle state for out‐
                standing task I/O, another task will be scheduled on this CPU.
            2. On a multi-core CPU, the task waiting for I/O to complete is not running on any CPU, so the iowait of each CPU is difficult to calculate.
            3. The value in this field may decrease in certain conditions.
        irq (since Linux 2.6.0-test4)
            (6) Time servicing interrupts.
        softirq (since Linux 2.6.0-test4)
            (7) Time servicing softirqs.
        steal (since Linux 2.6.11)
            (8) Stolen time, which is the time spent in other operating systems when running in a virtualized environment
        guest (since Linux 2.6.24)
            (9) Time spent running a virtual CPU for guest operating systems under the control of the Linux kernel.
        guest_nice (since Linux 2.6.33)
            (10) Time spent running a niced guest (virtual CPU for guest operating systems under the control of the Linux kernel).
```
### 上述所有相的和是CPU总的tick数
> The amount of time, measured in units of **USER_HZ** (1/100ths of a second on most architectures, use `sysconf(_SC_CLK_TCK)` to obtain the right value)

一般这个USER_HZ是100, 就是说每个CPU, 每秒有100个tick.
```shell
while true; do t0=`cat /proc/stat | grep cpu0 | awk '{print $2+$3+$4+$5+$6+$7+$8+$9+$10}'`;sleep 1;t1=`cat /proc/stat | grep cpu0 | awk '{print $2+$3+$4+$5+$6+$7+$8+$9+$10}'`;echo $(($t1 - $t0));done
101
104
103
101
```
一个系统上的所有核(每行)的和是差不多的.
```shell
$ cat /proc/stat | grep cpu
cpu 9669309 24418 17471207 10263522262 224627 0 828332 14242740 0 0
cpu0 533764 573 734456 427550594 11586 0 98187 615793 0 0
cpu1 383788 2184 626860 428073382 12937 0 39272 510523 0 0
cpu2 321855 1849 602457 428122456 12505 0 15910 490530 0 0
cpu3 302345 599 617397 427862041 12518 0 20603 638439 0 0

#在一个24核的qemu/kvm虚拟机上, 分别对每个CPU所有field相加, 结果是差不多的
$ cat /proc/stat | grep cpu | awk '{print $2+$3+$4+$5+$6+$7+$8+$9+$10}'
1.03062e+10
429553194
429657165
429575794
429462193
...
```

## 进程cpud利用率的计算
先算pid的tick和上次的差值
再算总的tick和上次的差值
两个差值相除

# strace看系统调用时间
`strace -ttT`选项可以显示每个系统调用花费的时间:
```shell
man strace
-tt If given twice, the time printed will include the microseconds.
-T Show the time spent in system calls. This records the time difference between the beginning and the end of each system call.

Linux Mint 19.1 Tessa $ strace -ttT ls
17:51:12.683070 execve("/bin/ls", ["ls"], 0x7ffd92408b48 /* 35 vars */) = 0 <0.000289>
17:51:12.683597 brk(NULL) = 0x558b9cd69000 <0.000067>
17:51:12.683844 access("/etc/ld.so.nohwcap", F_OK) = -1 ENOENT (No such file or directory) <0.000087>
17:51:12.684012 access("/etc/ld.so.preload", R_OK) = -1 ENOENT (No such file or directory) <0.000042>
17:51:12.684179 openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3 <0.000064>
17:51:12.684336 fstat(3, {st_mode=S_IFREG|0644, st_size=93197, ...}) = 0 <0.000062>
17:51:12.684529 mmap(NULL, 93197, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6953d86000 <0.000093>
17:51:12.684713 close(3) = 0 <0.000120>
17:51:12.684924 access("/etc/ld.so.nohwcap", F_OK) = -1 ENOENT (No such file or directory) <0.000093>
17:51:12.685325 openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libselinux.so.1", O_RDONLY|O_CLOEXEC) = 3 <0.000060>
17:51:12.685523 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\20b\0\0\0\0\0\0"..., 832) = 832 <0.000050>
17:51:12.685682 fstat(3, {st_mode=S_IFREG|0644, st_size=154832, ...}) = 0 <0.000055>
17:51:12.685842 mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6953d84000 <0.000055>
17:51:12.686006 mmap(NULL, 2259152, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f695394e000 <0.000094>
```
可以看到, 普通的系统调用在`0.1ms`以下.

## I2C访问时间过长
`09:14:41.979375 ioctl(38, _IOC(0, 0x7, 0x7, 0), 0x7ffbe168) = -1 EIO (Input/output error) <0.115278>`
这里的0.115278秒(115ms)实在是太长了.

```shell
09:14:41.857370 readv(37, [{iov_base="\0\0\1\0\0\0\0\0\0\0\0000\0\0\0\1\0\0\0\4\0\3\251\"\0\0\0\2\0\0\0\4"..., iov_len=72}], 1) = 72 <0.000103>
09:14:41.857861 stat("/etc/localtime", {st_mode=S_IFREG|0644, st_size=127, ...}) = 0 <0.000069>
09:14:41.858430 clock_gettime(CLOCK_MONOTONIC, {tv_sec=240055, tv_nsec=913917513}) = 0 <0.000071>
09:14:41.858764 openat(AT_FDCWD, "/dev/i2c-14", O_RDWR|O_LARGEFILE) = 38 <0.000261>
09:14:41.859242 ioctl(38, _IOC(0, 0x7, 0x7, 0), 0x7ffbe198) = -1 EIO (Input/output error) <0.115366>
09:14:41.975311 close(38) = 0 <0.000362>
09:14:41.976146 openat(AT_FDCWD, "/isam/slot_default/temp/temp0/temp", O_RDONLY|O_LARGEFILE) = 38 <0.000577>
09:14:41.976959 lseek(38, 0, SEEK_SET) = 0 <0.000138>
09:14:41.977252 read(38, " 34.500\0", 31) = 9 <0.000905>
09:14:41.978402 read(38, "", 22) = 0 <0.000268>
09:14:41.978903 close(38) = 0 <0.000043>
09:14:41.979192 openat(AT_FDCWD, "/dev/i2c-14", O_RDWR|O_LARGEFILE) = 38 <0.000052>
09:14:41.979375 ioctl(38, _IOC(0, 0x7, 0x7, 0), 0x7ffbe168) = -1 EIO (Input/output error) <0.115278>
09:14:42.095006 close(38) = 0 <0.000213>
09:14:42.095775 openat(AT_FDCWD, "/dev/i2c-14", O_RDWR|O_LARGEFILE) = 38 <0.000322>
09:14:42.096574 ioctl(38, _IOC(0, 0x7, 0x7, 0), 0x7ffbe168) = -1 EIO (Input/output error) <0.118028>
09:14:42.215039 close(38) = 0 <0.000037>
09:14:42.215294 epoll_wait(3, [], 32, 0) = 0 <0.000082>
09:14:42.215670 openat(AT_FDCWD, "/dev/i2c-15", O_RDWR|O_LARGEFILE) = 38 <0.000052>
09:14:42.215964 ioctl(38, _IOC(0, 0x7, 0x7, 0), 0x7ffbe198) = -1 EIO (Input/output error) <0.118636>
09:14:42.335088 close(38) = 0 <0.000315>
09:14:42.335935 openat(AT_FDCWD, "/dev/i2c-4", O_RDWR|O_LARGEFILE) = 38 <0.000066>
09:14:42.336191 ioctl(38, _IOC(0, 0x7, 0x7, 0), 0x7ffbe148) = 2 <0.000991>
09:14:42.337503 close(38) = 0 <0.000257>
09:14:42.338201 openat(AT_FDCWD, "/dev/i2c-4", O_RDWR|O_LARGEFILE) = 38 <0.00
```

# 在mips上使用perf和火焰图

## 用perf record生成原始数据
注意, 在mips上, 必须用`-g --call-graph dwarf`才能生成火焰图
```shell
#对整个系统做profiling
perf record -F 500 -e cycles -g --call-graph dwarf -a -- sleep 30
#对某个进程做profiling
perf record -F 500 -e cycles -g --call-graph dwarf -p `pidof switch_hwa_app` -- sleep 30
perf record -F 500 -e cycles -g --call-graph dwarf -p `pidof xpon_hwa_app` -- sleep 30
perf record -F 500 -e cycles -g --call-graph dwarf -p `pidof xponhwadp` -- sleep 30

#也可以只对用户态做profiling, 用-e cycles:u
perf record -F 1000 -e cycles:u -g --call-graph dwarf -p 18852 -- sleep 60
```

## 用perf script解析调用栈
```shell
#先准备好目录, 用sshfs存放script后的文件
sshfs bsfs yingjieb@172.24.213.190:/repo/yingjieb/ms/gopoc/poclog-ver20200101/profiling
#准备符号表目录, 一般板子上的二进制是strip过的, 要解析调用栈, 必须有符号表
#这里的sysroot就相当于板子的根目录
sshfs wsfs chwang@135.251.206.190:/repo/chwang/ms/buildroot_poc/output/host/mips64-buildroot-linux-gnu/sysroot
#即使sysroot目录, typeB里面的二进制也是没有的
#先用下面的命令找出哪些符号不能解析, 手动拷到sysroot对应目录下
perf script | grep unknown | sort -u
#比如下面这几个
/usr/lib/libevent_core-2.1.so.6.0.2
/mnt/isam/NW89AA62.801/switch_hwa_fglt-b_app
/mnt/isam/NW89AA62.801/libyAPI.so.0.0.2
/lib/libc-2.16.so
/usr/lib/libhxps.so
#拷完用这个确认一下
perf script --symfs wsfs --kallsyms /proc/kallsyms | grep unknown | sort -u
```
准备工作完成后, 开始解析perf data
```shell
#用symfs指定root目录, 用kallsyms指定kernel符号表
perf script --symfs wsfs --kallsyms /proc/kallsyms > bsfs/sysidle-switch_hwa_app/perf.script
```
## 在mint虚拟机上, 生成火焰图 
```shell
yingjieb@yingjieb-VirtualBox ~/work/share/gopoc/perflogpoc-ver20191227/profiling/sysidle-switch_hwa_app
Linux Mint 19.1 Tessa $ cat perf.script | ~/repo/FlameGraph/stackcollapse-perf.pl | ~/repo/FlameGraph/flamegraph.pl > ~/sf_work/tmp/poc.svg
```
