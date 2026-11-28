# 旧版场景自动化设计分析（参考结论，非施工方案）

- **日期**：2026-11-28
- **定位**：围绕旧项目 `Motu-Z/Zentwik` `main @ 727462601911609e6dacbe3cb8f144246829087a` 的整个场景自动化机制进行只读设计分析，提炼优点与可借鉴之处供新版后续演进参考；**不含施工步骤、不改产品代码**。
- **对照基线**：
  - 旧项目：`Motu-Z/Zentwik` `main @ 727462601911609e6dacbe3cb8f144246829087a`
  - 新版：`Motu-Z/Zentwik` `feat/gui-protocol @ 74df31a`（仅作现状对照）
  - 公开参考：`JamesCJ60/Universal-x86-Tuning-Utility`、`seerge/g-helper`

---

## 一、六个维度逐项分析

### 1. 触发模型
- **触发源与零轮询门槛（`observed`）**：`ScenarioTrigger`（`ScenarioService.cs:20-41`）仅包含 4 种事件：`ProcessRunning`、`OnBattery`、`OnAcPower`、`ProcessExited`。注释（`:15-19`）明确将「CPU 占用/温度」等需常驻采样的指标排除在外，守住「无人看界面时后台零采样占用」约束。
- **电平 vs 边沿语义（`observed`）**：前三者为电平条件（「此刻成立不成立」）；`ProcessExited` 为一次性边沿脉冲（`:34-40`），仅在进程消失的那一拍进入活跃集合，下一拍自动移出。
- **组合能力（`observed`）**：`ScenarioRule`（`:46-57`）每条规则仅持有一个 `Trigger` + 一个 `Argument` 字符串，**不支持**单条规则内多条件 `AND/OR` 组合。
- **进程名匹配（`observed`）**：`ActiveTriggerComparer`（`:822-835`）按 `StringComparer.OrdinalIgnoreCase` 精确匹配进程文件名（`ProcessRefresher.cs:444-445` 统一补全 `.exe`），**不支持**完整路径或通配符匹配；`PipeServerService.cs:2225-2232` 强制校验进程类必填进程名、电源类必须留空。
- **退出事件采集与去重（`observed`）**：`ScenarioMonitor.AppendExits`（`ScenarioMonitor.cs:253-284`）仅针对 `ScenarioService.ProcessExitArguments`（`:235-260` 仅启用中的 `ProcessExited` 规则去重名）维护 `_lastSeen`，做「上一拍在、这一拍不在」的差集；`800ms` 防抖窗口（`:57`）合并游戏启动时的子进程爆发；采集失败（`seen is null`）时直接跳过退出判定，防止一次采样异常触发全部退出规则（`:265-268`）。

### 2. 规则生命周期与仲裁
- **CRUD 与持久化（`observed`）**：`ScenarioService.Load/SetEnabled/Save/Delete`（`ScenarioService.cs:273-359`）整批序列化为单一 `scenarios.json`（`ScenarioFile` 含总闸 `Enabled` 与 `List<ScenarioRule>`，`:106-110`）。
- **引用完整性与孤儿清理（`observed`）**：
  - 关总闸或删规则时调用 `ReleaseScenarioBindings` 主动解绑相关进程调度（`:312-318,354`）；
  - 被引用的方案/掩码/电源计划被删时，`DetachPlan`（`:370-388`）**只置空规则中对应的动作字段**（「保持原样」），保留规则本身与其余动作，避免死引用每次命中都在日志里刷失败告警。
- **优先级与多场景冲突仲裁（`observed`）**：`Priority` 数值越小越优先（`:77-83`）；`ResolveOutcomeCore`（`:738-775`）先以 `HashSet` 去重活跃触发，再筛选 `Enabled && Matches && hasAction` 并按 `OrderBy(Priority)` 取**唯一胜出者**整体执行调校/调度/电源三项动作（`:727-729`），避免多条规则分别接管不同模块导致状态撕裂。

