# 三种独立方案

- 规范名称：三类领域方案（Three Independent Plans）
- 英文名称：CPU Tuning Plan / Power Plan / Scheduling Plan
- 领域：`domain`
- 核验状态：Proposed（项目核心不变量，源自 Accepted ADR-005 与 PROJECT_BRIEF）
- 权威来源：本仓库 ADR-005、PROJECT_BRIEF、research/01 §1/§5
- 相关文档：[scenes-arbitration.md](scenes-arbitration.md)、[state-ownership.md](state-ownership.md)、[../terminology/core-vocabulary.md](../terminology/core-vocabulary.md)

## 1. 定义（核心不变量）

三类方案是三个**独立领域对象**，各有自己的 ID、版本、应用与回滚路径；不存在跨越三者的"当前方案"概念。

| 方案 | 领域 | 主要写对象 |
|---|---|---|
| CPU Tuning Plan | CPU 调校 | AMD CO 步进 / FMax；Intel 电压偏移 / 倍频（会话级） |
| Power Plan | 电源 | Windows 电源计划 GUID + 策略项 |
| Scheduling Plan | 调度 | CPU Sets / 亲和性 / 优先级 |

## 2. 为什么独立（历史教训，Verified）

旧版把"调校+调度+电源"塞进一条规则，且明确"退出场景不恢复"；`PlanService` 先保存 `AppliedAt` 再写硬件，写失败时磁盘显示已应用。这导致实际生效配置与产品记录长期不一致。

## 3. 方案态势四层（Proposed）

定义（Definition）/ 期望状态（Desired）/ 观测状态（Observed）/ 硬件实际（Actual）。应用须回读验证：`ActualValue` ≠ `DesiredValue` 即判未生效。

持久化：
- 方案用 `planType` 判别字段存多态，不用"万能方案对象 + 可空字段"。
- 文档带 `SchemaVersion`，高于当前支持版本时拒绝加载并保留原文件。
- `PersistsAcrossReboot` 由 Provider 能力决定：AMD CO 为 true，Intel MSR 为 false。

## 4. 场景不是第四种方案

场景只声明触发条件、引用哪些方案的哪一版、优先级与退出策略。详见 [scenes-arbitration.md](scenes-arbitration.md)。

## 5. 常见误解

- "有一套当前方案" → 三个领域各有当前态，无跨领域"当前方案"。
- "调校方案包含出厂基线" → 基线是硬件属性，单独存储并记录来源/时间/硬件身份。
- "Intel 方案可持久化" → 当前为会话级，需新建重放链路才能持久化。

## 6. 核验状态

三类独立与场景定位 = Proposed（项目核心约定，已 Accepted）。
