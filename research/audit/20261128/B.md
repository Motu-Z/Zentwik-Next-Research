# 分片 B：CPU 调校（CO / FMax / 电压 / 频率 / 方案 CRUD）只读静态审计

- **旧项目基准**：`Motu-Z/Zentwik@main` (`727462601911609e6dacbe3cb8f144246829087a`)
- **新项目对照**：`Motu-Z/Zentwik@feat/gui-protocol` (`100e9c0f4f9e7f9b160f8b0bed9ea99afdf20922`)
- **研究文档基准**：`Motu-Z/Zentwik-Next-Research@main` (`7680e5109ed010e4681180393ff85fb2ce412554`)

## 1. 闭合计数

- **旧项目调校命令/方法数**：底层 SMU/MSR 槽位与寄存器 **9**（5 SMU + 2 MSR 寄存器 + 2 邮箱命令字）+ 调校核心方法 **48**（SMU 13 + AMD 服务/存储 9 + Intel 方法 13 + 方案 CRUD/混合核 13）= **57** 个单元。
- **新项目调校 op/方法数**：协议 Ops **8**（`plans.*` 6 + `hardware.tuning.read/intel` 2）+ AMD SMU 命令与方法 **14**（5 命令 + 9 方法）+ Intel MSR 命令与方法 **13**（4 寄存器/命令字 + 9 方法）+ 协调与领域规则 **12** = **47** 个单元。
- **对照项总数**：**32 项**（`B-01` ~ `B-32`）。

## 2. 逐项对照表（含调用链与验证级别）

