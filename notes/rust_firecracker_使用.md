- [编译](#编译)
  - [编译firecracker](#编译firecracker)
  - [编译kernel](#编译kernel)
  - [编译rootfs](#编译rootfs)
- [运行](#运行)
  - [配置文件方式运行](#配置文件方式运行)
  - [rest API方式运行](#rest-api方式运行)
- [devctr镜像](#devctr镜像)
- [顶层cargo](#顶层cargo)
- [firecracker/tools/devtool脚本](#firecrackertoolsdevtool脚本)
  - [cmd_build](#cmd_build)
    - [先build seccompiler](#先build-seccompiler)
    - [再build rebase-snap](#再build-rebase-snap)
    - [build firecracker](#build-firecracker)
    - [build jailer](#build-jailer)
  - [build_kernel](#build_kernel)
  - [build_rootfs](#build_rootfs)
    - [firecracker/resources/tests/init.c](#firecrackerresourcestestsinitc)
    - [firecracker/resources/tests/fillmem.c](#firecrackerresourcestestsfillmemc)
    - [firecracker/resources/tests/readmem.c](#firecrackerresourcestestsreadmemc)
    - [做镜像](#做镜像)

# 编译
先clone代码:
`git clone https://github.com/firecracker-microvm/firecracker`

## 编译firecracker
firecracker是rust写的, 但编译不需要本地依赖rust环境, 而是在docker内完成的.
使用了docker image`public.ecr.aws/firecracker/fcuvm:v35`, 大小3.25G

因为使用了x86_64-unknown-linux-musl做为target, 所以最后的可执行文件是静态链接的
* 默认debug版本:  
`tools/devtool build`  
生成:  
`build/cargo_target/x86_64-unknown-linux-musl/debug/firecracker` 38M, 静态链接, 带符号

* 指定release版本  
`tools/devtool build --release`  
生成:  
`build/cargo_target/x86_64-unknown-linux-musl/release/firecracker` 4.1M, 静态链接, 带符号

## 编译kernel
`tools/devtool build_kernel -c resources/guest_configs/microvm-kernel-x86_64-5.10.config -n 8`  
生成:  
`build/kernel/linux-5.10/vmlinux-5.10-x86_64.bin` 42M, 带符号的linux elf, 模块全部编入kernel.

## 编译rootfs
`tools/devtool build_rootfs -s 300MB`  
生成:  
`build/rootfs/bionic.rootfs.ext4` 300M

# 运行
## 配置文件方式运行
`build/cargo_target/x86_64-unknown-linux-musl/release/firecracker --api-sock /tmp/firecracker.socket --config-file myvmconfig.json`  
会打印kernel启动过程, 并自动以root登陆

myvmconfig.json内容如下:
```json
{
  "boot-source": {
    "kernel_image_path": "build/kernel/linux-5.10/vmlinux-5.10-x86_64.bin",
    "boot_args": "console=ttyS0 reboot=k panic=1 pci=off",
    "initrd_path": null
  },
  "drives": [
    {
      "drive_id": "rootfs",
      "path_on_host": "build/rootfs/bionic.rootfs.ext4",
      "is_root_device": true,
      "partuuid": null,
      "is_read_only": false,
      "cache_type": "Unsafe",
      "io_engine": "Sync",
      "rate_limiter": null
    }
  ],
  "machine-config": {
    "vcpu_count": 2,
    "mem_size_mib": 1024,
    "smt": false,
    "track_dirty_pages": false
  },
  "balloon": null,
  "network-interfaces": [],
  "vsock": null,
  "logger": null,
  "metrics": null,
  "mmds-config": null
}
```

* 跑的是ubuntu, 带systemd的
* 启动迅速
* reboot会触发kernel退出, 但并不重启
* 没有网络接口
* 根文件系统挂在/dev/vda上
* VM配置了1024M内存, 但运行时firecracker进程占用95M, 虚拟内存1032M.

## rest API方式运行
firecracker启动的时候要指定一个API socket, 每个VM一个. 使用这个socket, 可以用rest API方式来运行和管理VM.

# devctr镜像
devctr是开发中使用的镜像, 所有的操作都通过这个镜像完成.
* 基于ubuntu18
* 安装了常用的开发工具
```shell
binutils-dev
clang
cmake
gcc
等等
```
* 安装了rust
```shell
curl https://sh.rustup.rs -sSf | sh -s -- -y
rustup target add x86_64-unknown-linux-musl
rustup component add rustfmt
rustup component add clippy-preview
rustup install "stable"
```
* 使用了开源的init程序, 静态编译版本
```docker
# Add the tini init binary.
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION_TAG}/tini-static-amd64 /sbin/tini
RUN chmod +x /sbin/tini
WORKDIR  "$FIRECRACKER_SRC_DIR"
ENTRYPOINT ["/sbin/tini", "--"]
```

# 顶层cargo

cargo.toml
```toml
[workspace]
members = ["src/firecracker", "src/jailer", "src/seccompiler", "src/rebase-snap"]
default-members = ["src/firecracker"]

[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
lto = true

[patch.crates-io]
kvm-bindings = { git = "https://github.com/firecracker-microvm/kvm-bindings", tag = "v0.5.0-1", features = ["fam-wrappers"] }
```
cargo的build系统会自动维护cargo.lock来描述版本信息.
下面的命令可以更新依赖的版本信息:

```shell
$ cargo update            # updates all dependencies
$ cargo update -p regex   # updates just “regex”
```

# firecracker/tools/devtool脚本
```shell
# By default, all devtool commands run the container transparently, removing
# it after the command completes. Any persisting files will be stored under
# build/.
# If, for any reason, you want to access the container directly, please use
# `devtool shell`. This will perform the initial setup (bind-mounting the
# sources dir, setting privileges) and will then drop into a BASH shell inside
# the container.
#
# Building:
#   Run `./devtool build`.
#   By default, the debug binaries are built and placed under build/debug/.
#   To build the release version, run `./devtool build --release` instead.
#   You can then find the binaries under build/release/.
#
# Testing:
#   Run `./devtool test`.
#   This will run the entire integration test battery. The testing system is
#   based on pytest (http://pytest.org).
#
# Opening a shell prompt inside the development container:
#   Run `./devtool shell`.
#
# Additional information:
#   Run `./devtool help`.
```

run_devctr函数写的很好. docker -v的z参数表示可以共享, 参考https://docs.docker.com/storage/bind-mounts/#configure-the-selinux-label
```shell
# Helper function to run the dev container.
# Usage: run_devctr <docker args> -- <container args>
# Example: run_devctr --privileged -- bash -c "echo 'hello world'"
run_devctr() {
    docker_args=()
    ctr_args=()
    docker_args_done=false
    while [[ $# -gt 0 ]];  do
        [[ "$1"  =  "--" ]] && {
            docker_args_done=true
            shift
            continue
        }
        [[ $docker_args_done =  true ]] && ctr_args+=("$1") || docker_args+=("$1")
        shift
    done

    # If we're running in a terminal, pass the terminal to Docker and run
    # the container interactively
    [[ -t 0 ]] && docker_args+=("-i")
    [[ -t 1 ]] && docker_args+=("-t")

    # Try to pass these environments from host into container for network proxies
    proxies=(http_proxy HTTP_PROXY https_proxy HTTPS_PROXY no_proxy NO_PROXY)
    for i in  "${proxies[@]}";  do
        if [[ !  -z ${!i} ]];  then
            docker_args+=("--env") && docker_args+=("$i=${!i}")
        fi
    done

    # Finally, run the dev container
    # Use 'z' on the --volume parameter for docker to automatically relabel the
    # content and allow sharing between containers.
    docker run "${docker_args[@]}" \
        --rm \
        --volume /dev:/dev \
        --volume "$FC_ROOT_DIR:$CTR_FC_ROOT_DIR:z" \
        --env OPT_LOCAL_IMAGES_PATH="$(dirname "$CTR_MICROVM_IMAGES_DIR")" \
        --env PYTHONDONTWRITEBYTECODE=1 \
        "$DEVCTR_IMAGE"  "${ctr_args[@]}"
}
```

## cmd_build
默认debug版本, 默认libc是musl
target是`x86_64-unknown-linux-musl`

### 先build seccompiler
seccompiler是个单独的binary, 把json转成BPF程序保存到文件中.
```shell
    # Build seccompiler-bin.
    run_devctr \
        --user "$(id -u):$(id -g)" \
        --workdir "$CTR_FC_ROOT_DIR" \
        ${extra_args} \
        -- \
        cargo build -p seccompiler --bin seccompiler-bin \
            --target-dir "$CTR_CARGO_SECCOMPILER_TARGET_DIR" \
            "${cargo_args[@]}"
    ret=$?
```
注:
* `-p seccompiler`: 只build seccompiler

### 再build rebase-snap
> Tool that copies all the non-sparse sections from a diff file onto a base file

```shell
    # Build rebase-snap.
    run_devctr \
        --user "$(id -u):$(id -g)" \
        --workdir "$CTR_FC_ROOT_DIR" \
        ${extra_args} \
        -- \
        cargo build -p rebase-snap \
            --target-dir "$CTR_CARGO_REBASE_SNAP_TARGET_DIR" \
            "${cargo_args[@]}"
    ret=$?
```

### build firecracker
```shell
    # Build Firecracker.
    run_devctr \
        --user "$(id -u):$(id -g)" \
        --workdir "$CTR_FC_ROOT_DIR" \
        ${extra_args} \
        -- \
        cargo build \
            --target-dir "$CTR_CARGO_TARGET_DIR" \
            "${cargo_args[@]}"
    ret=$?
```

### build jailer
```shell
    # Build jailer only in case of musl for compatibility reasons.
    if [ "$libc" == "musl" ];then
        run_devctr \
            --user "$(id -u):$(id -g)" \
            --workdir "$CTR_FC_ROOT_DIR" \
            ${extra_args} \
            -- \
            cargo build -p jailer \
                --target-dir "$CTR_CARGO_TARGET_DIR" \
                "${cargo_args[@]}"
    fi
```

## build_kernel
比如:`./tools/devtool build_kernel -c resources/guest_configs/microvm-kernel-arm64-4.14.config`
```shell
    # arch不同, vmlinux的format也不同
    arch=$(uname -m)
    if [ "$arch" = "x86_64" ]; then
        target="vmlinux"
        cfg_pattern="x86"
        format="elf"
    elif [ "$arch" = "aarch64" ]; then
        target="Image"
        cfg_pattern="arm64"
        format="pe"
    
    recipe_url="https://raw.githubusercontent.com/rust-vmm/vmm-reference/$recipe_commit/resources/kernel/make_kernel.sh"
    # 从自己的github的另一个库rust-vmm/vmm-reference下载
    make_kernel.sh
    run_devctr \
        --user "$(id -u):$(id -g)" \
        --workdir "$kernel_dir_ctr" \
        -- /bin/bash -c "curl -LO "$recipe_url" && source make_kernel.sh && extract_kernel_srcs "$KERNEL_VERSION""
    cp "$KERNEL_CFG"  "$kernel_dir_host/linux-$KERNEL_VERSION/.config"
    KERNEL_BINARY_NAME="vmlinux-$KERNEL_VERSION-$arch.bin"
    
    #真正的make kernel
    run_devctr \
        --user "$(id -u):$(id -g)" \
        --workdir "$kernel_dir_ctr" \
        -- /bin/bash -c "source make_kernel.sh && make_kernel "$kernel_dir_ctr/linux-$KERNEL_VERSION" $format  $target "$nprocs" "$KERNEL_BINARY_NAME""
```

## build_rootfs
default rootfs size是300M, 用ubuntu18.04, 目标是$flavour.rootfs.ext4
先编译几个c文件, 用作测试?
```shell
        run_devctr \
        --workdir "$CTR_FC_ROOT_DIR" \
        -- /bin/bash -c "gcc -o  $rootfs_dir_ctr/init $resources_dir_ctr/init.c && \
        gcc -o  $rootfs_dir_ctr/fillmem $resources_dir_ctr/fillmem.c && \
        gcc -o  $rootfs_dir_ctr/readmem $resources_dir_ctr/readmem.c"
```

### firecracker/resources/tests/init.c
在调用`/sbin/openrc-init`之前, 向`/dev/mem`的特定地址(比如aarch64的0x40000000 1G)写入数字`123`
用于通知VMM kernel已经启动完毕
```c
// Base address values are defined in arch/src/lib.rs as arch::MMIO_MEM_START.
// Values are computed in arch/src/<arch>/mod.rs from the architecture layouts.
// Position on the bus is defined by MMIO_LEN increments, where MMIO_LEN is
// defined as 0x1000 in vmm/src/device_manager/mmio.rs.
#ifdef __x86_64__
#define MAGIC_MMIO_SIGNAL_GUEST_BOOT_COMPLETE 0xd0000000
#endif
#ifdef __aarch64__
#define MAGIC_MMIO_SIGNAL_GUEST_BOOT_COMPLETE 0x40000000
#endif

#define MAGIC_VALUE_SIGNAL_GUEST_BOOT_COMPLETE 123

int main () {
   int fd = open("/dev/mem", (O_RDWR | O_SYNC | O_CLOEXEC));
   int mapped_size = getpagesize();

   char *map_base = mmap(NULL,
        mapped_size,
        PROT_WRITE,
        MAP_SHARED,
        fd,
        MAGIC_MMIO_SIGNAL_GUEST_BOOT_COMPLETE);

   *map_base = MAGIC_VALUE_SIGNAL_GUEST_BOOT_COMPLETE;
   msync(map_base, mapped_size, MS_ASYNC);

   const char *init = "/sbin/openrc-init";

   char *const argv[] = { "/sbin/init", NULL };
   char *const envp[] = { };

   execve(init, argv, envp);
}
```

### firecracker/resources/tests/fillmem.c
Usage: ./fillmem mb_count  
先mmap再memset

### firecracker/resources/tests/readmem.c
Usage: ./readmem mb_count value

### 做镜像
用ubuntu18.04 container的
```shell
truncate -s "$SIZE" "$img_file"
mkfs.ext4 -F "$img_file"
docker run -v "$FC_ROOT_DIR:/firecracker" ubuntu:18.04 bash -s <<`EOF`
...
source $resource_dir/setup_rootfs.sh
mount rootfs.ext4 mnt
# 调用了setup_rootfs.sh的函数
# 安装udev systemd-sysv openssh-server iproute2
# 避免登陆
# 其他定制
prepare_fc_rootfs "$rootfs_dir" "$resource_dir"
dirs="bin etc home lib lib64 opt root sbin usr"
for d in $dirs; do tar c "/$d" | tar x -C $mnt_dir; done # 把bin etc等系统目录从container里面拷到rootfs.ext4里面
EOF
```
* 注: 使用`'EOF'`格式的heredoc, 其内部的变量不会展开
