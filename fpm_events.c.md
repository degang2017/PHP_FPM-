# fpm 事件模型

#### 事件标识
这里的TIMEOUT跟PERSIST对应，READ与EDGE对应。

```c
//事件超时,主要用于触发器
#define FPM_EV_TIMEOUT  (1 << 0)
//事件读
#define FPM_EV_READ     (1 << 1)
//事件持续化，这个事件会保持挂起状态，即使回调函数被执行,重新放回超时事件
#define FPM_EV_PERSIST  (1 << 2)
//边缘触发
#define FPM_EV_EDGE     (1 << 3)
//TIMEOUT的fd为-1
#define fpm_event_set_timer(ev, flags, cb, arg) fpm_event_set((ev), -1, (flags), (cb), (arg))
```

```c
//事件
struct fpm_event_s {          
    //文件描述符
    int fd;                   /* not set with FPM_EV_TIMEOUT */
    //事件触发时间点          
    struct timeval timeout;   /* next time to trigger */
    //事件超时触发间隔        
    struct timeval frequency; 
    //回调函数
    void (*callback)(struct fpm_event_s *, short, void *); 
    //参数
    void *arg;
    //事件标识
    int flags;
    //文件描述符值
    int index;                /* index of the fd in the ufds array */
    //事件类型
    short which;              /* type of event */            
};
//事件队列，双向链表            
typedef struct fpm_event_queue_s {  
    struct fpm_event_queue_s *prev;
    struct fpm_event_queue_s *next;
    struct fpm_event_s *ev; 
} fpm_event_queue;          

//事件轮询模块          
struct fpm_event_module_s {
    //模块名    
    const char *name;
    //是否支持边缘触发
    int support_edge_trigger;
    int (*init)(int max_fd);
    int (*clean)(void);
    //等待获取事件
    int (*wait)(struct fpm_event_queue_s *queue, unsigned long int timeout);
    int (*add)(struct fpm_event_s *ev);
    int (*remove)(struct fpm_event_s *ev);
};

void fpm_event_loop(int err);
//运行事件回调函数
void fpm_event_fire(struct fpm_event_s *ev);
int fpm_event_init_main();
int fpm_event_set(struct fpm_event_s *ev, int fd, int flags, void (*callback)(struct fpm_event_s *, short, void *), void *arg);
int fpm_event_add(struct fpm_event_s *ev, unsigned long int timeout); 
int fpm_event_del(struct fpm_event_s *ev);
//初始化事件,这里会根据用户配置来选择，如果是epoll, machanism值为epool
int fpm_event_pre_init(char *machanism); 
//事件模块名
const char *fpm_event_machanism_name();
//是否支持边缘触发 
int fpm_event_support_edge_trigger();

```

##### 事件队列分为两种

第一种timer触发器队列：signal、fpm_pctl_perform_idle_server_maintenance_heartbeat 、fpm_pctl_heartbeat及fpm_systemd_heartbeat定时器。

第二种文件描述符队列

| 函数 | 说明  |
| --- | --- |
|fpm_got_signal|主要用来主进程信号处理，子进程并没有采用此方式|
|fpm_pctl_heartbeat|主要检查慢日子|
|fpm_pctl_perform_idle_server_maintenance_heartbeat |主要worker进程动态管理，根据用户配置的进程模式，比如动态、静态、相应，另外一个就是更新记分板的统计数据|
|fpm_systemd_heartbeat|发送fpm状态信息给systemd|

