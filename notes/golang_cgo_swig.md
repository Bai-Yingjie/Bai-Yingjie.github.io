首先, swig是 - Simplified Wrapper and Interface Generator
> The  swig  command  is  used  to  create  wrapper  code to connect C and C++ code to scripting languages like Perl, Python, Tcl etc

[swig for go](http://www.swig.org/Doc4.0/Go.html)中说到, cgo生成的wrapper能调用c, 但不能直接调用c++.
而且, gcgo和gccgo在调用c的接口时, 是完全不一样的.
swig同时支持这两种toolchain, 并且保证类型安全.

从go1.1开始就支持swig了, 并且集成在go build命令中. 使用`go build -x -work`能看到详细使用swig的过程.

# 2010年的问答: Can I use shared objects with Go?

According to the Go [FAQ](http://golang.org/doc/go_faq.html#Do_Go_programs_link_with_Cpp_programs), you can call into C libraries using a "foreign function interface":

> Do Go programs link with C/C++ programs?
> 
> There are two Go compiler implementations, 6g and friends, generically called gc, and gccgo. Gc uses a different calling convention and linker and can therefore only be linked with C programs using the same convention. There is such a C compiler but no C++ compiler. Gccgo is a GCC front-end that can, with care, be linked with GCC-compiled C or C++ programs. However, because Go is garbage-collected it will be unwise to do so, at least naively.
> 
> There is a “foreign function interface” to allow safe calling of C-written libraries from Go code. We expect to use SWIG to extend this capability to C++ libraries. There is no safe way to call Go code from C or C++ yet.

根据说法, gccgo是可以不用cgo的接口, 直接调用c的. 但必须要十分清楚调用convention, 栈大小的限制等细节:
> You can also use cgo and SWIG with Gccgo and gollvm. Since they use a traditional API, it's also possible, with great care, to link code from these compilers directly with GCC/LLVM-compiled C or C++ programs. However, doing so safely requires an understanding of the calling conventions for all languages concerned, as well as concern for stack limits when calling C or C++ from Go.

见[gccgo安装使用文档](https://golang.org/doc/install/gccgo#C_Interoperability)

