# 命名规范

- 规范名称：Zentwik 命名规范（Naming Conventions）
- 领域：`terminology`
- 核验状态：Proposed（项目内部约定）
- 权威来源：.NET 设计指南（Microsoft）；JSON/TypeScript 惯例；本仓库既有代码与 ADR
- 相关文档：[core-vocabulary.md](core-vocabulary.md)、[numeric-semantics.md](../data/numeric-semantics.md)

## 1. 适用范围

代码标识符、文档字段、IPC 命令字、配置项、方案/场景 ID、文件名。命名不是风格偏好，而是检索与治理的基础设施。

## 2. 命名约定（Proposed）

| 对象 | 约定 | 示例 |
|---|---|---|
| C# 类型/方法 | PascalCase | `CurveOffsetStep`、`ApplyPlanAsync` |
| C# 私有字段/局部变量 | camelCase | `desiredValue` |
| IPC 命令字 | `域名.动作`，小写 | `telemetry.subscribe`、`plans.apply` |
| IPC 错误码 | `域.原因`，稳定枚举 | `protocol.version_mismatch`、`request.payload_invalid` |
| JSON 字段 | camelCase，显式单位后缀 | `coOffset`、`voltageOffsetMv`、`intervalMs` |
| 方案判别字段 | `planType`（保留字） | `cpuTuning` / `power` / `scheduling` |
| 文档文件名 | 主题-日期 或 kebab-case | `amd-curve-optimizer.md`、`M3-2026-11-25.md` |
| ADR 文件名 | `ADR-NNN-英文短名.md` | `ADR-005-plans-and-ownership.md` |
| 能力/模块 ID | 反向域名式点分 | `amd.cpu`、`core.runtime`、`generic.power` |

## 3. 单位与语义后缀（与数值规范联动）

字段名必须携带足够语义，禁止裸 `value`：

- 单位后缀：`mV`、`MHz`、`Ms`、`Percent`、`Ratio`、`Steps`、`Wh`
- 语义前缀：`raw`（原始编码）、`desired`、`actual`、`previous`、`baseline`
- 强类型优先于后缀，但 IPC/JSON 边界仍需后缀自描述

详见 [numeric-semantics.md](../data/numeric-semantics.md) §4、§8。

## 4. 状态与集合

- 生命周期状态用完整枚举名，不用缩写布尔：`Installed`/`Enabled`/`Loaded`/`Running`/`Healthy`/`Degraded`/`Failed`。
- 集合字段用复数：`plans`、`modules`、`failedCores`。
- 布尔字段用肯定式：`enabled`、`persistsAcrossReboot`，避免 `notSupported`。

## 5. 常见误解

| 误解 | 纠正 |
|---|---|
| "改名只是重构小事" | IPC 命令字、错误码、持久化字段都是契约，改名是破坏性变更 |
| "JSON 字段可以随意加" | 增量字段需兼容旧读者；见 serialization-persistence.md |
| "大小写不重要" | 身份比较用 Ordinal；显示排序用 CurrentCulture（research/03 §6） |

## 6. 核验状态

全部为 Zentwik Proposed 约定，尚未被产品代码全量统一。外部权威仅用于 C# 命名惯例参考。
