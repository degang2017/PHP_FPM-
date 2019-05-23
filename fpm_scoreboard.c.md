# 记分牌

记分牌主要记进程池请求数、执行时间、进程模型、子进程相关数据等

初始化记分牌的时候会根据最大进程数申请相应的内存空间映射到共享内存里，子进程记分牌会循环利用空闲的槽，所有记分牌更新设置都会使用自旋锁。


```c
struct fpm_scoreboard_proc_s {
	union {
		atomic_t lock;
		char dummy[16];
	};
    //使用数量
	int used;
    //开始时期
	time_t start_epoch;
    //进程id
	pid_t pid;
    //请求数
	unsigned long requests;
    //请求状态
	enum fpm_request_stage_e request_stage;
    //accepted 时间
	struct timeval accepted;
    //执行时间
	struct timeval duration;
    //accepted 时期
	time_t accepted_epoch;
    //时间
	struct timeval tv;
    //请求uri
	char request_uri[128];
    //参数
	char query_string[512];
    //method
	char request_method[16];
    //post 长度
	size_t content_length; /* used with POST only */
    //脚本名
	char script_filename[256];
    //auth 用户
	char auth_user[32];
#ifdef HAVE_TIMES
    //cpu accepted 耗时
	struct tms cpu_accepted;
    //执行cpu耗时
	struct timeval cpu_duration;
    //最后请求cpu耗时
	struct tms last_request_cpu;
    //最后请求cpu完成耗时
	struct timeval last_request_cpu_duration;
#endif
    //使用内存
	size_t memory;
};

struct fpm_scoreboard_s {
	union {
		atomic_t lock;
		char dummy[16];
	};
    //pool名
	char pool[32];
    //进程模式
	int pm;
    //开启时期
	time_t start_epoch;
    //空闲数
	int idle;
    //活动数
	int active;
    //最大活动数
	int active_max;
    //请求数
	unsigned long int requests;
    //最大子进程到达
	unsigned int max_children_reached;
    //socket监听队列
	int lq;
    //socket监听队列最大值
	int lq_max;
    //socket监听队列长度
	unsigned int lq_len;
    //子进程数
	unsigned int nprocs;
	int free_proc;
    //慢请求数
	unsigned long int slow_rq;
    //记分牌进程
	struct fpm_scoreboard_proc_s *procs[];
};
```

