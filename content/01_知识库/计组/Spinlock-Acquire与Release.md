# Spinlock-Acquire与Release

自旋锁 (Spinlock) 适用于**持有时间极短**的场景。

## Acquire (加锁)
```c
void acquire(struct spinlock *lk) {
  push_off(); // 1. 关中断 (防止死锁)
  if(holding(lk)) panic("acquire"); // 禁止重入

  // 2. 原子操作抢锁
  while(__sync_lock_test_and_set(&lk->locked, 1) != 0)
    ; // 自旋 (Spinning): 这里的空循环在消耗 CPU

  // 3. 内存屏障
  __sync_synchronize(); 
}
```

## Release (解锁)
```c
void release(struct spinlock *lk) {
  // 1. 内存屏障
  __sync_synchronize();

  // 2. 原子释放 (可以直接写 0，因为我们拥有锁)
  __sync_lock_release(&lk->locked);

  pop_off(); // 3. 开中断
}
```