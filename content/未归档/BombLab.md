---
creation date: 2026-01-06 16:24
modification date: 2026-01-06 16:24
---
解题的思路：首先，所有已知的只有 bomb 一个可执行文件、抹去了关键代码的 bomb.c，所以第一步得要先 `objdump -d bomb` 反编译出一个汇编版本。关键是怎么看汇编代码。我们所能控制的部分，一般是函数调用时的传参，所以关键先看 caller,看是否有可以利用的函数。然后发现了各个 phase 的函数调用。

### Phase 1

可以发现调用了一个 `strings_not_euqal` 的函数，读了一下发现又长又乱，这里不能死磕。从函数名可以推测，大概是判断字符串是否相等的函数，那么题目的意思就比较明显了，是要我们传入一个指定字符串。

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
### Phase 2 二倍关系
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
>mov 自带地址解引用！所以 `mov -0x4(%rbx),%eax` 是读取数组元素。

### Phase 3 跳转表
尝试用了 [[pwngdb插件]]，还是很不错的！也比较有方向了。

题目：

```asm
0000000000400f43 <phase_3>:
  400f43:	48 83 ec 18          	sub    $0x18,%rsp
  400f47:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  400f4c:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  400f51:	be cf 25 40 00       	mov    $0x4025cf,%esi
  400f56:	b8 00 00 00 00       	mov    $0x0,%eax
  400f5b:	e8 90 fc ff ff       	call   400bf0 <__isoc99_sscanf@plt>
  400f60:	83 f8 01             	cmp    $0x1,%eax
  400f63:	7f 05                	jg     400f6a <phase_3+0x27>
  400f65:	e8 d0 04 00 00       	call   40143a <explode_bomb>
  400f6a:	83 7c 24 08 07       	cmpl   $0x7,0x8(%rsp)
  400f6f:	77 3c                	ja     400fad <phase_3+0x6a>
  400f71:	8b 44 24 08          	mov    0x8(%rsp),%eax
  400f75:	ff 24 c5 70 24 40 00 	jmp    *0x402470(,%rax,8)
  400f7c:	b8 cf 00 00 00       	mov    $0xcf,%eax
  400f81:	eb 3b                	jmp    400fbe <phase_3+0x7b>
  400f83:	b8 c3 02 00 00       	mov    $0x2c3,%eax
  400f88:	eb 34                	jmp    400fbe <phase_3+0x7b>
  400f8a:	b8 00 01 00 00       	mov    $0x100,%eax
  400f8f:	eb 2d                	jmp    400fbe <phase_3+0x7b>
  400f91:	b8 85 01 00 00       	mov    $0x185,%eax
  400f96:	eb 26                	jmp    400fbe <phase_3+0x7b>
  400f98:	b8 ce 00 00 00       	mov    $0xce,%eax
  400f9d:	eb 1f                	jmp    400fbe <phase_3+0x7b>
  400f9f:	b8 aa 02 00 00       	mov    $0x2aa,%eax
  400fa4:	eb 18                	jmp    400fbe <phase_3+0x7b>
  400fa6:	b8 47 01 00 00       	mov    $0x147,%eax
  400fab:	eb 11                	jmp    400fbe <phase_3+0x7b>
  400fad:	e8 88 04 00 00       	call   40143a <explode_bomb>
  400fb2:	b8 00 00 00 00       	mov    $0x0,%eax
  400fb7:	eb 05                	jmp    400fbe <phase_3+0x7b>
  400fb9:	b8 37 01 00 00       	mov    $0x137,%eax
  400fbe:	3b 44 24 0c          	cmp    0xc(%rsp),%eax
  400fc2:	74 05                	je     400fc9 <phase_3+0x86>
  400fc4:	e8 71 04 00 00       	call   40143a <explode_bomb>
  400fc9:	48 83 c4 18          	add    $0x18,%rsp
  400fcd:	c3                   	ret
```

先输了 test 运行看看，然后 `telescope $0x4025cf` 可以看到 `%d %d` 推测是输入两个整数。输个 1,2 重新调，确实过了。

