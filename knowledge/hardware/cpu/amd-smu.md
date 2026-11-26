# AMD SMU 访问链路

- 规范名称：AMD SMU 邮箱访问链路（PawnIO / RyzenSMU）
- 英文名称：System Management Unit mailbox access
- 缩写：SMU、MP1、RSMU
- 领域：`hardware/cpu`
- 核验状态：混合（见分项）
- 权威来源：RyzenSMU.p 上游模块源码；PawnIO 项目文档（签名模块、许可、反作弊）；SkatterBencher（PBO+ / Fmax Override 范围）、MSI 官方博客（25 MHz 步进、+200 MHz）；本仓库 `research/02-hardware-access.md` §2/§11
- 相关文档：[amd-curve-optimizer.md](amd-curve-optimizer.md)、[security-service.md](../../windows/security-service.md)、[ENGINEERING_NOTES.md](../../ENGINEERING_NOTES.md) §2（PawnIO SDDL/IOCTL/`0x01` 语义/ioctl_send_smu_command 仅 RSMU/PCI 互斥，2026-11-26 上游源码逐字核验）

## 1. 定义与范围

SMU 是 AMD 处理器内负责电源管理与调校的固件微控制器。应用经 PawnIO 驱动加载 `RyzenSMU.bin` 模块，通过 PCI 配置空间寄存器组（cmd/rsp/args）发送邮箱命令。本仓库聚焦 CO（`0x06`/`0xD5`）与 FMax（`0x6E`/`0x70`）。

## 2. 调用栈（Verified，本仓库源码事实）

```
SmuReader（门面/状态机）→ SmuSession（命令语义）→ SmuChannel（ioctl_send_smu_command）
                                                 → PawnIoDevice（CreateFile + DeviceIoControl）
```

- 设备路径 `\\?\GLOBALROOT\Device\PawnIO`；两个 IOCTL：加载模块、执行函数。
  - **口径勘误（2026-11-26）**：`0x821`（加载）/ `0x841`（执行）是 `CTL_CODE(0xA1B2, func, METHOD_BUFFERED, FILE_ANY_ACCESS)` 的 **Function Code**，不是完整 IOCTL 值；完整值须按该宏计算。上游源码逐字见 [ENGINEERING_NOTES.md](../../ENGINEERING_NOTES.md) §2.2。
- 请求体 = 32 字节 ASCII 函数名 + N×8 字节参数；返回体按 8 字节对齐回填。
- 模块文件外置分发（`PawnIO/RyzenSMU.bin`），不内嵌单文件。

## 3. 命令号与世代绑定（Inferred 逆向口径 / 报告称，原始日志缺失）

| 家族 | 型号 | SetMargin(CO写) | GetMargin(CO读) | GetBoostLimit（读 FMax） | SetBoostLimit（写 FMax 上限） |
|---|---|---|---|---|---|
| 0x17 | Zen2 | 0x0A | 0x7C | 0x6E | 0x00（不支持） |
| 0x19 | Zen3 | 0x0A | 0x7C | 0x6E | 0x00 |
| 0x19 | Zen4 | 0x06 | 0xD5 | 0x6E | 0x70 |
| 0x1A | Zen5 | 0x06 | 0xD5 | 0x6E | 0x70 |

> **方向勘误（2026-11-26，本轮核验）**：正确方向是 **Get=0x6E / Set all cores=0x70**，`GetBoostLimit` 是**读**、`SetBoostLimit` 是**写**。本仓库该表自首版（提交 `4f60894`）起即为该方向，`research/02` §2.2 同向，**基线内不存在"Set=0x6E / Get=0x70 写反"的错误**。《Gemini 复审资料库 1》§2.1 声称本表写反，与本仓库实际内容及 Git 历史冲突，判定为**复审报告误读**，不采纳其"修正方向"建议（正确方向已在该表，见 `knowledge/AUDIT.md` §10 第 25 项）。本轮仍独立打开上游源码核验并固化证据如下。

**FMax / BoostLimit 命令方向的一手证据（Verified 源码事实；作为芯片协议仍为 Inferred）**：

- `irusanov/ZenStates-Core`（dev 分支）`Hardware/Smu/Settings/Zen4Settings.cs`：`Rsmu.SMU_MSG_GetBoostLimitFrequency = 0x6E;` 与 `Rsmu.SMU_MSG_SetBoostLimitFrequencyAllCores = 0x70;`（逐行打开，2026-11-26）。`Zen5Settings.cs` 仅 `class Zen5Settings : Zen4Settings`（只改 HSMP 地址），**继承**这两个命令号，方向不变。
- 独立实现交叉证据：`leogx9r/ryzen_smu` `docs/rsmu_commands.md` 在 Matisse & Vermeer 表记 `GetMaxFrequency | 0x6E | Res0:MHz`（读返回 MHz），进一步印证 `0x6E = Get`。该文件为 Zen2/3 口径，未列 Zen4 的 0x70，故 0x70 的"全核写"以 ZenStates 源码为准。

