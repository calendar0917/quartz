# swtch函数详解

位于 `kernel/swtch.S`。这是内核中唯一不遵守常规 C 函数返回逻辑的代码。

## 函数原型
`void swtch(struct context *old, struct context *new);`
*   `a0`: 指向旧上下文 (`old`)。
*   `a1`: 指向新上下文 (`new`)。

## 魔法步骤
1.  **Save Old:** 将当前的 `ra`, `sp`, `s0`... 保存到 `a0` 指向的内存中。
    *   *此时，`old->ra` 记录了调用 swtch 的那行代码地址。*
2.  **Load New:** 从 `a1` 指向的内存中加载 `ra`, `sp`, `s0`... 到 CPU 寄存器。
    *   **关键点:** 执行完 `ld sp, 8(a1)` 后，CPU 的栈指针瞬间变了！
3.  **Return:** 执行 `ret`。
    *   `ret` 本质是 `jr ra`。
    *   由于 `ra` 已经被替换成了新进程的地址（通常是 `scheduler` 或 `forkret`），CPU 跳转到了新代码执行。

**结论:** 这是一个“从 A 进，从 B 出”的函数。

---