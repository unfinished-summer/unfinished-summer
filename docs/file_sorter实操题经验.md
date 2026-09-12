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

（待完成）
