virtiofsd是有c版本和rust版本, 这里讨论的是rust版本.

virtiofsd是vhost-user的daemon程序, 运行在host上, 通过virtiofs协议对接Cloud-hypervisor的vhost后端, 作用是把host上的一个目录共享给guest.

- [使用场景](#使用场景)
- [Cargo.toml和Cargo.lock](#cargotoml和cargolock)
- [lib.rs](#librs)
- [main.rs](#mainrs)
  - [VhostUserFsThread结构体](#vhostuserfsthread结构体)
    - [多线程处理vring](#多线程处理vring)
  - [VhostUserBackend](#vhostuserbackend)

# 使用场景
首先需要在host上运行virtiofsd, 指定一个unix socket地址, 和一个共享目录.  
然后告诉cloud-hypervisor增加virtiofs设备.
```shell
WORK_DIR="/work"

cmd_mount() {
    while true; do
        virtiofsd --log-level error --seccomp none --cache=always --socket-path=$WORK_DIR/run/rootextra.sock --shared-dir=$WORK_DIR/rootfs
        echo "restarting virtiofsd"
    done &
    ch-remote --api-socket $WORK_DIR/run/clh.sock add-fs tag=rootextra,socket=$WORK_DIR/run/rootextra.sock
}
```
然后需要在guest的启动流程里面, mount这个"rootextra"
```shell
mkdir /rootextra
mount rootextra /rootextra -t virtiofs -o noatime
```

这里解释一下为什么要用`while true`来包住virtiofsd:

virtiofsd进程虽然是个daemon, 是socket server的角色, 但在socket断开链接的时候virtiofsd进程会退出, 这点设计很不友好.  
socket断开链接可能是VMM遇到了某些情况, 导致了KVM_EXIT_RESET, 从而和virtiofsd的链接中断. 导致virtiofsd退出. 但VMM可以resume, 然后依然想继续connect virtiofsd, 但后者已经不存在了, 导致超时错误, 出现下面的错误:
```shell
2022-12-26T06:47:08.579745672Z cloud-hypervisor is ready
2022-12-26T06:47:08.770061870Z Host kernel: Linux isam-reborn 5.4.209-1.el7.elrepo.x86_64 #1 SMP Wed Aug 3 09:03:41 EDT 2022 x86_64 Linux
2022-12-26T06:47:08.770251609Z VM Boot...
2022-12-26T06:47:08.867627592Z VM Redirect output to /work/run/boot.log...
2022-12-26T06:48:09.043679400Z VMM thread exited with error: Error running command: Server responded with an error: Error rebooting VM: InternalServerError: DeviceManagerApiError(ResponseRecv(RecvError))
2022-12-26T06:48:09.043719047Z (CreateVirtioFs(VhostUserConnect))
2022-12-26T06:48:09.046144519Z Error running command: Error opening HTTP socket: Connection refused (os error 111)
2022-12-26T06:48:09.081708726Z VM State: 
2022-12-26T06:48:09.082411890Z VM Ready...
2022-12-26T06:48:09.082683707Z cloud-hypervisor: 10.301501007s:  WARN:vmm/src/device_manager.rs:1854 -- Ignoring error from setting up SIGWINCH listener: Invalid argument (os error 22)
2022-12-26T06:48:09.082698957Z cloud-hypervisor: 70.509301809s:  ERROR:virtio-devices/src/vhost_user/vu_common_ctrl.rs:410 -- Failed connecting the backend after trying for 1 minute: VhostUserProtocol(SocketConnect(Os { code: 111, kind: ConnectionRefused, message: "Connection refused" }))
2022-12-26T06:48:09.083612791Z Error running command: Error opening HTTP socket: Connection refused (os error 111)
```

解决办法就是让virtiofsd不断的无条件重启.

# Cargo.toml和Cargo.lock
这是个典型的cli程序的rust工程.

* Cargo.toml描述了基本的项目信息和直接的package依赖. 依赖有:
  * [bitflags](https://crates.io/crates/bitflags): 提供`bitflags!`宏用于位操作. [`crates.io`](https://crates.io/crates/bitflags)显示有1870个包依赖它 [用法链接](https://docs.rs/bitflags/latest/bitflags/)
  * [capng](https://crates.io/crates/capng): 是libcap-ng的wrapper, 只有virtiofsd在用. 做为一个很薄的lib, 它只有两个源文件:
    * lib.rs提供了对外的接口
    * [binding.rs](https://github.com/slp/capng/blob/master/src/bindings.rs)是对c库`cap-ng`的接口的`extern "C"`声明, 对内(lib.rs)提供rust接口. 比如libcap-ng提供的C接口`int capng_name_to_capability(const char *name);`在这里被声明成了`pub fn capng_name_to_capability(name: *const ::std::os::raw::c_char) -> ::std::os::raw::c_int;` 这里注意`#[link(name = "cap-ng")]`的链接指示标记
  * [env_logger](https://crates.io/crates/env_logger): 使用环境变量控制log level. 被4630个包依赖. 使用时要同时声明对`log`包的依赖. 提供类似`info!`等记录log的宏.
  * [futures](https://crates.io/crates/futures): 异步编程框架. 提供`join!`, `select!`等宏. 被6957个包依赖, 其中包括[tokio](https://crates.io/crates/tokio)
  * [libc](https://crates.io/crates/libc): 提供对libc的引用.
  * [log](https://crates.io/crates/log): 有11367个包依赖log
  * [libseccomp-sys](https://crates.io/crates/libseccomp-sys): 比libseccomp更底层的C库binding
  * [structopt](https://crates.io/crates/structopt): 解析命令行的, 现在大多用clap
  * [vhost-user-backend](https://crates.io/crates/vhost-user-backend): vhost user后端的框架
  * [vhost](https://crates.io/crates/vhost): 支持如下三种vhost后端:
    * vhost: the dataplane is implemented by linux kernel
      * 比如vhost-net, vhost-scsi and vhost-vsock, 后端在kernel实现, VMM通过ioctl, eventfd等方式和kernel交互
    * vhost-user: the dataplane is implemented by dedicated vhost-user servers
      * 在用户态实现vhost后端, server和client通过unix socket来共享内存. 通过这个共享通道, 双方交换vqueue的信息, 共享vqueue的是master, 否则是slave. master和slave是按vqueue的所有者来分的, 和socket的server/client没有关系.
    * vDPA(vhost DataPath Accelerator): the dataplane is implemented by hardwares
  * [virtio-bindings](https://crates.io/crates/virtio-bindings): virtio的binding, `virtio-bindings = { version = "0.1", features = ["virtio-v5_0_0"] }`的意思是对应Linux kernel version 5.0
  * [vm-memory](https://crates.io/crates/vm-memory): 提供了访问VM物理内存的trait
  * [virtio-queue](https://crates.io/crates/virtio-queue): 提供了virtio queue描述符的实现, readme写的非常好, guest driver和vmm device之间通过vhost协议交互的过程写的很具体. 规范定义了两个format
    * split virtqueues: 这个crate支持这种
    * packed virtqueues: 暂时不支持
  * [vmm-sys-util](https://crates.io/crates/vmm-sys-util): 提供了一些基础功能的wrapper, 给其他rust-vmm组件使用.
  * [syslog](https://crates.io/crates/syslog): 和log配合使用, 输出到syslog
* Cargo.lock是cargo生成的完整的依赖. 和go.sum类似.

# lib.rs
main.rs引用了lib.rs, 后者声明了自己的mod, 每个mod对应一个rs文件.  
![](img/rust_virtiofsd_代码_20230109101423.png)  
这个设计奇怪的地方在于要在别人(lib.rs)的代码里声明自己(比如fuse.rs)是个mod(`pub mod fuse;`)...  
我认为最好应该是自己声明自己是个mod...

# main.rs
main.rs就是virtiofsd这个命令行程序的入口.

## VhostUserFsThread结构体
看名字, 这是个vhost线程的抽象
```rust
struct VhostUserFsThread<F: FileSystem + Send + Sync + 'static> {
    mem: Option<GuestMemoryAtomic<GuestMemoryMmap>>,
    kill_evt: EventFd,
    server: Arc<Server<F>>,
    // handle request from slave to master
    vu_req: Option<SlaveFsCacheReq>,
    event_idx: bool,
    pool: Option<ThreadPool>,
}
```
给这个结构体实现了Clone trait
```rust
impl<F: FileSystem + Send + Sync + 'static> Clone for VhostUserFsThread<F> {
    fn clone(&self) -> Self {
        VhostUserFsThread {
            mem: self.mem.clone(),
            kill_evt: self.kill_evt.try_clone().unwrap(),
            server: self.server.clone(),
            vu_req: self.vu_req.clone(),
            event_idx: self.event_idx,
            pool: self.pool.clone(),
        }
    }
}
```

这个结构体的方法:
```rust
impl<F: FileSystem + Send + Sync + 'static> VhostUserFsThread<F> {
    fn new(fs: F, thread_pool_size: usize) -> Result<Self>
    fn return_descriptor(
        vring_state: &mut VringState,
        head_index: u16,
        event_idx: bool,
        len: usize,
    )
    //线程池的方式处理vring
    fn process_queue_pool(&self, vring: VringMutex) -> Result<bool>
    //单线程
    fn process_queue_serial(&self, vring_state: &mut VringState) -> Result<bool>
}
```

### 多线程处理vring
`process_queue_pool`使用了线程池+async技术来处理vring
```rust
fn process_queue_pool(&self, vring: VringMutex) -> Result<bool> {
    let mut used_any = false;
    let atomic_mem = match &self.mem {
        Some(m) => m,
        None => return Err(Error::NoMemoryConfigured),
    };

    while let Some(avail_desc) = vring
        .get_mut()  //其他笔记说过, 这里实际上是调用了mutex.lock(), 但在下面一行就会马上unlock()
        .get_queue_mut()
        .iter(atomic_mem.memory())
        .map_err(|_| Error::IterateQueue)?
        .next()
    {
        used_any = true;

        // Prepare a set of objects that can be moved to the worker thread.
        let atomic_mem = atomic_mem.clone();
        let server = self.server.clone();
        let mut vu_req = self.vu_req.clone();
        let event_idx = self.event_idx;
        let worker_vring = vring.clone();
        let worker_desc = avail_desc.clone();
        //async后面跟的fn/闭包/block都不会阻塞当前执行, 他们会变成future, 在某个地方await的时候才真正执行
        //spawn_ok()这个方法就是创建一个task(没有说一定是线程)来执行后面的future. spawn_ok的意思是返回值永远是ok
        //move是变量所有权move到下面的闭包块中
        //这里的spwan+async+各种变量clone基本上等同于go语言中的go
        self.pool.as_ref().unwrap().spawn_ok(async move {
            let mem = atomic_mem.memory();
            let head_index = worker_desc.head_index();

            let reader = Reader::new(&mem, worker_desc.clone())
                .map_err(Error::QueueReader)
                .unwrap();
            let writer = Writer::new(&mem, worker_desc.clone())
                .map_err(Error::QueueWriter)
                .unwrap();

            let len = server
                .handle_message(reader, writer, vu_req.as_mut())
                .map_err(Error::ProcessQueue)
                .unwrap();

            Self::return_descriptor(&mut worker_vring.get_mut(), head_index, event_idx, len);
        });
    }

    Ok(used_any)
}
```

## VhostUserBackend
只是个简单的结构体, 内部是带读写锁的`VhostUserFsThread`
```rust
struct VhostUserFsBackend<F: FileSystem + Send + Sync + 'static> {
    thread: RwLock<VhostUserFsThread<F>>,
}
```
它实现了`VhostUserBackend`协议trait
```rust
impl<F: FileSystem + Send + Sync + 'static> VhostUserBackend<VringMutex> for VhostUserFsBackend<F> {
  fn num_queues(&self) -> usize //返回预设的2
  fn max_queue_size(&self) -> usize //返回预设的1024
  fn features(&self) -> u64 //返回预设的flag
  fn handle_event(...) //主处理入口
  等等
}
```
