# 03 · 桌面 UI、进程/IPC 与部署

- 状态：草案（决策输入，未评审）｜负责人：ui-ipc｜日期：2026-11-25
- 依据：旧项目 `Zentwik` v2.6.0-can16（管道协议 v23）
- 关联约束：权限边界（2）、实时监控分层（6）、插件隔离（8）、i18n/主题（9）、可观测性（11）

标注：**【已验证】** 本机实测或官方文档原文；**【推断】** 有依据未实测；**【建议】** 决策主张。

## 0. 结论摘要

1. **UI 选 WPF（.NET 10），图表自建渲染层**（DrawingVisual/WriteableBitmap + 快照推送），禁用 `Polyline` + 逐点绑定。理由是工具链、既有资产与可访问性，而非"框架排名"。
2. **30–60Hz 是渲染要求，不是数据要求**。本机实测（Ryzen 9 9955HX）证实：PDH 1ms 窗口的 `% Processor Performance` 标准差达 21 个百分点、4ms 窗口 35pp——"能调 4000 次/秒"不等于"有 4000 份独立信息"；且未调 `timeBeginPeriod` 时 1ms 睡眠实测约 15.6ms。故数据面定 1–10Hz 分档、渲染面 30/60Hz，两者用不同时钟驱动。
3. **进程模型：LocalSystem 服务（唯一特权方）+ 中完整性 UI（托盘宿主）**。放弃"每用户提权 Worker + 计划任务自启"；v1 只有两个进程。
4. **IPC：单条长连接命名管道 + 长度前缀二进制帧 + 请求 ID 多路复用 + 服务端事件推送**。旧的"每请求一连接 / 单命令字 / 单版本号 / JSON 套 JSON"无法支撑订阅式遥测，也无法安全暴露给插件。
5. **安全补三件事**：服务端校验调用方（PID→SID/完整性/会话）、UI 校验服务端（防管道抢占）、按方法授权矩阵。旧 DACL 用"当前用户 SID"，服务化后即 SYSTEM，UI 根本连不上。
6. **安装器：WiX v5（MSI + Burn）**，前提是接受作者成本；若不落服务且驱动由 PawnIO 自带安装器负责，沿用 Inno 更省。**不自带内核驱动**，只分发已签名的 PawnIO 模块。
7. **更正任务书前提**：本机存在 `cl.exe`（VS 2026 / MSVC 14.51.36231，19.51.36260），只是不在 PATH；Windows SDK 10.0.26100 也在。真正缺的是 Qt SDK、CMake、Rust。这不改变"Qt 不进 v1"，但理由从"编译不了"改为"新增工具链 + 授权决策 + 全量重写"。

## 1. 判定基线

| 事实 | 取值 | 证据 |
|---|---|---|
| .NET SDK / 运行时 | 10.0.401、10.0.403；含 WindowsDesktop.App 10.0.12/14 | 【已验证】dotnet CLI |
| MSVC / Windows SDK | cl 19.51.36260（14.51.36231，未入 PATH）；SDK 10.0.26100 | 【已验证】直接执行 cl.exe |
| VS / Node / WebView2 | VS 2026 Community；Node 22.22.2；WebView2 158.0.4370.33 | 【已验证】vswhere、注册表 |
| CMake / Qt / Rust | 均无 | 【已验证】 |
| 旧 UI | WPF net10.0-windows，单文件自包含，PMv2 DPI | 【已验证】csproj、app.manifest |
| 旧 Worker | `requireAdministrator` WinExe，托盘 + 管道服务端，Global 互斥体 | 【已验证】manifest、Program.cs |
| 旧协议 | 单行 JSON、512KB、版本 23、每连接一请求、DACL=当前用户 | 【已验证】Contract.cs、PipeServerService.cs |

## 2. A. UI 技术比较

### 2.1 先拆需求：谁真的在 60Hz

| 通道 | 有效更新率（实测/机制） | 界面需要 | 定档 |
|---|---|---|---|
| PDH 利用率/性能 | 可调用 4000/s，但 1–4ms 窗口 std 21–35pp，属量化噪声 | 平滑数字 + 不抖 | **10Hz** |
| Intel 倍频（0x198） | 30–60Hz 才有意义；RAPL（0x611）需 ≥250ms 差分 | 滚动曲线 | **10Hz（倍频）/ 2–4Hz（功耗）** |
| AMD PM table | 一次 transfer + memcpy，可批量 | 多核曲线 | **2–4Hz** |
| SMU 逐核 CO（0x7C/0xD5） | 每核一次 mailbox，16 核一轮 1.6–32ms | 仅应用/校验 | **不用于遥测** |
| 温度 / EC / 风扇 | 0x1B1 ~1Hz；EC 单字节 1–5ms；风扇受转速限制 | 数字 + 慢曲线 | **1Hz** |

