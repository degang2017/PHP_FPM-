# php-conf php配置文件解析


```c
//单向链表，存储key value,主要解析数组
struct key_value_s {
    struct key_value_s *next;
    char *key;
    char *value;
}
```

#### 全局配置

```c
struct fpm_global_config_s {  
    char *pid_file; //进程id文件地址           
    char *error_log; //错误日志地址
#ifdef HAVE_SYSLOG_H
    char *syslog_ident;  //系统日志，ident标识，一般设成程序名字
    int syslog_facility; //设备, user(1), local0~7预留，16~23给其他程序使用
#endif
    int log_level; //日志级别
    int emergency_restart_threshold; //紧急事件重启阀值
    int emergency_restart_interval; //紧急事件重新启动间隔
    int process_control_timeout; //设置子进程接受主进程复用信号的超时时间 
    int process_max; //进程最大数       
    int process_priority; //进程优先级，64是未设定
    int daemonize; //是否守护进程
    int rlimit_files; //设置文件打开描述符数
    int rlimit_core; //设置core文件大小
    char *events_mechanism; //设置事件，比如epoll,select等
#ifdef HAVE_SYSTEMD
    int systemd_watchdog; //是否启用看门狗, systemd 会经常性的向看门狗硬件发送信号；如果systemd或是系统本身挂起，这种信号就不会再有，看门狗就会导致自动重启
    int systemd_interval; //启动频率限制-1,不限制    
#endif
};
```

#### 进程池配置

```c
    struct fpm_worker_pool_config_s {
        char *name; //进程池名字
        char *prefix; //前缀
        char *user; //用户
        char *group; //组
        char *listen_address; //监听地址
        int listen_backlog; //监听队列大小
        //Using chown 
        char *listen_owner; //设置unix套接字权限，拥有者
        char *listen_group; //拥有组
        char *listen_mode;  //模式，比如0660
        char *listen_allowed_clients; //授权客户端地址，不填是所有，多个Ip用逗号分隔
        int process_priority; //进程优先级
        int process_dumpable; //设置进程标识，PR_SET_DUMPABLE
        int pm; //进程模式： static dynamic ondemand
        //静态：创建子进程数目
        //动态：根据请求数创建相应的进程，但是范围必须在pm.min_spare_servers和pm.max_spare_servers之间
        //按需：会把内存入在第一位，pm.process_idle_timeout空闲时间检查
        int pm_max_children; //最大子进程数
        int pm_start_servers; //启动创建子进程数，只有pm设置为dynamic时才使用，默认值 min_spare_servers + (max_spare_servers - min_spare_servers) / 2
        int pm_min_spare_servers; //动态方式空闲状态下最小的进程数
        int pm_max_spare_servers; //动态方式空闲状态下最大的进程数
        int pm_process_idle_timeout; //空闲进程超时时间
        int pm_max_requests; //最大请数求
        char *pm_status_path;  //可以通过url访问查看php-fpm当前进程的状态
        char *ping_path; //监控页面ping地址
        char *ping_response; //用户定义ping请求响应，默认值pong
        char *access_log; //访问日志地址
        char *access_format; //访问日志格式
        char *slowlog; //慢日志地址
        int request_slowlog_timeout; //慢日志超时时间
        int request_slowlog_trace_depth; //慢日志跟踪堆栈深度
        int request_terminate_timeout; //设置单个请求超时终止时间，该选项可能会对 php.ini 设置中的 max_execution_time
        //因为某些特殊原因没有中止运行的脚本有用。设置为 ‘0’ 表示 ‘Off’
        //可用单位：s（秒），m（分），h（小时）或者 d（天）。默认单位：s（秒）。默认值：0（关闭
        int rlimit_files; //设置文件描述符大小
        int rlimit_core; //设置core文件大小
        char *chroot; //设置php文件目录
        char *chdir;  //启动php-fpm会自动到该目录                   
        int catch_workers_output; //重定向运行过程中的stdout stderr到主要的错误日志文件中，如果没有设置，stdout stderr会根据fastCGI的规则被重定向/dev/null,默认值为空
        int clear_env; //清除fpm工作进程的环境变量，防止任意变量进入到辅助进程，值为no可以使用任意变量               
        char *security_limit_extensions; ///设置php解析脚本的后缀，防止恶意上传文件解析执行，可以是多个，用空格隔开: .php .php5
        struct key_value_s *env; //环境变量
        struct key_value_s *php_admin_values; //php admin值 
        struct key_value_s *php_values; //php values
        #ifdef HAVE_APPARMOR
        //AppArmor是Ubuntu，SuSE和其他Linux发行版上的默认强制访问控制模块
        //您可以限制进程的文件系统访问权限。已经有一些很好的方法可以为Php-fpm获取Apparmor的基本设置
        char *apparmor_hat;
        #endif
        #ifdef HAVE_FPM_ACL                  
        //Using Posix ACL 
        char *listen_acl_users; //posix访问控制，可以访问的用户
        char *listen_acl_groups;  //posix访问控制，可以访问的用户组
        #endif
    };     
    
```

