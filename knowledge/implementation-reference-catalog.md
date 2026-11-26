# Zentwik 外部参考项目与资料目录

- 文档性质：候选参考目录与研究任务入口，不是技术规范、架构决策或实现承诺。
- 目的：集中记录 Zentwik 后续开发可能需要调查的成熟项目、工具和官方资料，避免临时寻找、重复讨论或混淆领域。
- 状态：
  - **已研究**：已有逐行源码核对或明确研究记录，具体证据以专题文档为准。
  - **待研究**：值得调查，尚未完成源码或官方文档核验。
  - **功能对照**：闭源产品或商业工具，只用于观察功能和行为，不作为源码依据。
  - **官方资料**：平台/API/协议的一手文档。
- 原则：参考项目只提供可比较的实现方案。不得直接复制架构，不得把单一工具的限制当成 Windows/硬件限制，也不得把候选项视为必须引入的依赖。

## 1. 总览

| 领域 | 主要参考 | 状态 |
|---|---|---|
| AMD CPU 调校 | ZenStates-Core、SMUDebugTool、RyzenAdj | 已研究 |
| Intel CPU 调校 | Linux kernel、linux-intel-undervolt | 前者已研究，后者待研究 |
| 进程调度与约束 | Process Governor、Process Lasso、System Informer | 待研究/功能对照 |
| 电源计划 | Windows PowrProf、powercfg、Process Lasso、G-Helper | 官方资料已纳入规范；工具待研究/功能对照 |
| 场景与自动化 | Process Lasso、G-Helper、PowerToys | 功能对照/待研究 |
| 硬件监控 | LibreHardwareMonitor、HWiNFO、FanControl | 待研究/功能对照 |
| 服务与 IPC | StreamJsonRpc、PowerToys、.NET Runtime | 待研究 |
| 插件与模块 | PowerToys、.NET AssemblyLoadContext | 待研究/官方资料 |
| UI / MVVM | CommunityToolkit.Mvvm、WPF-UI、WPF | 待研究 |
| CLI | System.CommandLine、Spectre.Console | 待研究 |
| 安装与更新 | WiX Toolset、Velopack、Inno Setup | 待研究 |

## 2. CPU 与硬件调校

已研究项目的源码证据、文件位置和研究结论见 [成熟实现参考（按领域索引）](hardware/cpu/implementation-references.md)。

| 项目 | 地址 | 研究内容 |
|---|---|---|
| ZenStates-Core | https://github.com/irusanov/ZenStates-Core | APOB 核心映射、逻辑核到物理 CCD/CCX/Core、CO 掩码、逐核/全核命令、Curve Shaper |
| SMUDebugTool | https://github.com/irusanov/SMUDebugTool | CO 掩码独立实现、Curve Shaper 参数组织与输入钳制 |
| RyzenAdj | https://github.com/FlyGoat/RyzenAdj | AMD 家族命令分派、SMU 响应分类、功耗与调校接口 |
| Linux kernel | https://github.com/torvalds/linux | RAPL MSR 地址及内核侧接口实现 |
| linux-intel-undervolt | https://github.com/mihic/linux-intel-undervolt | Intel OC Mailbox 电压偏移编码与访问路径；社区逆向资料，不是 Intel 官方规范 |
| ryzen_smu | https://gitlab.com/leogx9r/ryzen_smu | Linux 下 AMD SMU 访问、驱动接口和命令封装；需核对维护状态、代际支持与许可证 |

## 3. 进程管理、调度与约束

这是 Zentwik Scheduling Plan、场景动作和进程规则的重点候选领域。

