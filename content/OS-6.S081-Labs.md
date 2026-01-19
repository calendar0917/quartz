---
id: OS-6.S081-Labs
aliases: []
tags: []
---

## 环境搭建

先克隆仓库，但是发现 `make qemu` 报错了，最后发现是库的导入顺序错了，因为默认的代码格式化插件将倒入库的顺序按字母排序，可 xv 对这方面没有优化，所以要先倒入 type.h，因为数据类型定义在里面，所以要改 `.clang-format`:

```yaml
---
SortIncludes: false
```

可能需要配 clangd 环境，不然 LSP 会有问题。

python 3.13 更新了库，需要将 gradelib.py 中的 pipes 个改成 shlex

需要将 makefile 里的 -Werror 参数删掉，不然 sh 的死循环过不了编译

较新的编译器会对函数指针匹配更严格，打开 `user/usertests.c` 给 `rwsbrk` 加上 `char* s` 参数名

## Lab01

第一章的 Lab 主要是了解系统调用的使用，并初步接触 xv 系统程序的编写。

### Sleep

由于不能用系统自带的 stdlib 等库，所以只能包含 `"user.h"` 等已经实现的，如果要自己添加（如 is_digit 等），需要在 userlib.c 当中编写，然后再 user.h 里声明。

实现起来并不困难，但是需要考虑清楚各种情况

```c
#include "user/user.h"
int main(int argc, char *argv[]) {
  // 错误检查
  if (argc != 2) {
    printf("Usage: sleep <sleeptime>\n");
  }
  char *p = argv[1];
  while (*p) {
    if (*p < '0' || *p > '9') {
      printf("sleep: invalid time interval %s\n", argv[1]);
      exit(1);
    }
    p++;
  }
  int time = atoi(argv[1]);
  // 成功，调用 sleep
  if (sleep(time) < 0) {
    printf("sleep failed\n");
    exit(1);
  }
  exit(0);
}
```

### pingpong 管道操作

由于要双方通信，所以需要两个管道。注意不用的管道要关闭，父进程需要 wait 子进程，以便回收。

```c
// 管道操作示例
// 子进程接收后再发送给父进程
#include "user.h"
int main(int argc, char *argv[]) {
  // 创建管道
  int p[2], pid, p2[2];
  pipe(p);
  pipe(p2);
  pid = fork();
  if (pid == 0) {
    // 子进程，接收，关闭写端
    char buf[8];

    close(p[1]);
    close(p2[0]);
    read(p[0], buf, 4);
    buf[4] = '\0';
    int child_pid;
    child_pid = getpid();
    printf("%d: received %s\n", child_pid, buf);
    close(p[0]);
    // 向父进程发信息
    write(p2[1], "pong", 4);
    close(p2[1]);
    exit(0);
  } else {
    // 父进程，发送，关闭读端
    char buf[8];
    close(p[0]);
    close(p2[1]);
    write(p[1], "ping", 4);
    close(p[1]);
    // 父进程，接收
    int par_pid;
    par_pid = getpid();
    read(p2[0], buf, 4);
    buf[4] = '\0';
    printf("%d: received %s\n", par_pid, buf);
    close(p2[0]);
    wait(0); // 要等待子进程结束,回收进程
  }
  exit(0);
}
```

### primes

比较有难度，思路比较难想。目标是通过管道来制作一个素数筛。管道的作用是在进程间传输数据，所以可以考虑将合法的素数在多个进程间不断过滤。难点在于多个管道的处理顺序,在管道使用完以后就要及时关闭。以及子进程要递归 wait。

