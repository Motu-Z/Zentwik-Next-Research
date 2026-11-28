# TDC/EDC 当前生效值回读通道穷举调研与影子基线方案静态审查

- **日期**：2026-11-28
- **基线**：
  - 产品新分支：`Motu-Z/Zentwik` `feat/gui-protocol @ e843e7a`（含 `40c049a`、`8e45742`、`e843e7a`）
  - 研究仓库：`Motu-Z/Zentwik-Next-Research` `main @ a3ba2a4`（`knowledge/hardware/cpu/amd-power-limit.md`、`project/AI_WORK_RULES.md`）
  - 外部上游源码：`namazso/PawnIO.Modules`（`RyzenSMU.p`、`AMDFamily17.p`）、`irusanov/ZenStates-Core`（`Hardware/Smu/Settings/Zen4Settings.cs`、`PawnIo/RyzenSmu.cs`）、`FlyGoat/RyzenAdj`（`lib/api.c`）
- **定位声明**：本报告为纯静态源码与既有实测记录核查，**不修改产品代码、不执行构建/硬件测试、不替用户裁决方案**。

---

## 一、A 项：TDC/EDC 当前生效值读通道穷举结论表

| # | 排查路径 | 具体证据（仓库 / 文件:行） | 级别 | 是否找到 |
|---|---|---|---|---|
| A-0（既有复核） | RyzenAdj `lib/api.c` 与研究文档核验 | `FlyGoat/RyzenAdj` `lib/api.c`：`get_stapm_limit(0x0)`/`get_fast_limit(0x8)`/`get_slow_limit(0x10)` 仅读前 6 float；`get_vrm_current` 与 `get_vrmmax_current` 未映射 `0x00621202`，直接返回 `NAN`（且 `set_vrmmax_current` 无 `FAM_DRAGONRANGE`/`FAM_FIRERANGE` 分支）。`amd-power-limit.md:67,135,141` 明确「无对称 Get 命令、PM 表无原始请求字段 = Verified」。 | `observed` | **未找到** |
| A-1（补查 1） | PawnIO `RyzenSMU.bin` / `AMDFamily17.bin` 导出 IOCTL 列表 | `namazso/PawnIO.Modules` `RyzenSMU.p` 仅导出 `ioctl_get_smu_version`、`ioctl_send_smu_command`（仅 RSMU）、`ioctl_read_smu_register`/`ioctl_write_smu_register`、`ioctl_resolve_pm_table`、`ioctl_read_pm_table`、`ioctl_update_pm_table`，检索 `tdc`/`edc`/`current`/`limit` 为 **0 命中**；`AMDFamily17.p` 仅导出 `ioctl_read_msr`/`ioctl_write_msr` 与 `ioctl_read_smn`/`ioctl_write_smn`，检索 `tdc`/`edc` 为 **0 命中**。 | `observed` | **未找到** |
| A-2（补查 2） | RSMU/MP1 命令表（Zen4/5 Settings 逐行） | `irusanov/ZenStates-Core` `Hardware/Smu/Settings/Zen4Settings.cs`：RSMU 写入命令为 `SMU_MSG_SetPPTLimit=0x56`、`SetSlowLimit=0xCB`、`SetTDCVDDLimit=0x57`、`SetEDCVDDLimit=0x58`；RSMU `Get` 命令中与功耗/电流相关仅有出厂熔断上限 `GetPboFusedPowerLimit=0xDC`、`GetPboFusedApuSlowLimit=0xDA`、`GetPboFusedVrmVddTdcCurrent=0xDB`、`GetPboFusedVrmSocTdcCurrent=0xD9`（静态熔断值，非当前生效值），检索 `GetTDC`/`GetEDC`/`GetVrm` 为 **0 命中**。 | `observed` | **未找到** |
| A-3（补查 3） | PM 表全字段（本机 `0x00621202`，128 float）对照 dump 与 RyzenAdj | `amd-power-limit.md:151-168`：2026-11-28 本机 `0x00621202`（表长 `0x994`）dump 仅确认 `[0]/[2]/[4]`（STAPM/FAST/SLOW limit）、`[9]/[20]`（域功率）、`[18]/[19]`（电压）、`[16]`（未知）；`ZenStates-Core` 中 `0x621202` 连 `offsetCoresPower` 亦为 `-1`（未映射）；`RyzenAdj` `get_vrm_current`/`get_vrmmax_current` 无 `0x00621202` 分支。未交叉验证的剩余 float 中是否存在未公开 TDC/EDC 上限偏移目前为 **`unknown`**（无任何源码或文档依据）。 | `observed`（已有记录未映射） / `unknown`（未公开偏移盲扫） | **未找到**（`[未找到]` 已知偏移） |

**A 项总结论**：在 PawnIO 模块 IOCTL、Zen4/5 SMU 命令表及已知 `0x00621202` PM 表映射中，**确认不存在已验证的 TDC/EDC 当前生效值读通道**。

