
# kernel链表
linux通过/linux/list.h文件，提供了标准的链表操作接口。内部是用双向链表实现的，并且体现了一个重要的思想，就是把链表操作和数据成员分离，这样一来，对这个双向链表本身的操作就可以抽象出来，不会因为数据成员的不同而变化。在软件设计中，分离变与不变，"求同存异"，是非常重要的思想。

下面以一个hash表为例，简介其用法。

在linux启动代码中，会对锁的依赖管理模块进行初始化，在kernel/lockdep.c中

## 定义全局hash表
```
static struct list_head classhash_table[4k];
```
`classhash_table`是个4k大小的数组，每个元素都是一个链表，是个典型的拉链hash结构。  
这个链表以list_head为节点，并不包含其他的数据成员。
```c
struct list_head {
    struct list_head *next, *prev;
};
```

## 对classhash_table进行初始化
```c
for (i = 0; i < CLASSHASH_SIZE; i++)
        INIT_LIST_HEAD(classhash_table + i);
```
`INIT_LIST_HEAD()`是`list.h`提供的内联函数，用于在运行时初始化一个链表，使链表头尾指向自己。

另外，list.h还提供了一个宏，用于定义并初始化一个链表：`LIST_HEAD(name)`

## 数据成员
后面会看到，实际上每个链表节点并不仅是一个list_head结构体，而是一个包含list_head的带其他数据成员的结构：
```c
struct lock_class {
    struct list_head        hash_entry;
    struct list_head        lock_entry;
    struct lockdep_subclass_key    *key;
    unsigned int            subclass;
    unsigned int            dep_gen_id;
    unsigned long           usage_mask;
    struct stack_trace      usage_traces[LOCK_USAGE_STATES];
    struct list_head        locks_after, locks_before;
    unsigned int            version;
    unsigned long           ops;
    const char              *name;
    int                     name_version;
};
```
可以看到，hash_entry就是被包含的list_head结构，它的作用是代表整个lock_class结构体，被链接到链表中。list_head可以在结构体的任意位置，用`container_of(ptr, type, member)`宏就可以找到母体的位置。
```c
#define container_of(ptr, type, member) ({            \
    const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
    (type *)( (char *)__mptr - offsetof(type,member) );})
```

## hash桶的拉链
在用hash函数算出key的hash值之后，就得到了一个在hash桶里的一个链表hash_head
```c
hash_head = classhash_table + hash_long(key)
```

## 遍历链表
最后用`list_for_each_entry(pos, head, member)`宏来遍历这个链表，其中，pos是个迭代变量，指向节点实体类型，在这里是lock_class；head是要遍历的链表头，在这里是hash_head；member是list_head在lock_class里的名称，在这里是hash_entry
```c
struct lock_class *class;

list_for_each_entry(class, hash_head, hash_entry) {
        if (class->key == key) {
            WARN_ON_ONCE(class->name != lock->name);
            return class;
        }
    }
```

# 打印table
```c
printf("%-25s%-20s%-10s%-10s%-10s\n", "Name", "Title", "Gross", "Tax", "Net");
printf("%-25s%-20s%-10.2f%-10.2f%-10.2f\n", name.c_str(), title.c_str(), gross, tax, net);
```

# vsystem
```c
int vsystem(char** command)
{
  int sysStatus = -1;
  pid_t pid = -1, waitResult = -1;
 
  pid = vfork();
  if (pid == 0)
  {
    /* This is the child process.  Execute the shell command. */
    if (execvp(command[0], command) < 0)    /* execvp does not return if successful */
      _exit(-1);
  }
  else if (pid < 0)
  {
    int the_errno = errno;
    err_printf(ERR_COM_UNEXP_COND, ERR_CLASS_RECOV, "Swm",
                 __FILE__, __LINE__, "vfork failed ; errno is %d\n\r", the_errno);
  }
  else
  {
    /* This is the parent process.  Wait for the child to complete. */
    while (1)
    {  
      waitResult = waitpid(pid, &sysStatus, 0);
      if (waitResult != pid)
      {
        if (errno == EINTR)
          continue;
      }
      else
      {
        if (WIFEXITED(sysStatus) != 0)     /* child process completed normally */
          sysStatus = WEXITSTATUS(sysStatus);
      }
      break;
    }
  }
 
  return sysStatus;
}
```

