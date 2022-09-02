- [主流的serde框架](#主流的serde框架)
  - [简单例子](#简单例子)
  - [serde_json](#serde_json)
    - [理论](#理论)
    - [untype的例子](#untype的例子)
    - [Index操作符重载](#index操作符重载)
    - [反序列化自定义struct](#反序列化自定义struct)
    - [json!宏构建json](#json宏构建json)
    - [序列化结构体](#序列化结构体)
    - [性能](#性能)
    - [依赖](#依赖)
  - [理论](#理论-1)
    - [29种类型](#29种类型)
    - [derive](#derive)
    - [属性](#属性)
    - [自己实现序列化](#自己实现序列化)
      - [序列化](#序列化)
      - [反序列化](#反序列化)
      - [反序列化的zero copy和生命周期标记](#反序列化的zero-copy和生命周期标记)

# 主流的serde框架
**Serde is a framework for _ser_ializing and _de_serializing Rust data structures efficiently and generically.**

## 简单例子
```rust
use serde::{Serialize, Deserialize};
extern crate serde_json; // 1.0.82

#[derive(Serialize, Deserialize, Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let point = Point { x: 1, y: 2 };

    // Convert the Point to a JSON string.
    let serialized = serde_json::to_string(&point).unwrap();

    // Prints serialized = {"x":1,"y":2}
    println!("serialized = {}", serialized);

    // Convert the JSON string back to a Point.
    let deserialized: Point = serde_json::from_str(&serialized).unwrap();

    // Prints deserialized = Point { x: 1, y: 2 }
    println!("deserialized = {:?}", deserialized);
}
```

## serde_json
### 理论
比如下面的json
```json
{
    "name": "John Doe",
    "age": 43,
    "address": {
        "street": "10 Downing Street",
        "city": "London"
    },
    "phones": [
        "+44 1234567",
        "+44 2345678"
    ]
}
```
json中的每个"value", 都是一个[`serde_json::Value`](https://docs.serde.rs/serde_json/value/enum.Value.html)
```rust
enum Value {
    Null,
    Bool(bool),
    Number(Number),
    String(String),
    Array(Vec<Value>),
    Object(Map<String, Value>),
}
```
* 用[`serde_json::from_str`](https://docs.serde.rs/serde_json/de/fn.from_str.html)从string里反序列化json
* 用[`from_slice`](https://docs.serde.rs/serde_json/de/fn.from_slice.html)从`&[u8]`反序列化json
* 用[`from_reader`](https://docs.serde.rs/serde_json/de/fn.from_reader.html)从文件或者TCP的socket的`io::Read`反序列化json

### untype的例子
所谓的untype就是说没有预定义好一个struct, 而是直接从JSON字符串里反序列化出一个对象, 这个对象的类型永远是`serde_json::Value`
```rust
use serde_json::{Result, Value};

fn untyped_example() -> Result<()> {
    // Some JSON input data as a &str. Maybe this comes from the user.
    let data = r#"
        {
            "name": "John Doe",
            "age": 43,
            "phones": [
                "+44 1234567",
                "+44 2345678"
            ]
        }"#;

    // Parse the string of data into serde_json::Value.
    let v: Value = serde_json::from_str(data)?;

    // Access parts of the data by indexing with square brackets.
    println!("Please call {} at the number {}", v["name"], v["phones"][0]);

    Ok(())
}

fn main() {
    untyped_example();
}

//结果
Please call "John Doe" at the number "+44 1234567"
```

rust的enum真是强大, 可以对`serde_json::Value`在运行时做各种操作, 比如`v["name"]`, 甚至`v["phones"][0]`都是合法的. -- 是因为`serde_json::Value`自己实现了`Index`操作符.
显然我们知道这些是合理的, 那如果故意用"不可能"的key去访问呢?

比如:
```rust
println!("Please call {} at the number {}", v["nnnnnnnnnnnn"], v["phones"][0][0][0]);

//能正常运行, 没有崩溃, 没有index越界, 结果为null
Please call null at the number null
```

如果最后整个打印v, 会得到:
```rust
println!("{:?}", v);

//结果
Object({"age": Number(43), "name": String("John Doe"), "phones": Array([String("+44 1234567"), String("+44 2345678")])})
```

### Index操作符重载
看起来v支持用index来访问, 是这个库有特殊的实现:
`https://github.com/serde-rs/json/blob/master/src/value/index.rs`
```rust
impl<I> ops::Index<I> for Value
where
    I: Index,
{
    ...
}
```
这个Index是下面:
关键先match v的类型, 再操作. 相当于操作符重载, 应该是rust比较高阶的用法.
```rust
impl Index for str {
    fn index_into<'v>(&self, v: &'v Value) -> Option<&'v Value> {
        match v {
            Value::Object(map) => map.get(self),
            _ => None,
        }
    }
    fn index_into_mut<'v>(&self, v: &'v mut Value) -> Option<&'v mut Value> {
        match v {
            Value::Object(map) => map.get_mut(self),
            _ => None,
        }
    }
    fn index_or_insert<'v>(&self, v: &'v mut Value) -> &'v mut Value {
        if let Value::Null = v {
            *v = Value::Object(Map::new());
        }
        match v {
            Value::Object(map) => map.entry(self.to_owned()).or_insert(Value::Null),
            _ => panic!("cannot access key {:?} in JSON {}", self, Type(v)),
        }
    }
}
```

自己实现的Index逻辑:
> The result of square bracket indexing like `v["name"]` is a borrow of the data at that index, so the type is `&Value`. A JSON map can be indexed with string keys, while a JSON array can be indexed with integer keys. If the type of the data is not right for the type with which it is being indexed, or if a map does not contain the key being indexed, or if the index into a vector is out of bounds, the returned element is `Value::Null`.

### 反序列化自定义struct
上面untype的例子中, 反序列化出来的对象只能是`serde_json::Value`, 虽然也可以按照json对号入座的访问里面的filed, 但实际是走的"通用"代码.
下面的例子可以直接返回一个strongly typed结构体:
```rust
use serde::{Deserialize, Serialize};
use serde_json::Result;

#[derive(Serialize, Deserialize)]
struct Person {
    name: String,
    age: u8,
    phones: Vec<String>,
}

fn typed_example() -> Result<()> {
    // Some JSON input data as a &str. Maybe this comes from the user.
    let data = r#"
        {
            "name": "John Doe",
            "age": 43,
            "phones": [
                "+44 1234567",
                "+44 2345678"
            ]
        }"#;

    // Parse the string of data into a Person object. This is exactly the
    // same function as the one that produced serde_json::Value above, but
    // now we are asking it for a Person as output.
    let p: Person = serde_json::from_str(data)?; //同样是from_str这个API, 会根据左侧变量的type来适配, 强大!

    // Do things just like with any other Rust data structure.
    println!("Please call {} at the number {}", p.name, p.phones[0]); //这里对name的访问也变成了p.name

    Ok(())
}
```

> This is the same `serde_json::from_str` function as before, but this time we assign the return value to a variable of type `Person` so Serde will automatically interpret the input data as a `Person` and produce informative error messages if the layout does not conform to what a `Person` is expected to look like.

> Any type that implements Serde's `Deserialize` trait can be deserialized this way. This includes built-in Rust standard library types like `Vec<T>` and `HashMap<K, V>`, as well as any structs or enums annotated with `#[derive(Deserialize)]`.

from_str()会返回不同的类型, 是因为它是个泛型函数:
```rust
pub fn from_str<'a, T>(s: &'a str) -> Result<T>
where
    T: de::Deserialize<'a>,
{
    //这里实际上是return from_trait(read::StrRead::new(s))
    //伴随着return, 返回类型T被传递进from_trait
    from_trait(read::StrRead::new(s))
}
```
这个泛型函数实例化类型T的传入是从返回值`Result<T>`推断出来的. 而这个返回值从变量类型而来:  
`let p: Person = serde_json::from_str(data)?`  
编译器推断出`T`就是`Person`

from_str继续把这个类型传递给from_trait.
```rust
fn from_trait<'de, R, T>(read: R) -> Result<T>
where
    R: Read<'de>,
    T: de::Deserialize<'de>,
{
    let mut de = Deserializer::new(read);
    let value = tri!(de::Deserialize::deserialize(&mut de));

    // Make sure the whole stream has been consumed.
    tri!(de.end());
    Ok(value)
}
```

### json!宏构建json
serde_json提供了`json!`宏来在代码里定义原始json文本数据.
```rust
use serde_json::json;

fn main() {
    // The type of `john` is `serde_json::Value`
    let john = json!({
        "name": "John Doe",
        "age": 43,
        "phones": [
            "+44 1234567",
            "+44 2345678"
        ]
    });

    println!("first phone number: {}", john["phones"][0]);

    // Convert to a string of JSON and print it out
    println!("{}", john.to_string());
}
```

json宏还提供更高级的功能: 引用变量
```rust
let full_name = "John Doe";
let age_last_year = 42;

// The type of `john` is `serde_json::Value`
let john = json!({
    "name": full_name,
    "age": age_last_year + 1, //这里可以引用上面的变量
    "phones": [
        format!("+44 {}", random_phone()) //也可以调用其他宏和函数
    ]
});
```
> One neat thing about the `json!` macro is that variables and expressions can be interpolated directly into the JSON value as you are building it. Serde will check at compile time that the value you are interpolating is able to be represented as JSON.

### 序列化结构体
* [`serde_json::to_string`](https://docs.serde.rs/serde_json/ser/fn.to_string.html) 序列化到string
* [`serde_json::to_vec`](https://docs.serde.rs/serde_json/ser/fn.to_vec.html)序列化到`Vec<u8>`
* [`serde_json::to_writer`](https://docs.serde.rs/serde_json/ser/fn.to_writer.html)序列化到文件或者TCP stream

> Any type that implements Serde's `Serialize` trait can be serialized this way. This includes built-in Rust standard library types like `Vec<T>` and `HashMap<K, V>`, as well as any structs or enums annotated with `#[derive(Serialize)]`.

### 性能
性能很好. 比最快的C实现还快
> It is fast. You should expect in the ballpark of 500 to 1000 megabytes per second deserialization and 600 to 900 megabytes per second serialization, depending on the characteristics of your data. This is competitive with the fastest C and C++ JSON libraries or even 30% faster for many use cases. Benchmarks live in the [serde-rs/json-benchmark](https://github.com/serde-rs/json-benchmark) repo.

### 依赖
只依赖内存allocator. 可以禁止其他default的feature, 只保留alloc
> Disable the default "std" feature and enable the "alloc" feature:
```toml
[dependencies]
serde_json = { version = "1.0", default-features = false, features = ["alloc"] }
```

## 理论
不同于其他使用发射来实现序列化的语言, serde用的是rust的trait.
实现了Serde's `Serialize` and `Deserialize`的struct都可以被序列化/反序列化.

### 29种类型
serde归纳了29种基础类型: 注意没有指针! 这点和tinygo不一样, tinygo用了反射, 而反射是支持指针的.
*   **14 primitive types**
    *   bool
    *   i8, i16, i32, i64, i128
    *   u8, u16, u32, u64, u128
    *   f32, f64
    *   char
*   **string**
    *   UTF-8 bytes with a length and no null terminator. May contain 0-bytes.
    *   When serializing, all strings are handled equally. When deserializing, there are three flavors of strings: transient, owned, and borrowed. This distinction is explained in [Understanding deserializer lifetimes](https://serde.rs/lifetimes.html) and is a key way that Serde enabled efficient zero-copy deserialization.
*   **byte array** - [u8]
    *   Similar to strings, during deserialization byte arrays can be transient, owned, or borrowed.
*   **option**
    *   Either none or some value.
*   **unit**
    *   The type of `()` in Rust. It represents an anonymous value containing no data.
*   **unit_struct**
    *   For example `struct Unit` or `PhantomData<T>`. It represents a named value containing no data.
*   **unit_variant**
    *   For example the `E::A` and `E::B` in `enum E { A, B }`.
*   **newtype_struct**
    *   For example `struct Millimeters(u8)`.
*   **newtype_variant**
    *   For example the `E::N` in `enum E { N(u8) }`.
*   **seq**
    *   A variably sized heterogeneous sequence of values, for example `Vec<T>` or `HashSet<T>`. When serializing, the length may or may not be known before iterating through all the data. When deserializing, the length is determined by looking at the serialized data. Note that a homogeneous Rust collection like `vec![Value::Bool(true), Value::Char('c')]` may serialize as a heterogeneous Serde seq, in this case containing a Serde bool followed by a Serde char.
*   **tuple**
    *   A statically sized heterogeneous sequence of values for which the length will be known at deserialization time without looking at the serialized data, for example `(u8,)` or `(String, u64, Vec<T>)` or `[u64; 10]`.
*   **tuple_struct**
    *   A named tuple, for example `struct Rgb(u8, u8, u8)`.
*   **tuple_variant**
    *   For example the `E::T` in `enum E { T(u8, u8) }`.
*   **map**
    *   A variably sized heterogeneous key-value pairing, for example `BTreeMap<K, V>`. When serializing, the length may or may not be known before iterating through all the entries. When deserializing, the length is determined by looking at the serialized data.
*   **struct**
    *   A statically sized heterogeneous key-value pairing in which the keys are compile-time constant strings and will be known at deserialization time without looking at the serialized data, for example `struct S { r: u8, g: u8, b: u8 }`.
*   **struct_variant**
    *   For example the `E::S` in `enum E { S { r: u8, g: u8, b: u8 } }`.
    
大多数的rust类型, 都能映射到serde类型:
* rust bool --> serde bool
* rust Rgb(u8,u8,u8) --> serde tuple struct
...

### derive
用`#[derive(Serialize, Deserialize)]`给每个结构体生成一对`Serialize` and `Deserialize` traits.

### 属性
有很多实用的属性, 比如:
```rust
#[serde(rename = "name")]
#[serde(rename_all = "...")] //比如UPPERCASE, camelCase...等等
#[serde(default)]
#[serde(alias = "name")]
#[serde(flatten)]
...很多...
```

### 自己实现序列化
一般用`#[derive(Serialize, Deserialize)]`配合[attributes](https://serde.rs/attributes.html)就够了. 但特殊case如果想自定义序列化, 可以自己实现[`Serialize`](https://docs.serde.rs/serde/ser/trait.Serialize.html) and [`Deserialize`](https://docs.serde.rs/serde/de/trait.Deserialize.html)这两个trait

每个trait都只有一个方法
```rust
pub trait Serialize {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer;
}

pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>;
}
```

#### 序列化
```rust
pub trait Serialize {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer;
}
```
This method's job is to take your type (`&self`) and map it into the [Serde data model](https://serde.rs/data-model.html) by invoking exactly one of the methods on the given [`Serializer`](https://docs.serde.rs/serde/ser/trait.Serializer.html).

Serializer也是个trait, 这个trait是不同类型的format(比如json, Bincode等). 并不是所有的序列化输出都是text或者bin的, 比如`serde_json::value::Serializer`(注意, 不是`serde_json::serializer`)就序列化到内存.

serde把结构体的到29种内部数据类型抽象成`trait Serialize`, 而把29种数据类型到输出format, 抽象成`trait Serializer`, 让二者解耦: 从结构体到29种数据类型OK, 就和最后的output格式无关. 这是个很巧妙的设计.

[`Serialize`](https://docs.rs/serde/latest/serde/trait.Serialize.html) ==> 29种内部数据类型 ==> [`Serializer`](https://docs.serde.rs/serde/ser/trait.Serializer.html)

比如序列化一个Map
```rust
impl<K, V> Serialize for MyMap<K, V>
where
    K: Serialize,
    V: Serialize,
{
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let mut map = serializer.serialize_map(Some(self.len()))?;
        for (k, v) in self { //因为知道self就是个map, 所以能用for in
            map.serialize_entry(k, v)?;
        }
        map.end()
    }
}
```

结构体有4类: 普通结构体, tuple结构体, newtype, unit结构体
```rust
// An ordinary struct. Use three-step process:
//   1. serialize_struct
//   2. serialize_field
//   3. end
struct Color {
    r: u8,
    g: u8,
    b: u8,
}

// A tuple struct. Use three-step process:
//   1. serialize_tuple_struct
//   2. serialize_field
//   3. end
struct Point2D(f64, f64);

// A newtype struct. Use serialize_newtype_struct.
struct Inches(u64);

// A unit struct. Use serialize_unit_struct.
struct Instance;
```

#### 反序列化
```rust
pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>;
}
```
> This method's job is to map the type into the [Serde data model](https://serde.rs/data-model.html) by providing the [`Deserializer`](https://docs.serde.rs/serde/trait.Deserializer.html) with a [`Visitor`](https://docs.serde.rs/serde/de/trait.Visitor.html) that can be driven by the `Deserializer` to construct an instance of your type.

`Deserializer`需要deserialize传入visitor trait.

反序列化比序列化更复杂点. 大体上根据序列化的format不同, 可以分为:
* 自解释的文本. 比如json, 序列化后的文本就能看出来对应的结构体.
这个情况可以用通用的反序列化`apideserialize_any`, 产生的对象是`serde_json::Value`
* 非自解释的bincode. 比如二进制的序列化方式, 只看output是不可能知道原始结构体的layout的.
这个情况需要编程时明确目标结构体的类型, 产生的对象是非`serde_json::Value`

#### 反序列化的zero copy和生命周期标记
rust能做到zero copy是因为生成的目标结构体, 能引用原始buffer里面的比如string等类型, 不用拷贝. 因为有生命周期标记.
比如:
```rust
#[derive(Deserialize)]
struct User<'a> {
    id: u32,
    name: &'a str,
    screen_name: &'a str,
    location: &'a str,
}
```
> Zero-copy deserialization means deserializing into a data structure, like the `User` struct above, that borrows string or byte array data from the string or byte array holding the input. This avoids allocating memory to store a string for each individual field and then copying string data out of the input over to the newly allocated field. Rust guarantees that the input data outlives the period during which the output data structure is in scope, meaning it is impossible to have dangling pointer errors as a result of losing the input data while the output data structure still refers to it.

上面的`User`结构体有多个对str的引用, 这些引用直接引用到原始buffer, 没有拷贝. rust保证原始buffer会一直有效直到这个结构体的生命周期结束.
注: `User`结构体本身不用生命周期标记, 它的标记`'a`是给里面的成员用的, 表示所有成员都有同样的生命周期.

反序列化的复杂点在于, 目标结构体要分配内存:
* **`<'de, T> where T: Deserialize<'de>`**
其引用的内存可能来自原始input的buffer中, 比如`serde_json::from_str`中, 原始str的生命周期也是caller提供的, 带生命周期标记的, 就可以被最终的目标结构体引用.
>This means "T can be deserialized from **some** lifetime." The caller gets to decide what lifetime that is. Typically this is used when the caller also provides the data that is being deserialized from, for example in a function like [`serde_json::from_str`](https://docs.serde.rs/serde_json/fn.from_str.html). In that case the input data must also have lifetime `'de`, for example it could be `&'de str`.

* **`<T> where T: DeserializeOwned`**
原始的input, 以及其伴随的buffer, 在反序列化之后会被free. 此时不能引用. 比如io的reader
>This means "T can be deserialized from **any** lifetime." The callee gets to decide what lifetime. Usually this is because the data that is being deserialized from is going to be thrown away before the function returns, so T must not be allowed to borrow from it. For example a function that accepts base64-encoded data as input, decodes it from base64, deserializes a value of type T, then throws away the result of base64 decoding. Another common use of this bound is functions that deserialize from an IO stream, such as [`serde_json::from_reader`](https://docs.serde.rs/serde_json/fn.from_reader.html).