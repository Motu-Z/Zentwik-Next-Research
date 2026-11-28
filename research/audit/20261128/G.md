# 分片 G：服务 / IPC / 协议 / 权限 / 安装升级 / 数据迁移 / 日志诊断 / 模块能力只读静态审计

- **旧项目基准**：`Motu-Z/Zentwik@main` (`727462601911609e6dacbe3cb8f144246829087a`)
- **新项目对照**：`Motu-Z/Zentwik@feat/gui-protocol` (`100e9c0f4f9e7f9b160f8b0bed9ea99afdf20922`)
- **研究文档基准**：`Motu-Z/Zentwik-Next-Research@main` (`7680e5109ed010e4681180393ff85fb2ce412554`)

## 1. 闭合计数汇总

| 统计项 | 计数 | 源码依据 |
|---|---:|---|
| 旧 IPC 命令数 | **46** | `Zentwik.Core/Ipc/WorkerPipeContract.cs:77-680` |
| 新 `Ops` 数量 | **38** (+2 `Events`, +28 `ErrorCodes`) | `src/Zentwik.Contracts/Protocol.cs:20-110`（全量在 `AgentSession.DispatchAsync` 分派） |
| 旧 `Capability` 数量 | **10** | `Zentwik.Core/Capabilities/CapabilityIds.cs:9-40` |
| 新 `Capability` 数量 | **9** | `src/Zentwik.Domain/Capabilities/Capability.cs:28-43` |
| 旧模块数 | **8** | `CoreRuntimeModule` / `CpuVendorModules` / `GenericModules` |
| 新模块数 | **7** (6 必选 + 1 可选 `power-limits`) | `src/Zentwik.Core/Modules/ModuleCatalog.cs:26-37` |
| 对照项总数 | **15** | `G-01` ~ `G-15` |

## 2. 逐项对照表

