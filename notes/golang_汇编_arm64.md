golang汇编语法参考: https://go.dev/doc/asm

- [pseudo寄存器](#pseudo寄存器)
- [函数](#函数)
- [汇编举例](#汇编举例)
	- [汇编访问go的结构体](#汇编访问go的结构体)
	- [MOV的方向是从左到右. 和linux cp命令一致](#mov的方向是从左到右-和linux-cp命令一致)
	- [访问runtime的g结构体和`timandy/routine`的实现](#访问runtime的g结构体和timandyroutine的实现)
		- [386和amd64](#386和amd64)
		- [arm](#arm)
		- [arm64](#arm64)
		- [mips/mips64](#mipsmips64)
		- [ppc64](#ppc64)
		- [loong64](#loong64)
		- [riscv](#riscv)
		- [总结](#总结)
	- [使用g的指针获取goroutine id](#使用g的指针获取goroutine-id)
		- [如何得到goid的偏移量](#如何得到goid的偏移量)
	- [使用`timandy/routine`的goroutine local](#使用timandyroutine的goroutine-local)
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

# 汇编举例
## 汇编访问go的结构体
比如
```go
type reader struct {
	buf [bufSize]byte
	r   int
}
```
在汇编里可以
* 用`reader__size`来表示`reader`结构体的size.
* 用`reader_buf` 和 `reader_r`分别表示结构体的两个域.

## MOV的方向是从左到右. 和linux cp命令一致
```c
MOVL	g(CX), AX     // Move g into AX.
MOVL	g_m(AX), BX   // Move g.m into BX.
```

## 访问runtime的g结构体和`timandy/routine`的实现
具体代码见`https://pkg.go.dev/cmd/internal/obj`

### 386和amd64
go标准库提供了`go_tls.h`, 其中定义了获取`g`的函数:
```go
#include "go_tls.h"
#include "go_asm.h"
...
get_tls(CX)
MOVL	g(CX), AX     // Move g into AX.
MOVL	g_m(AX), BX   // Move g.m into BX.
```
原理是使用一个不用的MMU寄存器来保存`g``, 把这个寄存器赋值给用户传入的寄存器`CX`, 这样`CX`就是`g`的指针.

`timandy/routine`的`386`实现如下
```go
#include "funcdata.h"
#include "go_asm.h"
#include "go_tls.h"
#include "textflag.h"

TEXT ·getgp(SB), NOSPLIT, $0-4
    get_tls(CX)
    MOVL    g(CX), AX
    MOVL    AX, ret+0(FP)
    RET
```
这里用到了`get_tls()`这个函数, 得到`g`的指针.

`amd64`的实现类似, 只是把`MOVL`换成`MOVQ`

### arm
arm使用`R10`保存`g`, 但在汇编里要直接使用`g`, 不要使用`R10`, 因为`R10`不识别.

```go
#include "funcdata.h"
#include "go_asm.h"
#include "textflag.h"

TEXT ·getgp(SB), NOSPLIT, $0-4
    MOVW    g, R8
    MOVW    R8, ret+0(FP)
    RET
```
这里的`MOVW g, R8`中的`g`, 就是`R10`

### arm64
arm64用`R28`保存`g`
```go
#include "funcdata.h"
#include "go_asm.h"
#include "textflag.h"

TEXT ·getgp(SB), NOSPLIT, $0-8
    MOVD    g, R8
    MOVD    R8, ret+0(FP)
    RET
```

### mips/mips64
mips使用`R30`保存`g`
```go
//go:build mips || mipsle
// +build mips mipsle

#include "funcdata.h"
#include "go_asm.h"
#include "textflag.h"

TEXT ·getgp(SB), NOSPLIT, $0-4
    MOVW    g, R8
    MOVW    R8, ret+0(FP)
    RET
```
64位版本:
```go
//go:build mips64 || mips64le
// +build mips64 mips64le

#include "funcdata.h"
#include "go_asm.h"
#include "textflag.h"

TEXT ·getgp(SB), NOSPLIT, $0-8
    MOVV    g, R8
    MOVV    R8, ret+0(FP)
    RET
```

### ppc64
ppc64使用`R30`保存`g`
```go
//go:build ppc64 || ppc64le
// +build ppc64 ppc64le

#include "funcdata.h"
#include "go_asm.h"
#include "textflag.h"

TEXT ·getgp(SB), NOSPLIT, $0-8
    MOVD    g, R8
    MOVD    R8, ret+0(FP)
    RET
```

### loong64
loong64使用`R22`保存`g`. 和MIPS不一样
```go
//go:build loong64
// +build loong64

#include "funcdata.h"
#include "go_asm.h"
#include "textflag.h"

TEXT ·getgp(SB), NOSPLIT, $0-8
    MOVV    g, R8
    MOVV    R8, ret+0(FP)
    RET
```

### riscv
riscv使用`X27`保存`g`
```go
#include "funcdata.h"
#include "go_asm.h"
#include "textflag.h"

TEXT ·getgp(SB), NOSPLIT, $0-8
    MOV    g, X10
    MOV    X10, ret+0(FP)
    RET
```

### 总结
几个RISC的实现都差不多, 都是用一个特殊的寄存器来保存`g`的指针, 在汇编里直接用`g`来表示这个特殊寄存器.

在汇编里实现`getgp`, 在`g.go`里面声明它:
```go
// getgp returns the pointer to the current runtime.g.
//
//go:nosplit
func getgp() unsafe.Pointer
```

## 使用g的指针获取goroutine id
`g`的结构体定义如下:
```go
type g struct {
	goid         int64
	paniconfault *bool
	gopc         *uintptr
	labels       *unsafe.Pointer
}
```
知道`g`的地址和`goid`的偏移量, 就能得到`goid`
```go
// getg returns current coroutine struct.
func getg() g {
	gp := getgp()
	if gp == nil {
		panic("Failed to get gp from runtime natively.")
	}
	return g{
		goid:         *(*int64)(add(gp, offsetGoid)),
		paniconfault: (*bool)(add(gp, offsetPaniconfault)),
		gopc:         (*uintptr)(add(gp, offsetGopc)),
		labels:       (*unsafe.Pointer)(add(gp, offsetLabels)),
	}
}

// Goid return the current goroutine's unique id.
func Goid() int64 {
	return getg().goid
}
```

### 如何得到goid的偏移量
`g`的"标准"定义在[runtime2.go](https://go.dev/src/runtime/runtime2.go), 它是个巨大的结构体:
```go
type g struct {
	// Stack parameters.
	// stack describes the actual stack memory: [stack.lo, stack.hi).
	// stackguard0 is the stack pointer compared in the Go stack growth prologue.
	// It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
	// stackguard1 is the stack pointer compared in the C stack growth prologue.
	// It is stack.lo+StackGuard on g0 and gsignal stacks.
	// It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
	stack       stack   // offset known to runtime/cgo
	stackguard0 uintptr // offset known to liblink
	stackguard1 uintptr // offset known to liblink

	_panic    *_panic // innermost panic - offset known to liblink
	_defer    *_defer // innermost defer
	m         *m      // current m; offset known to arm liblink
	sched     gobuf
	syscallsp uintptr // if status==Gsyscall, syscallsp = sched.sp to use during gc
	syscallpc uintptr // if status==Gsyscall, syscallpc = sched.pc to use during gc
	stktopsp  uintptr // expected sp at top of stack, to check in traceback
	// param is a generic pointer parameter field used to pass
	// values in particular contexts where other storage for the
	// parameter would be difficult to find. It is currently used
	// in three ways:
	// 1. When a channel operation wakes up a blocked goroutine, it sets param to
	//    point to the sudog of the completed blocking operation.
	// 2. By gcAssistAlloc1 to signal back to its caller that the goroutine completed
	//    the GC cycle. It is unsafe to do so in any other way, because the goroutine's
	//    stack may have moved in the meantime.
	// 3. By debugCallWrap to pass parameters to a new goroutine because allocating a
	//    closure in the runtime is forbidden.
	param        unsafe.Pointer
	atomicstatus atomic.Uint32
	stackLock    uint32 // sigprof/scang lock; TODO: fold in to atomicstatus
	goid         uint64
	schedlink    guintptr
	waitsince    int64      // approx time when the g become blocked
	waitreason   waitReason // if status==Gwaiting

	preempt       bool // preemption signal, duplicates stackguard0 = stackpreempt
	preemptStop   bool // transition to _Gpreempted on preemption; otherwise, just deschedule
	preemptShrink bool // shrink stack at synchronous safe point

	// asyncSafePoint is set if g is stopped at an asynchronous
	// safe point. This means there are frames on the stack
	// without precise pointer information.
	asyncSafePoint bool

	paniconfault bool // panic (instead of crash) on unexpected fault address
	gcscandone   bool // g has scanned stack; protected by _Gscan bit in status
	throwsplit   bool // must not split stack
	// activeStackChans indicates that there are unlocked channels
	// pointing into this goroutine's stack. If true, stack
	// copying needs to acquire channel locks to protect these
	// areas of the stack.
	activeStackChans bool
	// parkingOnChan indicates that the goroutine is about to
	// park on a chansend or chanrecv. Used to signal an unsafe point
	// for stack shrinking.
	parkingOnChan atomic.Bool

	raceignore    int8  // ignore race detection events
	tracking      bool  // whether we're tracking this G for sched latency statistics
	trackingSeq   uint8 // used to decide whether to track this G
	trackingStamp int64 // timestamp of when the G last started being tracked
	runnableTime  int64 // the amount of time spent runnable, cleared when running, only used when tracking
	lockedm       muintptr
	sig           uint32
	writebuf      []byte
	sigcode0      uintptr
	sigcode1      uintptr
	sigpc         uintptr
	parentGoid    uint64          // goid of goroutine that created this goroutine
	gopc          uintptr         // pc of go statement that created this goroutine
	ancestors     *[]ancestorInfo // ancestor information goroutine(s) that created this goroutine (only used if debug.tracebackancestors)
	startpc       uintptr         // pc of goroutine function
	racectx       uintptr
	waiting       *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order
	cgoCtxt       []uintptr      // cgo traceback context
	labels        unsafe.Pointer // profiler labels
	timer         *timer         // cached timer for time.Sleep
	selectDone    atomic.Uint32  // are we participating in a select and did someone win the race?

	// goroutineProfiled indicates the status of this goroutine's stack for the
	// current in-progress goroutine profile
	goroutineProfiled goroutineProfileStateHolder

	// Per-G tracer state.
	trace gTraceState

	// Per-G GC state

	// gcAssistBytes is this G's GC assist credit in terms of
	// bytes allocated. If this is positive, then the G has credit
	// to allocate gcAssistBytes bytes without assisting. If this
	// is negative, then the G must correct this by performing
	// scan work. We track this in bytes to make it fast to update
	// and check for debt in the malloc hot path. The assist ratio
	// determines how this corresponds to scan work debt.
	gcAssistBytes int64
}
```

虽然`goid`在这个结构体里, 但获取它的offset很有难度:
* `g`是`runtime`的私有结构体
* `goid`前面还有好些个fields, 不好直接"目测"出offset

`timandy/routine`解决了这个难题. 下面是原理.

1. 定义了"精简版"的`g`, 这里只关心`goid`, `gopc`等极少field
```go
type g struct {
	goid         int64
	paniconfault *bool
	gopc         *uintptr
	labels       *unsafe.Pointer
}
```
2. 在初始化的时候, 寻找`runtime.g`对应的类型, 利用反射得到`goid`这个域的offset
```go
var (
	offsetGoid         uintptr
	offsetPaniconfault uintptr
	offsetGopc         uintptr
	offsetLabels       uintptr
)

func init() {
	gt := getgt()
	offsetGoid = offset(gt, "goid")
	offsetPaniconfault = offset(gt, "paniconfault")
	offsetGopc = offset(gt, "gopc")
	offsetLabels = offset(gt, "labels")
}

// getgt returns the type of runtime.g.
//
//go:nosplit
func getgt() reflect.Type {
	return typeByString("runtime.g")
}

// offset returns the offset of the specified field.
func offset(t reflect.Type, f string) uintptr {
	field, found := t.FieldByName(f)
	if found {
		return field.Offset
	}
	panic(fmt.Sprintf("No such field '%v' of struct '%v.%v'.", f, t.PkgPath(), t.Name()))
}
```

`getgt()`使用了黑科技:
```go
// eface The empty interface struct.
type eface struct {
	_type unsafe.Pointer //"标准"的eface这里是 _type *_type
	data  unsafe.Pointer
}

// iface The interface struct.
type iface struct {
	tab  unsafe.Pointer //"标准"的iface这里是 tab *itab
	data unsafe.Pointer
}

// typelinks returns a slice of the sections in each module, and a slice of *rtype offsets in each module. The types in each module are sorted by string.
//
//go:linkname typelinks reflect.typelinks
func typelinks() (sections []unsafe.Pointer, offset [][]int32)

// resolveTypeOff resolves an *rtype offset from a base type.
//
//go:linkname resolveTypeOff reflect.resolveTypeOff
func resolveTypeOff(rtype unsafe.Pointer, off int32) unsafe.Pointer
```
解释:
* 用了`go:linkname`黑科技, 绕过golang的小写限制, 访问`reflect.typelinks()`函数, 它返回所有module的section信息.
* `typeByString()`函数遍历上面的`typelinks()`, 匹配每个类型的`string()`属性, 和`runtime.g`一致就返回其反射类型`reflect.Type`

`typeByString`代码如下:
```go
// typeByString returns the type whose 'String' property equals to the given string, or nil if not found.
func typeByString(str string) reflect.Type {
	// The s is search target
	s := str
	if len(str) == 0 || str[0] != '*' {
		s = "*" + s
	}
	// The typ is a struct iface{tab(ptr->reflect.Type), data(ptr->rtype)}
	typ := reflect.TypeOf(0)
	face := (*iface)(unsafe.Pointer(&typ))
	// Find the specified target through binary search algorithm
	sections, offset := typelinks()
	for offsI, offs := range offset {
		section := sections[offsI]
		// We are looking for the first index i where the string becomes >= s.
		// This is a copy of sort.Search, with f(h) replaced by (*typ[h].String() >= s).
		i, j := 0, len(offs)
		for i < j {
			h := i + (j-i)/2 // avoid overflow when computing h
			// i ≤ h < j
			face.data = resolveTypeOff(section, offs[h])
			if !(typ.String() >= s) {
				i = h + 1 // preserves f(i-1) == false
			} else {
				j = h // preserves f(j) == true
			}
		}
		// i == j, f(i-1) == false, and f(j) (= f(i)) == true  =>  answer is i.
		// Having found the first, linear scan forward to find the last.
		// We could do a second binary search, but the caller is going
		// to do a linear scan anyway.
		if i < len(offs) {
			face.data = resolveTypeOff(section, offs[i])
			if typ.Kind() == reflect.Ptr {
				if typ.String() == str {
					return typ
				}
				elem := typ.Elem()
				if elem.String() == str {
					return elem
				}
			}
		}
	}
	return nil
}
```

## 使用`timandy/routine`的goroutine local
`timandy/routine`给每个goroutine创建一个`thread`结构体:
```go
type threadLocalMap struct {
	table []any
}

type thread struct {
	labels                  map[string]string //pprof
	magic                   int64             //mark
	id                      int64             //goid
	threadLocals            *threadLocalMap
	inheritableThreadLocals *threadLocalMap
}
```
如果`g.label`是`nil`, 就创建`thread`结构体, 并将其指针保存到`g.label`  
注: `g.label`本来是给pprof用的.

`threadLocals`是个table: `table []any`, 用`index`来索引

`timandy/routine`提供的API `routine.NewThreadLocalWithInitial`返回一个`ThreadLocal`的interface变量,
一般用于初始化一个全局变量, 作用是这个全局变量在每个goroutine调用`ThreadLocal.Get()`的时候, 为新routine分配一个新的thread local的实例.

比如
```go
type uuidptr = *uuid.UUID

type infoPerRoutine struct {
	tracingID uuidptr
}

//一般用全局变量来做thread local
var routineLocal = routine.NewThreadLocalWithInitial(func() any {
	return &infoPerRoutine{}
})

func getRoutineLocal() *infoPerRoutine {
	return routineLocal.Get().(*infoPerRoutine)
}
```
在任意一个goroutine调用`getRoutineLocal()`会得到routine专属的`infoPerRoutine`实例.

# arm64汇编
https://pkg.go.dev/cmd/internal/obj/arm64#pkg-overview

## [Register mapping rules](https://pkg.go.dev/cmd/internal/obj/arm64#hdr-Register_mapping_rules)

1. All basic register names are written as Rn.
2. Go uses ZR as the zero register and RSP as the stack pointer.
3. Bn, Hn, Dn, Sn and Qn instructions are written as Fn in floating-point instructions and as Vn in SIMD instructions.
