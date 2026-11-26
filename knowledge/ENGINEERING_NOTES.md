# 工程约束与上游勘误补充（独立核验）

- 文档类型：跨领域工程避坑清单（不是参数权威定义，不替代各领域基础规范）
- 领域：`knowledge`（跨 windows / hardware/cpu / architecture / decisions）
- 核验状态：混合，逐条标注（见各节与 §14 汇总表）
- 建立日期：2026-11-26（依据《Gemini 复审资料库 1》提示，逐项独立打开一手资料核验；不照抄报告结论）
- 权威来源：Microsoft Learn、GitHub Advisory、dotnet/runtime 源码与 issue、Linux 内核源码、namazso/PawnIO 与 PawnIO.Modules 上游源码、irusanov/ZenStates-Core、georgewhewell/undervolt、remittor/pyhwinfo（均 2026-11-26 逐字访问，见各节链接）
- 相关文档：[AUDIT.md](AUDIT.md)、[INDEX.md](INDEX.md)、[windows/security-service.md](windows/security-service.md)、[hardware/cpu/amd-smu.md](hardware/cpu/amd-smu.md)、[hardware/cpu/intel-oc-mailbox.md](hardware/cpu/intel-oc-mailbox.md)

> **使用规则**：本文件只记录"对后续开发有实际价值的工程约束与上游勘误"。参数与协议权威定义仍在各基础规范（见 [AUDIT.md](AUDIT.md) §3 单一维护点），本文用交叉引用而非复制。凡标 Inferred / Unverified 者不得在实现中当作确定事实；凡本文与厂商官方一手资料冲突，以官方为准并记入 [AUDIT.md](AUDIT.md) §10。

---

## 1. Windows 命名管道安全：`CurrentUserOnly` 的边界与已知 Bug

核验状态：**API 语义 Verified（官方文档）；issue 存在与内容 Verified（一手）；Zentwik 当前设计评价 = 工程判断（Proposed）**。

### 1.1 官方语义

