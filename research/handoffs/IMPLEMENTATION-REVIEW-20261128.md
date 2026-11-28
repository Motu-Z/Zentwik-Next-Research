# 九项「已有 vs 缺」清单独立复核报告

- **复核日期**：2026-11-28
- **被复核文件**：`research/handoffs/IMPLEMENTATION-INVENTORY-20261128.md`（`Motu-Z/Zentwik-Next-Research` `main @ 3196963`）
- **对照基线**：
  - 产品新分支：`Motu-Z/Zentwik` `feat/gui-protocol @ 2d9351d`（基于 `100e9c0` 叠加 `2d9351d` 补丁核对）
  - 产品旧分支：`Motu-Z/Zentwik` `main @ 727462601911609e6dacbe3cb8f144246829087a`
  - 正本规范：`project/AI_WORK_RULES.md`、`project/PROJECT_BRIEF.md`、`knowledge/INDEX.md`
  - 审计参考：`research/handoffs/LEGACY-VS-NEXT-AUDIT-20261128.md`、`research/audit/20261128/A.md`…`J.md`
- **验证级别声明**：本轮复核在 Linux 沙盒内完成，全部结论均为**静态源码审计**；所有既有 `9955HX` 实测记录均标注为「引用既有实测记录（非本轮验证）」，不计为本轮验证。

---

## 1. 复核总表

| 编号 | Deepseek 结论 | 复核结论 | 具体证据（文件:行） | 严重度 |
|---|---|---|---|---|
| 1 | `process.*` op = 0，进程页空骨架（缺）；CPU% 与 `AI_WORK_RULES.md:59` 红线 1 字面冲突 | **需修正 / 遗漏** | `Protocol.cs:20-67`；`MainForm.cs:660-693`；`NativeProcessList.cs:46,92`；`AI_WORK_RULES.md:59,74` | 高 |
| 2 | `power.scheme.activate`（缺）；`power.active` 预留死分支可直接复用 | **需修正** | `Win32PowerPolicyService.cs:119-124`；`PowerSchemeService.cs:415-420`；`PlanCoordinator.cs:1297,1647-1657,1747-1750` | 中 |
| 3 | Desktop 单实例互斥体（缺） | **一致** | `Desktop/Program.cs:9-26`；`MainForm.cs:1725,1742-1755` | 低 |
| 4 | `SERVICE_CONFIG_FAILURE_ACTIONS_FLAG`（缺） | **需修正（路径缺漏）** | 实际路径 `src/Zentwik.Agent/Service/ServiceInstaller.cs:23-24,522-561`；`WindowsServiceHost.cs:121-126`；`AI_WORK_RULES.md:92` | 低 |
| 5 | 表单无 FMax / Intel 电压（部分） | **需修正（行号漂移）** | `2d9351d` 下 `MainViewModel.cs` 新增 `:212-215`，`BuildPlanFromForm` 移至 `:1095-1225`（`:1166`）；`PlanCoordinator.cs:1086-1122`；`AgentSession.cs:831` | 低 |
| 6 | `TriggerKinds` 漏 `ProcessExited`（部分） | **需修正（轻微行号漂移）** | `MainViewModel.cs:162-167`（未漂移），`CreateSceneAsync` 在 `2d9351d` 移至 `:1314-1318`；`AgentSession.cs:889-908` | 低 |
| 7 | `MainForm` 未订阅 `HistoryChanged`（部分）；折线图与无自绘冲突 | **需修正 / 遗漏** | `MainViewModel.cs:134,1467-1488`（`2d9351d` 约 `:1497`）；`MainForm.cs:308-354,727-743`；`Dtos.cs:25-36` | 中 |
| 8 | 缺 `EcoQoS`、未拦 `RealTime`（缺，且回读不可判定） | **需修正（补充等价判据机制）** | `AffinityScheduler.cs:94-119,158-167`；`PlanCoordinator.cs:1410-1433`；`StateLedger.cs:171-181` | 中 |
| 9 | 功耗墙未进 `StateLedger`（部分，有回读无回退） | **需修正（载体约束分析）** | `AgentSession.cs:356-401`；`PlanCoordinator.cs:51-58,1585,1721-1762`；`PROJECT_BRIEF.md:7-15`；`AI_WORK_RULES.md:79-83` | 高 |

---

## 2. 逐项展开