# C遍历目录
```c
/* print files in current directory in reverse order */
#include <dirent.h>
main(){
    struct dirent **namelist;
    int n;
 
    n = scandir(".", &namelist, 0, alphasort);
    if (n < 0)
        perror("scandir");
    else {
        while(n--) {
            printf("%s\n", namelist[n]->d_name);
            free(namelist[n]);
        }
        free(namelist);
    }
}
```

# pipe产生2个fd, 0为读, 1为写
fcntl可以对fd进行若干操作, 比如F_GETFL F_SETFL 
```c
static int updater_pipe_fd[2] = {-1,-1}; /* File descriptors for pipe to update the readset */
static void init_readset_updater(void)
{
  int flags;
 
  if (pipe(updater_pipe_fd) < 0) {
    uiodrv_printf(get_uiodrv_trc_error(), "%s: pipe call failed (errno=%d)!\n\r", __FUNCTION__, errno);
    return;
  }
 
  /* Make read and write ends of pipe nonblocking */
  flags = fcntl(updater_pipe_fd[READER_INDEX], F_GETFL);
  flags |= O_NONBLOCK;                  /* Make read end nonblocking */
  fcntl(updater_pipe_fd[READER_INDEX], F_SETFL, flags);
 
  flags = fcntl(updater_pipe_fd[WRITER_INDEX], F_GETFL);
  flags |= O_NONBLOCK;                  /* Make write end nonblocking */
  fcntl(updater_pipe_fd[WRITER_INDEX], F_SETFL, flags);
 
  FD_SET(updater_pipe_fd[READER_INDEX], &readset);  /* Add read end of pipe to 'readset' */
}
```
在上面的函数中, 创建了一个pipe, 共生成两个文件描述符, updater_pipe_fd[0],updater_pipe_fd[1]. 并在最后, 把updater_pipe_fd[0]放到readset中, 另外一个进程将会不断的select这个readset. 所以, 只要有人用updater_pipe_fd[1]写入(可能是另外一个不同的进程), 则select将会返回.

下面这个函数, 就演示了在select任务运行时, 通过在其他任务写updater_pipe_fd[1]任意一个字符(我们并不关心这个字符是什么意思), 来触发select任务中select返回;

那么为什么一定要触发select返回呢? 因为下面的函数想添加/删除一个fd到readSet, 但此时readSet虽然更新了, 但select任务可能还在阻塞, 这样就不能及时take这个新的readSet的变更.

```c
int uiodrv_set_intr_enable_state(DeviceNbr dev, IntrEnableState state)
{
  DeviceInfo *inst = findInstance(dev);
  if ((inst == NULL) || (inst->fd == -1))
    return -1;
 
  switch (state) {
    case E_uiodrv_disable_intr:
      FD_CLR(inst->fd, &readset);
      break;
    case E_uiodrv_enable_intr:
      FD_SET(inst->fd, &readset);
      break;
    default:
      return -1;
  }
 
  /* update readset by writing to selfpipe. This will trigger the processing of all the devices */
  if (write(updater_pipe_fd[WRITER_INDEX], "x", 1) < 0)
    return -1;
 
  return 0;
}
```

