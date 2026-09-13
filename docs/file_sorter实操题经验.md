# file_sorter 实操题经验总结

记录 file_sorter 项目递进式实操题的知识点、踩坑和经验。每完成一题追加一节。

---

## 题 1：FileInfo 信息打印器

### 题目目标
实现 FileInfo 结构体的信息打印，包括两个格式化函数：
- `formatFileSize(uint64_t bytes)`：文件大小自动换算 B/KB/MB，保留 1 位小数
- `formatFileTime(const fs::file_time_type& ftime)`：修改时间格式化为 `YYYY-MM-DD HH:MM:SS`

### 知识点 1：格式化函数应返回字符串，不直接打印

#### 现象
`formatFileSize` 函数内部直接用 `std::cout` 打印，最后 `return ""`，导致 `printFileInfo` 输出顺序错乱：`formatFileSize` 的内容先打印出来，然后才打印"文件大小: "标签，且标签后内容为空。

#### 排查与原因
- 函数签名为 `std::string formatFileSize(uint64_t bytes)`，职责是"格式化并返回字符串"
- `printFileInfo` 中调用方式为 `std::cout << "文件大小: " << formatFileSize(file.fileSize) << "\n"`
- 如果 `formatFileSize` 内部直接 `cout`，会在函数调用时就输出内容，打乱外层输出顺序
- `return ""` 导致外层 `<<` 运算符输出空字符串

#### 解决方案
用 `std::ostringstream` 代替 `std::cout`，把格式化内容写入字符串流，最后用 `.str()` 返回字符串：

```cpp
std::string formatFileSize(uint64_t bytes) {
    std::ostringstream oss;
    if (bytes < 1024) {
        oss << bytes << " B";
    }
    else if (bytes < 1024 * 1024) {
        double kb = static_cast<double>(bytes) / 1024.0;
        oss << std::fixed << std::setprecision(1) << kb << " KB";
    }
    else {
        double mb = static_cast<double>(bytes) / (1024.0 * 1024.0);
        oss << std::fixed << std::setprecision(1) << mb << " MB";
    }
    return oss.str();
}
```

#### 经验总结
- 格式化函数（`formatXxx`）只负责生成字符串，不负责输出，输出由调用者统一处理
- `std::ostringstream` 用法和 `std::cout` 完全一样，都是用 `<<` 运算符，区别是输出到内存字符串而不是控制台
- 职责分离：`format` 函数可复用于打印、写文件、发网络等多种场景
- `ostringstream` 里不要用 `std::endl`（会多余刷新并添加换行），换行由外层统一加

---

### 知识点 2：C++ 时间类型转换 4 步走

#### 现象
`fs::file_time_type` 不能直接格式化输出，需要经过多次类型转换才能得到 `YYYY-MM-DD HH:MM:SS` 格式的字符串。

#### 排查与原因
C++ 里有 4 种不同的时间类型，各管一摊，不能直接互转：

| 类型 | 是什么 | 能干嘛 |
|---|---|---|
| `fs::file_time_type` | 文件系统专用时间 | 从文件属性获取修改时间 |
| `std::chrono::system_clock::time_point` | C++ 通用时间点 | 时间加减、比较 |
| `std::time_t` | C 风格时间（整数） | 从 1970 年起的秒数 |
| `std::tm` | C 风格分解时间结构体 | 拆成年/月/日/时/分/秒 |

#### 解决方案
4 步转换：

```cpp
std::string formatFileTime(const fs::file_time_type& ftime) {
    // 第1步：file_time_type → system_clock::time_point（用时间差间接转换）
    auto sys_time = std::chrono::time_point_cast<std::chrono::system_clock::duration>(
        ftime - fs::file_time_type::clock::now() + std::chrono::system_clock::now()
    );
    // 第2步：time_point → time_t（秒数）
    std::time_t tt = std::chrono::system_clock::to_time_t(sys_time);
    // 第3步：time_t → tm（本地时间，分解为年月日时分秒）
    std::tm tm_buf;
    localtime_s(&tm_buf, &tt);  // Windows 安全版本
    // 第4步：tm → 格式化字符串
    std::ostringstream oss;
    oss << std::put_time(&tm_buf, "%Y-%m-%d %H:%M:%S");
    return oss.str();
}
```

关键细节：
- 第 1 步的间接转换是因为文件系统时钟和系统时钟的 epoch（时间起点）可能不同，不能直接转
- `localtime_s` 是 Windows 安全版本，`localtime` 有线程安全问题
- `tm` 结构体的 `tm_year` 是年份-1900，`tm_mon` 是月份 0-11，`put_time` 会自动处理这些偏移
- 格式符：`%Y`=4位年、`%m`=2位月、`%d`=2位日、`%H`=2位时、`%M`=2位分、`%S`=2位秒

