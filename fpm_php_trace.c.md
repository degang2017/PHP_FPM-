# php trace

这里主要是从zend_execute_data里获取函数文件相关信息，zend_execute_data是一个list,根据pool配置的调度深度获取相关execute_data信息.

获取对应vm实际地址：fpm_trace_mach、fpm_trace_pread及fpm_trace_ptrace

fpm_trace_get_strz 获取zval值
fpm_trace_get_long 获取对应vm值地址值

```c

static int fpm_php_trace_dump(struct fpm_child_s *child, FILE *slowlog) /* {{{ */
{
	//trace深度
	int callers_limit = child->wp->config->request_slowlog_trace_depth;
	//子进程id
	pid_t pid = child->pid;
	struct timeval tv;
	//日志buf大小
	static const int buf_size = 1024;
	//创建buf数组
	char buf[buf_size];
	//执行数据
	long execute_data;
	//路径
	long path_translated;
	long l;

	//获取当前时间
	gettimeofday(&tv, 0);

	//存储当前时间到buf中
	zlog_print_time(&tv, buf, buf_size);

	//进程池名和pid
	fprintf(slowlog, "\n%s [pool %s] pid %d\n", buf, child->wp->config->name, (int) pid);

	//获取路劲指针地址数据地址
	if (0 > fpm_trace_get_long((long) &SG(request_info).path_translated, &l)) {
		return -1;
	}

	path_translated = l;

	//获取脚本名
	if (0 > fpm_trace_get_strz(buf, buf_size, path_translated)) {
		return -1;
	}

	fprintf(slowlog, "script_filename = %s\n", buf);

	//获取当前执行代码的地址
	if (0 > fpm_trace_get_long((long) &EG(current_execute_data), &l)) {
		return -1;
	}

	execute_data = l;

	while (execute_data) {
		long function;
		long function_name;
		long file_name;
		long prev;
		uint32_t lineno = 0;

		//获取函数地址
		if (0 > fpm_trace_get_long(execute_data + offsetof(zend_execute_data, func), &l)) {
			return -1;
		}

		function = l;

		if (valid_ptr(function)) {
			//获取函数名地址
			if (0 > fpm_trace_get_long(function + offsetof(zend_function, common.function_name), &l)) {
				return -1;
			}

			function_name = l;

			//如果没有函数的话
			if (function_name == 0) {
				uint32_t *call_info = (uint32_t *)&l;
				//获取内型
				if (0 > fpm_trace_get_long(execute_data + offsetof(zend_execute_data, This.u1.type_info), &l)) {
					return -1;
				}

				//如果是堆栈的开始地址
				if (ZEND_CALL_KIND_EX((*call_info) >> ZEND_CALL_INFO_SHIFT) == ZEND_CALL_TOP_CODE) {
					return 0;
				} else if (ZEND_CALL_KIND_EX(*(call_info) >> ZEND_CALL_INFO_SHIFT) == ZEND_CALL_NESTED_CODE) {
					//引用文件或者eval函数
					memcpy(buf, "[INCLUDE_OR_EVAL]", sizeof("[INCLUDE_OR_EVAL]"));
				} else {
					ZEND_ASSERT(0);
				}
			} else {
				//参数
				if (0 > fpm_trace_get_strz(buf, buf_size, function_name + offsetof(zend_string, val))) {
					return -1;
				}

			}
		} else {
			memcpy(buf, "???", sizeof("???"));
		}
		
		fprintf(slowlog, "[0x%" PTR_FMT "lx] ", execute_data);

		fprintf(slowlog, "%s()", buf);

		*buf = '\0';

		//获取上一个zend_execute_data结构
		if (0 > fpm_trace_get_long(execute_data + offsetof(zend_execute_data, prev_execute_data), &l)) {
			return -1;
		}

		execute_data = prev = l;

		while (prev) {
			zend_uchar *type;

			//函数 地址
			if (0 > fpm_trace_get_long(prev + offsetof(zend_execute_data, func), &l)) {
				return -1;
			}

			function = l;

			if (!valid_ptr(function)) {
				break;
			}

			type = (zend_uchar *)&l;
			//类型
			if (0 > fpm_trace_get_long(function + offsetof(zend_function, type), &l)) { 
				return -1;
			}

			if (ZEND_USER_CODE(*type)) {
				if (0 > fpm_trace_get_long(function + offsetof(zend_op_array, filename), &l)) {
					return -1;
				}

				file_name = l;

				//参数
				if (0 > fpm_trace_get_strz(buf, buf_size, file_name + offsetof(zend_string, val))) {
					return -1;
				}

				//当前执行的opcode
				if (0 > fpm_trace_get_long(prev + offsetof(zend_execute_data, opline), &l)) {
					return -1;
				}

				if (valid_ptr(l)) {
					long opline = l;
					uint32_t *lu = (uint32_t *) &l;

					//文件行
					if (0 > fpm_trace_get_long(opline + offsetof(struct _zend_op, lineno), &l)) {
						return -1;
					}

					lineno = *lu;
				}
				break;
			} 

			//上一下execute_data
			if (0 > fpm_trace_get_long(prev + offsetof(zend_execute_data, prev_execute_data), &l)) {
				return -1;
			}

			prev = l;
		}

		fprintf(slowlog, " %s:%u\n", *buf ? buf : "unknown", lineno);
		//深度减一
		if (0 == --callers_limit) {
			break;
		}
	}

	return 0;
}
/* }}} */

void fpm_php_trace(struct fpm_child_s *child) /* {{{ */
{
	//记分牌慢日志数加1
	fpm_scoreboard_update(0, 0, 0, 0, 0, 0, 1, FPM_SCOREBOARD_ACTION_INC, child->wp->scoreboard);
	FILE *slowlog;

	zlog(ZLOG_NOTICE, "about to trace %d", (int) child->pid);

	slowlog = fopen(child->wp->config->slowlog, "a+");

	if (!slowlog) {
		zlog(ZLOG_SYSERROR, "unable to open slowlog (%s)", child->wp->config->slowlog);
		goto done0;
	}

	//pid 赋值
	if (0 > fpm_trace_ready(child->pid)) {
		goto done1;
	}

	if (0 > fpm_php_trace_dump(child, slowlog)) {
		fprintf(slowlog, "+++ dump failed\n");
	}

	//解决调试
	if (0 > fpm_trace_close(child->pid)) {
		goto done1;
	}

done1:
	fclose(slowlog);

done0:
	fpm_pctl_kill(child->pid, FPM_PCTL_CONT);
	child->tracer = 0;

	zlog(ZLOG_NOTICE, "finished trace of %d", (int) child->pid);
}
```



