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

## Lab03
### USYSCALL 分配页表空间

由于经常需要查询进程的 pid,如果每次查询都要经过内核翻译（更正：并不是不会翻译，而是不会切换上下文,让用户在自己的页表里就可以 getpid）会导致性能损失，所以考虑用一个页表来固定存储这部分信息。USYSCALL 是一块虚拟内存，内核和用户都可以访问（但只有内核能写），如果这块虚拟内存直接映射到了一个页表当中，就可以无需翻译直接访问，从而提高性能。

既然要分配一块内存，就要考虑声明-划分空间-释放时清除。

对于声明来说，直接在 proc 对应的结构体当中声明一个指向 usyscall 的指针即可。但是这个指针还需要对应的空间分配。

划分空间的任务要放到进程的初始化，即 `proc.c` 的 alloproc 函数当中。可以看到前面已经分配了 trapframe，所以需要再用 kalloc 分配一个页表给 usyscall.

```c
static struct proc *allocproc(void) {
  struct proc *p;

  for (p = proc; p < &proc[NPROC]; p++) {
	// 寻找没分配的 proc 空间，goto found
  }
  return 0;

found:
  p->pid = allocpid();
  p->state = USED;

  // Allocate a trapframe page.
  if ((p->trapframe = (struct trapframe *)kalloc()) == 0) {
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // 在初始化时，分配一个页表给 usyscall
  if ((p->usyscall = (struct usyscall *)kalloc()) == 0) {
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  p->usyscall->pid = p->pid;

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if (p->pagetable == 0) {
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```

分配完了以后还不够，还需要对这个页面进行映射，这里的 p->usyscall 是由 kalloc 分配的物理空间，得转到进程对应的页表当中。所以还要修改 proc_pagetable。这里添加映射时要注意，如果添加失败，需要将前面添加的映射删除，不然这个页表可能空间被释放了，但是仍然不能使用。

```c
pagetable_t proc_pagetable(struct proc *p) {
  pagetable_t pagetable;

  // An empty page table.
  pagetable = uvmcreate();
  if (pagetable == 0)
    return 0;

  // map the trampoline code (for system call return)
  // at the highest user virtual address.
  // only the supervisor uses it, on the way
  // to/from user space, so not PTE_U.
  if (mappages(pagetable, TRAMPOLINE, PGSIZE, (uint64)trampoline,
               PTE_R | PTE_X) < 0) {
    uvmfree(pagetable, 0);
    return 0;
  }

  // map the trapframe just below TRAMPOLINE, for trampoline.S.
  if (mappages(pagetable, TRAPFRAME, PGSIZE, (uint64)(p->trapframe),
               PTE_R | PTE_W) < 0) {
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

  // 注意映射失败时的释放顺序
  if (mappages(pagetable, USYSCALL, PGSIZE, (uint64)(p->usyscall),
               PTE_R | PTE_U) < 0) {
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmunmap(pagetable, TRAPFRAME, 1, 0);
    uvmfree(pagetable, 0);
	return 0;
  }
  return pagetable;
}
```

映射添加完，就是进程终止时释放的逻辑了。在 proc_freepagetable 当中，要将 usyscall 的 map 解除。

```c
void proc_freepagetable(pagetable_t pagetable, uint64 sz) {
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmunmap(pagetable, USYSCALL, 1, 0);
  uvmfree(pagetable, sz);
}
```

映射解除还不够，要将对应的物理地址空间释放，在 freeproc 当中添加：

```c
if (p->usyscall)
    kfree((void *)p->usyscall);
```

还是比较复杂的，有些找不到函数。但是逻辑基本理顺了，分配空间-建立映射-删除映射-释放空间。

### vmprint 打印进程信息

要求实现一个在系统启动时能打印页表的函数。感觉不太难，但是在实现的时候还是遇到了比较多的问题，思路也不太清晰。

首先是不知道怎么查看进程对应的地址，提示说看可以参考 freewalk，其实就是用递归，然后取 `pte = pagetable[i]` 来读取地址。实现 _vmprint 辅助函数如下：

