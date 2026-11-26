# 逐功能迁移映射表

- 用途：跟踪旧版能力 → 新版实现位置 → **软件实现状态** → **真机验证状态**。
- 依据：`research/01-legacy-inventory.md`（旧版事实） + 产品仓库 `next` 源码（新版事实）。
- 建立：2026-11-26（P0-C）。基线产品 `next` `bb5beb5`；构建 0 警告 0 错误；测试 162 例全通过；CI 通过。
- 维护规则（路线图 §10）：**每完成一个领域只更新对应行**，不重写本表结构；代码、测试、真机证据是唯一依据，文档勾选不算。

## 状态图例

| 维度 | 取值 |
|---|---|
| 软件状态 | `已迁移` 行为等价/改进；`部分` 主链路可用但缺子项；`未实现`；`不适用`（如托盘随 GUI 暂停） |
| 真机验证 | `已验证`（本机真机闭环）；`仅fake/单测`；`无需`（纯软件）；`未验证`（缺硬件/权限） |

> **软件状态与真机验证必须分账**：单测/fake 通过只能写"仅fake/单测"，不得写"已验证"。

## 主表

| # | 领域 | 旧版入口/核心类 | 新版位置 | 软件状态 | 真机验证 | 差异 / 待办 |
|---|---|---|---|---|---|---|
| 1 | AMD CO | `SmuReader`、`TuningApplyService`、`BootGuard` | `Platform/Windows/AmdSmuTuningProvider.cs`、`AmdCoreMap.cs`、`AmdApobCoreMap.cs`、`AmdWindowsTopology.cs`、`PawnIoSession.cs`；`Domain` `CurveOptimizerBounds`；`Core` `PlanCoordinator.ApplyCpu` | 已迁移（逐核读写、前值/回读/容差、方案应用、固件拒绝报 rejected） | 已验证（16/16 核读；核0 −10 写+回读+回退，账本 prev −18） | 效果层（频率/功耗/温度）不做观测，见 KNOWN_LIMITS §2 |
| 2 | AMD FMax | `SmuReader`、`TuningApplyService` | `Platform/FMaxAbstractions.cs`、`AmdSmuTuningProvider.cs`；`PlanCoordinator` | 已迁移（写前基线、±10 MHz 容差回读、Unsupported/Failed 区分） | 已验证（基线 4450→写 4750→回读→回退 4450） | 基线 0 判 Failed 而非 Unsupported |
| 3 | 硬件功耗限制（STAPM/PPT FAST/SLOW/TDC/EDC、Intel PL1/PL2/Tau） | 旧版曾实现后移除（`LEGACY_POWER_CONTROL_INVESTIGATION.md`） | 无 | 未实现 | 未验证 | **需产品确认范围**；HX 默认不展示常规入口，仅允许受控尝试；本机 HX 不作验证前置 |
| 4 | Intel 调校 | `IntelTuningChannel`、`IntelTuning` | `Platform/Windows/IntelMsrTuningProvider.cs`、`IntelMsrBits.cs`；`PlanCoordinator` | 已迁移（OC Mailbox 命令字、bit63 忙位轮询、`Global\Access_Intel_OC_Mailbox` 互斥、写后经读命令回读 2 mV） | 仅fake/单测（无 Intel 真机；`IntelMSR.bin` 在 AMD 返 ERROR_NOT_SUPPORTED） | 需 Intel 真机确认命令字/忙位/世代差异（KNOWN_LIMITS §4） |
| 5 | 电源方案 | `PowerPolicyService`、`PowerPolicyApi`（scheme CRUD + 8 策略项 AC/DC） | `Platform/Windows/Win32PowerPolicyService.cs`（GetActive/List/SetActive/Read/Write/Duplicate/Rename/Delete）、`Core/Power/PowerSchemeService.cs`、`Domain/Power/PowerSchemePresets.cs`；`PlanCoordinator.ApplyPower`；CLI `plans`/`power` | 已迁移：读策略/写策略/应用/激活 + scheme 增删改查（内置拒删、重名查重、活动先切走） | Win32 列表真实只读已验证；scheme CRUD 走 fake 端到端 + 单测（未在真实系统建/删方案） | 策略项集合与逐项错误语义仍待核对（P1-B 剩余） |
| 6 | 进程调度 | `ProcessWatcher`、`CpuSetScheduler`、`ProcessPriority`、`NativeProcessList`、`MaskBinding` | `Platform/Windows/AffinityScheduler.cs`、`NativeProcessList.cs`、`CpuSetsNative.cs`、`CpuSetsCapabilityProbe.cs`；`Core/Scenes/SchedulingWatcher.cs`、`PlanCoordinator.ApplyScheduling/Enforce` | 已迁移（亲和性/优先级/CPU Sets、持续 enforce、漂移检测、PID 复用按创建时间、核心停泊提示） | 已验证（affinity `0xF:BelowNormal`、cpusets `0-3` 系统侧复核并可恢复） | >64 逻辑处理器/多组不支持（明确拒绝，KNOWN_LIMITS §8） |
| 7 | 监控 | `ProcessRefresher`、`SmuReader` 快照 | `Platform/Windows/Win32TelemetrySource.cs`；`Core/Telemetry/TelemetryHub.cs`；CLI `monitor` | **部分**：CPU 总占用 + 内存；无温度/频率/功耗/GPU/磁盘/网络，无历史持久化 | 已验证（采样链路取到真实 CPU/内存） | P2 扩展指标；历史按需求 |
| 8 | 场景自动化 | `ScenarioService`、`ScenarioMonitor`（4 触发：进程运行/退出/电池/交流） | `Core/Scenes/SceneEngine.cs`；`Domain/Scenes/Scene.cs`；CLI `scenes` | **部分**：触发/仲裁（优先级→进入时间→ID）/所有权/退出恢复/保存即重应用已迁移；触发器为 manual/process/powerSource **三类** | 已验证（fake 端到端：apply→账本→场景接管与回退；真机注册链路） | 核对"进程启动/退出"是否需拆为两种；条件/持续时间/组合未做（P2） |
| 9 | 模块与能力发现 | `Capabilities/*`、`Modules/*`、`CapabilityIds` | `Core/Modules/ModuleCatalog.cs`、`Core/Capabilities/CapabilityService.cs`、`Domain/Capabilities/Capability.cs`、`Platform/Windows/WindowsCapabilityProbes.cs` | 已迁移（能力五态；跨厂商按任一适用聚合，另一厂商缺席不判降级） | 已验证（`caps` 真机 4/6 能力，`cpu-tuning` ready） | 无磁盘发现/启用禁用（旧版也无，非回归） |
| 10 | 界面设置 | `UiSettingsStore`、`AppSettings` | `Domain/Settings/UserSettings.cs`、`Core/State/StateStore.cs`；CLI `settings` | 已迁移（语言/主题/强调色/缩放；单一写入方为 Agent） | 无需（纯软件） | 旧版设置分裂两进程的问题已消除 |
| 11 | 日志诊断 | `LogService`、`LogOptions` | `Core/Diagnostics/DiagnosticLog.cs`、`DoctorService.cs`；CLI `logs`/`doctor` | 已迁移（分级、按天落盘、doctor 诊断） | 无需 | 结构化字段/操作 ID（旧版也无，非回归） |
| 12 | 托盘 | `TrayHost` | 无 | 未实现 | 不适用 | 随 WPF/GUI 阶段（P4） |
| 13 | 开机自启 | `AutoStartService`（schtasks） | 服务以 SCM 自动启动替代；无用户级"登录自启 UI" | 部分 | 已验证（服务 LocalSystem 自动启动） | 用户级自启随 GUI 阶段 |
| 14 | 安装升级 | Inno 安装包 + 便携 zip | 无安装器（从构建输出注册服务） | 未实现 | 不适用 | P3；固定安装目录 + 打包 + PawnIO 分发决策 |
| 15 | 单实例 | Worker/UI 双互斥体 | 服务单实例天然；客户端无状态 | 已迁移 | 无需 | — |

