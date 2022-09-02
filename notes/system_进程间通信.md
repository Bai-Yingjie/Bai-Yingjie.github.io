- [ipcs查看进程间通信的情况, 包括消息队列, 共享内存, semaphore](#ipcs查看进程间通信的情况-包括消息队列-共享内存-semaphore)
- [进程间通信类型](#进程间通信类型)
  - [signal](#signal)
  - [匿名管道](#匿名管道)
  - [有名管道和FIFO](#有名管道和fifo)
  - [消息队列](#消息队列)
  - [共享内存](#共享内存)
  - [信号量](#信号量)
  - [futex](#futex)
  - [Unix domain socket](#unix-domain-socket)
  - [Netlink socket](#netlink-socket)
  - [Network socket](#network-socket)
  - [Inotify机制](#inotify机制)
  - [FUSE文件系统](#fuse文件系统)
  - [D-BUS](#d-bus)

参考:
* http://www.chandrashekar.info/articles/linux-system-programming/introduction-to-linux-ipc-mechanims.html

* [https://tldp.org/LDP/tlk/ipc/ipc.html#:~:text=1%20System%20V%20IPC%20Mechanisms,all%20share%20common%20authentication%20methods.](https://tldp.org/LDP/tlk/ipc/ipc.html#:~:text=1%20System%20V%20IPC%20Mechanisms,all%20share%20common%20authentication%20methods.)

# ipcs查看进程间通信的情况, 包括消息队列, 共享内存, semaphore
```
Linux Mint 19 Tara $ ipcs

------ Message Queues --------
key msqid owner perms used-bytes messages

------ Shared Memory Segments --------
key shmid owner perms bytes nattch status
0x00000000 65536 bai 600 524288 2 dest
0x00000000 163841 bai 600 33554432 2 dest
0x00000000 196610 bai 600 33554432 2 dest
0x00000000 294915 bai 600 524288 2 dest
0x00000000 393220 bai 600 524288 2 dest
0x00000000 491525 bai 600 4194304 2 dest
0x00000000 524294 bai 777 2006712 2
0x00000000 622599 bai 777 3035136 2
0x00000000 655368 bai 600 4194304 2 dest
0x00000000 753673 bai 600 524288 2 dest
0x00000000 786442 bai 600 1048576 2 dest
0x00000000 884747 bai 600 524288 2 dest
0x00000000 917516 bai 777 2304 2
0x00000000 950285 bai 600 33554432 2 dest

------ Semaphore Arrays --------
key semid owner perms nsems

```

# 进程间通信类型
## signal
* overhead最小, 用来通知进程关于内核和其他进程状态的改变
* 内核打断进程正常的流程, 调用其注册的handler或者默认的handler
> Signals are the cheapest forms of IPC provided by Linux. Their primary use is to notify processes of change in states or events that occur within the kernel or other processes. We use signals in real world to convey messages with least overhead - think of hand and body gestures. For example, in a crowded gathering, we raise a hand to gain attention, wave hand at a friend to greet and so on.
On Linux, the kernel notifies a process when an event or state change occurs by interrupting the process's normal flow of execution and invoking one of the signal handler functinos registered by the process or by the invoking one of the default signal dispositions supplied by the kernel, for the said event.
## 匿名管道
* 生成两个描述符分别用于读和写
* 用于父子进程, 父进程创建管道, 在folk的时候, 这个管道被dup进子进程的空间
> Anonymous pipes (or simply pipes, for short) provide a mechanism for one process to stream data to another. A pipe has two ends associated with a pair of file descriptors - making it a one-to-one messaging or communication mechanism. One end of the pipe is the read-end which is associated with a file-descriptor that can only be read, and the other end is the write-end which is associated with a file descriptor that can only be written. This design means that pipes are essentially half-duplex.
Anonymous pipes can be setup and used only between processes that share parent-child relationship. Generally the parent process creates a pipe and then forks child processes. Each child process gets access to the pipe created by the parent process via the file descriptors that get duplicated into their address space. This allows the parent to communicate with its children, or the children to communicate with each other using the shared pipe.
Pipes are generally used to implement Producer-Consumer design amongst processes - where one or more processes would produce data and stream them on one end of the pipe, while other processes would consume the data stream from the other end of the pipe.

## 有名管道和FIFO
* 在两个独立的进程间, 打开一个特定的FIFO文件来通信
> Named pipes (or FIFO) are variants of pipe that allow communication between processes that are not related to each other. The processes communicate using named pipes by opening a special file known as a FIFO file. One process opens the FIFO file from writing while the other process opens the same file for reading. Thus any data written by the former process gets streamed through a pipe to the latter process. The FIFO file on disk acts as the contract between the two processes that wish to communicate.

## 消息队列
* 类似于邮箱, 一个进程写消息然后退出, 另一个进程可以从同一个消息队列里读消息
* 通信双方不需要建立连接, 而作为对比, pipe是需要先建立连接的.
* 支持多对多
> Message queues allow one or more processes to write messages, which will be read by one or more reading processes
* linux支持两种消息队列
    * system V: 带message号
    * posix: 带message优先级
    `man mq_overview`
    `mq_open mq_send mq_receive`等函数都是系统调用
> Message Queues are synonymous to mailboxes. One process writes a message packet on the message queue and exits. Another process can access the message packet from the same message queue at a latter point in time. The advantage of message queues over pipes/FIFOs are that the sender (or writer) processes do not have to wait for the receiver (or reader) processes to connect. Think of communication using pipes as similar to two people communicating over phone, while message queues are similar to two people communicating using mail or other messaging services.
There are two standard specifications for message queues.
SysV message queues.
The AT&T SysV message queues support message channeling. Each message packet sent by senders carry a message number. The receivers can either choose to receive message that match a particular message number, or receive all other messages excluding a particular message number or all messages.
POSIX message queues.
The POSIX message queues support message priorities. Each message packet sent by the senders carry a priority number along with the message payload. The messages get ordered based on the priority number in the message queue. When the receiver tries to read a message at a later point in time, the messages with higher priority numbers get delivered first. POSIX message queues also support asynchronous message delivery using threads or signal based notification.

## 共享内存
* 一个进程把自己进程空间的一部分共享给另一个.
* 有两种类型:
    * sysv: 比较古老
    * posix: 现代的, 使用ram文件系统的文件
> As the name implies, this IPC mechanism allows one process to share a region of memory in its address space with another. This allows two or more processes to communicate data more efficiently amongst themselves with minimal kernel intervention.
There are two standard specifications for Shared memory.
SysV Shared memory. Many applications even today use this mechanism for historical reasons. It follows some of the artifacts of SysV IPC semantics.
POSIX Shared memory. The POSIX specifications provide a more elegant approach towards implementing shared memory interface. On Linux, POSIX Shared memory is actually implemented by using files backed by RAM-based filesystem. I recommend using this mechanism over the SysV semantics due to a more elegant file based semantics.

## 信号量
> Semaphores are locking and synchronization mechanism used most widely when processes share resources. Linux supports both SysV semaphores and POSIX semaphores. POSIX semaphores provide a more simpler and elegant implementation and thus is most widely used when compared to SysV semaphores on Linux.

## futex
* linux系统调用
> Futexes are high-performance low-overhead locking mechanisms provided by the kernel. Direct use of futexes is highly discouraged in system programs. Futexes are used internally by POSIX threading API for condition variables and its mutex implementations.

## Unix domain socket
* C-S架构, 全双工, 支持stream和datagram方式
* 大型软件用的很多
> UNIX Domain Sockets provide a mechanism for implementing applications that communicate using the Client-Server architecture. They support both stream and datagram oriented communication, are full-duplex and support a variety of options. They are very widely used for developing many large-scale frameworks.

## Netlink socket
* 和socket接口一样, 但主要用于:
    * 内核线程和用户进程通信
    * 用户控件的进程间的广播通信
> Netlink sockets are similar to UNIX Domain Sockets in its API semantics - but used mainly for two purposes:
For communication between a process in user-space to a thread in kernel-space
For communication amongst processes in user-space using broadcast mode.

## Network socket
* 网络socket
> Based on the same API semantics like UNIX Domain Sockets, Network Sockets API provide mechanisms for communication between processes that run on different hosts on a network. Linux has rich support for features and various protocol stacks for using network sockets API. For all kinds of network programming and distributed programming - network socket APIs form the core interface.

## Inotify机制
* 监控文件系统改变的, 可以和poll select等联用.
inotify是Linux内核2.6.13 (June 18, 2005)版本新增的一个子系统（API），它提供了一种监控文件系统（基于inode的）事件的机制，可以监控文件系统的变化如文件修改、新增、删除等，并可以将相应的事件通知给应用程序。该机制由著名的桌面搜索引擎项目beagle引入用于替代此前具有类似功能但存在诸多缺陷的dnotify。
> The Inotify API on Linux provides a method for processes to know of any changes on a monitored file or a directory asynchronously. By adding a file to inotify watch-list, a process will be notified by the kernel on any changes to the file like open, read, write, changes to file stat, deleting a file and so on.

## FUSE文件系统
> FUSE provides a method to implement a fully functional filesystem in user-space. Various operations on the mounted FUSE filesystem would trigger functions registered by the user-space filesystem handler process. This technique can also be used as an IPC mechanism to implement Client-Server architecture without using socket API semantics.

## D-BUS
* 桌面系统用的多, 是建立在socket API基础上的多进程通信的系统
> D-Bus is a high-level IPC mechanism built generally on top of socket API that provides a mechanism for multiple processes to communicate with each other using various messaging patterns. D-Bus is a standards specification for processes communicating with each other and very widely used today by GUI implementations on Linux following Freedesktop.org specifications.