| 项目 | 地址 | 值得调查的内容 | 状态与边界 |
|---|---|---|---|
| Process Governor | https://github.com/lowleveldesign/process-governor | Job Object、进程启动与附加、Monitor/Service 模式、CPU Affinity、CPU Rate、优先级、Efficiency Mode、子进程约束、ETW 诊断 | 待研究；其文档将 Service 模式标为 Beta。CPU 参数不能代替对 CPU Sets 的独立研究 |
| Process Lasso | https://bitsum.com/ | ProBalance、CPU Limiter、Performance Mode、Watchdog、规则持久化、进程匹配与电源方案联动 | 功能对照；闭源商业产品 |
| System Informer | https://github.com/winsiderss/systeminformer | 进程/线程/句柄/模块/服务观察、Job Object、进程树、系统信息与诊断 | 待研究；重点参考诊断和系统对象观察 |
| Process Explorer | https://learn.microsoft.com/sysinternals/downloads/process-explorer | 进程树、句柄/DLL、签名、权限和进程诊断交互 | 功能对照；闭源工具 |

研究时必须区分：
- CPU Sets 是软调度偏好，Affinity 是硬约束；两者不是同一种功能。
- Job Object 的进程生命周期和资源限制，与 CPU Sets/Affinity 的调度约束不是同一层。
- 进程启动监听、运行中进程发现、进程退出、PID 复用和子进程继承需要分别研究。
- 规则持久化、实际应用状态、外部修改、用户手动接管和退出恢复需要分别处理。
- 不因某个工具采用常驻 Monitor/Service，就推导 Zentwik 也必须采用相同进程模型。

官方基础资料：
- CPU Sets：https://learn.microsoft.com/windows/win32/procthread/cpu-sets
- Process and Thread Functions：https://learn.microsoft.com/windows/win32/procthread/process-and-thread-functions
- Job Objects：https://learn.microsoft.com/windows/win32/procthread/job-objects
- Process Security and Access Rights：https://learn.microsoft.com/windows/win32/procthread/process-security-and-access-rights

## 4. Windows 电源计划与供电状态

Zentwik 的 Power Plan 是独立领域对象。第三方性能模式不应与 Windows 电源计划混为一谈。

官方资料：
| 资料 | 地址 | 用途 |
|---|---|---|
| Managing Power Schemes | https://learn.microsoft.com/windows/win32/power/managing-power-schemes | 电源方案枚举、复制、删除、激活与管理 |
| Power Management Functions | https://learn.microsoft.com/windows/win32/power/power-management-functions | PowrProf API 族与函数语义 |
| powercfg options | https://learn.microsoft.com/windows-hardware/design/device-experiences/powercfg-command-line-options | 方案查询、设置项、AC/DC 值、导入导出与诊断 |
| Power Setting GUIDs | https://learn.microsoft.com/windows/win32/power/power-setting-guids | 电源设置 GUID 和通知定义 |

工具与产品对照：
| 项目 | 地址 | 值得调查的内容 | 状态 |
|---|---|---|---|
| Process Lasso / ParkControl | https://bitsum.com/ | 性能模式、电源方案切换、空闲/负载策略、进程规则与电源状态关系 | 功能对照；闭源 |
| G-Helper | https://github.com/seerge/g-helper | Windows 电源计划、厂商性能模式、AC/DC 状态、设备特定控制之间的协调 | 待研究 |
| QuickCPU | https://coderbag.com/product/quickcpu | 核心停放、频率缩放、电源计划与 CPU 策略的操作模型 | 功能对照；闭源 |

规范事实以 [Windows 电源方案 API](windows/power-management.md) 为准。本目录不重复维护 API 语义。

重点调查：方案/子组/设置项枚举；AC/DC 两套值的读写、回读和活动方案重新激活；复制、改名、删除、导入导出；供电状态通知；Windows 电源计划与 OEM 性能模式、固件模式、CPU 调度规则之间的边界；多项写入部分失败时的报告与恢复。

## 5. 场景自动化与厂商控制

| 项目 | 地址 | 值得调查的内容 | 状态 |
|---|---|---|---|
| Process Lasso | https://bitsum.com/ | 进程匹配规则、触发条件、优先级、持续应用、用户覆盖和恢复 | 功能对照 |
| G-Helper | https://github.com/seerge/g-helper | 电源/性能模式切换、设备事件响应、后台任务与托盘交互 | 待研究 |
| PowerToys | https://github.com/microsoft/PowerToys | 模块启停、快捷键触发、配置和后台运行组件 | 待研究 |