```c
void _vmprint(pagetable_t pagetable, int level) {
  for (int i = 0; i < 512; i++) {
    pte_t pte = pagetable[i];
    if (pte & PTE_V) {
      // 根据深度打印缩进
      for (int j = 0; j < level; j++) {
        printf(".. ");
      }
      // 打印当前 pte 索引
      uint64 child = PTE2PA(pte);
      printf("%d: pte %p pa %p\n", i, pte, child);
      // 非叶子则递归
      if ((pte & (PTE_R | PTE_W | PTE_X)) == 0) {
        _vmprint((pagetable_t)child, level + 1);
      }
    }
  }
}
```

可能疑惑的点在于，不知道为什么会传一个 pagetable 进来。逻辑还是比较清晰的：遍历每层的 pagetable，如果不为空，就递归打印。用 level 来控制缩进。判断是否是叶子的逻辑有点不懂。

由于还需要打印查询的页表号，所以实现 vmprint:

```c
void vmprint(pagetable_t pagetable) {
  printf("page table %p\n", pagetable);
  _vmprint(pagetable, 1);
}
```

其实看一下调用就会比较清晰了,其实就是一个进程对应一个页表，所以查询的时候直接查对应页表就可以了……

```c
if (p->pid == 1)
    vmprint(p->pagetable);
```
### pgaccess 参数获取

目标是实现一个能够查看某个 pte 是否被访问过的函数。思路还是比较清晰的，查看进程对应页表 -> 用 walk 进行访问 -> 判断 PTE_A 标志位。

主要的原理是，PTE_A 是底层硬件会控制的标志位，如果某个页表被访问就会置一。但是这个 PTE_A 需要自己声明在 `riskv.h` 当中声明才能使用：

```c
#define PTE_A (1L << 6) // 1 -> user can access
```

看一下测试函数的调用： `if (pgaccess(buf, 32, &abits) < 0)` ，传入的 buf 是所查页表的首地址（刚用 malloc 分配），32 是需要查的页表数，abits 是存储返回信息的掩码，由于是 32 个页表，所以每个页表对应一位，如果置 1 就表示被访问过了。

pgaccess 实现时，需要用 argaddr、argint 来接收参数，接收了以后，循环遍历这 32 个页表，用 walk 找到对应的 物理地址，然后检查 PTE_A 标志位，如果是 1，就修改 mask,同时置 0.

```c
int sys_pgaccess(void) {
  // lab pgtbl: your code here.
  // 用 walk 寻找对应标志位，判断是否被访问过，然后取消标志
  // 读取参数 buf num addr
  uint64 va;
  int n_pages;
  uint64 addr;
  uint mask;
  mask = 0;
  if (argaddr(0, &va) < 0)
    return -1;
  if (argint(1, &n_pages) < 0)
    return -1;
  if (argaddr(2, &addr) < 0)
    return -1;

  if (n_pages > 32)
    n_pages = 32;
  struct proc *p = myproc();
  for (int i = 0; i < n_pages; i++) {
    // 记录当前页面的地址
    uint64 current_va = va + i * PGSIZE;
    // walk 按页寻找，0 表示如果没映射，返回空指针
    pte_t *pte = walk(p->pagetable, current_va, 0);
    // 检查记录
    if (pte != 0 && (*pte & PTE_V) && (*pte & PTE_A)) {
      // A 位是 1，说明已经被访问过，记录
      mask |= (1 << i);
      *pte &= ~PTE_A;
    }
  }
  if (copyout(p->pagetable, addr, (char *)&mask, sizeof(mask)) < 0) {
    return -1;
  }
  return 0;
}
```

实现起来还是比较清晰的，但是思路有点难出，各种已经封装好的函数不太熟悉，传值、接收参数的函数也不了解……解法是，先参考内核的源码，知道已经实现了什么函数（在 defs.h 里），对于不熟悉的内容先参考其他函数的实现方法。

总结一下 pagetable 的内容。每一个进程对应了一个 pagetable,在 xv6 当中，每个页表有 512 个 pte，共有三级，进程可以查询的就有 `512*512*512` 个地址（也就是虚拟地址）。在进程查询指定虚拟地址时， satp 寄存器会切换到该进程对应的页表，该页表的第三级就存储了对应的物理地址。

