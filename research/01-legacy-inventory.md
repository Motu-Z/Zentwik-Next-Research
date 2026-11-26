# 旧版 Zentwik 能力与实现盘点

> 基线：`D:\Projects\Zentwik Project\Zentwik` @ `bc3c59b`（2026-11-25，main），版本号 2.6.0-can16。
> 规模：Core 约 60 个类型、Worker 约 3.7k 行（`PipeServerService.cs` 单文件 3076 行）、UI 约 6k 行（`App.xaml.cs` 1633 行）、36 个 xunit 测试文件。
> 本文只记录能从源码指认的事实；无法确认的标注「未确认」。

## 0. 关键结论

1. 旧版是「单机双进程」工具：高权限 Worker 独占硬件写入与全部业务配置，WPF UI 只做展示与调用；没有服务化、没有插件、没有跨机能力。
2. 真正跑通的是四块：AMD CO/FMax、Windows 电源计划与策略、按进程名的绑核/优先级、四触发场景；其余（监控、自动化语义、模块生命周期）都只做到「够这台机器用」。
3. 协议是唯一集成面：46 条平铺命令 + 全局版本 23 + 字符串载荷，新增功能必然改协议；这是迁移时最先要拆的部分。
4. 自动化没有状态所有权：退出不恢复、无用户接管识别、一条规则同时改三个领域，与产品简报的核心约束直接冲突。
5. 安全边界不完整：提权 Worker 的命名管道只按用户 SID 放行，不校验调用方身份；同用户任意进程可下发 SMU 写入。

## 1. 能力矩阵

