除了普通文件fd, socket fd, Linux还提供了比较特殊的几种fd, 比如eventfd timerfd signalfd等等.

各种fd是系统调用
用perf看所有带fd的系统调用
`perf list | grep syscalls | grep fd | grep enter`
```sh
root@godev-server:/home/yingjieb# perf list | grep syscalls | grep fd | grep enter
  syscalls:sys_enter_eventfd                         [Tracepoint event]
  syscalls:sys_enter_eventfd2                        [Tracepoint event]
  syscalls:sys_enter_fdatasync                       [Tracepoint event]
  syscalls:sys_enter_gettimeofday                    [Tracepoint event]
  syscalls:sys_enter_memfd_create                    [Tracepoint event]
  syscalls:sys_enter_settimeofday                    [Tracepoint event]
  syscalls:sys_enter_signalfd                        [Tracepoint event]
  syscalls:sys_enter_signalfd4                       [Tracepoint event]
  syscalls:sys_enter_timerfd_create                  [Tracepoint event]
  syscalls:sys_enter_timerfd_gettime                 [Tracepoint event]
  syscalls:sys_enter_timerfd_settime                 [Tracepoint event]
  syscalls:sys_enter_userfaultfd                     [Tracepoint event]
```

下面就介绍一下这些fd的使用.

# signalfd

```c
#include <sys/signalfd.h>
#include <signal.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
 
#define handle_error(msg) \
    do { perror(msg); exit(EXIT_FAILURE); } while (0)
 
int
main(int argc, char *argv[])
{
    sigset_t mask;
    int sfd;
    struct signalfd_siginfo fdsi;
    ssize_t s;
 
    sigemptyset(&mask);
    sigaddset(&mask, SIGINT);
    sigaddset(&mask, SIGQUIT);
 
    /* 阻塞信号以使得它们不被默认的处理试方式处理 */
 
    if (sigprocmask(SIG_BLOCK, &mask, NULL) == -1)
        handle_error("sigprocmask");
 
    sfd = signalfd(-1, &mask, 0);
    if (sfd == -1)
        handle_error("signalfd");
 
    for (;;) {
        s = read(sfd, &fdsi, sizeof(struct signalfd_siginfo));
        if (s != sizeof(struct signalfd_siginfo))
            handle_error("read");
 
        if (fdsi.ssi_signo == SIGINT) {
            printf("Got SIGINT\n");
        } else if (fdsi.ssi_signo == SIGQUIT) {
            printf("Got SIGQUIT\n");
            exit(EXIT_SUCCESS);
        } else {
            printf("Read unexpected signal\n");
        }
    }
}
```

> 这个例子只是很简单的说明了使用signalfd的方法，并没有真正发挥它的作用，有了这个API，就可以将信号处理作为IO看待.  
每一个信号集合（或者某一个对应的信号）就会有对应的文件描述符，这样将信号处理的流程大大简化，将应用程序中的业务作为文件来操作，也体现了linux下的一切皆文件的说法，非常好，假如有很多种信号等待着处理，每一个信号描述符对待一种信号的处理，那么就可以将信号文件描述符设置为非阻塞，同时结合epoll使用，对信号的处理转化为IO复用，和这个有相似之处的API还有timerfd

# timerfd
使用timerfd的一个例子是libevent. 在linux下面, 默认的libevent使用epoll, epoll有个超时时间.  
libevent利用这个超时时间来做定时器的触发源, 即在event loop里面, 把定时器堆的时间做为超时时间.  
更具体一些, 在libevent初始化时:  
默认情况下, libevent不使用timerfd, `epollop->timerfd = -1`.  
但有`EVENT_BASE_FLAG_PRECISE_TIMER`情况下, 
```c
if ((base->flags & EVENT_BASE_FLAG_PRECISE_TIMER) &&
     base->monotonic_timer.monotonic_clock == CLOCK_MONOTONIC) {
         int fd;
        fd = epollop->timerfd = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK|TFD_CLOEXEC);
        if (epollop->timerfd >= 0) {
            struct epoll_event epev;
            memset(&epev, 0, sizeof(epev));
            epev.data.fd = epollop->timerfd;
            epev.events = EPOLLIN;
            //把这个fd加入epoll
            epoll_ctl(epollop->epfd, EPOLL_CTL_ADD, fd, &epev)
```