场景研究不能只看触发后执行了什么，还要检查冲突仲裁、重复触发、状态所有权、外部修改、用户手动接管、进程退出和恢复失败。

## 6. 硬件监控、传感器与风扇

| 项目 | 地址 | 值得调查的内容 | 状态与边界 |
|---|---|---|---|
| LibreHardwareMonitor | https://github.com/LibreHardwareMonitor/LibreHardwareMonitor | C# 硬件/传感器抽象、设备树、采样刷新、权限与兼容性 | 待研究；已有专题索引条目 |
| HWiNFO | https://www.hwinfo.com/ | 传感器覆盖范围、共享内存/外部读取接口、设备识别和传感器命名 | 功能对照；闭源，接口需核对授权与版本 |
| FanControl | https://github.com/Rem0o/FanControl.Releases | 风扇控制器抽象、曲线、混合传感器输入、控制策略和插件 | 待研究；重点限于风扇控制与传感器组合 |

研究时区分传感器读取、传感器派生、风扇控制和硬件调校写入；采样值不等于硬件原始寄存器值。

## 7. Windows 服务、后台进程与 IPC

| 项目 | 地址 | 值得调查的内容 | 状态 |
|---|---|---|---|
| StreamJsonRpc | https://github.com/microsoft/vs-streamjsonrpc | JSON-RPC 消息、请求/通知、取消、超时、流式调用与传输适配 | 待研究；先与现有 IPC 契约比较，不默认引入 |
| PowerToys | https://github.com/microsoft/PowerToys | Runner、模块接口、模块生命周期、命名管道/进程间通信 | 待研究 |
| .NET Runtime | https://github.com/dotnet/runtime | Generic Host、任务取消、管道和运行时行为底层实现 | 按具体问题定点研究 |
| Windows Classic Samples | https://github.com/microsoft/Windows-classic-samples | Windows 原生 API、服务、进程、权限、通知等示例 | 待研究；核对 Windows 版本和 API 适用范围 |

重点调查：服务启动/停止、崩溃恢复、调用方身份校验、命名管道 ACL、请求取消、超时、背压、协议版本、服务与 UI/CLI 生命周期解耦。

## 8. 插件与模块扩展

| 项目/资料 | 地址 | 值得调查的内容 | 状态与边界 |
|---|---|---|---|
| PowerToys | https://github.com/microsoft/PowerToys | Runner 与模块接口、模块注册、启停、配置、外部应用启动和 IPC | 待研究 |
| AssemblyLoadContext | https://learn.microsoft.com/dotnet/core/dependency-loading/understanding-assemblyloadcontext | 同进程程序集加载、依赖隔离与卸载边界 | 官方资料；不是安全沙箱 |
| .NET Plugin tutorial | https://learn.microsoft.com/dotnet/core/tutorials/creating-app-with-plugin-support | 插件契约、依赖加载和版本兼容示例 | 官方资料 |

插件的权限隔离、进程隔离、协议版本和信任模型必须由项目自身决策。加载程序集不等于安全隔离；不能因使用 .NET 插件教程就默认第三方插件在主进程内运行。

## 9. UI、MVVM 与桌面工程

本节仅供 UI 工作需要时参考；当前由其他工作流负责 UI 时，底层任务不应借此改动 UI。

| 项目 | 地址 | 值得调查的内容 | 状态与边界 |
|---|---|---|---|
| CommunityToolkit/dotnet | https://github.com/CommunityToolkit/dotnet | CommunityToolkit.Mvvm、ObservableProperty、RelayCommand、源生成器 | 待研究；评估是否需要依赖 |
| WPF-UI | https://github.com/lepoco/wpfui | WPF 控件、主题、导航和窗口交互 | 待研究；仅作控件参考，不默认替换自定义 UI |
| Microsoft WPF | https://github.com/dotnet/wpf | WPF 控件、布局、绑定、渲染与框架行为 | 官方实现源码；遇到具体框架问题时定点阅读 |

