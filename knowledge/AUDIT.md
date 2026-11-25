# 专项技术事实审计记录（AUDIT）

- 审计日期：2026-11-25
- 范围：`knowledge/` 规范群 + `research/` + `decisions/` + `project/` 全文档
- 目标：找出错误、过度概括、数值范围错误、单位错误、概念混淆；修正有充分依据者；不增加无意义文档
- 方法：内置联网检索 + 权威原文交叉核验（AMD / Intel / Microsoft 官方、维护者源码、专业社区）
- 说明：首轮用户指定的 /deep-research-pro、/tavily、/web-search-exa skill 当时未注册，故用等效检索完成；第二轮（见 §5）已用联网检索 + 厂商原文交叉核验。
- 第二轮复审：2026-11-25（承接首轮，聚焦首轮未发现的错误与用户新增实测证据）
- 第三轮专项审计：2026-11-25（用户设备实测数据真实性 + 外部资料研究质量；优先级高于新增知识）

## 1. 审计发现与处理（首轮）

| # | 原文（位置） | 问题 | 正确表述 | 证据来源 | 影响文件 / 处理 |
|---|---|---|---|---|---|
| 1 | `amd-curve-optimizer.md`："常见 BIOS 范围 约 -30 到 +30 (Inferred)"；"±50 是旧代码输入侧钳制、工具实现选择，非官方范围" | 把典型调校值当固件范围；把与固件一致的钳制说成工具自创 | 固件/AGESA 一般约 **-50…+50**（负向常达 -50，正向多数 BIOS +30）；旧版 Zentwik 钳制 -50…+50 与固件一致、非官方定义但非凭空 | vectree.io（老 Zen3/早 Zen4 停 -30、当前 AGESA 常达 -50）、SMUDebugTool 源码、vgoodslab 的 -50 实操、`research/02` §2.3 | 已改正文第 2/5/12 节 + INDEX + GLOSSARY；并按五层结构重写为权威定义 |
| 2 | knowledge 中**完全缺失** FMax 偏移范围 | 遗漏关键参数；且易与 FMax 绝对频率混淆 | Boost Clock Override / Fmax Override：**-1000…+200 MHz、25 MHz 步进**，Zen2 起；是**频率上限**非频率偏移；三者（FMax 绝对频率 / Override 偏移 / 实际运行频率）不得画等号 | SkatterBencher（PBO+ 原文 `-1000 MHz and +200 MHz in steps of 25 MHz`）、MSI 官方博客、`research/01` 的 FMax 绝对频率 4450–5650 MHz | 已在 `amd-smu.md` §4 建立分层权威定义，补 GLOSSARY/INDEX/units/README |
| 3 | `units.md`："功率 RAPL 单位来自 0x606 bits[3:0]，**2^unit W/LSB**，缺省 1/8 W" | **方向性单位错误**：写成 2 的正幂，实际是负幂，差若干数量级 | 功率 `1/2^PU W/LSB`（PU=bits[3:0]，缺省 3→1/8 W）；能量 `1/2^ESU J/LSB`（bits[12:8]，缺省 16→15.3 µJ）；时间 `1/2^TU s`（bits[19:16]，缺省 10→976 µs） | Intel SDM Vol.3B §14.10.1；kernel-internals.org；HardwareVisualizer RAPL 规范；IT4I 课件 | 已改 `units.md` §1/§3/§4；能量行补齐 |
| 4 | `research/02` §3.3："时间窗[23:17]，单位取自 `0x614` bits[3:0]（2^-unit，缺省 1/8 W）" | **量纲错误**：把功率单位错套到时间窗；`0x614` 是功率信息 MSR，不含时间单位 | 时间窗来自 `0x606` 的 Time Units（bits[19:16]，1/2^TU s，缺省 976 µs），编码 `Time=2^Y×(1+Z/4)×Time_Unit` | Intel SDM Vol.3B §14.9.3（Figure 14-32）、pepc-power.rst、xem.github.io SDM 镜像 | 保留原始历史描述，加【审计注记】指向 `units.md`；未删 research 原文 |
| 5 | `process-scheduling.md`（及 `research/02` §5）："`Type` 字段是 2 字节，按 4 字节读导致拓扑恒空" | 与官方结构冲突：`SYSTEM_CPU_SET_INFORMATION.Type` 是 `CPU_SET_INFORMATION_TYPE` 枚举，底层 4 字节 DWORD | 该 2 字节说法仅作"本仓库旧版实现现象"保留，**不是 Windows API 事实**；真机复核前 Unverified | Microsoft Learn `SYSTEM_CPU_SET_INFORMATION (winnt.h)` | 加【审计注记】，标注冲突与 Unverified；未改 research 原文 |
| 6 | 多份文档孤立记录数值范围，不说明属于哪一层 | **根本性问题**：软件钳制/典型值/固件限制混用，改一处无法防下一处重犯 | 建立五层参数记录规范（L1 固件 / L2 软件钳制 / L3 实际稳定 / L4 典型调校 / L5 项目约定） | 本次审计归纳 | 新增 `data/parameter-record-schema.md`；CO、FMax 改为分层权威定义；写入 `_TEMPLATE.md`、`AI_GUIDE.md` |

