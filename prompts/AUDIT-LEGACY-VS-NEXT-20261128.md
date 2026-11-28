# 任务 11（研究型）：旧项目 ↔ 新项目 ↔ 研究文档 逐项对照审计

> 执行者：Gemini（Linux 沙盒，有 GitHub MCP）。你是总协调者，按 `MASTER_COORDINATOR.md` §2 自行派发子 Agent。
> 下达方：本机 AI（Windows）。用户 2026-11-28 明确要求「逐项对照，不要遗漏，分 agent」，并要求由你执行。
> 本文件即唯一任务书。与产品仓库 `docs/coordination/gemini-task-01..10.md` 无关，那批已作废。

---

## 0. 一句话目标

以**旧项目实现**为基准、**新项目实现**为对照、**研究仓库文档**为第三极，逐项穷举并比对，
产出「已覆盖 / 部分 / 缺失 / 新有旧无 / 文档与现实冲突」的可追溯总表。只做静态审计，不改产品代码。

---

## 1. 仓库与基线（动手前先确认）

| 角色 | 仓库 / 分支 | 基线 |
|---|---|---|
| 旧项目（基准） | `Motu-Z/Zentwik` 的 `main` | 以 clone 到的 HEAD 为准，报告中写 SHA |
| 新项目（对照） | `Motu-Z/Zentwik` 的 `feat/gui-protocol` | 以 clone 到的 HEAD 为准，报告中写 SHA |
| 研究文档（第三极） | `Motu-Z/Zentwik-Next-Research` 的 `main` | `d3ed2cb`（2026-11-28 已推送） |

**基线说明**：`feat/gui-protocol` 已于 2026-11-28 15:37 推送至 `100e9c0`，与 GitHub 同步。
`9af3d03`（页签按老 UI 重排成七页、顶栏删掉）、`03f9997`（PM 表读数恒 0 的判据修复）、
`100e9c0`（功耗重复应用不再误报）三项均已在远端，对照时按此基线，不要误判为缺失。

**访问边界**：旧项目 Windows 本地 `D:\Projects\Zentwik Project` 你无法访问，只用 GitHub `main`；
本机 AI 会在汇总阶段补查本地正本。不要扫描磁盘、不要猜本地路径。

---

## 2. 口径与硬边界（违反即返工）

1. **旧项目是基准，不是待推翻的草稿。** 只写「有/无、用 A 还是 B、已知缺陷」，禁止价值判断。
2. **只读审计**：不改产品源码、不改协议/DTO/配置、不建分支、不 push 产品仓库、不跑构建/测试/真机
   （新项目 Windows-only，Linux 沙盒编译不了，报告里说明是静态对照）。
3. **UI 最新口径（用户 2026-11-28 拍板，覆盖旧文档）**：
   - 桌面端是 `src/Zentwik.Desktop` 的 **WinForms 裸控件**，只用 Label/Button/TextBox/ComboBox/
     CheckBox/DataGridView/GroupBox/TableLayoutPanel/SplitContainer；无主题、无强调色、无缩放、
     无自绘、无动画、无第三方 UI 包。Flutter/Dart 已整体删除。
   - **设置页必须保留，里面要放东西**；现在没实现或没对接不等于不做。
   - 界面先不处理 i18n ≠ 只做简体中文；**文案不许硬编码**，走 `Zentwik.Localization`。
   - 研究仓库远端新增的 UI 规范/原型（`AI_UI_DESIGN_GUIDANCE.md`、`UI_UX_DESIGN_SPEC.md`、
     v1–v3.4 原型群）标 **Deprecated，仅记差异**，不得当作当前要求。
4. 按 `AI_WORK_RULES.md` P0 读序：`knowledge/INDEX.md` → 领域规范 → 旧项目具体函数 →
   参考项目具体函数 → 一句话判断。不用 README 摘要代替源码。
5. 区分 observed / derived / unknown；缺项写 `[未找到]`，不许留空，不许补全常识。

---

## 3. 「项」的定义与每行字段

一个「项」= 旧项目一个可观察能力（页签 → 区块 → 按钮 / 命令 / 自动行为 / 后台服务动作）。
新有旧无的能力也要单独成项。

