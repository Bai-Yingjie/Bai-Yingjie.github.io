- [一些文档](#一些文档)
  - [kvm xen virtualbox比较](#kvm-xen-virtualbox比较)
  - [ubuntu guide](#ubuntu-guide)
  - [suse doc 非常详尽](#suse-doc-非常详尽)
- [qemu使用](#qemu使用)
  - [kvm只是个qemu的封装](#kvm只是个qemu的封装)
  - [安装](#安装)
  - [启动命令](#启动命令)
  - [从virtualbox迁移到kvm](#从virtualbox迁移到kvm)
  - [关于-nodefaults](#关于-nodefaults)
  - [vnc](#vnc)
  - [spice配合-vga qxl](#spice配合-vga-qxl)
  - [qemu gdb](#qemu-gdb)

# 一些文档
## kvm xen virtualbox比较
http://www.phoronix.com/scan.php?page=article&item=intel_haswell_virtualization&num=1

## ubuntu guide
https://help.ubuntu.com/community/KVM  
http://www.howtogeek.com/117635/how-to-install-kvm-and-create-virtual-machines-on-ubuntu/

## suse doc 非常详尽
http://doc.opensuse.org/products/draft/SLES/SLES-kvm_sd_draft/book.kvm.html  
http://doc.opensuse.org/products/draft/SLES/SLES-kvm_sd_draft/cha.qemu.guest_inst.html#cha.qemu.guest_inst.qemu-kvm  
http://doc.opensuse.org/products/draft/SLES/SLES-kvm_sd_draft/cha.qemu.running.html

# qemu使用
## kvm只是个qemu的封装
```shell
/usr/bin/kvm -nographic -monitor stdio -hda windowsC.img -m 2048 -smp 2 -cdrom Win2003.iso -boot c -soundhw es1370 -no-acpi -localtime -usb -usbdevice tablet -net nic,vlan=0,model=rtl8139 -net tap,vlan=0,ifname=tap0
```
```shell
$ cat /usr/bin/kvm
#! /bin/shexec
qemu-system-x86_64 -enable-kvm "$@"
```

## 安装
```shell
#检查cpu是否支持虚拟化
egrep -c ‘(svm|vmx)’ /proc/cpuinfo
#安装
sudo apt-get install qemu-kvm libvirt-bin bridge-utils virt-manager virt-viewer
#加入组
sudo adduser `id -un` libvirtd
#查看当前虚拟机
virsh -c qemu:///system list
```

## 启动命令
```
qemu-kvm -name "sles11" -M pc-0.12 -m 768 \
-smp 2 -boot c \
-drive file=/images/sles11/hda,if=virtio,index=0,media=disk,format=raw \
-net nic,model=virtio,macaddr=52:54:00:05:11:11 \
-vga cirrus -balloon virtio
-writeconfig cfg_file

qemu-system-x86_64 -enable-kvm -name ubuntutest -m 2048 -balloon virtio -hda ubuntutest.qcow2 -vnc :19 -net nic,model=virtio -net tap,ifname=tap0,script=no,downscript=n -monitor stdio

qemu-system-x86_64 -enable-kvm -name ubuntutest -m 2048 -balloon virtio -drive file=ubuntutest.qcow2,if=virtio -vnc :19 -net nic,model=virtio -net tap,ifname=tap0,script=no,downscript=n -monitor stdio
```

## 从virtualbox迁移到kvm
先转换磁盘格式, 即使是raw格式, 注意只要host文件系统支持hole, 也只会占用实际大小.  
比如我这个win7磁盘是64G的, 转换以后win7.raw只有19G, 其余的空间并不占用host磁盘. 但ls -l显示这个文件还是64G
`qemu-img convert win7.vdi -O raw win7.raw`

## 关于-nodefaults
默认的q35机器配置
```
#     00.0 - Host bridge
#     1f.0 - ISA bridge / LPC
#     1f.2 - SATA (AHCI) controller
#     1f.3 - SMBus controller
```
但启动qemu的时候, qemu会根据传入的参数, 添加默认的device. 但如果参数指定了具体的同类设备, 默认的就没了.

## vnc
带vnc参数时, 需要vnc到指定端口才有"视频"输出  
比如 -vnc :1 注:vnc的默认端口从5900开始  
另开一个窗口  
gvncviewer 0.0.0.0:1  
或gvncviewer 127.0.0.1:1

## spice配合-vga qxl
```
kvm -name GentooGuest -machine q35,usb=off -cpu host -smp 4 -m 2048 -realtime mlock=off -balloon virtio -drive file=gentoo-rootfs.ext4,if=virtio -vga qxl -nographic -spice port=5900,addr=127.0.0.1,disable-ticketing -writeconfig gentooguest.cfg
```
需要开个spice client  
```
apt install spice-client
spicec -h 127.0.0.1 -p 5900
```
按shift+f12退出guest窗口

## qemu gdb

In order to use gdb, launch QEMU with the '-s' option. It will wait for a gdb connection:
```
qemu-system-i386 -s -kernel arch/i386/boot/bzImage -hda root-2.4.20.img \
    -append "root=/dev/hda"
```
Connected to host network interface: tun0
Waiting gdb connection on port 1234
Then launch gdb on the 'vmlinux' executable:
```
> gdb vmlinux
```
In gdb, connect to QEMU:
```
(gdb) target remote localhost:1234
```
Then you can use gdb normally. For example, type 'c' to launch the kernel:
```
(gdb) c
```
Here are some useful tips in order to use gdb on system code:
Use `info reg` to display all the CPU registers.  
Use `x/10i $eip` to display the code at the PC position.  
Use `set architecture i8086` to dump 16 bit code. Then `usex/10i $cs*16+$eip` to dump the code at the PC position.