#### 经验总结
- C++ 时间转换记住 4 步走：`file_time_type` → `time_point` → `time_t` → `tm` → 字符串
- 第 1 步的间接转换是固定写法，不用完全理解原理，照抄即可
- Windows 下用 `localtime_s` 不用 `localtime`
- `std::put_time` 是 C++11 的格式化操纵器，比 `sprintf` 更安全、更类型安全
- 需要的头文件：`<chrono>`、`<ctime>`、`<iomanip>`、`<sstream>`

---

### 知识点 3：static_cast vs C 风格强制转换

#### 现象
代码中需要把 `uint64_t` 转为 `double` 进行浮点除法，可以用 `(double)bytes` 或 `static_cast<double>(bytes)`。

#### 排查与原因
- C 风格转换 `(double)x` 是"万能转换"，编译器几乎不做检查，什么都能转，包括危险操作（去掉 const、指针乱转）
- `static_cast<double>(x)` 是 C++ 风格转换，编译器会检查转换是否合法，不合法直接编译报错
- `static_cast` 只做良性转换（数值、继承、void*），不允许去掉 const 或指针类型乱转

#### 解决方案
C++ 项目中一律用 `static_cast`：

```cpp
double kb = static_cast<double>(bytes) / 1024.0;  // ✅ C++ 风格
// double kb = (double)bytes / 1024.0;             // ❌ C 风格，不推荐
```

#### 经验总结
- C++ 有四种转换运算符：`static_cast`（良性转换）、`dynamic_cast`（多态向下转型）、`const_cast`（去掉 const）、`reinterpret_cast`（重新解释内存，最危险）
- C 风格转换会按顺序尝试这四种，哪个能转就用哪个，所以最危险
- C++ 项目推荐用 `static_cast`，更安全、更可读、更容易搜索定位

---

### 最终验证输出

```
========================================
完整路径: "C:\Users\test\Documents\报告.pdf"
文件名:   报告.pdf
后缀:     pdf
文件大小: 3.5 MB
修改时间: 2026-09-12 20:06:27
分类:     文档
========================================
========================================
完整路径: "D:\test\note.txt"
文件名:   note.txt
后缀:     txt
文件大小: 856 B
修改时间: 2026-09-12 20:06:27
分类:     其他
========================================
```

---

## 题 2：分级日志系统

### 题目目标
实现一个分级日志系统，支持 INFO/WARN/ERROR 三个级别，输出格式为 `[时间] [级别] 消息`，包括核心 `log` 函数和三个便捷函数 `logInfo`/`logWarn`/`logError`。

### 知识点 1：enum class 作用域枚举必须加类名前缀

#### 现象
使用 `enum class LogLevel { INFO, WARN, ERROR };` 定义枚举后，在 switch 中写 `case INFO:` 报编译错误，提示未定义标识符 `INFO`。

#### 排查与原因
- `enum class` 是 C++11 引入的**作用域枚举**（scoped enumeration），枚举值被限定在枚举类型的作用域内
- 与普通 `enum` 不同，作用域枚举的值**不会泄漏到外部命名空间**，必须通过 `EnumName::Value` 的方式访问
- 写 `case INFO:` 时，编译器在当前作用域找不到 `INFO`，因为它被限定在 `LogLevel::` 作用域内

#### 解决方案
所有枚举值前加 `LogLevel::` 前缀：

```cpp
std::string levelToString(LogLevel level) {
    switch (level) {
    case LogLevel::INFO:  return "INFO";   // ✅ 加作用域前缀
    case LogLevel::WARN:  return "WARN";
    case LogLevel::ERROR: return "ERROR";
    default:               return "UNKNOWN";
    }
}
```

#### 经验总结
- `enum class`（作用域枚举）vs 普通 `enum`（无作用域枚举）的核心区别：作用域枚举的值必须加 `EnumName::` 前缀，不会隐式转换为整数
- 作用域枚举更类型安全、更不容易命名冲突，C++ 项目推荐优先使用 `enum class`
- 普通 `enum` 的值会泄漏到外部作用域，容易和其他变量/宏同名冲突，这也是题 2 遇到 Windows `ERROR` 宏冲突的间接原因之一

---

### 知识点 2：Windows.h 宏冲突与 #undef

#### 现象
定义 `enum class LogLevel { INFO, WARN, ERROR };` 后编译报 4 个语法错误（C2143 缺少分号、C2059 语法错误"常数"），全部指向 `ERROR` 所在位置。

#### 排查与原因
- `<windows.h>` 中定义了大量宏，包括 `#define ERROR 0`（用于 Windows 错误码常量）
- C/C++ 预处理器在编译前会进行纯文本宏替换，把所有 `ERROR` 替换为 `0`
- 代码 `enum class LogLevel { INFO, WARN, ERROR };` 被预处理器替换为 `enum class LogLevel { INFO, WARN, 0 };`
- `0` 是常数，不能作为枚举值名称，导致语法错误
- 错误信息中的"常数"就是被替换后的 `0`

#### 解决方案
在 `#include <windows.h>` 之后加 `#undef ERROR` 取消宏定义：

```cpp
#include <windows.h>
#undef ERROR   // 取消 Windows 定义的 ERROR 宏，避免和枚举值冲突

enum class LogLevel { INFO, WARN, ERROR };  // ✅ ERROR 不再被替换
```