**【已验证】** 旧版**没有周期遥测**：SMU 快照只在打开调校页/查询时按需读，`SmuReader.cs:345` 的"2 秒遥测定时器"是遗留注释。因此没有可继承的采样基线，订阅式遥测是新能力而非迁移。

**【建议】** 验收基线：温度/EC 类 ≥1Hz（p99 ≤2s），其余通道 ≥4Hz（p99 ≤500ms）；渲染面 30/60Hz 允许重复帧与插值；**禁止把原始高频样本直接画线**（噪声会被误读为硬件抖动），必须先做窗口聚合或 min/max 抽稀。缺失超过 p99 阈值时显示"数据陈旧"标记，不插值冒充新值；离散量（电源方案、开关状态）用阶梯保持，连续量用线性插值，**不做外推**。

**【建议】** 两个时钟必须分开：采样用 QPC + 高分辨率可等待定时器（`CreateWaitableTimerEx` + `HIGH_RESOLUTION`，避免 `timeBeginPeriod` 的全系统影响）；渲染用呈现时钟（WPF `CompositionTarget.Rendering`，按 Stopwatch 门限降到 30Hz）。用 `DispatcherTimer` 做 60Hz 采样既不准时也不均匀（实测 15.6ms 量级）。

### 2.2 运行时与交互能力（强/够用/需自建/弱）

| 维度 | WPF | WinUI 3 | Avalonia 11 | Qt 6 C++ | Tauri | .NET+WebView2 |
|---|---|---|---|---|---|---|
| 30–60Hz 多序列折线 | 够用（自建渲染层） | 够用（需 Composition/D2D 互操作） | 够用 | 强 | 强 | 强 |
| 大数据表格 | 够用（需列虚拟化） | 够用（DataGrid 非一等） | 够用 | 强 | 强 | 强 |
| 主题/强调色热切换 | 强（已验证） | 强 | 强 | 够用（QSS 受限） | 强 | 强 |
| 运行时语言切换 | 需自建 | 需自建 | 需自建 | 强（QTranslator 一等） | 强 | 强 |
| DPI/多显示器 | 够用（PMv2，位图需手工） | 强 | 强 | 强 | 强 | 强 |
| UIA 可访问性 | 强（参考实现级） | 够用 | 弱–够用 | 弱（桥不完整） | 够用 | 够用 |

### 2.3 交付与工程

| 维度 | WPF | WinUI 3 | Avalonia | Qt 6 | Tauri | .NET+WebView2 |
|---|---|---|---|---|---|---|
| 本机可构建 | 是（仅 SDK） | 是（需 NuGet 还原） | 是 | 缺 Qt/CMake | 否（缺 Rust） | 是 |
| 体积/依赖 | 框架依赖几 MB；单文件 60–90MB | 单文件仅"未打包+自包含"，且更慢更占内存【已验证】 | 60–80MB | 40–80MB | 小，依赖 WebView2 | 同 WPF |
| AOT/单文件 | 单文件 ✔，AOT ✘ | AOT 非官方主线 | 11.x 可用，需 CompiledBinding【推断】 | 不适用 | ✔ | 宿主 AOT ✘ |
| 调试/热重载 | 强（VS） | 强（VS） | 够用 | 强 | 强 | 强 |
| 长期维护 | 受支持，不加大特性【已验证】 | 微软主推，迭代快 | 社区+商业 | 周期长 | 生态年轻 | 双栈维护 |

官方原文（2026-11-25 访问）：WPF "is supported and continues to receive feature, performance, accessibility, and Fluent-style improvements"；同时对"新的通用 Windows 桌面应用"首推 WinUI 3。两者不矛盾：推荐针对通用模板，不是"必须迁移既有调优工具"。

### 2.4 影响决策的要点

