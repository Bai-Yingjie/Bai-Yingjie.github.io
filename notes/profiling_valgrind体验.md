在alpine linux上体验valgrind

# valgrind简介
[valgrind](https://valgrind.org/)是个基于代码instrumantation的工具集. 所谓的instrumentation就是动态修改目标程序的代码段, 插入一些调试代码来改变目标程序的行为, 从而能够检测或profiling目标程序.

用valgrind运行目标程序, valgrind会在结束的时候打印SUMMARY. 比如下面的例子中, valgrind默认使用Memcheck, SUMMARY显示`echo`命令总共调用了一次`alloc`, 一次`free`, 共使用4字节内存.
```shell
yingjieb@RebornLinux:yingjieb_rebornlinux_test:62545 ~/tmp
$ valgrind echo "hi"
==11292== Memcheck, a memory error detector
==11292== Copyright (C) 2002-2022, and GNU GPL'd, by Julian Seward et al.
==11292== Using Valgrind-3.20.0 and LibVEX; rerun with -h for copyright info
==11292== Command: echo hi
==11292==
hi
==11292==
==11292== HEAP SUMMARY:
==11292==     in use at exit: 0 bytes in 0 blocks
==11292==   total heap usage: 1 allocs, 1 frees, 4 bytes allocated
==11292==
==11292== All heap blocks were freed -- no leaks are possible
==11292==
==11292== For lists of detected and suppressed errors, rerun with: -s
==11292== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

## 工具包
valgrind的工具包中包括:
* Memcheck: 检测内存泄漏. Memcheck runs programs about 10--30x slower than normal.
* Cachegrind: 可以模拟cache, 检测程序的cache和访问内存的性能. Cachegrind runs programs about 20--100x slower than normal.
* Callgrind: cachegrind的升级版. 可以对接KCachegrind图形化
* Massif: 检测堆使用情况. Massif runs programs about 20x slower than normal.
* Helgrind: 检测使用pthread的程序有没有竞争问题
* DRD: 和helgrind差不多, 检测多线程竞争的
* DHAT: 检测堆的. 带GUI程序

## valgrind原理
原文: https://valgrind.org/docs/manual/mc-tech-docs.html

### 起源
valgrind是在2002年发布的. 在发布之前, 作者已经构思了5年. 在这5年之前又两年, 作者在搞Haskell编译器的code generator的时候积累了相关经验, 并写了用户态的x86指令interpreter, 并意识到JIT才是更好的方向.

2000年左右作者开始写JIT, 设计instrumentation的模式. 2001年底设计基本完成, 目标是调试即将到来的KDE 3. KDE 3的核心开发团队给了大量的反馈.

根据Unix的惯例, valgrind的代码被重写了2到3遍, 包括CPU仿真, 寄存器分配器, 符号表reader等等. 这个惯例的原文是"build one to throw away; you will anyway", 意思是"先造一个再扔掉；你反正也会这么做".

### 原理
valgrind被编译成`valgrind.so`, 用`LD_PRELOAD`做为目标binary的动态库被load, 并加上`-z initfirst`标记, 让valgrind先跑初始化. 这样valgrind就得到了控制权, 目标binary被valgrind转义, 真正的CPU被trapped并运行转义后的代码, 变成了合成CPU(`synthetic CPU`), 或者叫模拟CPU(`simulated CPU`). 

目标程序的`main`执行完, valgrind的结束代码运行, 停止这个转义, 退出CPU模拟, 做一些统计分析, 最后通过`exit()`退出.

valgrind并不直接运行目标binary, 而是边翻译边运行翻译后的代码. 翻译后的代码保存在translation cache(`vg_tc`)里.

这个JIT翻译的核心是`vg_dispatch.S`中的`dispatch`函数. 用x86的`call`指令运行翻译后的函数, 在这个函数返回之前, 把原始的code addr保存在`%eax`, 这样翻译后的函数`ret`后, 又回到`dispatch`, 随后查找`%eax`的原始代码地址, 进行下一次翻译.

TC cache(`vg_tc`)使用LRU方式管理.

目标程序在valgrind看来是client, client的`malloc`, `free`等内存管理函数被valgrind翻译成自己的函数. `malloc`会在shadow block里记录额外信息, 比如调用栈; `free`会查找对应地址的shadow block, 并报告可能的错误.

### 如何debug自己
作者假设了一个场景, 比如做为一个CPU simulator, valgrind运行了一个大的软件, 比如那个年代的Netscape. 如果运行了几百万行指令后, valgrind挂了, 怎么调试呢?

作者给出的方案是让valgrind随时可以退出指令翻译, 让CPU直接运行client的原始代码. 具体方式是使用signal通知valgrind.  
这样就可以做二分法来查找问题代码的位置.

### 作者强调简单
相对性能, 作者更注重简单, 正确.
* 大量使用断言, 即使有5%的性能损失
* 不依赖其他库, 不用很炫的"算法"
* 不用其他头文件
* 和client程序在同一个地址空间

### 自定义UCode
Valgrind的JIT本质上是x86-to-x86 JIT, 但直接翻译x86指令太tm复杂了(" just too darn complicated"), 所以作者设计了UCode中间指令.

UCode和RISC指令集很像, 遵循AT&T汇编风格, 源操作在前, 目的操作在后. 后来的linux命令族, 比如cp, 也延续了这个风格.

### UCode设计(略)
client的每个指令, 都翻译成UCode.  
UCode执行的时候很简单, 根据`vg_from_ucode.c`, 查找每个UCode对应的x86指令.