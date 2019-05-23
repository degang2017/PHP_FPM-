# fpm 子进程

```c
static void fpm_children_cleanup(int which, void *arg) /* {{{ */
{
	free(last_faults);
}
/* }}} */

//申请子进程
static struct fpm_child_s *fpm_child_alloc() /* {{{ */
{
	struct fpm_child_s *ret;

	//申请子进程结构体内存
	ret = malloc(sizeof(struct fpm_child_s));

	if (!ret) {
		return 0;
	}

	memset(ret, 0, sizeof(*ret));
	//记分牌索引值
	ret->scoreboard_i = -1;
	return ret;
}
/* }}} */

//释放子进程
static void fpm_child_free(struct fpm_child_s *child) /* {{{ */
{
	free(child);
}
/* }}} */

//关闭子进程
static void fpm_child_close(struct fpm_child_s *child, int in_event_loop) /* {{{ */
{
	if (child->fd_stdout != -1) {
		if (in_event_loop) {
			//调用事件回调用函数
			fpm_event_fire(&child->ev_stdout);
		}
		if (child->fd_stdout != -1) {
			close(child->fd_stdout);
		}
	}

	if (child->fd_stderr != -1) {
		if (in_event_loop) {
			fpm_event_fire(&child->ev_stderr);
		}
		if (child->fd_stderr != -1) {
			close(child->fd_stderr);
		}
	}

	fpm_child_free(child);
}
/* }}} */

//链接子进程
static void fpm_child_link(struct fpm_child_s *child) /* {{{ */
{
	struct fpm_worker_pool_s *wp = child->wp;

	++wp->running_children;
	++fpm_globals.running_children;

	child->next = wp->children;
	if (child->next) {
		child->next->prev = child;
	}
	child->prev = 0;
	wp->children = child;
}
/* }}} */

//解锁子进程
static void fpm_child_unlink(struct fpm_child_s *child) /* {{{ */
{
	--child->wp->running_children;
	--fpm_globals.running_children;

	if (child->prev) {
		child->prev->next = child->next;
	} else {
		child->wp->children = child->next;
	}

	if (child->next) {
		child->next->prev = child->prev;
	}
}
/* }}} */

//查询子进程
static struct fpm_child_s *fpm_child_find(pid_t pid) /* {{{ */
{
	struct fpm_worker_pool_s *wp;
	struct fpm_child_s *child = 0;

	for (wp = fpm_worker_all_pools; wp; wp = wp->next) {

		for (child = wp->children; child; child = child->next) {
			if (child->pid == pid) {
				break;
			}
		}

		if (child) break;
	}

	if (!child) {
		return 0;
	}

	return child;
}
/* }}} */

//初始化子进程
static void fpm_child_init(struct fpm_worker_pool_s *wp) /* {{{ */
{
	fpm_globals.max_requests = wp->config->pm_max_requests;
	fpm_globals.listening_socket = dup(wp->listening_socket);

	if (0 > fpm_stdio_init_child(wp)  ||
	    0 > fpm_log_init_child(wp)    ||
	    0 > fpm_status_init_child(wp) ||
	    0 > fpm_unix_init_child(wp)   ||
	    0 > fpm_signals_init_child()  ||
	    0 > fpm_env_init_child(wp)    ||
	    0 > fpm_php_init_child(wp)) {

		zlog(ZLOG_ERROR, "[pool %s] child failed to initialize", wp->config->name);
		exit(FPM_EXIT_SOFTWARE);
	}
}
/* }}} */

int fpm_children_free(struct fpm_child_s *child) /* {{{ */
{
	struct fpm_child_s *next;

	for (; child; child = next) {
		next = child->next;
		fpm_child_close(child, 0 /* in_event_loop */);
	}

	return 0;
}
/* }}} */

//子进程退出
void fpm_children_bury() /* {{{ */
{
	int status;
	pid_t pid;
	struct fpm_child_s *child;

	//避免僵尸进程,非阻塞状态等待
	while ( (pid = waitpid(-1, &status, WNOHANG | WUNTRACED)) > 0) {
		char buf[128];
		int severity = ZLOG_NOTICE;
		int restart_child = 1;

		child = fpm_child_find(pid);

		//子进程是否为正常退出的
		if (WIFEXITED(status)) {

			snprintf(buf, sizeof(buf), "with code %d", WEXITSTATUS(status));

			/* if it's been killed because of dynamic process management
			 * don't restart it automaticaly
			 */
			//空闲子进程kill
			if (child && child->idle_kill) {
				restart_child = 0;
			}

			if (WEXITSTATUS(status) != FPM_EXIT_OK) {
				severity = ZLOG_WARNING;
			}

		} else if (WIFSIGNALED(status)) { //如果子进程是因为信号而结束
			const char *signame = fpm_signal_names[WTERMSIG(status)];
			const char *have_core = WCOREDUMP(status) ? " - core dumped" : "";

			if (signame == NULL) {
				signame = "";
			}

			snprintf(buf, sizeof(buf), "on signal %d (%s%s)", WTERMSIG(status), signame, have_core);

			/* if it's been killed because of dynamic process management
			 * don't restart it automaticaly
			 */
			//如果是正常通过信号退出并且是空闲进程kill
			if (child && child->idle_kill && WTERMSIG(status) == SIGQUIT) {
				restart_child = 0;
			}

			//可能是请求丢失
			if (WTERMSIG(status) != SIGQUIT) { 
				severity = ZLOG_WARNING;
			}
		} else if (WIFSTOPPED(status)) { //如果子进程处于暂停执行情况

			zlog(ZLOG_NOTICE, "child %d stopped for tracing", (int) pid);

			//进程tracer
			if (child && child->tracer) {
				child->tracer(child);
			}

			continue;
		}

		//子进程存在
		if (child) {
			struct fpm_worker_pool_s *wp = child->wp;
			struct timeval tv1, tv2;

			//取消关联链表
			fpm_child_unlink(child);

			//释放子进程记分牌
			fpm_scoreboard_proc_free(wp->scoreboard, child->scoreboard_i);

			//获取当前时钟
			fpm_clock_get(&tv1);

			//进程运行时间
			timersub(&tv1, &child->started, &tv2);

			//是否需要重新启动子进程
			if (restart_child) {
				if (!fpm_pctl_can_spawn_children()) {
					severity = ZLOG_DEBUG;
				}
				zlog(severity, "[pool %s] child %d exited %s after %ld.%06d seconds from start", child->wp->config->name, (int) pid, buf, tv2.tv_sec, (int) tv2.tv_usec);
			} else {
				zlog(ZLOG_DEBUG, "[pool %s] child %d has been killed by the process management after %ld.%06d seconds from start", child->wp->config->name, (int) pid, tv2.tv_sec, (int) tv2.tv_usec);
			}

			//关闭子进程
			fpm_child_close(child, 1 /* in event_loop */);

			//子进程退出
			fpm_pctl_child_exited();

			//有错误信号，内存错误信号
			if (last_faults && (WTERMSIG(status) == SIGSEGV || WTERMSIG(status) == SIGBUS)) {
				time_t now = tv1.tv_sec;
				//需要重新启动
				int restart_condition = 1;
				int i;

				last_faults[fault++] = now;

				//紧急重启的阀值
				if (fault == fpm_global_config.emergency_restart_threshold) {
					fault = 0;
				}

				for (i = 0; i < fpm_global_config.emergency_restart_threshold; i++) {
					//重新阀值间隔
					if (now - last_faults[i] > fpm_global_config.emergency_restart_interval) {
						restart_condition = 0;
						break;
					}
				}

				if (restart_condition) {

					zlog(ZLOG_WARNING, "failed processes threshold (%d in %d sec) is reached, initiating reload", fpm_global_config.emergency_restart_threshold, fpm_global_config.emergency_restart_interval);
					//需要reload
					fpm_pctl(FPM_PCTL_STATE_RELOADING, FPM_PCTL_ACTION_SET);
				}
			}

			if (restart_child) {
				//创建子进程
				fpm_children_make(wp, 1 /* in event loop */, 1, 0);

				if (fpm_globals.is_child) {
					break;
				}
			}
		} else {
			zlog(ZLOG_ALERT, "oops, unknown child (%d) exited %s. Please open a bug report (https://bugs.php.net).", pid, buf);
		}
	}
}
/* }}} */

//资源准备
static struct fpm_child_s *fpm_resources_prepare(struct fpm_worker_pool_s *wp) /* {{{ */
{
	struct fpm_child_s *c;

	//申请子进程内存
	c = fpm_child_alloc();

	if (!c) {
		zlog(ZLOG_ERROR, "[pool %s] unable to malloc new child", wp->config->name);
		return 0;
	}

	c->wp = wp;
	c->fd_stdout = -1; c->fd_stderr = -1;

	//管道初始化
	if (0 > fpm_stdio_prepare_pipes(c)) {
		fpm_child_free(c);
		return 0;
	}

	//申请子记分牌
	if (0 > fpm_scoreboard_proc_alloc(wp->scoreboard, &c->scoreboard_i)) {
		fpm_stdio_discard_pipes(c);
		fpm_child_free(c);
		return 0;
	}

	return c;
}
/* }}} */

//资源禁止
static void fpm_resources_discard(struct fpm_child_s *child) /* {{{ */
{
	fpm_scoreboard_proc_free(child->wp->scoreboard, child->scoreboard_i);
	fpm_stdio_discard_pipes(child);
	fpm_child_free(child);
}
/* }}} */

//子进程资源使用
static void fpm_child_resources_use(struct fpm_child_s *child) /* {{{ */
{
	struct fpm_worker_pool_s *wp;
	for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
		if (wp == child->wp) {
			continue;
		}
		fpm_scoreboard_free(wp->scoreboard);
	}

	//子进程记分牌使用
	fpm_scoreboard_child_use(child->wp->scoreboard, child->scoreboard_i, getpid());
	fpm_stdio_child_use_pipes(child);
	fpm_child_free(child);
}
/* }}} */

static void fpm_parent_resources_use(struct fpm_child_s *child) /* {{{ */
{
	//父进程资源使用
	fpm_stdio_parent_use_pipes(child);
	fpm_child_link(child);
}
/* }}} */

//创建子进程
int fpm_children_make(struct fpm_worker_pool_s *wp, int in_event_loop, int nb_to_spawn, int is_debug) /* {{{ */
{
	pid_t pid;
	struct fpm_child_s *child;
	int max;
	static int warned = 0;

	if (wp->config->pm == PM_STYLE_DYNAMIC) {
		if (!in_event_loop) { /* starting */
			max = wp->config->pm_start_servers;
		} else {
			max = wp->running_children + nb_to_spawn;
		}
	} else if (wp->config->pm == PM_STYLE_ONDEMAND) {
		if (!in_event_loop) { /* starting */
			max = 0; /* do not create any child at startup */
		} else {
			max = wp->running_children + nb_to_spawn;
		}
	} else { /* PM_STYLE_STATIC */
		max = wp->config->pm_max_children;
	}

	/*
	 * fork children while:
	 *   - fpm_pctl_can_spawn_children : FPM is running in a NORMAL state (aka not restart, stop or reload)
	 *   - wp->running_children < max  : there is less than the max process for the current pool
	 *   - (fpm_global_config.process_max < 1 || fpm_globals.running_children < fpm_global_config.process_max):
	 *     if fpm_global_config.process_max is set, FPM has not fork this number of processes (globaly)
	 */
	while (fpm_pctl_can_spawn_children() && wp->running_children < max && (fpm_global_config.process_max < 1 || fpm_globals.running_children < fpm_global_config.process_max)) {

		warned = 0;
		child = fpm_resources_prepare(wp);

		if (!child) {
			return 2;
		}

		pid = fork();

		switch (pid) {

			case 0 :
				//子进程资源使用
				fpm_child_resources_use(child);
				fpm_globals.is_child = 1;
				//子进程初始化
				fpm_child_init(wp);
				return 0;

			case -1 :
				zlog(ZLOG_SYSERROR, "fork() failed");

				//子进程资源禁止
				fpm_resources_discard(child);
				return 2;

			default :
				child->pid = pid;
				fpm_clock_get(&child->started);
				//父进程资源使用
				fpm_parent_resources_use(child);

				zlog(is_debug ? ZLOG_DEBUG : ZLOG_NOTICE, "[pool %s] child %d started", wp->config->name, (int) pid);
		}

	}

	if (!warned && fpm_global_config.process_max > 0 && fpm_globals.running_children >= fpm_global_config.process_max) {
               if (wp->running_children < max) {
                       warned = 1;
                       zlog(ZLOG_WARNING, "The maximum number of processes has been reached. Please review your configuration and consider raising 'process.max'");
               }
	}

	return 1; /* we are done */
}
/* }}} */

int fpm_children_create_initial(struct fpm_worker_pool_s *wp) /* {{{ */
{
	//根据请求创建子进程
	if (wp->config->pm == PM_STYLE_ONDEMAND) {
		wp->ondemand_event = (struct fpm_event_s *)malloc(sizeof(struct fpm_event_s));

		if (!wp->ondemand_event) {
			zlog(ZLOG_ERROR, "[pool %s] unable to malloc the ondemand socket event", wp->config->name);
			// FIXME handle crash
			return 1;
		}

		memset(wp->ondemand_event, 0, sizeof(struct fpm_event_s));
		//根据socket可读事件进行创建子进程
		fpm_event_set(wp->ondemand_event, wp->listening_socket, FPM_EV_READ | FPM_EV_EDGE, fpm_pctl_on_socket_accept, wp);
		wp->socket_event_set = 1;
		fpm_event_add(wp->ondemand_event, 0);

		return 1;
	}
	return fpm_children_make(wp, 0 /* not in event loop yet */, 0, 1);
}
/* }}} */

int fpm_children_init_main() /* {{{ */
{
	if (fpm_global_config.emergency_restart_threshold &&
		fpm_global_config.emergency_restart_interval) {

		last_faults = malloc(sizeof(time_t) * fpm_global_config.emergency_restart_threshold);

		if (!last_faults) {
			return -1;
		}

		memset(last_faults, 0, sizeof(time_t) * fpm_global_config.emergency_restart_threshold);
	}

	if (0 > fpm_cleanup_add(FPM_CLEANUP_ALL, fpm_children_cleanup, 0)) {
		return -1;
	}

	return 0;
}
/* }}} */
```


