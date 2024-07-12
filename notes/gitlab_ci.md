- [在ubuntu上安装gitlab runner](#在ubuntu上安装gitlab-runner)
  - [命令参考汇总](#命令参考汇总)
- [安装和注册gitlab runner](#安装和注册gitlab-runner)
  - [runner介绍](#runner介绍)
  - [Group runner](#group-runner)
    - [Set up a group Runner manually](#set-up-a-group-runner-manually)
    - [前置条件:安装docker](#前置条件安装docker)
    - [安装runner](#安装runner)
    - [注册runner](#注册runner)
- [runner使用和`.gitlab-ci.yml`](#runner使用和gitlab-ciyml)
  - [runner管理](#runner管理)
  - [image和services](#image和services)
  - [docker runner和shell runner](#docker-runner和shell-runner)
  - [job和script](#job和script)
  - [全局配置](#全局配置)
  - [stages和workflow](#stages和workflow)
  - [include其他yml](#include其他yml)
  - [可配参数参考](#可配参数参考)
  - [预定义的变量](#预定义的变量)
- [How to do continuous integration like a boss](#how-to-do-continuous-integration-like-a-boss)
- [touble shooting](#touble-shooting)
  - [解决docker内git clone/下载失败问题](#解决docker内git-clone下载失败问题)
- [merge request](#merge-request)

gitlab支持pipeline, [官方详细参考, 很全, 很细节](https://docs.gitlab.com/ee/ci/yaml/README.html#before_script-and-after_script)
[docker方式参考, 实用](https://docs.gitlab.com/ee/ci/docker/using_docker_images.html#define-image-and-services-from-gitlab-ciyml)

[快速入门](https://docs.gitlab.com/ee/ci/quick_start/README.html)

# 在ubuntu上安装gitlab runner
## 命令参考汇总
以root用户运行
```shell
# 安装docker
apt update
apt install docker.io
# openstack默认mtu 1450, 而docker默认1500. 修改docker的mtu为1450
echo '{ "mtu":1450 }' > /etc/docker/daemon.json
systemctl restart docker

# 安装gitlab runner
# 指定版本15.4.2
curl -L --output /usr/local/bin/gitlab-runner "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/v15.4.2/binaries/gitlab-runner-linux-amd64"
chmod +x /usr/local/bin/gitlab-runner
useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
#修改concurrent为10, 即所有runner最大一共可以执行10个job
sed -i 's/concurrent = 1/concurrent = 10/' /etc/gitlab-runner/config.toml
usermod -aG docker gitlab-runner
systemctl start gitlab-runner

# 注册通用shell runner
gitlab-runner register \
  --url "https://gitlabe1.ext.net.nokia.com/" \
  --description "shell-rebornlinux" \
  --registration-token "GR1348941X4RqAaWzxPoe6YDPZKyk" \
  --executor "shell" \
  --tag-list "shell-generic"

# 注册通用docker runner
gitlab-runner register \
  --url "https://gitlabe1.ext.net.nokia.com/" \
  --description "docker-rebornlinux" \
  --registration-token "GR1348941X4RqAaWzxPoe6YDPZKyk" \
  --executor "docker" \
  --tag-list "docker-generic" \
  --docker-image alpine:latest \
  --run-untagged

# 注册专用runner
gitlab-runner register \
  --url "https://gitlabe1.ext.net.nokia.com/" \
  --description "docker-aports" \
  --registration-token "GR1348941X4RqAaWzxPoe6YDPZKyk" \
  --executor "docker" \
  --tag-list "docker-aports" \
  --limit 1 \
  --docker-image alpine:latest

# runner配置
cat /etc/gitlab-runner/config.toml
concurrent = 10
check_interval = 0

[session_server]
  session_timeout = 1800
[[runners]]
  name = "docker-aports"
  limit = 1
  url = "https://gitlabe1.ext.net.nokia.com/"
  id = 175571
  token = "TH7uqYiEYLAMwSz8BPZB"
  token_obtained_at = 2024-05-03T14:46:12Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/repo/distfiles:/var/cache/distfiles:rw"]
    shm_size = 0
```

# 安装和注册gitlab runner

## runner介绍
gitlab runner[简介](https://docs.gitlab.com/ee/ci/runners/README.html)是用来执行工程根目录下的`.gitlab-ci.yml`.
有三种类型的runner
*   [Shared](https://docs.gitlab.com/ee/ci/runners/README.html#shared-runners) (for all projects)
*   [Group](https://docs.gitlab.com/ee/ci/runners/README.html#group-runners) (for all projects in a group)
*   [Specific](https://docs.gitlab.com/ee/ci/runners/README.html#specific-runners) (for specific projects)

不同的job由不同的runner执行
这里我们实用Group runner

## Group runner
group的admin可以创建group runner. 到gitlab Settings页面 CI/CD下面的Runner配置页面, 找下面的信息
比如这个是https://gitlabe1.ext.net.nokia.com/groups/godevsig 的group runner信息
### Set up a group Runner manually
1.  [Install GitLab Runner](https://docs.gitlab.com/runner/install/)
2.  Specify the following URL during the Runner setup: `https://gitlabe1.ext.net.nokia.com/` 
3.  Use the following registration token during setup: `Aprw1hQ6nuxyra5dzVwQ` 
4.  Start the Runner!

### 前置条件:安装docker
[docker安装官方参考](https://docs.docker.com/engine/install/)
安装完毕后, 如果显示`docker ps` socket权限问题, 则需要把用户加入到docker组, 特别的, 这里要把`gitlab-runner`加进去
`sudo usermod -a -G docker gitlab-runner`

### 安装runner
增加gitlab源
```sh
# For Debian/Ubuntu/Mint
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash

# For RHEL/CentOS/Fedora
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
```

安装
```sh
# For Debian/Ubuntu/Mint
export GITLAB_RUNNER_DISABLE_SKEL=true; sudo -E apt-get install gitlab-runner

# For RHEL/CentOS/Fedora
export GITLAB_RUNNER_DISABLE_SKEL=true; sudo -E yum install gitlab-runner
```

### 注册runner
根据上面工程的信息, 注册runner
```sh
sudo gitlab-runner register \
  --url "https://gitlabe1.ext.net.nokia.com/" \
  --description "docker-godevsig" \
  --registration-token "Aprw1hQ6nuxyra5dzVwQ" \
  --executor "docker" \
  --tag-list "docker-generic" \
  --docker-image alpine:latest
```
New:
```sh
sudo gitlab-runner register \
  --url "https://gitlabe1.ext.net.nokia.com/" \
  --description "docker-rebornlinux" \
  --registration-token "GR1348941GUgbYxRVVanUsxWc_hSb" \
  --executor "docker" \
  --tag-list "docker-generic" \
  --docker-image alpine:latest \
  --run-untagged

sudo gitlab-runner register \
  --url "https://gitlabe1.ext.net.nokia.com/" \
  --description "shell-rebornlinux" \
  --registration-token "GR1348941GUgbYxRVVanUsxWc_hSb" \
  --executor "shell" \
  --tag-list "shell-generic" \
  --run-untagged
```

在ubuntu上, 看runner服务的状态
```sh
yingjieb@cloud-server-1:~$ systemctl status gitlab-runner
● gitlab-runner.service - GitLab Runner
   Loaded: loaded (/etc/systemd/system/gitlab-runner.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2020-09-19 16:11:41 UTC; 2 days ago
 Main PID: 1944 (gitlab-runner)
    Tasks: 35 (limit: 4915)
   CGroup: /system.slice/gitlab-runner.service
           └─1944 /usr/bin/gitlab-runner run --working-directory /home/gitlab-runner --config /etc/gitlab-runner/config.toml --service gitlab-runner --syslog --user gitlab-runner
```
* 工作目录是`/home/gitlab-runner`
* config文件`/etc/gitlab-runner/config.toml`
* 使用`gitlab-runner`用户

# runner使用和`.gitlab-ci.yml`
## runner管理
```sh
yingjieb@cloud-server-1:~$ gitlab-runner -h
支持很多命令
start
stop
restart
status
register
unregister
install
uninstall
等等
```

## image和services
* image: 可以放在default里面, 表示runner的基础docker镜像
* services: 也是docker镜像, 是给image提供服务的镜像
service的玩法是启动一个service镜像, 可以指定镜像, 可以修改镜像的entrypoint, 可以改默认

## docker runner和shell runner
runner可以run在docker里面, 也可以是实际host的shell
[docker runner使用说明](https://docs.gitlab.com/runner/executors/docker.html)
[shell runner使用说明](https://docs.gitlab.com/runner/executors/shell.html)

## job和script
[job](https://docs.gitlab.com/ee/ci/yaml/README.html#introduction)是runner的基础执行单元, job之间是并行执行的.
job是用户定义的, 但不能是[保留字](https://docs.gitlab.com/ee/ci/yaml/README.html#unavailable-names-for-jobs)
job的script是必须的, [可以有其他可选配置](https://docs.gitlab.com/ee/ci/yaml/README.html#configuration-parameters)

[官方例子for go:](https://gitlab.com/gitlab-org/gitlab-foss/-/blob/master/lib/gitlab/ci/templates/Go.gitlab-ci.yml)
```yaml
image: golang:latest

variables:
  # Please edit to your GitLab project
  REPO_NAME: gitlab.com/namespace/project

# The problem is that to be able to use go get, one needs to put
# the repository in the $GOPATH. So for example if your gitlab domain
# is gitlab.com, and that your repository is namespace/project, and
# the default GOPATH being /go, then you'd need to have your
# repository in /go/src/gitlab.com/namespace/project
# Thus, making a symbolic link corrects this.
before_script:
  - mkdir -p $GOPATH/src/$(dirname $REPO_NAME)
  - ln -svf $CI_PROJECT_DIR $GOPATH/src/$REPO_NAME
  - cd $GOPATH/src/$REPO_NAME

stages:
  - test
  - build
  - deploy

format:
  stage: test
  script:
    - go fmt $(go list ./... | grep -v /vendor/)
    - go vet $(go list ./... | grep -v /vendor/)
    - go test -race $(go list ./... | grep -v /vendor/)

compile:
  stage: build
  script:
    - go build -race -ldflags "-extldflags '-static'" -o $CI_PROJECT_DIR/mybinary
  artifacts:
    paths:
      - mybinary
```

## 全局配置
有几个配置[可以配成全局的](https://docs.gitlab.com/ee/ci/yaml/README.html#global-defaults)

## stages和workflow
stages是顺序执行的
```yaml
stages:
  - build
  - test
  - deploy
```
workflow是全局的执行条件, 是条件规则集合, 规则依次匹配
```yaml
workflow:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
      when: never
    - if: '$CI_PIPELINE_SOURCE == "push"'
      when: never
    - when: always
```
This example never allows pipelines for schedules or push (branches and tags) pipelines, but does allow pipelines in all other cases, including merge request pipelines.

workflow有[官方模板](https://docs.gitlab.com/ee/ci/yaml/README.html#workflowrules-templates)可以参考

## include其他yml
有4种include类型

| Method                                                                       | Description                                                    |
| ---------------------------------------------------------------------------- | -------------------------------------------------------------- |
| [`local`](https://docs.gitlab.com/ee/ci/yaml/README.html#includelocal)       | Include a file from the local project repository.              |
| [`file`](https://docs.gitlab.com/ee/ci/yaml/README.html#includefile)         | Include a file from a different project repository.            |
| [`remote`](https://docs.gitlab.com/ee/ci/yaml/README.html#includeremote)     | Include a file from a remote URL. Must be publicly accessible. |
| [`template`](https://docs.gitlab.com/ee/ci/yaml/README.html#includetemplate) | Include templates that are provided by GitLab.                 |

## 可配参数参考
[Parameter details](https://docs.gitlab.com/ee/ci/yaml/README.html#parameter-details)

## 预定义的变量
有些变量是[预定义的](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html)
比如
```
CI_PROJECT_DIR
CI_BUILDS_DIR
CI_PIPELINE_SOURCE 
CI_COMMIT_TAG
CI_COMMIT_BRANCH
```

其中几个在if条件里挺有用:

| Example rules                                        | Details                                                   |
| ---------------------------------------------------- | --------------------------------------------------------- |
| `if: '$CI_PIPELINE_SOURCE == "merge_request_event"'` | Control when merge request pipelines run.                 |
| `if: '$CI_PIPELINE_SOURCE == "push"'`                | Control when both branch pipelines and tag pipelines run. |
| `if: $CI_COMMIT_TAG`                                 | Control when tag pipelines run.                           |
| `if: $CI_COMMIT_BRANCH`                              | Control when branch pipelines run.                        |

# How to do continuous integration like a boss
参考[gitlab本身的ci配置](https://gitlab.com/gitlab-org/gitlab/blob/master/.gitlab-ci.yml)
和[这篇攻略](https://about.gitlab.com/blog/2017/11/27/go-tools-and-gitlab-how-to-do-continuous-integration-like-a-boss/)

# touble shooting
## 解决docker内git clone/下载失败问题
原因是docker内的mtu设置比host大.
修改方法如下
增加docker的配置文件, 指定mtu
```
yingjieb@cloud-server-1:~$ cat /etc/docker/daemon.json
{
  "mtu": 1400
}
```
然后重启docker daemon
`sudo systemctl restart docker`

# merge request
一个forked repo的developer给parent发出merge request, 会在forked repo下面执行gitlab ci的.
[过程如下](https://docs.gitlab.com/12.10/ee/ci/merge_request_pipelines/index.html#important-notes-about-merge-requests-from-forked-projects)
1.  Fork a parent project.
2.  Create a merge request from the forked project that targets the `master` branch in the parent project.
3.  A pipeline runs on the merge request.
4.  A maintainer from the parent project checks the pipeline result, and merge into a target branch if the latest pipeline has passed.

Currently, those pipelines are created in a **forked** project, not in the parent project. This means you cannot completely trust the pipeline result, because, technically, external contributors can disguise their pipeline results by tweaking their GitLab Runner in the forked project.

There are multiple reasons why GitLab doesn’t allow those pipelines to be created in the parent project, but one of the biggest reasons is security concern. External users could steal secret variables from the parent project by modifying `.gitlab-ci.yml`, which could be some sort of credentials. This should not happen.

目前的状态是, forked repo发出的merge request不能在parent执行.
有个proposal解决这个问题: [Allow fork pipelines to run in parent project](https://gitlab.com/gitlab-org/gitlab/-/issues/11934)
但现在的版本12.7.6还没有这个功能.
