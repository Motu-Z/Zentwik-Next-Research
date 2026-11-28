# 下一批待办实施预研简报（A/B/C 三项）

> **基线**：产品 `Motu-Z/Zentwik` `feat/gui-protocol` @ `61d5e6e586cbc060d97bab34285e8dfda0cc86ff`（本地静态源码基线 `100e9c0`，并已核对 `2d9351d`/`40c049a`/`74df31a`/`61d5e6e` 变更）；旧版对照 `Motu-Z/Zentwik` `main` @ `727462601911609e6dacbe3cb8f144246829087a`；研究基线 `Motu-Z/Zentwik-Next-Research` `main` @ `217afe8`。  
> **性质**：纯只读静态源码预研（Linux 沙盒未跑构建/测试），不改产品代码、不替用户裁决。所有结论均标 `observed` / `derived` / `unknown`，无证据标 `[未找到]`。

---

## 三项对照总表

| 编号 | 待办项 | 优先级 | 现状与断点 | 最小改动点数 | 核心坑点 | 本轮验证级别 |
|---|---|---|---|---:|---|---|
| **A** | `power.scheme.activate` | **P0** | `Win32PowerPolicyService.TrySetActiveScheme` 与 `PlanCoordinator` 中 `power.active` 读/回退分支已就位，缺 `Ops`/`AgentSession`/`PowerSchemeService`/CLI/UI 接线 | **9 处**（跨 `Contracts`/`Core`/`Agent`/`Cli`/`Desktop`/`Localization`） | 隐藏方案激活后变为临时可见；与 `StateLedger` `power.active` 所有权/用户接管边界；写策略后需 `RefreshActiveScheme` | 静态源码审计（`Core`/`Contracts` 可 fake 单测，Win32 切方案需真机） |
| **B** | Desktop 表单 FMax / Intel 电压输入 | **P0** | 后端 `Plans.cs` / `AgentSession` / `PlanCoordinator` / CLI `--fmax` `--intel-voltage` 全通；断点在 `MainViewModel.BuildPlanFromForm` 仅解析 CO / GUID / 调度串 | **5 处**（`MainForm`/`MainViewModel`/三语/`Desktop.Tests`） | `MainForm` 必须用 WinForms 原生毛坯控件；FMax `[1000,7000]` 为工具支持范围非固件限制；Intel mV 范围 `[-150,+100]` 交后端校验 | 静态源码审计（`BuildPlanFromForm` 与三语/烟雾测试可单测，写入需真机） |
| **C** | Desktop 单实例互斥体 | **P1** | `Desktop/Program.cs` 无互斥体与激活管道；已启用 `NotifyIcon` 与「关闭留托盘」，多开会冲突 | **5 处**（`Desktop` 单实例门/`Program.cs`/`MainForm.cs`/托盘退出） | 闸门必须在 `SetSynchronizationContext` 之后、`new MainViewModel` 之前；`MainFormSmokeTests` (`autoConnect:false`) 不走 `Program.Main` 但不可被静态构造拦截；`LocalSystem` 服务与桌面用户跨会话隔离 | 静态源码审计（纯逻辑可单测，多开抢锁/管道唤醒/托盘交互需 Windows 实测） |

---

## A. `power.scheme.activate`（P0）

### 1) 旧/新实现参照（`observed`）
- **旧版参照**：
  - `Zentwik.Core/Services/PowerPolicyService.cs:409-419,448-457`：`SetActiveScheme(Guid)` 先核对 `SchemeExists` 再调 `_api.SetActiveScheme(guid)`；`RefreshActiveScheme()` 在每次 `WriteCore` 写入 AC/DC 索引后重新调用一次 `PowerSetActiveScheme(active.Value)`。
  - `Zentwik.Core/Services/PowerPolicyService.cs:18-21`：明确解释为何必须在写策略后调用 `RefreshActiveScheme()`——`PowerWriteACValueIndex` 只改注册表存储值，电源管理器不会立刻按新值调度，必须再次调用 `PowerSetActiveScheme` 触发电源管理器重载。