```c
//信号处理
static void fpm_got_signal(struct fpm_event_s *ev, short which, void *arg) /* {{{ */
{
    char c;
    int res, ret;
    //获取事件文件描述符
    int fd = ev->fd;

    do {
        do {
            res = read(fd, &c, 1); 
        } while (res == -1 && errno == EINTR); //EINTR 当阻塞于某个慢系统调用返回的信息

        if (res <= 0) {
            //EWOULDBLOCK 用于非阻塞模式，不需要重新读或者写
            //EAGAIN 没有数据可读请稍后再试
            if (res < 0 && errno != EAGAIN && errno != EWOULDBLOCK) {
                zlog(ZLOG_SYSERROR, "unable to read from the signal pipe");
            }   
            return;
        }   

        //退出信息捕获
        switch (c) {
            //当子进程停止或退出时通知父进程
            case 'C' :                  /* SIGCHLD */
                zlog(ZLOG_DEBUG, "received SIGCHLD");
                fpm_children_bury();
                break;
            //中断进程,终端在用户按下CTRL+C发送到前台进程
            case 'I' :                  /* SIGINT  */
                zlog(ZLOG_DEBUG, "received SIGINT");
                zlog(ZLOG_NOTICE, "Terminating ...");
                fpm_pctl(FPM_PCTL_STATE_TERMINATING, FPM_PCTL_ACTION_SET);
                break;
            //是不带参数时kill发送的信号，进程终止运行
            case 'T' :                  /* SIGTERM */
                zlog(ZLOG_DEBUG, "received SIGTERM");
                zlog(ZLOG_NOTICE, "Terminating ...");
                fpm_pctl(FPM_PCTL_STATE_TERMINATING, FPM_PCTL_ACTION_SET);
                break;
            //是其控制终端发送到进程，当用户请求的过程中执行核心转储的信号
            case 'Q' :                  /* SIGQUIT */
                zlog(ZLOG_DEBUG, "received SIGQUIT");
                zlog(ZLOG_NOTICE, "Finishing ...");
                fpm_pctl(FPM_PCTL_STATE_FINISHING, FPM_PCTL_ACTION_SET);
                break;
            //用户自定义信号 默认处理：进程终止, 这里是重新打开log文件
            case '1' :                  /* SIGUSR1 */
                zlog(ZLOG_DEBUG, "received SIGUSR1");
                if (0 == fpm_stdio_open_error_log(1)) {
                    zlog(ZLOG_NOTICE, "error log file re-opened");
                } else {
                    zlog(ZLOG_ERROR, "unable to re-opened error log file");
                }   

                ret = fpm_log_open(1);
                if (ret == 0) {
                    zlog(ZLOG_NOTICE, "access log file re-opened");
                } else if (ret == -1) {
                    zlog(ZLOG_ERROR, "unable to re-opened access log file");
                }   
                /* else no access log are set */

                break;
            //用户自定义信号 默认处理：进程终止,这里是reloading事件
            case '2' :                  /* SIGUSR2 */
                zlog(ZLOG_DEBUG, "received SIGUSR2");
                zlog(ZLOG_NOTICE, "Reloading in progress ...");
                fpm_pctl(FPM_PCTL_STATE_RELOADING, FPM_PCTL_ACTION_SET);
                break;
        }   

        if (fpm_globals.is_child) {
            break;
        }   
    } while (1);
    return;
}

void fpm_pctl_heartbeat(struct fpm_event_s *ev, short which, void *arg) /* {{{ */
{
    static struct fpm_event_s heartbeat;
    struct timeval now;

    //不能为子进程
    if (fpm_globals.parent_pid != getpid()) {
        return; /* sanity check */
    }

    if (which == FPM_EV_TIMEOUT) {
        fpm_clock_get(&now);
        //查检request时间,慢日子
        fpm_pctl_check_request_timeout(&now);
        return;
    }

    /* ensure heartbeat is not lower than FPM_PCTL_MIN_HEARTBEAT */
    fpm_globals.heartbeat = MAX(fpm_globals.heartbeat, FPM_PCTL_MIN_HEARTBEAT);

    /* first call without setting to initialize the timer */
    zlog(ZLOG_DEBUG, "heartbeat have been set up with a timeout of %dms", fpm_globals.heartbeat);
    fpm_event_set_timer(&heartbeat, FPM_EV_PERSIST, &fpm_pctl_heartbeat, NULL);
    fpm_event_add(&heartbeat, fpm_globals.heartbeat);
}

void fpm_pctl_perform_idle_server_maintenance_heartbeat(struct fpm_event_s *ev, short which, void *arg) /* {{{ */
{
    static struct fpm_event_s heartbeat;
    struct timeval now;

    if (fpm_globals.parent_pid != getpid()) {
        return; /* sanity check */
    }

    if (which == FPM_EV_TIMEOUT) {
        fpm_clock_get(&now);
        if (fpm_pctl_can_spawn_children()) {
            //主要worker进程动态管理及更新记分板的统计数据
            fpm_pctl_perform_idle_server_maintenance(&now);

            /* if it's a child, stop here without creating the next event
             * this event is reserved to the master process
             */
            if (fpm_globals.is_child) {
                return;
            }
        }
        return;
    }

    /* first call without setting which to initialize the timer */
    fpm_event_set_timer(&heartbeat, FPM_EV_PERSIST, &fpm_pctl_perform_idle_server_maintenance_heartbeat, NULL);
    fpm_event_add(&heartbeat, FPM_IDLE_SERVER_MAINTENANCE_HEARTBEAT);
}
```

