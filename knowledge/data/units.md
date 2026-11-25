# 单位与换算

- 规范名称：单位与换算表（Units & Conversions）
- 领域：`data`
- 核验状态：Verified（物理单位定义）+ Inferred（社区经验值与社区逆向编码实现，已标注）
- 权威来源：IEEE 754；Intel/AMD/MS 官方文档与维护者源码（见各专项规范）
- 相关文档：[numeric-semantics.md](numeric-semantics.md)、[../hardware/cpu/amd-curve-optimizer.md](../hardware/cpu/amd-curve-optimizer.md)、[../hardware/cpu/intel-oc-mailbox.md](../hardware/cpu/intel-oc-mailbox.md)

## 1. 换算表

| 量 | 单位 | 换算 | 备注 |
|---|---|---|---|
| 电压 | V / mV | 1 V = 1000 mV | |
| Intel 电压偏移 | 1/1024 V 定点 | mV = code / 1.024；code = mV × 1.024 | 11 位有符号 S11.0.10，Inferred（社区逆向口径，非 Intel 官方规范） |
| AMD CO offset | 档（无单位） | **无固定 mV 换算** | 社区经验约 3–5 mV/档（Inferred），禁止当公式 |
| 频率 | Hz / MHz / GHz | 1 GHz = 1000 MHz = 10^9 Hz | Intel 倍频 ×100 MHz |
| FMax Override（Boost Clock Override） | MHz（有符号） | 频率**上限**覆盖 -1000…+200 MHz，步进 25 MHz；非频率偏移 | 上限非偏移，Zen2 起，见 amd-smu.md；回读容差 ≤10 MHz |
| 功率 | W / mW | RAPL：单位来自 MSR 0x606 bits[3:0]，**1/2^PU W/LSB**（2 的负幂） | 缺省 PU=0011b → 1/8 W；不得硬编码 |
| 能量 | J / µJ | RAPL：累加器差分 × 1/2^ESU J/LSB（0x606 bits[12:8]，缺省 ESU=16 → 15.3 µJ）；功耗 = 能量差 / 时间窗 | 32 位累加器须按模 2^32 处理回绕；单位须读寄存器，不得硬编码 |
| 时间 | s / ms / µs / 100ns | QPC tick → 秒用频率换算 | 采样用 QPC |
| 温度 | °C / K | 差值数值相同；绝对温度偏移 273.15 | |
| 百分比 / 比例 | % / ratio | ratio = percent / 100 | PDH 性能可 >100%，见 numeric-semantics §5 |

## 2. 换算规则（Proposed）

1. 换算函数命名体现方向，如 `codeToMillivolt`、`millivoltToCode`。
2. 单位标签在边界保留；内部强类型承载。
3. 舍入规则显式；负值按补码符号扩展后再换算。
4. 容差带单位，比较在工程值层。

## 3. 特别警示

- **CO offset 没有官方 mV 值**：任何 `coStepToMv()` 确定性函数都是错误。
- **Intel 1.024 是单位换算**：与 CO 的档位语义无关，二者不可共用类型。
- **RAPL 单位来自寄存器**：不得硬编码 1/8 W，须读 `0x606` 的 unit 字段。
- **RAPL 是 2 的负幂（1/2^n）**：功率 PU=bits[3:0]、能量 ESU=bits[12:8]、时间 TU=bits[19:16] 均为 `1/2^字段` 的缩放；写成 `2^unit` 是方向性错误，会使结果差若干个数量级。缺省 PU=3→1/8 W、ESU=16→15.3 µJ、TU=10→976 µs（Intel SDM Vol.3B §14.10.1；多源交叉）。
- **FMax Override 是频率上限不是偏移**：-1000…+200 MHz、25 MHz 步进、有符号；不得与 CO 的无单位档位或 Intel 的电压定点混用。

## 4. 核验状态

物理单位定义（V/mV、Hz/MHz/GHz、W、J、s 等）= Verified；AMD CO 写入位编码（掩码 + 16 位补码）= Verified 上游源码事实（见 amd-smu.md §3.1，作为硬件协议仍 Inferred）；Intel 电压偏移编码 S11.0.10 / ×1.024 = Inferred（社区逆向口径，非 Intel 官方规范，见 intel-oc-mailbox.md §4/§12）；RAPL 单位 1/2^n 及三项缺省值 = Verified（Intel SDM §14.10.1 + 多源交叉）；FMax Override 范围 -1000…+200 MHz / 25 MHz = Inferred（SkatterBencher/MSI 官方，专业社区来源，非 AMD 官方规范）；CO 每档 mV = Inferred；Zen5 的 1 MHz FMax 粒度 = Inferred。