**命令变体区分（避免把 Get/Set all cores 与别的命令混用）**：

| 变体 | Zen4 RSMU | 语义 | 证据 |
|---|---|---|---|
| `GetBoostLimitFrequency` | 0x6E | 读当前 Boost/FMax 频率上限 | ZenStates `Zen4Settings.cs`；ryzen_smu `GetMaxFrequency` |
| `SetBoostLimitFrequencyAllCores` | 0x70 | 写**全部核心**的 Boost 上限（Zentwik FMax 写入用此命令） | ZenStates `Zen4Settings.cs` |
| `SetOverclockFrequencyAllCores` / `PerCore` | 0x5F / 0x60 | 另一组 OC 频率命令（MP1 侧为 0x26 / 0x27），**与 0x70 不同**，不得混用 | ZenStates `Zen4Settings.cs` Overclock Options 段 |
| Zen2/3 `SetBoostLimit` | 0x00（旧版未配置） | 该版家族映射未配置全核写命令；这不等于已证明该世代所有 SKU 的硬件都不支持 FMax 写入 | 本表历史口径（Inferred，逆向） |

Zen4/5 补充命令号（**Verified 上游源码** `Zen4Settings.cs`，本轮逐行核对；作为硬件协议 Inferred）：全核 CO `SetAllDldoPsmMargin` = RSMU `0x07` / MP1 `0x36`；逐核 MP1 侧 `SetDldoPsmMargin=0x35`；Curve Shaper `Set=0xA6` / `Get=0x84`。RyzenAdj 对 FireRange/DragonRange 的逐核 `0x06` / 全核 `0x07` 独立交叉印证。按家族分派的完整对照见 [implementation-references.md](implementation-references.md) §4。

- 命令号来自逆向工具链口径，AMD 公开 PPR 不含 MP1 消息表（Inferred）。
- 家族判定是硬编码白名单，新 SKU（Hawk Point 0x75、部分 Dragon Range）可能误落到 Zen3 分支，发出错误世代命令（Inferred，高风险静默失败点）。

### 3.1 核心掩码编码（本仓库单一权威定义，其余文档只引用不重抄）

- 位布局：`[31:28]` = CCD index；`[27:24]` = CCX index；`[23:20]` = **核内序号（4 位数值字段，0–7）**；`[19:0]` 保留。**`[23:20]` 是数值字段，不是独热位。**
- 现代桌面（family > 17h）：`mask = (ccd << 28) | ((core % 8) << 20)`，CCX 位为 0（Zen3 起每 CCD 仅一个 CCX）。旧 17h 分支：`(ccd << 28) | ((ccx % 2) << 24) | ((core % 4) << 20)`。
- 来源（**Verified，外部源码逐字核对**）：`irusanov/ZenStates-Core` dev 分支 `Cpu.cs::MakeCoreMask`（源码注释 `// [31-28] ccd index`、`// [23-20] core index`）；`irusanov/SMUDebugTool` `SettingsForm.cs:2562-2584` 的 `EncodeCoreMarginBitmask` 兜底路径 `(ccd<<8 | local)<<20` 独立一致。
- 上游首选路径：经 APOB 核心映射 `MakeCoreMaskForLogicalCore` → `CoreMap.GetLogicalCore` 取得物理 CCD/CCX/Core，跳过被禁用的 CCD/核；不应仅用 `core/8`、`core%8` 简化推算（Inferred，来自源码结构）。
- **Zentwik 旧实现曾误用独热位 `1 << (20+local)`**（核 3 发 `0x00800000`，正确应为 `3<<20 = 0x00300000`），与本定义及外部权威实现不符；由此得到的 SMU 拒绝**不是平台限制**，见 `research/02` §11 第四轮审计注记。A1（2026-11-26）已删除该编码与"探测协商编码"逻辑，掩码一律由核映射产生。
- **无 APOB 时的映射来源（Verified 本机实测，Ryzen 9 9955HX / Family 1Ah Model 44）**：CPUID `0x8000001E` 的 `EBX[7:0]` 核号在本机是 **die 内连续编号 0–15**、`ECX[7:0]` NodeId 恒为 0，**不能**直接当作复合体内核号；CCD 分组须由 `GetLogicalProcessorInformationEx(RelationCache)` 的 L3 缓存域掩码给出（本机 2 个域：逻辑处理器 `0x0000FFFF` / `0xFFFF0000`，各 8 核 16 线程），再对 8 取模得到核内序号。`CPUID 0x80000026` 在本机只报 SMT/Core/Complex/Die 四级，其 `EDX` 是当前 x86APIC ID，不能给出 CCD 号。各域核数不一致（异构 CCX）时不产出映射。单机结论，不外推到其它 SKU。
- **APOB 可读性（Verified 环境事实）**：APOB 位于物理内存（上游扫描 `0xA200000` / `0x9F00000` / `0x4000000`），上游经 `inpoutx64`/`WinIO` 读取；PawnIO 官方模块集无物理内存读取能力，本机也未安装 inpoutx64。因此 APOB 路径需外部提供的原始表，不能默认可用。
- 写入参数（**本轮上游源码确认，Verified 源码事实**）：`args[0] = (mask & 0xfff00000) | margin16`，其中 `margin16 = margin<0 ? (0x100000+margin)&0xffff : margin`（`ZenStates-Core Utils.cs:180-184`）。即**掩码占高 12 位、margin 占低 16 位（16 位补码）、[19:16] 保留**。此前"低 16 位 vs 低 20 位"的位数歧义已由此澄清（历史记录中 `research/02` 的"低 20 位"是含保留位的宽松描述）。作为芯片硬件协议仍为 Inferred（AMD 未公开）。详见 [implementation-references.md](implementation-references.md) §3.2。

