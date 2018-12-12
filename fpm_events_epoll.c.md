# epoll 多路复用IO

它能减少程序在大量并发连接中只有少量活跃的情况下的系统CPU及缓存利用率，因为它不会复用文件描述符集合来传递结果而迫使开发者每次等待事件之前都必须重新准备要被侦听的文件描述符集合，另一点原因就是获取事件的时候，它无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入准备队列的描述符集合。epoll还支持水平触发（Level Triggered）外，还提供了边缘触发（Edge Triggered），这就使得用户空间程序有可能缓存IO状态，减少epoll_wait/epoll_pwait的调用，提高应用程序效率。

##### epoll 函数

##### epoll事件结构体

epoll_data *ptr 可以自定义数据，php-fpm的事件使用自定义数据


| events事件 | 说明  |
| --- | --- |
|EPOLLIN |表示对应的文件描述符可以读（包括对端SOCKET正常关闭）|
|EPOLLOUT|表示对应的文件描述符可以写|
|EPOLLPRI|表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）|
|EPOLLERR|表示对应的文件描述符发生错误|
|EPOLLHUP|表示对应的文件描述符被挂断|
|EPOLLET| 将EPOLL设为边缘触发(Edge Triggered)模式|
|EPOLLONESHOT|只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里|

```c
typedef union epoll_data {  
    void *ptr;  
    int fd;  
    uint32_t u32;  
    uint64_t u64;  
} epoll_data_t;
 
struct epoll_event {  
    uint32_t events;      // Epoll events  
    epoll_data_t data;      // User data variable  
};
```

```c
int epoll_create(int size);

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

//参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```


##### 事件增加、初始化及等待

```c

//事件模型
static struct fpm_event_module_s epoll_module = {
    .name = "epoll",
    //边缘触发
    .support_edge_trigger = 1,
    .init = fpm_event_epoll_init,
    .clean = fpm_event_epoll_clean,
    .wait = fpm_event_epoll_wait,
    .add = fpm_event_epoll_add,
    .remove = fpm_event_epoll_remove,
};

//epoll文件描述符集合
static struct epoll_event *epollfds = NULL;
//最大事件数
static int nepollfds = 0;
//epoll文件描述符
static int epollfd = -1;

#endif /* HAVE_EPOLL */

struct fpm_event_module_s *fpm_event_epoll_module() /* {{{ */
{
#if HAVE_EPOLL
    return &epoll_module;
#else
    return NULL;
#endif /* HAVE_EPOLL */
}
/* }}} */

#if HAVE_EPOLL

/*
 * Init the module
 */
static int fpm_event_epoll_init(int max) /* {{{ */
{
    if (max < 1) {
        return 0;
    }

    //初始化epoll
    epollfd = epoll_create(max + 1);
    if (epollfd < 0) {
        zlog(ZLOG_ERROR, "epoll: unable to initialize");
        return -1;
    }

    //分配文件描述符
    epollfds = malloc(sizeof(struct epoll_event) * max);
    if (!epollfds) {
        zlog(ZLOG_ERROR, "epoll: unable to allocate %d events", max);
        return -1;
    }
    memset(epollfds, 0, sizeof(struct epoll_event) * max);

    //保存最大值
    nepollfds = max;

    return 0;
}
/* }}} */

/*
 * wait for events or timeout
 */
static int fpm_event_epoll_wait(struct fpm_event_queue_s *queue, unsigned long int timeout) /* {{{ */
{
    int ret, i;

    //每次调用前清理一下epollfds数据
    memset(epollfds, 0, sizeof(struct epoll_event) * nepollfds);

    //事件等待
    ret = epoll_wait(epollfd, epollfds, nepollfds, timeout);
    if (ret == -1) {

        /* trigger error unless signal interrupt */
        if (errno != EINTR) {
            zlog(ZLOG_WARNING, "epoll_wait() returns %d", errno);
            return -1;
        }
    }

    //遍历返回事件
    for (i = 0; i < ret; i++) {

        //自定义数据是否为空
        if (!epollfds[i].data.ptr) {
            continue;
        }

        //回调函数，这里是用户注册时自定义数据
        fpm_event_fire((struct fpm_event_s *)epollfds[i].data.ptr);

        //子进程返回-2,子进程不需要时间触发器
        if (fpm_globals.parent_pid != getpid()) {
            return -2;
        }
    }

    return ret;
}
/* }}} */

/*
 * Add a FD to the fd set
 */
static int fpm_event_epoll_add(struct fpm_event_s *ev) /* {{{ */
{
    struct epoll_event e;

    /* fill epoll struct */
#if SIZEOF_SIZE_T == 4
    /* Completely initialize event data to prevent valgrind reports */
    e.data.u64 = 0;
#endif
    //这里data是个union,所以只使用e.data.ptr自定义数据
    e.events = EPOLLIN;
    e.data.fd = ev->fd;
    e.data.ptr = (void *)ev;

    //边缘触发
    if (ev->flags & FPM_EV_EDGE) {
        e.events = e.events | EPOLLET;
    }

    /* add the event to epoll internal queue */
    if (epoll_ctl(epollfd, EPOLL_CTL_ADD, ev->fd, &e) == -1) {
        zlog(ZLOG_ERROR, "epoll: unable to add fd %d", ev->fd);
        return -1;
    }

    //ev索引为文件描述符
    ev->index = ev->fd;
    return 0;
}
/* }}} */


```