| 编号 | 文档要求(路径:行) | 旧项与证据(文件:函数/行) | 新项与证据(文件:函数/op) | 状态 | 验证级别 | 备注 |
|---|---|---|---|---|---|---|
| B-01 | `amd-smu.md:22-25,72`：PawnIO `RyzenSMU.bin`，`0x01` 参数 `[1]` 回 `2` 探针 | `SmuChannel.cs:TryGetVersion:12-21`；`SmuSession.cs:TryTestMessage:39-46` | `AmdSmuTuningProvider.cs:OpenAndProbe:887-920`（`plans.apply` / `hardware.tuning.read` → `PlanCoordinator` → `AmdSmuTuningProvider`） | 一致 | 静态源码（文档 `0x01` 实测出处 Unknown） | `observed` |
| B-02 | `amd-smu.md:29-35,55`：家族命令号映射（Zen2/3 vs Zen4/5），防 MP1 移动型号误发 | `SmuMessages.cs:For:17-24`；`CpuIdentity.cs:ResolveFamily:82-88` | `AmdSmuTuningProvider.cs:TryResolveCommands:1139-1158`、`RequiresMp1Mailbox:1132-1137` | 部分 | 静态源码（命令号文档 `Inferred`） | `observed`：新版补齐 MP1 拦截 |
| B-03 | `amd-smu.md:59-66`：`[31:28]` CCD、`[27:24]` CCX、`[23:20]` 4 位核内数值 | `Smu/CoreMask.cs:Encode:9-15` | `AmdCoreMap.cs:AmdCoreMask.Encode:73-80`；`AmdSmuTuningProvider.cs:TryWriteOffset:148` | 一致 | 静态源码（文档 `Verified` 上游源码） | `observed` |
| B-04 | `amd-smu.md:62-65`：优先 APOB，回退 `CPUID 0x8000001E` + L3 域 | `CpuIdentity.cs:CoresPerCcdFor:109-115`；`SmuReader.cs:SplitCore:492-498`（线性） | `AmdCoreMapResolver.cs:Resolve:11-34`；`AmdApobSource.cs:TryLoad:11-32`；`AmdWindowsTopology.cs:Resolve:63` | 部分 | 静态源码（无 APOB 路径文档 `Verified` 单机实测） | `observed`：旧线性推算，新严格校验 |
| B-05 | `amd-smu.md:9,91`：跨进程 PCI 互斥锁 `Global\Access_PCI` | `SmuSession.cs:_gate:11`（仅进程内锁） | `AmdSmuTuningProvider.cs:TryEnterPci:1023-1054`（`Global\Access_PCI`） | 部分 | 静态源码（`ENGINEERING_NOTES.md §2`） | `observed`：旧缺跨进程互斥 |
| B-06 | `amd-smu.md:92`：SMU 失败诊断与错误码区分 | `SmuChannel.cs:Send:28-50`（`TrySend` 吞 HRESULT） | `AmdSmuTuningProvider.cs:DescribeFailure:1074-1086`（转译 Win32 31/170/121/1359） | 部分 | 静态源码（文档 `Inferred`） | `observed` |
| B-07 | `amd-curve-optimizer.md:40-49,78`、`units.md:15`：CO 为无单位 `step` | `TuningStore.cs:14-15`；`SmuReader.cs:336` | `Domain/Plans.cs:CoreTuning.Offset:20-21`；`AmdSmuTuningProvider.cs:84,125` | 一致 | 静态源码 | `observed` |
| B-08 | `amd-curve-optimizer.md:25,51-59,80`：L2 不对称钳制 `[-50, +30]` | `SmuReader.cs:318-319`（对称 `[-50, +50]`） | `Domain/Plans.cs:CurveOptimizerBounds:95-104`（`[-50, +30]`） | 部分 | 静态源码 | `observed` |
| B-09 | `amd-curve-optimizer.md:18,113-114`：全核/CCD/逐核与跳过未配置核 | `TuningApplyService.cs:Write:175-178`（扁平 `int[]`） | `CurveOptimizerConfig.Resolve:154-194`；`PlanCoordinator.cs:ApplyCpu:1005-1048` | 新有旧无 | 静态源码 | `observed`：Desktop UI `BuildPlanFromForm` 亦支持 `default/ccd/core/auto` |
| B-10 | `amd-curve-optimizer.md:35,104`：写前读基线、写入、写后回读校验 | `SmuReader.cs:WriteCurveOptimizerCore:353-427` | `AmdSmuTuningProvider.cs:TryWriteOffset:125-170`；`PlanCoordinator.cs:ApplyCpu:982-1084` | 一致 | 静态源码 | `observed`：新补全协调层前值入账与回退 |
| B-11 | `amd-curve-optimizer.md:26-29,140`：全核 CO 命令 `0x07/0x36` 与 Curve Shaper `0xA6/0x84` | 无 | 无 | 缺 | 静态源码 | `observed` |
| B-12 | `amd-smu.md:36-49,75`：FMax `0x6E/0x70` 不可与 `0x5F/0x60` 混用 | `SmuMessages.cs:17-24`；`SmuSession.cs:81-104` | `AmdSmuTuningProvider.cs:TryReadFMax/TryWriteFMax:173-258`；`FMaxAbstractions.cs:4` | 一致 | 静态源码 | `observed` |
| B-13 | `amd-smu.md:76-90`：明确区分 FMax 绝对频率与 FMax Override（`-1000..+200 MHz` 有符号偏移） | `SmuReader.cs:322-329`（绝对频率 `[4450, 5650]`） | `AmdSmuTuningProvider.cs:203`（绝对 Boost 上限 `[1000, 7000] MHz`，仅为工具支持范围而非固件限制）；`FMaxAbstractions.cs:9` | 缺（FMax Override 偏移参数未实现） | 静态源码 | `observed`：原判「文档与现实冲突」属误判——代码写的是绝对 Boost 上限，并非与偏移定义冲突，正确结论为「FMax Override 偏移参数未实现」；另 Desktop UI `BuildPlanFromForm` 无 FMax 输入。补充：产品 commit `2d9351d` 已将绝对频率改为钳制写入，账本记钳制后期望值与回读真实值 |
| B-14 | `amd-smu.md:74`：FMax 写前基线 `0` 禁止写入/回退 | `SmuReader.cs:WriteFMax:457-465`（基线 0 仍写） | `AmdSmuTuningProvider.cs:TryWriteFMax:217-235`（基线 0 拒写） | 部分 | 静态源码 | `observed` |
| B-15 | `amd-smu.md:102-104`：通道多态区分，不可压成 `bool` | `SmuReader.cs:Status:51` | `AmdSmuTuningProvider.cs:Query:60`（`HardwareAvailability`） | 一致 | 静态源码 | `observed` |
| B-16 | `intel-oc-mailbox.md:70-76`：MSR `0x150` + bit63 busy 轮询 | `IntelMsrDevice.cs:24-44`；`IntelTuning.cs:173-186`（无 busy 轮询） | `IntelMsrBits.cs:14-15,46`；`IntelMsrTuningProvider.cs:247-283` | 部分实现（仅模拟验证） | 静态源码 + `FakeIntelTuningProvider` | `observed`：缺 Intel 真机验证，仅 `FakeIntelTuningProvider` 覆盖 |
| B-17 | `intel-oc-mailbox.md:75`：跨进程锁 `Global\Access_Intel_OC_Mailbox` | `IntelTuning.cs:_gate:129`（仅进程内） | `IntelMsrTuningProvider.cs:14,285-316` | 新有旧无（仅模拟验证） | 静态源码 + Fake | `observed` |
| B-18 | `intel-oc-mailbox.md:79-96`：`S11.0.10` (`mV * 1.024`)、容差 2 mV | `IntelTuning.cs:24-61,126` | `IntelMsrBits.cs:22-45,59-60`；`IntelMsrTuningProvider.cs:85,114` | 一致（仅模拟验证） | 静态源码 + Fake | `observed` |
| B-19 | `intel-oc-mailbox.md:102,104`：输入范围与可表示性预检 | 无 | `IntelVoltageOffsetBounds:109-118`（`[-150,+100]`）；`IntelMsrTuningProvider.cs:79-90` | 新有旧无（仅模拟验证） | 静态源码 + Fake | `observed` |
| B-20 | `intel-oc-mailbox.md:31-68,72`：多电压平面（Core/GT/Cache） | `IntelTuning.cs:8-13,34-39,147-195`（三平面均支持） | `IntelMsrBits.cs:17-19`（仅定义常量，未接线）；`IntelMsrTuningProvider.cs:40-127`（仅 Core）；且 Desktop UI 无 Intel 写入入口 | 缺（GT/Cache 未接线） | 静态源码 | `observed`：GT/Cache 属「代码存在但未接线」，Core 平面在 Desktop UI 无创建入口 |
| B-21 | `intel-oc-mailbox.md:108`：MSR `0x1AD` 睿频倍频读取（8 档） | `IntelTuning.cs:DecodeRatios:64-70`（全 8 档） | `IntelMsrBits.cs:49-56`；`IntelMsrTuningProvider.cs:66-67`（仅 1C/8C） | 部分 | 静态源码 | `observed` |
| B-22 | `intel-oc-mailbox.md:108`：MSR `0x1AD` 睿频倍频写入与回读 | `IntelTuning.cs:ApplyRatios:202-241` | 无 | 缺 | 静态源码 | `observed` |
| B-23 | `intel-oc-mailbox.md:116,125`：写后回读区分 UVP 静默忽略 | `IntelTuning.cs:181-194` | `IntelMsrTuningProvider.cs:106-119` | 一致（仅模拟验证） | 静态源码 + Fake | `observed` |
| B-24 | 调校方案与调度、电源解耦 | `PlanService.cs:19-89` | `Domain/Plans/Plan.cs:26-45`；`Domain/Plans.cs:3-88` | 一致 | 静态源码 | `observed` |
| B-25 | 方案 CRUD（含改名、校验、引用保护） | `PlanService.cs:165-565` | `AgentSession.cs:185-202,432-502,773-842`（无独立 rename op，走 `plans.save`） | 部分 | 静态源码 | `observed` |
| B-26 | 方案应用与部分生效语义 | `TuningApplyService.cs:84-111`；`PlanService.cs:574-608` | `PlanCoordinator.cs:ApplyCpu:932-1208`；`AgentSession.cs:203-208` | 一致 | 静态源码 | `observed` |
| B-27 | 方案回退与出厂归零 | `TuningStore.cs:ResetToFactory:85-102`；`IntelTuning.cs:Restore:243-271` | `PlanCoordinator.cs:RevertPlanAsync:133-176`（缺一键归零出厂） | 部分 | 静态源码 | `observed` |
| B-28 | 持久化与 `PersistsAcrossReboot`、旧数据迁移 | `TuningStore.cs:47-75`；`PlanService.cs:208-399` | `PlanApplyRecord.PersistsAcrossReboot:20`；`DataDirectoryMigration.cs:19` | 一致 | 静态源码 | `observed` |
| B-29 | `amd-curve-optimizer.md:36,138`：休眠恢复核对与外部漂移检测 | 无（仅启动重放） | `PlanCoordinator.cs:ReconcileAfterResumeAsync:225-304`、`DetectDriftAsync:369-434` | 新有旧无（仅模拟验证） | 静态源码 + `SceneRecoveryTests` (Fake) | `observed`：无真机休眠实测 |
| B-30 | 混合核 P/E 拆分与默认调校方案生成 | `MixedCorePlan.cs:14-47`；`PlanService.cs:333-343` | 无（仅 `PowerSchemePresets.cs`） | 部分 | 静态源码 | `observed` |
| B-31 | 逻辑核掩码工具与 SMU 掩码分离 | `Helpers/CoreMask.cs:20-157` | `Domain/Plans.cs:CpuSetSpec:312-413` | 部分 | 静态源码 | `observed` |
| B-32 | `amd-curve-optimizer.md:115`：平台身份绑定防跨机误写 | 无 | `CpuTuningPlan.PlatformId:31`；`PlanCoordinator.cs:ApplyCpu:959-977` | 新有旧无 | 静态源码 | `observed` |
