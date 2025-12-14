---
creation date: 2025-12-12 22:57
modification date: 2025-12-12 22:57
---
# 程序的机器级表示
本章揭示了高级语言（C）如何被编译器翻译成机器语言（x86-64）。对于安全人员而言，这是理解 **控制流劫持 (Control Flow Hijacking)** 和 **逆向工程 (Reverse Engineering)** 的物理基础。掌握汇编不是为了写汇编，而是为了看透系统的“裸机”状态。

工具的学习：[[GDB-调试指南]] [[gcc]] [[objdump]]

## 1. 基础设施 (Infrastructure)
* **生产资料（存储）：** [[寄存器与数据传输]] - 16个通用寄存器的政治经济学。
* **物流系统（寻址）：** [[寻址模式与指令]] - 立即数、寄存器与内存的交互 (`mov` vs `lea`)。
* **工作现场（栈）：** [[栈帧与过程调用]] - 函数调用的舞台，控制权转移的核心 (`call`/`ret`)。

## 2. 逻辑控制 (Control Logic)
* **状态检测：** [[条件码与比较]] - EFLAGS 寄存器 (`ZF`, `SF`, `OF`)。
* **流程跳转：** [[跳转与循环]] - 从 `if/switch` 到汇编的 `jmp` 表。

## 3. 数据组织 (Data Organization)
* **线性结构：** [[数组与指针运算]] - 步长与地址计算的本质。
* **异构结构：** [[数组与指针运算#2. 结构体与对齐 (Structs & Alignment)|结构体与对齐]] - 内存对齐 (`Alignment`) 作为一种强制规范。

## 4. 安全对抗 (Security)
* **漏洞根源：** [[缓冲区溢出攻击]] - 栈破坏与控制流劫持。
* **防御机制：** [[对抗缓冲区溢出]] - 栈金丝雀 (Canary)、ASLR、NX 位。
## 5. BombLab
解题的思路：首先，所有已知的只有 bomb 一个可执行文件、抹去了关键代码的 bomb.c，所以第一步得要先 `objdump -d bomb` 反编译出一个汇编版本。关键是怎么看汇编代码。我们所能控制的部分，一般是函数调用时的传参，所以关键先看 caller,看是否有可以利用的函数。然后发现了各个 phase 的函数调用。

### Phase 1

可以发现调用了一个 `strings_not_euqal` 的函数，读了一下发现有长有乱，这里就能死磕。从函数名可以推测，大概是判断字符串是否相等的函数，那么题目的意思就比较明显了，是要我们传入一个指定字符串。

```asm
0000000000400ee0 <phase_1>:
  400ee0:	48 83 ec 08          	sub    $0x8,%rsp
  400ee4:	be 00 24 40 00       	mov    $0x402400,%esi
  400ee9:	e8 4a 04 00 00       	call   401338 <strings_not_equal>
  400eee:	85 c0                	test   %eax,%eax
  400ef0:	74 05                	je     400ef7 <phase_1+0x17>
  400ef2:	e8 43 05 00 00       	call   40143a <explode_bomb>
  400ef7:	48 83 c4 08          	add    $0x8,%rsp
  400efb:	c3                   	ret
```

这里的问题是，如何找到对应字符串。可以发现 mov 了一个东西到 esi（补充知识：[[寄存器与数据传输#寄存器角色分工 (System V AMD64 ABI)|各个寄存器的功能]]） 里， 可致 esi 是比较的参数，那么就打个断点，然后 `x/s $esi` 看一下到底传了什么，就能得到答案了。

关键是多用 gdb 去看寄存器、内存的内容到底是什么。
### Phase 2
```asm
0000000000400efc <phase_2>:
  400efc:	55                   	push   %rbp
  400efd:	53                   	push   %rbx
  400efe:	48 83 ec 28          	sub    $0x28,%rsp
  400f02:	48 89 e6             	mov    %rsp,%rsi
  400f05:	e8 52 05 00 00       	call   40145c <read_six_numbers>
  400f0a:	83 3c 24 01          	cmpl   $0x1,(%rsp)
  400f0e:	74 20                	je     400f30 <phase_2+0x34>
  400f10:	e8 25 05 00 00       	call   40143a <explode_bomb>
  400f15:	eb 19                	jmp    400f30 <phase_2+0x34>
  400f17:	8b 43 fc             	mov    -0x4(%rbx),%eax
  400f1a:	01 c0                	add    %eax,%eax
  400f1c:	39 03                	cmp    %eax,(%rbx)
  400f1e:	74 05                	je     400f25 <phase_2+0x29>
  400f20:	e8 15 05 00 00       	call   40143a <explode_bomb>
  400f25:	48 83 c3 04          	add    $0x4,%rbx
  400f29:	48 39 eb             	cmp    %rbp,%rbx
  400f2c:	75 e9                	jne    400f17 <phase_2+0x1b>
  400f2e:	eb 0c                	jmp    400f3c <phase_2+0x40>
  400f30:	48 8d 5c 24 04       	lea    0x4(%rsp),%rbx
  400f35:	48 8d 6c 24 18       	lea    0x18(%rsp),%rbp
  400f3a:	eb db                	jmp    400f17 <phase_2+0x1b>
  400f3c:	48 83 c4 28          	add    $0x28,%rsp
  400f40:	5b                   	pop    %rbx
  400f41:	5d                   	pop    %rbp
  400f42:	c3                   	ret
```

代码很长，重点要关注各个跳转指令，看数据的流动是怎么样的。先在纸上把各个寄存器的运算、转移写了一边，发现有一个类似数组-循环的结构，从 f17 - f35，是一个循环。可以猜测，发现就是一个二倍的关系，所以最后 `1 2 4 8 16 32`.

>![tip] 注意
> mov 自带地址解引用！所以 `mov    -0x4(%rbx),%eax` 是读取数组元素。
