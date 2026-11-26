<!-- 迁入研究仓库：2026-11-26。原始版本来自私有仓库 Motu-Z/2AI，原文件 SHA-256/Blob SHA 记录见下方说明。为保证研究仓库可独立复核，正文保留原报告内容。 -->

# 旧版 Zentwik 功耗控制实现及其删除原因：历史与技术调查报告

**调查日期**：2026-11-26  
**调查对象**：旧版 Zentwik（历史名称 ZenForge / Zenwik）仓库 `D:\Projects\Zentwik Project\Zentwik`，`main` 分支，HEAD `bc3c59b`  
**用途**：供新版 Zentwik 迁移规划使用。本报告只还原事实、核实限制、界定边界，不涉及方案设计或实现建议。

---

## 0. 证据分级与结论标注约定

本报告对每一条关键结论标注证据等级，避免把历史记录、推断和实测混为一谈：

| 标记        | 含义           | 判定标准                                                                           |
| --------- | ------------ | ------------------------------------------------------------------------------ |
| **【代码】**  | 已由代码证实       | 当前工作树或某个具体 Git 对象中可直接读取到类、方法、常量、命令字符串或 diff                                    |
| **【测试】**  | 有实际测试记录      | 工作日志 / 交接文档中明确记载了实测数值与结果；**注意**：本次调查在工作区内未找到对应的原始 `worker-*.log`，故属于"记录中的实测结论" |
| **【文档】**  | 历史文档或对话记录的结论 | 来自开发日志、交接文档、架构审计文档，未独立复现                                                       |
| **【推断】**  | 基于现有证据的推理    | 有依据但无直接证据链闭合                                                                   |
| **【未验证】** | 无证据支持        | 找到了提及但没有实测、实现或日志                                                               |

**根本约束**：云端对话检索（`conversation_search`）以功耗、SMU、HX、EC 等多组关键词查询均返回 0 条，时间范围 2026-11-19 至 2026-11-26。本报告的历史结论全部来自**本地**工作日志、Git 历史与项目文档。据此可以确认的事实与无法确认的部分，在第七章"证据缺口"中逐条列出。

---

## 1. 调查范围与证据来源

### 1.1 已查阅的证据

**本地记忆（项目工作日志，主证据）**

- `D:\Projects\Zentwik Project\.workbuddy\memory\2026-11-21.md`：AMD 功耗发布、can7 整体移除的完整决策记录（第 3–46 行）
- 同目录 `2026-11-20.md`：Intel 功耗限制落地（第 216–221 行）
- 同目录 `2026-11-24.md`：OC 定频探测与固件限制（第 21–23 行）
- 同目录 `MEMORY.md`：当前架构红线与已移除功能摘要（第 33、41–42 行）
- 同目录 `2026-11-03.md` 至 `2026-11-25.md`：SMU 通道、电源计划、平台通用化等旁证

**项目文档**

- `Zentwik/CHANGELOG.md`：0.8.2、0.8.3、0.8.5、2.6.0-can1～can7 各版本条目
- `Zentwik-Docs/Zentwik-HANDOFF.md`：第 3b、3c 节记录 Intel 调校与功耗移除结论
- `Zentwik-Docs/Zentwik-第二轮技术审计.md`（在 `Zentwik-Docs/` 下）、`Zentwik-模块化架构审计-第一阶段.md` 与 `Zentwik-模块化架构审计-第二阶段.md`（两者在工作区根目录）：证明当前仓库中不存在 EC / ACPI / BIOS 专属实现
- `Zentwik 产品能力盘点与演进路线（DeepSeek）.md` 等：证明新版规划中功耗采集仍为"未实现"

**Git 历史（`D:\Projects\Zentwik Project\Zentwik`，origin `git@github.com:Motu-Z/Zentwik.git`）**

- 提交信息检索：`git log --grep` 命中 8 个直接相关的提交
- 符号检索：`git log -S 'PowerLimit'` 命中引入、删除与重构提交
- 逐文件读取：`git show <commit>:<path>` 读取删除前的完整实现  
  |  |  |  |  |  |  |  
  | --- | --- | --- | --- | --- | --- |
- 全历史代码检索：`git grep -i 'mechrevo\|uniwill\|mqtt\|13688\|gcu\|ec_'` 覆盖前 400 个提交的全部源码

### 1.2 关键时间线

