# 旧项目 ↔ 新项目 ↔ 研究文档 三向对照审计总报告

- **审计日期**：2026-11-28
- **旧项目（基准）**：`Motu-Z/Zentwik` `main` @ `727462601911609e6dacbe3cb8f144246829087a`
- **新项目（对照）**：`Motu-Z/Zentwik` `feat/gui-protocol` @ `100e9c0f4f9e7f9b160f8b0bed9ea99afdf20922`
- **研究文档（第三极）**：`Motu-Z/Zentwik-Next-Research` `main` @ `7680e5109ed010e4681180393ff85fb2ce412554`
- **分片报告**：`research/audit/20261128/A.md` … `J.md`
- **验证口径**：本报告在 Linux 沙盒内完成**只读静态源码对照**，未执行 Windows/.NET 构建与真机运行；所有涉及实机或硬件的结论均标注出处验证级别，并严格执行「实际调用入口 → 端到端链路 → 验证级别 → 失败/缺模块行为」补充判定纪律。

---

## 1. 计数闭合核对

| 核对维度 | 计数 | 源码 / 文档出处 | 闭合状态 |
|---|---:|---|---|
| 旧 UI 页签 / 视图 | **8 页签 / 14 视图 (+1 主壳)** | `Zentwik.UI/MainWindow.xaml:208-276`, `Views/*.xaml` | 已闭合 (`A.md`) |
| 新 UI 页签 / 表格 | **7 页签 / 13 DataGridView** | `MainForm.cs:59-181,233-241`, `MainFormSmokeTests.cs:26-37` | 已闭合 (`A.md`) |
| 旧 CLI 命令 | **0**（仅 Worker/UI 启动开关） | `Zentwik.Worker/Program.cs`, `Zentwik.UI/App.xaml.cs` | 已闭合 (`H.md`) |
| 新 CLI 命令 / 退出码 | **16 命令 / 7 退出码 (0..6)** | `Zentwik.Cli/Program.cs:29-40`, `Commands.cs:18-35`, `Output.cs:6-15` | 已闭合 (`H.md`) |
| 旧 IPC 命令 | **46** | `Zentwik.Core/Ipc/WorkerPipeContract.cs:77-680` | 已闭合 (`G.md`) |
| 新协议 `Ops` / `Events` / `ErrorCodes` | **38 Ops / 2 Events / 28 ErrorCodes** | `Zentwik.Contracts/Protocol.cs:20-110`（全量在 `AgentSession.DispatchAsync` 分派） | 已闭合 (`G.md`) |
| 旧 / 新 `Capability` | **10 → 9** | `CapabilityIds.cs:9-40` → `Capability.cs:28-43` | 已闭合 (`G.md`) |
| 旧 / 新内置模块 | **8 → 7**（含 1 可选 `power-limits`） | `CoreRuntimeModule` 等 → `ModuleCatalog.cs:26-37` | 已闭合 (`G.md`) |
| ADR 文档数 | **8 篇**（`ADR-001`..`008`） | `decisions/ADR-*.md` | 已闭合 (`I.md`) |
| `MIGRATION_MATRIX.md` 主表 | **15 行**（`#1`..`#15`） | `project/MIGRATION_MATRIX.md:21-35` | 已闭合 (`I.md`) |
| 知识与研究扫描文档 | **39 篇**（26 规范 + 13 研究/交接） | `knowledge/` + `research/` | 已闭合 (`J.md`) |
| 10 分片审计条目总计 | **241 条** | A(14)+B(32)+C(16)+D(24)+E(20)+F(23)+G(15)+H(15)+I(44)+J(38) | 已闭合 |

---

## 2. 三张差异表

### 2.1 文档要求 vs 旧现实（`7274626`）