#### 经验总结
- `<windows.h>` 定义了大量常见名称的宏：`ERROR`、`INFO`、`DEBUG`、`MAX`、`MIN`、`OUT`、`IN` 等，Windows C++ 开发中经常和用户代码命名冲突
- 宏替换是纯文本替换，不做语法检查，冲突时往往报莫名其妙的语法错误，排查时要想到宏冲突的可能
- 解决方案：`#undef 宏名` 取消冲突的宏，或把自己的变量/枚举值改名（如 `ERROR_LEVEL`）
- 预防：包含 `<windows.h>` 后检查是否有常用名称被宏定义，必要时批量 `#undef`

---

### 知识点 3：便捷函数封装核心函数

#### 现象
需要支持 `logInfo("...")`、`logWarn("...")`、`logError("...")` 三种调用方式，同时又要有一个统一的核心 `log` 函数处理输出逻辑。

#### 排查与原因
- 如果每个级别都写一套完整的输出逻辑（获取时间、转级别字符串、组装、输出），会产生大量重复代码
- 核心逻辑（组装 `[时间] [级别] 消息`）对所有级别都是一样的，只有级别参数不同
- 便捷函数的作用是：封装常用调用方式，内部调用核心函数，减少重复代码

#### 解决方案
核心 `log` 函数处理所有输出逻辑，三个便捷函数只负责传入对应的级别：

```cpp
// 核心函数：处理所有输出逻辑
void log(LogLevel level, const std::string& message) {
    std::cout << "[" << currentTime() << "] [" << levelToString(level) << "] " << message << std::endl;
}

// 便捷函数：只传级别，内部调用核心函数
void logInfo(const std::string& message) {
    log(LogLevel::INFO, message);
}

void logWarn(const std::string& message) {
    log(LogLevel::WARN, message);
}

void logError(const std::string& message) {
    log(LogLevel::ERROR, message);
}
```

#### 经验总结
- 便捷函数（convenience function）模式：核心函数处理完整逻辑，便捷函数封装常用参数组合，内部调用核心函数
- 好处：减少重复代码、统一入口、便于维护（修改输出格式只改核心函数一处）
- 这是 C++ 标准库和很多开源项目的常见设计模式，如 `std::make_unique` 是 `new` 的便捷封装
- 调用方可以选择用便捷函数（简洁）或核心函数（灵活，可传任意级别）

---

### 知识点 4：当前时间获取比文件时间少一步转换

#### 现象
题 1 格式化文件修改时间需要 4 步转换（file_time_type → time_point → time_t → tm → 字符串），题 2 获取当前时间只需要 3 步。

#### 排查与原因
- 题 1 的输入是 `fs::file_time_type`，用的是**文件系统时钟**，和 `system_clock` 是两个不同的时钟，epoch（时间起点）可能不同，必须先用"时间差法"间接转换为 `system_clock::time_point`
- 题 2 获取当前时间直接调用 `std::chrono::system_clock::now()`，返回的本来就是 `system_clock::time_point`，不需要第 1 步转换
- 所以题 2 只有 3 步：`now()` → `time_t` → `tm` → 字符串

#### 解决方案
```cpp
std::string currentTime() {
    // 第1步：直接获取当前时间（已经是 system_clock::time_point，不需要转换）
    auto now = std::chrono::system_clock::now();
    
    // 第2步：转 time_t
    std::time_t tt = std::chrono::system_clock::to_time_t(now);
    
    // 第3步：转本地时间 tm
    std::tm tm_buf;
    localtime_s(&tm_buf, &tt);
    
    // 第4步：格式化字符串
    std::ostringstream oss;
    oss << std::put_time(&tm_buf, "%Y-%m-%d %H:%M:%S");
    return oss.str();
}
```

#### 经验总结
- 时间转换的步数取决于输入类型：输入是 `system_clock::time_point` 就少一步，输入是其他时钟的 `time_point`（如 `file_time_type`）就多一步间接转换
- `std::chrono::system_clock::now()` 是获取当前时间的标准方式，返回 `system_clock::time_point`
- 题 1 学的 4 步转换是通用模板，题 2 是简化版，核心的 `time_t → tm → 字符串` 三步完全一样
- 时间相关代码复用性很高，学会一次可以到处用

---

### 最终验证输出

```
[2026-09-12 20:52:58] [INFO] 开始扫描目录: D:\test
[2026-09-12 20:52:58] [INFO] 找到 100 个文件
[2026-09-12 20:52:58] [WARN] 跳过隐藏文件: .hidden.txt
[2026-09-12 20:52:58] [WARN] 文件重名，已自动改名: 报告(1).pdf
[2026-09-12 20:52:58] [ERROR] 无法打开目录: D:\not_exist
[2026-09-12 20:52:58] [ERROR] 文件移动失败: 权限不足
```

---

## 题 3：目录扫描器

