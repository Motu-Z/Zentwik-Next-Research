# ADR-004：UI 与服务的通信协议

- 状态：Accepted（M1 实现为第一阶段形态）
- 日期：2026-11-25
- 相关：`research/03-ui-ipc-deployment.md`

## 背景

旧协议：单连接、一请求一响应、单行 JSON、单一 `ProtocolVersion`（已到 23），任何命令增删都要求两端同版本安装；无推送，遥测靠轮询；错误以中文字符串返回，客户端无法稳定判定。

## 决策

命名管道，帧格式为 4 字节小端长度 + UTF-8 JSON 正文；请求带自增 `Id`，响应按 `Id` 配对；事件帧 `Id=0`。

- 命令字采用 `域名.动作`（`telemetry.subscribe`）。
- 错误返回稳定错误码（`protocol.version_mismatch`、`request.payload_invalid`）+ 可选 detail，客户端不解析文案。
- 背压：遥测在有积压时丢弃最新帧并计数，响应永不丢。
- M1 用 `PipeOptions.CurrentUserOnly`；服务化后改为显式 DACL + 拒绝远程 + 校验调用方 SID/完整性级别/会话。

## 协议 v2（M2）

命令面从"状态/能力/设置/遥测"扩到完整产品面：`modules.list`、`plans.{get,save,delete,apply,revert}`、`scenes.{get,save,delete,activate,deactivate}`、`ledger.{list,release}`、`hardware.tuning.read`、`diagnostics.{doctor,log}`；事件新增 `scene.changed`。错误码按域前缀扩展（`plan.*`、`scene.*`、`ledger.*`、`hardware.*`）。

已从 `CurrentUserOnly` 切换为显式 DACL + 调用方校验（见 ADR-003 实施记录）。测试仍直接跑真实管道，未降级为内存传输。

**切换决策的补充证据（2026-11-26，一手核验）**：`dotnet/runtime#123903`（issue open、修复 PR #136870 截至核验日未合并）确认 `PipeOptions.CurrentUserOnly` 自 .NET Core 2018 起误用 `WindowsIdentity.GetCurrent().Owner` 而非 `.User` 构建 DACL，在 LocalSystem 服务与管理员创建进程 owner SID 不一致时会误拒绝；且 MIC `NO_WRITE_UP` 只挡低→高，不能替代服务端显式 `TokenIntegrityLevel` 校验。二者共同支持本 ADR 从 `CurrentUserOnly` 迁移到显式 `PipeSecurity` + `NamedPipeServerStreamAcl.Create` 的决定。来源与准确措辞见 [ENGINEERING_NOTES.md §1](../knowledge/ENGINEERING_NOTES.md#1-windows-命名管道安全currentuseronly-的边界与已知-bug)。

## 候选

- **gRPC over named pipe**：成熟的多路复用与流，但引入 HTTP/2 依赖与代码生成，收益在当前数据量下不明显。
- **共享内存 + 事件**：适合高频大帧，复杂度高，当前 4–10 Hz、单帧 < 1 KB 不需要。
- **二进制列式遥测**：等实测证明 JSON 有成本再做。

## 后果

正面：可多路复用、可推送、错误可判定、协议演进不必强制同版本。
负面：JSON 有解析与分配开销；长度前缀需要自己处理半包（已用 `ReadAtLeastAsync` 处理）。

## 验证

- 已有测试：帧往返、超大帧拒绝、截断头拒绝、真实管道上的状态/设置/遥测/并发连接、版本不匹配返回稳定错误码。
- 待验证：服务化后的 ACL 与调用方校验；10 个客户端并发订阅时的积压与丢弃计数。