每行固定字段：

```
编号 | 文档要求(路径:行) | 旧项与证据(文件:函数/行) | 新项与证据(文件:函数/op) |
状态 | 验证级别 | 备注
```

- 状态枚举：`一致` / `部分` / `缺` / `新有旧无` / `文档与现实冲突` / `[未找到]` / `[远端基线缺失，需本机补查]`。
- 验证级别：`静态源码` / `单测` / `fake` / `管理员会话` / `Session 0` / `真机闭环`。静态审计默认写 `静态源码`，
  引用旧报告里的实测结论时写清出处级别，不冒充自己验证过。

---

## 4. 防遗漏机制（三向闭合，必须执行）

**三向全集，每条轨都要独立穷举一遍：**

| 侧 | 四条枚举轨 |
|---|---|
| 旧实现 | ① 旧 UI 页签/菜单/按钮 ② 旧 CLI 命令与参数 ③ 旧 Core Services 公共方法 ④ 旧文档/`memory`/`Zentwik-Docs` 列出的能力 |
| 新实现 | ① `MainForm.cs` 页签/控件/命令绑定 ② CLI Help 与命令 ③ `AgentSession` op 与 capability ID ④ 配置/持久化字段、安装器行为、服务生命周期 |
| 文档 | ① `project/` 产品要求与迁移矩阵 ② `decisions/ADR-001…008` ③ `knowledge/` 可验证断言 ④ `research/` 报告与 handoffs 的「已验证 vs 声称」⑤ `KNOWN_LIMITS.md` 不修项 ⑥ 远端 UI 规范/原型（标 Deprecated） |

**三向差集：**
- 文档有、两实现都无 → 未落地
- 旧有、新无 → 缺口
- 新有、旧无 → 新增（说明用户价值与是否路线图）
- 三者都有 → 比实现细节、状态机、回读语义、错误处理

**计数闭合（汇总时逐条核对，不等就退回该领域复查）：**
页签数、旧 CLI 命令数、新 CLI 命令数、op 数、capability 数、服务方法数、ADR 条数、
`MIGRATION_MATRIX.md` 行数、每条规范的可验证断言数，都必须等于穷举表对应行数。

---

## 5. 子 Agent 分片（A–J，边界不重叠）

你是总协调者，按下列 brief 派发 10 个独立子 Agent（可合并同名领域，不可合并不同领域）。
每个子 Agent 只审计自己那片，产出表 + 证据路径，不写文件；你汇总后统一落盘。