- **项 1（进程页）— `需修正 / 遗漏`**：
  - `observed`：清单正确指出 `Protocol.cs:20-67` 无 `process.*` op、`MainForm.cs:660-693` 为空骨架、`NativeProcessList.Capture/Count`（`src/Zentwik.Platform/Windows/NativeProcessList.cs:46,92`）仅返回 `(ProcessId, Name)` 且为 `internal`。
  - `derived`：**遗漏风险**：清单未提及新项目调度写入失败仍缺指数退避重试（`AI_WORK_RULES.md:74` 要求 2 秒起翻倍、5 次为限、上限 60 秒；对应审计项 `D-15`），若进程页允许手动或规则下发，遇到受保护进程将触发高频重试；亦未提及 `D-02`（CPU Sets 与 Affinity 互斥切换未自动清理另一侧旧限制）。关于 CPU% 与红线 1 的冲突判断见 §3.1。
- **项 2（活动电源方案切换）— `需修正`**：
  - `observed`：`Win32PowerPolicyService.TrySetActiveScheme:119-124` 确为内部实现；`PlanCoordinator.cs:1647-1657`（`RestoreTarget`）与 `:1747-1750`（`ReadTarget`）确已写好 `"power.active"` 分支且当前无调用方 `Acquire`。
  - `derived`：**过度断言**：「接入 activate 时可直接复用，回退链路不必新写」不严谨。`PowerSchemeService`（`power.scheme.*`）属于直接系统电源方案 CRUD，全程不经过 `StateLedger`；若 `power.scheme.activate` 作为手动单次操作执行，需先明确是否进入账本（若以 `Owner=User` 入账，还需解决与自动恢复的关系），并非零改造自动复用 `PlanCoordinator` 死分支。
- **项 3（Desktop 单实例互斥体）— `一致`**：
  - `observed`：`Desktop/Program.cs:9-26` 无互斥体，且 `MainForm.cs:1725,1742-1755` 已启用托盘与关闭进托盘；结论一致（补充：`Agent/Program.cs` 控制台模式亦无互斥体，见 `H-14`）。
- **项 4（服务恢复 Flag）— `需修正`**：
  - `observed`：清单引用路径写为 `ServiceInstaller.cs:522-561`，缺失目录前缀；实际路径为 `src/Zentwik.Agent/Service/ServiceInstaller.cs:23-24,522-561`。其余结论与 `AI_WORK_RULES.md:92`（「下次系统启动才生效」）一致。
- **项 5（表单缺 FMax / Intel 电压）— `需修正（行号漂移）`**：
  - `observed`：`2d9351d` 在 `MainViewModel.cs:212-215` 新增 `T(key, args)` 本地化方法，导致 `BuildPlanFromForm` 行号由 `100e9c0` 的 `1091-1221` 漂移至 `1095-1225`（CO 必填校验移至 `:1166`），`PlanCoordinator.cs` FMax 写入段亦有 `+2` 行调整（`:1086-1122`）。`2d9351d` 已将 FMax 改为 `[1000, 7000]` 钳制写入（`FMaxRange.Clamp`），新增输入框与错误提示须全部走 `T("...")` 本地化键。
- **项 6（漏 `ProcessExited`）— `需修正（轻微行号漂移）`**：
  - `observed`：`MainViewModel.TriggerKinds` 仍在 `:162-167` 未漂移；但 `CreateSceneAsync` 在 `2d9351d` 漂移至 `:1314-1318`。后端 `AgentSession.cs:889-908` 校验完整，结论成立。
- **项 7（未订阅 `HistoryChanged`）— `需修正 / 遗漏`**：
  - `observed`：`MainViewModel.cs` 的 `ApplySample(TelemetryDto)`（`100e9c0:1467-1488`，`2d9351d` 约 `:1479-1500`）仅将 `_cpuPercent` 入队 `_history` 并触发 `HistoryChanged?.Invoke([.. _history])`（仅传 `IReadOnlyList<double>`）。
  - `derived`：**关键遗漏**：`ApplySample` 完全丢弃了 `TelemetryDto` 中的 `TemperatureCelsius`、`PackagePowerWatts`、`CoreClocks`、`ProcessorUtilization`！因此**仅靠 `MainForm` 订阅 `HistoryChanged` 无法在监控页展示温度、功耗和逐核频率/占用**，还必须同步在 `MainViewModel` 补齐对应属性与 `Raise` 通知。另 `MainForm.cs:733` 已订阅 `TelemetryHistory.CollectionChanged`。
- **项 8（`EcoQoS` 与 `RealTime`）— `需修正`**：
  - `observed`：`AffinityScheduler.cs:158-167` 确用 `Enum.TryParse<ProcessPriorityClass>` 且未拦截 `RealTime`；`TryRead:94-119` 仅返回 `process.PriorityClass.ToString()`。
  - `derived`：清单漏查了 `StateLedger.cs:171-181` 已自带 `Func<string, string, bool>? equivalent` 比较委托机制，且 `PlanCoordinator.cs:1418` 目前使用 `string.Equals(actualPriority, priorityToApply, StringComparison.Ordinal)` 硬比对。详见 §3.3。