# 从内存地址拷贝成文件
可用dd命令代替,`dd if=/dev/mem of=some_file bs=1M skip=1984 count=5`
```c
/*
 * File:   copyFromMem.c
 * Author: lcarlier
 *
 * Created on March 18, 2014, 11:33 AM
 */
 
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <limits.h>
#include <stdlib.h>
#include <string.h>
 
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
 
#include <unistd.h>
 
#include <sys/mman.h>
 
long readNumFromStr(char *str,int base);
 
 
int main(int argc, char** argv) {
    if (argc != 4) {
        fprintf(stderr, "Wrong number of arguments. Usage:\n");
        fprintf(stderr, "\tcopyFromMem addrInMemory size outfile\n");
        return EXIT_FAILURE;
    }
 
    long int fromPhyMem;
    long int size;
    char *outFile;
 
    fromPhyMem = readNumFromStr(argv[1],16);
    size = readNumFromStr(argv[2],10);
    outFile = argv[3];
     
    if(fromPhyMem < 0){
        fprintf(stderr,"Negative address given. %ld\n",size);
        return (EXIT_FAILURE);
    }
     
    if(size < 0){
        fprintf(stderr,"Negative size given. %ld\n",size);
        return (EXIT_FAILURE);
    }
 
    int fdout;
    if((fdout = creat(outFile,S_IRWXU)) == -1){
        fprintf(stderr,"Error while creating file %s. Errno=%d (%s)\n",outFile,errno,strerror(errno));
        return EXIT_FAILURE;
    }
     
    int fddevmem;
    if((fddevmem = open("/dev/mem",O_RDONLY)) == -1){
        fprintf(stderr,"Error while opening /dev/mem. Errno=%d (%s)\n",errno,strerror(errno));
        return EXIT_FAILURE;
    }
     
    void *srcAddr;
    if((srcAddr = mmap(NULL,size,PROT_READ,MAP_PRIVATE,fddevmem,fromPhyMem)) == MAP_FAILED){
        fprintf(stderr,"Error while mmap. Errno=%d (%s)\n",errno,strerror(errno));
        return EXIT_FAILURE;
    }
     
    printf("Starting copying byte from address %#lx into %s\n",fromPhyMem,outFile);
     
    size_t nbByteCopied = -1;
    while((nbByteCopied = write(fdout,srcAddr,size))==-1){
        switch(errno){
            case EAGAIN: //or EWOULDBLOCK
                continue;
            default:
                fprintf(stderr,"Error while writing into file %s. Errno=%d (%s)\n",outFile,errno,strerror(errno));
                return EXIT_FAILURE;
        }
    }
     
    close(fdout);
     
 
    return (EXIT_SUCCESS);
}
 
long int readNumFromStr(char *str,int base){
    char *endptr;
    long int returnIntVal;
 
    errno = 0; /* To distinguish success/failure after call */
    returnIntVal = strtol(str, &endptr, base);
 
    /* Check for various possible errors */
 
    if ((errno == ERANGE && (returnIntVal == LONG_MAX || returnIntVal == LONG_MIN))
            || (errno != 0 && returnIntVal == 0)) {
        fprintf(stderr,"Error while reading %s: errno:%d (%s)\n",str,errno,strerror(errno));
        exit(EXIT_FAILURE);
    }
 
    if (endptr == str) {
        fprintf(stderr, "No digits were found in %s\n",str);
        exit(EXIT_FAILURE);
    }
    return returnIntVal;
}
```

# C处理文件
```c
static const char *PCI_DEVICE_FILENAME = "/proc/bus/pci/devices";
in = fopen(PCI_DEVICE_FILENAME, "r");
while (!feof(in))
    int count = fscanf(in, "%2x%2x %8x %x %Lx %Lx %Lx %Lx %Lx %Lx %Lx %Lx %Lx %Lx",
                       &bus, &devfn, &pci_id, &irq,
                       &bar0, &unused, &bar1, &unused, &unused, &unused, &unused,
                       &siz0, &unused, &siz1);
    fgets(rest_of_line, sizeof(rest_of_line), in)
fclose(in);
```

# 变长参数
查看帮助: `man stdarg`
```c
#include <stdarg.h>
void va_start(va_list ap, last);
type va_arg(va_list ap, type);
void va_end(va_list ap);
void va_copy(va_list dest, va_list src);
```

# 文件锁
```c
#include <unistd.h>
int lockf(int fd, int cmd, off_t len);
 
lock_file_name = "/tmp/octeon-remote-lock-xxxxxxxxx"
加锁:
lock_fd = open(lock_file_name, O_CREAT|O_WRONLY, 0666);
lockf(lock_fd, F_LOCK, 0)
 
解锁:
close(lock_fd);
```
原理:
互斥文件锁, 如果一个文件被锁住, 则调用者会阻塞(F_LOCK), 或者返回error(F_TLOCK)

# 带缩进的debug打印, 主要用于跟踪函数调用
比如使用时:
```c
static void default_unlock(void)
    OCTEON_REMOTE_DEBUG_CALLED();
    //函数体...
    OCTEON_REMOTE_DEBUG_RETURNED();
```

