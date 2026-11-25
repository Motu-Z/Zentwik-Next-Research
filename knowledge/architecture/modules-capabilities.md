# 模块、能力与生命周期

- 规范名称：模块与能力模型（Module & Capability Model）
- 英文名称：Module, Capability, Provider, Plugin
- 缩写：无
- 领域：`architecture`
- 核验状态：Proposed（项目内部约定，源自 Accepted ADR）
- 权威来源：本仓库 PROJECT_BRIEF、ARCHITECTURE_CONSTRAINTS §7/§8、ADR-005、research/01 §5
- 相关文档：[core-vocabulary.md](../terminology/core-vocabulary.md)、[security-service.md](../windows/security-service.md)、[lifecycle.md](lifecycle.md)

## 1. 定义（项目约定）

- **模块（Module）**：声明 ID、版本、能力、依赖、冲突、平台要求与生命周期的可加载单元。
- **能力（Capability）**：模块对外声明的一项可做事项；能力目录记录其状态。
- **Provider**：对某硬件平台实现某领域读写的组件；一个领域可有多个 Provider。
- **插件（Plugin）**：进程外、受控权限的扩展，默认独立进程。

## 2. 概念区分（裁决点）

| 概念 | 不是 | 说明 |
|---|---|---|
| 模块 | 不是业务领域/页面 | 一个模块可提供多种能力、贡献多个 UI 区块 |
| 能力 | 不是可用性状态 | 状态独立表达 |
| 领域 | 不是模块 | 一个领域可有多个 Provider |
| 内部模块 API | 不是稳定插件 API | 插件契约须单独版本化 |
| 安装 | 不是启用 | 五个状态维度独立 |

## 3. 生命周期状态（Verified 概念，来自旧版五维模型）

Installed → Enabled → Loaded → Running → Healthy / Degraded / Failed。

- 故障尽量局部化：缺少某模块不应使无关功能失效。
- 旧版五维模型（Installed/Enabled/HardwareSupported/RuntimeAvailable/Active + Failed）概念正确，可作基线。

## 4. 插件（长期方向）

- 区分内置模块、官方扩展、第三方插件。
- 第三方插件默认独立进程，通过受控 IPC 请求特权操作；权限按方法声明。
- v1 不开放第三方插件 UI；后续只允许声明式 UI schema，禁止插件 XAML 进入 UI 进程（XAML 可执行任意代码）。
- 预留版本协商、签名、超时、故障隔离与 capability token（绑定 PID+SID+启动时间防 PID 复用）。

## 5. 决议状态

模块=声明单元、能力=对外事项、Provider=平台实现、五态独立 = Proposed（项目约定）；插件隔离原则来自 Accepted ADR。

## 6. 关联文档

[core-vocabulary.md](../terminology/core-vocabulary.md) · [lifecycle.md](lifecycle.md) · [ipc.md](ipc.md) · [state-ownership.md](../domain/state-ownership.md)
