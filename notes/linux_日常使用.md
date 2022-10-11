- [使用overlay文件系统](#使用overlay文件系统)
- [查看是否用了proxy](#查看是否用了proxy)
- [记录tty输出并回放](#记录tty输出并回放)
- [编译安装openssh](#编译安装openssh)
- [ubuntu打开ssh服务](#ubuntu打开ssh服务)
- [ubuntu apt用代理](#ubuntu-apt用代理)
- [xiaoshujiang代码变整齐](#xiaoshujiang代码变整齐)
- [使用sshfs实现远程目录mount](#使用sshfs实现远程目录mount)
- [pkg-config工具用来检索系统中库文件的信息, 编译和链接的时候会用到.](#pkg-config工具用来检索系统中库文件的信息-编译和链接的时候会用到)
- [pandoc生成简历](#pandoc生成简历)
- [nmap 网络扫描](#nmap-网络扫描)
- [简单的文件服务器](#简单的文件服务器)
  - [或者用http-server](#或者用http-server)
- [ubuntu 打开nfs服务](#ubuntu-打开nfs服务)
  - [nfs v3 和v4的mount区别](#nfs-v3-和v4的mount区别)
  - [补充, mount fuse文件系统](#补充-mount-fuse文件系统)
  - [nfs服务端](#nfs服务端)
  - [在客户端mount](#在客户端mount)
    - [如果这个nfs mount点挂了, 用这个命令umount](#如果这个nfs-mount点挂了-用这个命令umount)
- [编译kernel](#编译kernel)
- [route](#route)
  - [SDP通过mini主机上外网](#sdp通过mini主机上外网)
  - [mini主机双网卡同时上内网和外网](#mini主机双网卡同时上内网和外网)
    - [最最新的配置文件](#最最新的配置文件)
    - [最新的配置文件](#最新的配置文件)
- [新机器pci错误](#新机器pci错误)
- [关于网络-域名debug](#关于网络-域名debug)
- [文本比较工具](#文本比较工具)
- [dhclient](#dhclient)
- [无线网卡命令](#无线网卡命令)
- [hostname](#hostname)
- [sudo 免密码](#sudo-免密码)
- [不启动GUI](#不启动gui)
- [wiz的搜索功能](#wiz的搜索功能)
- [修改grub2默认启动项](#修改grub2默认启动项)
- [解决中文乱码](#解决中文乱码)
- [使用tmux](#使用tmux)
- [vim支持系统clipboard](#vim支持系统clipboard)
- [ubuntu查看系统中已安装的包](#ubuntu查看系统中已安装的包)
- [locate每天更新数据库](#locate每天更新数据库)
- [ubuntu基本开发包](#ubuntu基本开发包)
- [vim插件](#vim插件)
- [开机自启动](#开机自启动)
- [SSD优化](#ssd优化)
- [升级kernel版本](#升级kernel版本)
- [调节笔记本亮度](#调节笔记本亮度)
- [笔记本省电模式](#笔记本省电模式)
- [笔记本触摸板](#笔记本触摸板)
- [n7260无线网卡 on linux](#n7260无线网卡-on-linux)
- [查看firmware版本](#查看firmware版本)
- [不跳转的google](#不跳转的google)

# 使用overlay文件系统
很简单, linux支持overlay文件系统, 它是个uinon多的文件系统, 底层(lower)文件系统可以是只读的, 上层(upper)文件系统可以是tmpfs. 在mount的时候指定
```sh
godevtmp=/tmp/godev/$user
tmphome=/tmp/godev/home/$user
mkdir -p $godevtmp/{upper,work}
sudo mount -t overlay overlay -o lowerdir=/home/$user,upperdir=$godevtmp/upper,workdir=$godevtmp/work $tmphome
```
* lowerdir: 底层文件系统目录
* upperdir: 上层文件系统目录
* workdir: 上层work目录
* 最后的挂载点: 任意已存在目录

# 查看是否用了proxy
```sh
#出现Proxy-Agent字样说明用了proxy
$ curl --head https://www.google.com/
HTTP/1.1 200 Connection Established
Proxy-Agent: Fortinet-Proxy/1.0
```

# 记录tty输出并回放
```sh
#记录, htop的输出也能记录
script --timing=time.log poc.log
#回放, ctrl+s暂停, ctrl+q恢复
scriptreplay --timing=time.log poc.log
```

# 编译安装openssh
```sh
wget https://ftp.yzu.edu.tw/pub/OpenBSD/OpenSSH/portable/openssh-8.0p1.tar.gz
cd openssh-8.0p1
./configure --prefix=$HOME
make
make install
/home/yingjieb/bin/ssh
/home/yingjieb/bin/scp
/home/yingjieb/sbin/sshd
```

# ubuntu打开ssh服务
```
sudo apt-get install openssh-server
```
查看是否ssh已经启动
```
netstat -tlp
```
或
```
ps -e | grep ssh
```
没启动的话要重启一下ssh服务
```
sudo /etc/init.d/ssh resart
```

# ubuntu apt用代理
```sh
Linux Mint 19.1 Tessa $ cat /etc/apt/apt.conf.d/proxy.conf
Acquire::http::Proxy "http://135.245.48.34:8000";
```

# xiaoshujiang代码变整齐
自定义css, 只管导出到wiz笔记的, xsj自己的预览还是一塌糊涂
```
.preview .xiaoshujiang_code_container>.xiaoshujiang_code_buttons{
    display: none;
}
body code, body .xiaoshujiang_code
{
    font-family:"Courier New";
}
```

# 使用sshfs实现远程目录mount
```sh
#在client上
sudo yum install fuse-sshfs
#mount远程目录, -C打开压缩, 性能高点
sshfs -C -o allow_other bai@10.64.17.45:/home/bai/share /home/bai/share
#umount
fusermount -u /home/bai/share
#keep alive, 每60秒发"keep alive"消息给server
#在client的~/.ssh/config里
ServerAliveInterval 60
#or
sshfs -o reconnect,ServerAliveInterval=15,ServerAliveCountMax=3 server:/path/to/mount
```

# pkg-config工具用来检索系统中库文件的信息, 编译和链接的时候会用到.
不同的库用.pc文件来管理, ARM64上一般在`/usr/lib64/pkgconfig`下面
也可以用`PKG_CONFIG_PATH`环境变量来指定

`export PKG_CONFIG_PATH=/opt/gtk/lib/pkgconfig:$PKG_CONFIG_PATH`

比如lua的.pc文件如下:
```sh
$ cat /usr/lib64/pkgconfig/lua.pc
V= 5.1
R= 5.1.4
prefix= /usr
exec_prefix=${prefix}
libdir= /usr/lib64
includedir=${prefix}/include
Name: Lua
Description: An Extensible Extension Language
Version: ${R}
Requires:
Libs: -llua -lm -ldl
Cflags: -I${includedir}
```
在`/usr/local/lib/pkgconfig/`下面新建lua.pc

# pandoc生成简历
```sh
git clone https://github.com/mszep/pandoc_resume
#有两种方式, 一是本地运行, 而是起一个docker实例
#本地运行
#这里有个bug, 需要安装pandoc最新的版本
wget https://github.com/jgm/pandoc/releases/download/2.2.1/pandoc-2.2.1-1-amd64.deb
sudo dpkg -i pandoc-2.2.1-1-amd64.deb
make
#docker运行类似, 是一个很好的dev-op应用实例
```
#ubuntu 删除老kernel
```
dpkg --list | grep linux-image 
sudo apt-get purge linux-image-x.x.x-x-generic
sudo update-grub2
```

# nmap 网络扫描
* 扫描单个IP地址
`nmap 192.168.56.1`
* 扫描一个网络中IP地址范围
`nmap 192.168.56.1-255`
* 扫描目标主机的特定端口
`nmap 192.168.56.1 -p 80`
* 扫描特定子网的特殊的端口
`nmap 192.168.56.0/24 –p 1-1000`
* 扫描子网, 找到 IP MAC对应关系  
`nmap -sn 10.239.120.1/24;` -sn指只ping扫描, 不做端口扫描  
`sudo nmap -sn 10.239.120.1/24 -oG -;` -oG是输出格式, 后面-是指stdout  
更好用的版本  
`sudo nmap -n -sP 10.239.120.1/24 | awk '/Nmap scan report/{printf $5;printf " ";getline;getline;print $3;}'`  
更更好用的版本  
`sudo nmap -n -sP 10.239.120.1/24 | awk '/Nmap scan report for/{printf $5;}/MAC Address:/{print " => "$3;}' | sort`  
`sudo nmap -n -sP 192.168.8.100-200 | awk '/Nmap scan report for/{printf $5;}/MAC Address:/{print " => "$3;}' | sort`

# 简单的文件服务器
在需要share的文件夹下面执行
```sh
nohup python -m SimpleHTTPServer &
或
nohup python -m SimpleHTTPServer 8088 &
```
默认是8000端口  
访问的话就
http://serverip:port

## 或者用http-server
```sh
apt install npm
sudo npm install http-server -g
sudo apt install nodejs-legacy
http-server -p 8000
```

# ubuntu 打开nfs服务
## nfs v3 和v4的mount区别
在server上
```sh
/etc/export
#注意nfsv4要加fsid=0
/home/yingjieb/work 192.168.2.0/24(rw,sync,insecure,no_subtree_check,no_root_squash,fsid=0)
```
在client上
```sh
#v3的mount, 带路径
mount 192.168.2.11:/home/yingjieb/work/nfsroot/mipsroot /root/remote -o nolock
#v4的mount, 不带路径
mount -t nfs4 192.168.2.11:nfsroot/mipsroot /root/remote -o nolock
```

## 补充, mount fuse文件系统
nfsv4才支持mount用户态的文件系统, 此时需要用"fsid="来显式指定  
没研究这个选项, 但似乎就是提供一个id号, 数字的, 好像任意填.
```sh
cat /etc/exports
/home/yingjieb/work/share/output 192.168.2.0/24(rw,sync,insecure,no_subtree_check,no_root_squash,all_squash,anonuid=1000,anongid=1000,fsid=0)
```
在板子上mount: 
```sh
mkdir -p /root/remote
#一定要加nolock选项, 否则会timeout
mount 192.168.2.11:/home/yingjieb/work/share/output /root/remote -o nolock
```
这里解释一下: 我的连接是这样的:
`board(192.168.2.12) --(nfs)--> (192.168.2.11)nfs server on PC Linux on virtual box(NAT via host) --(sshfs)--> work station(135.251.206.190)`
```sh
yingjieb@yingjieb-VirtualBox ~/work
Linux Mint 19.1 Tessa $ ls
share tftpd
#share是另外一个服务器的目录, 走sshfs
yingjieb@135.251.206.190:/repo/yingjieb/ms/buildroot on /home/yingjieb/work/share type fuse.sshfs (rw,nosuid,nodev,relatime,user_id=1000,group_id=1000,allow_other)
```

## nfs服务端
```sh
sudo apt install nfs-kernel-server
cat /etc/exports
/home/qdt yingjieb-gv(rw,sync,no_subtree_check,no_root_squash,all_squash,anonuid=1000,anongid=1000)
或
/home/bai/share 10.64.16.0/24(rw,sync,no_subtree_check,no_root_squash,all_squash,anonuid=1000,anongid=1000)
注: uid和gid的1000是指server端的用户qdt
/home/bai/share *(rw,sync,insecure,no_subtree_check,no_root_squash,all_squash,anonuid=1000,anongid=1000)
选项insecure可以让其他网络的client有权限连接, 否则会有`ount.nfs: access denied by server while mounting`错误
sudo exportfs -a
sudo systemctl start nfs-kernel-server.service
sudo systemctl enable nfs-kernel-server.service
```
注:
* NFS希望uid和gid在host上和client上是相同的, 要用all_squash和anonuid=1000,anongid=1000来指定, unkown用户的操作都被当成host上的1000用户 1000组(一般是第一个普通用户)
* 如果在客户端可以mount, 但有权限问题, 要检查host上的共享文件夹的权限, others要可读可写
比如:
```
drwxrwxrwx 3 bai bai 4096 Aug 14 16:55 share
```
* 修改`/etc/exports`要执行`sudo exportfs -rav`

## 在客户端mount
```sh
sudo yum install nfs-utils
sudo mount qdt-shlab-pc.ap.qualcomm.com:/home/qdt /local/qdt-shlab-pc/
```
### 如果这个nfs mount点挂了, 用这个命令umount
```sh
sudo umount -f -l /local/qdt-shlab-pc
```

# 编译kernel
```sh
make mrproper
make oldconfig
make defoldconfig
make -j32
make modules_install -j32
make install -j32
```

# route
## SDP通过mini主机上外网
在mini主机上, 外网是`wlp1s0: 10.75.61.131`, 内网是`enp0s31f6: 10.239.120.104`

原理见笔记 iptables详解
```sh
#打开转发
sudo sysctl -w net.ipv4.ip_forward=1
#用nat table, 路由后, 限定awsdp1的ip(10.239.120.133)可以nat方式从wlp1s0转发出去
sudo iptables -t nat -A POSTROUTING -j MASQUERADE -s 10.239.120.133/32 -o wlp1s0
```
在awsdp1上, 自己的IP是eth0 10.239.120.133
```sh
#因为内网防火墙原因, 默认路由无法上外网, 所以先添加所有10.0.0.0网段走内网路由
sudo ip route add 10.0.0.0/8 via 10.239.120.1
#这时可以安全的删除默认路由了(之前的默认路由也是走10.239.120.1), 因为有了上面那句, 所有目的地址是10.0.0.0网段的报文都走10.239.120.1.
sudo ip route del default
#增加默认路由到mini主机
sudo ip route add default via 10.239.120.104
#如果没有配域名服务器, 还要修改/etc/resolv.conf, 比如
echo "nameserver 10.65.0.1" > /etc/resolv.conf
```
```sh
qdt-shlab-pc: mini pc
登录: ssh qdt@qdt-shlab-pc
上外网之前要先查看一下ifconfig wlp1s0, 有ip说明已经up了
打开无线网卡上外网: sudo ifup wlp1s0
使用完毕后一定要关闭无线网卡: sudo ifdown wlp1s0
注: 路由表已经配置在文件里, 有线网络不需要操作.
其他机器上外网
在AWSDP1上:
sudo ip route add 10.0.0.0/8 via 10.239.120.1
sudo ip route del default
sudo ip route add default via 10.239.120.104
此时路由表应该是这样:
DestinationGateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.010.239.120.104  0.0.0.0         UG0      0        0 eth0
10.0.0.010.239.120.1    255.0.0.0       UG0      0        0 eth0
10.231.213.1010.239.120.1    255.255.255.255 UGH   100    00 eth0
10.239.120.00.0.0.0         255.255.252.0   U100    0        0 eth0
在qdt-shlab-pc上:
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -j MASQUERADE -s 10.239.120.133/32 -o wlp1s0
```

## mini主机双网卡同时上内网和外网
### 最最新的配置文件
```sh
$ cat interfaces
# interfaces(5) file used by ifup(8) and ifdown(8)
auto lo
iface lo inet loopback
auto enp0s31f6
iface enp0s31f6 inet dhcp
        post-up ip route add 10.0.0.0/8 via 10.239.120.1 dev enp0s31f6
#        post-up ip route del default 因为后面加了del default, 这里就必须不要; 否则后面再次del的时候, 会因为没有default而报错.
auto wlp1s0
iface wlp1s0 inet dhcp
        pre-up ip route del default
        pre-up wpa_supplicant -B -D wext -i wlp1s0 -c /etc/wpa_supplicant/wpa.conf
        post-up sysctl -w net.ipv4.ip_forward=1
        post-up iptables -t nat -A POSTROUTING -j MASQUERADE -s 10.239.120.0/22 -o wlp1s0
        post-down pkill wpa_supplicant
```

### 最新的配置文件
```sh
$ cat /etc/network/interfaces
# interfaces(5) file used by ifup(8) and ifdown(8)
auto lo
iface lo inet loopback
auto enp0s31f6
iface enp0s31f6 inet dhcp
        post-up ip route add 10.0.0.0/8 via 10.239.120.1 dev enp0s31f6
        post-up ip route del default
iface wlp1s0 inet dhcp
        pre-up wpa_supplicant -B -D wext -i wlp1s0 -c /etc/wpa_supplicant/wpa.conf
        post-up sysctl -w net.ipv4.ip_forward=1
        post-up iptables -t nat -A POSTROUTING -j MASQUERADE -s 10.239.120.0/22 -o wlp1s0
        post-down pkill wpa_supplicant
```
```sh
qdt@qdt-shlab-pc /etc/iproute2 $ cat /etc/resolv.conf 
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
nameserver 10.253.157.26
nameserver 10.253.157.27
search wlan.qualcomm.com
```
```sh
Hydra: K5x48Vz3
sudo systemctl stop network-manager.service
sudo systemctl disable network-manager.service
sudo systemctl start networking.service
sudo systemctl enable networking.service
qdt@qdt-shlab-pc ~ $ cat /etc/wpa_supplicant/wpa.conf
# reading passphrase from stdin
network={
    ssid="Hydra"
    #psk="K5x48Vz3"
    psk=529cd7878309812d395fb8bd999c203f35a83da1ca0a86fe83c0f7c5cd08efc4
}
qdt@qdt-shlab-pc ~ $ cat /etc/network/interfaces
# interfaces(5) file used by ifup(8) and ifdown(8)
auto lo
iface lo inet loopback
auto enp0s31f6
iface enp0s31f6 inet dhcp
    post-up ip route add 10.0.0.0/8 via 10.239.120.1 dev enp0s31f6
    post-up ip route del default
iface wlp1s0 inet dhcp
    pre-up wpa_supplicant -B -D wext -i wlp1s0 -c /etc/wpa_supplicant/wpa.conf
    post-down pkill wpa_supplicant
qdt@qdt-shlab-pc ~ $ sudo ifup wlp1s0
qdt@qdt-shlab-pc ~ $ sudo ifup wlp1s0
man interfaces
qdt@qdt-shlab-pc ~/Desktop $ cat /etc/network/interfaces
# interfaces(5) file used by ifup(8) and ifdown(8)
auto lo
iface lo inet loopback
auto enp0s31f6
iface enp0s31f6 inet dhcp
sudo dhclient -r wlp1s0
sudo ip route add 10.0.0.0/8 via 10.239.120.1 dev enp0s31f6
sudo ip route del default
sudo ip route add default via 10.75.60.1 dev wlp1s0
qdt@qdt-shlab-pc /etc/iproute2 $ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.75.60.1      0.0.0.0         UG    0      0        0 wlp1s0
10.0.0.0        10.239.120.1    255.0.0.0       UG    0      0        0 enp0s31f6
10.75.60.0      0.0.0.0         255.255.254.0   U     0      0        0 wlp1s0
10.239.120.0    0.0.0.0         255.255.252.0   U     0      0        0 enp0s31f6
sudo killall dhclient
sudo ip address add 10.75.61.106/255.255.254.0 dev wlp1s0
sudo ip route add default via 10.75.60.1 dev wlp1s0
sudo sh -c 'echo "nameserver 10.253.157.26" >> /etc/resolv.conf'
ip addr flush dev wlp1s0
    sudo wpa_passphrase Hydra >> /etc/wpa_supplicant/wpa.conf
    qdt@qdt-shlab-pc ~/Desktop $ cat /etc/wpa_supplicant/wpa.conf
    # reading passphrase from stdin
    network={
        ssid="Hydra"
        #psk="K5x48Vz3"
        psk=529cd7878309812d395fb8bd999c203f35a83da1ca0a86fe83c0f7c5cd08efc4
    }
    rfkill unblock all
    sudo ip link set wlp1s0 up
    sudo wpa_supplicant -B -D wext -i wlp1s0 -c /etc/wpa_supplicant/wpa.conf
    sudo dhclient wlp1s0
enp0s31f6:
qdt@qdt-shlab-pc /etc/NetworkManager $ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.239.120.1    0.0.0.0         UG    100    0        0 enp0s31f6
10.231.213.10   10.239.120.1    255.255.255.255 UGH   100    0        0 enp0s31f6
10.239.120.0    0.0.0.0         255.255.252.0   U     100    0        0 enp0s31f6
169.254.0.0     0.0.0.0         255.255.0.0     U     1000   0        0 enp0s31f6
qdt@qdt-shlab-pc /etc/NetworkManager $ ifconfig
enp0s31f6 Link encap:Ethernet  HWaddr 40:8d:5c:d1:d1:f1  
          inet addr:10.239.120.104  Bcast:10.239.123.255  Mask:255.255.252.0
          inet6 addr: fe80::428d:5cff:fed1:d1f1/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:15516 errors:0 dropped:197 overruns:0 frame:0
          TX packets:709 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:1586576 (1.5 MB)  TX bytes:73304 (73.3 KB)
          Interrupt:16 Memory:df100000-df120000 
qdt@qdt-shlab-pc /etc/iproute2 $ cat /etc/resolv.conf 
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
nameserver 10.231.213.10
nameserver 10.253.144.21
search ap.qualcomm.com
wlp1s0:
qdt@qdt-shlab-pc /etc/NetworkManager $ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.75.60.1      0.0.0.0         UG    600    0        0 wlp1s0
10.75.60.0      0.0.0.0         255.255.254.0   U     600    0        0 wlp1s0
10.231.213.10   10.75.60.1      255.255.255.255 UGH   600    0        0 wlp1s0
169.254.0.0     0.0.0.0         255.255.0.0     U     1000   0        0 wlp1s0
qdt@qdt-shlab-pc /etc/NetworkManager $ ifconfig wlp1s0
wlp1s0    Link encap:Ethernet  HWaddr 48:d2:24:d4:70:b9  
          inet addr:10.75.61.74  Bcast:10.75.61.255  Mask:255.255.254.0
          inet6 addr: fe80::4ad2:24ff:fed4:70b9/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1355 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1278 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:607469 (607.4 KB)  TX bytes:154983 (154.9 KB)
    qdt@qdt-shlab-pc /etc/iproute2 $ cat /etc/resolv.conf
    # Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
    #     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
    nameserver 10.253.157.26
    nameserver 10.253.157.27
    search wlan.qualcomm.com
qdt@qdt-shlab-pc /etc/NetworkManager $ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.239.120.1    0.0.0.0         UG    100    0        0 enp0s31f6
0.0.0.0         10.75.60.1      0.0.0.0         UG    600    0        0 wlp1s0
10.75.60.0      0.0.0.0         255.255.254.0   U     600    0        0 wlp1s0
10.231.213.10   10.239.120.1    255.255.255.255 UGH   100    0        0 enp0s31f6
10.231.213.10   10.75.60.1      255.255.255.255 UGH   600    0        0 wlp1s0
10.239.120.0    0.0.0.0         255.255.252.0   U     100    0        0 enp0s31f6
169.254.0.0     0.0.0.0         255.255.0.0     U     1000   0        0 wlp1s0
qdt@qdt-shlab-pc /etc/iproute2 $ cat /etc/resolv.conf 
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
nameserver 10.231.213.10
nameserver 10.253.144.21
nameserver 10.253.157.26
search ap.qualcomm.com wlan.qualcomm.com
```

# 新机器pci错误
```sh
1) edit /etc/default/grub and and add pci=noaer to the line starting with GRUB_CMDLINE_LINUX_DEFAULT. It will look like this:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash pci=noaer"
2) run "sudo update-grub"
3) reboo
```

# 关于网络-域名debug
* host用来查DNS服务器, ip到域名, 域名到ip; `nslookup`有一样的效果
* arp -a用来查本机arp表
* 网络debug用`cat /var/log/messages | egrep "dhclient|NetworkManager"`
* 用NetworkManager的话, nmcli也很有用
```sh
$ host 10.239.120.175
175.120.239.10.in-addr.arpa domain name pointer qdt-shlab-awsdp3-010239120175.qualcomm.com.
qdt@qdt-shlab-awsdp2 ~
$ host 10.239.120.12
12.120.239.10.in-addr.arpa domain name pointer qdt-shlab-awsdp4-010239120012.qualcomm.com.
qdt@qdt-shlab-awsdp2 ~
$ host 10.239.120.179
179.120.239.10.in-addr.arpa domain name pointer qdt-shlab-awsdp2.qualcomm.com.
$ arp -a -i enp0s31f6
pvg-b7-lab-fw-vlan337.qualcomm.com (10.239.120.1) at 00:1b:17:00:37:30 [ether] on enp0s31f6
qdt-shlab-awsdp4-010239120116.qualcomm.com (10.239.120.116) at 8c:fd:f0:06:95:0d [ether] on enp0s31f6
sudo cat /var/log/messages | egrep "dhclient|NetworkManager"
uuidgen eth0
nmcli con show 
nslookup qdt-shlab-awsdp2
nmcli connection
nmcli con show eth0
nmcli con modify eth0 ipv4.dhcp-hostname `hostname`
```

# 文本比较工具
```sh
apt install meld
```
# dhclient
dhclient会获取一个ip, 并持续保持在后台  
用sudo dhclient -r会释放现在的ip, 并杀掉原后台dhclient; 注意需要用sudo  
多次执行dhclient会产生多个后台进程

# 无线网卡命令
基本命令
```sh
iw dev
iw wlp1s0 link
iw wlp1s0 scan
```
参考How to connect to a WPA/WPA2 WiFi network using Linux command line
```sh
sudo wpa_passphrase Hydra >> /etc/wpa_supplicant/wpa.conf
qdt@qdt-shlab-pc ~/Desktop $ cat /etc/wpa_supplicant/wpa.conf
# reading passphrase from stdin
network={
    ssid="Hydra"
    #psk="K5x48Vz3"
    psk=529cd7878309812d395fb8bd999c203f35a83da1ca0a86fe83c0f7c5cd08efc4
}
```
```sh
rfkill unblock all
sudo ip link set wlp1s0 up
sudo wpa_supplicant -B -D wext -i wlp1s0 -c /etc/wpa_supplicant/wpa.conf
sudo dhclient wlp1s0
```
注: wpa_supplicant和dhclient都会在后台一直运行

# hostname
```sh
sudo nano /etc/hostname
sudo nano /etc/hosts
```

# sudo 免密码
```sh
sudo -s
visudo
```
在`%sudo`那行加 `NOPASSWD`:
```sh
# User privilege specification
root    ALL=(ALL:ALL) ALL
 
# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL
 
# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) NOPASSWD:ALL
```

# 不启动GUI
```
sudo systemctl disable mdm  
sudo systemctl start mdm
sudo systemctl enable mdm 
```

# wiz的搜索功能
一般为"或"搜索
```sh
linux版可以这样搜:"struct*file"
windows版: s:struct AND file
```

# 修改grub2默认启动项
修改/etc/default/grub
saved是说让grub记录上次的启动项
```sh
$ cat /etc/default/grub
# If you change this file, run 'update-grub' afterwards to update
# /boot/grub/grub.cfg.
# For full documentation of the options in this file, see:
#   info -f grub -n 'Simple configuration'
GRUB_DEFAULT=saved
GRUB_SAVEDEFAULT=true
```
还要运行
```
$ sudo update-grub2
```

# 解决中文乱码
```
iconv -f gbk -t utf8 readme.txt > out.txt
```

# 使用tmux
```sh
$tmux ls 查看有几个session
$tmux attach -t 1 进入第一个session
c-b c 新建一个window
c-b n 切换到下一个window
c-b 0 1 2 3 4 5 6 7 8 9 选择window
c-b & 退出当前window
c-b % 在当前panel左右新建一个panel
c-b " 在当前panel上下新建一个panel
c-b o 在panel之间切换
c-b ! 关闭所有分割出来的pannel
c-b x 退出光标所在的panel
c-b d 临时退出tmux, 进程不退出. 可用tmux attach再次进入
c-b ? 帮助
在tmux窗口里面敲exit退出当前窗口.
 
tmux kill-server 终止tmux server, 会杀掉所有tmux client
 
系统操作  
    ?   #列出所有快捷键；按q返回  
    d   #脱离当前会话；这样可以暂时返回Shell界面，输入tmux attach能够重新进入之前的会话  
    D   #选择要脱离的会话；在同时开启了多个会话时使用  
    Ctrl+z  #挂起当前会话  
    r   #强制重绘未脱离的会话  
    s   #选择并切换会话；在同时开启了多个会话时使用  
    :   #进入命令行模式；此时可以输入支持的命令，例如kill-server可以关闭服务器  
    [   #进入复制模式；此时的操作与vi/emacs相同，按q/Esc退出  
    ~   #列出提示信息缓存；其中包含了之前tmux返回的各种提示信息  
窗口操作  
    c   #创建新窗口  
    &   #关闭当前窗口  
    数字键 #切换至指定窗口  
    p   #切换至上一窗口  
    n   #切换至下一窗口  
    l   #在前后两个窗口间互相切换  
    w   #通过窗口列表切换窗口  
    ,   #重命名当前窗口；这样便于识别  
    .   #修改当前窗口编号；相当于窗口重新排序  
    f   #在所有窗口中查找指定文本  
面板操作  
    ”   #将当前面板平分为上下两块  
    %   #将当前面板平分为左右两块  
    x   #关闭当前面板  
    !   #将当前面板置于新窗口；即新建一个窗口，其中仅包含当前面板  
    Ctrl+方向键    #以1个单元格为单位移动边缘以调整当前面板大小  
    Alt+方向键 #以5个单元格为单位移动边缘以调整当前面板大小  
    Space   #在预置的面板布局中循环切换；依次包括even-horizontal、even-vertical、main-horizontal、main-vertical、tiled  
    q   #显示面板编号  
    o   #在当前窗口中选择下一面板  
    方向键 #移动光标以选择面板  
    {   #向前置换当前面板  
    }   #向后置换当前面板  
    Alt+o   #逆时针旋转当前窗口的面板  
    Ctrl+o  #顺时针旋转当前窗口的面板  
```
# vim支持系统clipboard
```sh
$apt install vim-gtk
在.vimrc里面加
set clipboard=unnamedplus
检查是否带clipboard功能
$vim --version
找+xterm_clipboard
```

# ubuntu查看系统中已安装的包
```
$dpkg -l | grep fcitx
```
这里的选项是list的意思
 
# locate每天更新数据库
突然发现每天第一次开机硬盘狂转，机器非常慢。
调查发现是locate每天都在更新数据库。
这个文件/etc/crontab， 是个周期任务控制文件。
上面写了要把/etc/cron.daily里面的脚本每天执行一遍， 时间在每天6点25.
这就解释了为什么第一次开机硬盘会狂闪。
 
解决办法：
```cp /etc/cron.daily/locate /etc/cron.weekly/```
把每天执行改为每周执行一次
 
# ubuntu基本开发包
```
apt install build-essential
sudo apt-get install libncurses5-dev
```

# vim插件
```sh
sudo apt- install vim-addon-manager vim-scripts
vim-addons install taglist
apt install astyle cscope ctags global
byj@byj-mint ~/.vim/plugin $ ls
autoformat.vim  defaults.vim  gtags.vim  mru.vim  taglist.vim
```

# 开机自启动
修改/etc/rc.local
加一行
```
/home/byj/goagent/goagent/local/proxy.sh start
```

# SSD优化
修改/etc/rc.local
```sh
echo 50 > /proc/sys/vm/dirty_ratio
echo 10 > /proc/sys/vm/dirty_background_ratio
echo 6000 > /proc/sys/vm/dirty_expire_centisecs
echo 1000 > /proc/sys/vm/dirty_writeback_centisecs
```
修改/etc/fstab
```sh
UUID=d039c104-49d4-464a-b3df-a962574fd46f /               ext4    noatime,nodiratime,errors=remount-ro 0       1
```

# 升级kernel版本
1. 到https://www.kernel.org
找稳定的版本号
2. 到ubuntu网站上下载对应的版本http://kernel.ubuntu.com/~kernel-ppa/mainline
一共有三个文件
```
linux-headers-3.16.2-031602-generic_3.16.2-031602.201409052035_amd64.deb
linux-headers-3.16.2-031602_3.16.2-031602.201409052035_all.deb
linux-image-3.16.2-031602-generic_3.16.2-031602.201409052035_amd64.deb
```
3. `mkdir linux-kernel-3.16.2 && mv ~/Downloads/linux-*.deb linux-kernel-3.16.2`
4. 安装
```sh
cd linux-kernel-3.16.2
su
dpkg -i *.deb
update-grub(可以省略, 上面的命令已包含)
```
5. 重启

# 调节笔记本亮度
```sh
su
echo 80 > /sys/class/backlight/intel_backlight/brightness
```

# 笔记本省电模式
安装tlp: https://linrunner.de/tlp/
```
sudo add-apt-repository ppa:linrunner/tlp
sudo apt install tlp tlp-rdw
sudo systemctl start tlp
sudo systemctl enable tlp
```
如果装了其他的省电工具
```
sudo systemctl stop laptop-mode
sudo systemctl disable laptop-mode
```
查看cpu调频的信息:
```
sudo cpupower frequency-info
```

我发现tlp会自动停掉wifi和蓝牙

# 笔记本触摸板
* 临时禁止触摸板：`sudo modprobe -r psmouse`
* 开启触摸板：`sudo modprobe -a psmouse`
* 永远禁用触摸板：
```
sudo vi /etc/modprobe.d/blacklist.conf
blacklist psmouse
```

# n7260无线网卡 on linux
iwlwifi.ko
禁用8022.11n
* 临时: 有时不好用
```
su
modprobe -r iwlwifi
modprobe iwlwifi 11n_disable=1
```
* 长久:
```
su
echo "options iwlwifi 11n_disable=1" >> /etc/modprobe.d/iwlwifi.conf
```

# 查看firmware版本
```
lshw
```

# 不跳转的google
```
http://www.google.com/ncr
```
