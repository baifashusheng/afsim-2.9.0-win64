---
kind: frontend_style
name: AFSIM Qt/C++ 桌面端 UI 主题与样式体系
category: frontend_style
scope:
    - '**'
source_files:
    - swdev/src/mover_creator/source/MoverCreatorMainWindow.cpp
    - swdev/src/mover_creator/ui/resources/icons/Dark/index.theme
    - swdev/src/mover_creator/ui/resources/icons/Light/index.theme
    - swdev/src/mover_creator/ui/resources/icons.qrc
    - swdev/src/mover_creator/ui/MoverCreatorMainWindow.ui
    - swdev/src/mover_creator/ui/SettingsDialog.ui
    - swdev/src/mystic/plugins/ResultBehaviorAnalysisTool/source/RvBAT_GraphicsNode.cpp
---

## 概述

AFSIM 是一个以 C++/Qt 为核心的仿真系统，其前端界面（GUI）采用 **Qt Widgets + Qt Designer (.ui)** 构建，没有使用任何 Web 技术栈（无 CSS/SCSS/Tailwind）。UI 样式通过 Qt 自身的 `QPalette`、`QIcon::themeName()` 以及少量内联 `setStyleSheet` 实现，核心集中在 `swdev/src/mover_creator/` 子工程中，并在 `mystic/plugins/ResultBehaviorAnalysisTool/` 中复用图标主题机制。

## 1. 使用的框架与工具

- **GUI 框架**: Qt Widgets（C++），界面布局由 Qt Designer 生成的 `.ui` XML 文件定义（见 `swdev/src/mover_creator/ui/*.ui`，共约 40 个对话框/控件页面）。
- **资源管理**: Qt Resource System (`*.qrc`)，见 `swdev/src/mover_creator/ui/resources/icons.qrc` 与 `movercreator.qrc`。
- **图标主题**: 遵循 FreeDesktop Icon Theme 规范，提供 `Dark` / `Light` 两套图标主题，位于 `swdev/src/mover_creator/ui/resources/icons/{Dark,Light}/index.theme`。
- **着色器/图形渲染**: OpenGL/GLSL 着色器位于 `resources/shaders/`（107 个 `.glsl` 文件），服务于 3D 可视化后端（OSG/OpenGL），不属于 UI 样式范畴。

## 2. 关键文件

- `swdev/src/mover_creator/source/MoverCreatorMainWindow.cpp` — 主题切换主逻辑：读取用户设置中的 `theme` 键（默认 `"Dark"`），构造 `QPalette` 映射表并应用。
- `swdev/src/mover_creator/ui/resources/icons/Dark/index.theme` / `Light/index.theme` — FreeDesktop 图标主题清单。
- `swdev/src/mover_creator/ui/resources/icons.qrc` — 注册 Dark/Light 两套图标主题资源。
- `swdev/src/mover_creator/ui/*.ui` — 所有 Qt Designer 界面定义（AboutDialog、MoverCreatorMainWindow、SettingsDialog 等）。
- `swdev/src/mystic/plugins/ResultBehaviorAnalysisTool/source/RvBAT_GraphicsNode.cpp` — 在自定义图形节点中根据 `QIcon::themeName() == "Dark"` 动态选择文本颜色与图标路径。

## 3. 架构与约定

### 主题模型
- 主题名称为字符串常量 `"Dark"` 与 `"Light"`，保存在用户设置键 `"theme"` 中（`MoverCreatorMainWindow.cpp:80,154,388`）。
- `SetupThemes()` 基于系统默认 palette 派生 Light 主题，再复制一份生成 Dark 主题，覆盖 Base/Button/ButtonText/Light/Text/Window 等 role 的 RGB 值（如 Dark 主题 Base=`#1E1E20`、Button=`#2D2D2F`、Text=`#DCDCDE`）。
- 通过 `QPalette` 而非全局 `QApplication::setStyle()` 切换，避免依赖平台样式引擎。

### 图标主题
- 两套图标目录 `icons/Dark/64x64/` 与 `icons/Light/64x64/`，每个目录下放置同名 PNG/SVG 图标。
- 代码通过 `QIcon::themeName()` 返回当前主题名（`"Dark"` 或 `"Light"`），动态拼接资源路径 `":/images/<theme>/60x60/<name>"`（见 RvBAT_GraphicsNode.cpp:107,599,615）。

### 样式注入方式
- 大部分控件通过 Qt Designer 的 `.ui` 文件声明布局，运行时不写 CSS。
- 少数场景用内联 `setStyleSheet("color: white; background - color: gray")` 强制覆盖控件外观（见 CreateAircraftDialog.cpp、ConvertBrawlerAircraftDialog.cpp 等多处对 ComboBox 的硬编码样式）。
- 部分控件通过 `GetPalette().color(QPalette::Link)` 从全局 Palette 取色，保证与主题一致（AboutDialog.cpp:180）。

### 其他 GUI 模块
- `evt_reader/wsftheme/`、`mover_creator/wsftheme/` 中存在 `wsftheme` 目录，但内容极少（仅 1 文件），未形成统一样式库。
- `wizard/`、`post_processor/WizPostProcessor/`、`sensor_plot/` 等其它 Qt 应用未见独立的样式主题实现，推测沿用系统默认样式。

## 4. 观察到的约定与约束

- **约定**：新增主题时需在 `MoverCreatorMainWindow::SetupThemes()` 中扩展 `mPaletteMap`，并在 `ApplySettings()` 中同步更新 SettingsDialog 的单选按钮状态（Dark/Light 互斥）。
- **约定**：图标资源需同时放入 `icons/Dark/64x64/` 与 `icons/Light/64x64/`，并通过 `icons.qrc` 注册；加载时使用 `QIcon::themeName()` 动态选择。
- **约定**：自定义绘制的图形元素（RvBAT_* 系列）通过 `QIcon::themeName() == "Dark"` 分支选择前景色（`#EEEEEE` vs `cDEFAULT_TEXT_COLOR_LIGHT`），以保持可读性。
- **约束**：仓库中不存在任何 CSS/SCSS/Less 文件或 Tailwind 配置；前端样式完全由 Qt 原生 API 控制。
- **约束**：主题切换不会调用 `QIcon::setThemeName()`（该调用被注释掉），而是通过 `QPalette` 替换 + 手动刷新子 widget 的 `ChangeTheme()` 完成。

## 总结

AFSIM 的前端风格体系是典型的 **Qt Widgets + Qt Designer + QPalette 主题** 方案，辅以 FreeDesktop 规范的图标主题。它没有引入任何现代前端样式工程（CSS/SCSS/Tailwind/Web Components），样式集中在 `mover_creator` 子工程，并以字符串常量 `"Dark"` / `"Light"` 作为主题标识。