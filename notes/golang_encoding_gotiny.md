- [警告](#警告)
  - [gotiny会直接用传入的buf](#gotiny会直接用传入的buf)
    - [原因](#原因)
    - [解决](#解决)
- [问答](#问答)
- [gotiny_test](#gotiny_test)
  - [baseTyp](#basetyp)
  - [带方法的类型](#带方法的类型)
  - [interface变量](#interface变量)
  - [构造测试数据](#构造测试数据)
  - [测试流程](#测试流程)
  - [补充 interface和reflect.Value](#补充-interface和reflectvalue)
- [例子](#例子)
- [代码阅读之encode](#代码阅读之encode)
  - [Marshal](#marshal)
  - [Encoder](#encoder)
  - [Encode](#encode)
  - [总结](#总结)
- [encode核心函数](#encode核心函数)
  - [基础类型编码](#基础类型编码)
    - [bool编码](#bool编码)
    - [int编码](#int编码)
  - [其他](#其他)
  - [非基础类型编码](#非基础类型编码)
    - [每个类型都对应一个encEngine](#每个类型都对应一个encengine)
    - [获取Interface的实体类型名](#获取interface的实体类型名)
    - [从指向interface的指针到其数据(解引用)](#从指向interface的指针到其数据解引用)
      - [疑问](#疑问)
    - [补充: reflect.Value源码](#补充-reflectvalue源码)
    - [总结](#总结-1)
- [代码阅读之decode](#代码阅读之decode)
  - [Unmarshal](#unmarshal)
  - [解码engine](#解码engine)
    - [Decoder类型](#decoder类型)
    - [解码bool类型](#解码bool类型)
    - [解码string](#解码string)
  - [getDecEngine()](#getdecengine)
  - [总结](#总结-2)
- [encoder和decoder总结](#encoder和decoder总结)

# 警告
## gotiny会直接用传入的buf
遇到一个问题, 用在循环里用gotiny解码, 传入了相同的buf:
```go
dec := gotiny.NewDecoderWithPtr((*streamTransportMsg)(nil))
bufSize := make([]byte, 4)
bufMsg := make([]byte, 512)
for {
    //从网络读size和buf到bufSize和bufMsg
    var tm streamTransportMsg
    dec.Decode(bufMsg, &tm) //解码到tm, tm是个结构体, 里面包括msg域是[]byte
    // swap src and dst
    streamChan <- &streamTransportMsg{srcChan: tm.dstChan, dstChan: tm.srcChan, msg: tm.msg}
}
```

奇怪的现象是, for循环里大概收到了几次的报文, 经过解码得到:
```
3
8
hello
world
```
在第9行, send到channel之前打印tm.msg, 都是对的.
但另外一个goroutine读这个streamChan, 也能得到4次数据, 但是是这样的:
```
8
8
world
world
```

### 原因
gotiny对`[]byte`的编码直接用了用户传入的buf
```go
func decBytes(d *Decoder, p unsafe.Pointer) {
    bytes := (*[]byte)(p)
    if d.decIsNotNil() {
        l := int(d.decUint32())
        *bytes = d.buf[d.index : d.index+l] //这里的d.buf就是Decode传入的用户buf
        d.index += l
    } else if !isNil(p) {
        *bytes = nil
    }
}
```
这么来看, 因为`bufMsg := make([]byte, 512)`是共享的, 另外一个goroutine还没有来得及读channel, 这个bufMsg就会被下次的Decode调用改变掉, 导致tm.msg变化.

除了`decBytes`, ~~字符串和int8/uint8都直接用了buf~~.  
--2022.09.10更新, string, int8, uint8在赋值的时候发生了值拷贝. 特别的, 把一个byte array强转成string, 虽然string是个胖指针, 但我认为其底层的字符串内容也发生了每个字节的拷贝.
```go
func decString(d *Decoder, p unsafe.Pointer) {
    l, val := int(d.decUint32()), (*string)(p)
    *val = string(d.buf[d.index : d.index+l])
    d.index += l
}
func decInt8(d *Decoder, p unsafe.Pointer)      { *(*int8)(p) = int8(d.buf[d.index]); d.index++ }
func decUint8(d *Decoder, p unsafe.Pointer)     { *(*uint8)(p) = d.buf[d.index]; d.index++ }
```


### 解决
给gotiny增加copy mode, 使能之后在decBytes时就拷贝一份
```go
	bytes := (*[]byte)(p)
	if d.decIsNotNil() {
		l := int(d.decUint32())
		if d.copyMode {
			buf := make([]byte, l)
			copy(buf, d.buf[d.index:d.index+l])
			*bytes = buf
		} else {
			*bytes = d.buf[d.index : d.index+l]
		}
		d.index += l
	} else if !isNil(p) {
		*bytes = nil
```
详见这个PR: https://github.com/niubaoshu/gotiny/pull/6

# 问答
1. 为什么Marshal强制要求入参是指针?  
答: 估计是性能方面的考虑. 入参是普通对象也是可以的, 但参数也会值拷贝. 强制要求入参是指针, 就避免了用户传入"值"带来的深拷贝. 这个设计挺好的.
2. 为什么要用这样的结构?`reflect.ValueOf(&接口变量).Elem().InterfaceData()[1]`  
从功能上看, 这和`reflect.ValueOf(接口变量).InterfaceData()[1]`是不是一样的?  
猜测: 我认为从功能上看是一样的. 因为对一个接口变量取地址再ValueOf再Elem就还原了这个接口变量. 那么作者这么做的用意是避免拷贝吗? 取地址再ValueOf能避免interface直接赋值带来的拷贝, 但是再Elem不是还要拷贝吗? 比如`rv := reflect.ValueOf(a)`实际上是得到`a`的**副本**的值
那这里空转了一下的意义又是什么呢?

# gotiny_test
## baseTyp
* 首先定义了一个baseTyp, 包含了基本类型, array, interface, 以及匿名包含了另外一个结构体
没有slice, 没有map
```go
type baseTyp struct {
    ... //基本类型
    array       [3]uint32
    inter       interface{}
    A //匿名包含A, A是个结构体, 里面是name phone等有实际意义的field
}
```
* 还有无限循环的自定义类型, 很神
```go
type (
cirTyp    *cirTyp
cirStruct struct {
        a int
        *cirStruct
    }
    cirMap   map[int]cirMap
    cirSlice []cirSlice
)
var(
    vcir        cirTyp
    v2cir       cirTyp = &vcir
    v3cir       cirTyp = &v2cir
)
```

## 带方法的类型
* tint实现了io readWriter
```go
type tint int
func (tint) Read([]byte) (int, error)  { return 0, nil }
func (tint) Write([]byte) (int, error) { return 0, nil }
func (tint) Close() error              { return nil }
```

* gotiny自己的编解码接口, 外部类型可以实现这两个接口, 框架会调用  
注意Encode只返回`[]byte`, 而Decode返回使用了多少个字节的buf.
两个接口都不返回错误. 那么错误只能通过panic传递.
```go
// 只应该由指针来实现该接口
type GoTinySerializer interface {
    // 编码方法，将对象的序列化结果append到入参数并返回，方法不应该修改入参数值原有的值
    GotinyEncode([]byte) []byte
    // 解码方法，将入参解码到对象里并返回使用的长度。方法从入参的第0个字节开始使用，并且不应该修改入参中的任何数据
    GotinyDecode([]byte) int
}
// 下面这个gotinyTest实现了上面的接口, 演示了
type gotinyTest string
func (v *gotinyTest) GotinyEncode(buf []byte) []byte {
    return append(buf, gotiny.Marshal((*string)(v))...)
}
func (v *gotinyTest) GotinyDecode(buf []byte) int {
    return gotiny.Unmarshal(buf, (*string)(v))
}
```

## interface变量
```go
    vnilptr     *int
    v2nilptr    []string
    vnilptrptr  = &vnilptr
    
    v0interface interface{}
    vinterface  interface{}        = varray
    v1interface io.ReadWriteCloser = tint(2)
    v2interface io.ReadWriteCloser = os.Stdin
    v3interface interface{}        = &vinterface
    v4interface interface{}        = &v1interface
    v5interface interface{}        = &v2interface
    v6interface interface{}        = &v3interface
    v7interface interface{}        = &v0interface
    v8interface interface{}        = &vnilptr
    v9interface interface{}        = &v8interface
```

## 构造测试数据
这里的数据是个interface切片
```go
vs = []interface{}{
    里面是各种生成的, 预赋值的变量
}
    length = len(vs)
    buf    = make([]byte, 0, 1<<14) //16k
    e      = gotiny.NewEncoder(vs...) //decoder要传入数据源
    d      = gotiny.NewDecoder(vs...) //decoder要传入同类型的数据接受变量
    c      = goutils.NewComparer()
    
    //准备好材料, 数据源有interface, 反射值, 指针的表达, 都是同一个对象
    srci = make([]interface{}, length) //数据源切片
    reti = make([]interface{}, length) //decode后切片
    srcv = make([]reflect.Value, length) //数据源的reflect.Value表达
    retv = make([]reflect.Value, length) //返回的reflect.Value表达
    srcp = make([]unsafe.Pointer, length) //数据源的指针表达
    retp = make([]unsafe.Pointer, length) //返回值的指针表达
    typs = make([]reflect.Type, length) //数据源的类型
    
    
    e.AppendTo(buf) //为什么要从16K开始?
    for i := 0; i < length; i++ {
        typs[i] = reflect.TypeOf(vs[i]) //TypeOf得到反射类型
        srcv[i] = reflect.ValueOf(vs[i]) //ValueOf得到反射值

        tempi := reflect.New(typs[i]) //New新建一个对象, 返回指针
        tempi.Elem().Set(srcv[i]) //给这个指针指向的对象赋值
        srci[i] = tempi.Interface() //srci就是这个指针的interface表达

        tempv := reflect.New(typs[i]) //新建一个对象, 用来存结果
        retv[i] = tempv.Elem() //值
        reti[i] = tempv.Interface() //指针的interface表达

        //下面这两句有点神
        srcp[i] = unsafe.Pointer(reflect.ValueOf(&srci[i]).Elem().InterfaceData()[1])
        retp[i] = unsafe.Pointer(reflect.ValueOf(&reti[i]).Elem().InterfaceData()[1])
    }
```
要看懂InterfaceData()函数, 要看源码: 把v.ptr当作`[2]uintptr`的指针, 取值后返回. 注意最外层的取值`*`操做, 就是拷贝`*[2]uintptr`指向的`[2]uintptr`的意思, 也就是返回"只读"的`[2]uintptr`
```go
// InterfaceData returns the interface v's value as a uintptr pair.
// It panics if v's Kind is not Interface.
func (v Value) InterfaceData() [2]uintptr {
    // TODO: deprecate this
    v.mustBe(Interface)
    // We treat this as a read operation, so we allow
    // it even for unexported data, because the caller
    // has to import "unsafe" to turn it into something
    // that can be abused.
    // Interface value is always bigger than a word; assume flagIndir.
    return *(*[2]uintptr)(v.ptr)
}
```

## 测试流程
```go
func TestEncodeDecode(t *testing.T) {
    buf := gotiny.Marshal(srci...) //srci是interface切片, 底层是指针
    gotiny.Unmarshal(buf, reti...) //retiye是interface切片, 实际也是指针
    for i, r := range reti {
        Assert(t, buf, srci[i], r) //最后判断编解码是否一致
    }
}
```

## 补充 interface和reflect.Value
这里的Value就是reflect.Value
```go
type Value struct {
    // typ holds the type of the value represented by a Value.
    typ *rtype

    // Pointer-valued data or, if flagIndir is set, pointer to data.
    // Valid when either flagIndir is set or typ.pointers() is true.
    ptr unsafe.Pointer
    
    //带了一个flag, 表示这个值的metadata
    // flag holds metadata about the value.
    // The lowest bits are flag bits:
    //    - flagStickyRO: obtained via unexported not embedded field, so read-only
    //    - flagEmbedRO: obtained via unexported embedded field, so read-only
    //    - flagIndir: val holds a pointer to the data 注意这里, ptr可能是"值", 也可能是指针
    //    - flagAddr: v.CanAddr is true (implies flagIndir)
    //    - flagMethod: v is a method value.
    // The next five bits give the Kind of the value.
    // This repeats typ.Kind() except for method values.
    // The remaining 23+ bits give a method number for method values.
    // If flag.kind() != Func, code can assume that flagMethod is unset.
    // If ifaceIndir(typ), code can assume that flagIndir is set.
    flag
    
    // A method value represents a curried method invocation
    // like r.Read for some receiver r. The typ+val+flag bits describe
    // the receiver r, but the flag's Kind bits say Func (methods are
    // functions), and the top bits of the flag give the method number
    // in r's type's method table.
}
```
interface有空的interface和带方法的interface两种: 大小都是两个指针
代码在`/usr/local/go/src/reflect/value.go`
```go
// emptyInterface is the header for an interface{} value.
type emptyInterface struct {
    typ  *rtype
    word unsafe.Pointer
}

// nonEmptyInterface is the header for an interface value with methods.
type nonEmptyInterface struct {
    // see ../runtime/iface.go:/Itab
    itab *struct {
        ityp *rtype // static interface type
        typ  *rtype // dynamic concrete type
        hash uint32 // copy of typ.hash
        _    [4]byte
        fun  [100000]unsafe.Pointer // method table
    }
    word unsafe.Pointer
}
```

# 例子
```go
    src1, src2 := "hello", []byte(" world!")
    ret1, ret2 := "", []byte{3, 4, 5}
    gotiny.Unmarshal(gotiny.Marshal(&src1, &src2), &ret1, &ret2)
    fmt.Println(ret1 + string(ret2)) // print "hello world!"
    
    enc := gotiny.NewEncoder(src1, src2)
    dec := gotiny.NewDecoder(ret1, ret2)
```

* Marshal可传入多个interface, 但必须是指针形式
* Unmarshal也是是多个interface, 必须是指针
* NewEncoder和NewDecoder要传入所有编解码类型的实例

# 代码阅读之encode
## Marshal
```go
func Marshal(is ...interface{}) []byte {
    return NewEncoderWithPtr(is...).Encode(is...)
}
```
实际每个Marshal都是先NewEncoder
Encoder要预先生成
```go
func NewEncoderWithPtr(ps ...interface{}) *Encoder {
    l := len(ps)
    engines := make([]encEng, l)
    for i := 0; i < l; i++ {
        rt := reflect.TypeOf(ps[i])
        if rt.Kind() != reflect.Ptr {
            panic("must a pointer type!")
        }
        engines[i] = getEncEngine(rt.Elem())
    }
    return &Encoder{
        length:  l,
        engines: engines,
    }
}
```
NewEncoderWithPtr函数的核心是build一个engine切片engines, 按照入参顺序建立编码引擎.
`engines[i] = getEncEngine(rt.Elem())`

`NewEncoder`函数类似, 但入参不是指针
```go
func NewEncoder(is ...interface{}) *Encoder {
    l := len(is)
    engines := make([]encEng, l)
    for i := 0; i < l; i++ {
        engines[i] = getEncEngine(reflect.TypeOf(is[i]))
    }
    return &Encoder{
        length:  l,
        engines: engines,
    }
}
```
反正都要`rt := reflect.TypeOf(ps[i])`或`reflect.TypeOf(is[i])`, 那这两个函数可以合并的吧

## Encoder
```go
type Encoder struct {
    buf     []byte //编码目的数组
    off     int
    boolPos int  //下一次要设置的bool在buf中的下标,即buf[boolPos]
    boolBit byte //下一次要设置的bool的buf[boolPos]中的bit位

    engines []encEng
    length  int
}
```

## Encode
encode的作用是, 按照出参顺序, 依次encode到byte数组里
```go
func (e *Encoder) Encode(is ...interface{}) []byte {
    engines := e.engines
    for i := 0; i < len(engines) && i < len(is); i++ {
        engines[i](e, (*[2]unsafe.Pointer)(unsafe.Pointer(&is[i]))[1])
    }
    return e.reset()
}

func (e *Encoder) reset() []byte {
    buf := e.buf
    e.buf = buf[:e.off]
    e.boolBit = 0
    e.boolPos = 0
    return buf
}
```
engines是`encEngine`数组  
`encEngine`是个函数`type encEng func(*Encoder, unsafe.Pointer)`
这里关键是这句:  
`engines[i](e, (*[2]unsafe.Pointer)(unsafe.Pointer(&is[i]))[1])`
其中`(unsafe.Pointer(&is[i]))`是把输入的参数取地址后转成`unsafe.Pointer`  
`(*[2]unsafe.Pointer)强转后的值[1]`是把强转后的地址强转成指向`unsafe.Pointer`的**数组** `[2]unsafe.Pointer`.  
这里用到了数组go的数组表达: 切片的表达是个结构体, 包括了数组的指针和大小, 但数组还是和C一样的"原始"样子: **指向首元素的指针可以代表这个数组**.  
比如
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

那么强转成`(*[2]unsafe.Pointer)`是什么道理呢? 原来这里用到了interface的内部表达:
```go
type eface struct { // 16 bytes
    _type *_type
    data  unsafe.Pointer
}
type iface struct { // 16 bytes
    tab  *itab
    data unsafe.Pointer
}
```
注意到这里取的就是interface的data域, 因为不论是eface还是iface, data域都是"第二个"`unsafe.Pointer`  
所以, 这里我猜, `type encEng func(*Encoder, unsafe.Pointer)`就是要把这个`unsafe.Pointer`指向的数据, 保存在encoder的buf中.

## 总结
* 每个入参interface都有对应的一个encEngine, 这个engine在NewEncoder的时候build好.
* encEngine负责把interface的data指针编码到encoder中

# encode核心函数
上面说到, 入参interface的encEngine需要预先build好. 那NewEncoder里面, 调用的`getEncEngine`就是干这个的.
到这里已经是解引用了, 这个reflect.Type就是入参interface的实际类型.
```go
func getEncEngine(rt reflect.Type) encEng {
    encLock.RLock()
    engine := rt2encEng[rt]
    encLock.RUnlock()
    if engine != nil {
        return engine
    }
    encLock.Lock()
    buildEncEngine(rt, &engine)
    encLock.Unlock()
    return engine
}
```
先在全局表里按照`reflect.Type`来查找`encEngine`  
这里写的很统一, 实际上, 为什么不用比如reflect.TypeOf(true)呢? 非要先取指针再Elem()  
这些都是基础类型
```go
    rt2encEng = map[reflect.Type]encEng{
        reflect.TypeOf((*bool)(nil)).Elem():           encBool,
        reflect.TypeOf((*int)(nil)).Elem():            encInt,
        reflect.TypeOf((*int8)(nil)).Elem():           encInt8,
        reflect.TypeOf((*int16)(nil)).Elem():          encInt16,
        reflect.TypeOf((*int32)(nil)).Elem():          encInt32,
        reflect.TypeOf((*int64)(nil)).Elem():          encInt64,
        reflect.TypeOf((*uint)(nil)).Elem():           encUint,
        reflect.TypeOf((*uint8)(nil)).Elem():          encUint8,
        reflect.TypeOf((*uint16)(nil)).Elem():         encUint16,
        reflect.TypeOf((*uint32)(nil)).Elem():         encUint32,
        reflect.TypeOf((*uint64)(nil)).Elem():         encUint64,
        reflect.TypeOf((*uintptr)(nil)).Elem():        encUintptr,
        reflect.TypeOf((*unsafe.Pointer)(nil)).Elem(): encPointer,
        reflect.TypeOf((*float32)(nil)).Elem():        encFloat32,
        reflect.TypeOf((*float64)(nil)).Elem():        encFloat64,
        reflect.TypeOf((*complex64)(nil)).Elem():      encComplex64,
        reflect.TypeOf((*complex128)(nil)).Elem():     encComplex128,
        reflect.TypeOf((*[]byte)(nil)).Elem():         encBytes,
        reflect.TypeOf((*string)(nil)).Elem():         encString,
        reflect.TypeOf((*time.Time)(nil)).Elem():      encTime,
        reflect.TypeOf((*struct{})(nil)).Elem():       encIgnore,
        reflect.TypeOf(nil):                           encIgnore,
    }
```
基础的编码函数在`encbase.go`
特别的, 空的结构体和nil按encIgnore编码(也就是啥也不做)
```go
func encIgnore(*Encoder, unsafe.Pointer)      {}
```

## 基础类型编码
### bool编码
比如对bool类型的编码: 作者github上是这么描述的:
> bool类型占用一位，真值编码为1，假值编码为0。当第一次遇到bool类型时会申请一个字节，将值编入最低位，第二次遇到时编入次低位，第九次遇到bool值时再申请一个字节编入最低位，以此类推。

```go
func encBool(e *Encoder, p unsafe.Pointer)    { e.encBool(*(*bool)(p)) }

func (e *Encoder) encBool(v bool) {
    if e.boolBit == 0 {
        e.boolPos = len(e.buf)
        e.buf = append(e.buf, 0)
        e.boolBit = 1
    }
    if v {
        e.buf[e.boolPos] |= e.boolBit
    }
    e.boolBit <<= 1
}
```
解释:
* 这里的boolBit是个byte类型的变量, 每左移8次后变回0.
* 每次变回0的时候, 就新申请一个byte.

### int编码
*   uint8和int8 类型作为一个字节编入字符串的下一个字节。
*   uint16,uint32,uint64,uint,uintptr 采用[Varints](https://developers.google.com/protocol-buffers/docs/encoding#varints)编码方式。
*   int16,int32,int64,int 采用ZigZag转换成一个无符号数后采用[Varints](https://developers.google.com/protocol-buffers/docs/encoding#varints)编码方式。

从代码上看, int先转为int64, 再转为uint64, 但不是直接转, 而是用zigzag方式转.
zigzag的思想来自于我们常用的int都是小整数, 比如15, 2048等等的. 那么可以"压缩"来减小体积: 把符号位移到最后, 再做处理. 因为有符号数是原码取反加一(即补码), 对计算机来说和一个非常大的无符号数差不多, 所以要把符号位挪到最后.
详见[zigzag算法详细解释](https://blog.csdn.net/zgwangbo/article/details/51590186)

```go
func encInt(e *Encoder, p unsafe.Pointer)     { e.encUint64(int64ToUint64(int64(*(*int)(p)))) }

// int -5 -4 -3 -2 -1 0 1 2 3 4 5  6
// uint 9  7  5  3  1 0 2 4 6 8 10 12
func int64ToUint64(v int64) uint64 {
    return uint64((v << 1) ^ (v >> 63))
}

// uint 9  7  5  3  1 0 2 4 6 8 10 12
// int -5 -4 -3 -2 -1 0 1 2 3 4 5  6
func uint64ToInt64(u uint64) int64 {
    v := int64(u)
    return (-(v & 1)) ^ (v>>1)&0x7FFFFFFFFFFFFFFF
}

func (e *Encoder) encUint64(v uint64) {
    switch {
    case v < 1<<7-1:
        e.buf = append(e.buf, byte(v))
    case v < 1<<14-1:
        e.buf = append(e.buf, byte(v)|0x80, byte(v>>7))
    case v < 1<<21-1:
        e.buf = append(e.buf, byte(v)|0x80, byte(v>>7)|0x80, byte(v>>14))
    case v < 1<<28-1:
        e.buf = append(e.buf, byte(v)|0x80, byte(v>>7)|0x80, byte(v>>14)|0x80, byte(v>>21))
    case v < 1<<35-1:
        e.buf = append(e.buf, byte(v)|0x80, byte(v>>7)|0x80, byte(v>>14)|0x80, byte(v>>21)|0x80, byte(v>>28))
    case v < 1<<42-1:
        e.buf = append(e.buf, byte(v)|0x80, byte(v>>7)|0x80, byte(v>>14)|0x80, byte(v>>21)|0x80, byte(v>>28)|0x80, byte(v>>35))
    case v < 1<<49-1:
        e.buf = append(e.buf, byte(v)|0x80, byte(v>>7)|0x80, byte(v>>14)|0x80, byte(v>>21)|0x80, byte(v>>28)|0x80, byte(v>>35)|0x80, byte(v>>42))
    case v < 1<<56-1:
        e.buf = append(e.buf, byte(v)|0x80, byte(v>>7)|0x80, byte(v>>14)|0x80, byte(v>>21)|0x80, byte(v>>28)|0x80, byte(v>>35)|0x80, byte(v>>42)|0x80, byte(v>>49))
    default:
        e.buf = append(e.buf, byte(v)|0x80, byte(v>>7)|0x80, byte(v>>14)|0x80, byte(v>>21)|0x80, byte(v>>28)|0x80, byte(v>>35)|0x80, byte(v>>42)|0x80, byte(v>>49)|0x80, byte(v>>56))
    }
}
```
所以一般情况下, 比如一个几百的无符号数, zigzag后也不大, 那走上面的switch case估计要走到第二个.
其实如果不考虑大小, 直接存储应该是最方便快速的.

## 其他
* float32和float64采用[gob](https://golang.org/pkg/encoding/gob/)中对浮点类型的编码方式。
* complex64类型会强转为一个uint64后采用uint64的编码方式。
* complex128类型分别将虚实部分作为float64类型编码。
* 字符串类型先将字符串长度强转为uint64类型编码，然后将字符串字节数组自身原样编码。
* 指针类型判断是否为nil，如果是nil，编入一个bool类型的false值后结束，如果不为nil，编入一个bool类型true值，之后将指针解引用，按解引用后的类型编码。
* array和slice类型先将长度强转为一个uint64后采用uint64的编码方式编入，然后将每一个元素安装自身的类型编码。
* map同上，先编入长度，然后编入一个健，后面跟健对应的值，在编入一个健，接着是值，以此类推。
* struct类型将结构体的所有成员按其类型编码，无论是否导出，非导出的字段也会编码。结构体会严格还原。
* 对于实现encoding包BinaryMarshaler/BinaryUnmarshaler 或 实现 gob包GobEncoder/GobDecoder 接口的类型会用实现的方法编码。
* 对于实现了gotiny.GoTinySerialize包的类型将采用实现的方法编码和解码
* channel和function不能编码

## 非基础类型编码
是这个函数负责的:`buildEncEngine()`
基本上能想象, 这是个不断递归的过程.
这个函数写的神了!
```go
func buildEncEngine(rt reflect.Type, engPtr *encEng) {
    ... //如果实现了其他编解码的接口, 先调用那些接口
    
    kind := rt.Kind()
    var eEng encEng
    switch kind {
    case reflect.Ptr: //解引用然后调用其encEngine
        defer buildEncEngine(rt.Elem(), &eEng)
        engine = func(e *Encoder, p unsafe.Pointer) {
            isNotNil := !isNil(p)
            e.encIsNotNil(isNotNil)
            if isNotNil {
                eEng(e, *(*unsafe.Pointer)(p))
            }
        }
    case reflect.Array:
        et, l := rt.Elem(), rt.Len()
        defer buildEncEngine(et, &eEng)
        size := et.Size()
        engine = func(e *Encoder, p unsafe.Pointer) {
            for i := 0; i < l; i++ {
                eEng(e, unsafe.Pointer(uintptr(p)+uintptr(i)*size)) //按元素偏移size
            }
        }
    case reflect.Slice: //slice和array类似, 但lenth是从SliceHeader得到的.
        et := rt.Elem()
        size := et.Size()
        defer buildEncEngine(et, &eEng)
        engine = func(e *Encoder, p unsafe.Pointer) {
            isNotNil := !isNil(p)
            e.encIsNotNil(isNotNil)
            if isNotNil {
                header := (*reflect.SliceHeader)(p)
                l := header.Len
                e.encLength(l)
                for i := 0; i < l; i++ {
                    eEng(e, unsafe.Pointer(header.Data+uintptr(i)*size))
                }
            }
        }
    case reflect.Map: //注意map要build两类的engine, 一个for key, 一个for value
        var kEng encEng
        defer buildEncEngine(rt.Key(), &kEng)
        defer buildEncEngine(rt.Elem(), &eEng)
        engine = func(e *Encoder, p unsafe.Pointer) {
            isNotNil := !isNil(p)
            e.encIsNotNil(isNotNil)
            if isNotNil {
                v := reflect.NewAt(rt, p).Elem()
                e.encLength(v.Len())
                keys := v.MapKeys()
                for i := 0; i < len(keys); i++ {
                    val := v.MapIndex(keys[i])
                    kEng(e, getUnsafePointer(&keys[i]))
                    eEng(e, getUnsafePointer(&val))
                }
            }
        }
    case reflect.Struct: //每个元素都要build engine
        fields, offs := getFieldType(rt, 0)
        nf := len(fields)
        fEngines := make([]encEng, nf)
        defer func() {
            for i := 0; i < nf; i++ {
                buildEncEngine(fields[i], &fEngines[i])
            }
        }()
        engine = func(e *Encoder, p unsafe.Pointer) {
            for i := 0; i < len(fEngines) && i < len(offs); i++ {
                fEngines[i](e, unsafe.Pointer(uintptr(p)+offs[i]))
            }
        }
    case reflect.Interface:
        if rt.NumMethod() > 0 { //iface
            engine = func(e *Encoder, p unsafe.Pointer) {
                isNotNil := !isNil(p)
                e.encIsNotNil(isNotNil)
                if isNotNil {
                    v := reflect.ValueOf(*(*interface {
                        M() //注意这里的M的意思是临时定义一个含有M方法的interface. M无入参, 无返回值
                    })(p))
                    et := v.Type()
                    e.encString(getNameOfType(et))
                    getEncEngine(et)(e, getUnsafePointer(&v))
                }
            }
        } else { //eface
            engine = func(e *Encoder, p unsafe.Pointer) {
                isNotNil := !isNil(p)
                e.encIsNotNil(isNotNil)
                if isNotNil {
                    v := reflect.ValueOf(*(*interface{})(p)) 
                    et := v.Type()
                    e.encString(getNameOfType(et))
                    getEncEngine(et)(e, getUnsafePointer(&v))
                }
            }
        }
    case reflect.Chan, reflect.Func:
        panic("not support " + rt.String() + " type")
    default:
        engine = encEngines[kind]
    }
    rt2encEng[rt] = engine //build一次, 永久使用
    *engPtr = engine
}
```
注:
* `func buildEncEngine(rt reflect.Type, engPtr *encEng)`的递归形式是出参形式, 而非return方式  
`type encEng func(*Encoder, unsafe.Pointer)`中的`unsafe.Pointer`指向的类型就是前面`rt reflect.Type`, 它们是NewEncoder的入参interface的两种表达, `rt reflect.Type`是类型, 是入参interface用`TypeOf`得来; `unsafe.Pointer`是数据, 是入参interface的data域`(*[2]unsafe.Pointer)(unsafe.Pointer(入参的地址))[1]`  
* interface类型先编码一个encIsNotNil, 再编码其concrete类型的名字`e.encString(getNameOfType(et))`, 最后编码其值.
* 也就是说interface类型的编码前面总是会先编码类型名.
* 整个buildEncEngine的过程是有锁的.

### 每个类型都对应一个encEngine
unsafe.Pointer指向这个类型的实例  
在编码Interface时, 先编码是否nil, 再编码类型名, 最后编码实体对象.
这里的类型名怎么来的? 首先要知道engine的数据指针`p unsafe.Pointer`对应的数据类型就是这个engine对应的类型. 

```go
case reflect.Interface:
        if rt.NumMethod() > 0 {
            engine = func(e *Encoder, p unsafe.Pointer) {
                isNotNil := !isNil(p)
                e.encIsNotNil(isNotNil)
                if isNotNil {
                    v := reflect.ValueOf(*(*interface {
                        M()
                    })(p))
                    et := v.Type()
                    e.encString(getNameOfType(et))
                    getEncEngine(et)(e, getUnsafePointer(&v))
                }
            }
        } else {
            engine = func(e *Encoder, p unsafe.Pointer) {
                isNotNil := !isNil(p)
                e.encIsNotNil(isNotNil)
                if isNotNil {
                    v := reflect.ValueOf(*(*interface{})(p))
                    et := v.Type()
                    e.encString(getNameOfType(et))
                    getEncEngine(et)(e, getUnsafePointer(&v))
                }
            }
        }
```
比如上面代码中, `p unsafe.Pointer`指向的是第一行的`reflect.Interface`. 已知这个data指针p, 想得到其值v, 有两种情况:
* 带方法的iface
这里定义了一个临时的方法`M()`, 用来占位?
```go
v := reflect.ValueOf(*(*interface {
                        M()
                    })(p))
```
* 不带方法的eface
```go
v := reflect.ValueOf(*(*interface{})(p))
```

### 获取Interface的实体类型名
得到v后, 从`et := v.Type()`中就能得到类型名`getNameOfType(et)`:
```go
func getNameOfType(rt reflect.Type) string {
    if name, has := type2name[rt]; has { //先从已知表中查找
        return name
    } else {
        return registerType(rt)
    }
}

func Register(i interface{}) string {
    return registerType(reflect.TypeOf(i))
}

func registerType(rt reflect.Type) string {
    name := GetNameByType(rt)
    RegisterName(name, rt)
    return name
}

func RegisterName(name string, rt reflect.Type) {
    if name == "" {
        panic("attempt to register empty name")
    }

    if rt == nil || rt.Kind() == reflect.Invalid {
        panic("attempt to register nil type or invalid type")
    }

    if _, has := type2name[rt]; has {
        panic("gotiny: registering duplicate types for " + GetNameByType(rt))
    }

    if _, has := name2type[name]; has {
        panic("gotiny: registering name" + name + " is exist")
    }
    name2type[name] = rt
    type2name[rt] = name
}
```
如果已知表中没有查到, 就要注册这个`rt`. 注册就是把type和name分别放到两个查找表中(name2type, type2name), 双向可查.
重点是
```go
func GetNameByType(rt reflect.Type) string {
    return string(getName([]byte(nil), rt))
}

func getName(prefix []byte, rt reflect.Type) []byte {
    if rt == nil || rt.Kind() == reflect.Invalid {
        return append(prefix, []byte("<nil>")...)
    }
    if rt.Name() == "" { //未命名的，组合类型
        switch rt.Kind() {
        case reflect.Ptr:
            return getName(append(prefix, '*'), rt.Elem())
        case reflect.Array: // array是带大小的, 比如[11]
            return getName(append(prefix, "["+strconv.Itoa(rt.Len())+"]"...), rt.Elem())
        case reflect.Slice: // slice只是加个[]
            return getName(append(prefix, '[', ']'), rt.Elem())
        case reflect.Struct: // struct的形式类似struct{f1 f1Type; f2 f2Type}
            prefix = append(prefix, "struct {"...)
            nf := rt.NumField()
            if nf > 0 {
                prefix = append(prefix, ' ')
            }
            for i := 0; i < nf; i++ {
                field := rt.Field(i)
                if field.Anonymous {
                    prefix = getName(prefix, field.Type)
                } else {
                    prefix = getName(append(prefix, field.Name+" "...), field.Type)
                }
                if i != nf-1 {
                    prefix = append(prefix, ';', ' ')
                } else {
                    prefix = append(prefix, ' ')
                }
            }
            return append(prefix, '}')
        case reflect.Map:
            return getName(append(getName(append(prefix, "map["...), rt.Key()), ']'), rt.Elem())
        case reflect.Interface: // interface会带所有的方法, 也是分号分隔.
            prefix = append(prefix, "interface {"...)
            nm := rt.NumMethod()
            if nm > 0 {
                prefix = append(prefix, ' ')
            }
            for i := 0; i < nm; i++ {
                method := rt.Method(i)
                fn := getName([]byte(nil), method.Type)
                prefix = append(prefix, method.Name+string(fn[4:])...)
                if i != nm-1 {
                    prefix = append(prefix, ';', ' ')
                } else {
                    prefix = append(prefix, ' ')
                }
            }
            return append(prefix, '}')
        case reflect.Func: //竟然还有Func
            prefix = append(prefix, "func("...)
            for i := 0; i < rt.NumIn(); i++ {
                prefix = getName(prefix, rt.In(i))
                if i != rt.NumIn()-1 {
                    prefix = append(prefix, ',', ' ')
                }
            }
            prefix = append(prefix, ')')
            no := rt.NumOut()
            if no > 0 {
                prefix = append(prefix, ' ')
            }
            if no > 1 {
                prefix = append(prefix, '(')
            }
            for i := 0; i < no; i++ {
                prefix = getName(prefix, rt.Out(i))
                if i != no-1 {
                    prefix = append(prefix, ',', ' ')
                }
            }
            if no > 1 {
                prefix = append(prefix, ')')
            }
            return prefix
        }
    }
    
    //有名的类型走这里, 直接加上package名和类型名
    if rt.PkgPath() == "" {
        prefix = append(prefix, rt.Name()...)
    } else {
        prefix = append(prefix, rt.PkgPath()+"."+rt.Name()...)
    }
    return prefix
}
```

### 从指向interface的指针到其数据(解引用)
还是下面这段代码, 已知指向interface `case reflect.Interface` 的指针p `p unsafe.Pointer` 
求其值的解码
```go
    case reflect.Interface:
        //化简为eface
            engine = func(e *Encoder, p unsafe.Pointer) {
                isNotNil := !isNil(p)
                e.encIsNotNil(isNotNil)
                if isNotNil {
                    v := reflect.ValueOf(*(*interface{})(p))
                    et := v.Type()
                    e.encString(getNameOfType(et))
                    getEncEngine(et)(e, getUnsafePointer(&v))
                }
            }
```
以eface为例. 首先p是个指针, 指向上级interface的data, 而这个data还是个interface. 这里先把p转成指向interface的指针, 那么再取值`*(*interface{})(p)`就是这个interface了.  
然后ValueOf这个值得到的v, 就是要被编码的interface的reflect.Value表达.  
那么接下来先编码代表这个类型的字符串.  
然后递归的调用`getEncEngine(et)(e, getUnsafePointer(&v))`

在unsafe.go中, 作者把`reflect.Value`以复制代码的形式, 定义成`refVal`.  
"原版"的`reflect.Value`的域成员是小写的无法导出, 只有源码拷贝定义才行.  
这里用到了go的链接指示词`//go:linkname flagIndir reflect.flagIndir`, 把`flagIndir`当作`reflect.flagIndir`来链接.

```go
type refVal struct {
    _    unsafe.Pointer
    ptr  unsafe.Pointer
    flag flag
}

type flag uintptr

//go:linkname flagIndir reflect.flagIndir
const flagIndir flag = 1 << 7

func getUnsafePointer(rv *reflect.Value) unsafe.Pointer {
    vv := (*refVal)(unsafe.Pointer(rv))
    if vv.flag&flagIndir == 0 { //如果flagIndir位是0, 这个ptr本身就是值
        return unsafe.Pointer(&vv.ptr)
    } else {
        return vv.ptr //ptr是指针
    }
}
```
这里的逻辑是, 如果`rv *reflect.Value`的ptr是个Pointer-valued data, 就返回它的地址; 如果是普通的指针, 就返回它本身.

#### 疑问
这里先取到指针p指向的interface, 获得其reflect.Value表达v, 然后用reflect.Value再去生成encEngine来调用, 并传入getUnsafePointer(&v)做为下一级的unsafe.Pointer.
问题是, 为什么不直接用interface呢?
像这样:
```go
    case reflect.Interface:
        //化简为eface
            engine = func(e *Encoder, p unsafe.Pointer) {
                isNotNil := !isNil(p)
                e.encIsNotNil(isNotNil)
                if isNotNil {
                    i := *(*interface{})(p) //先得到这个interface的值 i
                    et := reflect.TypeOf(i)
                    e.encString(getNameOfType(et))
                    getEncEngine(et)(e, (*[2]unsafe.Pointer)(unsafe.Pointer(&i))[1])
                    //或者直接用p
                    getEncEngine(et)(e, (*[2]unsafe.Pointer)(p)[1])
                }
            }
```
有两种可能:
1. 避免值拷贝. 很明显`i := *(*interface{})(p)`会发生interface的值拷贝, 但拷贝interface似乎不heavy?
2. interface里面存interface的情况下, 不能用"第二个"unsafe.Pointer指针找到data.

### 补充: reflect.Value源码
根据`/usr/local/go/src/reflect/value.go`的注释:
```go
type Value struct {
    // typ holds the type of the value represented by a Value.
    typ *rtype
    // Pointer-valued data or, if flagIndir is set, pointer to data.
    // Valid when either flagIndir is set or typ.pointers() is true.
    ptr unsafe.Pointer
    // flag holds metadata about the value.
    flag
}

type flag uintptr

const (
    flagKindWidth        = 5 // there are 27 kinds
    flagKindMask    flag = 1<<flagKindWidth - 1
    flagStickyRO    flag = 1 << 5
    flagEmbedRO     flag = 1 << 6
    flagIndir       flag = 1 << 7
    flagAddr        flag = 1 << 8
    flagMethod      flag = 1 << 9
    flagMethodShift      = 10
    flagRO          flag = flagStickyRO | flagEmbedRO
)
```


### 总结
* 为什么要defer里面递归?  
因为这个函数的最后, 把生成的engine要放到全局变量rt2encEng中, 以后再次遇到同类型的值, 就不需要再build一次了.  
defer的意思是先让本次的结果进到这个rt2encEng里面, 这样如果递归的过程还需要用到这个类型, 就可以直接查找到了. 很神
* `func buildEncEngine(rt reflect.Type, engPtr *encEng)`的递归形式是出参形式, 而非return方式  
`type encEng func(*Encoder, unsafe.Pointer)`中的`unsafe.Pointer`指向的类型就是前面`rt reflect.Type`, 它们是NewEncoder的入参interface的两种表达, `rt reflect.Type`是类型, 是入参interface用`TypeOf`得来; `unsafe.Pointer`是数据, 是入参interface的data域`(*[2]unsafe.Pointer)(unsafe.Pointer(入参的地址))[1]`
* interface类型先编码一个encIsNotNil, 再编码其concrete类型的名字`e.encString(getNameOfType(et))`, 最后编码其值.
* 也就是说interface类型的编码前面总是会先编码类型名.
* 整个buildEncEngine的过程是有锁的.

# 代码阅读之decode
## Unmarshal
```go
func Unmarshal(buf []byte, is ...interface{}) int {
    return NewDecoderWithPtr(is...).Decode(buf, is...)
}
```
Decode的核心逻辑是按照入参is的类型, 从buf里解码

## 解码engine
解码engine的形式竟然和编码一样:
`type decEng func(*Decoder, unsafe.Pointer)`
也是从一个全局表里查
```go
    rt2decEng = map[reflect.Type]decEng{
        reflect.TypeOf((*bool)(nil)).Elem():           decBool,
        reflect.TypeOf((*int)(nil)).Elem():            decInt,
        reflect.TypeOf((*int8)(nil)).Elem():           decInt8,
        reflect.TypeOf((*int16)(nil)).Elem():          decInt16,
        reflect.TypeOf((*int32)(nil)).Elem():          decInt32,
        reflect.TypeOf((*int64)(nil)).Elem():          decInt64,
        reflect.TypeOf((*uint)(nil)).Elem():           decUint,
        reflect.TypeOf((*uint8)(nil)).Elem():          decUint8,
        reflect.TypeOf((*uint16)(nil)).Elem():         decUint16,
        reflect.TypeOf((*uint32)(nil)).Elem():         decUint32,
        reflect.TypeOf((*uint64)(nil)).Elem():         decUint64,
        reflect.TypeOf((*uintptr)(nil)).Elem():        decUintptr,
        reflect.TypeOf((*unsafe.Pointer)(nil)).Elem(): decPointer,
        reflect.TypeOf((*float32)(nil)).Elem():        decFloat32,
        reflect.TypeOf((*float64)(nil)).Elem():        decFloat64,
        reflect.TypeOf((*complex64)(nil)).Elem():      decComplex64,
        reflect.TypeOf((*complex128)(nil)).Elem():     decComplex128,
        reflect.TypeOf((*[]byte)(nil)).Elem():         decBytes,
        reflect.TypeOf((*string)(nil)).Elem():         decString,
        reflect.TypeOf((*time.Time)(nil)).Elem():      decTime,
        reflect.TypeOf((*struct{})(nil)).Elem():       decIgnore,
        reflect.TypeOf(nil):                           decIgnore,
    }
```

### Decoder类型
```go
type Decoder struct {
    buf     []byte //buf
    index   int    //下一个要使用的字节在buf中的下标
    boolPos byte   //下一次要读取的bool在buf中的下标,即buf[boolPos]
    boolBit byte   //下一次要读取的bool的buf[boolPos]中的bit位

    engines []decEng //解码器集合
    length  int      //解码器数量
}
```

### 解码bool类型
解码的思路就是把数据还原, 写到unsafe.Pointer中. 
比如解码bool, 是编码bool的逆过程
```go
func decBool(d *Decoder, p unsafe.Pointer)      { *(*bool)(p) = d.decBool() }

func (d *Decoder) decBool() (b bool) {
    if d.boolBit == 0 {
        d.boolBit = 1
        d.boolPos = d.buf[d.index]
        d.index++
    }
    b = d.boolPos&d.boolBit != 0
    d.boolBit <<= 1
    return
}
```

### 解码string
先解出len, 把buf的index到index+len的字符串拷贝到指针p里.
```go
func decString(d *Decoder, p unsafe.Pointer) {
    l, val := int(d.decUint32()), (*string)(p)
    *val = string(d.buf[d.index : d.index+l])
    d.index += l
}
```

对比编码string的过程是先编码p指向的string的len, 再依次拷贝s的字符到buf
```go
func encString(e *Encoder, p unsafe.Pointer) {
    s := *(*string)(p)
    e.encUint32(uint32(len(s)))
    e.buf = append(e.buf, s...)
}
```

## getDecEngine()
和Marshal一样, Unmarshal也要先build解码器, 其核心是
`engines[i] = getDecEngine(reflect.TypeOf(入参interface))`
也是先从全局表查找, 基础类型的decEngine早已写好到rt2decEng
```go
func getDecEngine(rt reflect.Type) decEng {
    decLock.RLock()
    engine := rt2decEng[rt]
    decLock.RUnlock()
    if engine != nil {
        return engine
    }
    decLock.Lock()
    buildDecEngine(rt, &engine)
    decLock.Unlock()
    return engine
}
```
复合类型需要build
```go
func buildDecEngine(rt reflect.Type, engPtr *decEng) {
    //如果递归调用到这里, 先看看之前是否有新加入的decEng
    engine, has := rt2decEng[rt]
    if has {
        *engPtr = engine
        return
    }
    
    if _, engine = implementOtherSerializer(rt); engine != nil {
        rt2decEng[rt] = engine
        *engPtr = engine
        return
    }
    
    kind := rt.Kind()
    var eEng decEng
    switch kind {
    case reflect.Ptr: //p指向的是ptr
        et := rt.Elem()
        defer buildDecEngine(et, &eEng)
        engine = func(d *Decoder, p unsafe.Pointer) {
            if d.decIsNotNil() { //因为是ptr, 在编码的时候是先编码一个bool的, 标识是否nil
                if isNil(p) { // p指向的值是nil
                    *(*unsafe.Pointer)(p) = unsafe.Pointer(reflect.New(et).Elem().UnsafeAddr()) // 新申请一个et类型的变量空间, 并赋值给p; 现在p指向这个新的空间. 关键是reflect.New函数
                }
                eEng(d, *(*unsafe.Pointer)(p)) //调用指针指向的value的decEng
            } else if !isNil(p) { //原值(即编码时的值)就是nil
                *(*unsafe.Pointer)(p) = nil
            }
        }
    case reflect.Array: //和编码结构高度一致
        l, et := rt.Len(), rt.Elem()
        size := et.Size()
        defer buildDecEngine(et, &eEng)
        engine = func(d *Decoder, p unsafe.Pointer) {
            for i := 0; i < l; i++ { // 我怎么记得要先编码array的len呀? -- 没有, 我记错了
                eEng(d, unsafe.Pointer(uintptr(p)+uintptr(i)*size)) //似乎这里不用判断p是否为空
            }
        }
    case reflect.Slice:
        et := rt.Elem()
        size := et.Size()
        defer buildDecEngine(et, &eEng)
        engine = func(d *Decoder, p unsafe.Pointer) {
            header := (*reflect.SliceHeader)(p)
            if d.decIsNotNil() {
                l := d.decLength()
                if isNil(p) || header.Cap < l {
                    *header = reflect.SliceHeader{Data: reflect.MakeSlice(rt, l, l).Pointer(), Len: l, Cap: l}
                } else {
                    header.Len = l
                }
                for i := 0; i < l; i++ {
                    eEng(d, unsafe.Pointer(header.Data+uintptr(i)*size))
                }
            } else if !isNil(p) {
                *header = reflect.SliceHeader{}
            }
        }
    case reflect.Map:
        kt, vt := rt.Key(), rt.Elem()
        skt, svt := reflect.SliceOf(kt), reflect.SliceOf(vt)
        var kEng, vEng decEng
        defer buildDecEngine(kt, &kEng)
        defer buildDecEngine(vt, &vEng)
        engine = func(d *Decoder, p unsafe.Pointer) {
            if d.decIsNotNil() {
                l := d.decLength()
                var v reflect.Value
                if isNil(p) {
                    v = reflect.MakeMapWithSize(rt, l)
                    *(*unsafe.Pointer)(p) = unsafe.Pointer(v.Pointer())
                } else {
                    v = reflect.NewAt(rt, p).Elem()
                }
                keys, vals := reflect.MakeSlice(skt, l, l), reflect.MakeSlice(svt, l, l)
                for i := 0; i < l; i++ {
                    key, val := keys.Index(i), vals.Index(i)
                    kEng(d, unsafe.Pointer(key.UnsafeAddr()))
                    vEng(d, unsafe.Pointer(val.UnsafeAddr()))
                    v.SetMapIndex(key, val)
                }
            } else if !isNil(p) {
                *(*unsafe.Pointer)(p) = nil
            }
        }
    case reflect.Struct:
        fields, offs := getFieldType(rt, 0)
        nf := len(fields)
        fEngines := make([]decEng, nf)
        defer func() {
            for i := 0; i < nf; i++ {
                buildDecEngine(fields[i], &fEngines[i])
            }
        }()
        engine = func(d *Decoder, p unsafe.Pointer) {
            for i := 0; i < len(fEngines) && i < len(offs); i++ {
                fEngines[i](d, unsafe.Pointer(uintptr(p)+offs[i]))
            }
        }
    case reflect.Interface: //我主要想看interface
        engine = func(d *Decoder, p unsafe.Pointer) {
            if d.decIsNotNil() {
                name := ""
                decString(d, unsafe.Pointer(&name))
                et, has := name2type[name] //直接查表, 说明必须先手动register. 是不是需要补个自动register过程? 但似乎不好做... 这也是为什么gob也需要显式注册interface的concrete type
                if !has {
                    panic("unknown typ:" + name)
                }
                v := reflect.NewAt(rt, p).Elem()
                var ev reflect.Value
                if v.IsNil() || v.Elem().Type() != et {
                    ev = reflect.New(et).Elem()
                } else {
                    ev = v.Elem()
                }
                getDecEngine(et)(d, getUnsafePointer(&ev))
                v.Set(ev)
            } else if !isNil(p) {
                *(*unsafe.Pointer)(p) = nil
            }
        }
        
    case reflect.Chan, reflect.Func:
        panic("not support " + rt.String() + " type")
    default:
        engine = baseDecEngines[kind]
    }
    rt2decEng[rt] = engine
    *engPtr = engine
}
```

## 总结
* 解码的思路就是把数据还原, 写到unsafe.Pointer p中
* 但这里一个明显的不足是, 必须要预先指定要解码的数据类型. 解码器需要按照指定的入参类型来解码.
* 要decode interface, 需要预先显式注册该interface的实体类型(即建立name到reflect.Type的映射关系). 这点和gob一样. 编码interface的类型字符串只是用来当作key查找其reflect.Type类型的.

# encoder和decoder总结
* encoder根据入参类型build encEngine表; decoder类似
* 如果要decode interface, 要先注册其concrete类型
* encode不需要预先注册interface的concrete类型, encode流程里会自动注册.
