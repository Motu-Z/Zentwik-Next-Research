# AMD Curve Optimizer（曲线优化器）

- 规范名称：Curve Optimizer（CO）/ Curve Shaper
- 英文与官方名称：Curve Optimizer；属于 Precision Boost Overdrive（PBO）的一部分；Curve Shaper 为 Zen 5 引入的后续特性
- 缩写：CO、PBO
- 领域：`hardware/cpu`
- 核验状态：混合（见第 12 节分项标注）
- 权威来源（访问 2026-11-25）：
  - AMD Ryzen Master 产品页与 Release Notes（amd.com）：确认 CO/PBO 参数存在、可通过 Ryzen Master 或 BIOS 调整、重启保持需 BIOS 设置。[Ryzen Master](https://www.amd.com/en/products/software/ryzen-master.html)、[Release Notes 2.14.2.3341](https://amd.com/en/resources/support-articles/release-notes/RN-RYZEN-MASTER-2-14-2-3341.html)
  - AMD PPR（Processor Programming Reference）：公开文档**不提供 MP1/SMU 消息表与 CO 寄存器协议**（Inferred，依据社区工具链一致口径与 UCM 研究）
  - 社区与维护者资料：SkatterBencher（每档约 3–5 mV 的经验换算）、RyzenAdj / RyzenSMU.p 上游源码、laptopjudge 技术整理
  - 本仓库研究记录（原标"一手实测"，原始日志缺失、来源 Unknown，见第三轮审计）：`research/02-hardware-access.md` §2、§11
  - 实测记录字段与来源分类规范：[measurement-evidence-schema.md](../../data/measurement-evidence-schema.md)
- 相关文档：[numeric-semantics.md](../../data/numeric-semantics.md)、[units.md](../../data/units.md)、[amd-smu.md](amd-smu.md)、[intel-oc-mailbox.md](intel-oc-mailbox.md)、[implementation-references.md](implementation-references.md)（成熟工具怎么做 APOB/掩码/命令分派，实现参考非规范）

## 1. 准确定义（外部技术事实）

Curve Optimizer 是 AMD 在支持 PBO 的 Ryzen 处理器上提供的一项调校功能，允许对处理器**每条电压-频率（V/F）曲线**施加一个**逐核或全核的有符号偏移**。负值使核心在给定频率下请求更低电压，从而在 PPT/TDC/EDC 与温度预算内为 Precision Boost 留出更多升频空间；正值用于稳定较弱的核心。

它是曲线的**偏移量**，不是固定电压、不是倍频上限、不是功耗墙。CO 与 PBO Limits（PPT/TDC/EDC）、PBO Scalar、Max Boost Override 是不同的控制项。

## 2. 适用范围

- 硬件：支持 PBO 的 AMD Ryzen / Threadripper；CO 逐核设置通常在 Zen 2 及以后、经 AGESA/BIOS 暴露。具体可用性与范围由 CPU 型号、主板 BIOS 和 AGESA 决定（Verified：AMD FAQ 指出 PBO 需兼容型号与主板）。
- **档位范围不对称，不能写成"±50 对称"**：AMD 官方未发布固件/AGESA 命令层的档位范围。观测到的负向常达 −50（个别平台 −60），正向通常更小（Ryzen Master 显示上限 +30）。显示上限、工具输入上限与固件/邮箱接受范围三者必须分开（见第 5 节分层记录）。
- Zen 5 增加 Curve Shaper，与 CO 并存但属于**不同控制项、不同参数**，不得与 CO 共用数值表或单位约定（Verified 存在性：AMD Release Notes；结构细节见下）。
  - 结构：**5 个频段 × 3 个温段 = 15 个可调点**，叠加在 CO 之上做分点微调（Verified：SMU Debug Tool 源码 `SettingsForm.cs:352-365` 的 `CsTierNames` 5 项、`CsBandNames` 3 项、`CsControls[5,3]`；这是**工具源码事实**，AMD 未发布官方命令/单位定义）。
  - SMU Debug Tool 的 Curve Shaper 说明文字标注范围 −50…+30，但这是**该工具对 Curve Shaper 的表述**，既不是 CO 的范围，也未核验为 Zen5 固件范围（Unverified）。
  - 命令号与编码（**Verified 上游源码事实**，`irusanov/ZenStates-Core` `Zen4Settings.cs` / `SetCurveShaperMargin.cs`，本轮逐行核对）：Zen4/5 RSMU `SetCurveShaperMargin=0xA6`、`GetCurveShaperMargin=0x84`；写入参数 `marginHigh<<24 | marginMedium<<16 | marginLow<<8 | (1<<7) | frequencyTier&0x7F`，`frequencyTier` 限 0–4；`EncodeMargin` 把每个 margin 钳到 [−50,+30] 并存为 `sbyte`。**这正是 SMU Debug Tool 说明文字"−50…+30"的源码级来源——它是 Curve Shaper 的钳制，不是 CO 范围**。作为 Zen5 官方固件协议仍为 Inferred（AMD 未公开）。详见 [implementation-references.md](implementation-references.md) §2/§3.6。

## 3. 技术行为

1. CO 作用于 SMU 内部的 V/F 曲线，在 SMU 计算每个核心目标电压时叠加偏移，再转换为各核参考码（Inferred：内部数据路径 AMD 未公开）。
2. 偏移沿曲线生效，**并非每个频率点固定减去相同 mV**（Inferred，社区口径）。
3. 写入链路（外部工具）：经 PawnIO 加载 `RyzenSMU.bin`，通过 SMU 邮箱发送 `SetMargin` / `GetMargin` 命令，逐核写入并立即回读校验（协议链路 = Inferred 逆向口径；本仓库曾报告一次写入/回读过程，但原始日志缺失、来源 Unknown，见 `research/02-hardware-access.md` §2.3、§11 第三轮审计注记）。回读一致不等于效果生效，见 [measurement-evidence-schema.md](../../data/measurement-evidence-schema.md) §6。
4. 断电/休眠后是否保持：旧版假设 S3/S4 恢复后 CO 被清回出厂，该假设**未验证**，S3 通常不断电，不得当作事实（Unverified）。

## 4. 数据类型

- 概念类型：**有符号整数步进值（signed integer step / count）**，无量纲。
- 硬件协议中，偏移量随核心掩码一起放在低 16 位（负值以补码表示），见 `research/02-hardware-access.md` §2.2。
- Zentwik 内部约定（Proposed）：使用独立强类型 `CurveOffsetStep`（`i8`/`i16`），禁止与电压类型 `VoltageMillivolt` 互相隐式赋值。

## 5. 单位及取值范围（本参数的分层权威定义）

> 本节是 CO 参数在仓库内的**单一权威定义点**，其余文档只引用不重抄。分层规则见 [parameter-record-schema.md](../../data/parameter-record-schema.md)。

**参数记录**：Curve Optimizer offset
**语义**：Voltage/Frequency Curve Offset（逐核 V/F 曲线偏移） · **数据类型**：有符号整数（step/count） · **单位**：无（不是 mV）

| 层 | 取值 | 说明 / 来源 / 状态 |
|---|---|---|
| L1 固件/AGESA 实际允许 | **AMD 官方未发布**；观测负向常达 −50、正向多为 +30，二者不对称 | 固件接受范围无官方定义，只有工具/ BIOS 侧观测。**Inferred**（不得标为官方定义） |
| L1 Ryzen Master **显示**上限 | +30 / −50 | MSI 官方 FAQ（FAQ-10122）：Ryzen Master 显示的 Core Value 限定在 +30/−50，但**实际可设定并应用超过该显示范围的值**。这是显示/呈现层钳制，**不等于固件或邮箱接受范围**。Verified（厂商文档） |
| L2 软件输入钳制 | 旧版 Zentwik 输入侧 -50…+50 | `research/02` §2.3；本仓库实现选择，非固件事实。Verified |
| L2 第三方工具上限 | SMUDebugTool CO 输入框 ±999；G-Helper [-40,0] | 各工具源码/文档；软上限，非固件限制。Verified（工具事实）。见下方"SMU Debug Tool 局限归属" |
| L3 实际稳定可用 | 取决于 CPU 个体、散热、电压与负载 | 无通用数值，强核常只能 -5…−15，弱核可达更负。**Unverified，不填规范限制** |
| L4 典型调校值 | 社区常见 -10…−30 起点 | **仅作参考，非规范限制、非官方推荐** |
| L5 Zentwik 约定 | 字段 `coOffset`、类型 `CurveOffsetStep`（i8/i16）、单位 `step`、不持久化 mV | Proposed，见 §9 |

明细与证据：

| 项 | 取值 | 核验状态 |
|---|---|---|
| 单位 | 档（step / count），**无单位**，不是 mV | Verified（社区与工具链一致；AMD 官方未发布单位定义，属外部事实缺失） |
| 固件/AGESA 命令层档位范围 | **未知**：AMD 公开文档无 MP1/CO 命令参数表，官方 FAQ 也不给档位边界 | **Inferred / Unverified**。多源工具链观测负向可到 −50、Ryzen Master 显示正向到 +30，但这些是显示/工具层观测，不能声称是 AMD 发布的固件范围 |
| Ryzen Master 显示值域 | +30（正）/ −50（负），**显示钳制**，非固件上限 | Verified（MSI 官方 FAQ-10122，2026-11-25 访问）。厂商明确"实际可设定并应用超过 +30/−50 的值" → 显示范围 ≠ 固件范围 |
| 本仓库工具写入钳制范围 | -50 到 +50（旧代码输入侧钳制） | Verified（`research/02-hardware-access.md` §2.3）；本仓库实现选择，不宣称等于固件范围 |
| 第三方工具软上限（非固件限制） | SMUDebugTool CO 输入框 ±999（源码 literal，无命名常量）；G-Helper [-40,0] | Verified（工具源码/文档）；工具自身上限，**不得当作固件事实** |
| 等效电压 | 社区经验约 3–5 mV/档，且随频率点非线性变化 | Inferred，**禁止作为官方事实或换算公式写入产品契约** |

**SMU Debug Tool 局限归属（用户提供，待区分来源）**：用户通过其本机 SMU Debug Tool 使用 CO，**报告负向可设到 −50、正向只能到 +30**（来源类别 = **User-provided**：用户自述，未附原始日志或工具版本，本仓库未独立复核，故不标 Direct observation / Verified 实测）。这是一个**不对称**范围，进一步说明 CO 不能描述为对称的 ±50。该 ±50/+30 界限的来源**尚未确定**，不得混为一谈：

1. **不是该工具 CO 输入框的钳制**：源码 `SettingsForm.cs` 中 `numericUpDownCO_{i}` 的 `Minimum=-999 / Maximum=999`，Apply 路径无额外钳制（Verified：源码）。
2. **−50…+30 在源码中出现在 Curve Shaper 说明文字**（`labelCsInfo`），而 Curve Shaper 是**另一个参数**（见下），不能拿来解释 CO 的 ±50/+30。
3. 可能来自**固件/平台对该邮箱命令的接受范围**，也可能来自**该工具版本在写入侧的额外钳制**。二者需要分别验证：先用本仓库写入链路直接向 SMU 发 SetMargin，测试 +31…+50 是否被接受并回读一致（工具无关），再对照同机同工具的输入框行为。来源定性前标 **Unverified**。

> **关键结论 1**：AMD 从未发布"每档 = N mV"的官方换算。3–5 mV/档是社区实测经验值，随平台、频率点、温度而变。**任何把 CO offset 标注为 mV 的行为都是错误。**
>
> **关键结论 2**：CO **档位范围不对称**，不能写成对称的 "±50"。AMD 从未发布固件命令层的档位范围；Ryzen Master 的 +30/−50 是**显示钳制**（MSI FAQ-10122 明确实际可设定并应用超出该范围的值），第三方工具的输入上限（±999、[-40,0]）是**工具选择**。把显示/工具范围当成固件范围，或把负向与正向说成对称，都是错误。
>
> **易混淆点（务必区分三个维度）**：**档位范围**（负向常见 −50、正向常见 +30，不对称）是离散计数值，回答"能设多少档"；**每档 mV 等效**无官方固定值，回答"每档多少电压"；**显示/工具钳制**只是呈现或输入层限制，不是固件上限。三者互不替代，更不要把"−50 档"误读成"−50 mV"。

## 6. 转换规则

- 正确的换算只有一种方向在项目中被允许：**步进值 → 写入参数**，由 provider 完成。权威编码（**Verified 上游源码**，`ZenStates-Core` `Utils.cs:180-184` / `SetPsmMarginSingleCore.cs`）：`margin16 = margin<0 ? (0x100000+margin)&0xffff : margin`，`args[0] = (coreMask & 0xfff00000) | margin16`。即**掩码占高 12 位、margin 占低 16 位（16 位补码）、[19:16] 保留**；本轮源码确认此前"低 16 位 vs 低 20 位"歧义（见 [implementation-references.md](implementation-references.md) §3.2）。作为芯片硬件协议仍为 Inferred（AMD 未公开）。
- **禁止**提供 `Step → mV` 的确定性转换函数。若 UI 需要示意，只能标注为"约 3–5 mV/档，仅供参考，非官方换算"（Inferred）。
- 符号：负值为降压（undervolt），正值为加压。补码负值在写入低 16 位时必须保持符号扩展正确。

## 7. 关联术语

PBO、PPT、TDC、EDC、Precision Boost、V/F curve、Curve Shaper、SMU Mailbox、SetMargin（0x06）/ GetMargin（0xD5）、CoreMask、Preferred Core（CPPC）

## 8. 常见误解及错误用法

| 误解 | 纠正 | 依据 |
|---|---|---|
| "CO offset 单位是 mV" | 是步进档位，无单位；mV 等效值不固定且未公开 | Verified（社区一致）+ 本仓库纠错记录 |
| "-20 就是降 20 mV" | 不成立；3–5 mV/档只是粗略经验，非线性 | Inferred |
| "写 0 = 恢复默认" | 写入 0 的实际行为（**Verified 源码**）：向 SMU 发 `SetMargin`（`SetDldoPsmMargin`），`margin=0`，把该核 CO 偏移设为 **0 档，即无偏移**（`args[0]=(mask&0xfff00000)|0`，`MakePsmMarginArg(0)=0`；见 [implementation-references.md](implementation-references.md) §3.2）。工具的"Zero all"按钮就是把所有启用核设为 0，其基线是硬件读回值（`ResetToBaseline`），不假定某个非零出厂值。**在未调校/默认状态下，无偏移（0）就是中性默认值**，因此"写 0 = 回到 CO 无偏移的中性状态"成立。需注意：**"恢复到某台曾被人改过（含 OEM 预设）的既有配置"不等于写 0**——但"厂商出厂普遍预置负 CO"没有证据，不能当作常态假设。"恢复默认"若特指 BIOS 的 Load Optimized Defaults，那是整套 BIOS 操作，与单次 CO 写 0 不是一回事 | 写入行为/中性值 0 = Verified 上游源码（设置 0=无偏移）；默认态无偏移 = Inferred（AMD 未发布"出厂即负偏移"的规范）；见 `research/02` §11 第三轮注记 |
| "读到 0 = 默认状态" | 这是**读取语义**，与上一条"写入 0 的行为"是两个独立命题，不得互相替代：读到 0 说明当前该核偏移为 0（与"未施加偏移"一致），但仅凭一次读取仍不能证明设备从未被改过或处于 BIOS 出厂配置（那需要排除用户修改后的带日志读取）。反过来，读到非零也不等于"出厂预设" | 逻辑结论，见 [measurement-evidence-schema.md](../../data/measurement-evidence-schema.md) §3/§6 |
| "所有核都必须能读写" / "部分核被平台拒绝" | 不能用早期报告的"bit23 被拒"当作平台限制：该观测来自**错误的独热位编码** `1<<(20+local)`，与外部权威实现 `(core%8)<<20` 不符；应改用正确数值编码 + APOB 映射逐核验证 | 编码错误 = Verified（外部源码逐字核对，见 [amd-smu.md](amd-smu.md) §3.1）；"平台拒绝 bit23" = **已撤回**；用户报告其他工具可改全核 = User-provided；见 `research/02` §11 第四轮审计注记 |
| "命令号与固件无关，可全平台通用" | 命令号与 CPU 世代/型号强绑定；家族白名单误分类会发错世代命令 | Verified，`research/02` §2.2/§10 |
| "回读相同即写入成功" | 这是必要条件；仍需观测效果层（频率/功耗/温度）以排除钳位 | Inferred，`research/02` §8.3 |
| "CO 是 ±50 对称范围" | 不对称：负向常见 −50、正向常见 +30；且这些是工具/显示观测，非 AMD 发布的固件范围 | MSI FAQ（Verified 厂商文档）+ 用户提供实测（User-provided，未独立复核），见 §5 |
| "Ryzen Master 显示到 +30/−50 就是固件上限" | 显示钳制 ≠ 固件上限；MSI FAQ 明确实际可设定并应用超出显示范围的值 | Verified（MSI FAQ-10122） |
| "SMU Debug Tool 的 −50…+30 来自 CO 输入框" | 该工具 CO 输入框是 ±999；−50…+30 出现在 Curve Shaper 说明，属另一参数 | Verified（工具源码） |

## 9. Zentwik 内部约定（Proposed）

1. 字段统一命名 `coOffset`，类型 `CurveOffsetStep`，**不得**命名为 `millivolts`/`mv`。
2. CLI 打印格式：`core <n>  co <step>`，**不得**打印 `offset(mV)`。早期实现曾犯此错并已纠正（`research/02` §11）。
3. 能力状态区分 `Available` / `NotSupported` / `DriverUnavailable` / `ProbeFailed`；逐核失败时标记 `degraded` 并报 `error`，不阻塞其余核。**但失败必须先排除编码错误**：核心掩码按 [amd-smu.md](amd-smu.md) §3.1 的数值编码 `(ccd<<28)|((core%8)<<20)` 实现，不得用独热位 `1<<(20+local)`；否则会把实现缺陷误报为"平台部分可用"。
4. 方案引用了读不到前值的核时记 `Skipped` 且不写入。
5. 持久化必须记录来源（SMU 读出 / 用户输入 / 推断）、采集时间、CPU 身份（family/model/stepping/微码）；不得把"上次写入值"当出厂值（对应 `project/ARCHITECTURE_CONSTRAINTS.md` §10）。
6. `PlanApplyRecord.PersistsAcrossReboot` 对 AMD CO 为 `true`（ADR-005）。

## 10. 权威资料来源与访问状态

| 来源 | 类型 | 访问状态 | 用途 |
|---|---|---|---|
| AMD Ryzen Master 产品页 / Release Notes | 官方 | 已访问 2026-11-25 | CO/PBO 特性存在性、参数调整方式 |
| MSI FAQ-10122（Curve Optimizer Magnitude vs Ryzen Master 显示） | 厂商官方 FAQ | 已访问 2026-11-25 | Ryzen Master 显示钳制 +30/−50，实际可设定并应用超出该范围；证明显示范围 ≠ 固件范围 |
| AMD FAQ — Curve Optimizer Feature in Ryzen Master（Build 2.9.0.2093） | 官方 PDF | 已访问 2026-11-25 | CO 定义、All Core/Per Core、验证方式；官方 FAQ 未给固件档位边界 |
| SMU Debug Tool 源码 `SettingsForm.cs`（irusanov 仓库镜像） | 维护者源码 | 已访问 2026-11-25；第四轮逐行核对 `2562-2584` | CO 输入框 ±999（无命名常量）；−50…+30 出现在 Curve Shaper 说明文字；`EncodeCoreMarginBitmask` 数值编码 `(ccd<<8\|local)<<20` |
| ZenStates-Core `Cpu.cs`（irusanov，dev 分支） | 上游维护者源码 | **第四轮实际打开并逐字引用** | `MakeCoreMask`：桌面 `(ccd<<28)\|((core%8)<<20)`、`[23:20]`=核内序号数值字段；`MakeCoreMaskForLogicalCore` 走 APOB 映射；证明独热位编码错误、平台归因不成立 |
| AMD PPR | 官方 | 公开版本不含 MP1 消息表 | 不能用于证明命令号 |
| RyzenSMU.p / RyzenAdj 上游源码 | 维护者源码 | 经本仓库研究引用 | 邮箱与命令号口径 |
| vectree.io（Zen5 CO/Curve Shaper 分析） | 专业社区 | 已访问 2026-11-25 | BIOS 侧观测：老 Zen3/早 Zen4 停 -30，当前 AGESA 常见 −50；仅为工具/BIOS 侧观测，非官方固件范围（Inferred） |
| SMUDebugTool / DeepWiki 源码索引、G-Helper 文档 | 维护者源码/文档 | 已访问 2026-11-25 | 第三方工具软上限 ±999 / [-40,0]（非固件限制） |
| SkatterBencher 等社区分析 | 专业社区 | 二手线索 | 3–5 mV/档经验值（Inferred） |
| `research/02-hardware-access.md` §2/§11 | 本仓库研究记录（原标"实测"） | 原始日志缺失，多处来源 Unknown；第三轮降级、第四轮撤回平台归因 | 命令号（逆向口径 Inferred）、输入侧 ±50 钳制（旧实现记录）；"出厂 −11…−26"已撤回；"bit23 平台拒绝"已撤回（实为独热位编码错误） |

## 11. 待核验问题

- **修正编码后的实机复验（最高优先）**：旧实现用独热位 `1<<(20+local)` 导致核 3/11 掩码错误（应为 `3<<20`、`0x10300000`）；改用数值编码 + APOB 映射后，逐核读/写/回读确认 Ryzen 9 9955HX 全部 16 核可读、是否存在真实失败核，并保存原始日志与工具版本。用户已报告其他工具可改全核（User-provided），本仓库无硬件，保持 Unverified，见 `research/02` §11 第四轮审计注记。
- Zen5 / Hawk Point / Dragon Range 的家族白名单误分类是否已在新版模块中修复。
- S3/S4 恢复后 CO 是否丢失。
- **本机 SMU Debug Tool 的 −50/+30 界限来源**：工具无关直写测试 +31…+50 是否被 SMU 接受并回读一致；补记工具准确版本。来源定性前保持 Unverified。
- **Curve Shaper 固件协议**：5×3=15 点结构、命令号（Zen4/5 `0xA6`/`0x84`）与参数编码已由上游源码逐行确认（Verified 工具事实，见 §2 与 [implementation-references.md](implementation-references.md)）；但**官方单位定义与 Zen5 固件档位范围仍无 AMD 公开资料**（Unverified/Inferred 作为官方事实），不得与 CO 的数值表混用。

## 12. 核验状态分项

| 结论 | 状态 |
|---|---|
| CO 存在、属 PBO、可逐核、可通过 BIOS/Ryzen Master 调整 | Verified |
| CO 是步进档位而非 mV | Verified（工具链一致 + 本仓库纠错） |
| 固件/AGESA 命令层档位范围 | **Unverified / 官方未发布**；负向常见 −50、正向常见 +30 为工具/显示观测，非官方固件范围 |
| Ryzen Master 显示钳制 +30/−50，实际可超过 | Verified（MSI FAQ-10122 厂商文档） |
| 第三方工具软上限 ±999 / [-40,0]；−50…+30 属 Curve Shaper | Verified（工具事实/源码，非固件限制） |
| 用户 SMU Debug Tool 报告 CO 负向 −50、正向 +30，不对称 | **User-provided**（用户自述，未附原始日志/工具版本，本仓库未独立复核）；**来源（工具/固件/平台）Unverified** |
| 每档约 3–5 mV（经验，非线性） | Inferred，禁止当官方公式 |
| SMU 命令号 0x06/0xD5 | Inferred（逆向协议口径）；本仓库曾报告写入/回读过程但原始日志缺失、来源 Unknown |
| 核心掩码编码 `[23:20]` 为 4 位核内序号、非独热位 | **Verified（外部源码逐字核对，见 [amd-smu.md](amd-smu.md) §3.1）** |
| CO 写入参数 = 掩码高 12 位 + 16 位补码 margin；全核/逐核为不同命令 | **Verified 上游源码**（[implementation-references.md](implementation-references.md) §3.1/§3.2）；作为硬件协议 Inferred |
| Curve Shaper 命令号 Zen4/5 `0xA6`/`0x84`、5 频段、margin 钳 [−50,+30] | **Verified 工具源码**（本轮逐行核对）；作为 Zen5 官方固件协议 Inferred |
| 世代 CO 范围注释（Zen4 前 ±30 / Zen4+ ±50） | Verified 源码注释存在 / Inferred 作为官方事实（AMD 未发布） |
| "出厂可能为非零偏移（本仓库 14 核 -11…-26）" | **已撤回 / Unknown**：无原始数据、状态混淆（同会话有写入），见 `research/02` §11 第三轮审计注记。"写入 0 = 设 0 档 = 无偏移（中性默认）"为 Verified 源码行为；"厂商出厂普遍预置负 CO"无证据，不作常态假设；"读到 0/非零"的读取语义为独立逻辑结论；见 §8 |
| "bit23 部分核被平台拒绝" | **已撤回**：源于旧实现独热位编码错误（Verified 源码比对），非平台限制；用户报告其他工具可改全核 = User-provided（未附日志/工具版本）。修正编码后的实机结果 Unverified |
| 休眠恢复清回出厂的假设 | Unverified |
