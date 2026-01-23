---
creation date: 2025-12-11 11:16
modification date: 2025-12-11 11:16
---
## 简介

从 Neovim 0.5 开始，Lua 成为内置的一等公民配置语言。

核心优势：

1. **性能**：LuaJIT 比 Vimscript 快得多。
    
2. **语法**：更现代、通用，拥有更好的工具链支持（LSP）。
    
3. **API**：可以直接访问 Vim 的 C 核心 API (`vim.api`)。
    

---

## 01. 执行 Lua 代码 (Execution)

### 命令行执行

在 Neovim 命令模式下：

-单行：:lua print(vim.inspect(package.loaded))

- 执行当前文件：`:luafile %` 或 `:source %`
    

### 在 Vimscript 中嵌入 Lua

这在迁移旧配置时很有用。

Vim Script

```
" init.vim
lua << EOF
  local msg = 'Hello from Lua'
  print(msg)
EOF
```

### 在 Lua 中执行 Vimscript

有些插件或旧命令没有 Lua 接口，必须使用 `vim.cmd`。

Lua

```
vim.cmd("colorscheme gruvbox")
vim.cmd([[
  highlight Error guibg=red
  packadd packer.nvim
]])
```

---

## 02. 选项设置 (Options)

Neovim 提供了多种方式来设置 `set` 选项（如行号、缩进）。

### 作用域分类 (Scopes)

|**Lua 对象**|**对应 Vim 概念**|**用途**|
|---|---|---|
|**`vim.opt`**|`:set`|**推荐**。最强大的接口，支持类似列表的操作。|
|`vim.o`|`:set global`|全局选项 (Global options)。|
|`vim.wo`|`:setlocal` (Window)|窗口局部选项 (如折叠)。|
|`vim.bo`|`:setlocal` (Buffer)|缓冲区局部选项 (如文件类型)。|
|`vim.g`|`let g:var`|**全局变量** (Global variables)，常用于插件配置。|

### 使用 `vim.opt` (推荐)

`vim.opt` 表现得像 Lua 的 Table，支持添加、移除值。

```lua
-- bool 类型
vim.opt.number = true          -- set number
vim.opt.relativenumber = true  -- set relativenumber

-- string 类型
vim.opt.clipboard = "unnamedplus"

-- list/map 类型 (强大之处)
vim.opt.listchars:append("space:⋅")
vim.opt.listchars:append("eol:↴")
```

> [!TIP] vim.g 的常见用法
> 
> 在加载任何插件之前，通常需要设置 Leader 键：
> 
> vim.g.mapleader = " " (设置空格为 Leader)

---

## 03. 按键映射 (Keymaps)

核心函数: vim.keymap.set(mode, lhs, rhs, opts)

这是新版 API（替换了旧的 vim.api.nvim_set_keymap），更安全且支持 Lua 函数。

### 参数说明

- **mode**: 字符串或表。`'n'`(Normal), `'i'`(Insert), `'v'`(Visual) 等。
    
- **lhs**: 按键组合 (Left-Hand Side)。
    
- **rhs**: 命令或 Lua 函数 (Right-Hand Side)。
    
- **opts**: 表。常用 `{ desc = "描述", silent = true, remap = false }`。
    

### 示例

```lua
-- 1. 简单映射：将 <Space>w 映射为保存
vim.keymap.set('n', '<leader>w', '<cmd>write<cr>', { desc = "Save file" })

-- 2. 调用 Lua 函数 (这是 Vimscript 做不到的)
vim.keymap.set('n', '<leader>h', function()
    print("Hello from Lua function!")
end, { desc = "Print Hello" })

-- 3. 插件常用的 opts
local opts = { noremap = true, silent = true }
```

---

## 04. Vim API 与 函数调用

### `vim.api.*`

这是 Neovim 暴露给 Lua 的底层 C API。

- `vim.api.nvim_buf_get_lines()`: 获取缓冲区内容。
    
- `vim.api.nvim_win_get_cursor(0)`: 获取当前窗口光标位置。
    

### `vim.fn.*`

调用传统的 Vimscript 内置函数（Functions）。

当你在 Vim 文档中看到 call fnamemodify() 时，在 Lua 中这样用：

```lua
-- 获取当前文件所在目录
local path = vim.fn.expand('%:p:h')
-- 检查文件是否可执行
local is_exec = vim.fn.executable('git')
```

---

## 05. 自动命令 (Autocommands)

Lua 提供了原生的自动命令接口，不再需要 `vim.cmd('autocmd ...')`。

**核心函数**: `vim.api.nvim_create_autocmd`

### 步骤

1. (可选) 创建组 `augroup` 以便于管理和防重。
    
2. 创建自动命令 `autocmd`。
    

```Lua
-- 1. 创建组 (clear = true 保证重新加载配置时清除旧的，防止重复)
local highlight_group = vim.api.nvim_create_augroup('YankHighlight', { clear = true })

-- 2. 创建自动命令：复制(Yank)时高亮文本
vim.api.nvim_create_autocmd('TextYankPost', {
    group = highlight_group,
    pattern = '*',
    callback = function()
        vim.highlight.on_yank()
    end,
    desc = 'Highlight when yanking (copying) text',
})
```

---

## 06. 用户命令 (User Commands)

创建类似 `:MyCommand` 的自定义命令。

**核心函数**: `vim.api.nvim_create_user_command`

```Lua
vim.api.nvim_create_user_command('Hello', function(opts)
    print("Hello " .. (opts.args or "World"))
end, {
    nargs = '?', -- 参数数量：0或1
    desc = "Print hello message"
})

-- 使用: :Hello Neovim
```

---

## 07. 模块系统 (Modules)

为了保持 `init.lua` 整洁，应当将配置拆分。

### 目录结构

Neovim 会自动在 runtimepath 下的 lua/ 目录中查找模块。

通常结构：

```Plaintext
~/.config/nvim/
├── init.lua
└── lua/
    ├── core/
    │   ├── options.lua
    │   └── keymaps.lua
    └── plugins/
        └── lualine.lua
```

### 引入模块

在 `init.lua` 中：

```Lua
-- 注意：使用 . 代替 /
require('core.options')
require('core.keymaps')
```

> [!WARNING] 缓存机制
> 
> Lua 的 require 会缓存模块。如果修改了子模块文件并重新 :source init.lua，子模块不会自动更新。
> 
> 解决方法：重启 Neovim，或者使用能清除缓存的插件（如 plenary.nvim 的 reload 机制）。

---
相关链接：[[lua学习]]