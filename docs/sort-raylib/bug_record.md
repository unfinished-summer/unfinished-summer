# docs/sort-raylib/bug_record.md
## 项目背景
基于 C++ + Raylib 实现的排序算法可视化工具，支持冒泡、选择、插入、快速、归并五种排序动画，多线程渲染动画，实时高亮对比元素。
编译环境：Visual Studio 2026 MSVC x64 Debug

## 环境依赖说明
- 图形库：raylib 6.x
- 编译链：Windows MSVC
- 补充：vcpkg 可安装 raylib，但存在运行兼容故障

## 核心踩坑记录
### 核心唯一根坑：vcpkg 打包 raylib 6.x 与 VS2026 MSVC 不兼容
#### 现象
1. 编译、链接无报错，正常生成 exe；
2. 程序可弹出窗口，但窗口全屏纯白色；
3. 鼠标移入窗口持续转圈，窗口无响应，无任何文字、图形渲染；
4. 业务代码无逻辑缺陷，项目置顶配置官方预编译 raylib 后白屏卡死问题完全消失。

#### 根因
vcpkg 打包 raylib 时缺失关键渲染宏，`EndDrawing()` 内部窗口消息、帧缓冲刷新逻辑阻塞；仅 vcpkg 分发版本存在该缺陷，官方预编译包、自行源码编译无问题；VS2026 MSVC 校验规则更严格，放大该阻塞故障。

#### 解决方案（不卸载 vcpkg 内 raylib）
1. Raylib 官网下载 MSVC x64 独立预编译 raylib 二进制包；
2. VS 项目置顶配置路径，覆盖 vcpkg 库优先级：
   - C/C++ - 附加包含目录：填入 `raylib/include` 路径（置顶）
   - 链接器 - 附加库目录：填入 `raylib/lib-msvc-x64` 路径（置顶）
   - 链接器 - 输入 - 附加依赖项：`raylib.lib;opengl32.lib;gdi32.lib;winmm.lib`
3. 编译阶段优先使用官方无 bug raylib，无需卸载 vcpkg 已安装的 raylib。

### 连锁衍生问题（仅优先加载 vcpkg raylib 时触发）
#### 衍生坑 1：LNK1104 无法打开 xxx.exe
- 触发原因：vcpkg raylib 卡死程序，进程后台残留，exe 文件被系统占用锁定。
- 修复方案：任务管理器结束所有 `Project1.exe` 残留进程，清理解决方案后重新生成。

#### 衍生坑 2：开启 `FLAG_WINDOW_HIGHDPI` 后窗口左上角大块白块
- 触发原因：vcpkg raylib + 高分屏 DPI 标记，帧缓冲像素坐标错位。
- 修复方案：删除 `SetConfigFlags(FLAG_WINDOW_HIGHDPI);`，使用系统默认缩放。

#### 衍生坑 3：多线程数组读写画面撕裂、卡顿加剧
- 触发原因：vcpkg raylib 渲染循环阻塞，放大多线程无锁数据竞争卡顿错乱。
- 修复方案：使用 `std::mutex` 互斥锁保护全局数组，渲染前拷贝数组副本缩短锁占用时间；切换官方 raylib 后卡顿明显缓解。

#### 衍生坑 4：数值较小时柱状图高度归零、完全不可见
- 触发原因：vcpkg raylib 渲染精度缺陷，放大整数截断视觉问题。
- 修复方案：全程浮点运算计算柱高，运算完成后统一转换整型；官方 raylib 该现象几乎不可感知。

## 开发经验总结
1. 本项目白屏、鼠标转圈、渲染异常、链接报错所有问题，根源仅为 vcpkg 打包 raylib 与 VS2026 MSVC 不兼容；其余问题均为该底层打包 bug 的连锁副作用。
2. VS2026 MSVC 工具链校验更严格，vcpkg 未正确构建的图形库运行阻塞表现更严重。
3. 不想卸载 vcpkg raylib 时，通过项目配置置顶官方 raylib 路径，即可强制编译使用稳定版本。
4. 包管理器编译通过不等于运行稳定，图形渲染库容易存在打包层面隐藏运行缺陷。