报告称（来源 Unknown，原始日志缺失）：`0xD5` 在本机（Ryzen 9 9955HX / Fire Range 被识别为 Granite Ridge）走 RSMU 邮箱，box 寄存器组 cmd 0x3B10524 / rsp 0x3B10570 / args 0x3B10A40。**早期实现使用错误独热位编码时曾报告"bit23 对应核被拒、其余 14 核可读"，该平台归因已于第四轮撤回**（见上条与 `research/02` §11），不得再作为平台限制事实引用。见 [measurement-evidence-schema.md](../../data/measurement-evidence-schema.md)。

## 4. 写入、验证与失败处理（本仓库旧实现记录，Verified 设计原则 / 实测细节来源 Unknown）

- 就绪探针：发 `0x01`，参数 `[1,0,…]`，要求回 `2`；不通即 `ProbeFailed`，不进入写入。
- CO 写入：逐核 SetMargin → GetMargin 回读 → 不等即 `VerifyFailed`。
- FMax 写入：先读基线再写，回读容差 ≤10 MHz。旧版 `SmuSession.GetFMax()` 在 Get 命令未配置或 `TrySend` 失败时均返回 0；`TuningStore.ResetToFactory()` 将基线 0 解释为未知并跳过写回。故 0 不是平台不支持的证据，也无法区分命令缺失与通信失败。旧版 Zen2/Zen3 的 `SetBoostLimit=0` 是该实现未配置写命令，不应概括为移动平台硬件不实现 FMax。支持状态应按具体家族命令映射、SMU 响应与真实读写验证。
- **FMax 与 OC 定频是不同命令族，不可合并判断**：定频用 `0x5A/0x5C/0x5D`，FMax 上限用 `0x6E`/`0x70`。9955HX 上定频被固件关闭（`0x6F`→`0xE6`、`0x2C` 不存在），但 FMax 未被拒绝（2026-11-26 产品负责人确认；单机结论，写入证据待管理员会话复核）。不得把"定频不可行"写成"FMax 不可用"。
- **FMax / Boost Clock Override（本参数的分层权威定义）**：本节是仓库内单一权威定义点，其余文档只引用不重抄；分层规则见 [parameter-record-schema.md](../../data/parameter-record-schema.md)。

  **参数记录**：Boost Clock Override（Fmax Override / Auto OC）
  **语义**：Maximum Boost Frequency Ceiling Override · **数据类型**：有符号整数 · **单位**：MHz

  | 层 | 取值 | 说明 / 来源 / 状态 |
  |---|---|---|
  | L1 固件/AGESA 实际允许 | -1000…+200 MHz，步进 25 MHz | BIOS/PBO+ 侧，Zen 2 起；SkatterBencher、MSI 官方博客多源一致。Inferred（专业社区来源，非 AMD 官方规范；见 §9 核验状态分项） |
  | L1 工具侧更细粒度 | Zen5 用专用工具可低至 1 MHz | BIOS 菜单仍 25 MHz；SkatterBencher。Inferred |
  | L3 实际运行频率 | 由 Precision Boost 2 按温度/功耗/电流决定 | **+200 只抬上限，不保证实频升 200 MHz**；非参数范围 |
  | L5 Zentwik 约定 | 字段 `fmaxOverrideMHz`（i16，单位 MHz） | Proposed；不与 `CurveOffsetStep` 互换 |

  **术语辨析（不得画等号）**：FMax（Fused maximum frequency）是芯片出厂融合的最大频率限制器，是一个**绝对频率**；Boost Clock Override / Fmax Override / Auto OC 是 PBO+ 提供的**频率上限覆盖偏移参数**；实际运行频率是 Precision Boost 最终决定的值。三者是不同概念。不得把 FMax 绝对频率范围（如某 SKU 4450…5650 MHz，见 `research/01-legacy-inventory.md`）与 FMax Override 的偏移范围（-1000…+200 MHz）混淆。
  - 适用范围与条件：随 CPU 代际、AGESA、 BIOS 与工具而变，**不是所有处理器/BIOS/工具的绝对限制**（Unverified 部分按平台核验）。
  - 与 CO 的区别：CO 是无单位档位偏移 V/F 曲线，FMax Override 是带单位的频率上限覆盖，二者单位与语义均不同，禁止共用类型或换算。
