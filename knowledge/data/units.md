# 单位与换算

- 规范名称：单位与换算表（Units & Conversions）
- 领域：`data`
- 核验状态：Verified（物理单位与官方编码）+ Inferred（社区经验值，已标注）
- 权威来源：IEEE 754；Intel/AMD/MS 官方文档与维护者源码（见各专项规范）
- 相关文档：[numeric-semantics.md](numeric-semantics.md)、[../hardware/cpu/amd-curve-optimizer.md](../hardware/cpu/amd-curve-optimizer.md)、[../hardware/cpu/intel-oc-mailbox.md](../hardware/cpu/intel-oc-mailbox.md)

## 1. 换算表

| 量 | 单位 | 换算 | 备注 |
|---|---|---|---|
| 电压 | V / mV | 1 V = 1000 mV | |
| Intel 电压偏移 | 1/1024 V 定点 | mV = code / 1.024；code = mV × 1.024 | 11 位有符号 S11.0.10，Verified |
| AMD CO offset | 档（无单位） | **无固定 mV 换算** | 社区经验约 3–5 mV/档（Inferred），禁止当公式 |
| 频率 | Hz / MHz / GHz | 1 GHz = 1000 MHz = 10^9 Hz | Intel 倍频 ×100 MHz |
| FMax 步进 | MHz | 约 25 MHz（旧版） | 回读容差 ≤10 MHz |
| 功率 | W / mW | RAPL：单位来自 MSR 0x606 bits[3:0]，2^unit W/LSB | 缺省 1/8 W |
| 能量 | J / µJ | RAPL 累加器差分 | 须处理回绕 |
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

## 4. 核验状态

物理单位与 Intel/AMD 编码 = Verified；CO 每档 mV = Inferred；FMax 步进来自旧版实现 = Verified（实现事实）。
