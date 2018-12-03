# fpm 环境变量

这里需要了解一下argv和environ在内存的布局，他们内存分配是连续，都是以NULL结尾

由于设置进程名argv[0]可能长度不够，需要重新environ分配内存

```c
#ifndef HAVE_SETENV
# ifdef (__sparc__ || __sparc)
int setenv(char *name, char *value, int clobber) /* {{{ */
{
    char   *malloc();
    char   *getenv();
    char   *cp;

    if (clobber == 0 && getenv(name) != 0) {
        return 0;
    }

    if ((cp = malloc(strlen(name) + strlen(value) + 2)) == 0) {
        return 1;
    }
    sprintf(cp, "%s=%s", name, value);
    return putenv(cp);
}
/* }}} */
# else
int setenv(char *name, char *value, int overwrite) /* {{{ */
{
    int name_len = strlen(name);
    int value_len = strlen(value);
    //alloca它申请的是栈空间的内存，用完退出栈时自动释放
    char *var = alloca(name_len + 1 + value_len + 1);

    memcpy(var, name, name_len);

    var[name_len] = '=';

    memcpy(var + name_len + 1, value, value_len);

    //key=value拼接
    var[name_len + 1 + value_len] = '\0';

    return putenv(var);
}
/* }}} */
# endif
#endif

#ifndef HAVE_CLEARENV
void clearenv() /* {{{ */
{
    char **envp;
    char *s;

    /* this algo is the only one known to me
        that works well on all systems */
    //系统环境变量
    while (*(envp = environ)) {
        char *eq = strchr(*envp, '=');

        s = strdup(*envp);

        //获取环境变量key
        if (eq) s[eq - *envp] = '\0';

        unsetenv(s);
        free(s);
    }

}
/* }}} */
#endif

#ifndef HAVE_UNSETENV
void unsetenv(const char *name) /* {{{ */
{
    if(getenv(name) != NULL) {
        int ct = 0;
        int del = 0;

        while(environ[ct] != NULL) {
            //如果key值找到的话，del就是当前数组的索引
            if (nvmatch(name, environ[ct]) != 0) del=ct; /* <--- WTF?! */
            { ct++; } /* <--- WTF?! */
        }
        /* isn't needed free here?? */
        //将最后一个数组赋值给需要删除的
        environ[del] = environ[ct-1];
        //将最后一个数组赋值为空,内存并没有释放
        environ[ct-1] = NULL;
    }
}
/* }}} */

static char * nvmatch(char *s1, char *s2) /* {{{ */
{
    while(*s1 == *s2++)
    {
        if(*s1++ == '=') {
            return s2;
        }
    }
    if(*s1 == '\0' && *(s2-1) == '=') {
        return s2;
    }
    return NULL;
}
/* }}} */
#endif

void fpm_env_setproctitle(char *title) /* {{{ */
{
#ifdef HAVE_SETPROCTITLE
    setproctitle("%s", title);
#else
#ifdef __linux__
    if (fpm_env_argv != NULL && fpm_env_argv_len > strlen(SETPROCTITLE_PREFIX) + 3) {
        memset(fpm_env_argv[0], 0, fpm_env_argv_len);
        strncpy(fpm_env_argv[0], SETPROCTITLE_PREFIX, fpm_env_argv_len - 2);
        strncpy(fpm_env_argv[0] + strlen(SETPROCTITLE_PREFIX), title, fpm_env_argv_len - strlen(SETPROCTITLE_PREFIX) - 2);
        fpm_env_argv[1] = NULL;
    }
#endif
#endif
}
/* }}} */

//初始化子进程
int fpm_env_init_child(struct fpm_worker_pool_s *wp) /* {{{ */
{
    struct key_value_s *kv;
    char *title;
    //标题
    spprintf(&title, 0, "pool %s", wp->config->name);
    //设置进程名称
    fpm_env_setproctitle(title);
    efree(title);

    if (wp->config->clear_env) {
        //清理所有环境变量
        clearenv();
    }

    //将配置文件env设置到环境变量里
    for (kv = wp->config->env; kv; kv = kv->next) {
        setenv(kv->key, kv->value, 1);
    }

    if (wp->user) {
        setenv("USER", wp->user, 1);
    }

    if (wp->home) {
        setenv("HOME", wp->home, 1);
    }

    return 0;
}
/* }}} */

static int fpm_env_conf_wp(struct fpm_worker_pool_s *wp) /* {{{ */
{
    struct key_value_s *kv;

    //变量env数组
    for (kv = wp->config->env; kv; kv = kv->next) {
        //如果值第一个字符为$，取后面值，例如$HOSTNAME
        if (*kv->value == '$') {
            //这里加1，是去掉$字符
            char *value = getenv(kv->value + 1);

            if (!value) {
                value = "";
            }

            free(kv->value);
            //将取出的值给$xxx
            kv->value = strdup(value);
        }

        //如果在配置文件中指定这些VAR，则应删除自动检测值
        if (!strcmp(kv->key, "USER")) {
            free(wp->user);
            wp->user = 0;
        }

        if (!strcmp(kv->key, "HOME")) {
            free(wp->home);
            wp->home = 0;
        }
    }

    return 0;
}
/* }}} */
int fpm_env_init_main() /* {{{ */
{
    struct fpm_worker_pool_s *wp;
    int i;
    char *first = NULL;
    char *last = NULL;
    char *title;

    for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
        if (0 > fpm_env_conf_wp(wp)) {
            return -1;
        }
    }
#ifndef HAVE_SETPROCTITLE
#ifdef __linux__
    /*
     * This piece of code has been inspirated from nginx and pureftpd code, which
     * are under BSD licence.
     *
     * To change the process title in Linux we have to set argv[1] to NULL
     * and to copy the title to the same place where the argv[0] points to.
     * However, argv[0] may be too small to hold a new title.  Fortunately, Linux
     * store argv[] and environ[] one after another.  So we should ensure that is
     * the continuous memory and then we allocate the new memory for environ[]
     * and copy it.  After this we could use the memory starting from argv[0] for
     * our process title.
     */

    //进程名称通常是argv[0], argv和environ数组都是以NULL指针结尾
    //malloc开辟出新内存，将environ复制到新开辟的内存中，然后将environ指向新开辟的内存。这么做的目的有两个：1、保存经进程原有的environ；2、能
够接受相对较长的新的进程名称。
    //这里argv 和environ内存是连续的,所以需要扩展


    //遍历argv参数
    for (i = 0; i < fpm_globals.argc; i++) {
        //如果first为null，肯定是第一个
        if (first == NULL) {
            first = fpm_globals.argv[i];
        }
        //每次跟下个数组索引比较，直到不相等为止
        if (last == NULL || fpm_globals.argv[i] == last + 1) {
            last = fpm_globals.argv[i] + strlen(fpm_globals.argv[i]);
        }
    }
    //环境变量
    if (environ) {
        //遍历环境变量
        for (i = 0; environ[i]; i++) {
            if (first == NULL) {
                first = environ[i];
            }
            if (last == NULL || environ[i] == last + 1) {
                last = environ[i] + strlen(environ[i]);
            }
        }
    }
    if (first == NULL || last == NULL) {
        return 0;
    }

    //argv 到 environ 内存长度
    fpm_env_argv_len = last - first;
    fpm_env_argv = fpm_globals.argv;
    if (environ != NULL) {
        char **new_environ;
        unsigned int env_nb = 0U;

        //环境数组长度
        while (environ[env_nb]) {
            env_nb++;
        }

        //扩展环境变量大小，加1主要存储NULL值
        if ((new_environ = malloc((1U + env_nb) * sizeof (char *))) == NULL) {
            return -1;
        }
        new_environ[env_nb] = NULL;
        while (env_nb > 0U) {
            env_nb--;
            //重新赋值给新的环境变量
            new_environ[env_nb] = strdup(environ[env_nb]);
        }
        environ = new_environ;
    }
#endif
#endif

    spprintf(&title, 0, "master process (%s)", fpm_globals.config);
    fpm_env_setproctitle(title);
    efree(title);
    return 0;
}
/* }}} */
```