本章 Lab 的目的就在于体会内核对虚拟-物理地址的操控。在进程开始时，内核可以给对应的页表分配空间（USYSCALL），以实现访问加速。还可以访问对应物理地址的信息（vmprint），打印出虚拟页号对应的页表。内核中已经实现的 walk 函数可以将虚拟地址转换为对应的物理地址，从而读取其中的各种标志位信息（pgaccess）。

## Lab04

### 读汇编代码
### Backtrace

为了便于调试程序、检查错误，可以实现一个 backstrace 函数来输出函数调用层级。在此之前，需要了解一下栈帧的结构。

每个进程都会被分配一个 page,其中会有栈帧，而函数调用会给每一个函数划分新的栈帧。惯例是，fp 指向栈顶，`fp-8` 指向返回地址（也就是调用本函数的函数地址），`fp-16` 指向上一个函数的栈顶。所以，我们需要读取 fp-8 之后输出，然后将当前的 fp 变为 fp-16,再继续输出。终止的条件是和 `PGROUNDUP(fp)` 来比较，意味包含 fp 这个指针的内存页边界。

题目提供了 r_fp 汇编内联函数，用来从寄存器当中读取当前的 fp：

```c
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}
```

backtrace 实现如下，将其放到需要检查的地方就可以了：

```c
void backtrace(void) {
  printf("backtrace:\n");
  // 解析调用了当前函数的所有函数
  // 先获取栈顶指针
  uint64 fp = r_fp();
  // 获取栈的最高地址边界（终止条件）
  uint64 top = PGROUNDUP(fp);
  // 按照布局读取 ra 和 pre_fp
  while (fp < top) {
    uint64 ra = *(uint64 *)(fp - 8);
    uint64 pre_fp = *(uint64 *)(fp - 16);
    printf("%p\n", ra);
    fp = pre_fp;
  }
}
```

这里有一个假设，也就是每一个内核栈的大小都在一个页面以内，所以才可以和页面顶进行比较。且实际的使用中，应该把这个函数在 panic 后立即调用，方便查看问题来源

### Alarm

要求实现一个用户可以调用的 `sigalarm(n,handler)` 系统调用，作用是在指定时间后自动执行一个函数。

实现的思路是，给进程设置 alarm 相关属性，用于记录时间、目标函数等等。但是问题在于，时间到了以后，这个进程会被强制中断然后执行函数，这期间需要小心地保护好原进程，并在函数执行完以后能够还原。

首先是原理。在设置了 sigalalarm 后，不像之前的 trap lab 一样直接通过系统调用进入中断，而是要通过计时进入中断，也就是**时钟中断**。能实现的原因是，其实进程每个时钟都会进行时钟中断（否则死循环无法退出），只要在这个判断的后面加上 alarm 的判断就可以了。时钟中断和前面的 syscall trap 类似，都是在 trap.c 当中处理的，区别在于 `which_dev == 2`，意味从 CPU 来的中断。

那么到具体实现，首先要做的是考虑进程的属性要怎么修改，中断需要保存原来的 trapframe，需要记录经过的时间、总时间，需要记录是否已经进入了 handler（否则可能重复进入）。

```c
int alarm_interval;         // 闹钟间隔（n）
  void (*alarm_handler)();    // 闹钟函数指针
  int ticks_count;            // 距离上次响铃过了多少滴答
  struct trapframe *alarm_tf; // 【关键】用来备份响铃前的寄存器状态
  int is_alarm_running;       // 防止闹钟函数里再触发闹钟（重入保护）
```

然后是要实现 trap.c 当中的 usertrap 函数，当时钟中断导致控制权返回到内核时，需要做什么：

```c
if (which_dev == 2) {
    if (p->alarm_interval > 0) {
      p->ticks_count++; // 时间++
      if (p->ticks_count == p->alarm_interval && p->is_alarm_running == 0) {
        p->is_alarm_running = 1; // 防止重入
        memmove((void *)p->alarm_tf, p->trapframe,
                sizeof(struct trapframe));            // 保存现场
        p->trapframe->epc = (uint64)p->alarm_handler; // 跳转
        p->ticks_count = 0;
      }
    }
    yield();
  }
```