## 2. 复核后确认无误、不改的内容

- **Intel OC Mailbox 0x150**：11 位有符号 S11.0.10、`code=mV×1.024`、字段 bit[31:21]、控制字 `0x80000[plane]1[R/W]` —— 经 `mihic/linux-intel-undervolt` 原文确认，与现有文档一致。
- **Windows CPU Sets**：软亲和性、affinity mask 优先、多组不匹配忽略、Process Default 仅影响未显式设集合的线程 —— Microsoft 原文确认，一致。
- **Windows 电源 API**：写值须 `PowerSetActiveScheme` 激活、`PowerEnumerate` 16/17/18 与 259(ERROR_NO_MORE_ITEMS)、RootPowerKey 为 NULL —— MS Learn 确认，一致。
- **GPU / DDR5 内存**：均为待研究索引，无编造数值，保持 Unverified，不补写。
- **IPC / 持久化 / ADR**：错误码、SchemaVersion、原子替换等为项目约定或决策，已正确标注 Proposed/Verified，未发现范围或单位错误。

## 3. 单一维护点（权威定义）

| 参数 | 权威位置 |
|---|---|
| AMD Curve Optimizer offset | `hardware/cpu/amd-curve-optimizer.md` §5 |
| FMax / Boost Clock Override | `hardware/cpu/amd-smu.md` §4 |
| Intel OC Mailbox 电压偏移 | `hardware/cpu/intel-oc-mailbox.md` §4 |
| RAPL 功率/能量/时间单位 | `data/units.md` §1 |
| 参数分层规则 | `data/parameter-record-schema.md` |

其余文件只引用，不复制数值表。

## 4. 未核验 / 待办

- `0x610` PL1/PL2/tau 曾被实现后移除，移除理由无记录；时间窗编码已在本次更正，但旧版实现是否用错单位需查提交 `84bb998` 源码确认（Unverified）。
- `SYSTEM_CPU_SET_INFORMATION.Type` 的 2 字节说法与官方 4 字节枚举冲突，需真机/源码复核（Unverified）。
- 不同 CPU/AGESA/BIOS 下 CO 正向范围（+30 vs +50）的确切边界（Inferred）。

## 5. 第二轮复审（2026-11-25）—— 首轮未发现的问题与用户新增实测

首轮把 CO "固件/AGESA 档位范围 −50…+50" 标为 Verified，并经本次复审确认这是**证据等级错误 + 适用范围错误**：它把 Ryzen Master 的显示钳制、第三方工具的输入上限与固件实际接受范围混为一谈，并把 CO 描述成对称范围。用户新增的一手实测（SMU Debug Tool 负向 −50、正向 +30）进一步证明范围不对称。本轮修正如下。

