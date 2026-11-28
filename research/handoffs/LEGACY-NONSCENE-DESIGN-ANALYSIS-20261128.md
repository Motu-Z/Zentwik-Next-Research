# 旧版 Zentwik「非场景」领域设计分析（设计参考结论）

> **基线**：旧版 `Motu-Z/Zentwik` `main` @ `727462601911609e6dacbe3cb8f144246829087a`；新版对照 `Motu-Z/Zentwik` `feat/gui-protocol` @ `74df31a7b0b1f5fd267ab13b24278bf42bf91017`；研究索引 `Motu-Z/Zentwik-Next-Research` `main` @ `d50e39e`（`research/01-legacy-inventory.md` 与 `research/handoffs/LEGACY-VS-NEXT-AUDIT-20261128.md`）。  
> **定位**：本报告是场景设计分析（`d50e39e`）的姊妹篇，聚焦场景自动化之外的 6 大非场景领域，纯只读静态源码审计，不改产品代码、不跑构建、不给施工步骤、不替用户裁决。所有结论均区分 `observed`（源码直接可证）/ `derived`（基于多处源码推导）/ `unknown`（未获证据或需 Windows 真机验证），无证据处标 `[未找到]`。

---

## 1. 进程调度

### 旧版机制（`observed`）
- **全表刷新与滑窗缓存**：`ProcessRefresher.cs:24-31,178-189,423-535` 在独立后台线程跑单次 `CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS)`，以 `30s / 64 点` 环形队列（`CpuWindow`）计算 CPU%；路径解析（`QueryFullProcessImageNameW`，栈上 2048 字符缓冲 `:641-642`）按需缓存并在失败后设置 `30s` 冷却（`:132,184,536-560`）；通过 `TryGetSnapshot(maxAge)`（`:203`）复用近期快照。
- **六档优先级与 EcoQoS**：`ProcessPriority.cs:53-70,106-133,193-239` 定义 `High / AboveNormal / Normal / BelowNormal / Low / EcoQoS` 六档，刻意排除 `RealTime`（`:65-68`）。`OpenProcess` 同时请求 `PROCESS_SET_INFORMATION | PROCESS_QUERY_LIMITED_INFORMATION`（`:197`），避免受保护进程在 `SetPriorityClass` 报 `ERROR_ACCESS_DENIED(5)`（`:29-48`）；`EcoQoS` 先写 `Idle` 再调 `SetProcessInformation(ProcessPowerThrottling, ExecutionSpeed)`（`:214,228-239`），并识别 `87/1168` 为瞬态退出错误（`:274-275`）。
- **亲和性 / CPU Sets 写入与回读校验**：`CpuSetScheduler.cs:176-200,592-795` 默认走 `SetProcessDefaultCpuSets`（软提示，仅需 `PROCESS_SET_LIMITED_INFORMATION`，`:751`），在写入前调用 `GetProcessAffinityMask` 拦截外部窄亲和性冲突（`:794-825` 返回 `ErrorForeignAffinity = -3`）；显式走 `SetProcessAffinityMask` 时校验 `GetActiveProcessorGroupCount() <= 1`（`:577,594`）；每次写入后立即回读（`VerifyCpuSets` `:827` / `VerifyAffinityMask` `:903`），不符则判失败。`CpuTopologyProbe.cs:50-143` 用 `GetLogicalProcessorInformationEx(RelationProcessorCore)` 解析物理核掩码与 `EfficiencyClass`。
- **PID 复用防护**：`ProcessInstanceLedger.cs:13-100` 维护 `PID → CreationTime` 账本；`ProcessWatcher.cs:321-334,403-423` 每 60 轮（60s）全量扫描前调用 `TakeReused` 比对 `GetProcessTimes` 创建时间，若变化则调用 `ForgetInstance` 清空该 PID 在全部 8 个内部账本中的记录（读不到时跳过不误清，`:60-61`）。
- **调度漂移探测**：`ScheduleDriftTracker.cs:13-74` 记录 `(Pid, Name, Mask, ByAffinityMask)`；`ProcessWatcher.cs:356-378,445-470` 每 5 轮在 `_applyGate` **锁外**执行 `ProbeBinding` 回读比对，并在探测后二次校验 `TryGet` 防止账本并发变更造成误报（`:460-461`），日志经 `LogThrottle` 按 `drift|{Name}` 折叠（`:463-468`）。
- **失败重试与账本去重**：`ScheduleRetryLedger.cs:10-107` 对绑定失败的 PID 按指数退避重试（`baseDelayMs = 2000`，左移封顶 `60_000 ms`，最多 `maxAttempts = 5` 次，`:19,24,101-105`），避免永久占住「已处理」位或每轮刷错误；`AppliedTargetLedger.cs:13-61` 记录 `(Mask, ByAffinityMask, Priority)` 三元组，规则重载/方案切换重扫时三项未变直接跳过下发（`:30-33`）。

