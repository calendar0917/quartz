Dataview 的本质就是 **SQL for Markdown Files**。它把你的 Vault 当作一个数据库，每一个 .md 文件就是一行记录（Row），文件的属性（YAML Frontmatter）就是字段（Column）。

### 基本查询结构 (SELECT-FROM-WHERE)

```dateview
TABLE file.mtime AS "Last Mod"  -- SELECT: 选择展示哪些字段
FROM "项目"                     -- FROM: 指定源目录 (也就是你的 WHERE path LIKE '项目%')
WHERE file.name != "Template"   -- WHERE: 过滤条件
SORT file.mtime DESC            -- ORDER BY: 排序
LIMIT 5                         -- LIMIT: 限制条数
```

- **TABLE**: 输出为表格。
- **LIST**: 输出为无序列表
- **WITHOUT ID**: 默认情况下 Dataview 第一列会显示文件名。加上这个，你可以自定义第一列显示什么（比如把文件名改名为 "Project"）。

### 隐式字段 (Implicit Fields)

Dataview 会自动为每个文件生成元数据，不需要手动写，直接调用的：

- file.link: 文件的超链接。
- file.name: 文件名（不带后缀）。
- file.path: 完整路径（如 项目/CTF/Pwn_Staging.md）。
- file.ctime: 创建时间 (Created Time).
- file.mtime: 修改时间 (Modified Time)。
- file.folder: 文件所在的文件夹名。

### 自定义字段 (Custom Fields via YAML)

这是个性化的关键。你在笔记最开头的 --- 区域写的东西，都可以被查询。

**实操演示：**  
假设你在 项目 文件夹里有一个笔记叫 CTF-2025-Winter.md。  
请在笔记开头写入：

```yaml
status: active
priority: high
tags: [ctf, pwn]
```

然后，你可以修改 Homepage 的代码来**只显示**状态为 active 的项目：

```
TABLE status, priority
FROM "项目"
WHERE status = "active"  <-- 这里就是调用了你自定义的 YAML 属性
SORT priority DESC
```