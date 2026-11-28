# 九项待修缺口「已有 vs 缺」清单（Deepseek 产出，待独立复核）

- **产出方**：Deepseek（Windows 会话），2026-11-28
- **状态**：静态源码核对结论；**未经用户确认，不构成需求**；四个待拍板项见文末
- **基线**：产品 `feat/gui-protocol` @ `2d9351d`；研究仓库 `main` @ `dac46e3`
- **复核方**：Gemini（Linux 沙盒），复核报告写入 `research/handoffs/IMPLEMENTATION-REVIEW-20261128.md`

---

## 一、九项总表（全部为静态源码核对结论）

| # | 项 | 状态 | 断点/缺失位置 | 验证级别 |
|---|---|---|---|---|
| 1 | `process.*` op = 0，进程页空骨架 | **缺**（后端 op + UI 双缺） | `Protocol.cs:20-67` 无 process 域；`MainForm.cs:660-693` | 静态源码 |
| 2 | `power.scheme.activate` | **缺**（平台层已有，缺 op/契约/UI/CLI） | `PowerSchemeService.cs:415-420` 私有；`Commands.cs:391-405` 无 activate | 静态源码 |
| 3 | Desktop 单实例互斥体 | **缺** | `Desktop/Program.cs:9-26` | 静态源码 |
| 4 | `SERVICE_CONFIG_FAILURE_ACTIONS_FLAG` | **缺** | `ServiceInstaller.cs:522-561` 只设 level=2 | 静态源码 |
| 5 | 表单无 FMax / Intel 电压 | **部分**（断在 Desktop 表单） | `MainViewModel.cs:1091-1221` | 静态源码 |
| 6 | `TriggerKinds` 漏 `ProcessExited` | **部分**（断在 UI 下拉） | `MainViewModel.cs:162-167` | 静态源码 |
| 7 | `MainForm` 未订阅 `HistoryChanged` | **部分**（断在视图消费） | `MainForm.cs:727-743` 订阅 11 项，无此事件 | 静态源码 |
| 8 | 缺 EcoQoS、未拦 RealTime | **缺**（两处，且回读不可判定） | `AffinityScheduler.cs:158-167` | 静态源码 |
| 9 | 功耗墙未进 `StateLedger` | **部分**（有回读无回退） | `AgentSession.cs:356-401` 直调 provider | 静态源码 |

## 二、逐项明细

**1. 进程页（A-07 / D-20 / D-21）**

旧证据：`Zentwik.UI/ViewModels/ProcessViewModel.cs`（2236 行）——`RunningProcesses/FilteredProcesses:20-21`、9 个命令 `:122-130`、`ApplyProcessSnapshot:1128`、`ApplyBatchToSelected:1275`、`SaveRulesToPlanAsync:1465`；`ProcessRefresher.cs`（661 行）——Toolhelp 快照 + `QueryFullProcessImageName` 路径缓存 + `GetProcessTimes` 30 秒滑窗（`CpuWindowSeconds:178`、`MaxWindowPoints:64`）；`ProcessView.xaml.cs:17` 3 秒自动刷新，仅页面可见时启停。旧版 UI 与 Worker 同机直调，**无 op 层**。

新证据（已有，不重写）：`AffinityScheduler` 的 `Find/TryGetStartTicks/TryRead/TryApply`（`:18-177`）与 CPU Sets 读写（`:258-329`）；`PlanCoordinator.ApplyScheduling:1305-1497` 逐 PID 读写 + 账本 + 回读；`SchedulingWatcher` 2s/空闲 20s 持续纠偏（`:15,44`）；`NativeProcessList.Capture/Count`（`:46,92`）**仅回 PID+名字且是 internal**，全仓只有两处内部调用。

缺口分四块，范围需用户定：①只读枚举 op（PID/名字/路径/基础优先级/亲和性/CPU%）——需要新增平台层公开抽象，现有 `NativeProcessList` 不含路径与 CPU%；②单次手动下发 op，或强制走“建临时 SchedulingPlan”（旧版是直接下发且可存成规则）；③Desktop 表格与刷新按钮；④掩码库/拓扑勾选（A-08，P2）。

**必须先定的冲突**：CPU% 只能逐 PID `OpenProcess(GetProcessTimes)`，与 `AI_WORK_RULES §4` 红线 1「禁止逐进程 `OpenProcess` 做全表枚举」字面冲突。红线原文（`AI_WORK_RULES.md:59`）：「名字用 `CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS)` 一次快照；创建时间只对账本里的 PID 单查」。旧版做法是 3 秒一次 + 30 秒滑窗 + 路径缓存退避。沿用旧口径还是降级为“不显示 CPU%”，请用户定。