参考: [https://blog.csdn.net/KangRoger/article/details/47844443](https://blog.csdn.net/KangRoger/article/details/47844443)

所以这里的`timerfd_create()`就是timerfd的使用方法.

## timerfd API
```c
       #include <sys/timerfd.h>

       int timerfd_create(int clockid, int flags);

        //这个时间精度应该比ms高.
       int timerfd_settime(int fd, int flags,
                           const struct itimerspec *new_value,
                           struct itimerspec *old_value);

       int timerfd_gettime(int fd, struct itimerspec *curr_value);
       
       read系统调用会返回超时次数, 如果一次超时都没到, read阻塞.
```
1. timerfd_create用于创建一个定时器文件，函数返回值是一个文件句柄fd。
2. timerfd_settime用于设置新的超时时间，并开始计时。flag为0表示相对时间，为1表示绝对时间。new_value为这次设置的新时间，old_value为上次设置的时间。返回0表示设置成功。
3. timerfd_gettime用于获得定时器距离下次超时还剩下的时间。如果调用时定时器已经到期，并且该定时器处于循环模式（设置超时时间时struct itimerspec::it_interval不为0），那么调用此函数之后定时器重新开始计时。


## 注epoll使用简介
`man epoll`可以看到, epoll的API有
```c
epoll_create() : 用于创建epoll对象
epoll_ctl() : 用于管理fd set
epoll_wait() : 用于等待IO
```

### 边沿触发和电平触发
epoll有类似硬件的触发概念
* edge-triggered (ET): 边沿触发, 只有fd的状态有变化才触发. 例如epoll_wait返回一个fd可读, 它实际有2k的数据可以读, 但回调函数里只读了1k. 即使还有1k数据可读, 在ET触发模式下, epoll不会再触发fd可读.
用ET触发的推荐场景是:
    * 这个fd是非阻塞的
    * 并且read或者write返回EAGAIN
* level-triggered (LT): 电平触发, 这是默认触发方式

### epoll_wait
```c
#include <sys/epoll.h>
int epoll_wait(int epfd, struct epoll_event *events,
                int maxevents, int timeout);
```
这里的timeout单位是ms, 是用CLOCK_MONOTONIC度量的.
epoll_wait返回有几种可能:
* 底层driver有数据, event到达
* 超时, -1表示永久等待. 0表示立即返回
* 被signal

# eventfd概念
`eventfd()`是个系统调用, 生成一个event fd对象, 内核为其维护一个计数器, 用来做用户进程间的wait/nofify机制, 也可以被kernel用来通知用户态进程.
可以用来代替`pipe()`, 相比于pipe来说，少用了一个文件描述符，而且不必管理缓冲区，单纯的事件通知的话，方便很多  
在`fork()`的时候, 子进程继承eventfd, 对应同一个eventfd对象, 并且, 这个fd在`execve()`后仍然保持, 但如果有close-on-exec选项则不保持.

创建eventfd以后, 可以子进程写, 父进程读;
* 读的时候, 只要计数器非0, 就返回计数器值, 并reset到0; 计数器为0则block
* 计数器不溢出就可以写, 写的值加到计数器上. 溢出的话会阻塞.
* 也可以配合poll(), select()使用

## 用法
```c
#include <sys/eventfd.h>
int eventfd(unsigned int initval, int flags);
    read()
    write()
    close()

//glibc还提供:
typedef uint64_t eventfd_t;
int eventfd_read(int fd, eventfd_t *value);
int eventfd_write(int fd, eventfd_t value);
```

# 进程间共享文件描述符
在 `OVS架构和代码`中, qemu通过unix socket传递eventfd的文件描述符给OVS, 实际上是用了socket的`SCM_RIGHTS`方法.
实际上, 也可以用pipe之类的进程间通信为载体, 其底层是通过ioctl的`I_SENDFD`和`I_RECVFD`完成的.
详见
http://poincare.matf.bg.ac.rs/~ivana/courses/ps/sistemi_knjige/pomocno/apue/APUE/0201433079/ch17lev1sec4.html
出自`Advanced Programming in the UNIX® Environment: Second Edition 2005`  
看来这技术有十几年的时间了  
发送进程:
```c
#include "apue.h"
#include <stropts.h>

/*
 * Pass a file descriptor to another process.
 * If fd<0, then -fd is sent back instead as the error status.
 */
int
send_fd(int fd, int fd_to_send)
{
    char buf[2]; /* send_fd()/recv_fd() 2-byte protocol */
    
    buf[0] = 0; /* null byte flag to recv_fd() */
    if (fd_to_send < 0) {
        buf[1] = -fd_to_send; /* nonzero status means error */
        if (buf[1] == 0)
            buf[1] = 1; /* -256, etc. would screw up protocol */
    } else {
        buf[1] = 0; /* zero status means OK */
    }

    if (write(fd, buf, 2) != 2)
        return(-1);
    if (fd_to_send >= 0)
        if (ioctl(fd, I_SENDFD, fd_to_send) < 0)
            return(-1);
    return(0);
}
```
接收进程:
```c
//接收时, 第三个参数是strrecvfd 结构体
   struct strrecvfd {
       int fd; /* new descriptor */
       uid_t uid; /* effective user ID of sender */
       gid_t gid; /* effective group ID of sender */
       char fill[8];
   };

#include "apue.h"
#include <stropts.h>

/*
 * Receive a file descriptor from another process (a server).
 * In addition, any data received from the server is passed
 * to (*userfunc)(STDERR_FILENO, buf, nbytes). We have a
 * 2-byte protocol for receiving the fd from send_fd().
 */
int
recv_fd(int fd, ssize_t (*userfunc)(int, const void *, size_t))
{
    int newfd, nread, flag, status;
    char *ptr;
    char buf[MAXLINE];
    struct strbuf dat;
    struct strrecvfd recvfd;

    status = -1;
    for ( ; ; ) {
        dat.buf = buf;
        dat.maxlen = MAXLINE;
        flag = 0;
        if (getmsg(fd, NULL, &dat, &flag) < 0)
            err_sys("getmsg error");
        nread = dat.len;
        if (nread == 0) {
            err_ret("connection closed by server");
            return(-1);
        }
        /*
         * See if this is the final data with null & status.
         * Null must be next to last byte of buffer, status
         * byte is last byte. Zero status means there must
         * be a file descriptor to receive.
         */
        for (ptr = buf; ptr < &buf[nread]; ) {
            if (*ptr++ == 0) {
                if (ptr != &buf[nread-1])
                    err_dump("message format error");
                 status = *ptr & 0xFF; /* prevent sign extension */
                 if (status == 0) {
                     if (ioctl(fd, I_RECVFD, &recvfd) < 0)
                         return(-1);
                     newfd = recvfd.fd; /* new descriptor */
                 } else {
                     newfd = -status;
                 }
                 nread -= 2;
            }
        }
        if (nread > 0)
            if ((*userfunc)(STDERR_FILENO, buf, nread) != nread)
                 return(-1);

        if (status >= 0) /* final data has arrived */
            return(newfd); /* descriptor, or -status */
    }
}

```

下面的例子说明, 两个进程想要传递fd, 要先有个通道, 比如pipe, 或者socket, 用来传递fd.
```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stropts.h>
#include <stdio.h>

#define TESTFILE "/dev/null"
main(int argc, char *argv[])
{
    int fd;
    int pipefd[2];
    struct stat statbuf;

    stat(TESTFILE, &statbuf);
    statout(TESTFILE, &statbuf);
    pipe(pipefd);
    if (fork() == 0) {
        close(pipefd[0]);
        sendfd(pipefd[1]);
    } else {
        close(pipefd[1])
        recvfd(pipefd[0]);
    }
}

sendfd(int p)
{
    int tfd;

    tfd = open(TESTFILE, O_RDWR);
    ioctl(p, I_SENDFD, tfd);
}

recvfd(int p)
{
    struct strrecvfd rfdbuf;
    struct stat statbuf;
    char    fdbuf[32];

    ioctl(p, I_RECVFD, &rfdbuf);
    fstat(rfdbuf.fd, &statbuf);
    sprintf(fdbuf, "recvfd=%d", rfdbuf.fd);
    statout(fdbuf, &statbuf); 
}

statout(char *f, struct stat *s)
{
    printf("stat: from=%s mode=0%o, ino=%ld, dev=%lx, rdev=%lx\n",
    f, s->st_mode, s->st_ino, s->st_dev, s->st_rdev);
    fflush(stdout);
}
```
