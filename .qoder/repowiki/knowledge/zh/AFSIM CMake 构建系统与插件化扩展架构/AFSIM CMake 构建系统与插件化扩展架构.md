---
kind: build_system
name: AFSIM CMake 构建系统与插件化扩展架构
category: build_system
scope:
    - '**'
source_files:
    - swdev/src/CMakeLists.txt
    - swdev/src/CMakePresets.json
    - swdev/src/cmake/Modules/swdev_project.cmake
    - swdev/src/cmake/Modules/wsf_package.cmake
    - swdev/src/cmake/Modules/wsf_documentation.cmake
    - swdev/src/cmake/Modules/wsf_doxygen.cmake
    - swdev/src/cmake/Modules/ConfigureUnitTests.cmake
    - swdev/src/cmake/Modules/wsf_project_template.cmake
    - swdev/src/core/wsf/optional_projects.cmake
    - swdev/src/cmake/Presets/base.cmake
    - swdev/src/cmake/Presets/default.cmake
---

## 1. 构建系统总览

AFSIM 采用 **CMake** 作为唯一顶层构建系统，版本要求 `cmake_minimum_required(VERSION 3.7...3.12)`（`swdev/src/CMakeLists.txt:25`），C++ 标准固定为 `CMAKE_CXX_STANDARD 11`（第 32-34 行）。仓库不包含 Makefile、Dockerfile、CI YAML 或发布脚本；所有编译、打包、安装逻辑集中在 `swdev/src/` 下。

## 2. 核心文件与目录

- `swdev/src/CMakeLists.txt`：顶层入口，定义全局选项、版本号（`AFSIM_VERSION_MAJOR/MINOR/PATCH` 在 70-76 行）、安装路径、子项目注册。
- `swdev/src/cmake/Modules/`：自定义 CMake 模块集合，包括 `swdev_project.cmake`、`wsf_project_template.cmake`、`wsf_package.cmake`、`wsf_documentation.cmake`、`wsf_doxygen.cmake`、`ConfigureUnitTests.cmake`、`qt_project.cmake`、`vis_plugins.cmake` 等。
- `swdev/src/cmake/Presets/`：可选的构建预设片段（`base.cmake`、`default.cmake`、`exportable.cmake`、`restricted.cmake` 等），通过 `ENABLE_EXTENSION_DEFAULT` 控制默认启用的扩展集。
- `swdev/src/CMakePresets.json`：VS/Ninja 多生成器预设，覆盖 VS2015/2017/2019、Ninja、clang-cl、clang-linux 等组合。
- `swdev/src/core/wsf/optional_projects.cmake`：扫描并启用 WSF 扩展的核心机制。
- 各应用/扩展目录（`mission/`、`warlock/`、`mystic/`、`sensor_plot/`、`post_processor/`、`engage/`、`evt_reader/`、`mover_creator/`、`weapon_tools/`）各自拥有独立 `CMakeLists.txt` + `wsf_cmake_extension.cmake`。

## 3. 架构与约定

### 3.1 插件化扩展模型

顶层 CMake 通过 `file(GLOB_RECURSE ... wsf_cmake_extension.cmake)` 自动发现所有扩展（第 160-164 行），再由 `core/wsf/optional_projects.cmake` 统一加载。每个扩展必须提供 `wsf_cmake_extension.cmake` 并通过 `BUILD_WITH_<name>` 开关控制是否参与构建（见 `swdev/src/CMakeLists.txt:390-395`）。扩展可被放置在 `swdev/src/*/` 或外部 `afsim_*shared/*/wsf_module` 路径（第 140-145 行），支持通过 `WSF_ADD_EXTENSION_PATH` 缓存变量追加额外扩展目录。

### 3.2 构建产物布局

- 运行时 DLL/so 输出到 `${PROJECT_BINARY_DIR}`（Windows）或 `${PROJECT_BINARY_DIR}/bin/lib`（非 Windows），由 `INSTALL_DLL_PATH` / `INSTALL_EXE_PATH` / `INSTALL_PLUGIN_PATH` 控制（第 214-223 行）。
- 静态链接时 MSVC 强制 `/MT`，Linux 使用 `-static-libgcc`（第 280-301 行）。
- Linux 下统一设置 `-fPIC`、`-rdynamic -export-dynamic` 以导出符号（第 277-279、333-336 行）。
- 默认 `CMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/wsf_install`（第 210-212 行）。

