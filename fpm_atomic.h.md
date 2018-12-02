# fpm_atomic

atomic 原子操作

atomic_cmp_set() 设置原子值

fpm_spinlock()实现一个自旋锁

这里主讲一下( __amd64__ || __amd64 || __x86_64__ ) 这些平台实现

__asm__ 内嵌汇编用法

__asm__　__volatile__("Instruction List" : Output : Input : Clobber/Modify);

``` c
__asm__ volatile ( "lock;" "cmpxchgq %3, %1;" "sete %0;" :    
"=a" (res) : "m" (*lock), "a" (old), "r" (set) : "memory");

指令模块对应的关系 
%3 (set)
%1 (*lock)
%0 (res)

cmpxchgq 比较将交换操作数, 8字节（64位）比较交换

lock 指令会使用cpu LOCK# 信号，保证多处理系统竟争下互斥使用此内存地址，指令执行完，锁会释放

r 将输入变量放入通用寄存器，也就是eax，ebx，ecx，edx，esi，edi中的一个
a 将输入变量放入eax
m 内存变量
= 操作数在指令中只是写（输出操作数）

memory 作用
1、不能将该段内嵌汇编指令与前面的指令重新排序；必须前面的指令都执行完
2、不能将变量缓存到寄存器，因为这段代码可能会用到内存变量
如果汇编指令修改了内存，但是GCC本身不知道，国为输出部分没有描述，此时需要增加memory

```

#### 自旋锁

```c
static inline int fpm_spinlock(atomic_t *lock, int try_once) /* {{{ */
{
    //尝试获取一次锁
    if (try_once) {
        //设置锁的值为1
        return atomic_cmp_set(lock, 0, 1) ? 1 : 0;
    }   

    //循环获取锁，直到成功为止
    for (;;) {
        //设置自旋
        if (atomic_cmp_set(lock, 0, 1)) {
            break;
        }   
        
        //让出cpu，等待下次调度,竞争比较激烈的时候，通过给其他的线程或进程运行机会的方式来提升程序的性能     
        sched_yield();
    }   

    return 1;
}

//释放锁
#define fpm_unlock(lock) lock = 0

```


