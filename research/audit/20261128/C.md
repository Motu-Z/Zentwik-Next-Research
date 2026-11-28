# 分片 C：电源（方案 CRUD / 备份恢复 / 隐藏模板 / 参数 / 活动切换 / 功耗墙）只读静态审计

- **旧项目基准**：`Motu-Z/Zentwik@main` (`727462601911609e6dacbe3cb8f144246829087a`)
- **新项目对照**：`Motu-Z/Zentwik@feat/gui-protocol` (`100e9c0f4f9e7f9b160f8b0bed9ea99afdf20922`)
- **研究文档基准**：`Motu-Z/Zentwik-Next-Research@main` (`7680e5109ed010e4681180393ff85fb2ce412554`)

## 1. 闭合计数

- **旧项目电源方法数**：**75**（`PowerPolicyService` 29 + `PowerPolicyApi` 36 + `PowerPolicyDefaults` 2 组 + `PowerPolicySnapshot` 1 + `PowerSchemePreset` 2 + `SmuReader` 功耗墙 0 + `PowerViewModel` 14 UI 命令 - 构造/常量口径或含 14 P/Invoke，核心服务方法 53 + UI 命令 14）。
- **新项目电源 op 与服务方法数**：**11 个 op**（7 `power.scheme.*` + 2 `hardware.power.limit.*` + 2 `plans.apply/revert`）+ **78 个服务/提供者/领域方法**（`PowerSchemeService` 23 + `PowerSchemeBackupStore` 6 + `Win32PowerPolicyService` 19 + `PowerSchemePresets` 2 + `PowerLimits` 4 + `AmdSmuTuningProvider` 功耗墙 24）。
- **对照项总数**：**16 项**（`C-01` ~ `C-16`）。

## 2. 逐项对照表

