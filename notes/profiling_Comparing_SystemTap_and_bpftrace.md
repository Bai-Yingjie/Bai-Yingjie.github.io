原文: https://lwn.net/Articles/852112/ 原文下面的评论也要看

- [Installation](#installation)
- [Program structure](#program-structure)
- [Features](#features)
- [Real world uses](#real-world-uses)
- [Conclusions](#conclusions)

There are times when developers and system administrators need to diagnose problems in running code. The program to be examined can be a user-space process, the kernel, or both. Two of the major tools available on Linux to perform this sort of analysis are [SystemTap](https://sourceware.org/systemtap/wiki) and [bpftrace](https://github.com/iovisor/bpftrace). SystemTap has been available since 2005, while bpftrace is a more recent contender that, to some, may appear to have made SystemTap obsolete. However, SystemTap is still the preferred tool for some real-world use cases.

Although dynamic instrumentation capabilities, in the form of [KProbes](https://lwn.net/Articles/132196/), were added to Linux [as early as 2004](https://lwn.net/Articles/100332/), the functionality was hard to use and not particularly well known. Sun released [DTrace](https://www.usenix.org/legacy/event/usenix04/tech/general/cantrill.html) one year later, and soon that system became [one of the highlights of Solaris](https://lwn.net/Articles/244536/). Naturally, Linux users started [asking for something similar](https://lwn.net/Articles/162701/), and SystemTap quickly emerged as the most promising answer. But SystemTap was criticized as being difficult to get working, while DTrace on Solaris could be expected to simply work out of the box.

While DTrace came with both kernel and user-space tracing capabilities, it wasn't until 2012 that Linux gained support for user-space tracing in the form of [Uprobes](https://lwn.net/Articles/499190/). Around 2019, bpftrace [gained significant traction](https://lwn.net/Articles/793749/), in part due to the general attention being paid to the [various use cases for BPF](https://lwn.net/Articles/740157/). More recently, Oracle has been working on a [re-implementation of DTrace](https://lwn.net/Articles/840113/), for Linux, based on the latest tracing facilities in the kernel, although, at this point, it may be too late for DTrace given the options that are already available in this space.

The underlying kernel infrastructure used by both SystemTap and bpftrace is largely the same: KProbes, for dynamically tracing kernel functions, [tracepoints](https://lwn.net/Articles/346470/) for static kernel instrumentation, Uprobes for dynamic instrumentation of user-level functions, and [user-level statically defined tracing (USDT)](https://lwn.net/Articles/753601/) for static user-space instrumentation. Both systems allow instrumenting the kernel and user-space programs through a "script" in a high-level language that can be used to specify what needs to be probed and how.

The important design distinction between the two is that SystemTap translates the user-supplied script into C code, which is then compiled and loaded as a module into a running Linux kernel. Instead, bpftrace converts the script to [LLVM](https://llvm.org/) intermediate representation, which is then compiled to BPF. Using BPF has several advantages: creating and running a BPF program is significantly faster than building and loading a kernel module. Support for data structures consisting of key/value pairs can be easily added by using BPF maps. The [BPF verifier](https://www.spinics.net/lists/xdp-newbies/msg00185.html) ensures that BPF programs will not cause the system to crash, while the kernel module approach used by SystemTap implies the need for implementing various safety checks in the runtime. On the other hand, using BPF makes certain features hard to implement, for example, a custom stack walker, as we shall see later in the article.

The following example shows the similarity between the two systems from the user standpoint. A simple SystemTap program to instrument the kernel function `icmp_echo()` looks like this:

```c
    probe kernel.function("icmp_echo") {
        println("icmp_echo was called")
    }
```

The equivalent bpftrace program is:

```c
    kprobe:icmp_echo {
        print("icmp_echo was called")
    }
```

We will now look at the differences between SystemTap and bpftrace in terms of installation procedure, program structure, and features.

## Installation

Both SystemTap and bpftrace are packaged by all major Linux distributions and can be installed easily using the familiar package managers. SystemTap requires the Linux kernel headers to be installed in order to work, while bpftrace does not, as long as the kernel has [BPF Type Format](https://www.kernel.org/doc/html/latest/bpf/btf.html) (BTF) support enabled. Depending on whether the user wants to analyze a user-space program or the kernel, there might be additional requirements. For user-space software, both SystemTap and bpftrace require the debugging symbols of the software under examination. The details of how to install the symbol data depend on the distribution.

On systems with elfutils 0.178 or later, SystemTap makes the process of finding and installing the right debug symbols fully automatic by using a [remote debuginfod server](https://lwn.net/Articles/847256/). For example, on Debian systems:

```sh
    # export DEBUGINFOD_URLS=https://debuginfod.debian.net
    # export DEBUGINFOD_PROGRESS=1
    # stap -ve 'probe process("/bin/ls").function("format_user_or_group") { println(pp()) }'
    Downloading from https://debuginfod.debian.net/
    [...]
```

This feature is [not yet available](https://github.com/iovisor/bpftrace/issues/1774) for bpftrace.

For kernel instrumentation, SystemTap requires the kernel debugging symbols to be installed in order to use the advanced features of the tool, such as looking up the arguments or local variables of a function, as well as instrumenting specific lines of code within the function body. In this case, too, a remote debuginfod server can be used to automate the process.

## Program structure

Both systems provide an [AWK](https://lwn.net/Articles/820829/)-like language, inspired by DTrace's [D](https://docs.oracle.com/cd/E18752_01/html/819-5488/gcfqr.html), to describe predicates and actions. The bpftrace language is pretty much the same as D, and follows this general structure:

```sh
    probe-descriptions
    /predicate/
    {
        action-statements
    }
```

That is to say: when the probes fire, if the given (optional) predicate matches, perform the specified actions.

The structure of SystemTap programs is slightly different:

```sh
    probe PROBEPOINT [, PROBEPOINT] {
        [STMT ...]
    }
```

In SystemTap there is no support for specifying a predicate built into the language, but conditional statements can be used to achieve the same goal.

For example, the following bpftrace program prints all [`mmap()`](https://man7.org/linux/man-pages/man2/mmap.2.html) calls issued by the process with PID 31316:

```sh
    uprobe:/lib/x86_64-linux-gnu/libc.so.6:mmap
    /pid == 31316/
    {
        print("mmap by 31316")
    }
```

The SystemTap equivalent is:

```sh
    probe process("/lib/x86_64-linux-gnu/libc.so.6").function("mmap") {
        if (pid() == 31316) {
            println("mmap by 31316")
        }
    }
```

Data aggregation and reporting in bpftrace is done exactly the same way as it is done in DTrace. For example, the following program does a by-PID sum and aggregation of the number of bytes sent with the `tcp_sendmsg()` kernel function:

```sh
    $ sudo bpftrace -e 'kprobe:tcp_sendmsg { @bytes[pid] = sum(arg2); }'
    Attaching 1 probe...
    ^C

    @bytes[58832]: 75
    @bytes[58847]: 77
    @bytes[58852]: 857
```

Like DTrace, bpftrace defaults to automatically printing aggregation results when the program exits: no code had to be written to print the breakdown by PID above. The downside of this implicit behavior is that, to avoid automatic printing of all data structures, users have to explicitly `clear()` those that should not be printed. For instance, to change the script above and only print the top 5 processes, the `bytes` map must be cleared upon program termination.

```sh
    kprobe:tcp_sendmsg {
        @bytes[pid] = sum(arg2);
    }

    END {
        print(@bytes, 5);
        clear(@bytes);
    }
```

Some powerful facilities for [generating histograms](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#8-hist-log2-histogram) are available too, allowing for terse scripts such as the following, which operates on the number of bytes read in calls to `vfs_read()`:

```sh
    $ sudo bpftrace -e 'kretprobe:vfs_read { @bytes = hist(retval); }'
    Attaching 1 probe...
    ^C

    @bytes:
    (..., 0)             169 |@@                                                  |
    [0]                  206 |@@@                                                 |
    [1]                 1579 |@@@@@@@@@@@@@@@@@@@@@@@@@@@                         |
    [2, 4)                13 |                                                    |
    [4, 8)                 9 |                                                    |
    [8, 16)             2970 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
    [16, 32)              45 |                                                    |
    [32, 64)              91 |@                                                   |
    [64, 128)            108 |@                                                   |
    [128, 256)            10 |                                                    |
    [256, 512)             8 |                                                    |
    [512, 1K)             69 |@                                                   |
    [1K, 2K)              97 |@                                                   |
    [2K, 4K)              37 |                                                    |
    [4K, 8K)              64 |@                                                   |
    [8K, 16K)             24 |                                                    |
    [16K, 32K)            29 |                                                    |
    [32K, 64K)            80 |@                                                   |
    [64K, 128K)           18 |                                                    |
    [128K, 256K)           0 |                                                    |
    [256K, 512K)           2 |                                                    |
    [512K, 1M)             1 |                                                    |
```

Statistical aggregates are [also available in SystemTap](https://sourceware.org/systemtap/SystemTap_Beginners_Guide/arrayops-aggregates.html). The `<<<` operator allows adding values to a statistical aggregate. SystemTap does not automatically print aggregation results when the program exits, so it needs to be done explicitly.

```sh
    global bytes
    probe kernel.function("vfs_read").return {
        bytes <<< $return
    }

    probe end {
        print(@hist_log(bytes))
    }
```

## Features

A very useful feature of DTrace-like systems is the ability to obtain a stack trace to see which sequence of function calls lead to a given probe point. Kernel stack traces can be obtained in bpftrace as follows:

```sh
    kprobe:icmp_echo {
        print(kstack);
        exit()
    }
```

Equivalently, with SystemTap:

```sh
    probe kernel.function("icmp_echo") {
        print_backtrace();
        exit()
    }
```

An important problem affecting bpftrace is that it cannot generate user-space stack traces unless the program being traced was built with [frame pointers](https://en.wikipedia.org/wiki/Call_stack#Stack_and_frame_pointers). For the vast majority of cases, that means that users must recompile the software under examination in order to instrument it.

SystemTap's [user-space stack backtrace mechanism](https://sourceware.org/systemtap/SystemTap_Beginners_Guide/ustack.html), instead, provides a full stack trace by making use of debug information to walk the stack. This means that no recompilation is needed.

```sh
    probe process("/bin/ls").function("format_user_or_group") {
        print_ubacktrace();
        exit()
    }
```

The script above produces a full backtrace, here shortened for readability:

```sh
     0x55767a467f60 : format_user_or_group+0x0/0xc0 [/bin/ls]
     0x55767a46d26a : print_long_format+0x58a/0x9f0 [/bin/ls]
     0x55767a46d840 : print_current_files+0x170/0x3e0 [/bin/ls]
     0x55767a465d8d : main+0x62d/0x1a00 [/bin/ls]
```

The same feature is [unlikely to be added to bpftrace](https://github.com/iovisor/bpftrace/issues/1744), as it would need to be implemented either by the kernel or in BPF bytecode.

## Real world uses

Consider the following example of a practical production investigation that could not proceed further with bpftrace due to the backtrace limitation, so SystemTap needed to be used to track it down. At Wikimedia we ran into [an interesting problem](https://github.com/apache/trafficserver/issues/7423) with [LuaJIT](https://luajit.org/) after observing high system CPU usage on behalf of [Apache Traffic Server](https://trafficserver.apache.org/), we could confirm that it was due to `mmap()` being called unusually often:

```sh
    $ sudo bpftrace -e 'kprobe:do_mmap /pid == 31316/ { @[arg2]=count(); } interval:s:1 { exit(); }'
    Attaching 2 probes...
    @[65536]: 64988
```

That is where the investigation would have stopped, had it not been possible to generate user-space backtraces with SystemTap. Note that in this case the issue affected the Lua JIT component: rebuilding Apache Traffic Server with frame pointers to make bpftrace produce a stack trace would not have been sufficient, we would have had to rebuild LuaJIT too.

Another important advantage of SystemTap over bpftrace is that it allows accessing function arguments and local variables by their name. With bpftrace, arguments can only be accessed by name when instrumenting the kernel, and specifically when using [static kernel tracepoints](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#6-tracepoint-static-tracing-kernel-level-arguments) or the [experimental kfunc feature](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#15-kfunckretfunc-kernel-functions-tracing) that is available for recent kernels. The kfunc feature is based on [BPF trampolines](https://lwn.net/Articles/804937/) and seems promising. When using regular kprobes, or when instrumenting user-space software, bpftrace can access arguments only by position (arg0, arg1, ... argN).

SystemTap is also able to list available probe points by source file, and to match by filename in the definition of probes too. The feature can be used to focus the analysis only on specific areas of the code base. For instance, the following command can be used to list (`-L`) all of the functions defined in Apache Traffic Server's `iocore/cache/Cache.cc`:

```sh
    $ stap -L 'process("/usr/bin/traffic_server").function("*@./iocore/cache/Cache.cc")
```

It is often necessary to probe a specific point somewhere in the body of a function, rather than limiting the analysis to the function entry point or to the return statement. This can be done in SystemTap using [statement probes](https://sourceware.org/systemtap/langref/langrefse4.html#x6-370004.2); the following will list the probe points available along with the variables available at each point:

```sh
    $ stap -L 'process("/bin/ls").statement("format_user_or_group@src/ls.c:*")'
    process("/bin/ls").statement("format_user_or_group@src/ls.c:4110")\
        $name:char const* $id:long unsigned int $width:int
    process("/bin/ls").statement("format_user_or_group@src/ls.c:4115")\
        $name:char const* $id:long unsigned int $width:int
    process("/bin/ls").statement("format_user_or_group@src/ls.c:4116")\
        $width_gap:int $name:char const* $id:long unsigned int $width:int	
    process("/bin/ls").statement("format_user_or_group@src/ls.c:4118")\
        $pad:int $name:char const* $id:long unsigned int $width:int
    [...]
    process("/bin/ls").statement("format_user_or_group@src/ls.c:4131")\
        $name:char const* $id:long unsigned int $width:int $len:size_t
```

The full output shows that there are ten different lines that can be probed inside the function `format_user_or_group()`, together with the various variables available in scope. By looking at the source code we can see which one exactly needs to be probed, and write the SystemTap program accordingly.

To try to achieve the same goal with bpftrace we would need to [disassemble the function and specify the right offset](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#3-uprobeuretprobe-dynamic-tracing-user-level) to the Uprobe based on the assembly instead, which is cumbersome at best. Additionally, bpftrace [needs to be explicitly built](https://github.com/iovisor/bpftrace/issues/1632) with [Binary File Descriptor](https://en.wikipedia.org/wiki/Binary_File_Descriptor_library) (BFD) support for this feature to work.

While all software is sooner or later affected by bugs, issues affecting debugging tools are particularly thorny. [One specific issue](https://github.com/iovisor/bpftrace/issues/1305) affects bpftrace on systems with certain LLVM versions, and it seems worth mentioning. Due to [an LLVM bug](https://bugs.llvm.org/show_bug.cgi?id=47591) causing load/store instructions in the intermediate representation to be reordered when they should not be, valid bpftrace scripts can [misbehave](https://github.com/iovisor/bpftrace/issues/1332) in ways that are difficult to figure out. Adding or removing unrelated code might work around or trigger the bug. The same underlying LLVM bug [causes other bpftrace scripts to fail](https://github.com/iovisor/bpftrace/issues/1304). The problem has recently been fixed in LLVM 12; bpftrace users should ensure they are running a recent LLVM version that is not affected by this issue.

## Conclusions

SystemTap and bpftrace offer similar functionality, but differ significantly in their design choices by using loadable kernel module in one case and BPF in the other. The approach based on kernel modules offers greater flexibility, and allows implementing features that are hard if not impossible to do using BPF. On the other hand, BPF is an obviously good choice for tracing tools, as it provides a fast and safe environment to base observability tools on.

For many use cases, bpftrace just works out of the box, while SystemTap generally requires installing additional dependencies in order to take full advantage of all of its features. Bpftrace is generally faster, and provides various facilities for quick aggregation and reporting that are arguably simpler to use than those provided by SystemTap. On the other hand, SystemTap provides several distinguishing features such as: generating user-space backtraces without the need for frame pointers, accessing function arguments and local variables by name, and the ability to probe arbitrary statements. Both would seem to have their place for diagnosing problems in today's Linux systems.
