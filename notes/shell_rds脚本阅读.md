- [simsim_charts脚本](#simsim_charts脚本)
  - [process_data.sh](#process_datash)
  - [create_charts.sh](#create_chartssh)
  - [R语言](#r语言)
  - [rds lua脚本](#rds-lua脚本)
  - [rds bash脚本](#rds-bash脚本)

# simsim_charts脚本

## process_data.sh

* 使用方法

```sh
BENCH_SERIES=thunder_optimized_xfs MYSQL_TEMPLATES_NAME=optimized SYSBENCH_MAX_TIME=180 bash ./start_benchmarks.sh

SYSBENCH_MAX_TIME=180 CHART_SERIES=results/{thx_optimized_60s,thx_network_60s,x86_baseline_60s} CHART_NAME="_optimized_vs_network_vs_x86" sh ./process_data.sh

SYSBENCH_MAX_TIME=180 CHART_SERIES=results/{thx_optimized_60s,thx_network_60s,x86_baseline_60s} CHART_NAME="_optimized_vs_network_vs_x86" sh ./create_charts.sh

```

* 这个脚本是把所有的CHART_SERIES的tps和lat的相关信息, 全部提取到文件里

* `for series in $(eval echo $CHART_SERIES)`里面`eval echo $CHART_SERIES`的作用是**展开变量**

```sh
$ CHART_SERIES=results/{thx_optimized_60s,thx_network_60s,x86_baseline_60s}

$ echo $CHART_SERIES

results/{thx_optimized_60s,thx_network_60s,x86_baseline_60s}

$ eval echo $CHART_SERIES

results/thx_optimized_60s results/thx_network_60s results/x86_baseline_60s
```

* 一个标准的shell从usage开始


```shell
usage()
{
    cat <<EOF
Usage:
SYSBENCH_MAX_TIME=N CHART_SERIES=S CHART_NAME=S $0
EOF
}
```

* `for conf in A200 B5 C1 C2 C4 C8`重点是in后面的集合就只是用空格隔开

* 要把一段脚本的输出重定向到文件, 可以这样

```sh
(
some commands
) > output-filename

```

* awk的例子

```shell
awk '
BEGIN {OFS=","}
/tps:/ {
    sub("\\[ *", "", $0);
    sub("s]","",$1);
    sub(",","",$5);
    sub("ms","",$12);
    print '\"$s\",\"$conf\",\"$threads\",$((counter*duration))'+$1,$1,"tps",$5;
    print '\"$s\",\"$conf\",\"$threads\",$((counter*duration))'+$1,$1,"lat",$12
}' $filename
```

```shell
awk '
BEGIN {OFS=","}
/transactions:/ {sub("\\(","",$3); print '\"$s\",\"$conf\",\"$threads\",\"tps\",'$3}
/approx./ {sub("ms","",$4); print '\"$s\",\"$conf\",\"$threads\",\"lat\",'$4}' $f
```

```sh
awk '/search pattern1/ {Actions}
     /search pattern2/ {Actions}' file

```
    - awk '{pattern + action}' {filenames}, 按行处理, 对匹配patten的行, 顺序执行{}里面的操作
    - patten就是//里面的东西
    - $0代表整行,$1是第一个字段
    - sub(match, replace, string)是字符串替换
    - BEGIN是说在扫描之前执行的, 相对的END是在最后都扫描完了再执行的
    - OFS是输出的分隔符, FS是输入的分隔符, 默认都是space
    - print输出字符串要加"", 比如print "hello"; 上面转义了", 因为print后面接了单引号

* 输出文件的效果如

```
thunder_xfs,A200,1,tps,3737.49
thunder_xfs,A200,1,lat,75.43
thunder_xfs,A200,4,tps,5261.52
thunder_xfs,A200,4,lat,216.29
thunder_xfs,A200,8,tps,7595.05
thunder_xfs,A200,8,lat,313.97
thunder_xfs,B5,1,tps,245.66
thunder_xfs,B5,1,lat,25.24
thunder_xfs,B5,4,tps,715.88
thunder_xfs,B5,4,lat,39.76
thunder_xfs,B5,8,tps,1160.96
thunder_xfs,B5,8,lat,50.93
```

## create_charts.sh

## R语言

* 先要安装R, 似乎不用装那么多

**apt install r-base**

```
$ cat /etc/apt/sources.list
deb http://mirror.bjtu.edu.cn/cran/bin/linux/ubuntu trusty/
```

```
apt install libxt-dev
apt install libcurl4-openssl-dev
apt install gfortran
apt install libmpfr-dev
apt install liblapack-dev
apt install r-base
apt install r-cran*
```

还要进R的shell

```r
install.packages('ggplot2', dep = TRUE)
install.packages('gridExtra', dep = TRUE)
install.packages('reshape2', dep = TRUE)
install.packages('plyr', dep = TRUE)
install.packages('Matrix', dep = TRUE)
```

* data.frame是个表格, 一般每个列都是同样类型的元素(似乎不是必须?), 而每行好像是条记录的感觉

```r
> student<-data.frame(ID=c(11,12,13),Name=c("Devin","Edward","Wenli"),Gender=c("M","M","F"),Birthdate=c("1984-12-29","1983-5-6","1986-8-8"))
> student
  ID   Name Gender  Birthdate
1 11  Devin      M 1984-12-29
2 12 Edward      M   1983-5-6
3 13  Wenli      F   1986-8-8
```

* 代码注释之tps和latency

```r
#sb是data.frame类型
sb<-read.table("$r_data_file",sep=",",as.is=T,header=F)
#c(各个元素)形成一个list, 这里的c是个函数, 是combine的意思
colnames(sb)<-c("series","conf", "threads", "metric", "value")

#这里体现了函数语言的强大之处, metric作为入参是没有定义的, 但传到sb里面时, 就有定义了.
sb_tps<-subset(sb,grepl("tps", metric))
sb_lat<-subset(sb,grepl("lat", metric))

#重点来了, ggplot是个很强大的函数, 其基本思想是用+号来叠加图层
#aes很强大, 可以在ggplot中使用(相当于全局), 也可以在局部图层使用.
#aes里面的这个as.factor应该是把一个vector变成一个factor(因子), 这里面是按series来画不同颜色
#group是说按series来分组画?
q1<-ggplot(data=sb_tps,
           aes(threads,y=value,colour=as.factor(series),
               group=as.factor(series)))+
  geom_line(size=1)+
  geom_point()+
  scale_color_hue(l=55,name="")+
  guides(fill = guide_legend(title=NULL, reverse=FALSE))+
  ylab("TPS")+
  expand_limits(y=0)+
  xlab("threads per instance")+
  ggtitle(paste("simsim benchmark, 2 sockets, TPS" ))+

  #在底部显示legend(图例说明)
  theme(legend.position="bottom")+
  #以2为底的对数, 保证1 4 8 16在x轴等距
  scale_x_continuous(trans=log2_trans(),
                     breaks = c(1,4,8,16,32,64,128,256,512))+
  #facet(分面), 这里. ~ conf的意思是水平不变, 垂直以conf为类别来分面
  facet_grid( . ~ conf, scales="free_y")

svg("charts/simsim_tps${CHART_NAME}.svg", width=17)
print(q1)
dev.off()
```

* 代码注释之variance

```r
sb<-read.table("$r_data_file",sep=",",as.is=T,header=F)
#这里多了一个counter, 是time*threads, 作为x轴
colnames(sb)<-c("series","conf", "threads", "counter", "time", "metric", "value")
sb_tps<-subset(sb,grepl("tps", metric))
sb_lat<-subset(sb,grepl("lat", metric))

#counter作x轴, 还是以series作为组, 用不同颜色标记
q1<-ggplot(data=sb_tps,
           aes(counter,y=value,colour=as.factor(series),
               group=as.factor(series)))+

#因为是绘制抖动图, 只画点, 没有线
#  geom_line(size=1)+
  #这里的alpha应该是透明度的意思
  geom_point(alpha=8/10)+
  scale_color_hue(l=55,name="")+
  guides(fill = guide_legend(title=NULL, reverse=FALSE))+
  ylab("TPS")+
  expand_limits(x=1,y=0)+
  xlab("time * threads per instance")+
  ggtitle(paste("simsim benchmark, 2 sockets, TPS variance" ))+
  theme(legend.position="bottom")+

  #这里重点了, 用counter作x轴, 首先counter是连续的, 概念上是以秒为单位的, 按照1 4 8 16等threads数跑出来的.
  #但我们这里人们关注的是threads数不同的情况下的抖动图, 其实对x轴实际的数值并不关心.
  #所以这里设置了breaks和labels来标注threads的情况

  scale_x_continuous(#trans=log2_trans()
                    breaks = c(1,$((duration+1)),$((duration*2+1)),
                               $((duration*3+1)),$((duration*4+1)),
                               $((duration*5+1)),$((duration*6+1)),
                               $((duration*7+1)),$((duration*8+1))),
                    labels=c(1,4,8,16,32,64,128,256,512)
                     )+

#  scale_y_continuous(trans=log10_trans()) +
  #和上面不同, 这里以conf来水平分面
  facet_grid(conf ~ ., scales="free_y")
svg("charts/simsim_variance_tps${CHART_NAME}.svg", height=10, width=17)
print(q1)
```

## rds lua脚本

* 注释

单行注释`--`

多行注释

```lua
--[[
****************************************************************************
Input parameters:
--simsim-databases = N: the number of databases to create/use (1 by default)

****************************************************************************
--]]
```

* 感觉上lua和shell的语法类似

* sysbench的lua脚本可以用db相关操作
```lua
db_query("DROP DATABASE IF EXISTS " .. database_name)      db_query("CREATE DATABASE " .. database_name)
```

* 在```db_query("BEGIN")```和```db_query("COMMIT")```之间的东西就是一次transaction的内容

* 主要的东西在event里面, sysbench的runner线程会调用

```c
void *runner_thread(void *arg)
    do
        execute_request(test, &request, thread_id)
            test->ops.execute_request(r, thread_id)
                sb_lua_op_execute_request(sb_request_t *sb_req, int thread_id)
                    lua_getglobal(L, "event") //见下面
                    lua_pcall(L, 1, 1, 0)

    while ((request.type != SB_REQ_TYPE_NULL) && (!sb_globals.error) );
```

* simsim里面, event里做的事情是很多insert到不同的表, 这些表的格式都是在prepare阶段准备好的.

## rds bash脚本

* set -eu
shell内建的命令, 可以用help xxx来看帮助, 比如help for
这里
-e: 如果有命令返回非零值则立刻推出
-u: 没定义的变量直接使用时当成错误

* 小括号和后台(&)的异同

相同的地方都是会新建一个子进程来执行, 那么既然是子进程, 里面的变量啦,路径啦都不会影响父进程.

不同点在与: 后台执行是非阻塞的, 而小括号是阻塞的;

* 后台进程id

```shell
mysqld --defaults-file=$mysql_basedir/my.cnf \ 
--user=root >/dev/null 2>&1 &        pids[$i]=$!
```

**这里面有两个知识点, $!是最新加入后台的进程id; pids是个数组, 展开后如:pids[1]=1022**

* 如何打印大块文字? --用cat <<EOF
```shell
cat <<EOF
*****************************************************************************
Starting a benchmark series with the following parameters:
Data root directory:        $BENCH_ROOT
Benchmark series label:     $BENCH_SERIES
Single socket:          $BENCH_SINGLE_SOCKET
Results directory:      $RESULTS_DIR
sysbench directory:     $SYSBENCH_DIR
Configurations to run:      $MYSQL_CONFIGURATIONS
Subconfiguration for case C:    $MYSQL_C_INSTANCES
Threads combinations:       $SYSBENCH_THREAD_LIST
sysbench extra arguments:   $SYSBENCH_SIMSIM_ARGS
Single run duration:        ${SYSBENCH_MAX_TIME}s
*****************************************************************************
EOF
```

* wait用来等待所有的后台进程结束

不加参数是等待所有后台进程, 也可以这样
比如在for循环里
`job_ids="$job_ids $!"`

在循环外面:
`wait $job_ids`

* sysbench测试mysql的时候, 支持不同的端口号;下面的SB_PORTS就是这么一个变量"3001,3002,3003..."

```shell
    SB_SOCKETS="$BENCH_ROOT/${conf}${instances}/data3001/tmp/mysql.sock"
    SB_PORTS="3001"
    for i in $(seq 2 $instances)
    do
        mysql_port=$((3000 + i));
        mysql_basedir=$BENCH_ROOT/${conf}${instances}/data$mysql_port
        mysql_tmpdir=$mysql_basedir/tmp
        SB_SOCKETS="$SB_SOCKETS,$mysql_tmpdir/mysql.sock"
        SB_PORTS="$SB_PORTS,$mysql_port"
    done

#   SB_CONNECT_ARGS="--mysql-socket=$SB_SOCKETS"
    SB_CONNECT_ARGS="--mysql-host=127.0.0.1 --mysql-port=$SB_PORTS"
```

**这里面`SB_PORTS="$SB_PORTS,$mysql_port"`相当于一个列表变量**
