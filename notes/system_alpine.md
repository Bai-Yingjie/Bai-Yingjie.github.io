# 现代化的工程系统
alpine linux的全部开发都在https://gitlab.alpinelinux.org/alpine
* 自己搭建的gitlab服务器, 允许外部用户注册, fork库, 并提交MR
* 使用gitlab-ci的CI/CD做build test
* 用gitlab issue来跟踪bug
* 文档也是repo管理, 使用[Antora Playbook](https://docs.antora.org/antora/latest/playbook/)发布, 网页入口是https://alpinelinux.org/

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
