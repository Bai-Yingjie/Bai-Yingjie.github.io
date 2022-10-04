- [CentOS换kernel](#centos换kernel)
- [环境](#环境)
- [改grub](#改grub)
- [OVS-DPDK](#ovs-dpdk)
- [virt-manager](#virt-manager)
- [libvirt](#libvirt)
- [qemu](#qemu)
- [pktgen](#pktgen)
- [testpmd](#testpmd)
- [nfs mount](#nfs-mount)
- [perf ping](#perf-ping)
- [perf ovs](#perf-ovs)
- [OVS](#ovs)
- [记录线程30秒](#记录线程30秒)
- [内存屏障](#内存屏障)
- [查看详细的cache统计](#查看详细的cache统计)
- [hugepage](#hugepage)
  - [DPDK会用尽所有hugepage](#dpdk会用尽所有hugepage)
  - [kernel cmdline方式(old)](#kernel-cmdline方式old)

# CentOS换kernel
```
sudo rpm -ivh kernel-devel-4.14.62-5.hxt.aarch64.rpm kernel-debuginfo-4.14.62-5.hxt.aarch64.rpm kernel-debuginfo-common-aarch64-4.14.62-5.hxt.aarch64.rpm
```

# 环境
服务器 | Host A 10.129.41.129 | Host B 10.129.41.130 | Note
-----|-----|-----|-----|
Socket| 1 | 1 | 单socket
CPU | 48core@2.6G HXT1.0 | 46core@2.6G HXT1.0 | `sudo dmidecode -t processor`
MEM | 96G | 96G | `free -h`
NIC | MLX CX4121A 10G 0004:01:00.0 enP4p1s0f0| MLX CX4121A 10G 0004:01:00.0 enP4p1s0f0| `ibdev2netdev -v`
NIC | MLX CX4121A 10G 0004:01:00.1 enP4p1s0f1| MLX CX4121A 10G 0004:01:00.1 enP4p1s0f1| `ibdev2netdev -v`
OS | CentOS 7.5.1804 | CentOS 7.5.1804 | `cat /etc/redhat-release`
kernel | 4.14.62-5.hxt.aarch64 | 4.14.62-5.hxt.aarch64 | `uname -r`
Mellanox OFED version | 4.4-1.0.0.0 | 4.4-1.0.0.0 | `ofed_info -s`
QEMU version | NA | 2.12.1 | `qemu-system-aarch64 --version` 源码编译
DPDK version | 17.11.4 | 17.11.4 | 源码编译
pktgen version |3.4.9 | NA | 源码编译
OVS(with DPDK) version | NA | 2.9.3 | `sudo ovs-vsctl show` 源码编译
libvirt version | NA | 4.6.0 | 源码编译
virt-manager version | NA | 1.5.1 | 源码安装

# 改grub
```
sudo vi /etc/default/grub
sudo grub2-mkconfig  -o /boot/efi/EFI/centos/grub.cfg
```

# OVS-DPDK
```
sudo bash -c "echo 64 > /sys/kernel/mm/hugepages/hugepages-524288kB/nr_hugepages"
export PATH=$PATH:/usr/local/share/openvswitch/scripts
sudo ovs-ctl start
sudo ovs-ctl stop
```

# virt-manager
```bash
cd ~/repo/hxt/packages/virt-manager-1.5.1
sudo xauth add $(xauth -f /home/bai/.Xauthority list | tail -1)
sudo ./virt-manager
#启动一个新的vm
sudo tools/virsh create ~/repo/save/vm/2vhostuser.xml
```

# libvirt
```
cd ~/repo/hxt/packages/libvirt-4.6.0
sudo src/virtlogd &
sudo src/libvirtd &
```

# qemu
```
sudo qemu-system-x86_64 -m 1024 -smp 4 -cpu host -hda /home/user/us17_04vm1.qcow2 -boot c -enable-kvm -no-reboot -net none -nographic \
-chardev socket,id=char1,path=/run/openvswitch/vhost-user1 \
-netdev type=vhost-user,id=mynet1,chardev=char1,vhostforce \
-device virtio-net-pci,mac=00:00:00:00:00:01,netdev=mynet1 \
-object memory-backend-file,id=mem,size=1G,mem-path=/dev/hugepages,share=on 
-numa node,memdev=mem -mem-prealloc \
-virtfs local,path=/home/user/iperf_debs,mount_tag=host0,security_model=none,id=vm1_dev

	<interface type='vhostuser'>
      <mac address='52:54:00:38:fe:8c'/>
      <source type='unix' path='/usr/local/var/run/openvswitch/dpdkvhostuser0' mode='client'/>
      <model type='virtio'/>
      <driver queues='2'>
        <host mrg_rxbuf='on'/>
      </driver>
      <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x0'/>
    </interface>
	
    <interface type='vhostuser'>
      <mac address='52:54:00:f0:25:e4'/>
      <source type='unix' path='/usr/local/var/run/openvswitch/dpdkvhostuser1' mode='client'/>
      <model type='virtio'/>
      <driver queues='2'>
        <host mrg_rxbuf='on'/>
      </driver>
      <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x0'/>
    </interface>
```

# pktgen
```
sudo bash -c "echo 16 > /sys/kernel/mm/hugepages/hugepages-524288kB/nr_hugepages"
cd ~/repo/hxt/pktgen/pktgen-dpdk-pktgen-3.4.5 
#运行, [42:43].0的意思是rx用core42,tx用core43, core41为显示和timer 
#-T是打开颜色, -P是混杂模式 
sudo app/arm64-armv8a-linuxapp-gcc/pktgen -l 41-45 -w 0005:01:00.0 -- -T -P -m "[42:43].0"

Pktgen:/pktgen/bin/> stop all
Pktgen:/pktgen/bin/> set 0 dst mac 01:00:00:00:00:01
Pktgen:/pktgen/bin/> set 0 dst ip 1.0.0.1
```

# testpmd
```
sudo ./build/app/testpmd -l 36-41 -m 512 --file-prefix pg1 -w 0004:01:00.0 --proc-type auto txq_inline=256,rxq_cqe_comp_en=1,txqs_min_inline=8,txq_mpw_en=1 -- -i
sudo ./build/app/testpmd -l 42-45 -m 512 --file-prefix pg2 -w 0005:01:00.0 --proc-type auto txq_inline=256,rxq_cqe_comp_en=1,txqs_min_inline=8,txq_mpw_en=1 -- -i

sudo app/arm64-armv8a-linuxapp-gcc/pktgen -l 41-45 -w 0005:01:00.0 -- -T -P -m "[42:43].0"

135      ./app/app/${target}/pktgen -l 2-11 -n 3 --proc-type auto \
136         --socket-mem 512,512 --file-prefix pg1 \
137         -b 09:00.0 -b 09:00.1 -b 83:00.1 -b 06:00.0 \
138         -b 06:00.1 -b 08:00.0 -b 08:00.1 -- \
139         -T -P -m "[4:6].0, [5:7].1, [8:10].2, [9:11].3" \
140         -f themes/black-yellow.theme
141
142      ./app/app/${target}/pktgen -l 2,4-11 -n 3 --proc-type auto \
143         --socket-mem 512,512 --file-prefix pg2 \
144         -b 09:00.0 -b 09:00.1 -b 83:00.1 -b 87:00.0 \
145         -b 87:00.1 -b 89:00.0 -b 89:00.1 -- \
146         -T -P -m "[12:16].0, [13:17].1, [14:18].2, [15:19].3" \
147         -f themes/black-yellow.theme


-e L1-dcache-loads,L1-dcache-load-misses,L1-dcache-stores,L1-dcache-store-misses,branch-loads,branch-load-misses,qcom_pmuv3_0/l2d_cache/,qcom_pmuv3_0/l2d_cache_allocate/


stop
port stop all
port config all txd 1024
port config all rxd 1024
port start all
start
show port stats all
```

# nfs mount
```
sudo yum install nfs-utils
sudo umount -f -l /home/bai/share

sudo mount 10.64.17.45:/home/bai/share /home/bai/share
```

# perf ping
```
sudo perf record -g ping 5.5.5.1
sudo perf script | ~/repo/FlameGraph/stackcollapse-perf.pl | ~/repo/FlameGraph/flamegraph.pl > ping-localhost.svg

watch -n1 -d 'sudo ethtool -S enP5p1s0 | grep -v ": 0" | grep packets'
```

# perf ovs
```
sudo perf probe -x `which ovs-vswitchd` --add netdev_linux_send
sudo perf probe -x `which ovs-vswitchd` --add netdev_linux_tap_batch_send
sudo perf probe --add dev_hard_start_xmit


sudo perf record -e probe_ovs:netdev_linux_send -e probe_ovs:netdev_linux_tap_batch_send -e probe:dev_hard_start_xmit -e net:net_dev_start_xmit -p 3470 -g -o perf-ping22.data -- sleep 30

perf probe -x /lib64/libibverbs.so.1 -F

perf probe -x `which ovs-vswitchd` -F | grep ibv

sudo perf report -n

sudo ovs-appctl dpif/show

sudo tcpdump -i extbr0 --direction=out
```

# OVS

ovs-dpdk收发包代码
The existing netdev implementations may serve as useful examples during a port:
• lib/netdev-linux.c implements netdev functionality for Linux network devices, using Linux kernel calls. It may be a good place to start for full-featured netdev implementations.
ovs-dpdk 把enP4p1s0作为一个system接口来处理，通过raw socket从userpace层面从enP4p1s0收发报文,还是走的userpace datapath,最终报文是通过enp4P1s0的drive收发的。

ovs-dpdk 收报文
1.	main() 循环执行type_run()  -> dpif_run() -> dpif_netdev_run()   !netdev_is_pmd(port->netdev))--- dp_netdev_process_rxq_port()  //system port不会启pmd thread

2.	dp_netdev_process_rxq_port()
•	netdev_rxq_recv()->rx->netdev->netdev_class->rxq_recv(rx,batch) -=>netdev_linux_rxq_recv() -> netdev_linux_rxq_recv_sock()从socket绑定的eth口上收报文
•	dp_netdev_input() -> dp_netdev_input__() : 查emc，dpcls，upcall的流程
•	dp_netdev_pmd_flush_output_packets()-> dp_netdev_pmd_flush_output_on_port()->netdev_send() //从egress port出报

ovs-dpdk发报文(如果egress port是system接口)
netdev_send ()->netdev_linux_send()  netdev_linux_sock_batch_send() 调用raw socket直接从eth口发出去

# 记录线程30秒
```
sudo perf record -g -t 3546 -- sleep 30
```

# 内存屏障
~/share/repo/hxt/linux/Documentation/memory-barriers.txt

# 查看详细的cache统计
```bash
#注意是三个-d
sudo perf stat -d -d -d -C 38
```

# hugepage
```bash
#检查hugepage的配置
$ grep Huge /proc/meminfo
AnonHugePages: 0 kB
ShmemHugePages: 0 kB
HugePages_Total: 32
HugePages_Free: 8
HugePages_Rsvd: 0
HugePages_Surp: 0
Hugepagesize: 524288 kB
```
需要修改的话, 用root执行:
`sudo bash -c "echo 64 > /sys/kernel/mm/hugepages/hugepages-524288kB/nr_hugepages"`
这里我们为64个512M的hugepage预留了32G的内存,用free命令能够看到, 这32G的内存被算做"used"
```
$ free -h
              total used free shared buff/cache available
Mem: 95G 33G 61G 50M 1.0G 54G
Swap: 4.0G 0B 4.0G
```
hugetable需要mount为hugetlbfs才能够使用, 一般的教程会说
`mount -t hugetlbfs nodev /mnt/huge`
但在centos7.5上, 已经有个systemd的unit专门干这事, 以后就用`/dev/hugepages`就行
```bash
$ systemctl status dev-hugepages.mount
● dev-hugepages.mount - Huge Pages File System
   Loaded: loaded (/usr/lib/systemd/system/dev-hugepages.mount; static; vendor preset: disabled)
   Active: active (mounted) since Mon 2018-07-30 08:49:38 CST; 3 weeks 0 days ago
    Where: /dev/hugepages
     What: hugetlbfs
     Docs: https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt
           http://www.freedesktop.org/wiki/Software/systemd/APIFileSystems
  Process: 869 ExecMount=/bin/mount hugetlbfs /dev/hugepages -t hugetlbfs (code=exited, status=0/SUCCESS)
    Tasks: 0
```
## DPDK会用尽所有hugepage
而且推出时并不会release这些大页, 在`/dev/hugepages`里面`rtemap_`就是, 整好是配的64个大页.
```
bai@CentOS-43 /dev/hugepages
$ ls
libvirt    rtemap_11  rtemap_15  rtemap_19  rtemap_22  rtemap_26  rtemap_3   rtemap_33  rtemap_37  rtemap_40  rtemap_44  rtemap_48  rtemap_51  rtemap_55  rtemap_59  rtemap_62  rtemap_9
rtemap_0   rtemap_12  rtemap_16  rtemap_2   rtemap_23  rtemap_27  rtemap_30  rtemap_34  rtemap_38  rtemap_41  rtemap_45  rtemap_49  rtemap_52  rtemap_56  rtemap_6   rtemap_63
rtemap_1   rtemap_13  rtemap_17  rtemap_20  rtemap_24  rtemap_28  rtemap_31  rtemap_35  rtemap_39  rtemap_42  rtemap_46  rtemap_5   rtemap_53  rtemap_57  rtemap_60  rtemap_7
rtemap_10  rtemap_14  rtemap_18  rtemap_21  rtemap_25  rtemap_29  rtemap_32  rtemap_36  rtemap_4   rtemap_43  rtemap_47  rtemap_50  rtemap_54  rtemap_58  rtemap_61  rtemap_8
```
但不会影响DPDK下次运行, 它会直接找`rtemap_`使用.
手动删除这些文件会把大页还回给系统
用`pmap -x`查看testpmd进程, 可以看到类似下面的map
```
Address           Kbytes     RSS   Dirty Mode  Mapping
0000fff780000000  524288       0       0 rw-s- /dev/hugepages/rtemap_63
0000fff7a0000000  524288       0       0 rw-s- /dev/hugepages/rtemap_62
```

## kernel cmdline方式(old)
```bash
#reserve 4G, 每个1G, 如果系统有两个CPU node, 好像是平分
default_hugepagesz=1G hugepagesz=1G hugepages=4
```