| 编号 | 文档要求(路径:行) | 旧项与证据(文件:函数/行) | 新项与证据(文件:函数/op) | 状态 | 验证级别 | 备注 |
|---|---|---|---|---|---|---|
| G-01 | `ADR-003:13-16`；`security-service.md:13-15`：LocalSystem 服务独占特权 | `Zentwik.Worker/Program.cs:26-37`（每用户提权 Worker） | `Agent/Program.cs:37-146`、`WindowsServiceHost.cs:56-168`、`ServiceInstaller.cs:182-195` | 一致 | 静态源码 | `observed` |
| G-02 | `lifecycle.md:40-41`；`ADR-003:36-40`：服务 CLI 与 SCM 生命周期 | 无 SCM 宿主 | `AgentOptions.cs:54-68`、`ServiceInstaller.cs:134-441`、`WindowsServiceHost.cs:56-61` | 一致 | 静态源码 | `observed` |
| G-03 | `security-service.md:20-21,57`：SCM 恢复（5s/15s/60s）、退出码 1066、`fFailureActionsOnNonCrashFailures = TRUE` | 无 SCM 恢复 | `ServiceInstaller.cs:522-561`（5s/15s/60s）、`WindowsServiceHost.cs:121-126`（1066）；**未调用 `ChangeServiceConfig2W` 设 `SERVICE_CONFIG_FAILURE_ACTIONS_FLAG(4)`** | 文档与现实冲突 / 部分实现 | 静态源码 | `observed`：非崩溃异常退出恢复前提缺失 |
| G-04 | `lifecycle.md:42`：关闭回退、启动补回退、唤醒核对 | `BackendHostService.cs:178-193`（无退出回退） | `AgentServer.cs:152,213-228,289-309`、`Program.cs:111-115` | 一致 | 静态源码 + Fake 单测 | `observed`：缺真机休眠/重启验证 |
| G-05 | `ADR-004:13`；`ipc.md:13-14,21`：4B 长度前缀 + UTF-8 JSON、`ReadAtLeastAsync`、`Id` 配对、`Id=0` 事件 | `WorkerPipeContract.cs:10-12,52`（`\n` 分隔 512KB） | `Protocol.cs:11-18,112-206`（4MB、`ReadAtLeastAsync`）；`AgentConnection.cs:83,370-379` | 一致 | 静态源码 | `observed` |
| G-06 | `ipc.md:15`；`ADR-004:15,22`：`域名.动作` 38 个 `Ops` 与旧 46 命令对照 | 46 个短横线命令 | 38 `Ops` + 2 `Events`（`AgentSession.DispatchAsync` 全接线）；缺旧版 `query-processes`/`clear-all-schedules`/`*-mask`/`query-cpu-topology`/`clear-logs`/`factory-reset` | 部分 | 静态源码 | `observed` |
| G-07 | `ipc.md:20`；`ADR-004:17`：遥测背压丢帧计数、响应不丢、日志拉模式 | 全请求-响应；`snapshot-logs` | `AgentSession.cs:27,33-34,620-631,1040-1047`（`BacklogThreshold=256` 丢新帧并记 `_droppedSamples`）；`Ops.DiagnosticsLog` | 一致 | 静态源码 | `observed`：队列阈值为 256 非文档 2 帧 |
| G-08 | `ipc.md:16,26-27`；`ADR-004:16,22`：域前缀稳定错误码 | 中文 `Message` 字符串 | `Protocol.cs:75-110`（28 个稳定错误码） | 一致 | 静态源码 | `observed` |
| G-09 | `ipc.md:32-33`：版本协商与未知方法错误码 | `ProtocolVersion = 23` 严格相等 | `Protocol.cs:11,79`（`Version = 2` 严格相等）；`AgentSession.cs:424-428`（`request.unknown_operation`） | 部分 | 静态源码 | `observed`：未落地主版本+能力位协商 |
| G-10 | `security-service.md:26-27`：显式 DACL、`PIPE_REJECT_REMOTE_CLIENTS`、`FILE_FLAG_FIRST_PIPE_INSTANCE` | 仅当前用户 SID DACL | `PipeSecurityFactory.cs:13-40`（显式 DACL）；`AgentServer.cs:96-104`（未设 `FirstPipeInstance` 与远程拒绝标志） | 部分 | 静态源码 | `observed` |
| G-11 | `security-service.md:28-31`：调用方 PID/Session/TokenUser/IL≥Medium/OwnerSid 校验 + 拒连回帧 + 服务端 SID 校验 | 无双方校验 | `PipeCaller.cs:31-104`、`AgentServer.cs:123-136,249-284`、`PipeIdentity.cs:73-92`、`AgentConnection.cs:65-71` | 一致 | 静态源码 + `AccessControlTests` | `observed` |
| G-12 | `modules-capabilities.md:13-40`：七态能力、`AnyOf`、可选模块开关、第三方插件隔离 | 8 模块 10 能力，无开关 | `Capability.cs:4-13`、`ModuleCatalog.cs:26-37`（7 模块 9 能力，`power-limits` 可选）、`ModuleService.cs`；无插件进程加载器 | 部分 | 静态源码 | `observed`：插件隔离长期项未做 |
| G-13 | `ADR-006:12-16`；`installer.md:33-35`：分文件持久化、原子替换、`SchemaVersion` 拒载、目录搬迁 | 分文件（`tuning/masks/schedules/ui.json`） | `StateStore.cs:21,32-144`（合并存单文件 `state.json`，原子替换与 `SchemaVersion` 守卫完备）；`DataDirectoryMigration.cs` | 文档与现实冲突 | 静态源码 | `observed`：合并单文件 `state.json` 违背 ADR-006 分文件约定 |
| G-14 | `ipc.md:20`；`installer.md:29`：按天日志轮转与 `DoctorService` | `LogService.cs`（200 缓冲 + 保留天数清理 + `clear-logs`） | `DiagnosticLog.cs:7-143`（8MB 轮转、200 缓冲）；`DoctorService.cs:25-49`（`diagnostics.doctor`） | 一致 | 静态源码 | `observed`：新缺按天自动清理与 `clear-logs` |
| G-15 | `installer.md:1-43`：Inno 6、停服务、安装服务、`--enable-module power-limits`、PawnIO 检测、卸载删服务 | `installer/zentwik.iss:1-172` | `scripts/build-installer.ps1` + `scripts/installer/zentwik.iss:1-138`（文档写 `zentwik-next.iss` 与实际文件名冲突；无数字签名与自动更新） | 文档与现实冲突 | 静态源码 | `observed`：`zentwik.iss:58` 打包 `pkg\*` |
