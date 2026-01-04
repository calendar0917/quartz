---
creation date: 2025-12-28 16:46
modification date: 2025-12-28 16:46
draft: true
---
## 1. 读取单个进程信息 
目标：通过读取 `/proc/pid/stat` 里的信息，输出程序状态。

CMakeList 需要添加 `set(CMAKE_EXPORT_COMPILE_COMMANDS ON)` 才能让 nvim 跨文件处理,会生成 .json 文件

`std::optional：*optional_obj` 这里的 * 用于提取 optional 里的值,optional 用于返回 “有或没有”,自动处理比较方便

提取路径、打开文件的方式:

```c
namespace fs = std::filesystem
fs::path stat_path = fs::path("/proc") / std::to_string(pid) / "stat";
std::ifstream is(stat_path);
  // 判断文件是否存在且可读
  if (!is.is_open()) {
    return std::nullopt;
  }
  std::string content;
  if (std::getline(is, content)) {
    return content;
  }
```

主要是字符串处理:

```c
std::istringstream tokenStream(s); // 将 s 变为流，方便读取
std::getline(tokenStream, token, delimeter)
// 寻找括号
data.find('')、data.rfind('')
// 解析数字
std::stoi(string)
// 定位
string.substr(start,end)
```
## 2. 实现所有进程读取
