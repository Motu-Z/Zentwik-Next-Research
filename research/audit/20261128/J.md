# 分片 J：知识与研究文档（`knowledge/*` 与 `research/*` 可验证断言）只读静态审计

- **旧项目基准**：`Motu-Z/Zentwik@main` (`727462601911609e6dacbe3cb8f144246829087a`)
- **新项目对照**：`Motu-Z/Zentwik@feat/gui-protocol` (`100e9c0f4f9e7f9b160f8b0bed9ea99afdf20922`)
- **研究文档基准**：`Motu-Z/Zentwik-Next-Research@main` (`7680e5109ed010e4681180393ff85fb2ce412554`)

## 1. 闭合计数

- **扫描的知识与研究文档数**：**39 篇**（`knowledge/` 26 篇 + `research/` 13 篇，另核对 `knowledge/INDEX.md`）。
- **可验证断言总数**：**38 条**（`J-01` ~ `J-38`）。
- **状态分布**：`一致` 10、`部分` 11、`缺` 1、`新有旧无` 12、`文档与现实冲突` 4（含 `Deprecated`/过时残留 3 条：`J-04`、`J-30`、`J-35`）。

## 2. 核心可验证断言核验表

| 编号 | 文档要求(路径:行) | 旧项与证据(文件:函数/行) | 新项与证据(文件:函数/op) | 状态 | 验证级别 | 备注 |
|---|---|---|---|---|---|---|
| J-01 | `amd-smu.md:22-23`, `ENGINEERING_NOTES.md:53-55`：PawnIO 路径与 IOCTL `0x821`/`0x841` | `PawnIoDevice.cs:12-36` | `PawnIoSession.cs:10-22` | 一致 | 静态源码（文档 `Verified`） | `observed` |
| J-02 | `amd-smu.md:72`：`0x01` 参数 `[1]` 回 `2` 探针 | `SmuSession.cs:8,39-46` | `AmdSmuTuningProvider.cs:14,914-915` | 一致 | 静态源码（文档 `Inferred`） | `observed` |
| J-03 | `amd-smu.md:30-36`：Zen2/3 vs Zen4/5 命令号 | `SmuMessages.cs:17-22` | `AmdSmuTuningProvider.cs:1139-1152` | 一致 | 静态源码 | `observed` |
| J-04 | `amd-smu.md:59-66`：`[23:20]` 4 位数值核掩码（含 `Deprecated` 独热记录） | `Smu/CoreMask.cs:9-18` | `AmdCoreMap.cs:73-79` | 一致 | 静态源码（文档 `Verified`） | `observed` |
| J-05 | `amd-curve-optimizer.md:55,78,80`：L2 `[-50,+30]` 无 mV 换算 | `SmuReader.cs:318-319` (`±50`) | `Plans.cs:95-103` (`[-50,+30]`) | 部分 | 静态源码 | `observed` |
| J-06 | `amd-curve-optimizer.md:27-29`, `amd-smu.md:52`：Curve Shaper `0xA6/0x84` 与全核 `0x07/0x36` | 无 | 无 | 缺 | 静态源码 | `observed` |
| J-07 | `amd-smu.md:76-90`, `units.md:17`：明确区分 FMax 绝对频率与 Boost Clock Override 偏移（`-1000..+200 MHz`） | `SmuReader.cs:322-329` (绝对 `4450–5650`) | `AmdSmuTuningProvider.cs:203-252` (绝对 `1000–7000 MHz`，工具支持范围) | 缺（FMax Override 偏移参数未实现） | 静态源码 | `observed`：原判「文档与现实冲突」属误判，正确结论为「FMax Override 偏移参数未实现」；补充：产品 commit `2d9351d` 已改为绝对频率钳制写入 |
| J-08 | `amd-power-limit.md:29-38,133`：RSMU `0x56/0xCB/0x57/0x58` 与 MP1 拒写 | 已删除 | `AmdSmuTuningProvider.cs:22-25,797-802` | 新有旧无 | 静态源码 | `observed` |
| J-09 | `amd-power-limit.md:49-65,120`：PM 表 `0x03` 重试 3 次与 `1 W` 下界 | 已删除 | `AmdSmuTuningProvider.cs:716-773` | 新有旧无 | 静态源码审计 + 引用既有实测记录（非本轮验证） | `observed`：原判混写 `9955HX 实测` |
| J-10 | `amd-power-limit.md:71-96`：写后四态、`±2 W` 容差、探针、`mW/mA` | 已删除 | `PowerLimits.cs:38-204`, `AmdSmuTuningProvider.cs:378-555` | 新有旧无 | 静态源码审计 + 引用既有实测记录（非本轮验证） | `observed`：原判混写 `9955HX 实测` |
| J-11 | `ENGINEERING_NOTES.md:72`：`Global\Access_PCI` | 进程内锁 | `AmdSmuTuningProvider.cs:36-37,989`, `AmdSensorSource.cs:26-28,378` | 新有旧无 | 静态源码 | `observed` |
| J-12 | `amd-power-limit.md:176-243`：SMN `0x59800` + RAPL `0xC0010299/9B` + APERF `0xE8` | 无 | `AmdSensorSource.cs:17-25`, `AmdSensorDecoder.cs:9-57` | 新有旧无 | 静态源码审计 + 引用既有实测记录（非本轮验证） | `observed`：原判混写 `9955HX 实测` |
| J-13 | `intel-oc-mailbox.md:72-94`：`0x150` 三平面、`S11.0.10`、bit63 busy | `IntelTuning.cs:20-54`（3 平面，无 busy） | `IntelMsrBits.cs:10-43`, `IntelMsrTuningProvider.cs:77-113`（有 busy，仅 Core） | 部分（仅模拟验证） | 静态源码 + Fake | `observed` |
| J-14 | `intel-oc-mailbox.md:75`：`Global\Access_Intel_OC_Mailbox` | 进程内锁 | `IntelMsrTuningProvider.cs:14,50,287-328` | 新有旧无（仅模拟验证） | 静态源码 + Fake | `observed` |
| J-15 | `intel-oc-mailbox.md:103,124`：2 mV 容差与 `PersistsAcrossReboot=false` | `IntelTuning.cs:126` | `IntelMsrTuningProvider.cs`, `PlanCoordinator.cs:1199` | 部分（仅模拟验证） | 静态源码 + Fake | `observed` |
| J-16 | `research/02:94-95`, `sensors/README.md:25-30`：Intel `0x1AD`、`0x610`、`0x19C/0x1B1` | `0x1AD` 读写；无功耗/传感器 | `IntelMsrBits.cs:11,49-55`（仅 `0x1AD` 1C/8C 只读） | 部分 | 静态源码 | `observed` |
| J-17 | `power-management.md:19-28`：`PowerSetActiveScheme` 与 `ErrorNoMoreItems=259` | `PowerPolicyApi.cs:79,548` | `Win32PowerPolicyService.cs:13,48,121-122` | 一致 | 静态源码 | `observed` |
| J-18 | `power-management.md:87-90`：注册表子键数区分真方案与 overlay | `PowerEnumerate` | `Win32PowerPolicyService.cs:95-100` | 新有旧无 | 静态源码审计 + 引用既有实测记录（非本轮验证） | `observed`：原判混写 `9955HX 实测` |
| J-19 | `power-management.md:111-113`：`powercfg -export` + JSON 备份恢复 | 无 | `PowerSchemeBackupStore.cs`, `PowerSchemeService.cs` | 新有旧无 | 静态源码审计 + 引用既有实测记录（非本轮验证） | `observed`：原判混写 `9955HX 实测` |
| J-20 | `process-scheduling.md:37-38`：`SYSTEM_CPU_SET_INFORMATION.Type` 4 字节 | `CpuSetScheduler.cs:126,1290` (2B) | `CpuSetsNative.cs:121-122,139` (4B) | 部分 | 静态源码 | `observed` |
| J-21 | `process-scheduling.md:36-42`：亲和性互斥、>64 核、`STILL_ACTIVE`、`CPMINCORES` | `CpuSetScheduler.cs`, `ProcessWatcher.cs` | `CpuSetsNative.cs`, `AffinityScheduler.cs`, `PlanCoordinator.cs:711,786` | 一致 | 静态源码 | `observed` |
| J-22 | `security-service.md:20-21,57`, `ENGINEERING_NOTES.md:99-104`：1066 与 `SERVICE_FAILURE_ACTIONS_FLAG` | 无 SCM | `WindowsServiceHost.cs:125` (1066), `ServiceInstaller.cs:526-551` (缺 Flag=4) | 文档与现实冲突 | 静态源码 | `observed` |
| J-23 | `security-service.md:26-31,56`：管道 DACL、拒绝远程、首实例、调用方校验、客户端 `Identification` | 仅当前用户 DACL | `PipeSecurityFactory.cs`, `PipeCaller.cs`, `AgentServer.cs:96-104`（缺 `FirstPipeInstance`/`RejectRemoteClients`）, `AgentConnection.cs:41` | 部分 | 静态源码 | `observed` |
| J-24 | `ipc.md:13-16,21,32`：4B 长度前缀、v2、38 `Ops` | v23 单行 JSON | `Protocol.cs:11-67,152-188` | 一致 | 静态源码 | `observed` |
| J-25 | `ipc.md:20`：遥测队列 2 帧背压与 `droppedSinceLast` | 无 | `AgentSession.cs:27,33-34,620-624`（阈值 256，无 `droppedSinceLast` 下发） | 部分 | 静态源码 | `observed` |
| J-26 | `ipc.md:26,28`：稳定错误码与退出码 0..6 | 无 | `Protocol.cs:75-110`, `Output.cs:6-15` | 新有旧无 | 静态源码 | `observed` |
| J-27 | `ENGINEERING_NOTES.md:144-149`：NativeAOT 源生成 JSON | 源生成 | `ZentwikJson.cs:15-50` | 一致 | 静态源码 | `observed` |
| J-28 | `serialization-persistence.md:13-15`：分文件持久化 + `SchemaVersion` + 原子替换 | 分文件，无 `SchemaVersion` | `StateStore.cs:58-61,123-127`, `StateDocument.cs:9-28`（合并单文件 `state.json`） | 文档与现实冲突 | 静态源码 | `observed` |
| J-29 | `three-plans.md:29`：`planType` 多态判别 | 无 | `Plan.cs:26-30` | 新有旧无 | 静态源码 | `observed` |
| J-30 | `three-plans.md:34`：移除 `PlanRevision` 引用（含过时表述） | 无 `PlanRevision` | `Scene.cs:20-24` 已删 `PlanRevision`；`Plan.cs:36` 留 `Revision` | 部分 | 静态源码 | `observed`：文档残留旧表述 |
| J-31 | `scenes-arbitration.md:17-36`：四类触发与降序优先级 | 升序优先级，退出不恢复 | `Scene.cs:3-18,41-42`, `SceneEngine.cs:39-48` | 部分 | 静态源码审计 + 引用既有实测记录（非本轮验证） | `observed`：原判混写 `2026-11-26 实测` |
| J-32 | `state-ownership.md:24,31-36`：账本所有权、接管与回读失败状态 | 无 | `StateLedger.cs:3-25`, `PlanCoordinator.cs:1517-1522` | 新有旧无 | 静态源码 + Fake 单测 | `observed` |
| J-33 | `polling-budget.md:18-22,102-125`：轮询预算与 3600 帧环形缓冲 | 1s/3s 固定轮询 | `SchedulingWatcher.cs:16,150`, `UserSettings.cs:33-44`, `TelemetryHub.cs:77`, `TelemetryRingBuffer.cs:12` | 新有旧无 | 静态源码审计 + 引用既有实测记录（非本轮验证） | `observed`：原判混写 `2026-11-26/27 实测` |
| J-34 | `modules-capabilities.md:30,33`：七态、`AnyOf`、`Optional` | 五维模型 | `Capability.cs:4-13`, `ModuleCatalog.cs:11-54` | 部分 | 静态源码 | `observed` |
| J-35 | `installer.md:3,24-42`：Inno 6、目录、PawnIO 探测（含过时 `zentwik-next.iss`） | `installer/zentwik.iss` | `scripts/installer/zentwik.iss`, `DataDirectoryMigration.cs:10` | 一致 | 静态源码 | `observed` |
| J-36 | `research/03`, `research/04`, `Gemini-Hydrogen` 交接：声称桌面端为 WPF (`MainWindow.xaml`) | `Zentwik.UI` (WPF) | `Zentwik.Desktop.csproj:6` (WinForms `MainForm.cs`) | 文档与现实冲突 | 静态源码 | `observed` |
| J-37 | `numeric-semantics.md:94`, `naming-conventions.md:21,31`：CO 用步进不用 mV、`InvariantCulture` | `CoreMargins` / `milliVolts` | `Offset` / `millivolts` / `InvariantCulture` | 一致 | 静态源码 | `observed` |
| J-38 | `UI-IA-RECONCILIATION-20261128.md:14-24`：首页工作区+时间线 | 无监控 | 7 页签 `MainForm.cs` | 部分 | 静态源码 | `observed` |
