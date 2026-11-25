# 成熟实现参考（按开发领域索引）

- 定位：**实现参考**，不是权威规范、不是 Zentwik 设计契约。目的是理解成熟工具"用什么方法、为什么这样、如何处理硬件/固件差异、如何做异常与边界、有哪些已知限制"，为 Zentwik 后续开发提供可评估的选项。
- 标注：**【已研究】**＝本轮已逐行打开源码、结论见下文；**【仅建议】**＝只确认仓库存在与用途，未逐行研究、不作为技术结论来源。
- 权威基础规范：[amd-curve-optimizer.md](amd-curve-optimizer.md)（CO 单一权威定义）、[amd-smu.md](amd-smu.md)（SMU/掩码/命令号单一权威定义）、[intel-oc-mailbox.md](intel-oc-mailbox.md)

> 使用原则：源码行为只说明"该工具在该版本这样实现"，**不等于硬件事实、不等于 AMD/Intel 官方规范、不等于全平台通用**。一个工具的限制不得写成硬件限制（见 [measurement-evidence-schema.md](../../data/measurement-evidence-schema.md) §12）。所有命令号/编码以基础规范为准，本文引用不重抄数值表。

## 1. 按开发领域的参考项目速查

只列值得借鉴的成熟项目；没有合适项目的领域留空。每项给出仓库、用途、可借鉴的具体实现点。

### 1.1 AMD / Intel 硬件调校