### 3. 进入与回退
- **进入与幂等去重（`observed`）**：`ApplyProfile`（`:469-473`）比对 `_plans.ActiveId`、`ApplyPowerScheme`（`:655`）比对 `GetActiveScheme()`、`ApplyScheduling`（`:512-527,602-614`）以 `maskId|argument|pid@ticks`（含进程创建时间防 PID 复用）去重，避免 `800ms`/2s 重评时重复下发。
- **退出与无回退语义（`observed`）**：旧版**不保存前值、不负责退出回退**（`:37-38`）。`ProcessExited` 的脉冲触发模型恰好与无回退约定自洽：用户通过显式配置一条「`ProcessExited` → 切回日常/节能方案」规则来表达退出后的目标状态，而非由引擎自动回退。
- **所有权与手动接管（`observed`）**：无 `StateLedger`；因 `ApplyProfile`/`ApplyPowerScheme` 按系统当前活动方案去重（`:460`），若用户在无新事件期间手动切走方案，旧版不会强行改回，但一旦触发条件重新评估命中仍会切回规则目标。

### 4. UI 呈现与可排查性
- **两级触发选择器（`observed`）**：`ScenariosViewModel.cs:123-192` 将 4 种底层枚举组织为「大类（进程/电源）→ 状态（启动时/退出时 vs 拔下时/接通时）」两级下拉，切换至非进程大类时自动隐藏并清空 `EditorProcessName`（`:162-166`），杜绝「界面隐藏参数却被带入保存」的脏数据；并提供运行中进程搜索建议下拉（`PickProcessCommand`）。
- **规则列表可读性（`observed`）**：`ScenarioRuleItem`（`:1019-1052`）对停用规则使用 `Dimmed` 淡化展示但保留完整内容；`TriggerLabel` + `TriggerValue`（如 `IF 进程退出` / `game.exe`）分离展示条件与目标；引用方案缺失时显示 `（已删除）` 而非空白（`ScenarioService.cs:98-102`）。
- **「为什么没触发」的可排查性（`observed`）**：后端 `DescribeHit`（`ScenarioService.cs:788-812`）在日志写明「已应用方案 X ← 触发源」；`ResolveOutcome`（`:724`）设计为纯函数便于预览，但 `ScenariosViewModel.cs` 实际未接入 `ResolveOutcome` 也无 `LastOutcome` 字段（UI 仅能看到规则启用状态与动作配置，排查未命中仍需看日志）。

### 5. 可测性分层
- **纯判定单元测试（`observed`）**：因 `ScenarioService`（纯判定）与 `ScenarioMonitor`（线程/防抖/事件采集）解耦（`ScenarioMonitor.cs:10-13`），`ScenarioServiceTests.cs:124-168` 可在不启动任何真实进程的情况下，直接断言 `ProcessExited` 在 `ProcessRunning` 时不命中、仅在 `ProcessExited` 脉冲时命中、停用后立即移出 `ProcessExitArguments`。
- **VM 交互集成测试（`observed`）**：`IntegrationTest.cs:1890-1960` 直接驱动 `ScenariosViewModel`，验证两级下拉联动、进程搜索回填、切到电源自动隐藏搜索框、空进程名与三动作全空拦截、默认优先级递增。这种「Domain 纯函数单测 + VM 表单状态机集成测试」的分层有效隔离了平台进程噪声。

### 6. 对照新版（`feat/gui-protocol @ 74df31a`）
- 新版 `Scene.cs`、`SceneEngine.cs` 已将核心模型升级为 `Scene`（多 `SceneTrigger` / 多 `SceneAction` / `RestoreOnExit`）+ `_latched` 闩锁 + `StateLedger` 前值账本与用户接管保护 + 降序优先级仲裁（`SceneArbitration`），且后台已完整支持 `ProcessExited`（`Scene.cs:14`、`SceneEngine.cs:41-48`）；当前差距集中在 UI 入口（`MainViewModel.TriggerKinds` 缺 `ProcessExited`、无两级选择器与进程候选）及部分生命周期细节（全局总闸、`DetachPlan` 级联清理）。

---

## 二、「值得借鉴的设计」清单（按价值排序，非按实现难度）

