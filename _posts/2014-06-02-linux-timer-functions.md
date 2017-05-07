---
title: Linux 时间函数
layout: post
comments: true
language: chinese
category: [linux,program]
keywords: linux,program
description: 简单介绍下 Linux 中与时间相关的函数。
---

简单介绍下 Linux 中与时间相关的函数。


<!-- more -->

## 获取时间函数

简单介绍下与获取时间相关的系统调用。

### time() 秒级

{% highlight c %}
#include <time.h>

//----- time_t一般为long int，不过不同平台可能不同，打印可通过printf("%ju", (uintmax_t)ret)打印
time_t time(time_t *tloc);

char *ctime(const time_t *timep);                // 错误返回NULL
char *ctime_r(const time_t *timep, char *buf);   // buf至少26bytes，返回与buf相同
{% endhighlight %}

如果 tloc 不为 NULL，那么数据同时会保存在 tloc 中，成功返回从 epoch 开始的时间，单位为秒；否则返回 -1 。

### ftime() 毫秒级

{% highlight c %}
#include <sys/timeb.h>
struct timeb {
    time_t   time;                // 为1970-01-01至今的秒数
    unsigned   short   millitm;   // 毫秒
    short   timezonel;            // 为目前时区和Greenwich相差的时间，单位为分钟，东区为负
    short   dstflag;              // 非0代表启用夏时制
};
int ftime(struct timeb *tp);
{% endhighlight %}

总是返回 0 。


### gettimeofday() 微秒级

gettimeofday() 函数可以获得当前系统的绝对时间。

{% highlight c %}
struct timeval {
    time_t      tv_sec;
    suseconds_t tv_usec;
};
int gettimeofday ( struct timeval * tv , struct timezone * tz )
{% endhighlight %}

可以通过如下函数测试。

{% highlight c %}
#include <stdio.h>
#include <unistd.h>
#include <sys/time.h>
int main(void)
{
    int i=10000000;
    struct timeval begin, end;
    gettimeofday(&begin, NULL);
    while (--i);
    gettimeofday(&end, NULL);
    double span = end.tv_sec-begin.tv_sec + (end.tv_usec-begin.tv_usec)/1000000.0;
    printf("time: %.12f\n", span);
    return 0;
}
{% endhighlight %}




### clock_gettime() 纳秒级

编译连接时需要加上 ```-lrt```，不过不加也可以编译，应该是非实时的。```struct timespect *tp``` 用来存储当前的时间，其结构和函数声明如下，返回 0 表示成功，-1 表示失败。

{% highlight c %}
struct timespec {
    time_t tv_sec;    /* seconds */
    long tv_nsec;     /* nanoseconds */
};
int clock_gettime(clockid_t clk_id, struct timespect *tp);
{% endhighlight %}

clk_id 用于指定计时时钟的类型，简单列举如下几种，详见 ```man 2 clock_gettime``` 。

* CLOCK_REALTIME/CLOCK_REALTIME_COARSE<br>
系统实时时间 (wall-clock time)，即从 ```UTC 1970.01.01 00:00:00``` 开始计时，随系统实时时间改变而改变，包括通过系统函数手动调整系统时间，或者通过 ```adjtime()``` 或者 NTP 调整时间。

* CLOCK_MONOTONIC/CLOCK_MONOTONIC_COARSE<br>
从系统启动开始计时，不受系统时间被用户改变的影响，但会受像 ```adjtime()``` 或者 NTP 之类渐进调整的影响。

* CLOCK_MONOTONIC_RAW<br>
与上述的 ```CLOCK_MONOTONIC``` 相同，只是不会受 ```adjtime()``` 以及 NTP 的影响。

* CLOCK_PROCESS_CPUTIME_ID<br>
本进程到当前代码系统 CPU 花费的时间。

* CLOCK_THREAD_CPUTIME_ID<br>
本线程到当前代码系统 CPU 花费的时间。

可以通过如下程序测试。

{% highlight c %}
#include <time.h>
#include <stdio.h>

struct timespec diff(struct timespec start, struct timespec end)
{
    struct timespec temp;
    if ((end.tv_nsec-start.tv_nsec)<0) {
        temp.tv_sec = end.tv_sec-start.tv_sec-1;
        temp.tv_nsec = 1000000000+end.tv_nsec-start.tv_nsec;
    } else {
        temp.tv_sec = end.tv_sec-start.tv_sec;
        temp.tv_nsec = end.tv_nsec-start.tv_nsec;
    }
    return temp;
}