- **新版现状**：
  - 平台层已实现：`src/Zentwik.Platform/Windows/Win32PowerPolicyService.cs:119-124` 已实现 `TrySetActiveScheme(Guid scheme, out string? error)`（调用 `Native.PowerSetActiveScheme`）。
  - 领域/账本死分支已就位：`src/Zentwik.Core/Plans/PlanCoordinator.cs:1647-1657`（`RestoreTarget` 中 `"power.active"` 分支解析 `entry.PreviousValue` 为 `Guid` 并调 `_power.TrySetActiveScheme`）与 `:1747-1750`（`ReadTarget` 中 `"power.active"` 分支返回 `_power.GetActiveScheme()?.Guid.ToString("D")`），但当前无任何调用方对 `"power.active"` 执行 `_ledger.Acquire(...)`。
  - `PlanCoordinator.ApplyPower`（`:1226-1228`）刻意注释：`指定 BaseSchemeGuid 只表示「改哪个方案」，不隐含「切过去」`。
  - `PowerSchemeService.cs:415-420` 仅在 `SwitchAway(exclude)` 内部调用 `power.TrySetActiveScheme`。
  - `Protocol.cs:49-55` 仅有 `power.scheme.list / create / rename / delete / backup / backups / restore`，无 `power.scheme.activate`。

### 2) 最小改动点（共 9 处，逐跳链路）
1. **协议常量（`src/Zentwik.Contracts/Protocol.cs:49-55`）**：在 `Ops` 新增 `public const string PowerSchemeActivate = "power.scheme.activate";`（复用现有错误码 `PowerSchemeNotFound` `:94`、`PowerSchemeActivateFailed` 或 `PowerSchemeCreateFailed`/`PayloadInvalid`）。
2. **DTO 与序列化上下文（`src/Zentwik.Contracts/Dtos.cs`、`src/Zentwik.Contracts/Json/ZentwikJson.cs`）**：请求可直接复用既有 `PowerSchemeTarget(Guid Guid)`（已在 `Ops.PowerSchemeBackup` `:305` 使用）；响应可复用 `PowerSchemeDto`（返回激活后的最新 `PowerSchemeState`）。
3. **客户端连接方法（`src/Zentwik.Contracts/AgentConnection.cs:230-279`）**：新增 `ActivatePowerSchemeAsync(Guid scheme, CancellationToken ct = default)`，调用 `InvokeAsync(Ops.PowerSchemeActivate, new PowerSchemeTarget(scheme), ZentwikJson.Default.PowerSchemeTarget, ZentwikJson.Default.PowerSchemeDto, ct)`。
4. **核心服务激活方法（`src/Zentwik.Core/Power/PowerSchemeService.cs:128` 附近）**：新增 `PowerSchemeState Activate(Guid scheme)`：先调 `Find(scheme)` 确认存在（抛 `ErrorCodes.PowerSchemeNotFound`），再调 `power.TrySetActiveScheme(scheme, out var error)`，失败抛 `CoreException`；返回 `State(scheme, entry.FriendlyName)`。
5. **服务端分派（`src/Zentwik.Agent/AgentSession.cs:275-326`）**：在 `DispatchAsync` 增加 `case Ops.PowerSchemeActivate:`，解析 `PowerSchemeTarget`，调用 `PowerSchemes().Activate(payload.Guid)` 并序列化 `PowerSchemeDto`。
6. **CLI 子命令（`src/Zentwik.Cli/Commands.cs:396-403`）**：在 `PowerAsync` 增加 `"activate" | "set-active"` 分支，要求 `--scheme <GUID>`（或 `--id`），调用 `connection.ActivatePowerSchemeAsync(guid, ct)` 并输出 JSON。
7. **Desktop ViewModel（`src/Zentwik.Desktop/ViewModels/MainViewModel.cs:121-124,191-194,345-355,854-959`）**：新增 `ActivatePowerSchemeCommand`（`() => IsConnected && SelectedPowerScheme is { Active: false }`）与 `ActivateSelectedPowerSchemeAsync()`，成功后刷新 `PowerSchemes`（使 Grid 中「活动」列立即更新）。
8. **Desktop UI 按钮与三语键（`src/Zentwik.Desktop/MainForm.cs:561-570,870-960`、`src/Zentwik.Localization/Resources/strings.*.json:91-101`）**：在 `schemeButtons`（`FlowLayoutPanel`）加入 `_activatePowerSchemeButton`，绑定 `ActivatePowerSchemeCommand`，三语 JSON 新增 `power.scheme.activate`（例如「设为活动」/「設為使用中」/「Set Active」）。
9. **策略写后刷新评估（`src/Zentwik.Core/Plans/PlanCoordinator.cs:1276-1300`）**：`ApplyPower` 写入目标方案的 AC/DC 设置后，若目标方案正是当前活动方案（`schemeId == active?.Guid`），可调用 `_power.TrySetActiveScheme(schemeId, out _)` 触发电源管理器重载（对应旧版 `RefreshActiveScheme()` 语义）。

