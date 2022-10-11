- [关于pthread条件变量](#关于pthread条件变量)
  - [错误的例子](#错误的例子)
  - [改进, 用mutex保护predicate](#改进-用mutex保护predicate)
  - [用condition变量](#用condition变量)
    - [此时有两种写法来signal这个条件变量, 我看第一个写法更好.](#此时有两种写法来signal这个条件变量-我看第一个写法更好)
  - [总结](#总结)

# 关于pthread条件变量
在`pthread_cond_wait(pthread_cond_t *restrict cond,pthread_mutex_t *restrict mutex)`中, 第二个参数时个mutex锁, 它是干嘛的? 为什么条件变量还要个锁呢?

首先看一下pthread_cond_wait手册的说明:
* 这个函数自动释放muxte锁, 并导致调用线程在cond变量上阻塞; 成功返回的时候, mutex锁已经被获取, 即调用线程拥有该锁.
* 这个函数返回不一定是条件满足了, 也可能是被信号打断(信号也不是一定会导致这个线程wakeup, 具体要看系统调度). 所以出来以后, 还要检查条件(predicate)
* 注意这里说的条件(predicate)和条件变量(pthread_cond_t *restrict cond)是两个东西
* predicate是个判断条件

详细的问答在这里:
https://stackoverflow.com/questions/14924469/does-pthread-cond-waitcond-t-mutex-unlock-and-then-lock-the-mutex

这里简单解读一下:

## 错误的例子
```c
// 这里的fSet就是predicate, 即程序的"业务"逻辑判断条件;
// 但这样判断不对, 因为看到(!fSet)为真, 到sleep()之间, fSet可以被其他线程修改, 导致可能到sleep时, 条件不成立了; 多sleep()一会
// 多睡一会也不要紧, 但sleep()过程中, fSet可能又有变化, 但没办法被当前线程感知到.
bool fSet = false;

int WaitForTrue()
{
    while (!fSet)
    {
        sleep(n);
    }
}
```

## 改进, 用mutex保护predicate
```c
// 用mutex保护fSet, 当拥有mutex锁的时候, 进程可以放心的查看, 并睡觉了. 不会有人再改fSet
// 但问题是, 除了当前线程, 没人能够改fSet, 导致一直退不出循环.
pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;
bool fSet = false;

int WaitForTrue()
{
    pthread_mutex_lock(&mtx);
    while (!fSet)
        sleep(n);
    pthread_mutex_unlock(&mtx);
}

// 下面的版本多加了一个加锁, 解锁对, 这下别人可以改predicate了.
pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;
bool fSet = false;

int WaitForTrue()
{
    pthread_mutex_lock(&mtx);
    while (!fSet)
    {
        pthread_mutex_unlock(&mtx);
        // XXXXX
        sleep(n);
        // YYYYY
        pthread_mutex_lock(&mtx);
    }
    pthread_mutex_unlock(&mtx);
}

```

## 用condition变量
上面版本依然存在sleep期间, 对predicate不能感知的问题, 这时候就要用到条件变量了:

```c
int WaitForPredicate()
{
    // lock mutex (means:lock access to the predicate)
    pthread_mutex_lock(&mtx);

    // we can safely check this, since no one else should be 
    // changing it unless they have the mutex, which they don't
    // because we just locked it.
    while (!predicate)
    {
        // predicate not met, so begin waiting for notification
        // it has been changed *and* release access to change it
        // to anyone wanting to by unlatching the mutex, doing
        // both (start waiting and unlatching) atomically
        pthread_cond_wait(&cv,&mtx);

        // upon arriving here, the above returns with the mutex
        // latched (we own it). The predicate *may* be true, and
        // we'll be looping around to see if it is, but we can
        // safely do so because we own the mutex coming out of
        // the cv-wait call. 
    }

    // we still own the mutex here. further, we have assessed the 
    // predicate is true (thus how we broke the loop).

    // take whatever action needed. 

    // You *must* release the mutex before we leave. Remember, we
    // still own it even after the code above.
    pthread_mutex_unlock(&mtx);
}
```

### 此时有两种写法来signal这个条件变量, 我看第一个写法更好.
* mutex只保护predicate
```c
pthread_mutex_lock(&mtx);
TODO: change predicate state here as needed.
pthread_mutex_unlock(&mtx);
pthread_cond_signal(&cv);
```
* mutex也包含cond_signal
```c
pthread_mutex_lock(&mtx);
TODO: change predicate state here as needed.
pthread_cond_signal(&cv);
pthread_mutex_unlock(&mtx);
```

## 总结
> Never change, nor check, the predicate condition unless the mutex is locked. Ever.


