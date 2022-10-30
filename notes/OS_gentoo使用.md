- [gcc选择](#gcc选择)
- [常用目录](#常用目录)
- [portage 用git](#portage-用git)
- [history](#history)
- [arm64](#arm64)
- [整个系统重新build](#整个系统重新build)
- [修改make.conf](#修改makeconf)
- [修改源码编译](#修改源码编译)
  - [注意使用全路径ebuild](#注意使用全路径ebuild)
  - [查看编译错误](#查看编译错误)
  - [源码路径](#源码路径)
- [启动项相关](#启动项相关)
- [更新portage](#更新portage)
- [关于use](#关于use)
- [更新整个系统系统 --非常慢](#更新整个系统系统---非常慢)
- [指定版本安装](#指定版本安装)
- [添加udev支持](#添加udev支持)
- [自动起ssh服务](#自动起ssh服务)
- [检查一个命令属于哪个ebuild --貌似不太好用](#检查一个命令属于哪个ebuild---貌似不太好用)
- [检查一个ebuild包含那些文件](#检查一个ebuild包含那些文件)
- [安装常用软件](#安装常用软件)
- [eix用法](#eix用法)
- [调试手段更新](#调试手段更新)
- [gentoo的调试手段](#gentoo的调试手段)
- [错误解决](#错误解决)
  - [block B错误](#block-b错误)

# gcc选择
```sh
gcc-config -l
gcc-config mips64-unknown-linux-gnu-4.9.3
```
# 常用目录
```sh
# protage
/usr/portage
#下载目录
/usr/portage/distfiles
#编译目录
/var/tmp/portage
```

# portage 用git
https://gitweb.gentoo.org/repo/gentoo.git/  
https://anongit.gentoo.org/git/repo/gentoo.git  
拷贝到/usr/portage就好了

# history
```sh
emerge --info
emerge-webrsync
cat /etc/portage/make.conf
emerge -avtuDN @world
eselect news read
emerge --list-sets
eix
emerge gentoolkit
dispatch-conf
eix gcc
eix-update
emerge -pO @installed | grep glib
eselect profile list
emerge -avtDp @world
gcc -c -Q -march=native --help=target
#--emptytree, -e
#Reinstalls target atoms and their entire deep dependency tree, as though no packages are currently installed. You should run this with --pretend first to make sure the result is what you expect.
emerge -evp @world
emerge --depclean
gcc-config -l
#binutils-config -l
eselect binutils list
profile-config list
eclean-dist --deep
genkernel --menuconfig --no-strip all
```

# arm64
/etc/portage/make.conf
```sh
# These settings were set by the catalyst build script that automatically
# built this stage.
# Please consult /usr/share/portage/config/make.conf.example for a more
# detailed example.
CFLAGS="-march=native -mtune=native -Og -pipe -ggdb"
CXXFLAGS="${CFLAGS}"
ACCEPT_KEYWORDS="~arm64"
FEATURES="nostrip"
# WARNING: Changing your CHOST is not something that should be done lightly.
# Please consult https://wiki.gentoo.org/wiki/Changing_the_CHOST_variable before changing.
CHOST="aarch64-unknown-linux-gnu"
PORTDIR="/usr/portage"
DISTDIR="/usr/portage/distfiles"
PKGDIR="/usr/portage/packages"
# This sets the language of build output to English.
# Please keep this setting intact when reporting bugs.
LC_MESSAGES=C
```

# 整个系统重新build
```sh
emerge -ev @world
-e: emptytree
```

# 修改make.conf
```sh
root@localhost /etc/portage
#cat make.conf
# These settings were set by the catalyst build script that automatically
# built this stage.
# Please consult /usr/share/portage/config/make.conf.example for a more
# detailed example.
CFLAGS="-O2 -march=octeon2 -mabi=64 -pipe"
CXXFLAGS="${CFLAGS}"
# WARNING: Changing your CHOST is not something that should be done lightly.
# Please consult http://www.gentoo.org/doc/en/change-chost.xml before changing.
CHOST="mips64-unknown-linux-gnu"
PORTDIR="/usr/portage"
DISTDIR="${PORTDIR}/distfiles"
PKGDIR="${PORTDIR}/packages"
GENTOO_MIRRORS="http://mirrors.stuhome.net/gentoo"
SYNC="rsync://mirrors.stuhome.net/gentoo-portage"
MAKEOPTS="-j48"
```

# 修改源码编译
## 注意使用全路径ebuild
```sh
#ebuild /usr/portage/dev-db/mysql/mysql-5.6.21.ebuild clean
#ebuild /usr/portage/dev-db/mysql/mysql-5.6.21.ebuild fetch
#ebuild /usr/portage/dev-db/mysql/mysql-5.6.21.ebuild unpack
#ebuild /usr/portage/dev-db/mysql/mysql-5.6.21.ebuild prepare
#ebuild /usr/portage/dev-db/mysql/mysql-5.6.21.ebuild configure
#ebuild /usr/portage/dev-db/mysql/mysql-5.6.21.ebuild compile
#ebuild /usr/portage/dev-db/mysql/mysql-5.6.21.ebuild install
#ebuild /usr/portage/dev-db/mysql/mysql-5.6.21.ebuild qmerge
```

## 查看编译错误
`cat /var/tmp/portage/dev-db/mysql-5.6.21/temp/mysql_install_db.log`

## 源码路径
`cd /var/tmp/portage/dev-db/mysql-5.6.21/work/`

# 启动项相关
```sh
rc-update del keymaps boot
octeon-host proc # rc-update del udev sysinit
* service udev removed from runlevel sysinit
octeon-host proc # rc-update del urandom boot 
* service urandom removed from runlevel boot
```
```sh
#rc-config list
#rc-update show
#rc-update add
#rc-update del
```

# 更新portage
`#emerge --sync && emerge portage`  
根据提示要执行  
`emerge --oneshot portage`

# 关于use
对每个包进行use控制  
`/etc/portage/package.use`
查看use的含义  
`euse -i alsa`

# 更新整个系统系统 --非常慢
```sh
#emerge -avtuDN @world
-a --ask
-v --verbose
-t --tree
-u --update
-D --deep
-N --newuse
```
常用选项
```sh
--pretend
--autounmask-write
```
关于删除的选项
```sh
-C --unmerge 不检查依赖关系
-c --depclean 检查依赖关系
```
但提示需要更新/etc/portage此时需要:  
`#etc-update`  
或者, 用更好点的. dispatch-conf 使用的时候选u, 使用新的配置文件. 选z是使用旧的, 删除新的  
使用方法 http://www.gentoo-wiki.info/TIP_dispatch-conf  
`#dispatch-conf`

# 指定版本安装
```sh
emerge --search mysql
emerge -av --pretend =dev-db/mysql-5.6.16
emerge -av --pretend =mysql-5.6.16
```

# 添加udev支持
```sh
加USE="... udev ..."
更新use
emerge --ask --changed-use --deep @world
```

# 自动起ssh服务
```sh
rc-update add sshd default
#/etc/init.d/sshd start
```

# 检查一个命令属于哪个ebuild --貌似不太好用
`equery b /usr/sbin/lspci `

# 检查一个ebuild包含那些文件
`equery f mysql`

# 安装常用软件
```sh
安装lspci
#emerge pciutils
安装vim
#emerge vim
安装equery
#emerge gentoolkit
安装系统记录
#emerge syslog-ng
#rc-update add syslog-ng default
安装locate工具
#emerge mlocate
安装周期deamon
#emerge cronie
#rc-update add cronie default
安装dhcp client
#emerge dhcpcd
安装strace
#emerge strace
```
新增
```sh
#emerge eix
#emerge lshw
```

# eix用法
```sh
eix key_word
eix-sync
```

# 调试手段更新
https://wiki.gentoo.org/wiki/Debugging
```sh
dgentoo ~ # cat /etc/portage/env/*
CFLAGS="${CFLAGS} -ggdb"
CXXFLAGS="${CXXFLAGS} -ggdb"
FEATURES="${FEATURES} splitdebug compressdebug -nostrip"
USE="debug"
FEATURES="${FEATURES} installsources"
dgentoo /etc/portage # cat package.env
category/some-package debugsyms
category/some-library debugsyms installsources
```

# gentoo的调试手段
https://wiki.gentoo.org/wiki//etc/portage/env  
增加包单独的编译选项, 比如这里分别对libtool和mysql增加链接选项
```sh
/etc/portage/package.env
dev-db/mysql            ld.conf debug.conf
sys-devel/libtool      ld.conf
```
ld.conf 和 debug.conf是自己创建的文件  
`/etc/portage/env`目录下  
ld.conf
```sh
LDFLAGS="${LDFLAGS} -Wl,-lpthread,-ldl"
CFLAGS="${CFLAGS} -Wl,-lpthread,-ldl"
CXXFLAGS="${CXXFLAGS} -Wl,-lpthread,-ldl"
```
debug.conf
```sh
CFLAGS="${CFLAGS} -ggdb"
CXXFLAGS="${CXXFLAGS} -ggdb"
FEATURES="splitdebug"
```
[注]: -Wl是gcc传递给ld的格式, -Wl, aaa,bbb 相当于传递给ld aaa bbb
```sh
gentoo-host / # cat /etc/portage/package.env
dev-db/mysql            make.conf
```
```sh
gentoo-host / # cat /etc/portage/env/make.conf
CFLAGS="-O2 -march=octeon2 -mabi=64 -pipe"
CXXFLAGS="${CFLAGS}"
```

# 错误解决
`You're missing a /dev/fd symlink to /proc/self/fd.`  
解决:  
`#ln -s /proc/self/fd /dev/fd`

## block B错误
解决:  
先卸载阻挡的包, 再更新
```sh
#emerge --unmerge procps
#emerge --unmerge sysvinit
```