- **与 `ActiveScheme` 的账本语义（`StateLedger.cs:3-150`）**：
  - 用户主动点击「设为活动」或 CLI `power activate` 属于用户直接接管操作，不走 `PlanCoordinator.ApplyPower`，不应写入 `OwnerKind.Plan` / `OwnerKind.Scene`（或若账本中曾存在 `power.active` 条目，应按用户接管标记 `OwnerKind.User`，防止后续场景自动抢回）。
  - 若未来 `PowerPlan` 需要支持「切换系统活动方案」，则应在 `PlanCoordinator.ApplyPower` 对 `"power.active"` 调用 `_ledger.Acquire("power.active", source, priority, currentGuid, desiredGuid, plan.Id, owner)`（`Status = Applied`, `Restorable = true`），即可直接激活 `ReadTarget`/`RestoreTarget`（`:1647,:1747`）既有死分支。
- **失败行为（`observed` / `derived`）**：
  - 方案不存在或已被外部删除：`PowerSchemeService.Find(scheme)`（`:423-425`）抛出 `CoreException(ErrorCodes.PowerSchemeNotFound)`，CLI 返回对应非零退出码，Desktop `ShowNotice(..., isError: true)`。
  - `PowerSetActiveScheme` 返回非 0：由 `Win32PowerPolicyService.Describe("PowerSetActiveScheme", result)` 格式化 Win32 错误并抛出 `CoreException`。

### 3) 坑（`observed` / `derived`）
- **隐藏方案被激活后临时暴露**：若激活 `Visible == false` 的方案，Windows 会将其临时顶进 `PowerEnumerate` 可见列表；切走后再次消失（`PowerSchemeService.cs:89-93`、`PlanCoordinator.cs:1228`）。UI 上「活动」切换按钮对隐藏方案应谨慎或提示其临时可见性。
- **`PowerSchemeService.Activate` 不可阻断 `SwitchAway`**：`Delete` 内置的 `SwitchAway` 只从可见方案挑候选（`:418`），与主动激活相互独立。
- **写策略未二次激活不立即生效**：仅改写当前活动方案的 `AC/DC ValueIndex` 而不重调 `PowerSetActiveScheme`，会导致功耗/调度策略延迟到下次切换或重启才生效（旧版 `PowerPolicyService.cs:18-21` 已实证）。

### 4) 验证级别
- **静态源码审计**：当前未执行代码修改与测试。
- **可 fake 单测范围**：`PowerSchemeServiceTests`（使用 `FakePowerPolicyService`）可覆盖 `Activate` 成功、不存在抛 `PowerSchemeNotFound`、Win32 返回失败；`AgentPipeTests` 可覆盖 `Ops.PowerSchemeActivate` 往返；`PlanAndSceneTests` 可覆盖 `power.active` 账本语义。
- **仅 Windows 真机验证**：`Win32PowerPolicyService.TrySetActiveScheme` 真实切换系统电源计划及 `RefreshActiveScheme` 触发电源管理器重载效果。

### 5) 未找到项
- `[未找到]`：`Protocol.cs:94-101` 中尚未定义专门的 `ErrorCodes.PowerSchemeActivateFailed`（需新增或复用 `PowerSchemeCreateFailed`）。

---

## B. Desktop 表单 FMax / Intel 电压输入（P0）