### 设计优点（按价值排序）
1. **多账本正交解耦 + 统一实例清理**：将「已处理 / 绑定 / 亲和性 / 场景占用 / 失败重试 / 漂移期望 / 目标去重 / 实例时间戳」拆为职责单一的轻量账本（`ProcessWatcher.cs:100-109`），并用 `PruneTo(alive)` 与 `ForgetInstance(pid)`（`:363-370,413-423`）保证进程退出与 PID 复用时零残留。
2. **权限最小化与防误判诊断**：`SetPriorityClass` 补齐 `QUERY_LIMITED_INFORMATION`、`SetProcessDefaultCpuSets` 降权至 `SET_LIMITED_INFORMATION`，并将 `87/1168`（进程瞬态退出）与 `5`（反作弊/PPL 拒绝）区分开，既减少误报 Toast，又给出可解释错误码（`ProcessPriority.cs:242-275`）。
3. **写前冲突拦截 + 写后回读校验 + 锁外漂移探测**：写 CPU Sets 前检查外部窄亲和性、写后强制回读核对、漂移回读放到 `_applyGate` 锁外执行且探测后二次校验账本快照，兼顾准确性与扫描线程不阻塞。
4. **失败指数退避与重扫去重**：`ScheduleRetryLedger` 的 5 次封顶退避避免对受保护进程无限重试刷屏；`AppliedTargetLedger` 在规则变更全量重扫时跳过未变化进程，减少无意义的跨进程内核调用。

### 与新版现状的一句话差异（`observed`）
- 新版 `feat/gui-protocol` @ `74df31a`（经 `40c049a`）已补齐 `process.list`（`NtQuerySystemInformation(5)` + `(PID, CreateTime)` 30s/64 点 CPU% 滑窗）与 `EcoQoS` 读写对称，但仍缺少失败指数退避账本（`ScheduleRetryLedger`）、重扫目标去重账本（`AppliedTargetLedger`）以及跨处理器组调度能力（`>64` 逻辑核限制与旧版相同，另桌面端仍缺掩码可视化编辑器）。

### 可借鉴条目
1. `ScheduleRetryLedger` 的有界指数退避（`baseDelayMs=2s`、上限 `60s`、`maxAttempts=5`）与 `PruneTo(alive)` 清理模式，可直接防止受保护进程在定时周期内反复报错刷屏。
2. `AppliedTargetLedger` 的 `(Mask, ByAffinityMask, Priority)` 目标三元组去重，在规则变更触发全量重扫时消除冗余跨进程写入。
3. `ProcessPriority.ApplyOne` 的 `PROCESS_SET_INFORMATION | PROCESS_QUERY_LIMITED_INFORMATION` 句柄权限组合及 `87/1168` 瞬态退出过滤语义。
4. `ProcessWatcher.ProbeDrift` 在锁外执行跨进程回读并在输出前二次校验账本快照（`_drift.TryGet`）的并发防误报模式。

---

## 2. 电源方案

