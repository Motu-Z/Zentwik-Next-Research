# 分片 A：UI 页签 / 导航 / 控件 / 交互只读静态审计

- **旧项目基准**：`Motu-Z/Zentwik@main` (`727462601911609e6dacbe3cb8f144246829087a`)
- **新项目对照**：`Motu-Z/Zentwik@feat/gui-protocol` (`100e9c0f4f9e7f9b160f8b0bed9ea99afdf20922`)
- **研究文档基准**：`Motu-Z/Zentwik-Next-Research@main` (`7680e5109ed010e4681180393ff85fb2ce412554`)
- **判定纪律**：只读静态源码审计；区分 `observed` / `derived` / `unknown`；WinForms 裸控件为当前最新口径，`AI_UI_DESIGN_GUIDANCE.md` / `UI_UX_DESIGN_SPEC.md` / `prototypes/` 标为 `Deprecated，仅记差异`。

## 1. 闭合计数汇总

| 维度 | 计数 | 构成明细与源码依据 |
|---|---:|---|
| 旧 UI 页签数 / 视图数 | **8 页签 / 14 视图 (+1 主壳)** | `Zentwik.UI/MainWindow.xaml:208-276` 侧栏 8 导航 `RadioButton`；`Views/*.xaml` 14 视图/浮层 + `MainWindow.xaml`；另有托盘菜单（`Zentwik.Worker/Tray/TrayHost.cs`）与桌面右键菜单（`DesktopContextMenuService.cs`）。 |
| 旧 UI 控件 / 命令绑定数 | **186 控件 / 72 命令绑定** | `Button` 77、`RadioButton` 45、`ToggleButton` 20、`TextBox` 14、`Slider` 10、`ComboBox` 9、`Popup` 6、`ListBox` 4、`CheckBox` 1；XAML `Command="{Binding ...}"` 72 处。 |
| 新 UI 页签数 | **7 页签 (+3 内嵌分组)** | `MainForm.cs:175-181,233-241`：`监控` / `调校` / `进程` / `电源` / `场景` / `日志` / `设置`；方案、诊断、模块分别内嵌于调校、日志、设置页。 |
| 新 UI `DataGridView` 数 | **13 个** | `_capsGrid`、`_historyGrid`、`_coresGrid`、`_ledgerGrid`、`_plansGrid`、`_processGrid`、`_maskGrid`、`_powerSchemesGrid`、`_powerBackupsGrid`、`_scenesGrid`、`_logGrid`、`_doctorErrorsGrid`、`_modulesGrid`（`MainFormSmokeTests.cs:36-37` 断言 13 个）。 |
| 新 UI 按钮 / 命令绑定数 | **32 按钮 / 28+1 绑定 / 29 命令** | `MainForm.cs` 23 `Bind` + 5 `BindWithConfirm` + 3 直接 `Click` + 1 禁用占位（`_refreshProcessButton`）；`MainViewModel.cs:169-197` 共 29 个 `ICommand`。 |

## 2. 逐页逐控件静态对照表