| 日期         | 提交        | 事件                                             |
| ---------- | --------- | ---------------------------------------------- |
| 2026-10-29 | `48b54fd` | ZenForge 0.8.2：首次新增 PPT / TDC / EDC / 温度墙四项功耗墙 |
| 2026-10-29 | `d0872e4` | 0.8.3：功耗墙卡片上移至 FMax 之前                         |
| 2026-10-30 | `2999517` | 1.0.0-can4：引入电源方案（Windows 电源计划管理，与硬件功耗控制无关）    |
| 2026-10-30 | `575867e` | 0.8.5：**第一次删除**功耗墙 UI 与写入                      |
| 2026-11-12 | `defab5a` | 自研 SMU 层替换 ZenStates-Core（2.3.0）               |
| 2026-11-20 | `84bb998` | 2.6.0-can2：Intel 平台功耗限制                        |
| 2026-11-21 | `27dc193` | 2.6.0-can3：AMD 平台功耗限制                          |
| 2026-11-21 | `348712a` | 2.6.0-can7：**最终删除** AMD 与 Intel 功耗限制           |

Tag 已核对真实存在：`v2.6.0-can1`→`a7673da`、`can2`→`84bb998`、`can3`→`27dc193`、`can7`→`348712a`。

---

## 2. 原有架构与实现

历史上存在**两代相互独立**的功耗控制实现。两者代码不连续，删除理由、技术路径、运行形态都不同。迁移规划必须区分，不能视为同一功能的两次迭代。

### 2.1 第一代：ZenForge 0.8.2～0.8.4（单体应用，基于 ZenStates-Core）

**【代码】引入提交 `48b54fd`，13 个文件，+1490 行。**

#### 2.1.1 功能与参数

| 参数   | 单位 | 作用                             | 源码位置（提交 `48b54fd`）                                    |
| ---- | -- | ------------------------------ | ----------------------------------------------------- |
| PPT  | mW | 封装功耗上限（Package Power Tracking） | `Services/PowerLimitModels.cs` → `PowerLimitKind.Ppt` |
| TDC  | mA | VDD 持续电流上限                     | 同上 → `PowerLimitKind.Tdc`                             |
| EDC  | mA | VDD 峰值电流上限                     | 同上 → `PowerLimitKind.Edc`                             |
| Tctl | ℃  | 温度上限                           | 同上 → `PowerLimitKind.Tctl`                            |

**【代码】合法区间**（`PowerLimitBounds`）：PPT 5–500 W；TDC / EDC 1–500 A；Tctl 40–115 ℃；回读容差 ±2 ℃。

**【代码】三项设计约束**（`PowerLimitService` 类注释）：

1. 逐项独立写入，任一项失败不中断其他项；
2. 固定顺序 PPT → TDC → EDC → Tctl，理由是"先收紧温度墙会让后续写入被 SMU 反复拒绝"；
3. 不自动开启 OC 模式。UXTU 遇到 `CMD_REJECTED_PREREQ` 会先发 enable-oc，本实现识别前置条件失败后原样上报，由用户到 BIOS 开启 PBO。

#### 2.1.2 类、驱动与通信路径

**【代码】** 核心类：

- `Services/PowerLimitService.cs`：`Apply()` / `RestoreDefaults()` / `WriteOne()` / `Describe()`，逐项写入、错误分三层（不支持 / 前置条件不满足 / 瞬时故障）
- `Services/PowerLimitModels.cs`：`PowerLimitKind`、`PowerLimitOutcome`、`PowerLimitApplyResult`、`PowerLimitSet`、`PowerLimitBounds`
- `Helpers/PowerBaselineStore.cs`：出厂基线持久化（`power-baseline.json`）
- `Services/PowerBaselineBootstrap.cs`：启动时后台采集基线，CPU 或 BIOS 变更时作废旧基线
- `Services/SmuReader.cs`：`SetPowerLimit()` / `ReadPowerLimit()` / `CaptureBaseline()`
- `Services/TuningApplyService.cs`、`Services/SilentModeRunner.cs`：应用入口
- `Views/TuningPage.xaml(.cs)`：功耗墙 UI

**【代码】硬件通信**：`SmuReader.SetPowerLimit()` 调用 `ZenStates.Core` 第三方库的 `Cpu.SetPPTLimit()` / `SetTDCVDDLimit()` / `SetEDCVDDLimit()` / `SetTctlMax()`，返回 `ZenStates.Core.SMU.Status`（`OK`、`UNKNOWN_CMD`、`CMD_REJECTED_PREREQ`、`TIMEOUT_*`、`FAILED` 等）。邮箱握手与 PawnIO 交互封装在该库内部，**具体 SMU 命令号不在本项目源码中**，本次调查未反编译该 DLL。