### 旧版机制（`observed`）
- **方案 CRUD 与隐藏模板克隆**：`PowerPolicyApi.cs:27-460` 封装 `powrprof.dll`；`PowerSchemePreset.cs:42-99` 固化 Windows 4 个内置模板 GUID（`PowerSaver / Balanced / HighPerformance / UltimatePerformance`）及 `IsBuiltIn(guid)` 保护（按 GUID 而非名称判定，防止改名后被误删）。`PowerPolicyService.CreateScheme`（`PowerPolicyService.cs:481-513`）解决 Windows 隐藏模板坑：直接 `PowerDuplicateScheme` 派生 `PowerSaver/HighPerformance/UltimatePerformance` 会继承注册表隐藏标记导致方案切换后从 `PowerEnumerate` 消失，因此统一从 `Balanced` 派生、若模板为 `UltimatePerformance` 先叠加 `HighPerformance` 底座再 `CopySettings` 覆盖目标模板（`:493-502`），改名失败时执行补偿删除（`:504-508`），并同步复制描述（`:510,523-528`）。
- **删除保护与活动切换**：`DeleteScheme`（`:589-606`）禁止删内置 GUID；若目标正是当前活动方案，先调用 `SwitchAwayFrom`（`:609-622`）切到其他可用方案再删除；`SetActiveScheme`（`:448-457`）先核对 `SchemeExists` 再下发。
- **8 项 AC/DC 策略读回三元组与重新激活**：`PowerPolicySnapshot.cs:13-303` 为 AC 与 DC 共 16 个策略项分别建模 `(Value, Supported, Error)` 三元组（避免 `null` 混同「不支持」与「读取失败」）。`PowerPolicyService.cs:286-397,409-419` 逐项写值并在每轮写后调用 `RefreshActiveScheme()`（再调一次 `PowerSetActiveScheme` 强制电源管理器重载）。
- **与调校方案的生命周期隔离**：`PlanService.cs:19-108,574-659` 将 `PlanKind` 拆为 `Tuning / Scheduling / Power` 三类独立枚举与独立 `Active*Id`；电源方案在 `PlanEntry` 中仅保存 `PowerSchemeGuid` 与可选 `PowerPolicy`（`:60-85`），应用失败（如系统方案已被外部删除 `:639`）互不影响调校与调度分支。

### 设计优点（按价值排序）
1. **隐藏模板「Balanced 底座 + 分层覆盖」克隆策略**：精准规避 Windows `SYSTEM` 独占隐藏标记导致的「新建即消失」现象，同时通过 `HighPerformance → UltimatePerformance` 分层拷贝保留完整后台电源参数（`PowerPolicyService.cs:493-503`），且建名失败自动回滚删残留（`:504-508`）。
2. **`(Value, Supported, Error)` 三元组快照语义**：明确区分「硬件/固件不支持」「读取报错」「实际数值」，UI 据此决定渲染滑块还是展示不可用原因，消除了可空值语义歧义（`PowerPolicySnapshot.cs:13-22`）。
3. **内置 GUID 身份锚定与活动方案安全删改**：内置方案保护看 GUID 不看显示名，删除当前活动方案前自动切离，避免 `PowerDeleteScheme` 报非零错误。
4. **策略写入后显式 `RefreshActiveScheme` 刷新**：解决 Windows 电源管理器对已激活方案修改索引后不立即生效的平台特性（`PowerPolicyService.cs:19,409-419`）。

### 与新版现状的一句话差异（`observed`）
- 新版 `feat/gui-protocol` @ `74df31a` 补齐了旧版缺失的「注册表子键全枚举 + 双轨备份还原（`PowerSchemeBackupStore`）」，但尚未提供 `power.scheme.activate` 协议/UI 主动切换入口、8 项 AC/DC 策略的三元组快照与编辑入口，以及旧版的隐藏模板派生层。

### 可借鉴条目
1. `PowerPolicyService.CreateScheme` 的「`Balanced` 克隆 + 分层 `CopySettings` + 改名失败补偿删除」事务式派生流程。
2. `PowerPolicySnapshot` 的 `(Value, Supported, Error)` 三元组模型，用于向 UI/CLI 准确表达硬件差异化支持度。
3. `DeleteScheme` 的「按 GUID 保护内置方案 + 删除前 `SwitchAwayFrom`」前置保护。
4. `PowerPolicyService.RefreshActiveScheme()` 的写后二次激活刷新机制。

---

## 3. CPU 调校

