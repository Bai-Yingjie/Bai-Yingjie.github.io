- [target\_lib的makefile](#target_lib的makefile)
- [Makefile进阶](#makefile进阶)
  - [用`eval`在配方(recipe)里定义变量并传递给后面的命令](#用eval在配方recipe里定义变量并传递给后面的命令)
  - [根据target改变变量](#根据target改变变量)
  - [传入变量到Makefile](#传入变量到makefile)
  - [Makefile配方传递变量](#makefile配方传递变量)
- [Makefile很简单](#makefile很简单)
  - [第一版](#第一版)
  - [第二版](#第二版)
  - [第三版](#第三版)
  - [封存版](#封存版)
- [make速查手册](#make速查手册)
- [make 命令行变量](#make-命令行变量)
  - [现象](#现象)
  - [原因](#原因)

# target_lib的makefile
这个Makefile用来生成动态库
```
VERSION  = 1
RELEASE  = 0.0 
NAME     = libapp-config.so
SONAME   = $(NAME).$(VERSION)
LIB      = $(SONAME).$(RELEASE)
LIBOBJS  = app_config.o
INCLUDES = app_config.h

override CFLAGS += -Wall -Werror

.PHONY: all install clean

all: $(LIB)

$(LIBOBJS): override CFLAGS += -fPIC

$(LIB): $(LIBOBJS)
    $(CC) $(LDFLAGS) -shared -Wl,-soname,$(SONAME) -o $@ $^

install:
    mkdir -p $(DESTDIR)/usr/include/isam
    install $(INCLUDES) $(DESTDIR)/usr/include/isam
    
    mkdir -p $(DESTDIR)/usr/lib
    install $(LIB) $(DESTDIR)/usr/lib
    ln -sf $(LIB) $(DESTDIR)/usr/lib/$(NAME)
    ln -sf $(LIB) $(DESTDIR)/usr/lib/$(SONAME)

clean:
    rm -f *~ *.o $(LIB)
```

# Makefile进阶
## 用`eval`在配方(recipe)里定义变量并传递给后面的命令
Makefile里面执行配方(recipe)的多行shell命令, 每一行都是独立的, 不能共享变量.  
但`eval`可以打破这个限制. 详见[Makefile Spec的eval function小节](https://www.gnu.org/software/make/manual/make.html#Eval-Function)

下面的例子中用了`eval`来定义"临时变量", 可跨多行shell命令使用.
```makefile
proxy := http://0.0.0.0:8080
repo_head := $(shell git describe --tags --abbrev=0)-$(shell git rev-parse --short HEAD)
site := artifactory.com
artifactory := rebornlinux-docker-local.$(site)

define docker_push #from,to
	@echo Are you sure to push $(1) to $(2) ?
	@read -p "if not press Ctrl+C - enter to continue" tmp
	docker tag $(1) $(2)
	docker push $(2)
endef

default: devkit

base: docker_build.base
devkit: docker_build.devkit

docker_build.%:
	$(eval DOCKERFILE := $(@:docker_build.%=Dockerfile.%))
	$(eval IMG := $(@:docker_build.%=rebornlinux/%))
	docker build -f $(DOCKERFILE) -t $(IMG):$(repo_head) \
		--build-arg HTTP_PROXY=$(proxy) --build-arg HTTPS_PROXY=$(proxy) \
		--build-arg http_proxy=$(proxy) --build-arg https_proxy=$(proxy) \
		.
	docker tag $(IMG):$(repo_head) $(IMG):latest
	$(call docker_push,$(IMG):$(repo_head),$(artifactory)/$(IMG):$(repo_head))
	$(call docker_push,$(IMG):latest,$(artifactory)/$(IMG):latest)

```
解释:
* `$(@:docker_build.%=Dockerfile.%)`是内置函数`$(patsubst pattern,replacement,text)`的简化版, 详见[Makefile patsubst函数](https://www.gnu.org/software/make/manual/make.html#index-patsubst-1)
* `DOCKERFILE`被`eval`了以后, 就可以被后面的命令使用了.

## 根据target改变变量
```makefile
BLDTAGS := stdlib,adaptiveservice
LDFLAGS = -X 'github.com/godevsig/gshellos.version=$(GIT_TAG)' -X 'github.com/godevsig/gshellos.buildTags=$(BLDTAGS)'

build: dep
    @go build -tags $(BLDTAGS) -ldflags="$(LDFLAGS)" -o bin ./cmd/gshell

debug: BLDTAGS := $(BLDTAGS),debug
debug: build ## Build debug binary to bin dir

lite: BLDTAGS := $(BLDTAGS)
lite: build ## Build lite release binary to bin dir

full: BLDTAGS := $(BLDTAGS),echart,database
full: build ## Build full release binary to bin dir
```
关键是
`target … : variable-assignment`

参考: [Target-specific Variable Values](https://www.gnu.org/software/make/manual/make.html#Target_002dspecific)

## 传入变量到Makefile
[参考](https://stackoverflow.com/questions/2826029/passing-additional-variables-from-command-line-to-make)

You have several options to set up variables from outside your makefile:

*   **From environment** - each environment variable is transformed into a makefile variable with the same name and value.

    You may also want to set `-e` option (aka `--environments-override`) on, and your environment variables will override assignments made into makefile (unless these assignments themselves use the [`override` directive](http://www.gnu.org/software/make/manual/make.html#Override-Directive) . However, it's not recommended, and it's much better and flexible to use `?=` assignment (the conditional variable assignment operator, it only has an effect if the variable is not yet defined):

    ```makefile
    FOO?=default_value_if_not_set_in_environment
    ```

    Note that certain variables are not inherited from environment:

    *   `MAKE` is gotten from name of the script
    *   `SHELL` is either set within a makefile, or defaults to `/bin/sh` (rationale: commands are specified within the makefile, and they're shell-specific).
*   **From command line** - `make` can take variable assignments as part of his command line, mingled with targets:

    ```sh
    make target FOO=bar
    ```

    But then _all assignments to `FOO` variable within the makefile will be ignored_ unless you use the [`override` directive](http://www.gnu.org/software/make/manual/make.html#Override-Directive) in assignment. (The effect is the same as with `-e` option for environment variables).

*   **Exporting from the parent Make** - if you call Make from a Makefile, you usually shouldn't explicitly write variable assignments like this:

    ```makefile
    # Don't do this!
    target:
            $(MAKE) -C target CC=$(CC) CFLAGS=$(CFLAGS)
    ```

    Instead, better solution might be to export these variables. Exporting a variable makes it into the environment of every shell invocation, and Make calls from these commands pick these environment variable as specified above.

    ```makefile
    # Do like this
    CFLAGS=-g
    export CFLAGS
    target:
            $(MAKE) -C target
    ```

    You can also export _all_ variables by using `export` without arguments.

## Makefile配方传递变量
比如下面的例子的作用是用gofmt检查go文件的code格式, 有差异就出错
```makefile
format: ## Check coding style
    @DIFF=`gofmt -d .`; echo "$$DIFF"; test -z "$$DIFF"
```
比如格式不对:
```sh
$ make format
diff -u log/log.go.orig log/log.go
--- log/log.go.orig     2020-09-22 12:53:25.561731789 +0000
+++ log/log.go  2020-09-22 12:53:25.561731789 +0000
@@ -30,7 +30,7 @@

 // All predefined loglevels
 const (
-       Ltrace  Loglevel = 1
+       Ltrace Loglevel = 1
        Ldebug Loglevel = 2
        Linfo  Loglevel = 3
        Lwarn  Loglevel = 4
make: *** [Makefile:8: format] Error 1
```

要点:
* 配方要写在一行, 用`;`或者`&&`分隔. 多行要用`\`来连接
没有`\`的话, 每行都会被make**单独执行**, 也就是说变量不能多行传递的.
* `$`需要转义. 用`$$`
* echo后面跟引号是保留原始换行

结论: makefile里面的配方(recipe)是特殊的语法, 先由make解析一遍再调用shell命令. 具体来讲, 是make先展开变量, 所以一个`$`就会被make直接展开. 要把`$`传递给`shell`, 就要转义`$`

比如
```makefile
COVER_GOAL := 90
coverage: ## Generate global code coverage report
    @OUT=$$(./tools/coverage.sh "${PKG_LIST}"); echo "$$OUT"; \
    echo "$$OUT" | tail -n1 | awk -F"\t*| *|%" '{if ($$3<${COVER_GOAL}) {print "Coverage goal: ${COVER_GOAL}% not reached"; exit 1}}'
```
* COVER_GOAL就是一个`$`直接展开的
* 要给shell看的`$`, 包括awk里面的`$`, 都需要转义(即两个`$`)

# Makefile很简单
我至今还没真正搞懂make怎么工作的, 但读了下面的说明, 尝试写了比较"复杂"的makefile后, 感觉还挺简单的
详细说明: https://www.gnu.org/software/make/manual/make.html

需要注意的:
* make有两个阶段
    * 第一阶段读所有被包含的makefile文件, 初始化变量, 构建依赖树
    * 第二阶段执行配方(recipe)
* 变量的展开规则是: 立即展开发生在第一阶段; 延迟展开发生在需要时

```
immediate = deferred
immediate ?= deferred
immediate := immediate
immediate ::= immediate
immediate += deferred or immediate
immediate != immediate

define immediate
  deferred
endef

define immediate =
  deferred
endef

define immediate ?=
  deferred
endef

define immediate :=
  immediate
endef

define immediate ::=
  immediate
endef

define immediate +=
  deferred or immediate
endef

define immediate !=
  immediate
endef
```
* 规则也有立即展开和延迟展开的区别

```
immediate : immediate ; deferred
        deferred
```

## 第一版
```makefile
build: godev/godev-tools

godev/godev-tools: godev/vscode godev/gobin godev/go3rdparty

godev/%: Dockerfile.%
    @docker build -f $< -t $@ .
    @mkdir -p godev && touch $@
```
* `godev/godev-tools`实际上匹配了两个规则, 一个是显式的规则, 一个是%规则; 效果是同时生效
* 多个%规则匹配目标的话, 只有一个生效; 在依赖都存在的前提下, make选择一个最"具体"的%规则

## 第二版
```makefile
goVersions := 1.10 1.11 1.12 1.13 1.14

build: 1.13

$(goVersions): %: godev/godev-tools\:%

godev/godev-tools\:%: Dockerfile.godev-tools godev/vscode\:v2 godev/gobin\:1.13 godev/go3rdparty\:1.13
    docker build --build-arg TAG=$* -f $< -t $@ .

godev/vscode\:%: Dockerfile.vscode
    docker build --build-arg TAG=$* -f $< -t $@ .

godev/gobin\:%: Dockerfile.gobin
    docker build --build-arg TAG=$* -f $< -t $@ .

godev/go3rdparty\:%: Dockerfile.go3rdparty
    docker build --build-arg TAG=$* -f $< -t $@ .
```

* 用`make 1.14 --dry-run`来模拟执行配方(recipe)
* `$*`表示%匹配到的部分
* 删掉了touch文件部分, 因为docker build会使用cache功能, 即使重新构建没有改动也很快

## 第三版
```makefile
goVersions := 1.10 1.11 1.12 1.13 1.14
dockerFiles := $(wildcard Dockerfile.*)
images := $(subst Dockerfile.,, $(dockerFiles))

build: 1.13

$(goVersions): %: godev/godev-tools\:%

define GEN_TOOLS_RULE
godev/godev-tools\:$(version): godev/vscode\:v2
godev/godev-tools\:$(version): godev/gobin\:1.13
godev/godev-tools\:$(version): godev/go3rdparty\:1.13
endef

$(foreach version, $(goVersions), $(eval $(GEN_TOOLS_RULE)))

define GEN_IMG_RULE
godev/$(image)\:%: Dockerfile.$(image)
    docker build --build-arg TAG=$$* -f $$< -t $$@ .
endef

$(foreach image, $(images), $(eval $(GEN_IMG_RULE)))
```
* 这个版本用foreach和eval自动生成规则
* eval会展开2次, 所以这里的变量定义里加两个$; 没错, define是变量定义
* %规则和显式规则能同时生效, 在GEN_TOOLS_RULE里面, 每行的依赖是增量式的. 如果依赖都写在一行, 解析的时候似乎空格没有起到作用
* 这个版本稍显繁琐, 但结构更清晰, 避免了重复; 只是为了演示makefile的能力; 实际还是第二个版本简洁

## 封存版
```makefile
goVersions := 1.10.8 1.11.13 1.12.17 1.13.12 1.14.4
officialTags := $(foreach v, $(goVersions), official-$(v))
repoHead := $(shell git rev-parse --short HEAD)
artifactory := artifactory-blr1.int.net.nokia.com
godevsigArtifactory := godevsig-docker-local.$(artifactory)
img := godevsig/devtool

define docker_push #from,to
    @echo Are you sure to push $(1) to $(2) ?
    @read -p "if not press Ctrl+C - enter to continue" tmp
    docker tag $(1) $(2)
    docker push $(2)
endef

default: godevsig
    $(call docker_push,$(img):latest,$(godevsigArtifactory)/$(img):latest)

#customized by godevsig(https://gitlabe1.ext.net.nokia.com/godevsig) with gccgo supporting ppc/ppc64 
godevsig: godevsig-$(repoHead)
    docker tag $(img):$< $(img):latest
    $(call docker_push,$(img):$<,$(godevsigArtifactory)/$(img):$<)

godevsig-%: Dockerfile.gccgo
    docker build -f $< -t $(img):$@ .

#official gc go toolchain
official: official-1.13.12
    docker tag $(img):$< $(img):latest
    $(call docker_push,$(img):$<,$(godevsigArtifactory)/$(img):$<)

$(officialTags): official-%: Dockerfile.gc
    docker build --build-arg GOLANG_VERSION=$* -f $< -t $(img):$@ .
```

# make速查手册
https://www.gnu.org/software/make/manual/make.html

# make 命令行变量
## 现象
在Makefile里, 即使给JOBS赋值为1, 但最后执行目标all的时候, 总是33
文件出自`linux/tools/perf/Makefile`
```makefile
ifeq ($(JOBS),)
    JOBS := $(shell (getconf _NPROCESSORS_ONLN || egrep -c '^processor|^CPU[0-9]' /proc/cpuinfo) 2>/dev/null)
    ifeq ($(JOBS),0)
        JOBS := 1
    endif
endif
#这里, 我为了调试一个问题, 想临时关闭并行编译
JOBS = 1
all:
    #但最后结果却是, 这里的$(JOBS)总是33
    @printf ' BUILD: Doing '\''make \033[33m-j'$(JOBS)'\033[m'\'' parallel build\n'
    @$(MAKE) -f Makefile.perf --no-print-directory -j$(JOBS) O=$(FULL_O) $(SET_DEBUG) $@
```
## 原因
因为在调用这个makefile的时候, 从命令行传递过来很多变量, 其中就有JOBS=33  
类似这样
```sh
PATH="..." /usr/bin/make -j1 HOSTCC="gcc -O2 -I ... -I ...等选项" ARCH=mips CROSS_COMPILE="..." LD="... -m elf32btsmipn32" NO_LIBNUMA=1 JOBS=33 -C tools/perf/
```
根据解释:
https://stackoverflow.com/questions/2826029/  passing-additional-variables-from-command-line-to-make

make的变量来源有
* 环境变量, 即在make之前的变量: xxx=xxx make  
用-e 或--environments-override选项, 可以让环境变量override掉makefile里面的变量;  
但不推荐, 推荐在makefile里用?来表示如果没定义, 就定义. 比如:  
`FOO?=default_value_if_not_set_in_environment`
* 命令行变量: 本文中的case, 即make后面的变量, 类似: make xxx=xxx  
命令行变量默认会override makefile里的变量, 比如上面的case, makefile中的JOBS变量就被忽略了, 怎么改都不管用.  
可以使用override关键词来定义变量, 这样就可以改命令行变量了.  
https://www.gnu.org/software/make/manual/make.html#Override-Directive

```makefile
override variable = value
override variable := value
#这个是在命令变量variable基础上加
override variable += more text
```
* 从父make导入的变量, 即父make export过的变量, 比如
```
CFLAGS=-g
export CFLAGS
target:
        $(MAKE) -C target
```
