- [Time, Delays, and Deferred Work](#time-delays-and-deferred-work)
  - [时间表达和获取](#时间表达和获取)
    - [jiffies](#jiffies)
    - [高精度时间](#高精度时间)
    - [获取时间](#获取时间)
  - [delay](#delay)
    - [busy wait](#busy-wait)
    - [schedule](#schedule)
    - [schedule_timeout](#schedule_timeout)
    - [带timeout的休眠 `wait_event_timeout`](#带timeout的休眠-wait_event_timeout)
    - [ms us ns](#ms-us-ns)
  - [内核定时器](#内核定时器)
    - [API](#api)
  - [Workqueues](#workqueues)
    - [API](#api-1)
    - [使用默认的waitqueue](#使用默认的waitqueue)

参考: https://www.oreilly.com/library/view/linux-device-drivers/0596005903/ch07.html

LDD第七章: 

# Time, Delays, and Deferred Work

## 时间表达和获取
### jiffies
```c
#include <linux/jiffies.h>
unsigned long j, stamp_1, stamp_half, stamp_n;

j = jiffies; /* read the current value */
stamp_1 = j + HZ; /* 1 second in the future */
stamp_half = j + HZ/2; /* half a second */
stamp_n = j + n * HZ / 1000; /* n milliseconds */

//time的比较
int time_after(unsigned long a, unsigned long b);
int time_before(unsigned long a, unsigned long b);
int time_after_eq(unsigned long a, unsigned long b);
int time_before_eq(unsigned long a, unsigned long b);

//time的转换
#include <linux/time.h>

unsigned long timespec_to_jiffies(struct timespec *value);
void jiffies_to_timespec(unsigned long jiffies, struct timespec *value);
unsigned long timeval_to_jiffies(struct timeval *value);
void jiffies_to_timeval(unsigned long jiffies, struct timeval *value);

//64位的jiffies
#include <linux/jiffies.h>
u64 get_jiffies_64(void);
```

### 高精度时间
```c
//X86
unsigned long ini, end;
rdtscl(ini); rdtscl(end);
printk("time lapse: %li\n", end - ini);


//所有架构都有的
#include <linux/timex.h>
cycles_t get_cycles(void);
```

### 获取时间
```c
#include <linux/time.h>
void do_gettimeofday(struct timeval *tv);

#include <linux/time.h>
struct timespec current_kernel_time(void);
```


## delay
### busy wait
busy wait最费CPU
```
while (time_before(jiffies, j1))
    cpu_relax( );
```

### schedule
`schedule()`看起来很好, 但依旧比较费CPU. 因为这个进程还在run queue中, 如果没有其他可运行的任务, 就会反复不断的调度到本任务运行.  
不推荐
```c
while (time_before(jiffies, j1)) {
    schedule( );
}
```

### schedule_timeout
这个api不需要自己填wait_queue, 更简单. 
在timeout之前睡眠
```c
#include <linux/sched.h>

set_current_state(TASK_INTERRUPTIBLE);
schedule_timeout (delay);
```

### 带timeout的休眠 `wait_event_timeout`
```c
#include <linux/wait.h>
//在给定的wait queue休眠timeout个jiffies.
//进程被wake_up唤醒时, 检查condition, 为假就继续等待.
long wait_event_timeout(wait_queue_head_t q, condition, long timeout);
long wait_event_interruptible_timeout(wait_queue_head_t q,
                      condition, long timeout);
```

### ms us ns 
```c
#include <linux/delay.h>
void ndelay(unsigned long nsecs);
void udelay(unsigned long usecs);
void mdelay(unsigned long msecs);
```

## 内核定时器
内核定时器run用户定义的函数, 是异步的方式, 不在注册timer的进程上下文.
timer运行在软中断上下文, 限制挺多的:
* 不能访问用户态, 因为没有进程上下文
* current指针没意义. 这里的current指针是被中断打断的current, 对timer来说没用
* 不能休眠. 比如不能调用schedule等函数, 不能kmalloc, 不能调用`wait_`系列的函数, 不能用互斥量等
* 调用`in_interrupt()`函数会返回非零值, 因为timer处于软中断中.
* timer运行的核于注册的核永远是同一个. 因为用了per-cpu
* 本质上是异步执行, 必须考虑资源保护. 可以用spinlock

### API
```c
#include <linux/timer.h>
struct timer_list {
        /* ... */
        unsigned long expires;
        void (*function)(unsigned long);
        unsigned long data;
};


void init_timer(struct timer_list *timer);
struct timer_list TIMER_INITIALIZER(_function, _expires, _data);


void add_timer(struct timer_list * timer);
int del_timer(struct timer_list * timer);


int mod_timer(struct timer_list *timer, unsigned long expires);
int del_timer_sync(struct timer_list *timer);
int timer_pending(const struct timer_list * timer);
```

## Workqueues
workqueue工作在特殊的内核线程

### API
```c
struct workqueue_struct *create_workqueue(const char *name);
struct workqueue_struct *create_singlethread_workqueue(const char *name);
DECLARE_WORK(name, void (*function)(void *), void *data);
INIT_WORK(struct work_struct *work, void (*function)(void *), void *data);
PREPARE_WORK(struct work_struct *work, void (*function)(void *), void *data);
int queue_work(struct workqueue_struct *queue, struct work_struct *work);
int queue_delayed_work(struct workqueue_struct *queue, 
                       struct work_struct *work, unsigned long delay);
int cancel_delayed_work(struct work_struct *work);
void flush_workqueue(struct workqueue_struct *queue);
void destroy_workqueue(struct workqueue_struct *queue);
```

### 使用默认的waitqueue
```c
int schedule_work(struct work_struct *work);
int schedule_delayed_work(struct work_struct *work, unsigned long delay); 
```