| 领域 | 文档要求 | 旧现实 | 状态 |
|---|---|---|---|
| 进程与权限模型 | `ADR-003`：LocalSystem 服务独占句柄，中完整性 UI | 每用户提权 Worker + 高完整性 WPF UI，无服务宿主 | 缺 |
| IPC 协议 | `ADR-004`：4B 长度前缀、多路复用、事件推送、稳定错误码 | 单行 `\n` JSON (v23)，46 平铺命令，无推送，中文错误文案 | 缺 |
| 场景与状态账本 | `ADR-005` / `state-ownership.md`：4 触发、退出回退、所有权与用户接管 | 退出不回退、无前值账本、优先级数字小优先（与新规范相反） | 缺 / 冲突 |
| 电源备份与隐藏方案 | `power-management.md`：注册表子键过滤 + 双轨备份恢复 | 仅 `PowerEnumerate` 可见枚举，无删前备份与恢复 | 缺 |
| 硬件遥测与历史 | `polling-budget.md` / `amd-power-limit.md`：订阅驱动、3600 帧环形缓冲、SMN/RAPL/APERF | 无系统遥测、无温度/功耗/逐核频、仅调度页 3s 进程采样 | 缺 |
| AMD 功耗墙 | `amd-power-limit.md`：RSMU `0x56/0xCB/0x57/0x58` + PM 表回读 | 旧版 `348712a` 已删除，基线 `7274626` 无功耗墙代码 | 缺 |
| CLI 与 i18n | `ADR-007` / `ARCHITECTURE_CONSTRAINTS.md:9`：正式 CLI、三语资源表 | 无独立 CLI；全量简体中文硬编码 | 缺 |
| 开机防砖护栏 | （旧项目自建） | `BootGuard.cs` 连续 3 次写入未完成阻断 | 旧有文档未纳 |

### 2.2 文档要求 vs 新现实（`100e9c0`）

| 领域 | 文档要求 | 新现实 | 状态 |
|---|---|---|---|
| 桌面 UI 技术栈 | `ADR-002` 写 WPF；`AI_UI_DESIGN_GUIDANCE` 写 Flutter；`UI_UX_DESIGN_SPEC` 写 WPF 无原生控件 | `Zentwik.Desktop` 实为 **WinForms 裸控件**（`MainForm.cs` 7 页），Flutter 已删 | 文档与现实冲突（已按 §2.3 口径标 Deprecated） |
| 特权服务 NativeAOT | `ADR-001:13`：`Zentwik.Agent` 使用 NativeAOT | `Zentwik.Agent.csproj:7` 为 `<PublishAot>false</PublishAot>` | 文档与现实冲突 |
| 持久化分文件 | `ADR-006:12`、`serialization-persistence.md:13`：设置/方案/场景/账本分文件 | `StateStore.cs:21`、`StateDocument.cs` 全部合存在单文件 `state.json` | 文档与现实冲突 |
| 服务非崩溃恢复前提 | `security-service.md:20-21,57`：`1066` 退出码须置 `SERVICE_CONFIG_FAILURE_ACTIONS_FLAG=TRUE` | `WindowsServiceHost` 上报 `1066`，但 `ServiceInstaller.cs:526-551` 未设该 Flag | 文档与现实冲突 |
| FMax Override 语义 | `amd-smu.md §4`：`-1000..+200 MHz` 偏移量 | `AmdSmuTuningProvider.cs:203` 按 `[1000, 7000] MHz` 绝对频率直传 `0x70` | 文档与现实冲突 |
| 单实例与托盘 | `MIGRATION_MATRIX.md:35` 称客户端无状态无需单实例 | `Desktop` 已启用 `NotifyIcon` 托盘，但 `Desktop/Program.cs` 无互斥体，多开冲突 | 文档与现实冲突 |
| 本地化与硬编码 | 最新口径：文案不许硬编码，走 `Zentwik.Localization` | `MainForm.cs` 已走 `_catalog`，但 `MainViewModel.cs` 仍有 29 处硬编码中文 | 文档与现实冲突 |
| 安装脚本路径与合规 | `installer.md:3` 写 `zentwik-next.iss`；`ADR-008:43` 称分发合规未落地 | 实为 `scripts/installer/zentwik.iss`；`licenses/LGPL-2.1.txt` 与 `THIRD-PARTY-NOTICES.md` 已补齐 | 文档滞后 |
| 第三方插件隔离 | `PROJECT_BRIEF.md:23`、`modules-capabilities.md` | 无插件进程隔离与加载器 | 缺 |

