# 分片 F：场景自动化（触发 / 仲裁 / 账本 / 回退 / 休眠唤醒）只读静态审计

- **旧项目基准**：`Motu-Z/Zentwik@main` (`727462601911609e6dacbe3cb8f144246829087a`)
- **新项目对照**：`Motu-Z/Zentwik@feat/gui-protocol` (`100e9c0f4f9e7f9b160f8b0bed9ea99afdf20922`)
- **研究文档基准**：`Motu-Z/Zentwik-Next-Research@main` (`7680e5109ed010e4681180393ff85fb2ce412554`)

## 1. 闭合计数

- **对照项总数**：**23 项**（`F-01` ~ `F-23`）。
- **状态分布**：`一致` 9、`部分` 1、`缺` 1、`新有旧无` 12、`文档与现实冲突` 0。

## 2. 逐项对照表

| 编号 | 文档要求(路径:行) | 旧项与证据(文件:函数/行) | 新项与证据(文件:函数/op) | 状态 | 验证级别 | 备注 |
|---|---|---|---|---|---|---|
| F-01 | `scenes-arbitration.md:32`：`manual` 手动激活 | 无（`ScenarioService.cs:20-41` 无手动触发） | `Scene.cs:5,57-58`；`SceneEngine.cs:239-256,294-311`；`scenes.activate/deactivate`（UI/CLI 已接线） | 新有旧无 | 静态源码 | `observed` |
| F-02 | `scenes-arbitration.md:29`：`process` 持续条件 | `ScenarioService.cs:23`；`ScenarioMonitor.cs:216-239` | `Scene.cs:8`；`SceneEngine.cs:490-500` | 一致 | 静态源码 | `observed` |
| F-03 | `scenes-arbitration.md:25,30,36,67`：`processExited` 边沿触发（首拍不触发） | `ScenarioService.cs:40,235-260`；`ScenarioMonitor.cs:76,253-284` | `Scene.cs:14`；`SceneEngine.cs:41-46,180-233` | 部分实现 | 静态源码 | `observed`：后端与 CLI 已接线，**Desktop UI `MainViewModel.TriggerKinds` 缺失 `ProcessExited`** |
| F-04 | `scenes-arbitration.md:30,34,67`：`processExited` 保持生效直到停用/禁用/高优抢占 | `ScenarioService.cs:34-38`（不保持不回退） | `SceneEngine.cs:47-48,109,122,139,155-158,301`（`_latched`） | 新有旧无 | 静态源码（2026-11-26 实测记录） | `observed` |
| F-05 | `scenes-arbitration.md:31`：`powerSource` `ac`/`dc` 持续条件 | `PowerStatusWatcher.cs:40-106` | `Win32PowerStatus.cs:5-24`；`SceneEngine.cs:514-524` | 一致 | 静态源码 | `observed` |
| F-06 | `ScenarioMonitor.cs:15-19`：总闸开关与空闲零占用 | `ScenarioService.cs:205,300-322`；`ScenarioMonitor.cs:84-164` | `Scene.Enabled` 单场景开关；`SchedulingWatcher.cs:146-156`（空闲 20s 轮询） | 部分 | 静态源码 | `observed`：无全局总闸且空闲期仍 20s 轮询 |
| F-07 | `three-plans.md:12,33-35`：场景引用独立方案最新版 | `ScenarioService.cs:63-75,529-595`（耦合） | `Scene.cs:20-24,48`；`SceneEngine.cs:399-426`；`AgentSession.cs:461` | 一致 | 静态源码 | `observed` |
| F-08 | `scenes-arbitration.md:15-19,61`：优先级高者胜 → 进入时间早者胜 → ID 序数 | `ScenarioService.cs:77,754-771`（升序） | `Scene.cs:42,73-105`（降序 Priority → ActivatedAt → SceneId） | 一致 | 静态源码 | `observed` |
| F-09 | `scenes-arbitration.md:21`：单场景持有写权 | `ScenarioService.cs:727-771` | `SceneEngine.cs:142-168`；`StateLedger.cs:113-119` | 一致 | 静态源码 | `observed` |
| F-10 | `scenes-arbitration.md:42`：方案删除/引用缺失处理 | `ScenarioService.cs:370-419`（删后置空） | `AgentSession.cs:471-490,911-917`（保存/删除前置拦截） | 一致 | 静态源码 | `observed` |
| F-11 | `state-ownership.md:15-28`：账本全套字段 | `ScenarioBindingLedger.cs` / `AppliedTargetLedger.cs`（仅去重索引） | `StateLedger.cs:3-59`（前值/目标/实际/归属/优先级/可恢复/失败原因） | 新有旧无 | 静态源码 | `observed` |
| F-12 | `three-plans.md:22,26`：写前记前值→写入→回读→不符判失败 | 无统一回读校验 | `PlanCoordinator.cs:1502-1550`；`StateLedger.cs:171-189` | 新有旧无 | 静态源码 | `observed` |
| F-13 | `state-ownership.md:34,38`：高优先级接管保留原始前值 | 无 | `StateLedger.cs:113-150`（`DeniedLowerPriority`, `stillHeld`） | 新有旧无 | 静态源码 | `observed` |
| F-14 | `scenes-arbitration.md:40`：用户接管 `Owner=User` 与漂移交还 | `ScenarioService.cs:469` | `StateLedger.cs:236-262`；`PlanCoordinator.cs:369-434` | 新有旧无 | 静态源码 | `observed` |
| F-15 | `state-ownership.md:38`：显式释放恢复自动化接管 | 无 | `StateLedger.cs:264-278`；`PlanCoordinator.cs:913-930`；`ledger.list/release` | 新有旧无 | 静态源码 | `observed` |
| F-16 | `state-ownership.md:29`：调度进程目标追踪与 PID 复用 | `ScenarioBindingLedger.cs:35-74`；`ScenarioService.cs:602-614` | `StateLedger.cs:58`；`PlanCoordinator.cs:436-736,1679-1682` | 一致 | 静态源码 | `observed` |
| F-17 | `scenes-arbitration.md:40,46-47,60`：退出/停用/关闭代理回退 | `ScenarioService.cs:38,312-318`（退出不恢复） | `SceneEngine.cs:370-383,443-478`；`PlanCoordinator.cs:178-219`；`AgentServer.cs:289-309` | 新有旧无 | 静态源码 | `observed` |
| F-18 | `scenes-arbitration.md:48,55`：回退失败保留 `Applied+Restorable=true` | 无 | `StateLedger.cs:295-320`；`PlanCoordinator.cs:1557-1581` | 新有旧无 | 静态源码 + Fake 单测 | `observed` |
| F-19 | `scenes-arbitration.md:46-47,65`：同周期重启补回退、跨整机重启跳过 | 无 | `SceneEngine.cs:24,317-368`；`Program.cs:111` | 新有旧无（仅模拟验证） | 静态源码 + `SceneRecoveryTests` (Fake) | `observed`：无真机重启实测 |
| F-20 | `scenes-arbitration.md:49,65`：休眠唤醒监听 | `SleepResumeWatcher.cs:36-85`（`SystemEvents`） | `Win32SuspendResumeNotifications.cs:10-130`（`PowerRegisterSuspendResumeNotification` + 2s 防抖） | 一致 | 静态源码 | `observed` |
| F-21 | `scenes-arbitration.md:49,65`：唤醒核对实际值并决定确认/重写/用户接管/`Unknown` | `SleepResumeWatcher.cs:73-75`（无条件重放） | `AgentServer.cs:213-220`；`PlanCoordinator.cs:225-309`；`StateLedger.cs:221-234` | 新有旧无（仅模拟验证） | 静态源码 + `SceneRecoveryTests` (Fake) | `observed`：无真机休眠实测 |
| F-22 | 启动连续崩溃防砖护栏（无新文档要求） | `BootGuard.cs:10-181`（连续 3 次未完成阻断） | 无 | 缺 | 静态源码 | `observed` |
| F-23 | 场景与账本控制面及事件推送 | 无 `scenes.*`/`ledger.*` | `AgentSession.cs:226-273,504-550,1034-1037`（`Events.SceneChanged`） | 新有旧无 | 静态源码 | `observed` |
