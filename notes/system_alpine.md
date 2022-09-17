- [现代化的工程系统](#现代化的工程系统)
- [使用subgroup来组织repo](#使用subgroup来组织repo)
- [组织清爽, 源代码干净](#组织清爽-源代码干净)
- [aports](#aports)
  - [交叉编译](#交叉编译)
- [openrc](#openrc)
- [musl libc](#musl-libc)

# 现代化的工程系统
alpine linux的全部开发都在  
https://gitlab.alpinelinux.org/alpine  
mirror: https://git.alpinelinux.org/
* 自己搭建的gitlab服务器, 允许外部用户注册, fork库, 并提交MR
* 使用gitlab-ci的CI/CD做build test
* 用gitlab issue来跟踪bug
* 文档也是repo管理, 使用[Antora Playbook](https://docs.antora.org/antora/latest/playbook/)发布, 网页入口是 https://alpinelinux.org/

# 使用subgroup来组织repo
比如CI/CD工具库在`alpine/infra/docker/alpine-gitlab-ci`下面, 先是根alpine, 再是infra, 再是docker, 最后是repo

# 组织清爽, 源代码干净
比如`alpine-gitlab-ci/-/blob/master/overlay/usr/local/bin/build.sh`里面的shell 输出代码:
```sh
: "${CI_ALPINE_BUILD_OFFSET:=0}"
: "${CI_ALPINE_BUILD_LIMIT:=9999}"

msg() {
    local color=${2:-green}
    case "$color" in
        red) color="31";;
        green) color="32";;
        yellow) color="33";;
        blue) color="34";;
        *) color="32";;
    esac
    printf "\033[1;%sm>>>\033[1;0m %s\n" "$color" "$1" | xargs >&2
}

verbose() {
    echo "> " "$@"
    # shellcheck disable=SC2068
    $@
}

debugging() {
    [ -n "$CI_DEBUG_BUILD" ]
}

debug() {
    if debugging; then
        verbose "$@"
    fi
}

die() {
    msg "$1" red
    exit 1
}

capture_stderr() {
    "$@" 2>&1
}

report() {
    report=$1

    reportsdir=$APORTSDIR/logs/
    mkdir -p "$reportsdir"

    tee -a "$reportsdir/$report.log"
}
```

# aports
alpine支持的package都放在[aports](https://gitlab.alpinelinux.org/alpine/aports)这个库下面.
* main: alpine core team直接支持的package
* community: 由社区支持的package

参考: https://wiki.alpinelinux.org/wiki/Repositories

`/etc/apk/repositories`是package的配置
```
/ # cat /etc/apk/repositories
https://dl-cdn.alpinelinux.org/alpine/v3.15/main
https://dl-cdn.alpinelinux.org/alpine/v3.15/community
```
比如`https://dl-cdn.alpinelinux.org/alpine/v3.15/main`目录下包括了所有arch的预编译好的apk  
![](img/system_alpine_20220902115455.png)  
点进去看这些apk的修改时间是不一样的, 说明apk是按需编译的.

## 交叉编译
似乎可以用bootstrap.sh来生成交叉编译的工具链  
参考: [musl-cross-make](https://github.com/richfelker/musl-cross-make.git)

# openrc
alpine使用openrc  
https://wiki.alpinelinux.org/wiki/OpenRC

# musl libc
alpine使用musl libc, 我看好musl的轻量简洁.  
[几种libc的比较](http://www.etalabs.net/compare_libcs.html)
