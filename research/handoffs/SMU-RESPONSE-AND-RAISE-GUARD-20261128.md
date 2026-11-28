# SMU 响应码通道确认与 TDC/EDC 拉高保护设计预研

- **日期**：2026-11-28
- **基线**：
  - 产品新分支：`Motu-Z/Zentwik` `feat/gui-protocol @ fc55708`（含 `8d1201a`、`84c41c5`、`fc55708`）
  - 研究仓库：`Motu-Z/Zentwik-Next-Research` `main @ b53f1ca`（含 `research/handoffs/M4-底层接手-20261126.md`、`knowledge/hardware/cpu/amd-power-limit.md`）
  - 外部上游源码：`namazso/PawnIO.Modules`（`RyzenSMU.p`）、`FlyGoat/RyzenAdj`（`lib/nb_smu_ops.h:30-34`）、`irusanov/ZenStates-Core`（`Issues #20` / `Zen4Settings.cs`）
- **定位声明**：本报告为纯静态源码预研，**不修改产品代码、不执行构建/硬件测试、不替用户裁决**。

---

## 一、任务 A：SMU 响应码通道确认

### A.1 结论总表

| # | 核查项 | 证据（文件:行） | 级别 | 结论 |
|---|---|---|---|---|
| A-1 | `output[0]` 是否为 SMU 原始响应码；`MessageTest` 判 `2` vs RyzenAdj `OK=1` | `namazso/PawnIO.Modules` `RyzenSMU.p` (`ioctl_send_smu_command`、`send_command`)；`AmdSmuTuningProvider.cs:915,1003-1013`；`PawnIoSession.cs:101-125`；`RyzenAdj` `nb_smu_ops.h:30-34` | `observed` | **`output[0]` 不是 SMU 响应码**，而是回传的 `ARG0` 参数寄存器；`MessageTest(0x01, [1])` 返回 `probe[0]=2` 是固件 `ARG0` 回显（`1+1=2`），与 `REP_MSG_OK=0x1` **不是同一量纲**。 |
| A-2 | `WriteLimit` 最小改动点与三态映射；「响应码 OK 是否等价于值生效」 | `AmdSmuTuningProvider.cs:410-419,1074-1086`；`PowerLimits.cs:38-51`；`amd-power-limit.md:73-79` | `observed` | **响应码 OK 明确不等价于值生效**（仅表示邮箱收下命令）。`WriteLimit` 不读 `output[0]`，应从 `TrySend` 传出的 Win32 错误码映射 `0xFF/0xFE/0xFD/0xFC`。 |
| A-3 | 间接观测 `0x57`/`0x58` 电流限制生效的手段穷举 | `amd-power-limit.md:67,155-168`；`AmdSensorSource.cs`；`AmdSmuTuningProvider.cs` | `observed` | **`[未找到]`** 工具内任何可定量证明 `0x57`/`0x58` 写入值真实生效的间接手段。 |
| A-4 | 与 `M4-底层接手-20261126.md:157` 结论的关系 | `M4-底层接手-20261126.md:157`；`research/02-hardware-access.md:57`；`RyzenSMU.p` `switch (s)` | `observed` / `derived` | **部分细化，未完全推翻**：`RyzenSMU.p` 确实将 4 种非 OK 响应码分别映射到 4 个不同 `NTSTATUS`（对应不同 Win32 错误码），但 C# 侧仍拿不到原始 `NTSTATUS` / `SMU_RSP` 字节。 |

### A.2 详细证据与施工输入

