- [安装新版本GCC](#安装新版本gcc)
- [clear disk space](#clear-disk-space)
- [关于permission denied](#关于permission-denied)
- [x11 forwarding](#x11-forwarding)
- [查找系统中所有安装的kernel版本](#查找系统中所有安装的kernel版本)
- [查看rpm包含的文件](#查看rpm包含的文件)
- [安装htop](#安装htop)
- [关闭源的check](#关闭源的check)
- [开发环境](#开发环境)
- [KVM](#kvm)
- [修改yum源](#修改yum源)
- [安装rpm](#安装rpm)

# 安装新版本GCC
```sh
sudo yum install centos-release-scl-rh 
sudo yum-config-manager --enable centos-sclo-rh-testing 
#把centos-sclo-rh disable, 把centos-sclo-rh-testing enable 
sudo vi /etc/yum.repos.d/CentOS-SCLo-scl-rh.repo 
#列出devtoolset 
yum list available | grep devtoolset 
#安装devtoolset6 
yum install devtoolset-6 
#切换到gcc6 
scl enable devtoolset-6 bash 
```

# clear disk space
https://www.getpagespeed.com/server-setup/clear-disk-space-centos

# 关于permission denied
home目录有额外的权限控制, 默认other没有任何权限; 而有些进程比如qemu是libvirtd启动的, 一般为libvirt-qemu用户, 就不能访问个人home下面的东西; 似乎root用户都不能访问??!

有两套东西管权限, 一个是ACL, 另一个就是臭名昭著的seLinux
> setfacl -b will remove the ACL on a file. setfattr -x security.selinux will remove the SELinux file context, but you will probably have to boot with SELinux completely disabled.
```bash
#让other能完全访问home目录
sudo setfacl -m o:rwx /home/bai
$ sudo getfacl -e /home/bai
getfacl: Removing leading '/' from absolute path names
# file: home/bai
# owner: bai
# group: bai
user::rwx
group::rwx
other::rwx
#或者安全点, 指定用户
sudo setfacl -m u:libvirt-qemu:rx /home/murlock
```

# x11 forwarding
```
yum install -y xorg-x11-server-Xorg xorg-x11-xauth xorg-x11-apps
```

# 查找系统中所有安装的kernel版本
```
rpm -qa | grep kernel
```

# 查看rpm包含的文件
libfdt包括了操作device tree的库函数, 以rpm包的方式被centos收录发行.
```sh
$ rpm -ql libfdt-devel
/usr/include/fdt.h
/usr/include/libfdt.h
/usr/include/libfdt_env.h
/usr/lib64/libfdt.so
```

# 安装htop
```
#enable the epel release
sudo yum -y install epel-release
sudo yum install htop
```

# 关闭源的check
```
yum --nogpg install ...
```

# 开发环境
```
# yum groups list | grep -i devel
# yum groups install "C Development Tools and Libraries"
yum --nogpg groupinstall "Development Tools"
```

# KVM
```
sudo yum groupinstall "Virtualization*"
```

# 修改yum源
把everything iso mount到/mnt/centos
```sh
sudo mv CentOS-Base.repo CentOS-Base.repo.bak
[qdt@qdt-shlab-awsdp1 yum.repos.d]$ cat CentOS-Local.repo
[local]
name=local
baseurl=file:///mnt/centos
enabled=1
gpgcheck=1
gpgkey=file:///mnt/centos/RPM-GPG-KEY-CentOS-7
```

# 安装rpm
```
rpm -ivh $url
```