### 旧版机制（`observed`）
- **AMD CO / FMax 写入与容差回读**：`SmuReader.cs:318-478` 支持逐核 Curve Optimizer（软件输入钳制 `[-50, +50]`，`:318-319`）与绝对频率 FMax（`[4450, 5650] MHz`，`:322-323`）。写入 CO 时逐核下发并在全部写完后逐核回读校验，分离记录 `WriteFailed` 与 `VerifyFailed`（`:370-425,545-557`）；写入 FMax 时允许 `FMaxReadTolerance = 10 MHz` 容差以适配 SMU 硬件约 25 MHz 步进量化（`:326-329,467-470`）。`WaitForAvailability(maxSeconds=60)` 只重发测试消息不重建会话（`:159`）。
- **Intel 三平面电压 + 8 档睿频读写与基线恢复**：`IntelTuning.cs:18-294` 覆盖 OC Mailbox `0x150`（Core/GPU/Cache 三平面，按 `1.024` 缩放编码 `:29-59`）与 `MSR 0x1AD`（8 档睿频倍频 `:21,64-80`）。`IntelTuningSession` 写入电压后按 `VoltageTolerance = 2 mV` 逐平面回读（`:126,181-191`）；写入倍频后逐档比对（`:215-239`）；首次打开时抓拍 `_baseline` 快照，`RestoreBaseline = true` 时一键写回基线（`IntelTuningChannel.cs:199-205`）。
- **跨厂商抽象与状态机**：`IAmdTuningChannel.cs:7-80` 将公共生命周期抽象为 `ITuningChannel`（含 `CpuVendor` 与 5 态 `TuningChannelStatus`：`Unknown / Ready / NotSupported / DriverUnavailable / ProbeFailed`），而把 AMD SMU 与 Intel MSR 读写分别留在 `IAmdTuningChannel` 与 `IIntelTuningChannel`，刻意不强行统一异构数据结构（`:25-28,49-53`）。未命中厂商时通过 `UnsupportedTuningChannels` 占位（`CpuVendorProbe.cs` 探测）。
- **混合架构分组与防砖/休眠保护**：`MixedCorePlan.cs:11-48` 按 `EfficiencyClass` 将物理核拆分为 P/E 两组（多于两档时最高档归 P，其余全归 E，`:9,38-44`）。`BootGuard.cs:10-166` 实现两阶段提交：开机写调校前 `BeginWrite()` 置 `Pending=true` 并落盘，成功返回后 `MarkHealthy()` 清零；若连续 3 次在写入阶段崩溃/未完成（`MaxConsecutiveFailures = 3`，`:70`），下次启动 `Enter()` 自动置 `Blocked=true` 阻断开机自动应用。`SleepResumeWatcher.cs:17-80` 仅监听 `PowerModes.Resume`（忽略 `Suspend`）以在休眠唤醒后触发补写。

### 设计优点（按价值排序）
1. **开机连续崩溃防砖护栏（`BootGuard` 两阶段提交）**：以极低复杂度（单份 JSON 状态 + `BeginWrite / MarkHealthy`）彻底兜住激进 CO/FMax 导致开机蓝屏死循环的系统性风险（`BootGuard.cs:89-139`）。
2. **接口隔离原则（ISP）下的跨厂商抽象**：`ITuningChannel` 仅统一可用性状态机与 `WaitForAvailability`，`IAmdTuningChannel` 与 `IIntelTuningChannel` 各自保留原生语义，避免「最低公分母 DTO」污染两套硬件通道（`IAmdTuningChannel.cs:25-80`）。
3. **硬件量化感知的回读校验**：FMax 允许 `10 MHz` 步进量化容差（`SmuReader.cs:329,470`）、Intel 电压允许 `2 mV` 缩放舍入容差（`IntelTuning.cs:126,188`），并将「写失败」与「回读校验失败」拆开统计，减少硬件量化导致的假阴性。
4. **5 态诊断区分「平台不匹配」与「环境未就绪」**：`TuningChannelStatus` 明确拆出 `NotSupported / DriverUnavailable / ProbeFailed`（`IAmdTuningChannel.cs:7-23`），使 UI 与日志能准确区分「Intel 机器看 AMD 页」和「PawnIO 驱动未装」。

### 与新版现状的一句话差异（`observed`）
- 新版 `feat/gui-protocol` @ `74df31a` 在 AMD 侧显著增强（不对称 CO `[-50,+30]`、APOB 核映射、功耗墙并入 `CpuTuningPlan` 且于 `74df31a` 修正 mA/mW 单位换算），但尚未实现旧版的 `BootGuard` 开机防砖护栏、Intel GPU/Cache 双平面与 `0x1AD` 睿频写入（`IntelMsrTuningProvider` 仅支持 Core 电压写 + 1C/8C 倍频读）以及 Desktop 表单创建 FMax/Intel 方案的入口。

