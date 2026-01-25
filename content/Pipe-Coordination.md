# Pipe-Coordination

Pipe 是检验 Sleep/Wakeup 机制的最佳案例。

## 数据结构
Pipe 有一把锁 `pi->lock`，保护 `nread` (读指针) 和 `nwrite` (写指针)。

## Reader (PipeRead)
1.  `acquire(&pi->lock)`。
2.  `while(pi->nread == pi->nwrite)` (缓冲区空):
    *   如果是空的，调用 `sleep(&pi->nread, &pi->lock)`。
    *   *注意:* `sleep` 会释放 `pi->lock`，允许 Writer 进来写数据。
    *   醒来后，`sleep` 会重新拿到 `pi->lock`，再次检查 `while` 条件。
3.  读取数据。
4.  `wakeup(&pi->nwrite)` (通知 Writer：我有空位了)。
5.  `release(&pi->lock)`。

## Writer (PipeWrite)
1.  `acquire(&pi->lock)`。
2.  `while(full)`: `sleep(&pi->nwrite, &pi->lock)`。
3.  写入数据。
4.  `wakeup(&pi->nread)` (通知 Reader：我有数据了)。
5.  `release(&pi->lock)`。

---