### 3.3 版本与 Git 集成

版本字符串由 `AFSIM_VERSION_MAJOR/MINOR/PATCH`（70-73 行）与 Git 信息组合：若 `.git` 存在且分支名不匹配 `release_.*`，则追加提交日期作为 build 号；若处于 release 候选分支，后缀 `-rc`（第 84-120 行）。同时通过 `execute_process` 将 `.githooks` 目录设为 git hooks 路径，用于 clang-format pre-commit 钩子（第 114-119 行）。

### 3.4 文档与测试

- Doxygen：通过 `include(wsf_doxygen)` 和 `create_doxyfile()` 生成，可通过 `WSF_INSTALL_DOXYGEN` 选项安装（第 178-181、410-418 行）。
- Sphinx/RST：通过 `include(wsf_documentation)` 和 `create_docfile()` 生成用户文档，可通过 `WSF_INSTALL_DOCUMENTATION` 选项安装（第 183-187、420-428 行）。
- 单元测试：通过 `include(ConfigureUnitTests.cmake)` 集成 GoogleTest/GMock，目标命名遵循 `<app>_AUTO_TEST` / `<app>_REGRESSION_TEST`（第 360-362、228-232 行）。

### 3.5 打包与分发

通过 `include(wsf_package.cmake)` 调用 CPack，Windows 使用 WiX（`CPACK_WIX_COMPONENT_INSTALL ON`），Linux 生成带 GCC 版本的系统标签（如 `lnx64-gcc${GCC_MAJOR}`），并支持安装 MSVC 运行库（`WSF_INSTALL_DEPENDENCIES`）（`swdev/src/cmake/Modules/wsf_package.cmake`）。

### 3.6 构建预设体系

`CMakePresets.json` 定义了 configure/build preset 分层结构：基础 preset（`vs2019`、`ninja`、`clang`、`clang-linux`）被 `default-*` 组合 preset 继承，再映射到具体 Debug/Release 构建 preset。`swdev/src/cmake/Presets/*.cmake` 通过 `ENABLE_EXTENSION_DEFAULT=FALSE` 配合 `set(BUILD_WITH_xxx TRUE)` 精确控制哪些 WSF 扩展参与构建，`base.cmake` 仅启用 mission/wsf_grammar_check/wsf_parser 最小集。

## 4. 约定与约束

- **C++ 标准锁定**：`CMAKE_CXX_STANDARD 11`，禁止扩展（`CMAKE_CXX_EXTENSIONS OFF`）。
- **32 位弃用警告**：检测到 32 位构建时打印弃用警告（第 28-30 行）。
- **共享 vs 静态**：默认 `WSF_PLUGIN_BUILD=TRUE` 启用共享库/DLL 模式；关闭后 MSVC 强制 `/MT`，Linux 使用 `-static-libgcc`。
- **Unity Build**：开启 `CMAKE_UNITY_BUILD` 时 MSVC 自动添加 `/bigobj`，batch size 默认为 0（第 260-270 行）。
- **编译器加速**：MSVC 启用并行编译 `/MP`；Linux 可选项 `AFSIM_USE_CCACHE` 启用 ccache（第 51、316-323 行）。
- **ABI 兼容**：Linux 下强制 `_GLIBCXX_USE_CXX11_ABI=0` 以兼容预编译的 GCC 4.8 第三方库（第 329-332 行）。
- **插件安装路径一致性**：注释明确说明 `INSTALL_PLUGIN_PATH` 与 `INSTALL_EXE_PATH` 不一致会导致运行时插件加载失败（第 221-223 行）。
- **扩展发现约定**：扩展必须包含 `wsf_cmake_extension.cmake` 并在 `core/wsf/optional_projects.cmake` 中声明，否则不会被纳入构建。
- **Git Hooks**：构建时会把 `AFSIM_GIT_HOOKS_DIR`（默认 `.githooks`）设为 `core.hooksPath`，用于执行 clang-format 等预提交检查（第 114-119 行）。

## 5. 未观察到的内容

仓库中未发现 Dockerfile、docker-compose.yml、GitHub Actions、GitLab CI、Jenkinsfile、Azure Pipelines 等 CI/CD 配置文件；也未发现独立的 `build.sh`/`configure.sh` 脚本。构建完全依赖 CMake 及其 Presets 机制。