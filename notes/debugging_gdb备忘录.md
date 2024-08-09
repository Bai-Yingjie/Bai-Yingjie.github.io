- [gdb tui模式](#gdb-tui模式)
- [gdb看所有线程的调用栈](#gdb看所有线程的调用栈)
- [远程调试app, 用gdbserver和gdb远程调试板子](#远程调试app-用gdbserver和gdb远程调试板子)
- [远程调试uboot, 用jtag做gdb server](#远程调试uboot-用jtag做gdb-server)
- [远程调试, over串口](#远程调试-over串口)
- [gdb调试nand](#gdb调试nand)
- [gdb基本](#gdb基本)
  - [查看当前要执行的行](#查看当前要执行的行)
  - [临时变量](#临时变量)
  - [自动显示变量, 每次c后或n后, 自动显示其值](#自动显示变量-每次c后或n后-自动显示其值)
  - [按结构体方式查看内存地址](#按结构体方式查看内存地址)
  - [显示内存内容](#显示内存内容)
  - [计算数值, 16进制显示](#计算数值-16进制显示)
  - [指定显示格式](#指定显示格式)
  - [执行任意函数](#执行任意函数)
  - [查看结构体定义](#查看结构体定义)
  - [汇编和C混合显示](#汇编和c混合显示)
  - [info命令族](#info命令族)
  - [加入libc符号表](#加入libc符号表)
  - [调试模块, 加载符号表](#调试模块-加载符号表)
  - [查看地址处的代码](#查看地址处的代码)
  - [显示内存布局](#显示内存布局)

# gdb tui模式
tui模式下更像个ide调试界面, 信息更丰富.
```shell
#可以先gdb program core来启动, 在gdb里面用layout命令来显示源码 asm和reg窗口
#Ctrl + x，再按a：回到传统模式，即退出layout，回到执行layout之前的调试窗口
#Ctrl + x, o: 切换激活窗口, 切换到命令窗口可以上下翻历史命令


layout split
layout src
layout reg


#刷新窗口
refresh 或Ctrl + L

#上一条命令快捷键: Ctrl+p
```

# gdb看所有线程的调用栈
```c
(gdb) info threads
(gdb) thread apply all bt full
```

# 远程调试app, 用gdbserver和gdb远程调试板子

板子上配好ip: 192.168.2.12
```shell
#直接带好参数
~ # gdbserver :9123 /isam/user/eoe_filter -n eth0 -t tap-fwd -E
```
我在mint上, 用sshfs mount了服务器的buildroot目录; 所以我在mint上就能操作:
```shell
export PATH=$PATH:~/work/share/buildroot73/output/host/opt/ext-toolchain/bin
#cd到被调试的app目录下
yingjieb@yingjieb-VirtualBox ~/work/share/buildroot73/output/build/isam-linux-target-apps-22ef847b216e5d579c016ed9dd9795d393b56001
#这个eoe_filter是带符号表的
#能自动找到sysroot:/home/yingjieb/work/share/buildroot73/output/host/opt/ext-toolchain/mips64-octeon-linux-gnu/sys-root/
Linux Mint 19.1 Tessa $ mips64-octeon-linux-gnu-gdb eoe_filter
#板子上的程序会停在__start, __start是lib32/ld.so.1的符号
(gdb) target remote 192.168.2.12:9123
(gdb) handle SIG42 nostop noprint
(gdb) set print pretty on
#先断点到这个函数:
(gdb) b openTapItf
#继续执行直到断点
(gdb) c
#结合代码, 我要看tap设备是否创建成功:
(gdb) b eoe_filter.c:116
#在另外一个窗口ip a能看到tap-fwd
(gdb) info locals
#再往下跟, 发现是子线程调用了exit(), 导致tap设备被注销了.
(gdb) next
```

# 远程调试uboot, 用jtag做gdb server
```shell
cd /repo2/yingjieb/cavium/sdk31_508/usr/local/Cavium_Networks/OCTEON-SDK/tools/bin
//gdb调试elf文件
mipsisa64-octeon-elf-gdb /repo2/yingjieb/u-boot-octeon-sdk3.1/u-boot-octeon_fpxtb
//启动空的gdb后, 再加载elf文件
file /repo2/yingjieb/u-boot-octeon-sdk3.1/u-boot-octeon_fpxtb
//可以设置architecture, 如果直接加载elf文件则不用设置
set architecture mips:octeon3
//启动远程调试
target remote 135.251.9.60:30000
//使用monitor指令
monitor help
monitor B::break.set 0xc00a2ff8 /program
//查看程序地址?
info line eth_send
//设置程序参数
gdb --args prog args
set args --ddr0spd=fpxtbspd --ddr_clock_hz=667000000
b init_octeon3_ddr3_interface
```

# 远程调试, over串口
在板子上:
```shell
board:
stty -F /dev/ttyS0 115200
stty -F /dev/ttyS0 460800
stty -F /dev/ttyS0 -a
( taskset 1 gdbserver /dev/ttyS0 ./isam_app useSTDIO) &
```
在服务器上:
```shell
ASBLX28:/repo/yingjieb/fdt063/sw/vobs/esam/build/reborn/buildroot-isam-reborn-cavium-fgltb/output/host/usr/bin
$ mips64-octeon-linux-gnu-gdb /repo/yingjieb/fdt063/sw/vobs/esam/build/fglt-b/OS/application/isam_app.nostrip
#这个135.251.199.198:2009是串口服务器地址
(gdb) target remote 135.251.199.198:2009
(gdb) set sysroot /repo/yingjieb/fdt063/sw/vobs/esam/build/reborn/buildroot-isam-reborn-cavium-fgltb/output/staging
(gdb) set remotetimeout 60
(gdb) handle SIG42 nostop noprint
(gdb) set print pretty on
(gdb) detach
```

# gdb调试nand
target:
```shell
devmem 0x1070000000500 64 0
devmem 0x1070000000508 64 0
devmem 0x1070000000510 64 0
devmem 0x1070000000518 64 0
echo ttyS1 > /sys/module/kgdboc/parameters/kgdboc
echo g > /proc/sysrq-trigger
从kgdb切换到kdb, 盲敲$3#33
```

host:
```shell
mips64-octeon-linux-gnu-gdb vmlinux
target remote 135.251.199.198:2114
set print pretty on
set sysroot /repo/yingjieb/fdt063/sw/vobs/esam/build/reborn/buildroot-isam-reborn-cavium-fgltb/output/staging
set remotetimeout 20
detach
bt
c
```

# gdb基本
## 查看当前要执行的行
```shell
frame, 简写f
```

## 临时变量
```
set $i="hello"
```

## 自动显示变量, 每次c后或n后, 自动显示其值
```
display 变量名
```

## 按结构体方式查看内存地址

```shell
p *(struct mtd_info *)0x80000000885dc018
(gdb) p ((struct txq *)0xffff8119f800)->elts_n
```

## 显示内存内容
```shell
#从priv->data这个地址开始, 显示2048个单位(2048), 每个单位一个字节(b), 按16进制显示(x)
x/2048xb priv->data
```

## 计算数值, 16进制显示
```
(gdb) p /x 0xffff90c8c430+1914*8
$7 = 0xffff90c90000
```

## 指定显示格式
```shell
#有时候, gdb不能准确显示一个uint16_t类型的变量的值, 比如
(gdb) p nb_pkt_per_burst
$19 = 65568
#此时需要用x加FMT来看, 8xh是说显示8个, 安装16进制halfword显示
(gdb) x/8xh &nb_pkt_per_burst
0x9f0c36 <nb_pkt_per_burst>: 0x0020 0x0001 0x0000 0x0000 0x0000 0x0040 0x0000 0x0000
#如果知道这个变量是uint16_t, 下面的也可以; 相当于x/1dh
(gdb) x/dh &nb_pkt_per_burst
0x9f0c36 <nb_pkt_per_burst>: 32
```

## 执行任意函数
```shell
(gdb) p system("nandtest -k -o0x0 -l0x20000 /dev/mtd6")
$1 = 1
```

## 查看结构体定义
```shell
(gdb) ptype trx_sys_t
```

## 汇编和C混合显示
```shell
(gdb) disassemble /m function
(gdb) disassemble /m lj_meta_lookup
(gdb) disassemble /m dp_netdev_input__
```

## info命令族
```shell
#源码信息
info source
#当前局部变量
info locals
info files
info threads
info sharedlibrary
info signals
info line
info tasks
info variables
info symbol
info stack
info registers
#还有不少
help info
```

## 加入libc符号表
```shell
#必须使/repo/yingjieb/glibc-2.16.0能够被板子访问到
gdb oflt.nostrip
set args useSTDIO
set print pretty on
set sysroot /repo/yingjieb/fdt063/sw/vobs/esam/build/reborn/buildroot-isam-reborn-cavium-fgltb/output/staging
#dir的意思是增加源文件的search path到gdb
dir /repo/yingjieb/glibc-2.16.0
dir /repo/yingjieb/glibc-2.16.0/nptl
dir /repo/yingjieb/glibc-ports-2.16.0
handle SIG42 nostop noprint
b cs_dpll_resetTest
b oak_spi_datatransfer
b board_commands.c:153
b fork.c:133
b do_system
```

## 调试模块, 加载符号表
```shell
#加载ko符号表
/sys/module/spi_oak_island/sections # cat .text
0xffffffffc0002000
/sys/module/spi_oak_island/sections # cat .data
0xffffffffc0000000

(gdb) add-symbol-file /repo/yingjieb/fdt063/sw/vobs/esam/build/reborn/buildroot-isam-reborn-cavium-fgltb/output/build/isam-linux-drivers-custom/misc/spi-oak-island.ko 0xffffffffc0002000 -s .data 0xffffffffc000000
```

## 查看地址处的代码
```shell
# source file and line number for an instruction address
info line *0x<target_addr>
# source lines around an instruction address
list *0x<target_addr>
# assembly instructions at an address
disas 0x<target_addr>, or
x/20i 0x<target_addr>
```

## 显示内存布局
```shell
#显示所有内存section
info files
#比上面命令显示更全
maintenance info sections 
```