| # | 原文 / 位置 | 问题性质 | 正确表述与处理 | 证据来源 |
|---|---|---|---|---|
| 7 | 首轮 AUDIT #1 与 `amd-curve-optimizer.md` §2/§5："固件/AGESA 一般约 −50…+50"、"固件范围 Verified（范围本体）" | **证据等级错误 + 适用范围错误 + 数值语义**：把显示/工具范围当固件范围；把 CO 写成对称 ±50 | AMD 官方从未发布 CO 固件命令档位范围；Ryzen Master 的 +30/−50 是**显示钳制**（实际可设定并应用超出该范围的值）；SMU Debug Tool CO 输入框是 ±999、−50…+30 是其 Curve Shaper 说明。CO 范围**不对称**，固件范围标 **Unverified/官方未发布**。已重写 §2/§5/§8/§10/§11/§12 + INDEX + GLOSSARY | MSI FAQ-10122（厂商官方）、AMD Ryzen Master CO FAQ PDF、SMU Debug Tool `SettingsForm.cs` 源码、本机实测 |
| 8 | `amd-curve-optimizer.md`：Curve Shaper 仅一句"按温度/频段分点微调"，无结构与来源分层 | **文档结构 / 证据不足**：与 CO 混淆风险高 | 补记 Curve Shaper 为**独立参数**（5 频段×3 温段=15 点，Inferred），不与 CO 共用数值表；其 −50…+30 表述标注为工具说明文字、Unverified | 多家社区资料 + SMU Debug Tool `labelCsInfo` |
| 9 | 用户新增实测："SMU Debug Tool 负向 −50、正向 +30"此前未入任何文档 | **产品实现与文档不一致（缺记录）** | 作为本机、本工具版本的一手实测写入 §5/§12，标 Verified 实测；**界限来源（工具钳制/固件/平台）明确标 Unverified** 并给出工具无关直写验证方法，不与跨平台资料混为一谈 | 用户提供的一手实测 |

### 5.1 本轮复核后确认无误、不改的内容（针对首轮结论）

- **RAPL 单位 1/2^n 及缺省值**：Intel SDM Vol.3B §14.9.2 / Figure 14-31 原文确认 PU=1/2^PU（缺省 0011b→1/8 W）、ESU=1/2^ESU（缺省 10000b→15.3 µJ）、TU=1/2^TU（缺省 1010b→976 µs）；首轮 §3 方向性修正正确，不改。
- **FMax / Boost Clock Override −1000…+200 MHz / 25 MHz**：SkatterBencher #78/#92 原文确认"override the arbitrary clock frequency limit between −1000 MHz and +200 MHz in steps of 25 MHz"、"only adjusts the upper ceiling...doesn't act as a frequency offset"；Zen5 工具侧 1 MHz 粒度一致。首轮 §2 修正正确，不改。
- **Intel OC Mailbox 0x150**：IACR ePrint 2023/1679 与 Plundervolt 论文确认 11 位有符号偏移在 MSR 0x150 bits[31:21]、单位 1/1024 V、plane 索引 bits[42:40]（core0/GPU1/cache2/uncore3/analog4）；LKML 明确无公开架构文档。现有文档一致，不改。
- **`SYSTEM_CPU_SET_INFORMATION.Type` 为 4 字节**：Microsoft Learn winnt.h 结构确认 `DWORD Size` + `CPU_SET_INFORMATION_TYPE Type`（枚举，4 字节）。首轮 §5 审计注记正确，不改。
- **GPU / DDR5 待研究索引**：无可靠资料，保持 Unverified、不编造数值，符合"不补写"原则，不改。

### 5.2 本轮仍为 Unverified 的事项

- CO 固件/AGESA 命令层的真实档位边界（官方未发布；本轮由 Verified 降级）。
- 本机 SMU Debug Tool −50/+30 界限的来源：工具钳制 / 固件邮箱接受范围 / 平台实现，三者尚未分离。
- Curve Shaper 的官方单位定义、命令号与 Zen5 固件档位范围。
- 首轮 §4 遗留项（`0x610` 移除理由、`Type` 真机复核、`0x150` 锁定回读行为）保持 Unverified。

---

## 6. 第三轮专项审计（2026-11-25）——实测数据真实性 + 外部资料研究质量

用户最高优先级要求：以"出厂 −11…−26"为切入点，审计所有本机实测数据的真实性，并审查此前修改的外部研究过程是否可靠。本轮的结果**优先于扩充知识**。