### 可借鉴条目
1. `BootGuard` 的「写前 `Pending=true` → 写后 `MarkHealthy()` → 连续 3 次未闭合自动 `Blocked`」开机防砖护栏机制。
2. `IAmdTuningChannel.cs` 的 `ITuningChannel`（状态机 + 等待）与厂商专属接口分离设计，以及 5 态 `TuningChannelStatus` 故障分类。
3. `SmuReader`（`10 MHz` 步进容差）与 `IntelTuningSession`（`2 mV` 缩放容差）的硬件量化容差回读模型。
4. `IntelTuningCodec` 中 `0x150` OC Mailbox（Core/GPU/Cache 平面位）与 `0x1AD` 8 档睿频编解码及 `RestoreBaseline` 流程（已有 `IntelTuningCodecTests` 覆盖）。
5. `MixedCorePlan.TrySplit` 按 `EfficiencyClass` 自动归并 P / (E+LP-E) 双组的纯逻辑算法。

---

## 4. 服务与 IPC

### 旧版机制（`observed`）
- **权限边界与宿主演进起点**：旧版采用「每用户提权 Worker（`requireAdministrator`）+ WPF UI」双进程模型；`PipeServerService.cs:274` 通过 `CreatePipeSecurity()` 为命名管道设置仅限当前用户 SID 的 `ReadWrite` ACL（注：不校验调用方 PID/签名/完整性级别，属已知安全短板，见 `01-legacy-inventory.md:113`）。`BackendHostService.cs:115-225` 统一编排后台模块启动/停止顺序（先修自启路径、再开机应用调校、再启休眠/电源/进程监听与采样器，停止时逆序回收）。
- **命名管道协议与契约集中定义**：`WorkerPipeContract.cs:14-716` 集中声明 46 条命令常量、全局整型 `ProtocolVersion = 23`、单条消息硬上限 `MaxMessageBytes = 512 * 1024`（`:52`）及统一版本校验 `ValidateVersion`（`:694`）；`PipeServerService.cs:159,287,2983` 以 `maxNumberOfServerInstances: 1` 串行收发 UTF-8 单行 JSON。
- **生命周期协作**：`WorkerLifetime.cs:23-36` 封装 `CancellationTokenSource`；`Program.cs:26-120` 用 `Global\Zentwik.Worker.SingleInstance` 互斥体保证全局单实例，并在抢锁失败时不调用 `ReleaseMutex()` 直接退出（`:33`）。
- **日志缓冲、Flush 屏障与频控折叠**：`LogService.cs:22-560` 实现内存环形缓冲（超出 `MaxEntries + TrimBatch` 成批裁剪头部 `:439-441`）+ 按天异步落盘 + 终身累加计数器（`TotalInfo/Warn/Error/Critical` `:154-162`）+ `LastIssue` 快速索引（`:169`）；其 `Flush()`（`:65-66,213-218`）通过写盘锁同步等待后台写盘线程当前批次完成，确保 `Critical()` 崩溃前日志真正落盘。`LogThrottle.cs:22-91` 按 `key` 维护时间窗抑制高频重复日志，到期后附带 `（前 N 次已折叠）`（`:89`）。

### 设计优点（按价值排序）
1. **`LogService.Flush()` 写盘线程同步屏障**：避免异步日志常见的「队列已被后台线程取走、Flush 见空队列立即返回、进程退出导致最后几条致命日志丢失」竞态缺陷（`LogService.cs:65-66,213-218`）。
2. **`LogThrottle` 按键滑动窗口折叠 + 抑制计数回显**：在保持异常可感知的同时防止 1s 进程扫描或漂移探测把日志文件撑爆，且通过 `SuppressedSuffix` 保留真实发生频次（`LogThrottle.cs:50-90`）。
3. **单向依赖的契约集中化与硬上限保护**：`WorkerPipeContract` 统一约束双端协议版本、512 KB 报文上限与错版文案，配合 `IpcJsonContext` 源生成序列化适配 NativeAOT，在旧版架构下做到了零反射 IPC（`WorkerPipeContract.cs:37-52,694-716`）。
4. **`BackendHostService` 显式有序启停**：将依赖顺序（BootGuard 检查 → 调校 → Watcher/Refresher 启动，停止时逆序关闭采样与监听）固化在一处，避免模块散乱自启（`BackendHostService.cs:115-190`）。

### 与新版现状的一句话差异（`observed`）
- 新版 `feat/gui-protocol` @ `74df31a` 在架构与安全上全面超越旧版（`LocalSystem` Windows 服务、4B 长度前缀多路复用协议、事件推送、SID/IL/Session 调用方鉴权、结构化 `DiagnosticLog`），但尚缺少旧版 `LogThrottle` 的高频日志折叠计数机制与 `LogService.Flush` 级别的显式落盘屏障（另 `ServiceInstaller` 仍缺 `1066` 非崩溃恢复 Flag）。

