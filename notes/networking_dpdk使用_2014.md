- [对内核的要求](#对内核的要求)
- [hugepage的用法](#hugepage的用法)
- [编译](#编译)
- [UIO与VFIO](#uio与vfio)
- [unbind与bind](#unbind与bind)
- [HPET 高精度事件时钟](#hpet-高精度事件时钟)
- [如何防止业务core还被系统用来跑其他任务](#如何防止业务core还被系统用来跑其他任务)
- [当成一个网口](#当成一个网口)
- [关于IOMMU](#关于iommu)
- [一键运行](#一键运行)

# 对内核的要求
* UIO
* HUGETLBFS
* PROC_PAGE_MONITOR support 
* HPET and HPET_MMAP

# hugepage的用法
step1:
* 通过kernel command line  
在boot时  
```c
//reserve 1024个默认2M的
hugepages=1024
//reserve 4G, 每个1G
default_hugepagesz=1G hugepagesz=1G hugepages=4
```
注意: 如果系统有两个CPU node, 好像是平分
* 或者在系统起来以后
```shell
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
#如果有2个CPU
echo 1024 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
echo 1024 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
```
注意: 1G的页只能在启动时reserve

step2:
```shell
mkdir /mnt/huge
mount -t hugetlbfs nodev /mnt/huge
#或者在/etc/fstab里加
nodev /mnt/huge hugetlbfs defaults 0 0
#如果要用1G的
nodev /mnt/huge_1GB hugetlbfs pagesize=1GB 0 0
```

# 编译
```shell
make install T=x86_64*gcc
#只配置
make config T=x86_64-native-linuxapp-gcc
#编译完了有
$ ls x86_64-native-linuxapp-gcc
app build hostapp include kmod lib Makefile
```

# UIO与VFIO
* 以前是用UIO
```
sudo modprobe uio
sudo insmod kmod/igb_uio.ko
```
* 1.7版本以后有VFIO, 要求kernel版本在3.6.0以上; 还要求系统和bios支持IO虚拟化, 比如Intel VT-d
```
sudo modprobe vfio-pci
```

# unbind与bind
一个device要先从其他module unbind, 再bind到igb_uio或vfio-pci上
DPDK用`dpdk_nic_bind.py`脚本  
VFIO有几个限制:
* VF只能作用于VF本身
* PF要求所有的VF都bind到VFIO上
* 如果要用的device在桥后面, 因为这个桥就是一个IOMMU的group或者说domain, 所以这个桥必须从桥的驱动unbind, 然后bind到VFIO上

# HPET 高精度事件时钟
用这个命令查看系统是否支持
```
# grep hpet /proc/timer_list
```

# 如何防止业务core还被系统用来跑其他任务
在kenel启动参数加以下参数, 告诉kernel调度任务时不要用哪些core
`isolcpus=2,4,6`

# 当成一个网口
```
# insmod kmod/rte_kni.ko
```

# 关于IOMMU
内核需要配置以下
```
IOMMU_SUPPORT
IOMMU_API
INTEL_IOMMU
```
igb_uio要求在kernel命令行加`iommu=pt`, 使用pass through模式, 不查DMAR表; 而且, 如果内核没配`INTEL_IOMMU_DEFAULT_ON`, 还要加`intel_iommu=on`
而vfio-pci没有这些要求, `iommu=pt`和`iommu=on`都能工作

# 一键运行
setup.sh, 这个脚本完成
* Build the Intel® DPDK libraries
* Insert and remove the Intel ® DPDK IGB_UIO kernel module
* Insert and remove VFIO kernel modules
* Insert and remove the Intel ® DPDK KNI kernel module
* Create and delete hugepages for NUMA and non-NUMA cases
* View network port status and reserve ports for Intel® DPDK application use
* Set up permissions for using VFIO as a non-privileged user
* Run the test and testpmd applications
* Look at hugepages in the meminfo
* List hugepages in /mnt/huge
* Remove built Intel® DPDK libraries

用root运行
```
user@host:~/rte$ source tools/setup.sh
-----------------------------------------------------------------
RTE_SDK exported as /home/user/rte
-----------------------------------------------------------------
----------------------------------------------------------
Step 1: Select the DPDK environment to build
----------------------------------------------------------
[1] i686-native-linuxapp-gcc
[2] i686-native-linuxapp-icc
[3] x86_64-ivshmem-linuxapp-gcc
[4] x86_64-ivshmem-linuxapp-icc
[5] x86_64-native-bsdapp-gcc
[6] x86_64-native-linuxapp-gcc
[7] x86_64-native-linuxapp-icc
----------------------------------------------------------
Step 2: Setup linuxapp environment
----------------------------------------------------------
[8] Insert IGB UIO module
[9] Insert VFIO module
[10] Insert KNI module
[11] Setup hugepage mappings for non-NUMA systems
[12] Setup hugepage mappings for NUMA systems
[13] Display current Ethernet device settings
[14] Bind Ethernet device to IGB UIO module
[15] Bind Ethernet device to VFIO module
[16] Setup VFIO permissions
----------------------------------------------------------
Step 3: Run test application for linuxapp environment
----------------------------------------------------------
[17] Run test application ($RTE_TARGET/app/test)
[18] Run testpmd application in interactive mode ($RTE_TARGET/app/testpmd)
----------------------------------------------------------
Step 4: Other tools
----------------------------------------------------------
[19] List hugepage info from /proc/meminfo
----------------------------------------------------------
Step 5: Uninstall and system cleanup
----------------------------------------------------------
[20]
 Uninstall all targets
[21]
 Unbind NICs from IGB UIO driver
[22]
 Remove IGB UIO module
[23]
 Remove VFIO module
[24]
 Remove KNI module
[25]
 Remove hugepage mappings
[26] Exit Script
```