### 2.3 旧实现 vs 新实现（`7274626` vs `100e9c0`）

| 能力域 | 旧实现（`7274626`） | 新实现（`100e9c0`） | 结论 |
|---|---|---|---|
| **进程权限与服务** | 每用户提权 Worker | `LocalSystem` 服务 + 调用方 SID/IL/Session 校验 | 新补齐安全边界（缺非崩溃恢复 Flag 与管道 `FirstPipeInstance`） |
| **AMD CO / FMax** | 对称 `±50`、线性核映射、基线 0 仍写 | 不对称 `[-50,+30]`、APOB/CPUID+L3、`Global\Access_PCI`、基线 0 拒写 | 后端增强，但 **Desktop UI 无法创建 FMax 方案** |
| **AMD 功耗墙** | 无（已删） | RSMU 读写 + `0x03` 3 次重试 + `1 W` 下界 + 四态回读 + `±1 W` 探针 | 新有旧无（9955HX 管理员会话实测），但 **有回读无回退**（未进 `StateLedger`） |
| **Intel 调校** | Core/GT/Cache 三平面电压 + `0x1AD` 8 档睿频读写 | 仅 Core 平面电压（含 busy 轮询与跨进程锁）+ `0x1AD` 1C/8C 只读 | **部分/退化**，且仅 `FakeIntelTuningProvider` 验证，Desktop UI 无写入入口 |
| **电源方案管理** | 仅可见枚举 + 8 项策略 AC/DC UI 编辑 + 活动切换 | 注册表全枚举 + 双轨备份还原；通用 `PowerPlan` 有回读 | 备份还原新有旧无；**缺主动切换方案 op 与 8 项策略预设 UI** |
| **进程与调度** | Toolhelp 实时列表 + CPU% 采样 + `EcoQoS` + 掩码编辑器 + 退避重试 | 后端 `SchedulingPlan` 强制执行/漂移重写；**无 `process.*` op** | **UI 进程页为空骨架**、缺 `EcoQoS`、缺失败退避、缺 `ProcessRefresher` |
| **监控与遥测** | 仅调度页 3s 进程采样 | `Win32TelemetrySource` + `AmdSensorSource` + 3600 帧缓冲 + CLI | 新有旧无（9955HX 实测），但 **Desktop UI 未画折线、未显示温功与逐核频** |
| **场景与账本** | 退出不回退、无前值账本 | `StateLedger` + 用户接管 + 退出/同周期恢复 + 唤醒核对 | 后端完备，**Desktop UI 漏 `ProcessExited` 触发器**，唤醒/重启仅 Fake 单测 |
| **CLI / 设置 / 托盘** | 无 CLI；丰富设置页；Worker 托盘含方案菜单；双互斥体 | 16 命令 CLI（7 退出码）；WinForms 7 页；简单托盘；无单实例 | CLI 新有旧无；Desktop 缺单实例与采样间隔配置 |

---

## 3. 总表（按状态分组 & 用户影响排序）

### 3.1 缺口（旧有新无或文档要求未实现，按用户影响排序）

