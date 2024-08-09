- [类型断言很慢吗?](#类型断言很慢吗)
  - [结果](#结果)
- [zlib压缩](#zlib压缩)
- [gob编码](#gob编码)
  - [gob会缓存](#gob会缓存)
    - [结论](#结论)
  - [gob的编码规则](#gob的编码规则)
  - [decode时使用指针方式避免interface的值拷贝](#decode时使用指针方式避免interface的值拷贝)
    - [msg的值拷贝](#msg的值拷贝)
    - [优化成指针](#优化成指针)
  - [interface也可以序列化, 但需要Register](#interface也可以序列化-但需要register)
    - [输出](#输出)
    - [如果不Register会怎样?](#如果不register会怎样)
  - [顶层是interface{}的情况](#顶层是interface的情况)
    - [能直接encode interface{}](#能直接encode-interface)
    - [使用interface的地址来encode](#使用interface的地址来encode)
  - [Register()函数](#register函数)
    - [非要Register()吗?](#非要register吗)
- [没有特列, append也是值拷贝](#没有特列-append也是值拷贝)
- [要用interface抽象行为, 就不要多一层struct马甲.](#要用interface抽象行为-就不要多一层struct马甲)
- [interface{}变量可以直接和concrete类型的变量比较](#interface变量可以直接和concrete类型的变量比较)
- [Read不能保证全读](#read不能保证全读)
  - [用io.ReadFull](#用ioreadfull)
  - [没有io.WriteFull](#没有iowritefull)
- [string强转](#string强转)
- [先return再defer, defer里面能看到return的值](#先return再defer-defer里面能看到return的值)
- [包的初始化只执行一次](#包的初始化只执行一次)
- [goroutine与channel](#goroutine与channel)
  - [使用channel时一定要判断peer的goroutine是否还在1](#使用channel时一定要判断peer的goroutine是否还在1)
    - [问题场景](#问题场景)
    - [解决](#解决)
  - [使用channel时一定要判断peer的goroutine是否还在2](#使用channel时一定要判断peer的goroutine是否还在2)
    - [解决](#解决-1)
- [写空的channel不会panic](#写空的channel不会panic)
  - [简单的程序可以检测死锁](#简单的程序可以检测死锁)
  - [复杂的程序检测不出来, 直接卡住](#复杂的程序检测不出来-直接卡住)
- [patherror](#patherror)
- [永久阻塞](#永久阻塞)
- [selectgo源码杂记](#selectgo源码杂记)
  - [强转成切片指针](#强转成切片指针)
  - [突破数组大小限制](#突破数组大小限制)
  - [切片截取](#切片截取)
  - [子切片共享底层数组](#子切片共享底层数组)
- [go test](#go-test)
  - [测试对象方法](#测试对象方法)
  - [子项](#子项)
  - [性能测试](#性能测试)
- [不推荐用self或者this指代receiver](#不推荐用self或者this指代receiver)
  - [范式](#范式)
- [在链接阶段对全局变量赋值](#在链接阶段对全局变量赋值)
  - [使用场景](#使用场景)
  - [如何做到的?](#如何做到的)
    - [链接选项](#链接选项)
- [编译限制Build Constraints](#编译限制build-constraints)
  - [使用-tags参数指定用户自定义constraints](#使用-tags参数指定用户自定义constraints)
- [无表达式的switch](#无表达式的switch)
- [无缓冲和缓冲为1的通道不一样](#无缓冲和缓冲为1的通道不一样)
- [书](#书)
- [go内存模型](#go内存模型)
  - [解决1: 用channel](#解决1-用channel)
  - [解决2: 用sync](#解决2-用sync)
- [sync的once](#sync的once)

# 类型断言很慢吗?
答: 不慢, 甚至比直接函数调用还快... 黑科技
```go
package main

import (
    "testing"
)

type myint int64

type Inccer interface {
    inc()
}

func (i *myint) inc() {
    *i = *i + 1
}

func BenchmarkIntmethod(b *testing.B) {
    i := new(myint)
    incnIntmethod(i, b.N)
}

func BenchmarkInterface(b *testing.B) {
    i := new(myint)
    incnInterface(i, b.N)
}

func BenchmarkTypeSwitch(b *testing.B) {
    i := new(myint)
    incnSwitch(i, b.N)
}

func BenchmarkTypeAssertion(b *testing.B) {
    i := new(myint)
    incnAssertion(i, b.N)
}

func incnIntmethod(i *myint, n int) {
    for k := 0; k < n; k++ {
        i.inc()
    }
}

func incnInterface(any Inccer, n int) {
    for k := 0; k < n; k++ {
        any.inc()
    }
}

func incnSwitch(any Inccer, n int) {
    for k := 0; k < n; k++ {
        switch v := any.(type) {
        case *myint:
            v.inc()
        }
    }
}

func incnAssertion(any Inccer, n int) {
    for k := 0; k < n; k++ {
        if newint, ok := any.(*myint); ok {
            newint.inc()
        }
    }
}
```

## 结果
```shell
yingjieb@3a9f377eee5d /repo/yingjieb/godev/practice/src/benchmarks/typeassertion
$ go test -bench .
goos: linux
goarch: amd64
BenchmarkIntmethod-23           465990427                2.55 ns/op
BenchmarkInterface-23           269690563                4.46 ns/op
BenchmarkTypeSwitch-23          590743738                2.06 ns/op
BenchmarkTypeAssertion-23       577222344                2.10 ns/op
PASS
ok      _/repo/yingjieb/godev/practice/src/benchmarks/typeassertion     5.949s
```

* BenchmarkInterface: 通过interface直接调用最慢
* BenchmarkIntmethod: 直接函数调用做为基准
* BenchmarkTypeSwitch/BenchmarkTypeAssertion: 类型断言比直接函数调用还快!!!!!

# zlib压缩
zlib提供的压缩接口是io.Writer. 即`z := zlib.NewWriter(s)`是个io.Writer, 往里面写就是压缩写.
但要调用z.Close()接口做flush操作, close后数据才写入底层的io.Writer.
不想close的话, 调用Flush()接口也行.

# gob编码
json的编码体积偏大, 改用gob的性能和json差不多, 但体积能减小一半.
gob专用于go程序之间的数据编码方法, 借鉴并改进了了很多GPB的设计, 应该说是go世界的首选序列化反序列化方法.

## gob会缓存
比如发送方连续两次发送
```go
conn.enc.Encode(A)
conn.enc.Encode(B)
```
在接收方看起来, 连续两次Decode, 能还原A和B的值
```go
conn.enc.Decode(&A)
conn.enc.Decode(&B)
```
如果两次Encode间隔很短, 比如连续的2次Encode, 在对端Decode的时候, 第一把decode A的时候, 可能已经缓存了部分B的字节, 第二把decode可以用这个缓存.

但如果A的后面是对原始io.Reader的直接操做, 比如:
```go
conn.enc.Decode(&A)
io.Copy(os.Stdout, conn)
```
那么可能os.Stdout不会看到B, B缓存在第一把的Decode里面.
比如在发送方发送A和B之间加个sleep 1秒, 实验结果是对A的decode就不缓存.

### 结论
如果发送方encode间隔很短, gob会预取socket里面的紧跟着上次encode的内容, 那么:
* 接收方一直用gob去decode的话, 是没问题的.
* 但如果decode中途去直接操做io读, 是可能读不到数据的.

## gob的编码规则
* 以int为基础, size变长
* 第一次传输一个新的结构体的时候, 先传输这个结构体的定义, 即layout
* 后面传输的时候, 带结构体标识就可以了
* 即先描述这个东西长什么样子, 取个名字, 后面直接用名字指代.

## decode时使用指针方式避免interface的值拷贝
之前我的代码里定义了isMessageOut和messageIn两个接口
```go
// messageOut represents a message to be sent
type messageOut interface {
    isMessageOut()
}

// messageIn represents an incoming message, a handler is needed to process the message
type messageIn interface {
    // reply(if not nil) is the data that will be sent back to the connection,
    // the message may be marshaled or compressed.
    // remember in golang assignment to interface is also value copy,
    // so return reply as &someStruct whenever possible in your handler implementation.
    handle() (reply messageOut, err error)
}
```
我有个record类型的结构体要传输. 在接收端, 我定义了handle方法, 接收到messageIn类型的结构体就可以直接`handle()`了
```go
type record struct {
    Timestamp int64
    Payload   recordPayload
}

func (rcd record) handle() (reply messageOut, err error) {
    fmt.Println(rcd)
    return nil, nil
}

func fakeServer() {
    ...
    var msg messageIn
    for {
        decoder.Decode(&msg)
        reply, err := msg.handle()
    }
}
```
注意`decoder.Decode(&msg)`, 要求msg必须是messageIn, decoder会自动分配concrete类型实例并赋值给msg. 如果对端发过来的消息concrete类型不是messageIn, Decode会返回错误, 类似这样:
```shell
gob: local interface type *main.messageIn can only be decoded from remote interface type; received concrete type string
```
意思是对端发过来的是`string`类型, 我已经收好了; 但是你不是messageIn, 所以不符合用户要求.

### msg的值拷贝
上面的代码可以工作, 但有个性能问题. 注意到record的handle()接收record的值, 而不是指针.
所以第16行`reply, err := msg.handle()`时, 对msg发生了一次值拷贝. 
在go里面, 值拷贝是浅拷贝, 一般性能开销不大. 因为浅拷贝遇到切片, 字符串, map等等"引用"属性的对象, 浅拷贝只拷贝"指针", 不拷贝内容.
但这里为了进一步避免浅拷贝, 需要想办法把record的实现改成下面:注意只多了个*
```go
func (rcd *record) handle() (reply messageOut, err error) {
    fmt.Println(rcd)
    return nil, nil
}
```
编译没问题, 但运行时gob报错:
`gob: main.record is not assignable to type main.messageIn`
熟悉interface的同学应该知道这里的意思其实是: `decoder.Decode(&msg)` decode出来的"值", 不是messageIn, 不能赋值给msg.
那decode出来的"值"是什么呢?
这就要提到gob要求interface的具体类型要注册, 我是这样注册的:
```go
//在初始化路径上调用一次
gob.Register(record{})
```
那么decode出来的"值"就是record{}, 而record不是messageIn, `*record`才是

### 优化成指针
那么这样改就可以: 把`*record`注册给gob
```go
gob.Register(&record{})
```

## interface也可以序列化, 但需要Register
比如下面的代码中, 要marshal的是`record`结构体.
```go
type record struct {
    Timestamp int64 // time.Now().Unix()
    Payload   interface{}
}
```

但它的Payload部分是个interface, 可以是
* `string`
* `[]processInfo{}`
```go
type processInfo struct {
    Pid  int
    Ucpu string //%.2f
    Scpu string //%.2f
    Mem  uint64
    Name string
}
```

一个是内置的字符串, 一个是自定义的结构体数组.
对`record`类型来说, 这两个类型都是叫`Payload`
前面说过, 每个新东西都要描述一番, 取个名字. 但对interface来说, 它有很多面孔.
一个描述是不够的.
所以gob规定, interface所指代的具体类型, 要先注册.
下面的Register()调用就注册了这个结构体数组

```go
//把nil强转成目标类型的实例, 因为Register接收实例
//gob.Register([]processInfo{})也是可以的, 只要能得到实例
gob.Register([]processInfo(nil))

var tmp bytes.Buffer
enc := gob.NewEncoder(&tmp)
dec := gob.NewDecoder(&tmp)
//模拟processInfo切片
err := enc.Encode(record{time.Now().Unix(), []processInfo{}})
fmt.Println("encoded:", string(tmp.Bytes()))

//模拟一个hello字符串
err := enc.Encode(record{time.Now().Unix(), "hello"})
fmt.Println("encoded:", string(tmp.Bytes()))

err = dec.Decode(&data)
switch v := data.Payload.(type) {
case []processInfo:
        fmt.Println(data.Timestamp, "====processinfo ", v)
case string:
        fmt.Println(data.Timestamp, "====string ", v)
}
```

### 输出
这里用string强转了bytes, 有些不能打印字符.
```go
$ ./topid -p 2 -snapshot
//从这里可以看出来, 第一次描述了processInfo的layout
encoded: .record        TimestampPayload)[]main.processInfoD
                                                            processInfoPidUcpu
                                                                              Scpu
                                                                                  MemName
                                                                                         .000.0kthreadd
//能被decode还原
1590418131 ====processinfo  [{2 0.00 0.00 0 kthreadd}]

//Payload的string方式第一次出现 描述一下. 有些byte没打印, 但应该第一次的信息是全的.
encoded: string
               hello
1590418131 ====string  hello


//后面的打印就不带layout信息了
encoded: ;[]main.processInfo.000.0kthreadd
1590418132 ====processinfo  [{2 0.00 0.00 0 kthreadd}]
encoded: ;[]main.processInfo.000.0kthreadd
1590418133 ====processinfo  [{2 0.00 0.00 0 kthreadd}]

...
//不知为何, 原始string还是每次有带.
encoded: string
               hello
1590418161 ====string  hello
```

### 如果不Register会怎样?
不管encode还是decode, 都会打印提示:
```shell
gob: type not registered for interface: []main.processInfo
```

## 顶层是interface{}的情况
上面的例子中, bog可以编码结构体中间的field是interface{}的情况.
那么如果直接encode一个interface{}可以吗? 能decode吗?
先回答: 能; 能, 但需要技巧.

### 能直接encode interface{}
encode的入参就是interface{}类型, 即任何类型都可以被encode. 一个interface{}变量在被赋值的时候, runtime知道它的concrete类型.
参考 `神作: interface的运行时的lookup Go Data Structures: Interfaces`
gob会按照concreate类型传输.

所以这样的代码是可以被encode的.
```go
var msg interface{}
// msg = anyvariable
enc.Encode(msg)
```

但不能被decode.
```go
var msg interface{}
dec.Decode(&msg)
```
报错误:
```shell
gob: local interface type *interface {} can only be decoded from remote interface type; received concrete type sessionReq = struct { SessionTag string; SysInfo sysInfo = struct { BoardName string; CPUInfo string; KernelInfo string; PackageInfo packageInfo = struct { BuildVersion string; SwID string; BuildServer string; BuildDate string; Repo string; Branch string; }; }; }
```
这个错误说明两点:
* gob知道对方传输过来的结构体, 并且能精确解析
* 但gob不能把它decode给*interface {}, 即&msg; gob还提示, decode给interface必须对端也是interface类型.

### 使用interface的地址来encode
gob支持interface的传输, 比如在再上面的record类型中的interface就可以被传输和decode.
但顶层的interface需要些技巧.
在encode的时候, 传入interface的地址就行.
这样:
```go
// encode
var msg interface{}
// msg = anyvariable
enc.Encode(&msg)


// decode
var msg interface{}
dec.Decode(&msg)
```

很对称, 挺好的.

* gob会对指针解引用, encode时&msg被赋值给内部interface{}时, gob发现这是个指针类型, 指向interface类型. 所以gob按照interface类型来传输
* 根据interface的传输规则, encode端和decode端都要提前注册具体类型到gob

## Register()函数
使用了Type的String方法获得类型的名称
```go
func Register(value interface{}) {
    rt := reflect.TypeOf(value)
    name := rt.String()
    //处理指针的情况, 指针前面加*
    RegisterName(name, value)
```

gob包默认注册了基础类型
```go
func registerBasics() {
    Register(int(0))
    Register(int8(0))
    Register(int16(0))
    Register(int32(0))
    Register(int64(0))
    Register(uint(0))
    Register(uint8(0))
    Register(uint16(0))
    Register(uint32(0))
    Register(uint64(0))
    Register(float32(0))
    Register(float64(0))
    Register(complex64(0i))
    Register(complex128(0i))
    Register(uintptr(0))
    Register(false)
    Register("")
    Register([]byte(nil))
    Register([]int(nil))
    Register([]int8(nil))
    Register([]int16(nil))
    Register([]int32(nil))
    Register([]int64(nil))
    Register([]uint(nil))
    Register([]uint8(nil))
    Register([]uint16(nil))
    Register([]uint32(nil))
    Register([]uint64(nil))
    Register([]float32(nil))
    Register([]float64(nil))
    Register([]complex64(nil))
    Register([]complex128(nil))
    Register([]uintptr(nil))
    Register([]bool(nil))
    Register([]string(nil))
}
```

### 非要Register()吗?
既然有实例就能注册, 为什么不在encode/decode时自动注册了, 非要搞一个Register()?  
答: 可能是因为用了反射比较慢的缘故. 注册一次就够了, 每次都"注册"反射开销大.


# 没有特列, append也是值拷贝
那问题是, 做为入参传入append的时候, 是否已经发生了一次值拷贝, 然后再拷贝到[]slice里面去?

# 要用interface抽象行为, 就不要多一层struct马甲.
少用通用的interface然后再在里面搞类型断言;
而是用具体的interface, 这样在编译阶段就能"断言"类型. 比如下面的第45行.
```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    //"os"
    "bufio"
    "io"
)

type record interface {
    tag() string
    doPrint()
}

type teacher struct {
    Name string
}

type student struct {
    Id      int
    Name    string
    Class   int
    Email   string
    Message string
}

func (stdt *student) tag() string {
    return "student"
}

func (stdt *student) doPrint() {
    fmt.Println("do student", stdt)
}

func (tc *teacher) tag() string {
    return "teacher"
}

func (tc *teacher) doPrint() {
    fmt.Println("do teacher", tc)
}

func marshalRecord(w io.Writer, rcd record) error {
    jsn, err := json.Marshal(rcd)
    if err != nil {
        return err
    }

    w.Write([]byte(rcd.tag() + ": "))
    w.Write(jsn)
    w.Write([]byte{'\n'})

    return nil
}

func main() {
    var b bytes.Buffer

    stdt := student{Id: 9527, Name: "sam", Class: 3, Email: "sam@godev.com", Message: "hello\n world\n"}
    err := marshalRecord(&b, &stdt)
    if err != nil {
        fmt.Println(err)
    }

    tc := teacher{Name: "shuxue"}
    err = marshalRecord(&b, &tc)
    if err != nil {
        fmt.Println(err)
    }

    //b.WriteTo(os.Stdout)

    r := bufio.NewReader(&b)
    for {
        line, err := r.ReadBytes('\n')
        if err != nil {
            return
        }
        fmt.Printf("%s", line)

        sep := bytes.Index(line, []byte{':'})

        key := string(line[:sep])
        value := line[sep+2:]

        fmt.Printf("key: %s\n", key)
        fmt.Printf("value: %s\n", value)

        var rcd record
        switch key {
        case "student":
            rcd = &student{}
        case "teacher":
            rcd = &teacher{}
        }

        err = json.Unmarshal(value, rcd)
        if err != nil {
            fmt.Println(err)
            return
        }

        rcd.doPrint()
    }
}
```

# interface{}变量可以直接和concrete类型的变量比较
我实现了一个map, 提供set和get函数.
get出来的value是个万能interface{}, 
```go
func (im *intMap) set(k int, v interface{}) {
    _, has := im.mp[k]
    if !has {
        im.ks = append(im.ks, k)
    }

    im.mp[k] = v
}

func (im *intMap) get(k int) interface{} {
    v, has := im.mp[k]
    if has {
        return v
    }
    return nil
}

func main() {
    im := newIntMap(10)

    im.set(1, "1234")
    # v的类型是interface
    v := im.get(1)
    show(v)

    //interface变量可以直接和具体类型的值比较
    if v == "1234" {
        fmt.Println("interface{} can be compared directly with string")
    }

    im.set(2, &[]int{1, 2, 3})
    v = im.get(2)
    //这里可以比较, 但地址是不一样的
    if v == &[]int{1, 2, 3} {
        fmt.Println("should not be")
    }
    show(v)

    im.set(3, struct{ a, b, c int }{1, 2, 3})
    v = im.get(3)
    //结构体也可以比较, 但前提是结构体里面的元素都可以比较
    if v == struct{ a, b, c int }{1, 2, 3} {
        fmt.Println("interface{} can be compared directly with comparable struct")
    }
    show(v)

    im.set(4, 155)
    v = im.get(4)
    show(v)
    //可以直接和整型比较
    if v == 155 {
        fmt.Println("interface{} can be compared directly with int")
    }

    v = im.get(100)
    // 100不存在, v是nil
    // nil可以比较, 不会panic; 只是从来不一致.
    if v == 10086 {
        fmt.Println("nil interface{} can be compared, but never succeed")
    }
    show(v)

    //切片不能比较
    /*
        if v == []int{5,6,7} {
            fmt.Println("error! operator == not defined on slice")
        }
    */
}

//结果:
string:("1234")<16>
interface{} can be compared directly with string
*[]int:([]int{1, 2, 3})@[0xc0000ac040]<24>
interface{} can be compared directly with comparable struct
struct { a int; b int; c int }:(struct { a int; b int; c int }{a:1, b:2, c:3})<24>
int:(155)<8>
interface{} can be compared directly with int
<nil>:(<nil>)
```


# Read不能保证全读
golang的Reader不保证能read完整的len(buf), 即使没有到EOF, Read也不保证完整的Read.
所以Read会返回已经读的字节数n
```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

在C里面, 系统调用read()可能被信号打断而提前返回, 俗称短读.
一般的做法是自己写个包装, 用while一直读, 直到读完为止.

## 用io.ReadFull
在go里, io包已经提供了这个包装, 就是
```go
func ReadFull(r Reader, buf []byte) (n int, err error)
```
ReadFull保证填满buf, 除非EOF时buf还没填满, 此时返回ErrUnexpectedEOF
ReadFull实际上是调用ReadAtLeast

```go
func ReadAtLeast(r Reader, buf []byte, min int) (n int, err error) {
    if len(buf) < min {
        return 0, ErrShortBuffer
    }
    for n < min && err == nil {
        var nn int
        nn, err = r.Read(buf[n:])
        n += nn
    }
    if n >= min {
        err = nil
    } else if n > 0 && err == EOF {
        err = ErrUnexpectedEOF
    }
    return
}

```
ReadAtLeast()用一个循环反复Read()

## 没有io.WriteFull
标准库里面有ReadFull, 但没有WriteFull. 为什么呢?
有人还真实现了WriteFull, 有必要吗?

没必要: 因为read的语义允许short read而不返回error; 但write的语义是要写就都写完, 除非有错误.
> I would really rather not. ReadFull exists because Read is *allowed* to return less than
was asked without an error. Write is not. I don't want to encourage people to think that
buggy Writers are okay by introducing a buggy Writer fixer.


# string强转
结论: byte强转成string, 会去掉其中的0
```go
    buf := make([]byte, 32)
    buf = append(buf, '1', 0, '2')
    fmt.Println(buf)
    fmt.Println(string(buf))

//结果:
[0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 49 0 50]
12
```
说明byte切片转成string, 每个byte都被扫描, 去掉其中的0. 
也说明go里面的类型转换, 是有不小开销的.

# 先return再defer, defer里面能看到return的值
```go
func checkHierarchy(p int) (err error) {
    //defer里面可以直接使用returned变量
    defer func() {
        if err != nil {
            fmt.Println(err)
        } else {
            fmt.Println("no err returned")
        }
    }()
    
    //对返回变量直接赋值是可以的
    err = errors.New("default err")
    
    if p == 1 {
        return nil
    } else if p == 2 || p == 3{
        //相当于对err赋值. 
        return fmt.Errorf("return: %d", p)
    }
    
    //空的return会默认返回有名的return变量, 即上面的err
    return
}

func main() {
    checkHierarchy(1)
    checkHierarchy(2)
    checkHierarchy(3)
    checkHierarchy(4)
}

//输出
no err returned
return: 2
return: 3
default err
```

* defer里面可以直接使用return变量
* 空的return会默认返回有名的return变量
* 程序里可以提前对有名的return变量赋值, 然后用空return返回该值.
* 带值的return实际上会给有名的return变量赋值, defer的时候看到的是return的指定值.

defer在return之后执行 -- 先执行return那一行. 下面的代码会返回"Change World"而不是"Hello World"
```go
func foo() (result string) {
    defer func() {
        result = "Change World" // change value at the very last moment
    }()
    return "Hello World"
}
```

# 包的初始化只执行一次
> Package initialization is done only once even if package is imported many times.

# goroutine与channel
goroutine的生命周期和channel要配合, routine的产生与消亡要考虑对channel的影响
## 使用channel时一定要判断peer的goroutine是否还在1
这个例子中,`newPidInfo()`函数中, 起了goroutine `pi.checkThreads()`
```go
func newPidInfo(pid int, ppi *PidInfo) (*PidInfo, error) {
        pi.hierarchyDone = make(chan int)
        go func() {
            pi.err = pi.checkThreads()
        }()
        pi.triggerAndWaitHierarchy()
}

func (pi *PidInfo) checkThreads() error {
    defer func() { pi.hierarchyDone <- 1 }()
    
    ...
    doWork := func() error {
        ...
        pi.updateChildren() {
            ...
            for _, cpid := range pi.childrenIds {
                ...
                //这里, 触发下一层级的hierarchy更新
                childpi.triggerAndWaitHierarchy()
                ...
            }
            ...
        }
        ...
    }
    
    for {
        <-pi.hierarchyCheck
        if err := doWork(); err != nil {
            return fmt.Errorf("%s: %w", here(), err)
        }
        //notify this level check hierarchy done
        pi.hierarchyDone <- 1
    }
}

func (pi *PidInfo) triggerAndWaitHierarchy() {
    //trigger once, none blocking
    pi.hierarchyCheck <- 1
    //wait until check hierarchy done once
    <-pi.hierarchyDone
}
```
`pi.checkThreads()`虽然做了异常分支的channel处理, 比如`defer`那句.
但这里还是出了channel问题.


### 问题场景
1. 某次进程A要更新hierarchy, A发hierarchyCheck
2. A的checkThreads守护routine收到hierarchyCheck信号, 开始doWork()
3. A的doWork()里, 触发子进程B的hierarchyCheck
4. 子进程B的checkThreads守护routine开始doWork()
5. B的doWork()出错, B的hierarchyDone从异常分支写1, 通知A的doWork()往下走. 对应代码第20行
6. OK. 到这里还是正常的
7. 下次A要更新hierarchy, A发hierarchyCheck
8. A的checkThreads守护routine收到hierarchyCheck信号, 开始doWork()
9. A的doWork()里, 不知什么原因, A还是认为自己有子进程B. 触发子进程B的hierarchyCheck
10. 子进程B已经没有checkThreads守护routine
11. A永远阻塞在第42行

本质上, 出现问题的原因是: 子进程B的goroutine的异常分支已经完善, 但goroutine异常退出后, 业务逻辑不应该再去用channel和子进程B交互.
即永远判断要交互的goroutine是否存在.

### 解决
triggerAndWaitHierarchy()加异常判断
```go
func (pi *PidInfo) triggerAndWaitHierarchy() {
    //for some reason there is no checkThreads routine, thus nowhere we can send to and receive from
    //this can happen if the children file has the child, but actually the child is dying
    if pi.err != nil {
        //fmt.Println(pi.err)
        return
    }
    //trigger once, none blocking
    pi.hierarchyCheck <- 1
    //wait until check hierarchy done once
    <-pi.hierarchyDone
}
```


## 使用channel时一定要判断peer的goroutine是否还在2
比如下面的代码, 倒数第二行在写channel的时候, 对应的checkChild协程不一定能到达37行select.
实际上, 只有一个case的select可以只保留channel读部分.
```go
    for _, tid := range pi.threadIds {
        tid := tid
        if pi.childCheckers[tid] == nil {
            s := childChecker{make(chan int), make(chan []int, 1)}

            checkChild := func() {
                f, err := os.Open("/proc/" + pi.pidstr + "/task/" + strconv.Itoa(tid) + "/children")
                if err != nil {
                    return
                }
                defer f.Close()

                doWork := func() []int {
                    if _, err := f.Seek(0, 0); err != nil {
                        return nil
                    }
                    buf, err := ioutil.ReadAll(f)
                    if err != nil {
                        return nil
                    }
                    //ToDo: return nil if buf is empty

                    strs := strings.Fields(string(buf))

                    childrenIds := make([]int, len(strs))
                    for i := 0; i < len(strs); i++ {
                        childstr := strs[len(strs)-1-i]
                        pid, err := strconv.Atoi(childstr)
                        if err != nil {
                            return nil
                        }
                        childrenIds[i] = pid
                    }
                    return childrenIds
                }
                for {
                    select {
                    case <-s.triger:
                        s.children <- doWork()
                    }
                }
            }

            go checkChild()
            childCheckers[tid] = &s
        } else {
            //refill to the new map
            childCheckers[tid] = pi.childCheckers[tid]
        }

        //triger the childChecker routine
        //the assumption is using channel is cheaper than Open and Close. Is it true?
        childCheckers[tid].triger <- 1
    }
```
实际上, 这里还有一个错误. 在后面的处理里, 下面代码倒数第二行, 会阻塞的从`childCheckers[tid].children`读出数据, 但如果上面的goroutine异常退出了, 
```go
    //use new refilled map
    pi.childCheckers = childCheckers

    //with initial capacity of the previous run
    pi.childrenIds = make([]int, 0, len(pi.childrenIds))
    for _, tid := range pi.threadIds {
        //concatenate the children slices retrieved from channel to a single one
        pi.childrenIds = append(pi.childrenIds, <-childCheckers[tid].children...)
    }
```

### 解决
在goroutine刚开始, 加defer函数, 默认退出时读/写channel
这里用的非阻塞读写.
```go
            checkChild := func() {
                defer func() {
                    select {
                    case <-s.triger:
                    default:
                    }
                    select {
                    case s.children <- nil:
                    default:
                    }
                }()
            }
```


# 写空的channel不会panic
## 简单的程序可以检测死锁
```go
func main() {
    var c chan int

    c <- 1
}

//结果:
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send (nil chan)]:
main.main()
    /tmp/sandbox358600432/prog.go:15 +0x36
```

## 复杂的程序检测不出来, 直接卡住
```go
//pi.hierarchyDone是个空的channel
for {
        select {
        case <-pi.hierarchyCheck:
            if err := doWork(); err != nil {
                return fmt.Errorf("%s: %w", here(), err)
            }
            //写空的channel是能写的, 但永远没有人能读, 卡住
            pi.hierarchyDone <- 1
        }
    }
```

# patherror
```go
// requires go1.13 and later
func pathError(err error) bool {
    var perr *os.PathError
    if errors.As(err, &perr) {
        return true
    }

    return false
}
```

# 永久阻塞
空的select永远等待
```go
select {}
```

# selectgo源码杂记
selectgo()函数是select语句的runtime实现, 由编译器在编译时把select语句块转换为runtime的selectgo()的调用.

## 强转成切片指针
下面的代码是把一个指针, 强转成指向数组的指针;
go里面强转不是万能的, 指针必须转成指针
```go
    cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
    order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))
```
和下面的效果类似
```go
func main() {
    s := []int{1,2,3,4,5,6,7}
    
    sp := &s
    fmt.Println(sp)
    
    sp3 := (*[3]int)(unsafe.Pointer(&s[0]))
    fmt.Println(sp3)
}

//结果
&[1 2 3 4 5 6 7]
&[1 2 3]
```

## 突破数组大小限制
```go
func main() {
    s := [7]int{1,2,3,4,5,6,7}
    
    //数组的指针可以当数组用
    sp := &s
    //如果s是切片, 则下面语句报错:(type *[]int does not support indexing)
    sp[1] = 999
    fmt.Println(sp)
    
    //强转成指向数组的指针可以突破数组的length限制
    //但要注意, 你明确知道在干什么. 否则, segment fault
    snp := (*[20]int)(unsafe.Pointer(&s[0]))
    snp[15] = 995
    fmt.Println(snp)
}
```

## 切片截取
比如slice是个切片, 那么`slice[ i : j : k ]`是截取slice并限制capacity的切片:
```
Length:   j - i
Capacity: k - i
```
第三个参数的用法不常见, 但加了可以**限制**新切片的capacity的能力, 好处是防止越过capacity访问. 
下面的代码取自selectgo, 就使用了第二个冒号, 
```go
    scases := cas1[:ncases:ncases]
    pollorder := order1[:ncases:ncases]
    lockorder := order1[ncases:][:ncases:ncases]
```

## 子切片共享底层数组
子切片的capacity和其主切片的capacity有关, 因为他们都共享底层的数组.
比如下面的例子证明了, 子切片的修改会影响到母切片.
```go
func main() {
    s := []int{1,2,3,4,5,6,7}
    
    s1 := s[0:3]
    fmt.Println(s)
    fmt.Println(s1)
    
    //修改切片s1会影响原切片s
    s1[1] = 999
    fmt.Println(s)
    fmt.Println(s1)
    
    //非法访问, slice越界
    //panic: runtime error: index out of range [5] with length 3
    s1[5] = 888
}

结果:
[1 2 3 4 5 6 7]
[1 2 3]
[1 999 3 4 5 6 7]
[1 999 3]
```

# go test
## 测试对象方法
用Test对象名_方法名. 比如tengo代码中的
```go
func TestScript_Run(t *testing.T) {
    s := tengo.NewScript([]byte(`a := b`))
    err := s.Add("b", 5)
    require.NoError(t, err)
    c, err := s.Run()
    require.NoError(t, err)
    require.NotNil(t, c)
    compiledGet(t, c, "a", int64(5))
}
```

## 子项
Test 函数可以调用t.Run
```go
    func TestFoo(t *testing.T) {                
        // <setup code>                         
        t.Run("A=1", func(t *testing.T) { ... })
        t.Run("A=2", func(t *testing.T) { ... })
        t.Run("B=1", func(t *testing.T) { ... })
        // <tear-down code>
    }
```
go test可以指定子项:
```
go test -run Foo # Run top-level tests matching "Foo", such as "TestFooBar"
go test -run Foo/A= # For top-level tests matching "Foo", run subtests matching "A="
go test -run /A=1 # For all top-level tests, run subtests matching "A=1"
```

## 性能测试
用`go test -bench`
```go
func BenchmarkHello(b *testing.B) {
    big := NewBig()
    //开始循环测试之前先reset时间
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        //性能测试模式会测试这个循环里面的执行时间
        fmt.Sprintf("hello")
    }
}
```
用`b.RunParallel()`来并发执行, 和-cpu 1,2,4配合, 可以测一个核, 两个核, 四个核的并发性能
`go help testflag`查看详细的选项

# 不推荐用self或者this指代receiver
https://stackoverflow.com/questions/23482068/in-go-is-naming-the-receiver-variable-self-misleading-or-good-practice
receiver有两个作用,
第一, 声明了类型和方法的绑定关系.
第二, 在运行时给方法额外传了这个类型的参数. 对`v.Method()`的调用时编译器的语法糖,  本质上是`(T).Method(v)`. [参考这里](https://stackoverflow.com/questions/38025743/go-why-shouldnt-use-this-for-method-receiver-name)

C++和Python的方法基于具体对象, 所以this和self隐含代指这个具体对象的内存地址, 和方法是紧耦合关系.
而go的方法基于类型, 和具体对象是松耦合关系, 具体的对象只是做为额外的参数传给方法.

以下代码可以运行, 没有错误
```go
package main

import "fmt"

type T struct{}

func (t T) Method(msg string) {
    fmt.Println(msg)
}

func main() {
    t := T{}
    t.Method("hello") // this is valid
    //使用类型调用方法, 第一个参数是实例
    (T).Method(t, "world") // this too
}
```

## 范式
所以实例只是其中的一个参数, 要给个合适的名字, go惯例使用类型的第一个字母小写, 或者更贴切的名字.
```go
type MyStruct struct {
    Name string
} 

//这个更符合go的范式
func (m *MyStruct) MyMethod() error {
    // do something useful
}

//self的语义并不贴切
func (self *MyStruct) MyMethod() error {
    // do something useful
}
```

# 在链接阶段对全局变量赋值
## 使用场景
在vonu里面, 编译的时候传入了git commit id和编译时间; 在go build的时候用`-ldflags`传入
```
LDFLAGS=-ldflags '-X env.VOnuRevCommitId=$(VONUMGMT_REV_COMMIT_ID) -X "env.VOnuBuildDate=$(VONUMGMT_BUILD_DATE)"'
go build $(LDFLAGS) -tags static $(APP_MAIN)
```
这个`env.VOnuRevCommitId`是env包里的一个全局变量
```go
//定义的时候是nil
var VOnuRevCommitId string
//直接使用
func VOnuRevCommitIdInfo() string {
    return VOnuRevCommitId
}
```
## 如何做到的?
go build可以传入的选项有:
```
-a 强制全部重编
-work 不删除临时目录
-race 打开竞争检查
-buildmode 比如共享库方式的选择
-compiler 选gccgo或者gc
-gccgoflags
-gcflags
-ldflags arguments to pass on each go tool link invocation 这是本节的重点
-linkshared 链接共享库
-tags 自定义build constraints
-trimpath 不保存绝对路径, 这个功能很好!
```
### 链接选项
其中`-ldflags`里面说到go tool link, 那就要看它的help
```go
go tool link -h
//go tool link控制很底层的链接行为, 比如链接地址, 共享库路径
-T address 代码段起始地址
-X importpath.name=value 这就是本节用到的点, 定义package.name的变量未value, value是字符串
                                            链接的时候"初始化"这个变量
-cpuprofile 写profiling信息到文件
-dumpdep 
-linkmode
-buildmode

// 对减小size有好处
-s    disable symbol table
-w    disable DWARF generation
```
所以这里用了-X选项, 在链接的时候"初始化"变量值.

# 编译限制Build Constraints
详细说明: `go doc build`
编译限制用来指明一个文件是否要参与编译, 形式上要在文件开始的时候, "注释"编译限制, 比如:
```go
只在linux并且使用cgo, 或者OS x并且使用cgo情况下编译该文件
// +build linux,cgo darwin,cgo
只在ignore情况下编译, 即不参与编译. 因为没有东西会匹配ignore. 用其他怪异的tag也行, 但ignore的意思更贴切
// +build ignore
```
* 这个注释必须在package语句之前
* 内置的tag有:

```
During a particular build, the following words are satisfied:

    - the target operating system, as spelled by runtime.GOOS
    - the target architecture, as spelled by runtime.GOARCH
    - the compiler being used, either "gc" or "gccgo"
    - "cgo", if ctxt.CgoEnabled is true
    - "go1.1", from Go version 1.1 onward
    - "go1.2", from Go version 1.2 onward
    - "go1.3", from Go version 1.3 onward
    - "go1.4", from Go version 1.4 onward
    - "go1.5", from Go version 1.5 onward
    - "go1.6", from Go version 1.6 onward
    - "go1.7", from Go version 1.7 onward
    - "go1.8", from Go version 1.8 onward
    - "go1.9", from Go version 1.9 onward
    - "go1.10", from Go version 1.10 onward
    - "go1.11", from Go version 1.11 onward
    - "go1.12", from Go version 1.12 onward
    - "go1.13", from Go version 1.13 onward
    - "go1.14", from Go version 1.14 onward
    - any additional words listed in ctxt.BuildTags
```

另外, 如果文件名有如下形式, 则会被go build认为是隐含了对应tag的build constraint
```
*_GOOS
*_GOARCH
*_GOOS_GOARCH
```

## 使用-tags参数指定用户自定义constraints
比如在kafka.go最开始添加:
```
// +build !device
```
这里的device就是自定义的tag, 这里的意思是不带device的tag时, kafka.go才参与编译.
或者说有device的tag, kafka.go不编.
下面是测试结果: 带了device tag, 
```
$ RUNMODE=cloud go test -tags device
--- FAIL: TestMsgCall (0.00s)
    msgchan_test.go:20: No message channel for kafka
FAIL
exit status 1
FAIL msgchan 0.002s
```



# 无表达式的switch
switch 中的表达式是可选的，可以省略。如果省略表达式，则相当于 switch true，这种情况下会将每一个 case 的表达式的求值结果与 true 做比较，如果相等，则执行相应的代码。
```go
func main() {
    num := 75
    switch { // expression is omitted
    case num >= 0 && num <= 50:
        fmt.Println("num is greater than 0 and less than 50")
    case num >= 51 && num <= 100:
        fmt.Println("num is greater than 51 and less than 100")
    case num >= 101:
        fmt.Println("num is greater than 100")
    }
}
```
# 无缓冲和缓冲为1的通道不一样
无缓冲的通道，写会阻塞，直到有人读。
缓冲为1的通道，写第一个不会阻塞，而写第二个会。

# 书
https://www.cntofu.com/book/73/readme.html

# go内存模型
简单来说, 是和C语系一样: 可以编译时乱序, 执行时乱序(CPU特性)

那么, 下面的写法是不对的: 不能保证done在a的赋值之后执行.
```go
var a string
var done bool

func setup() {
    a = "hello, world"
    done = true
}

func main() {
    go setup()
    for !done {
    }
    print(a)
}
```

下面的例子更有隐蔽性: 即使在main看来, g不是nil了, 也不能保证g.msg有值.
```go
type T struct {
    msg string
}

var g *T

func setup() {
    t := new(T)
    t.msg = "hello, world"
    g = t
}

func main() {
    go setup()
    for g == nil {
    }
    print(g.msg)
}
```

## 解决1: 用channel
```go
var c = make(chan int, 10)
var a string

func f() {
    a = "hello, world"
    c <- 0
}

func main() {
    go f()
    <-c
    print(a)
}
```

## 解决2: 用sync
```go
var l sync.Mutex
var a string

func f() {
    a = "hello, world"
    l.Unlock()
}

func main() {
    l.Lock()
    go f()
    l.Lock()
    print(a)
}
```

# sync的once
once 保证之执行一次.
```go
var a string
var once sync.Once

func setup() {
    a = "hello, world"
}

func doprint() {
    once.Do(setup)
    print(a)
}

func twoprint() {
    go doprint()
    go doprint()
}
```
