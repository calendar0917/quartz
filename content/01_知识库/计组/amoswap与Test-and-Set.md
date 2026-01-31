# amoswap与Test-and-Set

锁的实现依赖于硬件提供的**原子指令**。普通指令无法保证“读+写”不被打断。

## RISC-V 指令: `amoswap` (Atomic Memory Operation Swap)
`amoswap.w.aq rd, rs2, (rs1)`
*   **功能:** 原子地读取 `(rs1)` 内存地址的旧值存入 `rd`，并将 `rs2` 的新值写入 `(rs1)`。
*   **原子性:** 硬件保证在这条指令执行期间，其他 CPU 无法读写该内存地址。

## C 语言映射
xv6 使用 GCC 内置函数 `__sync_lock_test_and_set(&lk->locked, 1)`。
*   如果返回 0: 说明旧值是 0 (未锁)，我成功写入了 1 (加锁)。**获取成功**。
*   如果返回 1: 说明旧值是 1 (已锁)，我写入 1 (保持锁定)。**获取失败，继续重试**。

---