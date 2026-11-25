# 生命周期状态模型

- 规范名称：生命周期与状态维度（Lifecycle & State Dimensions）
- 领域：`architecture`
- 核验状态：Proposed（项目内部约定）
- 权威来源：本仓库 ARCHITECTURE_CONSTRAINTS §7、ADR-003、research/01 §5.1
- 相关文档：[modules-capabilities.md](modules-capabilities.md)、[../domain/state-ownership.md](../domain/state-ownership.md)

## 1. 目的

区分"模块的存活状态""方案的应用状态""硬件的实际状态"三组独立的状态维度，禁止把它们压成一个布尔或一条字符串。

## 2. 模块生命周期（Proposed）

| 状态 | 含义 | 转化条件 |
|---|---|---|
| Installed | 文件存在、已注册 | 部署完成 |
| Enabled | 用户/策略允许启用 | 显式启用 |
| Loaded | 已加载进宿主 | 依赖满足 |
| Running | 正在执行 | 调度启动 |
| Healthy | 运行且自检通过 | 健康探针 |
| Degraded | 运行但部分能力受损 | 部分失败 |
| Failed | 失败 | 异常 |

状态查询必须能表达"某能力不可用但其余正常"，不得升级为全局错误页。

## 3. 方案态势（Proposed）

区分四层：定义（Definition）/ 期望状态（Desired）/ 观测状态（Observed）/ 硬件实际（Actual）。

- `ActualValue` 与 `DesiredValue` 不符即判定未生效，禁止写成"已应用"（ADR-005）。
- 回退失败保留 `Applied` + `Restorable=true` 供重试，不能标成"已恢复"。

## 4. 通道可用性（Verified 设计原则）

`Available` / `NotSupported` / `DriverUnavailable` / `ModuleMissing` / `ProbeFailed` / `PrivilegeMissing` 分开表达，不可压成 bool。

## 5. 服务与代理生命周期（Verified 实现）

- 服务宿主：`--service` / `--install-service` / `--uninstall-service` / `--start-service` / `--stop-service`。
- 启停走 SCM，状态等待带超时；异常终止写入服务状态以触发恢复。
- 关闭代理时回退活动场景；同启动周期内补回退遗留写入；整机重启后不擅自改写。

## 6. 常见误解

- "Failed 就整体不可用" → 故障应局部化。
- "期望值=实际值" → 请求型硬件可能忽略写入，须回读验证。
- "重启后恢复默认" → 固件持久化的值可能是用户有意保留的。

## 7. 核验状态

状态维度划分 = Proposed；服务生命周期实现细节 = Verified（本仓库交接记录）。
