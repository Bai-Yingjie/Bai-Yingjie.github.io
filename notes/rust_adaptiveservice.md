[adaptiveservice](https://github.com/godevsig/adaptiveservice)是我用go写的微服务消息框架, 其核心之一是用了go的反射来给每个数据struct绑定一个handler方法.

```go
if st.svc.canHandle(tm.msg) {
    mm := &metaKnownMsg{
        stream: ss,
        msg:    tm.msg.(KnownMessage),
    }
    mq.putMetaMsg(mm)
} else {
    ss.privateChan <- tm.msg
}
```

在rust里面没有反射, 如何实现从字节流数据(stream buffer)到具体结构体的生成? 生成的数据结构体能否"断言"成实现了KnownMessage的trait?  

先看看rust从类型上提供了什么语义, 这些语义能干什么.

# 5种"动态"类型
```rust
use std::any::Any;

struct Header {
    uuid: u64,
    protocol: String
}

// 第一种, 静态类型, 泛型是静态分发的, 即在编译的时候就生成好了"类型化"的代码.
// statically typed, no pointer dereference
struct GenericPacket<T> {
    header: Header,
    data: T
}

// 第二种, 用Any，任何类型都实现了Any
// uses the "Any" type to have dynamic typing
struct AnyPacket {
    header: Header,
    data: Any,
}

// 第三种，用enum穷尽所有可能的类型
// uses an enum to capture the differnet possible types
enum DataEnum {
    Integer(i32),
    Float(f32)
}
struct EnumPacket {
    header: Header,
    data: DataEnum,
}


// 第四种: 用trait object
trait DataTrait {
    // interface your data conforms to
}
struct TraitPacket<'a> {
    header: Header,
    data: &'a dyn DataTrait,  // uses a pointer dereference to something that implements DataTrait
}

// 第五种: 和第一种类型类似, 但是带具体方法的trait
// statically typed, but will accept any type that conforms to DataTrait
struct StaticTraitPacket<T: DataTrait> {
    header: Header,
    data: T,
}
```

# rust的泛型是静态的
参考: https://www.cs.brandeis.edu/~cs146a/rust/doc-02-21-2015/book/static-and-dynamic-dispatch.html

比如下面的trait:
```rust
trait Foo {
    fn method(&self) -> String;
}

impl Foo for u8 {
    fn method(&self) -> String { format!("u8: {}", *self) }
}

impl Foo for String {
    fn method(&self) -> String { format!("string: {}", *self) }
}
```
## 静态分发
比如下面的代码:
```rust
fn do_something<T: Foo>(x: T) {
    x.method();
}

fn main() {
    let x = 5u8;
    let y = "Hello".to_string();

    do_something(x);
    do_something(y);
}
```
rust会在编译阶段就知道`do_something()`的入参类型, 会静态的生成类型下面的静态代码:
```rust
fn do_something_u8(x: u8) {
    x.method();
}

fn do_something_string(x: String) {
    x.method();
}

fn main() {
    let x = 5u8;
    let y = "Hello".to_string();

    do_something_u8(x);
    do_something_string(y);
}
```
虽然"静态分发"会有一定的代码"重复"带来代码段的膨胀, 但一般问题不大, 直接函数调用的性能会好点, 比如可以被inline优化

## 动态分发 即trait objects
Rust provides dynamic dispatch through a feature called `trait objects`. Trait objects, like `&Foo` or `Box<Foo>`, are normal values that store a value of _any_ type that implements the given trait, where the precise type can only be known at runtime. The methods of the trait can be called on a trait object via a special record of function pointers (created and managed by the compiler).

A function that takes a trait object is not specialised to each of the types that implements `Foo`: only one copy is generated, often (but not always) resulting in less code bloat. However, this comes at the cost of requiring slower virtual function calls, and effectively inhibiting any chance of inlining and related optimisations from occurring.

Trait objects are both simple and complicated: their core representation and layout is quite straight-forward, but there are some curly error messages and surprising behaviours to discover.

### 从指针获取trait objects
There's two similar ways to get a trait object value: casts and coercions. If `T` is a type that implements a trait `Foo` (e.g. `u8` for the `Foo` above), then the two ways to get a `Foo` trait object out of a pointer to `T` look like:

```rust
let ref_to_t: &T = ...;

// `as` keyword for casting
let cast = ref_to_t as &Foo;

// using a `&T` in a place that has a known type of `&Foo` will implicitly coerce:
let coerce: &Foo = ref_to_t;

fn also_coerce(_unused: &Foo) {}
also_coerce(ref_to_t);
```

These trait object coercions and casts also work for pointers like `&mut T` to `&mut Foo` and `Box<T>` to `Box<Foo>`, but that's all at the moment. Coercions and casts are identical.

This operation can be seen as "erasing" the compiler's knowledge about the specific type of the pointer, and hence trait objects are sometimes referred to "type erasure"