### 可借鉴条目
1. `LogThrottle` 的按键时间窗限流与 `SuppressedSuffix`（折叠次数回显）机制，可直接用于新版高频守护/遥测异常日志降噪。
2. `LogService.Flush()` 同步等待后台写盘批次完成的屏障设计，以及 `Critical()` 自动触发 `Flush()` 的契约。
3. `LogService` 在环形缓冲裁剪之外保留终身单调计数器（`TotalInfo/Warn/Error/Critical`）与 `LastIssue` 的轻量诊断摘要思路。

---

## 5. 设置与周边

### 旧版机制（`observed`）
- **配置持久化与原子写**：`JsonFile.cs:18-48` 采用「`{path}.{pid}.{guid}.tmp` 写入 + `File.Move(..., overwrite: true)`」原子替换；`UiSettingsStore.cs:23-222` 集中管理主题、强调色、自启形态、关闭行为、开机调校、进程监听、防砖、休眠感知等默认值与 `ResetToFactory()`；`AppPaths.cs:68,120-250` 使用跨进程互斥体 `Local\Zentwik.ConfigMigration` 协调 UI 与 Worker 启动时的旧目录配置迁移（含 `AbandonedMutexException` 容错 `:249`）。
- **开机自启与路径自愈**：`AutoStartService.cs:19-620` 使用 `schtasks.exe /Create /XML`（UTF-16 BOM 编码 `:201`）注册 `ZentwikAutoStart` 计划任务，显式注册 OEM 代码页读取中文输出（`:109-123`），设置 15s 超时强杀防挂死（`:609-618`），创建后强制 `/Query` 回查确认落地（`:214-218`）；并在每次 Worker 启动时调用 `RepairRegisteredPath()`（`BackendHostService.cs:119` / `AutoStartService.cs:263-290`）检查便携版移动目录或升级后路径漂移并自动重写任务。
- **单实例与反向激活**：`SingleInstanceGate.cs:15-105` 用 `Local\Zentwik.UI.SingleInstance` 互斥体保证每用户会话单实例；二次启动若抢锁失败，通过单向命名管道 `Zentwik.UI.Activate` 发送 `show` 唤醒已有窗口（`:92-98`），Worker 托盘亦可通过该管道发送 `exit`（`:68-74`）。
- **托盘策略与方案快捷菜单**：`TrayPolicy.cs:39-68` 以纯函数 `ShouldShowTray(AutoStartMode, CloseBehavior)` 与 `Decide(shouldShow, exists)` 统一判定托盘显隐；`Program.cs:153-263` 在 Worker 托盘动态构建三类方案（调校/调度/电源）勾选子菜单，并受 `NotifyOnTraySuccess` 控制静默提示（`:203`）。
- **自检报告与日志查看**：`LogsViewModel.cs:13-625` 提供级别过滤（All/Info/Warn/Error）、关键词搜索、文件夹体积统计、清空确认与 `BuildDiagnostics()` 诊断文本生成；`AdvancedViewModel.cs:240` 与 `SettingsViewModel.cs` 串联导出与自检报告。

### 设计优点（按价值排序）
1. **`AutoStartService` 的「创建后回查 + 启动时路径自愈 + 超时强杀」闭环**：直接覆盖 Windows 计划任务的三大真实坑点（便携版挪目录失效、`schtasks` 返回 0 但未落地、OEM 代码页乱码与子进程挂死）（`AutoStartService.cs:123,214,263,609`）。
2. **`SingleInstanceGate` 互斥体 + 命名管道激活**：以约 100 行零依赖代码同时解决「最小化到托盘后重复双击图标无法唤出窗口」与「Worker 托盘退出 UI」两个桌面刚需（`SingleInstanceGate.cs:25-98`）。
3. **托盘显隐纯函数策略化（`TrayPolicy`）**：将复杂的「自启模式 × 关闭行为 × 当前托盘是否存在」状态矩阵收敛为无副作用纯函数，便于单测穷举（`TrayPolicy.cs:47-68`）。
4. **`JsonFile` 原子写 + `AppPaths` 迁移互斥锁**：防止崩溃写坏 JSON 配置，并在双进程并发启动时安全完成旧目录迁移（`JsonFile.cs:43-47`，`AppPaths.cs:120-144`）。