上面实现的时候一开始犯了个错，即 *alarm_tf 的空间分配。应该在进程初始化的事就将空间分配好（在结束的时候释放），如果在这里分配，会导致每次执行都分配出一块空间，但实际上只需要一块。以及，移交控制权的方式是直接修改 epc，这样在中断后可以改变程序流向。

接着是系统调用的书写。对于 sysalarm(n,handler) 来说，要做的就是将进程的 alarm 属性设置好：

```c
uint64 sys_sigalarm(void) {
  int n;
  uint64 handler;
  if (argint(0, &n) < 0 || argaddr(1, &handler) < 0) {
    return -1;
  }
  struct proc *p = myproc();
  p->alarm_interval = n;
  p->ticks_count = 0;
  p->alarm_handler = (void (*)())handler;
  return 0;
}
```

对于 sigreturn 来说，这是在 handler 执行完以后需要执行的（用户需要主动放到 handler 函数的返回前面），作用在于将执行权在转回到调用 handler 之前。关键是将原来保存 trapframe 复原：

```c
uint64 sys_sigreturn(void) {
  struct proc *p = myproc();
  memmove(p->trapframe, p->alarm_tf, sizeof(struct trapframe));
  p->is_alarm_running = 0;
  return p->trapframe->a0;
}
```

可以看到，最后的返回值有些特殊。为什么要返回 a0 呢？其实，sigreturn 也是一个系统调用，所以会进入到系统调用的中断，a0 表示的是系统调用的返回值。我们的目的是将寄存器全部复原为调用 handler 之前的样子，所以如果 sigreturn 的返回值修改了 a0,就会导致 a0 这个寄存器改变了！解决方法是，返回 a0 给 a0 这个寄存器，就完美符合要求。

## Lab05
### Copy-On Write

目标是实现写时复制。感觉难度上来了，涉及的过程比较多，要处理、考虑的也更细致了。写到最后 grade 过不了，先写 WP 复盘一下过程。

首先明确一下原理。写时复制指的是在调用 fork() 以后，先不划分出一块物理内存给子进程，而只是先将子进程的页表指向父进程的物理地址，等到父或子进程要写内存时，再触发 page fault 进行页面的分配。

原理很简单，但是实现起来就不一样了。首先，要弄清楚 fork 之后会做什么。伪代码如下：

```c
1. 分配进程号给子进程
2. 调用 uvmcopy
// Copy user memory from parent to child.
(uvmcopy(p->pagetable, np->pagetable, p->sz)
3. 复制其他信息给子进程
```

所以关键在于这里的 uvmcopy,会将父进程的内存复制到子进程里，如果不写时复制的话，在这里就直接分配了。修改 uvmcopy，主要内容在于设置标志位。

```c
int uvmcopy(pagetable_t old, pagetable_t new, uint64 sz) {
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for (i = 0; i < sz; i += PGSIZE) {
    if ((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if ((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if (flags & PTE_W) {
      flags &= ~PTE_W;  // 禁止写入
      flags |= PTE_COW; // 标记为 COW 页（需要先在 riscv.h 定义这个位）
    }
    // 这一步至关重要！否则父进程依然可以直接修改共享内存
    *pte = PA2PTE(pa) | flags;
    // 直接将新页的物理页指向旧页
    if (mappages(new, i, PGSIZE, pa, flags) != 0) {
      goto err;
    }
    ref_add(pa);
  }
  return 0;

err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```  

到这里就懵了，前面的基础没有打牢，看各种 pagetable 的操作都有一点生，需要再复习一下要点。
- pte （这里指最后一层）指的是页表项，可以读出 PA 和权限（flags）
  - 可以用 `PA2PTE` 转换得到
  - 可以用 `PTE2PA` 得到对应的物理地址
  - 可以用 `walk(pagetable,i,0)` 在多级页表找到对应的 pte，1 表示会新建页，0 表示不新建
- pa 指的是物理地址，存储的是具体数据
  - 用 mappages