### 题目目标
实现一个目录扫描函数，遍历指定目录下的所有文件（不递归子目录），跳过隐藏文件，收集每个文件的完整信息，返回 `std::vector<FileInfo>` 列表。

### 知识点 1：std::filesystem 目录遍历

#### 现象
需要遍历指定目录下的所有文件，获取每个文件的路径、大小、修改时间等信息。

#### 排查与原因
C++17 引入的 `std::filesystem` 库提供了目录遍历能力，有两个迭代器：
- `directory_iterator(path)`：只遍历当前目录的直接子条目，**不递归**
- `recursive_directory_iterator(path)`：递归遍历所有子目录

遍历时每个条目是 `directory_entry` 对象，提供以下常用方法：
- `entry.path()`：获取完整路径
- `entry.is_regular_file()`：判断是否为普通文件（跳过子目录）
- `entry.is_directory()`：判断是否为子目录
- `entry.file_size()`：获取文件大小（字节）
- `entry.last_write_time()`：获取最后修改时间

#### 解决方案
```cpp
for (const auto& entry : fs::directory_iterator(dirPath)) {
    // 只处理普通文件，跳过子目录
    if (!entry.is_regular_file()) continue;
    
    // 获取文件信息
    FileInfo info;
    info.fullPath = entry.path();
    info.fileSize = entry.file_size();
    info.modifyTime = entry.last_write_time();
    result.push_back(info);
}
```

#### 经验总结
- `directory_iterator` 不递归，`recursive_directory_iterator` 递归，根据需求选择
- `is_regular_file()` 过滤掉子目录、符号链接、设备文件等非普通文件
- `directory_entry` 的方法会缓存文件属性，多次调用不重复访问文件系统，性能好
- 遍历大目录时建议用 `directory_iterator` 而非先列目录再逐个 stat，减少系统调用

---

### 知识点 2：隐藏文件判断与越界防护

#### 现象
需要跳过隐藏文件（如 `.gitignore`、`.hidden`），判断文件名是否以 `.` 开头。

#### 排查与原因
- 隐藏文件的定义：文件名以 `.` 开头（Unix/Linux 惯例，Windows 也遵循）
- 获取文件名：`filePath.filename().string()`
- 判断首字符：`filename[0] == '.'`
- **坑**：如果文件名为空字符串，`filename[0]` 会越界访问，导致未定义行为
- 正常情况下文件名不会为空，但健壮的代码应该先判空再取首字符

#### 解决方案
```cpp
bool isHiddenFile(const fs::path& filePath) {
    std::string filename = filePath.filename().string();
    if (!filename.empty() && filename[0] == '.') {  // 先判空再取首字符
        return true;
    }
    return false;
}
```

#### 经验总结
- 访问字符串/数组的第一个元素前，必须先判空，避免越界
- `filename()` 返回的是 `path` 类型，需要 `.string()` 转成 `std::string` 才能用 `[0]` 取字符
- 隐藏文件判断只看文件名首字符，不看路径（`/home/.config/file.txt` 中 `file.txt` 不是隐藏文件）
- Windows 上还有一种"系统隐藏文件"（通过文件属性标记），但题 3 只处理以 `.` 开头的简单隐藏文件

---

### 知识点 3：异常隔离（重点）

#### 现象
扫描系统目录或包含特殊文件的目录时，某个文件可能无权限访问、被其他程序占用、或文件系统错误，导致整个扫描崩溃。

#### 排查与原因
- `file_size()`、`last_write_time()` 等操作在文件无权限或被占用时会抛出 `std::filesystem::filesystem_error` 异常
- 如果不捕获异常，一个文件的失败会导致整个 `for` 循环终止，后续文件都无法扫描
- 批量文件操作中，单个失败是常态，不应影响整体流程

#### 解决方案
用 `try-catch` 包裹单个文件的信息获取，捕获异常后打 WARN 日志，继续扫描下一个文件：

```cpp
for (const auto& entry : fs::directory_iterator(dirPath)) {
    if (!entry.is_regular_file()) continue;
    if (isHiddenFile(entry.path())) continue;
    
    try {
        FileInfo info;
        info.fullPath = entry.path();
        info.fileName = entry.path().filename().string();
        info.fileSize = entry.file_size();
        info.modifyTime = entry.last_write_time();
        result.push_back(info);
    }
    catch (const std::exception& e) {
        logWarn("跳过文件(读取失败): " + entry.path().string() + " - " + e.what());
    }
}
```

#### 经验总结
- 批量操作（扫描、下载、转换）中，异常隔离是必须的，不能让一个失败拖垮整体
- `try-catch` 放在循环内部，包裹单个文件的操作，而不是包裹整个循环
- 捕获异常后要记录日志（文件名 + 错误原因），便于事后排查
- `std::filesystem::filesystem_error` 继承自 `std::exception`，用 `const std::exception&` 捕获即可
- 异常隔离是 file_sorter 项目的核心设计原则之一，后续 mover、classifier 模块都要遵循

---

### 知识点 4：fs::path 的编码坑（最重点）

