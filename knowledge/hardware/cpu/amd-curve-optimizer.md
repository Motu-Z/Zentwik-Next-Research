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
- 相关文档：[numeric-semantics.md](../../data/numeric-semantics.md)、[units.md](../../data/units.md)、[amd-smu.md](amd-smu.md)、[intel-oc-mailbox.md](intel-oc-mailbox.md)

## 1. 准确定义（外部技术事实）

Curve Optimizer 是 AMD 在支持 PBO 的 Ryzen 处理器上提供的一项调校功能，允许对处理器**每条电压-频率（V/F）曲线**施加一个**逐核或全核的有符号偏移**。负值使核心在给定频率下请求更低电压，从而在 PPT/TDC/EDC 与温度预算内为 Precision Boost 留出更多升频空间；正值用于稳定较弱的核心。

它是曲线的**偏移量**，不是固定电压、不是倍频上限、不是功耗墙。CO 与 PBO Limits（PPT/TDC/EDC）、PBO Scalar、Max Boost Override 是不同的控制项。

## 2. 适用范围

- 硬件：支持 PBO 的 AMD Ryzen / Threadripper；CO 逐核设置通常在 Zen 2 及以后、经 AGESA/BIOS 暴露。具体可用性与范围由 CPU 型号、主板 BIOS 和 AGESA 决定（Verified：AMD FAQ 指出 PBO 需兼容型号与主板）。
- **档位范围不对称，不能写成"±50 对称"**：AMD 官方未发布固件/AGESA 命令层的档位范围。观测到的负向常达 −50（个别平台 −60），正向通常更小（Ryzen Master 显示上限 +30）。显示上限、工具输入上限与固件/邮箱接受范围三者必须分开（见第 5 节分层记录）。
- Zen 5 增加 Curve Shaper，与 CO 并存但属于**不同控制项、不同参数**，不得与 CO 共用数值表或单位约定（Verified 存在性：AMD Release Notes；结构细节见下）。
  - 结构：**5 个频段 × 3 个温段 = 15 个可调点**，叠加在 CO 之上做分点微调（Verified：SMU Debug Tool 源码 `SettingsForm.cs:352-365` 的 `CsTierNames` 5 项、`CsBandNames` 3 项、`CsControls[5,3]`；这是**工具源码事实**，AMD 未发布官方命令/单位定义）。
  - SMU Debug Tool 的 Curve Shaper 说明文字标注范围 −50…+30，但这是**该工具对 Curve Shaper 的表述**，既不是 CO 的范围，也未核验为 Zen5 固件范围（Unverified）。

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

- 正确的换算只有一种方向在项目中被允许：**步进值 → 写入参数**（`(coreMask & 0xFFF00000) | (step as int16)`），由 provider 完成（掩码公式来源于本仓库旧实现记录，来源类别 Unknown/原始日志缺失；公式语义 = Derived）。
- **禁止**提供 `Step → mV` 的确定性转换函数。若 UI 需要示意，只能标注为"约 3–5 mV/档，仅供参考，非官方换算"（Inferred）。
- 符号：负值为降压（undervolt），正值为加压。补码负值在写入低 16 位时必须保持符号扩展正确。

## 7. 关联术语

PBO、PPT、TDC、EDC、Precision Boost、V/F curve、Curve Shaper、SMU Mailbox、SetMargin（0x06）/ GetMargin（0xD5）、CoreMask、Preferred Core（CPPC）

## 8. 常见误解及错误用法

| 误解 | 纠正 | 依据 |
|---|---|---|
| "CO offset 单位是 mV" | 是步进档位，无单位；mV 等效值不固定且未公开 | Verified（社区一致）+ 本仓库纠错记录 |
| "-20 就是降 20 mV" | 不成立；3–5 mV/档只是粗略经验，非线性 | Inferred |
| "写 0 = 恢复默认" | 错。**读到 0 不等于默认状态**：读取值不能证明设备处于默认/出厂状态（默认状态须由 BIOS 恢复默认后、任何写入前的、带原始日志的读取确定） | 逻辑结论（依据 [measurement-evidence-schema.md](../../data/measurement-evidence-schema.md) §3/§6）。**原"出厂为 -11…-26"归因已于第三轮撤回：无原始数据、状态混淆，标 Unknown**，见 `research/02` §11 审计注记 |
| "所有核都必须能读写" | 本平台 bit23 对应核被邮箱拒绝，属部分可用 | 报告称（原始日志缺失，来源 Unknown），`research/02` §11；见 [measurement-evidence-schema.md](../../data/measurement-evidence-schema.md) |
| "命令号与固件无关，可全平台通用" | 命令号与 CPU 世代/型号强绑定；家族白名单误分类会发错世代命令 | Verified，`research/02` §2.2/§10 |
| "回读相同即写入成功" | 这是必要条件；仍需观测效果层（频率/功耗/温度）以排除钳位 | Inferred，`research/02` §8.3 |
| "CO 是 ±50 对称范围" | 不对称：负向常见 −50、正向常见 +30；且这些是工具/显示观测，非 AMD 发布的固件范围 | MSI FAQ（Verified 厂商文档）+ 用户提供实测（User-provided，未独立复核），见 §5 |
| "Ryzen Master 显示到 +30/−50 就是固件上限" | 显示钳制 ≠ 固件上限；MSI FAQ 明确实际可设定并应用超出显示范围的值 | Verified（MSI FAQ-10122） |
| "SMU Debug Tool 的 −50…+30 来自 CO 输入框" | 该工具 CO 输入框是 ±999；−50…+30 出现在 Curve Shaper 说明，属另一参数 | Verified（工具源码） |