---

## 二、B 项：影子基线方案静态审查（逐条）

### B-1. 能否复用 `StateLedger` 条目（`StateLedger.cs`）而非新造持久化状态？
- **事实核对（`observed`）**：
  - `LedgerEntry`（`src/Zentwik.Domain/Ownership/StateLedger.cs:31-59`）**没有 `CurrentValue` 字段**，仅有 `PreviousValue`（`:37`）、`DesiredValue`（`:39`）、`ActualValue`（`:41`）。
  - `StateStore` 已持久化 `StateLedger.Entries` 到 `state.json`（`PlanCoordinator.cs:73`）。
  - `PlanCoordinator.ApplyPowerLimits`（`40c049a:1287-1296, 1367-1371` & `8e45742`）当前硬编码将 `tdc`/`edc` 的 `Previous` 传为 `null` 并调用 `_ledger.MarkNotRestorable(...)`。
- **可复用路径与冲突点**：
  - **可复用点（`derived`）**：若用 `LedgerEntry.ActualValue` 表达「当前影子值」、用 `PreviousValue` 表达「本次方案写前基线」，可在不新增独立持久化文件的情况下复用 `StateStore` 落盘能力。
  - **冲突点 1（`Acquire` 入参依赖，`observed`）**：`StateLedger.Acquire`（`StateLedger.cs:94-152`）要求调用方传入 `currentValue` 作为非持有状态下的新 `PreviousValue`（`:139`）。当前 `ApplyPowerLimits` 传 `null` 且 `ReadTarget`（`40c049a:2001-2006`）对 `tdc`/`edc` 返回 `null`，必须改由协调器在无硬件读通道时回退读取影子值。
  - **冲突点 2（方案②语义与 `PreviousValue` 冲突，`derived`）**：方案②「每次写入成功后基线更新为本次写入值」若直接改写 `PreviousValue`，会导致 `revert` 写回本次写入值而非写前原值；正确分工应为：**`PreviousValue` 保持写前基线不变，`ActualValue` 在写入成功后更新为本次写入值**（被拒不更新）。
  - **冲突点 3（跨整机重启失效，`observed`/`derived`）**：`PlanCoordinator.cs:2042` 明确 SMU DPTC 功耗墙为**会话级**（重启由固件恢复）。若 `state.json` 跨整机重启保留了上次的 `ActualValue` 影子基线，重启后实际硬件已回出厂值而影子基线仍是旧值，必须在检测到新开机周期时使会话级影子基线失效。

### B-2. `revert` 后条目保留还是清除？下轮 `apply` 能否继承前值？
- **现状（`observed`）**：`StateLedger.MarkReverted(..., success: true)`（`StateLedger.cs:303-312`）**保留**条目在 `_entries` 中，将 `Owner = OwnerKind.None`、`Status = LedgerStatus.Reverted`、`ActualValue = restoredValue`、`Restorable = false`，不会清除。
- **继承结论（`derived`）**：
  - 在现有 `40c049a` 代码下，下轮 `apply` **无法继承**（硬编码传 `null`）。
  - 若在同一次开机周期内保留 `Reverted` 条目，且 `ApplyPowerLimits` 改从 `_ledger.Get(target)?.ActualValue` 读取影子当前值作为新一轮 `Acquire` 的 `currentValue`，则下轮 `apply` **可以继承**回退后的基线值；若条目被清除或跨整机重启，则无前值，只能退回 `NotRestorable`。

### B-3. 与不变量 5、场景联动、`D-15`/`D-02` 的相互作用
1. **不变量 5（`Owner=User` 不可自动恢复，`AI_WORK_RULES.md:83`）**：
   - **高风险冲突（`observed`）**：`StateLedger.MarkUserTakeover`（`StateLedger.cs:236-262`）将 `Owner` 设为 `OwnerKind.User` 且 `Restorable = false`，此后所有方案/场景 `Acquire` 均返回 `DeniedUserOwned`（`:108-110`）被永久阻断。因此**用户显式声明 TDC/EDC 初始基线绝不能走 `MarkUserTakeover`**，只能以 `Owner=None` 的基线记录或独立配置字段保存。
2. **场景联动（`SceneEngine.cs:414-418` & `StateLedger.cs:88-90, 303-312`）**：
   - **关键状态机冲突（`observed`）**：方案③要求「`revert` 写回基线值，无回读通道故标 `Unknown` 而非 `Applied`」。但当前 `MarkReverted`（`StateLedger.cs:303-312`）成功时固定设 `Owner = OwnerKind.None` + `Status = LedgerStatus.Reverted`；如果把 `revert` 后的条目状态改成 `LedgerStatus.Unknown`，且未同步把 `Owner` 置为 `OwnerKind.None`，`StateLedger.IsHeld`（`Owner is Scene or Plan && Status is Applied or Unknown`，`:88-90`）会判定它**仍被持有**，进而卡死低优先级场景或导致 `PlanRevert` 重复回退！若将 `Owner` 置 `None` 而 `Status = Unknown`，则需确认 UI/DTO 与 `IsHeld` 对 `Owner=None && Status=Unknown` 的语义一致性。
   - **激活计数兼容（`observed`）**：`40c049a` 已让 `SceneEngine.ActivateCoreAsync`（`SceneEngine.cs:417`）将 `PlanApplyState.Unknown` 计为激活成功（`applied++`），因此 `apply` 阶段 `Accepted → Unknown` 与场景联动兼容。