**【代码】读取能力不对称**：PPT / TDC / EDC 无读取接口，`ReadPowerLimit()` 恒返回 0；仅 Tctl 通过 `Cpu.GetSystemPowerLimit().TemperatureLimit` 取系统供电策略的温度上限，且该值与 `SetTctlMax` 并非同一设定项，故必须带容差。出厂基线取 `Cpu.GetPboFusedLimits()` 的 PPT / TDC，**EDC 无熔断通道，永久不参与恢复**。

#### 2.1.3 完整调用链

**【代码】**

```
[TuningPage 应用] → TuningApplyService.Apply → PowerLimitService.Apply(reader, set)
  → （按 Ppt→Tdc→Edc→Tctl 顺序）WriteOne
  → SmuReader.SetPowerLimit(kind, value)
  → ZenStates.Core Cpu.SetPPTLimit / SetTDCVDDLimit / SetEDCVDDLimit / SetTctlMax
  → SMU 邮箱（库内）→ PawnIO → CPU SMU

[静默自启 -silent] → SilentModeRunner.Run → TuningStore.HasPowerLimits
  → 同上 PowerLimitService.Apply

[启动基线] → PowerBaselineBootstrap.RunInBackground → SmuReader.CaptureBaseline
  → Cpu.GetPboFusedLimits / GetSystemPowerLimit → PowerBaselineStore.Save(power-baseline.json)
```

#### 2.1.4 平台差异

**【未验证】** 第一代没有按 family/model 分命令组，`ZenStates.Core` 库内部处理平台差异。本项目的源码与文档中**未找到**该代支持平台清单、真机型号或逐平台测试结果。README（`48b54fd`）只笼统描述功能，无平台限定说明。

### 2.2 第二代：Zentwik 2.6.0-can2 / can3（Worker + IPC 架构）

#### 2.2.1 Intel 分支（提交 `84bb998`，tag `v2.6.0-can2`）

**【代码】参数与硬件路径**：

- 持续功耗 `PowerLimit1Watts`（PL1）、短时功耗 `PowerLimit2Watts`（PL2）、短时功耗时长 `TauSeconds`
- 上限写 MSR `0x610`，单位取自 MSR `0x614` bits 3:0（每格 = 2^-unit W，读不到按 1/8 W）
- PL1 / PL2 各占 15 位，bit 15 / 47 为 enable；时间窗 7 位，编码为 `2^Z × (1 + Y/4)`，**raw 0 表示"不限"**，因此 1 秒不能编码为 0（单测捕获过此问题）
- 其余位（含 clamp）在 `ComposePowerLimit()` 中保持原样
- 通道：`Zentwik.Core/Smu/IntelMsrDevice.cs` 的 `ioctl_read_msr` / `ioctl_write_msr`，经 PawnIO 模块 `IntelMSR.bin`

**【代码】调用链**（Intel 功耗复用 `apply-tuning`，**未新增独立 IPC 命令**）：

```
TuningViewModel.Intel.cs → WorkerBridge.ApplyTuningAsync
  → IPC apply-tuning（IntelTuningPayload 携带 powerLimit1Watts / powerLimit2Watts / tauSeconds）
  → PipeServerService → SmuReader.ApplyTuning
  → IntelTuningSession.ApplyPowerLimits
  → IntelTuningCodec.ComposePowerLimit → IntelMsrDevice.TryWrite(MSR 0x610)
  → PawnIO IntelMSR.bin → CPU MSR
  → 回读 0x610 逐项校验
```

**【文档】** 该分支**写入路径从未真机验证**：本机是 Zen5（AMD），PawnIO 按厂商过滤模块，在 AMD 机器上加载 `IntelMSR.bin` 返回错误 50。只有单测与逻辑校验（`Zentwik-Docs/Zentwik-HANDOFF.md` 第 95 行）。

#### 2.2.2 AMD 分支（提交 `27dc193`，tag `v2.6.0-can3`）

**【代码】参数**：STAPM 持续功耗、FAST 短时功耗、SLOW 平均功耗，单位毫瓦。合法范围 5000–300000 mW（5–300 W），回读容差 1500 mW，PM 表按 1 W 取整。

**【代码】命令组**（`Zentwik.Core/Smu/SmuPowerLimit.cs` → `SmuPowerLimitMessages.For(familyId, modelId)`）：

- APU 组：`0x14` / `0x15` / `0x16`
- Fire Range / Dragon Range 组：`0x4F` / `0x3E` / `0x5F`（对应 `family=0x19,model=0x61` 与 `family=0x1A,model=0x44`）
- APU 覆盖范围：`(0x17, 0x60/0x68/0x90/0x91/0xA0)`、`(0x19, 0x40/0x44/0x50/0x74/0x75/0x78)`、`(0x1A, 0x20/0x24/0x60/0x70)`
- 未匹配任何一代 → `IsSupported=false`，整段 UI 不显示