```c
#include "user.h"

// 由于子进程有多级过滤，故用递归来解决
void seive(int *p_left) {
  // 输出传来的第一个数，一定是素数
  int first;
  if (read(p_left[0], &first, 4)) {
    printf("prime %d\n", first);
  } else {
    close(p_left[0]);
    exit(0);
  }
  int pid, n, p_right[2];
  pipe(p_right);
  pid = fork();
  if (pid == 0) {
    // 下一个过滤器
    close(p_right[1]);
    seive(p_right);
  } else {
    close(p_right[0]);
    while (read(p_left[0], &n, 4)) {
      if (n % first != 0) {
        write(p_right[1], &n, 4);
      }
    }
    close(p_left[0]);
    close(p_right[1]);
    wait(0);
  }
}

int main() {

  // 主进程创建所有数，并向下转发
  int p[2];
  pipe(p);

  int pid;
  pid = fork();
  if (pid == 0) {
    // 子进程，读取父进程传递来的数据
    close(p[1]);
    seive(p);
  } else {
    // 父进程，向下转发
    close(p[0]);
    for (int i = 2; i <= 35; i++) {
      write(p[1], &i, 4);
    }
    close(p[1]);
    wait(0);
  }
  exit(0);
}
```

### find

比较难的 Lab，主要的难点在于字符串的指针式处理、路径的字符串拼接。以及文件夹 - 文件的思想转换。

首先是 `open(path,0)` 的意义，如果 path 指向的是一个文件夹，那么 open 得到的是一个 dirent 流！dirent 则包含 inum、name。这里需要知道，操作系统中 name 并不能代表这个文件，文件的唯一标识是 inode-number。

而 `stat(path,&st)` 的作用是将 path 这个路径指向的信息存储到 st 当中，可以获取更多信息。`fstat(fd,&st)` 和 stat 的差别在于开销更小。

文件描述符记得随用随关

所以，整体的思路是：
- open 指定 path
- 用 `stat(path,&st)` `st.type` 来判断 path 指向的类型
- 如果是文件，比较文件名（需要从完整的 path 中提取出文件名）
- 如果是文件夹，拼接路径以后递归搜索（要注意排除 .\.. 的路径，否则会死循环）

```c
#include "kernel/types.h"
#include "kernel/fs.h"
#include "kernel/stat.h"
#include "user.h"

char *getname(char *path) {
  char *p;
  // 找到末尾 /
  for (p = path + strlen(path); p >= path && *p != '/'; p--) {
    ;
  }
  p++;
  return p;
}

void find(char *path, char *filename) {
  struct stat st;
  char buf[512], *p;
  int fd;
  ;
  struct dirent de;

  // 文件路径检查
  if ((fd = open(path, 0)) < 0) {
    fprintf(2, "find: cannot open %s\n", path);
    exit(0);
  }

  if (fstat(fd, &st) < 0) {
    close(fd);
    return;
  }
  // 终止情况，找到文件，判断
  if ((stat(path, &st) < 0))
    return;
  if (st.type == T_FILE) {
    // 获取文件名作比较
    if (strcmp(getname(path), filename) == 0) {
      printf("%s\n", path);
    }
    close(fd);
    return;
  }
  if (st.type == T_DIR) {
    // 拼接缓冲区,先存父路径,防止缓冲区溢出
    if (strlen(path) + 1 + DIRSIZ + 1 > sizeof(buf)) {
      close(fd);
      return;
    }
    strcpy(buf, path);
    p = buf + strlen(buf);
    *p++ = '/';

    while (read(fd, &de, sizeof(de)) == sizeof(de)) {
      // 过滤死循环
      if (de.inum == 0 || strcmp(de.name, ".") == 0 ||
          strcmp(de.name, "..") == 0) {
        continue;
      }
      // 拼接路径，在 p 的位置覆盖新名称
      memmove(p, de.name, DIRSIZ);
      p[DIRSIZ] = 0;
      // 递归
      find(buf, filename);
    }
  }
}

int main(int argc, char *argv[]) {
  char *path = argv[1];
  char *filename = argv[2];

  // 输入检查
  if (argc != 3) {
    printf("Usage: find <path> <filename>");
  }

  // 调用函数
  find(path, filename);
  exit(0);
}
```

### xargs

比较有意思的功能。比如 `echo world | xargs echo hello`，能够将 world 作为后面命令的参数。这在 rm 之类的命令很有用，可以批量处理。

但是实现起来会有一点歧义。一开始以为只有 echo 会被重复执行，但实际上重复执行的是 echo hello，所以只需要在原参数列表的最后拼接上前面管道来的就可以了。