| 优先级 | 编号 | 项与证据（旧 vs 新 / 文档） | 断点 / 缺失说明 | 建议下一步（仅建议） |
|---:|---|---|---|---|
| P0 | `D-20`/`D-21`/`A-07` | 进程管理后端 `process.*` op 与 Desktop 进程页（旧 `ProcessView.xaml` + `ProcessRefresher`；新 `MainForm.cs:168-172` 空骨架） | 无 `process.*` op、无 CPU% 滑窗与路径采集，按钮禁用 | 在 `Contracts/Protocol.cs` 与 `AgentSession` 补 `process.list/set` 并恢复进程采样器 |
| P0 | `C-07`/`A-09` | 主动切换系统活动电源方案（旧 `PowerPolicyService.SetActiveScheme`；新 `PowerSchemeService` / `AgentSession` 无对应 op） | `Win32PowerPolicyService.TrySetActiveScheme` 仅内部使用，未暴露协议与 UI/CLI | 新增 `power.scheme.activate` op 并在电源页加「设为活动」按钮 |
| P0 | `H-14`/`R-41` | Desktop 客户端单实例闸门（旧 `SingleInstanceGate.cs`；新 `Desktop/Program.cs` 无互斥体） | 开启「关闭留托盘」后多开会产生多窗口与多托盘图标 | 在 `Desktop/Program.cs` 补 `Mutex` + 激活已有窗口机制 |
| P0 | `G-03`/`J-22` | 服务非崩溃异常退出（`1066`）触发 SCM 恢复的前提未设（`ServiceInstaller.cs:526-551` 缺 `SERVICE_CONFIG_FAILURE_ACTIONS_FLAG`） | 未调用 `ChangeServiceConfig2W(..., 4, TRUE)`，非崩溃停止不重启 | 在 `ServiceInstaller.ConfigureRecovery` 补设 `fFailureActionsOnNonCrashFailures = true` |
| P1 | `B-20`/`B-22` | Intel GT/Cache 电压平面与 `0x1AD` 睿频倍频写入（旧 `IntelTuning.cs` 全支持；新 `IntelMsrBits` 仅定义常量未接线） | GT/Cache 属「代码存在但未接线」，`0x1AD` 写入完全缺失 | 补齐 `IntelMsrTuningProvider` 多平面与 `0x1AD` 写入，并在 Intel 真机验证 |
| P1 | `D-12` | 进程优先级 `EcoQoS` 档位与 `RealTime` 拦截（旧 `ProcessPriority.cs`；新 `AffinityScheduler.cs:158-167`） | 未调 `SetProcessInformation(ExecutionSpeed)`，且允许传入 `RealTime` | 在 `AffinityScheduler` 补 `EcoQoS` 分支并拦截 `RealTime` |
| P1 | `D-15` | 调度写入失败最大重试次数与指数退避（旧 `ScheduleRetryLedger.cs`；新 `PlanCoordinator` 每 2s 无限重试） | 受保护进程失败后每轮重复打错误与重试 | 在 `PlanCoordinator` 为调度失败目标引入退避计数 |
| P1 | `F-22` | 开机连续崩溃防砖护栏（旧 `BootGuard.cs`；新 `[未找到]`） | 新版无连续 3 次写入失败启动阻断 | 评估是否在 `Agent` 启动/恢复路径引入等价护栏 |
| P1 | `B-27` | 一键恢复出厂默认 CO/FMax（旧 `TuningStore.ResetToFactory`；新仅按方案前值 `Revert`） | 用户手动或跨版本遗留的硬件状态无一键归零入口 | 增加显式归零操作（或确认由 `StateLedger` 全权覆盖） |
| P2 | `E-16` | Intel 传感器（`0x19C/0x1B1/0x611/0x198`）与 AMD 每核功耗 `0xC001029A` | `AmdSensorSource` 仅支持 AMD 封装温功耗与逐核频 | 按需排入后续硬件遥测路线图 |
| P2 | `A-08`/`D-16` | 自定义核心掩码库与拓扑勾选器（旧 `MaskManagerOverlay.xaml`；新仅文本 `CpuSetSpec`） | 规则需手写字符串，缺少可视化拓扑勾选 | 在 WinForms 调校/进程页补充轻量核勾选对话框 |
| P2 | `B-11`/`J-06` | AMD Zen4/5 Curve Shaper (`0xA6/0x84`) 与全核 CO (`0x07/0x36`) | 新旧均未实现 | 按路线图延后 |
| P2 | `R-13`/`G-12` | 第三方插件独立进程隔离宿主 | 仅内置 `ModuleCatalog` | 属长期架构演进项，暂不阻塞当前版本 |

### 3.2 部分实现 / 调用链断裂（含断点位置）