**【代码】读数**：`SmuSession.ReadPowerLimits()` 先 `SmuChannel.TryResolvePmTable()` 解析 PM 表 DRAM 地址（只解析一次），再 `TryReadPmTable(rows[64])`，用 `SmuPowerLimitCodec.FromWatts()` 把每个 qword 的低 32 位当 float 解码。对应 PM 表的 `float[0]` / `[2]` / `[4]`。

**【代码】调用链**：

```
TuningViewModel.PowerLimits.cs → RefreshPowerLimitsAsync / ApplyPowerLimitsAsync
  → WorkerBridge.QueryPowerLimitAsync / ApplyPowerLimitAsync
  → IPC query-power-limit / apply-power-limit（协议 20→21）
  → PipeServerService.RunQueryPowerLimitAsync / RunApplyPowerLimitAsync
  → SmuReader.ReadPowerLimits / ApplyPowerLimits
  → SmuSession.ReadPowerLimits / ApplyPowerLimits
  → SmuChannel.TryReadPmTable / Send(ioctl_send_smu_command)
  → PawnIoDevice.Execute → PawnIO RyzenSMU.bin → CPU SMU 固件
```

写入后 `VerifyPowerLimits()` 最多重试 4 次、每次间隔 40 ms，因为 PM 表由固件周期性刷新。

**【代码】失败映射**（`SmuSession.DescribeCommandFailure`）：`0x8007054F` → "固件拒绝该命令，本机可能由厂商固件接管功耗"；`0x8007000D` → 固件不认识；`0x80070032` → 平台不支持；`0x80070015` → 设备未就绪。

**【文档】读数索引曾被误判为 bug**：开发过程中一度认为"功耗值 index 硬编码"是缺陷，随后对照 RyzenAdj 源码纠正——`get_stapm_limit()=table_values[0]`、`get_fast_limit()=[2]`、`get_slow_limit()=[4]`，与本项目逐位一致；ZenStates-Core 的 version 字典只覆盖传感器，不含功耗 limit，不能用来否证（`2026-11-21.md` 第 41 行）。**这是被撤销的误判，迁移时不要重犯。**

### 2.3 哪些功能通过 Windows 电源计划实现（须与上述严格区分）

**【代码】** 这一路是**操作系统层面的电源策略管理**，从未被删除，与 CPU 硬件功耗控制是两条独立通路：

- 服务层：`Zentwik.Core/Services/PowerPolicyService.cs`、`IPowerPolicyApi.cs`、`PowerPolicyApi.cs`、`PowerSchemePreset.cs`、`PowerStatusWatcher.cs`
- 8 项设置 GUID：EPP、Boost Mode、Max / Min Processor State、Cooling Policy、Core Parking（CPMINCORES）、Idle Disable（C-State）、ASPM
- 系统 API：写 `PowerWriteACValueIndex` 仅改方案存储值，必须再调 `PowerSetActiveScheme` 激活才生效（源码注释明确记载此坑）
- IPC 命令 7 条，当前全部仍在 `WorkerPipeContract.cs`：`query-power-policy`、`apply-power-policy`、`query-power-schemes`、`set-active-power-scheme`、`create-power-scheme`、`rename-power-scheme`、`delete-power-scheme`
- 方案存放于 `plans.json`（三类方案独立），保存方案不改绑系统 GUID

**【推断】** 这一路之所以完整保留，是因为它走 Windows 公开 API、不依赖厂商固件、不涉及硬件写入被拒的问题；但也意味着它只能调整系统调度策略，**不能设定 PPT / PL1 / 功耗墙等硬件上限**。

---

## 3. SMU 路径及其限制

### 3.1 为什么考虑 SMU

**【文档】** 早期项目的核心能力就是 AMD Curve Optimizer 与 FMax，已经通过 SMU 邮箱写入逐核电压偏移与频率上限。功耗上限作为同类"处理器参数"自然沿同一通道实现：第一代借 ZenStates-Core 封装的 SMU 命令，第二代改用自研 SMU 层（`defab5a`）后直接按 family/model 查命令组。**【推断】** 这是能力复用驱动的选型，不是调研后认定的最优路径。

### 3.2 HX 平台上遇到的具体限制

**【测试】本机实测结论（记录于 `2026-11-21.md` 第 9、42 行）**：

- 机型：Fire Range（开发机 Ryzen 9 9955HX，SMU 版本 `0x04621A00`，FMax 4750 MHz）
- **读成功**：PM 表读到 STAPM 116 W / FAST 139.2 W / SLOW 116 W
- **写失败**：三项 SMU 写入全部返回 `0xFD`