接下来是很多个 jmp，AI 说可以推测是跳转表，用 `telescope 跳转表地址` 来利用，但是没有想到。直接运行，然后发现到了最后一步比较 `cmp 0xc(%rsp),%eax`,这里需要想到 rsp + 12 是第二个输入的字符，而第一个是 rsp + 8,一开始分配的。

最后，看一下 $rsp+0xc 的值（用 x 看内存，而不是用 p 看寄存器）,发现确实是输入的第二个数（上面用的 2），那么就改一下判断就可以了，改成和 $eax 相同的十进制数即可。

payload：`1 311`

补充：跳转表 `jmp *Address(Index_Reg, Scale)`，本题有多解

### Phase 4 递归调用

```asm
000000000040100c <phase_4>:
  40100c:	48 83 ec 18          	sub    $0x18,%rsp
  401010:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  401015:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  40101a:	be cf 25 40 00       	mov    $0x4025cf,%esi
  40101f:	b8 00 00 00 00       	mov    $0x0,%eax
  401024:	e8 c7 fb ff ff       	call   400bf0 <__isoc99_sscanf@plt>
  401029:	83 f8 02             	cmp    $0x2,%eax
  40102c:	75 07                	jne    401035 <phase_4+0x29>
  40102e:	83 7c 24 08 0e       	cmpl   $0xe,0x8(%rsp)
  401033:	76 05                	jbe    40103a <phase_4+0x2e>
  401035:	e8 00 04 00 00       	call   40143a <explode_bomb>
  40103a:	ba 0e 00 00 00       	mov    $0xe,%edx
  40103f:	be 00 00 00 00       	mov    $0x0,%esi
  401044:	8b 7c 24 08          	mov    0x8(%rsp),%edi
  401048:	e8 81 ff ff ff       	call   400fce <func4>
  40104d:	85 c0                	test   %eax,%eax
  40104f:	75 07                	jne    401058 <phase_4+0x4c>
  401051:	83 7c 24 0c 00       	cmpl   $0x0,0xc(%rsp)
  401056:	74 05                	je     40105d <phase_4+0x51>
  401058:	e8 dd 03 00 00       	call   40143a <explode_bomb>
  40105d:	48 83 c4 18          	add    $0x18,%rsp
  401061:	c3                   	ret
```

```asm
0000000000400fce <func4>:
  400fce:	48 83 ec 08          	sub    $0x8,%rsp
  400fd2:	89 d0                	mov    %edx,%eax
  400fd4:	29 f0                	sub    %esi,%eax
  400fd6:	89 c1                	mov    %eax,%ecx
  400fd8:	c1 e9 1f             	shr    $0x1f,%ecx
  400fdb:	01 c8                	add    %ecx,%eax
  400fdd:	d1 f8                	sar    $1,%eax
  400fdf:	8d 0c 30             	lea    (%rax,%rsi,1),%ecx
  400fe2:	39 f9                	cmp    %edi,%ecx
  400fe4:	7e 0c                	jle    400ff2 <func4+0x24>
  400fe6:	8d 51 ff             	lea    -0x1(%rcx),%edx
  400fe9:	e8 e0 ff ff ff       	call   400fce <func4>
  400fee:	01 c0                	add    %eax,%eax
  400ff0:	eb 15                	jmp    401007 <func4+0x39>
  400ff2:	b8 00 00 00 00       	mov    $0x0,%eax
  400ff7:	39 f9                	cmp    %edi,%ecx
  400ff9:	7d 0c                	jge    401007 <func4+0x39>
  400ffb:	8d 71 01             	lea    0x1(%rcx),%esi
  400ffe:	e8 cb ff ff ff       	call   400fce <func4>
  401003:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax
  401007:	48 83 c4 08          	add    $0x8,%rsp
  40100b:	c3                   	ret
```

这题就不能光看了，解法是先大概观察一下汇编，然后发现参数比较多，就将各个寄存器代表的是第几个参数给记了一下，这样比较方便定位。过程有些复杂，尝试了以后感觉写下来不太现实，就将 func4 的汇编一句句翻译成 C 语言代码:

```c
int func4(int a,int b,int c,int d){
  int res = c;
  res -= b;
  d = res;
  res += res >> 31;
  res /= 2;
  d = res; // d = res / 2
  if(d <= a){
    res = 0;
    if(d >= a){
      return 0;
    }
    b = d + a;
    func4(a,b,c,d);
  }else{
    func4(a,b,c,d);
  }
}
```

这样看清楚多了，发现只要传入的第一个参数 a == 14 就可以直接返回了。最后是一个比较简单的判断，看寄存器发现第二个参数又和 0 进行了比较，就输入 `7 0`, 解决。

主要的启发是还原为 C 来观察，会清晰很多。以及各种连续的 mov 后面接着 call  可能代表给函数传参，关注递归函数的 ret 就是出口。

AI补充：

`res += res >> 31; res /= 2;` 实际是编译器对移位的补充，而不是直接用 `sar $1, %eax` 

不能只是逐字翻译，还要有工程直觉，观察特征：
- `shl $2, %rax 或 lea (,%rax,4), %rax`：可能是数组索引计算
- 连续的 `cmp` 配合 `ja/jb`（大范围跳转）：边界检查（Boundary Check） 或 Switch 跳转表。
- 寄存器 `%rdi, %rsi, %rdx` 的初始变动:输入变量溯源

画控制流图，还原拓扑结构而不是代码
### Phase 5 索引表混淆

`40106a:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax` 金丝雀,读取一个随机数并存储，用于检查栈内是否被强制修改。

`movzbl (%rbx,%rax,1),%ecx` 取一字节，零扩展，目标 l 四字节

```asm
  // cl 取低位地址存入
  40108f:	88 0c 24             	mov    %cl,(%rsp)
  401092:	48 8b 14 24          	mov    (%rsp),%rdx
```

```asm
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx
  查表掩盖,将 rdx 的内容在表内变换以后返回到 edx
```

```text
发现了变换以后，考虑可能变换是固定的，于是从头开始试，找到对应 flyers 的即可
这里直接穷举爆破了…… 没有别的思路
stuvwx -> uiersn
mnopqr -> bylmad
ghijkl -> snfotv
fjsnrg -> rouyds
abcdef -> adiuer
ionuvg -> flyers
```

```asm
0000000000401062 <phase_5>:
  401062:	53                   	push   %rbx
  401063:	48 83 ec 20          	sub    $0x20,%rsp
  401067:	48 89 fb             	mov    %rdi,%rbx
  40106a:	64 48 8b 04 25 28 00 	mov    %fs:0x28,%rax
  401071:	00 00 
  401073:	48 89 44 24 18       	mov    %rax,0x18(%rsp)
  401078:	31 c0                	xor    %eax,%eax
  40107a:	e8 9c 02 00 00       	call   40131b <string_length>
  40107f:	83 f8 06             	cmp    $0x6,%eax
  401082:	74 4e                	je     4010d2 <phase_5+0x70>
  401084:	e8 b1 03 00 00       	call   40143a <explode_bomb>
  401089:	eb 47                	jmp    4010d2 <phase_5+0x70>
  40108b:	0f b6 0c 03          	movzbl (%rbx,%rax,1),%ecx
  40108f:	88 0c 24             	mov    %cl,(%rsp)
  401092:	48 8b 14 24          	mov    (%rsp),%rdx
  401096:	83 e2 0f             	and    $0xf,%edx
  401099:	0f b6 92 b0 24 40 00 	movzbl 0x4024b0(%rdx),%edx
  4010a0:	88 54 04 10          	mov    %dl,0x10(%rsp,%rax,1)
  4010a4:	48 83 c0 01          	add    $0x1,%rax
  4010a8:	48 83 f8 06          	cmp    $0x6,%rax
  4010ac:	75 dd                	jne    40108b <phase_5+0x29>
  4010ae:	c6 44 24 16 00       	movb   $0x0,0x16(%rsp)
  4010b3:	be 5e 24 40 00       	mov    $0x40245e,%esi
  4010b8:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi
  4010bd:	e8 76 02 00 00       	call   401338 <strings_not_equal>
  4010c2:	85 c0                	test   %eax,%eax
  4010c4:	74 13                	je     4010d9 <phase_5+0x77>
  4010c6:	e8 6f 03 00 00       	call   40143a <explode_bomb>
  4010cb:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)
  4010d0:	eb 07                	jmp    4010d9 <phase_5+0x77>
  4010d2:	b8 00 00 00 00       	mov    $0x0,%eax
  4010d7:	eb b2                	jmp    40108b <phase_5+0x29>
  4010d9:	48 8b 44 24 18       	mov    0x18(%rsp),%rax
  4010de:	64 48 33 04 25 28 00 	xor    %fs:0x28,%rax
  4010e5:	00 00 
  4010e7:	74 05                	je     4010ee <phase_5+0x8c>
  4010e9:	e8 42 fa ff ff       	call   400b30 <__stack_chk_fail@plt>
  4010ee:	48 83 c4 20          	add    $0x20,%rsp
  4010f2:	5b                   	pop    %rbx
  4010f3:	c3                   	ret
```

