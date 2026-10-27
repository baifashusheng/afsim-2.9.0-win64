---
kind: logging_system
name: AFSIM 日志系统：基于 ut::log 的发布-订阅式结构化日志框架
category: logging_system
scope:
    - '**'
source_files:
    - swdev/src/tools/util/source/UtLog.hpp
    - swdev/src/tools/util/source/UtLogMessage.hpp
    - swdev/src/tools/util/source/UtLogStream.hpp
    - swdev/src/tools/util/source/UtLogPublisher.hpp
    - swdev/src/tools/util/source/UtLogSubscriber.hpp
    - swdev/src/tools/util/source/UtLogBatch.hpp
    - swdev/src/core/wsf/source/WsfConsole.hpp
    - swdev/src/core/wsf/source/WsfSystemLog.cpp
---

## 1. 使用的系统与架构

AFSIM 在 `swdev/src/tools/util/source/` 下实现了自研的 `ut::log` 日志框架，并在 `swdev/src/core/wsf/source/WsfConsole.hpp` 中提供控制台与文件两个 Sink（输出端），由 `WsfSystemLog.cpp` 提供传统 `.log` 文件兼容层。整体采用 **Publisher-Subscriber** 模式：

- 发布者：`UtLogPublisher`（位于 `tools/util/source`）
- 消息类型：`UtLogMessage` + `UtLogStream`（流式构造，支持 `AddNote()` 追加附加字段）
- 订阅者：`UtLogSubscriber` / `UtLogSubscriberT`，WSF 侧通过 `wsf::console::SubscriberBase` 派生 `ConsoleSubscriber`（stdout）和 `FileSubscriber`（文件）两类 sink。
- 线程安全：`UtLogBatch` 提供批量、不可交错的消息创建；`WsfSystemLog` 内部用 `std::mutex` 保护多文件写入。

该框架不是第三方库，而是仓库内 `tools/util` 共享工具库的一部分，被 WSF 核心、各领域扩展（wsf_mil、wsf_space 等）以及 GUI/CUI 应用统一引用。

## 2. 关键文件

- `swdev/src/tools/util/source/UtLog.hpp` — 公共 API 入口，声明 `ut::log::fatal/error/warning/debug/info/developer/write/newline` 工厂函数及 `DataMessageHelper`。
- `swdev/src/tools/util/source/UtLogMessage.hpp` — 消息数据结构。
- `swdev/src/tools/util/source/UtLogStream.hpp` — RAII 流式消息构造器（`operator<<` 风格）。
- `swdev/src/tools/util/source/UtLogPublisher.hpp` — 全局发布者。
- `swdev/src/tools/util/source/UtLogSubscriber.hpp` / `UtLogSubscriberT.hpp` — 订阅者基类。
- `swdev/src/tools/util/source/UtLogBatch.hpp` — 线程安全的批量消息构造。
- `swdev/src/core/wsf/source/WsfConsole.hpp` — WSF 对 `ut::log` 的集成：`OutputSettings`、`SubscriberBase`、`ConsoleSubscriber`、`FileSubscriber`、`ScenarioExtension`（解析 `console_output` 配置块）。
- `swdev/src/core/wsf/source/WsfSystemLog.cpp` — 遗留的 `.log` 文件写入器，默认生成 `<exe>.log`，支持 `log_file` 命令动态切换目标文件，并写版本信息。

## 3. 架构与约定

### 3.1 日志级别（TypeList）
`UtLog.hpp` 暴露六个预定义级别：`fatal`、`error`、`warning`、`debug`、`info`、`developer`。调用方通过 `ut::log::<level>()` 获取一个 `MessageStream`，再链式 `<<` 拼接内容；复杂消息可先赋值给 `auto out = ...`，在 RAII 作用域结束时自动发送，或使用 `out.AddNote() << "..."` 追加带标签的附注行。

头文件中注释明确约定：
- 使用 `operator<<` 而非 printf 风格；
- 跨作用域构造消息时需用 `{}` 包裹以保证 RAII 顺序；
- 需要传给 `std::ostream&` 的地方用 `MessageStream::GetStream()`；
- 自定义消息类型需通过 stream factory 传入 `TypeList` 并订阅对应类型。

### 3.2 输出路由
- `ConsoleSubscriber`：将格式化后的消息写入 stdout，默认启用所有类型，可通过 `console_output` 配置块选择子集（`mTypes`）。
- `FileSubscriber`：将消息写入磁盘文件，支持 `OpenFile()` 打开新文件。
- `WsfSystemLog`：独立于 `ut::log` 的传统路径，默认在 exe 目录创建 `<exe>.log`，并通过 `WriteLogEntry` 以 `YYYY-MM-DD HH:MM:SS <entry>` 格式写入；同时支持通过 `ProcessInput("log_file", ...)` 运行时切换目标文件，并记录 `version wsf <ver>` 与来源文件名。

### 3.3 线程模型
`WsfSystemLog::WriteLogEntry` 使用 `std::lock_guard<std::mutex>` 保护双文件（默认 log + 用户指定 log）写入；`UtLogBatch` 文档说明其用于“在多线程环境中创建不可交错的消息集合”。

### 3.4 与事件系统的关系
`WsfEventPipeInterface` 维护一组 `logger` 订阅者，并在选项变化时调用 `UpdateSubscriptions`，使事件管道能消费结构化日志消息。

## 4. 约定与约束

- **API 约定**：所有日志输出必须通过 `ut::log::*()` 工厂函数，禁止直接操作 `UtLogPublisher`（见 `UtLog.hpp` 注释中的用法示例）。
- **RAII 语义**：复杂消息应置于 `{}` 作用域内，让 `MessageStream` 析构负责发送，避免多条消息乱序（`UtLog.hpp` 注释明确要求）。
- **级别语义**：`info` 及以上级别控制 console 是否启用 format 输出（`OutputSettings.mSingleLine` 注释：“Format messages are enabled if Info messages are enabled”）。
- **刷新策略**：`OutputSettings.mForceFlushing` 默认 `true`，表示 fatal/error/warning 级别会强制 flush（`SubscriberBase` 注释说明）。
- **默认行为**：`WsfSystemLog` 构造函数默认创建 `<exe>.log`；可通过 `DisableDefaultLogFile()` 关闭（注释强调“Must be called prior to WsfApplication creation”）。
- **输入命令**：`WsfSystemLog::ProcessInput` 仅识别 `log` 与 `log_file` 两个命令，其他命令返回 `false`，这是实现层面强约束。
- **时间戳格式**：`WriteLogEntry` 固定使用 `strftime(timeStr, sizeof(timeStr), "%Y-%m-%d %H:%M:%S", localTimePtr)`，所有传统日志行前缀一致。

## 5. 观察到的使用模式

在 `sensor_plot_lib`、`wsf_mtt`、`wsf_mil` 等模块中，常见写法为：
```cpp
auto out = ut::log::error() << "Unable to open output file.";
// 或
ut::log::info() << "Writing output.";
```
错误场景多用 `ut::log::error()`，信息性流程点用 `ut::log::info()`，警告用 `ut::log::warning()`，未见直接使用 `fatal` 或 `developer` 的广泛模式（`developer` 可能面向调试构建）。