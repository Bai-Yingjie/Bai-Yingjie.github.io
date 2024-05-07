- [chroot到aarch64的alpine minirootfs](#chroot到aarch64的alpine-minirootfs)
- [docker 支持multi-arch](#docker-支持multi-arch)
  - [其实不需要`--platform`](#其实不需要--platform)
- [其他arch参考](#其他arch参考)
  - [mips64](#mips64)
  - [ppc](#ppc)
  - [ppc64](#ppc64)
  - [aarch64](#aarch64)

Linux Kernel提供了binfmt_misc机制来[执行任意类型的文件](profiling_调试和分析记录3.md#linux执行文件的过程).

简单来说
* 对于ELF可执行文件, Linux执行一个ELF文件的过程, 其实是执行ld.so, 再由后者去真正完成加载和执行.
* 对于其他文件可执行文件, 内核首先读这个文件的头128字节, 来确定用哪个handler来执行这个文件. 对不同类型的文件, 可以注册不同的handler.
  * 脚本文件一般都在文件第一行, 比如有`#!/bin/python2`的, 是用python2来执行.
  * 用binfmt_misc机制可以注册handler, 比如注册.jpg用feh打开: `echo ':fehjpg:E::jpg::/usr/bin/feh:' > /proc/sys/fs/binfmt_misc/register`

# chroot到aarch64的alpine minirootfs
思路是在x86_64的host上先起个container, 在container里面使用binfmt_misc机制注册qemu user mode来执行aarch64的bin.

首先, `binfmt_misc`可以在container里面使用. 一般的container是没有`/proc/sys/fs/binfmt_misc/register`文件的, 因为在container内部, `binfmt_misc`需要手动mount.

在alpine container里面操作:
```shell
$ lsmod | grep binfmt
binfmt_misc            20480  1

# 在container里面需要手动mount, 需要privileged选项
$ sudo mount binfmt_misc -t binfmt_misc /proc/sys/fs/binfmt_misc

# 我使用alpine系统, 安装qemu-aarch64
$ sudo apk add qemu-aarch64

$ which qemu-aarch64
/usr/bin/qemu-aarch64

# 注册qemu-aarch64
sudo sh -c 'echo ":qemu-aarch64:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\xb7\x00:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/qemu-aarch64:F" > /proc/sys/fs/binfmt_misc/register'

# 注册成功后可以看到多了qemu-aarch64文件.
$ ls /proc/sys/fs/binfmt_misc/qemu-aarch64
/proc/sys/fs/binfmt_misc/qemu-aarch64

# 这个文件用rm是删不掉的., 只有取消注册才能删掉.
sudo sh -c 'echo -1 > /proc/sys/fs/binfmt_misc/qemu-aarch64'
```

在container内部下载aarch64版本的minirootfs, 准备chroot
```shell
$ wget https://dl-cdn.alpinelinux.org/alpine/v3.18/releases/aarch64/alpine-minirootfs-3.18.6-aarch64.tar.gz

$ mkdir rootfs && cd rootfs && tar xvf ../alpine-minirootfs-3.18.6-aarch64.tar.gz

# 可以看到, rootfs下面的bin都是ARM aarch64的
$ file bin/busybox
bin/busybox: ELF 64-bit LSB pie executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-musl-aarch64.so.1, stripped
```

最后一步, chroot, 我用了unshare做了简单的名字空间隔离:
```shell
# bind mount当前的proc -- 这里不推荐, 因为外面的系统是x86_64的
#sudo mount --bind /proc proc

# 方法1: 直接指定chroot后, 执行/usr/bin/qemu-aarch64 /bin/sh
$ unshare -mprf env -i /usr/sbin/chroot . /usr/bin/qemu-aarch64 /bin/sh

# 方法2: 直接执行/bin/sh, 因为我们注册了qemu-aarch64的binfmt_misc handler
unshare -mprf env -i /usr/sbin/chroot . /bin/sh
# 如果想继承父进程的env, 就把env -i去掉.
unshare -mprf chroot . /bin/sh

# 经过验证, 下面的命令更好用
sudo unshare -mpf --setgroups allow chroot . /bin/sh

# 进入aarch64的chroot环境后, 可以直接执行shell命令
which sh
/bin/sh

# 做其他有意义的事情之前, 要mount proc, dev, sys等文件系统
mount -t devtmpfs none /dev
mkdir -p /dev/pts
mount -t devpts none /dev/pts
mount -t proc none /proc
mount -t sysfs none /sys
```
![](img/multi-arch_docker_binfmt_misc_20240506101349.png)

`unshare`的选项含义:
* `-m`: Unshare  the  mount  namespace
* `-p`: Unshare the PID namespace
* `-r`: 在内部使用root用户
* `-f`: 先fork一个子进程再run指定的命令. 否则直接在当前进程执行命令.
注意, 如果只用`unshare -mr`, chroot环境中则不能`mount -t proc none /proc`, 会报错`mount: permission denied (are you root?)`, 是因为在当前进程下已经不能再mount proc了, 解决办法是用`-f --fork`选项.
* `--mount-proc`: 这也是一个常用选项. 在执行命令之前mount proc. 但上面的例子里没有使用, 因为我们chroot进入的是aarch64系统, 用了这个选项也不起作用, 还是要在chroot里面自己mount proc. 如果是host native的命令, 比如`unshare -mprf --mount-proc ls /proc`, 则`--mount-proc`是管用的.

尝试在aarch64 rootfs里面编译成功:
```shell
apk add musl-dev gcc file
echo 'int main(){return 0;}' > main.c
gcc -O2 main.c
file a.out
a.out: ELF 64-bit LSB pie executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-musl-aarch64.so.1, with debug_info, not stripped
```

注: 
* 所有container, 包括host, 共享binfmt_misc的配置, 操作会互相影响. 因为只有一个binfmt_misc驱动.

参考:
* [qemu binfmt的magic查表](https://github.com/qemu/qemu/blob/master/scripts/qemu-binfmt-conf.sh)
* [binfmt_misc用法](https://www.kernel.org/doc/Documentation/admin-guide/binfmt-misc.rst)
* [alpine chroot参考](https://wiki.alpinelinux.org/wiki/How_to_make_a_cross_architecture_chroot)

# docker 支持multi-arch
docker run, build等命令支持`--platform`参数来运行其他arch的docker image

```shell
docker run --platform linux/arm64 -it docker-registry-remote.artifactory-blr1.int.net.nokia.com/alpine sh
```
![](img/multi-arch_docker_binfmt_misc_20240506150850.png)

如上图, 加了`--platform linux/arm64`后, docker会自动pull对应arch的image, 然后启动, 并成功运行了aarch64的alpine.

platform支持:
* linux/amd64: 64-bit x86 architecture (commonly referred to as x86_64 or AMD64)
* linux/arm64: 64-bit ARM architecture (ARMv8)
* linux/arm/v7: 32-bit ARM architecture (ARMv7)
* linux/arm/v6: 32-bit ARM architecture (ARMv6)
* linux/ppc64le: 64-bit PowerPC little-endian architecture
* linux/ppc64: 64-bit PowerPC architecture
* linux/s390x: 64-bit IBM Z architecture
* linux/386: 32-bit x86 architecture
* linux/riscv64: 64-bit RISC-V architecture

## 其实不需要`--platform`
`--platform`是告诉docker, 目标image的arch是什么类型的, 用于pull到目标arch的image.

其实如果已经pull下来了image, 用`docker images`是能够看到不同arch的image的. 比如刚刚使用的`alpine:latest`, 现在已经是arm64架构了:
```shell
$ docker inspect ace17d5d883e | grep "Architecture"
        "Architecture": "arm64",
```

直接`docker run`这个image, 也是可以的. 只是会有个warning
```shell
$ docker run -it ace17d5d883e sh
WARNING: The requested image's platform (linux/arm64/v8) does not match the detected host platform (linux/amd64) and no specific platform was requested
/ # ldd
musl libc (aarch64)
Version 1.2.4_git20230717
Dynamic Program Loader
Usage: /lib/ld-musl-aarch64.so.1 [options] [--] pathname
```

所以:
* 只要注册了`binfmt_misc`handler, 可以直接`docker run`目标image
* 用了`--platform`的好处是让docker自动pull对应arch的image

结论: 可以自己build不同arch的image, 然后直接`docker run`这个image id.

参考:
* https://docs.docker.com/build/guide/multi-platform/
* https://docs.docker.com/build/building/multi-platform/

# 其他arch参考
使用root用户执行
## mips64
```shell
mkdir -p ~/rootfs && cd ~/rootfs

apk add qemu-mips64

sh -c 'echo ":qemu-mips64:M::\x7fELF\x02\x02\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x08:\xff\xff\xff\xff\xff\xff\xff\x00\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff:/usr/bin/qemu-mips64:F" > /proc/sys/fs/binfmt_misc/register'

ARCH=mips64
mkdir -p $ARCH/etc/apk/
cp /etc/apk/repositories $ARCH/etc/apk/
apk add -p $ARCH --initdb -U --arch $ARCH --allow-untrusted alpine-base

unshare -mpf --setgroups allow chroot $ARCH /bin/sh

#在里面看到
# ldd
musl libc (mips64-sf)
Version 1.2.4
Dynamic Program Loader
Usage: /lib/ld-musl-mips64-sf.so.1 [options] [--] pathname
```

## ppc
```shell
mkdir -p ~/rootfs && cd ~/rootfs

apk add qemu-ppc

sh -c 'echo ":qemu-ppc:M::\x7fELF\x01\x02\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x14:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff:/usr/bin/qemu-ppc:F" > /proc/sys/fs/binfmt_misc/register'

ARCH=ppc
mkdir -p $ARCH/etc/apk/
cp /etc/apk/repositories $ARCH/etc/apk/
apk add -p $ARCH --initdb -U --arch $ARCH --allow-untrusted alpine-base

unshare -mpf --setgroups allow chroot $ARCH /bin/sh

#在里面看到
# ldd
musl libc (powerpc)
Version 1.2.4
Dynamic Program Loader
Usage: /lib/ld-musl-powerpc.so.1 [options] [--] pathname
```

## ppc64
```shell
mkdir -p ~/rootfs && cd ~/rootfs

apk add qemu-ppc64

sh -c 'echo ":qemu-ppc64:M::\x7fELF\x02\x02\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x15:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff:/usr/bin/qemu-ppc64:F" > /proc/sys/fs/binfmt_misc/register'

ARCH=ppc64
mkdir -p $ARCH/etc/apk/
cp /etc/apk/repositories $ARCH/etc/apk/
apk add -p $ARCH --initdb -U --arch $ARCH --allow-untrusted alpine-base

unshare -mpf --setgroups allow chroot $ARCH /bin/sh

#在里面看到
# ldd
musl libc (powerpc64)
Version 1.2.4
Dynamic Program Loader
Usage: /lib/ld-musl-powerpc64.so.1 [options] [--] pathname
```

## aarch64
```shell
mkdir -p ~/rootfs && cd ~/rootfs

apk add qemu-aarch64

sh -c 'echo ":qemu-aarch64:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\xb7\x00:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/qemu-aarch64:F" > /proc/sys/fs/binfmt_misc/register'

ARCH=aarch64
mkdir -p $ARCH/etc/apk/
cp /etc/apk/repositories $ARCH/etc/apk/
apk add -p $ARCH --initdb -U --arch $ARCH --allow-untrusted alpine-base

unshare -mpf --setgroups allow chroot $ARCH /bin/sh

#在里面看到
# ldd
musl libc (aarch64)
Version 1.2.4
Dynamic Program Loader
Usage: /lib/ld-musl-aarch64.so.1 [options] [--] pathname
```

如果有这样的error`00000040028F5B10:error:0A000086:SSL routines:tls_post_process_server_certificate:certificate verify failed:ssl/statem/statem_clnt.c:1889:`
需要:
```shell
apk add -p $ARCH --initdb -U --arch $ARCH --allow-untrusted ca-certificates
```