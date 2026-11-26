# ADR-008：硬件通道的驱动依赖（PawnIO）与 APOB 读取缺口

- 状态：Accepted（条款 1–3、5 已实施；条款 4 的分发合规尚未落地，见"后果"）
- 日期：2026-11-26
- 决策负责人：Motu-Z
- 相关研究报告：`research/02-hardware-access.md`、`knowledge/ENGINEERING_NOTES.md` §2、`knowledge/hardware/cpu/amd-smu.md` §5、`knowledge/hardware/cpu/amd-curve-optimizer.md`

## 背景

Zentwik 需要在用户态完成 AMD CO / FMax 与 Intel 电压偏移的读写，这要求内核级 MSR / SMU 邮箱访问。旧版 Zentwik 走 WinRing0 系驱动，该驱动已有公开漏洞记录、且被主流反作弊封禁，不能作为新工程的默认依赖。

同时，旧版 A1 阶段的核映射依赖 AGESA 的 APOB（PSP Output Block）原始表，从**物理内存**读取。需要回答两件事：

1. 新工程的硬件通道依赖哪个驱动，边界在哪。
2. APOB 读不到时，核映射还能不能成立。

这与 ADR-003（单特权进程持有设备句柄）直接相关：驱动访问必须收敛在 Agent 内。

## 决策

### 1. 唯一硬件驱动是 PawnIO，且只用维护者签名模块

- 硬件访问只经 PawnIO 设备 `\\?\GLOBALROOT\Device\PawnIO`；不自研内核驱动，不引入 WinRing0 / inpoutx64 / 任何第二个内核驱动。
- 只加载 `PawnIO.Modules` 中维护者签名的模块（AMD 用 `RyzenSMU.bin`，Intel 用 `IntelMSR.bin`）。**不自造 `.bin`**：签名版驱动只接受维护者签名的模块，自造模块无法加载，改造上游模块也超出维护能力。
- 驱动访问权只归 Agent（LocalSystem）。上游 SDDL 为 `D:P(A;;GA;;;SY)(A;;GA;;;BA)`，非提权进程打开即被拒；这与 ADR-003 一致，不需要额外加固。

### 2. 模块随产品分发，但驱动由用户自行安装

- 模块作为独立文件随 Agent 输出（仓库 `modules/PawnIO/*.bin` → 安装目录 `PawnIO/`），不内嵌进程序集、不重写、不混淆。
- 安装器**不安装 PawnIO 驱动**。用户自行安装（上游提供 `winget install namazso.PawnIO`）。
- 通道不可用必须按三态区分，不压成布尔：`driverUnavailable`（驱动未装）/ `moduleMissing`（模块缺失）/ `privilegeMissing`（驱动拒访，需管理员）/ `probeFailed`（硬件不应答）/ `rejected`（通道可用但平台拒绝该操作）。现有实现已如此。

### 3. 核映射不依赖 APOB

- 主路径：CPUID `0x8000001E` 取核号 → `GetLogicalProcessorInformationEx(RelationCache)` 取 L3 域 → 与物理核数交叉核对，得到 CCD/CCX/核的物理位置（A1 已实现）。
- APOB 降级为**可选增强**：只有在拿到外部 dump（用户提供或未来独立读取通道）时才用于校正；**不得**成为任何功能的启用前置条件，也不能因缺 APOB 就把核映射判为不可用。
- 该决策的直接后果：`AmdSmuTuningProvider` 在拿不到物理位置时返回 `rejected`（明确失败），而不是猜一个掩码写下去。

### 4. 分发许可义务（尚未落地）

- 驱动本体未随包分发，因此不触发 GPL-2.0 的组合作品问题：上游模块例外条款明确覆盖"仅通过设备 IOCTL 接口通信的独立模块"，Zentwik 与 PawnIO 之间正是这一关系。
- 但**模块是随包分发的 LGPL-2.1-or-later 二进制**，必须随包提供：LGPL-2.1 许可证全文、模块来源与版本标识、对应源码获取途径（`namazso/PawnIO.Modules`）。
- 若将来改为随包分发驱动本体，则须同时履行 GPL-2.0（提供 PawnIO 源代码或书面要约）；若不愿承担，上游提供替代授权（`admin@namazso.eu`），需在商业发布前确认。

### 5. 明确不做的

- 不做驱动签名自签发、不做测试签名安装引导。
- 不在非提权进程里尝试打开设备后"降级继续"——直接报 `privilegeMissing`。

## 候选方案

| 方案 | 优点 | 缺点 | 结论 |
|---|---|---|---|
| **A. PawnIO + 官方签名模块**（选中） | 驱动已签名、可持续更新；模块除外条款使闭源产品可合法分发；用户自装减少安装器风险 | 受限于上游提供的模块能力（无物理内存读、无任意 MSR）；上游单点维护 | 采用 |
| B. 自研/自签内核驱动 | 完全自由，可读 APOB、可做任意 MSR | 需 EV 证书 + attestation/WHQL 签名；蓝屏风险自担；反作弊判定与驱动分发责任全在自身；维护成本高 | 不采用 |
| C. 第二条驱动（inpoutx64 / WinRing0）专读 APOB | 直接补上物理内存读，旧版核映射代码可照搬 | 正是 PawnIO 要替代的对象：WinRing0 有公开漏洞、被反作弊封禁；双驱动同时驻留使攻击面与安装失败点翻倍；对内是"为一张只读表引入内核组件" | 不采用 |
| D. 纯 MSR，不用任何驱动 | 零驱动依赖 | 无法发 SMU 命令，CO/FMax 与 Intel 电压偏移全部不可用 | 不采用（仅作为监控子集的降级路径） |

