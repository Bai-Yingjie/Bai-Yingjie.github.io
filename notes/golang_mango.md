- [库地址](#库地址)
- [代码走读](#代码走读)
  - [接口抽象](#接口抽象)
    - [socket.go](#socketgo)
      - [socket](#socket)
      - [context](#context)
    - [options.go](#optionsgo)
    - [pipe.go](#pipego)
    - [message.go](#messagego)
      - [message pool](#message-pool)
      - [msg对象的buffer管理](#msg对象的buffer管理)
    - [protocol.go](#protocolgo)
      - [离用户最近的NewSocket](#离用户最近的newsocket)
    - [transport.go](#transportgo)
      - [TranPipe本质上是连接conn](#tranpipe本质上是连接conn)
      - [transport是NewDialer和NewListener和命名方式的组合](#transport是newdialer和newlistener和命名方式的组合)
      - [TranDialer TranListener](#trandialer-tranlistener)
      - [总结](#总结)
    - [顶层listener.go和dialer.go](#顶层listenergo和dialergo)
    - [device.go](#devicego)
  - [transport](#transport)
    - [transport/transport.go](#transporttransportgo)
    - [transport/handshaker.go](#transporthandshakergo)
    - [transport/conn.go](#transportconngo)
      - [recv和send写的很有水平](#recv和send写的很有水平)
      - [发送接收总结](#发送接收总结)
      - [option函数就是字符串版本的ioctl](#option函数就是字符串版本的ioctl)
      - [header和同步协议握手](#header和同步协议握手)
      - [后台握手接口](#后台握手接口)
    - [transport/tcp/tcp.go](#transporttcptcpgo)
      - [NewDialer和NewListener](#newdialer和newlistener)
      - [dialer](#dialer)
      - [listener](#listener)
      - [adress和close](#adress和close)
      - [Set和Get Option](#set和get-option)
      - [总结](#总结-1)
    - [transport小节](#transport小节)
    - [transport/inproc/inproc.go](#transportinprocinprocgo)
      - [全局listener表](#全局listener表)
      - [Listen](#listen)
      - [Accept](#accept)
      - [Dial](#dial)
      - [Send和Recv](#send和recv)
      - [总结](#总结-2)
    - [transport/ipc](#transportipc)
  - [internal/core](#internalcore)
    - [internal/core/socket.go](#internalcoresocketgo)
      - [NewDialer和NewListener](#newdialer和newlistener-1)
      - [socket具体定义](#socket具体定义)
      - [核心动作: addPipe](#核心动作-addpipe)
      - [send和recv](#send和recv-1)
    - [internal/core/pipe.go](#internalcorepipego)
      - [pipe定义](#pipe定义)
      - [pipeList](#pipelist)
      - [addPipe和ID生成](#addpipe和id生成)
      - [pipe的send和recv](#pipe的send和recv)
      - [为什么底层send或者recv失败要close掉pipe?](#为什么底层send或者recv失败要close掉pipe)
      - [携带额外数据的典型模式: SetPrivate](#携带额外数据的典型模式-setprivate)
    - [internal/core/dialer.go](#internalcoredialergo)
    - [internal/core/listener.go](#internalcorelistenergo)
    - [总结](#总结-3)
  - [protocol](#protocol)
    - [protocol/req/req.go](#protocolreqreqgo)
      - [核心结构体定义](#核心结构体定义)
      - [为什么要有context?](#为什么要有context)
      - [req的SendMsg](#req的sendmsg)
      - [发送小节](#发送小节)
      - [NewSocket](#newsocket)
      - [AddPipe方法](#addpipe方法)
      - [RecvMsg](#recvmsg)
      - [receiver()函数](#receiver函数)
      - [接收小结](#接收小结)
      - [REQ小结](#req小结)
    - [protocol/rep/rep.go](#protocolreprepgo)
      - [NewSocket](#newsocket-1)
      - [AddPipe](#addpipe)
      - [每个连接一个sender](#每个连接一个sender)
      - [每个连接一个receiver](#每个连接一个receiver)
      - [RecvMsg](#recvmsg-1)
      - [SendMsg](#sendmsg)
      - [REP小节](#rep小节)
    - [req rep疑问](#req-rep疑问)
    - [xreq和xrep](#xreq和xrep)
      - [xreq](#xreq)
      - [xrep](#xrep)
    - [再看核心层的核心价值](#再看核心层的核心价值)
      - [总结](#总结-4)

# 库地址
https://github.com/nanomsg/mangos.git
mangos是nanomsg的纯go版本实现.

> Mangos™ is an implementation in pure Go of the SP (“Scalability Protocols”) messaging system. These are colloquially known as a “nanomsg”.

> The design is intended to make it easy to add new transports with almost trivial effort, as well as new topologies (“protocols” in SP parlance.)

> At present, all of the Req/Rep, Pub/Sub, Pair, Bus, Push/Pull, and Surveyor/Respondent patterns are supported. This project also supports an experimental protocol called Star.

# 代码走读
mangos的顶层go文件, 是典型的抽象定义模式: 先在顶层定义好抽象, 子文件夹管实现. 这是自顶向下的设计方法.

## 接口抽象
### socket.go
socket.go提供了socket的抽象. 这里的socket是zeroMQ的socket概念, 是对更底层"socket"的场景化抽象.
socket是应用侧访问这个SP system的接口.
socket.go是对`internal/core/socket.go`的对外呈现, core的socket.go是实现者.

#### socket
接口设计的很简洁, 符合zeroMQ类似的抽象.
```go
// Socket is the main access handle applications use to access the SP
// system.  It is an abstraction of an application's "connection" to a
// messaging topology.  Applications can have more than one Socket open
// at a time.
type Socket interface {
    // Info returns information about the protocol (numbers and names)
    // and peer protocol.
    Info() ProtocolInfo

    // Close closes the open Socket.  Further operations on the socket
    // will return ErrClosed.
    Close() error

    // Send puts the message on the outbound send queue.  It blocks
    // until the message can be queued, or the send deadline expires.
    // If a queued message is later dropped for any reason,
    // there will be no notification back to the application.
    Send([]byte) error

    // Recv receives a complete message.  The entire message is received.
    Recv() ([]byte, error)

    // SendMsg puts the message on the outbound send.  It works like Send,
    // but allows the caller to supply message headers.  AGAIN, the Socket
    // ASSUMES OWNERSHIP OF THE MESSAGE.
    SendMsg(*Message) error

    // RecvMsg receives a complete message, including the message header,
    // which is useful for protocols in raw mode.
    RecvMsg() (*Message, error)

    // Dial connects a remote endpoint to the Socket.  The function
    // returns immediately, and an asynchronous goroutine is started to
    // establish and maintain the connection, reconnecting as needed.
    // If the address is invalid, then an error is returned.
    Dial(addr string) error

    DialOptions(addr string, options map[string]interface{}) error

    // NewDialer returns a Dialer object which can be used to get
    // access to the underlying configuration for dialing.
    NewDialer(addr string, options map[string]interface{}) (Dialer, error)

    // Listen connects a local endpoint to the Socket.  Remote peers
    // may connect (e.g. with Dial) and will each be "connected" to
    // the Socket.  The accepter logic is run in a separate goroutine.
    // The only error possible is if the address is invalid.
    Listen(addr string) error

    ListenOptions(addr string, options map[string]interface{}) error

    NewListener(addr string, options map[string]interface{}) (Listener, error)

    // GetOption is used to retrieve an option for a socket.
    GetOption(name string) (interface{}, error)

    // SetOption is used to set an option for a socket.
    SetOption(name string, value interface{}) error

    // OpenContext creates a new Context.  If a protocol does not
    // support separate contexts, this will return an error.
    OpenContext() (Context, error)

    // SetPipeEventHook sets a PipeEventHook function to be called when a
    // Pipe is added or removed from this socket (connect/disconnect).
    // The previous hook is returned (nil if none.)  (Only one hook can
    // be used at a time.)
    SetPipeEventHook(PipeEventHook) PipeEventHook
}
```
* Send和Recv是不带header的, 而SendMsg和RecvMsg带header; 发送都是发往outbound Q. Q满了会阻塞
* Dial表达的是connect, Listen是bind
* GetOption和SetOption用来设置底层socket

#### context
每个protocol都有个默认的context. 只有部分protocol允许自建context
context的抽象和socket一样? 只是socket的子集? -- context是建立在socket之上的, 顾名思义, 是带上下文的socket.
```go
// Context is a protocol context, and represents the upper side operations
// that applications will want to use.  Every socket has a default context,
// but only a certain protocols will allow the creation of additional
// Context instances (only if separate stateful contexts make sense for
// a given protocol).
type Context interface {

    // Close closes the open Socket.  Further operations on the socket
    // will return ErrClosed.
    Close() error

    // GetOption is used to retrieve an option for a socket.
    GetOption(name string) (interface{}, error)

    // SetOption is used to set an option for a socket.
    SetOption(name string, value interface{}) error

    // Send puts the message on the outbound send queue.  It blocks
    // until the message can be queued, or the send deadline expires.
    // If a queued message is later dropped for any reason,
    // there will be no notification back to the application.
    Send([]byte) error

    // Recv receives a complete message.  The entire message is received.
    Recv() ([]byte, error)

    // SendMsg puts the message on the outbound send.  It works like Send,
    // but allows the caller to supply message headers.  AGAIN, the Socket
    // ASSUMES OWNERSHIP OF THE MESSAGE.
    SendMsg(*Message) error

    // RecvMsg receives a complete message, including the message header,
    // which is useful for protocols in raw mode.
    RecvMsg() (*Message, error)
}
```

### options.go
上文提到的option, 都在这里定义, 注释写的非常确切.
options大大小小包括很多, 比如raw mode, 比如超时时间, Q的size
举几个例子:
```go
const (
    // OptionRaw is used to test if the socket in RAW mod.  The details of
    // how this varies from normal mode vary from protocol to protocol,
    // but RAW mode is generally minimal protocol processing, and
    // stateless.  RAW mode sockets are constructed with different
    // protocol constructor.  Raw mode is generally used with Device()
    // or similar proxy configurations.
    OptionRaw = "RAW"

    // OptionRecvDeadline is the time until the next Recv times out.  The
    // value is a time.Duration.  Zero value may be passed to indicate that
    // no timeout should be applied.  A negative value indicates a
    // non-blocking operation.  By default there is no timeout.
    OptionRecvDeadline = "RECV-DEADLINE" //默认不超时, 永远等
    
    // OptionRetryTime is used by REQ.  The argument is a time.Duration.
    // When a request has not been replied to within the given duration,
    // the request will automatically be resent to an available peer.
    // This value should be longer than the maximum possible processing
    // and transport time.  The value zero indicates that no automatic
    // retries should be sent.  The default value is one minute.
    //
    // Note that changing this option is only guaranteed to affect requests
    // sent after the option is set.  Changing the value while a request
    // is outstanding may not have the desired effect.
    OptionRetryTime = "RETRY-TIME" //默认一分钟
    
    // OptionSubscribe is used by SUB/XSUB.  The argument is a []byte.
    // The application will receive messages that start with this prefix.
    // Multiple subscriptions may be in effect on a given socket.  The
    // application will not receive messages that do not match any current
    // subscriptions.  (If there are no subscriptions for a SUB/XSUB
    // socket, then the application will not receive any messages.  An
    // empty prefix can be used to subscribe to all messages.)
    OptionSubscribe = "SUBSCRIBE" //subscribe通过option来操作
    
    // OptionWriteQLen is used to set the size, in messages, of the write
    // queue channel. By default, it's 128. This option cannot be set if
    // Dial or Listen has been called on the socket.
    OptionWriteQLen = "WRITEQ-LEN" //发送q的大小, 默认128个message
    
    // OptionLinger is used to set the linger property.  This is the amount
    // of time to wait for send queues to drain when Close() is called.
    // Close() may block for up to this long if there is unsent data, but
    // will return as soon as all data is delivered to the transport.
    // Value is a time.Duration.  Default is one second.
    OptionLinger = "LINGER" // close默认等待1秒, 好让还没发送出去的msg发出去.
    
    // OptionMaxRecvSize supplies the maximum receive size for inbound
    // messages.  This option exists because the wire protocol allows
    // the sender to specify the size of the incoming message, and
    // if the size were overly large, a bad remote actor could perform a
    // remote Denial-Of-Service by requesting ridiculously  large message
    // sizes and then stalling on send.  The default value is 1MB.
    //
    // A value of 0 removes the limit, but should not be used unless
    // absolutely sure that the peer is trustworthy.
    //
    // Not all transports honor this limit.  For example, this limit
    // makes no sense when used with inproc.
    //
    // Note that the size includes any Protocol specific header.  It is
    // better to pick a value that is a little too big, than too small.
    //
    // This option is only intended to prevent gross abuse  of the system,
    // and not a substitute for proper application message verification.
    //
    // This option is type int64.
    OptionMaxRecvSize = "MAX-RCV-SIZE" //这个厉害了, DDOS, 默认最大收1MB, 防止对端饱和攻击.
    
    // OptionReconnectTime is the initial interval used for connection
    // attempts.  If a connection attempt does not succeed, then ths socket
    // will wait this long before trying again.  An optional exponential
    // backoff may cause this value to grow.  See OptionMaxReconnectTime
    // for more details.   This is a time.Duration whose default value is
    // 100msec.  This option must be set before starting any dialers.
    OptionReconnectTime = "RECONNECT-TIME" //重连间隔, 默认100ms
    
    // OptionBestEffort enables non-blocking send operations on the
    // socket. Normally (for some socket types), a socket will block if
    // there are no receivers, or the receivers are unable to keep up
    // with the sender. (Multicast sockets types like Bus or Star do not
    // behave this way.)  If this option is set, instead of blocking, the
    // message will be silently discarded.  The value is a boolean, and
    // defaults to False.
    OptionBestEffort = "BEST-EFFORT" //这个厉害了, 有这个标记的socket, 如果遇到发送时对端没准备好等情况, 直接丢弃msg
    
    // OptionLocalAddr expresses a local address.  For dialers, this is
    // the (often random) address that was locally bound.  For listeners,
    // it is usually the service address.  The value is a net.Addr.  This
    // is generally a read-only value for pipes, though it might sometimes
    // be available on dialers or listeners.
    OptionLocalAddr = "LOCAL-ADDR" //获取本地地址

    // OptionRemoteAddr expresses a remote address.  For dialers, this is
    // the service address.  For listeners, its the address of the far
    // end dialer.  The value is a net.Addr.  It is generally read-only
    // and available only on pipes and dialers.
    OptionRemoteAddr = "REMOTE-ADDR" //获取对端地址, 对pipe和dialer有效.
    
    等等
```

### pipe.go
pipe似乎很重要, 但接口定义很简单:
```go
// Pipe represents the high level interface to a low level communications
// channel.  There is one of these associated with a given TCP connection,
// for example.  This interface is intended for application use.
//
// Note that applications cannot send or receive data on a Pipe directly.
type Pipe interface {

    // ID returns the numeric ID for this Pipe.  This will be a
    // 31 bit (bit 32 is clear) value for the Pipe, which is unique
    // across all other Pipe instances in the application, while
    // this Pipe exists.  (IDs are recycled on Close, but only after
    // all other Pipe values are used.)
    ID() uint32

    // Address returns the address (URL form) associated with the Pipe.
    // This matches the string passed to Dial() or Listen().
    Address() string

    // GetOption returns an arbitrary option.  The details will vary
    // for different transport types.
    GetOption(name string) (interface{}, error)

    // Listener returns the Listener for this Pipe, or nil if none.
    Listener() Listener

    // Dialer returns the Dialer for this Pipe, or nil if none.
    Dialer() Dialer

    // Close closes the Pipe.  This does a disconnect, or something similar.
    // Note that if a dialer is present and active, it will redial.
    Close() error
}
```

### message.go
message是对数据的承载载体, 包括header
对mesage的分包因protocol而异.
这里的message不包括比如tcp/ip头. 对transport来说, 这个message就是个大的payload.
```go
// Message encapsulates the messages that we exchange back and forth.  The
// meaning of the Header and Body fields, and where the splits occur, will
// vary depending on the protocol.  Note however that any headers applied by
// transport layers (including TCP/ethernet headers, and SP protocol
// independent length headers), are *not* included in the Header.
type Message struct {
    // Header carries any protocol (SP) specific header.  Applications
    // should not modify or use this unless they are using Raw mode.
    // No user data may be placed here.
    Header []byte

    // Body carries the body of the message.  This can also be thought
    // of as the message "payload".
    Body []byte

    // Pipe may be set on message receipt, to indicate the Pipe from
    // which the Message was received.  There are no guarantees that the
    // Pipe is still active, and applications should only use this for
    // informational purposes.
    Pipe Pipe //这个有点像channel in channel的意思.

    bbuf   []byte
    hbuf   []byte
    bsize  int
    refcnt int32
}
```

#### message pool
和标准库fmt包一样, msg也用了pool模式, 按照msg的size进行了分块.
这样做是为了减小GC的压力.
```go
type msgCacheInfo struct {
    maxbody int
    pool    *sync.Pool
}

func newMsg(sz int) *Message {
    m := &Message{}
    m.bbuf = make([]byte, 0, sz) //实际的size全部在body中分配
    m.hbuf = make([]byte, 0, 32) //这里很清楚, 默认的header大小为32字节
    m.bsize = sz
    return m
}

// We can tweak these!
var messageCache = []msgCacheInfo{
    {
        maxbody: 64,
        pool: &sync.Pool{
            New: func() interface{} { return newMsg(64) },
        },
    }, {
        maxbody: 128,
        pool: &sync.Pool{
            New: func() interface{} { return newMsg(128) },
        },
    }, {
    ...
    一直翻倍到65536
```
补充: sync.Pool可以传入一个New函数, 在pool里面没有对象的时候, 默认new一个.
有了这个Pool, 可以手动free msg, 进一步减轻gc压力.
```go
// Free releases the message to the pool from which it was allocated.
// While this is not strictly necessary thanks to GC, doing so allows
// for the resources to be recycled without engaging GC.  This can have
// rather substantial benefits for performance.
func (m *Message) Free() {
    if m != nil {
        if atomic.AddInt32(&m.refcnt, -1) == 0 {
            for i := range messageCache {
                if m.bsize == messageCache[i].maxbody {
                    messageCache[i].pool.Put(m) //free其实就是返还到pool
                    return
                }
            }
        }
    }
}
```
#### msg对象的buffer管理
msg对象有如下方法:
* 上面说的Free()
* Clone() 只是把引用计数加一, 表示这个msg是共享的. 调用Clone()的人不要修改msg, 因为这个msg还是别人的.
* Dup() 深拷贝这个msg, 可以修改.
* MakeUnique() 一般用`m = m.MakeUnique()`来保证这个msg是自己独享的: 如果msg的引用计数为1, 返回msg本身; 否则Dup()一份, 删掉原msg, 返回Dup的, 即MakeUnique()后, 原msg也不能使用了.

最后是NewMessage()函数: 从pool中new一个msg
```go
// NewMessage is the supported way to obtain a new Message.  This makes
// use of a "cache" which greatly reduces the load on the garbage collector.
func NewMessage(sz int) *Message {
    var m *Message
    for i := range messageCache {
        if sz < messageCache[i].maxbody {
            m = messageCache[i].pool.Get().(*Message)
            break
        }
    }
    if m == nil {
        m = newMsg(sz)
    }

    m.Body = m.bbuf //Body就是bbuf
    m.Header = m.hbuf //Header就是hbuf
    atomic.StoreInt32(&m.refcnt, 1)
    return m
}
```

### protocol.go
protocol就是场景化的套路类型: 比如Req/Rep Pub/Sub
```go
// Useful constants for protocol numbers.  Note that the major protocol number
// is stored in the upper 12 bits, and the minor (subprotocol) is located in
// the bottom 4 bits.
const (
    ProtoPair       = (1 * 16)
    ProtoPub        = (2 * 16)
    ProtoSub        = (2 * 16) + 1
    ProtoReq        = (3 * 16)
    ProtoRep        = (3 * 16) + 1
    ProtoPush       = (5 * 16)
    ProtoPull       = (5 * 16) + 1
    ProtoSurveyor   = (6 * 16) + 2
    ProtoRespondent = (6 * 16) + 3
    ProtoBus        = (7 * 16)
    ProtoStar       = (100 * 16) // Experimental!
)
```

protocol就是下面的接口:
```go
// ProtocolPipe represents the handle that a Protocol implementation has
// to the underlying stream transport.  It can be thought of as one side
// of a TCP, IPC, or other type of connection.
type ProtocolPipe interface {
    // ID returns a unique 31-bit value associated with this.
    // The value is unique for a given socket, at a given time.
    ID() uint32

    // Close does what you think.
    Close() error

    // SendMsg sends a message.  On success it returns nil. This is a
    // blocking call.
    SendMsg(*Message) error

    // RecvMsg receives a message.  It blocks until the message is
    // received.  On error, the pipe is closed and nil is returned.
    RecvMsg() *Message

    // SetPrivate is used to set protocol private data.
    SetPrivate(interface{})

    // GetPrivate returns the previously stored protocol private data.
    GetPrivate() interface{}
}
```

context就是protocol+使用上下文; 所有context都是stateful的.
奇怪的是`RecvMsg() (*Message, error)`的签名, 只有它和protocol长得不一样
```go
// ProtocolContext is a "context" for a protocol, which contains the
// various stateful operations such as timers, etc. necessary for
// running the protocol.  This is separable from the protocol itself
// as the protocol may permit the creation of multiple contexts.
type ProtocolContext interface {
    // Close closes the context.
    Close() error

    // SendMsg sends the message.  The message may be queued, or
    // may be delivered immediately, depending on the nature of
    // the protocol.  On success, the context assumes ownership
    // of the message.  On error, the caller retains ownership,
    // and may either resend the message or dispose of it otherwise.
    SendMsg(*Message) error

    // RecvMsg receives a complete message, including the message header,
    // which is useful for protocols in raw mode.
    RecvMsg() (*Message, error)

    // GetOption is used to retrieve the current value of an option.
    // If the protocol doesn't recognize the option, EBadOption should
    // be returned.
    GetOption(string) (interface{}, error)

    // SetOption is used to set an option.  EBadOption is returned if
    // the option name is not recognized, EBadValue if the value is
    // invalid.
    SetOption(string, interface{}) error
}
```
ProtocolBase匿名包含了ProtocolContext
```go
// ProtocolBase provides the protocol-specific handling for sockets.
// This is the new style API for sockets, and is how protocols provide
// their specific handling.
type ProtocolBase interface {
    ProtocolContext

    // Info returns the information describing this protocol.
    Info() ProtocolInfo

    // XXX: Revisit these when we can use Pipe natively.

    // AddPipe is called when a new Pipe is added to the socket.
    // Typically this is as a result of connect or accept completing.
    // The pipe ID will be unique for the socket at this time.
    // The implementation must not call back into the socket, but it
    // may reject the pipe by returning a non-nil result.
    AddPipe(ProtocolPipe) error

    // RemovePipe is called when a Pipe is removed from the socket.
    // Typically this indicates a disconnected or closed connection.
    // This is called exactly once, after the underlying transport pipe
    // is closed.  The Pipe ID will still be valid.
    RemovePipe(ProtocolPipe)

    // OpenContext is a request to create a unique instance of the
    // protocol state machine, allowing concurrent use of states on
    // a given protocol socket.  Protocols that don't support this
    // should return ErrProtoOp.
    OpenContext() (ProtocolContext, error)
}
```
ProtocolInfo定义:
```go
// ProtocolInfo is a description of the protocol.
type ProtocolInfo struct {
    Self     uint16
    Peer     uint16
    SelfName string
    PeerName string
}
```

#### 离用户最近的NewSocket
用户使用的时候, 一般会调用NewSocket, 这个API就是对下面protocol的MakeSocket的调用.
```go
// MakeSocket creates a Socket on top of a Protocol.
func MakeSocket(proto Protocol) Socket {
    return core.MakeSocket(proto)
}
```
核心层的实现如下
```go
func newSocket(proto mangos.ProtocolBase) *socket {
    s := &socket{
        proto:         proto,
        reconnMinTime: defaultReconnMinTime,
        reconnMaxTime: defaultReconnMaxTime,
        maxRxSize:     defaultMaxRxSize,
    }
    return s
}

// MakeSocket is intended for use by Protocol implementations.  The intention
// is that they can wrap this to provide a "proto.NewSocket()" implementation.
func MakeSocket(proto mangos.ProtocolBase) mangos.Socket {
    return newSocket(proto)
}
```

### transport.go
提供给transport类型实现方使用的统一的transport的抽象. 这是root下的tansport.go文件, transport下面还有自己的transport.go 这两个文件以后估计会merge到一起.

#### TranPipe本质上是连接conn
注意, ransport的实现方才关心pipe, 应用侧不要用pipe.
```go
// TranPipe behaves like a full-duplex message-oriented connection between two
// peers.  Callers may call operations on a Pipe simultaneously from
// different goroutines.  (These are different from net.Conn because they
// provide message oriented semantics.)
//
// Pipe is only intended for use by transport implementors, and should
// not be directly used in applications.
type TranPipe interface {

    // Send sends a complete message.  In the event of a partial send,
    // the Pipe will be closed, and an error is returned.  For reasons
    // of efficiency, we allow the message to be sent in a scatter/gather
    // list.
    Send(*Message) error

    // Recv receives a complete message.  In the event that either a
    // complete message could not be received, an error is returned
    // to the caller and the Pipe is closed.
    //
    // To mitigate Denial-of-Service attacks, we limit the max message
    // size to 1M.
    Recv() (*Message, error)

    // Close closes the underlying transport.  Further operations on
    // the Pipe will result in errors.  Note that messages that are
    // queued in transport buffers may still be received by the remote
    // peer.
    Close() error

    // GetOption returns an arbitrary transport specific option on a
    // pipe.  Options for pipes are read-only and specific to that
    // particular connection. If the property doesn't exist, then
    // ErrBadOption should be returned.
    GetOption(string) (interface{}, error)
}
```

#### transport是NewDialer和NewListener和命名方式的组合
下面会看到很清楚, 这个文件旨在统一transport的抽象, 我们从上到下看:

```go
// Transport is the interface for transport suppliers to implement.
type Transport interface {
    // Scheme returns a string used as the prefix for SP "addresses".
    // This is similar to a URI scheme.  For example, schemes can be
    // "tcp" (for "tcp://xxx..."), "ipc", "inproc", etc.
    Scheme() string

    // NewDialer creates a new Dialer for this Transport.
    NewDialer(url string, sock Socket) (TranDialer, error)

    // NewListener creates a new PipeListener for this Transport.
    // This generally also arranges for an OS-level file descriptor to be
    // opened, and bound to the the given address, as well as establishing
    // any "listen" backlog.
    NewListener(url string, sock Socket) (TranListener, error)
}
```
一个transport本质要提供
* 一个命名规则: 比如"tcp://xxx..."
* 一个NewDialer方法
* 一个NewListener方法

#### TranDialer TranListener
TranDialer TranListener都分别有别名Dialer和Listener
```go
// TranDialer represents the client side of a connection.  Clients initiate
// the connection.
//
// TranDialer is only intended for use by transport implementors, and should
// not be directly used in applications.
type TranDialer interface {
    // Dial is used to initiate a connection to a remote peer.
    Dial() (TranPipe, error)

    // SetOption sets a local option on the dialer.
    // ErrBadOption can be returned for unrecognized options.
    // ErrBadValue can be returned for incorrect value types.
    SetOption(name string, value interface{}) error

    // GetOption gets a local option from the dialer.
    // ErrBadOption can be returned for unrecognized options.
    GetOption(name string) (value interface{}, err error)
}

// TranListener represents the server side of a connection.  Servers respond
// to a connection request from clients.
//
// TranListener is only intended for use by transport implementors, and should
// not be directly used in applications.
type TranListener interface {

    // Listen actually begins listening on the interface.  It is
    // called just prior to the Accept() routine normally. It is
    // the socket equivalent of bind()+listen().
    Listen() error

    // Accept completes the server side of a connection.  Once the
    // connection is established and initial handshaking is complete,
    // the resulting connection is returned to the client.
    Accept() (TranPipe, error)

    // Close ceases any listening activity, and will specifically close
    // any underlying file descriptor.  Once this is done, the only way
    // to resume listening is to create a new Server instance.  Presumably
    // this function is only called when the last reference to the server
    // is about to go away.  Established connections are unaffected.
    Close() error

    // SetOption sets a local option on the listener.
    // ErrBadOption can be returned for unrecognized options.
    // ErrBadValue can be returned for incorrect value types.
    SetOption(name string, value interface{}) error

    // GetOption gets a local option from the listener.
    // ErrBadOption can be returned for unrecognized options.
    GetOption(name string) (value interface{}, err error)

    // Address gets the local address.  The value may not be meaningful
    // until Listen() has been called.
    Address() string
}

```
以上的"二级"接口依然不是最终的接口, 最终的接口是`TranPipe`, 就是本节最开头的接口定义.

#### 总结
这里对transport的抽象是"分级"的. 接口的函数返回接口, 是对"另一件事"的抽象.  
比如`Transport`的`NewDialer()`方法, 返回`TranDialer`抽象; 再由其的`Dial()`方法返回`TranPipe`抽象, 后者才有`Send(*Message) error`和`Recv() (*Message, error)`方法.
* 抽象是有层次的, 抽象返回抽象.

对比我模糊的对transport的抽象认知: transport似乎应该是包括了send recv等多个方法的单一接口.  
mangos的抽象更有层次, 更清晰.

### 顶层listener.go和dialer.go
和transport的listener和dialer完全不一样, 顶层的定义是面向用户api的.
```go
// Listener is an interface to the underlying listener for a transport
// and address.
type Listener interface {
    // Close closes the listener, and removes it from any active socket.
    // Further operations on the Listener will return ErrClosed.
    Close() error

    // Listen starts listening for new connectons on the address.
    Listen() error

    // Address returns the string (full URL) of the Listener.
    Address() string

    // SetOption sets an option on the Listener. Setting options
    // can only be done before Listen() has been called.
    SetOption(name string, value interface{}) error

    // GetOption gets an option value from the Listener.
    GetOption(name string) (interface{}, error)
}
```
```go
// Dialer is an interface to the underlying dialer for a transport
// and address.
type Dialer interface {
    // Close closes the dialer, and removes it from any active socket.
    // Further operations on the Dialer will return ErrClosed.
    Close() error

    // Dial starts connecting on the address.  If a connection fails,
    // it will restart.
    Dial() error

    // Address returns the string (full URL) of the Listener.
    Address() string

    // SetOption sets an option on the Dialer. Setting options
    // can only be done before Dial() has been called.
    SetOption(name string, value interface{}) error

    // GetOption gets an option value from the Listener.
    GetOption(name string) (interface{}, error)
}
```

### device.go
用于在socket之间转发, 这两个socket必须都是raw模式
```go
func Device(s1 Socket, s2 Socket) error {
    go forwarder(s1, s2)
    if s2 != s1 {
        go forwarder(s2, s1)
    }
    return nil
}

// Forwarder takes messages from one socket, and sends them to the other.
// The sockets must be of compatible types, and must be in Raw mode.
func forwarder(fromSock Socket, toSock Socket) {
    for {
        m, err := fromSock.RecvMsg()
        if err != nil {
            // Probably closed socket, nothing else we can do.
            return
        }

        err = toSock.SendMsg(m)
        if err != nil {
            return
        }
    }
}
```

## transport
### transport/transport.go
提供注册transport实现到全局变量表的方法
```go
var transports = map[string]Transport{}

// RegisterTransport is used to register the transport globally,
// after which it will be available for all sockets.  The
// transport will override any others registered for the same
// scheme.
func RegisterTransport(t Transport) {
    lock.Lock()
    transports[t.Scheme()] = t
    lock.Unlock()
}

// GetTransport is used by a socket to lookup the transport
// for a given scheme.
func GetTransport(scheme string) Transport {
    lock.RLock()
    defer lock.RUnlock()
    if t, ok := transports[scheme]; ok {
        return t
    }
    return nil
}
```
多出使用了技巧: type和等号实际上是alias
```go
// Pipe is a transport pipe.
type Pipe = mangos.TranPipe

// Dialer is a factory that creates Pipes by connecting to remote listeners.
type Dialer = mangos.TranDialer

// Listener is a factory that creates Pipes by listening to inbound dialers.
type Listener = mangos.TranListener

// Transport is our transport operations.
type Transport = mangos.Transport
```

### transport/handshaker.go
这里handshaker是对异步握手的抽象. 所谓异步握手就是握手会在后台进行, 不block当前流程.
```go
// Handshaker is used to support dealing with asynchronous
// handshaking used for some transports.  This allows the
// initial handshaking to be done in the background, without
// stalling the server's accept queue.  This is important to
// ensure that a slow remote peer cannot bog down the server
// or effect a denial-of-service for new connections.
type Handshaker interface {
    // Start injects a pipe into the handshaker.  The
    // handshaking is done asynchronously on a Go routine.
    Start(Pipe)

    // Waits for until a pipe has completely finished the
    // handshaking and returns it.
    Wait() (Pipe, error)

    // Close is used to close the handshaker.  Any existing
    // negotiations will be canceled, and the underlying
    // transport sockets will be closed.  Any new attempts
    // to start will return mangos.ErrClosed.
    Close()
}

```

### transport/conn.go
这个文件实现了基于net.conn的TranPipe. 这个TranPipe也有别名connPipe, 其他的stream式的transport实现可以被被包装成connPipe, 使用这里通用的分包方法和握手协议.
```go
// conn implements the Pipe interface on top of net.Conn.  The
// assumption is that transports using this have similar wire protocols,
// and conn is meant to be used as a building block.
type conn struct {
    c       net.Conn
    proto   ProtocolInfo
    open    bool
    options map[string]interface{} //对于option, 一把get set操作不频繁, 用string类的map再合适不过了.
    maxrx   int
    sync.Mutex
}
```

#### recv和send写的很有水平
对net库, encoding/binary库的使用很到位:
```go
// Recv implements the TranPipe Recv method.  The message received is expected
// as a 64-bit size (network byte order) followed by the message itself.
func (p *conn) Recv() (*Message, error) {
    var sz int64
    var err error
    var msg *Message
    
    //先读size
    //binary标准库解析字节序列到size, 到结构体都行. 需要指明大小端.
    if err = binary.Read(p.c, binary.BigEndian, &sz); err != nil {
        return nil, err
    }

    // Limit messages to the maximum receive value, if not
    // unlimited.  This avoids a potential denial of service.
    if sz < 0 || (p.maxrx > 0 && sz > int64(p.maxrx)) {
        return nil, mangos.ErrTooLong
    }
    
    //根据size准备buffer
    //这里是从pool里new msg
    msg = mangos.NewMessage(int(sz))
    msg.Body = msg.Body[0:sz]
    
    //用io.ReadFull读完整的msg, 也就是size长度的msg.
    //很明显, 这里的前提是stream方式的数据报.
    if _, err = io.ReadFull(p.c, msg.Body); err != nil {
        msg.Free()
        return nil, err
    }
    return msg, nil
}

// Send implements the Pipe Send method.  The message is sent as a 64-bit
// size (network byte order) followed by the message itself.
func (p *conn) Send(msg *Message) error {
    //使用net包的Buffer对象发送
    //net的Buffer是[][]byte, 是典型scatter模式, 即离散buffer模式.
    var buff = net.Buffers{}

    //这里size一定是大端格式的, 而且这个size是header和body的和.
    // Serialize the length header
    l := uint64(len(msg.Header) + len(msg.Body))
    lbyte := make([]byte, 8)
    binary.BigEndian.PutUint64(lbyte, l)

    //lbyte是大端, msg.Header, msg.Body还是原始字节序
    //用append来组包, 因为是二维byte, append只拷贝`[]byte`头, 不存在额外数据拷贝.
    // Attach the length header along with the actual header and body
    buff = append(buff, lbyte, msg.Header, msg.Body)

    //buffer自带的writeTo函数, 一把写完.
    if _, err := buff.WriteTo(p.c); err != nil {
        return err
    }

    msg.Free()
    return nil
}
```
注意, 以上的函数中的header部分是`[]byte`, 对其具体的header格式没有认知.

#### 发送接收总结
* 发送接收都是先有个size, 再根据size取出"payload". size是header+body的总大小
    * 因为共享size, 所以接收方没有办法分别知道Header和Body的大小. 所以只能都用Body来接收
    * 即发送方发的是Header+Body, 到接收方结构体中, Header为空, Body是原Header+Body
* 使用了encoding/binary对字节序列进行"解释", 比如size就是按大端字节序解析的.
* 发送的是net.Buffers, 是个二维byte `[][]byte`, 本质上是scatter的指针数组, 不拷贝数据
* 接收用的是sync.Pool的内存池化方式, 减小gc压力.
* 对header和body不做假设, 也没有知识. 发送接收认为他们都是[]byte

#### option函数就是字符串版本的ioctl
```go
func (p *conn) GetOption(n string) (interface{}, error) {
    switch n {
    case mangos.OptionMaxRecvSize:
        return p.maxrx, nil
    }
    if v, ok := p.options[n]; ok {
        return v, nil
    }
    return nil, mangos.ErrBadProperty
}

func (p *conn) SetOption(n string, v interface{}) {
    switch n {
    case mangos.OptionMaxRecvSize:
        p.maxrx = v.(int)
    }
    p.options[n] = v
}
```

#### header和同步协议握手
```go
// connHeader is exchanged during the initial handshake.
type connHeader struct {
    Zero     byte // must be zero
    S        byte // 'S'
    P        byte // 'P'
    Version  byte // only zero at present
    Proto    uint16
    Reserved uint16 // always zero at present
}
```
header里面固定的字段意义

握手的过程是交换header的过程.
```go
// handshake establishes an SP connection between peers.  Both sides must
// send the header, then both sides must wait for the peer's header.
// As a side effect, the peer's protocol number is stored in the conn.
// Also, various properties are initialized.
func (p *conn) handshake() error {
    var err error

    h := connHeader{S: 'S', P: 'P', Proto: p.proto.Self}
    //这里的binary.Write就是直接对底层conn p.c 做send操作.
    if err = binary.Write(p.c, binary.BigEndian, &h); err != nil {
        return err
    }
    
    //再从对端读出handshake信息
    if err = binary.Read(p.c, binary.BigEndian, &h); err != nil {
        _ = p.c.Close()
        return err
    }
    if h.Zero != 0 || h.S != 'S' || h.P != 'P' || h.Reserved != 0 {
        _ = p.c.Close()
        return mangos.ErrBadHeader
    }
    // The only version number we support at present is "0", at offset 3.
    if h.Version != 0 {
        _ = p.c.Close()
        return mangos.ErrBadVersion
    }

    // The protocol number lives as 16-bits (big-endian) at offset 4.
    if h.Proto != p.proto.Peer {
        _ = p.c.Close()
        return mangos.ErrBadProto
    }
    p.open = true
    return nil
}
```

#### 后台握手接口
上面是同步的握手函数, 是具体实现. 同步的接口可以被异步框架异步化, 来满足handshaker的要求
基本上, 握手是底层建立连接后要做的第一件事.
下面是conn.go提供的异步包装框架, 是承上(transport/handshaker.go要求的后台握手)启下(各个transport类型实现的handshake)
```go
type connHandshakerPipe interface {
    handshake() error //被包装的对象要实现handshake函数
    Pipe //和send recv等tranpipe接口, 这个接口是对所有transport类型的统一抽象.
}

type connHandshakerItem struct {
    c connHandshakerPipe
    e error
}
type connHandshaker struct {
    workq  map[connHandshakerPipe]bool //interface可以当map的key
    doneq  []*connHandshakerItem //注意这里, handshaker是个汇聚者, 这里包括了listen后accept的所有conn(被包装为connHandshakerPipe)
    closed bool
    cv     *sync.Cond
    sync.Mutex
}
```

先看start: 一个connHandshaker可以和多个pipe后台握手, 每个pipe都启动一个go routine; 一个pipe代表了一个conn
```go
func (h *connHandshaker) Start(p Pipe) {
    // If the following type assertion fails, then its a software bug.
    conn := p.(connHandshakerPipe)
    h.Lock()
    h.workq[conn] = true //标记这个conn正在握手中
    h.Unlock()
    go h.worker(conn)
}

func (h *connHandshaker) worker(conn connHandshakerPipe) {
    item := &connHandshakerItem{c: conn}
    item.e = conn.handshake() //这里直接调用同步版本的handshake函数. 此时必然已经建立连接, 否则也不会有conn对象.
    h.Lock()
    defer h.Unlock()

    delete(h.workq, conn)

    //有错误就close掉这个conn
    if item.e != nil {
        _ = item.c.Close()
        item.c = nil
    } else if h.closed {
        item.e = mangos.ErrClosed
        _ = item.c.Close()
    }
    
    //把结果放到doneq里
    h.doneq = append(h.doneq, item)
    h.cv.Broadcast() //广播会唤醒所有等待的goroutine, 但实际上wait路径上有锁, 结果是只能有一个routine在cv上wait. 其他人在锁上wait.
}
```

wait是等待返回任意一个已经完成握手的conn
```go
func (h *connHandshaker) Wait() (Pipe, error) {
    h.Lock()
    defer h.Unlock()
    for len(h.doneq) == 0 && !h.closed {
        h.cv.Wait() //持有锁的wait
    }
    if h.closed {
        return nil, mangos.ErrClosed
    }
    item := h.doneq[0] //按完成顺序取
    h.doneq = h.doneq[1:] //整个过程被锁保护
    return item.c, item.e
}
```

close
```go
func (h *connHandshaker) Close() {
    h.Lock()
    h.closed = true
    h.cv.Broadcast()
    for conn := range h.workq {
        _ = conn.Close()
    }
    for len(h.doneq) != 0 {
        item := h.doneq[0]
        h.doneq = h.doneq[1:]
        if item.c != nil {
            _ = item.c.Close()
        }
    }
    h.Unlock()
}
```

### transport/tcp/tcp.go
tcp实现了Scheme NewDialer NewListener方法, 满足了transport接口.
```go
type tcpTran int

func (t tcpTran) Scheme() string {
    return "tcp"
}
```
有意思的是, tcpTran只是个int类型的重命名.
在init里, 注册这个tcpTran类型
```go
const (
    // Transport is a transport.Transport for TCP.
    Transport = tcpTran(0)
)

//本质上是注册接口类型: transports[t.Scheme()] = t
func init() {
    transport.RegisterTransport(Transport)
}
```

#### NewDialer和NewListener
这两个接口类似工厂类, 其产品transport.Dialer和transport.Listener才是重点.

```go
func (t tcpTran) NewDialer(addr string, sock mangos.Socket) (transport.Dialer, error) {
    var err error
    //StripScheme把tcp://192.168.0.1:1234中的192.168.0.1:1234部分提取出来
    if addr, err = transport.StripScheme(t, addr); err != nil {
        return nil, err
    }

    // check to ensure the provided addr resolves correctly.
    //实际上是调用net.ResolveTCPAddr
    if _, err = transport.ResolveTCPAddr(addr); err != nil {
        return nil, err
    }

    d := &dialer{ //返回重点, 就是这个dialer
        addr:  addr,
        proto: sock.Info(), //这里transport也关心应用socket类型了?
        hs:    transport.NewConnHandshaker(), //handshake是应用侧socket交换信息的开始
    }

    return d, nil
}

func (t tcpTran) NewListener(addr string, sock mangos.Socket) (transport.Listener, error) {
    var err error
    l := &listener{ //返回重点, 就是这个listener
        proto:  sock.Info(),
        closeq: make(chan struct{}),
    }

    if addr, err = transport.StripScheme(t, addr); err != nil {
        return nil, err
    }

    l.addr = addr

    l.handshaker = transport.NewConnHandshaker()
    return l, nil
}
```

#### dialer
dialer要实现Dial GetOption和SetOption接口
```go
type dialer struct {
    addr        string
    proto       transport.ProtocolInfo
    hs          transport.Handshaker
    maxRecvSize int
    d           net.Dialer
    lock        sync.Mutex
}

func (d *dialer) Dial() (_ transport.Pipe, err error) {
    conn, err := d.d.Dial("tcp", d.addr) // 先dail得到conn, 也就是pipe
    if err != nil {
        return nil, err
    }

    p := transport.NewConnPipe(conn, d.proto) //包装已有的conn得到connPipe, stream方式的Conn都可以用它来实现transport. 但为什么这里要知道proto? proto是应用侧socket的类型, 包括自己和对端.
    d.lock.Lock()
    p.SetOption(mangos.OptionMaxRecvSize, d.maxRecvSize) //似乎默认maxRecvSize为0
    d.lock.Unlock()
    d.hs.Start(p) //里面会断言p是connHandshakerPipe, 因为NewConnPipe()返回的p, 实现了handshake()函数. 注意这里的handshake()函数是小写开头, 外部不能直接调用. 但go编译器的类型系统还是会判定p实现了不对外的handshake()函数.
    return d.hs.Wait() //先start再wait, 这就是同步化了, 实际就像是调了p.handshake()函数. 但实际上handshake()是不对外的, 只能用异步再同步化操作.
}
```

注: 这里把接口的隐含属性用的出神入化. p是个interface, 把p传入给`d.hs.Start(p)`时, p的方法集变成了TranPipe接口的方法集; 在`transport/conn.go`里的start实现中, 断言入参TranPipe是`conn := p.(connHandshakerPipe)`
因为p是`transport/conn.go`里NewConnPipe()函数返回的, 保证能被同一个文件里的start函数断言成功. 而且其签名函数handshake()可以小写. 这些都是内部的事情.

#### listener
listener要复杂点, 要实现Accept Listen Address Close和Set/Get Option的方法
```go
type listener struct {
    addr        string
    bound       net.Addr
    proto       transport.ProtocolInfo
    l           net.Listener
    lc          net.ListenConfig
    maxRecvSize int
    handshaker  transport.Handshaker
    closeq      chan struct{}
    once        sync.Once
    lock        sync.Mutex
}
```
Accept()一般紧跟着Listen()之后, 等待并返回协商成功的pipe
```go
func (l *listener) Accept() (transport.Pipe, error) {

    if l.l == nil {
        return nil, mangos.ErrClosed
    }
    return l.handshaker.Wait()
}
```
Listen启动了一个goroutine来做accept
```go
func (l *listener) Listen() (err error) {
    select {
    case <-l.closeq:
        return mangos.ErrClosed //如果之前这个listener被close()过, 就不能再listen了.
    default:
    }
    l.l, err = l.lc.Listen(context.Background(), "tcp", l.addr)
    if err != nil {
        return
    }
    l.bound = l.l.Addr()
    go func() {
        for {
            conn, err := l.l.Accept()
            if err != nil {
                select {
                case <-l.closeq:
                    return
                default:
                    // We probably should be checking
                    // if this is a temporary error.
                    // If we run out of files we will
                    // spin hard here.
                    time.Sleep(time.Millisecond)
                    continue
                }
            }
            p := transport.NewConnPipe(conn, l.proto) //stream式的conn都被包装成一个connPipe, connPipe定义了默认的分包规则和握手协议.
            l.lock.Lock()
            p.SetOption(mangos.OptionMaxRecvSize, l.maxRecvSize) //maxRecvSize设置改变后, 影响以后的conn
            l.lock.Unlock()
            l.handshaker.Start(p) //注意这里, listen的结果只有在这里有所体现, 和l联系在一起. 会在socket.Accept()用户侧api中wait并返回transport.Pipe
        }
    }()
    return
}
```

#### adress和close
address返回全称, 比如`tcp://192.168.1.1:1111`
close先用close channel的方式改变listener的状态, 这样其他例程可以安全的检查listener是否已经被close掉了. -- 似乎比bool的方式先进些?
close的意思是停止listen和accept, 但之前已经建立的连接不受影响.
```go
func (l *listener) Address() string {
    if b := l.bound; b != nil {
        return "tcp://" + b.String()
    }
    return "tcp://" + l.addr
}

func (l *listener) Close() error {
    l.once.Do(func() {
        close(l.closeq)
        if l.l != nil {
            _ = l.l.Close()
        }
        l.handshaker.Close()
    })
    return nil
}
```

#### Set和Get Option
和dialer类似, 有
* OptionMaxRecvSize
* OptionKeepAliveTime

#### 总结
tcp是stream方式的transport, 底层使用net标准库的方法, 最主要的是把net.Conn包装成了connPipe, 后者是mangos的统一化的流式pipe的实现, 有最简单的根据header+body的size定界方法, 和协议握手的方法.
可以说, tcp使用了connPipe的"helper"方法, 实现了流式transport, 满足了transport的所有接口要求.
~~需要注意的是, mangos做为nanomsg的纯go版本实现, 并没有follow其前身zeroMQ对报文格式的规定, 即分frame的规定.~~  -- 有待确认.

### transport小节
* transport是知道顶层socket信息的, 因为transport先于protocol动作, 比如在client dial的时候, 就要握手交换protocol的信息做验证. server listen的时候也如是. 

### transport/inproc/inproc.go
前面我们知道, transport首先要实现NewDialer, NewListener, 后续要实现Dial和Listen方法, 最后一层要实现Send和Recv. 注意Send和Recv都是规定好的`m *mangos.Message` -- 为什么不规定个interface来做msg抽象?

#### 全局listener表
```go
var listeners struct {
    // Who is listening, on which "address"?
    byAddr map[string]*listener //记录了全局的名字
    cv     sync.Cond
    mx     sync.Mutex
}
```

#### Listen
NewListener就不看了, 就是返回一个带Listen功能和Accept功能的对象.
Listen很简单, 如果要Listen的名字没有人用, 表示可以listen, 就记录到全局表中.
```go
func (l *listener) Listen() error {
    listeners.mx.Lock()
    if l.closed {
        listeners.mx.Unlock()
        return mangos.ErrClosed
    }
    if _, ok := listeners.byAddr[l.addr]; ok {
        listeners.mx.Unlock()
        return mangos.ErrAddrInUse
    }

    l.active = true
    listeners.byAddr[l.addr] = l
    listeners.cv.Broadcast() //这里要广播给谁呢?
    listeners.mx.Unlock()
    return nil
}
```

#### Accept
每次Accept生成一个新的inproc类型的server
```go
func (l *listener) Accept() (mangos.TranPipe, error) {
    server := &inproc{
        selfProto: l.selfProto,
        peerProto: l.peerProto,
        addr:      addr(l.addr),
    }
    server.readyq = make(chan struct{})
    server.closeq = make(chan struct{})

    listeners.mx.Lock()
    if !l.active || l.closed {
        listeners.mx.Unlock()
        return nil, mangos.ErrClosed
    }
    l.accepters = append(l.accepters, server) //这里这么早就add了, 不好吧? 要不放到case里 -- 根据下文逻辑, 一定要先在l.accepters里有server
    listeners.cv.Broadcast()
    listeners.mx.Unlock()

    select {
    case <-server.readyq: //等着有人写readq, 说明就有个新连接; 核心层会在for里调用这里的Accept
        return server, nil
    case <-server.closeq:
        return nil, mangos.ErrClosed
    }
}
```

#### Dial
```go
func (d *dialer) Dial() (transport.Pipe, error) {

    var server *inproc
    client := &inproc{
        selfProto: d.selfProto,
        peerProto: d.peerProto,
        addr:      addr(d.addr), //client也直接用dial的addr, 不合适吧?
    }
    client.readyq = make(chan struct{})
    client.closeq = make(chan struct{})

    listeners.mx.Lock()

    // NB: No timeouts here!
    for {
        var l *listener
        var ok bool
        if l, ok = listeners.byAddr[d.addr]; !ok || l == nil { //一定要先有listenr才能Dial, 不支持先有client, 再有server. -- 其实没必要
            listeners.mx.Unlock()
            return nil, mangos.ErrConnRefused
        }

        if (client.selfProto != l.peerProto) || //这是个简单的协商机制
            (client.peerProto != l.selfProto) {
            listeners.mx.Unlock()
            return nil, mangos.ErrBadProto
        }

        if len(l.accepters) != 0 {
            server = l.accepters[len(l.accepters)-1] //取最后一个server
            l.accepters = l.accepters[:len(l.accepters)-1]
            break
        }

        listeners.cv.Wait()
        continue
    }

    listeners.mx.Unlock()

    //到这里client和server已经配对.
    server.wq = make(chan *transport.Message) //才建立server的通道, 注意都是unbuffer类型的
    server.rq = make(chan *transport.Message)
    client.rq = server.wq //client和server共用一个channel, 只是方向不一样
    client.wq = server.rq
    server.peer = client
    client.peer = server

    close(server.readyq)
    close(client.readyq)
    return client, nil
}
```

#### Send和Recv
```go
func (p *inproc) Send(m *mangos.Message) error {

    // Upper protocols expect to have to pick header and body part.
    // Also we need to have a fresh copy of the message for receiver, to
    // break ownership.
    nmsg := mangos.NewMessage(len(m.Header) + len(m.Body))
    nmsg.Body = append(nmsg.Body, m.Header...)
    nmsg.Body = append(nmsg.Body, m.Body...) //复制报文, 但是不free?
    select {
    case p.wq <- nmsg: //就是把拷贝后的msg放到channel里
        return nil
    case <-p.closeq:
        nmsg.Free()
        return mangos.ErrClosed
    case <-p.peer.closeq:
        nmsg.Free()
        return mangos.ErrClosed
    }
}
```
```go
func (p *inproc) Recv() (*transport.Message, error) {

    select {
    case m := <-p.rq: //接收的时候不复制msg
        return m, nil
    case <-p.closeq:
        return nil, mangos.ErrClosed
    case <-p.peer.closeq:
        return nil, mangos.ErrClosed
    }
}
```

#### 总结
* server和client都是`*inproc`类型
* Send msg是拷贝式的.
* 基于名字查找和共享channel的消息传递, 开销比较小.

### transport/ipc


## internal/core
### internal/core/socket.go
#### NewDialer和NewListener
transport提供了`GetTransport()`函数从全局map表`var transports = map[string]Transport{}`中获取已经注册的transport工厂对象, 该对象的NewDialer和NewListener方法会新建连接.
而这里internal core里面, 就调用了`GetTransport()`, 根据传入的string, 返回`mangos.Dialer`. 注意这个Dialer已经不是transport的Dialer了.
```go
func (s *socket) NewDialer(addr string, options map[string]interface{}) (mangos.Dialer, error) {
    t := s.getTransport(addr)
    td, err := t.NewDialer(addr, s) //调用transport层的NewDialer
    d := &dialer{
        d:             td,
        s:             s,    //很重要, dialer保持了这个socket的信息
        reconnMinTime: s.reconnMinTime,
        reconnMaxTime: s.reconnMaxTime,
        asynch:        s.dialAsynch,
        addr:          addr,
    }
    for n, v := range options {
        //处理options
        d.SetOption(n, v)
        或td.SetOption(n, v)
    }
    s.dialers = append(s.dialers, d) //把刚才New的dialer关联到socket; socket可以有多个dialer
    return d, nil
}
```
上面的NewDialer就是顶层API
`func (s *socket) Dial(addr string) error`或`func (s *socket) DialOptions(addr string, opts map[string]interface{}) error`调用下来的.
```go
func (s *socket) Dial(addr string) error
    s.DialOptions(addr, nil)
        d, err := s.NewDialer(addr, opts)
        return d.Dial() //注意这里并不是transport的Dial, 而是应用侧的dial; 应用侧的dial签名不同, 它没有返回conn
```

同样的listen也是类似的过程
```go
socket.ListenOptions 或socket.Listen
    l := socket.NewListener()
    l.Listen() //注意这里开始Listen, 这也是应用侧的listen, 只返回error
```
后面会看到, 应用侧Dial和Listen后, conn的信息是通过socket的addPipe方法来保存的.

#### socket具体定义
在core看起来, socket持有多个listener, 多个dialer, 以及真正的conn(也就是这里的pipeList).
```go
// socket is the meaty part of the core information.
type socket struct {
    proto mangos.ProtocolBase

    sync.Mutex

    closed        bool          // true if Socket was closed at API level
    reconnMinTime time.Duration // reconnect time after error or disconnect
    reconnMaxTime time.Duration // max reconnect interval
    maxRxSize     int           // max recv size
    dialAsynch    bool          // asynchronous dialing?

    listeners []*listener
    dialers   []*dialer
    pipes     pipeList //定义于pipe.go
    pipehook  mangos.PipeEventHook //提供给app层的hook函数入口
}
```
最后一个hook再详细看一下: 在socket的pipe中枢系统发生变化时, 调用这个hook.
```go
const (
    // PipeEventAttaching is called before the Pipe is registered with the
    // socket.  The intention is to permit the application to reject
    // a pipe before it is attached.
    PipeEventAttaching = iota

    // PipeEventAttached occurs after the Pipe is attached.
    // Consequently, it is possible to use the Pipe for delivering
    // events to sockets, etc.
    PipeEventAttached

    // PipeEventDetached occurs after the Pipe has been detached
    // from the socket.
    PipeEventDetached
)
```
换算成y的说法就是. onConnect, onDisconnect.

#### 核心动作: addPipe
>   // AddPipe is called when a new Pipe is added to the socket.
    // Typically this is as a result of connect or accept completing.
    // The pipe ID will be unique for the socket at this time.
    // The implementation must not call back into the socket, but it
    // may reject the pipe by returning a non-nil result.
    就是说每当有连接的时候, 就会addPipe

```go
func (s *socket) addPipe(tp transport.Pipe, d *dialer, l *listener) 
    p := newPipe(tp, s, d, l)
    s.pipes.Add(p)
    ph(mangos.PipeEventAttaching, p) //调用hook
    s.proto.AddPipe(p) //这个pipe也会被add到ProtocolBase
    ph(mangos.PipeEventAttached, p) //调用hook
```
注意这里的这句代码有学问:
`s.proto.AddPipe(p)`
* `p`是个`*core.pipe`类型, 本身是小写的, 不能直接被外部使用. 但这里调用protocol的接口函数:
`AddPipe(p)`, 把`p`当作`ProtocolPipe`来看, 就把pipe的"能力"输出了.
* 这里的s.proto是mangos.ProtocolBase, 实际上是每个protocol的实现者的socket对象. 比如req.socket
* 要输出能力, 不一定要自己声明; 调用别人的接口, 把自己包装出去也可以.

#### send和recv
socket的重头戏, 这是核心层的对用户层API的具体实现的地方:
```go
func (s *socket) SendMsg(msg *Message) error {
    return s.proto.SendMsg(msg) //不要困惑 这里还没到transport. 这里的send需要protocol来决定发送给谁, 怎么发. 所以要用s.proto对象来send.
}

func (s *socket) Send(b []byte) error {
    msg := mangos.NewMessage(len(b))
    msg.Body = append(msg.Body, b...) //这里打散再拼接性能会不会有问题? 不用个bytes.Buffer啥的? 看这里的情况是拷贝发送. 这里https://gist.github.com/xogeny/b819af6a0cf8ba1caaef似乎是说copy和append性能差不多
    return s.SendMsg(msg)
}

func (s *socket) RecvMsg() (*Message, error) {
    return s.proto.RecvMsg()
}

func (s *socket) Recv() ([]byte, error) {
    msg, err := s.RecvMsg()
    if err != nil {
        return nil, err
    }
    b := make([]byte, 0, len(msg.Body)) //返回新buffer而不是transport侧底层的buffer; 而且是去了头的; 而且这个buffer是直接make出来的, 不是用的pool的buffer.
    b = append(b, msg.Body...)
    msg.Free() //注意这里的Free是返还给pool
    return b, nil
}
```

### internal/core/pipe.go

这里的pipe实现了多个接口:
* pipe.go.Pipe接口, 暂时不知道哪里用了
```go
// Pipe represents the high level interface to a low level communications
// channel.  There is one of these associated with a given TCP connection,
// for example.  This interface is intended for application use.
//
// Note that applications cannot send or receive data on a Pipe directly.
type Pipe interface {

    // ID returns the numeric ID for this Pipe.  This will be a
    // 31 bit (bit 32 is clear) value for the Pipe, which is unique
    // across all other Pipe instances in the application, while
    // this Pipe exists.  (IDs are recycled on Close, but only after
    // all other Pipe values are used.)
    ID() uint32

    // Address returns the address (URL form) associated with the Pipe.
    // This matches the string passed to Dial() or Listen().
    Address() string

    // GetOption returns an arbitrary option.  The details will vary
    // for different transport types.
    GetOption(name string) (interface{}, error)

    // Listener returns the Listener for this Pipe, or nil if none.
    Listener() Listener

    // Dialer returns the Dialer for this Pipe, or nil if none.
    Dialer() Dialer

    // Close closes the Pipe.  This does a disconnect, or something similar.
    // Note that if a dialer is present and active, it will redial.
    Close() error
}
```
* protocol.go.ProtocolPipe接口, 这个接口被protocol的实现者使用. 具体来讲, 在protocol的实现实例的AddPipe()被调用的时候, ProtocolPipe这个接口会被传入.
```go
// ProtocolPipe represents the handle that a Protocol implementation has
// to the underlying stream transport.  It can be thought of as one side
// of a TCP, IPC, or other type of connection.
type ProtocolPipe interface {
    // ID returns a unique 31-bit value associated with this.
    // The value is unique for a given socket, at a given time.
    ID() uint32

    // Close does what you think.
    Close() error

    // SendMsg sends a message.  On success it returns nil. This is a
    // blocking call.
    SendMsg(*Message) error

    // RecvMsg receives a message.  It blocks until the message is
    // received.  On error, the pipe is closed and nil is returned.
    RecvMsg() *Message

    // SetPrivate is used to set protocol private data.
    SetPrivate(interface{})

    // GetPrivate returns the previously stored protocol private data.
    GetPrivate() interface{}
}
```

#### pipe定义
在core看起来, pipe的信息很丰富: 它毫无疑问的持有底层transport的pipe对象, 同时它还知道这个pipe的listender和dialer, 即知道对端和自己的信息. 它还持有socket对象, 知道app层的想法.  
总结: pipe是个中枢.
```go
// pipe wraps the Pipe data structure with the stuff we need to keep
// for the core.  It implements the Pipe interface.
type pipe struct {
    id        uint32
    p         transport.Pipe
    l         *listener
    d         *dialer
    s         *socket
    closeOnce sync.Once
    data      interface{} // Protocol private
    added     bool
    closing   bool
    lock      sync.Mutex // held across calls to remPipe and addPipe
}
```
#### pipeList
pipeList是个map, 保存了所有的pipe实例, 每个pipe实例有个独特的uint32 ID
```go
type pipeList struct {
    pipes map[uint32]*pipe
    lock  sync.Mutex
}
```
#### addPipe和ID生成
socket中调用addPipe会新生成一个uint32 ID, 从随机数开始, 全局唯一.
原理是在全局表`map[uint32]struct{}`中, 依次找个没被使用的ID.
是的, ID被使用完了还会还到这个map里.
真正的add是调用Add把p按照p.id
```go
func (l *pipeList) Add(p *pipe) {
    l.lock.Lock()
    if l.pipes == nil {
        l.pipes = make(map[uint32]*pipe)
    }
    l.pipes[p.id] = p
    l.lock.Unlock()
}
```

#### pipe的send和recv
pipe是原始conn的封装, 所以send recv都是直接调用底层transport的接口; 所以到这里就不是应用侧的发送接收了.
```go
func (p *pipe) SendMsg(msg *mangos.Message) error {

    if err := p.p.Send(msg); err != nil {
        _ = p.Close()
        return err
    }
    return nil
}

func (p *pipe) RecvMsg() *mangos.Message {

    msg, err := p.p.Recv()
    if err != nil {
        _ = p.Close() //API定义的很明白, 如果底层Recv返回错误, 就close掉这个pipe
        return nil
    }
    msg.Pipe = p //这里值得注意, 从msg里能够拿到pipe的信息, 进而能拿到所有信息. 但不保证到msg的处理的时候, pipe还在.
    return msg
}
```

#### 为什么底层send或者recv失败要close掉pipe?
那后面还怎么整?  
而且在close中, 会彻底关闭transport层的连接
```go
func (p *pipe) Close() error {
    p.closeOnce.Do(func() {
        // Close the underlying transport pipe first.
        _ = p.p.Close()

        // Deregister it from the socket.  This will also arrange
        // for asynchronously running the event callback, and
        // releasing the pipe ID for reuse.
        p.lock.Lock()
        p.closing = true
        if p.added {
            p.s.remPipe(p)
        }
        p.lock.Unlock()

        if p.d != nil {
            // Inform the dialer so that it will redial.
            go p.d.pipeClosed() //这里值得关注, 里面会延迟调用redial函数重新建立连接, redial会不断重试.
        }
    })
    return nil
}
```
所以回答上面的问题, 发送或者接收失败都会close掉连接. 但close的流程最后启动了redial流程.  
也就是说, 比如发送失败的情况下, mangos并不是直接重新再send一次, 而是把整个连接都关掉, 再重新建立新连接来重发.

#### 携带额外数据的典型模式: SetPrivate
使用万能的interface做为输入
```go
func (p *pipe) SetPrivate(i interface{}) {
    p.data = i
}

func (p *pipe) GetPrivate() interface{} {
    return p.data
}
```

### internal/core/dialer.go
在socket.go里, NewDialer就是new的下面的dialer
```go
type dialer struct {
    sync.Mutex
    d             transport.Dialer
    s             *socket
    addr          string
    closed        bool
    active        bool
    asynch        bool
    redialer      *time.Timer
    reconnTime    time.Duration
    reconnMinTime time.Duration
    reconnMaxTime time.Duration
    closeq        chan struct{}
}
```

dial流程
```go
func (d *dialer) Dial() error {
    d.Lock()
    if d.active {
        d.Unlock()
        return mangos.ErrAddrInUse
    }
    if d.closed {
        d.Unlock()
        return mangos.ErrClosed
    }
    d.closeq = make(chan struct{})
    d.active = true
    d.reconnTime = d.reconnMinTime
    if d.asynch {
        go d.redial()
        d.Unlock()
        return nil
    }
    d.Unlock()
    return d.dial(false) //就是下面的dial函数
}

func (d *dialer) dial(redial bool) error {
    d.Lock()
    if d.closed {
        d.Unlock()
        return errors.ErrClosed
    }
    if d.asynch {
        redial = true
    }
    d.Unlock()

    p, err := d.d.Dial()
    if err == nil {
        d.s.addPipe(p, d, nil) //dial成功了就addPipe到socket
        return nil
    }

    //到这里就是不成功, 需要重试
    d.Lock()
    defer d.Unlock()

    // We're no longer dialing, so let another reschedule happen, if
    // appropriate.   This is quite possibly paranoia.  We should only
    // be in this routine in the following circumstances:
    //
    // 1. Initial dialing (via Dial())
    // 2. After a previously created pipe fails and is closed due to error.
    // 3. After timing out from a failed connection attempt.

    //没让你重试就返回
    if !redial {
        return err
    }
    switch err {
    case mangos.ErrClosed: //对端不给连
        // Stop redialing, no further action.

    default: //缓启动逻辑
        // Exponential backoff, and jitter.  Our backoff grows at
        // about 1.3x on average, so we don't penalize a failed
        // connection too badly.
        minfact := float64(1.1)
        maxfact := float64(1.5)
        actfact := rand.Float64()*(maxfact-minfact) + minfact
        rtime := d.reconnTime
        if d.reconnMaxTime != 0 {
            d.reconnTime = time.Duration(actfact * float64(d.reconnTime))
            if d.reconnTime > d.reconnMaxTime {
                d.reconnTime = d.reconnMaxTime
            }
        }
        d.redialer = time.AfterFunc(rtime, d.redial) //AfterFunc会启动一个goroutine来延迟执行d.redial; 这里还用了经典的 方法转函数的编译器黑科技
    }
    return err
}

func (d *dialer) redial() {
    _ = d.dial(true)
}
```

总的来说, dialer实现了对底层transport的dail, 支持后台redial, 支持dail失败重试(不是马上, 而是带缓启动的重试); 支持多种option:
* OptionReconnectTime
* OptionMaxReconnectTime
* OptionDialAsynch

### internal/core/listener.go
listener是对transport的listener的简单包装
```go
type listener struct {
    sync.Mutex
    l      transport.Listener
    s      *socket
    addr   string
    closed bool
    active bool
}
```

Listen先查看状态, 再调用transport的Listen.
```go
func (l *listener) Listen() error {
    // This function sets up a goroutine to accept inbound connections.
    // The accepted connection will be added to a list of accepted
    // connections.  The Listener just needs to listen continuously,
    // as we assume that we want to continue to receive inbound
    // connections without limit.

    l.Lock()
    if l.closed {
        l.Unlock()
        return mangos.ErrClosed
    }
    if l.active {
        l.Unlock()
        return mangos.ErrAddrInUse
    }
    l.active = true
    l.Unlock()
    if err := l.l.Listen(); err != nil {
        l.Lock()
        l.active = false
        l.Unlock()
        return err
    }

    go l.serve() //serve()函数才是不断的在循环里Accept连接的主体
    return nil
}
```
server()函数是被go的
```go
// serve spins in a loop, calling the accepter's Accept routine.
func (l *listener) serve() {
    for {
        l.Lock()
        if l.closed {
            l.Unlock()
            break
        }
        l.Unlock()

        // If the underlying PipeListener is closed, or not
        // listening, we expect to return back with an error.
        if tp, err := l.l.Accept(); err == mangos.ErrClosed {
            return
        } else if err == nil {
            l.s.addPipe(tp, nil, l) //把握手后的tranPipe加入socket.
        } else {
            // Debounce a little bit, to avoid thrashing the CPU.
            time.Sleep(time.Second / 100)
        }
    }
}
```

### 总结
core的核心是pipe. 它承上启下
* 上面的socket负责符合app的想象, socket的send和recv都要按照protocol的套路来整, 比如多播啥的
* 下面的dialer和listener对下对接transport的dialer和listener, 提供了额外的重试连接, 后台连接, routine化accpet等功能.
* dialer和listener成功返回的连接, 被包装成pipe, add到socket里. 核心是
`func (s *socket) addPipe(tp transport.Pipe, d *dialer, l *listener)`
    * 对dialer来说, 会这样调用`d.s.addPipe(p, d, nil)`
    * 对listener来说, 会这样调用`l.s.addPipe(tp, nil, l)`
    * 最终都是add到socket的pipes中, 这是个按ID索引的pipe表 `map[uint32]*pipe`

## protocol
protocol是高于socket层的抽象, 是socket的场景化模式的归纳, 继承自zeroMQ. 目前支持
* bus
* pair
* pub/sub
* pull/push
* req/rep
* star
* 以及以上带x版本的.

下面从最基本的req/rep开始, 看看protocol怎么实现的

### protocol/req/req.go
每个protocol都有统一的协议编号, req/rep的大协议号一样, 后四位小协议号不同
```go
// Protocol identity information.
const (
    Self     = protocol.ProtoReq //48
    Peer     = protocol.ProtoRep //49
    SelfName = "req"
    PeerName = "rep"
)
```

req.go只依赖protocol包, 它对core包是无感知的.所以虽然core.pipe是唯一的mangos.ProtocolPipe实现者, 但core.pipe也是小写, 而且req.go也不引用core, 那怎么实例化mangos.ProtocolPipe的?

#### 核心结构体定义
req的socket核心当然是socket的具体实例:
```go
type socket struct {
    sync.Mutex
    defCtx  *context              // default context
    ctxs    map[*context]struct{} // all contexts (set)
    ctxByID map[uint32]*context   // contexts by request ID
    nextID  uint32                // next request ID
    closed  bool                  // true if we are closed
    sendq   []*context            // contexts waiting to send
    readyq  []*pipe               // pipes available for sending
}

```
socket持有的context和pipe在本文件定义:
```go
type pipe struct {
    p      protocol.Pipe
    s      *socket //反向持有socket
    closed bool
}

type context struct {
    s          *socket //反向持有socket
    cond       *sync.Cond
    resendTime time.Duration     // tunable resend time
    sendExpire time.Duration     // how long to wait in send
    recvExpire time.Duration     // how long to wait in recv
    sendTimer  *time.Timer       // send timer
    recvTimer  *time.Timer       // recv timer
    resender   *time.Timer       // resend timeout
    reqMsg     *protocol.Message // message for transmit
    repMsg     *protocol.Message // received reply
    sendMsg    *protocol.Message // messaging waiting for send
    lastPipe   *pipe             // last pipe used for transmit
    reqID      uint32            // request ID
    recvWait   bool              // true if a thread is blocked in RecvMsg
    bestEffort bool              // if true, don't block waiting in send
    queued     bool              // true if we need to send a message
    closed     bool              // true if we are closed
}
```

#### 为什么要有context?
因为socket的SendMsg, RecvMsg都依赖context发送
```go
func (s *socket) SendMsg(m *protocol.Message) error {
    return s.defCtx.SendMsg(m)
}
```

#### req的SendMsg
最顶层是context send
```go
func (c *context) SendMsg(m *protocol.Message) error {
    s := c.s
    id := atomic.AddUint32(&s.nextID, 1)
    id |= 0x80000000

    // cooked mode, we stash the header
    m.Header = append([]byte{}, //按照大端字节序发id
        byte(id>>24), byte(id>>16), byte(id>>8), byte(id))

    s.Lock() //每个context只能同时发送一个msg
    defer s.Unlock()
    if s.closed || c.closed {
        return protocol.ErrClosed
    }

    //上一次还没发送出去, 哪里有问题了...
    //因为req/rep是同步模式, 取消上次的发送: 上次的msg会被free掉, 延迟的timer被stop; 并把这个context从socket的sendq里面删掉: s.sendq = append(s.sendq[:i], s.sendq[i+1:]...); 最后广播c.cond.Broadcast()
    c.cancel() // this cancels any pending send or recv calls
    c.unscheduleSend() //似乎这个函数多余了, 在c.cancel()中调用过了

    c.reqID = id
    c.queued = true
    c.sendMsg = m

    s.sendq = append(s.sendq, c) //本次的context加入socket的sendq

    if c.bestEffort {
        // for best effort case, we just immediately go the
        // reqMsg, and schedule it as a send.  No waiting.
        // This means that if the message cannot be delivered
        // immediately, it will still get a chance later.
        s.send() //没看懂上面的注释, 但这里明显是调用socket的原始send, 不管错误马上返回
        return nil //这里返回了, 那上面sendq怎么办? 不删掉吗?
    }

    expired := false
    if c.sendExpire > 0 {
        c.sendTimer = time.AfterFunc(c.sendExpire, func() { //超时就取消的函数
            s.Lock()
            if c.sendMsg == m {
                expired = true
                c.cancel() // also does a wake up
            }
            s.Unlock()
        })
    }

    s.send() //触发后台send

    // This sleeps until someone picks us up for scheduling.
    // It is responsible for providing the blocking semantic and
    // ultimately back-pressure.  Note that we will "continue" if
    // the send is canceled by a subsequent send.
    for c.sendMsg == m && !expired && !c.closed {
        //sync.Cond.Wait接口需要在for里面调用
        c.cond.Wait() //典型的场景是上面的s.send把c.sendMsg置为nil, 解除这里的for循环, 退出wait往下走. 但如果readyq中没有可用的pipe, 就是说没有有效的连接, SendMsg会block在这里, 直到有可用的连接为止.
    }
    if c.sendMsg == m { //到这里, 要么就是超时了, 要么就是close了;这个m现在是发不了了
        c.unscheduleSend() //删除m对应的context
        c.sendMsg = nil //就是说这个m不发了
        c.reqID = 0
        if c.closed {
            return protocol.ErrClosed
        }
        return protocol.ErrSendTimeout //返回错误, m并没有发送.
    }
    return nil
}
```
context send里面调用了socket send
注意这里的send没有任何参数和返回值: 需要知道的已经全部知道, 它要做的就是调度发送.
```go
func (s *socket) send() {
    for len(s.sendq) != 0 && len(s.readyq) != 0 { //有待发送的context 并且有ready的pipe
        c := s.sendq[0] //取出第一个待发context
        s.sendq = s.sendq[1:]
        c.queued = false

        var m *protocol.Message
        if m = c.sendMsg; m != nil { //从sendMsg转移到reqMsg
            c.reqMsg = m
            c.sendMsg = nil
            s.ctxByID[c.reqID] = c //按ID记录context
            c.cond.Broadcast()
        } else {
            m = c.reqMsg
        }
        m.Clone()    //增加msg的引用计数, 该msg是共享模式.
        
        p := s.readyq[0] //取第一个ready 的pipe
        s.readyq = s.readyq[1:]

        // Schedule a retransmit for the future.
        c.lastPipe = p //ready的pipe保存到lastPipe里
        if c.resendTime > 0 {
            id := c.reqID
            c.resender = time.AfterFunc(c.resendTime, func() { //默认一分钟重发
                c.resendMessage(id)
            })
        }
        go p.sendCtx(c, m) //后台发送, 注意go了以后, 就脱离了锁的保护了
    }
}
```
sendCtx()会调用底层的pipe来发送, 一般发送完还会调度自己继续发送.
```go
func (p *pipe) sendCtx(c *context, m *protocol.Message) {
    s := p.s

    // Send this message.  If an error occurs, we examine the
    // error.  If it is ErrClosed, we don't schedule our self.
    if err := p.p.SendMsg(m); err != nil {
        m.Free()
        if err == protocol.ErrClosed {
            return
        }
    }
    s.Lock() //重新调度发送还是需要获取锁.
    if !s.closed && !p.closed {
        s.readyq = append(s.readyq, p)
        s.send() //重新调度
    }
    s.Unlock()
}
```

#### 发送小节
* req/rep本质上是同步模式, 其实并没有同时发送的说法
* 但即使是简单的同步发送, 这里也要走三步:
    * 第一步, 要send的message会和context来结合, 被放到socket的sendq中等待发送
    * 第二步, 调用socket.send()来触发调度. 所谓调度就是从sendq取context, 从readyq取pipe
    * 第三步, 用选定的pipe来发送这个context中的msg. 每个待发送的context都会在独立的goroutine中发送. 在本步骤会触发接下来的调度send, 即第二三步是反复互相调用的, 直到所有待发的msg都发送完成.
* 综上, 这里的设计是典型的同步异步化, 再同步化的过程. 同步异步化实际上是先缓存(指针)再调度的模式, 为的是快速从发送返回; 再次同步化是为了模式简单?
* SendMsg一般情况下会快速返回, 但在后台发送
* 发送失败会重传

#### NewSocket
看了send, 我们再回过头来从最开始的NewSocket看起.NewSocket是对上的顶层API
在实现上, req用了core的MakeSocket接口, 传入一个req.socket实例当作protocolBase
```go
// NewSocket allocates a new Socket using the REQ protocol.
func NewSocket() (protocol.Socket, error) {
    return protocol.MakeSocket(NewProtocol()), nil
}

// NewProtocol allocates a new protocol implementation.
func NewProtocol() protocol.Protocol {
    s := &socket{
        nextID:  uint32(time.Now().UnixNano()), // quasi-random
        ctxs:    make(map[*context]struct{}),
        ctxByID: make(map[uint32]*context),
    }
    s.defCtx = &context{
        s:          s,
        cond:       sync.NewCond(s), //NewCond需要传入一个locker, 而s匿名包含了sync.Mutex, 就是个locker
        resendTime: time.Minute,
    }
    s.ctxs[s.defCtx] = struct{}{}
    return s
}
```

#### AddPipe方法
req.socket的AddPipe方法会被core.socket在顶层dial和listen的时候有新的conn产生时调用, 会把transport层的conn"通道"化, 保存在req.socket的readyq里. 对REQ类型的socket来说, 所谓的readyq就是所有对端能提供REP类型的socket.
```go
func (s *socket) AddPipe(pp protocol.Pipe) error {
    p := &pipe{
        p: pp,
        s: s,
    }
    pp.SetPrivate(p)
    s.Lock()
    defer s.Unlock()
    if s.closed {
        return protocol.ErrClosed
    }
    s.readyq = append(s.readyq, p)
    s.send() //有新的rep连接了, 触发一次调度send
    go p.receiver() //后面会讲
    return nil
}
```

#### RecvMsg
无论是send还是recv msg, 都要在context上下文中进行. 因为不同于linux概念上的socket只管"通道"; 这里的socket的概念是应用场景下的如何使用socket的抽象, 是有状态的, 必须在状态上下文中进行.
```go
func (s *socket) SendMsg(m *protocol.Message) error {
    return s.defCtx.SendMsg(m)
}

func (s *socket) RecvMsg() (*protocol.Message, error) {
    return s.defCtx.RecvMsg()
}
```

ok, 从RecvMsg开始: RecvMsg是要看状态的, 而且这个函数并不是调用原始接口recv, 而是"等待"msg到位 -- 应该有个后台的routine一直在收包.
```go
func (c *context) RecvMsg() (*protocol.Message, error) {
    s := c.s
    s.Lock()
    defer s.Unlock()
    if s.closed || c.closed {
        return nil, protocol.ErrClosed
    }
    if c.recvWait || c.reqID == 0 {
        return nil, protocol.ErrProtoState
    }
    c.recvWait = true
    id := c.reqID
    expired := false

    if c.recvExpire > 0 {
        c.recvTimer = time.AfterFunc(c.recvExpire, func() {
            s.Lock()
            if c.reqID == id {
                expired = true
                c.cancel()
            }
            s.Unlock()
        })
    }

    for id == c.reqID && c.repMsg == nil {
        c.cond.Wait() //在这里等待其他routine收包完成. 注意, cond.Wait会自动unlock s的mutex锁, 这是c.cond初始化时指明的. wait返回这个tmux锁会自动lock. 也就是说wait期间, s的mutex锁是开放的. 另外, go的mutex锁和routine没有关系, 可以在一个routine里lock, 但安排其他routine去unlock
    } //另外, cond.Wait是在for里循环的, 是有条件退出的. 即满足条件还是继续wait, 即使中间被唤醒过.

    m := c.repMsg
    c.reqID = 0
    c.repMsg = nil
    c.recvWait = false
    c.cond.Broadcast() //到这里是broadcast给谁呢?

    if m == nil {
        if expired {
            return nil, protocol.ErrRecvTimeout
        }
        if c.closed {
            return nil, protocol.ErrClosed
        }
        return nil, protocol.ErrCanceled
    }
    return m, nil
}
```

注: context.RecvMsg是一处cond.Wait()点, 另外一个点发生在context.SendMsg. 这两个地方的wait()都是在条件循环里, 即使广播式的`c.cond.Broadcast()`也没关系. 不是自己想要的退出条件, 还是会继续wait.

#### receiver()函数

前面提到, 当顶层比如dial()成功了之后, 系统会把一个pipe实例AddPipe()到protocol, 也就是add到这里.
这个add流程的最后, 会
`go p.receiver()` 也就是说一个握了手的连接, 都对应一个go routine, 来receive msg, 可以想象, 这个receiver一定是个for循环, 从底层收了msg之后来缓存到哪里.

```go
func (p *pipe) receiver() {
    s := p.s
    for {
        m := p.p.RecvMsg() //从底层收完整的msg
        if m == nil { //m为nil的时候, 说明底层recv出现了错误, 系统会关闭错误连接, 启动redial流程新建连接. 
            break
        }
        if len(m.Body) < 4 {
            m.Free()
            continue
        }
        m.Header = append(m.Header, m.Body[:4]...) //m.Body的前4个字节肯定被REP当作特殊用处了...
        m.Body = m.Body[4:]

        id := binary.BigEndian.Uint32(m.Header) //还原发送时的ID

        s.Lock()
        // Since we just received a reply, stick our send at the
        // head of the list, since that's a good indication that
        // we're ready for another request.
        for i, rp := range s.readyq {
            if p == rp { //这个pipe可以收rep, 把它换到readyq的头. readyq的头负责发送
                s.readyq[0], s.readyq[i] = s.readyq[i], s.readyq[0]
                break
            }
        }

        if c, ok := s.ctxByID[id]; ok { //按ID找到发送的context
            c.unscheduleSend()
            c.reqMsg.Free() //到这里已经接收到了REP, 肯定之前发送的REQ的msg就可以free了. 
            c.reqMsg = nil
            c.repMsg = m //收包就是收到c.repMsg中了.
            delete(s.ctxByID, id)
            if c.resender != nil {
                c.resender.Stop() //也不用resend了
                c.resender = nil
            }
            c.cond.Broadcast() //RecvMsg函数里正在等着这个广播呢
        } else {
            // No matching receiver so just drop it.
            m.Free()
        }
        s.Unlock()
    }

    go p.Close()
}
```

#### 接收小结
每个成功握手协商的连接都会被add到protocol里, 被protocol知道. protocol会启动一个receiver routine, 这个receiver负责收包. 然后从headder还原发送时的ID, 查表得到当时的context. 然后把接收到的msg放到`c.repMsg = m`, 最后返回这个msg

#### REQ小结
从用户侧看来, req的使用很简单:
```go
sock, err = req.NewSocket() //新建req的socket, 同时也新建了默认的context; 标准API的send recv都走默认的context
err = sock.Dial(url) //核心层会调用url代表的transport类型的Dial, 成功就AddPipe(); 失败会重试, 知道返回errClosed
sock.Send([]byte("DATE"))
msg, err = sock.Recv()
```

* req.NewSocket()调用核心层的MakeSocket(REQ的接口实例)来创建socket. 特别的, 默认的最大rx size是1M
* REQ的socket有
    * 默认的context
    * sendq 代表的要发送的context
    * readq 代表了所有可用的连接
    * nextID 用于给每个req分配一个唯一的ID, 这个ID用来反查ctxByID表得到context
* req.socket.AddPipe()会被核心层框架在新连接建立成功后调用. 这里的AddPipe()启动了receiver routine来不断的收报文.
    * 每个新连接都会AddPipe(), 所以一个连接一个receiver routine
    * 从收到的报文的Header部分恢复request ID, 这个ID是发送的时候填的, 代表了一个req的唯一存在.
    * 用这个ID查到当时发送的context, 并给这个context的等待routine发广播唤醒
* socket的接收必须结合context来接收, 默认使用default context来接收
    * context和具体的连接(pipe)没有绑定关系
    * RecvMsg()是用户行为, 没有msg的时候会阻塞. 某个连接的receiver routine收到报文后, 查到是这个context的报文, 其发送的广播唤醒会解除本RecvMsg路径上的wait.
    * RecvMsg()还负责唤醒本context上等待的SendMsg()
* SendMsg也是要结合context, 默认也是默认的context来发送
    * 得到requestID, 和context一起记录到socket的sendq中
    * 真正的发送实际上有点像softirq, 是个触发点: 调用send()的时候, 实际上是trigger了一次后台发送序列, 该序列中会把所有之前的报文都发出去. 真正的"通道"级发送是每个报文都在后台发送`go p.sendCtx(c, m)`; 在没有真正发送完成之前, 就"广播"到该context可以继续往下走了(见下一条), 每个待发的msg都"广播"一次.
    * 如果本context还是在发本msg, 或者没有超时, 就一直wait(一般这个条件不成立)
    * 综上, 发送是softirq式的触发一波异步发送(`go p.sendCtx(c, m)`), 但不用等真正的报文从"通道"socket发送完毕.
* 用户侧发送和接收都有超时设定
* 发送有重发机制, 默认1分钟重发. 但似乎性能很堪忧, 因为每个sendq里面的context, 在真正发送之前, 都无条件起一个1分钟定时器来重发. 就不能发送失败了再起定时器重发吗?

### protocol/rep/rep.go
#### NewSocket
NewSocket的套路和REQ一样: 调用核心层protocol的函数MakeSocket, 传入自己的接口实现
```go
// NewSocket allocates a new Socket using the REP protocol.
func NewSocket() (protocol.Socket, error) {
    //传入的参数是REP类型的protocol的实现的实例
    return protocol.MakeSocket(NewProtocol()), nil
}
```
REP类型的protocol的实现的实例:
```go
// NewProtocol allocates a protocol state for the REP protocol.
func NewProtocol() protocol.Protocol {
    s := &socket{
        ttl:      8, //默认最大支持8跳, 即中间有7个"router"存在
        contexts: make(map[*context]struct{}),
        recvQ:    make(chan recvQEntry), // unbuffered! 注释写的很清楚, 非缓冲的channel
        master: &context{
            closeQ: make(chan struct{}),
        },
    }
    s.master.s = s
    s.contexts[s.master] = struct{}{}
    return s
}
```
其核心实现结构体:
```go
type socket struct {
    closed   bool
    ttl      int
    sendQLen int
    recvQ    chan recvQEntry //这个socket只有recvQ, 没有sendQ? 而且这个recvQ可以大约认为只有1个槽位.
    contexts map[*context]struct{}
    master   *context
    sync.Mutex
}
```

#### AddPipe
在AddPipe()的时候, 同时起了sender routine和receiver routine.
这里的protocol.Pipe是对底层transport.Pipe的封装, 实现了核心层ProtocolPipe的接口要求.
```go
func (s *socket) AddPipe(pp protocol.Pipe) error {
    p := &pipe{
        p:      pp,
        s:      s,
        sendQ:  make(chan *protocol.Message, s.sendQLen), //sendQlen是有Q size的
        closeQ: make(chan struct{}),
    }
    pp.SetPrivate(p) //各种回调函数变身成了名正言顺的接口的使用
    s.Lock()
    if s.closed {
        s.Unlock()
        return protocol.ErrClosed
    }
    go p.sender() //每个连接一个sender
    go p.receiver() //每个连接一个receiver
    s.Unlock()
    return nil
}

func (s *socket) RemovePipe(pp protocol.Pipe) {
    p := pp.GetPrivate().(*pipe) //这里对应了AddPipe的时候SetPrivate(存钱), 这里就来取钱了.
    close(p.closeQ)
}
```
注意, 

* 这里的sendQLen默认为0; 但一般会调用`SetOption()`接口把`protocol.OptionWriteQLen`设为1, 需要手动设置.

```go
protocol/rep/rep_test.go-101-   p := GetSocket(t, xreq.NewSocket)
protocol/rep/rep_test.go:102:   MustSucceed(t, s.SetOption(mangos.OptionWriteQLen, 1))
protocol/rep/rep_test.go-103-   MustSucceed(t, p.SetOption(mangos.OptionReadQLen, 1))
```
本文语境下的
* socket有recvQ
* pipe有sendQ

#### 每个连接一个sender
从sendQ里面拿一个msg, 发送; 发送失败就关闭这个pipe.
```go
func (p *pipe) sender() {
    for {
        select {
        case m := <-p.sendQ:
            if p.p.SendMsg(m) != nil {
                p.close()
                return
            }
        case <-p.closeQ:
            return
        }
    }
}
```

#### 每个连接一个receiver
从底层通道收包, 检查hops是否超过ttl
最后放到socket的recvQ中
```go
func (p *pipe) receiver() {
    s := p.s
outer:
    for {
        m := p.p.RecvMsg() //从protocol pipe收msg, 其具体实现在core/pipe.go; 最终是transport层收包
        if m == nil {
            break
        }

        // Move backtrace from body to header.
        // 每4个字节表示一跳, 最开始都会有一跳; 最高位不是0就不是一跳
        // hop信息是保存在body里面的, 因为header好像是固定字节的, 不可能把所有hop都放进去.
        hops := 0
        for {
            if hops >= s.ttl {
                m.Free() // ErrTooManyHops
                continue outer
            }
            hops++
            if len(m.Body) < 4 {
                m.Free() // ErrGarbled
                continue outer
            }
            m.Header = append(m.Header, m.Body[:4]...)
            m.Body = m.Body[4:]
            // Check for high order bit set (0x80000000, big endian)
            if m.Header[len(m.Header)-4]&0x80 != 0 {
                break
            }
        }

        entry := recvQEntry{m: m, p: p}
        select {
        case s.recvQ <- entry: //收到放到recvQ
        case <-p.closeQ:
            // Either the pipe or the socket has closed (which
            // closes the pipe.)  In either case, we have no
            // way to return a response, so we have to abandon.
            m.Free()
            break outer
        }
    }
    go p.close()
}
```

#### RecvMsg
用户调用Recv, core会调用到这里
```go
func (c *context) RecvMsg() (*protocol.Message, error) {
    s := c.s
    s.Lock()

    //先是检查些状态
    if c.closed {
        s.Unlock()
        return nil, protocol.ErrClosed
    }
    //在这个recv没有结束之前, 不能再次recv; 就是说recv路径下不能有并发
    if c.recvWait {
        s.Unlock()
        return nil, protocol.ErrProtoState
    }
    c.recvWait = true

    cq := c.closeQ
    wq := nilQ
    expireTime := c.recvExpire
    s.Unlock()

    if expireTime > 0 {
        wq = time.After(expireTime)
    }

    var err error
    var m *protocol.Message
    var p *pipe

    select {
    case entry := <-s.recvQ: //从recvQ里面取一个. 这个recvQ是unbuffered的. 所有连接的receiver routine都会往这里写. 但读就只有用户态API调用下来, 而且整个都是unbuffered.
        m, p = entry.m, entry.p
    case <-wq:
        err = protocol.ErrRecvTimeout
    case <-cq:
        err = protocol.ErrClosed
    }

    s.Lock()

    if m != nil {
        c.backtrace = append([]byte{}, m.Header...) //这次的header保存在context的backtrace里, 注意backtrace只有一个msg的header
        m.Header = nil
        c.recvPipe = p //保存从哪个pipe来的msg
    }
    c.recvWait = false
    s.Unlock()
    return m, err
}
```

#### SendMsg
用户调用send, 核心层会调用到这里
rep服务器的逻辑是, 必须先recv, 再send, 这两者必须成对出现. 而且中间不能有其他的recv或send.
```go
func (c *context) SendMsg(m *protocol.Message) error {
    r := c.s
    r.Lock()

    if r.closed || c.closed {
        r.Unlock()
        return protocol.ErrClosed
    }
    if c.backtrace == nil { //没有recv过, 状态错误
        r.Unlock()
        return protocol.ErrProtoState
    }
    p := c.recvPipe //发送来的通道
    c.recvPipe = nil

    bestEffort := c.bestEffort
    timeQ := nilQ
    if bestEffort {
        timeQ = closedQ
    } else if c.sendExpire > 0 {
        timeQ = time.After(c.sendExpire)
    }

    m.Header = c.backtrace //回复给client的header就是client当时发过来的, 一模一样.
    c.backtrace = nil
    cq := c.closeQ
    r.Unlock()

    select { //这个是经典的select结构:带timeout, 带close的选择器
    case <-cq:
        m.Header = nil
        return protocol.ErrClosed
    case <-p.closeQ:
        // Pipe closed, so no way to get it to the recipient.
        // Just discard the message.
        m.Free()
        return nil
    case <-timeQ: //这里有个bug: 在best effort模式下, timeQ是已经关闭的Q, 根据select语义, 假如sendQ也能写, select会随机选一个case执行; 那么m可能根本就没有被尝试发送到sendQ中, 因为timeQ先执行了.
        if bestEffort {
            // No way to report to caller, so just discard
            // the message.
            m.Free()
            return nil
        }
        m.Header = nil
        return protocol.ErrSendTimeout

    case p.sendQ <- m: //"发送"到收报文的pipe的sendQ中
        return nil
    }
}
```

#### REP小节
从用户侧看来, 使用rep很简单:
```go
sock, err = rep.NewSocket() //新建一个rep socket实例
err = sock.Listen(url) //接受连接. 核心层有个goroutine会循环等待新连接. 每个成功建立的连接都会调用protocol的AddPipe()动作, 后者会起后台的sender和receiver
msg, err = sock.Recv() //用户发起一次recv, 会从socket.recvQ中接收一个msg, 这样"众多"的连接的receiver routine的写recvQ的动作才能往下走. 这个recvQ是阻塞的.

用户在recv和send之间做业务逻辑
err = sock.Send([]byte(d)) //因为recv记录了来的路径到上下文, send的reply会根据上下文沿路返回到client
```

* 所有连接背后的receiver收包后, 都会往`s.recvQ`中写一个entry(也就是msg), 这个recvQ是unbuffered.即所有写都会阻塞, 同时只有一个能写; 在用户没有调用最顶层的recv时, 全部连接的收包都不能继续.
* 当用户调用了顶层recv处理了一个req后, 下一个req的msg会被送入`s.recvQ`
* receiver收包后, 会把msg和代表连接信息的pipe一起, 发给`s.recvQ`
* 用户recv并进行了业务逻辑的处理后, 调用send, 把相应msg沿路返回发送回client.
* recv和send时严格的lock step模式. 即一个context同时只能处理一个请求; 我认为如果业务逻辑复杂, 
    * 要么调用顶层API`OpenContext()`创建并使用多context来并发处理 -- 可行.
    * 要么就用当前context, 但用go的方式处理业务逻辑? -- 不行, 因为recv和send之间用context来传递上下文, 比如recv后, 用`c.backtrace`保存req的header; send的时候又要用这个header. 如果产生并发, `c.backtrace`会被后续的新的recv覆盖.
* rep支持多跳(hops), hops信息被保存在msg.Body里面; 在receiver里面, hops信息被搬到msg.Header里
* 在socket级别只有一个unbuffered的recvQ, 没有sendQ; sendQ是代表连接的pipe的属性. 在发送的时候, rep msg会被先放到对应连接的sendQ里, 在sender routine里, 调用protocol pipe来真正发送.
    * 不要被protocol pipe的名字迷惑, 它实际上是核心层core对transport的pipe的包装.
* 不要使用bestEffort选项, rep会随机"不发"响应.
* 发送 reply失败没有重传, 超时了会返回`protocol.ErrSendTimeout`

### req rep疑问
* 为什么req的设计思路是sync.cond + softirq式的延迟执行, 而rep的设计思路是简单明了的channel?
* 在req和rep的receiver实现中, 都有下面的操作: 把body移动4个字节给header, 如果说body的前四个字节有特殊意义, 但为什么没有找到对应的发送时候填入的这四个字节? 他们是在哪里填入的?
```go
m.Header = append(m.Header, m.Body[:4]...)
m.Body = m.Body[4:]
```
答: 他们是在transport层填入的. 在transport层看来, header和body是一体的, transport先计算总的size(int64), 然后把(size, header, body)写入通道; 而接收的时候, 先用一次系统调用得出size, 再把剩下的内容全部放到body中. 这就解释了为什么上面的protocol层的代码中, 要从body中还原header了...

### xreq和xrep
xreq和xrep是简化版的req和rep, 他们都是同一个协议族.
* 他们都没有context的概念
* 实例代码在examples/raw

#### xreq
xreq似乎改用了channel, 更简单, 功能似乎也删减了...
* xreq的socket有recvQ和sendQ的channel, 里面是msg, 默认都是128个长度
* 在AddPipe()阶段改成了和上面rep一样的每个连接都有后台的sender和receiver
* 在receiver中, 还是会把对端发来的body的头四个字节当作header, 但这次是直接当header, 而不是append
```go
m.Header = m.Body[:4]
m.Body = m.Body[4:]
```
* 竟然还是有timeQ的bug????
* xreq没有失败自动重传

#### xrep
* 默认的recvQ有128的长度, 而rep是unbuffered
* AddPipe()依然有receiver和sender后台routine
* 取消了context, 在接收msg的时候, 把pipe信息直接加到msg header中.
* 发送msg的时候, 看header就知道pipe ID, 就能原路返回
* 这样搞就必须要求用户直接使用RecvMsg和SendMsg

### 再看核心层的核心价值
* 提供了socket对象的实现, 这个对象被protocol层传递给用户, 它就是用户看到的socket.
    * 这个socket对象持有的数据都是私有的, 对外只提供接口
    * 这个socket是client和server的结合体, 既有listener列表, 又有dialer列表
    * 持有代表连接的核心对象pipes, 被组织成按ID(unit32)查询的map
    * 有一把全局socket锁
    * 有其他的属性和标记, 比如是否异步dial, 最大接收size等.
* protocol层会调用core的MakeSocket()来创建socket, 核心层只是新建了一个socket结构体返回, 没有任何其他的routine的创建.
* 核心动作Dial和DialOptions最终由用户调用. 注意, 此时socket就已经知道transport的具体类型了. 实际动作包括两步:
    * 先调用和transport约定好的的NewDialer()接口`td, err := t.NewDialer(addr, s)`, 并包装这个`td`, 构成核心层的`dialer`结构体. 最后把这个结构体放到`s.dialers`中.
    * 调用同是核心层的dialer.go中的Dial
        * 检查状态, 调用transport层的Dial方法; 连接成功会调用核心层的addPipe()方法, 后面会讲.
        * redial模式下会后台dial, 这里提前返回nil; 后台redial有重试的避让策略, 避免短时间大量不断重试
        * 非redial模式下, dial不成功马上返回err
* 核心动作Listen是server端的用户动作
    * 第一步也是处理完option后, 包装transport层的`tl, err := t.NewListener(addr, s)`对象`tl`, 做为核心层的listener实例.
    * 第二步是调用transport的listen, 这个listen实际上是bind+listen的原始socket的动作
    * 第三步是起个后台routine来做循环`l.l.Accept()`, 并把建立好的pipe加到socket: `l.s.addPipe(tp, nil, l)`
* 核心动作在Dial有重试, Listen有后台routine做循环accept, 这些"附加"的功能是核心层提供的.
* 核心层的Send/SendMsg, Recv/RecvMsg都是用户调用触发的, 都是直接调用protocol层的SendMsg/RecvMsg
* 核心层的核心是pipe, 每个连接都会addPipe(); 核心层的pipe是个混合体, 有dialer和listener, 但一般场景下, 不是同时生效.
    * 每个连接, 不管是client dial的来的, 还是server listen得来的, 都对应一个核心层的pipe; 虽然实现在核心层, 但对外是以protocol.Pipe示人的, 即核心层的pipe实现了protocol层(更确切的说法是全局层, 对外层)Pipe规定的方法.
    * addPipe()会新生成一个pipe实例, 并分配唯一的ID(unit32); 这个pipe实例被socket保存在map表里, 方便后面根据ID查询
    * addPipe()会调用与protocol层约定好的AddPipe()方法`s.proto.AddPipe(p)`; 后者会保存p以便发送, 接收
* 核心层的pipe的Send和Recv以及Close, 提供失败重连的功能.
    * 因为在addPipe阶段, 会把核心层的pipe实例以protocol.Pipe接口的形式传递给protocol, protocol想真正发送 接收msg的时候, 会调用到核心层pipe提供的SendMsg()和RecvMsg()方法, 这两个方法都是调用transport层的发送接收方法.
    * 特别的, 从核心层收到的msg, 都会记录pipe信息到`msg.Pipe`域.
    * 任何transport层的send recv失败发生时, 都会调用p.Close(); 这个close的动作会触发重建连接(redial)动作, 说明系统任务这个transport通道已然不行了, 要重建. 同时, 也会调用protocol层的RemovePipe()接口, 来让protocol层知道这个pipe已经失效. 这是个很好的策略, 即发送失败不要简单重试, 而是重新建链.
    
* 核心层的Send()实现是拷贝式send, 即用户提供的buffer会被拷贝到新buffer后再send; Recv()也是拷贝式的,即接收的msg.Body会被拷贝到新buffer后返回新buffer; 题外: 作者特别喜欢用append来拷贝buffer
`msg.Body = append(msg.Body, b...)`
    * 但对应的SendMsg()和RecvMsg()就都不会有新buffer产生.
    
#### 总结
核心层对上直接承接了用户api的接口, 它既规定了protocol要实现的接口: AddPipe() SendMsg() RecvMsg(), 又规定了transport必须实现的接口: NewListener(), NewDialer(), Dial(), Listen(), Accept(). 通过这些规定的接口, 核心层把protocol层和transport层解耦了.  
核心层对连接的抽象是combine的, 即把client和server的dial和listen过程独立抽象, 但他们的结果都是新建一个pipe, 而这个pipe就承载了后面的发送, 接收功能. 核心层在建立连接后, 会把自己对protocol的承诺(也就是protocol.pipe)的实例, 传递`addPipe(p)`给protocol层, 这个p就代表了transport的抽象.
同时, 核心层还提供如下功能:
* 核心层会在redial模式下重试dial
* 核心层会启动后台routine帮助server来accept连接
* 核心层在调用transport层的发送接收api失败时, 会断开当前连接, 自动重建新连接.

核心层并没有提供以下机制:
* 队列
* 反压
* 统计
* 调度

即核心层负责抽象和解耦, 只提供最基本的功能.

