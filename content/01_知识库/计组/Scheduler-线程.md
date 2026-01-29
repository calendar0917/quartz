# Scheduler-线程

这是每个 CPU 核的“空闲状态”。当没有进程运行时，CPU 就跑在这个循环里。

## 核心循环
```c
void scheduler(void) {
  for(;;){ // 无限循环
    intr_on(); // 开中断 (防止死锁，接收时钟中断)
    
    for(p in proc_table){ // 遍历进程表
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        p->state = RUNNING;
        
        // 这里的 swtch 是把 CPU 从调度器切给进程 P
        swtch(&c->context, &p->context);
        
        // 当进程 P yield() 后，swtch 会返回到这里
        // 此时 p->lock 依然被持有！(由 P 进程带过来的)
      }
      release(&p->lock);
    }
  }
}
```