- va 指的是虚拟地址，需要用 
  - 映射是以页为基准的，所以需要 `PGROUNDDOWN` 来取整
- 位运算
  - `*pte & PTE_X` 检查
  - `*pte |= PTE_X` 设置
  - `*pte = PA2PTE(pa) | perm | PTE_V;` 根据 pa 设置页表项
  - `*pte &= ~PTE_X` 禁止

在继续写以前，会发现这里的引用计数还没有实现。要怎么实现呢？首先，这里的引用计数针对的是某块物理地址，如果有 pte 指向物理地址，就要增加引用计数。所以考虑用一个全局数组来记录，数组需要的大小为 物理大小/页大小，使得可以定位每一个页面的计数。还需要定义辅助函数，来对数组进行修改：

```c
struct {
  struct spinlock lock;
  int count[PHYSTOP / PGSIZE];
} mem_ref;

#define PA2INDEX(pa) (((uint64)pa) / PGSIZE)

void ref_add(uint64 pa) {
  if (pa >= PHYSTOP)
    return; // 安全检查
  acquire(&mem_ref.lock);
  mem_ref.count[PA2INDEX(pa)]++;
  printf("add ref: pa %p, count %d\n", pa, ref_get(pa));
  release(&mem_ref.lock);
}

int ref_get(uint64 pa) { return mem_ref.count[PA2INDEX(pa)]; }

// 建议直接在 kfree 中处理减少逻辑，或者写一个专用的 dec
int ref_dec(uint64 pa) {
  int c;
  acquire(&mem_ref.lock);
  mem_ref.count[PA2INDEX(pa)]--;
  c = mem_ref.count[PA2INDEX(pa)];
  release(&mem_ref.lock);
  return c;
}
```

重点在锁的使用。可能有多个进程同时在操作数组，所以需要将操作原子化，加一个自旋锁。

现在进程的创建部分已经做好了，接下来要做的就是对父/子进程的修改。如果有某个进程要被修改，应该要触发 pagefault（因为上面修改了权限），然后进入处理逻辑（trap.c）。

```c
else if (cause == 15) {
    if (uvmcowhandler(p->pagetable, r_stval()) < 0) {
      p->killed = 1;
}
```

中断号是 15,所以给 cause15 一个分支。这个分支的目的是，如果 pagefault 是写时复制引起的（通过标志位判断），就分配一个新的物理页面给子进程。将后面这个抽取出一个函数 uvmcowhandler：

```c
int uvmcowhandler(pagetable_t pagetable, uint64 va) {
  // 地址合法性判断
  va = PGROUNDDOWN(va);
  pte_t *pte = walk(pagetable, va, 0);
  if (pte == 0) {
    return -1;
  }
  if ((*pte & PTE_V) == 0 || (*pte & PTE_COW) == 0) {
    return -1;
  }
  // 获取页表头地址
  uint64 pa = PTE2PA(*pte);
  // 如果物理页的计数是 1,无需申请新页
  if (ref_get(pa) == 1) {
    *pte |= PTE_W;
    *pte &= ~PTE_COW;
    return 0;
  }
  // 否则，多人共享，需要申请
  char *mem = kalloc();
  if (mem == 0)
    return -1;
  memmove(mem, (char *)pa, PGSIZE);
  // 修改页表，指向新物理页,调整权限
  uint flags = PTE_FLAGS(*pte);
  flags |= PTE_W;
  flags &= ~PTE_COW;
  // 新表覆盖旧映射
  // mappages 如果发现已存在映射会报错，所以这里直接修改
  *pte = PA2PTE(mem) | flags;
  // 旧页 -1
  kfree((void *)pa);
  return 0;
}
```

这里又涉及到了 kfree，也需要根据写时复制来修改。首先减去一个引用，如果引用减到 0,才能进行释放物理空间：

```c
void kfree(void *pa) {
  struct run *r;

  if (((uint64)pa % PGSIZE) != 0 || (char *)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  if (ref_dec((uint64)pa) > 0) {
    return;
  } else {
    printf("real free: pa %p\n", pa);
    memset(pa, 1, PGSIZE);
    r = (struct run *)pa;
    acquire(&kmem.lock);
    r->next = kmem.freelist;
    kmem.freelist = r;
    release(&kmem.lock);
  }
}
```