| 编号 | 文档要求(路径:行) | 旧项与证据(文件:函数/行) | 新项与证据(文件:函数/op) | 状态 | 验证级别 | 备注 |
|---|---|---|---|---|---|---|
| C-01 | `power-management.md:41-94`：区分可见枚举与注册表全部方案，子键数 `>0` 过滤 overlay 占位键 | `PowerPolicyApi.cs:430-471`（仅枚举可见） | `Win32PowerPolicyService.cs:40-117`；`PowerSchemeService.cs:61-71,423-432`；`power.scheme.list`（UI/CLI 均已接线） | 一致 | 静态源码 | `observed` |
| C-02 | `power-management.md:48,60,81`：4 内置模板 GUID 保护禁删 | `PowerSchemePreset.cs:42-99`；`PowerPolicyService.cs:594-595` | `PowerSchemePresets.cs:7-29`；`PowerSchemeService.cs:131-134` | 一致 | 静态源码 | `observed` |
| C-03 | `power-management.md:49-52,66,127-131`：隐藏模板派生回退与 `--activate` | `PowerPolicyService.cs:481-556`（无条件回退） | `PowerSchemeService.cs:73-109`；`Ops.PowerSchemeCreate`（缺 `--activate` 参数） | 部分 | 静态源码 | `observed`：缺 `power create --activate` |
| C-04 | `power-management.md:24,79-80,92`：`PowerWriteFriendlyName` 改名与查重 | `PowerPolicyService.cs:559-577,624-635`（24 字符） | `PowerSchemeService.cs:111-127,444-458`（64 字符）；`Ops.PowerSchemeRename` | 一致 | 静态源码 | `observed` |
| C-05 | `power-management.md:26,81,113,142`：删前强制备份，切走活动方案至可见方案 | `PowerPolicyService.cs:589-621`（无删前备份） | `PowerSchemeService.cs:129-165,415-420`；`Ops.PowerSchemeDelete` | 一致 | 静态源码 | `observed` |
| C-06 | `power-management.md:95-113`：JSON 清单 + `.pow` 双轨备份与恢复 | 无 | `PowerSchemeBackupStore.cs:26-80`；`PowerSchemeService.cs:172-329`；`power.scheme.backup/backups/restore`（UI/CLI 已接线） | 新有旧无 | 静态源码（9955HX 2026-11-26 实测记录） | `observed` |
| C-07 | `power-management.md:19-20,66,140`：活动方案查询与主动切换 | `PowerPolicyService.cs:421-459`；`PowerViewModel.cs:1328-1376` | `Win32PowerPolicyService.cs:21-38,119-124`（仅内部）；**`PowerSchemeService` 与 `AgentSession` 无切换 op** | 缺 | 静态源码 | `observed`：无法从 CLI/Desktop 主动切换系统活动电源方案 |
| C-08 | `power-management.md:21,28,33-39`：8 项策略参数 AC/DC 独立读写与回读 | `PowerPolicyService.cs:35-62,168-392`（8 项固定 GUID + 范围，写后不回读） | 仅 `PlanCoordinator.ApplyPower:1210-1301` 接通用 `PowerPlan`（有回读与 `StateLedger` 回退，无 8 项固定预设与范围校验，Desktop UI 无法编辑 8 项参数） | 部分 | 静态源码 | `observed` |
| C-09 | `power-management.md:26,39`：供电状态 `GetSystemPowerStatus` | `PowerPolicyApi.cs:474-492`（`HasBattery, OnAcPower`） | `Win32PowerStatus.cs:7-18`（仅 `OnAcPower`，缺 `HasBattery`） | 部分 | 静态源码 | `observed` |
| C-10 | `power-management.md:24,32,50`：说明文字读写 | `PowerPolicyApi.cs:321-376` | `Win32PowerPolicyService.cs:243-266` | 一致 | 静态源码 | `observed` |
| C-11 | `LEGACY_POWER_CONTROL_INVESTIGATION.md:203,339`：系统方案与 `PowerPlan` 解耦 | `PowerViewModel.cs:489-515`（耦合） | `PowerSchemeService` (`power.scheme.*`) 与 `PlanCoordinator.ApplyPower` (`plans.*`) 分离 | 一致 | 静态源码 | `observed` |
| C-12 | `amd-power-limit.md:13-38,129,133-134`：RSMU `0x56/0xCB/0x57/0x58`（mW/mA）功耗墙与 MP1 拒写 | 无（已在 `348712a` 删除） | `AmdSmuTuningProvider.cs:22-28,260-293,347-405`；`hardware.power.limit.read/apply`（受可选模块 `power-limits` 门控；UI/CLI 已接线） | 部分（有回读但无回退） | 静态源码（9955HX 管理员会话实测：`100e9c0`） | `observed`：写后回读完整，但**未接入 `StateLedger` 与 `Revert`**（有回读无回退） |
| C-13 | `amd-power-limit.md:40-67,115,120-125`：PM 表 `0x03` 重试 3 次与 `1 W` 下界 | 无 | `AmdSmuTuningProvider.cs:615-772`（`03f9997` 修复 `1e-23` 残留与 Win32 `1359` 重试） | 新有旧无 | 静态源码 + 9955HX 管理员会话实测（2026-11-28） | `observed` |
| C-14 | `amd-power-limit.md:69-101,116-118`：写后回读四态 + `±1 W` 探针 + 3 次复原 | 无 | `PowerLimits.cs:38-229`；`AmdSmuTuningProvider.cs:367-577`（`100e9c0` 落地） | 新有旧无 | 静态源码 + 9955HX 管理员会话实测（2026-11-28） | `observed` |
| C-15 | `amd-power-limit.md:109-110`：范围校验与熔断值读取（`0xDC/0xDA/0xDB`） | 无 | `PowerLimits.cs:79-123`；`AmdSmuTuningProvider.cs:318-345,775-792` | 新有旧无 | 静态源码 + 9955HX 实测（2026-11-27） | `observed` |
| C-16 | `LEGACY_POWER_CONTROL_INVESTIGATION.md:138-160,352`：Intel 功耗墙与 EC 排除 | 无 | 无 | 一致 | 静态源码 | `observed` |
