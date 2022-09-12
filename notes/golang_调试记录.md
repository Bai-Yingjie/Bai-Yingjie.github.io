- [go-prompt 性能问题调查](#go-prompt-性能问题调查)
- [内存还给OS](#内存还给os)
  - [MADV_FREE和MADV_DONTNEED](#madv_free和madv_dontneed)
  - [GOGC比例](#gogc比例)
- [topid申请内存错误](#topid申请内存错误)
  - [背景](#背景)
    - [错误1](#错误1)
    - [错误2](#错误2)
    - [错误3](#错误3)
  - [调查](#调查)
    - [释放内存后运行 重启后运行 -- nok](#释放内存后运行-重启后运行----nok)
    - [golang版本1.13换到1.16问题依旧](#golang版本113换到116问题依旧)
    - [和版本相关?](#和版本相关)

# go-prompt 性能问题调查
go-prompt时一个交互式cli框架, 但似乎性能有点问题. 一个简单的cli例子程序CPU就占了10%.  
基于go-prompt的abs也一样空跑都占比10%以上.
看火焰图似乎是timer的使用问题.  
![](img/golang_调试记录_20220913000203.png)  
原因在于  
```go
func (p *Prompt) Run() {
    for {
        select {
            default:
            //有default就是非阻塞, 这里sleep 10ms
                time.Sleep(10 * time.Millisecond)
        }
    }
}
```

# 内存还给OS
## MADV_FREE和MADV_DONTNEED
go1.12版本中, gc把不用的内存还给OS时, 调用了`int madvise(void *addr, size_t length, int advice)`, 使用了`MADV_FREE`标记. `man madvise`说的很清楚, 虽然内存还给了OS, 但OS只有在系统内存不够时才回收. 导致go程序虽然认为自己已经还了内存给OS, 但其RSS一直居高不下.
> On Linux, the runtime now uses MADV_FREE to release unused memory. This is more efficient but may result in higher reported RSS. The kernel will reclaim the unused data when it is needed. To revert to the Go 1.11 behavior (MADV_DONTNEED), set the environment variable GODEBUG=madvdontneed=1.

go1.11及之前, gc用`MADV_DONTNEED`来还内存给OS, RSS会马上下降.
理论说法是, `MADV_FREE`性能更好, 但会导致RSS的值不准确. 实际上, 随着gc算法的提高, OS的`MADV_FREE` lazy回收模式, 就没有太大意义了.
go1.16版本又把`MADV_DONTNEED`的行为做为默认了.

注意: 如果用`GODEBUG=madvdontneed=1`还是不能让RSS下降, 需要搭配强制gc的方法, 否则一般情况下, gc要很长时间, 比如几分钟, 才工作.

go1.13里面有个优化提到: 之前, gc会持有内存5分钟或更长时间才开始还给OS, 但1.13会更积极的还内存.
> The runtime is now more aggressive at returning memory to the operating system to make it available to co-tenant applications. Previously, the runtime could retain memory for five or more minutes following a spike in the heap size. It will now begin returning it promptly after the heap shrinks. However, on many OSes, including Linux, the OS itself reclaims memory lazily, so process RSS will not decrease until the system is under memory pressure.

参考:
https://golang.org/pkg/runtime/ 搜索`madvdontneed`
下面就是最新的1.16版本的说法:
> madvdontneed: setting madvdontneed=0 will use MADV_FREE
instead of MADV_DONTNEED on Linux when returning memory to the
kernel. This is more efficient, but means RSS numbers will
drop only when the OS is under memory pressure.

[https://golang.org/doc/go1.12#runtime](https://golang.org/doc/go1.12#runtime "https://golang.org/doc/go1.12#runtime")
https://vec.io/posts/golang-and-memory

## GOGC比例
GOGC默认是100, 即新分配的内存和上次gc完成时剩下的内存的比例是1:1时, gc才启动. 也就是说, 只有内存翻了一倍时, gc才开始收集垃圾.
想控制RSS占用的话, GOGC设置小一点.

# topid申请内存错误
## 背景
某次版本后, topid启动马上出错:
每次栈还不一样, 但最后都是`runtime.mallocgc`的栈
### 错误1
```sh
~ # ./topid -tag ZAPPING_MCASTV4_v2 -p 1 -tree
Hello 你好 Hola Hallo Bonjour Ciao Χαίρετε こんにちは 여보세요
Version: 0.1.4
runtime: s.allocCount= 3 s.nelems= 5
fatal error: s.allocCount != s.nelems && freeIndex == s.nelems

goroutine 1 [running]:
runtime.throw(0x1a8caa, 0x31)
        runtime/panic.go:774 +0x54 fp=0x40000a0ca0 sp=0x40000a0c70 pc=0x3bc84
runtime.(*mcache).nextFree(0x7f82a2a6d0, 0xa2f47, 0x4000000180, 0x200000003, 0x4000000180)
        runtime/malloc.go:852 +0x204 fp=0x40000a0cf0 sp=0x40000a0ca0 pc=0x1aa44
runtime.mallocgc(0x600, 0x15d600, 0x1, 0x0)
        runtime/malloc.go:1022 +0x688 fp=0x40000a0db0 sp=0x40000a0cf0 pc=0x1b0e8
runtime.makeslice(0x15d600, 0x600, 0x600, 0x0)
        runtime/slice.go:49 +0x74 fp=0x40000a0de0 sp=0x40000a0db0 pc=0x51364
bytes.makeSlice(0x600, 0x0, 0x0, 0x0)
        bytes/buffer.go:229 +0x64 fp=0x40000a0e50 sp=0x40000a0de0 pc=0x7af84
bytes.(*Buffer).grow(0x40000a0f80, 0x200, 0x200)
        bytes/buffer.go:142 +0x12c fp=0x40000a0ea0 sp=0x40000a0e50 pc=0x7aabc
bytes.(*Buffer).ReadFrom(0x40000a0f80, 0x1d39e0, 0x4000081268, 0x3, 0x0, 0x0)
        bytes/buffer.go:202 +0x48 fp=0x40000a0f10 sp=0x40000a0ea0 pc=0x7ada8
io/ioutil.readAll(0x1d39e0, 0x4000081268, 0x200, 0x0, 0x0, 0x0, 0x0, 0x0)
        io/ioutil/ioutil.go:36 +0xb0 fp=0x40000a0fb0 sp=0x40000a0f10 pc=0x12bb80
io/ioutil.ReadAll(...)
        io/ioutil/ioutil.go:45
pidinfo.(*TidInfo).updateStat(0x40001830e0, 0x0, 0x0)
        pidinfo/pidinfo.go:486 +0x190 fp=0x40000a10f0 sp=0x40000a0fb0 pc=0x12d510
pidinfo.(*PidInfo).update(0x40001830e0, 0x40001472a0, 0x0)
        pidinfo/pidinfo.go:799 +0x2c fp=0x40000a1190 sp=0x40000a10f0 pc=0x13017c
pidinfo.newPidInfo(0x28e, 0x40000bee10, 0x28e, 0x2f4f20, 0x0)
        pidinfo/pidinfo.go:631 +0x2f8 fp=0x40000a1280 sp=0x40000a1190 pc=0x12e478
```

### 错误2
```sh
~ # Hello 你好 Hola Hallo Bonjour Ciao Χαίρετε こんにちは 여보세요
Version: 0.1.4
Visit below URL to get the chart:
http://10.182.105.138:9888/ZAPPING_MCASTV4_v2/1992780262
runtime: s.allocCount= 3 s.nelems= 5
fatal error: s.allocCount != s.nelems && freeIndex == s.nelems

goroutine 1 [running]:
runtime.throw(0x1a8caa, 0x31)
        runtime/panic.go:774 +0x54 fp=0x40000a0b00 sp=0x40000a0ad0 pc=0x3bc84
runtime.(*mcache).nextFree(0x7f887a26d0, 0xa2f47, 0x4000000180, 0x200000003, 0x4000000180)
        runtime/malloc.go:852 +0x204 fp=0x40000a0b50 sp=0x40000a0b00 pc=0x1aa44
runtime.mallocgc(0x600, 0x15d600, 0x1, 0x0)
        runtime/malloc.go:1022 +0x688 fp=0x40000a0c10 sp=0x40000a0b50 pc=0x1b0e8
runtime.makeslice(0x15d600, 0x600, 0x600, 0x0)
        runtime/slice.go:49 +0x74 fp=0x40000a0c40 sp=0x40000a0c10 pc=0x51364
bytes.makeSlice(0x600, 0x0, 0x0, 0x0)
        bytes/buffer.go:229 +0x64 fp=0x40000a0cb0 sp=0x40000a0c40 pc=0x7af84
bytes.(*Buffer).grow(0x40000a0de0, 0x200, 0x200)
        bytes/buffer.go:142 +0x12c fp=0x40000a0d00 sp=0x40000a0cb0 pc=0x7aabc
bytes.(*Buffer).ReadFrom(0x40000a0de0, 0x1d39e0, 0x40000851b8, 0xb7bc4, 0x40001354e0, 0x12)
        bytes/buffer.go:202 +0x48 fp=0x40000a0d70 sp=0x40000a0d00 pc=0x7ada8
io/ioutil.readAll(0x1d39e0, 0x40000851b8, 0x200, 0x0, 0x0, 0x0, 0x0, 0x0)
        io/ioutil/ioutil.go:36 +0xb0 fp=0x40000a0e10 sp=0x40000a0d70 pc=0x12bb80
io/ioutil.ReadFile(0x40001354e0, 0x12, 0x0, 0x0, 0x0, 0x0, 0x0)
        io/ioutil/ioutil.go:73 +0xc4 fp=0x40000a0eb0 sp=0x40000a0e10 pc=0x12bcf4
pidinfo.(*PidInfo).init(0x40001a84b0, 0xa, 0x1d6860)

```

### 错误3
```sh
./topid -tag ZAPPING_MCASTV4_v2 -p 1 -chartserver 10.182.105.138:9887 -i 3 -c 3600 -record -sys -child &
~ # Hello \xe4\xbd\xa0\xe5\xa5\xbd Hola Hallo Bonjour Ciao \xce\xa7\xce\xb1\xce\xaf\xcf\x81\xce\xb5\xcf\x84\xce\xb5 \xe3\x81\x93\xe3\x82\x93\xe3\x81\xab\xe3\x81\xa1\xe3\x81\xaf \xec\x97\xac\xeb\xb3\xb4\xec\x84\xb8\xec\x9a\x94
Version: 0.1.4
Visit below URL to get the chart:
http://10.182.105.138:9888/ZAPPING_MCASTV4_v2/2938934727
runtime: s.allocCount= 492 s.nelems= 512
fatal error: s.allocCount != s.nelems && freeIndex == s.nelems

goroutine 1 [running]:
runtime.throw(0x1a8caa, 0x31)
    runtime/panic.go:774 +0x54 fp=0x4000040b50 sp=0x4000040b20 pc=0x3bc84
runtime.(*mcache).nextFree(0x7f9c7ab6d0, 0x51305, 0x4000040c08, 0x38e8c, 0x4000040c18)
    runtime/malloc.go:852 +0x204 fp=0x4000040ba0 sp=0x4000040b50 pc=0x1aa44
runtime.mallocgc(0x8, 0x15d400, 0x0, 0x0)
    runtime/malloc.go:998 +0x4f4 fp=0x4000040c60 sp=0x4000040ba0 pc=0x1af54
runtime.convT64(0x1, 0x7f9a53ef28)
    runtime/iface.go:352 +0x5c fp=0x4000040c90 sp=0x4000040c60 pc=0x18d0c
internal/poll.errnoErr(...)
    internal/poll/errno_unix.go:32
internal/poll.(*pollDesc).init(0x4000219338, 0x4000219320, 0x1, 0x4000219320)
    internal/poll/fd_poll_runtime.go:45 +0x8c fp=0x4000040cd0 sp=0x4000040c90 pc=0xb312c
internal/poll.(*FD).Init(0x4000219320, 0x19fa28, 0x4, 0x1, 0x40001fe800, 0x0)
    internal/poll/fd_unix.go:63 +0x68 fp=0x4000040d00 sp=0x4000040cd0 pc=0xb3888
os.newFile(0x5, 0x40001fe7e0, 0x12, 0x1, 0x4000000000)
    os/file_unix.go:151 +0xe8 fp=0x4000040d60 sp=0x4000040d00 pc=0xb6ed8
os.openFileNolog(0x40001fe7e0, 0x12, 0x0, 0x0, 0x8, 0x12, 0x40001fe7e0)
    os/file_unix.go:222 +0x19c fp=0x4000040dc0 sp=0x4000040d60 pc=0xb715c
os.OpenFile(0x40001fe7e0, 0x12, 0x0, 0x0, 0x2, 0x40001fe7e0, 0x40001fe7ea)
    os/file.go:300 +0x54 fp=0x4000040e10 sp=0x4000040dc0 pc=0xb6964
os.Open(...)
    os/file.go:280
io/ioutil.ReadFile(0x40001fe7e0, 0x12, 0x0, 0x0, 0x0, 0x0, 0x0)
    io/ioutil/ioutil.go:53 +0x48 fp=0x4000040eb0 sp=0x4000040e10 pc=0x12bc78
pidinfo.(*PidInfo).init(0x400020d1d0, 0xa, 0x1d6860)
    pidinfo/pidinfo.go:587 +0x130 fp=0x4000040f50 sp=0x4000040eb0 pc=0x12dfe0
```

## 调查
### 释放内存后运行 重启后运行 -- nok
看起来是内存申请失败, 那么先系统内存的情况:
```sh
~ # free -m
              total        used        free      shared  buff/cache   available
Mem:            853         533           8         115         312         179
Swap:             0           0           0
```
强制释放内存试一下:
```sh
echo 3 > /proc/sys/vm/drop_caches
```
再看一下内存:
```sh
~ # free -m
              total        used        free      shared  buff/cache   available
Mem:            853         537         103         115         212         175
Swap:             0           0           0
```
有103M了, topid通常是占用RSS 6M, 那么100M应该是够了.
但运行还是一样的`runtime.mallocgc`之后`runtime.(*mcache).nextFree`错误.

重启板子再运行, 也是出错.

### golang版本1.13换到1.16问题依旧
注意1.16默认使用go mod, 需要手动off掉
```sh
export GOPATH=`pwd`
GOARCH=arm64 GO111MODULE=off go build topid.go
```

### 和版本相关?
288版本是好的, 289版本就不行了..
kernel改了什么?