### 1) 旧/新实现参照（`observed`）
- **新版后端已全通**：
  - 模型：`src/Zentwik.Domain/Plans.cs:41-44` 定义 `CpuTuningPlan.FMaxLimitMhz`（`int?`）与 `IntelVoltageOffsetMillivolts`（`int?`）；`:109-118` 定义 `IntelVoltageOffsetBounds`（`MinMillivolts = -150`, `MaxMillivolts = 100`）；`:295-305` `CurveOptimizerConfig.FormatPlanSummary` 已展示 FMax 与 Intel 电压。
  - CLI：`src/Zentwik.Cli/Commands.cs:785-810` 支持 `--fmax <MHz>` 与 `--intel-voltage <mV>`，且允许仅传其一（不强制必须有 CO）。
  - 服务端校验：`src/Zentwik.Agent/AgentSession.cs:831-839` 在 `ValidatePlan` 校验 `fmax is < 1000 or > 7000` 与 `IntelVoltageOffsetBounds.IsWithinRange(mv)`。
  - 执行与回读：`PlanCoordinator` 调 `IFMaxTuningProvider`（`FMaxAbstractions.cs:9` 容差 `10 MHz`，`2d9351d` 已改为钳制写入并记钳制后期望值与回读真实值）与 `IIntelTuningProvider`（`IntelMsrTuningProvider` Core 平面）。
- **Desktop 断点**：
  - `src/Zentwik.Desktop/ViewModels/MainViewModel.cs:1085-1171`（注：随 `2d9351d`/`40c049a` 演进后约在 `:1091-1221` 附近）中 `BuildPlanFromForm` 的 `case "cpuTuning":` 仅用 `NewPlanSpec` 解析 `default=... / ccdN=... / core=...`，完全未生成 `FMaxLimitMhz` 与 `IntelVoltageOffsetMillivolts`（`1156-1159` 甚至在无 CO 时直接抛 `ArgumentException`）。
  - `src/Zentwik.Desktop/MainForm.cs:71-81,370-382` 仅有 `_planKindCombo` / `_planIdBox` / `_planNameBox` / `_planSpecBox` 四个输入行，无 FMax 或 Intel 电压输入框。

### 2) 最小改动点（共 5 处，逐跳链路）
1. **Desktop 表单毛坯控件（`src/Zentwik.Desktop/MainForm.cs:71-81,370-382,796-799`）**：
   - 在 `_planForm` 的 `TableLayoutPanel`（严格 WinForms 原生毛坯控件：`Label` + `TextBox`，禁止自绘/主题）增加两行：`_planFMaxCaption` + `_planFMaxBox`、`_planIntelVoltageCaption` + `_planIntelVoltageBox`。
   - 仅在 `NewPlanKind == "cpuTuning"` 时显示（或切换非 `cpuTuning` 时隐藏/置灰），避免干扰 `power` 与 `scheduling` 表单。
   - 绑定 `TextChanged` 同步至 `_vm.NewPlanFMax` / `_vm.NewPlanIntelVoltage`（遵循 `_syncing` 防重入规范 `:1135-1139`）。
2. **ViewModel 属性与清理（`src/Zentwik.Desktop/ViewModels/MainViewModel.cs`）**：
   - 新增 `string NewPlanFMax` 与 `string NewPlanIntelVoltage` 属性（并在切换/重置表单时清空）。
3. **表单解析与放宽前置守卫（`src/Zentwik.Desktop/ViewModels/MainViewModel.cs:1085-1171`）**：
   - 在 `case "cpuTuning":` 中：当 `NewPlanSpec` 非空时沿用既有 CO token 解析（允许 `NewPlanSpec` 为空，只要 FMax 或 Intel 电压至少填了一项）；解析 `NewPlanFMax` 为 `int?`、解析 `NewPlanIntelVoltage` 为 `int?`。
   - 将守卫 `:1156` 改为 `if (defaultCo is null && ccdOffsets.Count == 0 && cores.Count == 0 && fmax is null && intelVoltage is null)` 才报错。
   - 在构造 `CpuTuningPlan`（`:1161-1170`）时传入 `FMaxLimitMhz = fmax` 与 `IntelVoltageOffsetMillivolts = intelVoltage`。
