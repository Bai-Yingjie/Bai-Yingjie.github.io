- [structopt](#structopt)
- [log](#log)
  - [env\_logger](#env_logger)
- [常用宏1](#常用宏1)
- [常用宏2](#常用宏2)
  - [由编译器实现的builtin宏](#由编译器实现的builtin宏)
- [trait Iterator](#trait-iterator)
  - [比较常用的Iterator方法](#比较常用的iterator方法)
  - [collect](#collect)
  - [IntoIterator](#intoiterator)
- [数组泛型的方法`impl<T> [T]`](#数组泛型的方法implt-t)
- [Vec](#vec)
- [Option](#option)
- [Result](#result)
  - [Result\<T, E\>的方法:](#resultt-e的方法)
  - [Result\<\&T, E\>的方法](#resultt-e的方法-1)
  - [其他Result](#其他result)
  - [Result实现了如下的trait](#result实现了如下的trait)
- [env](#env)
- [取消impl trait](#取消impl-trait)
- [文件](#文件)
  - [使用举例](#使用举例)
    - [写文件](#写文件)
    - [读文件到String](#读文件到string)
    - [更有效率的读文件](#更有效率的读文件)
    - [不同选项open](#不同选项open)
    - [`&File`也能modify 文件](#file也能modify-文件)
  - [File对象](#file对象)
  - [常用函数](#常用函数)
  - [File的方法](#file的方法)
  - [其他impl的trait](#其他impl的trait)
  - [实现Read trait](#实现read-trait)
  - [实现Write trait](#实现write-trait)
  - [实现Seek trait](#实现seek-trait)
  - [Read Write Seek for \&File又来一遍!!!!!](#read-write-seek-for-file又来一遍)
  - [open最后调用](#open最后调用)
- [IO](#io)
  - [常用函数](#常用函数-1)
  - [Read trait](#read-trait)
  - [Write trait](#write-trait)
  - [Seek trait](#seek-trait)
  - [BufRead是Read的派生trait](#bufread是read的派生trait)

# [structopt](https://docs.rs/structopt/0.3.26/structopt/#)
structopt用来把命令行参数转成结构体
定义结构体的时候, 用structopt来"标记", 比如:
```rust
#[derive(Clone, Debug, StructOpt)]
#[structopt(name = "virtiofsd backend", about = "Launch a virtiofsd backend.")]
struct Opt {
    /// Shared directory path
    #[structopt(long)]
    shared_dir: Option<String>,

    /// vhost-user socket path [deprecated]
    #[structopt(long, required_unless_one = &["fd", "socket-path", "print-capabilities"])]
    socket: Option<String>,

    /// vhost-user socket path
    #[structopt(long = "socket-path", required_unless_one = &["fd", "socket", "print-capabilities"])]
    socket_path: Option<String>,
    ...
}
```
* 三斜线的注释会变成`--help`里面的option的说明, 输出格式还挺整齐的. `-h`的显示会更简洁一些, 只包括注释的第一行.
* 变量下划线会变成中横线, 比如`shared_dir`会在`--help`下显示`--shared-dir <shared-dir>                    Shared directory path`

使用的时候, 比如在main里面:
```rust
fn main() {
    let opt = Opt::from_args()
}
```

注: structopt已经停止开发, 建议使用clap: **Command Line Argument Parser for Rust**

# log
https://crates.io/crates/log
要在cargo.toml里面声明依赖:
```toml
[dependencies]
log = "0.4"
```
这个库设计的很合理.
对用户提供几个log的宏:
[`error!`](https://docs.rs/log/latest/log/macro.error.html), [`warn!`](https://docs.rs/log/latest/log/macro.warn.html), [`info!`](https://docs.rs/log/latest/log/macro.info.html), [`debug!`](https://docs.rs/log/latest/log/macro.debug.html) and [`trace!`](https://docs.rs/log/latest/log/macro.trace.html)

* lib里面, 只使用这几个输出宏
* bin里面, 负责初始化后端的logging实现, 如果没有初始化, 那上面的几个输出宏就类似noop
    * 使用[set_boxed_logger](https://docs.rs/log/latest/log/fn.set_boxed_logger.html "log::set_boxed_logger fn") Sets the global logger to a `Box<Log>`.
    * 或[set_logger](https://docs.rs/log/latest/log/fn.set_logger.html "log::set_logger fn") Sets the global logger to a `&'static Log`.

可选的logging实现有:
*   Simple minimal loggers:
    *   [env_logger](https://docs.rs/env_logger/*/env_logger/)
    *   [simple_logger](https://github.com/borntyping/rust-simple_logger)
    *   [simplelog](https://github.com/drakulix/simplelog.rs)
    *   [pretty_env_logger](https://docs.rs/pretty_env_logger/*/pretty_env_logger/)
    *   [stderrlog](https://docs.rs/stderrlog/*/stderrlog/)
    *   [flexi_logger](https://docs.rs/flexi_logger/*/flexi_logger/)
*   Complex configurable frameworks:
    *   [log4rs](https://docs.rs/log4rs/*/log4rs/)
    *   [fern](https://docs.rs/fern/*/fern/)
*   Adaptors for other facilities:
    *   [syslog](https://docs.rs/syslog/*/syslog/)
    *   [slog-stdlog](https://docs.rs/slog-stdlog/*/slog_stdlog/)
    *   [systemd-journal-logger](https://docs.rs/systemd-journal-logger/*/systemd_journal_logger/)
    *   [android_log](https://docs.rs/android_log/*/android_log/)
    *   [win_dbg_logger](https://docs.rs/win_dbg_logger/*/win_dbg_logger/)
    *   [db_logger]
*   For WebAssembly binaries:
    *   [console_log](https://docs.rs/console_log/*/console_log/)
*   For dynamic libraries:
    *   You may need to construct an FFI-safe wrapper over `log` to initialize in your libraries
    
## env_logger
比如这样在代码里:
```rust
use log::{debug, error, log_enabled, info, Level};

env_logger::init();

debug!("this is a debug {}", "message");
error!("this is printed by default");

if log_enabled!(Level::Info) {
    let x = 3 * 4; // expensive computation
    info!("the answer was: {}", x);
}
```
使用时:
```sh
$ RUST_LOG=debug ./main
[2017-11-09T02:12:24Z DEBUG main] this is a debug message
[2017-11-09T02:12:24Z ERROR main] this is printed by default
[2017-11-09T02:12:24Z INFO main] the answer was: 12
```

可以按module来指定level
```sh
$ RUST_LOG=main=info ./main
[2017-11-09T02:12:24Z ERROR main] this is printed by default
[2017-11-09T02:12:24Z INFO main] the answer was: 12
```

又比如在virtiofsd里面是这样用的:
```rust
fn set_default_logger(log_level: LevelFilter) {
    if env::var("RUST_LOG").is_err() {
        env::set_var("RUST_LOG", log_level.to_string());
    }
    env_logger::init();
}
```

# 常用宏1
代码在`lib/rustlib/src/rust/library/std/src/macros.rs`
* panic
* print
* println
* eprint
* eprintln
* dbg

# 常用宏2
代码在`lib/rustlib/src/rust/library/core/src/macros/mod.rs`
* panic!
```rust
panic!();
panic!("this is a {} {message}", "fancy", message = "message");
```
* assert_eq! assert_ne!
```rust
assert_eq!(a, b); // a b是两个表达式
assert_ne!(a, b);
```
* assert_matches!
```rust
assert_matches!(a, Some(_));
assert_matches!(b, None);
let c = Ok("abc".to_string());
assert_matches!(c, Ok(x) | Err(x) if x.len() < 100);
```
* debug_assert! debug_assert_eq! debug_assert_ne! 只有在debug版本里才使能
```rust
debug_assert!(true);
```
* matches!
```rust
let foo = 'f';
assert!(matches!(foo, 'A'..='Z' | 'a'..='z'));
let bar = Some(4);
assert!(matches!(bar, Some(x) if x > 2));
```
* ?和r#try!  
try!是个宏, 但需要用raw方式来调用, `r#try`. 现在可以用?来代替
```rust
enum MyError {
    FileWriteError
}
impl From<io::Error> for MyError {
    fn from(e: io::Error) -> MyError {
        MyError::FileWriteError
    }
}
// The preferred method of quick returning Errors
fn write_to_file_question() -> Result<(), MyError> {
    let mut file = File::create("my_best_friends.txt")?;
    file.write_all(b"This is a list of my best friends.")?;
    Ok(())
}
// The previous method of quick returning Errors
fn write_to_file_using_try() -> Result<(), MyError> {
    let mut file = r#try!(File::create("my_best_friends.txt"));
    r#try!(file.write_all(b"This is a list of my best friends."));
    Ok(())
}
```
* write! writeln! 写入buffer
```rust
fn main() -> std::io::Result<()> {
    let mut w = Vec::new();
    write!(&mut w, "test")?;
    write!(&mut w, "formatted {}", "arguments")?;
    assert_eq!(w, b"testformatted arguments");
    Ok(())
}
let mut s = String::new();
writeln!(&mut s, "{} {}", "abc", 123)?; // uses fmt::Write::write_fmt
```
* unreachable! unimplemented! todo! 代码实现阶段用到的宏

## 由编译器实现的builtin宏
* compile_error!
```rust
#[cfg(not(any(feature = "foo", feature = "bar")))]
compile_error!("Either feature \"foo\" or \"bar\" must be enabled for this crate.");
```
* format_args! const_format_args! format_args_nl! 格式化宏, 用于format!宏
* env! option_env! 在编译时获取env, 注意不是运行时
```rust
let path: &'static str = env!("PATH");
let key: Option<&'static str> = option_env!("SECRET_KEY");
```
* concat_idents! 多个标识符连起来成为一个
```rust
fn foobar() -> u32 { 23 }
let f = concat_idents!(foo, bar);
println!("{}", f());
```
* concat_bytes! 连接字符
* concat!
```rust
let s = concat!("test", 10, 'b', true);
assert_eq!(s, "test10btrue");
```
* line! column! file! 编译的文件, 行号等; module_path! module路径
```rust
let current_line = line!();
println!("defined on line: {}", current_line);
```
* stringify!
```rust
let one_plus_one = stringify!(1 + 1);
assert_eq!(one_plus_one, "1 + 1");
```
* include_str! 编译时从文件读入string; include_bytes! 编译时从文件读入bytes; 文件是相对当前编译文件的路径.
```rust
//spanish.in里面是adiós
let my_str = include_str!("spanish.in");
assert_eq!(my_str, "adiós\n");
```
* cfg!
```rust
let my_directory = if cfg!(windows) {
    "windows-specific-directory"
} else {
    "unix-directory"
};
```
* include! 把文件导入进来按表达式来编译
* assert!
* derive!
* test
* bench
* global_allocator
* cfg_accessible

# trait Iterator
看起来只要实现了next就是个Iterator了... 其他都有默认实现, 真方便
```rust
pub trait Iterator {
    type Item;
    
    //需要用户实现的:
    fn next(&mut self) -> Option<Self::Item>;
    
    //有默认实现的:
    fn size_hint(&self) -> (usize, Option<usize>) //默认返回0, 用户要自己实现更适合自己的方法.
    fn count(self) -> usize
    fn last(self) -> Option<Self::Item>
    fn advance_by(&mut self, n: usize) -> Result<(), usize>
    fn nth(&mut self, n: usize) -> Option<Self::Item>
    fn step_by(self, step: usize) -> StepBy<Self>
    fn chain<U>(self, other: U) -> Chain<Self, U::IntoIter>
    fn zip<U>(self, other: U) -> Zip<Self, U::IntoIter>
    fn intersperse(self, separator: Self::Item) -> Intersperse<Self>
    fn intersperse_with<G>(self, separator: G) -> IntersperseWith<Self, G>
    fn map<B, F>(self, f: F) -> Map<Self, F>
    fn for_each<F>(self, f: F)
    fn filter<P>(self, predicate: P) -> Filter<Self, P>
    fn filter_map<B, F>(self, f: F) -> FilterMap<Self, F>
    fn enumerate(self) -> Enumerate<Self> //还是返回一个Iterator, 元素是(index, value)
    fn peekable(self) -> Peekable<Self>
    fn skip_while<P>(self, predicate: P) -> SkipWhile<Self, P>
    fn take_while<P>(self, predicate: P) -> TakeWhile<Self, P>
    fn map_while<B, P>(self, predicate: P) -> MapWhile<Self, P>
    fn skip(self, n: usize) -> Skip<Self>
    fn take(self, n: usize) -> Take<Self>
    fn scan<St, B, F>(self, initial_state: St, f: F) -> Scan<Self, St, F>
    fn flat_map<U, F>(self, f: F) -> FlatMap<Self, U, F>
    fn flatten(self) -> Flatten<Self>
    fn fuse(self) -> Fuse<Self>
    fn inspect<F>(self, f: F) -> Inspect<Self, F>
    fn by_ref(&mut self) -> &mut Self
    fn collect<B: FromIterator<Self::Item>>(self) -> B
    fn try_collect<B>(&mut self) -> ChangeOutputType<Self::Item, B>
    fn partition<B, F>(self, f: F) -> (B, B)
    fn partition_in_place<'a, T: 'a, P>(mut self, ref mut predicate: P) -> usize
    fn is_partitioned<P>(mut self, mut predicate: P) -> bool
    fn try_fold<B, F, R>(&mut self, init: B, mut f: F) -> R
    fn try_for_each<F, R>(&mut self, f: F) -> R
    fn fold<B, F>(mut self, init: B, mut f: F) -> B
    fn reduce<F>(mut self, f: F) -> Option<Self::Item>
    fn try_reduce<F, R>(&mut self, f: F) -> ChangeOutputType<R, Option<R::Output>>
    fn all<F>(&mut self, f: F) -> bool
    fn any<F>(&mut self, f: F) -> bool
    fn find<P>(&mut self, predicate: P) -> Option<Self::Item>
    fn find_map<B, F>(&mut self, f: F) -> Option<B>
    fn try_find<F, R>(&mut self, f: F) -> ChangeOutputType<R, Option<Self::Item>>
    fn position<P>(&mut self, predicate: P) -> Option<usize>
    fn rposition<P>(&mut self, predicate: P) -> Option<usize>
    fn max(self) -> Option<Self::Item>
    fn min(self) -> Option<Self::Item>
    fn max_by_key<B: Ord, F>(self, f: F) -> Option<Self::Item>
    fn max_by<F>(self, compare: F) -> Option<Self::Item>
    fn min_by_key<B: Ord, F>(self, f: F) -> Option<Self::Item>
    fn min_by<F>(self, compare: F) -> Option<Self::Item>
    fn rev(self) -> Rev<Self>
    fn unzip<A, B, FromA, FromB>(self) -> (FromA, FromB)
    fn copied<'a, T: 'a>(self) -> Copied<Self>
    fn cloned<'a, T: 'a>(self) -> Cloned<Self>
    fn cycle(self) -> Cycle<Self>
    fn sum<S>(self) -> S
    fn product<P>(self) -> P
    fn cmp<I>(self, other: I) -> Ordering
    fn cmp_by<I, F>(mut self, other: I, mut cmp: F) -> Ordering
    fn partial_cmp<I>(self, other: I) -> Option<Ordering>
    fn partial_cmp_by<I, F>(mut self, other: I, mut partial_cmp: F) -> Option<Ordering>
    fn eq<I>(self, other: I) -> bool
    fn eq_by<I, F>(mut self, other: I, mut eq: F) -> bool
    fn ne<I>(self, other: I) -> bool
    fn lt<I>(self, other: I) -> bool
    fn le<I>(self, other: I) -> bool
    fn gt<I>(self, other: I) -> bool
    fn ge<I>(self, other: I) -> bool
    fn is_sorted(self) -> bool
    fn is_sorted_by<F>(mut self, compare: F) -> bool
    fn is_sorted_by_key<F, K>(self, f: F) -> bool
}
```

## 比较常用的Iterator方法
* filter: 对Self的关联类型的借用`&Self::Item`调用闭包函数, 返回另一个Iterator
* map: 也是返回另一个Iterator
* 最后collect: 把Iterator"重组"成一个collect对象.

## collect
filter在前面讲过. 这里看一下collect:

collect基础用法如下:
```rust
let a = [1, 2, 3];

let doubled: Vec<i32> = a.iter()
                         .map(|&x| x * 2)
                         .collect();

assert_eq!(vec![2, 4, 6], doubled);
```
注意, 目标变量doubled需要显式指定类型, 要不然collect不知道你要"重组"成什么样的collect对象.
常用的就是collect成Vec.

下面是Iterator的默认collect实现:
```rust
//这里面很晦涩, collect返回一个泛型B, 这个B是要满足`FromIterator<Self::Item>`即Iterator的关联类型实例化的`FromIterator`
//这个B是编译器自己推断的, 或者根据左值(比如上面的let doubled: Vec<i32>), 或者用户指定, 比如更上面的.collect::<Vec<_>>();
fn collect<B: FromIterator<Self::Item>>(self) -> B
where
    Self: Sized,
{
    //这里trait名称::trait函数这个调用方式看起来无比奇怪, 有点自己调用自己的意思
    //但我理解下面的FromIterator已经是个具体的类型, 由编译器自动推导出来的:
    //比如上面的左值let doubled: Vec<i32>, 到这里就应该是调用Vec<i32>的from_iter
    FromIterator::from_iter(self)
}

//这里是说要想满足FromIterator这个trait, 就必须实现from_iter这个函数;
//而from_iter这个函数入参是个满足泛型T约束的iter, 这个T需要是个IntoIterator(一般的容器类型(collect类型)都实现了IntoIterator)
pub trait FromIterator<A>: Sized {
    fn from_iter<T: IntoIterator<Item = A>>(iter: T) -> Self;
}

```
到这里就清楚了, 对左值`let doubled: Vec<i32>`的用.collect方法生成的情况, 最后调用的是Vec的from_iter()函数
```rust
impl<T> FromIterator<T> for Vec<T> {
    #[inline]
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Vec<T> {
        <Self as SpecFromIter<T, I::IntoIter>>::from_iter(iter.into_iter())
    }
}
```

这里把Self转换成了`SpecFromIter<T, I::IntoIter>`, 实例化后是`SpecFromIter<i32, I::IntoIter>`
而这里的`I::IntoIter`其实就是变量a的类型`vec<i32>`的IntoIter

`SpecFromIter<T, I>`说的是要干一件从IntoIter I到Self<T>的事.

```rust
pub(super) trait SpecFromIter<T, I> {
    fn from_iter(iter: I) -> Self;
}

impl<T, I> SpecFromIter<T, I> for Vec<T>
where
    I: Iterator<Item = T>,
{
    default fn from_iter(iterator: I) -> Self {
        SpecFromIterNested::from_iter(iterator)
    }
}
```
最后调用到:
```rust
pub(super) trait SpecFromIterNested<T, I> {
    fn from_iter(iter: I) -> Self;
}

impl<T, I> SpecFromIterNested<T, I> for Vec<T>
where
    I: Iterator<Item = T>,
{
    default fn from_iter(mut iterator: I) -> Self {
        // Unroll the first iteration, as the vector is going to be
        // expanded on this iteration in every case when the iterable is not
        // empty, but the loop in extend_desugared() is not going to see the
        // vector being full in the few subsequent loop iterations.
        // So we get better branch prediction.
        let mut vector = match iterator.next() {
            None => return Vec::new(),
            Some(element) => {
                let (lower, _) = iterator.size_hint();
                let initial_capacity =
                    cmp::max(RawVec::<T>::MIN_NON_ZERO_CAP, lower.saturating_add(1));
                let mut vector = Vec::with_capacity(initial_capacity);
                unsafe {
                    // SAFETY: We requested capacity at least 1
                    ptr::write(vector.as_mut_ptr(), element);
                    vector.set_len(1);
                }
                vector
            }
        };
        // must delegate to spec_extend() since extend() itself delegates
        // to spec_from for empty Vecs
        <Vec<T> as SpecExtend<T, I>>::spec_extend(&mut vector, iterator);
        vector
    }
}
```


## IntoIterator
IntoIterator的关联类型`type IntoIter`是个trait, 即这个关联类型的具体类型要符合Iterator约束.
```rust
pub trait IntoIterator {
    /// The type of the elements being iterated over.
    type Item;

    /// Which kind of iterator are we turning this into?
    type IntoIter: Iterator<Item = Self::Item>;

    fn into_iter(self) -> Self::IntoIter;
}
```
自己实现IntoIterator
```rust
// A sample collection, that's just a wrapper over Vec<T>
#[derive(Debug)]
struct MyCollection(Vec<i32>);

// Let's give it some methods so we can create one and add things
// to it.
impl MyCollection {
    fn new() -> MyCollection {
        MyCollection(Vec::new())
    }

    fn add(&mut self, elem: i32) {
        self.0.push(elem);
    }
}

// and we'll implement IntoIterator
impl IntoIterator for MyCollection {
    type Item = i32;
    type IntoIter = std::vec::IntoIter<Self::Item>; //注意这里实例化了IntoIter类型

    fn into_iter(self) -> Self::IntoIter {
        self.0.into_iter()
    }
}

// Now we can make a new collection...
let mut c = MyCollection::new();

// ... add some stuff to it ...
c.add(0);
c.add(1);
c.add(2);

// ... and then turn it into an Iterator:
for (i, n) in c.into_iter().enumerate() {
    assert_eq!(i as i32, n);
}
```

比如BTreeMap就实现了into_iter的方法
```
impl<'a, K, V> IntoIterator for &'a BTreeMap<K, V> {
    type Item = (&'a K, &'a V);
    type IntoIter = Iter<'a, K, V>;

    fn into_iter(self) -> Iter<'a, K, V> {
        self.iter()
    }
}
```
可以看到BTreeMap的into_iter其实就是self.iter(), 反回的都是`Iter<'a, K, V>`这个结构体(这个结构体实现了Iterator).

# 数组泛型的方法`impl<T> [T]`
数组泛型实现了多种方法, 比如join.
```rust
impl<T> [T] {
    pub fn sort(&mut self)
    where
        T: Ord,
    
    pub fn sort_by<F>(&mut self, mut compare: F)
    where
        F: FnMut(&T, &T) -> Ordering,
        
    pub fn sort_by_key<K, F>(&mut self, mut f: F)
    where
        F: FnMut(&T) -> K,
        K: Ord,
    
    pub fn sort_by_cached_key<K, F>(&mut self, f: F)
    where
        F: FnMut(&T) -> K,
        K: Ord,
        
    pub fn to_vec(&self) -> Vec<T>
    where
        T: Clone,
        
    pub fn to_vec_in<A: Allocator>(&self, alloc: A) -> Vec<T, A>
    where
        T: Clone,
    
    pub fn into_vec<A: Allocator>(self: Box<Self, A>) -> Vec<T, A>
    
    pub fn repeat(&self, n: usize) -> Vec<T>
    where
        T: Copy,
        
    pub fn concat<Item: ?Sized>(&self) -> <Self as Concat<Item>>::Output
    where
        Self: Concat<Item>,
        
    pub fn join<Separator>(&self, sep: Separator) -> <Self as Join<Separator>>::Output
    where
        Self: Join<Separator>, //这里首先约束Self是Join<Separator>
    {
        Join::join(self, sep) //这里调用了约束的join函数. 
    }
    
    pub fn connect<Separator>(&self, sep: Separator) -> <Self as Join<Separator>>::Output
    where
        Self: Join<Separator>,
}
```

但因为T是泛型, 要实现有用的方法, 比如对T进行约束.
比如join就要求`[T]`满足:`Self: Join<Separator>`
```rust
pub trait Join<Separator> {
    /// The resulting type after concatenation
    type Output;

    /// Implementation of [`[T]::join`](slice::join)
    fn join(slice: &Self, sep: Separator) -> Self::Output;
}
```
注意上面的Separator是泛型的类型指示符, 指代具体类型.

上面的例子非常有趣, 泛型`[T]`实现了`join`方法, 而这个join方法看起来又调用了"Self"的join.

是Self有两套join实现吗?
-- 是.
`[T]`有join方法, 没毛病.
而`[V]`实现了Join trait, 也实现了join函数. 如下:
```rust
impl<T: Clone, V: Borrow<[T]>> Join<&T> for [V] {
    type Output = Vec<T>;

    fn join(slice: &Self, sep: &T) -> Vec<T> {
        let mut iter = slice.iter();
        let first = match iter.next() {
            Some(first) => first,
            None => return vec![],
        };
        let size = slice.iter().map(|v| v.borrow().len()).sum::<usize>() + slice.len() - 1;
        let mut result = Vec::with_capacity(size);
        result.extend_from_slice(first.borrow());

        for v in iter {
            result.push(sep.clone());
            result.extend_from_slice(v.borrow())
        }
        result
    }
}
```
这是两套语义, rust并没有因为看见Self有join方法, 就像go一样, 自动推断Self实现了Join trait;
相反的, 用户需要明确声明, 我实现了Join trait(for 关键词).
这种情况下, 会同时存在两套join. 在本例中, 前者还调用了后者.

这不是多此一举吗? 一个同名的join调来调去有意思吗?
--不是. 有意思.
因为Separator不同, 实际调用的Join trait也不同.
比如如果Separator是`&T`, 就需要实现`Join<&T>`的trait. 代码见上面
如果Separator是`&[T]`, 就需要实现`Join<&[T]>`的trait, 如下:
```rust
impl<T: Clone, V: Borrow<[T]>> Join<&[T]> for [V] {
    type Output = Vec<T>;

    fn join(slice: &Self, sep: &[T]) -> Vec<T> {
        let mut iter = slice.iter();
        let first = match iter.next() {
            Some(first) => first,
            None => return vec![],
        };
        let size =
            slice.iter().map(|v| v.borrow().len()).sum::<usize>() + sep.len() * (slice.len() - 1);
        let mut result = Vec::with_capacity(size);
        result.extend_from_slice(first.borrow());

        for v in iter {
            result.extend_from_slice(sep);
            result.extend_from_slice(v.borrow())
        }
        result
    }
}
```
比如字符串的:
```rust
impl<S: Borrow<str>> Join<&str> for [S] {
    type Output = String;

    fn join(slice: &Self, sep: &str) -> String {
        unsafe { String::from_utf8_unchecked(join_generic_copy(slice, sep.as_bytes())) }
    }
}
```

# Vec
Vec结构体:
```rust
pub struct Vec<T, A: Allocator = Global> {
    buf: RawVec<T, A>,
    len: usize,
}
```
方法:
```rust
impl<T> Vec<T> {
     pub const fn new() -> Self
     pub fn with_capacity(capacity: usize) -> Self
     pub unsafe fn from_raw_parts(ptr: *mut T, length: usize, capacity: usize) -> Self
}
```
带allocator的Vec有更多的方法:
```rust
impl<T, A: Allocator> Vec<T, A> {
    pub const fn new_in(alloc: A) -> Self
    pub fn with_capacity_in(capacity: usize, alloc: A) -> Self
    pub fn capacity(&self) -> usize
    pub fn reserve(&mut self, additional: usize)
    pub fn try_reserve(&mut self, additional: usize) -> Result<(), TryReserveError>
    pub fn shrink_to_fit(&mut self)
    pub fn shrink_to(&mut self, min_capacity: usize)
    pub fn into_boxed_slice(mut self) -> Box<[T], A>
    pub fn truncate(&mut self, len: usize)
    pub fn as_slice(&self) -> &[T]
    pub fn as_mut_slice(&mut self) -> &mut [T]
    pub fn as_ptr(&self) -> *const T
    pub fn allocator(&self) -> &A
    pub unsafe fn set_len(&mut self, new_len: usize)
    pub fn swap_remove(&mut self, index: usize) -> T
    pub fn insert(&mut self, index: usize, element: T)
    pub fn remove(&mut self, index: usize) -> T
    pub fn retain<F>(&mut self, mut f: F)
    pub fn dedup_by<F>(&mut self, mut same_bucket: F)
    pub fn push(&mut self, value: T)
    pub fn pop(&mut self) -> Option<T>
    pub fn append(&mut self, other: &mut Self)
    
    pub fn drain<R>(&mut self, range: R) -> Drain<'_, T, A> //返回一个iterator, 包括range内的所有元素
    pub fn clear(&mut self) //移出所有元素
    pub fn len(&self) -> usize
    pub fn is_empty(&self) -> bool
    pub fn split_off(&mut self, at: usize) -> Self
    pub fn resize_with<F>(&mut self, new_len: usize, f: F)
    pub fn leak<'a>(self) -> &'a mut [T]
    
}
```
更具化的泛型
```rust
impl<T: Clone, A: Allocator> Vec<T, A> {
    pub fn resize(&mut self, new_len: usize, value: T)
    pub fn extend_from_slice(&mut self, other: &[T])
    pub fn extend_from_within<R>(&mut self, src: R)
}
```

Vec实现了解引用
```rust
impl<T, A: Allocator> ops::Deref for Vec<T, A> {
    type Target = [T];

    fn deref(&self) -> &[T] {
        unsafe { slice::from_raw_parts(self.as_ptr(), self.len) }
    }
}
```

# Option
Option的方法如下:
```rust
impl<T> Option<T> {
    pub const fn is_some(&self) -> bool
    pub fn is_some_with(&self, f: impl FnOnce(&T) -> bool) -> bool
    pub const fn is_none(&self) -> bool
    pub const fn as_ref(&self) -> Option<&T>
    pub const fn as_mut(&mut self) -> Option<&mut T>
    pub const fn as_pin_ref(self: Pin<&Self>) -> Option<Pin<&T>>
    pub const fn as_pin_mut(self: Pin<&mut Self>) -> Option<Pin<&mut T>>
    
    pub const fn expect(self, msg: &str) -> T
    pub const fn unwrap(self) -> T
    pub const fn unwrap_or(self, default: T) -> T
    
    pub const fn map<U, F>(self, f: F) -> Option<U>
    pub const fn filter<P>(self, predicate: P) -> Self
    pub const fn inspect<F>(self, f: F) -> Self
    pub const fn ok_or<E>(self, err: E) -> Result<T, E>
    pub const fn iter(&self) -> Iter<'_, T>
     
    pub const fn and_then<U, F>(self, f: F) -> Option<U>
    pub const fn and<U>(self, optb: Option<U>) -> Option<U>
    pub const fn or(self, optb: Option<T>) -> Option<T>
    pub const fn or_else<F>(self, f: F) -> Option<T>
    pub const fn xor(self, optb: Option<T>) -> Option<T>
    pub const fn insert(&mut self, value: T) -> &mut T
    pub const fn get_or_insert(&mut self, value: T) -> &mut T
    
    pub const fn take(&mut self) -> Option<T>
    pub const fn replace(&mut self, value: T) -> Option<T>
    pub const fn contains<U>(&self, x: &U) -> bool
    pub const fn zip<U>(self, other: Option<U>) -> Option<(T, U)>
}
```

# Result
Result是经常用的rust抽象, 用于返回值的处理, 很优雅.
代码在`lib/rustlib/src/rust/library/core/src/result.rs`
```rust
#[derive(Copy, PartialEq, PartialOrd, Eq, Ord, Debug, Hash)]
pub enum Result<T, E> {
    /// Contains the success value
    #[lang = "Ok"]
    Ok(#[stable(feature = "rust1", since = "1.0.0")] T),

    /// Contains the error value
    #[lang = "Err"]
    Err(#[stable(feature = "rust1", since = "1.0.0")] E),
}
```
## Result<T, E>的方法:
```rust
impl<T, E> Result<T, E> {
    pub const fn is_ok(&self) -> bool

    /// let x: Result<u32, &str> = Ok(2);
    /// assert_eq!(x.is_ok_with(|&x| x > 1), true);
    ///
    /// let x: Result<u32, &str> = Ok(0);
    /// assert_eq!(x.is_ok_with(|&x| x > 1), false);
    ///
    /// let x: Result<u32, &str> = Err("hey");
    /// assert_eq!(x.is_ok_with(|&x| x > 1), false);
    pub fn is_ok_with(&self, f: impl FnOnce(&T) -> bool) -> bool
    
    pub const fn is_err(&self) -> bool
    
    ///  let  x: Result<u32, Error> =  Err(Error::new(ErrorKind::NotFound, "!"));
    ///  assert_eq!(x.is_err_with(|x| x.kind() == ErrorKind::NotFound), true);
    ///
    ///  let  x: Result<u32, Error> =  Err(Error::new(ErrorKind::PermissionDenied, "!"));
    ///  assert_eq!(x.is_err_with(|x| x.kind() == ErrorKind::NotFound), false);
    ///
    ///  let  x: Result<u32, Error> =  Ok(123);
    ///  assert_eq!(x.is_err_with(|x| x.kind() == ErrorKind::NotFound), false);
    pub  fn  is_err_with(&self, f:  impl  FnOnce(&E) ->  bool) ->  bool
    
    pub  fn  ok(self) ->  Option<T> //把Result<T, E>转换为Option<T>
    
    pub  fn  err(self) ->  Option<E>
    
    pub const fn as_ref(&self) -> Result<&T, &E>
    pub const fn as_mut(&mut self) -> Result<&mut T, &mut E>
    
    //调用F把Result<T, E>转为Result<U, E>
    pub fn map<U, F: FnOnce(T) -> U>(self, op: F) -> Result<U, E>
    pub fn map_or<U, F: FnOnce(T) -> U>(self, default: U, f: F) -> U
    pub fn map_or_else<U, D: FnOnce(E) -> U, F: FnOnce(T) -> U>(self, default: D, f: F) -> U
    
    pub fn map_err<F, O: FnOnce(E) -> F>(self, op: O) -> Result<T, F>
    pub fn inspect<F: FnOnce(&T)>(self, f: F) -> Self
    pub fn inspect_err<F: FnOnce(&E)>(self, f: F) -> Self
    
    /// let x: Result<String, u32> = Ok("hello".to_string());
    /// let y: Result<&str, &u32> = Ok("hello");
    /// assert_eq!(x.as_deref(), y);
    ///
    /// let x: Result<String, u32> = Err(42);
    /// let y: Result<&str, &u32> = Err(&42);
    pub fn as_deref(&self) -> Result<&T::Target, &E>
    where
        T: Deref,
    
    pub fn iter(&self) -> Iter<'_, T>
    pub fn iter_mut(&mut self) -> IterMut<'_, T>
    
    //返回Ok里面的T, 如果Error就panic
    pub fn expect(self, msg: &str) -> T
    where
        E: fmt::Debug,
    
    // unwrap也返回T, 也可能panic, 但似乎就没有panic message
    pub fn unwrap(self) -> T
    where
        E: fmt::Debug,
    
    // 也是unwrap, 但不panic, 不Ok就返回default
    pub fn unwrap_or_default(self) -> T
    where
        T: Default,
    
    //unwrap不panic, 不ok则返回指定的default
    pub fn unwrap_or(self, default: T) -> T
    
    pub fn unwrap_or_else<F: FnOnce(E) -> T>(self, op: F) -> T
    
    // 实际上是unwrap E
    pub fn expect_err(self, msg: &str) -> E
    where
        T: fmt::Debug,
        
    pub fn unwrap_err(self) -> E
    where
        T: fmt::Debug,
        
    pub fn into_ok(self) -> T
    where
        E: Into<!>,
        
    pub fn into_err(self) -> E
    where
        T: Into<!>,
    
    // and表示检测x, y中只要有Error, 就返回Error
    /// let x: Result<u32, &str> = Ok(2);
    /// let y: Result<&str, &str> = Err("late error");
    /// assert_eq!(x.and(y), Err("late error"));
    pub fn and<U>(self, res: Result<U, E>) -> Result<U, E>
    
    /// assert_eq!(Ok(2).and_then(sq_then_to_string), Ok(4.to_string()));
    /// assert_eq!(Err("not a number").and_then(sq_then_to_string), Err("not a number"));
    pub fn and_then<U, F: FnOnce(T) -> Result<U, E>>(self, op: F) -> Result<U, E>
    
    pub fn or<F>(self, res: Result<T, F>) -> Result<T, F>
    pub fn or_else<F, O: FnOnce(E) -> Result<T, F>>(self, op: O) -> Result<T, F>
    
    /// let x: Result<u32, &str> = Ok(2);
    /// assert_eq!(x.contains(&2), true);
    ///
    /// let x: Result<u32, &str> = Ok(3);
    /// assert_eq!(x.contains(&2), false);
    ///
    /// let x: Result<u32, &str> = Err("Some error message");
    /// assert_eq!(x.contains(&2), false);
    pub fn contains<U>(&self, x: &U) -> bool
    where
        U: PartialEq<T>,
}
```
注: const fn表示这个fn可以用在const上下文中

## Result<&T, E>的方法
Result<&T, E>也能调用Result<T, E>的方法, 比如下面copied函数中, 直接用了self.map, 因为Result<T, E>是个范围很广的泛型, 自然也就包括了Result<&T, E>, 这里要把T看成是&T'
```rust
impl<T, E> Result<&T, E> {
    pub  fn  copied(self) ->  Result<T, E>
    where
        T:  Copy,
    {
        self.map(|&t|  t) //注意这里, map的F是FnOnce(T), 是针对Result<T, E>来说的; 这里应该传入"&T", 那么形式上&t就是T, 然后返回t就是返回T. 看起来挺难理解的...
    }
    
    pub fn cloned(self) -> Result<T, E>
    where
        T: Clone,
    {
        self.map(|t| t.clone()) //这里就没有那么绕, t就是&T
    }
}
```

## 其他Result
```rust
impl<T, E> Result<&mut  T, E>
impl<T, E> Result<Option<T>, E>
impl<T, E> Result<Result<T, E>, E>
impl<T> Result<T, T>
```

## Result实现了如下的trait
```rust
impl<T: Clone, E: Clone> Clone for Result<T, E> 
impl<T, E> IntoIterator for Result<T, E>
impl<'a, T, E> IntoIterator  for  &'a  Result<T, E>
impl<'a, T, E> IntoIterator  for  &'a  mut  Result<T, E>
```

# env
```rust
// 这个就是golang的os.args
let args: Vec<String> = env::args().collect();
```

# 取消impl trait
就是取消 取消 取消!
```rust
impl !Send for Args {}

impl !Sync for Args {}
```

# 文件
`lib/rustlib/src/rust/library/std/src/fs.rs`

## 使用举例
### 写文件
```rust
use std::fs::File;
use std::io::prelude::*;

fn main() -> std::io::Result<()> {
    let mut file = File::create("foo.txt")?;
    file.write_all(b"Hello, world!")?;
    Ok(())
}
```
### 读文件到String
```rust
use std::fs::File;
use std::io::prelude::*;

fn main() -> std::io::Result<()> {
    let mut file = File::open("foo.txt")?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    assert_eq!(contents, "Hello, world!");
    Ok(())
}
```
注意这里, `file.read_to_string()`, file对象impl Read, 有`read_to_string()`方法, 就在本文件;
而vscode点击进去, 却是`lib/rustlib/src/rust/library/std/src/io/mod.rs`中io:Read这个trait的默认实现.

### 更有效率的读文件
```rust
use std::fs::File;
use std::io::BufReader;
use std::io::prelude::*;

fn main() -> std::io::Result<()> {
    let file = File::open("foo.txt")?;
    let mut buf_reader = BufReader::new(file);
    let mut contents = String::new();
    buf_reader.read_to_string(&mut contents)?;
    assert_eq!(contents, "Hello, world!");
    Ok(())
}
```

### 不同选项open
```rust
let file = OpenOptions::new()
            .read(true)
            .write(true)
            .create(true)
            .open("foo.txt");
```

### `&File`也能modify 文件
> Note that, although read and write methods require a `&mut File`, because
of the interfaces for [`Read`] and [`Write`], the holder of a `&File` can
still modify the file, either through methods that take `&File` or by
retrieving the underlying OS object and modifying the file that way.
Additionally, many operating systems allow concurrent modification of files
by different processes. Avoid assuming that holding a `&File` means that the
file will not change.

这里是说read和write方法都要求`&mut File`, 但实际上, `&File`的持有者也可以修改文件.
这里提醒大家不要认为持有`&File`的人就"没危险"了, 他们也可以修改你的文件, 因为文件系统允许多进程打开一个文件.

## File对象
```rust
pub struct File {
    inner: fs_imp::File,
}
```
还有几个"辅助"元组对象定义:
```rust
pub struct Metadata(fs_imp::FileAttr);
pub struct ReadDir(fs_imp::ReadDir);
pub struct DirEntry(fs_imp::DirEntry);
pub struct OpenOptions(fs_imp::OpenOptions);
pub struct Permissions(fs_imp::FilePermissions);
pub struct FileType(fs_imp::FileType);
pub struct DirBuilder {
    inner: fs_imp::DirBuilder,
    recursive: bool,
}
```

## 常用函数
* 读所有文件内容到Vec<u8> `pub  fn  read<P:  AsRef<Path>>(path:  P) ->  io::Result<Vec<u8>>`
```rust
use std::fs;
use std::net::SocketAddr;
fn main() -> Result<(), Box<dyn std::error::Error + 'static>> {
    let foo: SocketAddr = String::from_utf8_lossy(&fs::read("address.txt")?).parse()?;
    Ok(())
}
```
* 读所有文件内容到String `pub fn read_to_string<P: AsRef<Path>>(path: P) -> io::Result<String>`
```rust
use std::fs;
use std::net::SocketAddr;
use std::error::Error;
fn main() -> Result<(), Box<dyn Error>> {
    let foo: SocketAddr = fs::read_to_string("address.txt")?.parse()?;
    Ok(())
}
```
* 简单的写slice到文件 `pub fn write<P: AsRef<Path>, C: AsRef<[u8]>>(path: P, contents: C) -> io::Result<()>`
```rust
use std::fs;
fn main() -> std::io::Result<()> {
    fs::write("foo.txt", b"Lorem ipsum")?; //&str可以被认为是AsRef<[u8]>
    fs::write("bar.txt", "dolor sit")?;
    Ok(())
}
```
* remove文件 `pub fn remove_file<P: AsRef<Path>>(path: P) -> io::Result<()>`
```rust
use std::fs;
fn main() -> std::io::Result<()> {
    fs::remove_file("a.txt")?;
    Ok(())
}
```
* metadata
```rust
fn main() -> std::io::Result<()> {
    let attr = fs::metadata("/some/file/path.txt")?;
    // inspect attr ...
    Ok(())
}
```
* symlink_metadata
* rename
* copy
* hard_link
* soft_link
* read_link
* canonicalize 可能和abs path差不多
* create_dir
* create_dir_all
* remove_dir
* remove_dir_all
* read_dir
例子1
```rust
use std::io;
use std::fs::{self, DirEntry};
use std::path::Path;
// one possible implementation of walking a directory only visiting files
fn visit_dirs(dir: &Path, cb: &dyn Fn(&DirEntry)) -> io::Result<()> {
    if dir.is_dir() {
        for entry in fs::read_dir(dir)? {
            let entry = entry?;
            let path = entry.path();
            if path.is_dir() {
                visit_dirs(&path, cb)?;
            } else {
                cb(&entry);
            }
        }
    }
    Ok(())
}
```
例子2
```rust
use std::{fs, io};
fn main() -> io::Result<()> {
    let mut entries = fs::read_dir(".")?
        .map(|res| res.map(|e| e.path()))
        .collect::<Result<Vec<_>, io::Error>>()?;
    // The order in which `read_dir` returns entries is not guaranteed. If reproducible
    // ordering is required the entries should be explicitly sorted.
    entries.sort();
    // The entries have now been sorted by their path.
    Ok(())
}
```
* set_permissions
```rust
use std::fs;
fn main() -> std::io::Result<()> {
    let mut perms = fs::metadata("foo.txt")?.permissions();
    perms.set_readonly(true);
    fs::set_permissions("foo.txt", perms)?;
    Ok(())
}
```

## File的方法
```rust
impl File {
}
```
* open `pub fn open<P: AsRef<Path>>(path: P) -> io::Result<File>`
```rust
use std::fs::File;
fn main() -> std::io::Result<()> {
    let mut f = File::open("foo.txt")?;
    Ok(())
}
```
* create `pub fn create<P: AsRef<Path>>(path: P) -> io::Result<File>`
```rust
use std::fs::File;
fn main() -> std::io::Result<()> {
    let mut f = File::create("foo.txt")?;
    Ok(())
}
```
* options `pub fn options() -> OpenOptions`
```rust
use std::fs::File;
fn main() -> std::io::Result<()> {
    let mut f = File::options().append(true).open("example.log")?;
    Ok(())
}
```
* sync_all 就是fsync `pub fn sync_all(&self) -> io::Result<()>`
```rust
use std::fs::File;
use std::io::prelude::*;
fn main() -> std::io::Result<()> {
    let mut f = File::create("foo.txt")?;
    f.write_all(b"Hello, world!")?;
    f.sync_all()?;
    Ok(())
}
```
* sync_data 比sync_all少一些disk操作 `pub fn sync_data(&self) -> io::Result<()>`
```rust
use std::fs::File;
use std::io::prelude::*;
fn main() -> std::io::Result<()> {
    let mut f = File::create("foo.txt")?;
    f.write_all(b"Hello, world!")?;
    f.sync_data()?;
    Ok(())
}
```
* set_len 设文件大小, 若原本size小, 就shrink文件到新size; 如果原size小, 则剩下的都填0
```rust
use std::fs::File;
fn main() -> std::io::Result<()> {
    let mut f = File::create("foo.txt")?;
    f.set_len(10)?;
    Ok(())
}
```
* metadata
* try_clone
* set_permissions

## 其他impl的trait
```rust
impl AsInner<fs_imp::File> for File
impl FromInner<fs_imp::File> for File
impl IntoInner<fs_imp::File> for File
impl fmt::Debug for File
```
根据注释, 还实现了比如AsFd等trait. 但不知道藏在哪里实现的???
```rust
// In addition to the `impl`s here, `File` also has `impl`s for
// `AsFd`/`From<OwnedFd>`/`Into<OwnedFd>` and
// `AsRawFd`/`IntoRawFd`/`FromRawFd`, on Unix and WASI, and
// `AsHandle`/`From<OwnedHandle>`/`Into<OwnedHandle>` and
// `AsRawHandle`/`IntoRawHandle`/`FromRawHandle` on Windows.
```
## 实现Read trait
```rust
pub struct File {
    inner: fs_imp::File,
}
```
这里显得很啰嗦, 基本都是调用inner的对应函数. 因为File包括了inner.
因为rust没有继承就要再写一遍wrapper吗???
```rust
impl Read for File {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        self.inner.read(buf)
    }

    fn read_vectored(&mut self, bufs: &mut [IoSliceMut<'_>]) -> io::Result<usize> {
        self.inner.read_vectored(bufs)
    }

    fn read_buf(&mut self, buf: &mut ReadBuf<'_>) -> io::Result<()> {
        self.inner.read_buf(buf)
    }

    #[inline]
    fn is_read_vectored(&self) -> bool {
        self.inner.is_read_vectored()
    }

    // Reserves space in the buffer based on the file size when available.
    fn read_to_end(&mut self, buf: &mut Vec<u8>) -> io::Result<usize> {
        buf.reserve(buffer_capacity_required(self));
        io::default_read_to_end(self, buf)
    }

    // Reserves space in the buffer based on the file size when available.
    fn read_to_string(&mut self, buf: &mut String) -> io::Result<usize> {
        buf.reserve(buffer_capacity_required(self));
        io::default_read_to_string(self, buf)
    }
}
```
## 实现Write trait
```rust
impl Write for File {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        self.inner.write(buf)
    }

    fn write_vectored(&mut self, bufs: &[IoSlice<'_>]) -> io::Result<usize> {
        self.inner.write_vectored(bufs)
    }

    #[inline]
    fn is_write_vectored(&self) -> bool {
        self.inner.is_write_vectored()
    }

    fn flush(&mut self) -> io::Result<()> {
        self.inner.flush()
    }
}
```
## 实现Seek trait
```rust
impl Seek for File {
    fn seek(&mut self, pos: SeekFrom) -> io::Result<u64> {
        self.inner.seek(pos)
    }
}
```
## Read Write Seek for &File又来一遍!!!!!
有意思吗?
```rust
impl Read for &File {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        self.inner.read(buf)
    }

    fn read_buf(&mut self, buf: &mut ReadBuf<'_>) -> io::Result<()> {
        self.inner.read_buf(buf)
    }

    fn read_vectored(&mut self, bufs: &mut [IoSliceMut<'_>]) -> io::Result<usize> {
        self.inner.read_vectored(bufs)
    }

    #[inline]
    fn is_read_vectored(&self) -> bool {
        self.inner.is_read_vectored()
    }

    // Reserves space in the buffer based on the file size when available.
    fn read_to_end(&mut self, buf: &mut Vec<u8>) -> io::Result<usize> {
        buf.reserve(buffer_capacity_required(self));
        io::default_read_to_end(self, buf)
    }

    // Reserves space in the buffer based on the file size when available.
    fn read_to_string(&mut self, buf: &mut String) -> io::Result<usize> {
        buf.reserve(buffer_capacity_required(self));
        io::default_read_to_string(self, buf)
    }
}
```

## open最后调用
```rust
use crate::sys::fs as fs_imp;
fn _open(&self, path: &Path) -> io::Result<File> {
        fs_imp::File::open(path, &self.0).map(|inner| File { inner })
    }
```
`fs_imp::File::open`代码在`lib/rustlib/src/rust/library/std/src/sys/unix/fs.rs`
里面调用了很多libc的函数.

# IO
## 常用函数
```rust
pub fn read_to_string<R: Read>(mut reader: R) -> Result<String>
//这个是内部使用的函数
fn read_until<R: BufRead + ?Sized>(r: &mut R, delim: u8, buf: &mut Vec<u8>) -> Result<usize>
```
## Read trait
```rust
pub trait Read {
    /// use std::io;
    /// use std::io::prelude::*;
    /// use std::fs::File;
    ///
    /// fn main() -> io::Result<()> {
    ///     let mut f = File::open("foo.txt")?;
    ///     let mut buffer = [0; 10];
    ///
    ///     // read up to 10 bytes
    ///     let n = f.read(&mut buffer[..])?;
    ///
    ///     println!("The bytes: {:?}", &buffer[..n]);
    ///     Ok(())
    /// }
    //也是传入一个[u8]的数组, 返回大小
    fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
    
    //vector方式读
    fn read_vectored(&mut self, bufs: &mut [IoSliceMut<'_>]) -> Result<usize> {
        default_read_vectored(|b| self.read(b), bufs)
    }
    
    //有没有vector读, 默认false
    fn is_read_vectored(&self) -> bool {
        false
    }
    
    /// use std::io;
    /// use std::io::prelude::*;
    /// use std::fs::File;
    ///
    /// fn main() -> io::Result<()> {
    ///     let mut f = File::open("foo.txt")?;
    ///     let mut buffer = Vec::new();
    ///
    ///     // read the whole file
    ///     f.read_to_end(&mut buffer)?;
    ///     Ok(())
    /// }
    //读完所有byte
    fn read_to_end(&mut self, buf: &mut Vec<u8>) -> Result<usize> {
        default_read_to_end(self, buf)
    }
    
    /// use std::io;
    /// use std::io::prelude::*;
    /// use std::fs::File;
    ///
    /// fn main() -> io::Result<()> {
    ///     let mut f = File::open("foo.txt")?;
    ///     let mut buffer = String::new();
    ///
    ///     f.read_to_string(&mut buffer)?;
    ///     Ok(())
    /// }
    //读完所有string
    fn read_to_string(&mut self, buf: &mut String) -> Result<usize> {
        default_read_to_string(self, buf)
    }
    
    /// use std::io;
    /// use std::io::prelude::*;
    /// use std::fs::File;
    ///
    /// fn main() -> io::Result<()> {
    ///     let mut f = File::open("foo.txt")?;
    ///     let mut buffer = [0; 10];
    ///
    ///     // read exactly 10 bytes
    ///     f.read_exact(&mut buffer)?;
    ///     Ok(())
    /// }
    //一直读到size
    fn read_exact(&mut self, buf: &mut [u8]) -> Result<()> {
        default_read_exact(self, buf)
    }
    
    //read到ReadBuf
    fn read_buf(&mut self, buf: &mut ReadBuf<'_>) -> Result<()> {
        default_read_buf(|b| self.read(b), buf)
    }
    fn read_buf_exact(&mut self, buf: &mut ReadBuf<'_>) -> Result<()>
    
    //借用
    fn by_ref(&mut self) -> &mut Self
    
    //把这个Read转换为byte iterator
    fn  bytes(self) ->  Bytes<Self>
    
    /// use std::io;
    /// use std::io::prelude::*;
    /// use std::fs::File;
    ///
    /// fn main() -> io::Result<()> {
    ///     let mut f1 = File::open("foo.txt")?;
    ///     let mut f2 = File::open("bar.txt")?;
    ///
    ///     let mut handle = f1.chain(f2);
    ///     let mut buffer = String::new();
    ///
    ///     // read the value into a String. We could use any Read method here,
    ///     // this is just one example.
    ///     handle.read_to_string(&mut buffer)?;
    ///     Ok(())
    /// }
    //和next Read成链, 先读Self, 接着读next
    fn chain<R: Read>(self, next: R) -> Chain<Self, R>
    
    /// use std::io;
    /// use std::io::prelude::*;
    /// use std::fs::File;
    ///
    /// fn main() -> io::Result<()> {
    ///     let mut f = File::open("foo.txt")?;
    ///     let mut buffer = [0; 5];
    ///
    ///     // read at most five bytes
    ///     let mut handle = f.take(5);
    ///
    ///     handle.read(&mut buffer)?;
    ///     Ok(())
    /// }
    //返回一个新的Read, 但只读limit字节
    fn take(self, limit: u64) -> Take<Self>
}
```

## Write trait
```rust
pub  trait  Write {
    //很自然的, 这里的buf是个借用
    //和read一样, write也可以部分写
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn write_vectored(&mut self, bufs: &[IoSlice<'_>]) -> Result<usize>
    fn is_write_vectored(&self) -> bool
    //这个flush是write独有的
    fn flush(&mut self) -> Result<()>;
    
    //全写
    fn write_all(&mut self, mut buf: &[u8]) -> Result<()>
    
    fn write_all_vectored(&mut self, mut bufs: &mut [IoSlice<'_>]) -> Result<()>
    //写格式化string到Write
    fn write_fmt(&mut self, fmt: fmt::Arguments<'_>) -> Result<()>
    
    fn by_ref(&mut self) -> &mut Self
}
```

## Seek trait
```rust
pub trait Seek {
    fn seek(&mut self, pos: SeekFrom) -> Result<u64>;
    //从头开始
    fn rewind(&mut self) -> Result<()>
    //返回这个stream的字节数
    fn stream_len(&mut self) -> Result<u64>
    //stream的当前位置
    fn stream_position(&mut self) -> Result<u64>
}
```

## BufRead是Read的派生trait
BufRead自带内部buffer
比如`stdin.lock()`就实现了BufRead
```rust
use std::io;
use std::io::prelude::*;
let stdin = io::stdin();
for line in stdin.lock().lines() {
    println!("{}", line.unwrap());
}
```
比如可以用`BufReader::new(r)`把Reader r转换为BufReader
```rust
use std::io::{self, BufReader};
use std::io::prelude::*;
use std::fs::File;

fn main() -> io::Result<()> {
    let f = File::open("foo.txt")?;
    let f = BufReader::new(f);

    for line in f.lines() {
        println!("{}", line.unwrap());
    }

    Ok(())
}
```

下面是BufRead的定义:
```rust
pub trait BufRead: Read {
    //读出内部buffer, 并从内部reader填入新数据
    fn fill_buf(&mut self) -> Result<&[u8]>;
    //amt数量的字节已经被consume了
    fn consume(&mut self, amt: usize);
    fn has_data_left(&mut self) -> Result<bool>
    
    fn read_until(&mut self, byte: u8, buf: &mut Vec<u8>) -> Result<usize>
    
    fn read_line(&mut self, buf: &mut String) -> Result<usize>
    
    //返回一个按照分隔符byte分割的iterator
    fn split(self, byte: u8) -> Split<Self>
    
    //返回按行分隔的iterator
    fn lines(self) -> Lines<Self>
}
```
