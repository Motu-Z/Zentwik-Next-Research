# ADR-003：进程与权限模型

- 状态：Accepted（服务宿主与调用方校验已实现，安装实测待补）
- 日期：2026-11-25
- 相关：`research/04-tech-stack-evaluation.md`、`research/02-hardware-access.md`

## 背景

旧版是"中完整性 UI + 每用户提权 Worker + 计划任务自启"。它带来每次登录的提权依赖、跨会话不可见、以及一个真实缺口：管道只按当前用户 SID 放行，同用户任意中完整性进程都能下发写 SMU 的请求。

## 决策

- **LocalSystem 服务**（`Zentwik.Agent`）为唯一特权方：独占 PawnIO 句柄、执行 SMU/MSR/电源/调度写入、运行采样与场景引擎。
- **UI** 以中完整性运行，只发命令、收事件，不碰设备句柄。
- 第三方插件将来以独立进程接入，通过受控 IPC 申请特权操作。
- 设备句柄不通过 IPC 传递。

## 证据

- 旧版 SMU 串行化依赖进程内 `static lock`；两个进程同时持句柄时无跨进程仲裁（`research/02` §2.3）。
- 旧版管道鉴权只校验用户 SID（`research/01` §4），等价于同用户任意进程可提权写入。
- 服务形态可在安装时一次性授权，避免每次登录的 UAC 依赖。

## 后果

正面：权限边界收敛到单一进程；崩溃隔离；升级由 SCM 控制；跨会话（锁屏、远程）一致。
负面：Session 0 与用户会话通信需要显式处理（管道 ACL、调用方校验、会话枚举）；调试服务比调试控制台程序麻烦；卸载必须确保硬件状态已恢复。

## 验证与回滚

- 必须先验证（1 天）：服务安装/卸载、UI 连接、受限令牌下的授权测试、服务崩溃后 UI 行为。
- 回滚条件：服务化导致安装失败率或调试成本不可接受。回滚目标为"每用户提权代理 + 计划任务自启"，但必须保留调用方校验。

## 实施记录（M2）

- `Zentwik.Agent --service`：`StartServiceCtrlDispatcherW` + `RegisterServiceCtrlHandlerExW` + `SetServiceStatus`；`Environment.UserInteractive` 为真时退回控制台，便于开发调试。
- `--install-service` / `--uninstall-service`：`CreateServiceW` 自动启动、`LocalSystem`；安装时把当前用户 SID 写入 `owner.sid`，供管道放行判定。
- 管道用 `NamedPipeServerStreamAcl.Create` + 显式 `PipeSecurity`：SYSTEM/Administrators 完全控制、AuthenticatedUser 读写、数据目录所有者完全控制。
- 每个连接建立前校验调用方：`GetNamedPipeClientProcessId` → `OpenProcess` → `OpenProcessToken` → TokenUser / TokenElevation / TokenIntegrityLevel；拒绝 Session 0、低于 Medium 完整性、非 `owner.sid` 的调用方。
- M3 补充：`--start-service` / `--stop-service` 走 SCM，状态等待带 20 秒超时；安装时写入恢复动作（5s/15s/60s 重启，1 天重置重启计数）；服务异常终止把失败写进服务状态（1066）而不是静默退出，否则 SCM 不会执行恢复动作。
- 拒绝连接时先回一帧 `request.unauthorized` 再断开：直接关管道会让客户端把权限拒绝误判成代理崩溃。用例覆盖 Session 0、低完整性、非所有者 SID，以及"伪造 owner.sid 让真实管道拒绝"的端到端路径。
- 仍未验证：真实 `sc create` 安装/卸载、Session 0 服务与用户会话的实际通信、受限令牌进程的越权拒绝（当前用伪造所有者 SID 等价模拟，未另造降权进程）。