int main(void)
{
    struct timespec t, begin, end;
    int temp, i;
    clock_gettime(CLOCK_REALTIME, &t);
    printf("CLOCK_REALTIME: %d, %d\n", t.tv_sec, t.tv_nsec);
    clock_gettime(CLOCK_MONOTONIC, &t);
    printf("CLOCK_MONOTONIC: %d, %d\n", t.tv_sec, t.tv_nsec);
    clock_gettime(CLOCK_THREAD_CPUTIME_ID, &t);
    printf("CLOCK_THREAD_CPUTIME_ID: %d, %d\n", t.tv_sec, t.tv_nsec);

    clock_gettime(CLOCK_PROCESS_CPUTIME_ID, &begin);
    printf("CLOCK_PROCESS_CPUTIME_ID: %d, %d\n", begin.tv_sec, begin.tv_nsec);
    for (i = 0; i < 242000000; i++)
        temp+=temp;
    clock_gettime(CLOCK_PROCESS_CPUTIME_ID, &end);
    t = diff(begin, end);
    printf("diff CLOCK_PROCESS_CPUTIME_ID: %d, %d\n", t.tv_sec, t.tv_nsec);
}
{% endhighlight %}

<!--

### 时间转换相关

将时间转换为自 1970.01.01 以来逝去时间的秒数，发生错误时返回 -1 。

{% highlight text %}
struct tm {
    int tm_sec;         /* seconds */
    int tm_min;         /* minutes */
    int tm_hour;        /* hours */
    int tm_mday;        /* day of the month */
    int tm_mon;         /* month */
    int tm_year;        /* year */
    int tm_wday;        /* day of the week */
    int tm_yday;        /* day in the year */
    int tm_isdst;       /* daylight saving time */
};

time_t mktime(struct tm * timeptr);
{% endhighlight %}

如果时间在夏令时，tm_isdst 设置为1，否则设置为0，若未知，则设置为 -1。



### ANSI clock()

clock() 返回值类型是 clock_t，该值除以 CLOCKS_PER_SEC (GNU 定义为 1000000) 得出消耗的 CPU 时间，一般用两次 clock() 来计算进程自身运行的时间。不过，该函数存在如下的问题：

* 对于 32bits 如果超过 72 分钟，就有可能会导致溢出；
* 该函数没有考虑 CPU 被子进程使用的情况；
* 也不能区分用户空间和内核空间。

因此，该函数在 Linux 系统上几乎没有意义，可以通过如下程序进行简单的测试。

{% highlight c %}
#include <time.h>
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    int i = 1000;
    clock_t start, finish;
    double  duration;
    printf( "Time to do %ld empty loops is ", i );
    start = clock();
    while (--i){
        system("cd");
    }
    finish = clock();
    duration = (double)(finish - start) / CLOCKS_PER_SEC;
    printf("%f seconds\n", duration);
    return 0;
}
{% endhighlight %}

接下来，可以通过如下方式进行测试。

{% highlight text %}
$ gcc test.c -o test
$ time ./test
Time to do 1000 empty loops is 0.070000 seconds

real    0m1.471s
user    0m0.551s
sys     0m0.749s
{% endhighlight %}

实际上，程序调用 ```system("cd");``` 主要是系统模式子进程的消耗，如上程序不能体现这一点 。




二)times()时间函数
1)概述:
原型如下：
clock_t times(struct tms *buf);


tms结构体如下:
strace tms{
 clock_t tms_utime;
 clock_t tms_stime;
 clock_t tms_cutime;
 clock_t tms_cstime;
}

注释:
tms_utime记录的是进程执行用户代码的时间.
tms_stime记录的是进程执行内核代码的时间.
tms_cutime记录的是子进程执行用户代码的时间.
tms_cstime记录的是子进程执行内核代码的时间.

2)测试:
vi test2.c
#include <sys/times.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

static void do_cmd(char *);
static void pr_times(clock_t, struct tms *, struct tms *);


int main(int argc, char *argv[]){
        int i;
        for(i=1; argv[i]!=NULL; i++){
                do_cmd(argv[i]);
        }
        exit(1);
}
static void do_cmd(char *cmd){
        struct tms tmsstart, tmsend;
        clock_t start, end;
        int status;
        if((start=times(&tmsstart))== -1)
                puts("times error");
        if((status=system(cmd))<0)
                puts("system error");
        if((end=times(&tmsend))== -1)
                puts("times error");
        pr_times(end-start, &tmsstart, &tmsend);
        exit(0);
}
static void pr_times(clock_t real, struct tms *tmsstart, struct tms *tmsend){
        static long clktck=0;
        if(0 == clktck)
                if((clktck=sysconf(_SC_CLK_TCK))<0)
                           puts("sysconf err");
        printf("real:%7.2f\n", real/(double)clktck);
        printf("user-cpu:%7.2f\n", (tmsend->tms_utime - tmsstart->tms_utime)/(double)clktck);
        printf("system-cpu:%7.2f\n", (tmsend->tms_stime - tmsstart->tms_stime)/(double)clktck);
        printf("child-user-cpu:%7.2f\n", (tmsend->tms_cutime - tmsstart->tms_cutime)/(double)clktck);
        printf("child-system-cpu:%7.2f\n", (tmsend->tms_cstime - tmsstart->tms_cstime)/(double)clktck);
}

