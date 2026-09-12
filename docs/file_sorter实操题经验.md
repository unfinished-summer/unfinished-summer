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

（待完成）