- **图表瓶颈不在 GPU，在"每帧重建几何 + 绑定"**。可行路径：采样层 `float[]` 环形缓冲 → UI 只取最新快照（`Interlocked.Exchange` 双缓冲）→ 每帧一条 `StreamGeometry` + 按像素列 min/max 抽稀。禁止 `ItemsControl` 每点一元素、每秒数百次集合变更。
- **表格**：进程 ≤ 2000 行用 `DataGrid` 行虚拟化即可；历史监控必须分页查询，不得整表入内存。
- **主题**是 WPF 最实在的优势：旧 `Tokens.Dark/Light.xaml` + `Accent.xaml` + 槽位替换**已验证**。
- **i18n** 在所有 XAML 方案里成本相近（`x:Static` 到 resx 是静态的），不构成选型差异；方案见第 6 节。
- **可访问性**：自绘图表必须实现 `AutomationPeer`，暴露"当前值摘要 + 数据表替代"；自定义焦点指示对比度 ≥ 3:1。
- **交付**：WPF 自包含单文件在旧 CI 已跑通（`PublishSingleFile` + 原生库自解压），是已验证的最低风险路径。

### 2.5 推荐与成立前提

**【建议】WPF（.NET 10）+ 自建图表控件 + CommunityToolkit.Mvvm**。前提：

- **P1** v1 只承诺 Windows 10 1809+ / 11；跨平台按能力逐项定义。
- **P2** 团队已具备 WPF/XAML 产能（旧项目即证据），迁移是重构而非重学。
- **P3** 不为 UI 引入第二套构建链（Node/Rust/Qt）。
- **P4** 实时要求 = 4–10Hz 数据 + 30/60Hz 渲染、≤16 序列 × ≤600 可视点，不是百万点散点。
- **P5** UIA 可访问性是硬要求。
- **P6** 接受框架不新增平台特性，Fluent 视觉由自有令牌层实现。

### 2.6 回退边界

| 触发 | 动作 | 代价 | 决策截止点 |
|---|---|---|---|
| P1 失效（需 macOS/Linux） | 换 Avalonia，保留 MVVM/令牌/契约 | 图表控件重写、UIA 覆盖下降 | 自建控件超过 5 个前 |
| P4 失效（60Hz×数十万点） | WPF 外壳 + GPU 渲染岛（原生 SwapChain / WebView2 WebGL），或迁 WinUI 3+D2D | 进程内互操作复杂度 | 首个监控页定稿前，先做 1 天原型 |
| P3 失效（允许前端栈） | 保留服务与 IPC，UI 换 .NET+WebView2 | 双语言、内存 +100MB 级、逐控件验 a11y | 首个正式版前 |
| P5 降级 | 仍用 WPF | 需书面接受辅助技术缺陷 | 评审签署 |
| P6 失效（WPF 停更） | 迁 WinUI 3 / Avalonia | 全量 XAML 迁移 | 年度架构复审 |

**不推荐 Tauri 作为 v1**：缺 Rust 只是表层；核心是特权/驱动仍在 C#，会形成 Rust+TS+C# 三语言与两套 IPC 的长期维护面。

## 3. B. 进程与权限模型

### 3.1 旧形态的结构性代价

