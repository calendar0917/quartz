---
creation date: 2025-12-11 10:54
modification date: 2025-12-11 10:54
---
来源: Learn Lua in Y Minutes

## 简介

Lua 是一种轻量级、可嵌入的脚本语言。它的核心设计哲学是简洁、高效和可扩展。

核心特点：

1. **Table 是唯一的数据结构**。
    
2. **索引从 1 开始**（与大多数语言不同）。
    
3. **函数是一等公民**。
    
4. 没有类（Class），通过 Metatable 模拟面向对象。
    

---

## 01. 基础语法与变量 (Basics)

### 注释

- 单行注释：`--`
    
- 多行注释：`--[[ ... ]]`
    

### 变量与作用域

Lua 默认变量是**全局**的，局部变量必须使用 `local` 关键字。

> [!WARNING] 最佳实践
> 
> 始终使用 local 定义变量，除非你真的需要全局变量。这能提高性能并避免命名空间污染。

```lua
num = 42  -- 全局变量 (实际上是 _G 表中的字段)
local s = "walternate"  -- 局部变量
```

### 基本数据类型

Lua 是动态类型语言。

- **nil**: 表示无效值，未赋值的变量默认为 `nil`。
    
- **boolean**: `true` / `false` (注意：只有 `nil` 和 `false` 为假，`0` 和 `""` 均为真)。
    
- **number**: 通常是双精度浮点数（Lua 5.3+ 支持整型）。
    
- **string**: 字符串，不可变。
    

```lua
local is_valid = true
local count = 10
local text = 'Hello' .. ' World' -- 字符串拼接使用 ..
```

---

## 02. 控制流 (Control Flow)

### 条件判断

结构：`if ... elseif ... else ... end`

```lua
if num > 40 then
    print('over 40')
elseif s ~= 'walternate' then -- 不等于使用 ~=
    io.write('not over 40\n')
else
    print('default')
end
```

### 循环结构

#### 1. While 循环

```Lua
while num < 50 do
    num = num + 1
end
```

#### 2. Repeat-Until (类似 do-while)

```Lua
repeat
    print('Running')
until num == 0
```

#### 3. For 循环

- **数值型**: `for var = start, end, step do`
    
- **泛型**: `for key, value in pairs(table) do`
    

```Lua
-- 从 1 到 100
for i = 1, 100 do end
-- 泛型遍历
for key, value in pairs(my_table) do end
```

---

## 03. 函数 (Functions)

### 核心特性

- **一等公民**：可以赋值给变量，可以作为参数传递。
    
- **闭包 (Closure)**：可以捕获外部作用域的变量。
    
- **多返回值**：原生支持返回多个结果。
    

### 定义与调用

```Lua
function fib(n)
    if n < 2 then return 1 end
    return fib(n - 2) + fib(n - 1)
end

-- 匿名函数与闭包
function adder(x)
    return function(y) return x + y end
end
local add2 = adder(2)
print(add2(10)) -- 输出 12
```

### 特殊参数

- `...`: 可变参数列表。

```Lua
function printer(...)
    local args = {...} -- 将参数打包成 table
    for _, v in ipairs(args) do print(v) end
end
```

---

## 04. 表 (Tables)

**Table 是 Lua 中唯一的数据结构**。它既是数组（Array），也是字典（Map/Dictionary）。

### 作为字典（键值对）

键可以是除 `nil` 外的任何值。

```Lua
local t = {key1 = 'value1', key2 = false}
print(t.key1)  -- 语法糖，等同于 t["key1"]
t.newKey = {}  -- 添加新键值对
t.key2 = nil   -- 删除键值对
```

### 作为数组（列表）

**重要**：Lua 的数组索引默认从 **1** 开始。

```Lua
local list = { 'apple', 'banana', 'orange' }
print(list[1]) -- 输出 'apple'

-- 遍历数组 (使用 ipairs 保证顺序)
for i, v in ipairs(list) do
    print(i, v)
end
```

> [!NOTE] pairs vs ipairs
> 
> - `pairs(t)`: 遍历表的所有键值对（无序）。
>     
> - `ipairs(t)`: 仅遍历表的整数索引部分（从 1 开始，直到遇到 nil），保证顺序。
>     

---

## 05. 元表 (Metatables) & 模拟 OOP

元表允许我们改变 Table 的行为（例如操作符重载）。这是 Lua 模拟面向对象（OOP）的基础。

### 常用元方法

- `__add`: 重载 `+` 运算符。
    
- `__tostring`: 重载 `print()` 时的行为。
    
- `__index`: **最关键**。当在表中找不到某个 Key 时，会去元表的 `__index` 中查找。
    

### 模拟类与继承

```Lua
local Dog = {} 

function Dog:new()
    local newObj = {sound = 'woof'}
    self.__index = self -- 将元表的索引指向类自身
    return setmetatable(newObj, self)
end

function Dog:makeSound()
    print('I say ' .. self.sound)
end

local mrDog = Dog:new()
mrDog:makeSound() -- 输出 "I say woof"
```

> [!TIP] `.` 和 `:` 的区别
> 
> - `Dog.makeSound(mrDog)`: 必须显式传递 self 参数。
>     
> - `mrDog:makeSound()`: 语法糖，自动将调用者 (`mrDog`) 作为第一个参数 (`self`) 传入。
>     

---

## 06. 模块与协程 (Modules & Coroutines)

### 模块 (Modules)

Lua 模块通常是一个返回 Table 的脚本文件。

```Lua
-- mod.lua
local M = {}
local function private() ... end
function M.public() ... end
return M

-- main.lua
local mod = require('mod')
```

### 协程 (Coroutines)

协作式多线程，可以在执行中暂停（yield）并在稍后恢复（resume）。

```Lua

local co = coroutine.create(function()
    for i = 1, 10 do
        print(i)
        coroutine.yield() -- 暂停执行
    end
end)

coroutine.resume(co) -- 执行一次，打印 1
coroutine.resume(co) -- 恢复执行，打印 2
```