| 项目 | 状态 | 主要用途 | 值得 Zentwik 借鉴的具体实现 |
|---|---|---|---|
| [irusanov/ZenStates-Core](https://github.com/irusanov/ZenStates-Core) | 【已研究】 | APOB 映射、CO/Curve Shaper 命令编码 | 逻辑→物理核映射并跳过禁用核（§2）；16 位补码 margin 编码（§3.2）；全核/逐核分命令与世代分派（§3.1/§4） |
| [irusanov/SMUDebugTool](https://github.com/irusanov/SMUDebugTool) | 【已研究】 | CO 掩码、Curve Shaper | `EncodeCoreMarginBitmask` 对 CO 掩码的独立实现（§2.3）；Curve Shaper 五频段结构与输入钳制（§3.6） |
| [FlyGoat/RyzenAdj](https://github.com/FlyGoat/RyzenAdj) | 【已研究】 | 按 CPU 家族分派命令 | 家族枚举分派命令号；用 SMU 响应码区分"命令不存在"与"命令被拒"三态（§3.5/§4） |
| [torvalds/linux](https://github.com/torvalds/linux) | 【已研究】 | RAPL MSR 地址 | `msr-index.h` 中 RAPL 三 MSR 地址；OC Mailbox 无架构性定义（§5） |
| [mihic/linux-intel-undervolt](https://github.com/mihic/linux-intel-undervolt) | 【仅建议】 | Intel 电压偏移逆向 | Intel 0x150 电压编码思路（社区逆向、单机测试，仅 Inferred；见 [intel-oc-mailbox.md](intel-oc-mailbox.md) §4）——未逐行研究，仅作方向参考 |

### 1.2 硬件监控与传感器采集

| 项目 | 状态 | 主要用途 | 值得 Zentwik 借鉴的具体实现 |
|---|---|---|---|
| [LibreHardwareMonitor/LibreHardwareMonitor](https://github.com/LibreHardwareMonitor/LibreHardwareMonitor) | 【仅建议】 | 硬件传感器读取、PM Table | C# 传感器抽象与采样组织方式。**未逐行研究**；其逐核 CO 写入不是主用途，不能作为 CO 写入参考 |

### 1.3 Windows 服务与 IPC

暂无合适的成熟参考项目，留空。旧版实现事实见 [research/03](../../../research/03-ui-ipc-deployment.md) 与 [security-service.md](../../windows/security-service.md)，本轮不新增外部项目。

### 1.4 WPF / MVVM / UI

| 项目 | 状态 | 主要用途 | 值得 Zentwik 借鉴的具体实现 |
|---|---|---|---|
| [CommunityToolkit/dotnet](https://github.com/CommunityToolkit/dotnet) | 【仅建议】 | WPF/.NET MVVM 工具包 | MVVM 命令/属性样板、源生成器用法。**未逐行研究**；Zentwik 已定 WPF 自绘图表，只参考 MVVM 工程结构，不引入第三方图表库 |

### 1.5 进程管理与场景自动化

暂无合适的成熟参考项目，留空。本领域的官方 API 事实（CPU Sets、电源方案）已在 `knowledge/windows/` 单独维护。

### 1.6 插件与扩展机制

暂无合适的成熟参考项目，留空。Zentwik 的进程外插件隔离属项目约定（见 [modules-capabilities.md](../../architecture/modules-capabilities.md)），不套用现成插件框架。

### 1.7 已研究项目的逐行访问位置（证据留痕）

| 项目 | 分支 | 具体文件与行 | 用途 |
|---|---|---|---|
| `irusanov/ZenStates-Core` | dev | `Hardware/Apob/ApobCoreMap.cs`（526 行，全文）；`Hardware/Smu/Commands/SetPsmMarginSingleCore.cs`、`GetPsmMarginSingleCore.cs`、`SetPsmMarginAllCores.cs`、`SetCurveShaperMargin.cs`、`GetCurveShaperMargin.cs`；`Hardware/Smu/Settings/Zen4Settings.cs`、`Zen5Settings.cs`；`Utils.cs:178-197`；`Cpu.cs`（`MakeCoreMask`/`MakeCoreMaskForLogicalCore`） | APOB 映射、CO/Curve Shaper 命令编码、世代命令号、写入参数编码（Verified 源码事实） |
| `irusanov/SMUDebugTool` | master | `SettingsForm.cs:2562-2584`（`EncodeCoreMarginBitmask`）、`:352-365`、`:507`、`:808-809` | CO 掩码独立印证、Curve Shaper 结构、工具输入上限（Verified 源码事实） |
| `FlyGoat/RyzenAdj` | master | `lib/api.c:1404-1479`（`set_coall`/`set_coper`/`set_cogfx`）、`:439-452`（`_do_adjust_psmu` 宏与响应分类）；`lib/ryzenadj.h:17-34`（家族枚举） | 按 CPU 家族分派命令号、响应码三态分类（Verified 源码事实） |
| Linux kernel | — | `arch/x86/include/asm/msr-index.h`（`MSR_RAPL_POWER_UNIT=0x606`、`MSR_PKG_POWER_LIMIT=0x610`、`MSR_PKG_ENERGY_STATUS=0x611`）；该文件**无** `MSR_IA32_OC_MAILBOX` | RAPL MSR 地址确认、OC Mailbox 非架构性佐证（Verified 源码事实） |
| MSI FAQ-10122 / SkatterBencher / Intel SDM 镜像 | — | 见 [AUDIT.md](../../AUDIT.md) §6.2 逐字核对记录 | CO 显示钳制、FMax、RAPL 单位、Intel 编码来源定性（文档，非源码） |

未逐行打开、不作为结论来源：AMD Ryzen Master（闭源，仅官方 FAQ/Release Notes）、UXTU。

## 2. APOB 核心映射（重点主题）

### 2.1 APOB 是什么、解决什么问题

APOB = AGESA 在启动阶段（ABL，AGESA Boot Loader）留下的 **PSP/AGESA 输出缓冲区**，其中 group APOB_CCX（type 3=CCD 映射、type 1=CCX/complex 映射）保存了 ABL 建立的**逻辑核 → 物理 CCD/CCX/Core 映射**。

- 来源：`ApobCoreMap.cs` 类注释明确写 "The CCD / CCX / core logical to physical map the ABL leaves in the APOB (group APOB_CCX)"；映射类型常量 `APOB_CCX_GROUP=3`、`CCX_MAP_TYPE=1`、`CCD_MAP_TYPE=3`（对应 openSIL `APOBCMN.h` 的 `APOB_CCX_LOGICAL_TO_PHYSICAL_MAP_TYPE` / `APOB_CCD_LOGICAL_TO_PHYSICAL_MAP_TYPE`）。（Verified 源码注释与常量）
- 为什么需要：OS 枚举的逻辑核序号（0..N-1）与物理 CCD/CCX/Core 位置**不是可靠的一一对应**——存在被熔断/禁用的 CCD 与核、SMT 开关、不同 SKU 的核心配置。CO 掩码 `[23:20]` 需要的是**物理** core index，不能简单用 `logical/8`、`logical%8` 推算。（Verified 源码结构推论 + `MakeCoreMaskForLogicalCore` 实现）

### 2.2 数据结构与映射字段

`ApobCoreMapCore` 每个核含：`LogicalIndex`（在整张映射的逻辑序位置，即 OS 枚举顺序）、`LogicalCcd/Ccx/Core`、`PhysicalCcd/Ccx/Core`、`ThreadEnabled[]`（每线程启用状态）、`EnabledThreads`。`ApobCoreMap` 提供 `GetLogicalCore(logicalIndex)`（按逻辑序取核，**跳过禁用核**）、`GetPhysicalCoreMask(physicalCcd)`、`PhysicalCcdMask/Count`、`PhysicalCcxCount`、`SlotLayout`。

关键语义（Verified 源码）：
- `Cores` 列表**只含启用的核**——`TryDecode` 中 `physicalCore==NOT_PRESENT(0xFF)` 或线程标志无效时 `continue`，不加入列表。因此逻辑序号是"启用核的序号"，自动跳过被禁用/熔断的核。
- `Parse()` **优先 CCD 映射（type 3），失败才退回 CCX 映射（type 1）**，并按 InstanceId（socket/die）分别产出映射。
- 布局（数组维度）不是硬编码：源码从 entry 大小推导 `CcdSlots/ComplexSlotsPerCcd/CoreSlotsPerComplex/ThreadSlotsPerCore`，候选优先 `{threads=2,1}`、`{cores=8,16,4}`、`{complexes=1,2}`，并用 `expectedThreads`（系统逻辑处理器数）打分选择最可能布局。（Verified 源码）

### 2.3 从核心索引到 CO 掩码（完整链路）

成熟工具的完整路径（Verified 源码，`Cpu.cs` + `SettingsForm.cs`）：

```
OS 逻辑核序号
  → ApobCoreMap.GetLogicalCore(i) 得 PhysicalCcd/PhysicalCcx/PhysicalCore（跳过禁用核）
  → MakeCoreMask(physicalCore, physicalCcd, physicalCcx)
      现代桌面(family>0x17): (ccd << 28) | ((core % 8) << 20)
      旧 17h:                (ccd << 28) | ((ccx % 2) << 24) | ((core % 4) << 20)
  → SetPsmMarginSingleCore.Execute(coreMask, margin)
      args[0] = (coreMask & 0xfff00000) | MakePsmMarginArg(margin)
```

`SMUDebugTool.EncodeCoreMarginBitmask` 独立实现同一逻辑：APU 直接返回 `coreIndex`；有 APOB 映射走 `MakeCoreMaskForLogicalCore`；兜底 `(ccdIndex<<8 | localCoreIndex) << 20`（等价于数值编码）。（Verified 源码）

> 结论：CO 掩码的权威定义见 [amd-smu.md](amd-smu.md) §3.1。**必须优先走 APOB 物理映射**；只有在无映射时才退回简化推算，且简化推算在熔断核/非连续拓扑上会错。Zentwik 早期独热位编码错误正是绕过了这条链路（见 [AUDIT.md](../../AUDIT.md) §7）。

## 3. CO 写入、回读与部分失败处理

### 3.1 全核与逐核是不同命令（不是同一命令换掩码）

`ZenStates-Core` 的 `SetPsmMarginSingleCore`（逐核）与 `SetPsmMarginAllCores`（全核）是**两个命令类、不同命令号**：Zen4 RSMU 逐核 `SetDldoPsmMargin=0x06`、全核 `SetAllDldoPsmMargin=0x07`；MP1 侧逐核 `0x35`、全核 `0x36`（Verified `Zen4Settings.cs`）。`RyzenAdj` 独立印证：FireRange/DragonRange 全核 `_do_adjust_psmu(0x7)`、逐核 `(0x6)`（Verified `api.c:1426/1455`）。

实现要点：全核写入用专门的 all-cores 命令，不要逐核循环冒充"全核"；两者语义与生效范围不同。

### 3.2 写入参数编码（16 位补码 margin）

`Utils.MakePsmMarginArg`（`Utils.cs:180-184`，Verified）：

```csharp
public static uint MakePsmMarginArg(int margin) {
    int offset = margin < 0 ? 0x100000 : 0;
    return (uint)(offset + margin) & 0xffff;
}
```

即 **margin 以 16 位补码放到参数低 16 位**，`args[0] = (coreMask & 0xfff00000) | margin16`；注释明确 `[31:28]CCD / [27:24]CCX / [23:20]core / [19:16]reserved / [15:0]CO margin`。这澄清了仓库历史记录中"低 16 位 vs 低 20 位"的歧义：**掩码占高 12 位、margin 占低 16 位、中间 [19:16] 保留**（Verified 源码；此前 `amd-smu.md` §3.1 标"需实机确认"的位数差异，本轮由源码确认）。

### 3.3 世代范围差异（工具口径，非官方）

`SetPsmMarginSingleCore.cs`/`SetPsmMarginAllCores.cs`/`Utils.cs:178` 源码注释一致："CO margin range from -30 to 30 before Zen 4, and from -50 to 50 on Zen 4 and newer"。这是**上游维护者的口径**（Verified 源码注释存在），但不是 AMD 官方规范；与仓库 [amd-curve-optimizer.md](amd-curve-optimizer.md) §5"官方未发布固件档位范围"不冲突——一个是工具注释，一个是官方事实缺失，须分开标注。（Inferred 作为硬件事实）

### 3.4 回读路径的多世代分派（成熟做法）

`GetPsmMarginSingleCore.cs` 展示处理不同硬件的三种路径（Verified 源码）：
1. 常规（Zen4 等）：RSMU `GetDldoPsmMargin`（Zen4=0xD5）直接读回 count。
2. TYPE_CPU9（Bristol Ridge，老 APU）：RSMU 读命令不可用，改走 MP1 读两组 GBV（IEEE754 浮点 V/F 点），外推默认 P0、换算成 mV，再按 `MillivoltsPerCount=3.0f` 反推 count。
3. 读失败返回 `CMD_REJECTED_PREREQ` 并把 args[0] 置 0，不返回伪值。

`SetPsmMarginAllCores.cs` 同理分派：TYPE_CPU9 用 GBV 加权（P0 100%/P1 75%/P2 50%/P3 25%，并设 `MinSafeGuardbandV=0.020` 安全下限）；TYPE_CPU0 用 legacy VID（`CurveOptimizerToVid`）；默认用 PsmMargin。

> 对 Zentwik 的启示：**同一"CO 写入"在不同 CPU 世代底层是完全不同的操作**（邮箱 count / GBV 浮点 / legacy VID）。不能假设一套编码全平台通用；能力探测必须落到"该平台走哪条路径"，而不是只判"支持/不支持 CO"。（Derived 设计启示，非产品约定）

### 3.5 响应三态分类（命令不存在 vs 被拒）

`RyzenAdj._do_adjust_psmu` 宏（Verified 源码）：`REP_MSG_OK` → 成功；`REP_MSG_UnknownCmd` → `ADJ_ERR_SMU_UNSUPPORTED`；其他 → `ADJ_ERR_SMU_REJECTED`。即**通过解析 SMU 响应码区分"命令不存在"与"命令被拒"**。这优于只取 Win32 错误码的做法（后者把两者压成同一错误）。

### 3.6 已知限制与边界（工具事实）

- `RyzenAdj set_cogfx`：Strix Halo 分支注释 "0xB7 is rejected on this architecture" 且不加任何命令——说明同一命令并非全架构可用。（Verified 源码注释）
- `GetCurveShaperMargin` 注释：Zen4/5 `GetCurveShaperMargin=0x84`，"first 5 arguments are frequency tiers [min, low, medium, high, maximum], 6th unused"——印证 Curve Shaper 5 频段结构（Verified 源码）。
- `SetCurveShaperMargin` 的 `EncodeMargin` 把 margin 钳到 [−50,+30] 并转 `sbyte`，`frequencyTier` 限 0–4（Verified 源码）——这是 SMU Debug Tool 说明文字"−50…+30"出现在 Curve Shaper 上下文的**源码级解释**：它是 Curve Shaper 的钳制，不是 CO 的范围。
- 命令号随厂商/世代而异（见下节），且公开 PPR 不含 MP1 消息表——所有命令号都是逆向口径（Inferred，见 [amd-smu.md](amd-smu.md) §3）。

## 4. 命令号按家族分派对照（工具事实，Inferred 作为硬件协议）

| 家族/代际 | 逐核 CO 命令 | 全核 CO 命令 | 邮箱 | 来源 |
|---|---|---|---|---|
| Zen4 Raphael / Zen5（桌面） | RSMU `0x06` | RSMU `0x07` | RSMU | ZenStates `Zen4Settings.cs` + RyzenAdj（FireRange/DragonRange 同值） |
| Zen4/5 MP1 侧 | MP1 `0x35` | MP1 `0x36` | MP1 | ZenStates `Zen4Settings.cs` |
| Cezanne / Renoir / Lucienne | `0x54`（MP1） | `0x55`（MP1） | MP1 | RyzenAdj `api.c` |
| Phoenix / Rembrandt / Vangogh / HawkPoint / KrackanPoint / StrixPoint | `0x4B`（MP1） | `0x4C`（MP1） | MP1 | RyzenAdj `api.c` |
| Curve Shaper（Zen4/5） | `Set=0xA6` / `Get=0x84`（RSMU） | — | RSMU | ZenStates `Zen4Settings.cs` |

> 表内数值为**两个独立上游工具源码交叉一致的部分**标 Verified 源码事实；作为"芯片硬件协议"则统一为 Inferred（AMD 未公开）。**不得**把某一行推广到所有 AMD 平台（见 [measurement-evidence-schema.md](../../data/measurement-evidence-schema.md)）。

## 5. 对 Intel 侧的实现参考要点

- RAPL MSR 地址：`MSR_RAPL_POWER_UNIT=0x606`、`MSR_PKG_POWER_LIMIT=0x610`、`MSR_PKG_ENERGY_STATUS=0x611`（Linux kernel `msr-index.h`，Verified 源码）。单位换算公式（功率 1/2^PU、能量 1/2^ESU、时间 1/2^TU）见 [units.md](../../data/units.md)，本轮未重新展开。
- OC Mailbox `0x150` **不在** kernel `msr-index.h` 中——佐证其为非架构接口、无公开 MSR 定义（Verified 该"缺失"事实）。
- Intel 电压偏移编码（11 位 S11.0.10、×1.024）的上游来源 `linux-intel-undervolt` 自述为逆向、单机测试，只能标 Inferred（见 [intel-oc-mailbox.md](intel-oc-mailbox.md) §4 与 [AUDIT.md](../../AUDIT.md) §6.2）。

## 6. 未解决 / 需实机验证

1. APOB 映射在 Zentwik 目标平台（Ryzen 9 9955HX / Fire Range）上是否可读、布局维度为何——本仓库无原始 APOB dump。A1（2026-11-26）已确认**读取通道**问题：上游 `Apob` 经 `inpoutx64`/`WinIO` 读物理内存（扫描 `0xA200000`/`0x9F00000`/`0x4000000`），PawnIO 官方模块集不含物理内存读取，本机也未装 inpoutx64；因此 APOB 只能由外部提供的原始表进入。Zentwik 当前的替代来源见 [amd-smu.md](amd-smu.md) §3.1（CPUID 核号 + L3 域，单机已实测；是否等价于 APOB 物理核号仍 Unverified）。
2. 修正为数值编码 + APOB 映射后，16 核是否全部可读写（用户报告"其他工具可改全核" = User-provided，未独立复核）。
3. SMU Debug Tool −50/+30 界限来源（工具钳制 / 邮箱接受范围）——AUDIT §6/§7 待核。
4. 各命令号在具体 SKU + AGESA 组合下的真实有效性——需逐平台回读矩阵，禁止单机外推。

## 7. 核验状态分项

| 结论 | 状态 |
|---|---|
| APOB 结构/字段/解析逻辑、跳过禁用核、布局推导 | Verified（上游源码逐行） |
| CO 掩码完整链路（APOB → MakeCoreMask → mask） | Verified（两处独立上游源码一致） |
| 全核/逐核是不同命令；写入参数为 16 位补码 margin | Verified（上游源码） |
| 命令号家族分派表数值 | Verified 源码事实 / Inferred 作为硬件协议 |
| 世代 CO 范围 −30…+30 / −50…+50、Curve Shaper −50…+30 钳制 | Verified 源码存在 / Inferred 作为官方事实（AMD 未发布） |
| RyzenAdj 响应三态分类、Strix Halo 0xB7 被拒 | Verified（源码） |
| 对本平台 APOB 可读性、全核可写性、界限来源 | Unverified（无硬件/无原始日志；用户线索 = User-provided） |

本参考只维护"上游怎么做"，不维护参数权威定义（定义在基础规范）；不修改产品源码、不审查 Next。
