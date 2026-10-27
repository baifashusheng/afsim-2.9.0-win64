---
kind: error_handling
name: AFSIM/WSF 异常与错误处理体系
category: error_handling
scope:
    - '**'
source_files:
    - swdev/src/tools/util/source/UtException.hpp
    - swdev/src/tools/util/source/UtInput.hpp
    - swdev/src/core/wsf/source/WsfException.hpp
    - swdev/src/core/wsf/source/WsfApplication.hpp
    - swdev/src/core/wsf/source/WsfStandardApplication.cpp
    - swdev/src/core/wsf/source/WsfScenario.cpp
    - swdev/src/core/wsf/source/WsfAttributeContainer.cpp
---

## 1. 整体方案

AFSIM 的 C++ 代码库采用 **C++ 异常机制**（`std::exception` 派生类）作为统一的错误传播方式，没有使用返回值错误码、`assert` 崩溃或 `panic/recover`。核心由两层组成：

- **基础层**：`swdev/src/tools/util/source/UtException.hpp` 提供 `UtException`（继承自 `std::exception`）、`FatalException`（不继承任何基类，用于致命错误）、`HardwareException`（硬件异常映射）、`ut::ReportException` / `ut::SetupThreadErrorHandling` / `ut::SetApplicationErrorHandling` 等应用级终止与日志 API。
- **框架层**：`swdev/src/core/wsf/source/WsfException.hpp` 定义 `WsfException : public UtException` 以及 `WsfUncloneableException`；`WsfApplication::Exception` 和 `FeatureNotPresent` 进一步扩展 WSF 应用语义。
- **输入解析层**：`swdev/src/tools/util/source/UtInput.hpp` 在 `UtInput` 内部定义 `ExceptionBase` 及其派生类 `BadValue` / `EndOfData` / `OutOfContext` / `UnknownCommand` / `UnknownUnits`，所有输入解析器统一通过抛出这些异常来报告用户输入错误。

WSF 各子模块（`wsf_mil`、`wsf_space`、`wsf_nx`、`wsf_ripr` 等）遵循同一模式：业务异常继承 `WsfException`，输入相关异常使用 `UtInput::*`。

## 2. 关键文件与包

| 路径 | 职责 |
|---|---|
| `swdev/src/tools/util/source/UtException.hpp` | 根异常 `UtException`、致命异常 `FatalException`、硬件异常 `HardwareException`、线程/进程级错误处理入口 |
| `swdev/src/tools/util/source/UtInput.hpp` | 输入解析异常族 `UtInput::ExceptionBase` / `BadValue` / `EndOfData` / `OutOfContext` / `UnknownCommand` / `UnknownUnits` |
| `swdev/src/core/wsf/source/WsfException.hpp` | WSF 框架异常 `WsfException`、`WsfUncloneableException` |
| `swdev/src/core/wsf/source/WsfApplication.hpp` | WSF 应用异常 `WsfApplication::Exception`、`FeatureNotPresent` |
| `swdev/src/core/wsf/source/WsfStandardApplication.cpp` | 标准应用入口，捕获 `WsfApplication::Exception`、`WsfException` 并输出结果 |
| `swdev/src/core/wsf/source/WsfScenario.cpp` | 场景加载时 `catch (...)` 兜底 |
| `swdev/src/core/wsf/source/WsfAttributeContainer.cpp` | 典型用法：捕获 `UtInput::ExceptionBase` 后包装为 `UtInput::BadValue` 重新抛出 |

## 3. 架构与约定

### 3.1 异常层次
```
std::exception
└── UtException (带 stacktrace)
    ├── WsfException (WSF 框架层)
    │   └── WsfUncloneableException
    ├── WsfApplication::Exception (含返回码)
    │   └── FeatureNotPresent
    └── UtInput::ExceptionBase (输入层)
        ├── BadValue
        ├── EndOfData
        ├── OutOfContext
        ├── UnknownCommand
        └── UnknownUnits
```

`FatalException` 是独立类型（不继承 `UtException`），专门用于不可恢复的致命错误，顶层 catch 可区分它与普通异常。

### 3.2 硬件异常提升
`UtException.hpp` 中 `PromoteHardwareExceptions(bool)` 在启用 CMake 选项 `PROMOTE_HARDWARE_EXCEPTIONS` 时将 Windows SEH 信号 / Linux 信号转换为 `ut::HardwareException`，使除零、空指针解引用等也能被 `catch`。

### 3.3 应用级错误处理入口
- `ut::SetApplicationErrorHandling(terminate_handler, stack_size)`：一次性设置全局 terminate handler。
- `ut::SetupThreadErrorHandling(stack_size)`：每个新线程调用，设置该线程的 terminate handler 与硬件异常策略。
- `ut::ReportException(source, std::exception_ptr, additional_info)`：顶层 `catch(...)` 中统一调用，记录到应用 crash log。
- `WsfApplication` 构造时调用 `ut::SetupThreadErrorHandling()`（见 `WsfApplication.cpp:51`）。

### 3.4 输入解析错误模式
输入解析器（如 `WsfAdvancedBehaviorTree.cpp`、`WsfAntennaPattern.cpp`、`WsfBehaviorTree.cpp`、`WsfAttributeContainer.cpp`）普遍使用 `throw UtInput::BadValue(aInput, msg)` 或 `throw UtInput::UnknownCommand(aInput)`。上层（如 `WsfAttributeContainer.cpp`）捕获 `UtInput::ExceptionBase` 后将其包装为更具体的 `BadValue` 再抛出，保持位置信息一致。

### 3.5 顶层捕获模式
`WsfStandardApplication.cpp` 中的 `RunEventLoop` / `ProcessCommandLineCommands` 等入口函数以 `catch (WsfApplication::Exception&)`、`catch (WsfException& e)`、`catch (...)` 分层捕获，将异常转换为 `SimulationResult` 或退出码，而不是让异常继续向上传播。

## 4. 约定与约束

- **业务异常必须继承 `UtException` 或其派生类**：`WsfException`、`WsfApplication::Exception` 均如此，确保 `what()` 与 stacktrace 可用。
- **输入解析错误统一走 `UtInput::*` 异常族**：所有解析器抛出的都是 `UtInput::BadValue` / `UnknownCommand` / `EndOfData` / `OutOfContext` / `UnknownUnits`，调用方按 `UtInput::ExceptionBase` 捕获。
- **致命错误使用 `ut::FatalException`**：文档注释明确说明“should only be caught at application scope”，不应被普通业务逻辑捕获。
- **新线程需调用 `ut::SetupThreadErrorHandling()`**：头文件注释要求“MUST BE CALLED ON EVERY THREAD THAT USES MICROSOFT STRUCTURED EXCEPTIONS”。
- **顶层应使用 `ut::ReportException` 兜底**：头文件中给出示例 `catch(...) { ut::ReportException("AFSIM", std::current_exception()); }`。
- **WSF 应用入口统一捕获 `WsfApplication::Exception` 与 `WsfException`**：`WsfStandardApplication.cpp` 展示了这一模式，其他应用（sensor_plot、mover_creator、wizard 等）也遵循。
- **未观察到 panic/recover 或返回值错误码的错误处理风格**：整个仓库以 C++ 异常为主流，未见 `errno`、`HRESULT` 或自定义错误枚举作为主要错误传播手段。