#### 添加事件

```c
static int fpm_event_queue_add(struct fpm_event_queue_s **queue, struct fpm_event_s *ev) /* {{{ */
{
    struct fpm_event_queue_s *elt;

    if (!queue || !ev) {
        return -1;
    }

    if (fpm_event_queue_isset(*queue, ev)) {
        return 0;
    }

    if (!(elt = malloc(sizeof(struct fpm_event_queue_s)))) {
        zlog(ZLOG_SYSERROR, "Unable to add the event to queue: malloc() failed");
        return -1;
    }
    elt->prev = NULL;
    elt->next = NULL;
    elt->ev = ev;

    //将事件加入队列
    if (*queue) {
        (*queue)->prev = elt;
        elt->next = *queue;
    }
    *queue = elt;

    //只有队列是fd才会把把事件加入事件模块里
    if (*queue == fpm_event_queue_fd && module->add) {
        module->add(ev);
    }          
               
    return 0;  
} 

int fpm_event_add(struct fpm_event_s *ev, unsigned long int frequency) /* {{{ */
{
    struct timeval now;
    struct timeval tmp;

    if (!ev) {
        return -1;
    }

    ev->index = -1;

    //只有标识为只读取事件
    if (ev->flags & FPM_EV_READ) {
        ev->which = FPM_EV_READ;
        //添加fd队列
        if (fpm_event_queue_add(&fpm_event_queue_fd, ev) != 0) {
            return -1;
        }
        return 0;
    }
    
    //时间队列
    ev->which = FPM_EV_TIMEOUT;
    
    //获取当前的时钟
    fpm_clock_get(&now);
    //如果间隔时间大于1000，将间隔时间当成毫妙计算
    if (frequency >= 1000) {
        tmp.tv_sec = frequency / 1000;
        tmp.tv_usec = (frequency % 1000) * 1000;
    } else {
        //秒
        tmp.tv_sec = 0;
        tmp.tv_usec = frequency * 1000;
    }
    ev->frequency = tmp;
    //设置下一个触发时间点
    fpm_event_set_timeout(ev, now);
    
    //添加到定时器里
    if (fpm_event_queue_add(&fpm_event_queue_timer, ev) != 0) {
        return -1;
    }
    
    return 0;
}
```

#### 定时器原理

定时器结构是双向列表，添加事件的时候会根据frequency间隔时间来来计算触发时间，
timeout计算就是当前时钟加上间隔时间，取出定时器列队里将要发现的触发时间，将其值设置成超时时间，如果wait有返回值，将会在定时器队列取出超时的，然后调用回调用函数，执行相应的信号、进程状态及记分器等。

