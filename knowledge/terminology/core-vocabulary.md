# 核心词汇

- 规范名称：Zentwik 核心词汇（Core Vocabulary）
- 领域：`terminology`
- 核验状态：Proposed（项目内部语义定义）
- 权威来源：本仓库 PROJECT_BRIEF、ARCHITECTURE_CONSTRAINTS、ADR-001..007
- 相关文档：[naming-conventions.md](naming-conventions.md)、[../architecture/modules-capabilities.md](../architecture/modules-capabilities.md)、[../domain/three-plans.md](../domain/three-plans.md)

## 1. 目的

本文件是"哪些概念不是同一件事"的权威裁决点。术语一旦在此定义，其他文档引用而不重复定义。

## 2. 术语表

| 术语 | 定义 | 不是 | 详见 |
|---|---|---|---|
| 模块（Module） | 声明 ID/版本/能力/依赖/平台要求的可加载单元 | 不是业务领域，不是 UI 页面 | [modules-capabilities.md](../architecture/modules-capabilities.md) |
| 能力（Capability） | 模块对外声明的一项可做事项 | 不是可用性状态 | 同上 |
| 状态（State） | 对象标识/版本/当前态势 | 不是安装状态 | 同上 |
| 业务领域（Domain） | CPU 调校 / 电源 / 调度 / 监控 / 自动化 | 不等于模块 | ../domain/ |
| 硬件平台 · Provider | 对某平台实现某领域读写的具体组件 | 不是领域本身 | modules-capabilities.md |
| 产品功能 | 面向用户的完整价值单元 | 不等于单个能力 | PROJECT_BRIEF |
| UI 区块（Contribution） | 模块向界面贡献的一块区域 | 不等于页面 | modules-capabilities.md |
| 插件（Plugin） | 进程外、受控权限的扩展 | 不是内置模块 | modules-capabilities.md |
| 方案（Plan） | 三种独立领域对象之一，带版本与应用/回滚路径 | 不是"当前方案" | [three-plans.md](../domain/three-plans.md) |
| 场景（Scene） | 跨领域协调机制，声明触发/引用/优先级/退出策略 | **不是第四种方案** | [scenes-arbitration.md](../domain/scenes-arbitration.md) |
| 状态账本（Ledger） | 记录每个被写对象的值来源与所有权的机制 | 不是日志 | [state-ownership.md](../domain/state-ownership.md) |
| 安装/启用/加载/运行/健康 | 五个独立的模块状态维度 | 不可合并为单一布尔 | modules-capabilities.md |
| 通道状态 | `Available`/`NotSupported`/`DriverUnavailable`/`ProbeFailed` | 不可压成 bool | [ipc.md](../architecture/ipc.md) |

## 3. 生命周期与态势（区分维度）

- 模块生命周期：Installed → Enabled → Loaded → Running → Healthy/Degraded/Failed
- 方案态势：定义（Definition）/ 期望状态（Desired）/ 观测状态（Observed）/ 硬件实际状态（Actual）
- 账本所有权：Owner ∈ {Scene, Plan, User, System}；前值/目标值/实际值/来源/优先级/恢复资格分列

## 4. 常见误解

| 误解 | 纠正 |
|---|---|
| "一个领域只有一个实现" | 一个领域可有多个 Provider |
| "场景是一种方案" | 场景是协调机制，不是第四种方案 |
| "模块=页面" | 一个模块可贡献多个 UI 区块 |
| "capability 可用就是 healthy" | 五态独立，缺一不可 |

## 5. 核验状态

均为 Zentwik 项目约定（Proposed），源自已 Accepted 的 ADR 与产品简报，作为内部唯一定义源。
