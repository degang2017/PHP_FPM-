# fpm log

这里主要是用户自定义日志格式，根据相应的变量进行解析，比如%{user}C, 先提取user，然后解析C。这里大部分信息都是通过记分器获取的。

日志格式说明

```c
;access.format = "%R - %u %t \"%m %r%Q%q\" %s %f %{mili}d %{kilo}M %C%%"

; The access log format.
; The following syntax is allowed
;  %%: the '%' character
;  %C: %CPU used by the request
;      it can accept the following format:
;      - %{user}C for user CPU only
;      - %{system}C for system CPU only
;      - %{total}C  for user + system CPU (default)
;  %d: time taken to serve the request
;      it can accept the following format:
;      - %{seconds}d (default)
;      - %{miliseconds}d
;      - %{mili}d
;      - %{microseconds}d
;      - %{micro}d
;  %e: an environment variable (same as $_ENV or $_SERVER)
;      it must be associated with embraces to specify the name of the env 
;      variable. Some exemples:
;      - server specifics like: %{REQUEST_METHOD}e or %{SERVER_PROTOCOL}e
;      - HTTP headers like: %{HTTP_HOST}e or %{HTTP_USER_AGENT}e
;  %f: script filename
;  %l: content-length of the request (for POST request only)
;  %m: request method
;  %M: peak of memory allocated by PHP 
;      it can accept the following format:
;      - %{bytes}M (default)
;      - %{kilobytes}M
;      - %{kilo}M
;      - %{megabytes}M
;      - %{mega}M
;  %n: pool name
;  %o: output header
;      it must be associated with embraces to specify the name of the header:
;      - %{Content-Type}o
;      - %{X-Powered-By}o
;      - %{Transfert-Encoding}o
;      - ....
;  %p: PID of the child that serviced the request
;  %P: PID of the parent of the child that serviced the request
;  %q: the query string
;  %Q: the '?' character if query string exists
;  %r: the request URI (without the query string, see %q and %Q) 
;  %R: remote IP address
;  %s: status (response code)
;  %t: server time the request was received
;      it can accept a strftime(3) format:
;      %d/%b/%Y:%H:%M:%S %z (default)
;      The strftime(3) format must be encapsuled in a %{<strftime_format>}t tag 
;      e.g. for a ISO8601 formatted timestring, use: %{%Y-%m-%dT%H:%M:%S%z}t
;  %T: time the log has been written (the request has finished)
;      it can accept a strftime(3) format:
;      %d/%b/%Y:%H:%M:%S %z (default)
;      The strftime(3) format must be encapsuled in a %{<strftime_format>}t tag 
;      e.g. for a ISO8601 formatted timestring, use: %{%Y-%m-%dT%H:%M:%S%z}t
;  %u: remote user

```

##### log 操作

