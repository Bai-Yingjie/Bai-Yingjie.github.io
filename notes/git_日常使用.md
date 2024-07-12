- [git commit rang](#git-commit-rang)
- [git分支操作](#git分支操作)
  - [查看分支](#查看分支)
    - [查看本地分支](#查看本地分支)
    - [查看远程分支](#查看远程分支)
    - [查看所有分支](#查看所有分支)
  - [本地创建新的分支](#本地创建新的分支)
  - [切换到新的分支](#切换到新的分支)
  - [创建+切换分支](#创建切换分支)
  - [将新分支推送到github](#将新分支推送到github)
  - [删除本地分支](#删除本地分支)
  - [删除github远程分支](#删除github远程分支)
- [github Readme模板](#github-readme模板)
- [我的git操作记录](#我的git操作记录)
  - [修改作者和邮箱](#修改作者和邮箱)
  - [修改commit时间](#修改commit时间)
  - [同步upstream的更改](#同步upstream的更改)
    - [复杂步骤](#复杂步骤)
    - [简单步骤](#简单步骤)
  - [使用不同的用户名commit](#使用不同的用户名commit)
  - [同步remote已删除的分支](#同步remote已删除的分支)
  - [找到当前branch最近的tag](#找到当前branch最近的tag)
  - [其他分支的修改拿到本分支](#其他分支的修改拿到本分支)
  - [diff和patch](#diff和patch)
  - [从其他git repo导入到新git库](#从其他git-repo导入到新git库)
    - [Git global setup](#git-global-setup)
    - [Create a new repository](#create-a-new-repository)
    - [Push an existing folder](#push-an-existing-folder)
    - [Push an existing Git repository](#push-an-existing-git-repository)
  - [git新建远程分支](#git新建远程分支)
  - [git 查看log摘要](#git-查看log摘要)
  - [git checkout branch](#git-checkout-branch)
  - [git commit set格式](#git-commit-set格式)
  - [git 修改commit](#git-修改commit)
  - [git undo reset](#git-undo-reset)
  - [git调整commit顺序](#git调整commit顺序)
  - [linux kernel:找到一个函数/字符串 相关的commit](#linux-kernel找到一个函数字符串-相关的commit)
  - [linux kernel: 找到一个commit对应的kernel release版本](#linux-kernel-找到一个commit对应的kernel-release版本)
  - [保存现场 两个现场切换 git stash](#保存现场-两个现场切换-git-stash)
  - [git查看某个版本的文件, 不checkout](#git查看某个版本的文件-不checkout)
  - [git设置允许push到非bare的库, 在remote端运行](#git设置允许push到非bare的库-在remote端运行)
  - [git取消commit](#git取消commit)
  - [git大文件支持](#git大文件支持)
  - [git打包](#git打包)
  - [push到远程repo](#push到远程repo)
  - [单一分支clone](#单一分支clone)
  - [git下载远程分支](#git下载远程分支)
  - [创建本地分支来跟踪远程分支](#创建本地分支来跟踪远程分支)
  - [git merge本地改动](#git-merge本地改动)
  - [git比较某次commit的改动](#git比较某次commit的改动)
    - [查看单个文件](#查看单个文件)
  - [git比较本地修改](#git比较本地修改)
  - [还是犯了remote错误](#还是犯了remote错误)
  - [为什么git clone一个本地的repo没有branch?](#为什么git-clone一个本地的repo没有branch)
  - [git切换分支](#git切换分支)
  - [放弃本地更改](#放弃本地更改)
  - [git merge 不commit](#git-merge-不commit)
  - [git tag](#git-tag)
  - [git log显示文件](#git-log显示文件)
  - [git使用kdiff查看改动](#git使用kdiff查看改动)
  - [git更改remote](#git更改remote)
  - [git 从别人那里pull](#git-从别人那里pull)
  - [git恢复文件](#git恢复文件)
  - [git clean, 删除所有未跟踪的文件](#git-clean-删除所有未跟踪的文件)
  - [git查看clone地址](#git查看clone地址)

# git commit rang
```shell
# 从720936d888c到f2779584678的所有commit, 包括720936d888c
git cherry-pick 720936d888c^..f2779584678
```

# git分支操作
```
git clone https://github.com/siskinc/siskinc.github.io
```

## 查看分支
### 查看本地分支
```shell
$ git branch
* master
```
### 查看远程分支
```shell
git branch -r
```
### 查看所有分支
```shell
git branch -a
```

## 本地创建新的分支
```shell
git branch [branch name]
```

## 切换到新的分支
```shell
git checkout [branch name]
```

## 创建+切换分支
```shell
git checkout -b [branch name]
```
相当于
```shell
git branch [branch name]
git checkout [branch name]
```
## 将新分支推送到github
```shell
git push origin [branch name]
```
## 删除本地分支
```shell
git branch -d [branch name]
```
## 删除github远程分支
```shell
git push origin :[branch name]
```

# github Readme模板
https://github.com/golangci/golangci-lint
```markdown
<p align="center">
  <img alt="golangci-lint logo" src="assets/go.png" height="150" />
  <h3 align="center">golangci-lint</h3>
  <p align="center">Fast linters runner for Go</p>
</p>

---

`golangci-lint` is a fast Go linters runner. It runs linters in parallel, uses caching, supports `yaml` config, has integrations
with all major IDE and has dozens of linters included.

## Install `golangci-lint`

- [On my machine](https://golangci-lint.run/usage/install/#local-installation);
- [On CI/CD systems](https://golangci-lint.run/usage/install/#ci-installation).

## Documentation

Documentation is hosted at https://golangci-lint.run.

## Badges

![Build Status](https://github.com/golangci/golangci-lint/workflows/CI/badge.svg)
[![License](https://img.shields.io/github/license/golangci/golangci-lint)](/LICENSE)
[![Release](https://img.shields.io/github/release/golangci/golangci-lint.svg)](https://github.com/golangci/golangci-lint/releases/latest)
[![Docker](https://img.shields.io/docker/pulls/golangci/golangci-lint)](https://hub.docker.com/r/golangci/golangci-lint)
[![Github Releases Stats of golangci-lint](https://img.shields.io/github/downloads/golangci/golangci-lint/total.svg?logo=github)](https://somsubhra.com/github-release-stats/?username=golangci&repository=golangci-lint)

## Contributors

This project exists thanks to all the people who contribute. [How to contribute](https://golangci-lint.run/contributing/quick-start/).
```

# 我的git操作记录

## 修改作者和邮箱
https://stackoverflow.com/questions/750172/how-to-change-the-author-and-committer-name-and-e-mail-of-multiple-commits-in-gi
```shell
git rebase -i 82350e5
git commit --amend --author "Bai Yingjie <byj.tea@gmail.com>"
git rebase --continue
```
用`--exec`选项可以让这个过程更丝滑一些:
```
git rebase 82350e5 --exec 'git commit --amend --author="Bai Yingjie <byj.tea@gmail.com>"'
```

## 修改commit时间
```shell
git commit --amend --date="Tue 20 Dec 2021 14:14:22 AM UTC" --no-edit
```

## 同步upstream的更改
我有crosstool-ng的本地repo  
`git@gitlabe1.ext.net.nokia.com:godevsig/crosstool-ng.git`
里面有两个分支, godev和master

现在我想同步上游repo的更改
### 复杂步骤

```shell
# 直接拉上游更改
git checkout master
git pull https://github.com/crosstool-ng/crosstool-ng
# 上游更新push到组织repo. 此步必须
git push

# 第二步, 切换到godev分支, 拉刚才push的master
git checkout godev
git pull origin master
提示merge失败
```

### 简单步骤
```shell
# 直接拉上游更改
git checkout master
git pull https://github.com/crosstool-ng/crosstool-ng

# 第二步, 切换到godev分支, rebase
git checkout godev
git rebase master

# 用--skip可以跳过某个冲突的commit, 后续的commit会继续
git rebase --skip
```

## 使用不同的用户名commit
```
git commit --author="Bai Yingjie <byj.tea@gmail.com>"
```

## 同步remote已删除的分支
```shell
# 查看
$ git remote show origin
# 删除本地多余分支, 但已经checkout的分支还在
$ git remote prune origin

# 手动删除本地checkout的分支
git branch -d [branch_name]
```

## 找到当前branch最近的tag
```shell
yingjieb@9102a93a554e /repo/yingjieb/godevsig/golang-go
$ git describe --tags --abbrev=0
go1.13.15
```

## 其他分支的修改拿到本分支
比如我在godevsig分支有个commit:
```
e164f53422  2020-10-22  Bai Yingjie     godevsig: add support to use gccgo for ppc
```
现在我想把它拿到release-branch.go1.13分支
```shell
# 先更新本地working tree
git checkout release-branch.go1.13

# cherry pick后, 自动会commit
git cherry-pick e164f53422

# 在release-branch.go1.13分支, 就有这个改动了. 但commit id不一样
222b096725  2020-10-22  Bai Yingjie     godevsig: add support to use gccgo for ppc
```

## diff和patch
比如A和B都是一个repo的两个拷贝.
某种原因, 我想把A里面还没有commit的改动, 拿到B里面来
```shell
#在A中:
git diff > ppc.patch
#在B中:
git apply /path/to/A/ppc.patch
```

## 从其他git repo导入到新git库

### Git global setup
```shell
git config --global user.name "Bai Yingjie"
git config --global user.email "yingjie.bai@nokia-sbell.com"
```

### Create a new repository
```shell
git clone git@bhgitlab.int.net.nokia.com:godev/golang-go.git
cd golang-go
touch README.md
git add README.md
git commit -m "add README"
git push -u origin master
```

### Push an existing folder
```shell
cd existing_folder
git init
git remote add origin git@bhgitlab.int.net.nokia.com:godev/golang-go.git
git add .
git commit -m "Initial commit"
git push -u origin master
```

### Push an existing Git repository
```shell
cd existing_repo
git remote rename origin old-origin
git remote add origin git@bhgitlab.int.net.nokia.com:godev/golang-go.git
git push -u origin --all
git push -u origin --tags

#我的操作
cd crosstool-ng
git remote rename origin old-origin
git remote add origin git@gitlabe1.ext.net.nokia.com:godevsig/crosstool-ng.git
#我只想有master分支
git push -u origin master
#我指向有最新的tag
git push -u origin crosstool-ng-1.24.0
```

其他方法
```shell
#直接改origin
git remote set-url origin git@bhgitlab.int.net.nokia.com:godev/golang-go.git

#或者加个remote
git remote add gitlab git@bhgitlab.int.net.nokia.com:godev/golang-go.git
#把之前默认的origin改成gitlab
git push gitlab master
```

## git新建远程分支
比如现在在master分支上
```shell
# 创建本地分支, 并切换到这个新建的分支
git checkout -b test
# 做修改
...
# push到远程分支, 格式git push <远程主机名> <本地分支名>:<远程分支名>
git push origin test:test
```

删除分支
```shell
git push origin :test //将一个空分支推送到远程即为删除
//或者
git push origin --delete test
```

## git 查看log摘要
`git shortlog`可以看repo的提交信息, 每个作者名下的commit都是哪些.  
`git shortlog -s`不输出具体的commit, 只给出数目统计

比如:
```shell
$ git shortlog -s
     1  Author A
    42  Author B
     5  Author C
```

## git checkout branch
这是个老生常谈的问题了, 用`git branch -avv`看到的branch 名字  
比如
`remotes/origin/master-FNMS-63345-compatible-package-log-for-vonumgmt-gongc`

用下面的命令checkout并跟踪branch, 跟踪的意思是git pull的时候会更新当前的working copy  
`git checkout master-FNMS-62052-Heartbeat_support`  
注意:
* 不要加-b选项, 我试下来-b会导致不track远程分支
* 去掉remotes/origin


## git commit set格式
git的命令很多时候支持传入commit set,
用`man gitrevisions`可以查到详细的语法格式  
比如:
```shell
#生成从A到B的所有commit的patch, 不包括A, 但包括B
git format-patch A..B
```

## git 修改commit
在本地还没提交的commit, 可以用`git rebase -i`来修改, 过程和`hg histedit`一样  
比如我要修改552476b75408 这个commit
```shell
77d3efca122b 2019-12-28 Bai Yingjie powerpc/io: use phys_addr_t in virt_to_phys/phys_to_virt
552476b75408 2019-11-25 Bai Yingjie powerpc/mpc85xx: also write addr_h to spin table for 64bit boot entry
d5c2c21e7a74 2019-12-28 Bai Yingjie powerpc32/booke: consistently return phys_addr_t in __pa()
46cf053efec6 2019-12-22 Linus Torvalds Linux 5.5-rc3
```
那我要找到它的前一个commit:  
`git rebase -i d5c2c21e7a74`

把要修改的那条commit的状态从pick改为edit
此时, 修改要修改的文件, commit必须用:
`git commit --all --amend`

确认无误后提交  
然后用
`git rebase --continue`
来完成这次修改

## git undo reset
`git reset`会把commit丢弃掉, 比如
```shell
Linux Mint 19.1 Tessa $ git ls
77d3efca122b 2019-12-28 Bai Yingjie powerpc/io: use phys_addr_t in virt_to_phys/phys_to_virt
552476b75408 2019-11-25 Bai Yingjie powerpc/mpc85xx: also write addr_h to spin table for 64bit boot entry
d5c2c21e7a74 2019-12-28 Bai Yingjie powerpc32/booke: consistently return phys_addr_t in __pa()
46cf053efec6 2019-12-22 Linus Torvalds Linux 5.5-rc3
```
我执行了
```shell
git reset 552476b75408
```
最上面的commit就没了. 如果想把它找回来, 这时候就要用`git reflog`找到对应的操作历史
```shell
Linux Mint 19.1 Tessa $ git reflog
552476b75408 (HEAD -> master) HEAD@{0}: reset: moving to 552476b75408
77d3efca122b HEAD@{1}: rebase -i (finish): returning to refs/heads/master
77d3efca122b HEAD@{2}: rebase -i (pick): powerpc/io: use phys_addr_t in virt_to_phys/phys_to_virt
552476b75408 (HEAD -> master) HEAD@{3}: rebase -i (pick): powerpc/mpc85xx: also write addr_h to spin table for 64bit boot entry
d5c2c21e7a74 HEAD@{4}: rebase -i (pick): powerpc32/booke: consistently return phys_addr_t in __pa()
46cf053efec6 (tag: v5.5-rc3, origin/master, origin/HEAD) HEAD@{5}: rebase -i (start): checkout 46cf053efec6
```
结果如下:
```shell
yingjieb@yingjieb-VirtualBox ~/repo/linux
Linux Mint 19.1 Tessa $ git reset 'HEAD@{1}'
yingjieb@yingjieb-VirtualBox ~/repo/linux
Linux Mint 19.1 Tessa $ git ls
77d3efca122b 2019-12-28 Bai Yingjie powerpc/io: use phys_addr_t in virt_to_phys/phys_to_virt
552476b75408 2019-11-25 Bai Yingjie powerpc/mpc85xx: also write addr_h to spin table for 64bit boot entry
d5c2c21e7a74 2019-12-28 Bai Yingjie powerpc32/booke: consistently return phys_addr_t in __pa()
46cf053efec6 2019-12-22 Linus Torvalds Linux 5.5-rc3
```

## git调整commit顺序
调整前, 我想把3d5ecc00a315 和ca81e2ba497d 顺序调换一下
```shell
b0389050a303 2019-12-28 Bai Yingjie powerpc/io: use phys_addr_t in virt_to_phys/phys_to_virt
3d5ecc00a315 2019-12-28 Bai Yingjie powerpc32/booke: consistently return phys_addr_t in __pa()
ca81e2ba497d 2019-11-25 Bai Yingjie powerpc/mpc85xx: also write addr_h to spin table for 64bit boot entry
46cf053efec6 2019-12-22 Linus Torvalds Linux 5.5-rc3
```
用这个命令:
```shell
#-i表示--interactive, 46cf053efec6是他们公共的父节点
git rebase -i 46cf053efec6
```
这个命令会跳出编辑框来, 只要把要调整的两行顺序调换一下就好了. 和`hg histedit`过程差不多

调整后:
```shell
77d3efca122b 2019-12-28 Bai Yingjie powerpc/io: use phys_addr_t in virt_to_phys/phys_to_virt
552476b75408 2019-11-25 Bai Yingjie powerpc/mpc85xx: also write addr_h to spin table for 64bit boot entry
d5c2c21e7a74 2019-12-28 Bai Yingjie powerpc32/booke: consistently return phys_addr_t in __pa()
46cf053efec6 2019-12-22 Linus Torvalds Linux 5.5-rc3
```

## linux kernel:找到一个函数/字符串 相关的commit
比如, `drivers/i2c/busses/i2c-ocores.c`里面, 3.10的时候还有`of_i2c_register_devices`, 但4.9就没有了.

我想找到哪个commit删除了这个函数的使用, 它又用了什么新方法?
```shell
git log drivers/i2c/busses/i2c-ocores.c -S of_i2c_register_devices
# 新修改: -S不能在最后
git log -S __noreturn arch/powerpc/include/asm/machdep.h
```

## linux kernel: 找到一个commit对应的kernel release版本
```shell
Linux Mint 19.1 Tessa $ git show 687b81d083c08
yingjieb@yingjieb-VirtualBox ~/repo/linux
Linux Mint 19.1 Tessa $ git describe --contains 687b81d083c08
v3.12-rc1~140^2~14
```

## 保存现场 两个现场切换 git stash
```shell
# 保存现场, --all包括所有untracked和iggnored文件
Linux Mint 19 Tara $ git stash push --all
Saved working directory and index state WIP on hxt-dev-v17.08: 7030d3888 Extand RXD TXD size to 2048
# 可以用-m加说明
Linux Mint 19 Tara $ git stash push --all -m "aarch64 17.11"
Saved working directory and index state On 17.11: aarch64 17.11

# 看看当前的stash
Linux Mint 19 Tara $ git stash list
stash@{0}: WIP on hxt-dev-v17.08: 7030d3888 Extand RXD TXD size to 2048
# 看某个stash的改动
Linux Mint 19 Tara $ git stash show stash@{0}
 config/common_base | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
# 此时我切换到一个新的tag v18.05, 编译, 再stash; 
Linux Mint 19 Tara $ git stash list
stash@{0}: WIP on (no branch): a5dce5555 version: 18.05.0
stash@{1}: WIP on hxt-dev-v17.08: 7030d3888 Extand RXD TXD size to 2048
# 还是切换回hxt-dev-v17.08
git checkout hxt-dev-v17.08
# 这里用apply, 而不是pop; 因为我想反复切换现场
git stash apply stash@{1}
#其实还是pop好
git stash pop stash@{1}

# show tag with message
git tag -ln
```

## git查看某个版本的文件, 不checkout
```shell
git show remotes/origin/releases:drivers/net/mlx5/mlx5_rxtx.h
```

## git设置允许push到非bare的库, 在remote端运行
```shell
git config receive.denyCurrentBranch ignore
```
## git取消commit
如果commit还没有push, 可以直接
`git reset changset`
此时工作区的内容还是保留的

## git大文件支持
```shell
wget https://github.com/git-lfs/git-lfs/releases/download/v2.0.0/git-lfs-linux-amd64-2.0.0.tar.gz
```
https://github.com/git-lfs/git-lfs/wiki/Tutorial
```shell
git lfs track "*.tar.*"
git add .gitattributes
git add *
git commit
git lfs ls-files
```
注: 好像不好用

## git打包
```shell
yingjieb@yingjieb-gv /local/mnt/workspace/common/ls/kernel
$ git archive remotes/qserver/qserver-4.9 -o ../../archives/kernel.tar.gz
git archive HEAD -o ../dpdk-intel-only-cacheline64.tar.gz
```
## push到远程repo
我想把本地的linux的repo push到云服务器上.  
现在云服务器上创建个空的repo库, 没有这个空的库, 在本地push的时候会说没有远程的库
```shell
baiyingjie@caviumtech /home/share/src/alikernel/caviumkernel
$ git init
Initialized empty Git repository in /home/share/src/alikernel/caviumkernel/.git/
```
然后在本地, 先要add一个remote, 网上都是图省事add origin, 但我这里是不行的, 因为origin是我的上游库, 我还要拿来更新

注意这里的ssh库的格式
```shell
byj@mint ~/repo/git/cavium/thunder/sdk/sdk-master/linux/kernel/linux-aarch64
$ git remote add caviumtech ssh://baiyingjie@caviumtech.cn/home/share/src/alikernel/caviumkernel
$ git remote -v
caviumtech    ssh://baiyingjie@caviumtech.cn/home/share/src/alikernel/caviumkernel (fetch)
caviumtech    ssh://baiyingjie@caviumtech.cn/home/share/src/alikernel/caviumkernel (push)
origin    git://cagit1.caveonetworks.com/thunder/sdk/linux-aarch64.git (fetch)
origin    git://cagit1.caveonetworks.com/thunder/sdk/linux-aarch64.git (push)
```
然后就可以push了, 这里面的-u据说是:
> 由于远程库是空的，我们第一次推送master分支时，加上了-u参数，Git不但会把本地的master分支内容推送的远程新的master分支，还会把本地的master分支和远程的master分支关联起来，在以后的推送或者拉取时就可以简化命令。**但我怀疑这里有个副作用, 就是把本地的分支指向了这个远程分支, 而我需要保留这个指向到origin**

```shell
byj@mint ~/repo/git/cavium/thunder/sdk/sdk-master/linux/kernel/linux-aarch64
$ git push caviumtech alibaba-4.2
```

## 单一分支clone
`git clone git://cagit1.caveonetworks.com/thunder/boot/atf.git -b thunder-master --single-branch --depth=10`

## git下载远程分支
git默认会把remote上的更新都fetch下来的, 但我这里想下的一个分支为什么没有呢?
```shell
$ git branch -avv
* thunder-master                efe9731 [origin/thunder-master] i2c: thunderx: update to v8
  remotes/origin/thunder-master efe9731 i2c: thunderx: update to v8
$ cat .git/FETCH_HEAD 
efe97310c4c06b046f2f14006ccc990243da9307    branch 'thunder-master' of git://cagit1.caveonetworks.com/thunder/sdk/linux-aarch64
```
```
git fetch origin alibaba-4.2
```
以上这句会更新.git/FETCH_HEAD, 不推荐用

关键是这里
```shell
$ cat .git/config 
[core]
repositoryformatversion = 0
filemode = true
bare = false
logallrefupdates = true
[remote "origin"]
url = git://cagit1.caveonetworks.com/thunder/sdk/linux-aarch64.git
fetch = +refs/heads/thunder-master:refs/remotes/origin/thunder-master
添加下面这行
fetch = +refs/heads/alibaba-4.2:refs/remotes/origin/alibaba-4.2
[branch "thunder-master"]
remote = origin
merge = refs/heads/thunder-master
```
* 这里面origin是个默认的名字, 来指代原始url

以后git branch -avv 就能看到两个
```shell
$ git branch -avv
* thunder-master                efe9731 [origin/thunder-master] i2c: thunderx: update to v8
  remotes/origin/alibaba-4.2    0e253af Compiler bug workaround!!!
  remotes/origin/thunder-master efe9731 i2c: thunderx: update to v8
```
然后就可以checkout了
`$ git checkout alibaba-4.2`

## 创建本地分支来跟踪远程分支
```shell
root@CVMX55 ~/src/git/LuaJIT
# git branch -avv
* master                3d4c9f9 [origin/master] FFI: Fix SPLIT pass for CONV i64.u64.
  remotes/origin/HEAD   -> origin/master
  remotes/origin/master 3d4c9f9 FFI: Fix SPLIT pass for CONV i64.u64.
  remotes/origin/v2.1   22e7b00 DynASM/x64: Fix for full VREG support.
root@CVMX55 ~/src/git/LuaJIT
# git checkout --track remotes/origin/v2.1
Branch v2.1 set up to track remote branch v2.1 from origin.
Switched to a new branch 'v2.1'
```
or
`# git checkout -b v2.1 remotes/origin/v2.1`

## git merge本地改动
先暂存本地改动
```shell
git stash
git merge origin/master
```
pop出来的时候, 已经merge好了?
`git stash pop`

## git比较某次commit的改动
```shell
git difftool a450650dd9cbfff454a1f5cd443b8a2a3ed3206f^!
git diff 15dc8^..15dc8
git show a450650dd9cbfff454a1f5cd443b8a2a3ed3206f
```

### 查看单个文件
```
git show 6194e7c4 src/omci/mealarm.go
```

## git比较本地修改
git add以后就不能直接diff本地修改了, 需要加cached选项
`git diff --cached`

## 还是犯了remote错误
背景是我在mint上拷贝了ma的库, 从美国更新, remote是美国, 可以看到有很多remote分支.

但用的命令是git pull, 它只会更新当前branch

服务器上的库是clone mint机器的, 所以此时remote是mint, 只有我曾经在mint上checkout过的两个本地分支;  
而我只更新了master, 而stable不是当前分支, 所以mint上的local的stable分支从来不变.  
导致服务器上的remotes/origin/stable分支也不变.  
所以fae从stable merge也就从来没有新东西.

解决办法是想办法把所有mint的本地分支都更新. 比如checkout后pull或者rebase(感觉rebase更靠谱点)
```shell
yingjie@cavium /cavium/repo/git/thunder/linux-aarch64
$ git branch -a
* fae
  thunder-master
  thunder-stable
  remotes/origin/HEAD -> origin/thunder-master
  remotes/origin/thunder-master
  remotes/origin/thunder-stable
```
thunder-stable 和 remotes/origin/thunder-stable不是一回事
前者是local的分支, 后者和origin/thunder-stable是一个东西, 是remote上的分支  
`git log thunder-stable` 和 `git log origin/thunder-stable`是两码事

## 为什么git clone一个本地的repo没有branch?
现象:从原始repo clone的库(比如叫repo1)用git branch -a看是有很多分支的;

但从repo1再clone为repo2, 就只有master分支了.

原因是: 
第一次clone: 对repo1来说, origin是原始库  
第二次clone: 对repo2来说, origin是repo1  
而clone会带origin库本地的branch, 第一次原始库里面的branch带到repo1里, 但不在repo1本地;  
所以第二次clone发现repo1本地没有branch, 就没clone

解决办法: 
1. 不能连到原始库:在repo1里面跟踪每个分支
2. 能连原始库:在repo2里面`git remote add` 原始库

[补充]: 
* git clone的是remote库当前的分支
* git fetch是说从remote库拿到最新的东西, "应该是包括所有分支的东西"
* git merge 分支名是说把一个分支merge到当前工作区(也可以说是当前分支), 默认会直接commit
* git pull 是前面两个动作的组合, fetch都会成功, 但如果本地的当前分支和remote的当前分支不是同一个, git会抱怨不知道merge哪个分支.
* 在fae分支push, 提示receive.denyCurrentBranch问题, 应该设为ignore
* git push如果和remote的当前分支一样就应该会成功
* git checkout -b fae应该隐含了commit操作, 或类似的操作, 因为本地repo会记住这个分支

## git切换分支
本地内容也会切换
```
git checkout master
git checkout fae
```

## 放弃本地更改
放弃本分支的修改
`git reset --hard`

完整的写法是
`git reset --hard origin/master`

## git merge 不commit
```
git merge master --no-commit
```

## git tag
新增一个tag
`git tag -a -m "tag 0.1" 0.1`

push到remote
`git push --tag`

## git log显示文件
```
git log --stat
```

## git使用kdiff查看改动
`$ git difftool 4bc4c5c6fdf06d1d7b8fd1b6dbe1f6d721d05389 965697b896aeb5993e973942aebac19945436e87 -d`

也可以和上一次比, 用revision~1表示上次的
`$ git difftool bc6c72ff37dd9deb7c83d9bf77e713b747976140 bc6c72ff37dd9deb7c83d9bf77e713b747976140~1 -d`

## git更改remote
```shell
git remote set-url origin ssh://byj@192.168.1.217/home/byj/repo/git/thunder/ma/sdk/linux/kernel/linux-aarch64/
git remote -v
```

## git 从别人那里pull
```
$ git pull ssh://byj@192.168.1.217//home/byj/repo/git/thunder/ma/sdk master
```
后面的master是分支名
不加的话, 会失败:
```
You asked to pull from the remote 'ssh://byj@192.168.1.217/home/byj/repo/git/thunder/ma/sdk', but did not specify
a branch. Because this is not the default configured remote
for your current branch, you must specify a branch on the command line.
```

## git恢复文件
```
git checkout filename
```

## git clean, 删除所有未跟踪的文件
```
git clean -fdx
```
参考:
> For all git clean commands, add -n for dry-run.
> Clean all untracked files and directories, except ignored files (listed in .gitignore):
> git clean -fd
> Also clean ignored files:
> git clean -fdx
> Only clean ignored files (useful for removing build artifacts, to ensure a fresh build from scratch, without discarding new files/directories you haven't committed yet):
> git clean -fdX

## git查看clone地址
```
git remote -v
```
