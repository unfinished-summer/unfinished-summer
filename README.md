# 👋 Hi, I'm 半夏未央

计算机专业学生，对 **系统编程、网络安全、游戏开发** 方向感兴趣。
主要写 C/C++，也用 Python 和前端，习惯把踩过的坑记下来归档。

---

## 🌱 开源贡献

- **[py-simple-wrap #328](https://github.com/sara-czasak/py-simple-wrap/pull/328)** — 为 `easy_random` 模块新增 `random_color()` 工具函数及单元测试，PR 已合并进 main，收录于项目 [CONTRIBUTORS.md](https://github.com/sara-czasak/py-simple-wrap/blob/main/CONTRIBUTORS.md)

---

## 📂 个人项目

- **file_sorter** — C++ 文件自动分类整理工具，命令行版，支持后缀分类、重名处理、配置文件注入、预览/执行双模式
- **sort-raylib** — C++ + Raylib 排序算法可视化，支持冒泡/选择/插入/快排/归并动画，多线程渲染（踩坑记录见下方文档）

---

## 🛠 技术栈

- **语言**：C/C++（C++20）、Python、JavaScript/HTML/CSS
- **工具**：Git、CMake、Visual Studio、vcpkg
- **方向**：系统编程、文件处理、图形可视化

---

## 📚 踩坑文档归档

本仓库用于归档开发过程中遇到的各类踩坑记录，方便复盘查阅：

- [Raylib 排序可视化项目踩坑文档](./docs/sort-raylib/bug_record.md) — vcpkg 与 MSVC 不兼容、白屏、多线程渲染
- [通用环境踩坑记录](./docs/通用环境踩坑.md) — 头文件路径、LF/CRLF、中文乱码、.gitignore 误配置
- [file_sorter 实操题经验总结](./docs/file_sorter实操题经验.md) — C++ 工程化递进练习（10 题）：日志、扫描、分类、移动、配置注入、命令行解析
