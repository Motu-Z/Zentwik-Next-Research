# GPU 技术（待研究索引）

- 领域：`hardware/gpu`
- 状态：**尚无可靠资料，仅建立扩展入口**，不编造定义。
- 相关领域：本仓库产品领域列出 GPU 温度/频率/功耗/利用率监控，但研究报告未覆盖 GPU 技术细节。

## 1. 计划条目（Unverified，均为待研究问题）

| 主题 | 厂商/来源 | 状态 |
|---|---|---|
| NVIDIA 遥测接口（NVML） | NVIDIA 官方 NVML API | Unverified |
| AMD GPU 遥测（ADL / ROCm SMI） | AMD 官方 SDK | Unverified |
| GPU 帧时间（PresentMon / ETW） | Microsoft / Intel | Unverified（`research/02` §6 提及机制） |
| GPU 电压/频率调校 | 厂商工具 | Unverified，是否纳入范围待定 |

## 2. 建立规范时的要求

按 [knowledge/_TEMPLATE.md](../../_TEMPLATE.md) 填写；区分外部事实与项目约定；数值语义遵循 [numeric-semantics.md](../../data/numeric-semantics.md)；核验后标注状态与来源章节。

## 3. 注意

不要在未核验前把 GPU 数值与 CPU 数值共用类型或单位约定。
