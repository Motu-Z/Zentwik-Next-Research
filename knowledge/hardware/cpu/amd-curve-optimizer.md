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
  - 本仓库一手实测：`research/02-hardware-access.md` §2、§11
- 相关文档：[numeric-semantics.md](../../data/numeric-semantics.md)、[units.md](../../data/units.md)、[amd-smu.md](amd-smu.md)、[intel-oc-mailbox.md](intel-oc-mailbox.md)

## 1. 准确定义（外部技术事实）

Curve Optimizer 是 AMD 在支持 PBO 的 Ryzen 处理器上提供的一项调校功能，允许对处理器**每条电压-频率（V/F）曲线**施加一个**逐核或全核的有符号偏移**。负值使核心在给定频率下请求更低电压，从而在 PPT/TDC/EDC 与温度预算内为 Precision Boost 留出更多升频空间；正值用于稳定较弱的核心。

它是曲线的**偏移量**，不是固定电压、不是倍频上限、不是功耗墙。CO 与 PBO Limits（PPT/TDC/EDC）、PBO Scalar、Max Boost Override 是不同的控制项。

## 2. 适用范围

- 硬件：支持 PBO 的 AMD Ryzen / Threadripper；CO 逐核设置通常在 Zen 2 及以后、经 AGESA/BIOS 暴露。具体可用性与范围由 CPU 型号、主板 BIOS 和 AGESA 决定（Verified：AMD FAQ 指出 PBO 需兼容型号与主板）。固件/AGESA 命令层一般约 -50…+50 档（负向通常可达 -50；正向多数 BIOS/Ryzen Master 只到 +30，部分工具到 +50），见第 5 节。
- Zen 5 增加 Curve Shaper（按温度/频段分点微调），与 CO 并存但属于不同控制项（Verified：社区与 AMD Release Notes 曲线优化器自动测试描述）。

## 3. 技术行为

1. CO 作用于 SMU 内部的 V/F 曲线，在 SMU 计算每个核心目标电压时叠加偏移，再转换为各核参考码（Inferred：内部数据路径 AMD 未公开）。
2. 偏移沿曲线生效，**并非每个频率点固定减去相同 mV**（Inferred，社区口径）。
3. 写入链路（外部工具）：经 PawnIO 加载 `RyzenSMU.bin`，通过 SMU 邮箱发送 `SetMargin` / `GetMargin` 命令，逐核写入并立即回读校验（Verified：本仓库实测，见 `research/02-hardware-access.md` §2.3、§11）。
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
| L1 固件/AGESA 实际允许 | 约 -50…+50 档（正向多数 BIOS/Ryzen Master 到 +30） | 多源工具链一致；AMD 官方未公开固件命令参数。Verified（范围本体）/ Inferred（正向 +30 的厂商差异） |
| L2 软件输入钳制 | 旧版 Zentwik 输入侧 -50…+50 | `research/02` §2.3；`workbuddy-*` 工具实现选择，非固件事实。Verified |
| L2 第三方工具上限 | SMUDebugTool ±999、G-Helper [-40,0] | 各工具源码/文档；软上限，非固件限制。Verified |
| L3 实际稳定可用 | 取决于 CPU 个体、散热、电压与负载 | 无通用数值，强核常只能 -5…−15，弱核可达更负。**Unverified，不填规范限制** |
| L4 典型调校值 | 社区常见 -10…−30 起点 | **仅作参考，非规范限制、非官方推荐** |
| L5 Zentwik 约定 | 字段 `coOffset`、类型 `CurveOffsetStep`（i8/i16）、单位 `step`、不持久化 mV | Proposed，见 §9 |

明细与证据：

| 项 | 取值 | 核验状态 |
|---|---|---|
| 单位 | 档（step / count），**无单位**，不是 mV | Verified（社区与工具链一致；AMD 官方未发布单位定义，属外部事实缺失） |
| 固件/AGESA 命令层档位范围 | 约 **-50 到 +50**（负向通常可达 -50；正向多数 BIOS/Ryzen Master 只到 +30，部分工具到 +50） | Verified（多源工具链一致：vectree.io、SMUDebugTool 源码、vgoodslab 的 -50 实操、本仓库旧代码钳制；AMD 官方 FAQ 不公布固件命令参数，故不标为官方定义）；正向 +30 上限因厂商而异 = Inferred |
| 本仓库工具写入钳制范围 | -50 到 +50（旧代码输入侧钳制） | Verified（`research/02-hardware-access.md` §2.3）；该钳制**与固件常见档位范围一致**，不是工具凭空自创，也不是 AMD 官方公布值 |
| 第三方工具软上限（非固件限制） | SMUDebugTool ±999、G-Helper [-40,0] | Verified（各工具源码/文档）；工具自身的保守或宽容上限，**不得当作固件事实** |
| 等效电压 | 社区经验约 3–5 mV/档，且随频率点非线性变化 | Inferred，**禁止作为官方事实或换算公式写入产品契约** |