| 领域 | 旧版实际实现 | 状态 | 关键证据 |
|---|---|---|---|
| AMD 调校 | 逐核 Curve Optimizer（软件输入钳制 -50..+50）、FMax（特定机型观测的绝对频率区间 4450–5650MHz，步进约 25，不同于 Boost Clock Override 偏移）；读快照、逐核写入+回读校验；开机静默应用、休眠恢复补写、防砖护栏；可存为调校方案并应用 | 已实现可用（需 PawnIO + 管理员 + AMD） | `Services/SmuReader.cs`、`Services/TuningApplyService.cs`、`Services/BootGuard.cs`、`Services/SleepResumeWatcher.cs` |
| Intel 调校 | MSR 三平面（Core/GPU/Cache）电压偏移 + 8 档倍频上限；逐平面写并回读（容差 2mV）、倍频一次性写但逐档比对，不一致判失败；`apply-intel-tuning` 支持 `restore` 回基线 | 部分实现：会话级能力，不进行方案保存，不进 `plans.json`，开机/休眠不会重放（启动路径只用 AMD 通道） | `Services/IntelTuningChannel.cs`、`Services/IntelTuning.cs`、`Modules/CpuVendorModules.cs:68`、`Worker/BackendHostService.cs:221` |
| 电源方案 | 枚举/切换/派生新建/改名/删除 Windows 电源计划（内置方案禁删）；AC+DC 各 8 项策略：EPP、Boost、最大/最小处理器状态、散热方式、核心停泊下限、空闲禁用、ASPM；逐项写、逐项报错后重新激活 | 已实现可用（部分项随硬件缺失，快照按项给 supported/error） | `Services/PowerPolicyService.cs`、`Services/PowerPolicyApi.cs`、`Ipc/PowerSchemeListPayload.cs` |
| 进程调度 | 按进程名匹配规则；CPU Sets 或 Affinity Mask 二选一下发；6 档优先级（含 EcoQoS，不含 RealTime）；1s 扫描、失败重试账本、漂移探测、PID 复用识别、解绑；自定义掩码最多 10 套 | 已实现可用；但单次写入只覆盖一个处理器组（>64 逻辑处理器直接拒绝） | `Services/ProcessWatcher.cs`、`Services/CpuSetScheduler.cs`、`Services/ProcessPriority.cs`、`Services/MaskBinding.cs:14` |
| 监控 | 仅进程维度：PID、映像名、路径、CPU 占用（30s 滑动窗口）、优先级类；SMU 快照给 CO/FMax；系统电源 AC/DC 状态 | 部分实现：无温度/频率/功耗/GPU/内存/磁盘/网络，无历史持久化，无采样背压与过期策略（仅 2s 快照复用） | `Services/ProcessRefresher.cs:178`、`Ipc/ProcessEntryPayload.cs`、`Services/SmuReader.cs:511` |
| 场景自动化 | 4 种触发（进程运行/进程退出/电池/交流）；动作 = 调校方案 + 调度方案（或掩码）+ 电源方案；命中规则按 `Priority` 升序取第一条，一条规则同时决定三块；800ms 防抖 | 部分实现：**退出不恢复**；无条件、无持续时间、无按领域优先级、无用户接管识别 | `Services/ScenarioService.cs:439-456,738-775`、`Services/ScenarioMonitor.cs:57` |
| 模块与能力发现 | 进程内静态模块描述符 + DI 注册；8 个模块（core.runtime、amd.cpu、intel.cpu、generic.scheduling/power/plan/scenario/shell）；能力目录 5 位状态（Installed/Enabled/HardwareSupported/RuntimeAvailable/Active）；`query-capabilities` 下发快照 | 部分实现：无清单文件、无磁盘发现、无启用/禁用、无独立加载、无健康/降级，全部同进程同容器 | `Capabilities/*`、`Modules/*`、`Capabilities/CapabilityIds.cs`（10 项能力） |
| 界面设置 | 主题（System/Light/Dark）、跟随系统强调色/自定义色、窗口材质、动效、自启形态、关闭行为、启动应用调校、监听进程、错误通知、Toast 时长、防砖、休眠感知 | 已实现可用；但数据被拆到两个进程各写一部分（见 §4.3） | `Helpers/UiSettingsStore.cs`、`UI/ViewModels/AppSettings.cs` |
| 日志诊断 | 内存环形缓冲 200 条 + 按天落盘（`worker-`/`zentwik-` 前缀分离），留存 14 天；`snapshot-logs` 文本协议（首行统计 + 最近 200 条）；清空、导出到桌面、自检报告 | 已实现可用；无结构化字段、无操作/请求关联 ID | `Services/LogService.cs`、`Services/LogOptions.cs`、`Worker/PipeServerService.cs:373` |
| 托盘 | Worker 自建托盘（隐藏顶层窗口 + Shell_NotifyIcon + 原生菜单）；菜单可切三类方案；显示/退出界面；跟随设置显示或隐藏 | 已实现可用 | `Worker/Tray/TrayHost.cs`、`Worker/Program.cs:153-226` |
| 开机自启 | `schtasks /Create /XML` 创建计划任务 `ZentwikAutoStart`，四种形态、登录延时、最高权限；创建后回查；启动时自愈路径 | 已实现可用 | `Services/AutoStartService.cs:50,199-222` |
| 安装升级 | Inno Setup 两档安装包（完整/精简）+ 两种便携 zip；覆盖安装即升级；启动时迁移旧目录 ZenForge/Zenwik→Zentwik；卸载删计划任务与可选配置 | 部分实现：无自动更新（「检查更新」只打开 Releases 页）；不安装 PawnIO 驱动；卸载不恢复已写入的 CO/FMax | `installer/zentwik.iss`、`Helpers/AppPaths.cs:54`、`UI/Views/SettingsView.xaml.cs:160` |
| 单实例 | Worker `Global\` 互斥体；UI `Local\` 互斥体 + 激活管道（show/exit） | 已实现可用 | `Worker/Program.cs:26`、`UI/Services/SingleInstanceGate.cs` |

补充边界（影响后续设计判断）：

- 三类方案在数据结构上已拆开，但应用动作仍共享 `PlanService` 一个入口与一份 `plans.json`（`PlanFile` 三个 ActiveId 平铺）。
- 调校方案只存 CO/FMax，不含 FMax 出厂基线（基线属于硬件属性，单独在 `tuning.json`）。
- 场景的电源动作只切「系统电源计划」，不写策略参数；调度动作可引用调度方案或单套掩码，两者语义不同。

## 2. IPC 表面

### 2.1 命令字（46 条，`WorkerPipeContract`）

| 组 | 命令 |
|---|---|
| 生命周期 | `ping`、`shutdown`、`open-ui`、`register-autostart`、`unregister-autostart` |
| 调校 | `apply-tuning`、`query-smu-status`、`query-smu-snapshot`、`query-intel-tuning`、`apply-intel-tuning`、`reset-tuning-safe` |
| 能力/拓扑 | `query-capabilities`、`query-cpu-topology` |
| 三类方案 | `query-plans`、`query-active-plans`、`save-plan`、`delete-plan`、`rename-plan`、`apply-plan` |
| 进程/调度 | `query-processes`、`clear-all-schedules`、`save-scheduling-rule`、`delete-scheduling-rule`、`query-scheduling-rules` |
| 掩码 | `query-masks`、`save-mask`、`delete-mask`、`query-mask-usage` |
| 场景 | `query-scenario-rules`、`save-scenario-rule`、`delete-scenario-rule`、`query-scenario-state`、`set-scenario-enabled` |
| 电源 | `query-power-policy`、`apply-power-policy`、`query-power-schemes`、`set-active-power-scheme`、`create-power-scheme`、`rename-power-scheme`、`delete-power-scheme` |
| 设置/诊断 | `query-ui-settings`、`save-ui-settings`、`snapshot-logs`、`clear-logs`、`query-boot-guard`、`factory-reset` |

反向激活管道另有 2 条纯文本指令：`show`、`exit`（Worker 托盘 → UI）。

### 2.2 载荷 Type（Payload 是字符串，形状按命令解释）

| 命令组 | 请求载荷 | 响应载荷 |
|---|---|---|
| ping / shutdown / open-ui / query-* | 无 | 无 / `pong` / `bye` |
| register-autostart | `AutoStartMode` 枚举名裸字符串（空 = `WorkerWithUI`） | 无 |
| apply-tuning | 方案 Id 或 null（null = 按盘上 `tuning.json`） | 状态字 `applied`/`skipped`/`failed` |
| apply-intel-tuning | `ApplyIntelTuningPayload`（字段可空，`restore=true` 回基线） | 写后重读的 `IntelTuningPayload` |
| query-smu-status | 无 | `SmuStatusPayload`（available/error/vendor/intelTuning/elevated） |
| query-smu-snapshot | 无 | `TuningSnapshotPayload`（逐核 + FMax + 可写区间） |
| query-plans | `PlanQueryPayload{kind}` | `PlanListPayload{activeId,plans[]}` |
| save-plan | `SavePlanPayload` | 新方案 Id |
| apply/delete-plan | `PlanRefPayload{kind,id}` | 状态字 |
| rename-plan | `RenamePlanPayload` | 无 |
| clear-all-schedules | 进程 Id 字符串或 null（null = 全部解绑） | 无（Message 带数量） |
| save-scheduling-rule | `SaveScheduleRulePayload{processName,targetType,targetValue,priority}` | 无 |
| delete-scheduling-rule | 规则 Id（匹配不到回退按进程名） | 无 |
| save-mask / query-mask-usage | `SaveMaskPayload` / `{maskId}` | `MaskListPayload` / `MaskUsagePayload` |
| save-scenario-rule | `ScenarioRulePayload` | 无 |
| apply-power-policy | `ApplyPowerPolicyPayload`（10 个可空字段） | 写后快照 |
| create/rename-power-scheme | 对应 Payload | 刷新后的 `PowerSchemeListPayload` |
| save-ui-settings | `UiSettingsPayload`（增量） | 写后全量 |
| snapshot-logs | 无 | 纯文本：首行统计 + 200 行明细 |

### 2.3 协议与传输事实

- 报文：UTF-8 无 BOM 单行 JSON，字段仅 `Version`/`Success`(响应)/`Command`/`Payload`；`Payload` 是字符串，形状由命令自行解释（`Ipc/PipeRequest.cs`、`Ipc/PipeResponse.cs`）。
- 上限：单条 512 KB；Worker 读侧硬上限、客户端写/读两侧都校验（`WorkerPipeContract.cs:52`、`Worker/PipeServerService.cs:2983`）。
- 版本：全局整数 23，任一端不匹配即拒绝，无降级；`Version=0` 视作未赋值（`PipeRequest.cs:26`、`WorkerPipeContract.cs:694`）。
- 连接模型：`maxNumberOfServerInstances: 1`，一个连接只处理一请求一响应后关闭；UI 每次调用新建连接（`Worker/PipeServerService.cs:159`、`UI/Services/WorkerBridge.cs:120-148`）。无推送/订阅，界面全靠轮询：方案 30s + 窗口激活、进程页 3s。
- 超时：UI 默认 3s；失败一次即把状态灯置离线，再尝试拉起 Worker；启动等待 90s、失败冷却 30s。
- 单实例：Worker 用 `Global\Zentwik.Worker.SingleInstance`；UI 用 `Local\Zentwik.UI.SingleInstance`，跨 RDP 会话各自允许一个。

### 2.4 版本 23 的演进说明了什么

版本历史全部写在 `WorkerPipeContract.cs:23-36`，可读出三点结构性事实：

1. **协议即功能清单**：每一版都是「加命令 / 改字段」，没有独立的契约版本或能力协商；`query-capabilities` 本身也要靠 v23 才能问出来，客户端无法在旧协议上发现新能力。
2. **删除与回退是常态**：v5 删 `apply-process-priority`、v22 删掉 v21 刚加的 `query-power-limit`/`apply-power-limit`；说明功能在协议层被直接增删，不做弃用期。
   - 功耗限制的状态需精确标注：曾实现（Intel `84bb998`、AMD `27dc193`，字段为持续/短时/平均 `stapm`/`fast`/`slow`，单位瓦），2.6.0-can7 由 `348712a` 整体移除、协议升到 22，**提交正文为空**（后续已通过工作日志调查还原决策背景，见 [LEGACY_POWER_CONTROL_INVESTIGATION.md](LEGACY_POWER_CONTROL_INVESTIGATION.md)）。因此既不能当作现有能力，也不能推断为「全平台技术上不可行」。
3. **线格式会整体改写**：v19 掩码从整数数组改成 128 位十六进制串、v9 把方案拆成三类、v12 改掩码下发方式。这类改动一律靠「同版本安装」兜底，代价是任意一端混装即整机不可用（固定文案「UI 与 Worker 版本不一致，请重新安装」）。

## 3. 进程与权限边界事实

| 维度 | UI | Worker |
|---|---|---|
| 清单/提权 | 无 `requestedExecutionLevel`（asInvoker） | `requireAdministrator`、`uiAccess=false` |
| 框架 | WPF `net10.0-windows`，反射序列化 | `WinExe`，Native AOT 发布（CI 命令行传 `PublishAot`） |
| 谁拉起谁 | `runas` 拉 Worker（`WorkerLauncher.cs:101-107`） | `Process.Start` 拉 UI，**继承提权令牌**（`Worker/Program.cs:341-348`） |
| 单实例 | `Local\` 互斥体 + 激活管道 | `Global\` 互斥体，抢不到即退出并写一行 `startup_crash.log` |
| 写权限面 | 仅自己的 `theme.json`/`ui-entry.json`/`ui-state.json`/日志 | SMU/MSR、`schtasks`、HKLM 电源项、全部业务 JSON |
| 管道 ACL | 客户端无校验 | 仅当前用户 SID 的 `ReadWrite`，显式 DACL |

由此产生的确定事实：

1. **UI 完整性级别取决于启动路径**：用户手动启动是 medium，被 Worker 拉起是 high；界面没有对此做任何区分或限制。
2. **管道调用方身份未验证**：Worker 以高完整性运行，DACL 只区分「同一用户」，不校验调用方 PID/签名/完整性级别。同一登录会话下的任意 medium 进程都能连上并下发 `apply-tuning`（写 SMU）、`register-autostart`（建高权限计划任务）。这是旧版最直接的安全缺口（事实陈述，非推测）。
3. **数据目录共享**：UI 与 Worker 共用 `%APPDATA%\Zentwik`，靠文件名与日志前缀隔离；两边都可能触发一次旧目录迁移（`AppPaths`，迁移锁 `Local\Zentwik.ConfigMigration`）。
4. **AOT 边界**：Core + Worker 全部走源生成 JSON（`IpcJsonContext`/`AppJsonContext`），反射重载在 AOT 下会运行期抛异常；UI 不受此约束。

## 4. 技术债与设计缺陷

1. **命令字平铺 + 巨型分派**：46 个 `const string` + 180 行 `switch` 集中在一个 3076 行文件（`Worker/PipeServerService.cs:285-463`）；新增功能必须同时改常量、switch、payload 类型、版本号与文档注释，漏一项就是静默失败。
2. **版本号是唯一兼容机制**：任一端升级即要求整机同版本；没有按命令协商、没有能力探测前置。`query-capabilities` 只能在 v23 之后问，形成「先要知道能力，才能问能力」。
3. **状态所有权分裂**：Worker 自称是 `ui.json`/`plans.json`/`schedules.json`/`masks.json`/`scenarios.json`/`tuning.json` 的唯一写入方（契约注释反复强调），但 UI 直接写 `theme.json`、`ui-entry.json`、`ui-state.json`；Worker 又用 `JsonDocument` 手工读 UI 拥有的 `ui-entry.json` 决定托盘菜单（`Worker/Program.cs:267-291`）。契约文档与实现不一致。
4. **`schedules.json` 双向读写**：`ProcessWatcher` 每 3s `Reload()` 一次磁盘（`ProcessWatcher.cs:1069-1085`），同时 Worker 是写入方；外部误改/半写文件会把内存中的正确规则整体替换掉。契约注释「每次查询都 Load 只是白做」与监听侧行为相反。
5. **场景无状态所有权**：`Notify` 在无规则命中时「刻意什么都不做」，明确不负责切回（`ScenarioService.cs:447-451`）；退出后调校、电源方案保持场景值，只有 PID 绑核通过 `ScenarioBindingLedger` 在解绑时释放。`PROJECT_BRIEF` 要求的「记录前值/来源/所有权/恢复资格」在旧版完全不存在。
6. **场景仲裁粒度错位**：一条命中规则同时决定调校 + 调度 + 电源三块（`ScenarioService.cs:727-730`），无法表达「游戏规则只改调度、插电规则只改电源」；`Priority` 只做整体排序，没有按领域冲突消解。
7. **调校落盘先于硬件写入，且契约注释与实际行为矛盾**：`PlanService.Apply` 先 `_tuning.Save(...)`（`PlanService.cs:586-592`）再 `ApplyFromStore`（:594-597）；而 `TuningStore.Save` 一进来就把 `AppliedAt` 设为当前时间（`TuningStore.cs:61-63`）。但 `WorkerPipeContract.cs:136` 注释写的是「`Success = true` 表示 SMU 写入已全部完成并回写 `AppliedAt`」——字段语义被文档定义为「成功后才回写」，实现却在写入前就打戳。SMU 失败时磁盘上是「新值 + 已应用时间」，与硬件实际值不一致，且下次开机会重试这套从未成功过的参数；这属于契约与实现不一致，不只是调用顺序问题。
8. **应用不可回滚**：电源方案先切换系统计划再写策略快照，策略写失败时计划已切走，只回一句拼接错误（`Worker/PipeServerService.cs:1116-1131`）；`factory-reset` 逐条删除、异常即中断，无事务、无中途回滚（`PipeServerService.cs:742-800`）。
9. **错误被大面积吞掉**：`BootGuard.Load` 读失败即当全新状态（`BootGuard.cs:161-166`）、各 Store `Load` 读失败即空档、`TuningStore.Load` 同；用户只看到「配置没了」，日志里没有可归因的失败链路。
10. **调度匹配维度不足**：规则只按进程名（忽略大小写），`Path` 字段存了但不参与任何判断（`SchedulingStore.cs:66-71`）；同一 exe 名不同来源无法区分，也没有用户/会话/签名维度。
11. **跨处理器组能力缺失**：表达层用 128 位 `CoreMask`，写入层只接受单个 `ulong`，>64 逻辑处理器直接拒绝（`MaskBinding.cs:14-27`）；多组 CPU Sets 属于「不在本次范围」。
12. **监控只有进程 CPU**：无温度、频率、功耗、GPU、内存、I/O、网络；无历史存储，翻历史只能看文本日志；`ProcessRefresher` 无周期采样，完全由查询驱动（`ProcessRefresher.cs:262-286`），「实时监控」在旧版不成立。
13. **模块系统是编译期清单**：模块是硬编码 `IModuleRegistrar`，能力来自 `ModuleDescriptor.Provides`；无清单文件、无启用开关、无隔离、无第三方入口；`Failed` 之外没有 Degraded/Unhealthy，失败只影响显示不影响重启恢复。
14. **i18n 完全缺失**：界面字符串硬编码中文，安装器只有 `ChineseSimplified.isl`；产品简报要求的 zh-CN/zh-TW/en-US 运行时切换无任何基础设施。
15. **诊断可观测性弱**：日志无结构化字段、无操作 ID、无「谁发起/目标/实际值」；`snapshot-logs` 用 `|` 分隔的文本协议，正文含 `|` 靠 `Split('|',3)` 规避（`PipeServerService.cs:1225` 附近），扩展字段只能靠约定。
16. **卸载/降级不留后路**：卸载只杀进程、删计划任务（`installer/zentwik.iss:67-70`），不写回 CO/FMax；卸载后硬件仍停在自定义调校值，用户无提示。
17. **命名与语义错位**：`clear-all-schedules` 实际是「解绑进程」（`RunClearAllSchedulesAsync`），不是清空规则；`ScheduleRule` 注释里还残留「从 schedules.json 换成 schedules.json」这类改名未清理的说明，说明协议与命名的历史包袱已影响到可维护性。
18. **并发与阻塞面**：`PipeServerService` 中大量 `await Task.Run(...)` 包同步 Win32 调用，单实例管道串行处理；任一命令卡住（如 `WaitForAvailability(60)`）就阻塞整条管道 60s，UI 默认 3s 超时后只能显示「未连接」。
19. **测试覆盖偏逻辑层**：36 个测试集中在编解码、掩码、规则键、漂移、护栏、契约上限；无进程绑核、无 SMU、无电源 API 的集成测试，硬件相关行为只能靠人工验收。
20. **配置文件原子写但不加锁**：`JsonFile.Write` 用「进程 ID + GUID 临时名 + `File.Move` 覆盖」（`JsonFile.cs:43-47`），解决了半截文件，但没有跨进程写序列化；UI/Worker 同写一个文件时仍由「最后写入者胜出」。

## 5. 迁移判定

### 5.1 可直接迁移（逻辑独立、边界清晰）

| 资产 | 理由 |
|---|---|
| `CpuTopologyProbe`、`MaskBinding`/`MaskFormatter`/`MaskResolver`、`CoreMask` | 纯计算 + 单个 Win32 查询，无宿主依赖；注意 `MaskBinding` 的单组限制要在新架构显式建模 |
| `CpuSetScheduler`、`ProcessPriority`、`NativeProcessList` | Win32 封装完整，含错误码、回读校验、实例时间戳；可原样做成平台 provider |
| `PowerPolicyApi`/`PowerPolicyService` | Win32 电源 API 封装 + 逐项校验文案；`PowerPolicySnapshot` 的「支持/值/错误」三元组可直接复用 |
| `AutoStartService` | `schtasks /XML` + 回查 + 路径自愈，边界清楚 |
| `LogService` + `JsonFile`（原子写） | 环形缓冲/批量落盘/临时文件替换可直接复用 |
| `BootGuard`、`SleepResumeWatcher`、`PowerStatusWatcher` | 单一职责、无 UI/协议耦合 |
| `CapabilityState`/`ModuleState` 五维模型 | 概念正确（安装/启用/硬件/运行/活动 + Failed），可直接作为新能力模型基线 |

### 5.2 需重写（与旧架构强耦合）

| 资产 | 理由 |
|---|---|
| `WorkerPipeContract` + `PipeServerService` 分派 | 46 条平铺命令、全局版本号、字符串 `Payload`、单请求连接；应改为版本化契约 + 按域分组 + 能力协商，不能逐条搬运 |
| `WorkerBridge` / `WorkerLauncher` | 每次调用新建连接、3s 超时、90s 启动等待、无推送；监控与场景需要事件流而非轮询，重写传输层 |
| `ScenarioService` / `ScenarioMonitor` | 无状态所有权、无退出恢复、单规则决定三域；按简报必须重建「操作对象/前值/目标值/来源/所有权/恢复资格」模型 |
| `PlanService` | 把磁盘持久化、生效标记、硬件应用三件事揉在一个类（780 行），且应用顺序有前值不一致问题；需拆成「定义 / 期望 / 观测 / 实际」四层 |
| `PipeServerService` 内各 `Run*` 处理器 | 与旧 payload 类型、旧命令名、旧 `Success+Message` 语义绑定；随契约一并重写 |
| UI 外壳（`App.xaml.cs` 1633 行、`MainWindow`、各 ViewModel 单文件） | 页面与 IPC 调用、诊断开关、截图/自检路径深度耦合；新架构的模块贡献 UI 区块无法在其上生长 |

### 5.3 需补全（旧版本本来就不完整）

| 缺口 | 旧版现状 |
|---|---|
| 监控体系 | 无温度/频率/功耗/GPU/内存/I/O/网络，无采样与历史分层，无背压与过期数据策略 |
| 自动化语义 | 无条件（AND/OR）、无持续时间、无按域优先级、无用户手动接管、无执行历史、无幂等与重启恢复 |
| 模块/插件生命周期 | 无清单、无启用禁用、无独立进程、无权限声明、无版本协商、无第三方 API |
| Intel 调校闭环 | 属会话级能力：不持久化、不进方案、不参与开机应用与休眠补写；新项目要做 Intel 持久化属于新增能力，需要独立方案存储 + 重放链路 |
| 跨处理器组调度 | 表达层 128 位、执行层 64 位，>64 逻辑处理器不可用 |
| 设置所有权 | 设置分散在 Worker/UI 各自的 JSON，缺单一写入方与统一迁移 |
| i18n / 主题扩展 | 无资源化、无运行时切换、无模块资源扩展 |
| 安装/卸载安全 | 不装驱动、不校验签名、不恢复硬件默认值、无自动更新通道 |
| 可观测性 | 无操作审计（谁/何时/目标/实际值/失败步骤/恢复结果），仅文本日志 |

### 5.4 迁移前必须先定的事（由旧版直接暴露）

1. Worker 的权限来源：计划任务、Windows 服务还是按需提权；卸载/升级时如何回收。
2. 管道调用方鉴权：仅 DACL 不够，需要 PID→SID/签名/完整性校验，或改为 Service + 受控 RPC。
3. 配置单一写入方：`ui.json` 与 `theme.json`/`ui-entry.json`/`ui-state.json` 必须收拢，否则新架构会复制旧版的同步问题。
4. 硬件状态与期望状态分离：调校必须能表达「磁盘值 / 请求值 / 回读值 / 实际值」，否则旧版 §4.7 的问题会原样带入。
5. 场景退出语义：先定义「不恢复」「恢复」与「用户接管」的优先级，再决定是否迁移旧规则数据结构。

## 6. 源码导航（供其他 Agent 直接引用）

| 关注点 | 入口文件 | 说明 |
|---|---|---|
| 协议契约 | `Core/Ipc/WorkerPipeContract.cs` | 46 命令、版本、上限、名称 |
| 管道服务 | `Worker/PipeServerService.cs` | 接受循环、ACL、命令分派、全部处理器 |
| Worker 启动 | `Worker/Program.cs`、`Worker/BackendHostService.cs` | 互斥体、模块启动顺序、托盘、拉起 UI |
| UI 通信 | `UI/Services/WorkerBridge.cs`、`WorkerLauncher.cs` | 单次连接、超时、提权拉 Worker |
| 调校 | `Core/Services/SmuReader.cs`、`TuningApplyService.cs`、`IntelTuningChannel.cs` | AMD SMU / Intel MSR 两条独立通道 |
| 调度 | `Core/Services/ProcessWatcher.cs`、`CpuSetScheduler.cs`、`ProcessPriority.cs` | 扫描、规则匹配、绑定、优先级 |
| 掩码 | `Core/Helpers/MaskStore.cs`、`Core/Services/MaskBinding.cs` | 10 套上限、128→64 位收敛 |
| 电源 | `Core/Services/PowerPolicyService.cs`、`PowerPolicyApi.cs`、`PowerSchemePreset.cs` | 策略读写、方案 CRUD |
| 方案 | `Core/Services/PlanService.cs` | 三类方案、ActiveId、应用编排 |
| 场景 | `Core/Services/ScenarioService.cs`、`ScenarioMonitor.cs` | 判定/仲裁、事件源与防抖 |
| 状态账本 | `Core/Services/AppliedTargetLedger.cs`、`ScenarioBindingLedger.cs`、`ScheduleDriftTracker.cs`、`ProcessInstanceLedger.cs` | 去重、归属、漂移、PID 复用 |
| 持久化 | `Core/Helpers/JsonFile.cs`、`*Store.cs`、`AppPaths.cs` | 原子写、各域存储、目录迁移 |
| 能力/模块 | `Core/Capabilities/*`、`Core/Modules/*` | 目录、状态位、DI 注册 |
| 自启/托盘 | `Core/Services/AutoStartService.cs`、`Worker/Tray/*` | 计划任务、原生托盘 |
| 日志 | `Core/Services/LogService.cs`、`LogOptions.cs` | 环形缓冲、按天文件、留存 |
| 安装 | `installer/zentwik.iss`、`build/zip_portable.py` | 安装包、便携包、卸载动作 |
| 构建验证 | `.github/workflows/aot-probe.yml`、`release.yml` | AOT 探针、发布流水线 |

## 7. 未确认

- PawnIO 驱动本身的安装/签名校验流程不在本仓库内（只加载 `PawnIO\*.bin` 模块，见 `Smu/SmuSession.cs:28-34`）；驱动分发方式未确认。
- 旧版是否在真实 >64 逻辑处理器机器上验证过 CPU Sets 行为：代码路径明确拒绝高位掩码，但无测试覆盖跨组场景。
- `build/uitest`、`build/workertest` 下的产物是否与当前源码同步：未核对。
