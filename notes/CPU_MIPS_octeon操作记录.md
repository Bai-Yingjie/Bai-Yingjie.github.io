- [uboot从usb load到spi flash](#uboot从usb-load到spi-flash)
- [编译kernel+debian并运行](#编译kerneldebian并运行)
  - [安装必须的包](#安装必须的包)
  - [编译前配置好环境变量](#编译前配置好环境变量)
  - [make hello](#make-hello)
- [升级uboot](#升级uboot)
- [rootfs over nfs](#rootfs-over-nfs)
- [78xx uboot nfs](#78xx-uboot-nfs)
- [关闭nmi wdt](#关闭nmi-wdt)
- [使用SD卡编译debian](#使用sd卡编译debian)
- [新版uboot起kernel](#新版uboot起kernel)
- [evb7000-sff sd](#evb7000-sff-sd)
- [编译SE](#编译se)
- [uboot运行SE](#uboot运行se)
- [uboot静态ip](#uboot静态ip)
- [uboot自动起linux](#uboot自动起linux)
- [strip vmlinux](#strip-vmlinux)
- [oct-sim](#oct-sim)
- [tftp vmlinux](#tftp-vmlinux)
- [tftpboot](#tftpboot)
- [运行iperf测试](#运行iperf测试)
- [debian 打开dhcp client动态获取ip](#debian-打开dhcp-client动态获取ip)
- [查kernel版本（编译时）](#查kernel版本编译时)
  - [linux编译](#linux编译)
- [只编译debian目录](#只编译debian目录)
- [debain启动runlevel 0](#debain启动runlevel-0)
- [使用分离的cpio做initramfs](#使用分离的cpio做initramfs)

# uboot从usb load到spi flash
```
usb start
usb info
usb storage
usb tree
fatls usb 0:1
fatls usb 0:1 /78
sf probe
fatload usb 0:1 0 /78/spi-boot.bin
sf update $(fileaddr) 0 $(filesize)
fatload usb 0:1 0 /78/u-boot-octeon_generic_spi_stage2.bin
sf update $(fileaddr) 0x10000 $(filesize)
fatload usb 0:1 0 /78/u-boot-octeon_ebb7800.bin
sf update $(fileaddr) 0x100000 $(filesize)
```

# 编译kernel+debian并运行
```sh
#编译内核
#到linux下面
make kernel-deb
#编译debian并烧到U盘中，注意这里sdc是新插入的U盘
#到dabian目录下
$sudo make DISK=/dev/sdc compact-flash OCTEON_ROOT=/home/byj/repo/hg/OCTEON-SDK-3.1-pristine
Octeon ebb6100# fatload ide0 0 0 vmlinux.64
Octeon ebb6100# printenv loadaddr 
loadaddr=0x20000000
Octeon ebb6100# bootoctlinux 0 coremask=0xf root=/dev/sda2 loglevel=8
#注意:默认有个run level的问题, 不指定会到init 0
Octeon ebb6100# fatload mmc 0 0 vmlinux.64
Octeon ebb6100# printenv loadaddr 
loadaddr=0x20000000
Octeon ebb6100# bootoctlinux 0 coremask=0xf root=/dev/sda2 rw 3 loglevel=8
#保存到uboot命令, 注意分号前面需要转义
setenv sda2boot fatload mmc 0 0 vmlinux.64\;bootoctlinux 0 coremask=0xf root=/dev/sda2 rw 3 mem=4G
run sda2boot
```

## 安装必须的包
```
apt install build-essential
apt install libncurses5-dev
apt install flex bison
apt install automake
```

## 编译前配置好环境变量
```
cd OCTEON-SDK-3.1
source env-setup OCTEON_CN71XX
```

## make hello
```
cd examples/hello
make run
```

# 升级uboot
先`tftp`
再`bootloaderupdate`

# rootfs over nfs
```sh
#安装nfs, server端和client端都要装
yum install nfs-utils nfs-utils-lib
apt-get install nfs-utils nfs-utils-lib
#配置nfs
$vi /etc/exports
$cat /etc/exports 
/home/yingjie/gentoo4oct/hg-nfsroot/nfsroot *(async,rw,no_root_squash,no_subtree_check)
$sudo exportfs -a
#开始nfs服务
chkconfig nfs on
service nfs start
/etc/init.d/nfs start
#客户端
#在mint上测试, 需要按照
$ apt install nfs-client
$ sudo mount -t nfs -o nolock 192.168.1.99:/home/yingjie/gentoo4oct/hg-nfsroot/nfsroot nfsroot/
#但是提示
mount.nfs: access denied by server while mounting 192.168.1.99:/home/yingjie/gentoo4oct/hg-nfsroot/nfsroot
#网上提示需要在server端运行
sudo exportfs -a
#再次在client端mount就好了

#可以在client上查看server export哪些目录
$ showmount -e 192.168.1.99
Export list for 192.168.1.99:
/home/yingjie/gentoo4oct/hg-nfsroot/nfsroot *
```

# 78xx uboot nfs
```sh
#需要内核支持NFS和DNS
Octeon ebb7800# setenv ipaddr 192.168.1.113
Octeon ebb7800# setenv serverip 192.168.1.99
Octeon ebb7800# setenv gatewayip 192.168.1.1
Octeon ebb7800# setenv netmask 255.255.255.0
Octeon ebb7800# setenv nfsroot /home/yingjie/gentoo4oct/hg-nfsroot/nfsroot
Octeon ebb7800# tftp $(loadaddr) $(serverip):vmlinux.78nfs.strip
Octeon ebb7800# bootoctlinux $(loadaddr) numcores=$(numcores) root=/dev/nfs rw nfsroot=$(serverip):$(nfsroot),nolock,tcp ip=$(ipaddr):$(serverip):$(gatewayip):$(netmask)::eth0:off mem=128G
#保存为命令, 注意$和;前面的\
Octeon ebb7800# setenv nfsboot tftp \$(loadaddr) \$(serverip):vmlinux.78nfs.strip\;bootoctlinux \$(loadaddr) numcores=\$(numcores) root=/dev/nfs rw nfsroot=\$(serverip):\$(nfsroot),nolock,tcp ip=\$(ipaddr):\$(serverip):\$(gatewayip):\$(netmask)::eth0:off mem=128G
Octeon ebb7800# run nfsboot
```

# 关闭nmi wdt
```sh
#78
devmem 0x1010000020000 64 0
for i in `seq 0 47`;do busybox devmem $((i*8+0x1010000020000)) 64;done
for i in `seq 0 47`;do busybox devmem $((i*8+0x1010000020000)) 64 0;done
#68
0x0001070100100000
for i in `seq 2 27`;do busybox devmem $((i*8+0x0001070100100000)) 64;done
for i in `seq 2 31`;do busybox devmem $((i*8+0x0001070100100000)) 64 0;done
```

# 使用SD卡编译debian
```sh
. env-setup OCTEON_CN70XX
cd linux/
make kernel-deb
cd linux/debian/
make DISK=/dev/mmcblk0 P1=p1 P2=p2 P1_SIZE=256M compact-flash
```

# 新版uboot起kernel
```
mem=2G
```

# evb7000-sff sd
```sh
Octeon evb7000_sff# fatls mmc 1
Octeon evb7000_sff# fatload mmc 1 0 octboot2.bin
[root@dev-lnx1 OCTEON-SDK]# . env-setup OCTEON_CN71XX
[root@dev-lnx1 OCTEON-SDK]# cd linux/
[root@dev-lnx1 linux]# make kernel
Octeon evb7000_sff# tftp 0 vmlinux.64
tftp:
Octeon evb7000_sff# tftp 0 passthrough
Octeon evb7000_sff# bootoct 0 numcores=4
sd:
Octeon evb7000_sff# fatload mmc 1 0 passthrough
Octeon evb7000_sff# bootoct 0 numcores=4
example:


#为啥压缩了就不行呢？
Octeon evb7000_sff# fatls mmc 1
  8271440 vmlinux.64 
 21950338 vmlinux-br-testbench-2014-10-08.strip.lzma
Octeon evb7000_sff# fatload mmc 1 0x30000000 vmlinux-br-testbench-2014-10-08.strip.lzma
reading vmlinux-br-testbench-2014-10-08.strip.lzma
21950338 bytes read in 2617 ms (8 MiB/s)
Octeon evb7000_sff# unlzma 0x30000000 $(loadaddr)
Uncompressed size: 29469392 = 0x1C1AAD0
Octeon evb7000_sff# bootoctlinux $(loadaddr) numcores=$(numcores) mem=2G


#这个可以
# mount /dev/mmcblk0p2 rootfs/
debian
fatload mmc 1 0 vmlinux.64
bootoctlinux $(loadaddr) numcores=$(numcores) mem=2G root=/dev/mmcblk0p2
embedded
Octeon evb7000_sff# fatls mmc 1
  8271440 vmlinux.64 
 21950338 vmlinux-br-testbench-2014-10-08.strip.lzma 
 29469392 vmlinux-br-tb.strip 
3 file(s), 0 dir(s)
Octeon evb7000_sff# fatload mmc 1 0 vmlinux-br-tb.strip
reading vmlinux-br-tb.strip
29469392 bytes read in 3437 ms (8.2 MiB/s)
Octeon evb7000_sff# bootoctlinux $(loadaddr) numcores=$(numcores) mem=2G
argv[2]: numcores=4
argv[3]: mem=2G
Allocating memory for ELF segment: addr: 0xffffffff80100000 (adjusted to: 0x100000), size 0x1d89ce0
## Loading big-endian Linux kernel with entry point: 0xffffffff806cf260 ...
Bootloader: Done loading app on coremask: 0xf
Starting cores:
 0xf
```

# 编译SE
```
cd examples/hello
make run
```

# uboot运行SE
```sh
#运行SE, tftp方式下载:
Octeon evb7000_sff# tftp 0 passthrough
Octeon evb7000_sff# bootoct 0 numcores=4
#运行SE, 从sd加载:
Octeon evb7000_sff# fatload mmc 1 0 passthrough
Octeon evb7000_sff# bootoct 0 numcores=4
```

# uboot静态ip
```
setenv ethact octeth0
setenv ethprime octeth0
setenv serverip 192.168.1.100
setenv ipaddr 192.168.1.33
saveenv
```

# uboot自动起linux
```sh
Octeon evb7000_sff# setenv bootcmd run linux_mmc
Octeon evb7000_sff# printenv
autoload=n
baudrate=115200
bf=bootoct $(flash_unused_addr) forceboot numcores=$(numcores)
boardname=evb7000_sff
bootcmd=linux_mmc
bootdelay=0
bootloader_flash_update=bootloaderupdate
burn_app=erase $(flash_unused_addr) +$(filesize);cp.b $(fileaddr) $(flash_unused_addr) $(filesize)
dram_size_mbytes=1024
env_addr=1fbf0000
env_size=10000
eth1addr=02:54:71:69:f3:01
eth2addr=02:54:71:69:f3:02
eth3addr=02:54:71:69:f3:03
eth4addr=02:54:71:69:f3:04
eth5addr=02:54:71:69:f3:05
eth6addr=02:54:71:69:f3:06
eth7addr=02:54:71:69:f3:07
eth8addr=02:54:71:69:f3:08
ethact=octeth0
ethaddr=02:54:71:69:f3:00
flash_base_addr=1f400000
flash_size=800000
flash_unused_addr=1f680000
flash_unused_size=580000
ipaddr=192.168.1.112
linux_mmc=fatload mmc 0 $(loadaddr) vmlinux.64;bootoctlinux $(loadaddr)
loadaddr=0x20000000
ls=fatls mmc 0
numcores=4
octeon_failsafe_mode=0
octeon_ram_mode=0
serial#=2.0G1409-000206
serverip=192.168.1.99
stderr=serial
stdin=serial,pci,bootcmd
stdout=serial
uboot_flash_addr=bf520000
uboot_flash_size=60160000
ver=U-Boot 2013.07 ( (U-BOOT build: 93, SDK version: 3.1.1-525), svnversion: u-boot:99862M, exec:)-svn99835 (Build time: Jun 07 2014 - 07:13:20)
 
Environment size: 1246/65532 bytes
```

# strip vmlinux
```sh
/home/byj/repo/hg/xrepo/buildroot/output/host/usr/bin/mips64-octeon-linux-gnu-strip -o vmlinux.strip vmlinux
yingjie@aircraft ~/xrepo/buildroot/output/images
../host/usr/bin/mips64-octeon-linux-gnu-strip -o vmlinux.strip vmlinux
```

# oct-sim
```sh
#terminal1:
byj@byj-mint ~/repo/hg/OCTEON-SDK-3.1-pristine/linux/kernel
$oct-sim /home/byj/repo/hg/xrepo/buildroot/output/images/vmlinux.strip -envfile=u-boot-env -memsize=2048 -uart0=2020 -noperf -numcores=1 -quiet
#terminal2:
$telnet localhost 2020
```

# tftp vmlinux
```sh
Octeon evb7000_sff# dhcp
Octeon evb7000_sff# tftp 192.168.1.99:vmlinux.strip
Octeon evb7000_sff# bootoctlinux $(loadaddr) numcores=$(numcores) mem=2G
```


# tftpboot
```sh
#安装和配置atftpd
$apt install atftpd
$mkdir -p /home/byj/tmp/tftpboot
$sudo chmod -R 777 /home/byj/tmp/tftpboot
$sudo chown -R nobody /home/byj/tmp/tftpboot
$sudo vi /etc/default/atftpd
#填下面的
USE_INETD=false
OPTIONS="--daemon --tftpd-timeout 300 --retry-timeout 5 --mcast-port 1758 --mcast-addr 239.239.239.0-255 --mcast-ttl 1 --maxthread 100 --verbose=5 /home/byj/tmp/tftpboot"
$sudo invoke-rc.d atftpd start
#或者
$sudo /etc/init.d/atftpd start
#重新配置的话需要
$sudo /etc/init.d/atftpd restart
#停止
$sudo /etc/init.d/atftpd stop


#板子上
#配网口
Octeon evb7000_sff# dhcp
Octeon evb7000_sff# tftp 0 192.168.1.16:filename
```


# 运行iperf测试
```sh
#服务器上
[yingjie@aircraft temp]$ iperf3 -s -p 12345 -i 1

#板子上
root@octeon:~# iperf3 -c 192.168.1.99 -p 12345 -i 1 -t 10 -f M
```

# debian 打开dhcp client动态获取ip
https://wiki.debian.org/NetworkConfiguration
```sh
root@octeon:~# dhclient
#查看路由是否正确
root@octeon:~# route
root@octeon:~# ip route
```

# 查kernel版本（编译时）
```sh
#内核源码树下
make kernelrelease ARCH=mips CROSS_COMPILE=mips64-octeon-linux-gnu-
```

## linux编译
```sh
cd linux
byj@byj-mint ~/repo/hg/OCTEON-SDK-3.1/linux
$make help
grep: kernel/linux/.config: No such file or directory
Supply the build target:
    kernel - Build the Linux kernel supporting all Cavium Octeon reference boards
    kernel-deb - Linux kernel without the rootfs
    sim - Octeon simulation environment
    setup-octeon2 - Enable config options for running on OcteonII hardware
    setup-octeon2-sim - Enable config options for running on OcteonII simulation
    flash - Copy kernel onto compact flash at mount /mnt/cf1
    strip - Strip symbols out of the kernel image
    tftp - Copy a stripped kernel to /tftpboot
    test - Test an existing simulator build
    clean - Remove all generated files and the KERNEL CONFIG
sudo make kernel-deb
make menuconfig ARCH=mips
```

# 只编译debian目录
```sh
#需要进入root模式
#之前编译过kernel-deb（或者在内核源码树里面make vmlinux.64 modules ARCH=mips CROSS_COMPILE=mips64-octeon-linux-gnu-）
cd linux/debian/
make target-dir
#现在在当前目录下会有target-dir目录，这个就是debian的rootfs
#然后的问题是，如何把debian这个rootfs集成到kernel里？
#参考gentoo的这篇文档 http://wiki.gentoo.org/wiki/Custom_Initramfs
#进入linux/kernel/linux
make menuconfig ARCH=mips 注意要加ARCH！！！！！
-->General setup
      -->[*]Initial RAM filesystem and RAM disk (initramfs/initrd) support
      -->填好rootfs的目录（../../debian/target-dir）


#最后结果是：
#太大了编不下！debian目录target-dir有1.7个G，压缩后的cpio还有500M
root@byj-mint /home/byj/repo/hg/OCTEON-SDK-3.1-pristine/linux/kernel/linux
#llh usr/
total 1.5G
-rw-r--r-- 1 root root 511M 9月 21 21:50 built-in.o
-rwxr-xr-x 1 root root 19K 9月 21 21:13 gen_init_cpio
-rw-r--r-- 1 byj byj 13K 9月 13 11:32 gen_init_cpio.c
-rw-r--r-- 1 root root 511M 9月 21 21:49 initramfs_data.cpio.gz
-rw-r--r-- 1 root root 511M 9月 21 21:49 initramfs_data.o
-rw-r--r-- 1 byj byj 1.3K 9月 13 11:32 initramfs_data.S
-rw-r--r-- 1 byj byj 5.4K 9月 13 11:32 Kconfig
-rw-r--r-- 1 byj byj 2.4K 9月 13 11:32 Makefile
-rw-r--r-- 1 root root 0 9月 21 21:22 modules.builtin
-rw-r--r-- 1 root root 0 9月 21 21:50 modules.order
```

# debain启动runlevel 0
```
在bootoctlinux 0 root=/dev/sda2 之后紧接着传一个runlevel参数就可以了， 就是那个3
cat /proc/cmdline
bootoctlinux 0 root=/dev/sda2 3 console=ttyS0,115200
```

# 使用分离的cpio做initramfs
```sh
$ cd linux
$ make kernel-deb
$ cp kernel/linux/vmlinux.64 /var/lib/tftpboot/
$ cd linux/embedded_rootfs
$ make initramfs
$ ls -l rootfs.cpio.gz | gawk -- '{printf "%x\n", ($5 + 0x10000) - ($5 % 0x10000)}'
2e0000
$ cp rootfs.cpio.gz /var/lib/tftpboot/
Octeon ebh5600# freeprint


Printing bootmem block list, descriptor: 0x0000000000024108, head is 0x00000000081c0940
Descriptor version: 3.0
Block address: 0x00000000081c0940, size: 0x0000000007e2d2c0, next: 0x0000000026000000
Block address: 0x0000000026000000, size: 0x00000000da000000, next: 0x0000000410000000
Block address: 0x0000000410000000, size: 0x000000000ff00000, next: 0x0000000000000000
 
Octeon ebh5600# namedalloc my_initrd 0x2e0000 0x81d0000
Allocated 0x00000000002e0000 bytes at address: 0x00000000081d0000, name: my_initrd
 
Octeon ebh5600# tftpboot 0x81d0000 rootfs.cpio.gz
Using octmgmt0 device
TFTP from server 10.1.1.1; our IP address is 10.2.0.33
Filename 'rootfs.cpio.gz'.
Load address: 0x81d0000
Loading: #####################
done
Bytes transferred = 3000755 (2dc9b3 hex), 6398 Kbytes/sec
WARNING: Data loaded outside of the reserved load area, memory corruption may occur.
WARNING: Please refer to the bootloader memory map documentation for more information.
 
Octeon ebh5600# tftpboot $(loadaddr) vmlinux.64
Using octmgmt0 device
TFTP from server 10.1.1.1; our IP address is 10.2.0.33
Filename 'vmlinux.64'.
Load address: 0x20000000
Loading: #################################################################
     #################################################################
     #################################################################
     #################################################################
     #################################################################
     #################################################################
     ###############################
done
Bytes transferred = 60285919 (397e3df hex), 6930 Kbytes/sec
 
Octeon ebh5600# bootoctlinux $(loadaddr) numcores=$(numcores) endbootargs rd_name=my_initrd mem=1024M
argv[2]: numcores=12
argv[3]: endbootargs
ELF file is 64 bit
Attempting to allocate memory for ELF segment: addr: 0xffffffff81100000 (adjusted to: 0x0000000001100000), size 0x9fc080
Allocated memory for ELF segment: addr: 0xffffffff81100000, size 0x9fc080
Processing PHDR 0
  Loading 984c80 bytes at ffffffff81100000
  Clearing 77400 bytes at ffffffff81a84c80
## Loading Linux kernel with entry point: 0xffffffff81105f90 ...
Bootloader: Done loading app on coremask: 0xfff
Linux version 2.6.32.13-Cavium-Octeon (hello@xyz.zzz) (gcc version 4.3.3 (Cavium Inc. Development Version) ) #251 SMP Tue Jun 15 17:27:41 PDT 2010
CVMSEG size: 2 cache lines (256 bytes)
bootconsole [early0] enabled
CPU revision is: 000d0409 (Cavium Octeon+)
Checking for the multiply/shift bug... no.
Checking for the daddiu bug... no.
Determined physical RAM map:
memory: 00000000002e0000 @ 00000000081d0000 (usable after init)
memory: 000000000004a000 @ 0000000001a46000 (usable)
memory: 0000000006400000 @ 0000000001b00000 (usable)
memory: 0000000007800000 @ 0000000008500000 (usable)
memory: 0000000032400000 @ 0000000020000000 (usable)
Wasting 376656 bytes for tracking 6726 unused pages
Initial ramdisk at: 0xa8000000081d0000 (3014656 bytes)
Zone PFN ranges:
  Normal 0x00001a46 -> 0x00052400
Movable zone start PFN for each node
early_node_map[5] active PFN ranges
```