```c
//事件循环
void fpm_event_loop(int err) /* {{{ */
{
    //信号事件
    static struct fpm_event_s signal_fd_event;

    //必须是主进程
    if (fpm_globals.parent_pid != getpid()) {
        return;
    }

    //信号接收事件，主要是master之间通信,通过socketpair建立
    fpm_event_set(&signal_fd_event, fpm_signals_get_fd(), FPM_EV_READ, &fpm_got_signal, NULL);
    fpm_event_add(&signal_fd_event, 0);

    //添加心跳事件,主要检查慢日子
    if (fpm_globals.heartbeat > 0) {
        fpm_pctl_heartbeat(NULL, 0, NULL);
    }

    if (!err) {
        //主要worker进程动态管理，根据用户配置的进程模式，比如动态、静态、相应，另外一个就是更新记分板的统计数据
        fpm_pctl_perform_idle_server_maintenance_heartbeat(NULL, 0, NULL);                                                                   

        zlog(ZLOG_DEBUG, "%zu bytes have been reserved in SHM", fpm_shm_get_size_allocated());                                               
        zlog(ZLOG_NOTICE, "ready to handle connections");

#ifdef HAVE_SYSTEMD           
        fpm_systemd_heartbeat(NULL, 0, NULL);
#endif
    }

    while (1) {
        struct fpm_event_queue_s *q, *q2;
        struct timeval ms;    
        struct timeval tmp;   
        struct timeval now;   
        unsigned long int timeout;     
        int ret;

        //必须是主进程
        if (fpm_globals.parent_pid != getpid()) {
            return;
        }

        //时钟
        fpm_clock_get(&now);  
        timerclear(&ms);      

        //查找需要发生的超时，就是timeout为最小值                                                                                            
        q = fpm_event_queue_timer;     
        while (q) {
            if (!timerisset(&ms)) {    
                ms = q->ev->timeout;       
            } else {          
                if (timercmp(&q->ev->timeout, &ms, <)) {                                                                                     
                    ms = q->ev->timeout;       
                }
            }
            q = q->next;      
        }

        //如果没有设置或者没有超时，默认为1秒                                                                                                
        if (!timerisset(&ms) || timercmp(&ms, &now, <) || timercmp(&ms, &now, ==)) {                                                         
            //毫秒            
            timeout = 1000;   
        } else {
            timersub(&ms, &now, &tmp); 
            //转换成毫秒
            timeout = (tmp.tv_sec * 1000) + (tmp.tv_usec / 1000) + 1;                                                                        
        }

        //等待事件发生        
        ret = module->wait(fpm_event_queue_fd, timeout);                                                                                     

        /* is a child, nothing to do here */                                                                                                 
        if (ret == -2) {
            return;
        }

        if (ret > 0) {        
            zlog(ZLOG_DEBUG, "event module triggered %d events", ret);                                                                       
        }

        /* trigger timers */  
        q = fpm_event_queue_timer;     
        while (q) {
            fpm_clock_get(&now);       
            if (q->ev) {      
                //触发事件    
                if (timercmp(&now, &q->ev->timeout, >) || timercmp(&now, &q->ev->timeout, ==)) {                                             
                    //超时函数回调             
                    fpm_event_fire(q->ev);         
                    /* sanity check */             
                    if (fpm_globals.parent_pid != getpid()) {
                        return;                    
                    }         
                    if (q->ev->flags & FPM_EV_PERSIST) {                                                                                     
                        fpm_event_set_timeout(q->ev, now);
                    } else { /* delete the event */    
                        q2 = q;                    
                        if (q->prev) {                 
                            q->prev->next = q->next;   
                        }     
                        if (q->next) {                 
                            q->next->prev = q->prev;   
                        }
                        if (q == fpm_event_queue_timer) {                                                                                    
                            fpm_event_queue_timer = q->next; 
                            if (fpm_event_queue_timer) {   
                                fpm_event_queue_timer->prev = NULL;
                            } 
                        }     
                        q = q->next;                   
                        free(q2);                      
                        continue;                      
                    }
                }             
            }
            q = q->next;      
        }
    }
}
```