| 编号 | 文档要求(路径:行) | 旧项与证据(文件:函数/行) | 新项与证据(文件:函数/op) | 状态 | 验证级别 | 备注 |
|---|---|---|---|---|---|---|
| A-01 | `docs/coordination/NEXT.md:5-8,53`（WinForms 裸控件，无主题/强调色/缩放/自绘/动画，Flutter 已删）；`project/UI_UX_DESIGN_SPEC.md:17-34`（Deprecated，仅记差异） | WPF 无边框窗口 + `WindowChrome` + `Tokens.Dark/Light` + `Accent` + 300ms 侧栏动画（`MainWindow.xaml:1-34,47-58`；`MainWindow.xaml.cs:151-168`） | `Form` 原生边框，系统默认色，无主题/强调色/缩放/自绘/动画，`AutoScaleMode.Dpi`（`MainForm.cs:14-23,216-222`；`MainViewModel.cs:48-52` 仅协议字段透传） | 一致 | 静态源码 | `observed`：Deprecated 规范已废止 |
| A-02 | `docs/coordination/NEXT.md:18`；`MainFormSmokeTests.cs:26-29`（7 页签口径） | 左侧 8 个 `RadioButton`：调校/调度/电源/场景/日志/通用/行为/高级（`MainWindow.xaml:208-276`） | 顶部 `TabControl` 7 页：监控/调校/进程/电源/场景/日志/设置（`MainForm.cs:175-181,232-241`） | 一致 | 静态源码 | `observed`：`MainForm.cs:14` 类注释仍写“九页”，属注释滞后 |
| A-03 | `docs/coordination/NEXT.md`；`StatusStrip` 在底部 | 侧栏底部 Worker 连通圆点 + 点击重连；顶右角自绘窗控（`MainWindow.xaml:278-357`） | 底部 `StatusStrip`（`Dock=Bottom`）含 `_agentStatus` 与 `_dataDirStatus`；顶部可关闭 `_noticePanel`（`MainForm.cs:43-48,253-304`） | 部分 | 静态源码 | `observed`：缺点击重连入口，改后台自动重连 |
| A-04 | `docs/protocol/gui-contract.md:25-47` | 无独立监控页 | **监控页** `_monitorPage`：4 指标 + `_capsGrid` + `_historyGrid` + `刷新`/`刷新历史`（`MainForm.cs:308-354`；`telemetry.subscribe`, `agent.status`, `capabilities.list`, `telemetry.history`） | 部分实现 | 静态源码 | `observed`：`MainViewModel.cs:1487` 发 `HistoryChanged`，但 `MainForm.cs` 未订阅绘制折线，且实时卡未显示温度/功耗/逐核频 |
| A-05 | `docs/protocol/gui-contract.md:98-99,186` | **调校页**：AMD 全核/CCD/逐核 CO 滑块 + `FMax` 滑块 + Intel 电压/8 列睿频写入（`TuningView.xaml:160-747`） | **调校页** `_tuningPage`：只读 `_tuningSummaryLabel` / `_intelSummaryLabel` / `_coresGrid` + `_ledgerGrid` + `_plansBox`（`MainForm.cs:443-495`） | 部分 | 静态源码 | `observed`：无逐核滑块；`MainViewModel.BuildPlanFromForm` 仅解析 CO，**无法在 UI 创建 FMax 或 Intel 电压方案**（调用链断在 UI 表单） |
| A-06 | `docs/protocol/gui-contract.md:75-80,182-190` | `ProfileBar` + `ProfileOverlay` 弹出层（`ProfileBar.xaml:15-103`） | 内嵌 `_plansBox`：`_plansGrid` + `应用`/`回退`/`删除`/`新建方案` + 折叠表单 `_planForm`（`MainForm.cs:358-393`；`plans.*`） | 部分 | 静态源码 | `observed`：新建方案为单行文本解析，无行内重命名 |
| A-07 | `docs/protocol/gui-contract.md:197-208` | **调度页**：搜索/过滤/排序/批量掩码与优先级/行内下拉（`ProcessView.xaml:33-414`） | **进程页** `_processPage`：提示缺 `process.*` + 空 `_processGrid`/`_maskGrid` + 禁用 `_refreshProcessButton`（`MainForm.cs:168-172,660-693`） | 缺 | 静态源码 | `observed`：后端无 `process.*` op，UI 为空骨架 |
| A-08 | `docs/protocol/gui-contract.md:188` | **规则与掩码抽屉** + `MaskManagerOverlay`（`MaskManagerOverlay.xaml:163-517`） | 仅 `_maskGrid` 空占位（`MainForm.cs:170,672`），需在方案文本框手写 `进程=掩码:优先级[:cpusets]` | 缺 | 静态源码 | `observed`：无独立掩码库 CRUD 与核拓扑勾选器 |
| A-09 | `docs/protocol/gui-contract.md:89-95,100-101` | **电源页**：系统方案卡片 + 新建/改名/删除/切换 + AC/DC 8 项策略滑块（`PowerView.xaml:64-749`） | **电源页** `_powerPage`：功耗墙读写（`_powerLimitsBox`）+ `_powerSchemesGrid`（派生/重命名/备份/删除）+ `_powerBackupsGrid`（还原）（`MainForm.cs:519-601`） | 部分 | 静态源码 | `observed`：新增功耗墙与备份还原；缺 8 项策略 AC/DC 编辑 UI 与 UI 主动切换活动方案按钮 |
| A-10 | `docs/protocol/gui-contract.md:81-86,192-193` | **场景自动化页**：总控开关 + IF/THEN 编辑器 + 进程补全 + 拖拽排序（`ScenariosView.xaml:285-926`） | **场景页** `_scenesPage`：`_scenesGrid` + `激活`/`停用当前`/`启用停用`/`删除`/`新建`（`MainForm.cs:397-439`） | 部分 | 静态源码 | `observed`：`MainViewModel.TriggerKinds:162-167` 漏掉 `ProcessExited`（后端已支持，UI 未接线） |
| A-11 | `docs/protocol/gui-contract.md:102-103` | **日志页**：搜索/级别过滤/统计卡/打开目录/复制诊断/清空（`LogsView.xaml:176-525`） | **日志页** `_logPage`：`_logGrid` + 内嵌 `_doctorBox`（摘要+最近错误）+ `刷新日志`/`刷新自检`（`MainForm.cs:605-656`） | 部分 | 静态源码 | `observed`：新增自检摘要；缺搜索、级别过滤、清空与打开目录 |
| A-12 | 最新口径：设置页保留且有内容 | **通用/行为/高级**三页（主题/强调色/缩放/桌面右键/托盘/防砖/休眠感知/重置） | **设置页** `_settingsPage`：语言下拉（仅 `zh-CN`）、开机自启、关闭进托盘、启动提示 + 内嵌 `_modulesBox`（`MainForm.cs:499-515,697-723`） | 部分 | 静态源码 | `observed`：设置页有内容且承接模块启停；未暴露 `Theme/Accent/UiScale/MonitorIntervalMs/SensorIntervalMs` |
| A-13 | 最新口径：文案不许硬编码，走 `Zentwik.Localization` | 全量硬编码中文 | `MainForm.cs:848-957` 走 `_catalog`；但 `MainViewModel.cs:704-1823` 有 29 处硬编码中文通知/异常/摘要；`SupportedLanguages=["zh-CN"]` | 文档与现实冲突 | 静态源码 | `observed`：ViewModel 层存在硬编码违规 |
| A-14 | `MainForm.cs:1707-1755` | `ConfirmOverlay` + `ToastHost` + Worker 托盘（含方案切换子菜单） | 5 个危险按钮用 `MessageBox.Show(OKCancel)`；顶部 `_noticePanel`；Desktop `NotifyIcon` 仅「显示/退出」 | 部分 | 静态源码 | `observed`：托盘无方案切换子菜单 |