- `PipeOptions.CurrentUserOnly`（[MS Learn PipeOptions Enum](https://learn.microsoft.com/en-us/dotnet/api/system.io.pipes.pipeoptions?view=net-10.0)，2026-11-26 逐字）原文：*"When used to create a NamedPipeServerStream instance, indicates that the pipe can only be connected to a client created by the same user... On Windows, it verifies both the user account and elevation level."*
- Mandatory Integrity Control（[MS Learn Mandatory Integrity Control](https://learn.microsoft.com/en-us/windows/win32/secauthz/mandatory-integrity-control)，2026-11-26 逐字）原文：*"a principal with a low integrity level cannot write to an object with a medium integrity level, even if that object's DACL allows write access"*；*"By default, the system creates every object with an access mask of SYSTEM_MANDATORY_LABEL_NO_WRITE_UP"*；*"Standard users receive medium, elevated users receive high... system services receive system integrity"*。

### 1.2 MIC 的单向性（关键）

MIC 默认是 **NO_WRITE_UP（禁止向上写）**：低完整性主体不能写高完整性对象。它**不禁止**高完整性客户端连接/写中完整性服务端——即"高 → 中"方向不受 MIC 阻挡。

- **官方事实（Verified）**：NO_WRITE_UP 只阻止低→高；官方原文未逐字陈述高→中可连通，这是 NO_WRITE_UP 语义的直接推论。
- **推论（Inferred，不可标 Verified）**：因此仅靠 MIC/`CurrentUserOnly` 的提权校验，**不能**替代服务端对调用方 `TokenIntegrityLevel` 的显式校验；一个被提权（High）的本地进程仍可连到中完整性服务端。Zentwik 现有 [security-service.md](windows/security-service.md) §3/§4 已在服务端显式读取 `TokenIntegrityLevel` 并要求 ≥ Medium，方向正确；建议把"必须校验上限、不只校验下限"的工程理由写进实现注释（Proposed，仅文档建议，不碰产品代码）。

### 1.3 dotnet/runtime#123903（一手核实）

- Issue：[github.com/dotnet/runtime/issues/123903](https://github.com/dotnet/runtime/issues/123903)，标题 *"PipeOptions.CurrentUserOnly incorrectly uses WindowsIdentity.GetCurrent().Owner to get the current user SID"*。
- **状态（Verified，GitHub API 2026-11-26）**：`state=open`、`created_at=2026-02-02`、标签 `bug` / `area-System.IO`、milestone `12.0.0`；修复 PR [#136870](https://github.com/dotnet/runtime/pulls/136870)（标题 *"Use current user SID for PipeOptions.CurrentUserOnly (#123903)"*）状态 `open`、`merged=false`。**截至核验日未修复、未合并**，不得写成"已修复"。
- Issue 原文关键句（逐字）：*"The Windows implementation of PipeOptions.CurrentUserOnly incorrectly uses WindowsIdentity.GetCurrent().Owner to get the current user SID. It should be using WindowsIdentity.GetCurrent().User..."*；复现场景：*"two processes running as SYSTEM. The client process was created from an admin user process, so its owner was Administrators, and the server process was a Windows Service (created by the SCM), so its owner was SYSTEM. Both processes had a user SID of SYSTEM... but couldn't connect because they had different owner SIDs"*。
- **适用范围（Verified 原文）**：*"this has been implemented incorrectly since PipeOptions.CurrentUserOnly was introduced in .NET Core in 2018"*；报告者测试环境 `.NET 8 / Windows 11 x64`。即 2018 起所有含该选项的 .NET 版本都受影响，Zentwik 早期 M1 的 `CurrentUserOnly` 路径同属受影响面。
- **对本项目的意义（工程判断）**：该 bug 与"LocalSystem 服务 + 管理员创建进程"这一 Zentwik 目标形态直接相关——owner SID 差异会导致 `CurrentUserOnly` 误拒绝。这从上游侧印证了从 `CurrentUserOnly` 迁移到**显式 `PipeSecurity` + `NamedPipeServerStreamAcl.Create` + 调用方校验**的必要性（ADR-004 / ADR-003 已实施，决策正确）。注意：该 bug 的表现是"同 user SID 但不同 owner SID 被拒"，与 §1.2 的"高完整性客户端可连中完整性服务端"是**两个不同问题**，不得混为一谈。

## 2. PawnIO：真实设备路径、IOCTL、SDDL、RyzenSMU.p 与全局互斥

核验状态：**全部 Verified（上游源码逐字，2026-11-26）**；对 Zentwik 的影响为工程判断。

### 2.1 设备创建与 SDDL（纠正 Gemini 报告的来源说法）

- 驱动 `namazso/PawnIO` `PawnIO/src/driver.cpp` 用的是 **`IoCreateDevice`**（源码：`auto status = IoCreateDevice(driver_object, sizeof(device_extension), nullptr, k_device_type, FILE_DEVICE_SECURE_OPEN, FALSE, &device_object);`），**不是** `IoCreateDeviceSecure`。Gemini 报告称"从 namazso/PawnIO.Setup#7 确认其调用了 IoCreateDeviceSecure"与当前主源码不符，**不采纳**。
- 精确 SDDL 在 INF：`PawnIO/PawnIO.inf.in` 的 `[Custom_Security]` 段：`HKR,,Security,,"D:P(A;;GA;;;SY)(A;;GA;;;BA)"`——即仅 SYSTEM（SY）与 Builtin Administrators（BA）获得完全控制（GA），DACL 受保护（P）。这解释了"非提权进程 CreateFile 被拒、SYSTEM 服务可打开"的实测现象（后者为报告称，来源 Unknown）。
- 设备类型与路径：`pawnio_um.h` `k_device_type = 41394`（0xA1B2）；`k_device_path = L"\\Device\\" PAWNIO_NAME`，DOS 路径标记 `DEPRECATED`；用户态经 `\\?\GLOBALROOT\Device\PawnIO` 访问（本仓库 `research/02` §2.1 口径）。

### 2.2 IOCTL 构造与"0x841"的真实含义（勘误）

`PawnIO/include/pawnio_um.h` 逐字：

```c
IOCTL_PIO_LOAD_BINARY = CTL_CODE(k_device_type, 0x821, METHOD_BUFFERED, FILE_ANY_ACCESS),
IOCTL_PIO_EXECUTE_FN  = CTL_CODE(k_device_type, 0x841, METHOD_BUFFERED, FILE_ANY_ACCESS),
IOCTL_PIO_VERSION     = CTL_CODE(k_device_type, 0x861, METHOD_BUFFERED, FILE_ANY_ACCESS)
```

- **关键区分**：本仓库 `research/02` §2.1 与 `amd-smu.md` §2 中写的 IOCTL `0x821` / `0x841` 是 **CTL_CODE 的 Function Code**，不是完整 IOCTL 值；完整值须按 `CTL_CODE(0xA1B2, func, METHOD_BUFFERED, FILE_ANY_ACCESS)` 计算。Gemini 报告提到的 `0xA1B22084`（issue #1 讨论）是 `0x821` 的完整值形式，二者不矛盾但口径不同；**文档应注明是 function code**（Proposed：由本文记录，不改 research 原文以保持历史）。
- `driver.cpp` 的 `IOCTL_PIO_EXECUTE_FN` 分支确认请求/响应体：驱动对 `SystemBuffer` 直接 `vm_execute_function(...)`，`out_length > in_length` 时补零；上层 `DEFINE_IOCTL_SIZED` 宏决定参数个数（见下）。32 字节 ASCII 函数名 + N×8 参数格式来自模块侧 ABI（本仓库口径），驱动层只透传缓冲区（Verified 源码：`vm_execute_function(FsContext, SystemBuffer, in_length, SystemBuffer, out_length)`）。

### 2.3 RyzenSMU.p：关键常量、状态码与函数路径

来源 `namazso/PawnIO.Modules` `RyzenSMU.p`（LGPL-2.1-or-later；2026-11-26 逐字）：

- `const SMU_RETRIES_MAX = 8096;`（Verified，与 `research/02` §2.3/§11 的"8096 次"一致）。
- **0x01 的真实语义（勘误）**：`const SMUStatus: { SMU_Busy = 0x00, SMU_OK = 0x01, ... SMU_UnknownCmd = 0xFE, SMU_Failed = 0xFF }`。即 **`0x01` 是响应状态码 `SMU_OK`，不是命令号**。
  - 全局命令表中 `TestMessage = 0x01`（`Arg0:X, Res0:(X+1)`）来自 `ryzen_smu docs/rsmu_commands.md`，与 SMUStatus.SMU_OK=0x01 是两个不同语境的 0x01。
  - 本仓库 `research/02` §11 记录的"SMU 测试消息 `0x01` 回 `2`"指应用层 `TestMessage(0x01)` 命令；`ioctl_get_smu_version` 源码实际发的是 `send_command(0x02, args)`（`args[0]=1`；Carrizo 家族改读固件 header）。Gemini 报告"0x01 仅作为 SMUStatus.SMU_OK 状态码；文档中的 0x01 探针应属 Zentwik 自身应用层的 TestMessage 调用"——**前半句 Verified，后半句属推断**：源码 `RyzenSMU.p` 未导出名为 `TestMessage` 的专用函数，但 `ioctl_send_smu_command(0x01,...)` 可发该全局命令，故应用层经通用命令函数发 0x01 是合理口径（Inferred）。
- `ioctl_send_smu_command` 源码：`DEFINE_IOCTL_SIZED(ioctl_send_smu_command, 7, 6)`，校验 `(in[0] & ~0xFF) != 0 → STATUS_NOT_SUPPORTED`（只接受 8 位命令），取 `args[0..5]=in[1..6]`，调用 `send_command(in[0], args)`。
- **`send_command` 与邮箱选择（重点）**：`send_command` 使用 `k_addrinfo[k_addridx[g_code_name]]` 的 `.cmd/.rsp/.args`，索引 0–4 分别为 `0x3B10524/0x3B10570/0x3B10A40`、`0x3B1051C/0x3B10568/0x3B10590`、`0x3B10A20/0x3B10A80/0x3B10A88`、`0x3B10924/0x3B10970/0x3B10A40`、`0xFFF00724/0xFFF00764/0xFFF007A4`（MMIO）。这些是 **RSMU 风格 cmd/rsp/args 邮箱寄存器组**；**模块未导出独立的 MP1 邮箱发送函数**（未找到 `ioctl_send_mp1_smu_command`）。因此 Gemini 报告"`ioctl_send_smu_command` 硬编码仅发 RSMU 邮箱"的**结论方向成立（Verified 源码事实）**，但"来自 ZenStates-Core Issue #20 的明确警告"这一出处不准确：Issue #20（标题 "Replacement for WinRing0"，open）讨论的是 PawnIO 替代 WinRing0 的能力缺口，维护者评论提到需扩展模块支持不同邮箱，并未逐字写出"硬编码仅 RSMU"的警告。**以源码为准，出处更正。**
- MP1 路径：`ioctl_read_smu_register` / `ioctl_write_smu_register` 经 `check_smu_register_range` 校验后由 `read_reg`/`write_reg` 访问——普通地址走 PCI 配置 `0xC4/0xC8`（Carrizo 为 `0xB8/0xBC`），`SMU_MMIO_SENTINEL`(0xFFF00000) 页走 MMIO。若未来支持移动 APU 的 MP1 命令（如 `0x4B/0x4C`、`0x54/0x55`），**不能**简单复用 `ioctl_send_smu_command`，须按目标寄存器/邮箱选用 register read/write 路径并自行处理仲裁（工程约束，Proposed；是否足够覆盖 MP1 消息邮箱仍需实机验证，Unverified）。
- **全局 PCI 互斥**：`RyzenSMU.p` 在 `read_reg`/`write_reg` 等函数处有 6 处相同注释：*"/// @warning You should acquire the \"\\BaseNamedObjects\\Access_PCI\" mutant before calling this"*（Verified 源码注释）。即真实对象名是 NT 命名空间 `\\BaseNamedObjects\\Access_PCI`（Win32 侧对应 `Global\Access_PCI`）；**未在该源码中找到 `WR0_WaitPciBus`**（Gemini 报告提及，标 Unverified，不得当作已核实事实）。调用方若不与 HWiNFO/ZenTimings 等工具共享该互斥，并发访问 PCI `[0xC4,0xC8]` 存在竞态风险（Inferred，机制合理但无跨工具实测）。

### 2.4 对 Zentwik 的实际约束（Proposed，仅文档）

1. 设备句柄单一特权进程持有（ADR-003 已定），与 INF SDDL 的 SYSTEM/BA-only 模型一致。
2. 写入命令前：确认命令号是 8 位（模块侧硬约束）；FMax 用 `0x6E` 读 / `0x70` 全核写（见 [amd-smu.md](hardware/cpu/amd-smu.md) §3）；移动 APU MP1 命令不得走 `ioctl_send_smu_command`。
3. 协议线格式（32B 函数名 + N×8 参数）与 IOCTL function code 在文档中分开表述；完整 IOCTL 值由 `CTL_CODE` 计算。
4. 不修改产品代码；本节供 M4+ 硬件通道设计与评审引用。

## 3. Intel OC Mailbox 全局互斥锁生态

核验状态：**pyhwinfo 用法 Verified（源码）；"所有工具都遵守" = Unverified/否定**。

- `remittor/pyhwinfo` `msrbox.py`（2026-11-26 逐字）定义：
  ```python
  LOCAL_OC_MAILBOX_MUTEX_NAME  = r"Local\Access_Intel_OC_Mailbox"
  GLOBAL_OC_MAILBOX_MUTEX_NAME = r"Global\Access_Intel_OC_Mailbox"
  ```
  `init_mutex()` 实际 `CreateMutexW(GLOBAL_OC_MAILBOX_MUTEX_NAME)`；`check_mailbox_mutex()` 先尝试 `OpenMutexW` 检测已存在者。
- **已确认使用者**：仅 `remittor/pyhwinfo`（本轮逐字核对）。Gemini 报告称 Core Temp 行为报告（VirusTotal）也证实存在，本轮未打开该报告，维持 Unverified。
- **不得声称**：ThrottleStop / LibreHardwareMonitor / OpenHardwareMonitor 都使用同一互斥锁——本轮未找到这些项目的一手源码证据，**明确标 Unverified**。这是工具间的**生态约定**，不是 Windows 或 Intel 的官方保证（Verified 该"缺乏官方保证"事实：OC Mailbox 无公开架构文档，见 [intel-oc-mailbox.md](hardware/cpu/intel-oc-mailbox.md) §1）。
- 工程含义：Zentwik 若使用 MSR 0x150，应创建/获取同名全局互斥以避免半写状态（沿用 [intel-oc-mailbox.md](hardware/cpu/intel-oc-mailbox.md) §3.4 现有约定），但不能假设其他工具也会配合。

## 4. Windows 服务恢复：`SERVICE_FAILURE_ACTIONS_FLAG`

核验状态：**Verified（Microsoft Learn 原文）**。

- 来源：[SERVICE_FAILURE_ACTIONS_FLAG (winsvc.h)](https://learn.microsoft.com/en-us/windows/win32/api/winsvc/ns-winsvc-service_failure_actions_flag)（2026-11-26 逐字），字段 `fFailureActionsOnNonCrashFailures`：
  - **FALSE（默认）**原文：*"If this member is FALSE and the service has configured failure actions, the failure actions are queued only if the service terminates without reporting a status of SERVICE_STOPPED."*（只有崩溃、未上报 SERVICE_STOPPED 才触发恢复）
  - **TRUE** 原文：*"...the failure actions are queued if the service process terminates without reporting a status of SERVICE_STOPPED or if it enters the SERVICE_STOPPED state but the dwWin32ExitCode member of the SERVICE_STATUS structure is not ERROR_SUCCESS (0)."*（非崩溃但以非零退出码进入 SERVICE_STOPPED 也触发）
  - **生效条件（关键，纠正 Gemini 报告措辞）**：Remarks 原文 *"The change takes effect the next time the system is started."* —— 是**下次系统启动**生效，不是"重启服务"。Gemini 报告"需重启服务才能生效"不准确。
  - 另注：*"This setting is ignored unless the service has configured failure actions."* 配置入口为 `ChangeServiceConfig2`。
- **对 Zentwik 的影响（Proposed）**：[security-service.md](windows/security-service.md) §2 / ADR-003 实施记录依赖"服务异常终止上报 1066（ERROR_SERVICE_SPECIFIC_ERROR，非 0）→ SCM 执行恢复"。按官方语义，**该路径只在 `fFailureActionsOnNonCrashFailures=TRUE` 时成立**，且此标志更改需重启系统生效。当前文档未记录该标志的取值与生效延误，属**依赖前提缺失**：应在服务安装逻辑审阅时显式设置 TRUE 并在部署说明中写明"首次安装/改此标志后需重启系统才生效"；本轮不修改产品代码，仅记录约束与待办（见 §12 待核验）。

## 5. Intel RAPL / TPMI：平台差异（不是全面替代）

核验状态：**Verified（Linux 内核源码）；对 Zentwik 影响 = 工程判断**。

- 来源：`torvalds/linux` `drivers/powercap/intel_rapl_tpmi.c`（2026-11-26 逐字），头部 `Copyright (c) 2023, Intel Corporation`，驱动名 `intel_rapl_tpmi: Intel RAPL driver via TPMI interface`，`MODULE_DESCRIPTION("Intel RAPL TPMI Driver")`。
- 接口机制：`#include <linux/intel_vsec.h>`、`#include <linux/auxiliary_bus.h>`，设备 ID `.name = "intel_vsec.tpmi-rapl"`，`devm_ioremap_resource` + `readq`/`writeq` 访问（MMIO，PCIe VSEC），与 `intel_rapl_msr.c` 的 MSR 路径是**并存的独立驱动**（本驱动设 `trp->priv.type = RAPL_IF_TPMI`）。
- **单位布局不同（对换算有直接影响）**：TPMI Unit 寄存器 `TPMI_POWER_UNIT_OFFSET 0x00 / MASK GENMASK(3,0)`、`TPMI_ENERGY_UNIT_OFFSET 0x06 / MASK GENMASK_ULL(10,6)`、`TPMI_TIME_UNIT_OFFSET 0x0C / MASK GENMASK_ULL(15,12)`；能量单位 `TPMI_ENERGY_UNIT_SCALE=1000`（pJ 量级）、功率 `MICROWATT_PER_WATT >> value`、时间 `USEC_PER_SEC >> value`。这与传统 MSR 0x606 的 `power bits[3:0]/energy bits[12:8]/time bits[19:16]` **位置不同**，不得把 MSR 单位解析函数套用到 TPMI。
- 平台覆盖：本源码文件没有逐字写"Meteor Lake / Sapphire Rapids"型号名单（该名单在内核其他文件/文档中）。Gemini 报告"Meteor Lake+ 客户端与 Sapphire/Granite Rapids 服务器走 TPMI"的具体型号映射在本轮**未取得逐字一手证据，标 Inferred/待核**；可确认的是 TPMI 路径存在且由 `intel_vsec` 枚举、与 MSR 路径并存（Verified）。
- **对 Zentwik 的实际影响（Proposed）**：当前 Intel 监控实现基于 PawnIO `IntelMSR.bin` 白名单（MSR 0x606/0x610/0x611）。在启用 TPMI 的新平台上，传统 RAPL MSR 可能不提供全部域/功率数据，须把"通道类型（MSR/TPMI）"纳入能力探测与单位解析分支；**不能**把 TPMI 描述成所有 RAPL MSR 的全面替代，也不能假定现有 1/2^n 单位公式对所有平台通用。相关单位权威定义仍在 [units.md](data/units.md) §1。

## 6. WPF 渲染与节能

核验状态：官方建议 Verified；节能影响 = 官方措辞 + 推导，分开标注。

- 官方原文（[Optimizing Performance: Other Recommendations](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/advanced/optimizing-performance-other-recommendations)，"CompositionTarget.Rendering Event" 段，2026-11-26 逐字）：*"The CompositionTarget.Rendering event causes WPF to continuously animate. If you use this event, detach it at every opportunity."*
  - **Verified**：该事件会导致 WPF 持续动画，官方建议"一有机会就取消订阅"。
  - **Inferred（推导，非官方逐字）**：持续注册等于持续重绘，因此窗口最小化、页面不活跃或无需刷新时应注销以省电。官方**未**在该页逐字写"power saving / minimized"，不得标 Verified。
- `RenderingEventArgs.RenderingTime` 用于帧时间差计算：官方 API 存在（[RenderingEventArgs](https://learn.microsoft.com/en-us/dotnet/api/system.windows.media.renderingeventargs)），Zentwik 现用 Stopwatch 门限降到 30Hz（[research/03](../research/03-ui-ipc-deployment.md) §2.1），二者均可；属实现选择（Proposed）。
- D3D9On12 映射层：Gemini 报告引 dotnet/wpf#7062 称新款 GPU 上 WPF 经 D3D9On12 运行。本轮**未打开该 issue 逐字核对，标 Unverified**（待核：dotnet/wpf#7062 原文与适用 GPU 范围）。现有 ADR-002"渲染停留在 DirectX 9 代"结论不变（WPF 公开架构事实），但不写成"永不经过 DX12 映射"。
- **对 Zentwik 监控 UI 的实现建议（Proposed，不碰产品代码）**：仅在监控页可见且服务有数据时订阅 `CompositionTarget.Rendering`；`Window.StateChanged` 到 Minimized、`Unloaded`、页面切走时 `-=`；用 `RenderingTime` 或单一 QPC 时钟做 30Hz 门限；避免多页各自订阅造成多套渲染循环。

## 7. WPF DataGrid 虚拟化默认值

核验状态：**Verified（Microsoft Learn API 文档，2026-11-26 逐字）**。

| 属性 | 注册默认值 | 官方原文 | 含义 |
|---|---|---|---|
| `EnableRowVirtualization` | **`true`** | *"The registered default is true."* / *"To improve performance... set to true by default"* | DataGrid 只为可见项创建 `DataGridRow`，滚动出视口后回收；行虚拟化默认已开启 |
| `EnableColumnVirtualization` | **`false`** | *"The registered default is false."* | 列虚拟化默认**关闭**，所有绑定列都会生成单元格 |

来源：[EnableRowVirtualization](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.datagrid.enablerowvirtualization?view=windowsdesktop-11.0)、[EnableColumnVirtualization](https://learn.microsoft.com/en-us/dotnet/api/system.windows.controls.datagrid.enablecolumnvirtualization?view=windowsdesktop-11.0)。

- **对 Zentwik 的实际影响（Proposed）**：[research/03](../research/03-ui-ipc-deployment.md) §2.4"进程 ≤2000 行用 DataGrid 行虚拟化即可"的前提正确（行默认开），但**高密度多列监控表必须显式设置 `EnableColumnVirtualization=true`**，否则列数增长时单元格全量实例化。该默认值此前未在任何文档写明，属工程遗漏，本条补齐。

## 8. NativeAOT 下 JSON 枚举转换器

核验状态：**Verified（.NET 官方 source-generation 文档，2026-11-26 逐字）**。

- 官方原文（[Source generation - System.Text.Json](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/source-generation?pivots=dotnet-8-0)）：*"To serialize enum names as strings using source generation, use the `JsonStringEnumConverter<TEnum>` converter. (The non-generic `JsonStringEnumConverter` type is not supported by the Native AOT runtime.)"*
- **准确表述（不得泛化）**：
  - 非泛型 `JsonStringEnumConverter` 在 **Native AOT runtime** 下不支持（Verified 官方逐字）。这**不等于**"所有情况下非泛型都不可用"——在常规 JIT 序列化中它仍可用。
  - 泛型 `JsonStringEnumConverter<TEnum>` 是 source generation / AOT 下把枚举序列化为字符串的方式（Verified）；另有 blanket policy `[JsonSourceGenerationOptions(UseStringEnumConverter = true)]` 可对上下文内所有枚举生效（Verified 官方示例）。
  - 反射式序列化默认在 Native AOT 下受限；可用 `JsonSerializerIsReflectionEnabledByDefault=false` 显式关闭（Verified）。
- **适用范围说明**：Zentwik 服务端计划 NativeAOT（[ADR-001](../decisions/ADR-001-language-runtime.md)），IPC 控制面用 JSON 源生成（[research/03](../research/03-ui-ipc-deployment.md) §4.3）。若 IPC 信封中任何字段以字符串枚举表达，须用 `JsonStringEnumConverter<TEnum>` 或 blanket policy，不能用非泛型版本；**但** WPF 客户端与当前 M1–M3 服务端尚未启用 NativeAOT，非泛型版本在这些 JIT 场景并不违规（避免把风险写成"已存在的错误"）。

## 9. Avalonia / Tmds.DBus.Protocol 漏洞（勘误结论索引）

详见 [ADR-002](../decisions/ADR-002-desktop-ui.md) 否决项勘误与 [research/04](../research/04-tech-stack-evaluation.md) §4.3；一手证据（GHSA 公告、Avalonia 12.0.0/12.0.1 Release Notes、Datasync#499/PR#508）均于 2026-11-26 逐字核对。摘要：`0.21.2` 属 Avalonia **11.3.x**；Avalonia 12.0.0 搭载 **0.90.3**（同样受影响），12.0.1 升 **0.92.0** 修复；GHSA-xrw6-gwf8-vvr9 / CVE-2026-39959 High/CVSS 7.1；NuGet 审计按 TFM 依赖图而非运行时 OS。本文件不重复证据表。

## 10. MSIX 服务/驱动支持（勘误结论索引）

详见 [research/03](../research/03-ui-ipc-deployment.md) §5 MSIX 判定勘误；Microsoft 官方 "Converting an installer with services" 与 "MSIX Packaging Tool Known Issues" 于 2026-11-26 逐字核对。摘要：Win10 2004 起有限支持服务；官方原文 *"MSIX currently does not support driver installation"*；Zentwik 因 PawnIO 驱动排除 MSIX 的结论成立，但理由必须是驱动而非服务。本文件不重复证据表。

## 11. Intel 电压平面跨代差异（勘误结论索引）

详见 [intel-oc-mailbox.md](hardware/cpu/intel-oc-mailbox.md) §2.1；undervolt `PLANES` 与 pyhwinfo `MAILBOX_OC_DOMAIN_ID_*` 于 2026-11-26 逐字核对。摘要：Haswell–10th Gen Plane 3=uncore/4=analogio；12th Gen+ Plane 3=Reserved/4=System Agent/5=L2 Atom/6=Memory Controller；11th Gen 未覆盖 = Unknown。本文件不重复证据表。

## 12. 待核验与证据缺口（第三优先级：不强行下结论）

| # | 命题 | 已确认事实 | 现有证据 | 缺失证据 | 提升等级所需材料 |
|---|---|---|---|---|---|
| 1 | Zen2/3 CO 命令 `0x0A`/`0x7C` | 本仓库历史表如此记录 | `research/02` §2.2、[amd-smu.md](hardware/cpu/amd-smu.md) §3 | 本轮未在 ZenStates-Core 对应平台文件逐字检出 | 上游 Zen2/3 平台源码逐行 + 真机回读矩阵；保持 Inferred |
| 2 | MP1 侧 CO `0x35`/`0x36` | Zen4/5 MP1 逐核/全核 = `0x35`/`0x36`（Verified `Zen4Settings.cs`） | ZenStates 源码 | Zen2/3 的 MP1 侧对应命令未检出 | 对应平台源码；不得把 Zen4/5 值外推 |
| 3 | SMUDebugTool 界面钳位 ±999 / Curve Shaper `[-50,+30]` / AllCcdZero 归零 | 源码事实已在 [AUDIT.md](AUDIT.md) §6.2/§9 逐字核对 | `SettingsForm.cs:808-809,507,674-680` | 本轮未重新打开原文（沿用历史核验，非本轮 Verified） | 若需本轮 Verified 须重开源码定位；维持原等级 |
| 4 | 9955HX APOB 维度与全 16 核可写 | 本机 CPUID 核号 + L3 域拓扑为单机观测（报告称/User-provided） | [amd-smu.md](hardware/cpu/amd-smu.md) §3.1、[A1 交接](../research/handoffs/A1-2026-11-26.md) | 无原始 APOB dump、无写入成功日志、无工具名/版本 | 原始 APOB 表 + 16 核逐核写/读/效果日志；保持 Unverified，**Gemini 的"认可"不是实机验证** |
| 5 | PawnIO 驱动 SDDL | INF 中 `D:P(A;;GA;;;SY)(A;;GA;;;BA)`（Verified 本轮） | `PawnIO.inf.in [Custom_Security]` | 无（本项已闭环） | 已 Verified；若需强证据可再查驱动加载后 `sdshow` 输出 |
| 6 | `WR0_WaitPciBus` 互斥 | 未在 RyzenSMU.p 找到 | 本轮源码搜索 | 无任何一手来源 | 找到定义它的上游项目源码；保持 Unverified，改用已 Verified 的 `\\BaseNamedObjects\\Access_PCI` |
| 7 | Intel 互斥锁生态覆盖（ThrottleStop 等） | 仅 pyhwinfo 确认 | `msrbox.py` | 其他工具源码/行为报告 | 逐项目源码；保持 Unverified，不泛化 |
| 8 | TPMI 具体平台名单（Meteor Lake / Sapphire Rapids 等） | TPMI 驱动存在且与 MSR 并存（Verified） | `intel_rapl_tpmi.c` | 型号名单未在该源码逐字出现 | Intel 官方平台文档或内核 Kconfig/绑定表；保持 Inferred |
| 9 | D3D9On12 映射层适用 GPU 范围 | 未核实 | Gemini 报告引 dotnet/wpf#7062（未打开） | issue 原文 | 打开 issue 逐字核对；保持 Unverified |
| 10 | 无原始日志的历史实测（PDH 4000/s、std 21–35pp、15.6ms） | 机制为官方事实；具体数字报告称 | [AUDIT.md](AUDIT.md) §6.1 | 原始日志 | 实机重测并保存原始输出；保持 Unknown |
| 11 | `0x610` PL1/PL2 移除理由 | 提交正文为空 | `research/02` §3.3 | 提交作者说明/旧 issue | 补作决策记录；保持 Unverified |

## 13. 常见误解（汇总）

| 误解 | 纠正 | 依据 |
|---|---|---|
| "`CurrentUserOnly` 已校验提权，不必再查完整性" | MIC 只挡低→高；高完整性客户端仍可连中完整性服务端，须显式校验 `TokenIntegrityLevel` 上限 | §1.2（NO_WRITE_UP 官方语义 + 推导） |
| "dotnet/runtime#123903 已修复" | 截至 2026-11-26 issue open、PR 未合并，影响 .NET Core 2018 起 | §1.3（GitHub API/原文） |
| "PawnIO 用 IoCreateDeviceSecure" | 主源码用 `IoCreateDevice`；SDDL 在 INF 注册表项 | §2.1（源码） |
| "IOCTL 0x841 是完整值" | `0x841` 是 function code，完整值由 `CTL_CODE(0xA1B2,0x841,...)` 计算 | §2.2（pawnio_um.h） |
| "`0x01` 是就绪探针命令" | RyzenSMU.p 中 `0x01` 是响应状态 `SMU_OK`；作为命令是全局 `TestMessage`，须看语境 | §2.3（源码 + ryzen_smu 命令表） |
| "所有工具都用 `Global\Access_Intel_OC_Mailbox`" | 仅 pyhwinfo 一手确认；其余 Unverified，且无官方保证 | §3 |
| "服务恢复改 flag 重启服务即生效" | 官方：下次**系统启动**生效；且 FALSE 时非零退出码进入 STOPPED 不触发 | §4（MS Learn 原文） |
| "TPMI 全面替代 RAPL MSR" | 并存独立驱动，单位布局不同；具体平台名单待核 | §5 |
| "DataGrid 默认行列都虚拟化" | 行默认 true、列默认 **false** | §7（官方 API） |
| "NativeAOT 下非泛型 JsonStringEnumConverter 在任何情况下都不可用" | 仅 Native AOT runtime 不支持；JIT 可用，AOT 用泛型版本或 blanket policy | §8（官方逐字） |

## 14. 核验状态汇总

| 结论 | 状态 |
|---|---|
| `CurrentUserOnly` 官方语义、MIC NO_WRITE_UP、#123903 存在与内容/未修复状态 | Verified（一手） |
| "高完整性客户端可连中完整性服务端" | Inferred（NO_WRITE_UP 单向推论，无官方逐字） |
| PawnIO 设备 SDDL、IOCTL function code、`SMU_RETRIES_MAX=8096`、`SMU_OK=0x01`、send_command 邮箱选择、`Access_PCI` 互斥注释 | Verified（上游源码逐字） |
| "ioctl_send_smu_command 仅 RSMU" | Verified 源码事实（出处由 Issue #20 更正为源码） |
| `WR0_WaitPciBus`、Avalonia 之外的互斥锁生态、D3D9On12 适用范围、TPMI 平台名单 | Unverified（无一手） |
| `SERVICE_FAILURE_ACTIONS_FLAG` 语义与"下次系统启动生效" | Verified（MS Learn 逐字） |
| TPMI 驱动存在、MMIO/VSEC、单位布局与 MSR 不同 | Verified（Linux 内核源码逐字） |
| WPF `CompositionTarget.Rendering` "持续动画、尽快注销" | Verified（官方）；"省电/最小化注销" = Inferred 推导 |
| DataGrid 行默认 true / 列默认 false | Verified（官方 API） |
| NativeAOT 非泛型转换器限制与泛型替代 | Verified（官方逐字） |
| 第三优先级各低证据项 | Inferred / Unverified（见 §12，不提升） |

> 本文件不修改产品源码、不触碰产品仓库；所有"对 Zentwik 的影响"均为文档层工程约束（Proposed），供后续开发评审引用。