#### 现象
加了 `/utf-8` 编译选项和 `SetConsoleOutputCP(CP_UTF8)` 后，源码里的中文字符串正常显示，但 `std::cout << file.fullPath` 输出的中文路径仍然乱码。

#### 排查与原因
- Windows 上 `std::filesystem::path` 内部存储的是 **UTF-16**（`wchar_t`）
- `path::string()` 方法用 **ANSI 代码页（GBK）** 把内部的 UTF-16 转成窄字符串
- **关键**：这个转换是标准库的**运行时行为**，不受 `/utf-8` 编译选项影响
- `/utf-8` 只控制源码字符串字面量的编译期编码，管不到 `fs::path` 的内部转换
- 所以即使加了 `/utf-8`，`path::string()` 和 `operator<<` 输出的仍然是 GBK
- UTF-8 控制台解读 GBK 字节 → 乱码

| 方法 | 返回编码 | 受 /utf-8 影响 |
|---|---|---|
| `path::string()` | GBK（ANSI 代码页） | ❌ 不受影响 |
| `path::u8string()` | UTF-8 | ✅ 直接转 UTF-8 |
| `path::wstring()` | UTF-16 | - |

#### 解决方案
所有 `fs::path` 转字符串都用 `.u8string()`，绕过 ANSI 代码页：

```cpp
// 填充 FileInfo 时
info.fullPath = entry.path().u8string();           // ✅ UTF-8
info.fileName = entry.path().filename().u8string(); // ✅ UTF-8
info.ext = entry.path().extension().u8string();      // ✅ UTF-8

// 输出时（C++20 的 u8string 不能直接 cout，需要转）
std::cout << std::string(file.fullPath.begin(), file.fullPath.end());
```

#### 经验总结
- Windows 上 `fs::path` 的编码是 `std::filesystem` 最大的坑，几乎所有中文路径乱码都源于此
- 记住：`string()` = GBK，`u8string()` = UTF-8，Windows 上用 UTF-8 必须选 `u8string()`
- `/utf-8` 编译选项管不到标准库运行时行为，不要以为加了 `/utf-8` 就万事大吉
- 异常日志里的路径也要用 `.u8string()`，否则触发异常时日志里的路径也乱码
- 这是 file_sorter 项目全模块都要注意的问题，scanner、mover、organizer 都涉及路径处理

---

### 知识点 5：完整 UTF-8 方案三步（缺一不可）

#### 现象
配置 UTF-8 环境时，经常出现"中文标签正常但中文路径乱码"或"全部乱码"的部分正常现象。

#### 排查与原因
完整 UTF-8 方案需要三步同时配置，任何一步缺失都会导致部分或全部乱码：

| 步骤 | 操作 | 作用 | 缺失后果 |
|---|---|---|---|
| 1 | `.vcxproj` 加 `/utf-8` | 源码字符串字面量运行时为 UTF-8 | 中文标签乱码 |
| 2 | `main()` 加 `SetConsoleOutputCP(CP_UTF8)` | 控制台按 UTF-8 解读输出 | 全部乱码 |
| 3 | `fs::path` 用 `.u8string()` | 绕过 ANSI 代码页，强制 UTF-8 | 中文路径/文件名乱码 |

- 只做 1、2：标签正常，路径乱码（最常见的坑）
- 只做 2、3：路径正常，标签乱码
- 只做 1、3：标签和路径都是 UTF-8，但控制台按 GBK 解读 → 全部乱码
- 三步全做：全部正常

#### 解决方案
三步同时配置：

```xml
<!-- 第1步：.vcxproj 加 /utf-8 -->
<AdditionalOptions>/utf-8 %(AdditionalOptions)</AdditionalOptions>
```

```cpp
// 第2步：main() 开头加
SetConsoleOutputCP(CP_UTF8);
SetConsoleCP(CP_UTF8);  // 输入也用 UTF-8（可选）

// 第3步：所有 fs::path 转字符串用 .u8string()
info.fullPath = entry.path().u8string();
```

#### 经验总结
- 完整 UTF-8 方案是"三件套"，缺一不可，记成口诀：`/utf-8` + `SetConsoleOutputCP` + `u8string()`
- 排查中文乱码时，按这三步逐一检查，哪步缺了补哪步
- 如果项目不需要跨平台，用方案 A（UTF-8 BOM + 默认 GBK 控制台）更简单，不需要这三步
- 跨平台/开源项目推荐方案 B（完整 UTF-8），一次配置全平台一致
- file_sorter 项目目前用方案 B，所有模块都要遵循这三步

---

### 知识点 6：std::u8string 与 C++17/C++20 差异

#### 现象
C++20 环境下，`std::cout << file.fullPath`（`fullPath` 是 `std::u8string`）报编译错误，提示没有匹配的 `operator<<`。

#### 排查与原因
- C++17：`path::u8string()` 返回 `std::string`，可直接 `cout`
- C++20：`path::u8string()` 返回 `std::u8string`（即 `std::basic_string<char8_t>`）
- C++20 引入了 `char8_t` 类型，用于表示 UTF-8 代码单元，与 `char` 不隐式转换
- `std::u8string` 不能直接 `cout`，因为 `char8_t` 没有定义 `operator<<`
- 这是 C++20 的破坏性变更，C++17 能编译的代码升到 C++20 可能报错

