- [非本地跳转](#非本地跳转)
  - [默认不改变signal](#默认不改变signal)
  - [完全的上下文控制](#完全的上下文控制)
- [signal](#signal)
  - [程序错误类signal, 默认的处理是终止进程](#程序错误类signal-默认的处理是终止进程)
  - [结束类signal](#结束类signal)
  - [定时类signal](#定时类signal)
  - [异步signal, 默认忽略](#异步signal-默认忽略)
  - [任务控制类signal](#任务控制类signal)
  - [操作error signal, 默认是终止进程](#操作error-signal-默认是终止进程)
  - [其他signal](#其他signal)
  - [打印signal类型](#打印signal类型)
  - [signal处理](#signal处理)
  - [sigaction](#sigaction)
  - [signal的flag](#signal的flag)
  - [signal的初始状态](#signal的初始状态)
  - [signal处理函数](#signal处理函数)
  - [signal handler被打断?](#signal-handler被打断)
  - [signal合并](#signal合并)
  - [signal的处理原则](#signal的处理原则)
  - [库函数被打断?](#库函数被打断)
  - [触发signal](#触发signal)
  - [向其他进程发signal](#向其他进程发signal)
  - [kill用于进程间同步](#kill用于进程间同步)
  - [signal的阻塞](#signal的阻塞)
  - [正在pending的signal(已到达, 未处理)](#正在pending的signal已到达-未处理)
  - [等待signal](#等待signal)
  - [signal的栈](#signal的栈)
- [基础系统接口](#基础系统接口)
  - [参数解析之getopt_long](#参数解析之getopt_long)
  - [参数解析之argp_parse](#参数解析之argp_parse)
  - [子选项支持](#子选项支持)
  - [环境变量](#环境变量)
  - [syscall](#syscall)
  - [终结程序](#终结程序)
- [进程](#进程)
  - [fork](#fork)
  - [exec](#exec)
  - [等待子进程终止](#等待子进程终止)
  - [子进程退出状态](#子进程退出状态)
- [job控制](#job控制)
  - [获得控制终端名字(和ttyname有什么区别?)](#获得控制终端名字和ttyname有什么区别)
  - [进程组api](#进程组api)
  - [一个简单的shell](#一个简单的shell)
- [用户和组](#用户和组)
  - [api](#api)
- [系统管理](#系统管理)
- [调试](#调试)
  - [调用栈](#调用栈)
- [pthread](#pthread)
  - [gnu扩展](#gnu扩展)
- [libc的系统桩函数(略)](#libc的系统桩函数略)
- [libc基础函数](#libc基础函数)

# 非本地跳转
```c
#include <setjmp.h>
//结构体, 用来保存上下文
jmp_buf
/*返回0表示设置jmp_buf成功
**非零值表示有一次非本地跳转, 值是longjmp给出的
**实际上, segjump是个宏
*/
int setjmp (jmp buf state)
void longjmp (jmp buf state, int value)
```
举例:
```c
#include <setjmp.h>
#include <stdlib.h>
#include <stdio.h>
jmp_buf main_loop;
void
abort_to_main_loop(int status)
{
    longjmp(main_loop, status);
}
int
main(void)
{
    while (1)
        if (setjmp(main_loop))
            puts("Back at main loop....");
        else
            do_command();
}
void
do_command(void)
{
    char buffer[128];
 
    if (fgets(buffer, 128, stdin) == NULL)
        abort_to_main_loop(-1);
    else
        exit(EXIT_SUCCESS);
}
```
## 默认不改变signal
但也可以保存当时的signal, 非本地跳转时恢复
```c
#include <setjmp.h>
//结构体
sigjmp_buf
//savesigs是要保存的当时的signal set
int sigsetjmp (sigjmp buf state, int savesigs)
//signal在longjump时会恢复
void siglongjmp (sigjmp buf state, int value)
```

## 完全的上下文控制
```c
#include <ucontext.h>
//比jmp_buf信息更多, 开销也更大
ucontext_t
    ucontext_t *uc_link
    sigset_t uc_sigmask
    stack_t uc_stack
    mcontext_t uc_mcontext
//初始化上下文ucp
int getcontext (ucontext t *ucp)
//修改上下文
void makecontext (ucontext t *ucp, void (*func) (void), int argc, . . . )
//恢复上下文
int setcontext (const ucontext t *ucp)
//保存现在的上下文到oucp, 并切换到ucp
int swapcontext (ucontext t *restrict oucp, const ucontext t *restrict ucp)
```
举例:场景是运行一个很复杂的搜索函数fn, 这个函数复杂到不能很简单的返回
```c
int
random_search(int n, int (*fp)(int, ucontext_t *))
{
    volatile int cnt = 0;
    ucontext_t uc;
 
    /* Safe current context. */
    if (getcontext(&uc) < 0)
        return -1;
 
    /* If we have not tried n times try again. */
    if (cnt++ < n)
 
        /* Call the function with a new random number
        and the context. */
        if (fp(rand(), &uc) != 0)
            /* We found what we were looking for. */
            return 1;
 
    /* Not found. */
    return 0;
}
```
举例:
```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <ucontext.h>
#include <sys/time.h>
/* Set by the signal handler. */
static volatile int expired;
/* The contexts. */
static ucontext_t uc[3];
/* We do only a certain number of switches. */
static int switches;
/* This is the function doing the work. It is just a
skeleton, real code has to be filled in. */
static void
f(int n)
{
    int m = 0;
 
    while (1) {
        /* This is where the work would be done. */
        if (++m % 100 == 0) {
            putchar(’.’);
            fflush(stdout);
        }
 
        /* Regularly the expire variable must be checked. */
        if (expired) {
            /* We do not want the program to run forever. */
            if (++switches == 20)
                return;
 
            printf("\nswitching from %d to %d\n", n, 3 - n);
            expired = 0;
            /* Switch to the other context, saving the current one. */
            swapcontext(&uc[n], &uc[3 - n]);
        }
    }
}
/* This is the signal handler which simply set the variable. */
void
handler(int signal)
{
    expired = 1;
}
int
main(void)
{
    struct sigaction sa;
    struct itimerval it;
    char st1[8192];
    char st2[8192];
    /* Initialize the data structures for the interval timer. */
    sa.sa_flags = SA_RESTART;
    sigfillset(&sa.sa_mask);
    sa.sa_handler = handler;
    it.it_interval.tv_sec = 0;
    it.it_interval.tv_usec = 1;
    it.it_value = it.it_interval;
 
    /* Install the timer and get the context we can manipulate. */
    if (sigaction(SIGPROF, &sa, NULL) < 0
        || setitimer(ITIMER_PROF, &it, NULL) < 0
        || getcontext(&uc[1]) == -1
        || getcontext(&uc[2]) == -1)
        abort();
 
    /* Create a context with a separate stack which causes the
    function f to be call with the parameter 1.
    Note that the uc_link points to the main context
    which will cause the program to terminate once the function
    return. */
    uc[1].uc_link = &uc[0];
    uc[1].uc_stack.ss_sp = st1;
    uc[1].uc_stack.ss_size = sizeof st1;
    makecontext(&uc[1], (void (*)(void)) f, 1, 1);
    /* Similarly, but 2 is passed as the parameter to f. */
    uc[2].uc_link = &uc[0];
    uc[2].uc_stack.ss_sp = st2;
    uc[2].uc_stack.ss_size = sizeof st2;
    makecontext(&uc[2], (void (*)(void)) f, 1, 2);
    /* Start running. */
    swapcontext(&uc[0], &uc[1]);
    putchar(’\n’);
    return 0;
}
```

# signal
*  由程序自身引起的error触发的signal一般都是同步的
*  外部触发的signal一般都是异步的.
*  如果一个signal正在被处理, 那处理期间这个信号是阻塞的.
*  如果两个不同的signal间隔很短, one handler can run within another
## 程序错误类signal, 默认的处理是终止进程
*  SIGFPE: 运算错误, 包括除零和浮点异常等所有的运算错误
*  SIGILL: 非法指令, 可能是特权指令或根本不是条CPU指令
*  SIGSEGV: 著名的段错误. 访问非法数据
*  SIGBUS: bus error. 可能是访问了不存在的地址, 或地址不对齐
*  SIGABRT: 程序自己调用了abort()
*  SIGTRAP: 断点, 用于debugger
*  SIGSYS: 非法系统调用, 可能不存在这个调用号
*  SIGEMT: 虚拟指令错误?
## 结束类signal
*  SIGTERM: 礼貌的让一个进程终止. 和SIGKILL不同, SIGTERM可以被block, handle和ignore.
注: shell下的kill命令默认是发SIGTERM信号
*  SIGINT: ctrl+C, 用户中断程序的执行
*  SIGQUIT: ctrl+\, 会生成coredump. 推荐收到这个信号的时候不清理现场, 以便定位.
*  SIGKILL: 马上终止进程. 不能被处理或忽略, 也不会阻塞
*  SIGHUP: 一般表示用户中断了一个会话
## 定时类signal
*  SIGALRM: 定时器时间到. 比如alarm
*  SIGVTALRM: 虚拟定时器
*  SIGPROF: 性能分析
## 异步signal, 默认忽略
*  SIGIO: 异步io, 需要先用fcntl()来使能异步模式
*  SIGURG: socket的urgent事件
*  SIGPOLL: 类似SIGIO
## 任务控制类signal
*  SIGCHLD SIGCLD: 子进程终止或停止时发送给父进程, 默认忽略
*  SIGSTOP: 停止一个进程. 注意不是终止. 不能阻塞, 处理和忽略
*  SIGTSTP: crtl+Z, 与SIGSTOP不同的是, 它可以处理或忽略. 
*  SIGTTIN: 处于后台运行的程序试图从terminal获取输入, 默认停止进程.
## 操作error signal, 默认是终止进程
*  SIGPIPE: 管道error. 比如在读open之前写pipe, 或在connect之前写socket
*  SIGLOST: 服务进程意外终止
*  SIGXCPU: 进程超出资源限制. 见系统资源
*  SIGXFSZ: 文件大小超出限制
## 其他signal
*  SIGUSR1 SIGUSR2: 常用于简单进程间通信, 默认终止进程
*  SIGWINCH: 窗口大小改变. 默认ignore
*  SIGINFO: ??
## 打印signal类型
```c
#include <string.h>
char * strsignal (int signum)
#include <signal.h>
void psignal (int signum, const char *message)
```

## signal处理
```c
#include <signal.h>
//sighandler_t
void handler (int signum) { ... }
/*注册handler, action可以是个handler函数, 也可以是SIG_DFL, SIG_IGN
**注意不同的系统有两种模式: BSD模式, SVID模式
**SVID系统下面, handler只运行一次, 然后handler会被deinstall
**BSD系统下面, handler一直有效
**GNU的libc是BSD模式
*/
sighandler_t signal (int signum, sighandler t action)
```
举例: 如果默认是ignore, 则不改变它的行为
```c
#include <signal.h>
void
termination_handler(int signum)
{
    struct temp_file *p;
 
    for (p = temp_file_list; p; p = p->next)
        unlink(p->name);
}
int
main(void)
{
    ...
 
    if (signal(SIGINT, termination_handler) == SIG_IGN)
        signal(SIGINT, SIG_IGN);
 
    if (signal(SIGHUP, termination_handler) == SIG_IGN)
        signal(SIGHUP, SIG_IGN);
 
    if (signal(SIGTERM, termination_handler) == SIG_IGN)
        signal(SIGTERM, SIG_IGN);
 
    ...
}
```

## sigaction
```c
struct sigaction
    sighandler_t sa_handler
    sigset_t sa_mask //当sa_handler执行时, 需要阻塞的signal set
    int sa_flags
//推荐用sigaction, action为NULL表示查询
int sigaction (int signum, const struct sigaction *restrict action, struct sigaction *restrict old-action)
```
举例:
```c
#include <signal.h>
void
termination_handler(int signum)
{
    struct temp_file *p;
 
    for (p = temp_file_list; p; p = p->next)
        unlink(p->name);
}
int
main(void)
{
    ...
    struct sigaction new_action, old_action;
    /* Set up the structure to specify the new action. */
    new_action.sa_handler = termination_handler;
    sigemptyset(&new_action.sa_mask);
    new_action.sa_flags = 0;
    sigaction(SIGINT, NULL, &old_action);
 
    if (old_action.sa_handler != SIG_IGN)
        sigaction(SIGINT, &new_action, NULL);
 
    sigaction(SIGHUP, NULL, &old_action);
 
    if (old_action.sa_handler != SIG_IGN)
        sigaction(SIGHUP, &new_action, NULL);
 
    sigaction(SIGTERM, NULL, &old_action);
 
    if (old_action.sa_handler != SIG_IGN)
        sigaction(SIGTERM, &new_action, NULL);
 
    ...
}
```
## signal的flag
*  SA_NOCLDSTOP: 只对SIGCHLD有效. 只报告子进程终止
*  SA_ONSTACK: 不使用signal栈.
*  SA_RESTART: 被signal打断的库函数会resume, 否则直接返回失败, 此时error是EINTR.
## signal的初始状态
一般都是从父进程继承的, 但如果子进程用exec*来执行其他程序, 则signal会被初始化为默认状态.
举例: 在signal不是被ignore的时候才用sigaction.
是否因为一旦signal被ignore过, 那么即使重新调用sigaction也不会生效?--经过验证, 不是
```c
struct sigaction temp;
sigaction (SIGHUP, NULL, &temp);
if (temp.sa_handler != SIG_IGN)
{
temp.sa_handler = handle_sighup;
sigemptyset (&temp.sa_mask);
sigaction (SIGHUP, &temp, NULL);
}
```

## signal处理函数
*  handler可以修改全局变量, 这个变量应该是sig_atomic_t 类型
*  handler可以return
*  handler可以终止程序
*  handler可以非本地跳转
举例: return的情况
```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
/* This flag controls termination of the main loop. */
volatile sig_atomic_t keep_going = 1;
/* The signal handler just clears the flag and re-enables itself. */
void
catch_alarm(int sig)
{
    keep_going = 0;
    signal(sig, catch_alarm);
}
void
do_stuff(void)
{
    puts("Doing stuff while waiting for alarm....");
}
int
main(void)
{
    /* Establish a handler for SIGALRM signals. */
    signal(SIGALRM, catch_alarm);
    /* Set an alarm to go off in a little while. */
    alarm(2);
 
    /* Check the flag once in a while to see when to quit. */
    while (keep_going)
        do_stuff();
 
    return EXIT_SUCCESS;
}
```
举例: 终止的情况
```c
volatile sig_atomic_t fatal_error_in_progress = 0;
void
fatal_error_signal(int sig)
{
    /* Since this handler is established for more than one kind of signal,
    it might still get invoked recursively by delivery of some other kind
    of signal. Use a static variable to keep track of that. */
    if (fatal_error_in_progress)
        raise(sig);
 
    fatal_error_in_progress = 1;
    /* Now do the clean up actions:
    - reset terminal modes
    - kill child processes
    - remove lock files */
    ...
    /* Now reraise the signal. We reactivate the signal’s
    default handling, which is to terminate the process.
    We could just call exit or abort,
    but reraising the signal sets the return status
    from the process correctly. */
    signal(sig, SIG_DFL);
    raise(sig);
}
```
举例: 非本地跳转的情况. 
注意由于一些重要的结构体或变量可能还没有修改完毕就被signal打断, 此时如果进行非本地跳转的话, 那些重要结构可能不完整, 这种情况有两种办法可以避免:
*  主程序修改重要变量时阻塞signal
*  在signal handler里面重新初始化这些重要变量
```c
#include <signal.h>
#include <setjmp.h>
jmp_buf return_to_top_level;
volatile sig_atomic_t waiting_for_input;
void
handle_sigint(int signum)
{
    /* We may have been waiting for input when the signal arrived,
    but we are no longer waiting once we transfer control. */
    waiting_for_input = 0;
    longjmp(return_to_top_level, 1);
}
int
main(void)
{
    ...
    signal(SIGINT, sigint_handler);
    ...
 
    while (1) {
        prepare_for_command();
 
        if (setjmp(return_to_top_level) == 0)
            read_and_execute_command();
    }
}
/* Imagine this is a subroutine used by various commands. */
char *
read_data()
{
    if (input_from_terminal) {
        waiting_for_input = 1;
        ...
        waiting_for_input = 0;
    } else {
        ...
    }
}
```

## signal handler被打断?
正在运行的handler会自动block它的signum, 但是其他种类的signal依然可以打断当前的handler.
这也是为什么sigaction有个sa_mask的原因, 被mask的signal会被阻塞.
sigprocmask函数可以临时改变这个mask, 但只在handler里面生效. 因为这个handler return以后, mask会恢复.

## signal合并
同类signal在没处理之前会被合并为一个, 所以对signal的计数是不准的. 尤其注意对SIGCHLD是如此.
举例: 
```c
struct process {
    struct process *next;
    /* The process ID of this child. */
    int pid;
    /* The descriptor of the pipe or pseudo terminal
    on which output comes from this child. */
    int input_descriptor;
    /* Nonzero if this process has stopped or terminated. */
    sig_atomic_t have_status;
    /* The status of this child; 0 if running,
    otherwise a status value from waitpid. */
    int status;
};
struct process *process_list;
//This example also uses a flag to indicate whether signals have arrived since some time
//in the past—whenever the program last cleared it to zero.
/* Nonzero means some child’s status has changed
so look at process_list for the details. */
int process_status_change;
Here is the handler itself:
void
sigchld_handler(int signo)
{
    int old_errno = errno;
 
    while (1) {
        register int pid;
        int w;
        struct process *p;
 
        /* Keep asking for a status until we get a definitive result. */
        do {
            errno = 0;
            pid = waitpid(WAIT_ANY, &w, WNOHANG | WUNTRACED);
        } while (pid <= 0 && errno == EINTR);
 
        if (pid <= 0) {
            /* A real failure means there are no more
            stopped or terminated child processes, so return. */
            errno = old_errno;
            return;
        }
 
        /* Find the process that signaled us, and record its status. */
        for (p = process_list; p; p = p->next)
            if (p->pid == pid) {
                p->status = w;
                /* Indicate that the status field
                has data to look at. We do this only after storing it. */
                p->have_status = 1;
 
                /* If process has terminated, stop waiting for its output. */
                if (WIFSIGNALED(w) || WIFEXITED(w))
                    if (p->input_descriptor)
                        FD_CLR(p->input_descriptor, &input_wait_mask);
 
                /* The program should check this flag from time to time
                to see if there is any news in process_list. */
                ++process_status_change;
            }
 
        /* Loop around to handle all the processes
        that have something to tell us. */
    }
}
//Here is the proper way to check the flag process_status_change:
if (process_status_change)
{
    struct process *p;
    process_status_change = 0;
 
    for (p = process_list; p; p = p->next)
        if (p->have_status) {
            ... Examine p->status ...
        }
}
```
举例: 主程序怎么知道handler调用过呢? 这个例子是有个变量, 只在handler里面改. 主程序查询这个变量
```c
sig_atomic_t process_status_change;
sig_atomic_t last_process_status_change;
... {
    sig_atomic_t prev = last_process_status_change;
    last_process_status_change = process_status_change;
 
    if (last_process_status_change != prev)
    {
        struct process *p;
 
        for (p = process_list; p; p = p->next)
            if (p->have_status) {
                ... Examine p->status ...
            }
    }
}
```

## signal的处理原则
时刻记住handler的处理是异步的, 可能在任何时间点打断主程序, 包括关键数据正在改写过程中, 或不可重入的函数正在执行中等等, 甚至一个整形变量正在被赋值.
最好的情况下, handler只是对一个变量赋值, 真正的事情, 需要等到主程序检测到这个变量变化以后再处理.
需要注意的是, 一个不可重入的函数, 也不是一定不能在handler里面使用, 要使用它需要保证一个前提:
保证主程序用这个函数的时候不会被handler打断, 怎么保证呢?
*  主程序根本不会调这个函数, 或类似的函数
*  主程序调这个函数时显式阻塞影响它的signal
下面是handler的一些基本原则:
*  handler需要访问的全局变量声明成volatile, 和sig_atomic_t(其实就是int, 因为int的读写是原子的)?
*  比如gethostbyname这个函数, 会返回一个静态变量, 这静态变量可能下次调用gethostbyname就变了.
此时假设主程序正在调用gethostbyname还没返回, 被signal打断了, handler里面也调了gethostbyname, 那么主程序的返回值就不对了.
*  比如printf, 其内部使用同一个数据结构, 在主程序和handler里面同时调用会互相干扰.
*  malloc和free也不可重入, 在handler使用的内存可以预先在主程序申请好.
*  errno也不可重入, 但是可以在handler开始先保存errno, 最后恢复它.
其实这招也很通用, 如果主程序和handler要修改同一个变量, 在handler里面先保存再恢复.
错误举例: 这个例子里面, 主程序不断的给memory赋值, handler里面可能会打出0,1或1,0.
```c
#include <signal.h>
#include <stdio.h>
volatile struct two_words {
    int a, b;
} memory;
void
handler(int signum)
{
//printf is safe here because main not use it
    printf("%d,%d\n", memory.a, memory.b);
    alarm(1);
}
int
main(void)
{
    static struct two_words zeros = { 0, 0 }, ones = { 1, 1 };
    signal(SIGALRM, handler);
    memory = zeros;
    alarm(1);
 
    while (1) {
        memory = zeros;
        memory = ones;
    }
}
```

## 库函数被打断?
*  库函数返回EINTR表示被打断, 可以用sigaction的SA_RESTART来设置自动重试.
也可以TEMP_FAILURE_RETRY (expression)重试
*  read和write不受SA_RESTART控制, 因为可能有部分的数据已经传送完毕了. 
这种情况通过返回值来表示已经传送的字节数, 表示"部分成功"
## 触发signal
```c
//向自己发signal
int raise (int signum)
```
典型的应用是在handler里面再次触发signal:
```c
#include <signal.h>
/* When a stop signal arrives, set the action back to the default
and then resend the signal after doing cleanup actions. */
void
tstp_handler(int sig)
{
    signal(SIGTSTP, SIG_DFL);
    /* Do cleanup actions here. */
    ...
    raise(SIGTSTP);
}
/* When the process is continued again, restore the signal handler. */
void
cont_handler(int sig)
{
    signal(SIGCONT, cont_handler);
    signal(SIGTSTP, tstp_handler);
}
/* Enable both handlers during program initialization. */
int
main(void)
{
    signal(SIGCONT, cont_handler);
    signal(SIGTSTP, tstp_handler);
    ...
}
```

## 向其他进程发signal
```c
/* pid > 0         正常的pid
** pid == 0       同一个进程组的所有进程
** pid < -1        -pid的进程组
** pid == -1      向所有进程发(root权限), 向同一个用户的所有进程发
** 使用kill有权限要求, root或是同一个用户的进程
*/
int kill (pid t pid, int signum)
```

## kill用于进程间同步
```c
#include <signal.h>
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
/* When a SIGUSR1 signal arrives, set this variable. */
volatile sig_atomic_t usr_interrupt = 0;
void
synch_signal(int sig)
{
    usr_interrupt = 1;
}
/* The child process executes this function. */
void
child_function(void)
{
    /* Perform initialization. */
    printf("I'm herehistory! My pid is %d.\n", (int) getpid());
    /* Let parent know you’re done. */
    kill(getppid(), SIGUSR1);
    /* Continue with execution. */
    puts("Bye, now....");
    exit(0);
}
int
main(void)
{
    struct sigaction usr_action;
    sigset_t block_mask;
    pid_t child_id;
    /* Establish the signal handler. */
    sigfillset(&block_mask);
    usr_action.sa_handler = synch_signal;
    usr_action.sa_mask = block_mask;
    usr_action.sa_flags = 0;
    sigaction(SIGUSR1, &usr_action, NULL);
    /* Create the child process. */
    child_id = fork();
 
    if (child_id == 0)
        child_function();  /* Does not return. */
 
    /* Busy wait for the child to send a signal. */
    while (!usr_interrupt)
        ;
 
    /* Now continue execution. */
    puts("That’s all, folks!");
    return 0;
}
```

## signal的阻塞
*  用sigprocmask可以显式的阻塞signal, 用于保护敏感变量
*  sigaction的sa_mask可以设置在运行handler时mask掉相应的signal, 但hander完成后会恢复原来的mask
这里需要用到sigset_t
```c
//初始化为全0的set
int sigemptyset (sigset t *set)
//初始化为全1的set
int sigfillset (sigset t *set)
//添加
int sigaddset (sigset t *set, int signum)
//删除
int sigdelset (sigset t *set, int signum)
//判断
int sigismember (const sigset t *set, int signum)
//在多线程模型里, 每个线程都有自己的signal mask, 而sigprocmask的行为是不确定的. 此时要用pthread_sigmask
int sigprocmask (int how, const sigset t *restrict set, sigset t *restrict oldset)
举例: 
```c
/* This variable is set by the SIGALRM signal handler. */
volatile sig_atomic_t flag = 0;
int
main(void)
{
    sigset_t block_alarm;
    ...
    /* Initialize the signal mask. */
    sigemptyset(&block_alarm);
    sigaddset(&block_alarm, SIGALRM);
 
    while (1) {
        /* Check if a signal has arrived; if so, reset the flag. */
        sigprocmask(SIG_BLOCK, &block_alarm, NULL);
 
        if (flag) {
 
            actions - if - not - arrived
            flag = 0;
    }
 
    sigprocmask(SIG_UNBLOCK, &block_alarm, NULL);
        ...
    }
}
```
举例: 保护handler不被其他signal打断
```c
#include <signal.h>
#include <stddef.h>
void catch_stop();
void
install_handler(void)
{
    struct sigaction setup_action;
    sigset_t block_mask;
    sigemptyset(&block_mask);
    /* Block other terminal-generated signals while handler runs. */
    sigaddset(&block_mask, SIGINT);
    sigaddset(&block_mask, SIGQUIT);
    setup_action.sa_handler = catch_stop;
    setup_action.sa_mask = block_mask;
    setup_action.sa_flags = 0;
    sigaction(SIGTSTP, &setup_action, NULL);
}
```

## 正在pending的signal(已到达, 未处理)
```c
//一般用不到这函数, 除非是在sigprocmask或sa_mask保护的代码里
int sigpending (sigset t *set)
```

## 等待signal
```c
#include <unistd.h>
//阻塞进程直到signal执行
int pause (void)
需要注意:
```c
/* usr_interrupt is set by the signal handler. */
if (!usr_interrupt)
    //如果在这之间signal来了, 并让usr_interrupt=1, 那么这个signal的实际工作不会得到处理.
    pause ();
/* Do work once the signal arrives. */
...
```
可以改成这样: 不用pause()
```c
/* usr_interrupt is set by the signal handler.
while (!usr_interrupt)
sleep (1);
/* Do work once the signal arrives. */
...
```
另外一种方法:
```c
//阻塞进程, block set里面的signal. 这个函数一直阻塞, 直到非set成员的signal的handler返回以后才返回.
int sigsuspend (const sigset t *set)
```

## signal的栈
signal的栈在特殊的地方(不是进程栈吗?--默认是用进程栈), 可以用sigaltstack设定.
```c
stack_t
    void *ss_sp
    size_t ss_size
    int ss_flags
        SS_DISABLE: 不使用signal栈
        SS_ONSTACK: 为1表示正在用signal栈. 为0表示用的是进程的栈. 由OS设置
/* 用这个函数在板子上测试, 结果为
** ss_sp:(nil), ss_size:0, ss_flags:0x2(SS_DISABLE)
** 表示不用signal的栈, 使用进程栈
*/
int sigaltstack (const stack t *restrict stack, stack t *restrict oldstack)
```

# 基础系统接口
```c
//main
int main (int argc, char *argv[])
//长参数形式: --name=value
## 参数解析之getopt 
```c
#include <unistd.h>
/* 会用到几个变量
** int opterr; int optopt; int optind; char * optarg;
** : 表示需要参数 :: 表示参数可选
** 在循环里调用
*/
int getopt (int argc, char *const *argv, const char *options)
```
举例:
```c
#include <ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
int
main(int argc, char **argv)
{
    int aflag = 0;
    int bflag = 0;
    char *cvalue = NULL;
    int index;
    int c;
    opterr = 0;
 
    while ((c = getopt(argc, argv, "abc:")) != -1)
        switch (c) {
        case ’a’:
            aflag = 1;
            break;
 
        case ’b’:
            bflag = 1;
            break;
 
        case ’c’:
            cvalue = optarg;
            break;
 
        case ’?’ :
                if (optopt == ’c’)
                    fprintf(stderr, "Option -%c requires an argument.\n", optopt);
                else if (isprint(optopt))
                    fprintf(stderr, "Unknown option ‘-%c’.\n", optopt);
                else
                    fprintf(stderr,
                            "Unknown option character ‘\x%x’.\n",
                            optopt);
 
            return 1;
 
        default:
            abort();
        }
 
    printf("aflag = %d, bflag = %d, cvalue = %s\n",
           aflag, bflag, cvalue);
 
    for (index = optind; index < argc; index++)
        printf("Non-option argument %s\n", argv[index]);
 
    return 0;
}
```
结果:
```c
% testopt
aflag = 0, bflag = 0, cvalue = (null)
% testopt -a -b
aflag = 1, bflag = 1, cvalue = (null)
% testopt -ab
aflag = 1, bflag = 1, cvalue = (null)
% testopt -c foo
aflag = 0, bflag = 0, cvalue = foo
% testopt -cfoo
aflag = 0, bflag = 0, cvalue = foo
% testopt arg1
aflag = 0, bflag = 0, cvalue = (null)
Non-option argument arg1
% testopt -a arg1
aflag = 1, bflag = 0, cvalue = (null)
Non-option argument arg1
% testopt -c foo arg1
aflag = 0, bflag = 0, cvalue = foo
Non-option argument arg1
% testopt -a -- -b
aflag = 1, bflag = 0, cvalue = (null)
Non-option argument -b
% testopt -a -aflag = 1, bflag = 0, cvalue = (null)
Non-option argument -
```

## 参数解析之getopt_long 
```c
#include <getopt.h>
int getopt_long (int argc, char *const *argv, const char *shortopts, const struct option *longopts, int *indexptr)
struct option
    const char *name
    int has_arg
    int *flag
    int val
```
举例:
```c
#include <stdio.h>
#include <stdlib.h>
#include <getopt.h>
/* Flag set by ‘--verbose’. */
static int verbose_flag;
int
main(int argc, char **argv)
{
    int c;
 
    while (1) {
        static struct option long_options[] = {
            /* These options set a flag. */
            {"verbose", no_argument, &verbose_flag, 1},
            {"brief", no_argument, &verbose_flag, 0},
            /* These options don’t set a flag.
            We distinguish them by their indices. */
            {"add", no_argument, 0, ’a’},
            {"append", no_argument, 0, ’b’},
            {"delete", required_argument, 0, ’d’},
            {"create", required_argument, 0, ’c’},
            {"file", required_argument, 0, ’f’},
            {0, 0, 0, 0}
        };
        /* getopt_long stores the option index here. */
        int option_index = 0;
        c = getopt_long(argc, argv, "abc:d:f:",
                        long_options, &option_index);
 
        /* Detect the end of the options. */
        if (c == -1)
            break;
 
        switch (c) {
        case 0:
 
            /* If this option set a flag, do nothing else now. */
            if (long_options[option_index].flag != 0)
                break;
 
            printf("option %s", long_options[option_index].name);
 
            if (optarg)
                printf(" with arg %s", optarg);
 
            printf("\n");
            break;
 
        case ’a’:
            puts("option -a\n");
            break;
 
        case ’b’:
            puts("option -b\n");
            break;
 
        case ’c’:
            printf("option -c with value ‘%s’\n", optarg);
            break;
 
        case ’d’:
            printf("option -d with value ‘%s’\n", optarg);
            break;
 
        case ’f’:
            printf("option -f with value ‘%s’\n", optarg);
            break;
 
        case ’?’ :
                /* getopt_long already printed an error message. */
                break;
 
        default:
            abort();
        }
    }
 
    /* Instead of reporting ‘--verbose’
    and ‘--brief’ as they are encountered,
    we report the final status resulting from them. */
    if (verbose_flag)
        puts("verbose flag is set");
 
    /* Print any remaining command line arguments (not options). */
    if (optind < argc) {
        printf("non-option ARGV-elements: ");
 
        while (optind < argc)
            printf("%s ", argv[optind++]);
 
        putchar(’\n’);
    }
 
    exit(0);
}
```

## 参数解析之argp_parse
```c
//好像很复杂
error_t argp_parse (const struct argp *argp, int argc, char **argv, unsigned flags, int *arg_index, void *input)
```
举例:最简单
```c
/* This is (probably) the smallest possible program that
uses argp. It won’t do much except give an error
messages and exit when there are any arguments, and print
a (rather pointless) messages for –help. */
#include <stdlib.h>
#include <argp.h>
int
main(int argc, char **argv)
{
    argp_parse(0, argc, argv, 0, 0, 0);
    exit(0);
}
```
举例: 使用默认选项的例子
```c
/* This program doesn’t use any options or arguments, but uses
argp to be compliant with the GNU standard command line
format.
In addition to making sure no arguments are given, and
implementing a –help option, this example will have a
–version option, and will put the given documentation string
and bug address in the –help output, as per GNU standards.
The variable ARGP contains the argument parser specification;
adding fields to this structure is the way most parameters are
passed to argp parse (the first three fields are usually used,
but not in this small program). There are also two global
variables that argp knows about defined here,
ARGP PROGRAM VERSION and ARGP PROGRAM BUG ADDRESS (they are
global variables because they will almost always be constant
for a given program, even if it uses different argument
parsers for various tasks). */
#include <stdlib.h>
#include <argp.h>
const char *argp_program_version =
    "argp-ex2 1.0";
const char *argp_program_bug_address =
    "<bug-gnu-utils@gnu.org>";
/* Program documentation. */
static char doc[] =
    "Argp example #2 -- a pretty minimal program using argp";
/* Our argument parser. The options, parser, and
args_doc fields are zero because we have neither options or
arguments; doc and argp_program_bug_address will be
used in the output for ‘--help’, and the ‘--version’
option will print out argp_program_version. */
static struct argp argp = { 0, 0, 0, doc };
int
main(int argc, char **argv)
{
    argp_parse(&argp, argc, argv, 0, 0, 0);
    exit(0);
}
```
举例: 增加一些option
```c
/* This program uses the same features as example 2, and uses options and
arguments.
We now use the first four fields in ARGP, so here’s a description of them:
OPTIONS – A pointer to a vector of struct argp option (see below)
PARSER – A function to parse a single option, called by argp
ARGS DOC – A string describing how the non-option arguments should look
DOC – A descriptive string about this program; if it contains a
vertical tab character (\v), the part after it will be
printed *following* the options
The function PARSER takes the following arguments:
KEY – An integer specifying which option this is (taken
from the KEY field in each struct argp option), or
a special key specifying something else; the only
special keys we use here are ARGP KEY ARG, meaning
a non-option argument, and ARGP KEY END, meaning
that all arguments have been parsed
ARG – For an option KEY, the string value of its
argument, or NULL if it has none
STATE– A pointer to a struct argp state, containing
various useful information about the parsing state; used here
are the INPUT field, which reflects the INPUT argument to
argp parse, and the ARG NUM field, which is the number of the
current non-option argument being parsed
It should return either 0, meaning success, ARGP ERR UNKNOWN, meaning the
given KEY wasn’t recognized, or an errno value indicating some other
error.
Note that in this example, main uses a structure to communicate with the
parse opt function, a pointer to which it passes in the INPUT argument to
argp parse. Of course, it’s also possible to use global variables
instead, but this is somewhat more flexible.
The OPTIONS field contains a pointer to a vector of struct argp option’s;
that structure has the following fields (if you assign your option
structures using array initialization like this example, unspecified
fields will be defaulted to 0, and need not be specified):
NAME – The name of this option’s long option (may be zero)
KEY – The KEY to pass to the PARSER function when parsing this option,
*and* the name of this option’s short option, if it is a
printable ascii character
ARG – The name of this option’s argument, if any
FLAGS – Flags describing this option; some of them are:
OPTION ARG OPTIONAL – The argument to this option is optional
OPTION ALIAS – This option is an alias for the
previous option
OPTION HIDDEN – Don’t show this option in –help output
DOC – A documentation string for this option, shown in –help output
An options vector should be terminated by an option with all fields zero. */
#include <stdlib.h>
#include <argp.h>
const char *argp_program_version =
    "argp-ex3 1.0";
const char *argp_program_bug_address =
    "<bug-gnu-utils@gnu.org>";
/* Program documentation. */
static char doc[] =
    "Argp example #3 -- a program with options and arguments using argp";
/* A description of the arguments we accept. */
static char args_doc[] = "ARG1 ARG2";
/* The options we understand. */
static struct argp_option options[] = {
    {"verbose", ’v’, 0, 0, "Produce verbose output" },
    {"quiet", ’q’, 0, 0, "Don’t produce any output" },
    {"silent", ’s’, 0, OPTION_ALIAS },
    {
        "output", ’o’, "FILE", 0,
        "Output to FILE instead of standard output"
    },
    { 0 }
};
/* Used by main to communicate with parse_opt. */
struct arguments {
    char *args[2]; /* arg1 & arg2 */
    int silent, verbose;
    char *output_file;
};
/* Parse a single option. */
static error_t
parse_opt(int key, char *arg, struct argp_state *state)
{
    /* Get the input argument from argp_parse, which we
    know is a pointer to our arguments structure. */
    struct arguments *arguments = state->input;
 
    switch (key) {
    case ’q’:
    case ’s’:
        arguments->silent = 1;
        break;
 
    case ’v’:
        arguments->verbose = 1;
        break;
 
    case ’o’:
        arguments->output_file = arg;
        break;
 
    case ARGP_KEY_ARG:
        if (state->arg_num >= 2)
            /* Too many arguments. */
            argp_usage(state);
 
        arguments->args[state->arg_num] = arg;
        break;
 
    case ARGP_KEY_END:
        if (state->arg_num < 2)
            /* Not enough arguments. */
            argp_usage(state);
 
        break;
 
    default:
        return ARGP_ERR_UNKNOWN;
    }
 
    return 0;
}
/* Our argp parser. */
static struct argp argp = { options, parse_opt, args_doc, doc };
int
main(int argc, char **argv)
{
    struct arguments arguments;
    /* Default values. */
    arguments.silent = 0;
    arguments.verbose = 0;
    arguments.output_file = "-";
    /* Parse our arguments; every option seen by parse_opt will
    be reflected in arguments. */
    argp_parse(&argp, argc, argv, 0, 0, &arguments);
    printf("ARG1 = %s\nARG2 = %s\nOUTPUT_FILE = %s\n"
           "VERBOSE = %s\nSILENT = %s\n",
           arguments.args[0], arguments.args[1],
           arguments.output_file,
           arguments.verbose ? "yes" : "no",
           arguments.silent ? "yes" : "no");
    exit(0);
}
```
举例: 更复杂的一个例子
```c
/* This program uses the same features as example 3, but has more
options, and somewhat more structure in the -help output. It
also shows how you can ‘steal’ the remainder of the input
arguments past a certain point, for programs that accept a
list of items. It also shows the special argp KEY value
ARGP KEY NO ARGS, which is only given if no non-option
arguments were supplied to the program.
For structuring the help output, two features are used,
*headers* which are entries in the options vector with the
first four fields being zero, and a two part documentation
string (in the variable DOC), which allows documentation both
before and after the options; the two parts of DOC are
separated by a vertical-tab character (’\v’, or ’\013’). By
convention, the documentation before the options is just a
short string saying what the program does, and that afterwards
is longer, describing the behavior in more detail. All
documentation strings are automatically filled for output,
although newlines may be included to force a line break at a
particular point. All documentation strings are also passed to
the ‘gettext’ function, for possible translation into the
current locale. */
#include <stdlib.h>
#include <error.h>
#include <argp.h>
const char *argp_program_version =
    "argp-ex4 1.0";
const char *argp_program_bug_address =
    "<bug-gnu-utils@prep.ai.mit.edu>";
/* Program documentation. */
static char doc[] =
    "Argp example #4 -- a program with somewhat more complicatedoptions\vThis part of the documentation comes *after* the options;note that the text is automatically filled, but it’s possibleto force a line-break, e.g.\n<-- here.";
/* A description of the arguments we accept. */
static char args_doc[] = "ARG1 [STRING...]";
/* Keys for options without short-options. */
#define OPT_ABORT 1 /* –abort */
/* The options we understand. */
static struct argp_option options[] = {
    {"verbose", ’v’, 0, 0, "Produce verbose output" },
    {"quiet", ’q’, 0, 0, "Don’t produce any output" },
    {"silent", ’s’, 0, OPTION_ALIAS },
    {
        "output", ’o’, "FILE", 0,
        "Output to FILE instead of standard output"
    },
    {0, 0, 0, 0, "The following options should be grouped together:" },
    {
        "repeat", ’r’, "COUNT", OPTION_ARG_OPTIONAL,
        "Repeat the output COUNT (default 10) times"
    },
    {"abort", OPT_ABORT, 0, 0, "Abort before showing any output"},
    { 0 }
};
/* Used by main to communicate with parse_opt. */
struct arguments {
    char *arg1; /* arg1 */
    char **strings; /* [string . . .] */
    int silent, verbose, abort; /* ‘-s’, ‘-v’, ‘--abort’ */
    char *output_file; /* file arg to ‘--output’ */
    int repeat_count; /* count arg to ‘--repeat’ */
};
/* Parse a single option. */
static error_t
parse_opt(int key, char *arg, struct argp_state *state)
{
    /* Get the input argument from argp_parse, which we
    know is a pointer to our arguments structure. */
    struct arguments *arguments = state->input;
 
    switch (key) {
    case ’q’:
    case ’s’:
        arguments->silent = 1;
        break;
 
    case ’v’:
        arguments->verbose = 1;
        break;
 
    case ’o’:
        arguments->output_file = arg;
        break;
 
    case ’r’:
        arguments->repeat_count = arg ? atoi(arg) : 10;
        break;
 
    case OPT_ABORT:
        arguments->abort = 1;
        break;
 
    case ARGP_KEY_NO_ARGS:
        argp_usage(state);
 
    case ARGP_KEY_ARG:
        /* Here we know that state->arg_num == 0, since we
        force argument parsing to end before any more arguments can
        get here. */
        arguments->arg1 = arg;
        /* Now we consume all the rest of the arguments.
        state->next is the index in state->argv of the
        next argument to be parsed, which is the first string
        we’re interested in, so we can just use
        &state->argv[state->next] as the value for
        arguments->strings.
        In addition, by setting state->next to the end
        of the arguments, we can force argp to stop parsing here and
        return. */
        arguments->strings = &state->argv[state->next];
        state->next = state->argc;
        break;
 
    default:
        return ARGP_ERR_UNKNOWN;
    }
 
    return 0;
}
/* Our argp parser. */
static struct argp argp = { options, parse_opt, args_doc, doc };
int
main(int argc, char **argv)
{
    int i, j;
    struct arguments arguments;
    /* Default values. */
    arguments.silent = 0;
    arguments.verbose = 0;
    arguments.output_file = "-";
    arguments.repeat_count = 1;
    arguments.abort = 0;
    /* Parse our arguments; every option seen by parse_opt will be
    reflected in arguments. */
    argp_parse(&argp, argc, argv, 0, 0, &arguments);
 
    if (arguments.abort)
        error(10, 0, "ABORTED");
 
    for (i = 0; i < arguments.repeat_count; i++) {
        printf("ARG1 = %s\n", arguments.arg1);
        printf("STRINGS = ");
 
        for (j = 0; arguments.strings[j]; j++)
            printf(j == 0 ? "%s" : ", %s", arguments.strings[j]);
 
        printf("\n");
        printf("OUTPUT_FILE = %s\nVERBOSE = %s\nSILENT = %s\n",
               arguments.output_file,
               arguments.verbose ? "yes" : "no",
               arguments.silent ? "yes" : "no");
    }
 
    exit(0);
}
```

## 子选项支持
```c
int getsubopt (char **optionp, char *const *tokens, char **valuep)
```
举例:
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
int do_all;
const char *type;
int read_size;
int write_size;
int read_only;
enum {
    RO_OPTION = 0,
    RW_OPTION,
    READ_SIZE_OPTION,
    WRITE_SIZE_OPTION,
    THE_END
};
const char *mount_opts[] = {
    [RO_OPTION] = "ro",
    [RW_OPTION] = "rw",
    [READ_SIZE_OPTION] = "rsize",
    [WRITE_SIZE_OPTION] = "wsize",
    [THE_END] = NULL
};
int
main(int argc, char **argv)
{
    char *subopts, *value;
    int opt;
 
    while ((opt = getopt(argc, argv, "at:o:")) != -1)
        switch (opt) {
        case ’a’:
            do_all = 1;
            break;
 
        case ’t’:
            type = optarg;
            break;
 
        case ’o’:
            subopts = optarg;
 
            while (*subopts != ’\0’)
                switch (getsubopt(&subopts, mount_opts, &value)) {
                case RO_OPTION:
                    read_only = 1;
                    break;
 
                case RW_OPTION:
                    read_only = 0;
                    break;
 
                case READ_SIZE_OPTION:
                    if (value == NULL)
                        abort();
 
                    read_size = atoi(value);
                    break;
 
                case WRITE_SIZE_OPTION:
                    if (value == NULL)
                        abort();
 
                    write_size = atoi(value);
                    break;
 
                default:
                    /* Unknown suboption. */
                    printf("Unknown suboption ‘%s’\n", value);
                    break;
                }
 
            break;
 
        default:
            abort();
        }
 
    /* Do the real work. */
    return 0;
}
```

## 环境变量
```c
#include <stdlib.h>
char * getenv (const char *name)
char * secure_getenv (const char *name)
//name=value, 如果没有=, 相当于定义xxxx为空
int putenv (char *string)
int setenv (const char *name, const char *value, int replace)
int unsetenv (const char *name)
int clearenv (void)
//env的全局数组, 每一项都是name=value
#include <unistd.h>
char ** environ
```

## syscall
```c
#include <unistd.h>
#include <sys/syscall.h>
```
例如:
```c
#include <unistd.h>
#include <sys/syscall.h>
#include <errno.h>
...
int rc;
rc = syscall(SYS_chmod, "/etc/passwd", 0444);
if (rc == -1)
fprintf(stderr, "chmod failed, errno = %d\n", errno);
```
又如:
```c
#include <sys/types.h>
#include <sys/stat.h>
#include <errno.h>
...
int rc;
rc = chmod("/etc/passwd", 0444);
if (rc == -1)
fprintf(stderr, "chmod failed, errno = %d\n", errno);
```

## 终结程序
*  main的return
*  abort会触发SIGABRT
*  exit
```c
/*终止进程, 不返回; main的return值随后被传入status
**status范围是0-255, 通常0(EXIT_SUCCESS)表示成功, 1(EXIT_FAILURE)表示失败
**用atexit或on_exit注册的函数会被调用
**关闭所有stream, flush buffers, remove tmpfile
**最后调用_exit终止进程
*/
#include <stdlib.h>
void exit (int status)
```c
//exit钩子函数
int atexit (void (*function) (void))
int on_exit (void (*function)(int status, void *arg), void *arg)
//abort函数, 会触发SIGABRT. 不会回调exit钩子. 不推荐使用
void abort (void)
/*真正的终止函数, 不管是哪种终止, 最终都会调用它:
** 1. 关闭所有文件描述符, 注意它不会把流文件flush buffer(exit会).
** 2. 报告进程的exit status到父进程的wait
** 3. 下面的子进程被init进程接管
** 4. 向父进程发送SIGCHLD
** 5. 如果这个进程是个终端会话主进程(比如ssh), 那么它会发送SIGHUP到所有前台子进程, 并释放这个终端.
** 6. 如果进程终止导致进程组变为孤儿, 则这个孤儿进程组会变为stop状态, 然后组里的每个进程都会收到SIGHUP和SIGCONT
*/
void _exit (int status)
```

# 进程
```c
//以阻塞方式执行一个子程序
#include <stdlib.h>
int system (const char *command)
//进程id和父进程id
#include <sys/types.h>
pid_t getpid (void)
pid_t getppid (void)
```

## fork
```c
/* 新进程
** pid不同
** 父子共享文件描述符的posion?
** 子进程不继承文件锁
** 不继承alarm
** 继承signal, 以及block mask, 但清除pending的signal
*/
#include <unistd.h>
pid_t fork (void)
//共享父进程地址空间
pid_t vfork (void)
```

## exec
```c
/*exec
** pid ppid group session user pwd不变
** alarm不变
** fd不变, 但stream都没了(因为stream属于进程空间, 而fd是内核管理?)
** signal变成default, 但ignore的signal继承
** 时间片不变
*/
int execv (const char *filename, char *const argv[])
int execl (const char *filename, const char *arg0, . . . )
int execve (const char *filename, char *const argv[], char *const env[])
int execle (const char *filename, const char *arg0, . . ., char *const env[])
int execvp (const char *filename, char *const argv[])
int execlp (const char *filename, const char *arg0, . . . )
```

## 等待子进程终止
```c
pid_t waitpid (pid t pid, int *status-ptr, int options)
//相当于waitpid (-1, &status, 0)
pid_t wait (int *status-ptr)
//多了一个参数usage, 子进程的资源
pid_t wait4 (pid t pid, int *status-ptr, int options, struct rusage *usage)
```
举例: 使用signal来处理子进程终止
```c
void
sigchld_handler(int signum)
{
    int pid, status, serrno;
    serrno = errno;
 
    while (1) {
        pid = waitpid(WAIT_ANY, &status, WNOHANG);
 
        if (pid < 0) {
            perror("waitpid");
            break;
        }
 
        if (pid == 0)
            break;
 
        notice_termination(pid, status);
    }
 
    errno = serrno;
}
```

## 子进程退出状态
```c
//子进程用exit或_exit退出
int WIFEXITED (int status)
//上面为真, 则这个返回main的返回值, 或调用exit()传入的返回值
int WEXITSTATUS (int status)
//signal导致子进程终止
int WIFSIGNALED (int status)
//上面为真时返回signal号
int WTERMSIG (int status)
//是否产生了core dump
int WCOREDUMP (int status)
//子进程stop
int WIFSTOPPED (int status)
//signal导致子进程stop
int WSTOPSIG (int status)
举例:
```c
#include <stddef.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
/* Execute the command using this shell program. */
#define SHELL "/bin/sh"
int
my_system(const char *command)
{
    int status;
    pid_t pid;
    pid = fork();
 
    if (pid == 0) {
        /* This is the child process. Execute the shell command. */
        execl(SHELL, SHELL, "-c", command, NULL);
        /*exec失败, 直接终止子进程; 不要调用exit(), 因为它会flush从父进程继承过来的stream
        **而这些stream会被父进程flush, 导致output两次
        */
        _exit(EXIT_FAILURE);
    } else if (pid < 0)
        /* The fork failed. Report failure. */
        status = -1;
    else
 
        /* This is the parent process. Wait for the child to complete. */
        if (waitpid(pid, &status, 0) != pid)
            status = -1;
 
    return status;
}
```

# job控制
*  signal的分发是由进程组决定的. 前台进程组的每个进程都会收到由用户触发的SIGTSTP, SIGQUIT, SIGINT 信号
后台进程组不允许读控制台, 否则会触发 SIGTTIN and SIGTTOU信号到组内每个进程(默认stop), 但后台进程组一般可以写控制台
*  shell会把命令管道分成若干进程组, 并选定一个进程组为前台进程组, 该进程组享有控制台. 剩下的进程组为后台进程组.
*  一般一次登录产生一个会话, 里面包含若个进程组. 
*  会话leader进程也称为终端的控制进程, 这个进程挂了, 则这个会话的进程组就成了孤儿进程组, 每个进程(还是前天进程?)都会收到SIGHUP信号
*  tcsetpgrp可以获得终端控制权, shell起前台进程时需要调用, 前台进程返回后, shell也要调用tcsetpgrp一次以重新获得终端控制权.
*  终端控制权切换时, 需要用tcgetattr()和tcsetattr()来恢复终端属性.
默认终止该进程
## 获得控制终端名字(和ttyname有什么区别?)
```c
char * ctermid (char *string)
```
## 进程组api
```c
//调用进程会新建会话, 并成为leader, 其pid==进程组id
pid_t setsid (void)
//获得session leader的进程组id
pid_t getsid (pid t pid)
//获得调用进程的进程组id
pid_t getpgrp (void)
//获得pid的进程组id
int getpgid (pid t pid)
//将pid加入pgid组
int setpgid (pid t pid, pid t pgid)
//获得控制台进程组
pid_t tcgetpgrp (int filedes)
//设置控制台进程组
int tcsetpgrp (int filedes, pid t pgid)
```
## 一个简单的shell
```c
/* A process is a single process. */
typedef struct process {
    struct process *next; /* next process in pipeline */
    char **argv; /* for exec */
    pid_t pid; /* process ID */
    char completed; /* true if process has completed */
    char stopped; /* true if process has stopped */
    int status; /* reported status value */
} process;
/* A job is a pipeline of processes. */
typedef struct job {
    struct job *next; /* next active job */
    char *command; /* command line, used for messages */
    process *first_process; /* list of processes in this job */
    pid_t pgid; /* process group ID */
    char notified; /* true if user told about stopped job */
    struct termios tmodes; /* saved terminal modes */
    int stdin, stdout, stderr; /* standard i/o channels */
} job;
/* The active jobs are linked into a list. This is its head. */
job *first_job = NULL;
/* Find the active job with the indicated pgid. */
job *
find_job(pid_t pgid)
{
    job *j;
 
    for (j = first_job; j; j = j->next)
        if (j->pgid == pgid)
            return j;
 
    return NULL;
}
/* Return true if all processes in the job have stopped or completed. */
int
job_is_stopped(job *j)
{
    process *p;
 
    for (p = j->first_process; p; p = p->next)
        if (ps > completed && !p->stopped)
            return 0;
 
    return 1;
}
/* Return true if all processes in the job have completed. */
int
job_is_completed(job *j)
{
    process *p;
 
    for (p = j->first_process; p; p = p->next)
        if (!p->completed)
            return 0;
 
    return 1;
}
 
/* Keep track of attributes of the shell. */
#include <sys/types.h>
#include <termios.h>
#include <unistd.h>
pid_t shell_pgid;
struct termios shell_tmodes;
int shell_terminal;
int shell_is_interactive;
/* Make sure the shell is running interactively as the foreground job
before proceeding. */
void
init_shell()
{
    /* See if we are running interactively. */
    shell_terminal = STDIN_FILENO;
    shell_is_interactive = isatty(shell_terminal);
 
    if (shell_is_interactive) {
        /* Loop until we are in the foreground. */
        while (tcgetpgrp(shell_terminal) != (shell_pgid = getpgrp()))
            kill(- shell_pgid, SIGTTIN);
 
        /* Ignore interactive and job-control signals. */
        signal(SIGINT, SIG_IGN);
        signal(SIGQUIT, SIG_IGN);
        signal(SIGTSTP, SIG_IGN);
        signal(SIGTTIN, SIG_IGN);
        signal(SIGTTOU, SIG_IGN);
        signal(SIGCHLD, SIG_IGN);
        /* Put ourselves in our own process group. */
        shell_pgid = getpid();
 
        if (setpgid(shell_pgid, shell_pgid) < 0) {
            perror("Couldn’t put the shell in its own process group");
            exit(1);
        }
 
        /* Grab control of the terminal. */
        tcsetpgrp(shell_terminal, shell_pgid);
        /* Save default terminal attributes for shell. */
        tcgetattr(shell_terminal, &shell_tmodes);
    }
}
 
void
launch_process(process *p, pid_t pgid,
               int infile, int outfile, int errfile,
               int foreground)
{
    pid_t pid;
 
    if (shell_is_interactive) {
        /* Put the process into the process group and give the process group
        the terminal, if appropriate.
        This has to be done both by the shell and in the individual
        child processes because of potential race conditions. */
        pid = getpid();
 
        if (pgid == 0) pgid = pid;
 
        setpgid(pid, pgid);
 
        if (foreground)
            tcsetpgrp(shell_terminal, pgid);
 
        /* Set the handling for job control signals back to the default. */
        signal(SIGINT, SIG_DFL);
        signal(SIGQUIT, SIG_DFL);
        signal(SIGTSTP, SIG_DFL);
        signal(SIGTTIN, SIG_DFL);
        signal(SIGTTOU, SIG_DFL);
        signal(SIGCHLD, SIG_DFL);
    }
 
    /* Set the standard input/output channels of the new process. */
    if (infile != STDIN_FILENO) {
        dup2(infile, STDIN_FILENO);
        close(infile);
    }
 
    if (outfile != STDOUT_FILENO) {
        dup2(outfile, STDOUT_FILENO);
        close(outfile);
    }
 
    if (errfile != STDERR_FILENO) {
        dup2(errfile, STDERR_FILENO);
        close(errfile);
    }
 
    /* Exec the new process. Make sure we exit. */
    execvp(p->argv[0], p->argv);
    perror("execvp");
    exit(1);
}
 
void
launch_job(job *j, int foreground)
{
    process *p;
    pid_t pid;
    int mypipe[2], infile, outfile;
    infile = j->stdin;
 
    for (p = j->first_process; p; p = p->next) {
        /* Set up pipes, if necessary. */
        if (p->next) {
            if (pipe(mypipe) < 0) {
                perror("pipe");
                exit(1);
            }
 
            outfile = mypipe[1];
        } else
            outfile = j->stdout;
 
        /* Fork the child processes. */
        pid = fork();
 
        if (pid == 0)
            /* This is the child process. */
            launch_process(p, j->pgid, infile,
                           outfile, j->stderr, foreground);
        else if (pid < 0) {
            /* The fork failed. */
            perror("fork");
            exit(1);
        } else {
            /* This is the parent process. */
            p->pid = pid;
 
            if (shell_is_interactive) {
                if (!j->pgid)
                    j->pgid = pid;
 
                setpgid(pid, j->pgid);
            }
        }
 
        /* Clean up after pipes. */
        if (infile != j->stdin)
            close(infile);
 
        if (outfile != j->stdout)
            close(outfile);
 
        infile = mypipe[0];
    }
 
    format_job_info(j, "launched");
 
    if (!shell_is_interactive)
        wait_for_job(j);
    else if (foreground)
        put_job_in_foreground(j, 0);
    else
        put_job_in_background(j, 0);
}
 
/* Put job j in the foreground. If cont is nonzero,
restore the saved terminal modes and send the process group a
SIGCONT signal to wake it up before we block. */
void
put_job_in_foreground(job *j, int cont)
{
    /* Put the job into the foreground. */
    tcsetpgrp(shell_terminal, j->pgid);
 
    /* Send the job a continue signal, if necessary. */
    if (cont) {
        tcsetattr(shell_terminal, TCSADRAIN, &j->tmodes);
 
        if (kill(- j->pgid, SIGCONT) < 0)
            perror("kill (SIGCONT)");
    }
 
    /* Wait for it to report. */
    wait_for_job(j);
    /* Put the shell back in the foreground. */
    tcsetpgrp(shell_terminal, shell_pgid);
    /* Restore the shell’s terminal modes. */
    tcgetattr(shell_terminal, &j->tmodes);
    tcsetattr(shell_terminal, TCSADRAIN, &shell_tmodes);
}
 
/* Put a job in the background. If the cont argument is true, send
the process group a SIGCONT signal to wake it up. */
void
put_job_in_background(job *j, int cont)
{
    /* Send the job a continue signal, if necessary. */
    if (cont)
        if (kill(-j->pgid, SIGCONT) < 0)
            perror("kill (SIGCONT)");
}
 
/* Store the status of the process pid that was returned by waitpid.
Return 0 if all went well, nonzero otherwise. */
int
mark_process_status(pid_t pid, int status)
{
    job *j;
    process *p;
 
    if (pid > 0) {
        /* Update the record for the process. */
        for (j = first_job; j; j = j->next)
            for (p = j->first_process; p; p = p->next)
                if (p->pid == pid) {
                    p->status = status;
 
                    if (WIFSTOPPED(status))
                        p->stopped = 1;
                    else {
                        p->completed = 1;
 
                        if (WIFSIGNALED(status))
                            fprintf(stderr, "%d: Terminated by signal %d.\n",
                                    (int) pid, WTERMSIG(p->status));
                    }
 
                    return 0;
                }
 
        fprintf(stderr, "No child process %d.\n", pid);
        return -1;
    } else if (pid == 0 || errno == ECHILD)
        /* No processes ready to report. */
        return -1;
    else {
        /* Other weird errors. */
        perror("waitpid");
        return -1;
    }
}
/* Check for processes that have status information available,
without blocking. */
void
update_status(void)
{
    int status;
    pid_t pid;
 
    do
        pid = waitpid(WAIT_ANY, &status, WUNTRACED | WNOHANG);
 
    while (!mark_process_status(pid, status));
}
/* Check for processes that have status information available,
blocking until all processes in the given job have reported. */
void
wait_for_job(job *j)
{
    int status;
    pid_t pid;
 
    do
        pid = waitpid(WAIT_ANY, &status, WUNTRACED);
 
    while (!mark_process_status(pid, status)
           && !job_is_stopped(j)
           && !job_is_completed(j));
}
/* Format information about job status for the user to look at. */
void
format_job_info(job *j, const char *status)
{
    fprintf(stderr, "%ld (%s): %s\n", (long)j->pgid, status, j->command);
}
/* Notify the user about stopped or terminated jobs.
Delete terminated jobs from the active job list. */
void
do_job_notification(void)
{
    job *j, *jlast, *jnext;
    process *p;
    /* Update status information for child processes. */
    update_status();
    jlast = NULL;
 
    for (j = first_job; j; j = jnext) {
        jnext = j->next;
 
        /* If all processes have completed, tell the user the job has
        completed and delete it from the list of active jobs. */
        if (job_is_completed(j)) {
            format_job_info(j, "completed");
 
            if (jlast)
                jlast->next = jnext;
            else
                first_job = jnext;
 
            free_job(j);
        }
        /* Notify the user about stopped jobs,
        marking them so that we won’t do this more than once. */
        else if (job_is_stopped(j) && j > notified) {
            format_job_info(j, "stopped");
            j->notified = 1;
            jlast = j;
        }
        /* Don’t say anything about jobs that are still running. */
        else
            jlast = j;
    }
}
/* Mark a stopped job J as being running again. */
void
mark_job_as_running(job *j)
{
    Process *p;
 
    for (p = j->first_process; p; p = p->next)
        p->stopped = 0;
 
    j->notified = 0;
}
/* Continue the job J. */
void
continue_job(job *j, int foreground)
{
    mark_job_as_running(j);
 
    if (foreground)
        put_job_in_foreground(j, 1);
    else
        put_job_in_background(j, 1);
}
```

# 用户和组
*  一个进程有effective用户ID(权限控制)和real用户ID(who创建了这个进程)
## api
```c
uid_t getuid (void)
gid_t getgid (void)
uid_t geteuid (void)
gid_t getegid (void)
int getgroups (int count, gid t *groups)
int seteuid (uid t neweuid)
int setuid (uid t newuid)
int setreuid (uid t ruid, uid t euid)
int setegid (gid t newgid)
int setgid (gid t newgid)
int setregid (gid t rgid, gid t egid)
int setgroups (size t count, const gid t *groups)
int initgroups (const char *user, gid t group)
int getgrouplist (const char *user, gid t group, gid t *groups, int *ngroups)
char * getlogin (void)
char * cuserid (char *string)
int login_tty (int filedes)
void login (const struct utmp *entry)
int logout (const char *ut_line)
void logwtmp (const char *ut_line, const char *ut_name, const char *ut_host)
```

# 系统管理
```c
int gethostname (char *name, size t size)
int sethostname (const char *name, size t length)
int getdomainnname (char *name, size t length)
int setdomainname (const char *name, size t length)
long int gethostid (void)
int sethostid (long int id)
int uname (struct utsname *info)
int mount (const char *special_file, const char *dir, const char *fstype, unsigned long int options, const void *data)
int umount (const char *file)
```
例如:
```c
#include <sys/mount.h>
mount("/dev/hdb", "/cdrom", MS_MGC_VAL | MS_RDONLY | MS_NOSUID, "");
mount("/dev/hda2", "/mnt", MS_MGC_VAL | MS_REMOUNT, "");
```
```c
//系统参数
#include <sys/sysctl.h>
int sysctl (int *names, int nlen, void *oldval, size t *oldlenp, void *newval, size t newlen)
```c
//获取系统运行时配置参数
long int sysconf (int parameter)
 
int
have_job_control (void)
{
    #ifdef _POSIX_JOB_CONTROL
        return 1;
    #else
    int value = sysconf (_SC_JOB_CONTROL);
    if (value < 0)
        /* If the system is that badly wedged,
        there’s no use trying to go on. */
        fatal (strerror (errno));
    return value;
    #endif
}
int
get_child_max ()
{
    #ifdef CHILD_MAX
    return CHILD_MAX;
    #else
    int value = sysconf (_SC_CHILD_MAX);
    if (value < 0)
        fatal (strerror (errno));
    return value;
    #endif
}
```

# 调试
## 调用栈
```c
/*调用栈
**buffer里一个frame指针对应一项
**返回实际的栈帧数
**优化选项会影响栈回溯, 比如inline和frame pointer elimination
*/
#include <execinfo.h>
int backtrace (void **buffer, int size)
//解析调用栈符号表, 前提是运行的elf不是strip过的?
char ** backtrace_symbols (void *const *buffer, int size)
举例:
```c
#include <execinfo.h>
#include <stdio.h>
#include <stdlib.h>
/* Obtain a backtrace and print it to stdout. */
void
print_trace(void)
{
    void *array[10];
    size_t size;
    char **strings;
    size_t i;
    size = backtrace(array, 10);
    strings = backtrace_symbols(array, size);
    printf("Obtained %zd stack frames.\n", size);
 
    for (i = 0; i < size; i++)
        printf("%s\n", strings[i]);
 
    free(strings);
}
/* A dummy function to make the backtrace more interesting. */
void
dummy_function(void)
{
    print_trace();
}
int
main(void)
{
    dummy_function();
    return 0;
}
```

# pthread
详见POSIX Threads Programming(非常强大)
## gnu扩展
```c
//线程独立的data
int pthread_key_create (pthread key t *key, void (*destructor)(void*))
int pthread_key_delete (pthread key t key)
void *pthread_getspecific (pthread key t key)
int pthread_setspecific (pthread key t key, const void *value)
```

# libc的系统桩函数(略)

# libc基础函数
```c
//断言, false则打印__FILE__ __LINE__并abort(). 断言可以用宏NDEBUG禁止断言检查
#include <assert.h>
void assert (int expression)
void assert_perror (int errnum)

//可变参数
#include <stdarg.h>
```
举例:
```c
#include <stdarg.h>
#include <stdio.h>
int
add_em_up(int count, ...)
{
    va_list ap;
    int i, sum;
    va_start(ap, count);  /* Initialize the argument list. */
    sum = 0;
 
    for (i = 0; i < count; i++)
        sum += va_arg(ap, int);  /* Get the next argument value. */
 
    va_end(ap);  /* Clean up. */
    return sum;
}
int
main(void)
{
    /* This call prints 16. */
    printf("%d\n", add_em_up(3, 5, 5, 6));
    /* This call prints 55. */
    printf("%d\n", add_em_up(10, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10));
    return 0;
}

//结构体偏移量
size_t offsetof (type, member)
```
