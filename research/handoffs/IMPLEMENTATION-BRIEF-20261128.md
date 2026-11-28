# 实施预研简报：功耗墙并入 `CpuTuningPlan` 与进程页 CPU%（静态源码输入）

- **日期**：2026-11-28
- **目标读者**：Windows 侧实施 AI（Deepseek）
- **参考基线**：
  - 研究仓库：`Motu-Z/Zentwik-Next-Research` `main @ a7f8180`（`project/AI_WORK_RULES.md`、`project/PROJECT_BRIEF.md`、`research/handoffs/LEGACY-VS-NEXT-AUDIT-20261128.md`、`research/handoffs/IMPLEMENTATION-REVIEW-20261128.md`）
  - 产品新分支：`Motu-Z/Zentwik` `feat/gui-protocol @ 2d9351d`（基于 `100e9c0` 叠加 `2d9351d` 补丁核对）
  - 产品旧分支：`Motu-Z/Zentwik` `main @ 727462601911609e6dacbe3cb8f144246829087a`
- **定位声明**：本简报仅提供静态源码对照与改动点输入，**不裁决未决设计选项、不修改产品代码、不执行构建/测试**。

---

## 一、两个设计点四字段对照总表

| 维度 | 设计点 A：功耗墙并入 `CpuTuningPlan` | 设计点 B：进程页 CPU% 合规等价实现 |
|---|---|---|
| **1. 旧实现参照** | **`[旧项目亦无]`**：`7274626` 全仓检索 `PowerLimit` / `FastLimit` / `SlowLimit` / `TdcLimit` / `EdcLimit` / `0x56` / `0xCB` 命中 0 条（`348712a` 已删除），无旧版功耗墙账本或回退代码可参照。 | `ProcessView.xaml.cs:17,32-36`（可见时 `3s` 定时器）；`ProcessRefresher.cs:178-189,423-555,609-659`（Toolhelp 快照 + 每 PID `OpenProcess(GetProcessTimes)` + `30s/64点` 滑窗 + `QueryFullProcessImageNameW` 缓存与 `30s` 失败退避 + `PruneDeadSamples`）。 |
| **2. 新改动点（按 `2d9351d`）** | **7 处核心改动点**：① `Plans.cs:41-45` 扩 `CpuTuningPlan` 字段及 `:295-303` 摘要；② `AgentSession.cs:828-837` 校验 `PowerLimitBounds`；③ `Agent/Program.cs:107` 向 `PlanCoordinator` 注入 `IPowerLimitProvider`（`:88`）与模块启用判定；④ `PlanCoordinator.cs:51-65` 接构造参数；⑤ `ApplyCpu`（`:1140` 后）加 `cpu.power.limit.*` 读基线/写入/四态映射；⑥ `ReapplyTarget:314`、`RestoreTarget:1583`、`ReadTarget:1719` 补分支；⑦ 如需 UI 录入，扩展 `MainViewModel.BuildPlanFromForm:1095-1225` 与本地化键。 | **6 处核心改动点**：① `Native.cs:62-67` 已有 `NtQuerySystemInformation`，新增 `SystemProcessInformation=5` 常量与 `SYSTEM_PROCESS_INFORMATION` 布局；② 新建或扩展进程快照源（`Zentwik.Platform/Windows/`），按 `NextEntryOffset` 遍历拿 `PID/ImageName/KernelTime/UserTime/CreateTime/BasePriority`；③ `Zentwik.Core` 维护 `(Pid, CreateTime)` `30s/64点` 滑窗计算 `[0,100]` CPU%；④ 2–3s TTL 缓存避免双重开销；⑤ `Contracts/Protocol.cs` + `AgentSession` 暴露 `process.list`（及按需下发 op）；⑥ `MainForm.cs:660-693` 进程页毛坯 `DataGridView` 绑定。 |
| **3. 坑与注意事项** | ① `power-limits` 未启用或平台不匹配必须标 `LedgerStatus.Skipped`，禁止连带失败同方案 CO/FMax；② `Diverged` 与 `Accepted`（无回读通道）不可简单按 `Actual==Desired` 处理；③ **`PowerLimitCurrent` 无 `TdcAmps/EdcAmps` 实时读数**，这两项缺 `PreviousValue` 无法自动回退；④ 保留独立 `hardware.power.limit.apply` 时需防 `Owner=User`（不变量 5）与场景互相踩踏；⑤ OEM 切电源模式可能改写 DPTC 触发误漂移。 | ① **AOT 结构体对齐风险**：`SYSTEM_PROCESS_INFORMATION` 含 `nint`/`UNICODE_STRING` 填充，仅能 `blittable` 指针偏移遍历，且需在 Windows x64 实测；② `SystemProcessInformation=5` **不含完整 exe 路径**，若需路径仍要 `QueryFullProcessImageNameW`；③ 若退回照搬旧版，每 3s 约 150–800 次 `OpenProcess` 将直接违反 `AI_WORK_RULES.md:59` 红线 1；④ 须与 `SchedulingWatcher` 2s/20s 循环隔离。 |
| **4. 证据级别** | `observed`（旧仓 0 命中、`Plans.cs`、`PowerLimits.cs`、`PlanCoordinator.cs`、`Program.cs`、`AgentSession.cs`）；`derived`（OEM 电源模式改写 DPTC 漂移风险、跨目标影响推导）；`unknown`（真实 SMU `Diverged/Accepted` 在各机型/Session 0 下表现）。 | `observed`（`ProcessRefresher.cs`、`ProcessView.xaml.cs`、`Native.cs`、`ProcessorLoadReader.cs`、`SchedulingWatcher.cs`、`AI_WORK_RULES.md:59`）；`derived`（`SystemProcessInformation=5` 布局与 AOT 兼容推导）；`unknown`（目标 Windows x64 AOT 实测偏移与开销）。 |

