---
kind: dependency_management
name: AFSIM 依赖管理：CMake 驱动的预构建第三方库与 Python 包锁定
category: dependency_management
scope:
    - '**'
source_files:
    - swdev/src/tools/3rd_party-cmake/config.cmake
    - swdev/src/tools/3rd_party-cmake/shared.cmake
    - swdev/src/tools/3rd_party-cmake/qt-5.12.11.cmake
    - swdev/src/tools/3rd_party-cmake/gdal-3.3.2.cmake
    - swdev/src/tools/3rd_party-cmake/gtest-1.8.0.cmake
    - swdev/src/tools/3rd_party-cmake/osg-3.6.3.cmake
    - swdev/src/tools/3rd_party-cmake/proj-8.1.1.cmake
    - swdev/src/tools/3rd_party-cmake/sqlite-3.32.3.cmake
    - swdev/src/tools/3rd_party-cmake/tiff-4.3.0.cmake
    - swdev/src/tools/3rd_party-cmake/tinyxml2-7.1.0.cmake
    - swdev/src/tools/3rd_party-cmake/sdl-2.0.16.cmake
    - swdev/src/tools/3rd_party-cmake/geos-3.5.1.cmake
    - swdev/src/tools/3rd_party-cmake/osgEarth-2.10.1.cmake
    - swdev/src/requirements.txt
    - tools/pyrunplotter/requirements.txt
---

## 1. 使用的系统/方法

AFSIM 仓库采用 **两层依赖管理**：
- C++ 第三方库通过自研的 `swdev/src/tools/3rd_party-cmake/` 工具链，基于 CMake 宏（`swdev_acquire_packages`、`swdev_package_include_directories`、`swdev_package_link_libraries`）从预构建的二进制归档中获取并安装；
- Python 脚本依赖通过各子目录下的 `requirements.txt` 使用 pip 风格版本锁定。

仓库没有使用 npm/yarn/pipenv/conda 等现代包管理器作为主依赖入口，也没有 vendoring 源码——所有第三方 C/C++ 源码以**预编译二进制归档**形式提供。

## 2. 关键文件与目录

- `swdev/src/tools/3rd_party-cmake/config.cmake`：核心逻辑。定义 `SWDEV_THIRD_PARTY_ROOT`、搜索路径、包下载/解压/校验流程。
- `swdev/src/tools/3rd_party-cmake/shared.cmake`：平台/架构检测（MSVC vs GCC、x86/x64、Linux distro 后缀）、DLL 复制与 install 规则。
- `swdev/src/tools/3rd_party-cmake/<pkg>-<version>.cmake`：每个第三方库一个版本化配置文件（如 `qt-5.12.11.cmake`、`gdal-3.3.2.cmake`、`gtest-1.8.0.cmake`、`osg-3.6.3.cmake`、`proj-8.1.1.cmake`、`sqlite-3.32.3.cmake`、`tiff-4.3.0.cmake`、`tinyxml2-7.1.0.cmake`、`sdl-2.0.16.cmake`、`geos-3.5.1.cmake`），声明 `<PKG>_ROOT_DIR`、`<PKG>_INCLUDE_DIR`、`<PKG>_LIBRARY` 等变量供上层链接。
- `swdev/dependencies/3rd_party/`：存放预构建第三方包的根目录（当前快照为空，实际包由构建时从网络或共享存储拉取到 `SWDEV_THIRD_PARTY_ROOT`）。
- `swdev/dependencies/resources/`：地图数据（bald_earth_db、bluemarble_db、naturalearth_db、political_db 等）与 3D 模型资源，随产品分发。
- `swdev/src/requirements.txt`：Sphinx==4.4.0, myst-parser==0.16.1（文档生成）。
- `tools/pyrunplotter/requirements.txt`：matplotlib==3.4.3（绘图工具）。

## 3. 架构与约定

### 3.1 预构建包查找顺序
`config.cmake` 按以下优先级确定 `SWDEV_THIRD_PARTY_ROOT`：
1. 显式传入的 `SWDEV_THIRD_PARTY_ROOT`（CACHE PATH）；
2. 环境变量 `SWDEV_THIRD_PARTY_PATH`；
3. 相对路径 `../3rd_party` / `../../3rd_party` / `../../../3rd_party`；
4. 回退到 `${CMAKE_BINARY_DIR}/3rd_party`（每构建隔离）。