4. **校验边界与三语键（`src/Zentwik.Localization/Resources/strings.*.json`、`MainForm.cs:875-879`）**：
   - **校验边界**：FMax 在 UI 仅做整数格式解析（允许 `[1000, 7000] MHz` 属于工具支持范围而非固件限制，具体钳制与硬件能力由后端 `AgentSession`/`PlanCoordinator` 裁决）；Intel mV 仅做整数解析，范围 `[-150, +100] mV` 交由后端 `AgentSession.ValidatePlan:836-838` 校验并返回本地化/错误明细。
   - **三语键**：在 `strings.zh-CN.json` / `strings.zh-TW.json` / `strings.en-US.json` 新增 `plans.fmax`（如「FMax 上限 (MHz)」）、`plans.intel_voltage`（如「Intel 电压偏移 (mV)」）及对应格式错误键（不许硬编码中文）。
5. **可测性单测（`tests/Zentwik.Desktop.Tests/`）**：
   - 抽出或暴露 `BuildPlanFromForm` / 纯解析 helper，对「纯 CO」「纯 FMax」「纯 Intel mV」「CO+FMax 组合」「全空报错」「非整数格式报错」做无窗口纯逻辑单测，并在 `MainFormSmokeTests` 验证新控件标签在三语下非空。

### 3) 坑（`observed` / `derived`）
- **切页/切类型残留值**：若用户在 `cpuTuning` 填了 FMax 后切换 `NewPlanKind` 到 `power` 再切回，或保存成功后未清空 `NewPlanFMax` / `NewPlanIntelVoltage`，容易误提交脏值。
- **把 `[1000, 7000]` 当成固件硬边界**：客户端切勿把 `1000-7000 MHz` 写死为「固件上限」文案，以免与 `2d9351d` 口径冲突。
- **`NewPlanSpec` 空串分支**：原 `:1103` `spec.Split(...)` 在 `spec` 为空时会得到空集合，但原 `:1156` 直接拒绝空 CO；必须同步放宽 `:1156` 才能支持「只建 FMax 方案」或「只建 Intel 降压方案」。

### 4) 验证级别
- **静态源码审计**：当前未执行代码修改与测试。
- **可 fake / 单测范围**：表单解析逻辑、`CpuTuningPlan` 构造、三语资源完整性、`MainFormSmokeTests` 控件树检查均可 100% 离线单测（Windows 环境跑 Desktop 测试）。
- **仅真机验证**：FMax 写入钳制回读与 Intel MSR 0x150 写入回读。

### 5) 未找到项
- `[未找到]`：Desktop 端尚未发现独立的 `CreatePlanFromForm` 单元测试文件（目前 `tests/Zentwik.Desktop.Tests/` 仅 `MainFormSmokeTests.cs` 与 `61d5e6e` 新增的 `TriggerKindsTests.cs`）。

---

## C. Desktop 单实例互斥体（P1）

### 1) 旧/新实现参照（`observed`）
- **旧版参照（`Zentwik.UI/Services/SingleInstanceGate.cs:15-105`）**：
  - 互斥体名：`Local\Zentwik.UI.SingleInstance`（`:18`），每用户登录会话一个实例（不跨 RDP 会话阻塞）。
  - 激活管道：`Zentwik.UI.Activate`（`:20`，引用 `WorkerPipeContract.UiActivatePipeName`），独立后台线程 `Listen(onShow, onExit)`（`:41-85`）阻塞等待单行文本 `show` / `exit`。
  - 二次启动：`TryTakeOwnership()` 返回 `false` 后，调用激活客户端连入 `Zentwik.UI.Activate` 写入 `"show"` 并直接退出（`:92-98`）。
- **新版现状（`observed`）**：
  - `src/Zentwik.Desktop/Program.cs:9-27`：仅执行 `ApplicationConfiguration.Initialize()` → `SynchronizationContext.SetSynchronizationContext(new WindowsFormsSynchronizationContext())` → `new MainViewModel(catalog)` → `Application.Run(new MainForm(viewModel, catalog))`，**无任何互斥体或激活管道**。
  - `src/Zentwik.Desktop/MainForm.cs:1707-1755` 与 `ShellBehavior.cs:13-56`：已启用 `NotifyIcon` 托盘与 `CloseToTray`（点右上角关闭仅 `Hide()` 并 `EnsureTray()`），导致用户再次双击 exe 时会开出第二个窗口与第二个托盘图标，且各自竞争 `desktop-shell.json` 与 Agent 连接。

