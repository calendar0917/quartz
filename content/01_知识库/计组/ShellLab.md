## 01 Ctrl+d 退出

好像已经给出了  
```c
    if (feof(stdin)) { /* End of file (ctrl-d) */
      fflush(stdout);
      exit(0);
    }
```

## 02 内置 quit

在 eval 函数当中实现，要先处理输入（利用已经给出的 parseline 函数），然后执行即可。

如果先进行判断 `cmdline == "quit"`，会导致最后有换行符的影响！

```c
// 参数处理
  char *argv[MAXARGS]; // 存放参数
  char buf[MAXLINE];   // 存放修改后的命令行副本
  int bg;              // 是否后台运行
  pid_t pid;
  strcpy(buf, cmdline); // 拷贝备份,因为 parseline 会改变原字符串
  bg = parseline(buf, argv);
  // 02 内置quit指令,要先处理输入再判断，否则 \n 会有影响
  if (strcmp(argv[0], "quit") == 0) {
    exit(0);
  }
```

## 03 waitpid 处理前台进程 

```c
if (!bg) {
    pid = fork();
    if (pid == 0) {
      if (execve(argv[0], argv, environ) < 0) {
        fprintf(stderr, "%s command not found\n", argv[0]);
        exit(1);
      }
    } else {
      waitpid(pid, NULL, 0);
    }
  }
```

## 04 sigchld_handler 异步收割后台进程

当子进程退出后，会向父进程发出 SIGCHLD 信号，内核会强制中断父进程的工作流，跳转到 sigchld_handler 指向的处理地址，所以只需要在一开始的时候将处理地址安装好即可。

安装：`  Signal(SIGCHLD, sigchld_handler);`

Signal 是 sigaction 的封装：

```c
handler_t *Signal(int signum, handler_t *handler) {
  struct sigaction action, old_action;

  action.sa_handler = handler;
  sigemptyset(&action.sa_mask); /* block sigs of type being handled */
  action.sa_flags = SA_RESTART; /* restart syscalls if possible */

  if (sigaction(signum, &action, &old_action) < 0)
    unix_error("Signal error");
  return (old_action.sa_handler);
}
```

而 handler 的实现需要有: while + waitpid （回收子进程）+ WNOHANG,同时要注意，由于可能在任何时候打断父进程，所以需要保存 errno 变量（用于记录系统调用结果的）。

```c
void sigchld_handler(int sig) {
  int old_errno = errno;
  int state;
  pid_t pid;

  while((pid = waitpid(-1, &state, WNOHANG | WUNTRACED)) > 0){
    deletejob(jobs, pid);
  }
  errno = old_errno;
}
```

然后在 eval 的分支中添加 addjobs 即可。

## 05 处理内置 jobs

现在发现，如果和以前一样，直接在 eval 当中判断内置函数已经不行了，因为原来的 quit 可以直接退出，而这里的 jobs 需要在执行完以后再继续等待。解决办法是实现 buildin_cmd 函数，如果执行了内部函数，就不需要再 fork 了。

```c
int builtin_cmd(char **argv) {
  // 返回 1 则不继续执行
  if (strcmp(argv[0], "quit") == 0) {
    exit(0);
  }
  if (strcmp(argv[0], "jobs") == 0) {
    listjobs(jobs);
    return 1;
  }
  return 0;
}
```

但是，这样写是有隐患的
- 如果父进程还没有写进 addjob 就查询，会返回 jid=0
- 如果后台子进程运行极快，会导致在调用 addjob 之前该进程已经死亡，就会触发 deletejob,删除一个空 job.

所以，需要 sigprocmask 函数来让 addjob 成为一个不可打断的操作,让内核发出的死亡信号延缓执行

逻辑：屏蔽-fork - 子进程重置屏蔽、父进程保护 addjob - add后要恢复prev

