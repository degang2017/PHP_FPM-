# fpm shm

void *mmap(void *start,size_t length,int prot,int flags,int fd,off_t offsize); 

参数说明：

| 参数 | 说明 |
| --- | --- |
| start | 射的内存起始地址，通常设为0，代表让系统自动选定地址，映射成功后返回该地址|
|length|将文件中多大的部分映射到内存|
|prot|映射区域保护方式，可以分为4种方式的组合：PROT_EXEC、PROT_READ、PROT_WRITE及PROT_NONE|
|flags|影响映射区域的各种特性|
|fd|要映射到内存中的文件描述符。如果使用匿名内存映射时，即flags中设置了MAP_ANONYMOUS，fd设为-1。有些系统不支持匿名内存映射，则可以使用fopen打开/dev/zero文件，然后对该文件进行映射，可以同样达到匿名内存映射的效果|
|offset|文件映射的偏移量，通常设置为0，代表从文件最前方开始对应，offset必须是PAGE_SIZE的整数倍

#### flags

| 参数 | 说明|
| --- | --- |
|MAP_FIXED|如果参数start所指的地址无法成功建立映射时，则放弃映射，不对地址做修正。通常不鼓励用此旗标|
|MAP_SHARED|对映射区域的写入数据会复制回文件内，而且允许其他映射该文件的进程共享|
|MAP_PRIVATE|对映射区域的写入操作会产生一个映射文件的复制，即私的"写入时复制"（copy on write）对此区域作的任何修改都不会写回原来的文件内容|
|MAP_ANONYMOUS|建立匿名映射。此时会忽略参数fd，不涉及文件，而且映射区域无法和其他进程共享|
|MAP_DENYWRITE|只允许对映射区域的写入操作，其他对文件直接写入的操作将会被拒绝|
|MAP_LOCKED|将映射区域锁定住，这表示该区域不会被置换（swap）|

#### 返回值
若映射成功则返回映射区的内存起始地址，否则返回MAP_FAILED(－1)，错误原因存于errno 中。

错误代码：
EBADF  参数fd 不是有效的文件描述词
EACCES 存取权限有误。如果是MAP_PRIVATE 情况下文件必须可读，使用MAP_SHARED则要有PROT_WRITE以及该文件要能写入。
EINVAL 参数start、length 或offset有一个不合法。
EAGAIN 文件被锁住，或是有太多内存被锁住。
ENOMEM 内存不足。



```c
static size_t fpm_shm_size = 0;

void *fpm_shm_alloc(size_t size) /* {{{ */
{
    void *mem;

    //创建匿名映射，允许其他映射该文件的进程共享，此映射区域可读可写
    mem = mmap(0, size, PROT_READ | PROT_WRITE, MAP_ANONYMOUS | MAP_SHARED, -1, 0); 

#ifdef MAP_FAILED
    if (mem == MAP_FAILED) {
        zlog(ZLOG_SYSERROR, "unable to allocate %zu bytes in shared memory: %s", size, strerror(errno));
        return NULL;
    }   
#endif

    if (!mem) {
        zlog(ZLOG_SYSERROR, "unable to allocate %zu bytes in shared memory", size);
        return NULL;
    }   

    //内存映射大小
    fpm_shm_size += size;
    return mem;
}
/* }}} */

int fpm_shm_free(void *mem, size_t size) /* {{{ */
{
    if (!mem) {
        zlog(ZLOG_ERROR, "mem is NULL");
        return 0;
    }   

    //解除映射关系
    if (munmap(mem, size) == -1) {
        zlog(ZLOG_SYSERROR, "Unable to free shm");
        return 0;
    }   

    if (fpm_shm_size - size > 0) {
        fpm_shm_size -= size;
    } else {
        fpm_shm_size = 0;
    }   

    return 1;
}
/* }}} */
```