- **项 9（功耗墙未进账本）— `需修正`**：
  - `observed`：`AgentSession.cs:356-401` 确实直调 `provider.ApplyPowerLimits`，`PlanCoordinator.cs:51-58` 构造函数确未注入 `IPowerLimitProvider`。验证级别标注正确（区分了单测与既有 9955HX 实测）。载体评估见 §3.4。

---

## 3. 四个待拍板项的独立意见（仅分析与倾向，不替用户裁决）

### 3.1 进程页 CPU% 与红线 1 冲突
- **原文核对（`observed`）**：`AI_WORK_RULES.md:59` 明确规定：「**禁止逐进程 `OpenProcess` 做全表枚举。** 名字用 `CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS)` 一次快照；创建时间只对账本里的 PID 单查（`OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION)` + `GetProcessTimes`）。」
- **是否违反（`derived`）**：若像旧版 `ProcessRefresher` 那样每 3 秒对快照里的全部数百个 PID 逐一 `OpenProcess` 调 `GetProcessTimes` 与 `QueryFullProcessImageName`，**确实违反**红线 1 的「禁止逐进程 `OpenProcess` 做全表枚举」及「只对账本里的 PID 单查」约束。
- **合规替代方案（不替用户选）**：
  1. **仅对账本/目标 PID 单查 CPU%（严格合规）**：全表用 `NativeProcessList.Capture()`（零 `OpenProcess`）列出 PID 与进程名；仅对 `StateLedger` 已追踪或当前选中的 PID 调用 `OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION) + GetProcessTimes` 展示 CPU%/优先级/亲和性，其余行 CPU% 留空或显示 `-`。
  2. **改用单次内核快照 `NtQuerySystemInformation(SystemProcessInformation=5)`（`derived`，需 Windows 验证）**：一次系统调用直接返回全表 PID、映像名、`KernelTime/UserTime` 与 `BasePriority`，全程无需逐 PID `OpenProcess`（但需在 Windows AOT 下实测结构体对齐）。
  3. **页面可见门控豁免**：仅在进程页打开时按 3s 间隔 + 路径缓存采集（需用户明确豁免红线 1）。
- **独立倾向**：若不愿豁免红线 1，倾向 **方案 1（全表快照 + 仅对账本/选中 PID 单查）**，改动最小且完全符合 `AI_WORK_RULES.md:59`；若必须全表展示 CPU%，倾向 **方案 2（`NtQuerySystemInformation`）** 以避免逐进程 `OpenProcess`。

### 3.2 监控页形态
- **口径核对（`observed`）**：`AUDIT-LEGACY-VS-NEXT-20261128.md §2.3` 明确限定控件集合为：`Label / Button / TextBox / ComboBox / CheckBox / DataGridView / GroupBox / TableLayoutPanel / SplitContainer`，且「无主题、无强调色、无缩放、无自绘、无动画、无第三方 UI 包」。
- **合规性结论（`derived`）**：
  - `Label` 数值行（在 `_metricBox` 的 `TableLayoutPanel` 中追加温度、封装功耗、逐核频率/占用文本）与 `DataGridView` 追加行（`_historyGrid` 实时刷新）**完全在明确列举的毛坯控件白名单内**。
  - `ProgressBar` 虽属标准系统控件（无自绘），但未出现在 §2.3 显式列举的 9 种控件名单中；折线图若通过 GDI+/自绘 `Paint` 实现则直接违反「无自绘」。
- **独立倾向**：倾向 **`Label` 数值行 + `DataGridView` 实时追加历史行**（同时补齐 `MainViewModel.ApplySample` 对温度/功耗/逐核频率的属性暴露），100% 落在白名单控件内，不引入 `ProgressBar` 或任何自绘争议。

### 3.3 `EcoQoS` 回读与 `StateLedger` 判定
- **源码核对（`observed`）**：`AffinityScheduler.TryRead:110` 仅读 `process.PriorityClass.ToString()`，确实无法区分 `Idle` 与 `EcoQoS`；`PlanCoordinator.cs:1418` 用 `string.Equals` 比对，`StateLedger.Commit:171-181` 已有 `equivalent` 委托。
- **可行性与阻塞判断（`derived`）**：
  - 补 `GetProcessInformation(ProcessPowerThrottling)`（约 20 行 P/Invoke）是完全可行的：回读时若 `PriorityClass == Idle` 且 `ControlMask & StateMask & ExecutionSpeed != 0` 则返回 `"EcoQoS"`，可精准发现外部解除 `EcoQoS` 的漂移。
  - 若用 `StateLedger.Commit` 的 `equivalent` 判据（`actual == "Idle" && desired == "EcoQoS"` 视为一致），实现更简，但**无法识别外部把 `EcoQoS` 清回普通 `Idle` 的漂移**。
  - **是否阻塞实施**：**不阻塞**。禁止 `RealTime` 与 `SetProcessInformation(ExecutionSpeed)` 写入可立即实施，回读策略仅影响 `TryRead` 是否补 1 个 Win32 查询函数。
