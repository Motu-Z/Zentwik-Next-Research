# 分片 E：监控遥测（温度 / 功耗 / 逐核频率 / 利用率 / 历史 / 轮询）只读静态审计

- **旧项目基准**：`Motu-Z/Zentwik@main` (`727462601911609e6dacbe3cb8f144246829087a`)
- **新项目对照**：`Motu-Z/Zentwik@feat/gui-protocol` (`100e9c0f4f9e7f9b160f8b0bed9ea99afdf20922`)
- **研究文档基准**：`Motu-Z/Zentwik-Next-Research@main` (`7680e5109ed010e4681180393ff85fb2ce412554`)

## 1. 闭合计数

- **对照项总数**：**20 项**（`E-01` ~ `E-20`）。
- **状态分布**：`一致` 2、`部分` 2、`缺` 1、`新有旧无` 14、`文档与现实冲突` 1。

## 2. 逐项对照表

| 编号 | 文档要求(路径:行) | 旧项与证据(文件:函数/行) | 新项与证据(文件:函数/op) | 状态 | 验证级别 | 备注 |
|---|---|---|---|---|---|---|
| E-01 | `polling-budget.md:22,25`：仅订阅期间采样 | 无系统遥测（仅 `ProcessRefresher`） | `TelemetryHub.RunSubscriptionAsync:64-106`；`AgentSession.cs:569-661`（`telemetry.subscribe` / `unsubscribe`） | 新有旧无 | 静态源码 | `observed` |
| E-02 | `polling-budget.md:37,102,107`：`MonitorIntervalMs` 500–5000 ms（默认 1000） | `ProcessView.xaml.cs:32`（3s 定时器） | `UserSettingsRules:33-37`；`AgentSession.cs:573-593,644`（返回 `SubscriptionAck`） | 新有旧无 | 静态源码 | `observed` |
| E-03 | `polling-budget.md:103,129`：`SensorIntervalMs` 500–30000 ms，`0` 关闭传感器 | 无 | `UserSettingsRules:40-44`；`AgentSession.ClampSensorInterval:648-649` 夹取到 `[500,30000]` | 部分 | 静态源码 | `observed`：协议端无法传 `0` 关闭传感器 |
| E-04 | `polling-budget.md:57,108-109`：双时钟独立与缓存沿用 | 无 | `TelemetryHub.RunSubscriptionAsync:75-85,117-137` | 新有旧无 | 静态源码 | `observed` |
| E-05 | `sensors/README.md:38-40`：数据面 `≥4Hz` + WPF `CompositionTarget.Rendering` 30/60Hz | WPF 3s `DispatcherTimer`，无图表 | `PeriodicTimer` 下限 `500 ms`（`≤2Hz`）；WinForms 无 `CompositionTarget.Rendering` | 文档与现实冲突 | 静态源码 | `observed`：`sensors/README.md` 过时 |
| E-06 | `polling-budget.md:83`：禁调 `timeBeginPeriod` | 未调用 | 未调用 | 一致 | 静态源码 | `observed` |
| E-07 | `polling-budget.md:22`：CPU 总利用率与物理内存 | 无整机遥测 | `Win32TelemetrySource.ReadCpu:43-76`（`GetSystemTimes`）、`Sample:27-39`（`GlobalMemoryStatusEx`） | 新有旧无 | 静态源码 | `observed`：UI/CLI 已接线 |
| E-08 | `polling-budget.md:22,35,61`：进程计数 2s TTL + Toolhelp 过滤 `ThreadCount==0` | 无系统遥测计数 | `Win32TelemetrySource.CountProcesses:79-90`；`NativeProcessList.Count:15-45` | 新有旧无 | 静态源码 | `observed` |
| E-09 | `polling-budget.md:22,123`：逐逻辑核占用 `NtQuerySystemInformation` | 无 | `ProcessorLoadReader.Read:37-93`；`ProcessorUsage.Percent:10-20`（首帧 `null`） | 新有旧无 | 静态源码 | `observed`：Desktop UI 未消费展示 |
| E-10 | `amd-power-limit.md:174`：AMD 传感器走 `AMDFamily17.bin`，10s 失败重试 | 无 | `AmdSensorSource.cs:25,31,284-336`（仅 Family `17h–1Ah`） | 新有旧无 | 静态源码 | `observed` |
| E-11 | `amd-power-limit.md:176-181`：Tctl SMN `0x00059800` + `Global\Access_PCI` | 无 | `AmdSensorSource.TryReadTemperature:368-399`；`AmdSensorDecoder:16-21` | 新有旧无 | 静态源码 + 9955HX 实测（2026-11-27） | `observed`：Desktop UI 实时卡未展示 |
| E-12 | `amd-power-limit.md:185-192`：RAPL `0xC0010299` / `0xC001029B` 封装功耗 | 无 | `AmdSensorDecoder:24-39`；`AmdSensorSource.UpdatePackagePower:128-144` | 新有旧无 | 静态源码 + 9955HX 实测（2026-11-27） | `observed`：Desktop UI 实时卡未展示 |
| E-13 | `amd-power-limit.md:226-245`：APERF/MPERF 逐核均频 | 无 | `AmdSensorSource.SampleClocks:180-275`；`AmdSensorDecoder:49-58` | 新有旧无 | 静态源码 + 9955HX 实测（2026-11-27） | `observed`：Desktop UI 未展示 |
| E-14 | `amd-power-limit.md:226`：睡眠核省略，不填 0 MHz | 无 | `AmdSensorDecoder.AverageMegahertz:51-54` 返回 `null`；`SampleClocks:260-263` 跳过 | 新有旧无 | 静态源码 | `observed` |
| E-15 | `amd-power-limit.md:143-168`：不用 PM 表做遥测 | 未使用 | `AmdSensorSource.cs:12` 仅用 SMN+MSR | 一致 | 静态源码 | `observed` |
| E-16 | `sensors/README.md:17-30`：Intel 传感器（`0x19C/0x1B1/0x611/0x198`）及 AMD 每核功耗 `0xC001029A` | 无 | 无 | 缺 | 静态源码 | `observed` |
| E-17 | `polling-budget.md:125`：3600 帧内存环形缓冲 | 无 | `TelemetryRingBuffer.DefaultCapacity = 3600`（`TelemetryRingBuffer.cs:12`） | 新有旧无 | 静态源码 | `observed` |
| E-18 | `polling-budget.md:107,125`：`telemetry.subscribe` / `history` 协议与背压 | 无 | `Protocol.cs:28-30,71`；`AgentSession.cs:413-421,569-688`（`BacklogThreshold=256`） | 新有旧无 | 静态源码 | `observed` |
| E-19 | `polling-budget.md:102-107`：CLI `monitor` 与 `history` | 无 | `Commands.cs:MonitorAsync:136-219`、`HistoryAsync:1285-1302` | 新有旧无 | 静态源码 | `observed` |
| E-20 | `polling-budget.md:125`：桌面端订阅、实时指标、历史列表与折线图 | 无监控页 | `MainViewModel.cs:534,1467-1488`（订阅 500ms、触发 `HistoryChanged`）+ `MainForm.cs:308-354`（`_historyGrid`） | 部分实现 | 静态源码 | `observed`：`MainForm.cs` 未订阅 `HistoryChanged` 画折线图；实时区域未展示温度/功耗/逐核频率/逐核占用 |