不因参考 UI 项目而更改已确定的产品视觉方向、强调色自定义能力或页面结构。

## 10. CLI 与配置

| 项目 | 地址 | 值得调查的内容 | 状态 |
|---|---|---|---|
| System.CommandLine | https://github.com/dotnet/command-line-api | 命令树、参数解析、验证、补全、帮助生成 | 待研究 |
| Spectre.Console | https://github.com/spectreconsole/spectre.console | 终端表格、进度、提示、交互式 CLI 输出 | 待研究；展示层工具，不是领域模型 |

仅当现有 CLI 的命令定义、帮助、验证或输出确有痛点时评估依赖，不为统一而重写。

## 11. 安装、升级与维护

| 项目 | 地址 | 值得调查的内容 | 状态 |
|---|---|---|---|
| WiX Toolset | https://github.com/wixtoolset/wix | Windows Installer/MSI、组件、升级、修复、服务安装与卸载 | 待研究 |
| Velopack | https://github.com/velopack/velopack | 桌面应用打包、差分更新、版本切换、回滚与安装生命周期 | 待研究；核对服务/驱动部署需求 |
| Inno Setup | https://github.com/jrsoftware/issrc | 安装脚本、升级/卸载、自定义安装流程 | 待研究；与 MSI 方案比较，不预设选型 |

安装器选择必须结合服务权限、驱动依赖、升级中断恢复、数据迁移、卸载清理和维护入口设计，不只比较安装包大小。

## 12. 研究优先级建议

以下是研究建议，不是开发路线图：

1. **Process Governor**：优先核对 Job Object、进程监视、规则更新、子进程和服务模式；明确其 CPU Affinity 与 CPU Sets 的差别。
2. **Process Lasso**：从官方文档和行为层面对照进程规则、ProBalance、Watchdog、电源方案联动与用户覆盖。
3. **G-Helper**：核对电源计划、AC/DC、OEM 性能模式和后台生命周期的协调方式。
4. **System Informer**：研究 Windows 进程/线程/Job/服务对象的观察和诊断。
5. **PowerToys**：研究模块 Runner、生命周期和 IPC；不要直接移植整体架构。
6. **StreamJsonRpc**：与 Zentwik 现有 IPC 契约比较后，再决定是否值得采用。
7. **LibreHardwareMonitor / HWiNFO**：按传感器缺口逐项研究，不把监控库升级成硬件控制依赖。
8. **WiX / Velopack / Inno Setup**：进入安装维护开发前进行方案比较。
9. **UI / CLI 候选**：仅在对应工作实际需要时研究。

## 13. 后续 AI/开发者执行要求

1. 先读 AI_GUIDE.md、knowledge/INDEX.md 和对应领域规范。
2. 查阅本目录，优先使用用户明确指定的参考项目。
3. 实际打开相关源码文件和函数，不以 README、搜索摘要或其他 AI 总结代替源码研究。
4. 记录仓库分支/提交、文件路径、类型/函数和访问日期；重要结论应可复查。
5. 对比 Zentwik 当前实现与旧版事实，明确哪些问题已解决、哪些仍存在。
6. 区分上游源码事实、官方平台事实、推断、项目约定和未验证事项。
7. 检查许可证、维护状态、支持平台、依赖与安全边界。
8. 只提出与当前任务相关的借鉴点；不为了“参考完整”引入依赖、进程、服务或架构。
9. 复杂/高风险功能先给技术评估；不要在未经确认时擅自重构。
10. 研究完成后更新本目录状态，并把长期稳定的技术结论写入对应专题规范；不要重复维护技术定义。

## 14. 维护记录

- 2026-11-26：建立跨领域参考项目目录。既有已研究项目沿用 hardware/cpu/implementation-references.md 的研究状态；新增候选项目统一标为待研究或功能对照，不将候选误记为已完成研究。