**【代码 vs 文档的重要区分】**：

- 代码中能检索到的是驱动层 HRESULT `0x8007054F` 的描述映射（`SmuSession.cs:262`），即**固件拒绝**
- `0xFD` 这个具体返回值出现在工作日志与交接文档中，被解释为 `REP_MSG_CmdRejectedPrereq`（命令被识别，但前置条件不满足）
- **本次调查在工作区内未找到原始 `worker-*.log`**，无法从一手日志复现 `0xFD` 的逐条上下文。二者层级不同（固件返回码 vs 驱动 HRESULT），但指向同一类"固件拒绝"结果

**【推断】结论**：不是"命令不存在"（那会返回 `UNKNOWN_CMD` / `0x8007000D`），也不是"写入后无效"（因为写入即被拒、回读不可能变化），而是**固件认识命令但拒绝执行**，即前置条件不满足。

### 3.3 旁证：OC 通道同样被关闭

**【文档】2026-11-24 只读探测**（`%TEMP%\zw-ocprobe`，未入仓库）：

- `0x6F` 返回 `0xE6` → `IsOverclockable=0`、PBO=1
- `0x2C`（GetOverclockCap）返回 `0x80070057`（命令不存在）
- 结论：定频（`0x5A/0x5C/0x5D`）与分核频率上限都进不去

**【推断】** 当时判断 OC 通道被固件关闭"与功耗限制被拒同源"。这是**推断而非证实**——两条证据都指向固件层面的限制，但本次调查没有找到能证明二者共用同一开关的代码或寄存器证据。

### 3.4 限制范围：设备特有还是一类平台

**【测试】已验证**：仅限本机 Fire Range（9955HX）在当时的 BIOS / 固件版本下。

**【未验证】**：

- APU 命令组（`0x14/0x15/0x16`）只做了代码支持与单测，从未真机写入验证
- 其他 HX 机型、其他 BIOS / AGESA 版本未测试
- 因此"移动端功耗墙由 EC / 厂商固件管控"是**记录中的解释性结论**，不能表述为"所有 HX 平台 SMU 功耗写入都不可用"这类普遍事实


**【文档】** 交接文档明确写："PM 表读数本身与 RyzenAdj 一致，但 EC 在更外层限得更低，看到的数不代表实际上限"。其中"EC 在更外层限得更低"属于**【推断】**——PM 表读数与 RyzenAdj 一致是证据，EC 限值更低是解释。

---

## 4. EC 与其他替代路径调查

### 4.1 EC 控制

**【文档】当时确实调查过，但仅停留在路径识别层面**：
- 工作日志（`2026-11-21.md` 第 42 行）记录：应用层入口只有机械革命 GCU 服务的 MQTT（`127.0.0.1:13688`），来源为 `L-Mechrevo-Next` 逆向项目
- 该结论针对的机型是 Uniwill 模具，即本机所属的一类模具

**【代码】全历史检索结果**：对前 400 个提交的全部源码检索 `mechrevo` / `uniwill` / `mqtt` / `13688` / `gcu` / `ec_` 等关键词，**没有任何 Zentwik 源码实现**。仅在早期二进制依赖 `ZenStates-Core.dll` 中有字符串命中，无法判定其具体含义。架构审计文档亦明确："当前仓库中没有 ACPI / EC / BIOS / P-state 专属实现的代码证据"。

**【未验证】**：
- 未找到任何 EC 寄存器定义、读写尝试、内核接口或厂商工具调用的记录
- 未找到对 GCU MQTT 的实际读写测试、报文样例或连通性验证
- 未找到对写入风险的评估文档

**【推断】** 未形成实现的原因至少包括：寄存器定义依赖具体模具与固件版本、缺乏跨机型通用性、需要对接厂商私有服务、存在写入风险、维护成本高。这些是**基于证据不足的合理推断**，不是当时逐条记录的决策理由。

### 4.2 其他路径

| 路径 | 调查程度 | 结论 |
| --- | --- | --- |
| 厂商工具（RyzenAdj） | 【文档】用于**对照验证** PM 表读数索引 | 只作为参考实现，未集成为运行时依赖 |
| OC 定频 / 分核 FMax | 【文档】实测 `0x6F`→`0xE6`、`0x2C` 不存在 | 明确不可行，已放弃；记录注明"不再试探 `0x29`" |
| 自研 SMU 层直写 | 【测试】实测写入被拒 `0xFD` | 本机不可行 |
| PawnIO 多模块（Intel 侧） | 【未验证】AMD 机器上加载 `IntelMSR.bin` 返回错误 50 | 无 Intel 真机，无法验证 |
| 阈值自动化（按功耗 / 温度触发） | 【文档】新版盘点确认无任何性能采样 | 未实现，属于路线图空位 |

