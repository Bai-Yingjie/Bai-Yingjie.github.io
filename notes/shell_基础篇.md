- [shell脚本块注释](#shell脚本块注释)
- [带超时的重试命令](#带超时的重试命令)
- [Here Document的用法: 一行输入](#here-document的用法-一行输入)
- [if判断和\&\&的区别](#if判断和的区别)
- [bash的双中括号](#bash的双中括号)
  - [举例](#举例)
- [date举例](#date举例)
- [网络脚本直接执行](#网络脚本直接执行)
- [ulimit和prlimit](#ulimit和prlimit)
- [curl下载](#curl下载)
- [暂停和继续一个进程的执行, 不用gdb](#暂停和继续一个进程的执行-不用gdb)
  - [补充发送其他信号的行为](#补充发送其他信号的行为)
- [查看一个进程的子进程](#查看一个进程的子进程)
- [跳过前几行](#跳过前几行)
- [重定向前面不能有空格](#重定向前面不能有空格)
- [shell单引号和双引号的重要区别](#shell单引号和双引号的重要区别)
- [保存多行输出到shell变量](#保存多行输出到shell变量)
- [pstree pgrep and ps](#pstree-pgrep-and-ps)
- [rsync 实例](#rsync-实例)
- [sar 汇总](#sar-汇总)
- [从变量read](#从变量read)
- [out put htop](#out-put-htop)
- [为什么\&后面加分号不行?](#为什么后面加分号不行)
- [sort按多列排序](#sort按多列排序)
- [shell注释一块代码](#shell注释一块代码)
- [默认变量值](#默认变量值)
- [mtr 一个更好用的traceroute](#mtr-一个更好用的traceroute)
- [ssh config 快捷登录](#ssh-config-快捷登录)
- [给log文件加时间戳](#给log文件加时间戳)
- [cp最好加-a](#cp最好加-a)
- [如何查丢包](#如何查丢包)
- [sar 查看block io的使用情况](#sar-查看block-io的使用情况)
- [telnet记录log](#telnet记录log)
- [kill某用户所有进程](#kill某用户所有进程)
- [UUID和GUID](#uuid和guid)
- [关于grub](#关于grub)
- [使用netcat导出打印](#使用netcat导出打印)
- [查看CPU运行队列情况](#查看cpu运行队列情况)
- [网络流量监控](#网络流量监控)
- [查看uuid](#查看uuid)
- [为什么静态ip会丢失](#为什么静态ip会丢失)
- [用tail查看系统log](#用tail查看系统log)
- [代码打补丁](#代码打补丁)
- [如何查看io占用高的进程](#如何查看io占用高的进程)
- [linux访问win共享](#linux访问win共享)
- [ubuntu杀进程和服务](#ubuntu杀进程和服务)
- [去掉最后几个字符](#去掉最后几个字符)
- [在shell里用exec eval source区别](#在shell里用exec-eval-source区别)
- [查看线程, 大写H](#查看线程-大写h)
- [linux启动脚本顺序](#linux启动脚本顺序)
- [文件权限](#文件权限)
- [SELinux的文件权限扩展](#selinux的文件权限扩展)
- [shell for](#shell-for)
- [观察82599的中断](#观察82599的中断)
- [关于中断balance](#关于中断balance)
- [多个jpg图片转到pdf](#多个jpg图片转到pdf)
- [合并多个pdf](#合并多个pdf)
- [ssh X11及VNC](#ssh-x11及vnc)
  - [firefox over ssh x11 forwarding](#firefox-over-ssh-x11-forwarding)
  - [让远程的firefox更快](#让远程的firefox更快)
- [tools](#tools)
  - [change tmux prefix](#change-tmux-prefix)
  - [tmux复制](#tmux复制)
- [network](#network)
  - [mount fae storage](#mount-fae-storage)
- [新建一个用户到特定组](#新建一个用户到特定组)
- [查看虚拟分区](#查看虚拟分区)
- [thunder通过服务器上网](#thunder通过服务器上网)
- [linux路由参考](#linux路由参考)
- [brctl就是个交换机](#brctl就是个交换机)
- [wiz的搜索功能](#wiz的搜索功能)
- [批量修改jpg图片大小--xarg位置指代](#批量修改jpg图片大小--xarg位置指代)
- [ubuntu查看一个文件属于哪个package](#ubuntu查看一个文件属于哪个package)
- [ubuntu查看一个package包含哪些文件](#ubuntu查看一个package包含哪些文件)
- [egrep 分组或](#egrep-分组或)
- [删除mysql的cmake相关的文件](#删除mysql的cmake相关的文件)
- [cpu hotplug](#cpu-hotplug)
- [grub reserve 内存](#grub-reserve-内存)
- [给用户组加写权限](#给用户组加写权限)
- [修改文件所属用户](#修改文件所属用户)
- [增加共享库路经](#增加共享库路经)
- [查看端口占用](#查看端口占用)
- [查找大于1M的文件](#查找大于1m的文件)
- [wget下载](#wget下载)
- [通过gcc查找libc路径 -- -print-file-name=xxx](#通过gcc查找libc路径-----print-file-namexxx)
- [各种进制转十进制](#各种进制转十进制)
- [bc --shell下的C计算器](#bc---shell下的c计算器)
- [CPU热插拔](#cpu热插拔)
- [hexdump查ascii码 -v选项](#hexdump查ascii码--v选项)
- [hexdump指定格式](#hexdump指定格式)
- [查看温感](#查看温感)
- [trap命令 --指定shell中处理signal](#trap命令---指定shell中处理signal)
- [创建ram文件系统](#创建ram文件系统)
- [heredoc格式代码](#heredoc格式代码)
- [install命令](#install命令)
- [readlink --读取链接文件](#readlink---读取链接文件)
- [重新修改tmpfs的大小](#重新修改tmpfs的大小)
- [fg bg --前台运行 后台运行](#fg-bg---前台运行-后台运行)
- [shell写字符串, 以'\\0'结尾](#shell写字符串-以0结尾)
- [写十六进制数据到文件, 关键在于引号](#写十六进制数据到文件-关键在于引号)
- [批量处理软链接之cscope.files](#批量处理软链接之cscopefiles)
- [readelf结果用sort排序, 按照第三列数字排序](#readelf结果用sort排序-按照第三列数字排序)
- [批量删除ZF\_ ZL\_ ZG](#批量删除zf_-zl_-zg)
- [批量格式化代码](#批量格式化代码)
- [带时间的平均负载](#带时间的平均负载)
- [rpm解压](#rpm解压)
- [大小写转换](#大小写转换)
- [解析C文件全部函数到h文件声明, 用于偷懒声明所有函数到头文件](#解析c文件全部函数到h文件声明-用于偷懒声明所有函数到头文件)
- [strace 重定向](#strace-重定向)
- [通过elf生成工程文件列表 --利用gdb](#通过elf生成工程文件列表---利用gdb)
- [sync命令使用, 把file buffer 刷进物理器件](#sync命令使用-把file-buffer-刷进物理器件)
- [查找src下面的代码目录, find可以限制搜索深度](#查找src下面的代码目录-find可以限制搜索深度)
- [查看当前文件夹大小](#查看当前文件夹大小)
- [删除文件夹下面的所有链接文件](#删除文件夹下面的所有链接文件)
- [压缩当前目录下sw\_c文件夹](#压缩当前目录下sw_c文件夹)
- [根据一个文本文件列表压缩 --hT](#根据一个文本文件列表压缩---ht)
- [tar.xz格式压缩解压](#tarxz格式压缩解压)
- [在当前路径下的所有文件中，搜索关键字符串](#在当前路径下的所有文件中搜索关键字符串)
- [ls按修改时间排序](#ls按修改时间排序)
- [ls按大小排序](#ls按大小排序)
- [查看键盘映射](#查看键盘映射)
- [限制深度的find](#限制深度的find)
- [查看系统支持的文件系统类型](#查看系统支持的文件系统类型)
- [替换字符串](#替换字符串)
- [批量软链接](#批量软链接)
- [tee](#tee)
- [处理软链接, Thomas版](#处理软链接-thomas版)
- [关闭printk打印](#关闭printk打印)
- [usb串口 --lsusb --minicom](#usb串口---lsusb---minicom)
- [locate--linux下面的everything](#locate--linux下面的everything)
- [rsync拷贝目录，去除hg](#rsync拷贝目录去除hg)

# shell脚本块注释
```
#!/bin/bash
echo before comment
: <<'END'
bla bla
blurfl
END
echo after comment
```

# 带超时的重试命令
用timeout和until的组合:  
比如下面的dockerfile里面的RUN语句, code-server可能会在装extension的时候卡住, 用timeout来强制2m退出, 并用until循环来重试.
```sh
RUN until timeout 2m code-server \
    --user-data-dir /usr/local/share/code-server \
    --install-extension golang.go \
    --install-extension ms-python.python \
    --install-extension formulahendry.code-runner \
    --install-extension eamodio.gitlens \
    --install-extension oderwat.indent-rainbow \
    --install-extension vscode-icons-team.vscode-icons \
    --install-extension esbenp.prettier-vscode \
    --install-extension streetsidesoftware.code-spell-checker \
    ;do echo Retry installing extensions...; done
```

# Here Document的用法: 一行输入
用`<<< 输入`可以达到`bash -c`一样的效果
```sh
$ bash -c "echo hello"
hello

$ bash <<< "echo hello"
hello
```
令一个例子是
```sh
$ ./gshell <<< 'fmt:=import("fmt"); fmt.println("hello")'
hello
```
具体见`man bash`, 搜索`Here Strings`

# if判断和&&的区别
返回值不同  
比如
```sh
if true; then echo 111; fi
#结果为 111
echo $?
#返回值为 0

if false; then echo 111; fi
#没有输出, 说明if条件不成立
echo $?
#返回值还是 0, 说明shell认为没有错误
```

而`&&`就不一样
```sh
false
echo $?
#直接返回 1

false && echo 111
#没有输出, 说明echo没有执行; 到这里和if的效果是一样的
echo $?
#但返回值为 1

true && echo 111
#打印111
echo $?
#返回值为 0
```

结论: 
* if和`&&`都有根据条件控制执行的功能
* if语句块结束后, 永远返回0值, 表示成功; 而`&&`返回值由最后一个语句决定, 可能是0, 也可能是1. 
* 所以在某些脚本中, 如果最后的返回值重要, 就要考虑用哪种判断方式.

# bash的双中括号
help里面说的很清楚, `[[ expression ]]`的语义更明确, 
```sh
help [[
[[ ... ]]: [[ expression ]]
是test的扩展语法, 支持
( EXPRESSION )
! EXPRESSION
EXPR1 && EXPR2
EXPR1 || EXPR2
```
特别的, 当使用`==`或`!=`时, 右侧的string会被当作pattern来匹配; 这个匹配和`case in`的语义一样
当使用`=~`时, 右侧的string是正则表达式.

## 举例
```sh
#!/bin/bash

# Only continue for 'develop' or 'release/*' branches
BRANCH_REGEX="^(develop$|release//*)"

if [[ $BRANCH =~ $BRANCH_REGEX ]];
then
    echo "BRANCH '$BRANCH' matches BRANCH_REGEX '$BRANCH_REGEX'"
else
    echo "BRANCH '$BRANCH' DOES NOT MATCH BRANCH_REGEX '$BRANCH_REGEX'"
fi
```
```sh
[[ $TEST =~ ^[[:alnum:][:blank:][:punct:]]+$ ]]
```

参考: https://stackoverflow.com/questions/18709962/regex-matching-in-a-bash-if-statement/18710850

# date举例
```sh
#给文件名用
$ date +"%Y%m%d%H%M%S"
20201020115510

#给log时间戳用
date +"%F %H:%M:%S.%N"
```

# 网络脚本直接执行
```sh
curl -s http://10.182.105.179:8088/godevtools/godevtool | bash -s -h
curl -s http://10.182.105.179:8088/godevtools/godevtool | bash -s vscode start
```
* curl下载文件, 输出到stdout; 
    * -s是quiet模式
    * 如果加-O就是保存同名文件到本地
* bash -s可以加参数 [如何通过pipe给bash传参数](https://stackoverflow.com/questions/4642915/passing-parameters-to-bash-when-executing-a-script-fetched-by-curl)

```sh
-s        If the -s option is present, or if no arguments remain after option processing, then commands are read from the standard input.  This option  allows  the  positional parameters to be set when invoking an interactive shell.
```

# ulimit和prlimit
ulimit可以配置当前shell的资源限制, 一般用:
```sh
ulimit -n 限制打开文件的个数
```

prlimit是个命令, 对应同名的系统调用, 可以动态配置一个pid的资源
```sh
prlimit --pid 13134 --rss --nofile=1024:4095
    Display the limits of the RSS, and set the soft and hard limits for the number of open files to 1024 and 4095, respectively.
    
prlimit --pid $$ --nproc=unlimited
    Set for the current process both the soft and ceiling values for the number of processes to unlimited.

prlimit --cpu=10 sort -u hugefile
    Set both the soft and hard CPU time limit to ten seconds and run 'sort'.
```
显示当前进程(即执行prlimit进程的进程)的limit
```sh
$ prlimit --pid $$
RESOURCE   DESCRIPTION                             SOFT      HARD UNITS
AS         address space limit                unlimited unlimited bytes
CORE       max core file size                         0 unlimited bytes
CPU        CPU time                           unlimited unlimited seconds
DATA       max data size                      unlimited unlimited bytes
FSIZE      max file size                      unlimited unlimited bytes
LOCKS      max number of file locks held      unlimited unlimited locks
MEMLOCK    max locked-in-memory address space  16777216  16777216 bytes
MSGQUEUE   max bytes in POSIX mqueues            819200    819200 bytes
NICE       max nice prio allowed to raise             0         0
NOFILE     max number of open files                1024   1048576 files
NPROC      max number of processes               289710    289710 processes
RSS        max resident set size              unlimited unlimited bytes
RTPRIO     max real-time priority                     0         0
RTTIME     timeout for real-time tasks        unlimited unlimited microsecs
SIGPENDING max number of pending signals         289710    289710 signals
STACK      max stack size                       8388608 unlimited bytes
```

# curl下载
```sh
curl -o go.tar.gz -L https://dl.google.com/go/go$GOLANG_VERSION.linux-amd64.tar.gz

# -L的意思是让curl follow redirection
```

# 暂停和继续一个进程的执行, 不用gdb
我在前台跑了topid程序
```sh
#前台运行
./topid

#其他窗口执行

#暂停进程
kill -SIGSTOP `pidof topid`
#效果和ctrl+z一样, 原窗口会打印
#[1]+ Stopped ./topid

#继续执行
kill -SIGCONT `pidof topid`

#最后的效果是程序转入后台执行
```


## 补充发送其他信号的行为
```sh
kill -SIGTRAP `pidof topid` : go程序退出, 打印调用栈
kill -SIGQUIT `pidof topid` : 同上
kill -SIGSEGV `pidof topid` : 同上, 就像程序本身segment fault一样
kill -SIGTERM `pidof topid` : 没有打印, 直接退出
```

# 查看一个进程的子进程
```sh
cat /proc/pid/task/tid/children
```

# 跳过前几行
```sh
#这里的tail -n +2就是跳过n-1行, 也就是跳过1行
(echo time Goroutine Thread numGC heapSys heapIdle heapInuse heapReleased && tail -n +2 vonuerr.log | awk '{printf("%s %s %s %s %s %s %s %s\n", $3,$5,$7,$9,$13,$17,$19,$21)}') > go.csv
```

# 重定向前面不能有空格
重定向前面有空格不行:
`find / -type d -name usr 2 > /dev/null`
错误提示`find: paths must precede expression: 2`

下面的可以:
`find / -type d -name usr 2>/dev/null`
`find / -type d -name usr 2> /dev/null`

# shell单引号和双引号的重要区别
双引号展开, 单引号不展开
```sh
#单引号
cmd='echo date: $(date +%s%6N)'
#执行有变化
$ eval $cmd
date: 1539248827389418
bai@CentOS-43 ~/repo/save
$ eval $cmd
date: 1539248829149500

#双引号
cmd="echo date: $(date +%s%6N)"
#执行无变化
$ eval $cmd
date: 1539248841366200
bai@CentOS-43 ~/repo/save
$ eval $cmd
date: 1539248841366200
bai@CentOS-43 ~/repo/save
$ eval $cmd
date: 1539248841366200
bai@CentOS-43 ~/repo/save

#结论: 双引号有展开属性, 这里在赋值的时候就把date算好了
#赋给cmd时date值已经固定了, 所以双引号无变化
#单引号不会展开
```

# 保存多行输出到shell变量
```sh
#多行的输出可以保存到变量
v1=$(ethtool -S enP5p1s0 | grep packets | grep -v ": 0" && echo date: $(date +%s%6N))
#但在使用的时候, 要加""来保存换行
echo "$v1"
#不加"",换行会变成空格
echo $v1
```

# pstree pgrep and ps
```sh
$ pstree `pgrep qemu`
qemu-system-aar───9*[{qemu-system-aar}]
$ ps -eLo tid,pid,ppid,psr,stat,%cpu,rss,cmd --sort=-%cpu
```

# rsync 实例
```sh
#第一个是用ssh传输
rsync -av --progress -e ssh DATA white@192.168.2.3:/var/services/homes/white/shl
rsync -av --progress  white@192.168.2.3:/var/services/homes/white/shl
```

# sar 汇总
`watch -dn1 'sar -Bwqr -dp -n DEV -u ALL 1 1 | grep Average'`
```sh
sar -Bwqr -dp -n DEV -u ALL 1

sar -qr -dp -n DEV --human -u ALL -P ALL 1 1
```
* -B: 页表
* -w: task
* -q:CPU task q
* -r: mem
* -dp: block IO
* -n DEV: network
* -u: CPU
* -P:cpu_list | ALL
后面是interval和count

# 从变量read
```sh
# Split the TARGET variable into three elements separated by hyphens
IFS=- read -r NAME ARCH SUFFIX <<< "${TARGET}"
```
# out put htop
```sh
yingjieb@yingjieb-gv ~/tmp
$ echo q | htop | aha --black --line-fix >> htop.html
```

# 为什么&后面加分号不行?
```sh
$ for i in {1..10};do echo $i &; done
-bash: syntax error near unexpected token `;'
```
不加分号是可以的
```sh
$ for i in {1..10};do echo $i & done
```
因为&符号表示前面的命令后台执行, 本身就隐含了分隔符的作用. 而此时再加一个分隔符, 但shell认为前面并没有命令, 所以报错.

# sort按多列排序
```sh
#先按第三列排序, 相同再按第二列排序, 最后按第一列
sort -k3,3 -k2,2 -k1
```

# shell注释一块代码
```sh
#!/bin/bash
echo before comment
: <<'END'
bla bla
blurfl
END
echo after comment
```
* 一个冒号是个空命令, 什么都不干. 
> The ' and ' around the END delimiter are important, otherwise things inside the block like for example $(command) will be parsed and executed.
> That said -- it's running a command (:) that doesn't read its input and always exits with a successful value, and sending the "comment" as input.

# 默认变量值
关键是:-
```sh
interface="$1"
interrupt_name=${interface:-"eth3"}"-TxRx-"
```
如果interface不为空, 就用interface, 否则用默认的eth3

也可以从位置参数获取变量值, 如果用户没有传入位置参数, 则使用默认值.比如:
```sh
work_dir=`pwd`
dev=${1:-/dev/sdc}
ubuntu_size=${2:-500G}
tar_fedora=${3:-fedora-with-native-kernel-repo-gcc-update-to-20150423.tar.bz2}
```

# mtr 一个更好用的traceroute
```sh
mtr -n baidu.com
```

# ssh config 快捷登录
```sh
man ssh_config

cat ~/.ssh/config

Host aliclient
    Hostname 192.168.10.2
    User root
    IdentityFile ~/.ssh/id_rsa_alibaba_bench
    ServerAliveInterval 60

Host aliserver
    Hostname 30.2.47.247
    User root
    IdentityFile ~/.ssh/id_rsa_alibaba_bench
    ProxyCommand ssh -q -W %h:%p aliclient
    ServerAliveInterval 60
```
注: ProxyCommand 这句可神奇了, 它可以让ssh先登录到一台中转机器上, 然后再登录到目标机器

# 给log文件加时间戳
用ts命令
```sh
apt install moreutils
$ ping www.baidu.com | ts
Dec 24 12:59:04 PING www.a.shifen.com (111.13.100.91) 56(84) bytes of data.
Dec 24 12:59:04 64 bytes from 111.13.100.91: icmp_seq=1 ttl=54 time=52.2 ms
Dec 24 12:59:05 64 bytes from 111.13.100.91: icmp_seq=2 ttl=54 time=40.2 ms
Dec 24 12:59:06 64 bytes from 111.13.100.91: icmp_seq=3 ttl=54 time=53.1 ms
Dec 24 12:59:07 64 bytes from 111.13.100.91: icmp_seq=4 ttl=54 time=47.8 ms
Dec 24 12:59:09 64 bytes from 111.13.100.91: icmp_seq=6 ttl=54 time=39.3 ms
Dec 24 12:59:10 64 bytes from 111.13.100.91: icmp_seq=7 ttl=54 time=37.1 ms
Dec 24 12:59:11 64 bytes from 111.13.100.91: icmp_seq=8 ttl=54 time=36.1 ms
Dec 24 12:59:12 64 bytes from 111.13.100.91: icmp_seq=9 ttl=54 time=51.8 ms
Dec 24 12:59:13 64 bytes from 111.13.100.91: icmp_seq=10 ttl=54 time=54.2 ms
```

# cp最好加-a
比如当前目录下, *.lua都是链接文件
普通cp会follow这个链接, 拷真正的文件
```sh
cp *.lua /tmp/tmp/
```

-a选项会保留软链接
```sh
cp -a *.lua /tmp/tmp/
```

# 如何查丢包
接口侧
```sh
$ sudo ethtool -S wlan0
```
网络测
```sh
$ netstat -i
```

# sar 查看block io的使用情况
```sh
$ sar -dp 1
Linux 3.16.2-031602-generic (mint) 11/03/2015 _x86_64_    (4 CPU)

09:20:27 PM DEV tps rd_sec/s wr_sec/s avgrq-sz avgqu-sz await svctm %util
09:20:28 PM sda 1.00 0.00 8.00 8.00 0.03 76.00 32.00 3.20
09:20:28 PM sdb 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00

09:20:28 PM DEV tps rd_sec/s wr_sec/s avgrq-sz avgqu-sz await svctm %util
09:20:29 PM sda 1.00 0.00 240.00 240.00 0.00 4.00 4.00 0.40
09:20:29 PM sdb 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00
```
-d: 查看block dev, 一般-dp一起用, 可以看到具体的sda/sdb名称

# telnet记录log
```sh
telnet 192.168.1.31 10001 | tee crb2s_reboot_at_os.log
```

# kill某用户所有进程
```sh
[root@cavium tmp]# killall -9 -u james
[root@cavium tmp]# killall -9 -u guest
```

# UUID和GUID
GUID是微软对UUID的一种实现
如何查看UUID

注意, 用dd命令clone的盘有着一样的UUID, 由此可见, UUID是保存在分区的某个地方--据说是超级块里.
```sh
ls -l /dev/disk/by-uuid/ 
$ sudo blkid /dev/sdb1
/dev/sdb1: UUID="d039c104-49d4-464a-b3df-a962574fd46f" TYPE="ext4"
```
在Grub中的应用
```sh
title Ubuntu hardy (development branch), kernel 2.6.24-16-generic
root (hd2,0)
kernel /boot/vmlinuz-2.6.24-16-generic root=UUID=c73a37c8-ef7f-40e4-b9de-8b2f81038441 ro quiet splash
initrd /boot/initrd.img-2.6.24-16-generic
quiet
```

# 关于grub
```sh
设dtb
devicetree
设kernel
linux
```

# 使用netcat导出打印
前提是两台机器的ip能连上
服务端:
```sh
dmesg | nc -l 1234
```
客户端:
```sh
nc 127.0.0.1 1234
```
缺点是在客户端输入的东西好像在服务端解析的不好.

# 查看CPU运行队列情况
```sh
sar -q 1
07:12:09 PM   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
07:12:10 PM         0       557      0.02      0.04      0.05         0
07:12:11 PM         0       557      0.02      0.04      0.05         0
07:12:12 PM         0       557      0.02      0.04      0.05         0
07:12:13 PM         0       557      0.02      0.04      0.05         0
07:12:14 PM         0       557      0.01      0.04      0.05         0
07:12:15 PM         0       557      0.01      0.04      0.05         0
07:12:16 PM         0       557      0.01      0.04      0.05         0
07:12:17 PM         0       557      0.01      0.04      0.05         0
```
其中runq-sz就是等待队列的长度
```sh
mpstat -A
```
可以报告更多详细的CPU使用率等信息

# 网络流量监控
```sh
sar -n DEV 1
```

这里的-n选项指的是network, 里面有很多子项
DEV, EDEV, NFS, NFSD, SOCK, IP, EIP, ICMP, EICMP, TCP, ETCP, UDP, SOCK6, IP6, EIP6, ICMP6, EICMP6 and UDP6

# 查看uuid
```sh
blkid
lsblk -f
```

# 为什么静态ip会丢失
eth7配了ip以后一会就丢失了
```
ifconfig eth7 8.8.8.4 up
```
因为一个service, 叫NetworkManager, 它会感知到eth-all的状态, 然后用dhclient配ip
用`tail -f /var/log/messages`就能看到dhclient和NetworkManager对eth7的操作
停止NetworkManager就OK.
```
service NetworkManager stop
```

补充:
在cetos下, /etc/sysconfig/network-scripts下面的脚本应该是管网络的

# 用tail查看系统log
```
tail -f /var/log/messages
```
也有人说
```
tail -f /var/log/kern.log
```
或
```
watch 'dmesg | tail -50'
```
或
```
cat /proc/kmsg
```

我的结论: 看这个文件就行了/var/log/messages, 已经包括了demsg的内容

# 代码打补丁
注意看当前的目录和patch文件的相对关系
比如patch文件里
```
diff -Naurp base/megaraid_sas.h fc19/megaraid_sas.h
```
而当前目录下就有megaraid_sas.h, 说明该P1
```
$ patch -p1 < patches/fc19.patch
```

# 如何查看io占用高的进程
io总体使用情况
```
iostat -x 1
apt install iotop
iotop
```

# linux访问win共享
`apt install samba-client`
这里有个问题, 就是主机名字没法解析, 此时需要先在window上ping主机名  
win7:
```sh
ping socrates
得到Ping cafp01.caveonetworks.com [10.17.5.53]
ping mafp01
得到Ping mafp01.caveonetworks.com [10.11.1.36]
```

linux:
查看共享
```sh
$ smbclient -L //10.17.5.53 -N
mount
$ sudo mount -t cifs //10.17.5.53/ftproot /mnt -o user=,password=
```
拷贝--断点续传
```sh
$ rsync -v --progress --append  "/mnt/FAE/Users/VarunS/ThunderX/Ubuntu FS/ThunderX-ubuntu-8G-v3-FAE.tar.bz2" ThunderX-ubuntu-8G-v3-FAE.tar.bz2
```

# ubuntu杀进程和服务
```sh
ps aux | grep -v "\[.*\]" | grep -E "cron|dhclient|38400|rsyslogd" | awk '{print $2}' | xargs kill -9
```
一般上面的命令杀了进程以后, 那些进程又会自动生成.
下面的方法杀的彻底
```sh
services="cron resolvconf rsyslog udev dbus upstart-file-bridge upstart-socket-bridge upstart-udev-bridge tty1 tty2 tty3 tty4 tty5 tty6 tty7 tty8 tty9"
for s in $services;do service $s stop;done
```

# 去掉最后几个字符
```sh
byj@byj-Aspire-1830T ~
$echo abcd | sed s'/.$//'
abc
byj@byj-Aspire-1830T ~
$echo abcd | sed s'/..$//'
ab
```

# 在shell里用exec eval source区别

* eval 执行一个命令
* exec 在新进程中执行一个命令，并且终止当前进程
* source 在当前进程中执行脚本

# 查看线程, 大写H
```
$ ps auxH | wc -l
```
查看tid号
`ps -efL`
LWP那一列就是线程号, 如果是单独的线程, 应该和进程号一样.
[byj]注: `ps -eLf`更实用一点

# linux启动脚本顺序
```sh
init
  rc-->/etc/rc.d/rc3.d/*-->rc.local
  getty-->login-->/bin/bash
  /etc/profile-->~/.bash_profile-->~/.bash_login-->~/.profile-->~/.bashrc
```

# 文件权限
other不可写
```
chmod o-w repo -R
```
所有可执行, 一般目录都是这个属性
```
chmod a+x repo -R
```
用户组可写
```
chmod g+w repo/ -R
```

# SELinux的文件权限扩展
```sh
[root@cavium cavium]# ll
total 313568
drwxr-xr-x  2 root    root        4096 Jan 19 13:14 nfs
drwxrwxr-x  6 root    cavium      4096 Dec 28 21:18 repo
-rw-r--r--  1 yingjie cavium 303327016 Jan 19 11:19 RHELSA-1.6-Server.img.xz
drwxrwxrwx. 2 root    cavium      4096 Dec 23 17:29 share
-rw-r--r--  1 root    root      158761 Jan 12 15:19 tests.tar.gz
-rw-r--r--  1 root    root    17590232 Jan 20 20:12 thunder-bootfs.img
```
一般的权限后面有个., 这个是**selinux security context**, 很多时候, 这个东西会很麻烦
比如明明有写权限的文件不能访问, git库不能push等等

`ls -Z`可以查看这个权限
```sh
[root@cavium cavium]# ls -Z
drwxr-xr-x  root    root   ?                                nfs
drwxrwxr-x  root    cavium ?                                repo
-rw-r--r--  yingjie cavium ?                                RHELSA-1.6-Server.img.xz
drwxrwxrwx. root    cavium unconfined_u:object_r:home_root_t:s0 share
-rw-r--r--  root    root   ?                                tests.tar.gz
-rw-r--r--  root    root   ?                                thunder-bootfs.img
```
`chcon`命令可以改这个selinux security context权限
`setfattr -x security.selinux` 可以去掉这个扩展权限
`find . -exec setfattr -x security.selinux {} ;`
或者
`[root@cavium /]# find cavium/ -exec setfattr -x security.selinux {} ;`

# shell for
```sh
$ for i in {1..5};do echo next $i;done
next 1
next 2
next 3
next 4
next 5
```
```sh
repos='"." "bootloader/edk2/" "bootloader/grub2/grub/" "bootloader/trusted-firmware/atf/" "bootloader/u-boot/" "linux/kernel/linux-aarch64/"'
$ for i in $repos;do echo 1111111 $i;done
1111111 "."
1111111 "bootloader/edk2/"
1111111 "bootloader/grub2/grub/"
1111111 "bootloader/trusted-firmware/atf/"
1111111 "bootloader/u-boot/"
1111111 "linux/kernel/linux-aarch64/"
```
注:**做为一个变量, repos表示一个list, 此时也可不加每个元素的双引号"", 也可以打印每个元素;
但在for里面直接写引号, 比如`for i in '1 2 3 4 ';do echo dddd$i;done`则只能打印一行.**

# 观察82599的中断
`watch -d -n 1 "cat /proc/interrupts | egrep 'eth5|CPU0'"`

# 关于中断balance
设置哪几个CPU可以处理中断
`$ echo f0 > /proc/irq/11/smp_affinity`

> 在网络非常 heavy 的情况下，对于文件服务器、高流量 Web 服务器这样的应用来说，把不同的网卡 IRQ 均衡绑定到不同的 CPU 上将会减轻某个 CPU 的负担，提高多个 CPU 整体处理中断的能力；对于数据库服务器这样的应用来说，把磁盘控制器绑到一个 CPU、把网卡绑定到另一个 CPU 将会提高数据库的响应时间、优化性能。合理的根据自己的生产环境和应用的特点来平衡 IRQ 中断有助于提高系统的整体吞吐能力和性能。
 
注意：在手动绑定 IRQ 到 CPU 之前需要先停掉 irqbalance 这个服务，irqbalance 是个服务进程、是用来自动绑定和平衡 IRQ 的：
`$ /etc/init.d/irqbalance stop`
使用的脚本
```sh
#!/bin/sh -e
interrupt_name="eth3-TxRx-"
core_num=`cat /proc/cpuinfo  | grep "processor.*:" -c`
irq_num=`cat /proc/interrupts | grep $interrupt_name -c`
[ $irq_num -eq $core_num ] || (echo 'Error, Call Cavium FAE!!!' && exit)
irq_list=`cat /proc/interrupts | grep $interrupt_name | awk -F ":" '{print $1}'`
first_riq=`cat /proc/interrupts | grep ${interrupt_name}'0' | awk -F ":" '{print $1}'`
for irq in $irq_list
do
                duty_core=$(($irq - $first_riq))
                cpu_mask=$((1<<$duty_core))
                hex_core_mask=`printf '%016x' $cpu_mask`
                high=`echo $hex_core_mask | cut -c 9-16`
                low=`echo $hex_core_mask | cut -c 1-8`
                combine=$low","$high
                echo Setting irq: $irq to core mask: $combine
                echo $combine >/proc/irq/$irq/smp_affinity
done
```

# 多个jpg图片转到pdf
`convert *.jpg myeduction.pdf`

# 合并多个pdf
`pdfunite in-1.pdf in-2.pdf in-n.pdf out.pdf`

# ssh X11及VNC
## firefox over ssh x11 forwarding
本来想开VNC server来远程图形方式访问办公室的机器, 但发现其实ssh的xorg forwarding功能就基本满足需求:
需要打开-X选项, 默认是关闭的.
比如在我的笔记本上:
`ssh -X baiyingjie@caviumsh.f3322.net`
而此时登录到了办公室DELL的机器上, 此时敲firefox, 会在我的笔记本上打开DELL机器的firefox
```
baiyingjie@mserver-dell ~
$ firefox
```
此时的firefox里面的内容其实是DELL机器上的, 比如我可以访问inventec2机器的BMC, 直接在地址栏里192.168.1.12就OK了.

## 让远程的firefox更快
上面的方法在功能上已经没什么问题了, 但使用下来感觉很卡, 延时太高. 这里面的原因, 网上说法是ssh的加密算法问题, 而不是firefox本身的问题--有待确认, 但感觉加了压缩选项就好很多, 似乎和加密算法关系不大.
默认的ssh由于使用比较强力的加解密算法, 比如AES, 可能会比较卡;
改成下面的加密算法会好很多
```sh
ssh -X -C -c blowfish-cbc,arcfour baiyingjie@caviumsh.f3322.net

-X: 使能x11 forwarding
-C: 使能压缩, 经验证这个选项提升速度最明显
-c: 指定cipher, 大约就是加解密算法吧
```

# tools
## change tmux prefix
```
C-b :
set prefix C-a
```

## tmux复制
.tmuxrc里面加
```
bind-key -t vi-copy 'v' begin-selection
bind-key -t vi-copy 'y' copy-selection
```
先`C-b [`进入copy mode
v选中, y复制
`C-b ]`粘贴

# network
## mount fae storage
`sudo mount -t cifs castr1:/FAE -o username=ybai,password=CQUbyj@2010 castorage`
这个地址好像不认, 但没关系, 可以这样找到ip
```
$ ping castr1.caveonetworks.com
PING castr1.caveonetworks.com (10.18.5.15) 56(84) bytes of data.
```
所以这样应该可以了--还是不行
`sudo mount -t cifs 10.18.5.15:/FAE -o username=ybai,password=CQUbyj@2010 castorage`
最终版本 --验证可以
`sudo mount -t cifs //10.18.5.15/FAE -o username=ybai,password=CQUbyj@2011 castorage`
如果想断点续传
`rsync -rv --progress --append /mnt/MARKETING/Product_Releases/ThunderX/CDK/Linux/GoldenImage .`

# 新建一个用户到特定组
`useradd joel -g cavium`

# 查看虚拟分区
`kpatx和losetup`  --原理可能是自动检测image的分区表
* 先创建一个1G大小的映像文件来做实验
`dd bs=4096 if=/dev/zero of=~/hd.img count=262144`
* 将映像文件挂接到loopX中去
`losetup /dev/loopX ~/hd.img`
* 对loopX进行分区
`fdisk /dev/loopX`
* 我这里分了两个区，每个去512M大小
```
      Device Boot      Start         End      Blocks   Id  System
/dev/loopXpY            2048     1050623      524288   83  Linux
/dev/loopXpY         1050624     2097151      523264   83  Linux
```
* 正戏来了，使用kpartd装载映像，使用kpartx是需要root用户的，因为是用root登录的，所以不用使用sudo。从前面的命令就可以看出来...
`kpartx -av ～/hd.img`
* 装载之后，就可以在/dev/mapper/目录下看到两个loopXpY的文件了。
* 接下来对loopXpY进行格式化了。
`mkfs.vfat /dev/mapper/loopXpY`
* 然后挂载文件系统。
`mount /dev/mapper/loop1p1 /media/hd1`
* OK，罗嗦完了。

# thunder通过服务器上网
服务器em1 192.168.1.5可上外网
eth5是10G口, 和thunder的10G口eth3相连
首先, iptables是个用户态工具, 负责配规则. kernel里面的netfilter负责执行这些规则. 
编译内核时需要打开netfilter功能
在服务器上:
* 先查link状态
`ethtool eth5`
* 设ip
`ifconfig eth5 9.9.9.99 up`
* 打开转发
`sysctl -w net.ipv4.ip_forward=1`
* 设iptable, 所有9.9.9.0网络通过em1走nat
`iptables -t nat -A POSTROUTING -j MASQUERADE -s 9.9.9.0/24 -o em1`
所有配置OK
但这里我记两个问题:
1. 为什么看不到POSTROUTING链? 也看不到刚才设的规则. 在mint上也一样, 但可以工作
```sh
[root@cavium yingjie]# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     udp  --  anywhere             anywhere            udp dpt:domain 
ACCEPT     tcp  --  anywhere             anywhere            tcp dpt:domain 
ACCEPT     udp  --  anywhere             anywhere            udp dpt:bootps 
ACCEPT     tcp  --  anywhere             anywhere            tcp dpt:bootps 
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     all  --  anywhere             192.168.122.0/24    state RELATED,ESTABLISHED 
ACCEPT     all  --  192.168.122.0/24     anywhere            
ACCEPT     all  --  anywhere             anywhere            
REJECT     all  --  anywhere             anywhere            reject-with icmp-port-unreachable 
REJECT     all  --  anywhere             anywhere            reject-with icmp-port-unreachable 
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```
解答:
iptables -L这个命令没有指定table, 默认是filter, 那只显示filter涉及的链, 就只有三个.
用这个可以查刚才的nat规则
`iptables -t nat -L`

2. 我没有配route. --需要确认是不是真的不用配?

在thunder上:
* 配ip
`ifconfig eht3 9.9.9.98 up`
* 配路由
`route add default gw 9.9.9.9`
* 配DNS --这里需要确认以下, DNS没有更好的办法配了么?
`cat /et/resolv.conf`
* ping一下对端
`ping 9.9.9.9``
* ping一下真正的网关
`ping 192.168.1.1`
* ping一下baidu
`ping www.baidu.com`

# linux路由参考
```sh
在linux主机中开启iptables,做SNAT(源地址转换)；下面我就以RHEL为例了，，
1.eth0(网卡1----外网)，设置公网IP，开启路由转发（#vim /etc/sysctl.conf   修改net.ipv4.ip_forward = 1），
2.eth1(网卡2----内网)，设置内网IP，
3.iptalbes添加规则：
iptables -t nat -A POSTROUTING -o eth1 -s 192.168.100.0/24 -j SNAT --to-source 外网IP
注：192.168.100.0/24是内网网段，如果外网地址不是静态的，SNAT也可以如下添加：iptables -t nat -A POSTROUTING -o eth1 -s 192.168.100.0 -j MASQUERADE（表示自动匹配IP地址）
4.就是设置路由器了，自己搞定吧。
```

# brctl就是个交换机
```sh
brctl addbr testbridge
brctl addif testbridge eth5
brctl addif testbridge eth6
```
这里需要解释一下, br下面的port应该是没有ip的, 从逻辑上, 几个port合并成一个聚合的interface, 就是br, 对外只有一个ip
这也是为什么要给br设一个ip
```sh
ifconfig eth5 0.0.0.0
ifconfig eth6 0.0.0.0
ifconfig testbridge up
ifconfig testbridge 192.168.1.30 netmask 255.255.255.0 up
```

# wiz的搜索功能
一般为"或"搜索
linux版可以这样搜: `"struct*file"`
windows版: `s:struct AND file`

# 批量修改jpg图片大小--xarg位置指代
```sh
find . -iname "*.jpg" | xargs -l -i convert -resize 50% {} /tmp/{}
mkdir tmp && find . -iname "*.jpg" | xargs -l -i convert -resize 40% -rotate 180 {} tmp/{}
```
注: xargs有个参数-i 表示用{}替代上个命令的输出; 但现在推荐用-I replace-str
我自己的例子:
把CNNIC-SDK-SRC目录下的源文件考到src目录下, 去掉路径
`find CNNIC-SDK-SRC/ -iname "*.[ch]" | xargs -i cp {} src`

# ubuntu查看一个文件属于哪个package
第一种情况, 检查已经安装的文件
```
$ dpkg -S /usr/share/man/man1/qemu-system-mips.1.gz
qemu-system-mips: /usr/share/man/man1/qemu-system-mips.1.gz
```
第二种情况, 即使没装过这个文件
```
apt install apt-file
sudo apt-file update
$ apt-file search /usr/share/doc/qemu/q35-chipset.cfg
qemu: /usr/share/doc/qemu/q35-chipset.cfg
```

# ubuntu查看一个package包含哪些文件
```
dpkg -L qemu
dpkg -L qemu-system-mips
```

# egrep 分组或
`hg st -q | egrep -vi "cmake|Makefile"`

# 删除mysql的cmake相关的文件
`find -iname '*cmake*' -not -name CMakeLists.txt -not -path "*/.hg/*" -exec rm -rf {} \+`

# cpu hotplug
`echo 0 > /sys/devices/system/cpu/cpuX/online`

# grub reserve 内存
```sh
改/boot/grub/grub.conf, 在kernel 一行加
memmap=10M$1024M
memmap=7G$1G
```
```sh
memmap=nn[KMG]@ss[KMG]
[KNL] Force usage of a specific region of memory
Region of memory to be used, from ss to ss+nn.
memmap=nn[KMG]#ss[KMG]
[KNL,ACPI] Mark specific memory as ACPI data.
Region of memory to be used, from ss to ss+nn.
memmap=nn[KMG]$ss[KMG]
[KNL,ACPI] Mark specific memory as reserved.
Region of memory to be used, from ss to ss+nn.
Example: Exclude memory from 0x18690000-0x1869ffff
        memmap=64K$0x18690000
        or
        memmap=0x10000$0x18690000
```

# 给用户组加写权限
`chmod g+w OCTEON-SDK-3.1.0 -R`

# 修改文件所属用户
`chown byj.byj Makefile`

# 增加共享库路经
```sh
LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH
```

# 查看端口占用
```sh
netstat -apn | grep 500
-a: all
-p: 显示进程名
-n: 不解析地址名, 这会快很多
显示是ipsec charon这个进程, 进程号是1670
udp        0      0 0.0.0.0:4500            0.0.0.0:*                           1670/charon     
udp        0      0 0.0.0.0:500             0.0.0.0:*                           1670/charon     
udp6       0      0 :::4500                 :::*                                1670/charon     
udp6       0      0 :::500                  :::*                                1670/charon
先不着急用kill命令
用ipsec stop
```

# 查找大于1M的文件
`find -type f -size +1M | xargs ls -lh`

# wget下载
`$ wget --no-check-certificate https://www.kernel.org/pub/software/scm/git/git-2.1.0.tar.gz`
 
# 通过gcc查找libc路径 -- -print-file-name=xxx
```sh
$ mips64-octeon-linux-gnu-gcc -march=octeon3 -mabi=n32 -EB -pipe -msoft-float -print-file-name=libc.a
/repo/yingjieb/fgltb/sw/vobs/esam/build/reborn/buildroot-isam-reborn-cavium-fgltb/output/host/opt/ext-toolchain/bin/../mips64-octeon-linux-gnu/sys-root/usr/lib/../lib32-fp/libc.a
```

# 各种进制转十进制
```sh
$ echo $((2#111111))
63
$ echo $((16#4f))
79
$ echo $((0x4f)) 
79
```
附: 十进制转十六进制, 以下几种都行
```sh
printf "%x\n" 34
echo "obase=16; 34" | bc
( echo "obase=16" ; cat file_of_integers ) | bc
```

# bc --shell下的C计算器
其实bc可以进行任意进制转换
ibase是输入的进制
obase是输出的进制
bc其实是个语言, 语法类似C
```sh
$ echo "i=10; i++;i++" | bc
10
11
```

# CPU热插拔
```sh
# echo 0 > /sys/devices/system/cpu/cpu2/online
     Reset core 2. Available Coremask = 3fc
# grep "processor" /proc/cpuinfo
     processor           : 0
     processor          : 1
# echo 1 > /sys/devices/system/cpu/cpu2/online
     SMP: Booting CPU02 (CoreId 2)...
     CPU revision is: (Cavium Octeon II)
     Cpu 2 online
# grep "processor" /proc/cpuinfo
     processor           : 0
     processor           : 1
     processor           : 2
```

# hexdump查ascii码 -v选项
```sh
//-v表示接受输入, -n8表示有8个字节的数据, 缺点是只能输入键盘字符
/isam/user # hexdump -v -n8
01abAB%^
0000000 3031 6162 4142 255e                   
0000008
```

# hexdump指定格式
```
//跳过6个字节, 显示6个字节, 1/1表示1次, 每次1个字节, 按照%02X格式
/isam/user # hexdump -v -n6 -s6 -e'1/1 "%02X"' /isam/prozone/PBDD
060000000010/isam/user #
```

# 查看温感
```
dts
i2c
compatible = "ti,tmp432","ti,tmp431";
compatible = "ti,lm75";

cat /sys/class/hwmon/hwmon1/device/temp3_input
45250
表示45.25°C
```

# trap命令 --指定shell中处理signal
用kill -l查看所有的signal号
```sh
# Ignore SIGPIPE
# When a script is run from an ssh session that is closed before the script is
# finished, and the script writes to stdout (which no longer is connected to
# something) then the script would get a SIGPIPE signal and be aborted.
# This is normally not desired, so ignore the SIGPIPE signal entirely.
log_sigpipe() {
    # only if __sigpipe_occurred is not set
    if [ -z ${__sigpipe_occurred+x} ]; then
        echo "$(date): $0 ($$): SIGPIPE occurred, ignoring (parent $(pid_to_procname $PPID) ($PPID))" >> /isam/logs/info_siglog
        __sigpipe_occurred=1
    fi
}
trap log_sigpipe SIGPIPE
```

# 创建ram文件系统
`mount -o size=16G -t tmpfs none /mnt/tmpfs`

# heredoc格式代码
```sh
astyle --mode=c --style=linux -spfUcH <<CODE
any code
...
CODE
```

# install命令
在做共享库的Makefile中, 用到了install命令, 这个命令和cp功能差不多.
```sh
mkdir -p $(DESTDIR)/usr/lib
install $(LIB) $(DESTDIR)/usr/lib
ln -sf $(LIB) $(DESTDIR)/usr/lib/$(NAME)
ln -sf $(LIB) $(DESTDIR)/usr/lib/$(SONAME)
```

# readlink --读取链接文件
比如build是个链接
```sh
$ readlink -f build
/repo/yingjieb/fdt063/sw/vobs/esam/build
```

# 重新修改tmpfs的大小
`mount -o remount,size=XXX /tmp`
或者
```sh
$ hg diff -c7639b7191348
diff --git a/board/Alcatel-Lucent/isam-reborn/common/post_build.sh b/board/Alcatel-Lucent/isam-reborn/common/post_build.sh
--- a/board/Alcatel-Lucent/isam-reborn/common/post_build.sh
+++ b/board/Alcatel-Lucent/isam-reborn/common/post_build.sh
@@ -37,3 +37,10 @@ sed -i 's%^root:[^:]*:%root:$6$Pb/CNtO0N
 #     cat $archdir/mdev-extra.conf >> $targetdir/etc/mdev.conf
#
cat $commondir/mdev-base.conf > $targetdir/etc/mdev.conf
+
+# Restrict size of /tmp
+# By default /tmp has a maximum size of RAMsize/2, which is very big.
+# This means that someone can write up to RAMsize/2 to /tmp, causing only half
+# of RAM memory to be usable by real software. Since we should not need that
+# large a /tmp directory, limit its size.
+sed -i 's%^.*[ \t]/tmp[ \t].*$%tmpfs           /tmp           tmpfs    defaults,size=64M 0      0%' $targetdir/etc/fstab
```

# fg bg --前台运行 后台运行
```sh
fg %1
bg后台运行的好处是程序还在跑, 而ctrl+z程序不跑了
hello: 
后台运行的程序用printf向默认控制台打印也能输出
```

# shell写字符串, 以'\0'结尾
`printf "$string\0" > $tmpfile`

# 写十六进制数据到文件, 关键在于引号
```sh
/isam/user # printf '\xde\xad\xbe\xef' > file
/isam/user # hexdump -C file
00000000  de ad be ef                                       |....|
00000004
/isam/user # echo -ne "\xde\xad\xbe\xef" > file
/isam/user # hexdump -C file
00000000  de ad be ef                                       |....|
00000004
```
没有引号就不行
```sh
/isam/user # echo -e \xde\xad\xbe\xef > file
/isam/user # hexdump -C file
00000000  78 64 65 78 61 64 78 62  65 78 65 66 0a           |xdexadxbexef.|
0000000d
```

# 批量处理软链接之cscope.files
`cat cscope.files | while read f; do if [ ! -h $f ]; then echo $f; fi; done`

# readelf结果用sort排序, 按照第三列数字排序
`cat uboot.readelf | sed -e '1,79d' -e '3128,$d' | sort -rnk 3`

# 批量删除ZF_ ZL_ ZG
`cat cscope.files | egrep "isam|fpxt|fglt|nvps|vipr|rant" | xargs sed -i -e "s%Z[LFG]_%%g"`

# 批量格式化代码
使用4个空格
`cat cscope.files | xargs astyle --mode=c --style=linux -spfUcH`
使用tab
`cat cscope.files | xargs astyle --mode=c --style=linux -tpfUcH`
最终版
`cat cscope.files | egrep "isam|fpxt|fglt|nvps|vipr|rant" | xargs astyle --mode=c --style=linux -tpfUcH`

# 带时间的平均负载
```sh
$ echo "$(date +%H:%M:%S) # $(cat /proc/loadavg)"
10:07:14 # 0.98 1.00 1.15 2/723 26465
```

# rpm解压
`rpm2cpio OCTEON-LINUX-2.3.0-427.i386.rpm | cpio -div`

# 大小写转换
```sh
tr '[:upper:]' '[:lower:]' < input.txt > output.txt
sed -e 's/\(.*\)/\L\1/' input.txt > output.txt
sed -e 's/\(.*\)/\U\1/' input.txt > output.txt
```

# 解析C文件全部函数到h文件声明, 用于偷懒声明所有函数到头文件
`cat src/board_cpld.c | egrep "(void|unsigned char|u_int8)[[:blank:]]+[0-9a-zA-Z_]+\(.*\)" | sed -e 's/\(.*\)/\1;/g'`

# strace 重定向
```sh
strace echo "groad" &> myfile.txt
strace echo "groad" > myfile.txt 2>&1
```

# 通过elf生成工程文件列表 --利用gdb
```sh
mips64-octeon-linux-gnu-gdb -ex="info sources" -ex="quit" isam_app.nostrip | sed -e '1,15d' -e 's/,/\n/g' | sed -e '/^ *$/d' -e 's/^ *//g' > temp.list
find -L `cat cscope.files| egrep "/flat/" | sed 's!\(.*/flat/[^/]*\).*!\1!g' | sort -u` -iname "*.h" -o -iname "*.hh" -o -iname "*.hpp"
```

# sync命令使用, 把file buffer 刷进物理器件
```sh
/ # time dd if=/dev/zero of=/bigfile bs=65536 count=2048
2048+0 records in
2048+0 records out
real    0m 0.29s
user    0m 0.00s
sys     0m 0.30s

The command line (and similar ones) I used when sync was in order:
/ # time dd if=/dev/zero of=/bigfile bs=65536 count=2048 && time sync
2048+0 records in
2048+0 records out
real    0m 3.11s
user    0m 0.01s
sys     0m 0.15s
```

# 查找src下面的代码目录, find可以限制搜索深度
```sh
output=`find -maxdepth 1 -type d | grep -i socket`
echo $output
echo $output >> src_dirs.byj
sort -u src_dirs.byj
mkcsfiles `cat src_dirs.byj.sort`
注: sort -u能够去掉重复行
```

# 查看当前文件夹大小
`du -sh`

# 删除文件夹下面的所有链接文件
`find . -type l | xargs rm -rf`

# 压缩当前目录下sw_c文件夹
`tar -zcvf sw_c.tar.gz sw_c`

# 根据一个文本文件列表压缩 --hT
`tar cvf /repo1/yingjieb/isam_lis/sw/build/fant-f/OS/fant-f_osw.tar -hT /repo1/yingjieb/isam_lis/sw/build/fant-f/OS/path`

# tar.xz格式压缩解压
压缩
`tar -Jcvf etc.tar.xz /etc`
解压
`tar -Jxf etc.tar.xz`
 
# 在当前路径下的所有文件中，搜索关键字符串
`grep -rn byj .`

# ls按修改时间排序
`cat cscope.files | xargs ls -lt | more`

# ls按大小排序
`cat cscope.files | xargs ls -lhS | more`

# 查看键盘映射
`stty -a`

# 限制深度的find
`find -maxdepth 2 -name ".stamp*" | xargs rm -f`

# 查看系统支持的文件系统类型
`cat /proc/filesystems`

# 替换字符串
```sh
$ sst=abcdefcd
只替换第一个位置
$ echo ${sst/cd/55}
ab55efcd
ASBLX28:/home/yingjieb
全替换
$ echo ${sst//cd/55}
ab55ef55
${string/#substring/replacement}
如果$substring匹配$string的开头部分, 那么就用$replacement来替换$substring.
${string/%substring/replacement}
如果$substring匹配$string的结尾部分, 那么就用$replacement来替换$substring.
```

# 批量软链接
`for file in *; do; if [ ! -L $file -a "$file" != "Makefile" ]; then; echo ln $file...;ln -sf ../../../../../executive/$file $file;fi;done`

# tee
tee指令会从标准输入设备读取数据，将其内容输出到标准输出设备,同时保存成文件。我们可利用tee把管道导入的数据存成文件，甚至一次保存数份文件。

# 处理软链接, Thomas版
`find arch -type l | xargs ls -1l | grep executive | awk '{sub("../../","",$11); printf "ln -sf %s %s\n",$11,$9}'`

# 关闭printk打印
`echo 1 1 1 1>/proc/sys/kernel/printk`

# usb串口 --lsusb --minicom
```sh
插入usb串口以后，lsusb可以查看usb总线下面的设备。动态的。
$lsusb
此时查看
$ls /dev/tty*
可以看到新增了四个ttyUSB
/dev/ttyUSB0 /dev/ttyUSB1 /dev/ttyUSB2 /dev/ttyUSB3 
现在可以安装minicom
apt install minicom
```

# locate--linux下面的everything
```sh
byj@byj-mint ~/repo/hg/OCTEON-SDK-3.1-tools
$locate OCTEON-SDK-3.1-p2-tools.tar.xz
/home/byj/repo/alu/buildroot-isam-reborn-cavium-fgltb/dl/OCTEON-SDK-3.1-p2-tools.tar.xz
/home/byj/repo/alu/fgltb/sw/vobs/esam/build/reborn/buildroot-isam-reborn-cavium-fgltb/dl/OCTEON-SDK-3.1-p2-tools.tar.xz
/home/byj/share/OCTEON-SDK-3.1-p2-tools.tar.xz
```

# rsync拷贝目录，去除hg
`rsync -av --exclude .hg OCTEON-SDK-3.1-pristine OCTEON-SDK-3.1`