| Agent | 领域 | 旧侧入口 | 新侧入口 | 完成标准 |
|---|---|---|---|---|
| A | UI 页签/导航/控件/交互（逐页逐控件） | 旧 `Zentwik.UI` 全部页签 | `MainForm.cs` + `ViewModels/MainViewModel.cs` | 旧页签/控件逐项列出，新页签逐项对照；按 §2.3 口径标注，UI 规范只记差异 |
| B | CPU 调校（CO/FMax/电压/频率/方案 CRUD） | 旧 Core 调校服务与 `SmuReader` | `AmdSmuTuningProvider` / `IntelMsrTuningProvider` / `PlanCoordinator.ApplyCpu` | 命令号、掩码、单位、写前读基线/写后回读逐项比对 |
| C | 电源（方案 CRUD/备份恢复/隐藏模板/参数/活动切换） | 旧 `PowerPolicyService` | `PowerSchemeService` / `IPowerPolicyService` / CLI `power` | GUID 判定、隐藏方案、备份/恢复语义逐项比对 |
| D | 进程与调度（枚举/亲和性/优先级/CPU Sets/规则） | 旧 `NativeProcessList` / `ProcessWatcher` / `ProcessInstanceLedger` | `AffinityScheduler` / `SchedulingPlan` / `SchedulingWatcher` / `PlanCoordinator.ApplyScheduling` | 枚举方式、句柄权限、PID 复用、读回语义、失败重试逐项比对 |
| E | 监控遥测（温度/功耗/逐核频率/利用率/历史/图表） | 旧监控服务与 `SmuReader` | `AmdSensorSource` / `TelemetryRingBuffer` / CLI `monitor`/`history` | 通道、单位、间隔、缓存、历史容量逐项比对 |
| F | 场景自动化（触发/仲裁/账本/回退/休眠唤醒） | 旧场景引擎 | `SceneEngine` / `StateLedger` / `ReconcileAfterResumeAsync` | 四类触发、优先级、所有权、用户接管、回退失败语义逐项比对 |
| G | 服务/IPC/协议/权限/安装升级/数据迁移/日志诊断 | 旧 Worker + 安装脚本 | `AgentServer` / `AgentSession` / `PipeIdentity` / `ServiceInstaller` / 安装器 | 帧格式、错误码、DACL/SID/完整性、恢复动作、升级路径逐项比对 |
| H | CLI/设置/i18n/托盘/自启/单实例 | 旧 CLI + 托盘 | `Zentwik.Cli` / `settings` op / `Zentwik.Localization` | 命令/退出码/参数逐项比对；按 §2.3 口径，明确设置页要放东西、文案不许硬编码 |
| I | 项目文档与决策 | `project/PROJECT_BRIEF`、`ARCHITECTURE_CONSTRAINTS`、`MIGRATION_MATRIX`、`RESEARCH_PLAN`、`PRODUCT_MIGRATION_AND_ROADMAP`、`KNOWN_LIMITS`、`INSTALLATION_DEPLOYMENT_MAINTENANCE`、`decisions/ADR-001…008`、`prompts/`、`templates/` | 同左对照两实现 | 每条要求/ADR 逐条对照，标「文档声称 vs 代码现实」 |
| J | 知识与研究文档 | `knowledge/INDEX` 与 `knowledge/` 全部规范（`data/`、`hardware/cpu/`、`windows/`、`architecture/`、`domain/`、`terminology/`、`ENGINEERING_NOTES`、`AUDIT`）+ `research/01…04`、`LEGACY_POWER_CONTROL_INVESTIGATION`、`research/handoffs/*` | 同左对照两实现 | 每条可验证断言对照代码常量与行为，标 Verified/Inferred/Unverified/Deprecated |

注意：I/J 的「新侧入口」列是文档自身，仍需把每条断言落到旧项目与新项目的具体源码证据上。

---

## 6. 汇总（你做，不交给子 Agent）

合并 10 份表 → 三向差集 + 计数闭合 → 三张差异表：
1. 文档要求 vs 旧现实
2. 文档要求 vs 新现实
3. 旧 vs 新

总表按「缺口 / 部分 / 新有旧无 / 冲突」分组，缺口按用户影响排序，每条给出建议下一步（仅建议，不实施）。
明确列出：无法从 Linux 沙盒验证的项、依赖 Windows 本地正本的项、你判定为 unknown 的项。

---

## 7. 交付物

写入研究仓库（本机 AI 已推送 `d3ed2cb` 作为基线，你的提交基于它）：

- `research/handoffs/LEGACY-VS-NEXT-AUDIT-20261128.md`：总表 + 三张差异表 + 缺口清单 + 未验证项。
- `research/audit/20261128/A.md` … `J.md`：10 份分报告（含各自全集计数与证据）。
- 提交只 add 本次文件，禁 `git add -A`；本机 git 无 user.name/email，提交时临时导出
  `GIT_AUTHOR_NAME/GIT_AUTHOR_EMAIL/GIT_COMMITTER_NAME/GIT_COMMITTER_EMAIL`
  （`Motu-Z <ZMotuic@gmail.com>`），不写 git config。
- 推到 GitHub 后在本文件末尾追加一行「已完成 + 提交 SHA + 报告路径」，便于本机确认。

---

## 8. 停止与升级条件

- 三个仓库之一无法访问 → 明确报告哪个、错误是什么，已能做的部分照常交付，不虚构。
- 某项证据冲突 → 保留冲突并写出「意图 vs 现实」，不自行裁决、不折中。
- 10 个分片全部有表且计数闭合后结束；不扩大范围到「实现补全」或「修复缺陷」。

## 9. 明确不做

不改产品代码、不改协议、不跑构建/测试、不碰老版安装、不写产品仓库文件、不把 UI 原型当作当前需求。
