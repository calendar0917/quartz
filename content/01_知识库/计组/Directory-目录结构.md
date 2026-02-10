# Directory-目录结构

目录 (Directory) 在 xv6 中没有任何黑魔法，它就是一种**特殊类型 (T_DIR)** 的文件。

## 内部结构
它的数据块里存的不是文本，而是一系列 `struct dirent`：
```c
struct dirent {
  ushort inum;        // Inode Number (2 bytes)
  char name[DIRSIZ];  // File Name (14 bytes)
};
```

- 每个条目 16 字节。
- inum = 0 表示该条目为空（被删除了）。

## 查找逻辑

当你要找 ls 时，内核就在当前目录的数据块里遍历 dirent 数组，对比 name 是否为 "ls"。如果找到了，就返回对应的 inum。