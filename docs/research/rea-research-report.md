# REA 项目调研报告

- 调研日期：2026-10-30
- 上游仓库：[`morluto/rea`](https://github.com/morluto/rea)
- 代码基线：`main`，commit `317bf3fe235e655b2d752d3be261cc0da7d00f2b`
- 项目版本：`rea-agents` 6.8.0
- 许可证：MIT（第三方组件和商业分析工具另受其各自许可约束）

## 一、结论摘要

REA（Reverse Engineer Anything）是面向 AI 编程智能体的本地逆向分析工具集。它通过一个 Node.js CLI 和 MCP 服务，把多种文件格式解析器、浏览器/设备观测能力以及 Hopper、Ghidra、IDA、Kuna 等外部分析引擎统一成可调用的工具。分析结果带有结构化证据、来源和限制，供接入的智能体继续追问、解释或辅助实现。

它不是一个独立的大模型，也不是单一反编译器：需要用户配置 Codex、Claude Code 等 MCP 客户端；原生二进制深度反编译通常还需要另行安装并授权分析引擎。多种格式的静态解析则由 REA 自身实现，可不依赖这些引擎。

## 二、它能做什么

1. **原生程序分析**：对目标文件做函数、伪代码/汇编、字符串、符号、调用和引用等分析；深度分析由已配置的 Hopper、Ghidra、IDA 或 Kuna 提供。还包括 objdump、Rizin、GDB 以及 ELF/core 文件布局等适配能力。
2. **应用与 Web 分析**：静态检查 JavaScript/Electron 目录或 ASAR，重建模块和导入关系，检查路由、source map、IPC 与原生模块边界；通过浏览器调试协议/Playwright 观察页面、脚本、运行时覆盖率和网络行为。
3. **托管代码与移动端**：静态读取 .NET 元数据和 CIL；分析 Android APK 清单、类和方法，并通过调用方提供的 ADB 连接设备进行包、进程、日志、截图或文件操作。Apktool、JADX、Frida、JEB 等能力依目标与环境而定。
4. **多种二进制/数据格式**：内置或集成解析涵盖 Go/Rust 二进制、PE/ELF 元数据、WASM、EVM、GPU 代码对象、GGUF/safetensors、Unity IL2CPP、Godot PCK、Erlang BEAM、SQLite、PCAP、崩溃记录和固件等。每种格式的支持范围和限制不同，应以对应指南为准。
5. **调查编排与证据留存**：可在同一会话中保存分析记录、比较结果、来源和未知项；智能体可以基于已留存证据继续查询，而不是只拿到一段无来源的文本。

以上是代码和项目文档声明的能力清单，不代表本机已经具备对应的引擎、设备或工具链，也不代表对任意文件都能成功分析。

## 三、实现方式

### 3.1 运行入口

`scripts/rea.mjs` 是可执行分发入口，将命令路由到一次性 CLI、默认的 MCP stdio 服务，或显式启用的 Streamable HTTP 服务。`src/cli.ts` 和 MCP server 复用 `src/application/` 中的工作流；CLI 不是另一套独立分析实现。

### 3.2 分层结构

- **contracts/domain**：定义输入输出契约、工具族、Evidence 和错误语义，以及不依赖具体引擎的核心规则。
- **providers/adapters**：把具体引擎、浏览器、文件格式、设备和外部命令包装成统一接口。Hopper 和 Ghidra 等还通过随包的 Python/Java bridge 与引擎交互。
- **application**：组织跨提供方的分析、调查、会话、比较和证据记录流程。
- **composition**：创建 provider、会话和生命周期，并将它们组装给 CLI/MCP 使用。原生二进制深度分析由 provider registry 选择；可选提供方按需加载。
- **CLI/MCP adapters**：把命令行参数或 MCP 请求转成共享应用层调用，再将结构化结果返回给调用方。

主要链路可以概括为：

```text
Codex / Claude Code 等智能体
        | MCP（默认 stdio，也可选 HTTP）
        v
REA server -> shared application workflows -> provider/session composition
        ^                                             |
        |                                             v
CLI one-shot --------------------------> 格式解析器、浏览器/设备、外部逆向引擎
        |
        +---- 结构化结果、Evidence、限制和错误
```

### 3.3 Evidence 与会话

Evidence 不只是最终回答。MCP 返回结构化结果和证据标识；会话层保留分析记录及其语义身份，支持后续读取、比较和导出。代码还区分 provider 原始输出、归一化结果、profile、未知项和限制。这使智能体可以引用分析事实，也让“未检查/无法确认”不必被误写成“没有”。

### 3.4 生命周期与工具注册

`createServer` 按工具族注册 MCP 工具，provider 则由 composition 创建。仓库包含进程监督、超时、取消、资源归属和清理/重试逻辑；这些设施尤其影响会启动外部程序、浏览器、分析引擎或访问设备的工作流。

## 四、安装与运行前提

- 项目要求 Node.js `22.23.4+`（22.x）、`24.22.1+`（24.x）或 `26.12.1+`，并使用 npm。安装智能体集成的推荐入口是 `npx rea-agents setup`；setup 会展示计划并要求批准，再写入所选客户端配置和工作流指引，并对已有配置进行备份。
- MCP 默认使用 stdio；Streamable HTTP 是可选传输。
- 纯静态解析和深度反编译的依赖不同。例如 .NET 静态检查不需要原生分析引擎；原生函数反编译需要 Hopper/Ghidra/IDA/Kuna 中受支持的一种。Android、浏览器、WASM、PCAP、固件等工作流还可能要求 JDK/JADX、浏览器、WABT、tshark、Binwalk/Unblob 或 ADB 等外部工具。
- 仓库在 Linux 上对部分进程所有权检查要求 PATH 中有 procps 兼容的 `ps`。不同目标类型、操作系统与引擎组合应按各自安装指南核验。

## 五、边界、安全与适用性

- 项目定位是“本地分析 + 智能体消费结果”。这不等于所有数据都不会离开本机：工具结果会交给所配置的智能体/模型提供商，需同时评估其数据政策。某些工作流会执行或观察程序、连接运行时/浏览器、捕获网络流量、操作设备，可能产生外部副作用。
- 应仅分析自己拥有或已获授权的程序、设备、数据和服务。项目免责声明明确排除未授权访问、绕过许可/DRM/反作弊、窃取凭证等用途。IDA、Hopper、JEB 等商业工具的许可由用户自行取得；REA 的 MIT 许可证不会授予这些工具的使用权。
- 反编译、启发式推断和 AI 辅助解释可能不完整或不准确。对于重要结论，应核对 Evidence、原始来源、覆盖范围和限制，并在真实目标/引擎上复验。
- 项目更适合需要让智能体调用多类逆向工具并保留证据的研究、安全分析和软件互操作场景；若只需要单一反编译器，直接使用对应工具可能更简单。

## 六、调研范围与验证说明

本报告以 2026-10-30 上游 `main` 的上述 commit 为基线，阅读了 README、包元数据、架构图、安装/测试指南，以及 CLI/MCP 入口、工具注册、provider composition 和 Evidence 相关源码。报告是源码与文档调研，不是对各类分析引擎的实机能力认证。

本机 Node.js 为 `v23.11.0`，不符合该版本的 `engines` 范围；因此本次没有运行 npm 构建或测试，也没有连接 Hopper、Ghidra、IDA、Android 设备或浏览器做端到端验证。仓库本身定义了 Vitest 源码、边界、MCP、验收等测试通道以及单独的真实工具链验证通道；其测试指南也明确指出，注入式 provider 测试不能代替真实 provider 验证。

## 七、主要一手资料

- [上游 README：能力矩阵、快速开始和限制](https://github.com/morluto/rea/blob/317bf3fe235e655b2d752d3be261cc0da7d00f2b/README.md)
- [package.json：版本、Node.js 范围、入口和依赖](https://github.com/morluto/rea/blob/317bf3fe235e655b2d752d3be261cc0da7d00f2b/package.json)
- [架构图](https://github.com/morluto/rea/blob/317bf3fe235e655b2d752d3be261cc0da7d00f2b/docs/architecture.mermaid)
- [MCP runtime](https://github.com/morluto/rea/blob/317bf3fe235e655b2d752d3be261cc0da7d00f2b/src/main.ts) 与 [CLI](https://github.com/morluto/rea/blob/317bf3fe235e655b2d752d3be261cc0da7d00f2b/src/cli.ts)
- [二进制分析 provider composition](https://github.com/morluto/rea/blob/317bf3fe235e655b2d752d3be261cc0da7d00f2b/src/composition/binary.ts) 与 [MCP 工具注册](https://github.com/morluto/rea/blob/317bf3fe235e655b2d752d3be261cc0da7d00f2b/src/server/createServer.ts)
- [EvidenceLedger](https://github.com/morluto/rea/blob/317bf3fe235e655b2d752d3be261cc0da7d00f2b/src/application/investigation/EvidenceLedger.ts)、[安装指南](https://github.com/morluto/rea/blob/317bf3fe235e655b2d752d3be261cc0da7d00f2b/docs/installation.md)、[测试指南](https://github.com/morluto/rea/blob/317bf3fe235e655b2d752d3be261cc0da7d00f2b/docs/testing.md) 和 [免责声明](https://github.com/morluto/rea/blob/317bf3fe235e655b2d752d3be261cc0da7d00f2b/DISCLAIMER.md)