实现的思路是：
- 用 buf 处理参数列表，整合给 xargs 后的命令
- 注意前面的输出可能包含多行，也就需要每一行执行一次，这就要求将 `\n` 替换为 `\0`，以便存入 argv 的是标准的字符串

难点还是在于字符串的处理，对各种指针形式还是有点混淆。比如 `char* argv[]` 是字符串数组，所以最后要新建一个字符串数组来存储而非字符串。以及每次只需要更新 new_argv 的最后一项即可，因为前面的参数都是要重复执行的。

[[C语言的指针]]

```c
#include "kernel/param.h"
#include "user.h"

int main(int argc, char *argv[]) {
  if (argc < 1) {
    printf("Usage: ... | xargs <application> args");
  }
  // 接收前面管道来的参数，由于有可能传递多行过来，所以需要用 \n 来截断
  // 接收到一行就将其执行一次
  // 这样就不用考虑转变参数列表的问题了
  char buf[512];
  char *new_argv[MAXARG]; // 新的参数列表
  int i = 0;

  // 预填充
  for (int j = 1; j < argc; j++) {
    new_argv[j - 1] = argv[j];
  }
  while (read(0, &buf[i], 1) > 0) {
    if (buf[i] == '\n') {
      // 读取了一行
      buf[i] = 0;
      new_argv[argc - 1] = buf;
      new_argv[argc] = 0;
      int pid = 0;
      pid = fork();
      if (pid == 0) {
        // 子进程，执行
        exec(argv[1], new_argv);
        printf("xargs: exec failure");
        exit(1);
      } else {
        wait(0);
      }
      i = 0; // 从头开始读取下一个参数
    } else {
      i++;
    }
  }
  exit(0);
}
```

## Lab02
### trace 系统调用的实现

实验已经提供了 user/trace.c，需要完成的是实现 trace 程序的执行过程。打开 trace.c 可以看到 trace() 报错了，因为这个系统调用还没有在 user.h 当中声明。trace.c 实现的是，`trace <mask> <command> ...`，trace.c 在 exec command 之前会调用 trace(mask)，以标记要追踪的系统调用。

所以需要实现的就是 `trace(mask)` 这里对系统调用的追踪。追踪需要特定的标记位，所以要给 `kernel/proc.h` 的 `struct proc` 结构体里加一个 `int trace_mask`。然后要编写 `kernel/sysproc.c` 当中的 sys_trace 函数。

但是编写函数还不够，还要考虑为什么 trace.c 调用了 `trace(num)` 就可以找到 `sys_trace` 呢？为了建立这样的联系，需要在 `user/usys.pl` 当中加入 `entry("trace")`，这样，调用 trace 以后就会跳转到 perl 脚本生成的一段汇编代码当中，寻找 trace 对应的系统调用编号（SYS_trace)，然后 ecall 执行，从用户态跳转到内核态。

```asm
.global trace
trace:
 li a7, SYS_trace    # 将系统调用编号（比如 22）放入 a7 寄存器
 ecall               # 触发硬件中断，跳入内核
 ret
```

所以还需要
- 在 `kernel/syscall.h` 当中建立 trace 对应的编号
- 在 `kernel/syscall.c` 的数组里建立编号-函数的映射
- 在 `kernel/sysproc.c` 当中写出具体逻辑
- syscall.c 负责检查、打印，sys_trace 负责设置标记位
- 修改 **makefile** 以编译 trace.c

编写 sys_trace 时要注意，这个系统调用不是像普通函数一样传递参数，而是通过 `myproc()` 来获取上下文以后，直接读取寄存器里的参数。

```c
uint64 sys_trace() {
  int mask;
  struct proc *p = myproc();
  if(argint(0, &mask) < 0){
    return -1;
  }
  p->trace_mask = mask;
  return 0;
}
```

编写 sys_call.c 时要注意，需要维护一个字符串数组以实现追踪对象的输出

