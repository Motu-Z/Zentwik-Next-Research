# 全局术语索引（GLOSSARY）

按术语快速定位规范。格式：**术语** — 一句话定义 → 规范链接。带 ⚠ 的为高风险易混淆术语。

## A–C

- **AC / DC** — 电源策略的插电/电池两套值 → [power-management.md](windows/power-management.md)
- **能力（Capability）** — 模块对外声明的可做事项，不等于可用性状态 → [core-vocabulary.md](terminology/core-vocabulary.md)
- **通道状态** — Available / NotSupported / DriverUnavailable / ProbeFailed 等，不可压成 bool → [lifecycle.md](architecture/lifecycle.md)、[amd-smu.md](hardware/cpu/amd-smu.md)
- **CO（Curve Optimizer）** ⚠ — AMD 曲线优化器，**有符号步进档位，非 mV** → [amd-curve-optimizer.md](hardware/cpu/amd-curve-optimizer.md)
- **CPU Sets** — Windows 软亲和性，优先级低于 affinity mask → [process-scheduling.md](windows/process-scheduling.md)
- **Curve Shaper** — Zen 5 按频段×温段 5×3=15 点微调（Verified：工具源码事实，非 AMD 官方协议），与 CO 是不同参数、不共用数值表 → [amd-curve-optimizer.md](hardware/cpu/amd-curve-optimizer.md)

## D–F

- **账单 / 账本（Ledger）** — 记录写对象前值/目标/实际/所有权/恢复资格的状态机 → [state-ownership.md](domain/state-ownership.md)
- **FMax / Boost Clock Override** ⚠ — AMD 频率**上限**覆盖，-1000…+200 MHz、25 MHz 步进（非偏移、非 mV） → [amd-smu.md](hardware/cpu/amd-smu.md)、[units.md](data/units.md)
- **定点编码值** ⚠ — 整数表示缩放后的物理量，须反解单位 → [numeric-semantics.md](data/numeric-semantics.md)
- **Provider** — 某领域在某硬件平台的具体实现；一领域可多 Provider → [modules-capabilities.md](architecture/modules-capabilities.md)

## G–M

- **工程值 / 原始值** ⚠ — 换算后的人读值 vs 硬件整数编码 → [numeric-semantics.md](data/numeric-semantics.md)
- **GetMargin / SetMargin** — AMD CO 读/写邮箱命令（0xD5/0x06 等，按世代） → [amd-smu.md](hardware/cpu/amd-smu.md)
- **模块（Module）** — 声明 ID/版本/能力/依赖的加载单元 → [modules-capabilities.md](architecture/modules-capabilities.md)

## N–P

- **OC Mailbox（0x150）** ⚠ — Intel 电压偏移邮箱，11 位有符号、单位 1/1024 V → [intel-oc-mailbox.md](hardware/cpu/intel-oc-mailbox.md)
- **PBO** — AMD Precision Boost Overdrive，含 CO、Limits、Scalar → [amd-curve-optimizer.md](hardware/cpu/amd-curve-optimizer.md)
- **插件（Plugin）** — 进程外受控权限扩展，非内置模块 → [modules-capabilities.md](architecture/modules-capabilities.md)
- **Power Plan** — 三种独立方案之一（电源） → [three-plans.md](domain/three-plans.md)
- **PowerSetActiveScheme** — 电源策略写后必须激活才生效 → [power-management.md](windows/power-management.md)
- **Processor Group** — Windows 处理器组，亲和性 64 位/组 → [process-scheduling.md](windows/process-scheduling.md)

## R–S

- **Scheduling Plan** — 三种独立方案之一（调度） → [three-plans.md](domain/three-plans.md)
- **场景（Scene）** ⚠ — 跨领域协调机制，**不是第四种方案** → [scenes-arbitration.md](domain/scenes-arbitration.md)
- **SchemaVersion** — 文档版本；高于支持版本时拒绝加载并保留原文件 → [serialization-persistence.md](data/serialization-persistence.md)
- **SMU** — AMD 系统管理单元，CO/FMax 经邮箱访问 → [amd-smu.md](hardware/cpu/amd-smu.md)
- **状态所有权（Owner）** — Scene/Plan/User/System；用户接管优先于自动恢复 → [state-ownership.md](domain/state-ownership.md)
- **步进量** ⚠ — 无量纲档位，无可靠 mV 换算 → [amd-curve-optimizer.md](hardware/cpu/amd-curve-optimizer.md)

## T–U

- **三种独立方案** — CPU Tuning / Power / Scheduling，各有生命周期 → [three-plans.md](domain/three-plans.md)
- **UVP（Undervolt Protection）** — Intel 12 代起降压保护，与 VBS/HVCI 组合时禁用运行时降压 → [intel-oc-mailbox.md](hardware/cpu/intel-oc-mailbox.md)

## 数值状态词（跨规范）

Verified（已核实）· Inferred（有依据推断）· Proposed（Zentwik 建议/约定）· Unverified（未核实）· Deprecated（已废弃）。定义见 [README.md](README.md)。

## 数据来源与设备状态词（实测记录，跨规范）

数据来源七类：**User-provided**（用户提供，未独立复核）· **Direct observation**（有原始日志支持的直接观测）· **Reproduced**（可按步骤复现）· **Derived**（推导）· **Inferred**（推断）· **Example**（示例）· **Unknown**（来源不明）。

设备状态：出厂 / BIOS 默认 / 当前 BIOS 配置 / 当前 SMU 状态 / 用户设置后 / 软件运行时修改后 / 重启后 / 恢复默认后 / 状态未知。

> 读取到非零值 ≠ 出厂值；读取到零值 ≠ 默认状态；回读一致 ≠ 实际生效。定义与判定规则见 [measurement-evidence-schema.md](data/measurement-evidence-schema.md)。
