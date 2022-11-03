go1.18引入了泛型的语法.

- [类型的属性](#类型的属性)
  - [底层类型 Underlying types](#底层类型-underlying-types)
  - [核心类型 core type](#核心类型-core-type)
    - [有core type的interface举例](#有core-type的interface举例)
    - [没有core type的interface举例](#没有core-type的interface举例)
    - [bytestring特殊core type](#bytestring特殊core-type)
- [interface类型](#interface类型)
  - [带方法的interface](#带方法的interface)
  - [interface可以被嵌入其他interface](#interface可以被嵌入其他interface)
  - [泛型interface](#泛型interface)
- [泛型](#泛型)
  - [类型约束 type constraint](#类型约束-type-constraint)
  - [实例化](#实例化)
- [泛型举例](#泛型举例)
  - [使用泛型前](#使用泛型前)
  - [使用泛型后](#使用泛型后)
  - [使用类型约束让代码更可读](#使用类型约束让代码更可读)

# 类型的属性
参考: https://go.dev/ref/spec#Type_parameter_declarations

## 底层类型 Underlying types
每个类型T都有一个`底层类型`  
> Each type T has an underlying type: If T is one of the predeclared boolean, numeric, or string types, or a type literal, the corresponding underlying type is T itself. Otherwise, T's underlying type is the underlying type of the type to which T refers in its declaration. For a type parameter that is the underlying type of its type constraint, which is always an interface.

```go
type (
	A1 = string     //底层类型是string
	A2 = A1         //底层类型是string
)

type (
	B1 string       //底层类型是string
	B2 B1           //底层类型是string
	B3 []B1         //底层类型是[]B1
	B4 B3           //底层类型是[]B1
)

func f[P any](x P) { … }    //P的底层类型是interface{}
```

在比如`~[]byte`表示底层类型是`[]byte`

## 核心类型 core type
* 每个非interface类型T都有个`核心类型`, 就是T的`底层类型`
* 不是所有的interface T都有`核心类型`, 有核心类型的T要满足下面任意一项:
  * 如果T的type set只有一个底层类型U, 那么T有核心类型为U
  * 如果T的type set只有相同方向的channel类型, channel的元素是E, 那么T的核心类型是`chan E`, 或者带方向的`chan<- E` or `<-chan E`
* 其他的interface都没有核心类型

根据以上规则, interface的核心类型不可能是自定义类型, 或者其他的interface类型.

### 有core type的interface举例
```go
type Celsius float32
type Kelvin  float32

interface{ int }                          // int
interface{ Celsius|Kelvin }               // float32
interface{ ~chan int }                    // chan int
interface{ ~chan int|~chan<- int }        // chan<- int
interface{ ~[]*data; String() string }    // []*data
```

### 没有core type的interface举例
```go
interface{}                               // no single underlying type
interface{ Celsius|float64 }              // no single underlying type
interface{ chan int | chan<- string }     // channels have different element types
interface{ <-chan int | chan<- int }      // directional channels have different directions
```

### bytestring特殊core type
> if there are exactly two types, []byte and string, which are the underlying types of all types in the type set of interface T, the core type of T is called bytestring.

```go
interface{ int }                          // int (same as ordinary core type)
interface{ []byte | string }              // bytestring
interface{ ~[]byte | myString }           // bytestring
```

# interface类型
> An interface type defines a type set. A variable of interface type can store a value of any type that is in the type set of the interface. Such a type is said to implement the interface. The value of an uninitialized variable of interface type is nil.

就是说一个interface类型实际上表示了一个类型的集合. 一个interface变量就是这个集合中的一个类型的实例. 这个集合中所有的类型都实现了这个interface的要求.

```
InterfaceType  = "interface" "{" { InterfaceElem ";" } "}" .
InterfaceElem  = MethodElem | TypeElem .
MethodElem     = MethodName Signature .
MethodName     = identifier .
TypeElem       = TypeTerm { "|" TypeTerm } .
TypeTerm       = Type | UnderlyingType .
UnderlyingType = "~" Type .
```

注意看上面的interface声明的语法:
* interface不仅可以包含常见的的方法(MethodElem), 还可以包括类型(TypeElem)
* 类型可以用`|`表示或
* `~`表示底层类型

> An interface type is specified by a list of interface elements. An interface element is either a method or a type element, where a type element is a union of one or more type terms. A type term is either a single type or a single underlying type.

## 带方法的interface
最常见的interface是带方法约束的(可以为空), 比如:
```go
// A simple File interface.
interface {
	Read([]byte) (int, error)
	Write([]byte) (int, error)
	Close() error
}
```
## interface可以被嵌入其他interface
比如
```go
type Reader interface {
	Read(p []byte) (n int, err error)
	Close() error
}

type Writer interface {
	Write(p []byte) (n int, err error)
	Close() error
}

// ReadWriter's methods are Read, Write, and Close.
type ReadWriter interface {
	Reader  // includes methods of Reader in ReadWriter's method set
	Writer  // includes methods of Writer in ReadWriter's method set
}
```

## 泛型interface
1.18引入了泛型的概念, 使得interface的概念从方法的约束扩展到了类型的约束, 从而interface可以表达一个类型的集合

比如`interface {int}`就表示只包括int类型的集合;  
再比如`interface {~int}`就表示底层类型是int的集合, 这个集合就比上面的大.

现在interface可以包括方法, 或类型, 甚至方法和类型的混合, 比如下面的例子表示一个实现了String()方法的底层类型是int的类型
```go
// An interface representing all types with underlying type int that implement the String method.
interface {
	~int
	String() string
}
```

interface体里的约束是与的关系, 所以下面的例子表示一个空的集合, 因为没有一个类型既是int, 又是string
```go
// An interface representing an empty type set: there is no type that is both an int and a string.
interface {
	int
	string
}
```

汇总如下:
```go
// An interface representing only the type int.
interface {
	int
}

// An interface representing all types with underlying type int.
interface {
	~int
}

// An interface representing all types with underlying type int that implement the String method.
interface {
	~int
	String() string
}

// An interface representing an empty type set: there is no type that is both an int and a string.
interface {
	int
	string
}
```

可以使用`|`表示或, 比如下面的例子表示所有底层类型是float32或float64的类型:
```go
// The Float interface represents all floating-point types
// (including any named types whose underlying types are
// either float32 or float64).
type Float interface {
	~float32 | ~float64
}
```

# 泛型
比如下面的泛型函数:
```go
func min[T ~int|~float64](x, y T) T {
	if x < y {
		return x
	}
	return y
}
```

函数名后面的方括号里面是类型参数type parameter. 比如下面的都是类型参数:
```go
[P any]
[S interface{ ~[]byte|string }]
[S ~[]E, E any]
[P Constraint[int]]
[_ any]
```
这里面的`any`, `interface{ ~[]byte|string }`学名叫做`TypeConstraint`  
完整的形式应该是`interface{...}`, 其中`interface`有时候可以省略, 比如:
```go
[T []P]                      // = [T interface{[]P}]
[T ~int]                     // = [T interface{~int}]
[T int|string]               // = [T interface{int|string}]
type Constraint ~int         // illegal: ~int is not inside a type parameter list
```

## 类型约束 type constraint
下面的Number就是个类型约束:
```go
type Number interface {
    int64 | float64
}
```

## 实例化
泛型的函数在使用的时候必须实例化, 比如下面的`f := min`就不合法, 因为min此时min并不知道入参类型, 无法实例化.  
编译器会根据实际的入参类型来推断并实例化泛型函数
```go
func min[T ~int|~float64](x, y T) T { … }

f := min                   // illegal: min must be instantiated with type arguments when used without being called
minInt := min[int]         // minInt has type func(x, y int) int
a := minInt(2, 3)          // a has value 2 of type int
b := min[float64](2.0, 3)  // b has value 2.0 of type float64
c := min(b, -1)            // c has value -1.0 of type float64
```

再比如下面的泛型函数非常典型, 对slice S中的每个元素做apply操作.
```go
func apply[S ~[]E, E any](s S, f(E) E) S { … }

f0 := apply[]                  // illegal: type argument list cannot be empty
f1 := apply[[]int]             // type argument for S explicitly provided, type argument for E inferred
f2 := apply[[]string, string]  // both type arguments explicitly provided

var bytes []byte
r := apply(bytes, func(byte) byte { … })  // both type arguments inferred from the function arguments
```

# 泛型举例
## 使用泛型前
```go
// SumInts adds together the values of m.
func SumInts(m map[string]int64) int64 {
    var s int64
    for _, v := range m {
        s += v
    }
    return s
}

// SumFloats adds together the values of m.
func SumFloats(m map[string]float64) float64 {
    var s float64
    for _, v := range m {
        s += v
    }
    return s
}
```

## 使用泛型后
```go
// SumIntsOrFloats sums the values of map m. It supports both int64 and float64
// as types for map values.
func SumIntsOrFloats[K comparable, V int64 | float64](m map[K]V) V {
    var s V
    for _, v := range m {
        s += v
    }
    return s
}
```
调用`SumIntsOrFloats`的时候, 必须让编译器知道如何实例化.
* 可以显式的指定入参类型: 比如`SumIntsOrFloats[string, int64](ints)`, 或者`SumIntsOrFloats[string, float64](floats))`
* 也可以让编译器推断: `SumIntsOrFloats(ints)`, 或者`SumIntsOrFloats(floats)`

## 使用类型约束让代码更可读
```go
type Number interface {
    int64 | float64
}

// SumNumbers sums the values of map m. It supports both integers
// and floats as map values.
func SumNumbers[K comparable, V Number](m map[K]V) V {
    var s V
    for _, v := range m {
        s += v
    }
    return s
}
```