---

## 二、设计点 A 详述：功耗墙并入 `CpuTuningPlan`

### 1. 旧实现参照
- **`[旧项目亦无]`**（`observed`）：在 `Motu-Z/Zentwik@main`（`7274626`）全仓 `.cs` 文件检索功耗墙与 RSMU 命令常量（`PowerLimit`、`FastLimit`、`SlowLimit`、`TdcLimit`、`EdcLimit`、`0x56`、`0xCB`）结果均为 **0 命中**（提交 `348712a` 已在基线前移除旧功耗墙代码）。旧项目不存在可参照的功耗墙账本、回读或回退逻辑。

### 2. 新改动点（按 `2d9351d`）
- **领域模型（`src/Zentwik.Domain/Plans.cs:41-45`、`295-303`）**：
  - 在 `CpuTuningPlan` 上增加可空字段（例如 `FastWatts`、`SlowWatts`、`TdcAmps`、`EdcAmps`，单位 W / A，或可空 `PowerLimitRequest?`），默认 `null` 表示该方案不改动功耗墙，保持对现有 JSON 状态文件的向后兼容（`AI_WORK_RULES.md:96`）。
  - `CurveOptimizerConfig.FormatPlanSummary`（`Plans.cs:295-303`）同步补充功耗墙摘要，供 `MainViewModel.BuildPlanRow` 与 CLI 列表展示。
- **契约与序列化（`src/Zentwik.Contracts/`）**：
  - 若直接加 4 个 `int?` 属性到 `CpuTuningPlan`，`ZentwikJson.cs` 已注册 `Plan` / `CpuTuningPlan` 多态源生成，**无需新增 `Ops` 常量**（直接复用 `plans.save` / `plans.apply` / `plans.revert`）；若使用嵌套 record，需确认 `ZentwikJson` 已包含 `PowerLimitRequest`。
- **入参校验（`src/Zentwik.Agent/AgentSession.cs:828-837`）**：
  - 在 `ValidatePlan` 的 `CpuTuningPlan` 分支调用 `PowerLimitBounds.Validate(limits)`（`src/Zentwik.Domain/Power/PowerLimits.cs:79`），范围非法时抛 `CoreException(ErrorCodes.PlanInvalid, ...)`。
  - **关键约束**：不得在 `ValidatePlan` 校验 `power-limits` 模块是否启用或当前是否为 AMD 平台，否则用户将无法在模块停用时保存方案（须与 `FMaxLimitMhz` / `IntelVoltageOffsetMillivolts` 一致，延后到 `ApplyCpu` 按 `Skipped` 处理）。
- **依赖注入链（`src/Zentwik.Agent/Program.cs:88,107` → `src/Zentwik.Core/Plans/PlanCoordinator.cs:51-65`）**：
  - 当前 `Program.cs:88` 已构造 `IPowerLimitProvider powerLimits`，但 `:107` 构造 `PlanCoordinator` 时未传入。
  - 须给 `PlanCoordinator` 构造函数追加可选参数 `IPowerLimitProvider? powerLimits = null, Func<bool>? isPowerLimitsEnabled = null`（保持既有单测调用兼容），并在 `Program.cs:107` 接入。