具体实现: 用全局变量debug_indent来实现缩进
```c
#define OCTEON_REMOTE_DEBUG_CALLED(format, ...) \
    octeon_remote_debug_call(0, "Called %s(" format ")\n", __FUNCTION__, ##__VA_ARGS__);
#define OCTEON_REMOTE_DEBUG_RETURNED(format, ...) \
    octeon_remote_debug_call(1, "Return %s(" format ")\n", __FUNCTION__, ##__VA_ARGS__);
 
void octeon_remote_debug_call(int is_return, const char *format, ...)
{
    const int level = 3;                         
    if (level <= (0xff & remote_funcs.debug))    
    { 
        va_list args;                            
        if (is_return)
            remote_funcs.debug_indent--;         
        va_start(args, format);                  
        octeon_remote_output_arg(level, 1, format, args); 
        va_end(args);
        if (!is_return)
            remote_funcs.debug_indent++;         
    } 
}
static void octeon_remote_output_arg(int level, int show_banner,
                                     const char *format, va_list ap)                   
{
    if (level <= (0xff & remote_funcs.debug))
    { 
        /* Setting bit 8 in the debug level disables color */
        int use_color = !(remote_funcs.debug & 0x100);    
        if (use_color)
        {
            int color = 1; /* Index 1 is the normal terminal color */
            if (level < 0)
                color = 0; /* Index 0 is the error color */       
            else if (level+1 < (int)(sizeof(DEBUG_LEVEL_COLORS)/sizeof(DEBUG_LEVEL_COLORS[0])))
                color = level+1;
            printf("\033[%dm", DEBUG_LEVEL_COLORS[color]);    
        }
        if (show_banner)
        {
            char buf[32];
            time_t t;
            struct tm lt;
            t = time(NULL);
            localtime_r(&t, &lt);
            strftime(buf, sizeof(buf), "%m-%d %H:%M:%S", &lt);
            printf("%s:%*s", buf, remote_funcs.debug_indent*2+1, "");
        }
        vprintf(format, ap);
        if (use_color)
            printf("\033[0m");
        fflush(stdout);
    }
}
```

# c下的函数重载, 用weak属性
```c
int env_init(void) __attribute__((__weak__));
```

# 寄存器变量
```c
#define DECLARE_GLOBAL_DATA_PTR        register volatile gd_t *gd asm ("k0")
```

# 字符串转int
```c
#include <stdlib.h>
int atoi(const char *nptr);
```

# getopt 参数处理函数
```c
#include <unistd.h>
int getopt(int argc, char * const argv[],
                  const char *optstring);
  
#include <getopt.h>
int getopt_long(int argc, char * const argv[],
                  const char *optstring,
                  const struct option *longopts, int *longindex);
```
例子:
```c
static struct option long_options[] =
{
    /* These options set a flag. */
    {"memonly", no_argument,       &memonly_flag, 1},
    {"scantwsi", no_argument,       &twsi_scan_flag, 1},
    {"dumpspd", no_argument,       &dump_spd_flag, 1},
    {"readlevel", no_argument,       &read_level_flag, 1},
    /* These options don't set a flag.
       We distinguish them by their indices. */
    {"board",   required_argument, 0, 0},
    {"board_type",required_argument, 0, 0},
    {"envfile",   required_argument, 0, 0},
    {"ddr0spd",   required_argument, 0, 0},
    {"ddr1spd",   required_argument, 0, 0},
    {"ddr2spd",   required_argument, 0, 0},
    {"ddr3spd",   required_argument, 0, 0},
    {"help", no_argument, 0, 'h'},
    {"version", no_argument, 0, 'v'},
    {"board_delay",   required_argument, 0, 0},
    {"ddr_ref_hz",   required_argument, 0, 0},
    {"ddr_clock_hz",   required_argument, 0, 0},
    {"cpu_ref_hz",   required_argument, 0, 0},
    {0, 0, 0, 0}
};
/* getopt_long stores the option index here. */
int option_index = 0;

c = getopt_long (argc, (char * const *)argv, "hv",
                 long_options, &option_index);
```

# getenv() 从环境变量获取值
```c
#include <stdlib.h>
char *getenv(const char *name);
  
例子:
char *remote_spec = getenv("OCTEON_REMOTE_PROTOCOL");
```