1. **PawnIO `RyzenSMU.p` 真实映射（`observed`）**：
   - `RyzenSMU.p` 中 `ioctl_send_smu_command` 调用 `send_command(in[0], args)`：
     - 若 `!NT_SUCCESS(status)`，模块**直接返回 `status`**（不填充 `out[0..5]`），由 Windows I/O 管理器经 `RtlNtStatusToDosError` 转为 Win32 错误码供 `PawnIoSession.Execute`（`PawnIoSession.cs:112-115`）读取；
     - 若 `NT_SUCCESS(status)`，模块将 `args[0..5]` 拷入 `out[0..5]` 并返回 `STATUS_SUCCESS`。
   - `RyzenSMU.p` 内 `SMU_RSP` → `NTSTATUS` → Win32 错误码对照（其中 Win32 转换值由 Windows `NTSTATUS` 映射规则推导，`derived`）：
     | SMU `RSP`（`RyzenAdj`） | `RyzenSMU.p` 返回 `NTSTATUS`（`observed`） | 预期 Win32 错误码（`derived`） | `AmdSmuTuningProvider.cs:1076-1083` 现状（`observed`） |
     |---|---|---|---|
     | `SMU_OK (0x1)` | `STATUS_SUCCESS` | `0`（`Execute` 返回 `true`） | 成功分支 |
     | `SMU_Failed (0xFF)` | `STATUS_UNSUCCESSFUL` | `31` (`ERROR_GEN_FAILURE`) | 已映射 `31` |
     | `SMU_UnknownCmd (0xFE)` | `STATUS_INVALID_INFO_CLASS` | `87` (`ERROR_INVALID_PARAMETER`) | **缺失**（落入默认分支） |
     | `SMU_CmdRejectedPrereq (0xFD)` | `STATUS_INTERNAL_ERROR` | `1359` (`ERROR_INTERNAL_ERROR`) | **注释误判**：`:1081` 误写成 `ERROR_ALREADY_EXISTS`（实为 `183`） |
     | `SMU_CmdRejectedBusy (0xFC)` | `STATUS_DEVICE_BUSY` | `170` (`ERROR_BUSY`) | 已映射 `170` |
2. **`WriteLimit`（`AmdSmuTuningProvider.cs:410-419`）最小改动点**：
   - 给 `TrySend` 增加重载或 `out int win32Code` 输出参数；
   - 三态映射对应 `PowerLimitWriteState`（`PowerLimits.cs:38-51`）：
     - `Execute == true`（`SMU_OK 0x1`）→ `PowerLimitWriteState.Accepted`（等待 PM 表回读；TDC/EDC 保持 `Accepted`）；
     - `code == 87`（`0xFE UnknownCmd`）→ `PowerLimitWriteState.Rejected`，`Detail` 明确写「命令不被平台/邮箱支持（`0xFE`）」；
     - `code == 170`（`0xFC Busy`）或 `code == 1359`（`0xFD Prereq`）→ `PowerLimitWriteState.Rejected`（注明暂忙或前置条件未满足，可按需对 `170` 短重试）；
     - `code == 31`（`0xFF Failed`）或 `121`（超时）→ `PowerLimitWriteState.Rejected`。
   - 同步修正 `DescribeFailure`（`:1076-1083`）补充 `87` 并更正 `1359` 的 Win32 名称说明。
3. **间接观测手段穷举（均无法定量证明 `0x57`/`0x58` 生效）**：
   - PM 表 `0x00621202`：`amd-power-limit.md:67,155-168` 仅有功率与未知字段，无 TDC/EDC 限值与电流遥测偏移（`[未找到]`）；
   - RAPL（`0xC0010299/9B`）与 SMN 温度（`0x00059800`）：测的是封装总功耗（W）与 Tctl 温度（°C），不是 VDD 电流（A）；
   - `APERF/MPERF`（`0xC00000E7/E8`）：仅观测平均频率，无法区分限频源自 PPT、TDC、EDC 还是温度墙；
   - `AMDFamily17.bin`：未读 SVI3/VRM 电流传感器（`[未找到]`）。
4. **与 `M4-底层接手-20261126.md:157` 的关系**：
   - `M4:157` 针对的是 `DeviceIoControl` 只拿 `GetLastWin32Error()`、拿不到原始 `NTSTATUS`/`SMU_RSP` 字节。
   - **未推翻底层事实**（C# 侧仍拿不到原始 `NTSTATUS`，且模块内部其他校验失败也可能映射到 `87`/`31` 造成混叠）；但**细化了可行性**：上游 `RyzenSMU.p` 确实把 `0xFF/0xFE/0xFD/0xFC` 拆成了 4 个不同 `NTSTATUS`，在 Windows 上预期映射为互不冲突的 `31/87/1359/170`（待管理员会话实测确认 `87`）。

---

## 二、任务 B：TDC/EDC 拉高保护设计审查

### B.1 结论总表

