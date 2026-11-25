# 硬件参数分层记录规范（Parameter Record Schema）

- 规范名称：硬件参数分层记录规范
- 英文名称：Hardware Parameter Record Schema
- 领域：`data`
- 核验状态：Proposed（Zentwik 长期写作规则）
- 权威来源：本次专项技术事实审计（见 [AUDIT.md](../AUDIT.md)）
- 相关文档：[numeric-semantics.md](numeric-semantics.md)、[units.md](units.md)、[_TEMPLATE.md](../_TEMPLATE.md)、[AI_GUIDE.md](../../AI_GUIDE.md)

## 1. 为什么需要这份规范

本次审计暴露的根本问题不是某个数值写错，而是**技术文档只记录一个孤立的数字，没有记录这个数字属于哪一层**。把"常见调校值"当"固件限制"、把"软件输入钳制"当"硬件上限"、把"BIOS 菜单范围"当"芯片固有属性"，都是同一类错误的不同表现。仅把 ±30 改成 ±50 不能防止下一次在 PPT、PL1/PL2、GPU 电压或 DDR5 时序上重犯。

因此规定：**任何硬件参数在文档中出现时，必须说明它属于哪一层；层与层之间不可互相替代。**

## 2. 参数分层模型（五层，必须区分）

| 层 | 含义 | 谁定义 | 可否当规范限制 |
|---|---|---|---|
| L1 硬件/固件实际允许范围 | 硅片/固件邮箱真实接受的编码范围 | 芯片设计 + 固件（AGESA/微码） | 是（经核验后），但须注明平台/代际 |
| L2 软件输入/钳制范围 | 某版本工具在输入侧自行 clamp 的范围 | 工具实现者 | **否**，只是该软件的选择 |
| L3 实际稳定可用范围 | 具体芯片个体在给定条件下不崩溃的范围 | 物理个体 + 散热/电压/负载 | **否**，非规范值 |
| L4 典型调校值 | 社区/经验常见起点或目标值 | 社区经验 | **否**，不得作为限制或官方推荐 |
| L5 项目内部约定 | Zentwik 选择的字段/类型/单位/默认策略 | Zentwik | 是，但须显式标注为项目约定 |

> 关键：L1 才接近"技术事实"，且必须带平台条件；L2/L3/L4 都不是硬件事实，标注为官方限制即属错误。

## 3. 参数记录结构（每个硬件参数必须填写的字段）

```
参数：<名称>
语义：<英文/官方名称，如 Voltage/Frequency Curve Offset>
数据类型：<signed integer / unsigned / fixed-point / enum ...>
单位：<无 / mV / MHz / W / step ...；无单位要写"无">
L1 固件/硬件范围：<最小值..最大值，单位，步进，适用代际/固件/BIOS 条件，来源，核验状态>
L2 软件输入范围：<哪个版本/工具钳制到多少，来源>
L3 实际稳定范围：<取决于哪些条件；标注 Unverified，不填具体数>
L4 典型调校值：<仅参考，明确"非规范限制">
适用条件：<CPU/代际/AGESA/BIOS/工具>
权威来源：<官方/维护者/社区，尽量精确到章节>
核验状态：Verified / Inferred / Proposed / Unverified / Deprecated
Zentwik 约定：<字段名、类型、单位后缀、是否持久化，标注 Proposed>
```

无法确认的层填 `Unverified` 并说明验证方式，**不得自行填数以凑齐表格**。

## 4. 权威定义位置（单一维护点）

以下参数已有完整分层记录，是本仓库的权威定义；其他文件只能引用，不得复制其数值表：

- **AMD Curve Optimizer（CO）**：[amd-curve-optimizer.md](../hardware/cpu/amd-curve-optimizer.md) §5
- **FMax / Boost Clock Override**：[amd-smu.md](../hardware/cpu/amd-smu.md) §4
- **Intel OC Mailbox 电压偏移**：[intel-oc-mailbox.md](../hardware/cpu/intel-oc-mailbox.md) §4
- **RAPL 功率/能量/时间单位**：[units.md](units.md) §1

引用格式示例：`见 amd-curve-optimizer.md §5 的分层记录`（以文字注明权威位置即可），不要在引用处重抄范围数字。

## 5. 写作规则（写入 AI_GUIDE 与模板）

1. 出现数值范围时，先问"这是 L1…L5 哪一层"，答不出就是定义缺陷。
2. 一个范围必须同时给出：最小值、最大值、单位、步进、适用硬件/代际、固件或 BIOS 条件、来源、核验状态。
3. 无单位整数（枚举、位域、档位、序号）同样适用本规范，不得因"没有单位"就省略类型与层。
4. 同义不同源的名称（FMax / Fmax Override / Auto OC / Boost Clock Override）须指出是否同一参数，不得直接画等号。
5. 经验值写进 L4 并标注"非规范"；软件钳制写进 L2 并标注工具版本；两者都不得冒充 L1。
6. 单位换算的方向（乘/除、2^n 还是 1/2^n）必须写明，方向错误会造成数量级偏差（见 [AUDIT.md](../AUDIT.md) 的 RAPL 案例）。