- **协调器四处链路（`PlanCoordinator.cs`）**：
  - **账本目标名**：建议使用 `cpu.power.limit.fast`、`cpu.power.limit.slow`、`cpu.power.limit.tdc`、`cpu.power.limit.edc`（对应单位 W / W / A / A）。
  - **Apply（`ApplyCpu` `:1140` 之后）**：检查 `isPowerLimitsEnabled?.Invoke() == true` 且 `_powerLimits is not null` 且 `ProbePowerLimit()` 可用；不满足时输出 `LedgerStatus.Skipped`（不写入账本、不阻断方案内 CO/FMax）。可用时先 `ReadPowerLimits()` 获取 `Before`，再调 `ApplyPowerLimits(limits)` 并按 4 项分别入账。
  - **Reapply（`ReapplyTarget:314-359`）**、**Restore（`RestoreTarget:1583-1633`）**、**Read（`ReadTarget:1719-1760`）**：补充 `cpu.power.limit.*` 分支；注意 `RestoreTarget` 必须由 `LedgerEntry.PreviousValue` 解析整数后构造单项 `PowerLimitRequest` 调用 `ApplyPowerLimits`。
- **端到端调用链**：`Desktop/CLI (plans.apply/revert)` → `AgentSession` → `PlanCoordinator.ApplyCpu / Revert` → `IPowerLimitProvider.ReadPowerLimits / ApplyPowerLimits` → `StateLedger.Acquire / Commit / Release`。

### 3. 坑与注意事项
1. **可选模块与平台不匹配降级**：`power-limits` 是全仓唯一可选模块（`ModuleCatalog.cs`）。模块停用或非 AMD 平台时，功耗墙目标必须标 `LedgerStatus.Skipped`，**绝不能**像 `needsAmd` 的 CO 那样提前整单返回失败，影响同方案内其他已配置目标（参考 `PlanCoordinator.cs:1130-1134` FMax 的隔离写法）。
2. **`TdcAmps` / `EdcAmps` 缺乏实时回读前值（重要硬坑，`observed`）**：`PowerLimitCurrent`（`PowerLimits.cs:19`）仅读取 PM 表的 `StapmWatts`、`FastWatts`、`SlowWatts`，**没有 `TdcAmps` 与 `EdcAmps` 的当前生效值**（`FusedPowerLimits` 是出厂熔断上限，不是当前值）。因此 TDC/EDC 写入前拿不到 `PreviousValue`，退出或 `plans.revert` 时 `RestoreTarget` 会命中 `PreviousValue is null`（`PlanCoordinator.cs:1585`）导致无法自动回退，必须明确告知用户或记录 `Restorable = false`。
3. **四态回读与账本状态映射（`PowerLimits.cs:38-51`）**：
   - `Applied` → `_ledger.Commit`；
   - `Rejected` → `_ledger.Fail`；
   - `Diverged`（SMU 接受但固件折算为另一数值）→ 若用默认精确比较会判 `Fail`，需按 `AI_WORK_RULES.md:81`（`Actual ≠ Desired` 即未生效）保持 `Failed` 还是定义容差；
   - `Accepted`（无可用回读通道，`Actual == null`）→ 必须走 `_ledger.MarkUnknown`（`AI_WORK_RULES.md:84`）。
4. **`Owner=User` 冲突**：若继续保留独立 `hardware.power.limit.apply` op，需注意 `AI_WORK_RULES.md:83`（不变量 5）：`Owner=User` 的条目不得被场景覆盖；反之若保留独立直写不进账本，则仍存在双写互相覆盖风险。
5. **场景联动边界**：电源方案切换（Windows `powercfg`）与 `CpuTuningPlan`（SMU DPTC）在部分笔记本固件下可能存在底层联动（切换系统电源模式会重置 PPT），唤醒或切方案后 `DetectDriftAsync` 可能观测到外部漂移。