## 9. Zentwik 内部约定（Proposed）

1. 字段统一命名 `coOffset`，类型 `CurveOffsetStep`，**不得**命名为 `millivolts`/`mv`。
2. CLI 打印格式：`core <n>  co <step>`，**不得**打印 `offset(mV)`。早期实现曾犯此错并已纠正（`research/02` §11）。
3. 能力状态区分 `Available` / `NotSupported` / `DriverUnavailable` / `ProbeFailed`；部分核可读时标记 `degraded` 并逐核报 `error`，不阻塞其余核。
4. 方案引用了读不到前值的核时记 `Skipped` 且不写入。
5. 持久化必须记录来源（SMU 读出 / 用户输入 / 推断）、采集时间、CPU 身份（family/model/stepping/微码）；不得把"上次写入值"当出厂值（对应 `project/ARCHITECTURE_CONSTRAINTS.md` §10）。
6. `PlanApplyRecord.PersistsAcrossReboot` 对 AMD CO 为 `true`（ADR-005）。

## 10. 权威资料来源与访问状态

| 来源 | 类型 | 访问状态 | 用途 |
|---|---|---|---|
| AMD Ryzen Master 产品页 / Release Notes | 官方 | 已访问 2026-11-25 | CO/PBO 特性存在性、参数调整方式 |
| MSI FAQ-10122（Curve Optimizer Magnitude vs Ryzen Master 显示） | 厂商官方 FAQ | 已访问 2026-11-25 | Ryzen Master 显示钳制 +30/−50，实际可设定并应用超出该范围；证明显示范围 ≠ 固件范围 |
| AMD FAQ — Curve Optimizer Feature in Ryzen Master（Build 2.9.0.2093） | 官方 PDF | 已访问 2026-11-25 | CO 定义、All Core/Per Core、验证方式；官方 FAQ 未给固件档位边界 |
| SMU Debug Tool 源码 `SettingsForm.cs`（irusanov 仓库镜像） | 维护者源码 | 已访问 2026-11-25 | CO 输入框 ±999（无命名常量）；−50…+30 出现在 Curve Shaper 说明文字 |
| AMD PPR | 官方 | 公开版本不含 MP1 消息表 | 不能用于证明命令号 |
| RyzenSMU.p / RyzenAdj 上游源码 | 维护者源码 | 经本仓库研究引用 | 邮箱与命令号口径 |
| vectree.io（Zen5 CO/Curve Shaper 分析） | 专业社区 | 已访问 2026-11-25 | BIOS 侧观测：老 Zen3/早 Zen4 停 -30，当前 AGESA 常见 −50；仅为工具/BIOS 侧观测，非官方固件范围（Inferred） |
| SMUDebugTool / DeepWiki 源码索引、G-Helper 文档 | 维护者源码/文档 | 已访问 2026-11-25 | 第三方工具软上限 ±999 / [-40,0]（非固件限制） |
| SkatterBencher 等社区分析 | 专业社区 | 二手线索 | 3–5 mV/档经验值（Inferred） |
| `research/02-hardware-access.md` §2/§11 | 本仓库研究记录（原标"实测"） | 原始日志缺失，多处来源 Unknown；第三轮已降级 | 命令号/掩码（逆向口径 Inferred）、输入侧 ±50 钳制（旧实现记录）；"出厂 −11…−26"已撤回 |

## 11. 待核验问题

- bit23 在 Ryzen 9 9955HX 上被拒的根因（独热位编码差异 vs 该平台不接受该 bit），需一次完整掩码扫描确认。
- Zen5 / Hawk Point / Dragon Range 的家族白名单误分类是否已在新版模块中修复。
- S3/S4 恢复后 CO 是否丢失。
- **本机 SMU Debug Tool 的 −50/+30 界限来源**：工具无关直写测试 +31…+50 是否被 SMU 接受并回读一致；补记工具准确版本。来源定性前保持 Unverified。
- **Curve Shaper 的档位范围与命令号**：结构 5×3=15 点已由工具源码逐行确认（Verified 工具事实，`SettingsForm.cs:352-365`）；但其独立命令号、官方单位定义与 Zen5 固件档位范围仍未核验（Unverified），不得与 CO 的数值表混用。

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
| SMU 命令号 0x06/0xD5、核心掩码编码 | Inferred（逆向协议口径）；本仓库曾报告写入/回读过程但原始日志缺失、来源 Unknown |
| "出厂可能为非零偏移（本仓库 14 核 -11…-26）" | **已撤回 / Unknown**：无原始数据、状态混淆（同会话有写入），见 `research/02` §11 第三轮审计注记。仅保留"读到 0 ≠ 默认状态"为逻辑结论 |
| bit23 部分核被邮箱拒绝 | 报告称，来源 Unknown（原始日志缺失）；不得标 Verified 实测 |
| 休眠恢复清回出厂的假设 | Unverified |
