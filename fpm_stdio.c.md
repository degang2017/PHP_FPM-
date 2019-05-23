# fpm stdio

```c

//确认/dev/null、STDIN_FILENO及STDOUT_FILENO文件描述符是否可用
int fpm_stdio_init_main() /* {{{ */
{
    //打开空设备
	int fd = open("/dev/null", O_RDWR);

	if (0 > fd) {
		zlog(ZLOG_SYSERROR, "failed to init stdio: open(\"/dev/null\")");
		return -1;
	}

    //复制文件描述符
	if (0 > dup2(fd, STDIN_FILENO) || 0 > dup2(fd, STDOUT_FILENO)) {
		zlog(ZLOG_SYSERROR, "failed to init stdio: dup2()");
		close(fd);
		return -1;
	}
    //关闭文件描述符
	close(fd);
	return 0;
}
/* }}} */

//使用错误日志
static inline int fpm_use_error_log() {  /* {{{ */
	/*
	 * the error_log is NOT used when running in foreground
	 * and from a tty (user looking at output).
	 * So, error_log is used by
	 * - SysV init launch php-fpm as a daemon
	 * - Systemd launch php-fpm in foreground
	 */
#if HAVE_UNISTD_H
    //如果是守护进程或者非终端或者强制错误信息写入,将使用错误日志
	if (fpm_global_config.daemonize || (!isatty(STDERR_FILENO) && !fpm_globals.force_stderr)) {
#else
	if (fpm_global_config.daemonize) {
#endif
		return 1;
	}
	return 0;
}

/* }}} */
int fpm_stdio_init_final() /* {{{ */
{
    //如果使用错误日志
	if (fpm_use_error_log()) {
        //错误日志不是标准错误日志文件描述符
		if (fpm_globals.error_log_fd > 0 && fpm_globals.error_log_fd != STDERR_FILENO) {

			/* there might be messages to stderr from other parts of the code, we need to log them all */
            //将error_log_fd文件描述符复制到标签错误文件描述符里
			if (0 > dup2(fpm_globals.error_log_fd, STDERR_FILENO)) {
				zlog(ZLOG_SYSERROR, "failed to init stdio: dup2()");
				return -1;
			}
		}
#ifdef HAVE_SYSLOG_H
		else if (fpm_globals.error_log_fd == ZLOG_SYSLOG) {
			/* dup to /dev/null when using syslog */
			dup2(STDOUT_FILENO, STDERR_FILENO);
		}
#endif
	}
	zlog_set_launched();
	return 0;
}
/* }}} */

int fpm_stdio_init_child(struct fpm_worker_pool_s *wp) /* {{{ */
{
#ifdef HAVE_SYSLOG_H
    //确认关闭系统日志，不要使用Php syslog日志
	if (fpm_globals.error_log_fd == ZLOG_SYSLOG) {
		//确保关闭Syslog，不要用php Syslog
		closelog(); 
	} else
#endif

	/* Notice: child cannot use master error_log
	 * because not aware when being reopen
	 * else, should use if (!fpm_use_error_log())
	 */
	if (fpm_globals.error_log_fd > 0) {
		close(fpm_globals.error_log_fd);
	}
	fpm_globals.error_log_fd = -1;
	zlog_set_fd(-1);

	return 0;
}
/* }}} */

//子进程管道通信
static void fpm_stdio_child_said(struct fpm_event_s *ev, short which, void *arg) /* {{{ */
{
	static const int max_buf_size = 1024;
	int fd = ev->fd;
	char buf[max_buf_size];
	struct fpm_child_s *child;
	int is_stdout;
	struct fpm_event_s *event;
	int fifo_in = 1, fifo_out = 1;
	int is_last_message = 0;
	int in_buf = 0;
	int res;

	if (!arg) {
		return;
	}
	//子进程
	child = (struct fpm_child_s *)arg;
	//是否有输出
	is_stdout = (fd == child->fd_stdout);
	if (is_stdout) {
		event = &child->ev_stdout;
	} else {
		//标准错误
		event = &child->ev_stderr;
	}

	while (fifo_in || fifo_out) {
		//读取
		if (fifo_in) {
			res = read(fd, buf + in_buf, max_buf_size - 1 - in_buf);
			if (res <= 0) { /* no data */
				fifo_in = 0;
				if (res < 0 && (errno == EAGAIN || errno == EWOULDBLOCK)) {
					/* just no more data ready */
				} else { /* error or pipe is closed */

					if (res < 0) { /* error */
						zlog(ZLOG_SYSERROR, "unable to read what child say");
					}

					fpm_event_del(event);
					is_last_message = 1;

					if (is_stdout) {
						close(child->fd_stdout);
						child->fd_stdout = -1;
					} else {
						close(child->fd_stderr);
						child->fd_stderr = -1;
					}
				}
			} else {
				in_buf += res;
			}
		}

		if (fifo_out) {
			if (in_buf == 0) {
				fifo_out = 0;
			} else {
				char *nl;
				int should_print = 0;
				buf[in_buf] = '\0';

				/* FIXME: there might be binary data */

				/* we should print if no more space in the buffer */
				if (in_buf == max_buf_size - 1) {
					should_print = 1;
				}

				/* we should print if no more data to come */
				if (!fifo_in) {
					should_print = 1;
				}

				//如果有换行，先打印换行
				nl = strchr(buf, '\n');
				if (nl || should_print) {

					if (nl) {
						*nl = '\0';
					}

					zlog(ZLOG_WARNING, "[pool %s] child %d said into %s: \"%s\"%s", child->wp->config->name,
					  (int) child->pid, is_stdout ? "stdout" : "stderr", buf, is_last_message ? ", pipe is closed" : "");

					if (nl) {
						int out_buf = 1 + nl - buf;
						memmove(buf, buf + out_buf, in_buf - out_buf);
						in_buf -= out_buf;
					} else {
						in_buf = 0;
					}
				}
			}
		}
	}
}
/* }}} */

int fpm_stdio_prepare_pipes(struct fpm_child_s *child) /* {{{ */
{
	//是否捕获子进程标准输出
	if (0 == child->wp->config->catch_workers_output) { /* not required */
		return 0;
	}

	//打开管道
	if (0 > pipe(fd_stdout)) {
		zlog(ZLOG_SYSERROR, "failed to prepare the stdout pipe");
		return -1;
	}

	if (0 > pipe(fd_stderr)) {
		zlog(ZLOG_SYSERROR, "failed to prepare the stderr pipe");
		close(fd_stdout[0]);
		close(fd_stdout[1]);
		return -1;
	}

	if (0 > fd_set_blocked(fd_stdout[0], 0) || 0 > fd_set_blocked(fd_stderr[0], 0)) {
		zlog(ZLOG_SYSERROR, "failed to unblock pipes");
		close(fd_stdout[0]);
		close(fd_stdout[1]);
		close(fd_stderr[0]);
		close(fd_stderr[1]);
		return -1;
	}
	return 0;
}
/* }}} */

int fpm_stdio_parent_use_pipes(struct fpm_child_s *child) /* {{{ */
{
	if (0 == child->wp->config->catch_workers_output) { /* not required */
		return 0;
	}

	close(fd_stdout[1]);
	close(fd_stderr[1]);

	child->fd_stdout = fd_stdout[0];
	child->fd_stderr = fd_stderr[0];

	//添加事件
	fpm_event_set(&child->ev_stdout, child->fd_stdout, FPM_EV_READ, fpm_stdio_child_said, child);
	fpm_event_add(&child->ev_stdout, 0);

	fpm_event_set(&child->ev_stderr, child->fd_stderr, FPM_EV_READ, fpm_stdio_child_said, child);
	fpm_event_add(&child->ev_stderr, 0);
	return 0;
}
/* }}} */

//禁止管道
int fpm_stdio_discard_pipes(struct fpm_child_s *child) /* {{{ */
{
	if (0 == child->wp->config->catch_workers_output) { /* not required */
		return 0;
	}

	close(fd_stdout[1]);
	close(fd_stderr[1]);

	close(fd_stdout[0]);
	close(fd_stderr[0]);
	return 0;
}
/* }}} */

//子进程使用管道
void fpm_stdio_child_use_pipes(struct fpm_child_s *child) /* {{{ */
{
	if (child->wp->config->catch_workers_output) {
		dup2(fd_stdout[1], STDOUT_FILENO);
		dup2(fd_stderr[1], STDERR_FILENO);w
		close(fd_stdout[0]); close(fd_stdout[1]);
		close(fd_stderr[0]); close(fd_stderr[1]);
	} else {
		/* stdout of parent is always /dev/null */
		dup2(STDOUT_FILENO, STDERR_FILENO);
	}
}
/* }}} */

//打开系统错误日志
int fpm_stdio_open_error_log(int reopen) /* {{{ */
{
	int fd;

#ifdef HAVE_SYSLOG_H
	if (!strcasecmp(fpm_global_config.error_log, "syslog")) {
		openlog(fpm_global_config.syslog_ident, LOG_PID | LOG_CONS, fpm_global_config.syslog_facility);
		fpm_globals.error_log_fd = ZLOG_SYSLOG;
		if (fpm_use_error_log()) {
			zlog_set_fd(fpm_globals.error_log_fd);
		}
		return 0;
	}
#endif

	fd = open(fpm_global_config.error_log, O_WRONLY | O_APPEND | O_CREAT, S_IRUSR | S_IWUSR);
	if (0 > fd) {
		zlog(ZLOG_SYSERROR, "failed to open error_log (%s)", fpm_global_config.error_log);
		return -1;
	}

	if (reopen) {
		if (fpm_use_error_log()) {
			dup2(fd, STDERR_FILENO);
		}

		dup2(fd, fpm_globals.error_log_fd);
		close(fd);
		fd = fpm_globals.error_log_fd; /* for FD_CLOSEXEC to work */
	} else {
		fpm_globals.error_log_fd = fd;
		if (fpm_use_error_log()) {
			zlog_set_fd(fpm_globals.error_log_fd);
		}
	}
	if (0 > fcntl(fd, F_SETFD, fcntl(fd, F_GETFD) | FD_CLOEXEC)) {
		zlog(ZLOG_WARNING, "failed to change attribute of error_log");
	}
	return 0;
} 
```