### 4. 验证范围限制
- Linux 沙盒仅能完成静态源码核对；Windows 侧可用 `FakePowerLimitProvider`（`src/Zentwik.Platform/Fakes/FakeHardware.cs:581`）覆盖 `PlanCoordinator` 的 `Apply/Reapply/Revert/Restore/Skipped` 单测；**真实 SMU DPTC 写前基线、四态回读与 Session 0 行为均无法通过 fake 验证**，须在 Windows 管理员会话与 Session 0 真机闭环。

---

## 三、设计点 B 详述：进程页 CPU% 的合规等价实现

### 1. 旧实现参照（`7274626`）
- **刷新入口与节流（`Zentwik.UI/Views/ProcessView.xaml.cs:17,32-36`）**：`AutoRefreshSeconds = 3`，使用 `DispatcherTimer(DispatcherPriority.Background)`，仅在 `Loaded`~`Unloaded`（页面可见）期间启停。
- **采集与滑窗（`Zentwik.Core/Services/ProcessRefresher.cs`）**：
  - `Collect()`（`:423-479`）：单次 `CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0)` 遍历进程取 `PID`、`ExeFile`、`pcPriClassBase`；
  - `ReadProcessTimes()`（`:609-629`）：逐 PID 调 `OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION, false, pid)` + `GetProcessTimes` 取 `KernelTicks + UserTicks`（单位 100 ns FILETIME）；
  - `ComputeCpu()`（`:178-189,491-523`）：维护 `CpuWindowSeconds = 30.0`、`MaxWindowPoints = 64` 的队列，拿窗口**最老点**与当前点做差：
    `pct = TimeSpan.FromTicks(cpuTicks).TotalSeconds / elapsedSeconds / ProcessorCount * 100.0`，钳制 `[0, 100]`，避免 3s 采样间隔抖动引起列表按 CPU% 排序时频繁跳位；
  - `ResolvePath()`（`:184,536-555,632-659`）：`OpenProcess` + `QueryFullProcessImageNameW`，成功后缓存 `PathResolved = true`，失败按 `PathRetrySeconds = 30.0` 退避；
  - `PruneDeadSamples()`（`:563-596`）：清理已退出 PID，同时天然隔离 PID 复用。

### 2. 合规路径可行性（`NtQuerySystemInformation(SystemProcessInformation = 5)`）
- **现有可复用基础（`observed`）**：`src/Zentwik.Platform/Windows/Native.cs:62-67` 已声明源生成 `[LibraryImport("ntdll.dll")] NtQuerySystemInformation`，且 `ProcessorLoadReader.cs:56-60` 已在 NativeAOT 下稳定调用 `SystemProcessorPerformanceInformation = 8` 并处理 `StatusInfoLengthMismatch (0xC0000004)`。
- **一次系统调用拿全表时间片（`derived`）**：
  - 传入 `SystemProcessInformation = 5` 可在**零 `OpenProcess`** 前提下，通过一次系统调用返回变长 `SYSTEM_PROCESS_INFORMATION` 链表（按 `NextEntryOffset` 步进），直接获取全表每个进程的：`UniqueProcessId`、`ImageName`（`UNICODE_STRING`）、`KernelTime`（100 ns）、`UserTime`（100 ns）、`CreateTime`（100 ns，可用于 PID 复用识别）、`BasePriority`、`NumberOfThreads`。
- **NativeAOT 下的实现与对齐注意事项（`derived`）**：
  - 严禁反射封送结构体；使用 `NativeMemory.Alloc` 分配非托管缓冲区，遇 `0xC0000004` 按 `returnLength + 余量` 扩容重试，遍历时使用 `blittable` `struct` 指针按 `NextEntryOffset` 跳转；
  - **64 位内存布局对齐**：x64 Windows 上 `SYSTEM_PROCESS_INFORMATION` 中 `ImageName`（`UNICODE_STRING`：2B `Length` + 2B `MaximumLength` + **4B 对齐填充** + 8B `Buffer` 指针 = 16B）、`UniqueProcessId` 为 `nint`（8B）、`CreateTime`/`UserTime`/`KernelTime` 为 `long`（8B）；`ImageName.Buffer` 指向同块内核缓冲区内偏移，须在进程内立即 `new string((char*)buffer, 0, length / 2)` 拷贝出托管字符串再释放缓冲区；
  - **CPU% 计算等价性**：`KernelTime + UserTime` 与旧版 `GetProcessTimes` 同为 100 ns 单位，沿用 `(Pid, CreateTime)` 为键的 `30s / 64 点` 滑窗与 `ProcessorCount` 归一化公式，即可得到与旧版一致的 CPU% 读数与排序稳定性；
  - **与 `NativeProcessList` 配对**：`SystemProcessInformation=5` 本身已含 `UniqueProcessId` 与 `ImageName`（仅 Idle PID 0 的 `ImageName` 为空），可独立成表；若仍保留 `NativeProcessList.Capture()` 作为轻量底座，亦可通过 `PID` 字典配对时间片。
