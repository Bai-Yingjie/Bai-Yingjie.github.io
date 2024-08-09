- [configure](#configure)
- [autogen](#autogen)
- [两个文件需要编辑](#两个文件需要编辑)
- [Autotools运行流程](#autotools运行流程)

# configure
```shell
# 参考README和INSTALL
# 对于大多数用automake系统的开源组件, 一般的操作如下:
./configure
#configure允许很多参数, 比如
# ./configure CC=c99 CFLAGS=-g LIBS=-lposix
# ./configure --prefix=$HOME CC=c99
# 用下面这个可以编过
./configure --prefix=$HOME CFLAGS=-std=gnu99
make
make install
```

# autogen
> htop出2.0版本了, 这个工具似乎已经逐渐取代top, 成为linux日常使用额必备利器. 这里记录编译htop的过程.

从git clone下来的代码, 根据readme所说, 需要这样来编译安装
`./autogen.sh && ./configure && make`
那么autogen里面是什么呢? 自动生成configure和Makefile又是什么原理呢?
`autogen.sh`只有短短几行
```shell
$ cat autogen.sh 
#!/bin/sh
mkdir -p m4
autoreconf --install --force
```
autoreconf的说明, 见
https://www.gnu.org/savannah-checkouts/gnu/autoconf/manual/autoconf-2.69/html_node/autoreconf-Invocation.html

autoreconf运行autoconf, autoheader, aclocal, automake, libtoolize, 和autopoint

也可参考:
http://www.baidu.com/link?url=y_4Or_vnHU2ZgcBOZFsRMYGSrQvMsdRAt6yGbE4R1Lx3Xvtpe-wMFtNGBhF5jLcRA80X9-hcBnStMBzFH7Z4N2fkYeq8ssusBRP-E8Bzply

# 两个文件需要编辑
* configure.ac（主要被 autoconf 使用）  
configure.ac 置于项目根目录
* Makefile.am（主要被 automake 使用）  
Makefile.am 必须有一个放在项目根目录，可以在所有有必要的子目录下单独编写放置   
Makefile.am --(automake)--> Makefile.in  
Makefile.in ---(./configure)--> Makefile

# Autotools运行流程
1. 执行autoscan命令。这个命令主要用于扫描工作目录，并且生成configure.scan文件。
2. 修改configure.scan为configure.ac文件，并且修改配置内容。
3. 执行aclocal命令。扫描 configure.ac 文件生成 aclocal.m4文件。
4. 执行autoconf命令。这个命令将 configure.ac 文件中的宏展开，生成 configure 脚本。
5. 执行autoheader命令。该命令生成 config.h.in 文件。
6. 新增Makefile.am文件，修改配置内容
7. 执行automake --add-missing命令。该命令生成 Makefile.in 文件。
8. 执行 ./congigure命令。将Makefile.in命令生成Makefile文件。
9. 执行make命令。生成可执行文件。