#### 解决方案
C++20 下把 `u8string` 转成普通 `string` 再输出：

```cpp
// 方法 1：迭代器构造（推荐，清晰可读）
std::string(file.fullPath.begin(), file.fullPath.end())

// 方法 2：reinterpret_cast（性能好，但不够安全）
reinterpret_cast<const char*>(file.fullPath.c_str())

// 封装成工具函数
std::string u8toString(const std::u8string& u8str) {
    return std::string(u8str.begin(), u8str.end());
}
```

#### 经验总结
- C++17 和 C++20 的 `u8string()` 返回类型不同，这是 C++20 的破坏性变更
- 项目用 C++17 就行（`std::filesystem` 是 C++17 引入的），避免 `char8_t` 的麻烦
- 如果必须用 C++20，封装一个 `u8toString()` 工具函数，所有输出统一调用
- `char8_t` 的设计初衷是类型安全（区分 UTF-8 字符串和普通字节串），但实际使用中增加了转换成本
- file_sorter 项目目前用 C++17，`u8string()` 返回 `std::string`，可直接使用

---

### 知识点 7：clog vs cout vs cerr

#### 现象
日志输出和正常程序输出都用 `std::cout`，无法分离，重定向时日志和结果混在一起。

#### 排查与原因
C++ 标准库提供了三个标准流，用途不同：

| 流 | 绑定 | 缓冲 | 用途 |
|---|---|---|---|
| `cout` | stdout（标准输出） | 行缓冲 | 程序正常输出结果 |
| `clog` | stderr（标准错误） | 全缓冲 | 日志信息（INFO/WARN） |
| `cerr` | stderr（标准错误） | 无缓冲（立即刷新） | 错误信息（ERROR） |

- `cout` 和 `clog`/`cerr` 绑定到不同的文件描述符（1 和 2），可以独立重定向
- `clog` 全缓冲，性能好，适合频繁的日志输出
- `cerr` 无缓冲，每次输出立即刷新，适合关键错误（确保崩溃前能输出）
- 都用 `cout` 的话，重定向 `> result.txt` 会把日志也写进文件，无法分离

#### 解决方案
日志用 `clog`，正常输出用 `cout`，错误用 `cerr`：

```cpp
// 日志 → clog（stderr）
void logInfo(const std::string& message) {
    std::clog << "[INFO] " << message << std::endl;
}

// 正常输出 → cout（stdout）
void printFileInfo(const FileInfo& file) {
    std::cout << "文件名: " << file.fileName << "\n";
}

// 错误 → cerr（stderr，无缓冲）
void logError(const std::string& message) {
    std::cerr << "[ERROR] " << message << std::endl;
}
```

重定向分离：
```bash
# 正常输出存文件，日志显示在屏幕
my_program.exe > result.txt

# 只看错误日志
my_program.exe 2> error.log

# 输出和日志分别存文件
my_program.exe > result.txt 2> app.log
```

#### 经验总结
- 日志和正常输出分离是工程实践的基本要求，不要都用 `cout`
- `clog`（全缓冲）适合 INFO/WARN 日志，`cerr`（无缓冲）适合 ERROR 错误
- 好处：可独立重定向、性能更好（clog 全缓冲减少系统调用）、语义清晰
- file_sorter 项目的 logger 模块遵循此规范：INFO/WARN 用 `clog`，ERROR 用 `cerr`
- 题 2 的分级日志系统已经体现了这个设计，题 3 继续沿用

---

### 踩坑记录

1. **只加 `SetConsoleOutputCP` 不加 `/utf-8`** → 源码字符串 GBK，控制台 UTF-8 → 全部乱码
2. **`fs::path` 直接 `cout`** → Windows 上输出 GBK，UTF-8 控制台 → 路径乱码
3. **`std::u8string` 直接 `cout`** → C++20 编译错误（`char8_t` 没有 `operator<<`）
4. **`filename[0]` 未判空** → 空文件名时越界访问
5. **异常不隔离** → 一个无权限文件导致整个扫描崩溃
6. **异常日志里用 `.string()`** → 触发异常时日志里的路径也乱码（要用 `.u8string()`）

---

### 最终验证输出

```
扫描目录: C:\Users\test\Documents

[INFO] 扫描完成，共找到 3 个文件

===== 扫描结果 =====
----------------------------------------
文件名:   报告.pdf
后缀:     .pdf
大小:     1048576 B
路径:     "C:\Users\test\Documents\报告.pdf"
----------------------------------------
文件名:   代码.cpp
后缀:     .cpp
大小:     4096 B
路径:     "C:\Users\test\Documents\代码.cpp"
----------------------------------------
文件名:   note.txt
后缀:     .txt
大小:     856 B
路径:     "C:\Users\test\Documents\note.txt"
----------------------------------------

共 3 个文件
```