既然修改了 kfree,就要注意到 kinit 时，freerange 其实是调用了 kfree 的，作用是在初始化的时候，将所有的空间都释放。但是由于要写时复制，如果这时直接 kfree,那么原来的引用计数可能会变为 -1，所以需要在 kfree 之前将计数记为 1：

```c
void freerange(void *pa_start, void *pa_end) {
  char *p;
  p = (char *)PGROUNDUP((uint64)pa_start);
  for (; p + PGSIZE <= (char *)pa_end; p += PGSIZE) {
    mem_ref.count[PA2INDEX(p)] = 1;
    kfree(p);
  }
}
```

至此，一个比较完整的流程过完了。但是，还有边界条件要考虑。如果写是从内核 copyout 到用户态的，是不会触发 pagefault 的！因为 copyout 的默认实现用的是 walkaddr，也就是直接找、写物理地址，是不会经过内核的判断的。所以还需要修改：

```c
int copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len) {
  uint64 n, va0, pa0;
  pte_t *pte;

  while (len > 0) {
    va0 = PGROUNDDOWN(dstva);
    // 找到对应页表项
    pte = walk(pagetable, va0, 0);
    if (pte == 0 || (*pte & PTE_V) == 0)
      return -1;

    if (*pte & PTE_COW) {
      // 是 cow 位，需要复制
      if (uvmcowhandler(pagetable, va0) < 0) {
        return -1;
      }
    }
    // 处理完 cow 后，再获取最新物理地址
    pa0 = walkaddr(pagetable, va0);
    if (pa0 == 0)
      return -1;

    n = PGSIZE - (dstva - va0);
    if (n > len)
      n = len;

    memmove((void *)(pa0 + (dstva - va0)), src, n);
    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

经过了一番检查，发现 simple 没有通过的原因是在写时复制 kalloc 后，没有设置引用为 1！

接下来就是多进程的同步问题，原因应该是要加锁？

又经过了一番检查，发现其实原本没问题，只是因为有个 printf 没有删掉。但是过程中又接触到几个点：
- ref_get 要不要加锁？测试过后发现，不加锁也是可以的。但是这可能是因为xv6 的页表写保护机制，只允许一个进程来写页表
- kalloc 后引用设置为 1，应该放在哪里？应该要放在 kalloc 里面，分配完以后就要马上置 1.如果放在调用 uvmhandler 后面，可能会导致部分页表没有正确的引用计数，因为不是所有的分配都会走 uvmhandler。比如 exec,sbrk 等等。

## Lab06
### uthread

> 写得不太顺，主要是检查了很久没有检查出逻辑错误……debug 非常低效

目标是实现一个用户态的多线程，其实类似于内核态，关键在于模拟程序间的跳转。

原理部分，跳转的过程是：
- 初始化线程，从而能够触发 schedule 函数，加入到线程组里
- 进入 schedule 以后，会进行轮询，寻找 RUNABLE 进程来 switch 执行
- switch 是汇编语句，用于将各个进程的切换，其实就是将 callee 寄存器保存

因为是 schedule 调用的 switch，所以这里其实相当于是一个函数调用，默认会存储一些寄存器。同样的，只要 callee 的 sp、ra 指针改变了，就可以跳转到对应的线程继续执行。

于是开始实现，不过实现前先看一下 main 函数：

```c
int main(int argc, char *argv[]) {
  a_started = b_started = c_started = 0;
  a_n = b_n = c_n = 0;
  thread_init();
  thread_create(thread_a);
  thread_create(thread_b);
  thread_create(thread_c);
  thread_schedule();
  exit(0);
}
```

可以看到，先有了 init，然后将三个函数作为线程参数传递给了 create。init 的逻辑就是初始化出主线程，主线程虽然不会被调用，但是要考虑到，在进行第一次 switch 的时候，如果没有某个线程和 A 进行交换，那么就会有一个随机的地址被换上去，那么就需要一个线程来和它“交换”。

```c
void thread_init(void) {
  current_thread = &all_thread[0];
  current_thread->state = RUNNING;
}
```

之后，有必要看一下 thread 是怎么定义的：

```c
struct thread {
  struct context context;
  int state;              /* FREE, RUNNING, RUNNABLE */
  char stack[STACK_SIZE]; /* the thread's stack */
};
```

可以看到，一个线程对应了一个 context 和 stack。context 结构体用于保存上下文，而 stack 其实就是一个数组，作用就是充当线程的栈！相当于模拟了一个函数的栈出来。

```c
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  ...
  uint64 s11;
};
```

而 context 重要的地方在于，它保存了线程执行的信息，由于线程是作为函数被调用的，所以记录 ra、sp 就可以切换线程。具体要怎么实现呢？涉及到了寄存器的操作，就得要用汇编代码了：

```asm
thread_switch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        ...
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ...
        ld s11, 104(a1)
        
		ret