```c
void syscall(void) {
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if (num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
    // 执行完 syscall 的具体调用以后，判断这个调用是否被标记
    if ((1 << num) & p->trace_mask) {
      printf("%d: syscall %s -> %d\n", p->pid, syscall_names[num],
             p->trapframe->a0);
    }
  } else {
    printf("%d %s: unknown sys call %d\n", p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

sys_trace，只负责修改标志位

```c
uint64 sys_trace() {
  int mask;
  struct proc *p = myproc();
  if (argint(0, &mask) < 0) {
    return -1;
  }
  p->trace_mask = mask;
  return 0;
}
```

最后，还需要保证 fork() 系统调用在执行时，能够给孩子继承跟踪状态，修改 fork() 函数：

```c
// kernel/proc.c -> fork()
np->trace_mask = p->trace_mask;
```

整体来说不是很难，但是细节很多。需要对用户态-内核态有一个更清晰的认识。程序的运行大致是：
- trace.c 编译出的 trace 运行
- 运行后调用了 trace(num) 系统调用
- syscall(trace) 调用 `sys_proc.c` 当中的 sys_trace，修改标志位
- trace.c 继续运行，exec(target,argv)
- 由于是同一个进程，所以标志位不会被改变，执行 sys_call(target) 后会判断，如果被标记则输出追踪结果。

### sysinfo 记录内存、进程信息

> 需要像前一个 lab 一样补充基础信息声明。

要求是完成 sysinfotest.c 的测试，目标是统计剩余内存和已有进程。统计内存的函数要放在 `kernel/kalloc.c` 当中，统计进程的函数要放在 `kernel/proc.c` 当中。

读源码可以发现 run、kmem 这两个结构体，而 kmem 当中有 freelist 的链表。也就是说，空余的空间被作为一个链表存储了。所以只需要遍历、记录链表的总节点数，再乘上每个节点代表的物理空间大小就可以求出剩余大小了。

```c
uint64
count_free_mem(void){
  struct run *r;
  uint64 count = 0;
  acquire(&kmem.lock);
  r = kmem.freelist;
  while(r){
    count++;
    r = r->next;
  }
  release(&kmem.lock);
  return count * PGSIZE;
}
```

写完以后还不够，需要将函数放到 `kernel/defs.h` 中声明，这样 `kernel/sysproc.c` 中的 sysinfo 就可以调用该函数了。

类似地，还需要在 `kernel/proc.c` 当中编写 `count_active_procs`,在 proc.h 中可以看到进程结构体的信息，其中有 state 记录了进程的状态，而 proc.c 当中有 `struct proc proc[NPROC];` 的声明，也就是所有的进程都存储到一个数组里了，所以只要遍历这个数组就可以实现进程统计。

```c
uint64 count_active_procs(void) {
  int n = 0;
  for (int i = 0; i < NPROC; i++) {
    if (proc[i].state != UNUSED) {
      n++;
    }
  }
  return n;
}
```

写完了辅助函数以后，要怎么组合起来呢？当然是要整合到 sysinfo 这个系统调用里，但是系统调用是运行在内核态的，读取参数、返回参数给用户态都需要特定的方式。由于传递的是地址，所以需要 `argaddr` 来读取地址->存入内核态，返回时，需要 `copyout` 到指定进程的页表地址（因为用户态的进程是虚拟化的）

```c
int sys_sysinfo(void) {
  struct sysinfo info;
  info.freemem = count_free_mem();
  info.nproc = count_active_procs();
  uint64 addr;
  if (argaddr(0, &addr) < 0) {
    return -1;
  }
  struct proc *p;
  p = myproc();
  if (copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0) {
    return -1;
  }
  return 0;
}
```

整体而言更复杂了一些，一开始对声明、实现要放到哪里都不太清晰，而且源码里存储内存、进程状态的内容也是问了 AI 才知道的。sysinfo 的实现逻辑如下：
- 先在 user.h 当中声明系统调用、结构体，让用户态可以使用
- 然后再 kernel/sysinfo.h 当中实现结构体
- 将系统调用注册到 syscall 的列表里，并且 extern
- 开始在 kernel/sysproc.c 当中编写 sys_sysinfo，发现需要统计内存、进程
- 统计内存的放到 kernel/kalloc.c,进程的放到 kernel/proc.c
- 最后通过 argaddr 读取用户态的地址参数，copyout 到指定进程的地址处

## 