| # | 审查维度 | 证据（文件:行） | 级别 | 结论与要点 |
|---|---|---|---|---|
| B-1 | 基准选择完整分支（TDC vs EDC） | `AmdSmuTuningProvider.cs:318-344`；`PlanCoordinator.cs:1302-1312` (`8d1201a`) | `observed` / `derived` | **关键发现**：`TryReadFusedPowerLimits`（`:339`）**对 `EdcAmps` 硬编码传 `null`**（RSMU `0xDB` 为 VDD TDC、`0xD9` 为 SOC TDC，无 VDD EDC 熔断命令）。因此 **EDC 没有出厂熔断值兜底**，未声明影子基线时必然进入「无基准」分支。 |
| B-2 | 协议 / DTO / CLI / UI 最小改动点 | `Protocol.cs`、`Dtos.cs:218`、`AgentSession.cs`、`Commands.cs`、`MainViewModel.cs`、`MainForm.cs` | `observed` / `derived` | 需覆盖 **`hardware.power.limit.apply` 与 `plans.apply` 双入口**，否则用户在方案页应用高 TDC/EDC 的 `CpuTuningPlan` 会漏过或无法确认；错误码新增 `ErrorCodes.PowerLimitRaiseConfirm = "power.limit_raise_confirm"`。 |
| B-3 | 回退豁免与 `MarkRevertedUnverified` 共存 | `PlanCoordinator.cs:1465-1497,1999-2015,2108-2111` (`8d1201a`)；`StateLedger.cs:346-362` | `observed` | `RestoreUnconfirmedPowerLimit` 与 `ReapplyTarget` 必须走内部直写/豁免标记，**严禁**触发拉高拦截，否则「60 A → 回退 108 A」会自锁，导致 `MarkRevertedUnverified` 无法释放所有权。 |
| B-4 | 警告文案与验证级别 | `strings.*.json` (`fc55708`)；`ProcessAndPowerLimitTests.cs` | `observed` / `derived` | 三语键须包含请求值、基准/无基准状态、熔断值非安全线提示；逻辑拦截与回退豁免可 `fake` 单测，真实 SMU 响应与弹窗交互须 Windows 实测。 |
| B-5 | UI 责任边界表达 | `AUDIT-LEGACY-VS-NEXT-20261128.md §2.3`（毛坯控件口径） | `derived` | 遵守 WinForms 裸控件口径，用标准警告 `MessageBox` + 明确动作按钮（「确认拉高并自担风险」），明示无硬件读回、无供电裕量校验、后果自负。 |

### B.2 逐条详细设计输入

#### 1. 基准选择的完整分支（TDC 与 EDC 各自独立判定）
- 对请求中的 `TdcAmps`（若非 `null`）：
  1. 若 `PowerBaseline?.TdcAmps is int shadowTdc` → 基准 `refTdc = shadowTdc`（来源：`shadow`）；
  2. 否则尝试 `_powerLimits.TryReadFusedPowerLimits(out var fused, out _)`，若 `fused.TdcAmps is int fusedTdc` → 基准 `refTdc = fusedTdc`（来源：`fused`，须标注「出厂硬件上限，非散热安全线」）；
  3. 否则（未声明且 `0xDB` 读取失败或返回 `null`）→ **无基准**。
- 对请求中的 `EdcAmps`（若非 `null`）：
  1. 若 `PowerBaseline?.EdcAmps is int shadowEdc` → 基准 `refEdc = shadowEdc`（来源：`shadow`）；
  2. `FusedPowerLimits.EdcAmps` 当前恒为 `null`（`AmdSmuTuningProvider.cs:339`，无 VDD EDC 熔断命令 `[未找到]`）；
  3. 否则 → **无基准**。
- **放行 / 拦截判定**：
  - 有基准且 `requested <= refVal`（拉低或持平）→ 直接通过；
  - 有基准且 `requested > refVal`（拉高），或**无基准**：若 `allowRaise == false`，服务端拒绝写入并返回 `ErrorCodes.PowerLimitRaiseConfirm`（在细节/负载中写明哪一项触发：超过影子基线 / 超过熔断值 / 无基准）。

#### 2. 协议 / DTO / CLI / UI 最小改动点（按 `fc55708`）
- **`src/Zentwik.Contracts/Protocol.cs`**：`ErrorCodes` 新增 `PowerLimitRaiseConfirm = "power.limit_raise_confirm"`。
- **`src/Zentwik.Contracts/Dtos.cs`**：
  - `PowerLimitApplyRequest` 新增 `bool AllowRaise = false`；
  - `PlanApplyRequest`（或 `AgentSession` `Ops.PlansApply` 请求体）同步评估增加 `bool AllowRaise = false`（若仅改 `PowerLimitApplyRequest`，`CpuTuningPlan` 路径需明确如何传确认标志）。
