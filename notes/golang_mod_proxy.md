- [go get 和 replace](#go-get-和-replace)
- [gocenter已经停止维护了](#gocenter已经停止维护了)
- [go center使用心得](#go-center使用心得)
- [gitlab ci clone私有库](#gitlab-ci-clone私有库)
- [go mode 使用私有repo](#go-mode-使用私有repo)
  - [使用ssh方式clone](#使用ssh方式clone)
    - [GOPROXY问题和解决](#goproxy问题和解决)
    - [go clean](#go-clean)
- [go mod 使用replace指定私有repo](#go-mod-使用replace指定私有repo)
- [artifactory和goproxy的问题汇总](#artifactory和goproxy的问题汇总)
  - [godevsig-gocenter-remote获取不到新版本](#godevsig-gocenter-remote获取不到新版本)
  - [gitlab默认不支持goproxy](#gitlab默认不支持goproxy)
- [artifactory的repository概念](#artifactory的repository概念)
  - [一个repository只对应一个类型](#一个repository只对应一个类型)
  - [generic类型](#generic类型)
  - [local repository](#local-repository)
  - [remote repository](#remote-repository)
  - [Virtual Repositories](#virtual-repositories)
- [GOPROXY和artifactory](#goproxy和artifactory)
  - [私有repo](#私有repo)
  - [私有GOPROXY](#私有goproxy)
  - [setup私有GOPROXY](#setup私有goproxy)
- [go list](#go-list)
  - [package模式](#package模式)
  - [module模式](#module模式)
- [国内可用的proxy](#国内可用的proxy)
- [go mod模式和普通模式的go get区别](#go-mod模式和普通模式的go-get区别)
  - [普通模式下](#普通模式下)
  - [go mod模式下](#go-mod模式下)
  - [go mode模式下的help](#go-mode模式下的help)
- [go mod使用记录](#go-mod使用记录)
  - [go.mod](#gomod)
  - [本地import和外部repo import](#本地import和外部repo-import)
  - [sum DB审查](#sum-db审查)
- [go mod机制](#go-mod机制)
  - [定义go module](#定义go-module)
  - [创建一个go module](#创建一个go-module)
  - [go.mod文件维护](#gomod文件维护)
  - [版本格式](#版本格式)
  - [兼容性公约](#兼容性公约)
    - [Using v2 releases](#using-v2-releases)
    - [Using v1 releases](#using-v1-releases)

# GOPROXY=direct
有时候github的库更新了, 但go get或者go mod等命令不能及时的更新版本号, 比如:
```sh
go get github.com/godevsig/adaptiveservice@master
```
总是取到master的次新版本, 即使等了一段时间也不行.

估计是proxy cache还没有更新, 用下面的命令可以及时更新:
```sh
GOPROXY=direct go get github.com/godevsig/adaptiveservice@master
```

# go get 和 replace
有replace的时候, 比如:
```go
module github.com/godevsig/gshellos

go 1.13

require (
    github.com/d5/tengo/v2 v2.7.0
    github.com/peterh/liner v1.2.1
)

replace github.com/d5/tengo/v2 => github.com/godevsig/tengo/v2 v2.7.1-0.20210415163628-cf3fef922971
```
直接`go get github.com/godevsig/tengo/v2`会报错:
```
go get: github.com/godevsig/tengo/v2@v2.7.0: parsing go.mod:
        module declares its path as: github.com/d5/tengo/v2
                but was required as: github.com/godevsig/tengo/v2

```

那么怎么更新replace的版本呢?

要先改go.mod, 最后一行指定branch是dev
```
replace github.com/d5/tengo/v2 => github.com/godevsig/tengo/v2 dev
```
然后执行`go get`
```
$ go get
go: finding github.com/godevsig/tengo/v2 dev
go: downloading github.com/godevsig/tengo/v2 v2.7.1-0.20210415163628-cf3fef922971
go: extracting github.com/godevsig/tengo/v2 v2.7.1-0.20210415163628-cf3fef922971

```
会自动更新go.mod文件, 最后一行变为:
```
replace github.com/d5/tengo/v2 => github.com/godevsig/tengo/v2 v2.7.1-0.20210415163628-cf3fef922971
```

# gocenter已经停止维护了
访问https://search.gocenter.io/, 显示gocenter服务已经终止了. 最后一句
> You’ve arrived at this page because the era of Bintray, JCenter, GoCenter, and ChartCenter has ended – as newer and better options have emerged.
We’re proud of the benefits they provided, as they helped spur software innovation and bolstered the work of brilliant developers. **But you know what they say. Don’t overstay your welcome. Know when it’s time to bow out. Make a graceful exit.**

> The Go team has built a module repository for Go developers called Pkg.go.dev.

所以, 之前的
```
GOPROXY="https://artifactory.com/artifactory/api/go/godevsig-go-virtual,direct"
```
要改成默认的:
```
GOPROXY="https://proxy.golang.org,direct"
```

如果没有你的package, 要自己添加以下: https://go.dev/about#adding-a-package
Data for the site is downloaded from [proxy.golang.org](https://proxy.golang.org/). We monitor the [Go Module Index](https://index.golang.org/index) regularly for new packages to add to pkg.go.dev. If you don’t see a package on pkg.go.dev, you can add it by doing one of the following:

*   Visiting that page on pkg.go.dev, and clicking the “Request” button. For example:
    https://pkg.go.dev/example.com/my/module

*   Making a request to proxy.golang.org for the module version, to any endpoint specified by the [Module proxy protocol](https://golang.org/cmd/go/#hdr-Module_proxy_protocol). For example:
    https://proxy.golang.org/example.com/my/module/@v/v1.0.0.info

*   Downloading the package via the [go command](https://golang.org/cmd/go/#hdr-Add_dependencies_to_current_module_and_install_them). For example:
    `GOPROXY=https://proxy.golang.org GO111MODULE=on go get example.com/my/module@v1.0.0`

# go center使用心得
* 使用`export GOPROXY=https://gocenter.io`时, 默认是不包括你的github项目的.
需要在主页https://search.gocenter.io/ 点添加module按钮来手动添加
* 似乎gocenter不会主动持续的对你的库更新版本, 比如通过gocenter来`go get`时, latest版本不会更新
    * 直接`go get github.com/godevsig/gshellos`不会拿到最新版本
    * 对于Pseudo-Versions来说, 需要用户指定版本号, 比如
    `go get github.com/godevsig/gshellos@6d66a9c`
    这样才会触发gocenter进行一次缓存
* tag规范:https://jfrog.com/blog/go-big-with-pseudo-versions-and-gocenter/

如果不是proxy管理的, 是可以直接更新的:
```
$ go get gitlab.com/godevsig/system
go: finding gitlab.com/godevsig/system latest
go: downloading gitlab.com/godevsig/system v0.0.0-20210115023458-e89ec0eae327
go: extracting gitlab.com/godevsig/system v0.0.0-20210115023458-e89ec0eae327
```

# gitlab ci clone私有库

参考网上的经验, 在`.gitlab-ci.yml`中加入`before_script`, 这样在每个script执行之前, 都会跑这个git配置, 意思是用git的url替换功能, "加工"url地址.
```
default:
  image:
    name: godevsig-docker-local.artifactory.com/godevsig/devtool:godevsig-ee82473
    entrypoint: [""]
  before_script:
    - git config --global url."https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.com".insteadOf "https://gitlab.com"

```
因为go默认使用https方式get代码, 这里被替换的url字符串为`https://gitlab.com`, 将其前面加上`https://gitlab-ci-token:${CI_JOB_TOKEN}`
使用gitlab内置的`CI_JOB_TOKEN`变量, 可以绕过私有库不能clone的限制, 把私有库clone下来.

# go mode 使用私有repo
比如`https://gitlab.com/godev/system.git`不是public, 不能直接clone
go build等命令没法获取到这个库.
我用replace关键词暂时解决了本地编译的问题
```
replace (
    github.com/d5/tengo/v2 => /repo/yingjieb/godevsig/tengo
    gitlab.com/godevsig/system => /repo/yingjieb/godevsig/system
)

require (
    github.com/d5/tengo/v2 v2.6.2
    github.com/peterh/liner v1.2.1
    gitlab.com/godevsig/system v0.0.0-00010101000000-000000000000
)
```

但这里的system版本号似乎很怪异:`v0.0.0-00010101000000-00000000000`
这个大概是个特殊的版本号, 因为go 命令其实没有办法获取到
因为go命令默认使用https去clone. 对于私有仓库来说, 对外不可见, 当然不能用https去get.

## 使用ssh方式clone
用git config命令, 把https方式clone改成git方式, 这样如果你本地有ssh权限clone这个私有库, go命令也能成功, 因为go命令底层也是调用git命令clone的.
```
$ git config --global url."git@mygitlab.com:".insteadOf "http://mygitlab.com/"

// 其实就是在 .gitconfig 增加了配置
$ cat ~/.gitconfig

[url "git@mygitlab.com:"]
insteadOf = http://mygitlab.com/

//注意： git@mygitlab.com: 后面有个冒号 :, 且 http://mygitlab.com 后面有 /
```

我的成功例子:
```
git config --global url."git@gitlab.com:godevsig".insteadOf "https://gitlab.com/godevsig"
```
会在`~/.gitconfig`中增加:
```
[url "git@gitlab.com:godevsig"]
    insteadOf = https://gitlab.com/godevsig
```

### GOPROXY问题和解决
设置git的ssh方式替换https方式后, 把go.mod里面的特殊行删除:
```
gitlab.com/godevsig/system v0.0.0-00010101000000-000000000000
```
然后使用`go get`命令更新go.mod
发现go命令能正确发现这个私有库的版本号`v0.0.0-20201225044511-46ec8cf4b75c`, 但下载失败, 看起来和GOPROXY的设置有关
```
yingjieb@73e57632f306 /repo/yingjieb/godevsig/gshell
$ go get
go: finding gitlab.com/godevsig/system/pidinfo latest
go: finding gitlab.com/godevsig/system latest
go: downloading gitlab.com/godevsig/system v0.0.0-20201225044511-46ec8cf4b75c
build gitlab.com/godevsig/gshell: cannot load gitlab.com/godevsig/system/pidinfo: gitlab.com/godevsig/system@v0.0.0-20201225044511-46ec8cf4b75c: reading https://artifactory.com/artifactory/api/go/godevsig-go-virtual/gitlab.com/godevsig/system/@v/v0.0.0-20201225044511-46ec8cf4b75c.zip: 500 Internal Server Error
```

修改私有库不经过GOPROXY是可以的:
```
GONOPROXY="*.net.com" go get
```

### go clean
下面的命令会把所有的cache都清除, 导致go get下次会全新下载
```
go clean -cache -modcache -i -r
```
不加参数的go clean好像都不删除cache的.


# go mod 使用replace指定私有repo
在gshell中, 引用了库`github.com/abs-lang/abs`, 但我想修改这个package, 但依旧保留`import "github.com/abs-lang/abs"`
怎么做? -- go.mod中使用replace关键词
```
$ cat go.mod
module gitlab.com/godev/gshell

go 1.13

replace github.com/abs-lang/abs => /repo/yingjieb/godev/abs

require (
    github.com/abs-lang/abs v0.0.0-20190921150801-05c699c0415e
    github.com/peterh/liner v1.2.1
}
```
上面replace的意思是用本地路径替换目标package, 但源代码不用修改.
如果希望用私有repo, replace要求私有repo要有版本号.

# artifactory和goproxy的问题汇总
## godevsig-gocenter-remote获取不到新版本
现象:
```
$ GONOSUMDB="gitlab.com/*" GOPROXY="https://artifactory.com/artifactory/godevsig-gocenter-remote,direct" GO111MODULE=on go get -v golang.org/x/tools/gopls

go: golang.org/x/tools/gopls upgrade => v0.5.1
```
问题: 应该获取到新版本v0.5.3
解决: 使用virtual repo, 而且必须加`api/go`前缀就能下载最新的tag
```
GONOSUMDB="gitlab.com/*" GOPROXY="https://artifactory.com/artifactory/api/go/godevsig-go-virtual,direct" GO111MODULE=on go get -v golang.org/x/tools/gopls
go: golang.org/x/tools/gopls upgrade => v0.5.3
```
注: 对`godevsig-gocenter-remote`加`api/go`是不行的.
原因: 不是virtual的repo, 好像只能当cache用.

## gitlab默认不支持goproxy
现象:
`godevsig-go-virtual`包括`godevsig-gocenter-remote`和`godevsig-gogitlab-remote`后者proxy ` https://gitlab.com`
但get `gitlab.com/godevsig/compatible`失败
```
GONOSUMDB="gitlab.com/*" GOPROXY="https://artifactory.com/artifactory/api/go/godevsig-go-virtual,direct" GO111MODULE=on go get -v gitlab.com/godevsig/compatible
go get gitlab.com/godevsig/compatible: no matching versions for query "upgrade"
```
而不加proxy能成功.
```
GO111MODULE=on go get -v gitlab.com/godevsig/compatible
get "gitlab.com/godevsig/compatible": found meta tag get.metaImport{Prefix:"gitlab.com/godevsig/compatible", VCS:"git", RepoRoot:"https://gitlab.com/godevsig/compatible.git"} at //gitlab.com/godevsig/compatible?go-get=1
go: downloading gitlab.com/godevsig/compatible v0.1.0
go: gitlab.com/godevsig/compatible upgrade => v0.1.0

```

原因:
https://docs.gitlab.com/ee/user/packages/go_proxy/
需要管理员手动打开, 在gitlab rails控制台
可以全部打开
`Feature.enable(:go_proxy)`
可以按project打开
```
Feature.enable(:go_proxy,  Project.find(1))  Feature.disable(:go_proxy,  Project.find(2))
```

进展: gitlab 13.3才有这个功能

临时方案: 从virtual中去掉gitlab的汇聚. 待gitlab go proxy使能后再加
临时只virtual gocenter的.
```
GONOSUMDB="gitlab.com/*" GOPROXY="https://artifactory.com/artifactory/api/go/godevsig-go-virtual,direct" GO111MODULE=on go get -v gitlab.com/godevsig/compatible
get "gitlab.com/godevsig/compatible": found meta tag get.metaImport{Prefix:"gitlab.com/godevsig/compatible", VCS:"git", RepoRoot:"https://gitlab.com/godevsig/compatible.git"} at //gitlab.com/godevsig/compatible?go-get=1
go: gitlab.com/godevsig/compatible upgrade => v0.1.0
```

添加写申请:
https://nsnsi.service-now.com/ess?id=create_ticket&sys_id=cdb84691db3d8f80012570600f96196a&returnPage=request_services&returnPage2=browse_it_services

# artifactory的repository概念
https://www.jfrog.com/confluence/display/JFROG/Repository+Management
repository包括:
*   [Local](https://www.jfrog.com/confluence/display/JFROG/Repository+Management#RepositoryManagement-LocalRepositories)
*   [Remote](https://www.jfrog.com/confluence/display/JFROG/Repository+Management#RepositoryManagement-RemoteRepositories)
*   [Virtual](https://www.jfrog.com/confluence/display/JFROG/Repository+Management#RepositoryManagement-VirtualRepositories)
*   Distribution
    *   [Release Bundle Repository](https://www.jfrog.com/confluence/display/JFROG/Release+Bundle+Repositories)
    *   [Bintray Distribution Repository](https://www.jfrog.com/confluence/display/JFROG/Bintray+Distribution+Repositories)
    
## 一个repository只对应一个类型
特别的, virtual的repository只能汇聚同一类型的实体repository.
虽然没有强制要求不能上传不同类型的artifact, 但不推荐这样.

## generic类型
generic类型的repository能放任何东西, 没有特殊的包管理.

## local repository
访问路径:
`http://<host>:<port>/artifactory/<local-repository-name>/<artifact-path>`

## remote repository
实际上是个proxy和cache
访问格式
`http://<host>:<port>/artifactory/<remote-repository-name>/<artifact-path>`
如果确定artifact已经被cache了, 可以直接访问:
``http://<host>:<port>/artifactory/<remote-repository-name>-cache/<artifact-path>``

## Virtual Repositories
虚拟的repository是汇聚同类repository用的.
搜索的顺序是先local的, 再remote的. 同类的需要看list顺序, 可以改的.

# GOPROXY和artifactory
JFrog Artifactory推出了`https://gocenter.io`
使用只需要
```
export GOPROXY=https://gocenter.io
```
这个代理完全能取代go1.13开始的默认GOPROXY设置.

注意: `GOPROXY`只在go mod模式下才有用. 使能go mod模式, 要么目录下有go.mod文件, 要么需要强制指定`GO111MODULE=on`

使用`GoCenter`比直接从github上下载更快, GoCenter网页也能显示更详细的信息.

## 私有repo
使用公共repo时, `GOSUMDB`要去默认的公共校验中心`sum.golang.org`校验文件安全性.
而私有库显然没有入这个"sum DB", 拉取私有库会报错.
通常使用`GOPRIVATE`环境变量传入需要bypass sum db的私有库

GoCenter提供了同时使用公共`gocenter.io`和私有库的方法:
```
export GOPROXY=https://gocenter.io,direct
export GOPRIVATE=*.internal.mycompany.com
```
这样能解决问题, 但没法防止私有库的owner去改tag, 改了tag, 就不能重复构建同样的版本了.

## 私有GOPROXY
私有GOPROXY能解决上面的问题.
私有GOPROXY能同时cache公共GOPROXY和内部私有repo
![](https://media.jfrog.com/wp-content/uploads/2020/05/06184621/GoProxyKnot-Diagram-4-1024x607.png.webp)
> In Artifactory, a combination of a remote repository for GoCenter, a remote Go module repository that points to private GitHub repos (for private modules) and a local Go module repository can be combined into a single virtual repository, to access as a single unit.

使用私有GOPROXY, GONOSUMDB来完成任务:
```
$ export GOPROXY="https://:@my.artifactory.server/artifactory/api/go/go"
$ export GONOSUMDB="github.com/mycompany/*,github.com/mypersonal/*"
```

## setup私有GOPROXY
[官方参考](https://www.jfrog.com/confluence/display/JFROG/Go+Registry)

这里看起来我需要:
* go remote repo: proxy gocenter
* go remote repo: proxy gitlab
* go vitual repo
* general local repo: 

参考: 
[artifactory的go registry说明](https://www.jfrog.com/confluence/display/JFROG/Go+Registry)
[经验文档1](https://jfrog.com/blog/why-goproxy-matters-and-which-to-pick/)
[经验文档2](https://jfrog.com/blog/how-to-use-gocenter-with-golang-1-13/)

# go list
## package模式
`go list` 用于显示当前目录下的package的import path
```
yingjieb@f8530ea27843 /repo/yingjieb/3rdparty/delve/pkg/proc
$ go list
github.com/go-delve/delve/pkg/proc
```

-f选项其实是个模板, 可以显示go tool内部的管理数据, 比如查看package的import了谁, 以及依赖谁(即所有递归的import)
```
% go list -f '{{ .Imports }}' github.com/davecheney/profile
[io/ioutil log os os/signal path/filepath runtime runtime/pprof]
% go list -f '{{ .Deps }}' github.com/davecheney/profile
[bufio bytes errors fmt io io/ioutil log math os os/signal path/filepath reflect runtime runtime/pprof sort strconv strings sync sync/atomic syscall text/tabwriter time unicode unicode/utf8 unsafe]
```

-deps 选项可以递归的list依赖. 也就是说默认不打印依赖.

列出当前目录下的package, 其中`...`是go tool通用的通配符, 匹配所有.
注意, 不要用`go list ...`, 意思是列出所有目录下的package
```
yingjieb@f8530ea27843 /repo/yingjieb/3rdparty/delve/pkg/proc
$ go list ./...
github.com/go-delve/delve/pkg/proc
github.com/go-delve/delve/pkg/proc/core
github.com/go-delve/delve/pkg/proc/core/minidump
github.com/go-delve/delve/pkg/proc/fbsdutil
github.com/go-delve/delve/pkg/proc/gdbserial
github.com/go-delve/delve/pkg/proc/linutil
github.com/go-delve/delve/pkg/proc/native
github.com/go-delve/delve/pkg/proc/test
github.com/go-delve/delve/pkg/proc/winutil

```

## module模式
module对应的是package的自然集合, 通常是一个repo.
`go list -m all` 用于列出所有的import的包(main包和main的依赖包)
```
The arguments to list -m are interpreted as a list of modules, not packages.
The main module is the module containing the current directory.
The active modules are the main module and its dependencies.
With no arguments, list -m shows the main module.
With arguments, list -m shows the modules specified by the arguments.
Any of the active modules can be specified by its module path.
The special pattern "all" specifies all the active modules, first the main
module and then dependencies sorted by module path.
A pattern containing "..." specifies the active modules whose
module paths match the pattern.
```


# 国内可用的proxy
https://goproxy.cn/
```
$ export GO111MODULE=on
$ export GOPROXY=https://goproxy.cn
```

# go mod模式和普通模式的go get区别
## 普通模式下
GOPROXY不起作用, go get直接从目标地址下载
```
go get -v github.com/naoina/go-stringutil
# 下载后的代码以src形式保存在GOPATH下的
./src/github.com/naoina/go-stringutil
```

## go mod模式下
```
# 使用godevsig的proxy
GOPROXY="https://artifactory.com:443/artifactory/godevsig-go-virtual" GO111MODULE=on go get -v github.com/naoina/go-stringutil
# 下载后的代码以pkg cache的形式保存在GOPATH下的
./pkg/mod/cache/download/github.com/naoina/go-stringutil
```
注: GOPROXY生效后, 在 https://artifactory.com/artifactory/webapp/#/artifacts/browse/tree/General/godevsig-gocenter-remote-cache 下, 能找到已经cache的package

## go mode模式下的help
连help输出也不一样
`GO111MODULE=on go help get`
* 第一步解析依赖, 对每个package pattern, 依次做版本检查:
    * 先检查最新的tag, 比如v1.2.3
    * 没有release的tag, 就用pre tag, 比如v0.0.1-pre1
    * 没有tag, 就用最新的commit
    * 可以用@version下载指定版本, @v1会下载v1的最新版本; @commitid也行
* 对于间接依赖, go get会follow go.mod的指示
* go get后面是空的情况下, get当前目录下的依赖
* 第二步时下载, build, install; 在package下面没有东西可以build的时候, build和install有可能被忽略
* 可以用`...`的pattern

# go mod使用记录
比如库`gitlab.com/godevsig/compatible`

## go.mod
使用`go mod init gitlab.com/godevsig/compatible`
得到如下go.mod: 声明了本库的module名. 官方说法是, 这个名字必须和库地址一致.见[cmd/go: Why not separate the module name and the download URL?](https://github.com/golang/go/issues/30242)
```
module gitlab.com/godevsig/compatible

go 1.13
```

## 本地import和外部repo import
这里的本地引用是说通一个repo下, 不同package之间的引用. 本地引用和外部引用没有任何区别:
在log同一级的目录msg下: main.go
```
package main

import "gitlab.com/godevsig/compatible/log"

func main() {
    lg := log.DefaultStream.NewLogger("msgdriven")

    lg.Infolnn("hello msg")
}
```
但本地引用情况下, 本地修改能够立即生效. 比如我在log包里加了一个API, 本地修改不入库, `lg.Infolnn()`, 本地其他包能够立刻引用新的API.

外部引用的情况下, 只认已经入库的内容. 因为`lg.Infolnn()`的修改还没有入库, 外部的repo是不知道的.
错误如下:
```
$ go build
# main
./hello.go:42:4: lg.Infolnn undefined (type *log.Logger has no field or method Infolnn)
```

## sum DB审查
默认配置下, go get/build发现go.mod文件后, 开启go mod模式, 自动拉取依赖的库. 但出现`410 Gone`错误
```
$ go build
go: finding gitlab.com/godevsig/compatible latest
go: finding gitlab.com/godevsig/compatible/log latest
go: downloading gitlab.com/godevsig/compatible v0.0.0-20200811070332-66acc8ba0617
verifying gitlab.com/godevsig/compatible@v0.0.0-20200811070332-66acc8ba0617: gitlab.com/godevsig/compatible@v0.0.0-20200811070332-66acc8ba0617: reading htt
ps://sum.golang.org/lookup/gitlab.com/godevsig/compatible@v0.0.0-20200811070332-66acc8ba0617: 410 Gone
```
看过程是它能取到版本, 但verifying checksum出错了. 因为go mod要去`sum.golang.org`获取校验, 但我们引用的module是私有库, 肯定在golang.org里面没有.

解决: 配置环境变量
`GOSUMDB=off`
参考: [Golang-执行go get私有库提示”410 Gone“ 解决办法](https://www.cnblogs.com/zhaof/archive/2020/02/21/12341688.html)

或者使用
```
GONOSUMDB="*.net.com/*"
```

# go mod机制
`go help modules` `go help go.mod` `go help mod`
go modules是go的官方包管理机制, 用于替代老的GOPATH环境变量来指定版本依赖的方式.

go tools1.13支持go modules. 
* 环境变量GO111MODULE=auto的默认方式下, 如果目录下有go.mod文件, 则使能go modules模式; 否则还是用老的GOPATH模式
* GOPATH在go modules模式下, 只用于存放下载的源码: GOPATH/pkg/mod, 和按照后的二进制: GOPATH/bin
* GO111MODULE=on 强制使用go modules模式

## 定义go module
一个go module是一个目录, 该目录下有go.mod文件, 该目录也称为module root.
比如下面的go.mod声明了一个module, 它的import path是`example.com/m`; 它还依赖指定版本的`golang.org/x/text`和`gopkg.in/yaml.v2`
```
module example.com/m

require (
        golang.org/x/text v0.3.0
        gopkg.in/yaml.v2 v2.1.0
)
```

## 创建一个go module
在工程目录下, 执行`go mod init example.com/m`会创建go.mod
go.mod文件创建后, go命令比如go build, go get会自动更新go.mod.
go命令会在当前目录找go.mod, 没有再到父目录及其再往上的父目录寻找go.mod

在哪里执行go命令, 那个目录就是main module. **只有**main module的go.mod文件的有`replace`和`exclude`关键词才有效; 不是main module, 这些关键词被忽略.

build list是构建main module的依赖列表
> A Go module is a collection of related Go packages that are versioned together as a single unit.

`go list`命令用于查看main module的build list
```
        go list -m              # print path of main module
        go list -m -f={{.Dir}}  # print root directory of main module
        go list -m all          # print build list
```

## go.mod文件维护
go.mod被设计为人和go命令都可读可修改. go命令比如go build, 如果发现源码里面有新增的`import example/m`关键词, 就会自动添加`example/m`的最新version到go.mod
* `go mod tidy`命令可以整理go.mod文件, 删除不再需要的module.
* `// indirect`指示间接依赖
* `go get`命令会更新go.mod的版本. 比如升级一个module, 那其他依赖这个module的modules也会被自动更新到相应版本.

## 版本格式
版本号的核心思想是版本号是可以比较新旧的.

对没有版本管理的repo, 假的版本号格式是:`v0.0.0-yyyymmddhhmmss-abcdefabcdef`
其中时间是commit时间, 后面是commit hash.

go的命令支持module版本指定:
* `v1.2.3`指定了一个具体的版本
* `v1`会被扩展成最新的v1版本
* `<v1.2.3`和`>=v1.5.6`
* `latest`被扩展成最新的tagged版本, 或者, 对于没有版本管理的库, 使用最新的commit
* `upgrade`: 和latest差不多
* `patch`: 和latest差不多
* 其他: commit hash, tag名, 分支名, 会选择源码的指定版本.
```
go get github.com/gorilla/mux@latest    # same (@latest is default for 'go get')
go get github.com/gorilla/mux@v1.6.2    # records v1.6.2
go get github.com/gorilla/mux@e3702bed2 # records v1.6.2
go get github.com/gorilla/mux@c856192   # records v0.0.0-20180517173623-c85619274f5d
go get github.com/gorilla/mux@master    # records current meaning of master
```

## 兼容性公约
如果一个package的新老版本的import路径一致, 那么新版本必须兼容老版本.
对不兼容的版本, 解决方案是import路径加v2, 比如:
go.mod里显式写明: `module example.com/m/v2`
在引用时也写明引用v2里面的一个package
```
import example.com/m/v2/sub/pkg
```

比如`urfave/cli`的v1和v2版本:
### Using v2 releases
`$ GO111MODULE=on go get github.com/urfave/cli/v2`

```
...
import (
  "github.com/urfave/cli/v2" // imports as package "cli"
)
...
```

### Using v1 releases

`$ GO111MODULE=on go get github.com/urfave/cli`

```
...
import (
  "github.com/urfave/cli"
)
...
```