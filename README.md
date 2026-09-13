# README.md
## Hi there 👋
<!--
同名个人仓库，仅存放各类开发项目踩坑文档，不包含任何项目源代码
所有故障记录分类归档在 docs 目录下，便于后续查阅复用
-->
### 关于我
- 计算机专业学生，对系统编程、安全与游戏开发领域感兴趣
- 主要技术栈：C/C++、Python、前端
- 兴趣：系统编程、逆向分析、漏洞挖掘、游戏开发
- 本仓库用于归档开发过程中遇到的各类踩坑记录

### 仓库介绍
本仓库是个人全开发场景踩坑归档库，**只存储各类项目故障记录文档，无任何业务源码、工程文件**。
收录内容涵盖 C++ 图形可视化、前端、算法、编译环境、包管理器、多线程、VS 开发环境等各类开发踩坑，用于复盘、避坑、查阅解决方案。

### 仓库目录结构
```
.
├── docs/
│   ├── sort-raylib/
│   │   └── bug_record.md   # C++ Raylib 排序算法可视化项目完整踩坑记录
│   ├── 通用环境踩坑.md      # 跨项目通用环境、工具类故障记录
│   ├── file_sorter实操题经验.md  # file_sorter 项目递进式实操题知识点与经验总结
│   └── 其他项目文件夹/      # 后续新增项目踩坑单独建文件夹存放
├── .gitignore               # 仓库忽略规则
└── README.md                # 仓库总介绍
```

### 当前已收录文档简介
#### 1. sort-raylib 排序可视化项目文档
文档路径：`docs/sort-raylib/bug_record.md`
1. 项目：C++ + Raylib 排序算法可视化工具，支持冒泡/选择/插入/快排/归并动画，多线程渲染
2. 编译环境：Visual Studio 2026 MSVC x64 Debug
3. 核心故障：vcpkg 打包 raylib 6.x 与 VS2026 MSVC 运行不兼容（白屏、窗口卡死、鼠标转圈）
4. 配套内容：核心根因、无卸载vcpkg的修复方案、四类连锁衍生故障、完整开发总结

#### 2. 通用环境踩坑记录
文档路径：`docs/通用环境踩坑.md`
1. C++ 头文件包含路径错误（无法打开包括文件 / 附加包含目录设置）
2. 文件行尾不一致（LF vs CRLF / 批量标准化）
3. 源码编码与中文乱码（UTF-8 BOM / /utf-8 编译选项）
4. .gitignore 误配置导致所有文件被忽略（.gitignore vs .gitattributes 语法区分）

#### 3. file_sorter 实操题经验总结
文档路径：`docs/file_sorter实操题经验.md`
1. 题 1：FileInfo 信息打印器 — 格式化函数职责分离（ostringstream 返回字符串）、C++ 时间类型转换 4 步走（file_time_type → time_point → time_t → tm → 字符串）、static_cast vs C 风格强制转换
2. 题 2：分级日志系统 — enum class 作用域枚举必须加类名前缀、Windows.h 宏冲突与 #undef、便捷函数封装核心函数、当前时间获取比文件时间少一步转换
3. 题 3：目录扫描器 — std::filesystem 目录遍历、隐藏文件判断与越界防护、异常隔离、fs::path 编码坑（string()=GBK / u8string()=UTF-8）、完整 UTF-8 方案三步（/utf-8 + SetConsoleOutputCP + u8string）、std::u8string 与 C++17/C++20 差异、clog vs cout vs cerr
4. 题 4：后缀分类器 — 后缀标准化（去点+转小写）、分类规则表集中管理（unordered_map + static const 单例）、enum class 转中文（switch）、默认值处理（未知后缀返回 OTHER）、char 是基本类型没有成员方法
5. 后续每完成一题追加一节，记录知识点、踩坑和经验

### 仓库使用规范
1. 每个新项目踩坑单独在 `docs/` 下新建同名文件夹，内部存放该项目专属 `bug_record.md`；
2. 通用、跨项目的环境/工具故障单独写通用文档，不归属单一项目；
3. 文档统一格式：项目背景、环境依赖、核心坑、衍生坑、根因、解决方案、经验总结；
4. 仓库不提交任何代码、解决方案、编译产物，仅保留 Markdown 文字记录。

### 适用人群
- C++ / Windows MSVC / Raylib 图形开发
- vcpkg 包管理器环境配置
- 多线程动画、可视化程序开发
- 各类编译、链接、运行时故障排查参考

### .gitignore
- [.gitignore](.gitignore)

## 文档快速跳转
- [Raylib排序可视化项目踩坑文档](./docs/sort-raylib/bug_record.md)
- [通用环境踩坑记录](./docs/通用环境踩坑.md)
- [file_sorter 实操题经验总结](./docs/file_sorter实操题经验.md)
- 后续新增项目文档会在此处补充链接