```c
void eval(char *cmdline) {
    // ... 解析参数等 ...

    sigset_t mask_all, mask_one, prev_one;
    sigfillset(&mask_all);         // 包含所有信号的集合
    sigemptyset(&mask_one);
    sigaddset(&mask_one, SIGCHLD); // 只包含 SIGCHLD 的集合

    /* 步骤 1：在 fork 之前屏蔽 SIGCHLD */
    // 这样子进程即使现在死了，SIGCHLD 也会被排队挂起，不会触发 handler
    sigprocmask(SIG_BLOCK, &mask_one, &prev_one); 

    if ((pid = fork()) == 0) {   /* 子进程 */
        /* 步骤 2：子进程继承了屏蔽位，必须在 execve 前恢复 */
        sigprocmask(SIG_SETMASK, &prev_one, NULL); 
        if (execve(argv[0], argv, environ) < 0) {
            // 报错处理...
            exit(1);
        }
    }

    /* 父进程 */
    /* 步骤 3：在 addjob 期间屏蔽所有信号（可选但推荐，保护全局变量 jobs） */
    sigprocmask(SIG_BLOCK, &mask_all, NULL); 
    
    // 现在 addjob 是绝对安全的，因为 handler 被堵在门外了
    addjob(jobs, pid, (bg ? BG : FG), cmdline); 

    /* 步骤 4：解除屏蔽，让挂起的信号“喷涌而出” */
    // 此时如果子进程早已结束，handler 会在这一行代码执行完后立即触发
    sigprocmask(SIG_SETMASK, &prev_one, NULL); 

    if (!bg) {
        waitpid(pid, NULL, 0); // 或者使用 waitfg
    } else {
        printf("[%d] (%d) %s", pid2jid(pid), pid, cmdline);
    }
}
```

在写到这里的时候，发现前台的处理有问题，因为显式地调用了 waitpid,所以不会将前台的程序交给 handler 来处理，优解是将前台的 waitpid 替换为一个 waitfg 函数，只进行 sleep，以中断程序并且不抢占 waitpid：

```c
void waitfg(pid_t pid) {
    sigset_t empty;
    sigemptyset(&empty);
    
    // 只要前台进程还在，就“挂起”等待信号
    while (fgpid(jobs) == pid) {
        // sigsuspend 会暂时清空屏蔽位并让进程休眠
        // 直到捕捉到一个信号并从 handler 返回
        sigsuspend(&empty); 
    }
}
```

## 06 SIGINT Ctrl+C 终止前台进程

由于已经安装了 sigint_handler，所以只需要进行一次信号转发，将接收到的 SIGINT 转换为传递给前台的 kill 函数。但是由于前台进程可能会 fork 出很多个进程来执行操作，所以需要用 kill(-pid,SIGINT) 来进行组 kill

```c
void sigint_handler(int sig) {
  int old_errno = errno;
  pid_t pid = fgpid(jobs);
  if(pid != 0){
    kill(pid,SIGINT);
  }
  errno = old_errno;
}
```

但是这只是发送了关闭信号，题目要求关闭以后输出信息，这就需要将逻辑添加到 sigchld_handler 当中。用 status 来记录退出的信息，用 WIFSIGNALED、WIFEXITED 来判断。

```c
void sigchld_handler(int sig) {
  int old_errno = errno;
  int status;
  pid_t pid;

  while ((pid = waitpid(-1, &status, WNOHANG | WUNTRACED)) > 0) {
    if(WIFSIGNALED(status)){
      // 被异常信号终止
      printf("Job [%d] (%d) terminated by signal %d\n", 
                   pid2jid(pid), pid, WTERMSIG(status));
    }
    else if(WIFEXITED(status)){
      // ...
    }
    deletejob(jobs, pid);
  }
  errno = old_errno;
}
```

## 07 SIGINT 只发送给前台

在 06 已经完成

## 08 SIGTSTP 只发送给前台

写到这里发现有点混淆了 `sigstp_handler` 和 `sigchld_handler`，chld 是用来收尸的，而 sigstp 是用来发送信号的。当信号给到 shell 的时候，通过 stp 发送给前台进程，然后进程挂起，发送信号给 chld。

```c
void sigtstp_handler(int sig) { 
  int old_errno = errno;
  pid_t pid = fgpid(jobs);
  if(pid != 0){
    kill(-pid,SIGTSTP);
  }
  errno = old_errno;
}
```

但是在验证的时候发现 sigstp 一直无法正常输出，最后问了 AI 才发现是在生成子进程后，没有和父进程分开 pgid,即还需要给子进程 `setpgid(0, 0);`,否则 C-z 会发送给所有进程，导致 shell 将自己也挂起了（因为 kill(-pid,...)) ？