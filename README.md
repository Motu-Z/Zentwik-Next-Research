# Zentwik Next Research

Zentwik Next 的产品重定义、技术预研、架构决策与原型验证工作区。

## 项目关系

- **Zentwik（旧项目）**：现有功能、算法、硬件行为、用户工作流和历史实现的事实来源。
- **Zentwik Next**：重新定义产品和架构后的新实现。允许重写、补全和适度扩展，不受旧技术栈约束。
- **本仓库**：存放研究任务、证据、原型、决策记录和架构基线。

旧项目是迁移参考，不是新项目的设计边界。

## 当前阶段

阶段 0 已完成（产品定义、旧版盘点、硬件链路、UI/IPC、技术栈评估与 ADR）。M2 完成应用层拆分、CLI 正式入口、服务化骨架、AMD CO 通道与场景接入账本；M3 完成 IPC 拒绝显式报错、服务启停与崩溃恢复动作、场景退出/重启回退、界面可访问性与列表虚拟化实测。产品代码位于 `Motu-Z/Zentwik` 的 `next` 分支，`main` 未改动。

成果索引：

- `research/01-legacy-inventory.md` 旧版功能与实现盘点
- `research/02-hardware-access.md` AMD SMU / Intel MSR / 电源 / 调度链路
- `research/03-ui-ipc-deployment.md` UI 框架、进程模型、IPC、部署
- `research/04-tech-stack-evaluation.md` 全栈评估（含实测数据）
- `decisions/ADR-001..007` 技术栈、UI、进程、协议、方案与所有权、持久化、CLI 契约
- `research/handoffs/M1-2026-11-25.md` M1 交接（骨架与纵向切片）
- `research/handoffs/M2-2026-11-25.md` M2 交接（CLI、服务化、硬件通道）
- `research/handoffs/M3-2026-11-25.md` 当前交接（IPC 拒绝、服务恢复动作、场景退出回退、界面可访问性实测）
- 产品代码：`D:\Projects\Zentwik Next`（README 含构建、运行、命令面与退出码）

## 核心原则

1. 以真实需求为中心，证据优先，区分事实、推断、假设和建议。
2. 允许重新设计，不机械继承旧项目的技术栈、抽象和缺陷。
3. 适度扩展：收益明确、成本合理且紧密相关的能力可纳入；重大扩展进入路线图。
4. 不为模块化而模块化；抽象必须带来隔离、复用、替换或治理收益。
5. 优先验证硬件访问、驱动权限、实时数据、自动化恢复和插件隔离等高风险点。
6. 总协调 AI 可以自行派发子智能体，但必须统一任务边界、证据标准、产出格式和交叉审查。
7. 文档以简体中文为主，技术名词和代码标识保留原文。

## 核心业务模型

CPU 调校方案（CPU Tuning Plan）、电源方案（Power Plan）、进程调度方案（Scheduling Plan）必须保持三个独立领域对象，各自拥有完整生命周期。场景不是第四种方案，而是跨领域协调机制，也可以执行独立动作。

自动化必须处理多规则命中、优先级、状态所有权、用户手动接管、退出恢复和执行历史。禁止采用无条件快照回写覆盖用户后续修改。

模块、能力、业务领域、硬件平台、产品功能和 UI 页面是不同概念。一个领域可以有多个实现提供者，一个模块可提供多个能力并贡献多个 UI 区块。区分安装、启用、加载、运行和健康状态。

插件系统是长期方向。区分内置模块、官方扩展和第三方插件；第三方插件默认独立进程，通过受控 IPC 请求特权操作。第一版不要求插件市场，但架构应预留版本化协议、权限、隔离和兼容机制。

国际化从第一版纳入，首批语言为 zh-CN、zh-TW、en-US，支持运行时切换和缺失翻译回退。用户自建名称不随语言切换改变。主题、强调色、信息密度和缩放均应可配置。

## 研究范围

研究 Windows 硬件访问与系统服务、AMD SMU、Intel MSR/OC Mailbox、PawnIO 等访问链路；比较 Rust、C#/.NET、C++ 及混合架构；比较 Tauri、Avalonia、Qt、WinUI 等 UI 技术；研究模块运行时、插件隔离、数据模型、IPC、监控、方案应用、自动化恢复、国际化、部署、测试和诊断。

Windows 是首要目标。跨平台按能力逐项定义，不得为跨平台抽象牺牲 Windows 核心能力。

## 目录规划

- PROJECT_BRIEF.md：产品目标与核心业务模型
- ARCHITECTURE_CONSTRAINTS.md：设计约束
- RESEARCH_PLAN.md：阶段与任务计划
- research/：分领域研究报告
- prototypes/：隔离原型与实验
- decisions/：ADR 决策记录
- templates/：研究报告模板
- prompts/MASTER_COORDINATOR.md：多智能体总协调提示词

## 旧项目

https://github.com/Motu-Z/Zentwik