### 6.1 实测数据来源审计（发现的问题）

| # | 位置 | 原表述 / 问题 | 性质 | 处理 |
|---|---|---|---|---|
| 10 | `research/02` §11 与 `amd-curve-optimizer.md` §8/§12 | "本机 14 个可读核出厂即为非零（-11…-26）"，后者标 Verified 实测 | **无来源 + 设备状态混淆 + 证据等级虚高**：找不到原始日志；同会话记载了核 0 写入 `-22→-10`，不满足出厂状态；归因提出者与依据不可追溯 | **撤回"出厂"归因**，保留数值作历史记录标 Unknown；仅保留"读到 0 ≠ 默认状态"的逻辑结论；见 `research/02` §11 审计注记、`amd-curve-optimizer.md` §8/§12 |
| 11 | `research/02` §2.3/§11、`amd-smu.md` §3/§4/§9 | 写入/回读、权限、bit23 部分核被拒标"Verified（本仓库实测）" | **证据等级虚高**：仓库 55 个文件全为 Markdown，无任何原始日志/ledger/result 文件；回读一致不等于效果生效 | 降级为"报告称 / 来源 Unknown（原始日志缺失）"；回读语义边界写入新规范 §6 |
| 12 | 用户 SMU Debug Tool −50/+30（第二轮写入） | 标"Verified 实测（本机）" | **数据来源类别错误**：这是用户自述，仓库无原始输出、无工具版本、未独立复核 | 改标 **User-provided**（用户提供，未独立复核），不升格为 Direct observation |
| 13 | `research/02` §6、`research/03` §2.1、sensors/numeric-semantics | PDH 约 4000/s、std 21–35pp、15.6ms 等标"Verified 实测" | **证据等级虚高**：有采集方法叙述但无原始日志 | 具体数字降为"报告称 / Unknown"；分档结论作为设计原则独立保留（由文档化机制支持） |

**追溯结论**：`-11…-26` 与"14 个可读核"在 `git log -S/-G` 中可追溯到初始提交 `aa1c8de`（原文即如此，无附证），其后 `80a3275`、`57586bc` 未新增该数据的原始来源。**找不到原始数据来源**；"出厂"是首次写作时的归因，依据不可查，判定为无据归因。

### 6.2 外部资料研究质量审计（本轮实际访问并逐字核对的原始资料）