**2. 活动电源方案切换（C-07）**

旧证据：`PowerPolicyService.SetActiveScheme:448-459`（校验非空 → 存在性 → API）；`PowerViewModel.SwitchSchemeAsync:1328-1376`（失败回退选中项、成功后整表重读）；`PipeServerService.RunSetActivePowerScheme:2472-2510`（裸 GUID payload + `plans.MarkActive` 记账）。旧注释另记「同一方案再激活一次才会重载」。

新证据：`Win32PowerPolicyService.TrySetActiveScheme:119-124` **已实现**，但只被两处内部调用——`PlanCoordinator.cs:1297`（写值后重激活同一方案）、`:1656`（回退 `power.active`）。`PowerSchemeService.SwitchAway:415-420` 私有。`Ops` 无对应项，CLI `power` 七个子命令无 activate，电源页无按钮。

缺口：op 常量、`AgentSession` 分派、`AgentConnection` 方法、VM 命令、按钮（需三语键）、CLI 子命令。**附带事实**：`power.active` 的 `ReadTarget:1749-1752` 与 `RestoreTarget:1649-1659` 分支已写好但**无人 Acquire**，属预留死分支——接入 activate 时可直接复用，回退链路不必新写。

**3. 单实例（H-14）**

旧证据：`Zentwik.UI/Services/SingleInstanceGate.cs`——`Local\Zentwik.UI.SingleInstance` 互斥体 + 命名管道把“显示窗口”转给首实例后退出，按会话隔离。

新证据：`Desktop/Program.cs:9-26` 无互斥体；托盘 `MainForm.cs:1725` 与“关闭留托盘”`:1742-1755` 已启用 → 多开即多窗口多托盘图标。

缺口：`Mutex` + 二次实例通知 + 主窗口激活。注意 `MainForm(viewModel, catalog, autoConnect: false)` 是冒烟测试构造路径，闸门不能放在会阻塞测试的位置。

**4. 服务恢复 Flag（G-03 / J-22）**

旧证据：**无**——旧版是每用户提权 Worker，无 SCM 宿主，此项无旧实现可对照，属纯文档要求。

新证据：`ServiceInstaller.cs:23-24` 常量只有 `1/2`，`ApplyRecoveryActions:522-561` 只调 level=2 的 5s/15s/60s；`WindowsServiceHost.cs:121-126` 已上报 `1066`。

缺口：新增 level=4 常量与 `BOOL TRUE` 写入。**文档明确该设置“下次系统启动才生效”**，改完本机行为不会立即变化，验证需重启机器。代码可改；重装/刷新服务需另行申请，本轮不执行。

**5. 表单缺 FMax / Intel 电压（B-13 / A-05）**

旧证据：`TuningView.xaml:160-747`（FMax 滑块 + Intel 电压/8 列睿频）。

新证据（全链路已通，只差表单）：字段在 `Plans.cs:42,45`；CLI `--fmax/--intel-voltage` 在 `Commands.cs:785-820`；摘要显示在 `Plans.cs:295-303`；Intel 范围校验在 `AgentSession.cs:832`；写入与回读在 `PlanCoordinator.cs:1087,1142`；FMax 钳制语义已由 `2d9351d` 定为“不拒绝、应用时钳到 1000–7000”。断点仅在 `MainViewModel.cs:1091-1221`：只解析 `default/ccdN/N=`，且 `:1162` 要求至少一个 CO 值。

缺口：两个输入框 + 解析（Intel mV 越界交给后端拒绝，不在 UI 重复判）+ 三语键 + 表单行。后端有单测（`PlanAndSceneTests.cs:1001+`、`:592`），UI 无。

**6. 漏 ProcessExited（F-03 / A-10）**

旧证据：`ScenarioService.cs:40,235-260`、`ScenarioMonitor.cs:253-284`。

新证据：后端全通——`Scene.cs:14` 常量、`SceneEngine.cs:194,237,502`（边沿 + `_latched` 保持）、`AgentSession.cs:889-903`（校验与参数必填）、CLI `Help.cs:40`。仅 `MainViewModel.cs:162-167` 三项。

缺口：加一项常量与下拉。现 `CreateSceneAsync:1310-1314` 的“manual 才允许空参数”逻辑对 processExited 同样适用，无需改动。

**7. 未订阅 HistoryChanged（E-20 / A-04）**

旧证据：无监控页（旧版只有调度页 3 秒进程采样）。