编译:
gcc test2.c -o test2

测试这个程序:
time ./test2 "dd if=/dev/zero f=/dev/null bs=1M count=10000"
10000+0 records in
10000+0 records out
10485760000 bytes (10 GB) copied, 4.93028 s, 2.1 GB/s
real:   4.94
user-cpu:   0.00
system-cpu:   0.00
child-user-cpu:   0.01
child-system-cpu:   4.82

real    0m4.943s
user    0m0.016s
sys     0m4.828s

3)总结:
(1)通过这个测试,系统的time程序与test2程序输出基本一致了.
(2)(double)clktck是通过clktck=sysconf(_SC_CLK_TCK)来取的,也就是要得到user-cpu所占用的时间,就要用
(tmsend->tms_utime - tmsstart->tms_utime)/(double)clktck);
(3)clock_t times(struct tms *buf);返回值是过去一段时间内时钟嘀嗒的次数.
(4)times()函数返回值也是一个相对时间.


三)实时函数clock_gettime
在POSIX1003.1中增添了这个函数,它的原型如下：
int clock_gettime(clockid_t clk_id, struct timespec *tp);

它有以下的特点:
1)它也有一个时间结构体:timespec ,timespec计算时间次数的单位是十亿分之一秒.
strace timespec{
 time_t tv_sec;
 long tv_nsec;
}

2)clockid_t是确定哪个时钟类型.
CLOCK_REALTIME: 标准POSIX实时时钟
CLOCK_MONOTONIC: POSIX时钟,以恒定速率运行;不会复位和调整,它的取值和CLOCK_REALTIME是一样的.
CLOCK_PROCESS_CPUTIME_ID和CLOCK_THREAD_CPUTIME_ID是CPU中的硬件计时器中实现的.

3)测试:
#include<time.h>
#include<stdio.h>
#include<stdlib.h>

#define MILLION 1000000

int main(void)
{
        long int loop = 1000;
        struct timespec tpstart;
        struct timespec tpend;
        long timedif;
        clock_gettime(CLOCK_MONOTONIC, &tpstart);
        while (--loop){
                system("cd");
        }

        clock_gettime(CLOCK_MONOTONIC, &tpend);
        timedif = MILLION*(tpend.tv_sec-tpstart.tv_sec)+(tpend.tv_nsec-tpstart.tv_nsec)/1000;
        fprintf(stdout, "it took %ld microseconds\n", timedif);

        return 0;
}

编译:
gcc test3.c -lrt -o test3


计算时间:
time ./test3
it took 3463843 microseconds

real    0m3.467s
user    0m0.512s
sys     0m2.936s


2)
clock()函数的精确度是10毫秒(ms)
times()函数的精确度是10毫秒(ms)


3)测试4种函数的精确度:
vi test4.c

#include    <stdio.h>
#include    <stdlib.h>
#include    <unistd.h>
#include    <time.h>
#include    <sys/times.h>
#include    <sys/time.h>
#define WAIT for(i=0;i<298765432;i++);
#define MILLION    1000000
    int
main ( int argc, char *argv[] )
{
    int i;
    long ttt;
    clock_t s,e;
    struct tms aaa;

    s=clock();
    WAIT;
    e=clock();
    printf("clock time : %.12f\n",(e-s)/(double)CLOCKS_PER_SEC);

    long tps = sysconf(_SC_CLK_TCK);
    s=times(&aaa);
    WAIT;
    e=times(&aaa);
    printf("times time : %.12f\n",(e-s)/(double)tps);

    struct timeval tvs,tve;
    gettimeofday(&tvs,NULL);
    WAIT;
    gettimeofday(&tve,NULL);
    double span = tve.tv_sec-tvs.tv_sec + (tve.tv_usec-tvs.tv_usec)/1000000.0;
    printf("gettimeofday time: %.12f\n",span);

    struct timespec tpstart;
    struct timespec tpend;

    clock_gettime(CLOCK_REALTIME, &tpstart);
    WAIT;
    clock_gettime(CLOCK_REALTIME, &tpend);
    double timedif = (tpend.tv_sec-tpstart.tv_sec)+(tpend.tv_nsec-tpstart.tv_nsec)/1000000000.0;
    printf("clock_gettime time: %.12f\n", timedif);

    return EXIT_SUCCESS;
}

