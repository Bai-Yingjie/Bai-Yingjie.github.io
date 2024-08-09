- [uboot调试](#uboot调试)
  - [setenv](#setenv)
  - [在uboot下更新uboot](#在uboot下更新uboot)
  - [fdt命令](#fdt命令)
  - [load CF卡](#load-cf卡)
    - [找到sda1分区的offset, 在pc上mount](#找到sda1分区的offset-在pc上mount)

# uboot调试
## setenv
```shell
#增加reboot=0
setenv kernel_extra_args config_overlay=reboot=0

#修改波特率
setenv baudrate 115200
#设置完成后, uboot会自动把115200传给kernel:console=ttyS1,115200
```

## 在uboot下更新uboot
```c
#define PKGB_FIT_IMAGE_ADDR 0xffe00000
#define PKGA_FIT_IMAGE_ADDR 0xffd20000
#define UBOOT_FIT_IMAGE_SIZE 0x000e0000

ext2load isam_cf 0:0 0x10000000 /images/uboot/uboot3041.itb
//其实不需要protect off
protect off all
#ubootB
erase 0xffe00000 +0x000e0000
cp.b 0x10000000 0xffe00000 0x000e0000

#ubootA
erase 0xffd20000 +0x000e0000
cp.b 0x10000000 0xffd20000 0x000e0000
```

## fdt命令
```
fdt addr 0x11000000
fdt print /memory
```

## load CF卡
4G CF卡, 在linux起来后, 看到两个dev, 各有一个分区:  
`/dev/sda   /dev/sda1  /dev/sdb   /dev/sdb1`  
分区都是ext4格式. 分区表是mbr格式
```
/dev/sda1 on /mnt/cf type ext4 (rw,relatime,data=ordered)
/dev/sdb1 on /mnt/cf2 type ext4 (rw,relatime,data=ordered)
```
在uboot阶段, 会从cf卡load linux.itb
```
load_image_from_cf_card: load image /images/linuxA/linux.itb to 7a000000 done, size 14013800
prepare_images: load all images from fit sections
do_load_images: Kernel section at 0x7a0002a4 size=0x373d07
do_load_images: Current dtb name: dtb3041
do_load_images: Dtb section at 0x7a374090 size=0x95a7
do_load_images: Rootfs section at 0x7a386ea8 size=0x9d61c8
do_load_images: Copy rootfs to ram 0xc10000
```

这个linux.itb有几个部分, 是`vobs/esam/build/lsfx-a/OS/images/fant-f/linux.its`定义好的
* ./linux.itb.version 0.2K
* ./uImage 3.4M
* ./fant-f-y-3041.dtb 38K
* ./fant-f-y-4040.dtb 39K
* ./rootfs.cpio.xz 11M

----
物理上, 只有一个CF卡. 出现两个dev, 是dts里面配的:
* sda是高2G: 起始字节地址是 0x400000 * 512
* sdb是低2G: 起始字节地址是 0x1ec0 * 512

```c
/* compact-flash */
cf@1,0 {
    compatible = "alu,cf";
    reg = < 1 0x0800 0x0008    /* command register base */
            1 0x080e 0x0002    /* control register      */
          >;
    reg-swap = <1>;            /* data bus swapped      */

    /* isam_cf driver will number
     * the 1st present logic dev as sda,
     * the 2nd present logic dev ad sdb,
     * ...
     * We'd like to see the high 2G is taken as sda since the low 2G
     * will not be taken immediately by Linux during the migration.
     *
     */
    /* sda */
    cf-device@0 {
        /* This covers the whole high 2G */
        compatible = "alu,cf-device";
        /* first LBA sector (if ommitted, from end of previous
        * virtual device or zero if first virtual device) */
        start-sector = <0x400000>;
        /* last LBA sector (if ommitted, till end of physical device) */
        /* end-sector = <599999>; */
    };

    /* sdb */
    cf-device@1 {
        /* This covers the /dev/hda1-7(legacy Integrity) in low 2G */
        compatible = "alu,cf-device";
        start-sector = <0x1ec0>; /* this is the start LBA of /dev/hda1 */
        end-sector = <0x3d011f>; /* this is the end   LBA of /dev/hda7 */
    };
};
```

### 找到sda1分区的offset, 在pc上mount
```
fdisk -l /dev/sda
Disk /dev/sda: 1871 MB, 1962704896 bytes, 3833408 sectors
238 cylinders, 255 heads, 63 sectors/track
Units: sectors of 1 * 512 = 512 bytes

Device  Boot StartCHS    EndCHS        StartLBA     EndLBA    Sectors  Size Id Type
/dev/sda1    0,1,1       237,254,63          63    3823469    3823407 1866M 83 Linux
```
这里是说, sda1分区从sector 63开始, 到sector 3823469结束, 共3823407个

那么, 因为sda是cf卡的高2G, 那sda1所在的分区的offset是:
(0x400000 + 63) * 512 = 2147515904

在pc上mount这个分区
```shell
#按照计算好的分区offset来mount
sudo mount -o offset=2147515904 /dev/sdb mnt
#能看到文件
ls mnt/images/
cf_low_2g_reclaim  linuxA  linuxB

#用root用户权限, 拷贝linux.itb
cd mnt/images/linuxA/
cp /home/yingjieb/work/share/swms/vobs/esam/build/lsfx-a/OS/images/fant-f/linux.itb .
sync
```

参考: [https://superuser.com/questions/694430/how-to-inspect-disk-image](https://superuser.com/questions/694430/how-to-inspect-disk-image)