```c
static struct fpm_scoreboard_s *fpm_scoreboard = NULL;
//记分牌索引
static int fpm_scoreboard_i = -1;
#ifdef HAVE_TIMES
static float fpm_scoreboard_tick;
#endif

//记分牌初始化
int fpm_scoreboard_init_main() /* {{{ */
{
	struct fpm_worker_pool_s *wp;
	unsigned int i;

#ifdef HAVE_TIMES
#if (defined(HAVE_SYSCONF) && defined(_SC_CLK_TCK))
    //获取时钟的赫兹
	fpm_scoreboard_tick = sysconf(_SC_CLK_TCK);
#else /* _SC_CLK_TCK */
#ifdef HZ
	fpm_scoreboard_tick = HZ;
#else /* HZ */
	fpm_scoreboard_tick = 100;
#endif /* HZ */
#endif /* _SC_CLK_TCK */
	zlog(ZLOG_DEBUG, "got clock tick '%.0f'", fpm_scoreboard_tick);
#endif /* HAVE_TIMES */


    //遍历所有的进程池
	for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
        //记分牌大小及子进程大小
		size_t scoreboard_size, scoreboard_nprocs_size;
        //共享内存
		void *shm_mem;

        //如果最大子进程小于1
		if (wp->config->pm_max_children < 1) {
			zlog(ZLOG_ERROR, "[pool %s] Unable to create scoreboard SHM because max_client is not set", wp->config->name);
			return -1;
		}

        //如果已经初始过
		if (wp->scoreboard) {
			zlog(ZLOG_ERROR, "[pool %s] Unable to create scoreboard SHM because it already exists", wp->config->name);
			return -1;
		}

        //记分牌内存大小需要加上最大子进程数*fpm_scoreboard_proc_s
		scoreboard_size        = sizeof(struct fpm_scoreboard_s) + (wp->config->pm_max_children) * sizeof(struct fpm_scoreboard_proc_s *);
        //子进程记分牌内存大小
		scoreboard_nprocs_size = sizeof(struct fpm_scoreboard_proc_s) * wp->config->pm_max_children;
        //创建共享内存
		shm_mem                = fpm_shm_alloc(scoreboard_size + scoreboard_nprocs_size);

		if (!shm_mem) {
			return -1;
		}
		wp->scoreboard         = shm_mem;
        //记分牌子进程数
		wp->scoreboard->nprocs = wp->config->pm_max_children;
        //共享内存内存偏移量加上scoreboard大小，主要是为后面映射子进程记分牌
		shm_mem               += scoreboard_size;

        //为子进程记分牌映射共享内存
		for (i = 0; i < wp->scoreboard->nprocs; i++, shm_mem += sizeof(struct fpm_scoreboard_proc_s)) {
			wp->scoreboard->procs[i] = shm_mem;
		}

        //进程模式
		wp->scoreboard->pm          = wp->config->pm;
        //记分牌开始时间
		wp->scoreboard->start_epoch = time(NULL);
        //进程池名称
		strlcpy(wp->scoreboard->pool, wp->config->name, sizeof(wp->scoreboard->pool));
	}
	return 0;
}
/* }}} */

//记分牌更新
void fpm_scoreboard_update(int idle, int active, int lq, int lq_len, int requests, int max_children_reached, int slow_rq, int action, struct fpm_scoreboard_s *scoreboard) /* {{{ */
{
    //如果记分牌没有的话，直接将fpm_scoreboard赋值
	if (!scoreboard) {
		scoreboard = fpm_scoreboard;
	}
	if (!scoreboard) {
		zlog(ZLOG_WARNING, "Unable to update scoreboard: the SHM has not been found");
		return;
	}


    //更新的时间需要加上自旋锁
	fpm_spinlock(&scoreboard->lock, 0);
    //如果操作的动作是设置的话
	if (action == FPM_SCOREBOARD_ACTION_SET) {
        //设置空间进程数
		if (idle >= 0) {
			scoreboard->idle = idle;
		}
        //设置活动进程数
		if (active >= 0) {
			scoreboard->active = active;
		}
        //设置socket监听队列
		if (lq >= 0) {
			scoreboard->lq = lq;
		}
        //长度
		if (lq_len >= 0) {
			scoreboard->lq_len = lq_len;
		}
        //防止不并要的测试
#ifdef HAVE_FPM_LQ /* prevent unnecessary test */
		if (scoreboard->lq > scoreboard->lq_max) {
			scoreboard->lq_max = scoreboard->lq;
		}
#endif
        //请求数
		if (requests >= 0) {
			scoreboard->requests = requests;
		}

        //最大子进程到请求达数
		if (max_children_reached >= 0) {
			scoreboard->max_children_reached = max_children_reached;
		}
        //请求慢日志数
		if (slow_rq > 0) {
			scoreboard->slow_rq = slow_rq;
		}
	} else {
        //否则是自增
		if (scoreboard->idle + idle > 0) {
			scoreboard->idle += idle;
		} else {
			scoreboard->idle = 0;
		}

		if (scoreboard->active + active > 0) {
			scoreboard->active += active;
		} else {
			scoreboard->active = 0;
		}

		if (scoreboard->requests + requests > 0) {
			scoreboard->requests += requests;
		} else {
			scoreboard->requests = 0;
		}

		if (scoreboard->max_children_reached + max_children_reached > 0) {
			scoreboard->max_children_reached += max_children_reached;
		} else {
			scoreboard->max_children_reached = 0;
		}

		if (scoreboard->slow_rq + slow_rq > 0) {
			scoreboard->slow_rq += slow_rq;
		} else {
			scoreboard->slow_rq = 0;
		}
	}

	if (scoreboard->active > scoreboard->active_max) {
		scoreboard->active_max = scoreboard->active;
	}

    //释放自旋锁
	fpm_unlock(scoreboard->lock);
}
/* }}} */

//获取记分牌
struct fpm_scoreboard_s *fpm_scoreboard_get() /* {{{*/
{
	return fpm_scoreboard;
}
/* }}} */

//获取子进程记分牌
struct fpm_scoreboard_proc_s *fpm_scoreboard_proc_get(struct fpm_scoreboard_s *scoreboard, int child_index) /* {{{*/
{
	if (!scoreboard) {
		scoreboard = fpm_scoreboard;
	}

	if (!scoreboard) {
		return NULL;
	}

	if (child_index < 0) {
		child_index = fpm_scoreboard_i;
	}

	if (child_index < 0 || (unsigned int)child_index >= scoreboard->nprocs) {
		return NULL;
	}

	return scoreboard->procs[child_index];
}
/* }}} */

//获取记分牌锁
struct fpm_scoreboard_s *fpm_scoreboard_acquire(struct fpm_scoreboard_s *scoreboard, int nohang) /* {{{ */
{
	struct fpm_scoreboard_s *s;

	s = scoreboard ? scoreboard : fpm_scoreboard;
	if (!s) {
		return NULL;
	}

    //尝试获取锁
	if (!fpm_spinlock(&s->lock, nohang)) {
		return NULL;
	}
	return s;
}
/* }}} */

//释放记分牌锁
void fpm_scoreboard_release(struct fpm_scoreboard_s *scoreboard) {
	if (!scoreboard) {
		return;
	}

	scoreboard->lock = 0;
}

//获取子进程记分牌锁
struct fpm_scoreboard_proc_s *fpm_scoreboard_proc_acquire(struct fpm_scoreboard_s *scoreboard, int child_index, int nohang) /* {{{ */
{
	struct fpm_scoreboard_proc_s *proc;

	proc = fpm_scoreboard_proc_get(scoreboard, child_index);
	if (!proc) {
		return NULL;
	}

	if (!fpm_spinlock(&proc->lock, nohang)) {
		return NULL;
	}

	return proc;
}
/* }}} */

//释放子进程记分牌锁
void fpm_scoreboard_proc_release(struct fpm_scoreboard_proc_s *proc) /* {{{ */
{
	if (!proc) {
		return;
	}

	proc->lock = 0;
}

//释放记分牌共享内存
void fpm_scoreboard_free(struct fpm_scoreboard_s *scoreboard) /* {{{ */
{
	size_t scoreboard_size, scoreboard_nprocs_size;

	if (!scoreboard) {
		zlog(ZLOG_ERROR, "**scoreboard is NULL");
		return;
	}

	scoreboard_size        = sizeof(struct fpm_scoreboard_s) + (scoreboard->nprocs) * sizeof(struct fpm_scoreboard_proc_s *);
	scoreboard_nprocs_size = sizeof(struct fpm_scoreboard_proc_s) * scoreboard->nprocs;
	
	fpm_shm_free(scoreboard, scoreboard_size + scoreboard_nprocs_size);
}
/* }}} */

//设置子记分牌进程ID和进程开始时间
void fpm_scoreboard_child_use(struct fpm_scoreboard_s *scoreboard, int child_index, pid_t pid) /* {{{ */
{
	struct fpm_scoreboard_proc_s *proc;
	fpm_scoreboard = scoreboard;
	fpm_scoreboard_i = child_index;
	proc = fpm_scoreboard_proc_get(scoreboard, child_index);
	if (!proc) {
		return;
	}
	proc->pid = pid;
	proc->start_epoch = time(NULL);
}
/* }}} */

//清空子进程记分牌内存
void fpm_scoreboard_proc_free(struct fpm_scoreboard_s *scoreboard, int child_index) /* {{{ */
{
	if (!scoreboard) {
		return;
	}

	if (child_index < 0 || (unsigned int)child_index >= scoreboard->nprocs) {
		return;
	}

	if (scoreboard->procs[child_index] && scoreboard->procs[child_index]->used > 0) {
		memset(scoreboard->procs[child_index], 0, sizeof(struct fpm_scoreboard_proc_s));
	}

    //将此槽设置为空闲，给下个子进程分配
	scoreboard->free_proc = child_index;
}
/* }}} */

//子进程记分牌内存申请
int fpm_scoreboard_proc_alloc(struct fpm_scoreboard_s *scoreboard, int *child_index) /* {{{ */
{
	int i = -1;

	if (!scoreboard || !child_index) {
		return -1;
	}

	/* first try the slot which is supposed to be free */
    //如果有子进程记分牌释放了空间并且子进程id小于最大子进程数
	if (scoreboard->free_proc >= 0 && (unsigned int)scoreboard->free_proc < scoreboard->nprocs) {
        //子进程记分牌有数并且没有使用，将空闲的子进程记分牌分配给出来给新申请使用
		if (scoreboard->procs[scoreboard->free_proc] && !scoreboard->procs[scoreboard->free_proc]->used) {
			i = scoreboard->free_proc;
		}
	}

	if (i < 0) { /* the supposed free slot is not, let's search for a free slot */
		zlog(ZLOG_DEBUG, "[pool %s] the proc->free_slot was not free. Let's search", scoreboard->pool);
        //查询其他没有使用的槽
		for (i = 0; i < (int)scoreboard->nprocs; i++) {
			if (scoreboard->procs[i] && !scoreboard->procs[i]->used) { /* found */
				break;
			}
		}
	}

    //没有释放的槽
	if (i < 0 || i >= (int)scoreboard->nprocs) {
		zlog(ZLOG_ERROR, "[pool %s] no free scoreboard slot", scoreboard->pool);
		return -1;
	}

    //如果有使用的槽，将其设置成已使用
	scoreboard->procs[i]->used = 1;
	*child_index = i;

	/* supposed next slot is free */
    //如果没有空闲的槽，将空闲子记分牌设置成0
	if (i + 1 >= (int)scoreboard->nprocs) {
		scoreboard->free_proc = 0;
	} else {
		scoreboard->free_proc = i + 1;
	}

	return 0;
}
/* }}} */

#ifdef HAVE_TIMES
float fpm_scoreboard_get_tick() /* {{{ */
{
    //获取时钟的赫兹
	return fpm_scoreboard_tick;
}
/* }}} */
#endif


```