gcc -lrt test4.c -o test4
debian:/tmp# ./test4
clock time : 1.190000000000
times time : 1.180000000000
gettimeofday time: 1.186477000000
clock_gettime time: 1.179271718000

六)内核时钟

默认的Linux时钟周期是100HZ,而现在最新的内核时钟周期默认为250HZ.
如何得到内核的时钟周期呢?

grep ^CONFIG_HZ /boot/config-2.6.26-1-xen-amd64

CONFIG_HZ_250=y
CONFIG_HZ=250

结果就是250HZ.

而用sysconf(_SC_CLK_TCK);得到的却是100HZ
例如:
#include    <stdio.h>
#include    <stdlib.h>
#include    <unistd.h>
#include    <time.h>
#include    <sys/times.h>
#include    <sys/time.h>

int
main ( int argc, char *argv[] )
{
    long tps = sysconf(_SC_CLK_TCK);
    printf("%ld\n", tps);

    return EXIT_SUCCESS;
}

为什么得到的是不同的值呢？
因为sysconf(_SC_CLK_TCK)和CONFIG_HZ所代表的意义是不同的.
sysconf(_SC_CLK_TCK)是GNU标准库的clock_t频率.
它的定义位置在:/usr/include/asm/param.h

例如:
#ifndef HZ
#define HZ 100
#endif


最后总结一下内核时间:
内核的标准时间是jiffy,一个jiffy就是一个内部时钟周期,而内部时钟周期是由250HZ的频率所产生中的,也就是一个时钟滴答,间隔时间是4毫秒(ms).
也就是说:
1个jiffy=1个内部时钟周期=250HZ=1个时钟滴答=4毫秒

每经过一个时钟滴答就会调用一次时钟中断处理程序，处理程序用jiffy来累计时钟滴答数,每发生一次时钟中断就增1.
而每个中断之后,系统通过调度程序跟据时间片选择是否要进程继续运行,或让进程进入就绪状态.
最后需要说明的是每个操作系统的时钟滴答频率都是不一样的,LINUX可以选择(100,250,1000)HZ,而DOS的频率是55HZ.

七)为应用程序计时
用time程序可以监视任何命令或脚本占用CPU的情况.

1)bash内置命令time
例如:
time sleep 1
real    0m1.016s
user    0m0.000s
sys     0m0.004s


2)/usr/bin/time的一般命令行
例如:
\time sleep 1
0.00user 0.00system 0:01.01elapsed 0%CPU (0avgtext+0avgdata 0maxresident)k
0inputs+0outputs (1major+176minor)pagefaults 0swaps


注：
在命令前加上斜杠可以绕过内部命令.
/usr/bin/time还可以加上-v看到更具体的输出:
\time -v sleep 1
        Command being timed: "sleep 1"
        User time (seconds): 0.00
        System time (seconds): 0.00
        Percent of CPU this job got: 0%
        Elapsed (wall clock) time (h:mm:ss or m:ss): 0:01.00
        Average shared text size (kbytes): 0
        Average unshared data size (kbytes): 0
        Average stack size (kbytes): 0
        Average total size (kbytes): 0
        Maximum resident set size (kbytes): 0
        Average resident set size (kbytes): 0
        Major (requiring I/O) page faults: 0
        Minor (reclaiming a frame) page faults: 178
        Voluntary context switches: 2
        Involuntary context switches: 0
        Swaps: 0
        File system inputs: 0
        File system outputs: 0
        Socket messages sent: 0
        Socket messages received: 0
        Signals delivered: 0
        Page size (bytes): 4096
        Exit status: 0

这里的输出更多来源于结构体rusage.


最后，我们看到real time大于user time和sys time的总和，这说明进程不是在系统调用中阻塞,就是得不到运行的机会.
而sleep()的运用，也说明了这一点.





http://en.wikipedia.org/wiki/Year_2038_problem


    Identifier  Description
    Time
    manipulation    difftime    computes the difference between times
    time    returns the current time of the system as time since the epoch (which is usually the Unix epoch)
    clock   returns a processor tick count associated with the process

    Format
    conversions     asctime     converts a tm object to a textual representation (deprecated)
    strftime    converts a tm object to custom textual representation
    wcsftime    converts a tm object to custom wide string textual representation
    gmtime  converts time since the epoch to calendar time expressed as Coordinated Universal Time[2]
    localtime   converts time since the epoch to calendar time expressed as local time
    mktime  converts calendar time to time since the epoch
    Constants   CLOCKS_PER_SEC  number of processor clock ticks per second


    tm  calendar time type
    time_t  time since the epoch type
    clock_t     process running time type

http://www.cnblogs.com/wenqiang/p/5678451.html
http://www.cnblogs.com/xmphoenix/archive/2011/05/09/2041546.html
-->

{% highlight text %}
{% endhighlight %}
