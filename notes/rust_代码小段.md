- [micro_http](#micro_http)
- [channel in channel](#channel-in-channel)
- [epoll](#epoll)
- [反序列化到结构体](#反序列化到结构体)
- [match语句块做为值](#match语句块做为值)
- [命令行参数拿到文件名并读出其中字符串](#命令行参数拿到文件名并读出其中字符串)
- [从`Vec<&str>`到`Vec<String>`](#从vecstr到vecstring)
- [从&str返回任意类型](#从str返回任意类型)
  - [map_err(Error::Arch)?](#map_errerrorarch)
- [json文件 compile 序列化](#json文件-compile-序列化)
- [Err的使用方法](#err的使用方法)

# micro_http
用的是
`micro_http = { git = "https://github.com/firecracker-microvm/micro-http", branch = "main" }`
```rust
// 起http线程, 用的是micro_http的库
api::start_http_path_thread()
    let server = HttpServer::new_from_fd()
    start_http_thread(server)
        hread::Builder::new() //新线程
            loop {
                match server.requests() {
                    Ok(request_vec) => {
                        for server_request in request_vec {
                            server.respond(server_request.process(
                                |request| {
                                    handle_http_request(request, &api_notifier, &api_sender)
                                }
                            ))
                        }
                    }
                }
            }
```

处理就是从全局url路由表中get route 即`HTTP_ROUTES.routes.get(&path)`, 然后调用route的handle_request函数, 即调用`route.handle_request()`:
```rust
fn handle_http_request(
    request: &Request,
    api_notifier: &EventFd,
    api_sender: &Sender<ApiRequest>,
) -> Response {
    let path = request.uri().get_abs_path().to_string();
    let mut response = match HTTP_ROUTES.routes.get(&path) {
        Some(route) => match api_notifier.try_clone() {
            Ok(notifier) => route.handle_request(request, notifier, api_sender.clone()),
            Err(_) => error_response(
                HttpError::InternalServerError,
                StatusCode::InternalServerError,
            ),
        },
        None => error_response(HttpError::NotFound, StatusCode::NotFound),
    };

    response.set_server("Cloud Hypervisor API");
    response.set_content_type(MediaType::ApplicationJson);
    response
}
```

全局变量route是提前静态注册好的:
HTTP_ROUTES是个全局变量
```rust
lazy_static! {
    /// HTTP_ROUTES contain all the cloud-hypervisor HTTP routes.
    pub static ref HTTP_ROUTES: HttpRoutes = {
        let mut r = HttpRoutes {
            routes: HashMap::new(),
        };

        r.routes.insert(endpoint!("/vm.add-device"), Box::new(VmActionHandler::new(VmAction::AddDevice(Arc::default()))));
        r.routes.insert(endpoint!("/vm.add-user-device"), Box::new(VmActionHandler::new(VmAction::AddUserDevice(Arc::default()))));
        r.routes.insert(endpoint!("/vm.add-disk"), Box::new(VmActionHandler::new(VmAction::AddDisk(Arc::default()))));
        r.routes.insert(endpoint!("/vm.add-fs"), Box::new(VmActionHandler::new(VmAction::AddFs(Arc::default()))));
        r.routes.insert(endpoint!("/vm.add-net"), Box::new(VmActionHandler::new(VmAction::AddNet(Arc::default()))));
        r.routes.insert(endpoint!("/vm.add-pmem"), Box::new(VmActionHandler::new(VmAction::AddPmem(Arc::default()))));
        r.routes.insert(endpoint!("/vm.add-vdpa"), Box::new(VmActionHandler::new(VmAction::AddVdpa(Arc::default()))));
        r.routes.insert(endpoint!("/vm.add-vsock"), Box::new(VmActionHandler::new(VmAction::AddVsock(Arc::default()))));
        r.routes.insert(endpoint!("/vm.boot"), Box::new(VmActionHandler::new(VmAction::Boot)));
        r.routes.insert(endpoint!("/vm.counters"), Box::new(VmActionHandler::new(VmAction::Counters)));
        r.routes.insert(endpoint!("/vm.create"), Box::new(VmCreate {}));
        r.routes.insert(endpoint!("/vm.delete"), Box::new(VmActionHandler::new(VmAction::Delete)));
        r.routes.insert(endpoint!("/vm.info"), Box::new(VmInfo {}));
        r.routes.insert(endpoint!("/vm.pause"), Box::new(VmActionHandler::new(VmAction::Pause)));
        r.routes.insert(endpoint!("/vm.power-button"), Box::new(VmActionHandler::new(VmAction::PowerButton)));
        r.routes.insert(endpoint!("/vm.reboot"), Box::new(VmActionHandler::new(VmAction::Reboot)));
        r.routes.insert(endpoint!("/vm.receive-migration"), Box::new(VmActionHandler::new(VmAction::ReceiveMigration(Arc::default()))));
        r.routes.insert(endpoint!("/vm.remove-device"), Box::new(VmActionHandler::new(VmAction::RemoveDevice(Arc::default()))));
        r.routes.insert(endpoint!("/vm.resize"), Box::new(VmActionHandler::new(VmAction::Resize(Arc::default()))));
        r.routes.insert(endpoint!("/vm.resize-zone"), Box::new(VmActionHandler::new(VmAction::ResizeZone(Arc::default()))));
        r.routes.insert(endpoint!("/vm.restore"), Box::new(VmActionHandler::new(VmAction::Restore(Arc::default()))));
        r.routes.insert(endpoint!("/vm.resume"), Box::new(VmActionHandler::new(VmAction::Resume)));
        r.routes.insert(endpoint!("/vm.send-migration"), Box::new(VmActionHandler::new(VmAction::SendMigration(Arc::default()))));
        r.routes.insert(endpoint!("/vm.shutdown"), Box::new(VmActionHandler::new(VmAction::Shutdown)));
        r.routes.insert(endpoint!("/vm.snapshot"), Box::new(VmActionHandler::new(VmAction::Snapshot(Arc::default()))));
        r.routes.insert(endpoint!("/vmm.ping"), Box::new(VmmPing {}));
        r.routes.insert(endpoint!("/vmm.shutdown"), Box::new(VmmShutdown {}));

        r
    };
}
```
比如create的处理是这样的:
```rust
// /api/v1/vm.create handler
pub struct VmCreate {}

impl EndpointHandler for VmCreate {
    fn handle_request(
        &self,
        req: &Request,
        api_notifier: EventFd,
        api_sender: Sender<ApiRequest>,
    ) -> Response {
        match req.method() {
            Method::Put => {
                match &req.body {
                    Some(body) => {
                        // Deserialize into a VmConfig
                        let vm_config: VmConfig = match serde_json::from_slice(body.raw())
                            .map_err(HttpError::SerdeJsonDeserialize)
                        {
                            Ok(config) => config,
                            Err(e) => return error_response(e, StatusCode::BadRequest),
                        };

                        // Call vm_create()
                        match vm_create(api_notifier, api_sender, Arc::new(Mutex::new(vm_config)))
                            .map_err(HttpError::ApiError)
                        {
                            Ok(_) => Response::new(Version::Http11, StatusCode::NoContent),
                            Err(e) => error_response(e, StatusCode::InternalServerError),
                        }
                    }

                    None => Response::new(Version::Http11, StatusCode::BadRequest),
                }
            }

            _ => error_response(HttpError::BadRequest, StatusCode::BadRequest),
        }
    }
}
```

# channel in channel
使用标准库的channel方法
`pub fn channel<T>() -> (Sender<T>, Receiver<T>)`

发送方:
```rust
// 编译器会从下文推断出这个channel传输的是ApiRequest
let (api_request_sender, api_request_receiver) = std::sync::mpsc::channel();
// 构造内部channel
let (response_sender, response_receiver) = std::sync::mpsc::channel();

api_request_sender
    .send(ApiRequest::VmCreate(config, response_sender))
    .map_err(ApiError::RequestSend)?; //send也会出错, 一般都是对端链接断了
response_receiver.recv().map_err(ApiError::ResponseRecv)??; //这里有两个?, 解开2层Result, 因为外层的Result是recv加的.
```

上面的ApiRequest是类型下面的enum:
```rust
pub enum ApiRequest {
    /// Create the virtual machine. This request payload is a VM configuration
    /// (VmConfig).
    /// If the VMM API server could not create the VM, it will send a VmCreate
    /// error back.
    VmCreate(Arc<Mutex<VmConfig>>, Sender<ApiResponse>),

    /// Boot the previously created virtual machine.
    /// If the VM was not previously created, the VMM API server will send a
    /// VmBoot error back.
    VmBoot(Sender<ApiResponse>),
    ...
}
```
ApiResponse是:
```rust
/// This is the response sent by the VMM API server through the mpsc channel.
pub type ApiResponse = std::result::Result<ApiResponsePayload, ApiError>;
```

# epoll
用的是rust的epoll
```rust
// 新建epoll
let mut epoll = EpollContext::new().map_err(Error::Epoll)?;
    // 底层是epoll create
    let epoll_fd = epoll::create(true)

//增加event. 第一个参数是fd, 第二个是关联的dispatch时候用的token
epoll.add_event(&exit_evt, EpollDispatch::Exit)
    // 底层是epoll ctl, 和c版本一样, ctl add可以给fd绑定一个data

// wait
let num_events = match epoll::wait(epoll_fd, -1, &mut events[..])

//处理
for event in events.iter().take(num_events) {
    //这个就是当时add event的时候关联的data
    let dispatch_event: EpollDispatch = event.data.into();
    //根据dispatch_event来分发
    // 这样的好处是看到这个token, 就知道是哪个事件了, 就知道用哪个fd
    match dispatch_event {
        EpollDispatch::Unknown => {
        }
        EpollDispatch::Exit => {
        }
        EpollDispatch::Api => {
        }
    }
}
```

# 反序列化到结构体
VmmConfig是个结构体:
```rust
/// Used for configuring a vmm from one single json passed to the Firecracker process.
#[derive(Debug, Default, Deserialize, PartialEq, Serialize)]
pub struct VmmConfig {
    #[serde(rename = "balloon")]
    balloon_device: Option<BalloonDeviceConfig>,
    #[serde(rename = "drives")]
    block_devices: Vec<BlockDeviceConfig>,
    #[serde(rename = "boot-source")]
    boot_source: BootSourceConfig,
    #[serde(rename = "logger")]
    logger: Option<LoggerConfig>,
    #[serde(rename = "machine-config")]
    machine_config: Option<VmConfig>,
    #[serde(rename = "metrics")]
    metrics: Option<MetricsConfig>,
    #[serde(rename = "mmds-config")]
    mmds_config: Option<MmdsConfig>,
    #[serde(rename = "network-interfaces", default)]
    net_devices: Vec<NetworkInterfaceConfig>,
    #[serde(rename = "vsock")]
    vsock_device: Option<VsockDeviceConfig>,
}
```
用第三方库serde_json来反序列化, 得到结构体
```rust
let vmm_config: VmmConfig = serde_json::from_slice::<VmmConfig>(config_json.as_bytes())
            .map_err(Error::InvalidJson)?;
```
`from_slice::<VmmConfig>`是实例化泛型函数中T的意思
`pub fn from_slice<'a, T>(v: &'a [u8]) -> Result<T>`

# match语句块做为值
```rust
    // let后面pattern匹配, 匹配的是match语句块的值, 即最后的(res, vmm)
    let (_, vmm) = match build_microvm_from_json(
        seccomp_filters,
        &mut event_manager,
        // Safe to unwrap since '--no-api' requires this to be set.
        config_json.unwrap(),
        instance_info,
        bool_timer_enabled,
        mmds_size_limit,
        metadata_json,
    ) {
        Ok((res, vmm)) => (res, vmm),
        Err(exit_code) => return exit_code,
    };
```

# 命令行参数拿到文件名并读出其中字符串
两个map搞定: 第一个map传入的函数`fs::read_to_string`就已经把文件内容读出来了.
第二个map把上面的`Option<Result<String, Error>>`转换成`Option<String>`, 出错就panic
```rust
let vmm_config_json = arguments
        .single_value("config-file")
        .map(fs::read_to_string)
        .map(|x| x.expect("Unable to open or read from the configuration file"));
```

# 从`Vec<&str>`到`Vec<String>`
```rust
let args = vec!["binary-name", "--exec-file", "foo", "--help"]
    .into_iter()
    .map(String::from) //这里并没有自己写闭包,而是用了现成的函数
    .collect::<Vec<String>>();
```

# 从&str返回任意类型
比如要从字符串转换为如下enum
```rust
/// Supported target architectures.
#[allow(non_camel_case_types)]
#[derive(Debug, PartialEq, Clone, Copy)]
pub(crate) enum TargetArch {
    /// x86_64 arch
    x86_64,
    /// aarch64 arch
    aarch64,
}
```

代码如下
```rust
let  target_arch:  TargetArch  = "x86_64".try_into().map_err(Error::Arch)?;
```
try_info是个trait方法, 实现了`TryInfo<T>`, 而后者是个泛型
```rust
pub trait TryInto<T>: Sized {
    /// The type returned in the event of a conversion error.
    type Error;

    /// Performs the conversion.
    fn try_into(self) -> Result<T, Self::Error>;
}
```

实际上, &str有很多种try_into的实现, 它们都和目标类型有关.
这里的目标类型是自己定义的enum TargetArch
因为TryInfo是个泛型, 所以, 这里自己给&str实现了针对性的try_into:
```rust
impl TryInto<TargetArch> for &str {
    type Error = TargetArchError;
    fn try_into(self) -> std::result::Result<TargetArch, TargetArchError> {
        match self.to_lowercase().as_str() {
            "x86_64" => Ok(TargetArch::x86_64),
            "aarch64" => Ok(TargetArch::aarch64),
            _ => Err(TargetArchError::InvalidString(self.to_string())),
        }
    }
}
```
这也说明, 在rust里, 可以在自己模块给"别人"的类型实现某个trait.
注意这里, 原始trait要求try_into的签名是
`fn try_into(self) -> Result<T, Self::Error>`
而我们实现的时候, 可以实例化:
`fn try_into(self) -> std::result::Result<TargetArch, TargetArchError>`

## map_err(Error::Arch)?
有点奇怪, map_err的入参应该是个函数, 但`Error::Arch`只是个enum, 这样竟然也行?
```rust
#[derive(Debug)]
enum Error {
    Bincode(BincodeError),
    FileOpen(PathBuf, io::Error),
    FileFormat(FilterFormatError),
    Json(JSONError),
    MissingInputFile,
    MissingTargetArch,
    Arch(TargetArchError),
}
```

# json文件 compile 序列化
```rust
// &mut dyn Read时trait object吗
fn parse_json(reader: &mut dyn Read) -> Result<JsonFile> {
    //用了serde_json库, 在本crate的cargo.toml里面dependencies有
    // 从reader读json, 返回JsonFile
    // 如果有错误, 转换为本模块的Json错误
    serde_json::from_reader(reader).map_err(Error::Json)
}

fn compile(args: &Arguments) -> Result<()> {
    //一句话打开文件
    let input_file = File::open(&args.input_file)
        .map_err(|err| Error::FileOpen(PathBuf::from(&args.input_file), err))?;
    // new一个BufReader
    let mut input_reader = BufReader::new(input_file);
    // 从这个input_reader读
    let filters = parse_json(&mut input_reader)?;
    // new一个compiler
    let compiler = Compiler::new(args.target_arch);

    // transform the IR into a Map of BPFPrograms
    let bpf_data: HashMap<String, BpfProgram> = compiler
        .compile_blob(filters.0, args.is_basic) //filters.0是元组tuple的数字下标访问方式
        .map_err(Error::FileFormat)?;

    // serialize the BPF programs & output them to a file
    let output_file = File::create(&args.output_file)
        .map_err(|err| Error::FileOpen(PathBuf::from(&args.output_file), err))?;
    bincode::serialize_into(output_file, &bpf_data).map_err(Error::Bincode)?;

    Ok(())
}
```
注:
* Open以后没有Close, 因为input_file会被move进BufReader::new, 再被move出来, 所有权在这个函数结束的时候会被清理.

# Err的使用方法
```rust
fn main() {
    let mut arg_parser = build_arg_parser();

    //这里和golang的if err := xxx(); err != nil {}意思一样
    if let Err(err) = arg_parser.parse_from_cmdline() {
        eprintln!(
            "Arguments parsing error: {} \n\n\
             For more information try --help.",
            err
        );
        process::exit(EXIT_CODE_ERROR);
    }

    if arg_parser.arguments().flag_present("help") {
        println!("Seccompiler-bin v{}\n", SECCOMPILER_VERSION);
        println!("{}", arg_parser.formatted_help());
        return;
    }
    if arg_parser.arguments().flag_present("version") {
        println!("Seccompiler-bin v{}\n", SECCOMPILER_VERSION);
        return;
    }

    let args = get_argument_values(arg_parser.arguments()).unwrap_or_else(|err| {
        eprintln!(
            "{} \n\n\
            For more information try --help.",
            err
        );
        process::exit(EXIT_CODE_ERROR);
    });

    if let Err(err) = compile(&args) {
        eprintln!("Seccompiler error: {}", err);
        process::exit(EXIT_CODE_ERROR);
    }

    println!("Filter successfully compiled into: {}", args.output_file);
}
```