若找不到已存在的 root，则报错 `FATAL_ERROR`。

### 3.2 包源搜索路径
默认 `SWDEV_THIRD_PARTY_SEARCHPATH` 包含：
- Windows: `//sllabl1/data/3rd_party/${SWDEV_BASE_BRANCH}`、`//homer/l1r/data/3rd_party/${SWDEV_BASE_BRANCH}`；
- Linux: `/data/3rd_party/${SWDEV_BASE_BRANCH}`；
- 仓库内: `${afsim_SOURCE_DIR}/../dependencies/3rd_party`。

包名格式为 `<name>-<version>-<arch>-<sys>`，例如 `qt-5.12.11-x64-vs2017.tar.gz`，支持 `.tar.gz`、`.tar`、`.zip` 三种扩展名。

### 3.3 版本化与更新策略
- 每个第三方库在 `3rd_party-cmake/` 下有一个 `<pkg>-<version>.cmake` 文件，版本号硬编码在文件名中，升级需新增文件并调用方切换引用。
- 构建时比较本地 `sha512` 摘要与远程包对应的 `.sha512` 文件，不一致则删除旧目录并重新 `tar xzf` 解压。
- 已存在于 `SWDEV_THIRD_PARTY_ROOT` 中的包被视为“不可更新”（`packagesNotUpdatable` 计数），不会检查远端差异。

### 3.4 链接与安装约定
- 每个 `<pkg>-<version>.cmake` 暴露 `<PKG>_ROOT_DIR`、`<PKG>_INCLUDE_DIR`、`<PKG>_LIBRARY` 等变量；
- 上层通过 `swdev_package_include_directories(<target> <pkg>)` 和 `swdev_package_link_libraries(<target> <pkg>)` 统一注入 include 与 link；
- Windows 下 DLL 通过 `copy_third_party_dlls_to_bin` 复制到 `CMAKE_RUNTIME_OUTPUT_DIRECTORY`，install 阶段通过 `install_third_party` 将 dll/so 与 license 文本分别安装到 `INSTALL_DLL_PATH` 与 `resources/data/3rd_party_licenses`。

### 3.5 Python 依赖
Python 依赖仅出现在两个位置，且全部使用 `==` 精确版本锁定：
- `swdev/src/requirements.txt`：Sphinx 文档生成；
- `tools/pyrunplotter/requirements.txt`：matplotlib 绘图。

## 4. 观察到的约定与约束

- **约定**：所有 C++ 第三方库以预构建二进制归档形式提供，不 vendoring 源码；源码目录 `swdev/dependencies/3rd_party/<pkg>/` 存在但为空占位。
- **约定**：每个第三方库在 `swdev/src/tools/3rd_party-cmake/` 下维护一个 `<pkg>-<version>.cmake` 配置文件，版本号嵌入文件名。
- **约定**：包名遵循 `<name>-<version>-<arch>-<sys>` 命名规范，其中 arch 为 `x86`/`x64`，sys 在 Windows 上为 `vs2017`/`vs2015` 等 MSVC 版本后缀，Linux 上为 `-rh6`/`-fc6`/`-lnx`。
- **约束**：`config.cmake` 在找不到 `SWDEV_THIRD_PARTY_ROOT` 时直接 `message(FATAL_ERROR ...)`，即构建失败而非静默跳过。
- **约束**：Windows 下 `shared.cmake` 根据 `MSVC_VERSION` 映射到 `SWDEV_SYS_SUFFIX`（vs2010..vs2017），未知版本发出 WARNING。
- **约束**：Python 依赖使用 `requirements.txt` + `==` 精确版本锁定，无 `pip freeze` 生成的锁文件或私有 PyPI 索引配置。
- **约束**：包更新仅在本地不存在对应 `.sha512` 或与远端不一致时触发；若包已在 `SWDEV_THIRD_PARTY_ROOT` 中，则视为不可更新。
- **约束**：Qt 依赖通过 `find_package(Qt5Core REQUIRED)` 发现，并在 `qt_project.cmake` 中强制要求 Qt5 模块（Core/Gui/Network/OpenGL/Widgets 等）。