**重要限定**：**【推断】** "理论上可以通过 EC 操作"与"适合作为正式产品实现方案"是两件事。现有证据只支持前者曾被识别，不支持后者曾被验证或具备可行性。

---

## 5. 删除过程与决策依据

### 5.1 第一次删除（0.8.5，2026-10-30）

**【代码】提交 `575867e`**，提交信息原文：`Remove SMU power limit UI and writes; the platform has no usable channel for them`。删除 6 个功耗相关文件（含 `PowerLimitService.cs`、`PowerLimitModels.cs`、`PowerBaselineStore.cs`、`PowerBaselineBootstrap.cs`），净减约 500 行；CHANGELOG 0.8.5 记"移除功耗墙卡片"。

**【文档】具体依据缺失**：这次删除没有留下机型、测试数值或详细理由；"no usable channel"是原文中唯一的说明。它发生在一个包含后台服务重构、内存整理、崩溃修复的大提交中，**属于技术决策与重构合并进行的删除**，不能简单归类为"顺手删除"。

### 5.2 第二次（最终）删除（2.6.0-can7，2026-11-21）

**【文档】决策经过**（`2026-11-21.md` 第 38–46 行）：
1. 触发：用户问"Intel 功耗在笔记本上大概率也不可用，有必要撤包吗"
2. 追查：发现 AMD 功耗在本机写入被 EC 锁
3. 决定：AMD + Intel 功耗全部移除。用户判断 can 通道只有几个人用，不必留兼容

**【代码】提交 `348712a`**，31 个文件变更，+35 / −1313 行：
- 删除 4 个完整文件：`SmuPowerLimit.cs`、`PowerLimit.cs`、`PowerLimitPayload.cs`、`TuningViewModel.PowerLimits.cs`
- 清理 15+ 处引用：`SmuSession`（PM 表解析与写入）、`SmuChannel`（含 PM 表 ioctl）、`SmuReader` / `ISmuReader`、`IntelTuning`（MSR `0x610`/`0x614` 与时间窗编解码）、`IntelTuningPayload`、`TuningApplyService`、`PlanService` / `PlanPayloads`、`TuningStore`、`WorkerBridge`、`ProfileSession`、`TuningViewModel`、`TuningView.xaml` 段头、`PipeServerService` 两条命令与字段映射、相关测试
- 协议 21 → 22，删除 `query-power-limit` / `apply-power-limit`；`WorkerPipeContractTests` 断言改为 22
- 集成测试段头断言 6 → 4
- 验收：build 0/0、test 349（原 360，删 11 个功耗单测）、xamlcheck 五项 0、UiSmoke failures=0、selftest + itest 571 项全绿

**【代码】数据兼容**：旧 `plans.json` / `tuning.json` 中的 `PowerLimit*` 字段由 `System.Text.Json` 反序列化时忽略，**无迁移代码**。

### 5.3 决策性质判定

| 问题 | 判定 | 依据 |
| --- | --- | --- |
| 时间点 | 2026-11-21，tag `v2.6.0-can7` | commit `348712a` 与 tag 均已核实 |
| 为什么删除而非重构 | 【文档】写入通道在本机被固件拒绝，重构无法绕过；叠加 can 通道用户少、不留兼容的产品判断 | `2026-11-21.md` 第 40–43 行 |
| 删了什么 | 见 5.2 | Git diff |
| 永久 / 暂时 / 仅 HX | **【推断】整体永久移除，不是仅针对 HX，也不是单纯搁置**：APU 命令组一并删除，协议命令删除而非标记 deprecated，旧数据无迁移；唯一"暂缓"意味体现在"必要时可重新评估"的一般可能性，无文档表明有回归计划 | 同上 |
| Intel 是否同样验证失败 | **否**。【文档】Intel 写入通道从未真机验证，是因为 AMD 侧失败 + 用户判断笔记本大概率不可用而**一并移除**，不是实测失败 | `HANDOFF.md` 第 101 行 |
| 电源计划是否保留 | **是，完整保留** | 7 条 IPC 命令与 `PowerPolicyService` 全部在册 |
| 决策类型 | **主动技术决策 + 产品取舍**，不是资源不足导致的实现停滞，也不是纯重构顺手删除 | 触发对话、决策记录、删除范围三者一致 |

---

## 6. 功能边界与新版 Zentwik 迁移建议

三类功能必须分开处置。下表结论是**基于现有证据的迁移判断**，不是新方案设计。

### 6.1 Windows 电源计划管理（操作系统层）