## 决策理由

- **APOB 确实需要物理内存读，而 PawnIO 给不了**。证据：ZenStates-Core 的 `Apob.RawTable` 赋值处为 `_ioDriver.ReadMemory(new IntPtr(Address), (int)Header.TableSize)`（`Hardware/Apob/Apob.cs:277`），其实现走 inpoutx64 的 `MapPhysToLin`（`Drivers/IODriver.cs:105-121`）。即"读 APOB"与"用 PawnIO"在现有模块能力下互斥，不是配置问题。
- **CPUID 路径已被本机实测证明够用**。9955HX 上 `0x8000001E` 取到核号 0–15、L3 域 `0x0000FFFF`/`0xFFFF0000` = 2 CCD × 8 核，与物理核数一致；掩码由 `AmdCoreMask` 生成后 CO 写入链路可跑通。缺 APOB 的代价是拿不到 die 内拓扑的厂商原始标注，而非拿不到核映射。
- **引入第二个驱动不成比例**。APOB 在本工程只用于核物理位置的交叉校验，而它带来的成本是：一个已知有漏洞的驱动、一次反作弊误判风险、一套额外的安装/卸载/回滚路径。ADR-003 已把"单特权进程、单设备"作为结构前提，双驱动破坏该前提。
- 反作弊是产品级约束而非技术洁癖：FACEIT 曾整体封禁 PawnIO 所用签名证书。因此"能少一个内核组件就少一个"是硬要求，且必须在产品文档里如实告知用户该通道在部分反作弊环境不可用。

## 后果

正面：

- 安装面最小：一个可选驱动、用户自装、失败可解释（`driverUnavailable`）。
- 核映射不依赖不可读的资源，A1 实现在无 APOB 机器上也能给出确定结论。
- 许可路径清晰：闭源产品 + IOCTL 通信 + LGPL 模块，落在上游例外条款内。

负面 / 新增维护责任：

- **分发合规尚未落地**：仓库当前没有 GPL-2.0 / LGPL-2.1 文本，也没有模块来源说明（`modules/PawnIO/*.bin` 仅由 `Zentwik.Agent.csproj` 作为内容拷贝到输出 `PawnIO/`）。发布前必须补：两份许可证全文、模块来源与版本、源码获取途径。**这是本 ADR 唯一的未完成项。**
- 受上游能力限制：无任意 MSR 读写、无物理内存读。后续若需要 APOB、SPD、更多 SMU 邮箱（如移动 APU 的 MP1），只能等上游扩模块能力，或按方案 B 重新评估。
- 上游单点风险：模块更新节奏、签名证书状态（反作弊封禁）不受本工程控制。
- 本机遗留问题未解释：`0x00000000`（核 0）的寄存器读被拒，原因未定，不能据此推断平台限制。

## 验证与回滚

已完成的验证：

- 本机（9955HX / Zen5）实测 PawnIO 驱动 RUNNING，SMU 版本 `0x04621A00` 可读，核映射与 L3 域一致，CO 写入链路可发起（固件按其策略拒绝，属预期）。
- 非提权进程打开设备被拒 → `privilegeMissing`；服务（LocalSystem）可打开。
- Intel 通道在 AMD 机器上加载 `IntelMSR.bin` 返回错误 50（`ERROR_NOT_SUPPORTED`），只经 fake + 单测覆盖，**未在 Intel 硬件实测**。

回滚 / 重新评估条件：

- 上游停止签发模块、签名证书被主流反作弊普遍封禁、或 PawnIO 停止维护 → 按方案 B 重新评估（届时需给出证书与维护成本）。
- 若产品必须提供 APOB 级拓扑（例如逐 CCX 调校成为刚需），先把"外部 dump 导入"做成只读可选路径，仍不引入第二驱动；只有在 dump 导入被证明不够用时才重开本 ADR。

## 相关资料

- 上游许可证原文：<https://github.com/namazso/PawnIO>（GPL-2.0 + 模块例外；替代授权联系方式）
- 模块源码：<https://github.com/namazso/PawnIO.Modules>（`RyzenSMU.p` 为 LGPL-2.1-or-later）
- 设备路径 / IOCTL / SDDL / `send_command` 邮箱选择：`knowledge/ENGINEERING_NOTES.md` §2
- 签名、安装、反作弊边界：`knowledge/hardware/cpu/amd-smu.md` §5
- 参考实现（本地副本 `D:\Projects\Reference Projects\ZenStates-Core-AOT-master`）：`Hardware/Apob/Apob.cs`、`Drivers/IODriver.cs`
- 相关 ADR：[ADR-003 进程与权限模型](ADR-003-process-model.md)、[ADR-005 三种方案与状态所有权](ADR-005-plans-and-ownership.md)