```c
static char *fpm_log_format = NULL;
static int fpm_log_fd = -1; 

int fpm_log_open(int reopen) /* {{{ */
{
    struct fpm_worker_pool_s *wp;
    int ret = 1;

    int fd; 
    //循环遍历worker进程
    for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
        //没有配置访问日志文件跳过
        if (!wp->config->access_log) {
            continue;
        }   
        ret = 0;

        //打开文件，文件权限: 只写、追加、创建（文件不存在的则创建)及该文件所有都可写可读
        fd = open(wp->config->access_log, O_WRONLY | O_APPEND | O_CREAT, S_IRUSR | S_IWUSR);
        if (0 > fd) {
            zlog(ZLOG_SYSERROR, "failed to open access log (%s)", wp->config->access_log);
            return -1; 
        } else {
            zlog(ZLOG_DEBUG, "open access log (%s)", wp->config->access_log);
        }   

        //重新打开
        if (reopen) {
            //复制文件描述符
            dup2(fd, wp->log_fd);
            //关闭文件描述符
            close(fd);
            fd = wp->log_fd;
            //发送给子进程信号
            fpm_pctl_kill_all(SIGQUIT);
        } else {
            wp->log_fd = fd; 
        }   
        //FD_CLOSEXEC 使用exec执行的程序里，此描述符关闭，不能使用它，但是fork调用的子进程，不会关闭，可使用
        if (0 > fcntl(fd, F_SETFD, fcntl(fd, F_GETFD) | FD_CLOEXEC)) {
            zlog(ZLOG_WARNING, "failed to change attribute of access_log");
        }   
    }   

    return ret;
}
/* }}} */

/* }}} */
int fpm_log_init_child(struct fpm_worker_pool_s *wp)  /* {{{ */
{
    if (!wp || !wp->config) {
        return -1; 
    }   

    if (wp->config->access_log && *wp->config->access_log) {
        if (wp->config->access_format) {
            //log日志格式
            fpm_log_format = strdup(wp->config->access_format);
        }   
    }   

    if (fpm_log_fd == -1) {
        fpm_log_fd = wp->log_fd;
    }   


    for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
        //关闭其它的文件描述符，将其值设置为-1
        if (wp->log_fd > -1 && wp->log_fd != fpm_log_fd) {
            close(wp->log_fd);
            wp->log_fd = -1; 
        }   
    }   

    return 0;
}
/* }}} */


int fpm_log_write(char *log_format) /* {{{ */
{
    char *s, *b; 
    char buffer[FPM_LOG_BUFFER+1];
    int token, test;
    //len 解析字符串总长度，len2 单次解析字符串长度
    size_t len, len2;
    //进程记分器
    struct fpm_scoreboard_proc_s proc, *proc_p;
    //记分器
    struct fpm_scoreboard_s *scoreboard;
    //log 格式
    char tmp[129];
    char format[129];
    time_t now_epoch;
#ifdef HAVE_TIMES
    clock_t tms_total;
#endif

    if (!log_format && (!fpm_log_format || fpm_log_fd == -1)) {
        return -1;
    }

    if (!log_format) {
        log_format = fpm_log_format;
        test = 0;
    } else {
        test = 1;
    }

    //当前时间
    now_epoch = time(NULL);

    if (!test) {
        //获取记分器
        scoreboard = fpm_scoreboard_get();
        if (!scoreboard) {
            zlog(ZLOG_WARNING, "unable to get scoreboard while preparing the access log");
            return -1;
        }
        //获取当前进程记分器
        proc_p = fpm_scoreboard_proc_acquire(NULL, -1, 0);
        if (!proc_p) {
            zlog(ZLOG_WARNING, "[pool %s] Unable to acquire shm slot while preparing the access log", scoreboard->pool);
            return -1;
        }
        proc = *proc_p;
        //释放锁
        fpm_scoreboard_proc_release(proc_p);
    }

    token = 0;

    //buffer存放解析的log
    memset(buffer, '\0', sizeof(buffer));
    b = buffer;
    len = 0;


    s = log_format;

    while (*s != '\0') {
        /* Test is we have place for 1 more char. */
        if (len >= FPM_LOG_BUFFER) {
            zlog(ZLOG_NOTICE, "the log buffer is full (%d). The access log request has been truncated.", FPM_LOG_BUFFER);
            len = FPM_LOG_BUFFER;
            break;
        }

        if (!token && *s == '%') {
            //token标识
            token = 1;
            memset(format, '\0', sizeof(format)); /* reset format */
            s++;
            continue;
        }

        if (token) {
            token = 0;
            len2 = 0;
            switch (*s) {

                case '%': /* '%' */
                    *b = '%';
                    len2 = 1;
                    break;

#ifdef HAVE_TIMES
                case 'C': /* %CPU */
                    if (format[0] == '\0' || !strcasecmp(format, "total")) {
                        if (!test) {
                            //tms_utime: 记录进程执行用户代码时间
                            //tms_stime: 记录的是进程执行内核代码的时间
                            //tms_cutime: 记录的是子进程执行用户代码的时间
                            //tms_ustime: 记录的是子进程执行内核代码的时间
                            tms_total = proc.last_request_cpu.tms_utime + proc.last_request_cpu.tms_stime + proc.last_request_cpu.tms_cutime
+ proc.last_request_cpu.tms_cstime;
                        }
                    } else if (!strcasecmp(format, "user")) {
                        if (!test) {
                            tms_total = proc.last_request_cpu.tms_utime + proc.last_request_cpu.tms_cutime;
                        }
                    } else if (!strcasecmp(format, "system")) {
                        if (!test) {
                            tms_total = proc.last_request_cpu.tms_stime + proc.last_request_cpu.tms_cstime;
                        }
                    } else {
                        zlog(ZLOG_WARNING, "only 'total', 'user' or 'system' are allowed as a modifier for %%%c ('%s')", *s, format);
                        return -1;
                    }

                    //已解析的填充\0
                    format[0] = '\0';
                    if (!test) {
                        //计算cpu使用率 cpu总共执行时间/cpu周期/(接收请求后的时间)*100
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%.2f", tms_total / fpm_scoreboard_get_tick() / (proc.cpu_duration.tv_sec +
proc.cpu_duration.tv_usec / 1000000.) * 100.);
                    }
                    break;
#endif

                case 'd': /* duration µs */
                    //请求时间
                    /* seconds */
                    if (format[0] == '\0' || !strcasecmp(format, "seconds")) {
                        if (!test) {
                            len2 = snprintf(b, FPM_LOG_BUFFER - len, "%.3f", proc.duration.tv_sec + proc.duration.tv_usec / 1000000.);
                        }

                    /* miliseconds */
                    } else if (!strcasecmp(format, "miliseconds") || !strcasecmp(format, "mili")) {
                        if (!test) {
                            len2 = snprintf(b, FPM_LOG_BUFFER - len, "%.3f", proc.duration.tv_sec * 1000. + proc.duration.tv_usec / 1000.);
                        }

                    /* microseconds */
                    } else if (!strcasecmp(format, "microseconds") || !strcasecmp(format, "micro")) {
                        if (!test) {
                            len2 = snprintf(b, FPM_LOG_BUFFER - len, "%lu", proc.duration.tv_sec * 1000000UL + proc.duration.tv_usec);
                        }

                    } else {
                        zlog(ZLOG_WARNING, "only 'seconds', 'mili', 'miliseconds', 'micro' or 'microseconds' are allowed as a modifier for %%
%c ('%s')", *s, format);
                        return -1;
                    }
                    format[0] = '\0';
                    break;

                case 'e': /* fastcgi env  */
                    if (format[0] == '\0') {
                        zlog(ZLOG_WARNING, "the name of the environment variable must be set between embraces for %%%c", *s);
                        return -1;
                    }

                    if (!test) {
                        //获取fastcgi 环境变量
                        char *env = fcgi_getenv((fcgi_request*) SG(server_context), format, strlen(format));
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%s", env ? env : "-");
                    }
                    format[0] = '\0';
                    break;

                case 'f': /* script */
                    if (!test) {
                        //脚本名
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%s",  *proc.script_filename ? proc.script_filename : "-");
                    }
                    break;

                case 'l': /* content length */
                    if (!test) {
                        //内容长度
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%zu", proc.content_length);
                    }
                    break;

                case 'm': /* method */
                    if (!test) {
                        //request method
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%s", *proc.request_method ? proc.request_method : "-");
                    }
                    break;

                case 'M': /* memory */
                    //内存大小
                    /* seconds */
                    if (format[0] == '\0' || !strcasecmp(format, "bytes")) {
                        if (!test) {
                            len2 = snprintf(b, FPM_LOG_BUFFER - len, "%zu", proc.memory);
                        }

                    /* kilobytes */
                    } else if (!strcasecmp(format, "kilobytes") || !strcasecmp(format, "kilo")) {
                        if (!test) {
                            len2 = snprintf(b, FPM_LOG_BUFFER - len, "%zu", proc.memory / 1024);
                        }

                    /* megabytes */
                    } else if (!strcasecmp(format, "megabytes") || !strcasecmp(format, "mega")) {
                        if (!test) {
                            len2 = snprintf(b, FPM_LOG_BUFFER - len, "%zu", proc.memory / 1024 / 1024);
                        }

                    } else {
                        zlog(ZLOG_WARNING, "only 'bytes', 'kilo', 'kilobytes', 'mega' or 'megabytes' are allowed as a modifier for %%%c ('%s'
)", *s, format);
                        return -1;
                    }
                    format[0] = '\0';
                    break;

                case 'n': /* pool name */
                    if (!test) {
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%s", scoreboard->pool[0] ? scoreboard->pool : "-");
                    }
                    break;

                case 'o': /* header output  */
                    if (format[0] == '\0') {
                        zlog(ZLOG_WARNING, "the name of the header must be set between embraces for %%%c", *s);
                        return -1;
                    }
                    if (!test) {
                        sapi_header_struct *h;
                        zend_llist_position pos;
                        //获取sapi headers
                        sapi_headers_struct *sapi_headers = &SG(sapi_headers);
                        size_t format_len = strlen(format);

                        h = (sapi_header_struct*)zend_llist_get_first_ex(&sapi_headers->headers, &pos);
                        //循环遍历
                        while (h) {
                            char *header;
                            if (!h->header_len) {
                                h = (sapi_header_struct*)zend_llist_get_next_ex(&sapi_headers->headers, &pos);
                                continue;
                            }
                            //如果没有查找到继续遍历
                            if (!strstr(h->header, format)) {
                                h = (sapi_header_struct*)zend_llist_get_next_ex(&sapi_headers->headers, &pos);
                                continue;
                            }

                            /* test if enought char after the header name + ': ' */
                            if (h->header_len <= format_len + 2) {
                                h = (sapi_header_struct*)zend_llist_get_next_ex(&sapi_headers->headers, &pos);
                                continue;
                            }

                            if (h->header[format_len] != ':' || h->header[format_len + 1] != ' ') {
                                h = (sapi_header_struct*)zend_llist_get_next_ex(&sapi_headers->headers, &pos);
                                continue;
                            }

                            header = h->header + format_len + 2;
                            len2 = snprintf(b, FPM_LOG_BUFFER - len, "%s", header && *header ? header : "-");

                            /* found, done */
                            break;
                        }
                        if (!len2) {
                            len2 = 1;
                            *b = '-';
                        }
                    }
                    format[0] = '\0';
                    break;

                case 'p': /* PID */
                    if (!test) {
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%ld", (long)getpid());
                    }
                    break;

                case 'P': /* PID */
                    if (!test) {
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%ld", (long)getppid());
                    }
                    break;

                case 'q': /* query_string */
                    if (!test) {
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%s", proc.query_string);
                    }
                    break;

                case 'Q': /* '?' */
                    if (!test) {
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%s", *proc.query_string  ? "?" : "");
                    }
                    break;

                case 'r': /* request URI */
                    if (!test) {
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%s", proc.request_uri);
                    }
                    break;

                case 'R': /* remote IP address */
                    if (!test) {
                        const char *tmp = fcgi_get_last_client_ip();
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%s", tmp ? tmp : "-");
                    }
                    break;

                case 's': /* status */
                    if (!test) {
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%d", SG(sapi_headers).http_response_code);
                    }
                    break;

                case 'T':
                case 't': /* time */
                    if (!test) {
                        time_t *t;
                        if (*s == 't') {
                            t = &proc.accepted_epoch;
                        } else {
                            t = &now_epoch;
                        }
                        if (format[0] == '\0') {
                            strftime(tmp, sizeof(tmp) - 1, "%d/%b/%Y:%H:%M:%S %z", localtime(t));
                        } else {
                            strftime(tmp, sizeof(tmp) - 1, format, localtime(t));
                        }
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%s", tmp);
                    }
                    format[0] = '\0';
                    break;
                case 'u': /* remote user */
                    if (!test) {
                        len2 = snprintf(b, FPM_LOG_BUFFER - len, "%s", proc.auth_user);
                    }
                    break;

                case '{': /* complex var */
                    token = 1;
                    {
                        char *start;
                        size_t l;

                        start = ++s;

                        while (*s != '\0') {
                            if (*s == '}') {
                                //获取结束位置
                                l = s - start;

                                if (l >= sizeof(format) - 1) {
                                    l = sizeof(format) - 1;
                                }

                                //提取变量到format里
                                memcpy(format, start, l);
                                format[l] = '\0';
                                break;
                            }
                            s++;
                        }
                        if (s[1] == '\0') {
                            zlog(ZLOG_WARNING, "missing closing embrace in the access.format");
                            return -1;
                        }
                    }
                    break;

                default:
                    zlog(ZLOG_WARNING, "Invalid token in the access.format (%%%c)", *s);
                    return -1;
            }

            if (*s != '}' && format[0] != '\0') {
                zlog(ZLOG_WARNING, "embrace is not allowed for modifier %%%c", *s);
                return -1;
            }
            s++;
            if (!test) {
                //缓冲区偏移量+len2
                b += len2;
                len += len2;
            }
            if (len >= FPM_LOG_BUFFER) {
                zlog(ZLOG_NOTICE, "the log buffer is full (%d). The access log request has been truncated.", FPM_LOG_BUFFER);
                len = FPM_LOG_BUFFER;
                break;
            }
            continue;
        }

        if (!test) {
            // push the normal char to the output buffer
            *b = *s;
            b++;
            len++;
        }
        s++;
    }

    if (!test && strlen(buffer) > 0) {
        buffer[len] = '\n';
        //将内容写入log文件
        zend_quiet_write(fpm_log_fd, buffer, len + 1);
    }

    return 0;
}

```


