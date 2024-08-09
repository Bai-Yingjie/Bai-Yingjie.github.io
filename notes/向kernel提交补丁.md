- [背景](#背景)
- [使用git](#使用git)
  - [准备kernel的git库](#准备kernel的git库)
  - [patch从hg到git](#patch从hg到git)
  - [写git commit](#写git-commit)
  - [生成用于提交upstream的patch](#生成用于提交upstream的patch)
  - [检查patch格式](#检查patch格式)
  - [测试发送](#测试发送)
  - [找到maintainer](#找到maintainer)
  - [正式发送](#正式发送)
  - [其实没这么简单](#其实没这么简单)
  - [到patchwork上查提交](#到patchwork上查提交)
  - [补充, 用手机开热点可以发送](#补充-用手机开热点可以发送)
    - [先要设置gmail, 打开less secure app access](#先要设置gmail-打开less-secure-app-access)
    - [再用`git send-email`发送](#再用git-send-email发送)
    - [以上不行要打开临时访问, 据说只有10分钟.](#以上不行要打开临时访问-据说只有10分钟)
- [参考](#参考)

# 背景
我在linux-fsl里面有一个commit, 想提交到社区.
```
327 32db966721ed tip linux-4.9-preparation 2019-11-21 Bai Yingjie PPC32 smp boot: also write addr_h to spin table for a 64bit boot entry
```
下面记录一下如何提交补丁到upstream社区

# 使用git
我们内部使用hg, 但提交kernel的补丁要用git. 所以需要先在kernel的git上生成patch.

先要安装git-email, 用来发送patch
```
apt install git-email
```

在生成patch之前, 我要先查一下kernel的commit格式.
比如, 针对我要提交patch的文件, 看一下别人都怎么提交的:
```
git ls arch/powerpc/platforms/85xx/smp.c
```
看了一下, 大部分commiter都用`powerpc/mpc85xx`来开头, 表示子系统.
提交者主要来自freescale.com, 也有gmail.com, windriver.com等待.

OK. 可以准备patch了

## 准备kernel的git库
需要有个最新的kernel的git库
```
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```
或者已经有的库更新到最新: `git pull`

## patch从hg到git
hg export可以生成hg的patch, 但我没找到直接可以给git用的patch. 所以我们用半手工的方式生成patch
```shell
#先生成纯内容的patch
hg diff --git -c 327 | nc 11985
#在kernel的git库, 这里我用虚拟机clone的
nc 172.24.213.190 11985 > 85xx.patch
#patch打到kernel
git apply 85xx.patch
```

## 写git commit
```shell
#写commit
git add arch/powerpc/platforms/85xx/smp.c
#用-s生成Signed-off-by:字段. 内核commit惯例
git commit -s
```
commit之后, 这个patch是这样的:
```shell
f3f83cac875a 2019-11-25 Bai Yingjie powerpc/mpc85xx: also write addr_h to spin table for 64bit boot entry
219d54332a09 2019-11-24 Linus Torvalds Linux 5.4
b8387f6f3495 2019-11-24 Linus Torvalds Merge branch 'fixes' of git://git.kernel.org/pub/scm/linux/kernel/git/viro/vfs
3e5aeec0e267 2019-10-19 Maxime Bizon cramfs: fix usage on non-MTD device
```

## 生成用于提交upstream的patch
kernel的patch都是通过邮件发送的. 所以我要先生成patch.
用
```
git format-patch HEAD~<number of commits to convert to patches>
```
生成一系列patch

这里我只有一个改动, 用:
```
Linux Mint 19.1 Tessa $ git format-patch HEAD~ -o outgoing
outgoing/0001-powerpc-mpc85xx-also-write-addr_h-to-spin-table-for-.patch
```
生成的patch在outgoing文件夹下面. 一次邮件发送多个patch的时候, 用outgoing保存多个patch更方便.

看看这个文件: 它就是个文本格式的邮件.

这些patch用`git am <patchfile>`就被社区维护者打到kernel代码里.

注: 
* patch的后续版本可以用
`git format-patch -v2 HEAD~`来format, 邮件标题会自动加V2
* 多个patch的例子
```
Linux Mint 19.1 Tessa $ git format-patch HEAD~3 -o outgoing
outgoing/0001-powerpc-mpc85xx-also-write-addr_h-to-spin-table-for-.patch
outgoing/0002-powerpc32-booke-consistently-return-phys_addr_t-in-_.patch
outgoing/0003-powerpc-io-use-phys_addr_t-in-virt_to_phys-phys_to_v.patch
```

按照惯例, 邮件标题会加上` [PATCH] `, 比如  
`Subject: [PATCH] powerpc/mpc85xx: also write addr_h to spin table for 64bit`

发送邮箱就是我gitconfig的邮箱:
```shell
Linux Mint 19.1 Tessa $ cat ~/.gitconfig
[user]
        name = Bai Yingjie
        email = byj.tea@gmail.com
[sendemail]
        smtpencryption = tls
        smtpserver = smtp.gmail.com
        smtpuser = byj.tea@gmail.com
        smtpserverport = 587
[core]
        editor = vim
[diff]
        tool = vimdiff
[merge]
        tool = meld
[alias]
        ls = log '--pretty=format:"%h %ad%x09%an%x09%s"' --date=short
        lg = log --decorate --stat --graph
[push]
        default = matching
[http]
        proxy = http://135.245.48.34:8000
```

## 检查patch格式
在邮件发送patch之前, 用`./scripts/checkpatch.pl`检查一下patch的格式是否符合kernel的惯例.

如果有问题, 用`git commit --amend <files updated>`来做相应修改.

这里我的patch格式没问题:
```shell
Linux Mint 19.1 Tessa $ ./scripts/checkpatch.pl outgoing/*
total: 0 errors, 0 warnings, 14 lines checked

outgoing/0001-powerpc-mpc85xx-also-write-addr_h-to-spin-table-for-.patch has no obvious style problems and is ready for submission.
```

## 测试发送
```
git send-email --to byj8389@qq.com outgoing/*
```

## 找到maintainer
kernel提供了找到相关maintainer的脚本:`scripts/get_maintainer.pl`

```shell
Linux Mint 19.1 Tessa $ ./scripts/get_maintainer.pl outgoing/0001-powerpc-mpc85xx-also-write-addr_h-to-spin-table-for-.patch
Scott Wood <oss@buserror.net> (maintainer:LINUX FOR POWERPC EMBEDDED PPC83XX AND PPC85XX)
Kumar Gala <galak@kernel.crashing.org> (maintainer:LINUX FOR POWERPC EMBEDDED PPC83XX AND PPC85XX)
Benjamin Herrenschmidt <benh@kernel.crashing.org> (supporter:LINUX FOR POWERPC (32-BIT AND 64-BIT))
Paul Mackerras <paulus@samba.org> (supporter:LINUX FOR POWERPC (32-BIT AND 64-BIT))
Michael Ellerman <mpe@ellerman.id.au> (supporter:LINUX FOR POWERPC (32-BIT AND 64-BIT))
linuxppc-dev@lists.ozlabs.org (open list:LINUX FOR POWERPC EMBEDDED PPC83XX AND PPC85XX)
linux-kernel@vger.kernel.org (open list)
```

这个脚本也可以对文件使用:
```
./scripts/get_maintainer.pl -f arch/powerpc/platforms/85xx/smp.c
```
效果是一样的.

可以看到, `Scott Wood <oss@buserror.net>`和`Kumar Gala <galak@kernel.crashing.org>`是maintainer, 是email要to的人. 其他人cc


## 正式发送
```shell
git send-email outgoing/* --cc-cmd './scripts/get_maintainer.pl --norolestats --git outgoing/*' --to 'Scott Wood <oss@buserror.net>' --to 'Kumar Gala <galak@kernel.crashing.org>'
```

## 其实没这么简单
如果这么简单就好了. 实际上, 我不能通过gmail的smtp发送patch. 在公司, 587端口被防火墙了. 在家, google更是连不上.

但gmail邮箱是我主力提交代码的邮箱, 我还不想换.

所以, 需要曲线救国的方案. 如下:

先注册一个126的邮箱, 而且一定要在其设置里把smtp服务打开. 打开这个服务, 需要再设置个"授权码", 用于第三方登录的时候, 代替邮箱密码.

然后用如下命令提交patch, 就是说让126邮箱来发这个patch, 但内容还是我之前生成的, 在patch里用的是gmail邮箱.

先试一试:
```shell
git send-email 0001-powerpc-mpc85xx-also-write-addr_h-to-sp.patch --smtp-debug=1 --to yingjie.bai@nokia-sbell.com --smtp-encryption=ssl --smtp-server=smtp.126.com --smtp-server-port=994 --smtp-user=yingjie_bai@126.com --from=yingjie_bai@126.com
```

正式提交版本:
```shell
git send-email 0001-powerpc-mpc85xx-also-write-addr_h-to-sp.patch --smtp-debug=1 --to 'Scott Wood <oss@buserror.net>' --to 'Kumar Gala <galak@kernel.crashing.org>' --cc 'Benjamin Herrenschmidt <benh@kernel.crashing.org>' --cc 'Paul Mackerras <paulus@samba.org>' --cc 'Michael Ellerman <mpe@ellerman.id.au>' --cc 'linuxppc-dev@lists.ozlabs.org' --cc 'linux-kernel@vger.kernel.org' --smtp-encryption=ssl --smtp-server=smtp.126.com --smtp-server-port=994 --smtp-user=yingjie_bai@126.com --from=yingjie_bai@126.com
```
```shell
git send-email outgoing/* --cc-cmd './scripts/get_maintainer.pl --norolestats --git outgoing/*' --to 'Scott Wood <oss@buserror.net>' --to 'Kumar Gala <galak@kernel.crashing.org>' --smtp-encryption=ssl --smtp-server=smtp.126.com --smtp-server-port=994 --smtp-user=yingjie_bai@126.com --from=yingjie_bai@126.com
```

## 到patchwork上查提交
* [linux kernel mail list](https://lore.kernel.org/patchwork/project/lkml/list/)
* [本次提交](https://lore.kernel.org/patchwork/patch/1158715/)


## 补充, 用手机开热点可以发送
### 先要设置gmail, 打开less secure app access

![](img/向kernel提交补丁_20221011152921.png)  

### 再用`git send-email`发送
多试几次, 有的时候卡住, 有的时候可以输入密码.
第一次会有下面的提示, 按照提示的URL地址打开浏览器, google提示新设备access, 要发手机验证码验证
```shell
Password for 'smtp://byj.tea@gmail.com@smtp.gmail.com:587':
5.7.14 <https://accounts.google.com/signin/continue?sarp=1&scc=1&plt=AKgnsbt
5.7.14 TO77buewAb-9v-7bMFJtKvsLnToTRilFUvqtOzNwXu81Miu3opYz1tKgLMN7mgmKe5Xxy
5.7.14 XiEDQp94UFEPtbqRT_dtLs3QCXtPAs_x4fdayPuecWPQpRFap155Z4nD3HAUtYGn>
5.7.14 Please log in via your web browser and then try again.
5.7.14 Learn more at
5.7.14 https://support.google.com/mail/answer/78754 b65sm47429126pgc.18 - gsmtp
```
验证好了再次发送.
大部分时间还是不行
偶尔可以

### 以上不行要打开临时访问, 据说只有10分钟.
https://accounts.google.com/DisplayUnlockCaptcha


# 参考
* [简单全面的patch提交过程](http://nickdesaulniers.github.io/blog/2017/05/16/submitting-your-first-patch-to-the-linux-kernel-and-responding-to-feedback/)
* [FirstKernelPatch](https://kernelnewbies.org/FirstKernelPatch)
* https://www.kernel.org/doc/html/v4.17/process/submitting-patches.html