- **独立倾向**：倾向 **直接补 `GetProcessInformation(ProcessPowerThrottling)` 回读**，使 `TryRead` 语义与 `TryApply` 对称，避免引入「外部取消 EcoQoS 却无法检出漂移」的盲区。

### 3.4 功耗墙载体
- **规范核对（`observed`）**：`PROJECT_BRIEF.md:7` 将「电压/频率/**功耗**/核心分组」归入 **CPU Tuning**；`PROJECT_BRIEF.md:8` 将 **Power Management** 定义为 Windows 电源计划与扩展参数；`PROJECT_BRIEF.md:14` 与 `AI_WORK_RULES.md:79-80` 规定「CPU 调校方案、电源方案、调度方案是三个独立领域对象…场景不是第四种方案」。
- **逐项评估（`derived`）**：
  - **(a) 扩展 `PowerPlan` 加 limits 字段**：不新增第四类对象，但把 AMD SMU 硬件功耗墙（PPT/TDC/EDC）混入 Windows OS 电源计划（`powercfg`）领域，且 `PlanCoordinator.ApplyPower` 需额外耦合硬件 SMU provider，领域边界略混杂。
  - **(b) op 级账本（`source="manual"`）**：`StateLedger` 的 `Owner=User` 保护不被场景覆盖（不变量 5），但仍可单独提供手动回退入口；缺点是无法直接参与场景自动切换与退出回退。
  - **(c) 新建第四类对象**：**直接违背**「三个独立方案」不变量，不可选。
  - **补充第四视角（供用户参考，属 `CpuTuningPlan` 归属）**：按 `PROJECT_BRIEF.md:7` 原文，功耗本就属于 CPU Tuning 领域，将功耗墙作为 `CpuTuningPlan` 的可选字段（并允许 `hardware.power.limit.apply` 独立走 `Owner=User` 入账或走 `plans.apply`）在领域边界上比塞进 `PowerPlan` 更自洽。
- **独立倾向**：若仅在清单所列 (a)/(b)/(c) 中二选一（排除违背不变量的 c），且用户需要场景联动与退出自动回退，倾向 **(a) 或并入 `CpuTuningPlan`**（后者更贴合 `PROJECT_BRIEF.md:7`）；若功耗墙仅作手动一次性调节、不随场景切换，则 **(b) op 级账本** 侵入性最小。

---

## 4. 遗漏清单（审计报告已指出但 Deepseek 清单未覆盖的相关缺口/风险）

1. **进程调度写入失败无指数退避重试（`D-15` / `AI_WORK_RULES.md:74`）**：`PlanCoordinator` 对受保护进程每 2 秒无限重试，缺少旧版 `ScheduleRetryLedger` 的 2s→60s 退避与 5 次上限。
2. **CPU Sets 与 Affinity 互斥切换未自动清理旧限制（`D-02` / `AI_WORK_RULES.md:73`）**：规则类型切换时未先清除另一侧已有约束。
3. **监控页 `ApplySample` 未透传温度/功耗/逐核频率（`E-20` / `A-04`）**：即便 `MainForm` 订阅 `HistoryChanged`，`MainViewModel.ApplySample` 当前也未把 `TelemetryDto` 的温功与逐核数据暴露给视图。
4. **设置页采样间隔与多语言缺口（`H-06` / `H-08` / `H-15`）**：`PersistSettingsAsync` 未传 `MonitorIntervalMs`/`SensorIntervalMs`，设置页无对应控件；`SupportedLanguages` 锁死 `["zh-CN"]` 属缺口（清单仅在第四节顺带提及 `H-06`）。
5. **AMD 功耗墙仅支持部分，Intel 功耗墙与 EC 明确排除（`C-16`）**：UI/账本接入时需保留平台不支持时的 `Skipped` 降级语义。

---

## 5. 计数闭合

- 清单总项数：**9 项**（`#1` ~ `#9`）
- 本次复核覆盖：**9 项**（`#1` ~ `#9` 全覆盖，0 漏项）
- 复核结论分布：一致 **1** 项（`#3`）、需修正 **6** 项（`#2`, `#4`, `#5`, `#6`, `#8`, `#9`）、需修正 / 遗漏 **2** 项（`#1`, `#7`）、证据不足 **0** 项。
- 四个待拍板项覆盖：**4 / 4**（全部给出正本依据、可行方案与独立倾向，均未替用户裁决）。