旧形态：UI（中完整性）+ Worker（`requireAdministrator`，同目录，托盘在 Worker）+ 计划任务自启；`Global\` 互斥体保单实例；管道每请求一连接，DACL 为当前用户 SID。

| 现象 | 根因 | 后果 |
|---|---|---|
| 冷启动弹 UAC | 提权 exe | 用户放弃；自动化不可用 |
| 登录前无法应用基线 | 自启是"登录时"任务 | 重启后到登录前未调校 |
| 崩溃不恢复 | 无 SCM 监管 | 监控/自动化静默停摆 |
| 升级须先杀进程 | 文件被占用 | 失败留半套文件 |
| RDP 第二会话不可用 | 每会话一进程 + Global 互斥 | 多用户场景失效 |
| 同用户任意进程可写 SMU | 无调用方校验 | 本地提权 |
| （服务化后）UI 连不上 | DACL 取当前用户 = SYSTEM | 交互用户被拒 |

### 3.2 三种特权宿主

| 维度 | 系统服务 | 每用户提权进程（旧） | 计划任务（最高权限） |
|---|---|---|---|
| UAC | 仅安装 | 每次启动 | 无 |
| 启动时机 | 开机，登录前 | 登录后 | 登录后（或会话 0） |
| 会话隔离 | 会话 0，无 UI | 同用户会话 | 同左 |
| 崩溃恢复 | SCM + preshutdown | 无 | 有限 |
| 升级 | 停/换/启，可回滚 | 杀进程 | 杀进程+重注册 |
| 托盘/GUI | 不可能 | 原生 | 原生 |
| 权限粒度 | 全机 SYSTEM | 单用户管理员 | 同左 |
| 企业环境 | 标准 | 依赖 UAC 策略 | 常被 GPO 限制 |

### 3.3 推荐形态

**【建议】`ZentwikSvc`（LocalSystem，自动启动，SCM 恢复）+ `Zentwik.UI`（中完整性，登录自启，托盘宿主）**。

- 服务是唯一持有硬件写权限、驱动句柄、系统策略写权限的进程；UI 永不自我提权。
- 托盘在 UI：关闭窗口=隐藏；"退出"=进程结束（托盘消失）。**v1 不承诺"UI 全退但托盘常驻"**。
- 自启用 `HKCU\...\Run`（免管理员、免计划任务、避 GPO）；服务自启交给 SCM。
- 引入 `SessionAgent` 的唯一时机：要求"UI 可完全卸载但托盘/热键常驻"或需 per-session 窗口操作；届时它是中完整性薄进程。

收益：开机即生效、崩溃自恢复、升级路径清晰、多会话共享机器状态、UAC 只在安装/升级出现。
代价：服务不能弹 UI/UAC；多会话争用同一硬件状态，必须串行化 + 归属记录 + 冲突可见；SYSTEM 权限过大，需最小化 SeDebugPrivilege、仅本机、专用 SID ACL、签名发布。

### 3.4 职责边界

| 能力 | UI | 服务 | 插件宿主 | SessionAgent（后续） |
|---|---|---|---|---|
| 渲染/交互/本地化 | ✔ | ✘ | ✘ | ✘ |
| 硬件读写（SMU/MSR/驱动） | ✘ | ✔ | ✘ | ✘ |
| 电源/调度/进程策略 | ✘ | ✔ | ✘ | ✘ |
| 持久化主副本 | ✘ | ✔ | ✘ | ✘ |
| 权限校验/授权矩阵 | ✘ | ✔ | ✘ | ✘ |
| 第三方插件加载 | ✘ | ✘ | ✔（低完整性） | ✘ |
| 托盘/全局热键 | ✔ | ✘ | ✘ | ✔（若引入） |

**【建议】** 硬件设备句柄只在服务内打开并持有，**不通过 IPC 传递、不允许复制到 UI 或插件**；IPC 只暴露"读取快照 / 应用请求 / 订阅事件"。旧版 SMU 串行锁是进程内 `static`，两进程同时打开 `\Device\PawnIO` 时没有任何跨进程仲裁——单进程独占句柄是唯一可靠解法（依据 `02-hardware-access.md`）。

**【建议】** 插件不直连服务：插件 →（宿主 broker）→ 服务，服务看到的调用方是"插件宿主"，权限由服务签发的 capability token 限定（绑定 PID+SID+启动时间，防 PID 复用）。v1 不开放第三方插件 UI；后续只允许**声明式 UI schema**，禁止插件 XAML 进入 UI 进程（XAML 可执行任意代码）。

### 3.5 关键失败模式

1. **管道抢占**：中完整性进程先建同名管道伪装服务 → 服务端用 `FILE_FLAG_FIRST_PIPE_INSTANCE`；UI 连接后校验服务端 PID 的映像路径与签名。
2. **卸载后硬件仍处调校态** → 卸载前恢复默认，无法恢复时明确提示需重启。
3. **启动即写硬件导致引导失败循环** → 沿用 BootGuard：连续异常启动则跳过自动应用，仅起只读监控。
4. **休眠/唤醒状态漂移** → 注册电源通知，恢复后读回校验而非假定生效。
5. **两会话改电源方案冲突** → 串行化 + 返回 `CONFLICT_OWNED_BY_OTHER_SESSION`，不静默覆盖。
6. **配置所有权分裂**（旧版实测）：Worker 写 `ui.json`，UI 直接写 `theme.json`/`ui-entry.json`/`ui-state.json`，Worker 又手工读 `ui-entry.json` 决定托盘菜单。→ 按"谁消费"划分：仅 UI 消费的（主题、窗口位置、密度）归 UI；**跨进程消费的（自启形态、后台开关、托盘入口）归服务**，UI 只发命令、不写文件。

## 4. C. IPC 设计

### 4.1 传输方案

| 方案 | 延迟/吞吐 | 复杂度 | AOT | 结论 |
|---|---|---|---|---|
| 命名管道（字节 + 自定帧） | 本地几十 µs 往返 | 低 | 友好 | **推荐** |
| 命名管道（消息模式） | 同上 | 低 | 友好 | .NET 封装下"部分读"语义易错【推断】 |
| gRPC over pipe | 略高（HTTP/2） | 高（Kestrel + 自定义 ConnectCallback） | 体积/裁剪成本上升 | 仅在插件必须多语言时考虑 |
| ALPC | 最低 | 极高（无托管 API） | — | 拒绝 |
| 共享内存 + 事件 | 最低（零拷贝） | 中高（布局版本化、生命周期、ACL） | 友好 | 后续遥测快车道，不作主通道 |

### 4.2 旧协议的具体问题

| 问题 | 机制 | 后果 |
|---|---|---|
| 单一整数版本 | 任何改动即递增并拒绝对端 | 滚动升级不可能；混搭=全盘不可用 |
| 无请求 ID | 只能信号量串行 | 单飞、长操作阻塞一切、无法取消 |
| 每请求一连接 | UI 每次调用新建连接（默认 3s 超时），服务端 accept→读一行→回一行→关 | 连接/句柄抖动，无法推送 |
| 无推送 | 方案 30s、进程页 3s 轮询 | 刷新粗、CPU 浪费、事件延迟不可控 |
| `Payload` 是 string | JSON 里套 JSON | 双重转义、无 schema、AOT 反射风险 |
| 命令字平铺 | 46 条命令、v23 内演进 23 次，语义只在注释 | 无机器契约、无参数校验、无按方法授权 |
| `Success+Message` | 人读文本兼机器判据 | 改文案即破坏调用方（旧注释自认） |
| 512KB 单行 | 超限即协议错误 | 大响应被迫截断；无流式 |
| 无调用方校验 | 仅 DACL | 同用户任意进程可写 SMU |
| 无背压 | 写即阻塞 | 慢客户端拖住服务端 |

### 4.3 推荐帧格式

字节模式、小端、24 字节头：

| 偏移 | 类型 | 字段 | 说明 |
|---|---|---|---|
| 0 | u32 | Length | 载荷字节数，不含头 |
| 4 | u8 | Type | 1 Hello / 2 HelloAck / 3 Request / 4 Response / 5 Event / 6 Cancel / 7 Ping / 8 Pong / 9 Error |
| 5 | u8 | Flags | bit0 二进制 / bit1 分片 / bit2 压缩 |
| 6 | u16 | Channel | 0 控制 / 1 遥测 / 2 插件 |
| 8 | u64 | CorrelationId | 客户端单调递增；事件为 0 |
| 16 | u32 | MethodId | schema 生成的数字 ID；插件通道 `0xFFFFFFFF` 时名称在载荷 |
| 20 | u32 | DeadlineMs | 0=无截止，服务端据此拒绝过期请求 |

- 控制面：UTF-8 JSON + 源生成序列化，信封 `{ "params": {...} }`；可读、可抓包、AOT 安全。
- 遥测：列式二进制 `{streamId, firstSeq, sampleCount, intervalMs, channelMask}` + 每通道 `float32[]`，零每样本分配。
- 上限：控制 1 MiB、遥测 64 KiB；超限**断开并记录**（不尝试跳过，无法验证边界）。

### 4.4 关联、事件、背压

- 一条长连接承载全部请求；`CorrelationId` 客户端分配、服务端回填，**允许多请求并发**（默认在飞 8）。
- 订阅：`monitoring.subscribe {streams, intervalMs, window}` → `subscriptionId`；服务端按间隔合并推送 Event；断连即回收。
- 心跳：空闲 15s Ping，3 次无 Pong 判定半开并重建。

| 流 | 策略 | 必须可观测 |
|---|---|---|
| 遥测 | 有界队列（2 帧）+ **丢最旧** | 每帧 `droppedSinceLast` + 累计计数 |
| 状态事件 | 按 key 合并，latest-wins | 合并次数 |
| 进度 | 100ms 节流 | 阶段耗时 |
| 命令响应 | **绝不丢**，受 Deadline 约束 | 超时率 |
| 日志 | 拉模式（游标 `afterSeq`） | 落后量 |

UI 侧同样有界：只保留"最新快照 + 环形历史"，渲染线程每帧读快照，不在 UI 线程排队处理每样本。

### 4.5 版本协商（替代单版本号）

- `Hello {major, minor, features[], catalogHash}` → `HelloAck` 回交集。
- `major` 不同：`PROTOCOL_MAJOR_UNSUPPORTED` + 双方版本 + 可执行提示，然后关闭。
- `major` 同 `minor` 不同：按 features 交集工作；未知方法返回 `METHOD_UNKNOWN`，**绝不静默成功**（旧版最危险的失败模式是"旧端忽略字段仍回 Success"）。
- `catalogHash` 不同仅告警；`describe` 返回方法表与参数 schema 哈希，UI 据此灰化功能。
- 破坏性变更才升 `major`；增量字段不再强制全端升级。

### 4.6 错误模型

信封：`outcome ∈ {ok, unavailable, rejected, partial, failed}`，与传输层严格分离（管道断开=异常，不是 `failed`）。

```
{ "outcome": "partial", "code": "POWER_WRITE_PARTIAL", "category": "hardware",
  "retryable": false, "messageKey": "power.apply.partial",
  "messageArgs": { "failed": ["boostMode"] },
  "context": { "target": "scheme:381b4222", "stage": "verify" } }
