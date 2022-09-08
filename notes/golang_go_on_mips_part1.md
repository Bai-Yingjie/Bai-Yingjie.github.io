- [Background](#background)
- [Go toolchains](#go-toolchains)
- [Compile gc go toolchain](#compile-gc-go-toolchain)
  - [Compile gc go 1.4 for host](#compile-gc-go-14-for-host)
  - [Compile gc go cross toolchain](#compile-gc-go-cross-toolchain)
- [Cgo issue](#cgo-issue)
  - [Cgo use gcc compiler](#cgo-use-gcc-compiler)
  - [direct cgo to use N32 ABI](#direct-cgo-to-use-n32-abi)
- [How to proceed if cgo is not working?](#how-to-proceed-if-cgo-is-not-working)
  - [disable cgo support](#disable-cgo-support)
  - [use N64 ABI for MIPS64](#use-n64-abi-for-mips64)
- [hello.go](#hellogo)
- [Works next](#works-next)

# Background
In order to run go app on the board, we need to have a go compiler that compiles go sources into go executables. 
Here we have a simple hello example: `hello.go`
```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

For go is a modern programing language, it is pretty straightforward to do this in the cloud/server x86 environment:
```sh
$ sudo apt install golang-go
$ go build hello.go 
$ ls
hello hello.go
$ ./hello 
Hello, World!
```
OK, job is done, you can stop reading if you work on a system that uses Ubuntu/CentOS on X86, or even ARM64.

But for people who work in embedded environment like buildroot system on MIPS, please continue.

# Go toolchains
There are two official Go compiler toolchains.
* gc go toolchain: seems to be more popular, this toolchain itself is written in Go
x86/ubuntu reference: `sudo apt install golang-go`
* gccgo compiler: the traditional compiler using the GCC back end
x86/ubuntu reference: `sudo apt install gccgo-go`

At the moment, for MIPS boards like fglt-b, gcc version is 4.7.0, lower than the minimal gcc prerequisite version 4.7.1, which has support for go 1.0. gcc 8 supports go 1.10. 
source: https://golang.org/doc/install/gccgo

# Compile gc go toolchain
In order to have go executables runnable on board, a gc go toolchain needs to be compiled first which is a cross toolchain resides on host for embedded world.
As mentioned, gc go is written in go, so a go bootstrap compiler is needed to compile gc go toolchain. Actually, the bootstrap compiler is also gc go compiler version 1.4, which was the last distribution in which the toolchain was written in C.

## Compile gc go 1.4 for host
In buildroot, package go-bootstrap is the one does the job. 
see
* `buildroot/package/go-bootstrap/go-bootstrap.mk`
* `buildroot/output/build/host-go-bootstrap-1.4.3/src/make.bash`

the compiled outputs are installed to:
`buildroot/output/host/lib/go-1.4.3`

## Compile gc go cross toolchain 
In buildroot, package go is the one does the job.
Here we need to build a cross compiler, `GO_GOARCH` is used for this purpose, eg: for fglt-b, `GO_GOARCH = mips64`
If all things go well, the toolchain will be installed to `buildroot/output/host/lib/go`.
The complete build steps:
```
Building Go cmd/dist using /repo/yingjieb/ms/buildroot/output/host/lib/go-1.4.3.
Building Go toolchain1 using /repo/yingjieb/ms/buildroot/output/host/lib/go-1.4.3.
Building Go bootstrap cmd/go (go_bootstrap) using Go toolchain1.
Building Go toolchain2 using go_bootstrap and Go toolchain1.
Building Go toolchain3 using go_bootstrap and Go toolchain2.
Building packages and commands for host, linux/amd64.
# errors are seen in this step
Building packages and commands for target, linux/mips64.
```

# Cgo issue
> cgo is a component of go, which enables the creation of go packages that call C code.

In buildroot, package go fails to compile, because go doesn't support CGO linking on MIPS64x platforms.
See: https://github.com/karalabe/xgo/issues/46
The consequence is any go app that depends on BR2_PACKAGE_HOST_GO_CGO_LINKING_SUPPORTS is unable to compile, even not selectable by buildroot menuconfig.
Nearly all go packages depend on BR2_PACKAGE_HOST_GO_CGO_LINKING_SUPPORTS, they are: docker-proxy, docker-containerd, docker-engine, docker-cli, flannel, mender, runc.
For MIPS octeon family specifically, which is used widely by LTs, the error message is:
```sh
# runtime/cgo
/repo/yingjieb/ms/buildroot/output/host/opt/ext-toolchain/bin/../lib/gcc/mips64-octeon-linux-gnu/4.7.0/../../../../mips64-octeon-linux-gnu/bin/ld: skipping incompatible /repo/yingjieb/ms/buildroot/output/host/mips64-buildroot-linux-gnu/sysroot/usr/lib/libpthread.so when searching for -lpthread
/repo/yingjieb/ms/buildroot/output/host/mips64-buildroot-linux-gnu/sysroot/usr/lib/libgcc_s.so: could not read symbols: File in wrong format
collect2: error: ld returned 1 exit status
go tool dist: FAILED: /repo/yingjieb/ms/buildroot/output/build/host-go-1.11.6/pkg/tool/linux_amd64/go_bootstrap install -gcflags=all= -ldflags=all= -v std cmd: exit status 2
```

## Cgo use gcc compiler
cgo enables go packages call C code, so it must require a C compiler for the C code part. 
And we are in embedded world, so here we need a cross C compiler. 
Gc go make.bash uses CC_FOR_TARGET to specify the C compiler, on my environment, for example:
`CC_FOR_TARGET="/repo/yingjieb/ms/buildroot/output/host/bin/mips64-octeon-linux-gnu-gcc"` 
In fact, this toolchain is a wrapper executable, which wraps external gcc toolchain to generate n32 ABI objects, like this:
`ELF 32-bit MSB executable, MIPS, N32 MIPS64 rel2 version 1, dynamically linked (uses shared libs), for GNU/Linux 2.6.32, with unknown capability 0x41000000 = 0xf676e75, with unknown capability 0x10000 = 0x70401, not stripped`
>  The 64 bit N64 ABI allows for more arguments in registers for more efficient function calls when there are more than four parameters. There is also the N32 ABI which also allows for more arguments in registers. The return address when a function is called is stored in the $ra register automatically by use of the JAL (jump and link) or JALR (jump and link register) instructions.

Basically, N32 uses 64 bit registers in a 32 bit address space, meaning that pointers are 32 bit.

However, for some unknown reason, the actual output objects are N64 ABI:
```sh
#cgo puts its generated C files to /tmp dir:
yingjieb@FNSHA190 /tmp/go-build164444988/b032
$ ls
_cgo_export.c _cgo_flags _cgo_main.c _x001.o _x003.o _x005.o _x007.o _x009.o cgo.cgo1.go
_cgo_export.h _cgo_gotypes.go _cgo_main.o _x002.o _x004.o _x006.o _x008.o _x010.o cgo.cgo2.c

# the default ABI is N64
yingjieb@FNSHA190 /tmp/go-build164444988/b032
$ file _cgo_main.o
_cgo_main.o: ELF 64-bit MSB relocatable, MIPS, MIPS64 rel2 version 1 (SYSV), with unknown capability 0x410000000f676e75 = 0x1000000070401, not stripped
``` 

The reason maybe that Gcc toolchain uses N64 ABI by default to generates objects.
One can explicitly set `-mabi=n32` to `CFLAGS`, that's how we config buildroot to use N32 ABI for our MIPS octeon family boards.

## direct cgo to use N32 ABI
So, let's get back to the cgo compiling issue, basically, ld complains about the format of the to be linked shared objects are incompatible. 
We can direct cgo to generate N32 ABI object.

* Add below to `src/runtime/cgo/cgo.go`
```
#cgo mips64 CFLAGS: -mabi=n32
#cgo mips64 LDFLAGS: -Wl,-m,elf32btsmipn32
```
* Do `export CGO_LDFLAGS_ALLOW=".*"` before building go package

Let's compile it again...
This time the previous errors are gone, but we have new errors now:
```sh
# runtime/cgo
In file included from _cgo_export.c:4:0:
cgo-gcc-export-header-prolog:25:14: error: size of array '_check_for_64_bit_pointer_matching_GoInt' is negative
```
`_cgo_export.c` is an generated c file, located in `/tmp/go-buildxxxxxxxxx` where xxxxxxxxx is a random number generated by the build system, so as `_cgo_export.h`, which has the assertion:
```c
typedef GoInt64 GoInt;
/*
  static assertion to make sure the file is being used on architecture
  at least with matching size of GoInt.
*/
typedef char _check_for_64_bit_pointer_matching_GoInt[sizeof(void*)==64/8 ? 1:-1];
```
`src/cmd/cgo/out.go` is used to genertate `_cgo_export.h`:
It seems that cgo expects a goInt to be 64 bit, but N32's pointer is 32 bit, that's why we have the error message.
Disable the check temporally:
```c
//typedef char _check_for_GOINTBITS_bit_pointer_matching_GoInt[sizeof(void*)==GOINTBITS/8 ? 1:-1];
#temporary workaroud this check
typedef char _check_for_GOINTBITS_bit_pointer_matching_GoInt[1];
```

Let's rebuild it...
cgo build seems to pass, but after it we have more errors:
```sh
# plugin
/repo/yingjieb/ms/buildroot/output/host/mips64-buildroot-linux-gnu/sysroot/usr/lib/libdl.so: could not read symbols:File in wrong format 
# os/signal/internal/pty
/repo/yingjieb/ms/buildroot/output/host/mips64-buildroot-linux-gnu/sysroot/usr/lib/libgcc_s.so: could not read symbols: File in wrong format
# os/user
/repo/yingjieb/ms/buildroot/output/host/mips64-buildroot-linux-gnu/sysroot/usr/lib/libgcc_s.so: could not read symbols: File in wrong format
# net
/repo/yingjieb/ms/buildroot/output/host/mips64-buildroot-linux-gnu/sysroot/usr/lib/libgcc_s.so: could not read symbols: File in wrong format
```
Same issue, apply our changes to these files:
* src/plugin/plugin_dlopen.go
* src/os/signal/internal/pty/pty.go
* src/os/user/cgo_lookup_unix.go
* src/os/user/getgrouplist_unix.go
* src/os/user/listgroups_unix.go
* src/os/user/getgrouplist_darwin.go
* src/net/cgo_linux.go

Rebuild, OK, then the error message leads to:
```sh
/tmp/go-link-436592583/go.o: In function `runtime.save_g':                                                           
/repo/yingjieb/ms/buildroot/output/host/lib/go/src/runtime/tls_mips64x.s:20:(.text+0x6d24c): relocation truncated to fit: R_MIPS_TLS_TPREL_LO16 against `runtime.tls_g'
```
It seems to have come to the final stage to build the executable "go"(the go compiler), but apparently there is something wrong in `tls_mips64x.s`.
The file is not long, but requires some longer study time to adapt for N32 mode if we still want to go this way...
```go
// If !iscgo, this is a no-op.
//
// NOTE: mcall() assumes this clobbers only R23 (REGTMP).
TEXT runtime_save_g(SB),NOSPLIT|NOFRAME,$0-0
    MOVB runtime_iscgo(SB), R23
    BEQ R23, nocgo

    MOVV R3, R23 // save R3
    MOVV g, runtime_tls_g(SB) // TLS relocation clobbers R3
    MOVV R23, R3 // restore R3

nocgo:
    RET

TEXT runtime_load_g(SB),NOSPLIT|NOFRAME,$0-0
    MOVV runtime_tls_g(SB), g // TLS relocation clobbers R3
    RET

GLOBL runtime_tls_g(SB), TLSBSS, $8
```
There are high possibilities that other .s files need to be adapted.
A rough estimation is: there are 505 .s files, with 446 line containing "cgo" key word together.
```sh
yingjieb@FNSHA190 /repo/yingjieb/ms/buildroot/output/build/host-go-1.11.6/src
$ find -name "*.s" | wc -l
505

yingjieb@FNSHA190 /repo/yingjieb/ms/buildroot/output/build/host-go-1.11.6/src
$ find -name "*.s" | xargs cat | grep cgo | wc -l
446
```
So, enabling N32 ABI which is used by all our MIPS boards for Go is not something can be done in a short time.

# How to proceed if cgo is not working?
As analyzed, the effort to resolve cgo problem is not trivial, right now what can be done are:

## disable cgo support
add `HOST_GO_CGO_ENABLED = 0` to `package/go/go.mk`, which means the gc go toolchian will not have cgo support, which further results the libraries that depend on cgo are not available. 
I am not an export of go, but I did a simple counter, there are 208 lines in the toolchain src use cgo.
```
yingjieb@FNSHA190 /repo/yingjieb/ms/buildroot/output/build/host-go-1.11.6/src
$ grep -r 'import "C"' | wc -l
208
```

## use N64 ABI for MIPS64
What if cgo is the part of the heart of go toolchain and furthermore, is indispensable for most go apps? The other option we can consider is to use N64 ABI.
> N64 uses 64 bit registers and 64 bit address space.

As said, all our MIPS boards use N32, I don't know exactly the reasons, but a common one would be: the old code are written for 32 bit processor, engineers at 32 bit era had the assumption that pointers are always 32 bit.

This option remains to be investigated if decision made to go this way.

# hello.go
Finally, we come to our task, thanks for the **complicities** of **embedded** world and **weak** ecosystem of MIPS.
Now that we have the gc go toolchain built, yes, with cgo disabled.
In Buildroot, we leverage the make framework to construct go packages, for example, I added below files:
```
M package/Nokia/Config.in
M package/go/Config.in.host
M package/go/go.mk
A package/Nokia/go-prototype/Config.in
A package/Nokia/go-prototype/go-prototype.mk
```
And I setup a repo named go-prototype, which has the hello.go src.
Here I want to skip the details of how to create a package in Buildroot, which is basically to make a package that depends on package go(the gc go toolchain)

OK, build our go-protogype package and all other packages selected in buildroot, make rootfs, and load to our board.
Be noted that our hello.go doesn't need cgo, since it does not `import "C"`
Then on our board, I am using cfnt-b, which has the same CPU with fglt-b:

Type
```sh
/isam/slot_default/run # hello
Hello, World!
First go program on cfnt-b!
/isam/slot_default/run # which hello
/usr/bin/hello
/isam/slot_default/run # ls -lh /usr/bin/hello
-rwxr-xr-x 1 root root 1.4M Jun 28 2019 /usr/bin/hello
```
OK, we are good to run hello.go

# Works next
* run Go simple function with library(with the hope that cgo is not used)
* Go benchmarking research and selection(with the hope that cgo is not used)
* Benchmark go(with the hope that cgo is not used)
* investigate how to enable N64 ABI for MIPS(optional)