#### 配置文件初始化

```c
int fpm_conf_init_main(int test_conf, int force_daemon) /* {{{ */
{
    int ret;

    //前缀
    if (fpm_globals.prefix && *fpm_globals.prefix) {
        if (!fpm_conf_is_dir(fpm_globals.prefix)) {
            zlog(ZLOG_ERROR, "the global prefix '%s' does not exist or is not a directory", fpm_globals.prefix);
            return -1;
        }
    }

    //进程pid
    if (fpm_globals.pid && *fpm_globals.pid) {
        fpm_global_config.pid_file = strdup(fpm_globals.pid);
    }

    //配置文件
    if (fpm_globals.config == NULL) {
        char *tmp;

        if (fpm_globals.prefix == NULL) {
            spprintf(&tmp, 0, "%s/php-fpm.conf", PHP_SYSCONFDIR);
        } else {
            spprintf(&tmp, 0, "%s/etc/php-fpm.conf", fpm_globals.prefix);
        }

        if (!tmp) {
            zlog(ZLOG_SYSERROR, "spprintf() failed (tmp for fpm_globals.config)");
            return -1;
        }

        fpm_globals.config = strdup(tmp);
        efree(tmp);

        if (!fpm_globals.config) {
            zlog(ZLOG_SYSERROR, "spprintf() failed (fpm_globals.config)");
            return -1;
        }
    }

    //加载配置文件
    ret = fpm_conf_load_ini_file(fpm_globals.config);

    if (0 > ret) {
        zlog(ZLOG_ERROR, "failed to load configuration file '%s'", fpm_globals.config);
        return -1;
    }

    //将配置文件写入进程
    if (0 > fpm_conf_post_process(force_daemon)) {
        zlog(ZLOG_ERROR, "failed to post process the configuration");
        return -1;
    }

    //测试配置文件是否有错误
    if (test_conf) {
        if (test_conf > 1) {
            fpm_conf_dump();
        }
        zlog(ZLOG_NOTICE, "configuration file %s test is successful\n", fpm_globals.config);
        fpm_globals.test_successful = 1;
        return -1;
    }

    //注册清理函数
    if (0 > fpm_cleanup_add(FPM_CLEANUP_ALL, fpm_conf_cleanup, 0)) {
        return -1;
    }

    return 0;
}
```

##### ini配置文件解析结构

```c
struct ini_value_parser_s {
    char *name; //名字
    char *(*parser)(zval *, void **, intptr_t); //解析处理函数
    intptr_t offset; //偏移量
};

```

获取结构体字段偏移量：GO() WPO()

fpm_conf_set_xxx 转换配置文件值的类型,有integer、time、boolean、string、log_level、rlimit_core、pm、syslog_facility

配置文件解析过程：读取php-fpm.conf文件内容--->注册fpm_conf_ini_parser函数--->处理entry-->section--->pop_entry--->加载include文件--->关闭文件描述符--->复制配置文件到进程--->添加清理函数