（隐藏文件如 `.gitignore` 被跳过，不显示）

---

## 题 4：后缀分类器

### 题目目标
实现一个后缀分类函数，根据文件后缀将文件分类到 7 大类之一（IMAGE/DOCUMENT/VIDEO/AUDIO/ARCHIVE/PROGRAM/OTHER）。分类规则集中管理，便于后续扩展和维护。

### 知识点 1：后缀标准化（去点 + 转小写）

#### 现象
用户传入的后缀可能是 `.PDF`、`pdf`、`.Pdf`、`PDF` 等各种形式，如果直接拿去查表，`.PDF` 和 `pdf` 会被当成两个不同的键，导致匹配失败。

#### 排查与原因
- 分类规则表里的键是统一格式（小写、无点，如 `pdf`）
- 输入后缀的格式不统一：可能带点、可能大写、可能混合大小写
- 不标准化就直接查表，会因为格式不匹配导致查找失败
- 标准化是数据清洗的第一步，确保输入和规则表的键格式一致

#### 解决方案
`normalizeExtension` 函数做两件事：
1. 去掉开头的 `.`（如果有）
2. 全部转小写

```cpp
std::string normalizeExtension(const std::string& ext) {
    std::string result = ext;
    // 去掉开头的点
    if (!result.empty() && result[0] == '.') {
        result = result.substr(1);
    }
    // 全部转小写
    for (char& ch : result) {
        ch = std::tolower(static_cast<unsigned char>(ch));
    }
    return result;
}
```

#### 经验总结
- 标准化是数据处理的第一步，确保输入格式统一后再进行匹配/比较
- `std::tolower` 的参数必须转 `unsigned char`，否则负数字符（如中文）会导致未定义行为
- `substr` 返回新字符串，不会修改原字符串，必须赋值回去
- 空字符串要安全处理：先判空再取 `[0]`，避免越界访问
- 标准化函数应纯函数化：输入相同，输出一定相同，无副作用

---

### 知识点 2：分类规则表集中管理（unordered_map + static const 单例）

#### 现象
如果用一堆 if-else 判断后缀，代码冗长、难以维护、新增后缀要改逻辑代码。

#### 排查与原因
- if-else 写法：每新增一个后缀就要加一个 `else if`，逻辑和数据混在一起
- 后缀有几十个，if-else 会写几十行，可读性差
- 分类规则是**数据**，不是**逻辑**，应该和逻辑分离
- 用 map 存储后缀→分类的映射，数据和逻辑分离，新增后缀只改数据不改逻辑

#### 解决方案
用 `std::unordered_map<std::string, FileCategory>` 集中管理所有后缀映射，用 `static const` 局部变量实现单例：

```cpp
const std::unordered_map<std::string, FileCategory>& getCategoryMap() {
    static const std::unordered_map<std::string, FileCategory> categoryMap = {
        {"jpg", FileCategory::IMAGE}, {"jpeg", FileCategory::IMAGE},
        {"pdf", FileCategory::DOCUMENT}, {"doc", FileCategory::DOCUMENT},
        // ... 其他后缀
    };
    return categoryMap;
}
```

#### 经验总结
- 数据和逻辑分离：分类规则是数据，用 map 存储；分类逻辑是代码，用 find 查表
- `static const` 局部变量是 C++ 单例的最佳实践：只初始化一次、不可修改、作用域局部、延迟初始化、线程安全（C++11 起）
- `unordered_map` 平均 O(1) 查找，比 if-else 链（O(n)）快得多
- 新增后缀只改 map 数据，不改逻辑代码，符合开闭原则（对扩展开放，对修改关闭）
- 返回 `const` 引用，调用方不能修改规则表，保证数据安全

---

### 知识点 3：enum class 转中文（switch）

#### 现象
分类结果是 `FileCategory` 枚举值，需要转换成中文显示给用户（如"图片"、"文档"）。

#### 排查与原因
- `enum class` 是作用域枚举，不会隐式转换为整数，不能直接用数组下标
- 枚举值和中文名称的映射是固定的，用 switch 最清晰
- switch 的每个 case 对应一个枚举值，返回对应的中文，default 处理未知值

#### 解决方案
```cpp
std::string categoryToChinese(FileCategory category) {
    switch (category) {
    case FileCategory::IMAGE:    return "图片";
    case FileCategory::DOCUMENT: return "文档";
    case FileCategory::VIDEO:    return "视频";
    case FileCategory::AUDIO:    return "音频";
    case FileCategory::ARCHIVE:  return "压缩包";
    case FileCategory::PROGRAM:  return "程序/代码";
    case FileCategory::OTHER:    return "其他";
    default:                      return "未知";
    }
}
```

#### 经验总结
- enum class 转字符串用 switch 最清晰，每个 case 一目了然
- 必须加 `default` 处理未知枚举值，避免 switch 没有返回值导致未定义行为
- 也可以用 `std::unordered_map<FileCategory, std::string>` 实现，但 switch 性能更好（编译器可能优化为跳转表）
- 枚举转字符串是常见需求，建议每个枚举都配套一个 `xxxToString` 函数
- 题 2 的 `levelToString` 也是同样的模式，可复用