| 资料 | 类型 | 本轮访问情况 | 定位 | 支持的结论 |
|---|---|---|---|---|
| [MSI FAQ-10122](https://www.msi.com/faq/10122) | 厂商官方 FAQ | **实际打开，逐字引用** | 问答正文 | Ryzen Master 显示钳制 +30/−50，实际可设并应用超出该值 → 显示范围 ≠ 固件范围 |
| SMU Debug Tool `SettingsForm.cs`（irusanov） | 维护者源码 | **实际下载 3609 行并定位** | `SettingsForm.cs:352-365,507,808-809` | CO 输入框 ±999（808-809）；Curve Shaper 5 频×3 温=15 点（352-365）、说明文字"Range -50 to +30"（507） |
| [SkatterBencher #78](https://skatterbencher.com/2024/08/07/skatterbencher-78-ryzen-7-9700x-overclocked-to-5860-mhz/) | 专业社区 | **实际打开，逐字引用** | Fmax Boost Override 段 | -1000…+200 MHz / 25 MHz、仅调上限非偏移；该页无"50 steps"表述（该表述在 #69） |
| [Microsoft Learn SYSTEM_CPU_SET_INFORMATION](https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-system_cpu_set_information) | 官方开发者文档 | **实际打开，逐字引用** | Syntax 段 | `DWORD Size` + `CPU_SET_INFORMATION_TYPE Type`（枚举） |
| Intel SDM Vol.3B Figure 14-31（[xem 镜像](https://xem.github.io/minix86/manual/intel-x86-and-64-manual-vol3/o_fe12b1e2a880e0ce-498.html)） | 官方手册镜像 | **实际打开，逐字引用**；Intel 官方 PDF 直链返回 Access Denied | 14-32 页、Figure 14-31 | 功率 1/2^PU W（默认 0011b=1/8）、能量 1/2^ESU J（默认 10000b=15.3µJ）、时间 1/2^TU s（默认 1010b=976µs） |
| [mihic/linux-intel-undervolt README](https://raw.githubusercontent.com/mihic/linux-intel-undervolt/master/README.md) | 社区逆向工程 | **实际打开，逐字引用** | 全文 Disclaimer + Calculating/Setting 段 | 11 位、1/1024 V、plane 位；**原文明确"无官方 MSR 文档、逆向工程、仅在一台 i7-6700HQ 上测试"** |

#### 6.2.1 缺少外部依据 / 被证伪 / 仅为推断的内容

- **FMax / Boost Clock Override −1000…+200 MHz / 25 MHz**：此前标"Verified（SkatterBencher / MSI 官方）"。本轮核对 SkatterBencher 原文确认数字正确，但来源是**专业社区**，非 AMD 官方规范；且"每档 = N mV / up to 50 steps"的说法仅在另一篇（#69），不能张冠李戴。**降为 Inferred（社区原文已核对，非官方规范）**，见 `amd-smu.md` §9。
- **Intel 0x150 编码**：此前 AUDIT §2 称"经 linux-intel-undervolt 原文确认"。本轮打开原文发现作者自述**逆向工程、仅单机测试、无官方文档**，因此只能标 Inferred（社区逆向口径），不能当官方事实；此前 AUDIT §5.1 声称"IACR ePrint 2023/1679 与 Plundervolt 论文确认 plane 索引 bits[42:40]"——本轮未重新打开该论文逐字核对，**不维持 Verified，标 Inferred/待核**。
- **AMD Ryzen Master 产品页/Release Notes、vectree.io、G-Helper**：此前来源表标"已访问 2026-11-25"，本轮**未重新打开原文**，保留引用但核验状态维持其原级别，不新增 Verified 结论。
- **Curve Shaper 5×3=15 点**：此前标 Inferred。本轮从工具源码 352-365 行逐字确认结构，升级为 **Verified（工具源码事实，非 AMD 官方协议）**。
- **RAPL 单位方向（1/2^n）**：本轮经 Intel SDM 镜像 Figure 14-31 逐字复核，确认首轮修正正确。Intel 官方 PDF 直链本轮无法访问（Access Denied），据实说明。

#### 6.2.2 对自身此前修改的复核

- 第二轮由本 Agent 写入的"SMU Debug Tool −50/+30 = Verified 实测"属于**来源类别错误**（见 §6.1 #12），本轮已改 User-provided。这是此前修改自身的问题，非首轮遗留。
- 第二轮写入的"Curve Shaper 5×3=15（Inferred）"经本轮源码核对，证据充分，升级 Verified。
- 第二轮对 CO 范围不对称、显示钳制 ≠ 固件范围的核心修正，经 MSI FAQ 本轮重开逐字核对，**成立**。

### 6.3 新增规范与文件

- 新增 [measurement-evidence-schema.md](data/measurement-evidence-schema.md)：来源七分类、设备状态分类、必填字段、禁止归因清单、回读语义、结论—证据映射表、案例。
- 更新 AI_GUIDE/INDEX/GLOSSARY：把"实测数据真实性"纳入检索与产出规则。

### 6.4 需要用户补充实验记录的事项（本轮无法自行完成）

1. `-11…-26` 与 14 个可读核的**原始日志/工具输出**（若有），以及当时的读取工具与版本、接口、时间、CPU 身份。
2. 读取前是否做过 BIOS Load Optimized Defaults、是否曾写入 CO——以判定设备状态。
3. SMU Debug Tool 的确切版本号与原始输出，以把 User-provided 升级为 Direct observation。
4. 工具无关直写测试（+31…+50 是否被 SMU 接受）——判定 −50/+30 界限来自工具还是固件。
5. 未提供真实硬件前，bit23 掩码扫描、休眠恢复、Intel 平台 MSR 行为均无法实测，保持 Unverified，不伪造实验。

本轮未修改旧版产品 `main`，未触碰产品仓库，未 force push。
