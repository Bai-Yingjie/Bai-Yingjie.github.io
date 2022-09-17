- [socket](#socket)
  - [sock地址](#sock地址)
  - [接口名称, 比如eth0](#接口名称-比如eth0)
  - [local socket, 用于进程间通信 PF_LOCAL PF_UNIX PF_FILE](#local-socket-用于进程间通信-pf_local-pf_unix-pf_file)
  - [网络socket PF_INET PF_INET6](#网络socket-pf_inet-pf_inet6)
    - [主机名称, 比如alpha.gnu.org](#主机名称-比如alphagnuorg)
    - [端口和服务](#端口和服务)
    - [网络字节序](#网络字节序)
    - [举例](#举例)
  - [socket接口](#socket接口)
  - [socket读写](#socket读写)
  - [stream client举例](#stream-client举例)
  - [stream server举例](#stream-server举例)
  - [oob数据](#oob数据)
  - [datagram](#datagram)
  - [inetd守护进程](#inetd守护进程)
  - [socket配置](#socket配置)
- [terminal](#terminal)
  - [terminal的模式](#terminal的模式)
  - [对terminal的改动是对dev来说是全局的](#对terminal的改动是对dev来说是全局的)
  - [输入输出和控制属性, 见man tcsetattr](#输入输出和控制属性-见man-tcsetattr)
  - [local属性, ICANON ECHO* ISIG(ctrl+c, ctrl+\, ctrl+z)](#local属性-icanon-echo-isigctrlc-ctrl-ctrlz)
  - [虚拟终端, <-->master side <--> slave side<-->, 一对一](#虚拟终端---master-side----slave-side---一对一)
  - [BSD方式打开pty](#bsd方式打开pty)
- [syslog](#syslog)
  - [log的处理方式:](#log的处理方式)
  - [syslogd会处理](#syslogd会处理)
  - [syslogd的两个参数, 用一个数字来同时表示facility/priority](#syslogd的两个参数-用一个数字来同时表示facilitypriority)
  - [syslog接口](#syslog接口)
- [数学](#数学)
- [日期与时间](#日期与时间)
  - [格式化时间字符串](#格式化时间字符串)
  - [定时器](#定时器)
  - [sleep](#sleep)
  - [纳秒睡眠](#纳秒睡眠)
- [系统资源](#系统资源)
  - [资源统计](#资源统计)
  - [限制, 两种](#限制-两种)
  - [ulimit](#ulimit)
  - [实时进程优先级](#实时进程优先级)
  - [进程调度](#进程调度)
  - [nice值](#nice值)
  - [绑定cpus](#绑定cpus)
  - [可用内存页数目](#可用内存页数目)
  - [cpu个数](#cpu个数)
  - [当前系统负载, 1 5 15分钟平均负载](#当前系统负载-1-5-15分钟平均负载)

# socket
*  SOCK_STREAM
*  SOCK_DGRAM
*  SOCK_RAW  
PF_*: protocol family  
AF_*: address family 与PF_*是一一对应的
## sock地址
```c
#include <sys/socket.h>
```
AF_LOCAL AF_UNIX AF_FILE: 都是一回事, 进程间通信  
AF_INET:  
AF_INET6:  
```c
//设置socket地址
int bind (int socket, struct sockaddr *addr, socklen t length)
//读取socket地址
int getsockname (int socket, struct sockaddr *addr, socklen t *length-ptr)
```
## 接口名称, 比如eth0
```c
#include <net/if.h>
//接口名到index
unsigned int if_nametoindex (const char *ifname)
//index到接口名
char * if_indextoname (unsigned int ifindex, char *ifname)
//获取全部接口
struct if_nameindex * if_nameindex (void)
//释放if_nameindex分配的内存
void if_freenameindex (struct if nameindex *ptr)
```

## local socket, 用于进程间通信 PF_LOCAL PF_UNIX PF_FILE
local socket又称为unix或file socket, socket地址就是个文件名, 一般放在/tmp下面
当一个local socket关闭时, 也需要delete这个文件
用struct sockaddr_un表示地址
举例:
```c
#include <stddef.h>
#include <stdio.h>
#include <errno.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/un.h>
int
make_named_socket(const char *filename)
{
    struct sockaddr_un name;
    int sock;
    size_t size;
    /* Create the socket. */
    sock = socket(PF_LOCAL, SOCK_DGRAM, 0);
 
    if (sock < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }
 
    /* Bind a name to the socket. */
    name.sun_family = AF_LOCAL;
    strncpy(name.sun_path, filename, sizeof(name.sun_path));
    name.sun_path[sizeof(name.sun_path) - 1] = ’\0’;
    /* The size of the address is
    the offset of the start of the filename,
    plus its length (not including the terminating null byte).
    Alternatively you can just do:
    size = SUN LEN (&name);
    */
    size = (offsetof(struct sockaddr_un, sun_path)
            + strlen(name.sun_path));
 
    if (bind(sock, (struct sockaddr *) &name, size) < 0) {
        perror("bind");
        exit(EXIT_FAILURE);
    }
 
    return sock;
}
```

## 网络socket PF_INET PF_INET6 
这个socket用struct sockaddr_in或struct sockaddr_in6表示地址
ip地址用struct in_addr表示
```c
#include <arpa/inet.h>
//将192.168.1.1转换成32bit的ip地址
int inet_aton (const char *name, struct in addr *addr)
//相反
char * inet_ntoa (struct in addr addr)
```

### 主机名称, 比如alpha.gnu.org
```c
#include <netdb.h>
//用struct hostent表示一个host
struct hostent * gethostbyname (const char *name)
struct hostent * gethostbyaddr (const void *addr, socklen t length, int format)
```

### 端口和服务
```c
#include <netinet/in.h>
#include <netdb.h>
//每个服务对应struct servent
struct servent * getservbyname (const char *name, const char *proto)
struct servent * getservbyport (int port, const char *proto)
```

### 网络字节序
sin_port和sin_addr必须是网络字节序
```c
#include <netinet/in.h>
uint16_t htons (uint16 t hostshort)
uint16_t ntohs (uint16 t netshort)
uint32_t htonl (uint32 t hostlong)
uint32_t ntohl (uint32 t netlong)
```

### 举例
举例1:
```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <netinet/in.h>
int
make_socket(uint16_t port)
{
    int sock;
    struct sockaddr_in name;
    /* Create the socket. */
    sock = socket(PF_INET, SOCK_STREAM, 0);
 
    if (sock < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }
 
    /* Give the socket a name. */
    name.sin_family = AF_INET;
    name.sin_port = htons(port);
    name.sin_addr.s_addr = htonl(INADDR_ANY);
 
    if (bind(sock, (struct sockaddr *) &name, sizeof(name)) < 0) {
        perror("bind");
        exit(EXIT_FAILURE);
    }
 
    return sock;
}
```
举例2:
```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h>
void
init_sockaddr(struct sockaddr_in *name,
              const char *hostname,
              uint16_t port)
{
    struct hostent *hostinfo;
    name->sin_family = AF_INET;
    name->sin_port = htons(port);
    hostinfo = gethostbyname(hostname);
 
    if (hostinfo == NULL) {
        fprintf(stderr, "Unknown host %s.\n", hostname);
        exit(EXIT_FAILURE);
    }
 
    name->sin_addr = *(struct in_addr *) hostinfo->h_addr;
}
```

## socket接口
```c
#include <sys/socket.h>
//open
int socket (int namespace, int style, int protocol)
/*close
**可以直接close文件描述符, 此时colse会尝试发送未完成的data
**也可以int shutdown (int socket, int how), how可以指定
**0 拒绝接受data
**1 停止发送
**2 收发都停
*/
 
//创建一对socket
int socketpair (int namespace, int style, int protocol, int filedes[2])
//client连接, 阻塞等待server回应. 除非设了nonblocking模式
int connect (int socket, struct sockaddr *addr, socklen t length)
//server端
int listen (int socket, int n)
int accept (int socket, struct sockaddr *addr, socklen t *length_ptr) //addr是对端的地址
//获取对端地址, 前提是这个socket已经connect
int getpeername (int socket, struct sockaddr *addr, socklen t *length-ptr)
```

## socket读写
可以用read和write, 但使用send和recv能获得更多的控制.
其实, write就相当于send的flags为0
不管是write还是send, 向已经损坏的socket写会得到SIGPIPE
```c
ssize_t send (int socket, const void *buffer, size t size, int flags)
ssize_t recv (int socket, void *buffer, size t size, int flags)
```
flags:
*  MSG_OOB: out of band data
*  MSG_PEEK: 用而不取
*  MSG_DONTROUTE

## stream client举例
```c
#include <stdio.h>
#include <errno.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h>
#define PORT 5555
#define MESSAGE "Yow!!! Are we having fun yet?!?"
#define SERVERHOST "www.gnu.org"
write_to_server(int filedes)
{
    int nbytes;
    nbytes = write(filedes, MESSAGE, strlen(MESSAGE) + 1);
 
    if (nbytes < 0) {
        perror("write");
        exit(EXIT_FAILURE);
    }
}
int
main(void)
{
    extern void init_sockaddr(struct sockaddr_in * name,
                              const char *hostname,
                              uint16_t port);
    int sock;
    struct sockaddr_in servername;
    /* Create the socket. */
    sock = socket(PF_INET, SOCK_STREAM, 0);
 
    if (sock < 0) {
        perror("socket (client)");
        exit(EXIT_FAILURE);
    }
 
    /* Connect to the server. */
    init_sockaddr(&servername, SERVERHOST, PORT);
 
    if (0 > connect(sock,
                    (struct sockaddr *) &servername,
                    sizeof(servername))) {
        perror("connect (client)");
        exit(EXIT_FAILURE);
    }
 
    /* Send data to the server. */
    write_to_server(sock);
    close(sock);
    exit(EXIT_SUCCESS);
}
```
## stream server举例
```c
#include <stdio.h>
#include <errno.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h>
#define PORT 5555
#define MAXMSG 512
int
read_from_client(int filedes)
{
    char buffer[MAXMSG];
    int nbytes;
    nbytes = read(filedes, buffer, MAXMSG);
 
    if (nbytes < 0) {
        /* Read error. */
        perror("read");
        exit(EXIT_FAILURE);
    } else if (nbytes == 0)
        /* End-of-file. */
        return -1;
    else {
        /* Data read. */
        fprintf(stderr, "Server: got message: ‘%s’\n", buffer);
        return 0;
    }
}
int
main(void)
{
    extern int make_socket(uint16_t port);
    int sock;
    fd_set active_fd_set, read_fd_set;
    int i;
    struct sockaddr_in clientname;
    size_t size;
    /* Create the socket and set it up to accept connections. */
    sock = make_socket(PORT);
 
    if (listen(sock, 1) < 0) {
        perror("listen");
        exit(EXIT_FAILURE);
    }
 
    /* Initialize the set of active sockets. */
    FD_ZERO(&active_fd_set);
    FD_SET(sock, &active_fd_set);
 
    while (1) {
        /* Block until input arrives on one or more active sockets. */
        read_fd_set = active_fd_set;
 
        if (select(FD_SETSIZE, &read_fd_set, NULL, NULL, NULL) < 0) {
            perror("select");
            exit(EXIT_FAILURE);
        }
 
        /* Service all the sockets with input pending. */
        for (i = 0; i < FD_SETSIZE; ++i)
            if (FD_ISSET(i, &read_fd_set)) {
                if (i == sock) {
                    /* Connection request on original socket. */
                    int new;
                    size = sizeof(clientname);
                    new = accept(sock,
                                 (struct sockaddr *) &clientname,
                                 &size);
 
                    if (new < 0) {
                        perror("accept");
                        exit(EXIT_FAILURE);
                    }
 
                    fprintf(stderr,
                            "Server: connect from host %s, port %hd.\n",
                            inet_ntoa(clientname.sin_addr),
                            ntohs(clientname.sin_port));
                    FD_SET(new, &active_fd_set);
                } else {
                    /* Data arriving on an already-connected socket. */
                    if (read_from_client(i) < 0) {
                        close(i);
                        FD_CLR(i, &active_fd_set);
                    }
                }
            }
    }
}
```

## oob数据
OOB数据通过send和recv传递, 使用flag MSG_OOB, 拥有高优先级, 不会和普通数据一起排队.
OOB会触发信号SIGURG
发送OOB时, 会在普通数据流内加个mark, 表示"即将完成"OOB数据
举例:
```c
int
discard_until_mark(int socket)
{
    while (1) {
        /* This is not an arbitrary limit; any size will do. */
        char buffer[1024];
        int atmark, success;
        /* If we have reached the mark, return. */
        success = ioctl(socket, SIOCATMARK, &atmark);
 
        if (success < 0)
            perror("ioctl");
 
        if (atmark)
            return;
 
        /* Otherwise, read a bunch of ordinary data and discard it.
        This is guaranteed not to read past the mark
        if it starts before the mark. */
        success = read(socket, buffer, sizeof buffer);
 
        if (success < 0)
            perror("read");
    }
}
```
举例:
```c
struct buffer {
    char *buf;
    int size;
    struct buffer *next;
};
/* Read the out-of-band data from SOCKET and return it
as a ‘struct buffer’, which records the address of the data
and its size.
It may be necessary to read some ordinary data
in order to make room for the out-of-band data.
If so, the ordinary data are saved as a chain of buffers
found in the ‘next’ field of the value. */
struct buffer *
read_oob(int socket)
{
    struct buffer *tail = 0;
    struct buffer *list = 0;
 
    while (1) {
        /* This is an arbitrary limit.
        Does anyone know how to do this without a limit? */
#define BUF_SZ 1024
        char *buf = (char *) xmalloc(BUF_SZ);
        int success;
        int atmark;
        /* Try again to read the out-of-band data. */
        success = recv(socket, buf, BUF_SZ, MSG_OOB);
 
        if (success >= 0) {
            /* We got it, so return it. */
            struct buffer *link
                = (struct buffer *) xmalloc(sizeof(struct buffer));
            link->buf = buf;
            link->size = success;
            link->next = list;
            return link;
        }
 
        /* If we fail, see if we are at the mark. */
        success = ioctl(socket, SIOCATMARK, &atmark);
 
        if (success < 0)
            perror("ioctl");
 
        if (atmark) {
            /* At the mark; skipping past more ordinary data cannot help.
            So just wait a while. */
            sleep(1);
            continue;
        }
 
        /* Otherwise, read a bunch of ordinary data and save it.
        This is guaranteed not to read past the mark
        if it starts before the mark. */
        success = read(socket, buf, BUF_SZ);
 
        if (success < 0)
            perror("read");
 
        /* Save this data in the buffer list. */
        {
            struct buffer *link
                = (struct buffer *) xmalloc(sizeof(struct buffer));
            link->buf = buf;
            link->size = success;
 
            /* Add the new link to the end of the list. */
            if (tail)
                tail->next = link;
            else
                list = link;
 
            tail = link;
        }
    }
}
```

## datagram
```c
//发到数据addr
ssize_t sendto (int socket, const void *buffer, size t size, int flags, struct sockaddr *addr, socklen t length)
//接收数据, 源地址保存在*addr
ssize_t recvfrom (int socket, void *buffer, size t size, int flags, struct sockaddr *addr, socklen t *length-ptr)
```
举例: local+datagram 运行在同一个主机上, 进程间通信  
server
```c
#include <stdio.h>
#include <errno.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <sys/un.h>
#define SERVER "/tmp/serversocket"
#define MAXMSG 512
int
main(void)
{
    int sock;
    char message[MAXMSG];
    struct sockaddr_un name;
    size_t size;
    int nbytes;
    /* Remove the filename first, it’s ok if the call fails */
    unlink(SERVER);
    /* Make the socket, then loop endlessly. */
    sock = make_named_socket(SERVER);
 
    while (1) {
        /* Wait for a datagram. */
        size = sizeof(name);
        nbytes = recvfrom(sock, message, MAXMSG, 0,
                          (struct sockaddr *) & name, &size);
 
        if (nbytes < 0) {
            perror("recfrom (server)");
            exit(EXIT_FAILURE);
        }
 
        /* Give a diagnostic message. */
        fprintf(stderr, "Server: got message: %s\n", message);
        /* Bounce the message back to the sender. */
        nbytes = sendto(sock, message, nbytes, 0,
                        (struct sockaddr *) & name, size);
 
        if (nbytes < 0) {
            perror("sendto (server)");
            exit(EXIT_FAILURE);
        }
    }
}
```
client
```c
#include <stdio.h>
#include <errno.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <sys/un.h>
#define SERVER "/tmp/serversocket"
#define CLIENT "/tmp/mysocket"
#define MAXMSG 512
#define MESSAGE "Yow!!! Are we having fun yet?!?"
int
main(void)
{
    extern int make_named_socket(const char *name);
    int sock;
    char message[MAXMSG];
    struct sockaddr_un name;
    size_t size;
    int nbytes;
    /* Make the socket. */
    sock = make_named_socket(CLIENT);
    /* Initialize the server socket address. */
    name.sun_family = AF_LOCAL;
    strcpy(name.sun_path, SERVER);
    size = strlen(name.sun_path) + sizeof(name.sun_family);
    /* Send the datagram. */
    nbytes = sendto(sock, MESSAGE, strlen(MESSAGE) + 1, 0,
                    (struct sockaddr *) & name, size);
 
    if (nbytes < 0) {
        perror("sendto (client)");
        exit(EXIT_FAILURE);
    }
 
    /* Wait for a reply. */
    nbytes = recvfrom(sock, message, MAXMSG, 0, NULL, 0);
 
    if (nbytes < 0) {
        perror("recfrom (client)");
        exit(EXIT_FAILURE);
    }
 
    /* Print a diagnostic message. */
    fprintf(stderr, "Client: got message: %s\n", message);
    /* Clean up. */
    remove(CLIENT);
    close(sock);
}
```
## inetd守护进程
inetd根据配置文件/etc/inetd.conf来监听端口, 发现连接请求后, 就启动一个新的server进程, 运行配置文件制定的程序. 这个程序的标准输入输出已被重定向到socket.
这个配置文件格式: service style protocol wait username program arguments
service来源于/etc/services. 比如, 
ftp stream tcp nowait root /libexec/ftpd ftpd
talk dgram udp wait root /libexec/talkd talkd
## socket配置
```c
int getsockopt (int socket, int level, int optname, void *optval, socklen t *optlen-ptr)
int setsockopt (int socket, int level, int optname, const void *optval, socklen t optlen)
```
其中, level就是SOL_SOCKET
而optname可以是:
*  SO_DEBUG: 打开socket调试信息
*  SO_REUSEADDR: 使能两个socket同时使用一个端口
*  SO_KEEPALIVE: 使能周期发消息检测对端是否broken
*  SO_DONTROUTE: 报文不经过通常的routing过程, 直接发送到interface
*  SO_BROADCAST: datagram是否被广播
*  SO_OOBINLINE: OOB被放到普通数据中
*  SO_SNDBUF: 设置发送buffer大小
*  SO_RCVBUF: 设置接收buffer大小
*  SO_STYLE: 用于获取协议类型
*  SO_ERROR: 获取socket错误信息

# terminal
```c
//是否tty
int isatty (int filedes)
char * ttyname (int filedes)
```
tty的buffer在内核里, 而不是普通IObuffer

## terminal的模式
```c
#include <termios.h>
struct termios
    tcflag_t c_iflag //输入
    tcflag_t c_oflag //输出
    tcflag_t c_cflag //控制
    tcflag_t c_lflag //local
    cc_t c_cc[NCCS]
```

## 对terminal的改动是对dev来说是全局的
共享同一个terminal的其他进程也会改变.
```c
int tcgetattr (int filedes, struct termios *termios-p)
int tcsetattr (int filedes, int when, const struct termios *termios-p)
```
举例:
```c
int
set_istrip(int desc, int value)
{
    struct termios settings;
    int result;
    result = tcgetattr(desc, &settings);
 
    if (result < 0) {
        perror("error in tcgetattr");
        return 0;
    }
 
    settings.c_iflag &= ~ISTRIP;
 
    if (value)
        settings.c_iflag |= ISTRIP;
 
    result = tcsetattr(desc, TCSANOW, &settings);
 
    if (result < 0) {
        perror("error in tcsetattr");
        return 0;
    }
 
    return 1;
}
```
## 输入输出和控制属性, 见man tcsetattr
## local属性, ICANON ECHO* ISIG(ctrl+c, ctrl+\, ctrl+z) 
```c
//速率
int cfsetspeed (struct termios *termios-p, speed t speed)
//raw模式
void cfmakeraw (struct termios *termios-p)
//相当于
termios-p->c_iflag &= ~(IGNBRK|BRKINT|PARMRK|ISTRIP
                        |INLCR|IGNCR|ICRNL|IXON);
termios-p->c_oflag &= ~OPOST;
termios-p->c_lflag &= ~(ECHO|ECHONL|ICANON|ISIG|IEXTEN);
termios-p->c_cflag &= ~(CSIZE|PARENB);
termios-p->c_cflag |= CS8;
```
举例: raw模式, 不回显
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <termios.h>
/* Use this variable to remember original terminal attributes. */
struct termios saved_attributes;
void
reset_input_mode(void)
{
    tcsetattr(STDIN_FILENO, TCSANOW, &saved_attributes);
}
void
set_input_mode(void)
{
    struct termios tattr;
    char *name;
 
    /* Make sure stdin is a terminal. */
    if (!isatty(STDIN_FILENO)) {
        fprintf(stderr, "Not a terminal.\n");
        exit(EXIT_FAILURE);
    }
 
    /* Save the terminal attributes so we can restore them later. */
    tcgetattr(STDIN_FILENO, &saved_attributes);
    atexit(reset_input_mode);
    /* Set the funny terminal modes. */
    tcgetattr(STDIN_FILENO, &tattr);
    tattr.c_lflag &= ~(ICANON | ECHO); /* Clear ICANON and ECHO. */
    tattr.c_cc[VMIN] = 1;
    tattr.c_cc[VTIME] = 0;
    tcsetattr(STDIN_FILENO, TCSAFLUSH, &tattr);
}
main(void)
{
    char c;
    set_input_mode();
 
    while (1) {
        read(STDIN_FILENO, &c, 1);
 
        if (c == ’\004’) /* C-d */
            break;
        else
            putchar(c);
    }
 
    return EXIT_SUCCESS;
}
```
## 虚拟终端, <-->master side <--> slave side<-->, 一对一
```c
#include <stdlib.h>
//获取一个新的pty(master), 也可以直接open("/dev/ptmx",O_RDWR), 每次open都会返回一个独立的PTM, 并且在/dev/pts里生成一个相应的pys
int getpt (void)
//设置slave pty的权限
int grantpt (int filedes)
//在open slave之前必须先unlock
int unlockpt (int filedes)
//返回与master相关的slave的pts的文件名
char * ptsname (int filedes)
```
举例:
```c
int
open_pty_pair(int *amaster, int *aslave)
{
    int master, slave;
    char *name;
    master = getpt();
 
    if (master < 0)
        return 0;
 
    if (grantpt(master) < 0 || unlockpt(master) < 0)
        goto close_master;
 
    name = ptsname(master);
 
    if (name == NULL)
        goto close_master;
 
    slave = open(name, O_RDWR);
 
    if (slave == -1)
        goto close_master;
 
    if (isastream(slave)) {
        if (ioctl(slave, I_PUSH, "ptem") < 0
            || ioctl(slave, I_PUSH, "ldterm") < 0)
            goto close_slave;
    }
 
    *amaster = master;
    *aslave = slave;
    return 1;
close_slave:
    close(slave);
close_master:
    close(master);
    return 0;
}
```
```c
static int ptym_open (int *p_master, int *p_aux, char *p_slave_name)
{
  char *ptsnam;
 
  *p_master=open("/dev/ptmx",O_RDWR);
 
  if(*p_master < 0)
    return 0;
 
  ptsnam=ptsname(*p_master);
  if(ptsnam==NULL)
  {
    close(*p_master);
    *p_master=-1;
    return 0;
  }
 
  grantpt(*p_master);
  unlockpt(*p_master);
 
  *p_aux = open(ptsnam, O_RDWR);
  if (*p_aux < 0)
    return 0;
  strcpy(p_slave_name,ptsnam);
 
  return 1;
}
```
## BSD方式打开pty
```c
#include <pty.h>
int openpty (int *amaster, int *aslave, char *name, const struct termios *termp, const struct winsize *winp)
/*fork一个子进程, 使用刚open的slave pts作为子进程的控制终端;
**和fork类似, 返回0表示在子进程; 在父进程返回fork后的进程号. -1为错误
*/
int forkpty (int *amaster, char *name, const struct termios *termp, const struct winsize *winp)
```

# syslog
syslogd进程会处理syslog, 与之对应的是unix socket文件/dev/log, 配置文件是/etc/syslog.conf
## log的处理方式:
*  控制台
*  给某人发邮件
*  写到log文件
*  传递给另外的进程
*  丢弃
## syslogd会处理
*  网络log. 通过监听syslog UDP port
*  内核消息, Klogd传递给syslogd.
*  本地log. 通过/dev/log
## syslogd的两个参数, 用一个数字来同时表示facility/priority
*  facility: mail subsystem, FTP server, 
*  priority: debug, informational, warning, critical
## syslog接口
```c
#include <syslog.h>
//open, 选项有LOG_PERROR LOG_CONS LOG_PID, facility默认是LOG_USER
void openlog (const char *ident, int option, int facility)
//close
void closelog (void)
/*log接口, 可以不openlog 直接调用
**第一个参数是个参数对, 用宏LOG_MAKEPRI(LOG_USER, LOG_WARNING)可以生成
*/
void syslog (int facility_priority, const char *format, . . . )
```
* facility有:
  * LOG_USER LOG_MAIL LOG_DAEMON LOG_FTP 等等
* priority有: 如果只填priority, 则用默认的facility
  * LOG_EMERG LOG_ALERT LOG_CRIT LOG_ERR LOG_WARNING LOG_NOTICE LOG_INFO LOG_DEBUG
举例:
```c
#include <syslog.h>
syslog (LOG_MAKEPRI(LOG_LOCAL1, LOG_ERROR), "Unable to make network connection to %s. Error=%m", host);
```
再举例:用setlogmask()过滤掉debug和info, 这些log不会真正到syslog
```c
#include <syslog.h>
setlogmask (LOG_UPTO (LOG_NOTICE));
openlog ("exampleprog", LOG_CONS | LOG_PID | LOG_NDELAY, LOG_LOCAL1);
syslog (LOG_NOTICE, "Program started by User %d", getuid ());
syslog (LOG_INFO, "A tree falls in a forest");
closelog ();
```

# 数学
```c
//伪随机数
#include <stdlib.h>
int rand (void)
void srand (unsigned int seed)
```

# 日期与时间
```c
#include <time.h>
//间隔
double difftime (time t time1, time t time0)
//进程独占的时间, 不包括阻塞时间
#include <time.h>
clock_t clock (void)
```
举例:
```c
clock_t start, end;
double cpu_time_used;
start = clock();
... /* Do the work. */
end = clock();
cpu_time_used = ((double) (end - start)) / CLOCKS_PER_SEC;
```
```c
//进程的处理器时间
#include <sys/times.h>
clock_t times (struct tms *buffer)
//用到的结构体
struct tms //单位是clock
  clock_t tms_utime //进程用户态处理器时间
  clock_t tms_stime //进程内核态处理器时间
  clock_t tms_cutime //进程和子进程的用户态时间, 不包括还没结束的子进程
  clock_t tms_cstime //进程和子进程的内核态时间, 不包括还没结束的子进程
struct timeval
  long int tv_sec
  long int tv_usec
time_t //其实就是long int, 从1970年开始算起的seconds
//返回简单日历时间, 可以传入NULL
time_t time (time t *result)
//设置时间, 推荐用settimeofday
int stime (const time t *newtime)
//精确到微妙的日历时间, 从epoch(1970)算起
int gettimeofday (struct timeval *tp, struct timezone *tzp)
int settimeofday (const struct timeval *tp, const struct timezone *tzp)
//平滑的调整时间
int adjtime (const struct timeval *delta, struct timeval *olddelta)
//年月日小时分秒时间(broken-down time)
struct tm
  int tm_sec
  int tm_min
  int tm_hour
  int tm_mday
  int tm_mon
  int tm_year
  int tm_wday //0-6
  int tm_yday //0-365
  const char *tm_zone
 
struct tm * localtime (const time t *time)
//UTC/GMT时间
struct tm * gmtime (const time t *time)
//从broken-down时间到简单时间
time_t mktime (struct tm *brokentime)
//转换为时间字符串
char * asctime (const struct tm *brokentime)
char * ctime (const time t *time)
```

## 格式化时间字符串
```c
//从时间到字符串
size_t strftime (char *s, size t size, const char *template, const struct tm *brokentime)
//从字符串到时间
char * strptime (const char *s, const char *fmt, struct tm *tp)
```
举例:
```c
#include <time.h>
#include <stdio.h>
#define SIZE 256
int
main(void)
{
    char buffer[SIZE];
    time_t curtime;
    struct tm *loctime;
    /* Get the current time. */
    curtime = time(NULL);
    /* Convert it to local time representation. */
    loctime = localtime(&curtime);
    /* Print out the date and time in the standard format. */
    fputs(asctime(loctime), stdout);
    /* Print it out in a nice format. */
    strftime(buffer, SIZE, "Today is %A, %B %d.\n", loctime);
    fputs(buffer, stdout);
    strftime(buffer, SIZE, "The time is %I:%M %p.\n", loctime);
    fputs(buffer, stdout);
    return 0;
}
```

## 定时器
```c
struct itimerval
  struct timeval it_interval //周期
  struct timeval it_value //单次
/*which 对应三种:ITIMER_REAL, ITIMER_VIRTUAL, or ITIMER_PROF
**分别对应三种信号: SIGALRM, SIGVTALRM, SIGPROF
**在用sigaction注册信号处理函数时, SA_RESTART被用来表示被打断的系统调用自动重试.
**如果希望定时器可以打断阻塞中的系统调用, 则不要设置SA_RESTART标记
*/
int setitimer (int which, const struct itimerval *new, struct itimerval *old)
int getitimer (int which, struct itimerval *old)
//alarm是简化版的ITIMER_REAL, 传入0表示取消这个定时器
unsigned int alarm (unsigned int seconds)
```
alarm相当于:
```c
unsigned int
alarm(unsigned int seconds)
{
    struct itimerval old, new;
    new.it_interval.tv_usec = 0;
    new.it_interval.tv_sec = 0;
    new.it_value.tv_usec = 0;
    new.it_value.tv_sec = (long int) seconds;
 
    if (setitimer(ITIMER_REAL, &new, &old) < 0)
        return 0;
    else
        return old.it_value.tv_sec;
}
```

## sleep
```c
#include <unistd.h>
unsigned int sleep (unsigned int seconds)
```
注意sleep可以被信号打断而提前返回, 如果不想被打断, 可以用不包含任何fd的select(), 传入超时时间.
在GNU系统中, sleep和SIGALRM可以同时存在, 因为sleep不使用SIGALRM.

## 纳秒睡眠
```c
#include <time.h>
int nanosleep (const struct timespec *requested_time, struct timespec *remaining)
```

# 系统资源
## 资源统计
```c
#include <sys/resource.h>
int getrusage (int processes, struct rusage *rusage)
struct rusage
  struct timeval ru_utime //用户态时间
  struct timeval ru_stime //内核态时间
  long int ru_maxrss //最大run time内存, kb
  long int ru_ixrss //和其他进程共享的代码段, kb
  long int ru_idrss //不共享的数据段, kb
  long int ru_isrss //不共享的栈空间, kb
  long int ru_minflt //非io页fault
  long int ru_majflt //io页fault
  long int ru_nswap //被换出内存的时间
  long int ru_inblock //读硬盘次数
  long int ru_oublock //写硬盘次数
  long int ru_msgsnd //ipc发送次数
  long int ru_msgrcv //ipc接收次数
  long int ru_nsignals //被signal的次数
  long int ru_nvcsw //进程主动让出cpu次数
  long int ru_nivcs //进程被动让出cpu次数
```

## 限制, 两种
*  current limit: 进程可自己修改的limit
*  maximum limit: 只有超级用户才能改的limit
```c
int getrlimit (int resource, struct rlimit *rlp)
int setrlimit (int resource, const struct rlimit *rlp)
```

## ulimit
```c
#include <ulimit.h>
long int ulimit (int cmd, . . . )
```
## 实时进程优先级
实时进程0-99, 对同一个优先级的几个实时进程的调度分为:
*  FIFO: 先到先执行, 必须主动让出CPU(sched_yield)
*  RR: round robin, 同级时间片轮转. 这里的时间片不是从父进程继承而来, 在linux下面, 比普通进程的时间片小得多(150ms)
## 进程调度
```c
#include <sched.h>
//SCHED_OTHER SCHED_FIFO SCHED_RR
int sched_setscheduler (pid t pid, int policy, const struct sched param *param)
int sched_getscheduler (pid t pid)
/*我估计这个文档里的0优先级是普通优先级(100-139), 而大于0的优先级都是实时进程的优先级(0-99)
**后面提到的动态优先级是对这里的0号优先级说的
*/
//设置绝对优先级
int sched_setparam (pid t pid, const struct sched param *param)
//获取绝对优先级
int sched_getparam (pid t pid, struct sched param *param)
//最小 最大的允许优先级
int sched_get_priority_min (int policy)
int sched_get_priority_max (int policy)
//RR进程时间片
int sched_rr_get_interval (pid t pid, struct timespec *interval)
//主动让出cpu
int sched_yield (void)
```

## nice值
*  +19: 优先级最低, 时间片大约10ms
*  -20: 优先级最高, 时间片大约400ms
一般进程能够提高nice, 但不能降低nice
```c
#include <sys/resource.h>
//获取nice值, class = PRIO_PROCESS, PRIO_PGRP, PRIO_USER(同一user下的所有进程)
int getpriority (int class, int id)
//设置nice值
int setpriority (int class, int id, int niceval)
//提高nice
int nice (int increment)
//相当于:
int
nice (int increment)
{
    int result, old = getpriority (PRIO_PROCESS, 0);
    result = setpriority (PRIO_PROCESS, 0, old + increment);
    if (result != -1)
        return old + increment;
    else
        return -1;
}
```

## 绑定cpus
```c
#include <sched.h>
```
结构体cpu_set_t, 表示cpuset
```c
void CPU_ZERO (cpu set t *set)
void CPU_SET (int cpu, cpu set t *set)
void CPU_CLR (int cpu, cpu set t *set)
int CPU_ISSET (int cpu, const cpu set t *set)

//获取进程cpu mask
int sched_getaffinity (pid t pid, size t cpusetsize, cpu set t *cpuset)
//设置
int sched_setaffinity (pid t pid, size t cpusetsize, const cpu set t *cpuset)
```

## 可用内存页数目
```c
#include <unistd.h>
//获取当前进程page size
int getpagesize (void)
//获取系统物理页数, 但这些页不一定全能用
sysconf (_SC_PHYS_PAGES)
//获取当前进程当时可用的物理页, 不会影响其他进程
sysconf (_SC_AVPHYS_PAGES)
//也可用GNU的函数
#include <sys/sysinfo.h>
long int get_phys_pages (void)
long int get_avphys_pages (void)
```

## cpu个数
```c
//全部的cpu个数
sysconf (_SC_NPROCESSORS_CONF)
//online的cpu个数, 即可用的cpu个数
sysconf (_SC_NPROCESSORS_ONLN)
//也可用GNU的函数
#include <sys/sysinfo.h>
int get_nprocs_conf (void)
int get_nprocs (void)
```

## 当前系统负载, 1 5 15分钟平均负载
```c
#include <stdlib.h>
int getloadavg (double loadavg[], int nelem)
```