新证据：`MainViewModel.cs:134` 声明事件、`:1489-1495` 每帧 `Invoke`；`MainForm.cs:727-743` 订阅了 11 个 `CollectionChanged`，**没有** `HistoryChanged`；监控页 4 个指标只有 CPU/内存/进程数/核数（`:308-354`）；历史表格 6 列已含温度/功耗（`:945`）。

数据侧已齐：`TelemetryDto` 有 `TemperatureCelsius/PackagePowerWatts/CoreClocks/ProcessorUtilization`（`Dtos.cs:25-36`），传感器默认 1 秒一拍（`UserSettingsRules:44`），逐核频率随传感器节拍读取（`TelemetryHub.cs:84,135`），无需额外开关。

**缺口与硬约束冲突**：订阅本身一行即可，但“折线图”与「无自绘」冲突。可行形态只有 DataGridView 追加行 / Label 数值 / 系统 `ProgressBar`。形态需用户定。

**8. EcoQoS 与 RealTime（D-12）**

旧证据：`ProcessPriority.cs:59-69` 六档（含 `EcoQoS`，注释明写刻意不含 RealTime）；`:106-115` `ClassOf`（EcoQoS→IDLE）；`:193-239` `ApplyOne`——`SET_INFORMATION|QUERY_LIMITED_INFORMATION` 双权限，EcoQoS 额外 `SetProcessInformation(ProcessPowerThrottling=4, ExecutionSpeed=0x1)`；`:125-133` 把基础优先级 24/32 归 Normal。

新证据：`AffinityScheduler.cs:158-167` 直接 `Enum.TryParse<ProcessPriorityClass>`——`RealTime` 能解析通过**且会被真的写入**，`EcoQoS` 解析失败；写入只用 `process.PriorityClass`，无 `SetProcessInformation`。

缺口两点，另有第三点需用户拍板：
- 档位映射表（注意新约定是 `AboveNormal` 无空格，见 `PlanAndSceneTests.cs:484`，不能照搬旧版 `"Above Normal"`）；
- 显式拒绝 RealTime；
- **回读不可判定**：`TryRead` 只回 `PriorityClass.ToString()`，EcoQoS 与 Idle 读回来都是 `Idle`，账本会判 desired≠actual 而记失败。旧版同样只写不读。要进账本必须补 `GetProcessInformation(ProcessPowerThrottling)` 回读，或为 EcoQoS 定义等价判据。这是本项的主要设计决策。

**9. 功耗墙未进账本（C-12）**

旧证据：无（`348712a` 已删除）。

新证据：`AgentSession.cs:356-401` 直接 `provider.ApplyPowerLimits` 后回 DTO，全程不碰 `_ledger`；`PlanCoordinator` 的 `ReadTarget:1721-1762`、`ReapplyTarget:314`、`RestoreTarget:1585` 均无 `power.limit.*` 分支；且 `PlanCoordinator` 构造**不持有** `IPowerLimitProvider`（`:51-58` 只有 `_tuning/_intel/_power/_scheduler`）。

缺口：先定“载体”。三个候选：
- (a) 扩展 `PowerPlan` 加 limits 字段——改动最小，可直接复用 `plans.apply/revert` 与场景仲裁；
- (b) op 级账本（source=manual）——但 `Owner=User` 按不变量 5 不可被自动恢复，与“可回退”要分开表达；
- (c) 新建第四类对象——违背“三个独立方案”不变量，不建议。

另需给 `PlanCoordinator` 注入 provider；回退用 `ApplyPowerLimits(before)`，四态回读由 provider 内部给出。验证现状：provider 有单测（`PowerLimitReadBackTests`、`FakePowerLimitTests`），真机闭环是既有 9955HX 记录，**非本轮**。

## 三、需用户拍板的四件事（Deepseek 不自行决定）

1. **进程页范围**：只做只读枚举+展示，还是连单次下发与掩码库一起做？CPU% 是否沿用旧版“3 秒 + 30 秒滑窗”（与红线字面冲突，需豁免或降级）。
2. **监控页形态**：订阅后用表格追加行 / 数值标签 / `ProgressBar`，选哪一种（折线图受“无自绘”禁止）。
3. **EcoQoS 回读**：补 `GetProcessInformation` 回读，还是给 EcoQoS 单独定等价判据（读回 Idle 也判成功）。
4. **功耗墙载体**：选 (a) 扩 `PowerPlan` / (b) op 级账本 / 其他。

## 四、相邻但不在本轮九项内

`LocalizationCatalog.SupportedLanguages` 仍锁 `zh-CN`（H-06，审计标缺口）——与“界面先不处理 i18n 切换”不矛盾，但会影响以后开语言项，本轮不动。