##### fpm_conf_load_ini_file()
 
 ```c
 int fpm_conf_load_ini_file(char *filename) /* {{{ */                                                                                         
{
    int error = 0;
    char *buf = NULL, *newbuf = NULL;
    int bufsize = 0;
    int fd, n;
    int nb_read = 1;
    char c = '*';

    int ret = 1;

    if (!filename || !filename[0]) {                                                                                                         
        zlog(ZLOG_ERROR, "configuration filename is empty");
        return -1;
    }
    //打开文件
    fd = open(filename, O_RDONLY, 0);                                                                                                        
    if (fd < 0) {
        zlog(ZLOG_SYSERROR, "failed to open configuration file '%s'", filename);                                                             
        return -1;
    }

    //include 最多不能超过5个
    if (ini_recursion++ > 4) {
        zlog(ZLOG_ERROR, "failed to include more than 5 files recusively");                                                                  
        close(fd);
        return -1;
    }

    ini_lineno = 0;
    while (nb_read > 0) {     
        int tmp;
        ini_lineno++;
        ini_filename = filename;       
        //读取文件内容，逐行读取
        for (n = 0; (nb_read = read(fd, &c, sizeof(char))) == sizeof(char) && c != '\n'; n++) { 
            //如果相等，内存不够添加1024                                             
            if (n == bufsize) {        
                bufsize += 1024;   
                //+2主要存储\n\0        
                newbuf = (char*) realloc(buf, sizeof(char) * (bufsize + 2));                                                                 
                if (newbuf == NULL) {          
                    ini_recursion--;           
                    close(fd);
                    free(buf);
                    return -1;
                }
                buf = newbuf; 
            }

            buf[n] = c;       
        }
        //读取失败的话，重新读取
        if (n == 0) {         
            continue;         
        }
        /* always append newline and null terminate */                                                                                       
        buf[n++] = '\n';      
        buf[n] = '\0';
        //ini 解析
        tmp = zend_parse_ini_string(buf, 1, ZEND_INI_SCANNER_NORMAL, (zend_ini_parser_cb_t)fpm_conf_ini_parser, &error);                     
        ini_filename = filename;       
        if (error || tmp == FAILURE) {
            if (ini_include) free(ini_include);
            ini_recursion--;  
            close(fd);        
            free(buf);        
            return -1;        
        }
        //ini文件如果有引入其它配置
        if (ini_include) {    
            char *tmp = ini_include;   
            ini_include = NULL;        
            fpm_evaluate_full_path(&tmp, NULL, NULL, 0);
            //获取include文件，如果是*的话，会获取所有匹配的文件
            fpm_conf_ini_parser_include(tmp, &error);
            if (error) {
                free(tmp);
                ini_recursion--;           
                close(fd);
                free(buf);
                return -1;
            }
            free(tmp);
        }
    }
    free(buf);

    //处理完ini_recursion减一
    ini_recursion--;
    //关闭文件描述符
    close(fd);
    return ret;
}
 
 ```   

##### fpm_conf_ini_parser ini文件解析

INI文件由节、键、值组成

节 [section]  

参数（键=值） name=value

  　　
##### ini文件有几种类型

| 名称 |说明  |
| --- | --- |
|ZEND_INI_PARSER_SECTION| [www] |
|ZEND_INI_PARSER_ENTRY| group=nobody |
|ZEND_INI_PARSER_POP_ENTRY |php_admin_value[sendmail_path] = /usr/sbin/sendmail php_admin_value[error_log] = /var/log/fpm-php.www.log|

```c
static void fpm_conf_ini_parser(zval *arg1, zval *arg2, zval *arg3, int callback_type, void *arg) /* {{{ */
{
    int *error;

    if (!arg1 || !arg) return;
    error = (int *)arg;
    if (*error) return; /* We got already an error. Switch to the end. */

    switch(callback_type) {
        case ZEND_INI_PARSER_ENTRY:
            fpm_conf_ini_parser_entry(arg1, arg2, error);
            break;
        case ZEND_INI_PARSER_SECTION:
            fpm_conf_ini_parser_section(arg1, error);
            break;
        case ZEND_INI_PARSER_POP_ENTRY:
            fpm_conf_ini_parser_array(arg1, arg3, arg2, error);
            break;
        default:
            zlog(ZLOG_ERROR, "[%s:%d] Unknown INI syntax", ini_filename, ini_lineno);
            *error = 1;
            break;
    }
}

static void fpm_conf_ini_parser_entry(zval *name, zval *value, void *arg) /* {{{ */
{
    //value 解析结构
    struct ini_value_parser_s *parser;
    //配置
    void *config = NULL;      

    int *error = (int *)arg;  
    if (!value) {
        zlog(ZLOG_ERROR, "[%s:%d] value is NULL for a ZEND_INI_PARSER_ENTRY", ini_filename, ini_lineno);
        *error = 1;
        return;
    }
    //如果value值包含include,需要引用include文件里的内容
    if (!strcmp(Z_STRVAL_P(name), "include")) {
        //include只能一个
        if (ini_include) {    
            zlog(ZLOG_ERROR, "[%s:%d] two includes at the same time !", ini_filename, ini_lineno);
            *error = 1;
            return;           
        }
        ini_include = strdup(Z_STRVAL_P(value));
        return;
    }

    //如果是[global]解析数组就是ini_fpm_global_options,否则是pool
    if (!current_wp) { /* we are in the global section */
        parser = ini_fpm_global_options;
        config = &fpm_global_config;   
    } else {
        parser = ini_fpm_pool_options; 
        config = current_wp->config;   
    }

    //遍历配置文件字段
    for (; parser->name; parser++) {
        if (!strcasecmp(parser->name, Z_STRVAL_P(name))) {
            char *ret;        
            if (!parser->parser) {         
                zlog(ZLOG_ERROR, "[%s:%d] the parser for entry '%s' is not defined", ini_filename, ini_lineno, parser->name);
                *error = 1;   
                return;
            }
            //解析value,转换成相应的类型，比如int timer
            ret = parser->parser(value, &config, parser->offset);                                                                            
            if (ret) {        
                zlog(ZLOG_ERROR, "[%s:%d] unable to parse value for entry '%s': %s", ini_filename, ini_lineno, parser->name, ret);           
                *error = 1;   
                return;
            }

            /* all is good ! */            
            return;           
        }
    }

    /* nothing has been found if we got here */                                                                                              
    zlog(ZLOG_ERROR, "[%s:%d] unknown entry '%s'", ini_filename, ini_lineno, Z_STRVAL_P(name));                                              
    *error = 1;
}

static void fpm_conf_ini_parser_section(zval *section, void *arg) /* {{{ */
{
    struct fpm_worker_pool_s *wp;
    struct fpm_worker_pool_config_s *config;
    int *error = (int *)arg;

    /* switch to global conf */
    if (!strcasecmp(Z_STRVAL_P(section), "global")) {
        current_wp = NULL;
        return;
    }
    //循环遍历，获取当前section的pool
    for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
        if (!wp->config) continue;
        if (!wp->config->name) continue;
        if (!strcasecmp(wp->config->name, Z_STRVAL_P(section))) {
            /* Found a wp with the same name. Bring it back */
            current_wp = wp;
            return;
        }
    }

    /* 创建新的pool */
    config = (struct fpm_worker_pool_config_s *)fpm_worker_pool_config_alloc();
    if (!current_wp || !config) {
        zlog(ZLOG_ERROR, "[%s:%d] Unable to alloc a new WorkerPool for worker '%s'", ini_filename, ini_lineno, Z_STRVAL_P(section));
        *error = 1;
        return;
    }
    config->name = strdup(Z_STRVAL_P(section));
    if (!config->name) {
        zlog(ZLOG_ERROR, "[%s:%d] Unable to alloc memory for configuration name for worker '%s'", ini_filename, ini_lineno, Z_STRVAL_P(sectio
n));
        *error = 1;
        return;
    }
}

```