3. **`D-15`（调度退避）与 `D-02`（CPU Sets 互斥清理）**：
   - 均作用于 `sched.{pid}.*`，与 `cpu.power.limit.*` 正交；仅共享 `PlanCoordinator._gate` 锁，无逻辑冲突。

### B-4. 验证级别边界
- **可 `fake` / 静态验证**：影子基线 `apply → update → revert(Unknown) → re-apply` 状态流转、跨整机重启影子基线失效、未声明基线时保持 `NotRestorable`、`Owner=User` 隔离、序列化兼容。
- **必须真机验证**：RSMU `0x57`（TDC）/ `0x58`（EDC）在 9955HX 及目标机型的实际 SMU 响应码、写入是否被固件折算或静默钳制（无回读下只能靠负载电流遥测或 `0xDB` 熔断上限间接对照）、Session 0 `LocalSystem` 行为。

### B-5. 若不复用 `StateLedger`，最小新增状态的落点
- **落点（`derived`）**：在 `src/Zentwik.Domain/State/StateDocument.cs` 的 `StateDocument` record 新增可空属性，如：
  `public PowerLimitShadowBaseline? PowerLimitBaseline { get; init; }`（记录 `TdcAmps`、`EdcAmps`、`BootSessionId` / `BootTimeUtc`）。
- **Schema 兼容（`observed`）**：按 `AI_WORK_RULES.md:96`，STJ 对旧 `state.json` 缺失字段反序列化为 `null`，无需数据迁移脚本；在 `PlanCoordinator` 启动/恢复时比对 `IBootInfo` 开机标识，跨整机重启自动清空会话级影子基线。

---

## 三、给 Deepseek 的两条施工要点分支（不替用户裁决）

### 分支 1：若后续真机找到 TDC/EDC 读通道（例如盲扫 `0x00621202` 剩余 float 命中）
1. 在 `amd-power-limit.md` 补录实测偏移、交叉验证证据与容差，再扩展 `PowerLimitCurrent`（`PowerLimits.cs:19`）增加 `TdcAmps` / `EdcAmps`。
2. 在 `AmdSmuTuningProvider.TryReadCurrentPowerLimits` 填入读数（继续受 `>= 1` 有效下界约束，防 `1e-23` 残留）。
3. 删除 `PlanCoordinator.ApplyPowerLimits` 中 `tdc`/`edc` 传 `null` 与 `MarkNotRestorable` 特判，纳入与 `FAST/SLOW` 相同的 `RestorePowerLimit` 比例收敛回退与 `ReadTarget` 漂移核对。

### 分支 2：若确认无读通道，采用「影子基线」方案
1. **明确状态落点**：决定复用 `LedgerEntry.ActualValue`（需确保 `PreviousValue` 不被覆盖、`revert` 释放 `Owner=None` 时不触发 `IsHeld` 误判）或新增 `StateDocument.PowerLimitBaseline`（推荐评估跨整机重启清空逻辑）。
2. **明确交互入口**：在协议/设置/CLI 中提供「声明当前 TDC/EDC 基线」入口（禁止复用 `MarkUserTakeover`），未声明时保持现有 `MarkNotRestorable` 行为不变。
3. **跨整机重启失效**：结合 `IBootInfo` 在新开机周期自动清除上一会话的 `TDC/EDC` 影子基线，避免重启后固件已复位但影子基线仍按旧值回退。
4. **`revert` 状态语义**：若 `revert` 标 `LedgerStatus.Unknown`，须确保 `Owner` 释放为 `OwnerKind.None` 并核查 `PlanCoordinator.RevertPlanAsync` 与 UI `PlanRow` 的展示逻辑。

---

## 四、计数闭合

- **A 项补查路径**：**3 / 3** 全部覆盖（PawnIO IOCTL 列表、RSMU/MP1 Zen4/5 命令表、`0x00621202` PM 表全字段与 RyzenAdj 对照），另复核既有 `amd-power-limit.md` 结论，无遗漏；未公开偏移盲扫明确标 `unknown` / `[未找到]`。
- **B 项审查条目**：**5 / 5** 全部覆盖（账本复用可行性与冲突、`revert` 后保留与继承、不变量 5/场景/`D-15`/`D-02` 交互、验证级别边界、独立最小落点）。