### 2) 最小改动点（共 5 处）
1. **单实例闸门组件（新增 `src/Zentwik.Desktop/Services/SingleInstanceGate.cs`）**：
   - 复用旧版双通道模式：`Mutex(initiallyOwned: true, @"Local\Zentwik.Desktop.SingleInstance", out bool createdNew)` + `NamedPipeServerStream`（管道名建议按当前 SessionId 或默认 `Zentwik.Desktop.Activate`，只放行当前用户 SID）。
   - 提供 `TryAcquire()`、`NotifyExistingInstance()`（发送 `"show"`）、`StartListening(Action onShow, Action onExit, CancellationToken ct)` 及 `Dispose()`。
2. **`Program.Main` 闸门落点（`src/Zentwik.Desktop/Program.cs:11-26`）**：
   - **精确顺序**：必须在 `ApplicationConfiguration.Initialize()` 与 `SynchronizationContext.SetSynchronizationContext(new WindowsFormsSynchronizationContext())` **之后**、构造 `new MainViewModel(catalog)` **之前**执行 `TryAcquire()`。
   - **理由**：`MainViewModel` 构造函数会捕获 `SynchronizationContext` 并在 `MainForm` 中触发自动连接（`autoConnect: true`）；若在互斥体之前构造 VM/Form，二次启动会白建连接与管道会话。若抢锁失败，仅发 `"show"` 后 `return`，零窗口闪烁。
3. **冒烟测试构造路径隔离（`tests/Zentwik.Desktop.Tests/MainFormSmokeTests.cs:22-23,50-51,76-77`）**：
   - 冒烟测试直接调用 `new MainViewModel(catalog)` 与 `new MainForm(viewModel, catalog, autoConnect: false)`，**不经过 `Program.Main`**。因此互斥体与激活监听必须放在 `Program.Main`（或显式可注入/可关闭），**绝不可**放进 `MainForm` 或 `MainViewModel` 构造函数，否则并行跑多个 `[Fact]` 或宿主已打开 Desktop 时测试会互相阻塞。
4. **窗口唤醒与托盘退出协同（`src/Zentwik.Desktop/MainForm.cs:1707-1755`）**：
   - `onShow` 回调通过窗口 `BeginInvoke` 调度到 UI 线程执行现有 `ShowFromTray()`（`Show()` + `WindowState = Normal` + `Activate()` `:1734-1739`）。
   - `onExit` 回调（若保留外部退出通道）置 `_exitRequested = true` 并调用 `Close()`，绕过 `CloseToTray` 拦截（`:1744`）。
5. **激活管道安全与资源释放（`Program.cs:25`）**：
   - 在 `Application.Run(form)` 返回、`viewModel.DisposeAsync()` 之后释放激活管道 `CancellationTokenSource` 并 `ReleaseMutex()` / `Dispose()`。

