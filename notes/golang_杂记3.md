- [go按位取反(bitwise not)](#go按位取反bitwise-not)
- [go的相等性(==)](#go的相等性)
  - [普通类型的比较](#普通类型的比较)
  - [指针的相等性](#指针的相等性)
  - [channel的相等性](#channel的相等性)
  - [interface的相等性](#interface的相等性)
  - [结构体的相等性](#结构体的相等性)
  - [Array的相等性](#array的相等性)
  - [string](#string)
  - [[]byte用bytes.Equal比较](#byte用bytesequal比较)
  - [reflect.DeepEqual万能比较](#reflectdeepequal万能比较)
  - [cmp包](#cmp包)

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