# feof() 判断文件结尾
```c
#include <stdio.h>
       void clearerr(FILE *stream);
       int feof(FILE *stream);
       int ferror(FILE *stream);
       int fileno(FILE *stream);
```
例子:
```c
int copy_file_to_octeon_dram(uint64_t address, const char *filename)
{
    char buffer[65536] = {0};
    uint64_t bytes_copied = 0;
  
    FILE *infile = fopen(filename, "r");
    if (infile == NULL)
    {
        perror("Unable to open file");
        return -1;
    }
  
    while (!feof(infile))
    {
        int count = fread(buffer, 1, sizeof(buffer), infile);
        if (count > 0)
        {
            octeon_remote_write_mem(address, buffer, count);
            address += count;
            bytes_copied += count;
        }
    }
    fclose(infile);
    return bytes_copied;
}
```

# fwrite和fread
这两个函数时c库函数, 是对基础IO read和write的封装. 函数返回已经成功操作的单元个数. 如果返回值不足入参指定的个数, 只有两种可能:

* 到了文件eof
* 出现严重IO错误, 比如文件不存在等.

这两种可能都不能用retry-loop来循环, 但可以用`feof()`和`ferror()`来判断状态, 如果出现严重IO错误, 程序退出即可, 不必重试, 因为一般重试也没用.

# /dev/mem和mmap
```c
int file_handle = open(filename, O_RDWR);
result = mmap64(NULL, alength, PROT_READ|PROT_WRITE, MAP_SHARED, file_handle, physical_address & (int64_t)pagemask);
int pagemask = ~(sysconf(_SC_PAGESIZE)-1);
uint64_t offset = physical_address - (physical_address & (int64_t)pagemask);
int alength = (length + offset + ~pagemask) & pagemask;
close(file_handle);
return result + offset;
```

# 转为字符串的宏
```c
/* Indirect stringification.  Doing two levels allows the parameter to be a
 * macro itself.  For example, compile with -DFOO=bar, __stringify(FOO)
* converts to "bar".
*/
  
#define __stringify_1(x...)    #x
#define __stringify(x...)    __stringify_1(x)
```

# 使用if else if else结构
处理顺序执行的几个步骤, 每步都可能有异常分支的情形. 把实际干的活放在if判断语句里面.  
比如下面这个例子, 正常情况下, 每个if后面的判断都执行, 最后进入最后的else分支.
```c
if (!dev->netdev_ops) {
    free_netdev(dev);
} else if (register_netdev(dev) < 0) {
    dev_err(&pdev->dev,
        "Failed to register ethernet device for interface %d, port %d\n",
        interface, priv->ipd_port);
    free_netdev(dev);
} else {
    list_add_tail(&priv->list, &cvm_oct_list);
    if (cvm_oct_by_pkind[priv->ipd_pkind] == NULL)
        cvm_oct_by_pkind[priv->ipd_pkind] = priv;
    else
        cvm_oct_by_pkind[priv->ipd_pkind] = (void *)-1L;
 
    cvm_oct_add_ipd_port(priv);
    /* Each transmit queue will need its
     * own MAX_OUT_QUEUE_DEPTH worth of
     * WQE to track the transmit skbs.
     */
    cvm_oct_mem_fill_fpa(wqe_pool,
                 PER_DEVICE_EXTRA_WQE);
    num_devices_extra_wqe++;
    queue_delayed_work(cvm_oct_poll_queue,
               &priv->port_periodic_work, 5*HZ);
}
```

# uboot中fdt库函数使用
```c
bootbus_node_offset = fdt_path_offset(gd->fdt_blob, "/soc/bootbus")
node_offset = fdt_next_node(gd->fdt_blob, node_offset, &depth);
name = fdt_get_name(gd->fdt_blob, node_offset, NULL);
fdt_node_check_compatible(gd->fdt_blob, node_offset, "cfi-flash")
nodep = fdt_getprop(fdt_addr, node_offset, "reg", (int *)&len);
p = fdt_get_alias(working_fdt, name);
fdt_add_subnode(working_fdt, 0, "memory");
```

# 如何判断大小端
关键是头文件`endian.h`
这里面大小端的宏定义`__*_ENDIAN`
```c
#include <stdio.h>
#include <endian.h>

int main()
{
#ifdef __LITTLE_ENDIAN
	printf("little\n");
#else
	printf("big\n");
#endif
}
```

# 标准的int类型
在C库里面提供
```c
#include <stdint.h>

int8_t uint64_t 类似的
```