---
kind: configuration_system
name: AFSIM 配置系统：基于 WSF DSL 的声明式场景脚本
category: configuration_system
scope:
    - '**'
source_files:
    - swdev/src/core/wsf/grammar/wsf.ag
    - swdev/src/core/wsf/source/WsfApplication.hpp
    - swdev/src/core/wsf/source/WsfScenario.cpp
    - swdev/src/core/wsf/source/WsfSimulation.cpp
    - swdev/src/core/wsf/source/WsfComponentFactory.cpp
    - swdev/src/core/wsf/source/script/WsfScriptPlatformClass.cpp
    - demos/air_to_air/1v1.txt
    - demos/six_dof/std_setup.txt
    - demos/iads/setup.txt
---

## 1. 采用的系统与框架

AFSIM 不使用 YAML/TOML/JSON/env 等通用配置文件格式，而是使用**自研的 WSF（Workspace Simulation Framework）领域特定语言（DSL）**作为唯一的运行时配置载体。WSF 配置由位于 `swdev/src/core/wsf/grammar/wsf.ag` 的 ANTLR-like 语法文件定义，通过自定义解析器将 `.txt` 场景脚本解析为 WSF 对象模型（Platform、Sensor、Weapon、Processor、Scenario 等）。因此，“配置”在 AFSIM 中就是“可执行的仿真脚本”，而非静态键值对。

构建期与运行期的边界如下：
- **构建期**：CMake (`swdev/src/CMakeLists.txt`, `swdev/src/core/wsf/CMakeLists.txt`) 负责编译 C++ 源码、生成版本信息、打包依赖；不涉及应用配置加载。
- **运行期**：`WsfApplication` / `WsfStandardApplication` 启动后，通过 `WsfScenario` + `WsfSimulation` 读取用户提供的 `.txt` 场景文件，逐条命令调用 WSF 组件工厂注册实体类型并实例化。

## 2. 关键文件与包

- `swdev/src/core/wsf/grammar/wsf.ag` — WSF DSL 语法定义，包含 value/struct/enumeration/function/rule 等构造，是配置语法的权威来源。
- `swdev/src/core/wsf/source/WsfApplication.hpp` — 应用入口，提供 Feature 注册机制（`os:linux|windows|apple`、`cpu:intel`、`addr:32-bit|64-bit`、`build:debug|release`），供条件编译/条件加载。
- `swdev/src/core/wsf/source/WsfScenario.cpp` / `WsfSimulation.cpp` — 场景与仿真生命周期，驱动配置文件的顺序解析。
- `swdev/src/core/wsf/source/WsfComponentFactory*.cpp` — 组件工厂注册表，所有平台/传感器/武器/处理器类型通过工厂以字符串名注册，供配置脚本按名称引用。
- `swdev/src/core/wsf/source/script/*.cpp` — 脚本执行引擎，暴露给配置脚本的变量、函数、运算符（如 `->` 右箭头操作符）。
- `demos/*/` 下的 `.txt` 场景文件 — 实际的配置数据（例如 `demos/air_to_air/1v1.txt`、`demos/six_dof/std_setup.txt`、`demos/iads/setup.txt`）。
- `resources/data/` — 内置的预定义类型库（平台、传感器、武器、签名等），被场景脚本通过 `include` 或相对路径引用。

## 3. 架构与设计约定

### 3.1 配置即脚本（Script-as-Configuration）
WSF 配置不是纯数据文件，而是可执行脚本：每条语句对应一个 WSF 命令（如 `platform`, `sensor`, `weapon`, `processor`, `scenario`, `observer` 等），解析时直接调用对应的 C++ 构造函数或 setter。这使得配置可以包含逻辑控制（循环、条件分支）、函数调用和变量赋值。

### 3.2 条件配置与特性开关
`WsfApplication::RegisterFeature()` 注册环境特性（OS、CPU、地址位宽、构建类型），配置脚本可通过 `conditionals` 块或 `test_feature` 命令进行条件加载。这是 AFSIM 实现跨平台/跨构建变体配置的核心机制。

### 3.3 扩展点与插件化
- `WsfApplicationExtension` 允许扩展注册新的 WSF 命令、脚本类型、事件输出等。
- `WSF_REGISTER_EXTENSION(APP, NAME)` 宏用于在应用启动时自动发现并注册扩展。
- 插件通过 `WsfPluginManager` 动态加载，支持运行时扩展仿真能力。

### 3.4 配置分层与继承
WSF 语法支持 struct 嵌套、map/list 聚合、变量作用域（`push/pop`），以及 `include` 指令引入外部配置片段。典型模式是：
- `setup.txt` 定义全局常量、环境变量、公共类型
- 场景脚本 `include setup.txt` 后定义具体平台、传感器、武器、规则
- 子目录 `platforms/`、`sensors/`、`weapons/` 存放可复用类型定义

### 3.5 输入验证与错误处理
解析失败时抛出 `UtException` 派生异常，并通过 `ut::log` 输出结构化日志。每个 WSF 命令在解析阶段进行类型检查（value 类型匹配 struct 字段类型）。

## 4. 约定与约束

### 观察到的约定（描述性）
- 场景文件统一使用 `.txt` 后缀，遵循 WSF DSL 语法。
- 可复用类型放在 `platforms/`、`sensors/`、`weapons/`、`processors/`、`scenarios/`、`rules/` 等子目录中，通过相对路径 include。
- 全局设置集中在 `setup.txt`，场景脚本通过 `include setup.txt` 引用。
- 演示场景通常附带 `README.md` 说明运行方式与输出文件位置。
- Feature 命名采用 `namespace:value` 形式（如 `os:linux`、`build:debug`）。

### 明确约束（有代码/文档依据）
- **唯一应用实例**：`WsfApplication` 头注释明确要求“One instance of this object may exist at any given time.”（单例约束）。
- **Feature 范围**：标准 feature 仅包括 OS（`os:linux|windows|apple`）、CPU（`cpu:intel`）、地址位宽（`addr:32-bit|64-bit`）、构建类型（`build:debug|release`），扩展可追加自定义 feature。
- **DIS configuration state 范围**：`WsfScriptPlatformClass.cpp` 中对 `configurationState` 校验要求 `0 ≤ state ≤ 15`，超出范围记录警告。
- **WSF 语法版本**：`wsf.ag` 顶部声明 `(grammar-version 2.2.0)`，解析器据此选择语法行为。
- **条件块语义**：`WsfApplication` 文档注释指出配置文件中可使用 `conditionals` 块或 `test_feature` 命令测试 feature 存在性。

## 5. 与其他配置系统的关系

本仓库不存在以下常见配置机制：无 `.env`、无 `application.properties`、无 YAML/TOML 配置加载器、无环境变量注入层（除命令行参数外）。所有运行时配置均通过 WSF DSL 脚本完成，这是 AFSIM 作为领域仿真引擎的设计选择——将配置与仿真逻辑深度融合，便于表达复杂的战术行为、传感器参数、武器模型等高度结构化的领域知识。

对于需要外部参数注入的场景（如 CI 批量运行），工具链通过 Python 脚本（`tools/batch_run.pl`、`tools/pyrunplotter/`）替换模板中的占位符，而非通过运行时配置系统加载。
