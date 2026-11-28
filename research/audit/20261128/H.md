# 分片 H：CLI / 设置 / i18n / 托盘 / 自启 / 单实例只读静态审计

- **旧项目基准**：`Motu-Z/Zentwik@main` (`727462601911609e6dacbe3cb8f144246829087a`)
- **新项目对照**：`Motu-Z/Zentwik@feat/gui-protocol` (`100e9c0f4f9e7f9b160f8b0bed9ea99afdf20922`)
- **研究文档基准**：`Motu-Z/Zentwik-Next-Research@main` (`7680e5109ed010e4681180393ff85fb2ce412554`)

## 1. 闭合计数汇总

| 维度 | 旧项目（`7274626`） | 新项目（`100e9c0`） |
|---|---:|---:|
| CLI 顶层命令数 | 0（仅 Worker/UI 启动开关） | **16**（`help`/`version` + 14 代理命令；另有 Agent 11 个服务开关） |
| CLI 退出码数 | 2（`0`/`1`） | **7**（`0` Ok、`1` AgentUnavailable、`2` Usage、`3` Failed、`4` Unavailable、`5` Timeout、`6` Partial） |
| 设置字段数 | `UiSettings` 13 + `AppSettings` 10 + `UiEntry` 2 + `Theme` 3 + `IdleTrim` 1 | `UserSettings` **6**（协议）+ `ShellPreferences` **1**（`CloseToTray`）+ `StartupRegistration` **1** |
| 本地化语种与键数 | 0 语种 / 0 键（全硬编码） | 资源文件 **3 语种 × 163 键**；`SupportedCultures` 开放 **1 语种**（`zh-CN`） |
| 对照项总数 | — | **15 项**（`H-01` ~ `H-15`） |

## 2. 逐项对照表

| 编号 | 文档要求(路径:行) | 旧项与证据(文件:函数/行) | 新项与证据(文件:函数/op) | 状态 | 验证级别 | 备注 |
|---|---|---|---|---|---|---|
| H-01 | `ADR-007:13-14`：CLI 为正式入口，不引用 `Platform` | 无独立 CLI | `Zentwik.Cli.csproj`、`Program.cs:46-50`、`Commands.cs:13-36` | 新有旧无 | 静态源码 | `observed` |
| H-02 | `ADR-007:15`：命令覆盖状态/监控/设置/方案/场景/账本/诊断 | 无 | `Commands.cs:18-35`（14 代理命令）+ `Program.cs:29-40`（`help`/`version`） | 新有旧无 | 静态源码 | `observed` |
| H-03 | `ADR-007:16`：7 个退出码（0..6） | 仅 `0`/`1` | `Output.cs:6-15`（`ExitCodes` 0..6） | 新有旧无 | 静态源码 | `observed` |
| H-04 | `ADR-007:17-18`：`--json` 共用 DTO，不可用返退出码 4 | 纯文本探针 | `Output.cs:17-54`、`Commands.cs:44,74,232,257,644` | 新有旧无 | 静态源码 | `observed`：`PowerLimitGetAsync:312,323` 失败返 `3` 而非 `4` |
| H-05 | `naming-conventions.md:19,21`：`域名.动作` 小写与 camelCase | `UiSettingsPayload.cs:20-49` | `Protocol.cs:57-58`、`Dtos.cs:82-98` | 一致 | 静态源码 | `observed` |
| H-06 | `ARCHITECTURE_CONSTRAINTS.md:9`：第一版建 i18n 资源管理（最新口径：界面先不处理 i18n 切换 ≠ 只做简体中文；必须走 `Zentwik.Localization`） | 全硬编码中文 | `LocalizationCatalog.cs:13-235`（3 语种 × 163 键，`SupportedLanguages=["zh-CN"]`） | 缺（违反用户最新口径） | 静态源码 | `observed`：原判「部分」将 `SupportedLanguages` 锁 `zh-CN` 当作可接受现状，应标为**缺口** |
| H-07 | `ARCHITECTURE_CONSTRAINTS.md:9`：主题和强调色不得锁死 | `SettingsViewModel.cs:90-226`（Dark/Light/System + 强调色） | `UserSettings.cs:7-11`（协议字段保留）；`MainForm.cs:16-18`（仅系统默认色） | 缺 | 静态源码 | `observed`：按最新裸控件口径裁剪 |
| H-08 | 最新口径：设置页必须保留且有内容（未对接不等于不做） | `SettingsView.xaml` + `BehaviorView.xaml` | `MainForm.cs:697-723`（语言、自启、关闭进托盘、启动说明、模块启停） | 部分 | 静态源码 | `observed`：设置页已保留且有内容（不得写作不需要设置页）；当前未暴露 `MonitorIntervalMs`/`SensorIntervalMs` 属待补齐缺口 |
| H-09 | 最新口径：文案不许硬编码，必须走 `Zentwik.Localization` | 全硬编码中文 | `MainForm.cs:848-957` 走 `_catalog`；`MainViewModel.cs:704-1711` 在基线 `100e9c0` 时实际约 **35 行**含硬编码中文；CLI 亦硬编码中文 | 缺（违反用户最新口径） | 静态源码 | `observed`：原判「29 处、文档与现实冲突」已修正为「约 35 行、违反用户最新口径」。补充：桌面文案硬编码缺口已在产品 commit `2d9351d` 修复（`MainViewModel` 与 `MainForm` 列头全量走本地化表，新增 39 键 × 三语），因本报告基线为 `100e9c0` 故保留历史记录 |
| H-10 | `ARCHITECTURE_CONSTRAINTS.md:10`：设置独立持久化 | `ui.json`/`ui-settings.json`/`theme.json` 等多文件 | `UserSettings.cs:3-18`（存 `state.json`）+ `ShellBehavior.cs:13-57`（`desktop-shell.json` 存 `CloseToTray`） | 部分 | 静态源码 | `observed` |
| H-11 | 托盘宿主与生命周期 | `TrayHost.cs` + `TrayMenu.cs`（含三类方案动态切换子菜单、气泡、自适应图标） | `MainForm.cs:32-33,1707-1739`（WinForms `NotifyIcon`，仅「显示主窗口/退出」） | 部分 | 静态源码 | `observed`：无方案切换子菜单与气泡 |
| H-12 | 关闭主窗口行为 | `CloseBehavior.cs:15-98`（`Exit`/`Tray`/`WorkerOnly`） | `MainForm.cs:1741-1755`（`CloseToTray` 时 `Hide()`） | 部分 | 静态源码 | `observed` |
| H-13 | 开机自启注册 | `AutoStartService.cs`（`schtasks` 提权计划任务 + 路径修复） | `ShellBehavior.cs:60-128`（`HKCU\...\Run\Zentwik`，普通权限）+ `ServiceInstaller`（高权限服务） | 部分 | 静态源码 | `observed`：缺路径漂移自动修复 |
| H-14 | `MIGRATION_MATRIX.md:35`：单实例（文档称客户端无状态无需单实例） | `SingleInstanceGate.cs`（UI 互斥体+激活管道）+ `Worker` 互斥体 | `Desktop/Program.cs:9-26` **无互斥体**（开启托盘后多开会产生多窗口与多托盘图标）；`Agent/Program.cs` 控制台模式无互斥体 | 文档与现实冲突 | 静态源码 | `observed` |
| H-15 | 设置与协议同步完整性 | `AppSettings.cs:92-185`（全字段同步） | `MainViewModel.cs:1525`（`PersistSettingsAsync` 未传 `MonitorIntervalMs`/`SensorIntervalMs`） | 部分 | 静态源码 | `observed` |
