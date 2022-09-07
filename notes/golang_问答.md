- [用range迭代一个map的时候, 删除或者新增key安全吗?](#用range迭代一个map的时候-删除或者新增key安全吗)
- [net/Conn可以多个goroutine同时读写吗?](#netconn可以多个goroutine同时读写吗)
- [fmt.Println并发安全吗?](#fmtprintln并发安全吗)
  - [write系统调用是原子的吗?](#write系统调用是原子的吗)
    - [System call atomicity](#system-call-atomicity)

# 用range迭代一个map的时候, 删除或者新增key安全吗?
比如下面的代码:
```
for key := range m {
    if key.expired() {
        delete(m, key)
    }
}
```
答: 安全. 删除一个还没有被loop到的key, range保证不会这个key不会被loop到; 新增一个key, 则可能会也可能不会被loop到. 
因为map是hash桶, loop随机选择一个index然后依次看桶里的元素.

> The iteration order over maps is not specified and is not guaranteed to be the same from one iteration to the next. If map entries that have not yet been reached are removed during iteration, the corresponding iteration values will not be produced. If map entries are created during iteration, that entry may be produced during the iteration or may be skipped. The choice may vary for each entry created and from one iteration to the next. If the map is nil, the number of iterations is 0.

参考: [stackoverflow](https://stackoverflow.com/questions/23229975/is-it-safe-to-remove-selected-keys-from-map-within-a-range-loop)

# net/Conn可以多个goroutine同时读写吗?
可以, Conn是并发安全的
> Multiple goroutines may invoke methods on a Conn simultaneously.

参考笔记: 网络杂记2.md, 搜多线程能不能同时写同一个socket

# fmt.Println并发安全吗?
不安全.
见讨论: https://stackoverflow.com/questions/14694088/is-it-safe-for-more-than-one-goroutine-to-print-to-stdout
> go文档种, 并发安全的api都会说的; 没说的都是并发不安全的.
This is an instance of a more universal Go documentation rule: Things are not safe for concurrent access unless specified otherwise or where obvious from context.

```
Everything fmt does falls back to w.Write() as can be seen here. Because there's no locking around it, everything falls back to the implementation of Write(). As there is still no locking (for Stdout at least), there is no guarantee your output will not be mixed.

I'd recommend using a global log routine.

Furthermore, if you simply want to log data, use the log package, which locks access to the output properly. See the implementation for reference.
```

## write系统调用是原子的吗?
答: 好像应该是, 但实际上并不是; 但对于append模式下的文件来说, 实际上也是
https://cs61.seas.harvard.edu/site/2018/Storage4/#:~:text=System%20call%20atomicity,write%20%2C%20should%20have%20atomic%20effect.

### System call atomicity
Unix file system system calls, such as read and write, should have atomic effect. Atomicity is a correctness property that concerns concurrency—the behavior of a system when multiple computations are happening at the same time. (For example, multiple programs have the file open at the same time.)

An operation is atomic, or has atomic effect, if it always behaves as if it executes without interruption, at one precise moment in time, with no other operation happening. Atomicity is good because it makes complex behavior much easier to understand.

The standards that govern Unix say reads and writes should have atomic effect. It is the operating system kernel’s job to ensure this atomic effect, perhaps by preventing different programs from read or writing to the same file at the same time.

Unfortunately for our narrative, experiment shows that on Linux many write system calls do not have atomic effect, meaning Linux has bugs. But writes made in “append mode” (open(… O_APPEND) or fopen(…, "a")) do have atomic effect. In this mode, which is frequently used for log files, writes are always placed at the end of the open file, after all other data.