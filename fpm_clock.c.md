# fpm clock

时钟函数

int clock_gettime(clockid_t clk_id, struct timespec *tp);

clk_id 检索和设置的clk_id指定的时钟时间，下面是类型说明

| 名称 | 说明  |
| --- | --- |
|CLOCK_REALTIME|系统实时时间，随系统时间而改变，即从UTC1970-01-01 00:00:00开始，如果用户修该了时间，则对应的时间也相应改变|
|CLOCK_MONOTONIC|从系统启动这一刻起开始计时,不受系统时间被用户改变的影响|
|CLOCK_PROCESS_CPUTIME_ID|本进程到当前代码系统CPU花费的时间|
|CLOCK_THREAD_CPUTIME_ID|本线程到当前代码系统CPU花费的时间|


```c
static int monotonic_works;   

int fpm_clock_init() /* {{{ */
{
    struct timespec ts;       

    monotonic_works = 0;      

    //判断是否支持MONOTONIC
    if (0 == clock_gettime(CLOCK_MONOTONIC, &ts)) {
        monotonic_works = 1;  
    }

    return 0;
}
/* }}} */

int fpm_clock_get(struct timeval *tv) /* {{{ */
{
    if (monotonic_works) {    
        struct timespec ts;   
        //获取时钟时间
        if (0 > clock_gettime(CLOCK_MONOTONIC, &ts)) {
            zlog(ZLOG_SYSERROR, "clock_gettime() failed");
            return -1;
        }

        tv->tv_sec = ts.tv_sec;        
        tv->tv_usec = ts.tv_nsec / 1000;
        return 0;
    }
    //获得当前精确时间
    return gettimeofday(tv, 0);    
}

```

