# fpm_cleanup

清理函数注册

清理函数类型，这里主要用到移位运算，1<<number 就是2^number。 

FPM_CLEANUP_CHILD & FPM_CLEANUP_PARENT_EXIT 标识判断
FPM_CLEANUP_CHILD | FPM_CLEANUP_PARENT_EXIT 加入新的标识
FPM_CLEANUP_ALL = ~0 包含所有类型

```c
enum {
    FPM_CLEANUP_CHILD                   = (1 << 0), 
    FPM_CLEANUP_PARENT_EXIT             = (1 << 1), 
    FPM_CLEANUP_PARENT_EXIT_MAIN        = (1 << 2), 
    FPM_CLEANUP_PARENT_EXEC             = (1 << 3), 
    FPM_CLEANUP_PARENT                  = (1 << 1) | (1 << 2) | (1 << 3), 
    FPM_CLEANUP_ALL                     = ~0, 
};

```

fpm_cleanups_run 调用清理函数

```c
void fpm_cleanups_run(int type) /* {{{ */
{
    //获取数组最后一位
    struct cleanup_s *c = fpm_array_item_last(&cleanups);
    //已使用
    int cl = cleanups.used;

    //c-- 偏移量
    for ( ; cl--; c--) {
        //类型判断
        if (c->type & type) {
            //调用清理函数
            c->cleanup(type, c->arg);
        }   
    }   
    //释放数组
    fpm_array_free(&cleanups);
}
```


