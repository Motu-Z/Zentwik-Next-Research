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

## 3. 核验时的分层要求（写作规则，非技术事实）

建立 GPU 规范时，每个功率/频率/电压/温度参数必须按 [parameter-record-schema.md](../../data/parameter-record-schema.md) 区分并记录以下层次，**不得互相替代、不得凭单一来源下定论**：

1. **硬件能力**（芯片设计可承受的上限）；
2. **VBIOS / 固件限制**（具体卡出厂固化，可能与硬件能力不同）；
3. **驱动限制**（厂商驱动施加的策略上限）；
4. **工具限制**（第三方调校工具自身的输入/钳制范围，属 L2）；
5. **实际运行状态**（负载、温度、功耗墙共同决定的当前值，属 L3）。

尤其不得把"工具能输入某数值"当成"硬件/VBIOS 接受该数值"，也不得把"回读到写入值"当成"实际生效"（请求型接口可能忽略请求）。多个网站重复同一说法不构成多个独立证据。

## 4. 注意

不要在未核验前把 GPU 数值与 CPU 数值共用类型或单位约定；无可靠资料时保持 Unverified，不填具体范围。
