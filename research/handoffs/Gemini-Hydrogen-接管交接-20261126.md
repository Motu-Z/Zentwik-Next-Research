# Gemini Hydrogen 临时接管工程交接报告（2026-11-26）

执行范围：受用户委托在 DeepSeek 暂不可用期间接管 `Motu-Z/Zentwik-Next-Research`（`main`）与 `Motu-Z/Zentwik`（`next`）的审计、核心修复、GUI 完善与文档同步。

> **环境限制声明**：本轮执行环境为无 `.NET SDK` 的 Linux 沙盒（`which dotnet` 无可用 SDK），因此所有 C#/XAML 改动均通过静态语法、JSON 词条覆盖、资源键映射与签名比对完成核验，**尚未在本机 Windows 环境执行 `dotnet build Zentwik.slnx` 与 `dotnet test Zentwik.slnx`**，也未进行真实硬件写入。

---

## 1. 功能与任务状态总表

| 项目 | 状态 | 依据 / 提交 |
|---|---|---|
| 全库技术事实审计（第九轮） | **已完成** | `Motu-Z/Zentwik-Next-Research` `main` 提交 `0db1f60`、`010220c`、`293c8e1`（62 篇 Markdown 相对链接 0 断裂） |
| M4-A1（CO 数值编码 + APOB/Windows 拓扑） | **已完成（上游前置提交）** | `Motu-Z/Zentwik` `next` 提交 `cdf6aaa`、`ed4e55f` |
| 核心层 `PlanCoordinator` 电源/调度缺陷修复与单测 | **已完成（代码已推送，待 Windows 构建实测）** | `Motu-Z/Zentwik` `next` 提交 `1fa429f` |
| `ModuleCatalog` 详情去重 & CLI `ledger` 列宽对齐 | **已完成（代码已推送，待 Windows 构建实测）** | `Motu-Z/Zentwik` `next` 提交 `1fa429f` |
| `Zentwik.Desktop` WPF GUI 完善 | **已完成（代码已推送，待 Windows 构建与 STA UI 测试）** | `Motu-Z/Zentwik` `next` 提交 `a205989` |
| M4-A2（CO 范围按 L1/L2 分层与不对称提示） | **未实现** | 当前仍统一钳制 `±50` |
| M4-A3（FMax 写入闭环） | **未实现** | `PlanCoordinator` 仍显式 `Skipped` |
| M4-A4（Intel MSR 写入协议与方案接线） | **未实现** | `IIntelTuningProvider.TryWriteVoltageOffset` 仍无方案调用方 |
| M4-B1（Session 0 服务连接真机复验） | **待验证** | 需 Windows 管理员权限实机测试 |
| M4-B2（调度 `Enforce` watcher 与 CPU Sets 决策） | **未实现** | `SchedulingPlan.Rules.Enforce` 仍无后台 watcher |
| M4-C1（休眠唤醒实际值复核） | **未实现** | 尚无电源唤醒事件监听 |

---

## 2. `Motu-Z/Zentwik`（`next`）本轮提交明细

### 提交 1：`1fa429f09635ef702a3a9df921fbd1fed7e4b459`
`fix: 修复电源前置读取与调度部分接管越权写，去重模块描述与账本对齐`

1. **`src/Zentwik.Core/Plans/PlanCoordinator.cs`**：
   - `ApplyPower`：`TryReadValue` 读取失败时不再以 `0` 作为 `PreviousValue` 继续写入，而是记录 `LedgerStatus.Skipped` 并跳过写入；写入后若存在 `Applied` 项则检查 `TrySetActiveScheme` 返回值，失败时返回明确错误使结果降级为 `PlanApplyState.Partial`。
   - `ApplyScheduling`：将原先合并布尔值 `acquired |= ...` 改为按 `affinity` / `priority` 两字段分别检查账本所有权，拒绝的字段记录 `Skipped` 且仅向 `_scheduler.TryApply` 传入获得写权的字段（传 `null` 给被拒绝字段），消除部分接管时的越权覆盖。
2. **`src/Zentwik.Core/Modules/ModuleCatalog.cs`**：
   - `Describe` 拼接多个能力的 `Detail` 前增加 `.Distinct(StringComparer.Ordinal)`，避免多能力共用相同缺失提示时重复输出同一句话。
3. **`src/Zentwik.Cli/Commands.cs`**：
   - `LedgerAsync` 将 `int.MaxValue - 1` 及以上的优先级格式化为 `max`，避免 `p2147483646` 破坏终端列对齐。
4. **`tests/Zentwik.Core.Tests/PlanAndSceneTests.cs`**：
   - 新增 `Power_PreRead_Failure_Skips_Without_Writing`、`Power_Reactivation_Failure_Reports_Partial_And_Error`、`Scheduling_User_Owned_Field_Is_Not_Overwritten_When_Sibling_Field_Acquired` 3 个单元测试（使用私有 `FailingPowerPolicyService` 替身与 `FakeProcessScheduler`）。

### 提交 2：`a205989f628b4f4f3fdc639e9304360b97484e79`
`feat(desktop): 完善 WPF 主界面布局、方案与场景操作、调校与账本视图、缩放与错误提示`

1. **`src/Zentwik.Desktop/ViewModels/RelayCommand.cs`**：新增 `RelayCommand` / `AsyncRelayCommand`。
2. **`src/Zentwik.Desktop/ViewModels/MainViewModel.cs`**：
   - 保持仅依赖 `Zentwik.Contracts` / `Zentwik.Domain` / `Zentwik.Localization`，保留 `PlanRow(Kind, Name, Caption)` 三参兼容签名及用户自定义 `Language`/`Theme`/`Accent`/`UiScale`。
   - 接入方案应用/回退/删除/新建（CPU CO、Power、Scheduling）、场景激活/停用/启停/删除/新建、AMD/Intel 调校只读快照与逐核 CO 展示、状态账本列表与用户接管释放、全局提示条与 `UiScale` 持久化，以及 `Events.SceneChanged` 实时同步。
3. **`src/Zentwik.Desktop/MainWindow.xaml` & `Themes/Base.xaml`**：
   - 保留 `Plans` 页签在索引 `1` 且 `ListBox` 直接处于 `*` 网格行以维持 UI 虚拟化；所有交互控件（`Button`、`ComboBox`、`ListBox`、`CheckBox`、`TextBox`）均配置 `AutomationProperties.Name`；根节点绑定 `ScaleTransform` 实现 `UiScale` 缩放；新增“调校与账本”页签及全局通知栏。
4. **`src/Zentwik.Localization/Resources/strings.{zh-CN,zh-TW,en-US}.json`**：
   - 同步补齐全部新增界面键（共 64 键，三语 100% 对齐，已通过 Python 静态脚本校验与 `LocalizationBridge.KeySuffix` 映射校验）。

---

## 3. DeepSeek 恢复后建议优先执行

1. 在 Windows 本机拉取 `Motu-Z/Zentwik` `next` 最新提交（`a205989`），执行：
   ```bash
   dotnet build Zentwik.slnx
   dotnet test Zentwik.slnx
   ```
   确认新增 3 项 Core 单测及 `Zentwik.Desktop.Tests`（`AccessibilityTests`）全部通过、0 警告 0 错误。
2. 人工启动 `Zentwik.Agent --fake` 与 `Zentwik.Desktop`，走查方案创建/应用/回退、场景激活/停用、调校与账本释放、`UiScale` 切换与主题/强调色保持。
3. 按 `接手核验-20261126.md` 排期继续推进 M4-A2/A3/A4/B1/B2/C1。
