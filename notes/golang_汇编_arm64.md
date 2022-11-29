golang汇编语法参考: https://go.dev/doc/asm

- [pseudo寄存器](#pseudo寄存器)
- [函数](#函数)
- [arm64汇编](#arm64汇编)
  - [Register mapping rules](#register-mapping-rules)

# pseudo寄存器
* SB: Static base pointer 全局基地址. 比如foo(SB)就是foo这个symbol的地址
* FP: 帧指针. 用来传参的, 比如
    * `first_arg+0(FP)`: 第一个参数
    * `second_arg+8(FP)`: 第二个参数(64bit CPU)
* SP: 栈指针. 指向栈顶. 用于局部变量. CPU都有物理SP, 语法上看前缀来区分:
    * `x-8(SP)`, `y-4(SP)`: 使用pseudo SP
    * `-8(SP)`使用物理SP
* PC: 程序指针

# 函数
格式: `TEXT symbol(SB), [flags,] $framesize[-argsize]`
* symbol: 函数名
* SB: SB伪寄存器
* flags: 可以是
    * NOSPLIT: 不让编译器插入栈分裂的代码
    * WRAPPER: 不增加函数帧计数
    * NEEDCTXT: 需要上下文参数, 一般用于闭包
* framesize: 局部变量大小, 包含要传给子函数的参数部分
* argsize: 参数+返回值的大小, 可以省略由编译器自己推导

比如
```go
//go:nosplit
func swap(a, b int) (int, int)
```
可以写为:
```go
TEXT ·swap(SB), NOSPLIT, $0-32
或者
TEXT ·swap(SB), NOSPLIT, $0
```
这里`-32`是4个8字节的int, 即入参a, b和两个出参.  
注意go并不区分入参和出参
```go
func swap(a, b int) (int, int)
或
func swap(a, b, c, d int)
或
func swap() (a, b, c, d int)
或
func swap() (a, []int, d int)
```
汇编都一样

# arm64汇编
https://pkg.go.dev/cmd/internal/obj/arm64#pkg-overview

## [Register mapping rules](https://pkg.go.dev/cmd/internal/obj/arm64#hdr-Register_mapping_rules)

1. All basic register names are written as Rn.
2. Go uses ZR as the zero register and RSP as the stack pointer.
3. Bn, Hn, Dn, Sn and Qn instructions are written as Fn in floating-point instructions and as Vn in SIMD instructions.