1. **按需差集与零空转事件编排（`ScenarioMonitor.cs:15-24,253-284`）**：仅当存在启用的 `ProcessExited` 规则时才维护 `_lastSeen` 且只盯规则点名的进程集合，采样失败跳过退出判定；避免全表百余进程无意义比对与误触发。
2. **「大类 → 状态」两级触发 UI 与联动清参（`ScenariosViewModel.cs:123-167`）**：把底层扁平枚举收束为语义清晰的两级下拉，切类自动隐藏并清空无关参数，前端直接拦截「空进程名」与「全空动作」保存（`IntegrationTest.cs:1936-1947`）。
3. **单一胜出者整体仲裁 + 触发源归因日志（`ScenarioService.cs:727-729,788-812`）**：多条件同时命中时不拆分模块给不同规则，保证用户心智清晰；日志写明 `← 触发源` 且对丢失引用做重复告警抑制（`:480-493`）。
4. **纯判定与事件采集解耦的可测架构（`ScenarioMonitor.cs:10-13`、`ScenarioServiceTests.cs:124-168`）**：判定逻辑无 IO/无定时器，使边沿脉冲与监视名单缩减都能在纯单测里 100% 确定性验证。
5. **方案删除后的优雅降级 `DetachPlan`（`ScenarioService.cs:370-388`）**：只置空被删动作字段、保留规则与其余动作，UI 显示 `（已删除）` 而非留死 ID 刷错。
6. **子进程爆发 `800ms` 防抖与 `pid@ticks` 调度去重（`ScenarioMonitor.cs:57`、`ScenarioService.cs:602-614`）**：合并启动期子进程风暴，同时用创建时间区分 PID 复用。
7. **停用规则灰显保留配置（`ScenariosViewModel.cs:1019-1020`）**：`Dimmed` 视觉降权但保留可读性，方便用户排查与随时恢复。

---

## 三、三栏对照表（新版 `74df31a` vs 旧版 `7274626`）

| 新版已有（`feat/gui-protocol @ 74df31a`） | 旧版独有（`main @ 7274626`） | 旧版有但新版刻意不做 / 已被更优机制替代 |
|---|---|---|
| `SceneEngine` 后端完整支持 `ProcessExited` 与 `_latched` 闩锁维持（`Scene.cs:14`, `SceneEngine.cs:41-48`） | `ScenariosViewModel` 两级下拉（进程/电源 → 启动/退出/拔下/接通）与进程名搜索候选 | **无 `StateLedger` 的「不回退」模型**：新版刻意引入 `StateLedger`、`RestoreOnExit` 与 `MarkUserTakeover`，取代旧版无法自动还原的设计 |
| `StateLedger` 全字段前值账本、退出回退、休眠核对与用户接管保护 | 场景全局总闸 `ScenarioFile.Enabled` 与按需挂钩监听（`Sync` 零占用） | **直接调 `Process.GetProcessesByName`**：旧版 `ScenarioService.FindProcessIds:697` 用 .NET 进程名遍历，新版统一收口到单次快照 + 账本隔离 |
| 多 `SceneTrigger` 与多 `SceneAction` 模型（`Scene.cs:46-48`）、降序优先级 + 进入时间仲裁（`SceneArbitration`） | `DetachPlan`（方案删除只置空规则对应动作字段而不拒删） | **规则与调校耦合在单机 UI/Worker**：新版改为 `LocalSystem` 服务宿主 + `LastOutcome` 持久态 |
| `Scene.LastOutcome` 状态记录与 Desktop `MainViewModel.TriggerKinds` 基础入口 | `DescribeHit` 触发源归因日志、`800ms` 启动防抖与 `IntegrationTest` VM 状态机全量自检 | **数字小优先级的旧排序**：新版刻意改为数值越大越优先（`OrderByDescending`），语义更直观 |

---

## 四、其他开源项目参考（公开 GitHub 检索）

| 项目 / 仓库 | 观察到的机制（`observed`） | 与 Zentwik 场景自动化的对照 |
|---|---|---|
| `JamesCJ60/Universal-x86-Tuning-Utility`（`AdaptiveViewModel.cs`、`Adaptive.xaml`） | 提供基于游戏/应用列表自动切换预设（`Auto switch app profile`）以及电源/充电状态自动切预设（`Preset on Battery Charge`），带轮询周期配置（`nudPolling`）。 | 属单向条件触发覆盖预设，未见 `StateLedger` 级前值回退与用户接管仲裁。 |
| `seerge/g-helper`（`AppConfig.cs`、`Fans.cs`） | 围绕电源接入/电池与性能模式自动应用功率与风扇曲线（`auto_apply_power`、`auto_apply`）。 | 面向华硕笔记本硬件模式联动，未提供通用用户可编排的「进程启动/退出」规则引擎。 |
| Process Lasso 等闭源商业工具 | `[未找到]`（无公开源码仓库可供逐行核对）。 | — |

---

## 五、计数闭合

- **分析维度**：**6 / 6** 全部覆盖（触发模型、规则生命周期、进入与回退、UI 呈现、可测性、对照新版）。
- **可参考设计条目**：**7 条**（按价值排序）。
- **对照表**：三栏完整闭合；外部无源码项目明确标注 `[未找到]`。
