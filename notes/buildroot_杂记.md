- [buildroot修改包代码后编译](#buildroot修改包代码后编译)
- [常用命令](#常用命令)
- [重编linux](#重编linux)
- [OCTEON SDK的kernelconfig](#octeon-sdk的kernelconfig)

# buildroot修改包代码后编译
比如我想修改util-linux这个包的文件  
在第一次编译完后, 直接修改`buildrootPOC/output/build/util-linux-2.33/configure`  
强制编译script工具
```
28399 BUILD_SCRIPT_TRUE=
28400 BUILD_SCRIPT_FALSE='#'
28401 BUILD_SCRIPTREPLAY_TRUE=
28402 BUILD_SCRIPTREPLAY_FALSE='#'
```
因为我改的是configure文件, 需要重新配置才能生效
```
#make xxx-reconfigure比rebuild多了configure过程. 
make util-linux-reconfigure
```
每个包的精细控制如下:
```
Package-specific:
  <pkg> - Build and install <pkg> and all its dependencies
  <pkg>-source - Only download the source files for <pkg>
  <pkg>-extract - Extract <pkg> sources
  <pkg>-patch - Apply patches to <pkg>
  <pkg>-depends - Build <pkg>'s dependencies
  <pkg>-configure - Build <pkg> up to the configure step
  <pkg>-build - Build <pkg> up to the build step
  <pkg>-show-depends - List packages on which <pkg> depends
  <pkg>-show-rdepends - List packages which have <pkg> as a dependency
  <pkg>-show-recursive-depends
                         - Recursively list packages on which <pkg> depends
  <pkg>-show-recursive-rdepends
                         - Recursively list packages which have <pkg> as a dependency
  <pkg>-graph-depends - Generate a graph of <pkg>'s dependencies
  <pkg>-graph-rdepends - Generate a graph of <pkg>'s reverse dependencies
  <pkg>-dirclean - Remove <pkg> build directory
  <pkg>-reconfigure - Restart the build from the configure step
  <pkg>-rebuild - Restart the build from the build step
  <pkg>-source-check - Check package for valid download URLs
  <pkg>-all-source-check - Check package and its dependencies for valid download URLs
```

# 常用命令
```
make update-defconfig
make linux-tools-rebuild
make linux-tools
make rootfs-cpio
make linux-dirclean
make linux-rebuild
```

# 重编linux
```shell
cd buildroot/output/build/linux-custom
rm -f .stamp_*

cd buildroot
make linux

#重新同步代码到linux-custom, 重新配置, 重新编译
#观察发现是增量编译
make linux-rebuild
```

# OCTEON SDK的kernelconfig
```shell
cd /repo/yingjieb/dl/caviumsdk5.1/usr/local/Cavium_Networks/OCTEON-SDK/linux/kernel
#用SDK自带的defconfig
cp kernel.config linux/.config
cd linux/
make menuconfig ARCH=mips
#保存def文件
make savedefconfig ARCH=mips
```

