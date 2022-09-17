本文为阅读GNU libc手册的一些笔记

libc的手册从这里下载: https://www.gnu.org/software/libc/manual/pdf/libc.pdf

- [关于不同环境下函数调用是否安全](#关于不同环境下函数调用是否安全)
- [错误处理](#错误处理)
- [内存管理](#内存管理)
  - [内存分配](#内存分配)
  - [改变malloc的行为](#改变malloc的行为)
  - [内存检查](#内存检查)
  - [内存分配钩子](#内存分配钩子)
  - [内存debug](#内存debug)
  - [在栈上分配内存, 不需要free](#在栈上分配内存-不需要free)
  - [Obstacks](#obstacks)
  - [增大进程数据段](#增大进程数据段)
  - [锁页面, 使其不会被换出](#锁页面-使其不会被换出)
- [字符处理](#字符处理)
  - [类型判断和转换](#类型判断和转换)
  - [字符函数](#字符函数)
- [搜索与排序](#搜索与排序)
  - [比较函数原型: int comparison_fn_t (const void *, const void *);](#比较函数原型-int-comparison_fn_t-const-void--const-void-)
  - [数组](#数组)
  - [哈希](#哈希)
  - [树](#树)
- [正则](#正则)
- [输入输出之流文件](#输入输出之流文件)
  - [重定向](#重定向)
  - [流文件操作](#流文件操作)
  - [多线程与文件锁](#多线程与文件锁)
  - [关于宽字符](#关于宽字符)
  - [stream写](#stream写)
  - [stream读](#stream读)
  - [unread和peeking ahead](#unread和peeking-ahead)
  - [block读写](#block读写)
  - [格式化打印](#格式化打印)
  - [格式化输入](#格式化输入)
  - [检查文件尾, 常用于判断](#检查文件尾-常用于判断)
  - [文件位置](#文件位置)
  - [文件流的buffering](#文件流的buffering)
- [底层输入输出, 与之对应的是文件描述符](#底层输入输出-与之对应的是文件描述符)
  - [文件描述符和文件流互转](#文件描述符和文件流互转)
  - [mmap io](#mmap-io)
  - [共享内存和mmap](#共享内存和mmap)
  - [select](#select)
  - [同步io](#同步io)
  - [并行io之aio, _POSIX_ASYNCHRONOUS_IO](#并行io之aio-_posix_asynchronous_io)
  - [fcntl](#fcntl)
  - [文件锁操作](#文件锁操作)
  - [io重定向](#io重定向)
  - [ioctl](#ioctl)
- [文件系统接口](#文件系统接口)
  - [目录项](#目录项)
  - [遍历目录项](#遍历目录项)
  - [遍历目录树](#遍历目录树)
  - [文件链接](#文件链接)
  - [化简文件路径, 去掉. ..和多余的/](#化简文件路径-去掉-和多余的)
  - [删除与重命名](#删除与重命名)
  - [文件统计](#文件统计)
  - [文件类型判断, 入参是stat的st_mode域](#文件类型判断-入参是stat的st_mode域)
  - [访问权限](#访问权限)
  - [特殊文件](#特殊文件)
  - [临时文件](#临时文件)
- [管道](#管道)
  - [命名管道](#命名管道)
  - [一般来说, 管道的读写是原子的. 除非size大于PIPE_BUF](#一般来说-管道的读写是原子的-除非size大于pipe_buf)

# 关于不同环境下函数调用是否安全
*  MT-Safe: 线程安全, 多线程安全. 该函数可以在多线程环境下同时运行
*  AS-Safe: 异步调用安全, 一般用于指是否可以在signal handler里面用

# 错误处理
很多libc函数都会修改errno, 这是个全局变量, 在signal handler里面如果要修改errno, 记住先保存原来的值, 用完恢复.
可以用if来判断错误类型
`if (errno == EINTR)`
常用errno有:
*  EPERM: 权限错误
*  ENOENT: 没有这个文件或目录
*  EINTR: 被中断打断
*  EIO: 物理读写错误
*  ENXIO ENODEV: 没有这个设备或地址
*  EBADF: 文件描述符错误
*  ECHILD: 没有子进程
*  ENOMEM: 没有内存了
*  EBUSY: 忙
*  EINVAL: 参数错误
*  EAGAIN: 请重试, 当前操作被打断并立即退出, 操作未完成
*  ENOTSOCK: 没有这个socket
*  EMSGSIZE: socket 发送的msg太大
*  ENETDOWN: socket发现net down
相关函数
```c
#include <errno.h>
void perror (const char *message)
char * strerror (int errnum)
```
举例:
```c
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
FILE *
open_sesame (char *name)
{
    FILE *stream;
    errno = 0;
    stream = fopen (name, "r");
    if (stream == NULL)
    {
        fprintf (stderr, "%s: Couldn’t open file %s; %s\n",
        program_invocation_short_name, name, strerror (errno));
        exit (EXIT_FAILURE);
    }
    else
        return stream;
}
```

# 内存管理
## 内存分配
```c
#include <stdlib.h>
void * malloc (size t size) //如果分配的内存大于page size, 则malloc会使用mmap系统调用来分配内存
void free (void *ptr)
void * realloc (void *ptr, size t newsize) //在malloc得到的ptr基础上, 改变大小到newsize, 如果系统发现无法直接在后面增大, 那么会找到另外一片足够大的内存, 并拷贝原有的内容到新的ptr, 所以realloc也许会返回一个新的指针
void * calloc (size t count, size t eltsize) //分配内存并清零
void * aligned_alloc (size t alignment, size t size) //分配对齐的内存
```

## 改变malloc的行为
```c
#include <malloc.h>
int mallopt (int param, int value) //比较重要的是M_MMAP_THRESHOLD, 用来设定用mmap的阈值
```

## 内存检查
```c
#include <mcheck.h>
int mcheck (void (*abortfn) (enum mcheck status status))
enum mcheck_status mprobe (void *pointer)
```
用法:
*  在main（）函数开始时或适当位置显式地调用mcheck()
*  加-lmcheck重新连接可执行程序
*  设置环境变量MALLOC_CHECK_的值, MALLOC_CHECK_=1
*  gdb
```
(gdb) break main
Breakpoint 1, main (argc=2, argv=0xbffff964) at whatever.c:10
(gdb) command 1
Type commands for when breakpoint 1 is hit, one per line.
End with a line saying just "end".
>call mcheck(0)
>continue
>end
(gdb) ...
```

## 内存分配钩子
```c
#include <malloc.h>
__malloc_hook //函数指针, 类型: void *function (size_t size, const void *caller)
__realloc_hook //void *function (void *ptr, size_t size, const void *caller)
__free_hook //void function (void *ptr, const void *caller)
__memalign_hook //void *function (size_t alignment, size_t size, const void *caller)
```

## 内存debug
```c
#include <mcheck.h>
void mtrace (void) //这个函数会安装hook, 将malloc相关的strace输出到环境变量MALLOC_TRACE指向的文件
void muntrace (void)
```

## 在栈上分配内存, 不需要free
因为栈指针会自动回溯. 但要注意栈大小
```c
#include <stdlib.h>
void * alloca (size t size)
//举例:
int
open2 (char *str1, char *str2, int flags, int mode)
{
    //和局部字符数组相比, alloca的大小是自动扩展的.
    //GUN C标准支持动态数组, 比如这里也可以这样写 char name[strlen (str1) + strlen (str2) + 1];
    char *name = (char *) alloca (strlen (str1) + strlen (str2) + 1);
    stpcpy (stpcpy (name, str1), str2);
    return open (name, flags, mode);
}
```

## Obstacks
更详细信息参考附件
obstack是个内存池, 是个由不同size的object组成的栈, 所以它必须按后进先出的顺序free.
举例1:
```c
void test (int n) {
    struct obstack obst ;
    obstack_init (& obst );
    /* Allocate memory for a string of length n-1 */
    char * str = obstack_alloc (& obst , n * sizeof( str [0]));
    /* Allocate an array for n nodes */
    node_t ** nodes = obstack_alloc (& obst , n * sizeof( nodes [0]));
    /* Store the current mark of the obstack */
    void * mark = obstack_base (& obst );
    /* Allocate the nodes */
    for (i = 0; i < n; i ++)
        nodes [i] = obstack_alloc (& obst , sizeof( node [0]));
    /* All the marks are gone */
    obstack_free (& obst , mark );
    /* Everything has gone */
    obstack_free (& obst , NULL );
}
```
举例2: 自动增长
```c
int * read_ints (struct obstack * obst , FILE *f) {
    while (! feof (f )) {
        int x , res ;
        res = fscanf (f , "%d", & x );
        if ( res == 1)
            obstack_int_grow ( obst , x );
        else
            break;
    }
    return obstack_finish ( obst );
}
```

## 增大进程数据段
```c
int brk (void *addr) //用addr指定进程可用的end地址
void *sbrk (ptrdiff t delta) //用delta增长
```
## 锁页面, 使其不会被换出
需注意物理内存不换出会导致其他进程可用的物理页面减少
```c
int mlock (const void *addr, size t len)
int munlock (const void *addr, size t len)
int mlockall (int flags) //MCL_CURRENT|MCL_FUTURE
int munlockall (void)
```

# 字符处理
## 类型判断和转换
```c
#include <ctype.h>
int islower (int c)
int isupper (int c)
int isalpha (int c)
int isdigit (int c)
int isalnum (int c)
int isxdigit (int c)
int ispunct (int c)
int isspace (int c)
int isblank (int c)
int isgraph (int c)
int isprint (int c)
int iscntrl (int c)
int isascii (int c)
int tolower (int c)
int toupper (int c)
int toascii (int c)
```

## 字符函数
```c
#include <string.h>
size_t strlen (const char *s)
size_t strnlen (const char *s, size t maxlen)
void * memcpy (void *restrict to, const void *restrict from, size t size)
void * memmove (void *to, const void *from, size t size)
void * memset (void *block, int c, size t size)
char * strcpy (char *restrict to, const char *restrict from)
char * strncpy (char *restrict to, const char *restrict from, size t size)
char * strdup (const char *s) //把字符串s拷到malloc出来的内存里
char * strcat (char *restrict to, const char *restrict from)
int memcmp (const void *a1, const void *a2, size t size)
int strcmp (const char *s1, const char *s2)
int strcasecmp (const char *s1, const char *s2) //忽略大小写
int strncmp (const char *s1, const char *s2, size t size)
void * memchr (const void *block, int c, size t size)
char * strchr (const char *string, int c)
char * strrchr (const char *string, int c)
char * strstr (const char *haystack, const char *needle)
char * strtok (char *restrict newstring, const char *restrict delimiters)
char * basename (const char *filename)
char * dirname (char *path)
```

# 搜索与排序
libc提供最基础的排序的支持, 比如快排. 下次有人找你写个快排还需要自己写么?
```c
#include <stdlib.h>
```
## 比较函数原型: int comparison_fn_t (const void *, const void *);

例子:
```c
int
compare_doubles (const void *a, const void *b)
{
    const double *da = (const double *) a;
    const double *db = (const double *) b;
    return (*da > *db) - (*da < *db);
}
```

## 数组
```c
#include <search.h>
/* 在base数组内按照key搜索 */
void * lfind (const void *key, const void *base, size t *nmemb, size t size, comparison fn t compar)
/* 在排好序的数组内搜索 */
void * bsearch (const void *key, const void *array, size t count, size t size, comparison fn t compare)
 
#include <stdlib.h>
/* 快排 */
void qsort (void *array, size t count, size t size, comparison fn t compare)
```

## 哈希
```c
int hcreate (size t nel)
```

## 树
```c
/*找到则返回节点指针, 否则就将key加入到tree中*/
void * tsearch (const void *key, void **rootp, comparison_fn_t compar)
/*只查找不添加*/
void * tfind (const void *key, void *const *rootp, comparison fn t compar)
/*删除节点*/
void * tdelete (const void *key, void **rootp, comparison fn t compar)
/*销毁*/
void tdestroy (void *vroot, free fn t freefct)
/*遍历void __action_fn_t (const void *nodep, VISIT value, int level);*/
void twalk (const void *root, action fn t action)
```

# 正则
```c
#include <regex.h>
//在进行正则匹配之前, 要求先编译pattern, 到regex_t结构体中
int regcomp (regex t *restrict compiled, const char *restrict pattern, int cflags)
//匹配
int regexec (const regex t *restrict compiled, const char *restrict string, size t nmatch, regmatch t matchptr[restrict], int eflags)
//单词解析
int wordexp (const char * words, wordexp_t * word-vector-ptr, int flags)
void wordfree(wordexp_t *word-vector-ptr)
```
举例
```c
int
expand_and_execute(const char *program, const char **options)
{
    wordexp_t result;
    pid_t pid
    int status, i;
  
    /*
    Expand the string for the program to run.
    */
    switch (wordexp(program, &result, 0)) {
    case 0:  /*Successful*/
        break;
  
    case WRDE_NOSPACE:
        /*
        If the error was
        WRDE_NOSPACE
        ,
        then perhaps part of the result was allocated.
        */
        wordfree(&result);
  
    default:                    /*Some other error.*/
        return -1;
    }
  
    /*
    Expand the strings specified for the arguments.
    */
    for (i = 0; options[i] != NULL; i++) {
        if (wordexp(options[i], &result, WRDE_APPEND)) {
            wordfree(&result);
            return -1;
        }
    }
  
    pid = fork();
  
    if (pid == 0) {
        /*
        This is the child process.  Execute the command.
        */
        execv(result.we_wordv[0], result.we_wordv);
        exit(EXIT_FAILURE);
    } else if (pid < 0)
        /*
        The fork failed.  Report failure.
        */
        status = -1;
    else
        /*
        This is the parent process.  Wait for the child to complete.
        */
        if (waitpid(pid, &status, 0) != pid)
            status = -1;
  
    wordfree(&result);
    return status;
}
```

# 输入输出之流文件
流文件, 用FILE *代表, 默认已经打开的三个 FILE * stdin, FILE * stdout, FILE * stderr
## 重定向
```c
fclose (stdout);
stdout = fopen ("standard-output-file", "w");
```

可能有些系统stdout是个宏, 那可以用freopen(), 这个函数基本上是fclose和fopen的合体

## 流文件操作
```c
#include <stdio.h>
FILE *fopen(const char *path, const char *mode);
FILE *fdopen(int fd, const char *mode);
int fclose(FILE *fp);
```
带64后缀的fopen能够打开超过4G的文件

## 多线程与文件锁
每个文件都有个内部锁, 保证每次文件操作函数调用的原子性, 这个锁是隐含的. 所以多线程操作一个文件会隐含的顺序执行.
但有时这样还不够, 那么可以显示使用文件锁
注意: 文件锁加锁的对象是stream, 用于多线程之间的互斥.
如果要在多进程之间用文件互斥, 需使用lockf()调用. 详见代码片段==文件锁
```c
#include <stdio.h>
void flockfile(FILE *filehandle); //会导致进程阻塞
int ftrylockfile(FILE *filehandle);
void funlockfile(FILE *filehandle);
```
举例: 这里的flockfile()保护了下面的两句操作是原子的.
```c
FILE *fp;
{
    ...
    flockfile (fp);
    fputs ("This is test number ", fp);
    fprintf (fp, "%d\n", test);
    funlockfile (fp)
}
```

## 关于宽字符
libc支持在一个stream上允许宽字符和普通字符输出, 但两者只能选其一, 而且一旦选定就不能再更换.
宽字符操作函数和普通字符函数是对应的, 函数名有w的就是宽字符操作.
如果使用了宽字符操作函数, 这个stream就是宽字符模式, 否则就是普通字符模式.
```c
int fwide (FILE *stream, int mode) //这个函数可以设置模式, 要求是尽量早的调用
void
print_f (FILE *fp)
{
    if (fwide (fp, 0) > 0)
        /* Positive return value means wide orientation. */
        fputwc (L’f’, fp);
    else
        fputc (’f’, fp);
}
```

## stream写
```c
int fputc (int c, FILE *stream)
wint_t fputwc (wchar t wc, FILE *stream)
int fputs (const char *s, FILE *stream)
int fputws (const wchar t *ws, FILE *stream)
int puts (const char *s) //输出到stdout
```

## stream读
```c
int fgetc (FILE *stream)
wint_t fgetwc (FILE *stream)
/*以下是行模式*/
ssize_t getline (char **lineptr, size t *n, FILE *stream) //如果*lineptr是NULL而且n为0, 这个函数会自己malloc
ssize_t getdelim (char **lineptr, size t *n, int delimiter, FILE *stream) //可以指定分隔符
ssize_t getline (char **lineptr, size_t *n, FILE *stream)
{
    return getdelim (lineptr, n, ’\n’, stream);
}
char * fgets (char *s, int count, FILE *stream)
wchar_t * fgetws (wchar t *ws, int count, FILE *stream)
```

## unread和peeking ahead
```c
int ungetc (int c, FILE *stream) //把c放回stream, 只能放一次.
```
举例:
```c
#include <stdio.h>
#include <ctype.h>
void
skip_whitespace (FILE *stream)
{
    int c;
    do
        /* No need to check for EOF because it is not
        isspace, and ungetc ignores EOF. */
        c = getc (stream);
    while (isspace (c));
    ungetc (c, stream);
}
```

## block读写
```c
size_t fread (void *data, size t size, size t count, FILE *stream)
size_t fwrite (const void *data, size t size, size t count, FILE *stream)
```

## 格式化打印
```c
#include <stdio.h>
int printf (const char *template, . . . )
int wprintf (const wchar t *template, . . . )
int fprintf (FILE *stream, const char *template, . . . )
int fwprintf (FILE *stream, const wchar t *template, . . . )
int sprintf (char *s, const char *template, . . . )
int swprintf (wchar t *s, size t size, const wchar t *template, . . . )
/*动态分配内存*/
int asprintf (char **ptr, const char *template, . . . )
```
举例:
```c
/* Construct a message describing the value of a variable
whose name is name and whose value is value. */
char *
make_message (char *name, char *value)
{
    char *result;
    if (asprintf (&result, "value of %s is %s", name, value) < 0)
        return NULL;
    return result;
}

int vprintf (const char *template, va list ap)
```
举例:
```c
#include <stdio.h>
#include <stdarg.h>
void
eprintf (const char *template, ...)
{
    va_list ap;
    extern char *program_invocation_short_name;
    fprintf (stderr, "%s: ", program_invocation_short_name);
    va_start (ap, template);
    vfprintf (stderr, template, ap);
    va_end (ap);
}
```

## 格式化输入
```c
int scanf (const char *template, . . . )
int wscanf (const wchar t *template, . . . )
int fscanf (FILE *stream, const char *template, . . . )
int sscanf (const char *s, const char *template, . . . )
```

## 检查文件尾, 常用于判断
```c
int feof (FILE *stream)
int ferror (FILE *stream)
```

## 文件位置
```c
long int ftell (FILE *stream)
off_t ftello (FILE *stream)
off64_t ftello64 (FILE *stream)
int fseek (FILE *stream, long int offset, int whence) //SEEK_SET, SEEK_CUR, or SEEK_END
int fseeko (FILE *stream, off t offset, int whence)
//用以下一组函数处理位置更好
int fgetpos (FILE *stream, fpos t *position)
int fsetpos (FILE *stream, const fpos t *position)
```

## 文件流的buffering
文件流都有buffer, 而更底层的io读写没有.
buffer有三种模式:
*  unbuffer
*  line buffer
*  full buffer
一般的文件都是full buffer, 但和交互相关的设备比如terminal就是line buffer
刷buffer的函数:
```c
int fflush (FILE *stream) //如果入参是NULL, 则会flush所有打开的文件
```
改变buffer模式:
```c
#include <stdio.h>
int setvbuf (FILE *stream, char *buf, int mode, size t size) //_IOFBF _IOLBF _IONBF
```

# 底层输入输出, 与之对应的是文件描述符
```c
#include <unistd.h>
#include <fcntl.h>
int open (const char *filename, int flags[, mode t mode])
比如: open (filename, O_WRONLY | O_CREAT | O_TRUNC, mode)
int open64 (const char *filename, int flags[, mode t mode])
int close (int filedes)
ssize_t read (int filedes, void *buffer, size t size) //EAGAIN EINTR EIO
ssize_t pread (int filedes, void *buffer, size t size, off t offset) //从文件开始的offset开始读, 不会影响文件位置
ssize_t write (int filedes, const void *buffer, size t size)
ssize_t pwrite (int filedes, const void *buffer, size t size, off t offset) //类似pread
off_t lseek (int filedes, off t offset, int whence) //SEEK_SET, SEEK_CUR, or SEEK_END
off64_t lseek64 (int filedes, off64 t offset, int whence)
```
举例:
```c
//同一个文件, open两次, 每个描述符有独立的位置
{
    int d1, d2;
    char buf[4];
    d1 = open ("foo", O_RDONLY);
    d2 = open ("foo", O_RDONLY);
    lseek (d1, 1024, SEEK_SET);
    read (d2, buf, 4);
}
//但dup函数复制的描述符都共享文件位置
{
    int d1, d2, d3;
    char buf1[4], buf2[4];
    d1 = open ("foo", O_RDONLY);
    d2 = dup (d1);
    d3 = dup (d2);
    lseek (d3, 1024, SEEK_SET);
    read (d1, buf1, 4);
    read (d2, buf2, 4);
}
```

## 文件描述符和文件流互转
```c
FILE * fdopen (int filedes, const char *opentype)
int fileno (FILE *stream)
```

## mmap io
```c
#include <sys/mman.h>
size_t page_size = (size_t) sysconf (_SC_PAGESIZE);
void * mmap (void *address, size t length, int protect, int flags, int filedes, off t offset)
protect: PROT_READ, PROT_WRITE, and PROT_EXEC
flags:
MAP_PRIVATE: 对内存region的写不会写回到文件
MAP_SHARED: 对内存region的写会写到文件, 但真正写文件的操作发生在any time. 用msync可以强制写
MAP_ANONYMOUS: 匿名mmap, 不需要文件, 映射的内存被初始化为0; 可用来替代malloc, 但GNU的malloc已包含mmap功能.
void * mmap64 (void *address, size t length, int protect, int flags, int filedes, off64 t offset)
int munmap (void *addr, size t length)
int msync (void *address, size t length, int flags)
void * mremap (void *address, size t length, size t new_length, int flag)
```

## 共享内存和mmap
```c
//用这个函数可以获得一个文件描述符, 用于mmap. 不同进程之间可以访问同名的共享内存
int shm_open (const char *name, int oflag, mode t mode)
int shm_unlink (const char *name)
```

## select
```c
#include <sys/types.h>
void FD_ZERO (fd set *set)
void FD_SET (int filedes, fd set *set)
void FD_CLR (int filedes, fd set *set)
int FD_ISSET (int filedes, const fd set *set)
//select
int select (int nfds, fd set *read-fds, fd set *write-fds, fd set *except-fds, struct timeval *timeout)
```
举例:
```c
#include <errno.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/time.h>
int
input_timeout(int filedes, unsigned int seconds)
{
    fd_set set;
    struct timeval timeout;
    /* Initialize the file descriptor set. */
    FD_ZERO(&set);
    FD_SET(filedes, &set);
    /* Initialize the timeout data structure. */
    timeout.tv_sec = seconds;
    timeout.tv_usec = 0;
    /* select returns 0 if timeout, 1 if input available, -1 if error. */
    return TEMP_FAILURE_RETRY(select(FD_SETSIZE, &set, NULL, NULL, &timeout));
}
int
main(void)
{
    fprintf(stderr, "select returned %d.\n",
            input_timeout(STDIN_FILENO, 5));
    return 0;
}
```

## 同步io
即使write成功返回, 也并不意味着数据已经真正写进去了. 
```c
#include <unistd.h>
//这个函数保证所有的数据已经写入物理介质
void sync (void)
//对单文件
int fsync (int fildes)
```

## 并行io之aio, _POSIX_ASYNCHRONOUS_IO
```c
#include <aio.h>
//异步读写, 操作会被放进队列
int aio_read (struct aiocb *aiocbp)
int aio_write (struct aiocb *aiocbp)
//既然是异步操作, 就必然有个函数来检查状态
int aio_error (const struct aiocb *aiocbp)
//异步操作完成了, 检查返回值
ssize_t aio_return (struct aiocb *aiocbp)
//希望在某个点上保证同步
int aio_fsync (int op, struct aiocb *aiocbp)
//取消aio操作
int aio_cancel (int fildes, struct aiocb *aiocbp)
```

## fcntl 
```c
#include <fcntl.h>
int fcntl (int filedes, int command, . . . )
F_DUPFD F_GETFD F_SETFD F_GETFL F_SETFL F_GETLK F_SETLK  F_SETOWN
fcntl (filedes, F_SETFD, new-flags)
fcntl (filedes, F_SETOWN, pid)
```
F_GETFL F_SETFL包括:
*  O_RDONLY O_WRONLY O_RDWR //访问权限
*  O_NONBLOCK //不阻塞
*  O_APPEND //在文件尾添加
*  O_ASYNC //异步, 用SIGIO信号表示操作完成?
*  O_FSYNC

举例:
```c
/* Set the O_NONBLOCK flag of desc if value is nonzero,
or clear the flag if value is 0.
Return 0 on success, or -1 on error with errno set. */
int
set_nonblock_flag(int desc, int value)
{
    int oldflags = fcntl(desc, F_GETFL, 0);
 
    /* If reading the flags failed, return error indication now. */
    if (oldflags == -1)
        return -1;
 
    /* Set just the flag we want to set. */
    if (value != 0)
        oldflags |= O_NONBLOCK;
    else
        oldflags &= ~O_NONBLOCK;
 
    /* Store modified flag word in the descriptor. */
    return fcntl(desc, F_SETFL, oldflags);
}
```

## 文件锁操作
```c
fcntl (filedes, F_SETLK, lockp)
```
与lockf()等同
需要注意的是, 文件锁是个主动锁, 显示的调用获取锁才能保证互斥.
## io重定向
```c
int dup (int old)
//先close new, 再dup old
int dup2 (int old, int new)
```
举例:
```c
pid = fork();
 
if (pid == 0)
{
    char *filename;
    char *program;
    int file;
    ...
    file = TEMP_FAILURE_RETRY(open(filename, O_RDONLY));
    dup2(file, STDIN_FILENO);
    TEMP_FAILURE_RETRY(close(file));
    execv(program, NULL);
}
```

## ioctl
```c
#include <sys/ioctl.h>
int ioctl (int filedes, int command, . . . )
```

# 文件系统接口
```c
#include <dirent.h>
struct dirent
    char d_name[]
    ino_t d_fileno
    unsigned char d_namlen
    unsigned char d_type
```

## 目录项
```c
char * getcwd (char *buffer, size t size)
int chdir (const char *filename)
int fchdir (int filedes)
DIR * opendir (const char *dirname)
DIR * fdopendir (int fd)
int dirfd (DIR *dirstream)
struct dirent * readdir (DIR *dirstream)
int closedir (DIR *dirstream)
//重新init这个dirstream
void rewinddir (DIR *dirstream)
```
举例:
```c
#include <stdio.h>
#include <sys/types.h>
#include <dirent.h>
int
main(void)
{
    DIR *dp;
    struct dirent *ep;
    dp = opendir("./");
 
    if (dp != NULL) {
        while (ep = readdir(dp))
            puts(ep->d_name);
 
        (void) closedir(dp);
    } else
        perror("Couldn’t open the directory");
 
    return 0;
}
```

## 遍历目录项
```c
int scandir (const char *dir, struct dirent ***namelist, int (*selector) (const struct dirent *), int (*cmp) (const  truct dirent **, const struct dirent **))
```

## 遍历目录树
```c
int ftw (const char *filename, ftw func t func, int descriptors)
```

## 文件链接
```c
int link (const char *oldname, const char *newname)
int symlink (const char *oldname, const char *newname)
ssize_t readlink (const char *filename, char *buffer, size t size)
```

## 化简文件路径, 去掉. ..和多余的/
```c
char * canonicalize_file_name (const char *name)
char * realpath (const char *restrict name, char *restrict resolved)
```

## 删除与重命名
```c
int unlink (const char *filename)
int rmdir (const char *filename)
int remove (const char *filename)
int rename (const char *oldname, const char *newname)
int mkdir (const char *filename, mode t mode)
```

## 文件统计
```c
#include <sys/stat.h>
int stat (const char *filename, struct stat *buf)
//不follow link
int lstat (const char *filename, struct stat *buf)
int fstat (int filedes, struct stat *buf)
```

## 文件类型判断, 入参是stat的st_mode域
```c
int S_ISDIR (mode t m)
int S_ISCHR (mode t m)
```

## 访问权限
```c
#include <sys/stat.h>
S_IRUSR S_IWUSR S_IXUSR
mode_t getumask (void)
int chmod (const char *filename, mode t mode)
int fchmod (int filedes, mode t mode)
```

## 特殊文件
```c
int mknod (const char *filename, mode t mode, dev t dev)
```

## 临时文件
```c
FILE * tmpfile (void)
//临时文件名
char * tmpnam (char *result)
char * mktemp (char *template)
```

# 管道
```c
//使用管道必须保证读写两端同时open了, 才能开始读写.
int pipe (int filedes[2]) //filedes[0]为读, filedes[1]为写
```
在使用pipe时, 搭配使用fork(创建子进程), dup2(令子进程的标准输入或输出重定向到pipe)
或者, 使用
```c
FILE * popen (const char *command, const char *mode) //创建子进程执行command, 并重定向到pipe, 方向取决于mode, "r", "w"
int pclose (FILE *stream)
```
举例:
```c
#include <stdio.h>
#include <stdlib.h>
void
write_data(FILE * stream)
{
    int i;
 
    for (i = 0; i < 100; i++)
        fprintf(stream, "%d\n", i);
 
    if (ferror(stream)) {
        fprintf(stderr, "Output to stream failed.\n");
        exit(EXIT_FAILURE);
    }
}
int
main(void)
{
    FILE *output;
    output = popen("more", "w");
 
    if (!output) {
        fprintf(stderr,
                "incorrect parameters or too many files.\n");
        return EXIT_FAILURE;
    }
 
    write_data(output);
 
    if (pclose(output) != 0) {
        fprintf(stderr,
                "Could not run more or other error.\n");
    }
 
    return EXIT_SUCCESS;
}
```

## 命名管道
```c
#include <sys/stat.h>
int mkfifo (const char *filename, mode t mode)
```
## 一般来说, 管道的读写是原子的. 除非size大于PIPE_BUF