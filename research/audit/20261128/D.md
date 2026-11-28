# 分片 D：进程与调度（枚举 / 亲和性 / 优先级 / CPU Sets / 规则 / 重试与漂移）只读静态审计

- **旧项目基准**：`Motu-Z/Zentwik@main` (`727462601911609e6dacbe3cb8f144246829087a`)
- **新项目对照**：`Motu-Z/Zentwik@feat/gui-protocol` (`100e9c0f4f9e7f9b160f8b0bed9ea99afdf20922`)
- **研究文档基准**：`Motu-Z/Zentwik-Next-Research@main` (`7680e5109ed010e4681180393ff85fb2ce412554`)

## 1. 闭合计数

- **旧项目进程/调度方法数**（10 个目标文件）：**170**（`NativeProcessList` 7、`ProcessWatcher` 41、`ProcessInstanceLedger` 5、`CpuSetScheduler` 45、`ProcessPriority` 17、`ProcessRefresher` 25、`ScheduleDriftTracker` 7、`ScheduleRetryLedger` 8、`MaskResolver` 3、`CoreMask` 12）。
- **新项目调度方法数**（6 个目标模块）：**54**（`NativeProcessList` 7、`AffinityScheduler` 11、`CpuSetsNative` 10、`SchedulingWatcher` 7、`Domain/Plans.cs` 调度模型 4、`PlanCoordinator` 调度分支 15）。
- **新项目 `process.*` op 数**：**0**（`Protocol.cs` 全量 38 个 `Ops` 中无 `process.*`）。
- **对照项总数**：**24 项**（`D-01` ~ `D-24`）。

## 2. 逐项对照表