- **与 `LocalSystem` 服务的会话隔离关系（`observed` / `derived`）**：
  - `Zentwik.Agent` 以 `LocalSystem` 运行在 **Session 0**；`Zentwik.Desktop` 运行在交互用户的 **Session N (N ≥ 1)**。
  - `Local\Zentwik.Desktop.SingleInstance` 互斥体属于内核对象命名空间的 **Session 局部命名空间**（`\Sessions\N\BaseNamedObjects\`），天然按登录会话隔离，既不会与 Session 0 的 `Zentwik.Agent` 冲突，也不会阻断多用户/RDP 各自打开一个 Desktop。
  - `Zentwik.Agent` 侧对客户端仅通过命名管道 `AgentServer` 的 `CallerPolicy` 校验当前调用方身份，不依赖 `Desktop` 单实例；`Desktop` 单实例只解决同会话多 UI 窗口与多 `NotifyIcon` 冲突。

### 3) 坑（`observed` / `derived`）
- **`AbandonedMutexException` 容错**：若前一实例异常崩溃未正常 `ReleaseMutex()`，新实例 `WaitOne(0)` 会抛 `AbandonedMutexException`；必须捕获并视为已获得所有权，否则崩溃后用户将永远打不开界面。
- **管道名称跨会话冲突**：Windows 命名管道 `\\.\pipe\<name>` 默认是全局共享的（不像 `Local\` Mutex 自动按会话隔离）！若多用户同时登录同一台机器，固定管道名 `Zentwik.Desktop.Activate` 会导致后一个用户建管道失败或唤醒错会话。管道名建议拼接 `Process.GetCurrentProcess().SessionId`（如 `Zentwik.Desktop.Activate.{sessionId}`）或设置当前用户 SID ACL。
- **托盘退出必须清理 `_tray`**：`MainForm.Dispose` 需确保销毁 `_tray`（避免退出后托盘残影图标直到鼠标划过才消失）。

### 4) 验证级别
- **静态源码审计**：当前未执行代码修改与测试。
- **可单测范围**：`SingleInstanceGate` 若抽象互斥体/激活名称生成，可做纯逻辑命名测试；`MainFormSmokeTests` 继续走 `autoConnect: false` 验证不受闸门影响。
- **仅 Windows 实测**：双开进程抢 `Local\` Mutex 互斥、`AbandonedMutexException` 恢复、命名管道 `"show"` 把已隐藏到托盘的窗口拉回前台、多会话管道隔离与 `NotifyIcon` 销毁无残影。

### 5) 未找到项
- `[未找到]`：新版 `Zentwik.Desktop` 目录中当前无任何激活管道或 `Mutex` 既有实现。

---

## Deepseek 实施时需特别注意的 5 条

1. **严格遵守 WinForms 毛坯红线，不加任何主题/缩放/自绘**：项 B 新增的 FMax 与 Intel 电压控件只能使用原生 `Label` + `TextBox` 嵌入既有 `TableLayoutPanel`（`MainForm.cs:370`），不可引入自定义绘制或扩展样式；所有新增文案必须同步写入 `strings.zh-CN.json` / `strings.zh-TW.json` / `strings.en-US.json` 三语文件。
2. **区分「工具支持范围」与「固件限制」，勿在 UI 重复写死后端边界**：FMax `[1000, 7000] MHz` 仅是工具支持范围，且后端 `2d9351d` 已改为钳制写入；Intel mV `[-150, +100]` 由 `IntelVoltageOffsetBounds`（`AgentSession.cs:836`）权威校验。UI 表单只需做整数格式解析，避免前端与后端错误口径分裂。
3. **操作 `power.scheme.activate` 不得混入 `StateLedger` 计划所有权，且注意写策略重载**：用户主动切系统电源方案是用户操作，不应伪造 `OwnerKind.Plan`；若后续扩展 `PowerPlan` 自动切方案再接入 `power.active` 死分支。同时在 `PlanCoordinator.ApplyPower` 修改当前活动方案的策略项后，需留意旧版 `RefreshActiveScheme`（重调一次 `PowerSetActiveScheme`）对即时生效的必要性。
4. **单实例闸门切勿写进 `MainForm` / `MainViewModel` 构造函数**：`MainFormSmokeTests` 通过 `new MainForm(viewModel, catalog, autoConnect: false)` 构造窗口。互斥体与激活管道必须留在 `Program.Main` 中且位于 `SetSynchronizationContext` 之后、`new MainViewModel` 之前。
5. **命名管道跨会话隔离与崩溃互斥体接管**：`Local\` 互斥体天然按 Windows Session 隔离，但命名管道全局可见；`Zentwik.Desktop.Activate` 必须带 `SessionId` 或严格限定当前用户 SID，且必须捕获 `AbandonedMutexException` 防止进程崩溃后死锁无法再启动。

---

## 计数闭合核对

| 统计项 | 数值 | 明细 |
|---|---:|---|
| 覆盖待办项 | **3 项** | A (`power.scheme.activate`)、B (Desktop FMax / Intel 电压)、C (Desktop 单实例互斥体) |
| 最小改动点总计 | **19 处** | A：**9 处**；B：**5 处**；C：**5 处** |
| 未找到项标注 | **3 处 `[未找到]`** | A(`PowerSchemeActivateFailed` 错误码)、B(Desktop 表单解析单测)、C(Desktop 既有单实例实现) |