- 并发：旧版进程内 `static lock`，**跨进程不成立**；新项目句柄收归单一特权服务（ADR-003）。
- 超时：驱动内部轮询上限 8096 次；旧版只取 Win32 错误码而非 NTSTATUS，"区分拒绝与命令不存在"在实现上做不到（Inferred）。

## 5. 权限、签名与安装（Verified 文档 + 报告称）

- 非提权进程打开 PawnIO 设备返回访问被拒绝；SYSTEM 服务可打开（报告称，原始日志缺失、来源 Unknown；权限原理本身由 Windows ACL 文档 Verified 支持）。
- 签名版驱动**只加载维护者签名的模块**；新项目不能自造 `.bin`。
- 许可：驱动 GPL-2.0（含模块例外），官方模块 LGPL-2.1；商业内置需授权。
- 反作弊：FACEIT 曾整体封禁 PawnIO 所用签名证书，属已知不可用场景。
- 安装器不安装 PawnIO，由用户自装（`winget install namazso.PawnIO`）。

## 6. 通道三态（Verified 设计原则）

`NotSupported`（平台不符）/ `DriverUnavailable`（环境缺失）/ `ProbeFailed`（硬件不应答）必须分开，不可压成 bool；单项不可用不得让产品整体失效。

## 7. 常见误解

- "SMU 命令号与固件版本无关" → 与固件无直接绑定，但与平台/世代强绑定。
- "能在两个进程同时打开设备" → 无跨进程仲裁，须单进程独占。
- "功耗限制移除=技术不可行" → 移除提交无理由，不能据此推断。

## 8. 待核验

Zen3/4/5 各一台的 CO/FMax 读写矩阵；S3/S4 恢复后是否丢 CO；PawnIO 多进程并发 ACL。

## 9. 核验状态分项

命令号 = Inferred（逆向协议口径）；**FMax 方向 Get=`0x6E` / Set all cores=`0x70` = Verified 源码事实（ZenStates `Zen4Settings.cs` 逐行 + `Zen5Settings.cs` 继承关系 + ryzen_smu `0x6E GetMaxFrequency` 交叉，2026-11-26；见 §3 方向勘误）**，作为芯片协议仍为 Inferred，且本仓库基线自首版即正确、Gemini 报告的"写反"指控不成立；**其他 OC 频率变体 0x5F/0x60（RSMU）与 0x26/0x27（MP1）≠ 0x70 = Verified 源码事实（ZenStates `Zen4Settings.cs`）**；**核心掩码编码 `[23:20]` 为 4 位核内序号、非独热位 = Verified（外部源码 `Cpu.cs::MakeCoreMask` 与 `SettingsForm.cs` 逐字核对，见 §3.1）**；Zentwik 旧实现误用独热位 `1<<(20+local)` = Verified（源码比对），据此**撤回"平台拒绝 bit23"归因**（第四轮）；写入回读、权限 = 本仓库报告称，原始日志缺失、来源 Unknown（不得标 Verified 实测）；通道三态等设计原则 = Verified 设计决策；FMax Override 范围 -1000…+200 MHz / 25 MHz 步进、上限非偏移语义 = Inferred（专业社区 SkatterBencher 已逐字核对原文，非 AMD 官方规范）；家族误分类、Zen5 的 1 MHz 粒度 = Inferred；休眠清回出厂 = Unverified。