- **是否迁移**：**应当迁移并保留**。
- **理由**：【代码】走 Windows 公开 API，不依赖厂商固件；【文档】旧版已长期维护且多次修复（方案改绑、标记分层）；它是唯一能落地的"电源策略"能力。
- **路线图定位**：作为既有能力继续维护，**不与硬件功耗控制混为一谈**。界面上须避免"改 SMU"与"改系统电源"共用一个应用按钮的语义冲突——这是旧版日志中已记录的设计教训。
- **重新考虑的前提**：不适用。

### 6.2 CPU 硬件功耗控制（PPT / PL1 / 功耗墙 / 温度限制）

- **是否迁移**：**明确排除**，不进入新版路线图。
- **理由**：【测试】本机 Fire Range 的 SMU 写入被固件拒绝；【文档】Intel 侧从未验证；【代码】全部相关实现已删除且协议命令移除。
- **为什么不是"暂搁"**：现有证据不支持"等待固件开放"的假设；投入产出比在 can 通道阶段已被判定不划算。把已排除功能重新挂回路线图会造成规划噪声。
- **未来重新考虑的前提**（须**全部**满足，缺一不可）：
  1. 有可复现的真机证据表明某类平台的硬件功耗写入被接受且能持久生效（写入后回读一致、跨休眠保持）；
  2. 明确目标平台范围，并承认这是**按机型 / BIOS 版本**适配的能力，而非通用能力；
  3. 有稳定的驱动与权限方案，且固件升级不会使实现失效；
  4. 用户价值经确认，足以覆盖跨机型维护成本。

### 6.3 厂商 EC 控制

- **是否迁移**：**排除在路线图之外**，连"待验证"都不建议列入近期计划。
- **理由**：【未验证】只有路径线索（GCU MQTT），没有寄存器定义、实现代码、读写测试或可行性评估；【推断】跨机型碎片化、私有协议依赖与写入风险使其不适合作为通用产品能力。
- **与"不可行"的区别**：应表述为"**未调查完成且不具备产品化条件**"，而非"技术上绝对不可行"。不得因为理论上可操作就把它写成可交付方案。
- **重新考虑的前提**：需要先有针对具体机型的完整寄存器 / 接口文档、只读探测验证、写入回滚机制与风险评估；在没有这些之前，讨论实现没有意义。

### 6.4 一个容易遗漏的关联边界

**【文档】** 定频与分核 FMax（OC 通道）与功耗控制是**同源受限**的相邻能力：9955HX 上 `0x6F`→`0xE6`、`0x2C` 不存在，定频不可行，已放弃。新版规划中若出现"频率上限个性化"类需求，应直接引用这条结论，**不要重新试探 `0x29` 或其他未验证命令**。

---

## 7. 尚未解决的问题与证据缺口

| # | 缺口 | 影响 |
| --- | --- | --- |
| 1 | 云端对话检索返回 0 条，无法核对当时讨论原文 | 决策动机只能依赖本地工作日志，无法交叉验证用户当时的完整表述 |
| 2 | 工作区内未找到原始 `worker-*.log` | `0xFD` 的逐条日志、PM 表读数上下文无法一手复现；"写入全部 0xFD"是记录中的结论 |
| 3 | 第一代删除（`575867e`）没有机型与测试记录 | "no usable channel"的具体所指不明，无法判定是同一台机器还是另一平台 |
| 4 | APU 命令组无真机写入验证 | 不能断言 APU 平台同样被固件拒绝；删除决策对用户价值判断的依赖大于技术验证 |
| 5 | EC / GCU MQTT 无实现与实测记录 | 无法评估可行性、风险与维护成本，只能界定为"未完成调查" |
| 6 | Intel MSR `0x610` 从未真机验证 | 不能说"Intel 功耗也不可用"，只能说"未验证即移除" |
| 7 | OC 关闭（`0xE6`）与功耗拒绝（`0xFD`）是否同源开关未证实 | 报告中已标注为推断，不应在后续文档中升格为事实 |
| 8 | ZenStates-Core 内部 SMU 命令号未反编译 | 第一代的具体命令号与平台适配表缺失；迁移已排除该功能，缺口不阻塞决策 |

---

## 8. 关键引用索引