## 由本表导出的剩余开发任务（按优先级）

1. ~~**P1-B 补电源方案 CRUD**（#5）~~：已完成（2026-11-26，`power list/create/rename/delete` + `PowerSchemeService`）。剩余：核对策略项集合与逐项错误语义。
2. ~~**P0-E PawnIO/APOB 决策记录**~~：已完成（2026-11-26，[ADR-008](../decisions/ADR-008-hardware-driver-and-apob.md)）。结论：唯一硬件驱动为 PawnIO + 官方签名模块，不引入第二驱动；APOB 不可读（PawnIO 模块无物理内存读，ZenStates-Core 走 inpoutx64 `MapPhysToLin`），核映射主路径改为 CPUID `0x8000001E` + L3 域。**遗留未完成项：分发合规**——仓库尚无 GPL-2.0/LGPL-2.1 文本与模块来源说明。
3. **P0-D 硬件验证记录**：已分散在 `research/handoffs/M4-底层接手`，本表已汇总为"真机验证"列；如需独立证据文件再补。
4. **P1-A 剩余：硬件功耗限制**（#3）：需产品确认范围（HX 默认隐藏、受控尝试）。
5. **场景触发器核对**（#8）：确认 process 触发器是否覆盖"进程退出"。

## 阻塞项（缺条件，不在本轮）

| 项 | 阻塞原因 | 解除条件 |
|---|---|---|
| Intel 全通道 | 无 Intel 硬件 | 12/13/14 代真机 |
| 真实休眠/唤醒回调 | 需合盖唤醒 | 用户合盖后查日志 `系统恢复核对：…` |
| 重启后保持 | 需真实重启 | 重启后 `tuning` 对比 |
| 多处理器组 | 本机单组 | >64 逻辑处理器机器 |
| 功耗限制真机 | HX 固件/产品范围 | 产品确认 + 对应硬件 |