---

### 知识点 4：默认值处理（未知后缀返回 OTHER）

#### 现象
用户文件的后缀千奇百怪，不可能全部预定义在规则表里，未知后缀需要有合理的默认处理。

#### 排查与原因
- 规则表只覆盖常见后缀，未知后缀（如 `.xyz`、`.unknown`）查表会失败
- 如果不处理查找失败，可能返回无效枚举值或崩溃
- `OTHER` 分类就是为未知后缀准备的"兜底分类"
- `find` 失败时返回 `end()` 迭代器，需要判断后返回默认值

#### 解决方案
```cpp
FileCategory classifyByExtension(const std::string& ext) {
    std::string normalized = normalizeExtension(ext);
    const auto& categoryMap = getCategoryMap();
    auto it = categoryMap.find(normalized);
    if (it != categoryMap.end()) {
        return it->second;  // 找到，返回对应分类
    }
    return FileCategory::OTHER;  // 没找到，返回默认分类
}
```

#### 经验总结
- 查表操作必须处理"没找到"的情况，不能假设输入一定在表里
- `end()` 是哨兵值，表示"没找到"，`it != end()` 是标准的查找成功判断
- 默认值处理保证函数总有合法返回值，避免崩溃或无效状态
- `OTHER` 分类是兜底设计，所有无法分类的文件都归入此类
- 边界情况（空后缀、未知后缀、超长后缀）都应落入默认处理，不要特殊判断

---

### 知识点 5：char 是基本类型，没有成员方法

#### 现象
写 `for (auto& ch : result) { ch = ch.tolower(); }` 报编译错误，提示 `char` 没有 `tolower` 方法。

#### 排查与原因
- `ch` 的类型是 `char`，C++ 的基本类型（int、char、float 等）不是对象，没有成员方法
- `tolower` 是 C 标准库函数（`<cctype>`），不是 `char` 的方法
- 基本类型的操作都通过函数或运算符完成，不能用 `.` 调用方法
- 只有类类型（class/struct）的对象才能用 `.` 调用成员方法

#### 解决方案
```cpp
for (char& ch : result) {
    ch = std::tolower(static_cast<unsigned char>(ch));  // ✅ 用标准库函数
}
```

#### 经验总结
- C++ 基本类型（int/char/float/double/bool 等）不是对象，没有成员方法
- 字符操作（转大小写、判断字母/数字等）用 `<cctype>` 里的标准库函数：`tolower`、`toupper`、`isalpha`、`isdigit`、`isspace` 等
- 不要把 Java/C# 的"万物皆对象"思维带到 C++，C++ 基本类型就是纯数据
- 范围 for 循环 `for (char& ch : result)` 中 `ch` 是引用，修改 `ch` 会修改原字符串

---

### 踩坑记录

1. **`normalizeExtension` 逻辑写反**：`if (!result.empty()) return result;` 导致非空后缀直接返回，去点和转小写根本执行不到。应该是 `if (result.empty()) return result;`，或者直接删掉这行（空字符串后续操作也安全）。
2. **`substr` 没赋值**：`result.substr(1)` 返回新字符串，不会修改原 `result`，必须写成 `result = result.substr(1);`。
3. **`char.tolower()` 错误**：`char` 是基本类型，没有成员方法，要用 `std::tolower(ch)` 标准库函数。
4. **`std::tolower` 参数没转 `unsigned char`**：负数字符（如中文）传入会导致未定义行为，必须写 `std::tolower(static_cast<unsigned char>(ch))`。
5. **规则表用 if-else 链**：几十个后缀写 if-else 冗长难维护，应该用 `unordered_map` 集中管理，数据和逻辑分离。
6. **`find` 失败没处理**：查表后直接 `return it->second`，没判断 `it != end()`，未知后缀会解引用无效迭代器导致崩溃。

---

### 最终验证输出

```
===== 后缀分类测试 =====

原始后缀: .pdf  →  标准化: pdf  →  分类: 文档
原始后缀: PDF   →  标准化: pdf  →  分类: 文档
原始后缀: .JPG  →  标准化: jpg  →  分类: 图片
原始后缀: png   →  标准化: png  →  分类: 图片
原始后缀: .MP4  →  标准化: mp4  →  分类: 视频
原始后缀: mp3   →  标准化: mp3  →  分类: 音频
原始后缀: .zip  →  标准化: zip  →  分类: 压缩包
原始后缀: cpp   →  标准化: cpp  →  分类: 程序/代码
原始后缀: .exe  →  标准化: exe  →  分类: 程序/代码
原始后缀: .unknown  →  标准化: unknown  →  分类: 其他
原始后缀: .Md   →  标准化: md   →  分类: 文档
原始后缀: .HTML →  标准化: html →  分类: 程序/代码
原始后缀: (空)  →  标准化: (空) →  分类: 其他
```