```

- `code` 是稳定枚举（不随文案变）；`messageKey` 由 UI 本地化；服务端日志保留英文 detail。
- `category`：protocol/auth/validation/state/hardware/timeout/cancelled/internal。
- **通道可用性是一等字段，不允许压成 bool**：`channelState ∈ {ok, NotSupported, DriverUnavailable, ProbeFailed}` 随快照与错误一同返回。`NotSupported` = 平台不符（不是故障）、`DriverUnavailable` = 未装驱动/无权限、`ProbeFailed` = 设备已打开但硬件不应答；合并三者会重犯旧 CHANGELOG 记过的"把不适用报成不可写"（依据 `02-hardware-access.md`）。UI 必须能呈现"某能力不可用但其余功能正常"，不得升级为全局错误页。
- 严格区分"查询成功但目标不可用"（`ok` + `available:false`）与"查询失败"；旧项目此点做对，须保留为全局原则。
- 每个 code 在契约程序集有常量与单测，禁止临时拼字符串。

### 4.7 ACL 与调用方校验

1. 服务端显式 DACL（SYSTEM 全控 + Authenticated Users 读写）、`PIPE_REJECT_REMOTE_CLIENTS`、`FILE_FLAG_FIRST_PIPE_INSTANCE`；**不用"当前用户 SID"**。
2. 客户端 `SECURITY_SQOS_PRESENT | SECURITY_IDENTIFICATION`（禁止服务端借连接冒充）。
3. 服务端 `GetNamedPipeClientProcessId` → `OpenProcess(QUERY_LIMITED_INFORMATION)` → `OpenProcessToken(TOKEN_QUERY)` → SID / 完整性 / SessionId / ElevationType。
4. 判定：会话匹配（会话级操作）、完整性 ≥ Medium、插件宿主须持 capability token、可选 `WinVerifyTrust` 校验映像签名。
5. 按方法授权（read/monitor/write/hardware/admin），默认拒绝。
6. 审计 `ClientIdentity` + `CorrelationId`；服务端**不以客户端身份**执行硬件写入。

## 5. D. 安装、升级、生命周期、诊断

| 方案 | 服务 | 驱动 | 静默升级 | 回滚/修复 | 企业 | 成本 |
|---|---|---|---|---|---|---|
| WiX v5（MSI+Burn） | 原生 | DifxApp | 事务性 | 强 | 好 | 陡 |
| Inno Setup 6（旧） | `sc create` | 自定义 | 可用 | 弱（无 repair） | 一般 | 低 |
| 自带 updater | 自建 | 不适用 | 强（灰度） | 依赖安装器 | 差 | 中高 |
| MSIX | 不支持 | 不支持 | 强 | 强 | 好 | 与服务/驱动冲突，排除【推断】 |

**【建议】** WiX v5 + Burn（服务、可选驱动组件、运行时前置检查、MajorUpgrade 可回滚）。前提：接受 MSI 作者成本（WiX 4+ 为 dotnet tool，可在无 VS 的 runner 构建【推断】）；**驱动本体由 PawnIO 官方安装器/签名链负责**，我们只分发已签名模块，并在能力模型中如实标记"驱动未安装→该能力不可用"。PawnIO 安装需管理员、且可能被反作弊/安全软件拦截，因此 UI 必须把这种情况呈现为"单项能力不可用、其余功能正常"并给出获取/修复入口，**不得弹全局错误页或阻断启动**（依据 `02`）。若 v1 不落服务，则继续用 Inno。

**升级状态机（幂等）**：UI 确认 → 签名校验过的 updater（一次 UAC）→ `maintenance.enter`（暂停新命令、排空在飞、持久化期望状态、可选恢复安全默认）→ `sc stop`（preshutdown 30s）→ MSI 事务安装 → `sc start` → **期望状态对账**（读回硬件实际值，不一致按策略重放并记录）→ UI 重连并按 `major` 协商；旧 UI 不同 `major` 必须提示重启，不允许半新半旧静默工作。

**降级**：仅同 `major` 且 `dataSchemaVersion` 兼容；数据迁移只向前，迁移前备份 `settings.bak.<schema>`；跨 `major` 直接拒绝并提示导出。

**诊断**：WER `LocalDumps` 为 `ZentwikSvc.exe`/`Zentwik.UI.exe` 配 MiniDump（可选 full）；服务日志 `%ProgramData%\Zentwik\logs`（用户可读），UI 日志 `%LocalAppData%\Zentwik\logs`，JSON Lines + 滚动 10MB×10 + 14 天；`EventSource`/ETW 供帧耗时、队列深度、丢弃数；支持包 = 日志 + 版本 + 能力快照 + 硬件身份哈希（无序列号）+ 脱敏配置。每次写操作记录发起者/触发/目标/期望/实际/失败阶段/恢复结果（约束 11）。

**卸载**：停 UI → `maintenance.enter` → 恢复默认（默认询问）→ 停/删服务 → 删模块文件 → 删 `Run` 项与快捷方式；用户数据默认保留并告知路径（沿用旧行为），删除失败不中止卸载。

## 6. E. 国际化与主题

- **资源**：JSON 目录为唯一事实源（`strings/<module>/<locale>.json`），源生成强类型键，构建期内嵌（单文件友好），允许模块包扩展；CI 校验键集合/占位符一致、无空值、超长告警。resx 因"静态属性不可变 + 模块扩展繁琐"不采用。
- **切换**：单一 `ILocalizationService` + `PropertyChanged("Item[]")`；XAML 用 `{l:Loc Key=...}` 绑定索引器，切换不重建窗口。回退链 `zh-TW → zh-Hant → en-US`（**不回退 zh-CN**，简繁混排比英文更糟），`zh-CN → en-US`；缺键开发期抛异常，发布期显示 `⟦key⟧` 并上报。
- **复数/格式**：目录内 `{one, other}` 按 `PluralRules` 选择；中文无复数变化。**两套 Culture 分离**：`CurrentUICulture` 管文案，`CurrentCulture` 管数字/日期/单位；协议与持久化一律 invariant。实时数值用等宽数字（`NumeralAlignment=Tabular`），避免刷新时宽度跳动。
- **用户数据**：名称不翻译；ID 与显示名分离；身份比较 `Ordinal`，显示排序用 `CurrentCulture`；长度限制按字素簇（防 emoji 截断）。
- **CJK 字体**：字体链 `"Segoe UI Variable Text, Segoe UI, Microsoft YaHei UI, Microsoft JhengHei UI, Yu Gothic UI"`（Win10 无 Variable，必须保留 Segoe UI）；正文行高 +10–15%；小字号用 `TextFormattingMode=Display`；禁止定宽按钮（英文字符串长 30–40%）。
- **主题令牌**三层：原始色板 → 语义令牌（`color.surface.base`/`color.text.primary`/`color.state.danger`）→ 图表令牌（`chart.series.1..8` 固定 8 色，与强调色**解耦**，需色盲可辨 + 明暗双对比度，线型作第二编码）。强调色运行时生成 `accent.{base,hover,pressed,subtle,on-accent}` 并**对比度钳制**（文本 ≥4.5:1、非文本 ≥3:1，不达标自动翻转 on-accent）。高对比度下切系统色板并禁用自定义强调色与半透明。密度用令牌集，缩放用布局度量而非 `ScaleTransform`。

## 7. F. 测试策略

| 层级 | 目标 | 手段 | 门槛 |
|---|---|---|---|
| 领域单测 | 应用顺序、所有权账本、场景仲裁、掩码解析 | 纯内存 + 接口替身 | 分支覆盖 ≥80%，无真实硬件 |
| 契约/序列化 | 帧编解码、错误码、版本协商 | Golden 帧 + 截断/超长/垃圾 fuzz | 畸形输入不崩溃、不误判成功 |
| 管道集成 | 并发、背压、断连 | 进程内 + 跨进程桩；100 并发；`kill -9`；慢读者 | 无句柄泄漏；丢弃计数正确 |
| 授权 | 低权限/伪造调用方被拒 | 受限令牌 + 伪造 token | 拒绝率 100%，审计有记录 |
| 服务生命周期 | 安装/启停/preshutdown/重启 | SCM 脚本 + 断电（VM 快照） | 重启后与期望一致 |
| 硬件在环 | 写后读回、范围校验、重启保持 | AMD Zen3/4/5、Intel 混合、笔记本 ASPM | 写读一致；不支持项如实 `unavailable` |
| UI 冒烟/视觉 | 主题×语言、DPI 100/150/200 | 自动启动 + 截图对比 + 10 分钟 30Hz | 无崩溃；内存增长 <5%；无布局溢出 |
| 性能 | 帧预算、分配、CPU | ETW + dotnet-counters | UI 线程 ≤8ms/帧；稳态 gen0 ≈0 |
| 可访问性 | UIA 树、键盘、对比度、缩放 | Accessibility Insights + Narrator | 交互控件均有 Name/Role；焦点可见 |

补充：**升级中断测试**（"服务已停、文件换一半"处强杀，验证可重入）；**卸载后状态测试**（卸载→重启→读回，确认无残留欠压/超频）。CI 用 `windows-latest` + 固定 .NET 10 SDK 即可跑单元/契约/UI 冒烟（WPF 不需要 MSVC）；硬件与 ACL 测试进独立 lab，不进 PR 门禁。

## 8. 风险与未决

| 编号 | 问题 | 影响 | 责任方 | 关闭条件 |
|---|---|---|---|---|
| U1 | SMU/MSR/EC 的 IOCTL 延迟未实测 | 只影响遥测上限的精确值，不影响分档结论 | hardware 组 | 管理员会话跑 3 组微基准（MSR 读、AMD 整轮 CO、PM table） |
| U2 | WPF 60Hz 自绘未实测 | P4 前提缺数据 | ui-ipc | 1 天原型：16×600 点 @60Hz 帧耗时与 CPU |
| U3 | 服务化与旧版差距大 | 工期与回滚风险 | team-lead | 决策：v1 是否必须"登录前生效" |
| U4 | PawnIO 驱动安装责任边界 | 安装器合规 | hardware + lead | 明确谁安装/签名 |
| U5 | zh-TW 回退目标 | 文案质量 | 产品 | 签署 |
| U6 | 插件 UI 扩展形态 | 插件架构预留 | runtime | v1 冻结为"无第三方 UI" |
| U7 | 遥测列式布局长期兼容 | 数据面演进 | ui-ipc | 帧头带 `streamSchemaVersion` 并入协商 |

## 9. 参考资料（访问日期 2026-11-25）

1. Windows App SDK deployment overview（learn.microsoft.com/en-us/windows/apps/package-and-deploy/deploy-overview）：单文件仅"未打包+自包含"（WASDK 1.5+）；未打包自包含"加载更慢、内存更高"。
2. Client frameworks FAQ（.../windows/apps/get-started/client-frameworks-faq）：WPF 未被弃用，持续获得功能/性能/可访问性/Fluent 改进；新通用桌面应用首推 WinUI 3。
3. 旧项目只读：`WorkerPipeContract.cs`、`PipeRequest/Response.cs`、`WorkerPipeClient.cs`、`PipeServerService.cs`、`Program.cs`、`installer/zentwik.iss`、两个 `app.manifest`、CI 发布步骤。
4. 同批研究：`01-legacy-inventory.md`（命令清单/传输模型/配置所有权）、`02-hardware-access.md`（句柄独占/通道三态/PawnIO 安装约束）。
5. 本机实测：`dotnet --list-sdks`、`--list-runtimes`、直接执行 `cl.exe`、`reg query ...EdgeUpdate\Clients`、`node -v`。
6. Win32 命名管道安全 API（`CreateNamedPipe`、`PIPE_REJECT_REMOTE_CLIENTS`、`GetNamedPipeClientProcessId`、`SECURITY_SQOS_PRESENT`）按语义引用，未逐条抄录文档。

## 10. 协作记录

- 2026-11-25：同步 hardware 组——(a) 本机有 MSVC 14.51 与 Win SDK 10.0.26100（cl 未入 PATH），原生构建可用 vcvars；(b) 需其给出传感器实际采样率以关闭 U1。
- 2026-11-25：向 team-lead 说明"无 cl.exe"前提需更正，且不改变 UI 选型结论。
- 2026-11-25：采纳 `02-hardware-access.md` 两条输入——设备句柄不跨进程（3.4）、通道三态透传（4.6）；并补充 PawnIO 安装受阻时的 UI 降级呈现（5.2）。
- 2026-11-25：采纳 `01-legacy-inventory.md` 事实——46 命令/v23、3s 超时、方案 30s 与进程 3s 轮询（4.2）、配置所有权分裂（3.5 第 6 条）。
- 2026-11-25：采纳 `02` 第 6 节实测——PDH 1ms/4ms 窗口 std 21/35pp、1ms 睡眠实测 ≈15.6ms、旧版无周期遥测；据此重写 2.1（分档 10/2–4/1Hz、p99 ≤500ms 验收、双时钟、去噪后才可绘），并把 U1 缩小为"SMU/MSR/EC IOCTL 延迟待管理员基准"。