```

逻辑其实是很简单的，就是把 callee 寄存器交换，关键是 ra、sp 的交换。有了基本的结构，就可以创建线程了，也就是 create：

```c
void thread_create(void (*func)()) {
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE)
      break;
  }
  t->state = RUNNABLE;
  memset(&t->context, 0, sizeof(t->context)); // 清零
  t->context.ra = (uint64)func;
  uint64 stack_top = (uint64)t->stack + STACK_SIZE;
  t->context.sp = stack_top - stack_top % 16; // 对齐
  t->state = RUNNABLE;
}
```

写到这里的时候其实有点费解，有些想不通这里的**栈**是怎么一回事。还是要画图理解：

```
内存地址 (高位)
    +----------------------------------+  <-- t->stack + STACK_SIZE (数组末尾)
    |      初始对齐填充 (0 ~ 15 Bytes)   |
    +----------------------------------+  <-- 初始 context.sp (16字节对齐处)
    |         线程初始栈帧 (Empty)       |
    |               |                  |
    |               |                  |  栈向下生长 (Push)
    |               V                  |  sp 指针数值减小
    |                                  |
    + - - - - - - - - - - - - - - - - -+
    |          当前运行时的栈帧           |
    |      (局部变量, 函数调用记录等)       |
    +----------------------------------+  <-- 当前 CPU $sp 寄存器
    |               :                  |
    |               : (剩余可用空间)     |
    |               :                  |
    +----------------------------------+  <-- t->stack (数组基地址/地基)
    内存地址 (低位)