| 编号 | 文档要求(路径:行) | 旧项与证据(文件:函数/行) | 新项与证据(文件:函数/op) | 状态 | 验证级别 | 备注 |
|---|---|---|---|---|---|---|
| D-01 | `process-scheduling.md:40`：Toolhelp32 1 秒差集轮询，不用 WMI | `NativeProcessList.cs:64-100` + `ProcessWatcher.cs:298-388`（1s） | `NativeProcessList.cs:46-125` + `AffinityScheduler.cs:18-33`（1s TTL）+ `SchedulingWatcher.cs:37,136`（2s/20s） | 部分 | 静态源码 | `observed`：新版轮询 2s/空闲 20s，每轮全量匹配活跃规则 |
| D-02 | `process-scheduling.md:13-22,36`：CPU Sets 软约束 vs 亲和性硬约束互斥，切换前先清另一侧 | `ProcessWatcher.cs:760-799`；`CpuSetScheduler.cs:743-749` | `AffinityScheduler.cs:297-313`；`CpuSetsNative.cs:46-56`；`PlanCoordinator.cs:1385-1488` | 部分 | 静态源码 | `observed`：新版写 CPU Sets 前检外部窄亲和性，但切换/共存时不自动清除另一侧旧限制 |
| D-03 | `process-scheduling.md:25`：CPU Sets 用 `PROCESS_SET_LIMITED_INFORMATION`，Affinity 用 `PROCESS_SET_INFORMATION` | `CpuSetScheduler.cs:753-758,615,664` | `CpuSetsNative.cs:7-9` + `AffinityScheduler.cs:283-287,125,155` | 一致 | 静态源码 | `observed` |
| D-04 | `process-scheduling.md:30-33`：保护进程/退出错误处理不可省 | `CpuSetScheduler.cs:375-380`；`ProcessPriority.cs:197-266` | `AffinityScheduler.cs:114-118,172-176,206-210,289-293` | 部分 | 静态源码 | `observed`：新版不再细分错误 5 与 87/1168 |
| D-05 | `process-scheduling.md:37`：`GetSystemCpuSetInformation` 按 `Size` 步进与 `LastLevelCacheIndex` 分组 | `CpuSetScheduler.cs:1093-1313` | `CpuSetsNative.cs:86-154`（仅提取 `Id/Group/LogicalProcessorIndex`） | 部分 | 静态源码 | `observed`：丢弃 L3 分组与 P/E `EfficiencyClass` 拆分 |
| D-06 | `process-scheduling.md:38`：`Type` 字段 4 字节 `DWORD` vs 2 字节 `WORD` | `CpuSetScheduler.cs:126,1289-1290`（`ushort` 2 字节） | `CpuSetsNative.cs:122,130`（`ReadInt32` 4 字节） | 文档与现实冲突 | 静态源码 | `observed`：旧 2 字节、新 4 字节，需真机核验偏移 |
| D-07 | `process-scheduling.md:23-24,39,53`：`>64` 逻辑核与多处理器组拒绝 Affinity Mask | `CpuSetScheduler.cs:577,594-606` | `AffinityScheduler.cs:141-153` | 一致 | 静态源码 | `observed` |
| D-08 | `process-scheduling.md:40`：已退出句柄拦截（`STILL_ACTIVE=259`） | `CpuSetScheduler.cs:215-216` | `CpuSetsNative.cs:27-43`；`AffinityScheduler.cs:102,126,194,261` | 一致 | 静态源码 | `observed` |
| D-09 | `process-scheduling.md:41`：解绑仅清本工具绑定 PID | `ProcessWatcher.cs:734-754` | `PlanCoordinator.cs:543-565,1676-1714`（`StateLedger`） | 一致 | 静态源码 | `observed` |
| D-10 | `process-scheduling.md:42`：PID 复用检测（进程创建时间） | `ProcessInstanceLedger.cs:63-99` | `AffinityScheduler.cs:60-79`；`PlanCoordinator.cs:1915-1943` | 一致 | 静态源码 | `observed` |
| D-11 | `process-scheduling.md:42`：漂移每 ~5 秒只记录不重写 | `ScheduleDriftTracker.cs:13-74`；`ProcessWatcher.cs:445-470`（只记录不重写） | `PlanCoordinator.cs:567-723`（`rule.Enforce==true` 时直接重写） | 文档与现实冲突 | 静态源码 | `observed`：新版已演进为 Enforce 自动重施加 |
| D-12 | `process-scheduling.md:46-47`：优先级 6 档含 `EcoQoS`，不含 `RealTime` | `ProcessPriority.cs:69,106-115,228-239` | `AffinityScheduler.cs:158-167`（直接 `Enum.TryParse<ProcessPriorityClass>`） | 部分 | 静态源码 | `observed`：新版缺 `EcoQoS` 且未排除 `RealTime` |
| D-13 | 写入后回读校验 | `CpuSetScheduler.cs:827-934` | `PlanCoordinator.cs:1389-1481,1538-1549` | 一致 | 静态源码 | `observed` |
| D-14 | 核心停泊告警（`CPMINCORES < 100%`） | `ProcessWatcher.cs:931-968` | `PlanCoordinator.cs:786-830`（按 AC/DC） | 一致 | 静态源码 | `observed` |
| D-15 | 失败重试与指数退避 | `ScheduleRetryLedger.cs:39-105`（5 次，2s~60s） | 无退避上限，失败后下轮 2s 直接重试 | 缺 | 静态源码 | `observed` |
| D-16 | 规则模型与掩码库解析 | `CoreMask.cs:20-191`；`MaskResolver.cs:19-71` | `SchedulingRule` + `CpuSetSpec`（无 128 位掩码与独立掩码库） | 部分 | 静态源码 | `observed` |
| D-17 | 进程名归一化匹配 | `ProcessWatcher.cs:999-1063` | `AffinityScheduler.cs:35-58` | 一致 | 静态源码 | `observed` |
| D-18 | 多实例并发保护 | `ProcessWatcher.cs:176-208`（`Global\Zentwik.Singleton`） | `PlanCoordinator._gate`（进程内锁） | 部分 | 静态源码 | `observed` |
| D-19 | 后台进程全量采样（30s CPU 滑窗、映像路径、基础优先级） | `ProcessRefresher.cs:423-659` | 无 | 缺 | 静态源码 | `observed` |
| D-20 | 协议层 `process.*` op（实时进程枚举与单进程调控） | 旧 UI 直调 `ProcessRefresher`/`ProcessWatcher` | `Protocol.cs` 与 `AgentSession.cs` 共 0 个 `process.*` op | 缺 | 静态源码 | `observed`：只能通过 `SchedulingPlan` 间接调控 |
| D-21 | Desktop 进程管理页对接 | 旧 `ProcessView.xaml` 完整实现 | `MainForm.cs:168-172,664-672`（禁用按钮+空骨架） | 缺 | 静态源码 | `observed`：代码仅骨架 |
| D-22 | 调度强制耗时预算告警（>200ms） | 无 | `PlanCoordinator.cs:742-768` | 新有旧无 | 静态源码 | `observed` |
| D-23 | 场景/方案优先级仲裁与前值回退 | `ProcessWatcher.cs:426-438`（放回默认） | `PlanCoordinator.cs:467-565`（`StateLedger` 精确回退） | 新有旧无 | 静态源码 | `observed` |
| D-24 | 空闲自适应降频轮询（2s/20s） | 固定 1s | `SchedulingWatcher.cs:146-156` | 新有旧无 | 静态源码 | `observed` |