```python
print(chr(ord('f') - 0)) # 虽然没用到
```

payload: ionuvg

正解：

一些新用上的 gdb 命令：
- `delete num` 删除断点
- `checkpoint` 添加返回点，`restart num` 返回
- `x/s 0x... x/16c 0x...` 查询，并限定返回格式
- `until *0x...` 执行知道某地址
- `r < answer.txt` 传递参数，省去重复的麻烦

既然有了索引表地址，就去看看放的是什么！
- `movzbl (base, index, 1) + and $mask, index` 将八位映射到四位，可以猜测是索引表映射。四位有十六个，通过传入字母的最后一个字节的数字来定位。`and` 其实是掩码。
### Phase 6 链表重排

`0x0(%r13)` 在汇编中，如果 r13 是结构体，那么 0x0 指向第一个成员，而 `%r13` 表示的是实际地址。

在函数一开始 push 寄存器的值，是为了保存寄存器信息，那么在函数内部这些寄存器就可以随意使用，相当于变成了临时变量。

`read_six_numbers` 会将输入的 6 个数字，依次存入以 `%rsp` 为起始地址的栈内存中。可以通过 `x/6dw $rsp ` 进行验证。

`movslq %ebx,%rax` 符号扩展、32位 -> 64位

`x/40gx 0x603110` 看 40 个 8 字节（g）的内容，因为 64 位程序中结构体成员通常 8 字节对齐。可以直接看到所有 node 的 value。

```python
decimal1 = int(hex_str1, 16) # 进制转换（字符串）
a = 0xaaa # 然后直接 print,会自动转换
```

```asm
循环处理每一个数，变换为 7 - num,增加逆向难度
mov    %ecx,%edx             
sub    (%rax),%edx           
mov    %edx,(%rax)           
add    $0x4,%rax             
cmp    %rsi,%rax             
jne    401160 <phase_6+0x6c> 
```

```asm
处理链表排序的逻辑，按照输入顺序，7 - num 进行排序，并判断排序后节点是否为降序序列
mov    0x20(%rsp),%rbx      
lea    0x28(%rsp),%rax      
lea    0x50(%rsp),%rsi      
mov    %rbx,%rcx            
mov    (%rax),%rdx          
mov    %rdx,0x8(%rcx)       
add    $0x8,%rax            
cmp    %rsi,%rax            
je     4011d2 <phase_6+0xde>
mov    %rdx,%rcx            
jmp    4011bd <phase_6+0xc9>
movq   $0x0,0x8(%rdx)       
                            
mov    $0x5,%ebp            
mov    0x8(%rbx),%rax       
mov    (%rax),%eax          
cmp    %eax,(%rbx)          
jge    4011ee <phase_6+0xfa>
```

```asm
编译器的优化，直接 jne 相当于判断上一步的结果是否为 0，通过循环来进行遍历比较
mov    0x8(%rbx),%rbx        
sub    $0x1,%ebp             
jne    4011df <phase_6+0xeb> 
```

补充：
- `mov 0x8(%rax), %rax` → 链表遍历
- `mov %rdx, 0x8(%rax)` -> 指针劫持、修改连接

不要看单个的汇编语句，没有意义。要注意 Loop 的整体作用，数据的流动、地址存储的是什么。
