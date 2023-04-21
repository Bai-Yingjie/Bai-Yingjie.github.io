- [不能在for range里删除slice元素](#不能在for-range里删除slice元素)
- [什么时候用buffer为1的channel](#什么时候用buffer为1的channel)
- [go按位取反(bitwise not)](#go按位取反bitwise-not)
- [go的相等性(==)](#go的相等性)
  - [普通类型的比较](#普通类型的比较)
  - [指针的相等性](#指针的相等性)
  - [channel的相等性](#channel的相等性)
  - [interface的相等性](#interface的相等性)
  - [结构体的相等性](#结构体的相等性)
  - [Array的相等性](#array的相等性)
  - [string](#string)
  - [\[\]byte用bytes.Equal比较](#byte用bytesequal比较)
  - [reflect.DeepEqual万能比较](#reflectdeepequal万能比较)
  - [cmp包](#cmp包)
- [通过unix socket发送fd](#通过unix-socket发送fd)
  - [发送](#发送)
  - [接收](#接收)
  - [发送2](#发送2)
  - [接收2](#接收2)
- [创建临时文件并mmap成结构体](#创建临时文件并mmap成结构体)
  - [memfd\_create()系统调用](#memfd_create系统调用)
  - [gvisor中的使用场景](#gvisor中的使用场景)
- [用正则表达式 遍历`/proc/self/maps`](#用正则表达式-遍历procselfmaps)
- [递归缩进打印error](#递归缩进打印error)
- [读go micro](#读go-micro)
  - [cmd](#cmd)
    - [cmd.APP()](#cmdapp)
    - [已经注册的cmd](#已经注册的cmd)
  - [cli相关的cmd](#cli相关的cmd)
    - [cli子命令](#cli子命令)
    - [client接口](#client接口)
- [pattern match](#pattern-match)
- [读fs\_linux.go](#读fs_linuxgo)
- [善用字符串库函数--strings.Join](#善用字符串库函数--stringsjoin)
- [切片的插入](#切片的插入)
- [匿名函数执行](#匿名函数执行)

# 不能在for range里删除slice元素
我问chatgpt如何在for range里删除slice元素, chatgpt给出的代码:
```golang
package main

import "fmt"

func main() {
	// Create a slice of integers
	nums := []int{1, 2, 3, 4, 5}

	// Iterate over the slice using range
	for i, num := range nums { // for range在一开始就定好了要interate多少次
		fmt.Println(i, num, len(nums)) // 而且不受index越界限制
		//fmt.Println(i, num, nums[i], len(nums)) // 我加的这行会panic
		// Check if the element should be deleted
		if num == 3 || num == 4 {
			// Delete the element from the slice using append
			nums = append(nums[:i], nums[i+1:]...)
		}
	}

	// Print the updated slice
	fmt.Println(nums)
}

//结果
0 1 5
1 2 5
2 3 5
3 5 4
4 5 4 //注意看这行, 实际上新的nums已经没有index 4了, for range里面的i和num还是能在这一轮取到值
[1 2 4 5]
```

如果告诉chatgpt答案是错的, 它会修正代码:
```
ME: the output is [1 2 4 5], which is not correct

CHATGPT: You are correct, I apologize for the mistake in my previous answer. When you delete elements from a slice while iterating over it with range, you need to adjust the index variable to account for the removed elements.
```
再给出的答案是对的:
```golang
package main

import "fmt"

func main() {
	// Create a slice of integers
	nums := []int{1, 2, 3, 4, 5}

	// Iterate over the slice using range
	for i := 0; i < len(nums); i++ {
		num := nums[i]
		// Check if the element should be deleted
		if num == 3 || num == 4 {
			// Delete the element from the slice
			nums = append(nums[:i], nums[i+1:]...)
			// Adjust the index variable to account for the removed element
			i--
		}
	}

	// Print the updated slice
	fmt.Println(nums)
}
```

# 什么时候用buffer为1的channel
比如下面的代码:
```go
package main

import (
	"errors"
	"fmt"
	"time"
)

func main() {
	errorChan := make(chan error, 1)
	fmt.Println("start")
	time.AfterFunc(time.Second*5, func() { errorChan <- nil; fmt.Println("time out") })

	go func() {
		time.Sleep(time.Second * 3) // 实际场景可以是cmd.Wait()
		errorChan <- errors.New("new error")
		fmt.Println("error happened")
	}()

	fmt.Println(<-errorChan)

	time.Sleep(time.Second * 10)
	fmt.Println("done")
}
```


# go按位取反(bitwise not)
go没有专用的取反操作符, 但用异或可以取反:
```go
func main() {
    var bitwisenot byte = 0x0F

    // printing the number in 8-Bit
    fmt.Printf("%08b\n", bitwisenot) // 00001111

    fmt.Printf("%08b\n", ^bitwisenot) // 11110000
    fmt.Printf("%08b\n", 1^bitwisenot) // 00001110 和上面结果不一样

    fmt.Printf("%08b\n", ^0x0F) // -0010000 默认数字都是int
    fmt.Printf("%08b\n", ^(int)(0x0F)) // -0010000
    fmt.Printf("%08b\n", ^(uint)(0x0F)) // 1111111111111111111111111111111111111111111111111111111111110000 不带符号位
}
```
结果:
```
00001111
11110000
00001110
-0010000
-0010000
1111111111111111111111111111111111111111111111111111111111110000
```

# go的相等性(==)
首先, map和slice不能用`==`比较
function也不能比较.
```go
f := func(int) int { return 1 }
g := func(int) int { return 2 }
f == g
//这样比较会编译错误
```
但function可以跟nil比较.

## 普通类型的比较
boo, int, float, complex的比较就是普通比较. 但需要注意的是float的NaN不等于NaN
```go
nan := math.NaN()
pos_inf := math.Inf(1)
neg_inf := math.Inf(-1)
fmt.Println(nan == nan)         // false
fmt.Println(pos_inf == pos_inf) // true
fmt.Println(neg_inf == neg_inf) // true
fmt.Println(pos_inf == neg_inf) // false
```

## 指针的相等性
要么两个指针都是nil, 要么两个指针指向同样的地址:
```go
var p1, p2 *string
name := "foo"
fmt.Println(p1 == p2) // true
p1 = &name
p2 = &name
fmt.Println(p1)       // 0x40c148
fmt.Println(p2)       // 0x40c148
fmt.Println(&p1)      // 0x40c138
fmt.Println(&p2)      // 0x40c140
fmt.Println(*p1)      // foo
fmt.Println(*p2)      // foo
fmt.Println(p1 == p2) // true
```
需要注意的是, 两个不同的empty struct(即空的struct实例)的地址可能相等
> A struct or array type has size zero if it contains no fields (or elements, respectively) that have a size greater than zero. Two distinct zero-size variables may have the same address in memory.

```go
type S struct{}
func main() {
    var p1, p2 *S
    s1 := S{}
    s2 := S{}
    p1 = &s1
    p2 = &s2
    fmt.Printf("%p\n", p1) // 0x1e52bc
    fmt.Printf("%p\n", p2) // 0x1e52bc
    fmt.Println(p1)        // &{}
    fmt.Println(p2)        // &{}
    fmt.Println(&p1)       // 0x40c138
    fmt.Println(&p2)       // 0x40c140
    fmt.Println(*p1)       // {}
    fmt.Println(*p2)       // {}
    fmt.Println(p1 == p2)  // true 本来s1和s2不是一个东西, 当都是空, 他们的地址相同, 所以相等.
}
```
如果结构体非空, `S struct {f int}`, `p1`和`p2`就不相等了.

## channel的相等性
满足下面条件之一
* 两个chnnel都是nil
* 两个都是从同一个make函数生成的

```go
func f(ch1 chan int, ch2 *chan int) {
    fmt.Println(ch1 == *ch2) // true
}
func main() {
    var ch1, ch2 chan int
    fmt.Println(ch1 == ch2) // true
    ch1 = make(chan int)
    ch2 = make(chan int)
    fmt.Println(ch1 == ch2) // false
    ch2 = ch1
    fmt.Printf("%p\n", &ch1) // 0x40c138
    fmt.Printf("%p\n", &ch2) // 0x40c140
    fmt.Println(ch1 == ch2)  // true
    f(ch1, &ch1)
}
```

## interface的相等性
* 两个interface都是nil(注意动态类型也要是nil)
```go
type I interface{ m() }
type T []byte
func (t T) m() {}
func main() {
    var t T
    fmt.Println(t == nil) // true
    var i I = t
    fmt.Println(i == nil)                   // false
    fmt.Println(reflect.TypeOf(i))          // main.T
    fmt.Println(reflect.ValueOf(i).IsNil()) // true
}
```
* 动态类型相同, 并且动态值相等
```go
type A int
type B = A
type C int
type I interface{ m() }
func (a A) m() {}
func (c C) m() {}
func main() {
    var a I = A(1)
    var b I = B(1)
    var c I = C(1)
    fmt.Println(a == b) // true 这里A和B是强别名(=号别名), 类型是一样的.
    fmt.Println(b == c) // false 类型不同不相等
    fmt.Println(a == c) // false 类型不同不相等
}
```

类型I的interface变量i可以和普通类型X的实例x比较, 只要
* 类型X实现了接口I
* 类型X可以比较

所以i和x比较, 如果i的动态类型是X, i的动态值又等于x, 那么i和x相等
```go
type I interface{ m() }
type X int
func (x X) m() {}
type Y int
func (y Y) m() {}
type Z int
func main() {
    var i I = X(1)
    fmt.Println(i == X(1)) // true
    fmt.Println(i == Y(1)) // false
    // fmt.Println(i == Z(1)) // mismatched types I and C
    // fmt.Println(i == 1) // mismatched types I and int
}
```
如果动态类型相等, 但这个类型不能比较, 则会产生panic:
```go
type A []byte
func main() {
    var i interface{} = A{}
    var j interface{} = A{}
    fmt.Println(i == j)
}

panic: runtime error: comparing uncomparable type main.A
```

如果动态类型不一样, 那就直接不等:
```go
type A []byte
type B []byte
func main() {
    // A{} == A{} // slice can only be compared to nil
    var i interface{} = A{}
    var j interface{} = B{}
    fmt.Println(i == j) // false
}
```

## 结构体的相等性
首先, 结构体可以直接用`==`操作符比较.
如果里面的非`_`域都相等, 则两个结构体相等. 注意, 结构体里面的大写, 小写域都要相等.
```go
type A struct {
    _ float64
    f1 int
    F2 string
}
type B struct {
    _ float64
    f1 int
    F2 string
}
func main() {
    fmt.Println(A{1.1, 2, "x"} == A{0.1, 2, "x"}) // true
    // fmt.Println(A{} == B{}) // mismatched types A and B 
}
```

当判断`x==y`时, 只有x可以赋值给y或者y可以赋值给x才能用`==`操做符.
所以下面的判断是不行的, 编译时就会报错.
```go
A{} == B{}
```

## Array的相等性
注意这里说的是Array, 不是slice.
Array里面的每个元素都相等的话, 两个array相等.
```go
type T struct {
    name string
    age int
    _ float64
}
func main() {
   x := [...]float64{1.1, 2, 3.14}
   fmt.Println(x == [...]float64{1.1, 2, 3.14}) // true
   y := [1]T{{"foo", 1, 0}}
   fmt.Println(y == [1]T{{"foo", 1, 1}}) // true
}
```

## string
string的比较按照`[]byte`按字节比较.
```go
fmt.Println(strings.ToUpper("ł") == "Ł")     // true
fmt.Println("foo" == "foo")                  // true
fmt.Println("foo" == "FOO")                  // false
fmt.Println("Michał" == "Michal")            // false
fmt.Println("żondło" == "żondło")            // true
fmt.Println("żondło" != "żondło")            // false
fmt.Println(strings.EqualFold("ąĆź", "ĄćŹ")) // true
```

## []byte用bytes.Equal比较
切片不能直接比较. 但`bytes.Equal`可以比较两个`[]byte`
```go
s1 := []byte{'f', 'o', 'o'}
s2 := []byte{'f', 'o', 'o'}
fmt.Println(bytes.Equal(s1, s2)) // true
s2 = []byte{'b', 'a', 'r'}
fmt.Println(bytes.Equal(s1, s2)) // false
s2 = []byte{'f', 'O', 'O'}
fmt.Println(bytes.EqualFold(s1, s2)) // true
s1 = []byte("źdźbło")
s2 = []byte("źdŹbŁO")
fmt.Println(bytes.EqualFold(s1, s2)) // true
s1 = []byte{}
s2 = nil
fmt.Println(bytes.Equal(s1, s2)) // true
```

## reflect.DeepEqual万能比较
`func DeepEqual(x, y interface{}) bool`可以比较任意两个值.
比如map
```go
m1 := map[string]int{"foo": 1, "bar": 2}
m2 := map[string]int{"foo": 1, "bar": 2}
// fmt.Println(m1 == m2) // map can only be compared to nil
fmt.Println(reflect.DeepEqual(m1, m2)) // true
m2 = map[string]int{"foo": 1, "bar": 3}
fmt.Println(reflect.DeepEqual(m1, m2)) // false
m3 := map[string]interface{}{"foo": [2]int{1,2}}
m4 := map[string]interface{}{"foo": [2]int{1,2}}
fmt.Println(reflect.DeepEqual(m3, m4)) // true
var m5 map[float64]string
fmt.Println(reflect.DeepEqual(m5, nil)) // false
fmt.Println(m5 == nil) // true
```
比如slice
```go
s := []string{"foo"}
fmt.Println(reflect.DeepEqual(s, []string{"foo"})) // true
fmt.Println(reflect.DeepEqual(s, []string{"bar"})) // false
s = nil
fmt.Println(reflect.DeepEqual(s, []string{})) // false
s = []string{}
fmt.Println(reflect.DeepEqual(s, []string{})) // true
```
比如结构体
```go
type T struct {
    name string
    Age  int
}
func main() {
    t := T{"foo", 10}
    fmt.Println(reflect.DeepEqual(t, T{"bar", 20})) // false
    fmt.Println(reflect.DeepEqual(t, T{"bar", 10})) // false
    fmt.Println(reflect.DeepEqual(t, T{"foo", 10})) // true
}
```

## cmp包
google提供了cmp包, 可以打印两个值的差异
```go
import (
    "fmt"
    "github.com/google/go-cmp/cmp"
)
type T struct {
    Name string
    Age  int
    City string
}
func main() {
    x := T{"Michał", 99, "London"}
    y := T{"Adam", 88, "London"}
    if diff := cmp.Diff(x, y); diff != "" {
        fmt.Println(diff)
    }
}
```
输出
```diff
  main.T{
-       Name: "Michał",
+       Name: "Adam",
-       Age:  99,
+       Age:  88,
        City: "London",
  }
```

# 通过unix socket发送fd
gvisor的`pkg/unet/unet.go`里面提供了listen, accept等方法
unet是给server端用的. 比如read和write方法, 先尝试用阻塞式的`unix.RawSyscall(unix.SYS_RECVMSG, ...)`, 不行再用对fd的poll`unix.Syscall6(unix.SYS_PPOLL, ...)`

还提供了通过unix socket发送/接收fd的方法:
```go
// PackFDs packs the given list of FDs in the control message.
//
// This must be used prior to WriteVec.
func (c *ControlMessage) PackFDs(fds ...int) {
    *c = ControlMessage(unix.UnixRights(fds...))
}

// ExtractFDs returns the list of FDs in the control message.
//
// Either this or CloseFDs should be used after EnableFDs.
func (c *ControlMessage) ExtractFDs() ([]int, error) {
    msgs, err := unix.ParseSocketControlMessage(*c)
    if err != nil {
        return nil, err
    }
    var fds []int
    for _, msg := range msgs {
        thisFds, err := unix.ParseUnixRights(&msg)
        if err != nil {
            // Different control message.
            return nil, err
        }
        for _, fd := range thisFds {
            if fd >= 0 {
                fds = append(fds, fd)
            }
        }
    }
    return fds, nil
}
```

被extract出来的fd可以用比如下面的函数来生成一个File对象.
```go
//比如调用
os.NewFile(uintptr(fd), "urpc")

// NewFile returns a new File with the given file descriptor and
// name. The returned value will be nil if fd is not a valid file
// descriptor. On Unix systems, if the file descriptor is in
// non-blocking mode, NewFile will attempt to return a pollable File
// (one for which the SetDeadline methods work).
//
// After passing it to NewFile, fd may become invalid under the same
// conditions described in the comments of the Fd method, and the same
// constraints apply.
func NewFile(fd uintptr, name string) *File
```

## 发送
```go
//@pkg/lisafs/sock.go
// writeTo writes the passed iovec to the UDS and donates any passed FDs.
func writeTo(sock *unet.Socket, iovec [][]byte, dataLen int, fds []int) error {
    w := sock.Writer(true)
    //这里的fds是可选的, 会做为control msg来发送
    if len(fds) > 0 {
        w.PackFDs(fds...)
    }
    for n := 0; n < dataLen; {
        cur, err := w.WriteVec(iovec)
        ...
    }
}
```

## 接收
```go
//@pkg/lisafs/sock.go
// readFrom fills the passed buffer with data from the socket. It also returns
// any donated FDs.
func readFrom(sock *unet.Socket, buf []byte, wantFDs uint8) ([]int, error) {
    r := sock.Reader(true)
    r.EnableFDs(int(wantFDs))
    n := len(buf)
    for got := 0; got < n; {
        cur, err := r.ReadVec([][]byte{buf[got:]})
        ...
    }
}
```

## 发送2
```go
//@pkg/urpc/urpc.go
// marshal sends the given FD and json struct.
func marshal(s *unet.Socket, v interface{}, fs []*os.File) error {
    // Marshal to a buffer.
    data, err := json.Marshal(v)
    if err != nil {
        log.Warningf("urpc: error marshalling %s: %s", fmt.Sprintf("%v", v), err.Error())
        return err
    }

    // Write to the socket.
    w := s.Writer(true)
    if fs != nil {
        var fds []int
        for _, f := range fs {
            fds = append(fds, int(f.Fd()))
        }
        w.PackFDs(fds...)
    }
    ...
}
```
## 接收2
```go
//@pkg/urpc/urpc.go
// unmarhsal receives an FD (optional) and unmarshals the given struct.
func unmarshal(s *unet.Socket, v interface{}) ([]*os.File, error) {
    // Receive a single byte.
    r := s.Reader(true)
    r.EnableFDs(maxFiles)
    firstByte := make([]byte, 1)

    // Extract any FDs that may be there.
    if _, err := r.ReadVec([][]byte{firstByte}); err != nil {
        return nil, err
    }
    fds, err := r.ExtractFDs()
    ...
}
```

# 创建临时文件并mmap成结构体
## memfd_create()系统调用
```c
       #include <sys/memfd.h>

       int memfd_create(const char *name, unsigned int flags);
```
用tmpfs创建一个临时文件, 这个文件和通常文件系统没关系, 但可以支持所有文件操做.
可以用这个文件来共享内存:
* 进程A调用memfd_create(), 返回fd
* 进程B去打开`/proc/<pid>/fd/<fd>`(pid是进程A的pid, fd是A调用memfd_create()返回的fd号.), 打开后可以mmap, 就看到进程A一样的内容了.

## gvisor中的使用场景
```go
//RTMemoryStats是要被mmap的结构体
// RTMemoryStatsSize is the size of the RTMemoryStats struct.
var RTMemoryStatsSize = unsafe.Sizeof(RTMemoryStats{})

// RTMemoryStatsPointer casts addr to a RTMemoryStats pointer.
func RTMemoryStatsPointer(addr uintptr) *RTMemoryStats {
    return (*RTMemoryStats)(unsafe.Pointer(addr))
}


const name = "memory-usage"
fd, err := memutil.CreateMemFD(name, 0)
    p, err := unix.BytePtrFromString(name)
    fd, _, e := unix.Syscall(unix.SYS_MEMFD_CREATE, uintptr(unsafe.Pointer(p)), uintptr(flags), 0)
file := os.NewFile(uintptr(fd), name)
//设置文件大小为结构体RTMemoryStatsSize的大小
file.Truncate(int64(RTMemoryStatsSize))

//mmap这个文件
mmap, err := memutil.MapFile(0, RTMemoryStatsSize, unix.PROT_READ|unix.PROT_WRITE, unix.MAP_SHARED, file.Fd(), 0)

//现在全局变量MemoryAccounting.RTMemoryStats就指向了这个文件.
//直接用那个结构体不香吗?
MemoryAccounting = &MemoryLocked{
    File:          file,
    RTMemoryStats: RTMemoryStatsPointer(mmap),
}
```

# 用正则表达式 遍历`/proc/self/maps`
比如一个程序想解析当前进程的进程空间:
```sh
$ cat /proc/self/maps
55cb9cb6b000-55cb9cb73000 r-xp 00000000 fc:02 396                        /bin/cat
55cb9cd72000-55cb9cd73000 r--p 00007000 fc:02 396                        /bin/cat
55cb9cd73000-55cb9cd74000 rw-p 00008000 fc:02 396                        /bin/cat
55cb9e7a2000-55cb9e7c3000 rw-p 00000000 00:00 0                          [heap]
7f3367281000-7f336754d000 r--p 00000000 fc:02 7228                       /usr/lib/locale/locale-archive
7f336754d000-7f3367734000 r-xp 00000000 fc:02 44721                      /lib/x86_64-linux-gnu/libc-2.27.so
7f3367734000-7f3367934000 ---p 001e7000 fc:02 44721                      /lib/x86_64-linux-gnu/libc-2.27.so
7f3367934000-7f3367938000 r--p 001e7000 fc:02 44721                      /lib/x86_64-linux-gnu/libc-2.27.so
7f3367938000-7f336793a000 rw-p 001eb000 fc:02 44721                      /lib/x86_64-linux-gnu/libc-2.27.so
7f336793a000-7f336793e000 rw-p 00000000 00:00 0
7f336793e000-7f3367967000 r-xp 00000000 fc:02 44717                      /lib/x86_64-linux-gnu/ld-2.27.so
7f3367b33000-7f3367b57000 rw-p 00000000 00:00 0
7f3367b67000-7f3367b68000 r--p 00029000 fc:02 44717                      /lib/x86_64-linux-gnu/ld-2.27.so
7f3367b68000-7f3367b69000 rw-p 0002a000 fc:02 44717                      /lib/x86_64-linux-gnu/ld-2.27.so
7f3367b69000-7f3367b6a000 rw-p 00000000 00:00 0
7fff4ad74000-7fff4ad95000 rw-p 00000000 00:00 0                          [stack]
7fff4adda000-7fff4addd000 r--p 00000000 00:00 0                          [vvar]
7fff4addd000-7fff4addf000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
```

可以用regex
```go
// mapsLine matches a single line from /proc/PID/maps.
var mapsLine = regexp.MustCompile("([0-9a-f]+)-([0-9a-f]+) ([r-][w-][x-][sp]) ([0-9a-f]+) [0-9a-f]{2,3}:[0-9a-f]{2,} [0-9]+\\s+(.*)")

func parseMaps() {
    f, err := os.Open("/proc/self/maps")
    r := bufio.NewReader(f)
    for {
        b, err := r.ReadBytes('\n')
        m := mapsLine.FindSubmatch(b)
        start, err := strconv.ParseUint(string(m[1]), 16, 64)
        end, err := strconv.ParseUint(string(m[2]), 16, 64)
        read := m[3][0] == 'r'
        write := m[3][1] == 'w'
        execute := m[3][2] == 'x'
        shared := m[3][3] == 's'
        offset, err := strconv.ParseUint(string(m[4]), 16, 64)
    }
}
```

# 递归缩进打印error
```go
type vmError struct {
    self     error
    children []error
}

func (vme vmError) Error() string {
    var b strings.Builder
    fmt.Fprintf(&b, "----\n")
    if vme.self != nil {
        fmt.Fprintf(&b, "%v\n", vme.self)
    }
    for _, err := range vme.children {
        for _, s := range strings.Split(err.Error(), "\n") {
            if s != "\tat -" {
                fmt.Fprintf(&b, "\t%s\n", s)
            }
        }
    }
    return b.String()
}
```

# 读go micro
## cmd
micro的入口命令, 只调用了一个函数
这里import中的的v2选择了v2的tag.
```go
package main

import (
    "github.com/micro/micro/v2/cmd"
)

func main() {
    cmd.Init()
}
```
这里的Init原型很特别:
`cmd/cmd.go`中, 
```go
// Init initialised the command line
func Init(options ...micro.Option) {
    Setup(cmd.App(), options...)

    cmd.Init(
        cmd.Name(name),
        cmd.Description(description),
        cmd.Version(buildVersion()),
    )
}
```
这里面的`Option`是`type Option func(*Options)`, 一个函数签名, 入参是`*Options`
`Options`是个巨大的结构体, 包含了所有的micro的概念.
```go
type Options struct {
    Auth      auth.Auth
    Broker    broker.Broker
    Cmd       cmd.Cmd
    Config    config.Config
    Client    client.Client
    Server    server.Server
    Store     store.Store
    Registry  registry.Registry
    Runtime   runtime.Runtime
    Transport transport.Transport
    Profile   profile.Profile

    // Before and After funcs
    BeforeStart []func() error
    BeforeStop  []func() error
    AfterStart  []func() error
    AfterStop   []func() error

    // Other options for implementations of the interface
    // can be stored in a context
    Context context.Context

    Signal bool
}
```
![](img/golang_杂记3_20220902085229.png)  

### cmd.APP()
```go
func App() *cli.App {
    //这里的DefaultCmd是个interface, 由newCmd实例化
    //即var DefaultCmd = newCmd()
    return DefaultCmd.App()
}

type cmd struct {
    opts Options
    app  *cli.App
}

//newCmd返回的是Cmd这个interface
func newCmd(opts ...Option) Cmd {
    //实际的cmd是上面的结构体
    cmd := new(cmd)
    cmd.opts = options
    cmd.app = cli.NewApp()
    //返回cmd实例, 但从外面看起来是Cmd interface
    return cmd
}

//综上, DefaultCmd.App()实际上就是
func (c *cmd) App() *cli.App {
    return c.app
}
//那么c.app实际上就是cli.NewApp()的实例化结果.
```
上面的`cli.NewApp()`在[`github.com/micro/cli/v2`](https://github.com/asim/cli), 它实际上是`urfave/cli`的fork.

### 已经注册的cmd
`cmd/cmd.go`
```go
    app.Commands = append(app.Commands, new.Commands()...)
    app.Commands = append(app.Commands, runtime.Commands(options...)...)
    app.Commands = append(app.Commands, store.Commands(options...)...)
    app.Commands = append(app.Commands, config.Commands(options...)...)
    app.Commands = append(app.Commands, api.Commands(options...)...)
    app.Commands = append(app.Commands, auth.Commands()...)
    app.Commands = append(app.Commands, bot.Commands()...)
    app.Commands = append(app.Commands, cli.Commands()...)
    app.Commands = append(app.Commands, broker.Commands(options...)...)
    app.Commands = append(app.Commands, health.Commands(options...)...)
    app.Commands = append(app.Commands, proxy.Commands(options...)...)
    app.Commands = append(app.Commands, router.Commands(options...)...)
    app.Commands = append(app.Commands, tunnel.Commands(options...)...)
    app.Commands = append(app.Commands, network.Commands(options...)...)
    app.Commands = append(app.Commands, registry.Commands(options...)...)
    app.Commands = append(app.Commands, debug.Commands(options...)...)
    app.Commands = append(app.Commands, server.Commands(options...)...)
    app.Commands = append(app.Commands, service.Commands(options...)...)
    app.Commands = append(app.Commands, build.Commands()...)
    app.Commands = append(app.Commands, web.Commands(options...)...)
```

## cli相关的cmd
cli除了注册了自己的cmd, 还注册了call stream publish等命令.
而且`cli.Commands()中`, cli命令本身和所有的子命令都是同级的. 只是cli被安排在了第一个
```go
func Commands() []*cli.Command {
    commands := []*cli.Command{
        {
            Name:   "cli",
            Usage:  "Run the interactive CLI",
            Action: Run,
        },
        {
            Name:   "call",
            Usage:  "Call a service e.g micro call greeter Say.Hello '{\"name\": \"John\"}",
            //注意这里的Print是个闭包函数, 返回callService的包装. 类似装饰器
            Action: Print(callService),
            Flags: []cli.Flag{
                &cli.StringFlag{
                    Name:    "address",
                    Usage:   "Set the address of the service instance to call",
                    EnvVars: []string{"MICRO_ADDRESS"},
                },
                &cli.StringFlag{
                    Name:    "output, o",
                    Usage:   "Set the output format; json (default), raw",
                    EnvVars: []string{"MICRO_OUTPUT"},
                },
                &cli.StringSliceFlag{
                    Name:    "metadata",
                    Usage:   "A list of key-value pairs to be forwarded as metadata",
                    EnvVars: []string{"MICRO_METADATA"},
                },
            },
        },
        ...
    }
}
```

cli命令对应的Action是Run.
这个Run是个典型的Read-Eval-Print-Loop, 基本上是
```go
for {
    //先readline()
    args, err := r.Readline()
    //准备参数
    args = strings.TrimSpace(args)
    parts := strings.Split(args, " ")
    //找到cmd
    cmd, ok := commands[name]
    //执行cmd
    rsp, err := cmd.exec(c, parts[1:])
    println(string(rsp))
}
```

micro的交互式cli很简单, 就是在顶层有个循环来readline, 执行cmd.
没有其他功能.

### cli子命令
这些子命令, 比如call, publish, 实际调用的是`internal/command/cli/command.go`中的
```go
func CallService(c *cli.Context, args []string) ([]byte, error) {
    参数和返回值都通过json格式编码
    d := json.NewDecoder(strings.NewReader(req))
    ctx := callContext(c)
    creq := (*cmd.DefaultOptions().Client).NewRequest(service, endpoint, request, client.WithContentType("application/json"))
    
    //实际调用的是client的call接口, 这是个同步调用, 发req, 等rsp
    err = (*cmd.DefaultOptions().Client).Call(ctx, creq, &rsp, opts...)
    //等待rsp并打印结果.
}
```

### client接口
client是micro的核心抽象, 接口定义如下:
```go
// Client is the interface used to make requests to services.
// It supports Request/Response via Transport and Publishing via the Broker.
// It also supports bidirectional streaming of requests.
type Client interface {
    Init(...Option) error
    Options() Options
    NewMessage(topic string, msg interface{}, opts ...MessageOption) Message
    NewRequest(service, endpoint string, req interface{}, reqOpts ...RequestOption) Request
    Call(ctx context.Context, req Request, rsp interface{}, opts ...CallOption) error
    Stream(ctx context.Context, req Request, opts ...CallOption) (Stream, error)
    Publish(ctx context.Context, msg Message, opts ...PublishOption) error
    String() string
}

```
client支持Req/Rsp(通过Transport抽象), 支持Pub/Sub(通过Broker)抽象, 还支持stream模式.

# pattern match
简单的`*`通配符匹配
```go
func patternMatchs(str, pattern string) bool {
    if len(pattern) == 0 {
        return false
    }
    strs := strings.Split(pattern, "*")
    //fmt.Println(strs, len(strs))
    var pos, index int
    if index = strings.Index(str, strs[0]); index != 0 {
        return false
    }
    end := strs[len(strs)-1]
    if index = strings.LastIndex(str, end); index+len(end) != len(str) {
        return false
    }
    for i, substr := range strs {
        if i == 0 || i == len(strs)-1 || len(substr) == 0 {
            continue
        }
        index = strings.Index(str[pos:], substr)
        if index == -1 {
            return false
        }
        pos += index + len(substr)
    }
    return true
}
```

# 读fs_linux.go
`periph.io/x/periph@v3.6.2+incompatible/host/sysfs/fs_linux.go`


# 善用字符串库函数--strings.Join
比如解析或`|`关系, 普通的思路是自己组类似`flag1|flag2|flag3`的字符串.
实际上, 下面的代码片段先把`flag1 flag2 flag3`分别append到`[]string`切片, 最后用`strings.Join(out, "|")`加或`|`操作符.
```go
func (e epollEvent) String() string {
    var out []string
    for _, b := range bitmaskString {
        if e&b.e != 0 {
            out = append(out, b.s)
            e &^= b.e
        }
    }
    if e != 0 {
        out = append(out, "0x"+strconv.FormatUint(uint64(e), 16))
    }
    if len(out) == 0 {
        out = []string{"0"}
    }
    return strings.Join(out, "|")
}
```

# 切片的插入
这个切片是个二叉树的顺序表的表达, 和sort包的思路一致.
先找到`r.Name`的位置`i`, 然后用`append`一个`nil`的方式把切片扩一个位置出来, 然后用内置的`copy`函数把`i`后面的元素往后都挪一个位置; 最后把位置`i`的元素填上.

为了把位置`i`后面的元素都挪一个位置, 代价是把这后面的所有元素都copy一遍
这里的copy实际上是overlap的, copy的时候, 源为`l[i:]`, 目的是`l[i+1:]`, 目的比源向后移动一个位置, 而且目的的元素个数也少了一个.
注意, 如果按照普通的for循环式的copy思路, src和dst重叠时不能正常工作的.
有人讨论说[golang的copy语义类似memmove](https://github.com/golang/go/issues/4142)
* memcpy: 当src和dst重叠时, 结果不确定
* memmove: src和dst可以重叠, 结果是正确的拷贝; 可以理解成有个临时缓冲做中转. 实际上并不需要中间buffer, [只需要在开始的时候判断是从前往后拷贝还是从后往前拷贝就行了.](https://vgable.com/blog/2008/05/24/memcopy-memmove-and-speed-over-safety/#:~:text=The%20difference%20between%20memcpy%20and,It%20might%20work.&text=memcpy%20can%20be%20faster%2C%20and%20usually%20is.)
* 结论: golang的copy支持overlap, 可以正确的拷贝

```go
func insertRef(l []*Ref, r *Ref) []*Ref {
    n := r.Name
    i := search(len(l), func(i int) bool { return l[i].Name > n })
    l = append(l, nil)
    copy(l[i+1:], l[i:])
    l[i] = r
    return l
}
```
下面是对应的快排
注意在search函数里, 并不需要传入要查找的切片; search调用f的时候, `f`会比较切片元素, f会访问切片.
在本例中, `f`能访问insertRef函数的`l []*Ref`变量, 这也是一种闭包形式?
```go
// search implements the same algorithm as sort.Search().
//
// It was extracted to to not depend on sort, which depends on reflect.
func search(n int, f func(int) bool) int {
    lo := 0
    for hi := n; lo < hi; {
        if i := int(uint(lo+hi) >> 1); !f(i) {
            lo = i + 1
        } else {
            hi = i
        }
    }
    return lo
}
```

# 匿名函数执行
下面的例子中, 匿名行数在定义后就执行, 就像普通的代码块执行一样: 变量`r`和`err`都能直接被匿名函数引用到.
那为什么要用`func(){}()`这种形式呢? 
首先, 匿名函数也是要函数栈的, 多了匿名函数性能上不会更好;
但是, 因为遍历全局的map `byName` `byNumber` 有锁操作, 那么使用匿名函数配合defer关键词来加锁和解锁, 代码更优雅.
如果不要这个匿名的壳子, 要么不用defer, 自己找地方加`unlock`; 还想用defer的话, defer会在`r.Open()`后执行. 
估计这里的逻辑是: 一定要在`r.Open()`前`unlock`
```go
func Open(name string) (i2c.BusCloser, error) {
    var r *Ref
    var err error
    func() {
        mu.Lock()
        defer mu.Unlock()
        if len(byName) == 0 {
            err = errors.New("i2creg: no bus found; did you forget to call Init()?")
            return
        }
        if len(name) == 0 {
            r = getDefault()
            return
        }
        // Try by name, by alias, by number.
        if r = byName[name]; r == nil {
            if r = byAlias[name]; r == nil {
                if i, err2 := strconv.Atoi(name); err2 == nil {
                    r = byNumber[i]
                }
            }
        }
    }()
    if err != nil {
        return nil, err
    }
    if r == nil {
        return nil, errors.New("i2creg: can't open unknown bus: " + strconv.Quote(name))
    }
    return r.Open()
}
```