- **缺口边界**：`SystemProcessInformation=5` **不包含完整可执行文件路径**；若进程页要求展示完整路径，仍需按需或带缓存退避调用 `QueryFullProcessImageNameW`。

### 3. 若退回照搬旧版的红线冲突范围（不替用户裁决）
- `AI_WORK_RULES.md:59` 红线 1 原文：「**禁止逐进程 `OpenProcess` 做全表枚举。** 名字用 `CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS)` 一次快照；创建时间只对账本里的 PID 单查（`OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION)` + `GetProcessTimes`）。」
- **冲突范围（`observed`）**：若在进程页照搬旧版 `ProcessRefresher.Collect`，每 3 秒对快照内全部存活进程（常规桌面约 150–400 个 PID）逐一执行 `OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION)` 调 `GetProcessTimes`（每轮 150–400 次句柄开闭），叠加首次及失败每 30 秒重试的 `QueryFullProcessImageNameW`，单轮可达 150–800 次 `OpenProcess`，直接违反「禁止逐进程 `OpenProcess` 做全表枚举」与「只对账本里的 PID 单查」。

### 4. 刷新节奏共存与双重开销规避
- `SchedulingWatcher.cs:16,37` 后台固定以 **2 秒**（有活动场景/账本）或 **20 秒**（空闲）驱动 `PlanCoordinator`；其中调度巡检应继续使用 `NativeProcessList.Capture()`（零 `OpenProcess`）+ 仅对匹配/账本内 PID 单查 `TryGetStartTicks` / `TryRead`。
- 进程页全表采样应采用**客户端进程页可见时按 3 秒请求 + Agent 侧 2 秒 TTL 缓存**（参照旧版 `ProcessRefresher.TryGetSnapshot(TimeSpan maxAge)` `ProcessRefresher.cs:203`），避免多客户端或连续刷新触发重复全表采集，并确保无人打开进程页时零额外 CPU% 采集开销。

---

## 四、Deepseek 实施时需特别注意的 5 条

1. **禁止将旧版 `OpenProcess` 全表轮询悄悄塞入 `SchedulingWatcher` 2 秒后台循环**：即使进程页最终获准豁免采用旧版采集，也只能像旧版 `ProcessView.xaml.cs` 那样仅在进程页可见时按需触发并加 2 秒 TTL 缓存，不得污染常驻调度基线（`0.052%` 单核）。
2. **`PlanCoordinator` 构造与 `power-limits` 模块门控必须解耦降级**：注入 `IPowerLimitProvider` 的同时必须注入模块启用状态；`power-limits` 未启用、Intel 平台或 `TdcAmps/EdcAmps` 无回读前值时，仅将对应功耗墙目标记为 `Skipped` 或 `Restorable=false`，严禁阻断同方案内 CO/FMax 写入。
3. **`SYSTEM_PROCESS_INFORMATION` 结构体偏移与 AOT 指针遍历必须在 Windows x64 真机实测**：Linux 沙盒与 Fake 测试无法发现 4 字节对齐填充或 `UNICODE_STRING` 指针错误，任何偏移误差都会导致 AOT 运行时读到乱码或内存越界。
4. **严格遵守 WinForms 毛坯与本地化红线**：`2d9351d` 已将桌面文案全量接入 `Zentwik.Localization`（`MainViewModel.T(...)` 与 `MainForm.AddHeaders`），本次新增的进程页列头、报错提示与功耗墙摘要字段必须同步补齐 `zh-CN` / `zh-TW` / `en-US` 三语键，且仅使用允许的裸控件（不加主题、自绘、动画、副标题）。
5. **诚实标注验证级别**：`FakePowerLimitProvider` 与进程 CPU% 滑窗计算可写纯单测覆盖，但 `NtQuerySystemInformation(5)` 结构体解析、`power-limits` 在 `PlanCoordinator` 下的四态真实写回与回退、`LocalSystem` Session 0 权限边界均不得仅凭单测宣称「真机闭环」。