- **`src/Zentwik.Agent/AgentSession.cs` & `PlanCoordinator.cs`**：
  - `Ops.PowerLimitApply` 与 `Ops.PlansApply`（当方案含 `TdcAmps` 或 `EdcAmps` 时）统一调用服务端保护校验；不满足时抛 `CoreException(ErrorCodes.PowerLimitRaiseConfirm, message)`。
- **`src/Zentwik.Cli/Commands.cs` & `Help.cs`**：
  - `powerlimit set` 与 `plans apply` 增加 `--allow-raise` 开关；未传且命中 `power.limit_raise_confirm` 时打印警告并提示加 `--allow-raise` 确认（退出非零）。
- **`src/Zentwik.Desktop/ViewModels/MainViewModel.cs` & `MainForm.cs`**：
  - `ApplyPowerLimitsAsync`（及 `ApplyPlanAsync`）捕获 `AgentRequestException` 且 `ErrorCode == ErrorCodes.PowerLimitRaiseConfirm` 时，回调 UI 弹出 `MessageBox.Show(..., MessageBoxButtons.OKCancel, MessageBoxIcon.Warning)`；用户确认后带 `AllowRaise = true` 重发。

#### 3. 回退豁免与 `MarkRevertedUnverified` 共存
- `PlanCoordinator.RestoreTarget`（`8d1201a:2108-2111`）→ `RestoreUnconfirmedPowerLimit`（`:1465-1497`）→ `_powerLimits.ApplyPowerLimits` 属于内部回退路径，**不得走拉高保护**（或显式传入 `bypassRaiseGuard: true`）。
- 这样「声明基线 108 A → 应用 60 A（影子推进到 60 A）→ `revert` 写回 108 A」不会被 `108 > 60` 误拦，写后正常进入 `Restore`（`:2005`）调用 `_ledger.MarkRevertedUnverified` 释放 `Owner=None` 并保留备注，同时保持 `fc55708` 已验证的基线声明/清除同步逻辑不变。

#### 4. 警告文案要点与验证级别
- **三语键要点**（`strings.zh-CN.json` / `strings.zh-TW.json` / `strings.en-US.json`）：
  1. 明确列出触发原因（如「请求 TDC {0} A 高于当前基准 {1} A」或「未声明 EDC 基线，无法确认是否超出当前上限」）；
  2. 明确提示「出厂熔断值仅为芯片硬件上限，不代表当前主板供电/散热安全线」；
  3. 明确提示「继续写入可能导致供电过载、降频、异常关机或硬件损坏，后果由您自行承担」。
- **验证级别边界**：
  - **可 `fake` / 静态验证**：拉低放行、拉高/无基准拦截 `power.limit_raise_confirm`、`AllowRaise=true` 放行、熔断读取失败回退到无基准、`RestoreUnconfirmedPowerLimit` 回退豁免与 `MarkRevertedUnverified` 释放所有权。
  - **须真机验证**：9955HX 管理员/Session 0 下 `0x57`/`0x58` 写入响应码（`87/1359/170/31`）、`0xDB` 熔断读取与 WinForms 弹窗交互。

#### 5. 责任边界在 UI 的表达
- 遵守 WinForms 毛坯控件口径，使用标准 `MessageBoxIcon.Warning` 确认框：标题标明风险提示，正文如实说明「本工具无电流回读通道、无法检测主板 VRM/散热实时裕量，确认继续即由用户自行承担硬件与系统稳定性风险」，按钮避免模糊的「确定」，改用明确动作语义或 `OKCancel` 配明确提示句。

---

## 三、计数闭合

- **任务 A 核查项**：**4 / 4**（`output[0]` 语义与量纲、最小改动与生效判据、间接观测穷举、`M4:157` 关系），未找到项均已标 `[未找到]`。
- **任务 B 审查项**：**5 / 5**（基准分支、全链路改动点、回退豁免与 `MarkRevertedUnverified` 共存、三语警告与验证边界、UI 责任边界表达）。