**Git 提交**（均在 `D:\Projects\Zentwik Project\Zentwik`，完整 hash 可通过 `git show -s <短号>` 复核）
- `48b54fd81e6e49302cf27939f468c31137f3ebbc` — 0.8.2 新增四项功耗墙
- `d0872e4` — 0.8.3 功耗墙卡片移位
- `2999517` — 1.0.0-can4 引入电源方案
- `575867ed328af374e3d59cff2b7f46057785de6a` — 0.8.5 第一次删除功耗墙
- `defab5a8e617f6bbc82ff7e4bd965b3b4487474d` — 自研 SMU 层替换 ZenStates-Core
- `84bb998da78ebc6fa53e3697b3b6923fc225e6ae` — can2 Intel 功耗限制（tag `v2.6.0-can2`）
- `27dc1935dea6d65dad58f2d6fcbef5fee4c3aff4` — can3 AMD 功耗限制（tag `v2.6.0-can3`）
- `97df310` — can5 Intel 界面 SMU 术语修复
- `348712af0083c17f12c804ade4bd46e1a33ba3a1` — can7 最终删除（tag `v2.6.0-can7`）

**代码路径**（删除前版本用 `git show 348712a^:<path>` 读取）
- 第一代：`Services/PowerLimitService.cs`、`Services/PowerLimitModels.cs`、`Helpers/PowerBaselineStore.cs`、`Services/PowerBaselineBootstrap.cs`、`Services/SmuReader.cs`、`Services/SilentModeRunner.cs`（提交 `48b54fd`）
- 第二代 AMD：`Zentwik.Core/Smu/SmuPowerLimit.cs`、`Zentwik.Core/Smu/SmuSession.cs`、`Zentwik.Core/Smu/SmuChannel.cs`、`Zentwik.Core/Services/PowerLimit.cs`、`Zentwik.Core/Ipc/PowerLimitPayload.cs`、`Zentwik.UI/ViewModels/TuningViewModel.PowerLimits.cs`
- 第二代 Intel：`Zentwik.Core/Services/IntelTuning.cs`、`Zentwik.Core/Ipc/IntelTuningPayload.cs`、`Zentwik.Core/Smu/IntelMsrDevice.cs`
- 保留的电源计划：`Zentwik.Core/Services/PowerPolicyService.cs`、`IPowerPolicyApi.cs`、`PowerPolicyApi.cs`、`PowerSchemePreset.cs`、`Zentwik.Core/Ipc/WorkerPipeContract.cs`

**文档路径**
- `D:\Projects\Zentwik Project\.workbuddy\memory\2026-11-21.md`（核心决策记录，第 3–46 行）
- 同目录 `2026-11-20.md`（第 216–221 行）、`2026-11-24.md`（第 21–23 行）、`MEMORY.md`（第 33、41–42 行）
- `Zentwik/CHANGELOG.md`（0.8.2 / 0.8.3 / 0.8.5 / 2.6.0-can1～can7）
- `Zentwik-Docs/Zentwik-HANDOFF.md`（第 3b、3c 节）
- `Zentwik-Docs/Zentwik-第二轮技术审计.md`、工作区根目录的 `Zentwik-模块化架构审计-第一阶段.md`（无 EC / ACPI 实现的证据）

---

## 9. 结论摘要

1. 旧版功耗控制有**两代独立实现**：0.8.2 的 PPT/TDC/EDC/Tctl 四项功耗墙（ZenForge 单体 + ZenStates-Core），以及 2.6.0-can2/can3 的 Intel PL1/PL2/Tau + AMD STAPM/FAST/SLOW（Worker/IPC + 自研 SMU 层）。
2. **实测限制**只有一条硬证据：本机 Fire Range 上 PM 表读数正常，SMU 功耗写入被固件拒绝（`0xFD`，`REP_MSG_CmdRejectedPrereq`）。
3. **EC 只做了路径识别**（机械革命 GCU MQTT），没有任何实现、寄存器资料或读写测试；"EC 在外层限得更低"是解释性推断。
4. **最终删除**发生于 2026-11-21（commit `348712a`，tag `v2.6.0-can7`），是技术受阻叠加产品取舍的主动决策，覆盖 AMD 与 Intel 全部功耗限制，永久移除；Intel 侧是未验证即一并移除，不是实测失败。
5. **Windows 电源计划管理完整保留**，属于 OS 层能力，应迁移；CPU 硬件功耗控制与厂商 EC 控制应明确排除在新版路线图之外。
6. 报告中所有引用的提交号、tag、代码路径均已通过 Git 现场核对；无法核实的证据缺口已在第 7 章逐条列明。


---

## 迁移记录

- 迁入日期：2026-11-26
- 原始位置：`Motu-Z/2AI/LEGACY_POWER_CONTROL_INVESTIGATION.md`（私有仓库）
- 原始 Git Blob SHA：`473c5568a5ed358e20cb681cd70de6fc9b44b261`
- 本副本用途：作为 Zentwik Next Research 可独立访问的研究证据与历史调查记录。后续修订应在本研究仓库维护，并同步更新引用；不要依赖私有源文件才能理解本报告。
