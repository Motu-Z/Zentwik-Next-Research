# 分片 I：项目文档与架构决策（`project/*` 与 `decisions/ADR-001..008`）只读静态审计

- **旧项目基准**：`Motu-Z/Zentwik@main` (`727462601911609e6dacbe3cb8f144246829087a`)
- **新项目对照**：`Motu-Z/Zentwik@feat/gui-protocol` (`100e9c0f4f9e7f9b160f8b0bed9ea99afdf20922`)
- **研究文档基准**：`Motu-Z/Zentwik-Next-Research@main` (`7680e5109ed010e4681180393ff85fb2ce412554`)

## 1. 闭合计数

- **ADR 决策文档条数**：**8 篇**（`ADR-001` .. `ADR-008`，不含模板）。
- **`MIGRATION_MATRIX.md` 行数**：主表 **15 行**（`#1` .. `#15`）+ 阻塞项表 5 行。
- **要求条目总数**：**44 条**（`R-01` .. `R-44`）。
- **状态分布**：`一致` 13、`部分` 10、`缺` 1、`新有旧无` 9、`文档与现实冲突` 11。

## 2. 逐条对照表

| 编号 | 文档要求(路径:行) | 旧项与证据(文件:函数/行) | 新项与证据(文件:函数/op) | 状态 | 验证级别 | 备注 |
|---|---|---|---|---|---|---|
| R-01 | `ADR-001:13`：全栈 .NET 10；特权服务用 NativeAOT | `Zentwik.Worker.csproj` | `Zentwik.Agent.csproj:7` (`<PublishAot>false</PublishAot>`) | 文档与现实冲突 | 静态源码 | `observed`：`Agent` 未开 AOT |
| R-02 | `ADR-001:30`：`Zentwik.Platform` 隔离硬件访问 | 分散在 `Zentwik.Core` | `Zentwik.Platform/Abstractions.cs`；`Desktop` 不引用 `Platform` | 新有旧无 | 静态源码 | `observed` |
| R-03 | `ADR-001:26`：源生成 JSON 上下文适配 NativeAOT | `Zentwik.Core/Serialization/` | `Zentwik.Contracts/Json/ZentwikJson.cs:13-76` | 新有旧无 | 静态源码 | `observed` |
| R-04 | `ADR-002:13`：WPF + 自绘图表，ViewModel 无 UI 引用 | `Zentwik.UI` (WPF) | `Zentwik.Desktop.csproj:6` (WinForms 裸控件)；`MainViewModel.cs` 无 UI 引用 | 文档与现实冲突 | 静态源码 | `observed`：已由最新 WinForms 口径覆盖 |
| R-05 | `ADR-003:13-16,36-41`：LocalSystem 服务与调用方校验 | `Zentwik.Worker` | `WindowsServiceHost`、`ServiceInstaller`、`PipeCaller:31-60` | 新有旧无 | 静态源码 | `observed` |
| R-06 | `ADR-004:13-24`：4B 帧、多路复用、事件与背压 | `PipeServerService` | `Protocol.cs`、`AgentSession.cs:620-624` | 新有旧无 | 静态源码 | `observed` |
| R-07 | `ADR-005:13-19,23`：三方案、仲裁、`StateLedger`、`PersistsAcrossReboot` | `PlanService`、`ScenarioService` | `Plan.cs`、`StateLedger.cs`、`PlanCoordinator.cs` | 新有旧无 | 静态源码 | `observed` |
| R-08 | `ADR-006:12-17`：分文件持久化、原子替换、`SchemaVersion` | 分文件 JSON | `StateStore.cs:21`（合并写单文件 `state.json`） | 文档与现实冲突 | 静态源码 | `observed` |
| R-09 | `ADR-007:13-18`：正式 CLI，7 退出码，`--json` | 无 | `Zentwik.Cli` | 新有旧无 | 静态源码 | `observed` |
| R-10 | `ADR-008:21-31`：唯一驱动 PawnIO 与随包签名模块 | `PawnIoDevice.cs`、`libs/PawnIO` | `PawnIoSession.cs`、`modules/PawnIO` | 一致 | 静态源码 | `observed` |
| R-11 | `ADR-008:33-37`：APOB 可选增强，主路径 CPUID + L3 | `CpuTopologyProbe.cs` | `AmdCoreMapResolver.cs:11-33` | 新有旧无 | 静态源码 | `observed` |
| R-12 | `ADR-008:39-43,76`：声称「分发合规尚未落地」 | `LICENSE` | 已补 `licenses/LGPL-2.1.txt` 与 `THIRD-PARTY-NOTICES.md` | 文档与现实冲突 | 静态源码 | `observed`：文档未更新 |
| R-13 | `PROJECT_BRIEF.md:23`：第三方插件独立进程隔离 | 无 | 无插件宿主 | 缺 | 静态源码 | `observed` |
| R-14 | `PROJECT_BRIEF.md:24`：三语运行时切换、主题/强调色/缩放 | `UiSettingsStore.cs` | `SupportedLanguages=["zh-CN"]`；WinForms 无主题/强调色/缩放 | 部分 | 静态源码 | `observed` |
| R-15 | `ARCHITECTURE_CONSTRAINTS.md:5-16`：12 条不变量 | 旧缺陷多 | 10 条主体满足；第 8 条插件缺，第 10 条分文件部分 | 部分 | 静态源码 | `observed` |
| R-16 | `RESEARCH_PLAN.md:3-28`：阶段 0–5 推进 | `7274626` | 核心已落地，WinForms 壳迭代中 | 部分 | 静态源码 | `observed` |
| R-17 | `PRODUCT_MIGRATION_AND_ROADMAP.md:214-245`：CPU 调校 | `SmuReader`、`IntelTuning` | `AmdSmuTuningProvider`、`IntelMsrTuningProvider`（仅 Fake 测） | 部分 | 静态源码 | `observed` |
| R-18 | `PRODUCT_MIGRATION_AND_ROADMAP.md:247-275`：Power 方案 CRUD | `PowerPolicyService` | `Win32PowerPolicyService`、`PowerSchemeService` | 一致 | 静态源码 | `observed` |
| R-19 | `PRODUCT_MIGRATION_AND_ROADMAP.md:277-307`：Scheduling 调度 | `ProcessWatcher` | `AffinityScheduler`、`SchedulingWatcher`（缺 `EcoQoS`） | 部分 | 静态源码 | `observed` |
| R-20 | `PRODUCT_MIGRATION_AND_ROADMAP.md:309-347`：Scenes 自动化 | `ScenarioService` | `SceneEngine`（缺全局 `ExternalChangePolicy`） | 部分 | 静态源码 | `observed` |
| R-21 | `PRODUCT_MIGRATION_AND_ROADMAP.md:349-385`：Monitoring 监控 | `ProcessRefresher` | `Win32TelemetrySource`、`AmdSensorSource`（缺 GPU/磁盘/网络/进程级） | 部分 | 静态源码 | `observed` |
| R-22 | `PRODUCT_MIGRATION_AND_ROADMAP.md:387-411`：设置、诊断与托盘 | `UiSettingsStore`、`LogService`、`TrayHost` | `UserSettings`、`DiagnosticLog`、`DoctorService`、`NotifyIcon` | 部分 | 静态源码 | `observed` |
| R-23 | `PRODUCT_MIGRATION_AND_ROADMAP.md:491-509`：P0 里程碑 5 项 | — | 均已闭合 | 一致 | 静态源码 | `observed` |
| R-24 | `PRODUCT_MIGRATION_AND_ROADMAP.md:564-574`：P4 恢复 WPF 与模块 UI | WPF | WinForms 裸控件 | 文档与现实冲突 | 静态源码 | `observed` |
| R-25 | `KNOWN_LIMITS.md:8-71`：10 项已知限制 | — | 代码边界与 10 项一致 | 一致 | 静态源码 | `observed` |
| R-26 | `INSTALLATION_DEPLOYMENT_MAINTENANCE.md:14-38`：声称 UI+Worker 双进程与便携/Maintenance 模式 | `installer/zentwik.iss`、`zip_portable.py` | `scripts/installer/zentwik.iss`（已改 `LocalSystem` 服务；无便携与 Maintenance） | 文档与现实冲突 | 静态源码 | `observed` |
| R-27 | `MIGRATION_MATRIX.md:21` (#1 AMD CO) | `SmuReader.cs` | `AmdSmuTuningProvider.cs`、`PlanCoordinator.ApplyCpu` | 一致 | 静态源码 | `observed` |
| R-28 | `MIGRATION_MATRIX.md:22` (#2 AMD FMax) | `SmuReader.cs` | `FMaxAbstractions.cs`、`AmdSmuTuningProvider.cs`（UI 表单无输入） | 一致（后端） | 静态源码 | `observed` |
| R-29 | `MIGRATION_MATRIX.md:23` (#3 功耗限制) | 已移除 | `AmdSmuTuningProvider`（AMD Zen4/5 部分可用，无回退；Intel 无） | 部分 | 静态源码 | `observed` |
| R-30 | `MIGRATION_MATRIX.md:24` (#4 Intel 调校) | `IntelTuning.cs` | `IntelMsrTuningProvider.cs`（仅 Core 平面，仅 Fake 测） | 部分（仅模拟验证） | 静态源码 + Fake | `observed` |
| R-31 | `MIGRATION_MATRIX.md:25` (#5 电源方案 CRUD) | `PowerPolicyService.cs` | `Win32PowerPolicyService`、`PowerSchemeService` | 一致 | 静态源码 | `observed` |
| R-32 | `MIGRATION_MATRIX.md:26` (#6 进程调度) | `ProcessWatcher.cs` | `AffinityScheduler`、`SchedulingWatcher` | 一致 | 静态源码 | `observed` |
| R-33 | `MIGRATION_MATRIX.md:27` (#7 监控) | `ProcessRefresher.cs` | `Win32TelemetrySource`、`AmdSensorSource`、`TelemetryRingBuffer` | 部分 | 静态源码 | `observed` |
| R-34 | `MIGRATION_MATRIX.md:28` (#8 场景自动化) | `ScenarioService.cs` | `SceneEngine.cs`、`Scene.cs` | 部分 | 静态源码 | `observed` |
| R-35 | `MIGRATION_MATRIX.md:29` (#9 模块与能力) | `Capabilities/*` | `ModuleCatalog.cs`、`CapabilityService.cs` | 一致 | 静态源码 | `observed` |
| R-36 | `MIGRATION_MATRIX.md:30` (#10 界面设置) | `UiSettingsStore.cs` | `UserSettings.cs`、`StateStore.cs` | 一致 | 静态源码 | `observed` |
| R-37 | `MIGRATION_MATRIX.md:31` (#11 日志诊断) | `LogService.cs` | `DiagnosticLog.cs`、`DoctorService.cs` | 一致 | 静态源码 | `observed` |
| R-38 | `MIGRATION_MATRIX.md:32` (#12 托盘，矩阵称未实现) | `TrayHost` | `MainForm.cs:32` 已实现 `NotifyIcon` | 文档与现实冲突 | 静态源码 | `observed`：矩阵停在 `bb5beb5` |
| R-39 | `MIGRATION_MATRIX.md:33` (#13 自启，矩阵称无 UI 自启) | `AutoStartService.cs` | `ShellBehavior.cs:60` 已实现 `StartupRegistration` | 文档与现实冲突 | 静态源码 | `observed`：矩阵滞后 |
| R-40 | `MIGRATION_MATRIX.md:34` (#14 安装包与便携包) | `zentwik.iss`、`zip_portable.py` | `scripts/installer/zentwik.iss:58` 打包 `pkg\*`（含 `Zentwik.Desktop`）；无便携 zip | 部分 | 静态源码 | `observed`：核实 `zentwik.iss` 确打包 `pkg\*` |
| R-41 | `MIGRATION_MATRIX.md:35` (#15 单实例，矩阵称无需) | 双互斥体 | `Desktop/Program.cs` 无互斥体，但已带托盘驻留 | 文档与现实冲突 | 静态源码 | `observed`：多开会建多托盘图标 |
| R-42 | `project/AI_UI_DESIGN_GUIDANCE.md:4,26` (**Deprecated，仅记差异**：Flutter) | WPF | WinForms | 文档与现实冲突 | 静态源码 | `observed`：Deprecated |
| R-43 | `project/UI_UX_DESIGN_SPEC.md:5,17-93` (**Deprecated，仅记差异**：WPF/Bento) | WPF | WinForms 裸控件 | 文档与现实冲突 | 静态源码 | `observed`：Deprecated |
| R-44 | `prototypes/*.html` (**Deprecated，仅记差异**：21 份原型) | WPF | WinForms 裸控件 | 文档与现实冲突 | 静态源码 | `observed`：Deprecated |
