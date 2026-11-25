# 状态账本与所有权

- 规范名称：状态账本（State Ledger / Applied Target Ledger）
- 英文名称：State Ownership Ledger
- 缩写：账本
- 领域：`domain`
- 核验状态：Proposed（项目核心不变量，源自 Accepted ADR-005）
- 权威来源：本仓库 ADR-005、PROJECT_BRIEF、research/01 §5、ADR-006
- 相关文档：[scenes-arbitration.md](scenes-arbitration.md)、[three-plans.md](three-plans.md)、[../data/numeric-semantics.md](../data/numeric-semantics.md)

## 1. 定义

账本记录每个被写对象的完整写入上下文，用于冲突消解、退出恢复、失败重试与审计。它不是普通日志。

## 2. 账本字段（Proposed）

| 字段 | 含义 |
|---|---|
| Target | 操作对象（哪个硬件/设置项） |
| PreviousValue | 前值（写入前实际读到的值） |
| DesiredValue | 目标值 |
| ActualValue | 回读的实际值 |
| Source | 来源（SMU 读出/用户输入/推断） |
| Owner | 所有权：Scene / Plan / User / System |
| Priority | 优先级 |
| Restorable | 是否可恢复 |
| Failure | 失败原因 |

## 3. 核心不变量（Verified 设计 + 测试）

1. 用户手动接管优先于自动恢复：`Owner=User`、`Restorable=false`，场景退出不得覆盖。
2. `ActualValue` ≠ `DesiredValue` 即判定未生效，不写"已应用"。
3. 回退失败保留 `Applied` + `Restorable=true` 供重试，不标"已恢复"。
4. 高优先级接管保留原始前值（后续恢复回到最初状态）。
5. 漂移检测把所有权交给用户。
6. 写入失败不得写成已应用。

已覆盖测试：前值记录、实际值不符判失败、用户接管拦截、低优先级不可抢占、回退失败保持所有权、释放接管、注入时钟、方案版本变更跳过。

## 4. 持久化（ADR-006）

- 账本分文件存放，跨文件一致性需协调（无跨文件事务）。
- 文档带 `SchemaVersion`；高版本拒绝加载并保留原文件。
- 服务重启后须对账（Unverified）。

## 5. 数值语义

前值/目标值/实际值必须同单位同类型比较；带单位的物理量比较需容差。见 [numeric-semantics.md](../data/numeric-semantics.md)。

## 6. 常见误解

- "账本=日志" → 账本带所有权与恢复资格，是状态机。
- "快照回写即可" → 会覆盖用户后续修改。
- "回退失败就标恢复" → 应保留可重试状态。

## 7. 核验状态

字段与不变量 = Proposed（已 Accepted 并有单测）；服务重启对账、休眠唤醒复核 = Unverified。