| 编号 | 能力项 | 断点所在跳（入口 → op → 领域 → 平台） | 状态判定 |
|---|---|---|---|
| `B-13`/`A-05` | AMD FMax 与 Intel 电压方案在 Desktop 创建 | `MainViewModel.BuildPlanFromForm` 仅解析 CO，**无 FMax 与 Intel 电压输入分支**（CLI/JSON 可建，Desktop 不可建） | 部分实现（断在 Desktop UI 表单） |
| `F-03`/`A-10` | `processExited` 场景触发器 | 后端 `SceneEngine` 与 `Scene.cs` 已实现，但 `MainViewModel.TriggerKinds:162-167` **漏掉 `ProcessExited`** | 部分实现（断在 Desktop UI 下拉） |
| `E-20`/`A-04` | 监控折线图与实时温度/功耗/逐核频展示 | `MainViewModel.ApplySample:1487` 已发 `HistoryChanged`，但 `MainForm.cs` **未订阅重绘**，实时卡未接温度/功耗/逐核频 | 部分实现（断在 WinForms 视图消费） |
| `C-08`/`A-09` | 8 项处理器电源策略参数预设与 UI 编辑 | 仅 `PlanCoordinator.ApplyPower` 接通用 `PowerPlan`，**无 8 项固定 GUID 预设与范围校验，UI 无编辑入口** | 部分实现 |
| `H-08`/`H-15` | 设置页与采样间隔/多语言同步 | `UserSettings` 有 `MonitorIntervalMs`/`SensorIntervalMs`，但 `PersistSettingsAsync:1525` 未传，设置页无控件；`SupportedLanguages` 仅 `zh-CN` | 部分实现 |
| `D-02` | CPU Sets 与 Affinity Mask 互斥清理 | 写 CPU Sets 前检外部窄亲和性，但规则切换或共存时**不自动清理另一侧旧限制** | 部分实现 |
| `G-10`/`J-23` | 命名管道安全加固 | `AgentServer` 未设 `FirstPipeInstance` 与 `PIPE_REJECT_REMOTE_CLIENTS`；`AgentConnection` 未显式设 `TokenImpersonationLevel.Identification` | 部分实现 |

### 3.3 仅模拟验证（无真机闭环证据）

| 编号 | 能力项 | 验证证据 | 缺口说明 |
|---|---|---|---|
| `B-16`~`B-19`, `B-23` | Intel OC Mailbox (`0x150` bit63 busy、跨进程锁、2 mV 容差) | `FakeIntelTuningProvider`（`tests/Zentwik.Contracts.Tests` / `Core.Tests`） | 仅模拟验证；无 Intel 实机管理员/Session 0 测试 |
| `F-19`, `F-21` | 同周期重启补回退 (`RecoverAsync`) 与休眠唤醒核对 (`ReconcileAfterResumeAsync`) | `SceneRecoveryTests.cs`（`FakeBootInfo`, `FakeCpuTuningProvider`） | 仅模拟验证；`KNOWN_LIMITS.md` 亦标注休眠/重启待真机闭环 |

### 3.4 有回读但无回退

| 编号 | 能力项 | 证据 | 缺口说明 |
|---|---|---|---|
| `C-12`~`C-14` | AMD 功耗墙写入 (`hardware.power.limit.apply`) | `AmdSmuTuningProvider.ApplyPowerLimits` + `Verify` + `ProbeChannel` | 写入前读基线、写后做四态回读与 `±1 W` 探针，但**未接入 `StateLedger`**，无法通过 `plans.revert` 或场景退出回退 |

### 3.5 新有旧无（相对旧版 `7274626` 的净新增能力）

