# 专项技术事实审计记录（AUDIT）

- 审计日期：2026-11-25
- 范围：`knowledge/` 规范群 + `research/` + `decisions/` + `project/` 全文档
- 目标：找出错误、过度概括、数值范围错误、单位错误、概念混淆；修正有充分依据者；不增加无意义文档
- 方法：内置联网检索 + 权威原文交叉核验（AMD / Intel / Microsoft 官方、维护者源码、专业社区）
- 说明：用户指定的 /deep-research-pro、/tavily、/web-search-exa skill 在本会话未注册，无法调用，故用等效检索完成

## 1. 审计发现与处理

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
