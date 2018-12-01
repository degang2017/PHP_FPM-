# fpm.c

php-fpm 运行还初始化都在此代码执行

先看一下fpm.h文件


程序退出的时候，如果HAVE_SYSEXITS_H存在，将使用sysexits定义的宏。

可以查看/usr/include/sysexits.h文件

#### 退出状态说明


| 名称 | 说明  |
| --- | --- |
| FPM_EXIT_OK |  成功退出状态为0 |
|  FPM_EXIT_USAGE | 该命令使用不正确，例如使用错误的参数数量，错误标志及语法等 |
|FPM_EXIT_SOFTWARE| 检测到内部软件错误|
|FPM_EXIT_CONFIG|配置错误|

#### fpm_globals_s 结构体说明

```c
struct fpm_globals_s {
    //master进程id
    pid_t parent_pid;
    //参数数量
    int argc;
    //参数值
    char **argv;
    //配置文件路径
    char *config;
    //目录前缀,比如获取配置文件，会拼接前缀，一般在fpm_evaluate_full_path函数用到
    char *prefix;
    //进程id
    char *pid;
    //子进程是否运行
    int running_children;
    //错误日志文件描述符
    int error_log_fd;
    //日志错误级别,记录日志的等级，默认notice，可取值alert, error, warning, notice, debug
    int log_level;
    //监听的socket
    int listening_socket; /* for this child */
    //最大请求数
    int max_requests; /* for this child */
    //是否是子进程
    int is_child;
    //测试
    int test_successful;
    //心跳
    int heartbeat;
    //是否已root用户运行
    int run_as_root;
    //即使强制输出错误，php-fpm启动的时候可以添加-O, 即使在非tty下也要输出错误
    int force_stderr;
    //发送配置管道,fd[0]读，f[1]写
    int send_config_pipe[2];
};


```

#### 头文件说明

| 名称 | 说明  |
| --- | --- |
|fpm_children.h| 创建子进程|
|fpm_signals.h| 信号处理|
|fpm_env.h| 环境变量|
|fpm_events.h| 事件：定义触发器，IO模型|
|fpm_cleanup.h| 清理操作|
|fpm_sockets.h| socket 操作|
|fpm_unix.h| 初始化unix环境权限、socket等|
|fpm_process_ctl.h|进程管理|
|fpm_conf.h| 配置文件解析|
|fpm_worker_pool.h|进程池|
|fpm_scoreboard.h| 记分器例如cpu、内存等|
|fpm_stdio.h| io操作|
|fpm_log|log操作|


fpm_globals结构初始化，参数、配置文件、前缀、是否root运行、强制输出错误等。

#### 初始流程

```c
  if (0 > fpm_php_init_main()  //注册清理事件，php模块关闭：php_module_shutdown() 。sapi_shutdown() 清理sapi_globals全局变量         ||  
        0 > fpm_stdio_init_main() //初始化stdio,主要确认open,dup2函数是否可执行        ||  
        0 > fpm_conf_init_main(test_conf, force_daemon)  //初始配置文件php-fpm.conf||
        0 > fpm_unix_init_main()   //设置文件描述符大小，core文件大小 ，启动守护进程      ||  
        0 > fpm_scoreboard_init_main()  //记分器初始化  ||  
        0 > fpm_pctl_init_main()   //初始化进程操作，argv参数，注册清理函数fpm_pctl_cleanup()       ||  
        0 > fpm_env_init_main()      //环境变量     ||  
        0 > fpm_signals_init_main()  //注册信号，比如SIGINT,SIGUSR1,SIGUSR2等     ||  
        0 > fpm_children_init_main()  //初始化子进程，注册清理函数fpm_children_cleanup()    ||  
        0 > fpm_sockets_init_main()  //初始化socket     ||  
        0 > fpm_worker_pool_init_main()  //注册进程池清理函数fpm_worker_pool_cleanup()  ||  
        0 > fpm_event_init_main()) //初始化事件，注册清理函数fpm_enent_cleanup() {

        //如果测试，直接退出
        if (fpm_globals.test_successful) {
            exit(FPM_EXIT_OK);
        } else {
            zlog(ZLOG_ERROR, "FPM initialization failed");
            return -1; 
        }   
    }  
    
    //将pid写入pid文件里
    if (0 > fpm_conf_write_pid()) {
        zlog(ZLOG_ERROR, "FPM initialization failed");
        return -1;
    }            

```

fpm_run 创建子进程，如果是父进程，循环事件监听，子进程的话获取监听的socket。




