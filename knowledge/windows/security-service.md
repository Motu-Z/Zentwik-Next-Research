# Windows 服务、命名管道与调用方校验

- 规范名称：进程权限边界与 IPC 安全
- 英文名称：Windows Service, Named Pipe ACL, Caller Validation
- 缩写：SCM、DACL、SID、UAC、VBS、HVCI
- 领域：`windows`
- 核验状态：Verified（Microsoft API 语义 + 本仓库实现记录）
- 权威来源：MS Learn（CreateNamedPipe、PIPE_REJECT_REMOTE_CLIENTS、GetNamedPipeClientProcessId、SECURITY_SQOS_PRESENT）；本仓库 `research/03-ui-ipc-deployment.md` §3/§4、ADR-003
- 相关文档：[power-management.md](power-management.md)、[../architecture/ipc.md](../architecture/ipc.md)

## 1. 进程模型（项目决策，Accepted）

- LocalSystem 服务（如 `Zentwik.Agent`）为唯一特权方：独占设备句柄、执行硬件写入、采样、场景引擎。
- UI 中完整性运行，只发命令/收事件，不碰设备句柄。
- 第三方插件将来独立进程，经受控 IPC 申请特权操作。

## 2. 服务生命周期（Verified 实现）

- `--service`：`StartServiceCtrlDispatcherW` + `RegisterServiceCtrlHandlerExW` + `SetServiceStatus`；交互环境下退回控制台便于调试。
- 安装/卸载走 SCM；安装写恢复动作（5s/15s/60s 重启，计数 1 天重置）。
- 服务异常终止须把失败写进服务状态（如 1066），否则 SCM 不执行恢复动作。
- 会话 0 与用户会话通信须显式处理；调试服务比控制台复杂。

## 3. 命名管道安全（Verified API 语义）

- 显式 DACL（SYSTEM/Administrators 全控、Authenticated Users 读写、数据目录所有者全控）。
- `PIPE_REJECT_REMOTE_CLIENTS` 拒绝远程；`FILE_FLAG_FIRST_PIPE_INSTANCE` 防管道抢占。
- 调用方校验链：`GetNamedPipeClientProcessId` → `OpenProcess` → `OpenProcessToken` → TokenUser / TokenElevation / TokenIntegrityLevel。
- 判定：会话匹配、完整性 ≥ Medium、SID 为 owner.sid；拒绝 Session 0 与低完整性调用方。
- 客户端可用 `SECURITY_SQOS_PRESENT | SECURITY_IDENTIFICATION` 防服务端冒充。
- 拒绝连接时先回一帧 `request.unauthorized` 再断开——直接关管道会让客户端把权限拒绝误判成代理崩溃。

## 4. 完整性级别与令牌（外部事实 / Verified）

- 完整性级别（IL）：Untrusted/Low/Medium/High/System；进程不能向更低 IL 的对象写入（强制完整性控制）。
- UI 的 IL 取决于启动路径：用户手动启动为 medium，被提权 Worker 拉起为 high。
- 旧版安全缺口：管道 DACL 只按当前用户 SID，同用户任意中完整性进程可下发写 SMU 请求——事实陈述。

## 5. 数值与枚举语义

- SessionId、TokenIntegrityLevel、ElevationType 都是枚举/标识，无单位；比较用等值判断。
- 服务状态码（如 1066）、Win32 错误码（如 31、50、87）与 NTSTATUS 是三套不同编码，不得混用；旧版把 NTSTATUS 误当 Win32 码，削弱了错误区分能力。

## 6. 常见误解

- "DACL=当前用户 SID 就安全" → 同用户任意进程可提权写入。
- "服务化后 UI 自动能连" → DACL 取当前用户会变成 SYSTEM，交互用户被拒。
- "关管道就是拒绝" → 应显式回错误帧，否则误判为崩溃。

## 7. 核验状态

API 语义 = Verified；真实 `sc create` 安装、Session 0→用户会话通信、受限令牌负向测试 = Unverified（当前会话非管理员，仅验证权限守卫分支）。