1. **硬件功耗墙（可选模块 `power-limits`）**：RSMU DPTC 读写、PM 表 `0x03` 3 次重试与 `1 W` 下界、写后四态分类与探针复原（9955HX 管理员会话实测）。
2. **系统与硬件遥测**：`Win32TelemetrySource`、`ProcessorLoadReader`（每逻辑核）、`AmdSensorSource`（SMN 温度 / RAPL 功耗 / APERF 逐核均频，9955HX 实测）、`TelemetryRingBuffer`（3600 帧）与订阅背压控制。
3. **场景生命周期与状态账本**：`StateLedger` 全字段追踪、确定性三层仲裁、用户接管与释放（`ledger.list/release`）、`processExited` 闩锁与退出回退。
4. **电源方案备份与隐藏方案识别**：注册表子键过滤 overlay 占位键、JSON + `.pow` 双轨备份与从「平衡」派生还原（9955HX 实测）。
5. **独立 CLI（16 命令 / 7 退出码 / `--json`）与 Windows 服务化**：`Zentwik.Cli`、`Zentwik.Agent` (`LocalSystem` + `PipeCaller` 校验 + `DoctorService` + `DataDirectoryMigration`)。

### 3.6 文档与现实冲突（汇总）

1. **UI 技术栈**：`ADR-002` (WPF)、`AI_UI_DESIGN_GUIDANCE` (Flutter, Deprecated)、`UI_UX_DESIGN_SPEC` (WPF, Deprecated) 与现实 `WinForms` (`MainForm.cs`) 冲突。
2. **NativeAOT**：`ADR-001` 要求 `Zentwik.Agent` 用 NativeAOT，源码 `<PublishAot>false</PublishAot>`。
3. **分文件持久化**：`ADR-006` / `serialization-persistence.md` 要求分文件，源码全合并在单一 `state.json`。
4. **FMax Override 语义**：`amd-smu.md §4` 写 `-1000..+200 MHz` 偏移，新旧代码均按绝对 MHz 直传 `0x70`。
5. **非崩溃服务恢复 Flag**：`security-service.md:57` 要求设 `SERVICE_CONFIG_FAILURE_ACTIONS_FLAG`，`ServiceInstaller.cs` 未设。
6. **`MIGRATION_MATRIX.md` 滞后**：`#12 托盘`、`#13 自启`、`#15 单实例无需互斥体` 均与 `100e9c0` 实际代码冲突。
7. **硬编码文案**：最新口径禁止硬编码，但 `MainViewModel.cs` 仍有 29 处硬编码中文。
8. **其他过时文档**：`sensors/README.md` 仍写 `≥4Hz` 与 WPF `CompositionTarget.Rendering`；`installer.md` 写 `zentwik-next.iss`（实为 `zentwik.iss`）；`ADR-008` 称 LGPL-2.1 合规未落地（实际已补）；`three-plans.md` 残留 `PlanRevision` 描述。

---

## 4. 未验证项、本地正本依赖与 `unknown` 清单

1. **无法从 Linux 沙盒验证的项（需 Windows 环境构建与运行）**：
   - `dotnet build Zentwik.slnx`、`dotnet test`（包含 `MainFormSmokeTests` 与 WinForms UI 实拍）。
   - AMD 9955HX Session 0（Windows 服务态）下的 SMU/传感器/功耗墙与 SCM `1066` 恢复实测。
   - 真实休眠/睡眠恢复（`PBT_APMRESUMESUSPEND`）与跨整机重启对账（`RecoverAsync` + `ReconcileAfterResumeAsync`）。
   - `SYSTEM_CPU_SET_INFORMATION.Type` 在目标 Windows 版本上的 2 字节 vs 4 字节结构对齐实测。
   - Inno Setup 安装脚本 `scripts/installer/zentwik.iss` 的真实安装、升级覆盖与卸载流程。
2. **依赖 Windows 本地正本（`D:\Projects\Zentwik Project`）补查的项**：
   - 旧项目提交 `348712a`（can7）之前的历史功耗墙实现及本地未入 Git 的文档/工作区差异（`Zentwik-Docs` 如存在本地未提交正本）。
3. **判定为 `unknown` 的项**：
   - Intel 真机硬件行为：`IntelMsrTuningProvider` 在真实 Intel 桌面/移动处理器上的 `0x150` bit63 busy 时序、`S11.0.10` 写入与 UVP 截断表现目前为 `unknown`（仅有 `FakeIntelTuningProvider` 模拟）。
   - `0x01` 探针返回 `2` 的固件权威依据在文档中标注为来源未知（`unknown`）。
