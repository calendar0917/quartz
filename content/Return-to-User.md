# Return-to-User

这是从内核返回用户态的逆向过程，包含 C 语言准备 (`usertrapret`) 和 汇编恢复 (`userret`)。

## 上半场: Usertrapret (trap.c)
**核心任务:** 为**下一次** Trap 进来做好准备（因为寄存器是易失的）。
1.  **关中断:** `intr_off()`。接下来的操作必须原子化。
2.  **重置 stvec:** 指向 `uservec` (Trampoline)。
3.  **填充 Trapframe (Refill):**
    *   把当前的内核页表 (`satp`)、内核栈 (`kstack`)、`usertrap` 地址填入 Trapframe 的 kernel 字段。
    *   *思考:* 为什么现在填？因为一旦切回用户态，这些内核信息就无法访问了。
4.  **设置 S 态寄存器:**
    *   `sstatus`: 设置 SPP=User，开启 SPIE (允许用户态中断)。
    *   `sepc`: 设置为用户原本的 PC (或者 `ecall` 的下一条)。

## 下半场: Userret (trampoline.S)
1.  **切换页表:** `csrw satp, a0` (参数 a0 传递了用户页表地址)。**此刻起，只能访问用户内存。**
2.  **恢复寄存器:** `ld ra, 40(a0)`... 从 Trapframe 读回所有数据。
3.  **归还 a0:** `csrrw a0, sscratch, a0`。把 Trapframe 地址藏回 `sscratch`，恢复用户真正的 `a0`。
4.  **发射:** `sret`。

---
