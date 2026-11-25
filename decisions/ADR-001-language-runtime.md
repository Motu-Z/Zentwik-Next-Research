# ADR-001：核心语言与运行时

- 状态：Accepted（暂定，可替换边界见"验证与回滚"）
- 日期：2026-11-25
- 相关：`research/04-tech-stack-evaluation.md`

## 背景

新产品允许彻底重选技术栈。候选：C#/.NET 10、Rust、C++。真实风险不在运行时性能（数据面 4–10 Hz），而在硬件写入安全、状态正确性和单人长期维护成本。

## 决策

服务、领域逻辑、IPC 契约、UI 全部使用 C# / .NET 10。UI 进程 JIT + ReadyToRun，特权服务用 NativeAOT。

## 候选与否决理由

- **C++**：唯一独占优势是可直接写内核驱动。签名版 PawnIO 只加载维护者签名的 `.bin`，自造驱动不可分发，该优势在本项目不存在；剩余成本是最高缺陷率与最慢迭代。
- **Rust**：内存安全与体积优势真实，但落在一条很窄的 IOCTL 封送层上；代价是新工具链（本机未安装）、双语言边界、Win32 安全包装样板。收益与成本不匹配。
- **C#**：Win32/COM 互操作与调试体验最好，生态中已有 LibreHardwareMonitor、SMUDebugTool 等同类参考；本机已验证可构建并跑通 39 个测试。

## 后果

正面：单一语言贯通、调试链路完整、既有领域知识可迁移。
负面：GC 与 JIT 带来启动和内存开销；UI 不能 AOT；`unsafe` 面需人为收窄。

## 可替换边界

- 硬件访问全部经 `Zentwik.Platform` 接口（`ITelemetrySource`、`IPowerPolicyService`、`ICapabilityProbe`），Win32 细节不外泄。
- 若安全评审要求内存安全隔离，可用 Rust 重写该层并保持接口不变，其余部分不动。
- 推翻条件：出现必须在驱动层完成、签名模块无法覆盖的能力，或 .NET 桌面运行时停止支持 Windows。