```

其实就是要对**由高向低**生长有一个概念，`t->stack` 是指向数组起始位置，也就是模拟的地基，因为 sp 指针会不断减小，所以需要将末位映射为 sp 指针，这也是数组模拟栈的技巧吧。

然后就是 schedule 的书写了，将前面所有内容整合起来。

```c
void thread_schedule(void) {
  struct thread *t, *next_thread;

  /* Find another runnable thread. */
  next_thread = 0;
  t = current_thread + 1;
  for (int i = 0; i < MAX_THREAD; i++) {
    if (t >= all_thread + MAX_THREAD)
      t = all_thread;
    if (t->state == RUNNABLE) {
      next_thread = t;
      break;
    }
    t = t + 1;
  }

  if (next_thread == 0) {
    printf("thread_schedule: no runnable threads\n");
    exit(-1);
  }

  if (current_thread != next_thread) { /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    if (t == &all_thread[0])
      t->state = FREE;
    else if (t->state != FREE) {
      t->state = RUNNABLE;
    }

    thread_switch((uint64)&t->context, (uint64)&next_thread->context);
  } else
    next_thread = 0;
}
```

就是在这里有了个漏洞……一开始，没有将主线程设置为 FREE，也就导致了最后其实卡在了主线程。但是在调试的时候一直找不到要点（而且 gdb 的操作也都忘光了，效率很低，没有调试的思路）。花了很多时间过后，也是 AI 指出这里有问题。

现在反思起来，问题其实在于没有对整个实现流程有一个概念。而且对于汇编中对寄存器的操作细节其实也并没有理解得比较透彻，传递的 context 结构体变量顺序也是有讲究的（因为 ra、sp 地址在汇编当中是写死的）。

最后是 yield 函数,用于出让线程，其实就是调用 schedule 函数，这就要求每个线程内部加入 yield：

```c
void thread_yield(void) {
  current_thread->state = RUNNABLE;
  thread_schedule();
}
```

### Using Threads

主要是要学一下库函数的使用，和大锁->小锁的优化。

首先是一个 ph 文件，操作时多线程地 put 了一些数据到哈希表当中。这里用的是散列映射+头插法:


```c
static void put(int key, int value) {
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if (e) {
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
}
```

可以看到，用 NBUCKET 区分出了各个桶，所以其实对各个桶的操作是独立的。这样，就可以对每个桶加锁（而不是对整个 put 操作加锁），以提高效率。

```c
pthread_mutex_t locks[NBUCKET]; // 声明锁（需要是全局变量）
// 在 main 中初始化
for (int i = 0; i < NBUCKET; i++) {
    pthread_mutex_init(&locks[i], NULL);
}
```

然后在 put 当中准确加锁

```c
void put(int key, int value) {
    int i = key % NBUCKET; // 算出这个 key 属于哪个桶

    pthread_mutex_lock(&locks[i]);   // 只锁这一个桶！

    // --- 临界区：插入链表的操作 ---
    // ... insert code ...

    pthread_mutex_unlock(&locks[i]); // 解锁
}
```

### barrier

要求是实现一个 barrier 函数，每一个线程都会调用这个函数，函数的作用是让所有的线程卡在这里，全都到达了以后才继续执行。

既然要让线程等待，那么肯定需要锁，用什么锁呢？这里要先区分一下各种锁的用处：

| **特性**     | **Mutex (互斥锁)**    | **Spinlock (自旋锁)** | **Condition Variable (条件变量)** |
| ---------- | ------------------ | ------------------ | ----------------------------- |
| **等待方式**   | **休眠（Sleep-wait）** | **忙等（Busy-wait）**  | **被动等待状态改变**                  |
| **CPU 占用** | 低（线程被挂起）           | **极高（100% 占用一核）**  | 低（线程被挂起）                      |
| **适用场景**   | 临界区长、竞争激烈          | **临界区极短、多核 CPU**   | 线程间需要协作（如 Barrier）            |
| **开销**     | 高（涉及内核上下文切换）       | 极低（如果能很快拿到锁）       | 中（需要配合 Mutex 使用）              |

由于这里每个线程到达的时间比较长，所以不能用 spinlock；又由于要等待所有进程到达，存在条件，所以需要引入 `cond_wait` 和 `cond_broadcast`，作用是在 cond 条件完成以后统一唤醒。

比较复杂的细节是，需要防止 broken wake up，也就是 bstate 的整个大锁不只要包裹住 nthread++，还要将后面的唤醒、睡眠也包裹住，否则如果在进入睡眠以前被唤醒，就会导致 broadcast 的 miss。以及记录轮数也是必要的，不然有可能会有某个进程被唤醒以后，很快地进入到下一个 barrier，导致 nthread 被改变（不等于 N），那么前面刚被唤醒的进程会误认为还没有到齐，加一个 round 就可以解决这个问题。

> 写到这里有些懒了，这是 AI 生成的。

```c
static void barrier() {
  // 1. 加锁，保护 bstate 内部共享变量的修改
  pthread_mutex_lock(&bstate.barrier_mutex);

  // 2. 增加已到达屏障的线程计数
  bstate.nthread++;

  // 3. 判断是否是最后一个到达的线程
  if (bstate.nthread == nthread) {
    // 如果我是最后一个到的：

    bstate.round++;     // 增加轮次（对应 thread 函数中的 assert(i == t)）
    bstate.nthread = 0; // 重置计数器，以便下一轮循环使用

    // 唤醒所有正在等待的线程
    pthread_cond_broadcast(&bstate.barrier_cond);

  } else {
    // 如果我不是最后一个到的：

    // 调用 wait 会自动释放 mutex，并让线程进入休眠。
    // 当被唤醒并从 wait 返回时，线程会自动重新持有 mutex。
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  }

  // 4. 解锁
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```