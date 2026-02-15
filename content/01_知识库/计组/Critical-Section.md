# Critical-Section

## 定义
当多个 CPU (或线程) 同时访问共享内存，且至少有一个是写操作时，最终的结果取决于指令执行的**精确时序**。这种不确定性称为竞态条件。

## 本质
高级语言的一行代码 (如 `count++`) 在汇编层对应 **Load -> Modify -> Store** 三步。
如果两个 CPU 交错执行：
1. CPU A Load (val=0)
2. CPU B Load (val=0)
3. CPU A Store (val=1)
4. CPU B Store (val=1)
**结果：** 内存是 1，而不是 2。A 的更新丢失了 (Lost Update)。

## 安全视角
**TOCTOU (Time-of-check to time-of-use):** 攻击者在系统检查权限 (Check) 和使用资源 (Use) 的极短时间窗口内，修改了资源状态。这就是一种竞态漏洞。

---