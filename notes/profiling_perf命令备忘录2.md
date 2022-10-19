- [命令记录](#命令记录)
- [统计系统调用次数](#统计系统调用次数)
- [添加符号表](#添加符号表)
  - [背景](#背景)
  - [MIPS板子解析用户态调用栈要用`-g --call-graph dwarf`](#mips板子解析用户态调用栈要用-g---call-graph-dwarf)
  - [给`perf script`添加符号表](#给perf-script添加符号表)
  - [拷贝script后的文件到PC机, 生成火焰图](#拷贝script后的文件到pc机-生成火焰图)

# 命令记录
```sh
#使用perf的filter功能. 详见 调试和分析记录 -- perf使用实例
perf record -e syscalls:sys_enter_write -aR -g --call-graph dwarf --filter 'fd == 2' -- sleep 30
```

# 统计系统调用次数
perf stat就是统计
```sh
perf stat --log-fd 1 -e 'syscalls:sys_enter_*' -p 5464 -- sleep 10 | egrep -v "[[:space:]]+0"
```

# 添加符号表
## 背景
在板子上运行的`perf record`, 采样并记录程序运行时的PC地址. 这时是不需要符号表的.

在用`perf script`解析时, 要用符号表.

如果当时运行的app被strip过了, 是没有符号表的.  
没有符号表强行解析, 结果是不准的. 

## MIPS板子解析用户态调用栈要用`-g --call-graph dwarf`
```sh
#对pid为18852 记录60秒, 频率是1000HZ, 也就是1ms一次采样
#cycles:u表示只对用户态采样, 也可以用cycles:k只对内核态采样
perf record -F 1000 -e cycles:u -g --call-graph dwarf -p 18852 -- sleep 60
```

## 给`perf script`添加符号表
用`--symfs`指定个目录, `perf script`会当这个目录为`/`来找符号表
```sh
#record命令生成perf.data, 是perf script默认的data文件
#要成功运行下面的命令, 要把没有被strip的app, 按照根目录的相对位置, 放到--symfs下面
#我这里用buildroot的output/staging目录
sshfs wsfs user@ip:/buildroot_poc/output/host/mips64-buildroot-linux-gnu/sysroot
#可以先用这个命令找到需要哪些符号表
perf script | grep unknown | sort -u
#因为我们用了--symfs, 它就不在默认的路径下找符号表了
#包括kernel的符号表也就找不到了; 所以要用--kallsyms 配合使用
perf script --symfs wsfs --kallsyms /proc/kallsyms > bsfs/sysidle-switch_hwa_app/perf-u.script
```

## 拷贝script后的文件到PC机, 生成火焰图
```sh
cat perf.script | ~/repo/FlameGraph/stackcollapse-perf.pl | ~/repo/FlameGraph/flamegraph.pl > sample.svg
```