##### 进程配置
```c
static int fpm_conf_post_process(int force_daemon) /* {{{ */
{
    struct fpm_worker_pool_s *wp;

    //pid文件实际路径
    if (fpm_global_config.pid_file) {
        fpm_evaluate_full_path(&fpm_global_config.pid_file, NULL, PHP_LOCALSTATEDIR, 0);
    }
    
    //守护进程
    if (force_daemon >= 0) {
        /* forced from command line options */
        fpm_global_config.daemonize = force_daemon;
    }
    //日志级别
    fpm_globals.log_level = fpm_global_config.log_level;
    zlog_set_level(fpm_globals.log_level);

    if (fpm_global_config.process_max < 0) {
        zlog(ZLOG_ERROR, "process_max can't be negative");
        return -1;
    }

    if (fpm_global_config.process_priority != 64 && (fpm_global_config.process_priority < -19 || fpm_global_config.process_priority > 20)) {
        zlog(ZLOG_ERROR, "process.priority must be included into [-19,20]");
        return -1;
    }

    if (!fpm_global_config.error_log) {
        fpm_global_config.error_log = strdup("log/php-fpm.log");
    }

#ifdef HAVE_SYSTEMD
    if (0 > fpm_systemd_conf()) {
        return -1;
    }
#endif 

#ifdef HAVE_SYSLOG_H 
    //系统log
    if (!fpm_global_config.syslog_ident) {
        fpm_global_config.syslog_ident = strdup("php-fpm");
    }
    
    if (fpm_global_config.syslog_facility < 0) {
        fpm_global_config.syslog_facility = LOG_DAEMON;
    }   
            
    if (strcasecmp(fpm_global_config.error_log, "syslog") != 0)
#endif  
    {
        fpm_evaluate_full_path(&fpm_global_config.error_log, NULL, PHP_LOCALSTATEDIR, 0);
    }   
            
    //打开error日志文件描述符
    if (0 > fpm_stdio_open_error_log(0)) {
        return -1;
    }   
    
    //IO事件初始化
    if (0 > fpm_event_pre_init(fpm_global_config.events_mechanism)) {
        return -1;
    }       
            
    //进程配置 
    if (0 > fpm_conf_process_all_pools()) {
        return -1;
    }   
        
    //打开log
    if (0 > fpm_log_open(0)) {
        return -1;
    }   
    
    for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
        if (!wp->config->access_log || !*wp->config->access_log) {
            continue;
        }   
        //写入访问日志
        if (0 > fpm_log_write(wp->config->access_format)) {
            zlog(ZLOG_ERROR, "[pool %s] wrong format for access.format '%s'", wp->config->name, wp->config->access_format);
            return -1; 
        }   
    }
    
    return 0;
}  
```


