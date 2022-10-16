- [lua脚本的event如何调用](#lua脚本的event如何调用)
- [关于时间](#关于时间)
- [tx-rate如何实现](#tx-rate如何实现)

# lua脚本的event如何调用
C中:
```c
sysbench:
void *runner_thread(void *arg)
    do
        execute_request(test, &request, thread_id)
            test->ops.execute_request(r, thread_id)
                sb_lua_op_execute_request(sb_request_t *sb_req, int thread_id)
                    lua_getglobal(L, "event") //见下面
                    lua_pcall(L, 1, 1, 0)
    while ((request.type != SB_REQ_TYPE_NULL) && (!sb_globals.error) );
```
lua中
```lua
function event(thread_id)
   local table_name
   table_name = "sbtest".. sb_rand_uniform(1, oltp_tables_count)
   //这里边db_query和前面yuqing说的就连上了
   rs = db_query("SELECT c FROM ".. table_name .." WHERE id=" .. sb_rand(1, oltp_table_size))
end
```

# 关于时间
相对时间
```c
#ifdef HAVE_CLOCK_GETTIME     
# define SB_GETTIME(tsp) clock_gettime(CLOCK_REALTIME, tsp)
#else
用gettimeofday(&tv, NULL);
```
# tx-rate如何实现
```c
pthread_mutex_t event_queue_mutex;
static sb_list_t event_queue;
static pthread_cond_t event_queue_cv;
static event_queue_elem_t queue_array[MAX_QUEUE_LEN]; 100000


main
    run_test(test)
        //起report线程
        if (sb_globals.report_interval > 0)
            pthread_create(&report_thread, &thread_attr, &report_thread_proc,NULL)
        //起tx-rate线程, 这实际上是个节拍器
        if (sb_globals.tx_rate > 0)
            pthread_create(&eventgen_thread, &thread_attr, &eventgen_thread_proc,NULL)
                for (;;)
                    curr_ns = sb_timer_value(&sb_globals.exec_timer);
                    //随机的
                    intr_ns = (long) (log(1 - (double)sb_rnd() / (double)SB_MAX_RND) / (-(double)sb_globals.tx_rate)*1000000);
                    next_ns = next_ns + intr_ns*1000;
                    if (next_ns > curr_ns)
                        pause_ns = next_ns - curr_ns;
                    else
                        pause_ns = 1000;
                    usleep(pause_ns / 1000);
                    queue_array[i].event_time = sb_timer_value(&sb_globals.exec_timer);
                    SB_LIST_ADD_TAIL(&queue_array[i].listitem, &event_queue);
                    sb_globals.event_queue_length++;
                    pthread_cond_signal(&event_queue_cv);                    
        //起checkpoint线程
        if (sb_globals.n_checkpoints > 0)
            pthread_create(&checkpoints_thread, &thread_attr,&checkpoints_thread_proc, NULL)
        //起所有测试线程
        for(i = 0; i < sb_globals.num_threads; i++)
            pthread_create(&(threads[i].thread), &thread_attr,&runner_thread, (void*)(threads + i))
                sb_globals.num_running++;
                do
                    //tx-rate mode
                    if (sb_globals.tx_rate > 0)
                        //queue满了直接break
                        //获取全局互斥锁
                        //queue length为0说明都占满了, 此时需要等待条件量event_queue_cv, 这个东西会在tx-rate线程里面释放
                        //走到这说明queue又有空间了, 取一个entry出来
                        sb_globals.event_queue_length--
                        sb_globals.concurrency++
                        //释放全局互斥锁
                    request = get_request(test, thread_id);
                    execute_request(test, &request, thread_id)
                        test->ops.execute_request(r, thread_id)
                            sb_lua_op_execute_request(sb_request_t *sb_req, int thread_id)
                                lua_getglobal(L, "event")
                                lua_pcall(L, 1, 1, 0)
                    sb_globals.concurrency--;
                while ((request.type != SB_REQ_TYPE_NULL) && (!sb_globals.error) );
                test->ops.thread_done(thread_id);
                sb_globals.num_running--;
```