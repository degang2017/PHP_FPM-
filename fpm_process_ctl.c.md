# 进程控制器

```c


static void fpm_pctl_perform_idle_server_maintenance(struct timeval *now) /* {{{ */
{
	struct fpm_worker_pool_s *wp;

	for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
		struct fpm_child_s *child;
		struct fpm_child_s *last_idle_child = NULL;
		int idle = 0;
		int active = 0;
		int children_to_fork;
		unsigned cur_lq = 0;

		if (wp->config == NULL) continue;

		//遍历所有子进程
		for (child = wp->children; child; child = child->next) {
			//空闲子进程
			if (fpm_request_is_idle(child)) {
				//最后一个空闲子进程为空的话，将当前进程赋值
				if (last_idle_child == NULL) {
					last_idle_child = child;
				} else {
					//如果进程开始时间小于最后空闲子进程开始时间
					if (timercmp(&child->started, &last_idle_child->started, <)) {
						last_idle_child = child;
					}
				}
				//空闲进程加1
				idle++;
			} else {
				//活动进程加1
				active++;
			}
		}

		/* update status structure for all PMs */
		if (wp->listen_address_domain == FPM_AF_INET) {
			//如果当前监听队列为小于0
			if (0 > fpm_socket_get_listening_queue(wp->listening_socket, &cur_lq, NULL)) {
				cur_lq = 0;
#if 0
			} else {
				if (cur_lq > 0) {
					if (!wp->warn_lq) {
						zlog(ZLOG_WARNING, "[pool %s] listening queue is not empty, #%d requests are waiting to be served, consider raising pm.max_children setting (%d)", wp->config->name, cur_lq, wp->config->pm_max_children);
						wp->warn_lq = 1;
					}
				} else {
					wp->warn_lq = 0;
				}
#endif
			}
		}
		//更新记分牌
		fpm_scoreboard_update(idle, active, cur_lq, -1, -1, -1, 0, FPM_SCOREBOARD_ACTION_SET, wp->scoreboard);

		/* this is specific to PM_STYLE_ONDEMAND */
		//如果是ondemand模式
		if (wp->config->pm == PM_STYLE_ONDEMAND) {
			struct timeval last, now;

			zlog(ZLOG_DEBUG, "[pool %s] currently %d active children, %d spare children", wp->config->name, active, idle);

			if (!last_idle_child) continue;

			//获取请求最后活动进程时间
			fpm_request_last_activity(last_idle_child, &last);
			fpm_clock_get(&now);
			//当前时间减去进程空闲超时时间小于最后请求空闲进程时间
			if (last.tv_sec < now.tv_sec - wp->config->pm_process_idle_timeout) {
				last_idle_child->idle_kill = 1;
				//进程退出
				fpm_pctl_kill(last_idle_child->pid, FPM_PCTL_QUIT);
			}

			continue;
		}

		/* the rest is only used by PM_STYLE_DYNAMIC */
		if (wp->config->pm != PM_STYLE_DYNAMIC) continue;

		zlog(ZLOG_DEBUG, "[pool %s] currently %d active children, %d spare children, %d running children. Spawning rate %d", wp->config->name, active, idle, wp->running_children, wp->idle_spawn_rate);

		//空闲进程大于最大空闲进程并且最后一个空闲进程不为空
		if (idle > wp->config->pm_max_spare_servers && last_idle_child) {
			last_idle_child->idle_kill = 1;
			fpm_pctl_kill(last_idle_child->pid, FPM_PCTL_QUIT);
			wp->idle_spawn_rate = 1;
			continue;
		}

		//空闲进程小于最小空闲进程
		if (idle < wp->config->pm_min_spare_servers) {
			//运行子进程大于或等于最大子进程数
			if (wp->running_children >= wp->config->pm_max_children) {
				//非警告
				if (!wp->warn_max_children) {
					//更新记分牌，已达到最大进程数加1
					fpm_scoreboard_update(0, 0, 0, 0, 0, 1, 0, FPM_SCOREBOARD_ACTION_INC, wp->scoreboard);
					zlog(ZLOG_WARNING, "[pool %s] server reached pm.max_children setting (%d), consider raising it", wp->config->name, wp->config->pm_max_children);
					wp->warn_max_children = 1;
				}
				//基数进程数为1
				wp->idle_spawn_rate = 1;
				continue;
			}

			//进程基数大于8
			if (wp->idle_spawn_rate >= 8) {
				zlog(ZLOG_WARNING, "[pool %s] seems busy (you may need to increase pm.start_servers, or pm.min/max_spare_servers), spawning %d children, there are %d idle, and %d total children", wp->config->name, wp->idle_spawn_rate, idle, wp->running_children);
			}

			//计算要生成的空闲进程数
			children_to_fork = MIN(wp->idle_spawn_rate, wp->config->pm_min_spare_servers - idle);

			//确保不超过最大进程数
			children_to_fork = MIN(children_to_fork, wp->config->pm_max_children - wp->running_children);
			if (children_to_fork <= 0) {
				if (!wp->warn_max_children) {
					fpm_scoreboard_update(0, 0, 0, 0, 0, 1, 0, FPM_SCOREBOARD_ACTION_INC, wp->scoreboard);
					zlog(ZLOG_WARNING, "[pool %s] server reached pm.max_children setting (%d), consider raising it", wp->config->name, wp->config->pm_max_children);
					wp->warn_max_children = 1;
				}
				wp->idle_spawn_rate = 1;
				continue;
			}
			wp->warn_max_children = 0;

			fpm_children_make(wp, 1, children_to_fork, 1);

			/* if it's a child, stop here without creating the next event
			 * this event is reserved to the master process
			 */
			if (fpm_globals.is_child) {
				return;
			}

			zlog(ZLOG_DEBUG, "[pool %s] %d child(ren) have been created dynamically", wp->config->name, children_to_fork);

			/* Double the spawn rate for the next iteration */
			if (wp->idle_spawn_rate < FPM_MAX_SPAWN_RATE) {
				wp->idle_spawn_rate *= 2;
			}
			continue;
		}
		wp->idle_spawn_rate = 1;
	}
}
/* }}} */

void fpm_pctl_heartbeat(struct fpm_event_s *ev, short which, void *arg) /* {{{ */
{
	static struct fpm_event_s heartbeat;
	struct timeval now;

    //不能为主进程
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
/* }}} */
```


