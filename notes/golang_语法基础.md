- [初始化和空值](#初始化和空值)
  - [结构体](#结构体)
  - [声明空切片](#声明空切片)
    - [new切片](#new切片)
    - [空切片的实例化](#空切片的实例化)
  - [总结](#总结)
- [字符串支持比较操作符](#字符串支持比较操作符)
- [空白标识符](#空白标识符)
  - [空白标识符和err](#空白标识符和err)
  - [空白标识符和编译unused检查](#空白标识符和编译unused检查)
  - [空白标识符和类型检查](#空白标识符和类型检查)
- [go 是静态类型语言](#go-是静态类型语言)
  - [interface类型的变量可以重复赋值为任意类型](#interface类型的变量可以重复赋值为任意类型)
  - [可以在循环里用:语法糖赋值](#可以在循环里用语法糖赋值)
  - [连续赋值可以支持重复声明](#连续赋值可以支持重复声明)
- [结构体和json反射](#结构体和json反射)
  - [结构体定义里的反射字段](#结构体定义里的反射字段)
- [反射](#反射)
- [Golang的单引号、双引号与反引号](#golang的单引号双引号与反引号)
- [变长参数](#变长参数)
- [flag包用来解析cmd参数](#flag包用来解析cmd参数)
- [内置len copy 和cap](#内置len-copy-和cap)
- [字符串 字节数组 符文](#字符串-字节数组-符文)
- [常量和iota](#常量和iota)
- [格式化和scan](#格式化和scan)
  - [print](#print)
  - [scan](#scan)
- [减小go可执行文件的size](#减小go可执行文件的size)
- [go doc看说明](#go-doc看说明)
- [go内置pacakge](#go内置pacakge)
- [go 环境变量](#go-环境变量)
- [go test框架](#go-test框架)
- [远程包](#远程包)
- [go 工程布局(layout)](#go-工程布局layout)
  - [典型的go workspace布局](#典型的go-workspace布局)
  - [完整布局参考](#完整布局参考)
- [go知识点](#go知识点)
- [值传递和指针类型](#值传递和指针类型)
- [struct](#struct)
  - [形式](#形式)
  - [结构体方法](#结构体方法)
    - [基于指针对象的方法](#基于指针对象的方法)
  - [继承](#继承)
  - [结构体可以比较](#结构体可以比较)
  - [new分配内存](#new分配内存)
  - [工厂模式初始化](#工厂模式初始化)
- [接口 interface](#接口-interface)
  - [类型断言](#类型断言)
  - [类型断言判断对象是否实现了一个接口](#类型断言判断对象是否实现了一个接口)
- [goroutine](#goroutine)
  - [通道](#通道)
  - [带缓冲的通道](#带缓冲的通道)
  - [通道用close来关闭](#通道用close来关闭)
- [切片](#切片)
  - [切片的append](#切片的append)
- [map 集合](#map-集合)
  - [delete可以删除元素](#delete可以删除元素)
- [range](#range)

# 初始化和空值

## 结构体
先说结论, 初始化时没有赋值的结构体, 其**内容**是零值, 但这个对象的地址不是nil; 这个对象也不能和nil比较

```go
type MystructStr struct {
    s string
    a int
}
var ms MystructStr
//ms是在地址0xc0000d00e0上的24字节大小的结构体
({ 0 0}, main.MystructStr)@[*0xc0000d00e0, 24]
//&ms不是nil, ms本身也不能和nil比较
```

## 声明空切片
对于切片, map等对象来说, 变量名代指切片; 切片的地址可以和nil比较, 切片也可以和nil比较, 这是两码事.
切片和nil可以比较是go语法的规定.

```go
var sl []int

//判断成立, 会打印
//这里应该理解成, sl的内容为空; 注意sl的内容为空, 不是说它本身的地址是nil.
//做为一个变量, sl本身的地址不可能为nil.
if sl == nil {
    fmt.Println("nillll")
}
```

### new切片
对于new返回的地址, sn是真正的地址类似于`sn := &[]int{}`, 变量名指切片的地址  
那么sn就不是nil, 但它指向一个空的切片
```go
sn := new([]int)
if sn != nil {
    fmt.Println(sn)
}
    
//结果
&[]
```

### 空切片的实例化
```go
//si不为nil
si := []int{}
```

## 总结
```go
//len(s1)=0;cap(s1)=0;s1==nil
var s1 []int

//len(s1)=0;cap(s1)=0;s2!=nil
s2 := []int{}

//len(s3)=0;cap(s3)=0;s3!=nil
s3 := make([]int, 0)
```

* 指针是否为nil说的是指针是否指向东西. 切片等结构体头也类似于指针. 所以切片是否为nil也说的是切片头是否指向实际的东西
* 只有`var s1 []int`形式的声明, 声明了一个切片变量, 只是个声明, 没有给"指针"赋值, 所以此时s1为nil
* `new`也没有实例化, 但`new`返回指针, 指向interface对象本身, 所以也不是nil
* 其他形式, 比如`:=` `make`, 都会实例化, 即接口的"指针"字段都指向实例. 所以都不是nil


# 字符串支持比较操作符
原生的比较符可以直接用来比较字符串 `==, !=, >=, <=, <, >. `
它们都返回bool值
另外一种写法是`func Compare(str1, str2 string) int`
```go
    result1 := "GFG" > "Geeks"
    fmt.Println("Result 1: ", result1) 
  
    result2 := "GFG" < "Geeks"
    fmt.Println("Result 2: ", result2) 
  
    result3 := "Geeks" >= "for"
    fmt.Println("Result 3: ", result3) 
  
    result4 := "Geeks" <= "for"
    fmt.Println("Result 4: ", result4) 
  
    result5 := "Geeks" == "Geeks"
    fmt.Println("Result 5: ", result5) 
  
    result6 := "Geeks" != "for"
    fmt.Println("Result 6: ", result6)
```

# 空白标识符
空白标识符可以占位任何类型
## 空白标识符和err
```go
if _, err := os.Stat(path); os.IsNotExist(err) {
    fmt.Printf("%s does not exist\n", path)
}
```
有时为了省事, 直接把err给"占位"了, 这样是不对的.
```go
// Bad! This code will crash if path does not exist.
fi, _ := os.Stat(path)
if fi.IsDir() {
    fmt.Printf("%s is a directory\n", path)
}
```

## 空白标识符和编译unused检查
go会检查代码, 没有用的import和变量会报错误. 用占位符可以让编译器happy:
这里fmt和io以及变量fd没有使用, 一般编译会报错, 用占位符就不会.
```go
package main

import (
    "fmt"
    "io"
    "log"
    "os"
)

var _ = fmt.Printf // For debugging; delete when done.
var _ io.Reader // For debugging; delete when done.

func main() {
    fd, err := os.Open("test.go")
    if err != nil {
        log.Fatal(err)
    }
    // TODO: use fd.
    _ = fd
}
```

----
把import的包赋值给占位符, 相当于不用这个包, 但包里的init会被调用.
```go
import _ "net/http/pprof"
```

## 空白标识符和类型检查
go的类型检查发生在编译时, 传入的对象必须和函数的参数类型一致.
但也可以运行时检查: 下面是json的encoder代码, 它检查如果传入的值实现了json.Marshaler接口,
就调用这个值的`MarshalJSON`方法, 而不是调用标准的`MarshalJSON`方法.
`m, ok := val.(json.Marshaler)`

这种运行时检查很常见, 比如判断一个值是否实现了某个接口, 可以这样:
用空白标识符来占位返回的值, 只看ok.
```go
if _, ok := val.(json.Marshaler); ok {
    fmt.Printf("value %v of type %T implements json.Marshaler\n", val, val)
}
```

json.Marshaler和json.RawMessage的定义
```go
Linux Mint 19.1 Tessa $ go doc json.Marshaler
type Marshaler interface {
        MarshalJSON() ([]byte, error)
}
    Marshaler is the interface implemented by types that can marshal themselves
    into valid JSON.

yingjieb@yingjieb-VirtualBox ~/repo/myrepo/try
Linux Mint 19.1 Tessa $ go doc json.RawMessage
type RawMessage []byte
    RawMessage is a raw encoded JSON value. It implements Marshaler and
    Unmarshaler and can be used to delay JSON decoding or precompute a JSON
    encoding.

func (m RawMessage) MarshalJSON() ([]byte, error)
func (m *RawMessage) UnmarshalJSON(data []byte) error
```

如果`json.Marshaler`的定义变了, 那么一个原本实现了这个接口的类型,就不再有效了.
此时只有等到运行时的类型断言才能知道, 有办法在编译时就知道吗?
可以: 用空白标识符可以进行静态检查: 这里用`_`代替变量名
```go
//这里是个强制转换, 把nil转换为(*RawMessage)
//我认为这里转换为(RawMessage)也行.
var _ json.Marshaler = (*RawMessage)(nil)
```
如果`json.Marshaler`接口变化了, 这段代码就编不过.

# go 是静态类型语言
虽然go有语法糖, 可以根据右值来自动解析数据类型.
但不要把它当作动态语言来用了.
```go
// 编译器自动知道mys是string
mys := "hhhh"
// 给mys赋值64会报错: cannot use 64 (type int) as type string in assignment
mys = 64
```
动态语言, 变量可以随便赋值为不同种类的.

## interface类型的变量可以重复赋值为任意类型
> The interface{} (empty interface) type describes an interface with zero methods. Every Go type implements at least zero methods and therefore satisfies the empty interface.

```go
func describe(i interface{}) {
    fmt.Printf("(%v, %T)\n", i, i)
}

var mi interface{}
mi = "a string"
describe(mi)
mi = 2011
describe(mi)
mi = 2.777
describe(mi)

#输出
(a string, string)
(2011, int)
(2.777, float64)
```

## 可以在循环里用:语法糖赋值
```go
func main() {
    fmt.Println("Hello, playground")

    for i := 0; i < 10; i++ {
        // for里面冒号赋值没问题
        ii := i + 1
        fmt.Println(i, ii)
    }
}

```
## 连续赋值可以支持重复声明
一般在一个语句块里, 不能对单个变量重复用`:=`声明;
但可以对连续变量重复声明
> Unlike regular variable declarations, a short variable declaration may redeclare variables provided they were originally declared earlier in the same block with the same type, and at least one of the non-blank variables is new. As a consequence, redeclaration can only appear in a multi-variable short declaration. Redeclaration does not introduce a new variable; it just assigns a new value to the original.

```go
v := 1
v := 2
fmt.Println(v)


// 编译时, 只有v的第二次冒号赋值会报错.
./prog.go:18:4: no new variables on left side of :=

// 但下面是可以的
func main() {
    a, b := 1, 2
    c, b := 3, 4
    fmt.Println(a, b, c)
}

// 一个经常见的例子是
f, err := os.Open(name)
if err != nil {
    return err
}
d, err := f.Stat()
if err != nil {
    f.Close()
    return err
}
codeUsing(f, d)
```

注: open等调用返回的error是个内建的类型, 必须用`if err != nil`来判断;
不能用~~`if err`~~来判断, 因为error类型不是bool类型
`./prog.go:11:2: non-bool err (type error) used as if condition`



# 结构体和json反射
## 结构体定义里的反射字段
* 先定义和json对应的struct, 要导出的字段首字母大写. 
* `json.Unmarshal()`把json转为struct
* `json.Marshal()`把struct转为json
* 这里的反射大概是指
```go
//可以选择的控制字段有三种：
// -：不要解析这个字段
// omitempty：当字段为空（默认值）时，不要解析这个字段。比如 false、0、nil、长度为 0 的 array，map，slice，string
// FieldName：当解析 json 的时候，使用这个名字
type StudentWithOption struct {
    StudentId string //默认使用原定义中的值
    StudentName string `json:"sname"` // 解析（encode/decode） 的时候，使用 `sname`，而不是 `Field`
    StudentClass string `json:"class,omitempty"` // 解析的时候使用 `class`，如果struct 中这个值为空，就忽略它
    StudentTeacher string `json:"-"` // 解析的时候忽略该字段。默认情况下会解析这个字段，因为它是大写字母开头的
}
```

```go
//与json数据对应的结构体
type Server struct {
    ServerName string
    ServerIP string
}
// 数组对应slice
type ServerSlice struct {
    Servers []Server
}

//将JSON数据解析成结构体
package main

import (
    "encoding/json"
    "fmt"
)
func main() {
    var s ServerSlice
    str := `{"servers":[{"serverName":"TianJin","serverIP":"127.0.0.1"},
    {"serverName":"Beijing","serverIP":"127.0.0.2"}]}`
    json.Unmarshal([]byte(str), &s)
    fmt.Println(s)
}

----output-----
{[{TianJin 127.0.0.1} {Beijing 127.0.0.2}]}
```


# 反射
> Reflection（反射）在计算机中表示 程序能够检查自身结构的能力，尤其是类型

```go
func main() {
    var x float64 = 3.4
    fmt.Println(reflect.TypeOf(x))    //float64

    t := reflect.TypeOf(x)
    fmt.Println(t)    //float64
    // 注: 我认为其输出也可以叫reflect.Type
    fmt.Println(reflect.TypeOf(t))    //*reflect.rtype

    //相关代码在 go/src/reflect/value.go    
    v := reflect.ValueOf(x)
    fmt.Println(v)    //3.4
    fmt.Println(reflect.TypeOf(v))    //reflect.Value
    fmt.Println(v.Interface())    //3.4
    fmt.Println(v.Type())    //float64
}
```

* 变量包括（type, value）两部分
* type 包括 `static type`和`concrete type`. 简单来说 static type是你在编码是看见的类型(如int、string)，concrete type是runtime系统看见的类型
* 类型断言能否成功，取决于变量的`concrete type`，而不是static type. 因此，一个 reader变量如果它的concrete type也实现了write方法的话，它也可以被类型断言为writer.
* 只有interface类型才有反射一说
* 每个interface变量都有一个对应pair，pair中记录了实际变量的值和类型 `(value, type)`
* reflect包api

```go
//ValueOf用来获取输入参数接口中的数据的值，如果接口为空则返回0
func ValueOf(i interface{}) Value {...}

//func TypeOf(i interface{}) Type {...}
func TypeOf(i interface{}) Type {...}
```

* 反射可以大大提高程序的灵活性，使得interface{}有更大的发挥余地
    * 反射必须结合interface才玩得转
    * 变量的type要是concrete type的（也就是interface变量）才有反射一说
* 反射可以将“接口类型变量”转换为“反射类型对象”
    * 反射使用 TypeOf 和 ValueOf 函数从接口中获取目标对象信息
* 反射可以将“反射类型对象”转换为“接口类型变量
    * reflect.value.Interface().(已知的类型)
    * 遍历reflect.Type的Field获取其Field
* 反射可以修改反射类型对象，但是其值必须是“addressable”
    * 想要利用反射修改对象状态，前提是 interface.data 是 settable,即 pointer-interface
* 通过反射可以“动态”调用方法
* 因为Golang本身不支持模板，因此在以往需要使用模板的场景下往往就需要使用反射(reflect)来实现

# Golang的单引号、双引号与反引号
Go语言的字符串类型string在本质上就与其他语言的字符串类型不同：
* Java的String、C++的std::string以及Python3的str类型都只是定宽字符序列
* Go语言的字符串是一个用UTF-8编码的变宽字符序列，它的每一个字符都用一个或多个字节表示

即：一个Go语言字符串是一个任意字节的常量序列。

Golang的双引号和反引号都可用于表示一个常量字符串，不同在于：
* 双引号用来创建可解析的字符串字面量(支持转义，但不能用来引用多行)
* 反引号用来创建原生的字符串字面量，这些字符串可能由多行组成(不支持任何转义序列)，原生的字符串字面量多用于书写多行消息、HTML以及正则表达式

* 而单引号则用于表示Golang的一个特殊类型：rune，类似其他语言的byte但又不完全一样，是指：码点字面量（Unicode code point），不做任何转义的原始内容。

# 变长参数
```go
func append(slice []Type, elems ...Type) []Type

//举例
orig, err := ioutil.ReadFile(file)
//这里orig后面的三个点, 是展开orig的意思
in := append([]byte{}, orig...)

//举例
func sum(nums ...int) {
    fmt.Print(nums, " ") //输出如 [1, 2, 3] 之类的数组
    total := 0
    for _, num := range nums { //要的是值而不是下标
        total += num
    }
    fmt.Println(total)
}
func main() {
    sum(1, 2)
    sum(1, 2, 3)
 
    //传数组
    nums := []int{1, 2, 3, 4}
    //相当于把nums打散成元素, 依次传入sum
    sum(nums...)
}
```
# flag包用来解析cmd参数
```go
import "flag"
//定义flag, flag.Int返回一个指针: *int
var ip = flag.Int("flagname", 1234, "help message for flagname")
flag.Var(&flagVal, "name", "help message for flagname")
//定义完了调用, 调用完了指针才会有内容
flag.Parse()
//代码里使用flag
fmt.Println("ip has value ", *ip)
fmt.Println("flagvar has value ", flagvar)
//cmd参数形式
--flag
-flag
-flag=x
--flag=x
-flag x // non-boolean flags only
//integer可以是如下形式, 也可以为负数
1234, 0664, 0x1234
//Bool可以是
1, 0, t, f, T, F, true, false, TRUE, FALSE, True, False
```

# 内置len copy 和cap
```go
// cap是最大容量, len是实际容量
func cap(v Type) int
func len(v Type) int
func copy(dst, src []Type) int
```

# 字符串 字节数组 符文
* `[]byte`: byte数组

```go
b := []byte("ABC€")
fmt.Println(b) // [65 66 67 226 130 172]

//go的string是unicode编码(变长)的byte数组
s := string([]byte{65, 66, 67, 226, 130, 172})
fmt.Println(s) // ABC€
```
>  character € is encoded in UTF-8 using 3 bytes

* string: 本质上是只读的的byte数组, 对string的index返回对应的byte; 对大于255的字符(character)来说, 占多个byte

```go
func main() {
    const placeOfInterest = `⌘`

    fmt.Printf("plain string: ")
    fmt.Printf("%s", placeOfInterest)
    fmt.Printf("\n")

    fmt.Printf("quoted string: ")
    fmt.Printf("%+q", placeOfInterest)
    fmt.Printf("\n")

    fmt.Printf("hex bytes: ")
    for i := 0; i < len(placeOfInterest); i++ {
        fmt.Printf("%x ", placeOfInterest[i])
    }
    fmt.Printf("\n")
}
// 结果
plain string: ⌘
quoted string: "\u2318"
hex bytes: e2 8c 98
```

> UTF-8用1到6个字节编码Unicode字符

* rune: 是int32的别名

# 常量和iota
在常量表达式里面使用iota, 从0开始, 每行加一.
```go
type Stereotype int

const ( 
    TypicalNoob Stereotype = iota // 0 
    TypicalHipster // 1 TypicalHipster = iota
    TypicalUnixWizard // 2 TypicalUnixWizard = iota
    TypicalStartupFounder // 3 TypicalStartupFounder = iota
)
//如果两个const的赋值语句的表达式是一样的，那么可以省略后一个赋值表达式。
type AudioOutput int

const ( 
    OutMute AudioOutput = iota // 0 
    OutMono // 1 
    OutStereo // 2 
    _ 
    _ 
    OutSurround // 5 
)

type Allergen int

const ( 
    IgEggs Allergen = 1 << iota // 1 << 0 which is 00000001 
    IgChocolate // 1 << 1 which is 00000010 
    IgNuts // 1 << 2 which is 00000100 
    IgStrawberries // 1 << 3 which is 00001000 
    IgShellfish // 1 << 4 which is 00010000 
)

type ByteSize float64

const (
    _ = iota // ignore first value by assigning to blank identifier
    KB ByteSize = 1 << (10 * iota) // 1 << (10*1)
    MB // 1 << (10*2)
    GB // 1 << (10*3)
    TB // 1 << (10*4)
    PB // 1 << (10*5)
    EB // 1 << (10*6)
    ZB // 1 << (10*7)
    YB // 1 << (10*8)
)
```
# 格式化和scan
## print
```go
// 定义示例类型和变量
type Human struct {
    Name string
}

var people = Human{Name:"zhangsan"}
```
```go
普通占位符
占位符 说明 举例 输出
%v 相应值的默认格式。 Printf("%v", people) {zhangsan}，
%+v 打印结构体时，会添加字段名 Printf("%+v", people) {Name:zhangsan}
%#v 相应值的Go语法表示 Printf("%#v", people) main.Human{Name:"zhangsan"}
%T 相应值的类型的Go语法表示 Printf("%T", people) main.Human
%% 字面上的百分号，并非值的占位符 Printf("%%") %

func describe(i interface{}) {
    fmt.Printf("(%v, %T)\n", i, i)
}
```
```go
整数占位符
占位符 说明 举例 输出
%b 二进制表示 Printf("%b", 5) 101
%c 相应Unicode码点所表示的字符 Printf("%c", 0x4E2D) 中
%d 十进制表示 Printf("%d", 0x12) 18
%o 八进制表示 Printf("%d", 10) 12
%q 单引号围绕的字符字面值，由Go语法安全地转义 Printf("%q", 0x4E2D) '中'
%x 十六进制表示，字母形式为小写 a-f Printf("%x", 13) d
%X 十六进制表示，字母形式为大写 A-F Printf("%x", 13) D
%U Unicode格式：U+1234，等同于 "U+%04X" Printf("%U", 0x4E2D) U+4E2D
```
```go
字符串与字节切片
占位符 说明 举例 输出
%s 输出字符串表示（string类型或[]byte) Printf("%s", []byte("Go语言")) Go语言
%q 双引号围绕的字符串，由Go语法安全地转义 Printf("%q", "Go语言") "Go语言"
%x 十六进制，小写字母，每字节两个字符 Printf("%x", "golang") 676f6c616e67
%X 十六进制，大写字母，每字节两个字符 Printf("%X", "golang") 676F6C616E67
```
```go
指针
占位符 说明 举例 输出
%p 十六进制表示，前缀 0x Printf("%p", &people) 0x4f57f0
```
```go
其它标记
占位符 说明 举例 输出
+ 总打印数值的正负号；对于%q（%+q）保证只输出ASCII编码的字符。 
                                           Printf("%+q", "中文") "\u4e2d\u6587"
- 在右侧而非左侧填充空格（左对齐该区域）
# 备用格式：为八进制添加前导 0（%#o） Printf("%#U", '中') U+4E2D
       为十六进制添加前导 0x（%#x）或 0X（%#X），为 %p（%#p）去掉前导 0x；
       如果可能的话，%q（%#q）会打印原始 （即反引号围绕的）字符串；
       如果是可打印字符，%U（%#U）会写出该字符的
       Unicode 编码形式（如字符 x 会被打印成 U+0078 'x'）。
' ' (空格)为数值中省略的正负号留出空白（% d）；
       以十六进制（% x, % X）打印字符串或切片时，在字节之间用空格隔开
0 填充前导的0而非空格；对于数字，这会将填充移到正负号之后
```

## scan
```go
package main

import (
    "fmt"
)

func main() {
    var name string
    var age int
    n, err := fmt.Sscanf("Kim is 22 years old", "%s is %d years old", &name, &age)
    if err != nil {
        panic(err)
    }
    fmt.Printf("%d: %s, %d\n", n, name, age)
}
```

# 减小go可执行文件的size
```sh
#之前是15M, 带符号表
go build -ldflags "-s -w" main.go
#之后是7.3M, 不带符号表

go tool link -h
-s    disable symbol table
-w    disable DWARF generation
```
不影响panic的打印信息
比如, 在hello.go加入一行panic()
```sh
# 不要符号表, 不要DWARF
go build -ldflags "-s -w" hello.go

# 还是有panic信息
$ ./hello
panic:

goroutine 1 [running]:
main.main()
        /repo/yingjieb/godev/practice/src/examples/hello.go:39 +0xa3

```
注: 如果是gccgo, strip符号表会导致panic的打印没有调用栈信息.

# go doc看说明
```sh
#格式化输出的
go doc fmt
#命令行解析的
do doc flag
#导出变量的, via HTTP at /debug/vars in JSON format ???
go doc expvar
```
# go内置pacakge
go的内置package在toolchain的src目录下, 都是go文件.
```sh
yingjieb@yingjieb-VirtualBox ~/repo/gorepo/go/src
Linux Mint 19.1 Tessa $ ls
all.bash archive builtin clean.rc container debug flag html io make.bat mime os race.bat run.bat strconv testdata unicode
all.bat bootstrap.bash bytes cmd context encoding fmt image iostest.bash Make.dist naclmake.bash path reflect run.rc strings testing unsafe
all.rc bufio clean.bash cmp.bash crypto errors go index log make.rc nacltest.bash plugin regexp runtime sync text
androidtest.bash buildall.bash clean.bat compress database expvar hash internal make.bash math net race.bash run.bash sort syscall time
```

# go 环境变量

* `$GOROOT` : go toolchain的目录, 在编译go toolchain时写入默认值为all.bash的上级目录, 比如`/root/go-mips64`; 可用来在多个go toolchain间切换; buildroot默认在`HOST_GO_ROOT = $(HOST_DIR)/lib/go`, 见package/go/go.mk
* `$GOOS` and `$GOARCH` : 目标OS和CPU arch, 比如linux 和mips64. 默认是`$GOHOSTOS` and `$GOHOSTARCH`
* `$GOPATH` : 所有go程序的工作目录, 默认是用户home/go
* `$GOBIN` : go可执行二进制目录, 用go命令安装的bin在此. 默认是`$GOPATH/bin`
比如`go get golang.org/x/tools/cmd/godoc`下载, 编译, 安装`$GOBIN/godoc`

参考: [go环境变量](https://golang.org/doc/install/source#environment)

```
Linux Mint 19.1 Tessa $ go env
GOARCH="amd64"
GOBIN=""
GOCACHE="/home/yingjieb/.cache/go-build"
GOEXE=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOOS="linux"
GOPATH="/home/yingjieb/go"
GORACE=""
GOROOT="/usr/lib/go-1.10"
GOTMPDIR=""
GOTOOLDIR="/usr/lib/go-1.10/pkg/tool/linux_amd64"
GCCGO="gccgo"
CC="gcc"
CXX="g++"
CGO_ENABLED="1"
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=/tmp/go-build439003515=/tmp/go-build -gno-record-gcc-switches"

```

# go test框架
go有个集成的test框架, 包括
* go test命令
* 需包含testing包
* 被test的package下面, 创建`xxx_test.go`, 包含`func TestXxx(t *testing.T)`

比如`package stringutil`里实现了字符串反转的方法, 那么, 它的test要这么写

```go
package stringutil

import "testing"

func TestReverse(t *testing.T) {
    cases := []struct {
        in, want string
    }{
        {"Hello, world", "dlrow ,olleH"},
        {"Hello, 世界", "界世 ,olleH"},
        {"", ""},
    }
    for _, c := range cases {
        got := Reverse(c.in)
        if got != c.want {
            t.Errorf("Reverse(%q) == %q, want %q", c.in, got, c.want)
        }
    }
}
```

测试时, 
```sh
#从任意地方运行
go test github.com/user/stringutil
#如果从这个package下面运行
go test
```

# 远程包
有的包在github上, 用`go get`命令可以从远程repo下载 编译 安装指定包.
```sh
$ go get github.com/golang/example/hello
$ $GOPATH/bin/hello
Hello, Go examples!
```
被import的远程包, 本地没有的, 会被自动下载到workspace.

# go 工程布局(layout)
参考: https://golang.org/doc/code.html

go开发约定:
* go的所有程序都放在一个workspace下面
* 这个workspace下面放了很多repo, 用git或hg管理起来
* package name就是其package路径的basename, 比如crypto/rot13的package name就是rot13
go不要求package name是唯一的, 但要求其路径是唯一的.
* 一个repo包括一个或多个package
* 一个package放在一个目录下面, 包括一个或多个go源文件
* 一个package在workspace的路径, 就是它被import的路径
* 被import的package可以是remote的repo, go会自动下载到workspace里面

## 典型的go workspace布局
```
bin/
    hello # command executable
    outyet # command executable
src/
    github.com/golang/example/
        .git/ # Git repository metadata
    hello/
        hello.go # command source
    outyet/
        main.go # command source
        main_test.go # test source
    stringutil/
        reverse.go # package source
        reverse_test.go # test source
    golang.org/x/image/
        .git/ # Git repository metadata
    bmp/
        reader.go # package source
        writer.go # package source
    ... (many more repositories and packages omitted) ...
```
* `$GOPATH`就是这个workspace
```sh
export PATH=$PATH:$(go env GOPATH)/bin
export GOPATH=$(go env GOPATH)
```
* go install就是把编译好的可执行文件拷贝到`$GOPATH/bin`
比如可以在任意的地方执行`go install github.com/user/hello`
go会去`$GOPATH/src`找
* 对你写的go lib(不是以package main开头的是lib)来说, 
`go build`会把编译好的package保存在local build cache里
* go的package xxx, 这里的xxx是import路径的base name.
在引用的时候, 用相对src的路径引用

```go
package main

import (
    "fmt"
    "github.com/user/stringutil"
)

func main() {
    fmt.Println(stringutil.Reverse("!oG ,olleH"))
}
```
此时源文件布局如下:
```
bin/
    hello # command executable
src/
    github.com/user/
        hello/
            hello.go # command source
        stringutil/
            reverse.go # package source
```
## 完整布局参考
https://github.com/golang-standards/project-layout

# go知识点
```go
package main
import "fmt"
func main() {
   /* 这是我的第一个简单的程序 */
   fmt.Println("Hello, World!")
}
```
```sh
#运行
go run hello.go 
#只编译
go build hello.go
```
* `package main` package关键词表示这个文件属于哪个包, 一般都是多个源文件属于同一个包.
* import告诉go编译器要使用的包
* func main是程序开始执行的函数, 每个可执行的go程序必须包含main函数. 一般情况下, 程序会第一个执行main. 但如果有init()函数, 会先执行init()
* `{`不能占单独的一行
* 每行不必用`;`结尾
* 用`+`可以连接字符串, 比如`fmt.Println("Google" + "Runoob")`, 得到`GoogleRunoob`
* 当标识符（包括常量、变量、类型、函数名、结构字段等等）以一个大写字母开头，如：Group1，那么使用这种形式的标识符的对象就可以被外部包的代码所使用（客户端程序需要先导入这个包），这被称为导出（像面向对象语言中的 public）；标识符如果以小写字母开头，则对包外是不可见的，但是他们在整个包的内部是可见并且可用的（像面向对象语言中的 protected ）。
* go的数据类型有`bool byte(类似uint8) rune(类似int32) uintptr(无符号整型, 用于存放指针) int(32位或64位, 和体系架构有关) uint8/16/32/64 int8/16/32/64 float float32 float64 string struct channel 指针 数组 func 切片 interface map`等

```go
var a string = "Runoob"
//也可以省略类型, 编译器根据赋值自动推断
var a = "RUNOOB"
var b, c int = 1, 2
fmt.Println(a, b, c)
//没有初始化的变量, 默认为0, 或空字符串"", 或nil, 比如:
var a *int
var a []int
var a map[string] int
var a chan int
var a func(string) int
var a error // error 是接口
//也可以省略var, 用v_name := value形式, 但只能在函数体内出现?
f := "Runoob"
//空白标识符_只能写, 不能读, 用于占位, 抛弃不想要的值; 比如下面的值5就被抛弃了
_, b = 5, 7
//取地址&和取值*和C一样
``` 
* 用`type_name(expression)`做类型转换
* Go 没有三目运算符，所以不支持 ?: 形式的条件判断
* 循环只有`for`
```go
func main() {
    //也可以不要true, 直接一个for就行了
    for true {
        fmt.Printf("这是无限循环。\n");
    }
}
//一般的for形式都支持, 里面可以有break continue goto
for C < 100 {
for i := 1; i <= 9; i++ {
```
* 函数定义形式, 可以有多个返回值
```go
func function_name( [parameter list] ) [return_types] {
   函数体
}
/* 函数返回两个数的最大值 */
func max(num1, num2 int) int {
   /* 声明局部变量 */
   var result int

   if (num1 > num2) {
      result = num1
   } else {
      result = num2
   }
   return result 
}
```
* 函数可以直接"赋值"给变量, 在大部分现代语言中, 函数也可以当作变量
```go
func main(){
   /* 声明函数变量 */
   getSquareRoot := func(x float64) float64 {
      return math.Sqrt(x)
   }
   /* 使用函数 */
   fmt.Println(getSquareRoot(9))
}
```

* go的闭包, 在这里就是函数返回值是另一个函数

```go
//Go 语言支持匿名函数，可作为闭包。匿名函数是一个"内联"语句或表达式。匿名函数的优越性在于可以直接使用函数内的变量，不必申明。
//以下实例中，我们创建了函数 getSequence() ，返回另外一个函数。该函数的目的是在闭包中递增 i 变量，代码如下：

package main
import "fmt"
func getSequence() func() int {
   i:=0
   return func() int {
      i+=1
     return i  
   }
}

func main(){
   /* nextNumber 为一个函数，函数 i 为 0 */
   nextNumber := getSequence()  

   /* 调用 nextNumber 函数，i 变量自增 1 并返回 */
   fmt.Println(nextNumber())
   fmt.Println(nextNumber())
   fmt.Println(nextNumber())
   
   /* 创建新的函数 nextNumber1，并查看结果 */
   //重新调用getSequence()函数, i是重新申请的变量
   nextNumber1 := getSequence()  
   //这里的number会重新开始编号
   fmt.Println(nextNumber1())
   fmt.Println(nextNumber1())
   
   fmt.Println(nextNumber())
   fmt.Println(nextNumber())
}

//结果
1
2
3
1
2
//这里是nextNumber继续编号
4
5

//闭包也可以带参数
func main() {
    add_func := add(1,2)
    fmt.Println(add_func(1,1))
    fmt.Println(add_func(0,0))
    fmt.Println(add_func(2,2))
} 
// 闭包使用方法
//x1 x2是初始化add_func用的, x3 x4是传给add_func的
func add(x1, x2 int) func(x3 int,x4 int)(int,int,int) {
    i := 0
    return func(x3 int,x4 int) (int,int,int){ 
       i++
       return i,x1+x2,x3+x4
    }
}

//结果
1 3 2
2 3 0
3 3 4
```
* 和C一样, 局部变量在函数内使用; 全局变量在函数体外声明, 在整个包使用, 甚至可以被导出后的外部包使用.
* 数组变量定义: `var variable_name [SIZE] variable_type`, 和C一样, 下标从0开始
* 数组变量赋值
```go
var balance = [5]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
var balance = [...]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
var salary float32 = balance[9]
var a = [5][2]int{ {0,0}, {1,2}, {2,4}, {3,6},{4,8}}
```
* 指针和引用

```go
func main() {
   /* 定义局部变量 */
   var a int = 100
   var b int= 200

   fmt.Printf("交换前 a 的值 : %d\n", a )
   fmt.Printf("交换前 b 的值 : %d\n", b )

   /* 调用函数用于交换值
   * &a 指向 a 变量的地址
   * &b 指向 b 变量的地址
   */
   swap(&a, &b);

   fmt.Printf("交换后 a 的值 : %d\n", a )
   fmt.Printf("交换后 b 的值 : %d\n", b )
}

func swap(x *int, y *int) {
   var temp int
   temp = *x /* 保存 x 地址的值 */
   *x = *y /* 将 y 赋值给 x */
   *y = temp /* 将 temp 赋值给 y */
}
```
* 结构体, 和c不同的是, 结构体指针访问成员的时候, 也是用`.`

```go
type Books struct {
   title string
   author string
   subject string
   book_id int
}
//声明结构体变量并初始化
//variable_name := structure_variable_type {value1, value2...valuen}
//或
//variable_name := structure_variable_type { key1: value1, key2: value2..., keyn: valuen}
fmt.Println(Books{"Go 语言", "www.runoob.com", "Go 语言教程", 6495407})
   Book1.title = "Go 语言"
   Book1.author = "www.runoob.com"
   Book1.subject = "Go 语言教程"
   Book1.book_id = 6495407
```
* 切片, 数组的大小是固定的, 而切片大小可以变. "动态数组"

```go
func main() {
    /* 创建切片 */
    numbers := []int{0,1,2,3,4,5,6,7,8}   
    printSlice(numbers)

    /* 打印原始切片 */
    fmt.Println("numbers ==", numbers)

    /* 打印子切片从索引1(包含) 到索引4(不包含)*/
    fmt.Println("numbers[1:4] ==", numbers[1:4])

    /* 默认下限为 0*/
    fmt.Println("numbers[:3] ==", numbers[:3])

    /* 默认上限为 len(s)*/
    fmt.Println("numbers[4:] ==", numbers[4:])

    numbers1 := make([]int,0,5)
    printSlice(numbers1)

    /* 打印子切片从索引 0(包含) 到索引 2(不包含) */
    number2 := numbers[:2]
    printSlice(number2)

    /* 打印子切片从索引 2(包含) 到索引 5(不包含) */
    number3 := numbers[2:5]
    printSlice(number3)
}

func printSlice(x []int){
    fmt.Printf("len=%d cap=%d slice=%v\n",len(x),cap(x),x)
}
```

* range关键字用于 for 循环中迭代数组(array)、切片(slice)、通道(channel)或集合(map)的元素

```go
func main() {
    //这是我们使用range去求一个slice的和。使用数组跟这个很类似
    nums := []int{2, 3, 4}
    sum := 0
    for _, num := range nums {
        sum += num
    }
    fmt.Println("sum:", sum)
    //在数组上使用range将传入index和值两个变量。上面那个例子我们不需要使用该元素的序号，所以我们使用空白符"_"省略了。有时侯我们确实需要知道它的索引。
    for i, num := range nums {
        if num == 3 {
            fmt.Println("index:", i)
        }
    }
    //range也可以用在map的键值对上。
    kvs := map[string]string{"a": "apple", "b": "banana"}
    for k, v := range kvs {
        fmt.Printf("%s -> %s\n", k, v)
    }
    //range也可以用来枚举Unicode字符串。第一个参数是字符的索引，第二个是字符（Unicode的值）本身。
    for i, c := range "go" {
        fmt.Println(i, c)
    }
}
```

# 值传递和指针类型
有人总结: golang中的传参是值传递, 但因为map channel和slice等内置数据结构本身是**指针类型**, 所以它们作为参数传递时, **相当于传递了指针**.

# struct
## 形式
```go
type Books struct {
    title string
    author string
    subject string
    book_id int
}
```
## 结构体方法
Go 语言中同时有函数和方法。一个方法就是一个包含了接受者的函数，接受者可以是命名类型或者结构体类型的一个值或者是一个指针。所有给定类型的方法属于该类型的方法集
形式为: 注意和函数定义的区别
```go
func(receiver type)methodName([参数列表]) [返回值列表]{
}
```
普通函数定义为:
```go
func function_name( [parameter list] ) [return_types] {
    函数体
}
```
```go
package main

import (
    "fmt"
)

type Student struct{
    Name string
    Age int
}

func (stu *Student)Set(name string,age int){
    stu.Name = name
    stu.Age = age
}

func main(){
    var s Student
    s.Set("tome",23)
    fmt.Println(s)
}
//注意:方法的访问控制也是通过大小写控制的
//在上面这个例子中需要注意一个地方
//func (stu *Student)Set(name string,age int)
//这里使用的是(stu *Student)而不是(stu Student)这里其实是基于指针对象的方法
//当调用一个函数时，会对其每个参数值进行拷贝，如果一个函数需要更新一个变量，或者函数的其中一个参数是在太大
//我们希望能够避免进行这种默认的拷贝，这种情况下我们就需要用到指针了，所以在上一个代码例子中那样我们需要
//func (stu *Student)Set(name string,age int)来声明一个方法
```
### 基于指针对象的方法
```go
package main

import (
    "fmt"
)


type Point struct{
    X float64
    Y float64
}

func (p *Point) ScaleBy(factor float64){
    p.X *= factor
    p.Y *= factor
}

func main(){
    //两种方法
    //方法1
    r := &Point{1,2}
    r.ScaleBy(2)
    fmt.Println(*r)

    //方法2
    p := Point{1,2}
    pptr := &p
    pptr.ScaleBy(2)
    fmt.Println(p)

    //方法3
     p2 := Point{1,2}
     (&p2).ScaleBy(2)
     fmt.Println(p2)

     //相对来说方法2和方法3有点笨拙
     //方法4,go语言这里会自己判断p是一个Point类型的变量，
     //并且其方法需要一个Point指针作为指针接收器，直接可以用下面简单的方法
     p3 := Point{1,2}
     p3.ScaleBy(2)
     fmt.Println(p3)
}

//上面例子中最后一种方法，编译器会隐式的帮我们用&p的方法去调用ScaleBy这个方法
```
## 继承
```go
package main

import (
    "fmt"
)

type People struct{
    Name string
    Age int
}

type Student struct{
    People
    Score int
}

func main(){
    var s Student
    /*
    s.People.Name = "tome"
    s.People.Age = 23
    */
    //上面注释的用法可以简写为下面的方法
    s.Name = "tom"
    s.Age = 23

    s.Score = 100
    fmt.Printf("%+v\n",s)

//注意：关于字段冲突的问题，我们在People中定义了一个Name字段，在Student中再次定义Name,这个时候，我们通过s.Name获取的就是Student定义的Name字段
}
```
## 结构体可以比较
* 如果结构体的所有成员变量都是可比较的，那么结构体就可比较

* 如果结构体中存在不可比较的成员变量，那么结构体就不能比较

* map和切片不能比较

```go
package main

import (
    "fmt"
)

type Point struct{
    x int
    y int
}

func main(){
    p1 := Point{1,2}
    p2 := Point{2,3}
    p3 := Point{1,2}
    fmt.Println(p1==p2) //false
    fmt.Println(p1==p3) //true
}
```
## new分配内存
```go
package main

import (
    "fmt"
)

type Student struct {
    Id int
    Name string
}

func main() {
    s := new(Student)

    s.Id = 1
    s.Name = "test"
    s1 := Student{Id: 2, Name: "test1"}
    fmt.Println(s, s1)
}

//输出结果:
&{1 test} {2 test1}

//s 的类型为指针，s1 为一个Student类型; 
//说明new的返回值是指针, 而用struct_type{}声明的变量是struct_type实例本身.
```
## 工厂模式初始化
go没有构造函数, 所以工厂模式很常用
```go
package main

import (
    "fmt"
)

type Student struct{
    Name string
    Age int
}

func NewStudent(name string,age int) *Student{
    return &Student {
        Name:name,
        Age:age,
    }
}

func main(){
    s := NewStudent("tom",23)
    fmt.Println(s.Name)
}
```

# 接口 interface
Go 语言提供了另外一种数据类型即接口，它把所有的具有共性的方法定义在一起，任何其他类型只要实现了这些方法就是实现了这个接口。
```go
/* 定义接口 */
type interface_name interface {
   method_name1 [return_type]
   method_name2 [return_type]
   method_name3 [return_type]
   ...
   method_namen [return_type]
}

/* 定义结构体 */
type struct_name struct {
   /* variables */
}

/* 实现接口方法 */
func (struct_name_variable struct_name) method_name1() [return_type] {
   /* 方法实现 */
}
...
func (struct_name_variable struct_name) method_namen() [return_type] {
   /* 方法实现*/
}
```
```go
package main

import (
    "fmt"
)

type Phone interface {
    call()
}

type NokiaPhone struct {
}

func (nokiaPhone NokiaPhone) call() {
    fmt.Println("I am Nokia, I can call you!")
}

type IPhone struct {
}

func (iPhone IPhone) call() {
    fmt.Println("I am iPhone, I can call you!")
}

func main() {
    //这是interface类型的变量
    var phone Phone
    //new返回一个指针, 并对其内存清零; 
    //也可以写成phone = NokiaPhone{}, 或phone = &NokiaPhone{}
    phone = new(NokiaPhone)
    phone.call()

    phone = new(IPhone)
    phone.call()

}

//在上面的例子中，我们定义了一个接口Phone，接口里面有一个方法call()。
//然后我们在main函数里面定义了一个Phone类型变量，并分别为之赋#值为NokiaPhone和IPhone。
//然后调用call()方法，输出结果如下：

I am Nokia, I can call you!
I am iPhone, I can call you!
```

## 类型断言
类型断言提供了一个机制找出接口底层对应的实际类型: `t := i.(T)`
这个语句在断言接口i中实际包含了类型T，然后把底层类型T的值赋值给变量t
如果断言失败，i中没有包含T，这条语句会触发panic
为了测试接口是否包含指定的类型，类型断言会返回2个值，底层类型实际对应的值和一个bool值，来报告断言是否成功
`t, ok := i.(T)`
如果i中包含T，则t是底层类型的实际值，变量ok是真
如果断言失败，ok变量是假，t是一个零值类型T，不会触发panic，这个语法和对map操作类似

只有interface有类型断言, 因为interface可以指向任何东西, 所以要有办法知道它的运行时类型:
https://www.jianshu.com/p/6a46fc7b6e5b
* x.(T) 检查x的动态类型是否是T，其中x必须是接口值。
* switch形式:

```go
func main() {
    var x interface{}

    x = 17
    //fmt包看实际类型, 因为Printf的入参就是接口类型
    fmt.Printf("type x is %T, value x is %d\n", x, x)

    //这是个特殊的type switch, switch后面是个赋值表达式, 但case的对象是类型
    switch v := x.(type) {
    //case的顺序是有意义的，因为可能同时满足多个接口，不可以用fallthrough, default的位置无所谓。
    case nil:
        fmt.Printf(" x 的类型 :%T", v)
    case int:
        fmt.Printf("x 是 int 型, 值为%v", v)
    case float64:
        fmt.Printf("x 是 float64 型, 值为%v", v)
    case func(int) float64:
        fmt.Printf("x 是 func(int) 型, 值为%p", v)
    case bool, string:
        fmt.Printf("x 是 bool 或 string 型, 值为%v", v)
    default:
        fmt.Printf("未知型")
    }
}

//结果:
type x is int, value x is 17
x 是 int 型, 值为17
```


## 类型断言判断对象是否实现了一个接口
判断`val`是否实现了json.Marshaler需要的接口, 即val是否为json.Marshaler类型.
```go
if _, ok := val.(json.Marshaler); ok {
    fmt.Printf("value %v of type %T implements json.Marshaler\n", val, val)
}
```

# goroutine
goroutine 是轻量级线程，goroutine 的调度是由 Golang 运行时进行管理的。
```go
//语法
go 函数名(参数列表)
//比如
go f(x, y, z)
//并行执行, 就像shell的后台执行一样
```
```go
package main

import (
    "fmt"
    "time"
)

func say(s string) {
    for i := 0; i < 5; i++ {
        time.Sleep(100 * time.Millisecond)
        fmt.Println(s)
    }
}

func main() {
    go say("world")
    say("hello")
}

//执行以上代码，你会看到输出的 hello 和 world 是没有固定先后顺序。因为它们是两个 goroutine 在执行：
world
hello
hello
world
world
hello
hello
world
world
hello
```

## 通道
通道（channel）是用来传递数据的一个数据结构, 可用于两个 goroutine 之间通过传递一个指定类型的值来同步运行和通讯。操作符 <- 用于指定通道的方向，发送或接收。如果未指定方向，则为双向通道。
```go
// 声明一个通道很简单，我们使用chan关键字即可，通道在使用前必须先创建：
ch := make(chan int)
ch <- v // 把 v 发送到通道 ch
v := <-ch // 从 ch 接收数据, 并把值赋给 v
```
```go
// 也可以不用make来声明channel, 用var也可以; 默认初始化为nil
var c chan int
fmt.Println(c)
```
`select`语句形式上类似 `switch` 语句，但实际上和C里面的select差不多: 用于监听和channel有关的IO操作，当 IO 操作发生时，触发相应的动作.
* 和switch不同, select会对每个case, 都会对channel求值, 如果有可用的IO, 则随机执行其中一个case的后续指令.
* 如果没有可用的IO, 则执行default.
* 如果没有可用IO, 也没有default, 则一直阻塞到IO可用.
比如一个loadbalancer

```go
func (b *Banlancer) balance(work chan Request) {
    for {
        select {
            case req := <- work:
                b.dispatch(req)
            case w := <- b.done:
                b.completed(w)
        }
    }
}

func (b *Banlancer) dispatch(req Request) {
    w := heap.Pop(&b.pool)
    w.request <- req
    w.pending++
    heap.Push(&b.pool,w)
}
```

```go
package main

import "fmt"

func sum(s []int, c chan int) {
        sum := 0
        for _, v := range s {
                sum += v
        }
        c <- sum // 把 sum 发送到通道 c
}

func main() {
        s := []int{7, 2, 8, -9, 4, 0}

        c := make(chan int)
        go sum(s[:len(s)/2], c)
        go sum(s[len(s)/2:], c)
        x, y := <-c, <-c // 从通道 c 中接收

        fmt.Println(x, y, x+y)
}
//输出结果:
-5 17 12
```

## 带缓冲的通道
```go
// 默认情况下，通道是不带缓冲区的。发送端发送数据，同时必须又接收端相应的接收数据。
// 通道可以设置缓冲区，通过 make 的第二个参数指定缓冲区大小：
ch := make(chan int, 100)
```
```go
package main

import "fmt"

func main() {
    // 这里我们定义了一个可以存储整数类型的带缓冲通道
        // 缓冲区大小为2
        ch := make(chan int, 2)

        // 因为 ch 是带缓冲的通道，我们可以同时发送两个数据
        // 而不用立刻需要去同步读取数据
        ch <- 1
        ch <- 2

        // 获取这两个数据
        fmt.Println(<-ch)
        fmt.Println(<-ch)
}

//输出结果为：
1
2
```

## 通道用close来关闭
```go
//如果通道接收不到数据后 ok 就为 false，这时通道就可以使用 close() 函数来关闭
v, ok := <-ch
```
```go
package main

import (
        "fmt"
)

func fibonacci(n int, c chan int) {
        x, y := 0, 1
        for i := 0; i < n; i++ {
                c <- x
                x, y = y, x+y
        }
        close(c)
}

func main() {
        c := make(chan int, 10)
        go fibonacci(cap(c), c)
        // range 函数遍历每个从通道接收到的数据，因为 c 在发送完 10 个
        // 数据之后就关闭了通道，所以这里我们 range 函数在接收到 10 个数据
        // 之后就结束了。如果上面的 c 通道不关闭，那么 range 函数就不
        // 会结束，从而在接收第 11 个数据的时候就阻塞了。
        for i := range c {
                fmt.Println(i)
        }
}

//输出结果为：
0
1
1
2
3
5
8
13
21
34
```

# 切片
> 切片是对数组的描述, 一个数组可以有多个描述

用make创建切片的时候, `make([]type, length, capacity)`, 其中, length表示切片的大小, capacity表示切片底层array的大小; capacity容量够的话, append()不会产生新的更大的array

## 切片的append
arr从`[0 1 2 3 4 5 6 7]`变为`[0 1 2 3 4 5 6 10]`是因为`s3 := append(s2, 10)，s2=[5 6]`，再往后添加10的时候，把arr中的7变为了10。而后面再添加11、12时，因为已经超越了arr的cap，所以系统会重新分配更大的底层数组，而不再是对arr进行操作，原来的数组如果有人用就会依旧存在，如果没人用了就会自动垃圾回收
```go
package main

import "fmt"

func main() {
    arr := [...]int{0, 1, 2, 3, 4, 5, 6, 7}
    fmt.Println("arr: ", arr) // output：arr: [0 1 2 3 4 5 6 7]
    s1 := arr[2:6]
    s2 := s1[3:5] //其实s1的index只有0 1 2 3, 这里的3:5是3和4, 引用的是原始arr
    fmt.Println("s1: ", s1) // output：s1: [2 3 4 5]
    //len(s2)为2
    fmt.Println("s2: ", s2) // output：s2: [5 6]
    s3 := append(s2, 10)
    s4 := append(s3, 11)
    s5 := append(s4, 12)
    fmt.Println("s3: ", s3) // output：s3: [5 6 10]
    fmt.Println("s4: ", s4) // output：s4: [5 6 10 11]
    fmt.Println("s5: ", s5) // output：s5: [5 6 10 11 12]
    fmt.Println("arr: ", arr) // output：arr: [0 1 2 3 4 5 6 10]
}
```
当我们用append追加元素到切片时，如果容量不够，go就会创建一个新的切片变量, 并进行"深拷贝", 即把原来的array的元素值, 都拷贝到新的更大的array里面, 再append.
如果用量够用, 就不用"深拷贝"了
```go
func main() {
    var sa []string
    //用%p可以打印sa的地址, sa本来就是个
    fmt.Printf("addr:%p \t\tlen:%v content:%v\n",sa,len(sa),sa);
    for i:=0;i<10;i++{
        sa=append(sa,fmt.Sprintf("%v",i))
        fmt.Printf("addr:%p \t\tlen:%v content:%v\n",sa,len(sa),sa);
    }
    fmt.Printf("addr:%p \t\tlen:%v content:%v\n",sa,len(sa),sa);

}

---
Running ...
addr:0x0 len:0 content:[]
addr:0x1030e0c8 len:1 content:[0]
addr:0x10328120 len:2 content:[0 1]
addr:0x10322180 len:3 content:[0 1 2]
addr:0x10322180 len:4 content:[0 1 2 3]
addr:0x10342080 len:5 content:[0 1 2 3 4]
addr:0x10342080 len:6 content:[0 1 2 3 4 5]
addr:0x10342080 len:7 content:[0 1 2 3 4 5 6]
addr:0x10342080 len:8 content:[0 1 2 3 4 5 6 7]
addr:0x10324a00 len:9 content:[0 1 2 3 4 5 6 7 8]
addr:0x10324a00 len:10 content:[0 1 2 3 4 5 6 7 8 9]
addr:0x10324a00 len:10 content:[0 1 2 3 4 5 6 7 8 9]

//很明显，切片的地址经过了数次改变
```

# map 集合
Map 是一种无序的键值对的集合。Map 最重要的一点是通过 key 来快速检索数据，key 类似于索引，指向数据的值。
Map 是一种集合，所以我们可以像迭代数组和切片那样迭代它。不过，Map 是无序的，我们无法决定它的返回顺序，这是因为 Map 是使用 hash 表来实现的。
```go
/* 声明变量，默认 map 是 nil */
var map_variable map[key_data_type]value_data_type

/* 使用 make 函数 */
map_variable := make(map[key_data_type]value_data_type)
```

```go
package main

import "fmt"

func main() {
    var countryCapitalMap map[string]string /*创建集合 */
    countryCapitalMap = make(map[string]string)

    /* map插入key - value对,各个国家对应的首都 */
    countryCapitalMap [ "France" ] = "Paris"
    countryCapitalMap [ "Italy" ] = "罗马"
    countryCapitalMap [ "Japan" ] = "东京"
    countryCapitalMap [ "India " ] = "新德里"

    /*使用键输出地图值 */ for country := range countryCapitalMap {
        fmt.Println(country, "首都是", countryCapitalMap [country])
    }

    /*查看元素在集合中是否存在 */
    captial, ok := countryCapitalMap [ "美国" ] /*如果确定是真实的,则存在,否则不存在 */
    /*fmt.Println(captial) */
    /*fmt.Println(ok) */
    if (ok) {
        fmt.Println("美国的首都是", captial)
    } else {
        fmt.Println("美国的首都不存在")
    }
}
//运行结果
France 首都是 Paris
Italy 首都是 罗马
Japan 首都是 东京
India 首都是 新德里
美国的首都不存在
```

## delete可以删除元素

# range
用range可以对数组(array), 切片(slice), 通道(channel), 集合(map)进行遍历
```go
package main
import "fmt"
func main() {
    //这是我们使用range去求一个slice的和。使用数组跟这个很类似
    nums := []int{2, 3, 4}
    sum := 0
    for _, num := range nums {
        sum += num
    }
    fmt.Println("sum:", sum)
    //在数组上使用range将传入index和值两个变量。上面那个例子我们不需要使用该元素的序号，所以我们使用空白符"_"省略了。有时侯我们确实需要知道它的索引。
    for i, num := range nums {
        if num == 3 {
            fmt.Println("index:", i)
        }
    }
    //range也可以用在map的键值对上。
    kvs := map[string]string{"a": "apple", "b": "banana"}
    for k, v := range kvs {
        fmt.Printf("%s -> %s\n", k, v)
    }
    //range也可以用来枚举Unicode字符串。第一个参数是字符的索引，第二个是字符（Unicode的值）本身。
    for i, c := range "go" {
        fmt.Println(i, c)
    }
}

// 运行结果:
sum: 9
index: 1
a -> apple
b -> banana
0 103
1 111
```
