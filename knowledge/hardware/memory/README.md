# 内存技术（待研究索引）

- 领域：`hardware/memory`
- 状态：**尚无可靠资料，仅建立扩展入口**，不编造定义。
- 相关领域：产品监控领域列出内存占用；技术栈评估提及 RAM 超频（EXPO/XMP）但无专项研究。

## 1. 计划条目（Unverified）

| 主题 | 来源 | 状态 |
|---|---|---|
| 物理/虚拟内存指标（GlobalMemoryStatusEx） | Microsoft 官方 | Unverified |
| EXPO / XMP 内存超频 | AMD / Intel JEDEC | Unverified |
| 内存带宽/延迟遥测 | 厂商 SDK | Unverified |

## 2. 建立规范时的要求

区分"内存容量（字节）""占用率（百分比）""带宽（B/s）""频率（MHz）""时序（无量纲周期数）"；单位遵循 [units.md](../../data/units.md)；按 [knowledge/_TEMPLATE.md](../../_TEMPLATE.md) 核验后填写。

**已核验的单位语义（Verified，2026-11-25）**：DDR5 标称速率（如 DDR5-4800）是**传输速率 4800 MT/s**，不是 4800 MHz；对应 I/O 时钟约 2400 MHz（DDR 双沿，×2）。Windows 任务管理器常显示物理时钟（≈标称/2），CPU-Z/HWiNFO/BIOS 各层显示口径不一，比较前必须先确认单位（JEDEC 用 MT/s；Kingston FAQ KTM-071015-GEN-20、MSI 科普一致）。带宽 = MT/s × 64 bit ÷ 8（如 4800 MT/s → 38400 MB/s，即 PC5-38400）。时序（如 36-36-36-76）的单位是**时钟周期数**，不是 MT/s 或 ns。MCLK/UCLK/FCLK 是不同时钟域，不得合并为单一"内存频率"字段。

**待核验的表示差异（Unverified）**：BIOS、ZenTimings、SMU / PM table 对同一参数的十进制/十六进制表示与倍率口径可能不同；各平台对 Gear 模式、PMIC、训练行为的支持条件尚未核验。建立正式规范前不填具体时序范围或电压值。

## 3. 注意

不要为凑齐目录而编造内存时序或超频语义；资料不足保持待研究状态。