> **关键结论**：AMD 从未发布"每档 = N mV"的官方换算。3–5 mV/档是社区实测经验值，随平台、频率点、温度而变。**任何把 CO offset 标注为 mV 的行为都是错误。**
>
> **易混淆点（务必区分两个维度）**：**档位范围**（约 -50…+50）是离散的计数值范围，回答"能设多少档"；**每档 mV 等效**不存在官方固定值，回答"每档值多少电压"。前者是固件/BIOS 允许的档数，后者是无官方定义的经验换算——不要把"-50 档"误读成"-50 mV"。

## 6. 转换规则

- 正确的换算只有一种方向在项目中被允许：**步进值 → 写入参数**（`(coreMask & 0xFFF00000) | (step as int16)`），由 provider 完成（Verified：本仓库实测写入 `(mask & 0xFFF00000) | 值`）。
- **禁止**提供 `Step → mV` 的确定性转换函数。若 UI 需要示意，只能标注为"约 3–5 mV/档，仅供参考，非官方换算"（Inferred）。
- 符号：负值为降压（undervolt），正值为加压。补码负值在写入低 16 位时必须保持符号扩展正确。

## 7. 关联术语

PBO、PPT、TDC、EDC、Precision Boost、V/F curve、Curve Shaper、SMU Mailbox、SetMargin（0x06）/ GetMargin（0xD5）、CoreMask、Preferred Core（CPPC）

## 8. 常见误解及错误用法

| 误解 | 纠正 | 依据 |
|---|---|---|
| "CO offset 单位是 mV" | 是步进档位，无单位；mV 等效值不固定且未公开 | Verified（社区一致）+ 本仓库纠错记录 |
| "-20 就是降 20 mV" | 不成立；3–5 mV/档只是粗略经验，非线性 | Inferred |
| "写 0 = 恢复默认" | 错。出厂可能已预置非零偏移（本仓库 14 个可读核出厂为 -11…-26） | Verified 实测，`research/02` §11 |
| "所有核都必须能读写" | 本平台 bit23 对应核被邮箱拒绝，属部分可用 | Verified 实测，`research/02` §11 |
| "命令号与固件无关，可全平台通用" | 命令号与 CPU 世代/型号强绑定；家族白名单误分类会发错世代命令 | Verified，`research/02` §2.2/§10 |
| "回读相同即写入成功" | 这是必要条件；仍需观测效果层（频率/功耗/温度）以排除钳位 | Inferred，`research/02` §8.3 |

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
| AMD PPR | 官方 | 公开版本不含 MP1 消息表 | 不能用于证明命令号 |
| RyzenSMU.p / RyzenAdj 上游源码 | 维护者源码 | 经本仓库研究引用 | 邮箱与命令号口径 |
| vectree.io（Zen5 CO/Curve Shaper 分析） | 专业社区 | 已访问 2026-11-25 | 固件/BIOS 档位范围：老 Zen3/早 Zen4 停 -30，当前 AGESA 常达 -50 |
| SMUDebugTool / DeepWiki 源码索引、G-Helper 文档 | 维护者源码/文档 | 已访问 2026-11-25 | 第三方工具软上限 ±999 / [-40,0]（非固件限制） |
| SkatterBencher 等社区分析 | 专业社区 | 二手线索 | 3–5 mV/档经验值（Inferred） |
| `research/02-hardware-access.md` §2/§11 | 本仓库一手实测 | 已核验 | 命令号、掩码、实测纠错、输入侧 ±50 钳制 |

## 11. 待核验问题

- bit23 在 Ryzen 9 9955HX 上被拒的根因（独热位编码差异 vs 该平台不接受该 bit），需一次完整掩码扫描确认。
- Zen5 / Hawk Point / Dragon Range 的家族白名单误分类是否已在新版模块中修复。
- S3/S4 恢复后 CO 是否丢失。

## 12. 核验状态分项

| 结论 | 状态 |
|---|---|
| CO 存在、属 PBO、可逐核、可通过 BIOS/Ryzen Master 调整 | Verified |
| CO 是步进档位而非 mV | Verified（工具链一致 + 本仓库纠错） |
| 固件/BIOS 常见档位范围 -50…+50（正向多数 BIOS 到 +30） | Verified（多源工具链一致；AMD 官方未公开固件命令参数） |
| 第三方工具软上限 ±999 / [-40,0] | Verified（工具事实，非固件限制） |
| 每档约 3–5 mV（经验，非线性） | Inferred，禁止当官方公式 |
| SMU 命令号 0x06/0xD5、核心掩码编码 | Verified（本仓库实测）/ Inferred（协议来源） |
| 出厂可能为非零偏移、bit23 部分核被拒 | Verified 实测（本机） |
| 休眠恢复清回出厂的假设 | Unverified |