### 与新版现状的一句话差异（`observed`）
- 新版 `feat/gui-protocol` @ `74df31a` 已把配置收拢为单一 `state.json` 并新增三语本地化与 `DoctorService`，但尚未实现 `Desktop` 端单实例互斥体与激活唤醒（`SingleInstanceGate`）、托盘方案切换菜单、以及旧版 `AutoStartService` 的创建后回查与启动路径自愈逻辑。

### 可借鉴条目
1. `SingleInstanceGate`（`Local\` 互斥体 + 激活命名管道）用于解决新版启用托盘后的多开冲突与唤起已有窗口问题。
2. `AutoStartService` 的 `RepairRegisteredPath()` 启动路径自愈、创建后 `/Query` 回查及 `15s` 超时强杀机制。
3. `TrayPolicy.ShouldShowTray` / `Decide` 的纯函数状态矩阵建模方式与托盘三类方案快捷切换菜单交互。
4. `JsonFile.Write` 的同目录随机临时文件 + `File.Move(overwrite: true)` 原子替换及 `AppPaths` 的迁移互斥锁（含 `AbandonedMutexException` 恢复）。

---

## 6. 可测性

### 旧版机制（`observed`）
- **纯逻辑单元测试（主干）**：`Zentwik.Tests/` 共 `36` 个测试文件、约 `267` 个 `[Fact]/[Theory]` 用例（统计自 `Zentwik.Tests/*.cs`），借助 `11` 个手写 Fake（`Zentwik.Tests/Fakes/Fake*.cs`：`FakeAppPaths`、`FakeCpuSetScheduler`、`FakeLogService`、`FakeMaskStore`、`FakePowerPolicyApi`、`FakeProcessWatcher`、`FakeSchedulingStore`、`FakeSmuReader`、`FakeTuningStore`、`FakeUiSettingsStore`、`FakeWorkerLifetime`）隔离磁盘、Win32 与驱动。核心逻辑覆盖：掩码与拓扑编解码（`CoreMaskTests`、`MaskFormatterTests`、`MaskResolverTests`、`CpuTopologyGroupingTests`、`MixedCorePlanTests`）、Intel/IPC 编解码与契约上限（`IntelTuningCodecTests`、`IpcLimitsTests`、`WorkerPipeContractTests`）、调度账本与漂移/退避（`AppliedTargetLedgerTests`、`ProcessInstanceLedgerTests`、`ScheduleDriftTests`、`SchedulingRobustnessTests`）、护栏与策略（`BootGuardTests`、`PowerPolicyTests` 52 用例、`ApplyTuningDispatchTests` 51 用例、`TrayAndSettingsTests`、`LogServiceBatchTests`）。
- **UI 进程内集成自检（`--itest`）**：`Zentwik.UI/Services/IntegrationTest.cs:17-70`（3187 行）采用无反射强类型调用，在真实 WPF 窗口实例化下断言外壳尺寸、令牌一致、暗/亮主题切换、命中测试、命令语义与虚拟化性能。
- **仅真机/人工验收（缺自动化闭环）**：`CpuTopologyProbeTests.cs` 仅 1 个基础测试，无真实 `SetProcessDefaultCpuSets` / `SetPriorityClass` / `PawnIO` SMU / `IntelMsr` / `powrprof.dll` 的真机集成测试，硬件闭环依赖人工验收（与 `01-legacy-inventory.md:137` 一致）。

### 设计优点（按价值排序）
1. **核心算法与账本「零 Win32 依赖」剥离**：将 `ScheduleRetryLedger`、`ScheduleDriftTracker`、`AppliedTargetLedger`、`ProcessInstanceLedger`、`BootGuard`、`TrayPolicy`、`MixedCorePlan`、`IntelTuningCodec` 全部做成纯内存/可注入时钟或委托的 POCO，使高风险状态机能以毫秒级纯单测覆盖边界条件（如 `ProcessInstanceLedger.TakeReused`、PID 复用、指数退避）。
2. **契约/边界测试锁死隐式不变量**：通过 `WorkerPipeContractTests`、`IpcLimitsTests`、`CoreMaskWireFormatTests`、`BootGuardTests`、`PowerPolicyTests`（52 个用例覆盖隐藏模板克隆与跨字段校验）把容易在重构中静默退化的规则固化为测试。
3. **轻量 `Fakes/` 目录替代重型 Mock 框架**：11 个接口对应 11 个可复用 Fake，避免动态代理对 NativeAOT / 裁剪分析的干扰，且测试语义直白可读。
4. **UI `--itest` 自检模式**：无需外部 UI 自动化驱动即可在目标机直接跑通布局、主题与命令链路自检（`IntegrationTest.cs:17-70`）。

### 与新版现状的一句话差异（`observed`）
- 新版 `feat/gui-protocol` @ `74df31a` 延续了「纯逻辑 + Fake 单测 + 少量 UI 冒烟（`MainFormSmokeTests`）+ 真机人工闭环」的同构分层（且新增了 `PowerLimitWireTests` 等单位换算锁死测试），但在电源方案 CRUD/隐藏模板与桌面交互行为测试密度上仍低于旧版（旧版 `PowerPolicyTests` 52 例、`IntegrationTest` 17 个章节）。

### 可借鉴条目
1. 将状态机、账本与策略判定（如退避、漂移、PID 复用、防砖、托盘策略）彻底剥离出平台 API 与定时循环，通过注入 `Func<int, long>` 或 `TimeProvider`/tick 参数实现 100% 确定性纯单测。
2. 沿用无动态代理的静态 `Fake*` 实现，保持与 NativeAOT/单文件发布及严格编译检查的兼容性。
3. 将隐式平台坑（如 `PowerPolicyService.CreateScheme` 隐藏模板派生、`BootGuard` 连续 3 次未闭合阻断、`IntelTuningCodec` 1.024 缩放）沉淀为高密度回归用例，防止后续重构退化。

---

## 跨领域通用设计原则（按价值排序）

1. **所有权单一化与状态账本解耦**：每个状态维度（实例生命周期、目标期望、重试退避、漂移基线、防砖计数）各自持有最小正交账本，统一响应 `PruneTo(alive)` 与 `ForgetInstance(pid)`，避免「单标志位复用」导致的漏绑、误报或永久卡死。
2. **写前校验 + 写后容差回读 + 失败显式分流**：硬件与内核写入（SMU、MSR、CPU Sets、优先级、电源索引）不可假设「返回成功即生效」；必须区分「权限/保护拒绝」「硬件步进量化偏差」「进程瞬态退出」「平台不支持」，分别给予对应处置与诊断文案。
3. **两阶段安全护栏与失败补偿回滚**：对高风险持久化变更（开机调校应用、电源方案克隆改名）先记待决/后核销，或失败时执行补偿清理（如 `CreateScheme` 改名失败即删除临时克隆），防止半成品状态跨重启残留。
4. **锁外探测与二次核对并发防线**：在周期轮询服务中，将昂贵的跨进程/内核回读移到核心状态锁之外，并在输出告警或动作前二次核对账本版本/快照是否已被并发修改，消除竞态误报与长耗时阻塞。
5. **接口隔离优于最低公分母抽象**：跨硬件或跨能力域抽象（如 `ITuningChannel` vs `IAmdTuningChannel` / `IIntelTuningChannel`）只共享状态机与生命周期，不强行合并异构参数模型，避免空字段蔓延与契约失真。
6. **边界逻辑纯函数化以换取零成本高覆盖可测性**：把托盘显隐（`TrayPolicy`）、核分组（`MixedCorePlan`）、掩码解析、退避计算、线格式编解码从宿主循环与 Win32 P/Invoke 中抽离，使绝大部分复杂分支能在无管理员权限、无真机硬件的环境中完成确定性回归。

---

## 计数闭合核对

| 核对项 | 要求范围 | 实际覆盖 | 缺项 |
|---|---|---|---|
| 分析领域 | 6 个非场景领域（进程调度、电源方案、CPU 调校、服务与 IPC、设置与周边、可测性） | **6 / 6 全覆盖** | `[未找到]`（无缺项） |
| 每节固定四块 | 旧版机制（文件:行 + 证据级别）、设计优点（按价值排序）、与新版现状的一句话差异、可借鉴条目 | **6 / 6 全覆盖** | 无 |
| 跨领域通用设计原则 | 按价值排序清单 | **6 条**（按价值降序） | 无 |
| 可借鉴条目数统计 | 领域 1(4) + 领域 2(4) + 领域 3(5) + 领域 4(3) + 领域 5(4) + 领域 6(3) | **共 23 条** | 无 |
