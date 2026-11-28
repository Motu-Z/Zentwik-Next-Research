# AMD 硬件功耗墙（PPT / TDC / EDC）

- 规范名称：AMD 功耗墙读写通道（RSMU DPTC 与 PM 表）
- 英文名称：AMD power limit (PPT / TDC / EDC) via RSMU DPTC and PM table
- 缩写：PPT、TDC、EDC、STAPM、PM table
- 领域：`hardware/cpu`
- 核验状态：混合（命令号 Verified 上游源码；本机读写行为 Verified 实测；折算原因 Inferred）
- 权威来源：`Reference Projects/ZenStates-Core-AOT-master/Hardware/Smu/Settings/Zen4Settings.cs`（逐行）；`Reference Projects/RyzenAdj-master/lib/api.c`；本仓库 `ENGINEERING_NOTES.md` §2（`ioctl_send_smu_command` 仅 RSMU）；2026-11-27 本机实测
- 相关文档：[amd-smu.md](amd-smu.md)、[implementation-references.md](implementation-references.md)、[measurement-evidence-schema.md](../../data/measurement-evidence-schema.md)

## 1. 定义与范围

功耗墙是 SMU 固件里的功耗/电流上限，四类参数：

| 参数 | 含义 | 单位 | 本通道可写 | 当前值可读 |
|---|---|---|---|---|
| STAPM | 持续功耗上限 | W（SMU 用 mW） | 否（MP1 邮箱） | 是（PM 表，仅展示） |
| PPT FAST | 短时/峰值功耗上限 | W（SMU 用 mW） | 是（Zen4/5 RSMU） | 是（PM 表） |
| PPT SLOW | 长时平均功耗上限 | W（SMU 用 mW） | 是（Zen4/5 RSMU） | 是（PM 表） |
| TDC | VDD 持续电流上限 | A（SMU 用 mA） | 是（Zen4/5 RSMU） | 否 |
| EDC | VDD 峰值电流上限 | A（SMU 用 mA） | 是（Zen4/5 RSMU） | 否 |

"可写"只表示命令可达；是否被固件接受、是否按请求值生效，必须由回读判定（见 §4）。

## 2. 命令通道：RSMU 与 MP1 是两条不同的路

**这是历史上最容易出错的一点。** 同一组参数在两套邮箱里各有命令号，不能混用：

| 邮箱 | FAST | SLOW | TDC | EDC | STAPM | 证据 |
|---|---|---|---|---|---|---|
| RSMU（Zen4/5） | `0x56` | `0xCB` | `0x57` | `0x58` | 未配置 | ZenStates `Zen4Settings.cs` 逐行，Verified 源码事实 |
| MP1（Zen4/5） | `0x3E` | `0x5F` | `0x3C` | `0x3D` | `0x4F` | 同上；RyzenAdj 对 DragonRange/FireRange 的 `set_fast_limit` 用 `0x3E`、`set_slow_limit` 用 `0x5F` |
| MP1（Zen3） | `0x3D` | — | `0x3B` | `0x3C` | — | ZenStates `Zen3Settings.cs` |

PawnIO 的 `ioctl_send_smu_command` **只能发 RSMU**（`ENGINEERING_NOTES.md` §2，Verified 上游源码）。因此：

- 把 MP1 命令号经该 IOCTL 发出，等于用错误的命令号打 RSMU，固件会返回"命令被识别但前置条件不满足"类响应（旧记录中的 `0xFD`）。**这不能作为"该平台不支持功耗写入"的证据。**
- Zen3 及更早、以及需要 MP1 的移动型号，本通道**不具备**正确的写命令，应报不支持，不应发 MP1 命令号。

## 3. PM 表读取（当前生效值）

PM 表由固件维护在 DRAM，通过两个模块调用读取：

| 调用 | 输出 | 说明 |
|---|---|---|
| `ioctl_resolve_pm_table` | 2 × qword：`[version, base]` | 地址只需解析一次；`base == 0` 视为失败 |
| `ioctl_read_pm_table` | N × qword，每 qword 两项 float | 传 3 项即可拿到前 6 个 float |

**必须先请求传输。** 未传输时 DRAM 里是残留内容，读出来是规律但无意义的位模式（本机实测：前六项 float 全为 0 或 1e-29 量级）。正确顺序：

1. `ioctl_send_smu_command(0x03)`（`SMU_MSG_TransferTableToDram`）；
2. 等待约 20 ms（固件在传输完成后才刷新 DRAM）；
3. `ioctl_read_pm_table`。

RyzenAdj 的 `refresh_table` 同向：前六项为 0 时再发一次传输并重读（源码注释：Raven/Picasso 首次传输返回 OK 但数据为空）。表中若有有效值时可不传输，直接读内存即可；写入后必须强制重传才能看到新值。

**前六项偏移跨表版本稳定**（RyzenAdj `get_*_limit` 与旧版 Zentwik 一致）：

| float 索引 | 字节偏移 | 含义 |
|---|---|---|
| `[0]` / `[1]` | `0x00` / `0x04` | STAPM limit / 当前值 |
| `[2]` / `[3]` | `0x08` / `0x0C` | FAST limit / 当前值 |
| `[4]` / `[5]` | `0x10` / `0x14` | SLOW limit / 当前值 |

**判定「表里有没有值」的下界必须是 1 W，不能是 0**（2026-11-28 实测踩到）。未传输时的残留位模式解出来是 `1e-23` 这类**极小的正数**，用 `raw > 0` 判断会通过校验、再被 `Math.Round` 舍成 `0`，于是「全 0 就强制传输」的分支永远进不去，冷启动后一直读残留、一直显示 0 W，而且 `HasAny` 为真会把 0 当有效值上报。正确判据：`raw is >= 1 and <= 1000`，未填充返回 `null` 而不是 `0`。

TDC/EDC 的当前值不在稳定段内（偏移随表版本变化），本通道不读。

## 4. 写入与回读语义

写入值单位：功率 mW、电流 mA（ZenStates 的 `SetSmuLimit` 内部 `arg * 1000` 即 W→mW，可作交叉印证）。

写入后必须回读并分四类，不得把"SMU 没报错"当成功：

| 状态 | 判定 | 含义 |
|---|---|---|
| `applied` | 回读与请求在容差内（±2 W） | 按请求生效 |
| `diverged` | 回读有变化但与请求不符 | 固件按自身策略折算，**实际按回读值生效** |
| `accepted` | 无回读通道（TDC/EDC） | 只知道 SMU 收下了请求 |
| `rejected` | 写入报错，或回读完全没变 | 未生效 |

**写前必须读到可用 PM 表基线。** 读不到就写，等于放弃回读能力，无法判断结果。

## 5. 本机实测（Ryzen 9 9955HX / Family 1Ah Model 44 / SMU `0x04621A00`）

2026-11-27，管理员会话，PawnIO `RyzenSMU.bin`（md5 `d4d752c9…`，与旧版同一份模块）：

| 项目 | 结果 | 证据级别 |
|---|---|---|
| PM 表版本 | `0x00621202`，base `0x81ED80000` | 实测 |
| 当前生效值 | FAST 100 W / SLOW 68 W / STAPM 56 W | 实测 |
| SMU 熔断值 | FAST 160 W / SLOW 102 W / VDD TDC 108 A（`0xDC`/`0xDA`/`0xDB`，原始单位 mW/mA） | 实测 |
| RSMU `0x56`/`0xCB` 写入 | **被接受**，PM 表随之改变 | 实测 |
| 写入折算 | 回读 = 请求 × 0.8，四组一致：95→76、65→52、125→100、85→68 | 实测 |
| 基线复原 | 写回 125/85 精确回到 100/68 | 实测 |
| TDC/EDC 写入 | 未验证（无回读通道，不做无依据的硬件改动） | 未验证 |
| `0x03` 传输偶发被拒 | 驱动返回 Win32 `1359`（`ERROR_ALREADY_EXISTS`，「已有在途」），重发即成功；一次失败就让整帧读数作废，表现为「打开就报错、刷新才对」 | 实测 2026-11-28 |

**`0x03` 要允许重试**（2026-11-28）。传输是幂等的（把固件里的表重抄一遍到 DRAM），重试无副作用；实测最多 3 次、退避 100/200 ms 足以恢复。同轮还确认两条前置条件各自对应的错误码，可用于分诊：

| 现象 | Win32 | 说明 |
|---|---|---|
| 发 `0x03` 时不持 `Global\Access_PCI` | `1359` | 与其他 SMU 使用者撞车，拿锁后再发即可 |
| 未先 `ioctl_resolve_pm_table` 就 `ioctl_read_pm_table` | `21`（`ERROR_NOT_READY`） | 地址未解析，先 resolve |

**0.8 折算的原因 = Inferred，不是事实。** 最可能的解释是 RSMU 参数表示封装总功耗，而 PM 表的 limit 表示 CPU 部分份额，移动 SoC 有 iGPU/SoC 分项；但没有固件文档或寄存器证据闭合。不同机型/BIOS 的系数可能不同，不得把 0.8 写成通用换算。

**这推翻了旧结论"本机功耗写入被固件拒绝 `0xFD`"的适用范围**：旧版第二代用 MP1 命令号（`0x4F/0x3E/0x5F`）走 RSMU 通道，失败更可能来自通道错配；改用 RSMU 命令号后写入被接受。旧记录的 `0xFD` 本身仍成立（当时确实被拒），但归因需要修正。此判断为 Inferred（未复现旧版调用序列逐条对比）。

## 6. 边界

- 仅 Zen4/Zen5 的 RSMU DPTC 可用；Zen3/2 与走 MP1 的移动型号（如 `0x19` 的 `0x40/0x44/0x50/0x74/0x75/0x78`）不提供。
- STAPM 在 MP1，本通道不可达，只读展示。
- TDC/EDC 无回读，只能记 `accepted`。
- PM 表读数不代表 EC 在外层没有更低的限制；"看到的 limit"不等于"负载下一定跑到"。
- 写入会改变硬件状态且重启前持续；产品侧应作为可选组件、默认关闭，并要求写前基线可读。

## 7. 核验状态分项

RSMU `0x56/0xCB/0x57/0x58` 与 MP1 `0x4F/0x3E/0x5F/0x3C/0x3D` = Verified 上游源码逐行；`ioctl_send_smu_command` 仅 RSMU = Verified 上游源码；PM 表两个 IOCTL 与 `0x03` 传输 = Verified 实测 + RyzenAdj 源码同向；前六项偏移 = Verified（RyzenAdj 与旧版一致）；本机读写与 0.8 折算 = Verified 实测（已复原）；折算原因 = Inferred；旧版 `0xFD` 归因 = Inferred；TDC/EDC 写入与是否存在读命令 = Unverified；「下界必须是 1 W」与「`0x03` 需重试（1359）」= Verified 实测（2026-11-28）。

## 8. PM 表能否当遥测用（2026-11-27 探查结论）

目的是给监控补温度/功耗。结论：**能做但暂不做**，理由是通道与口径都不满足"只写可核实事实"。

### 8.1 传输语义（实测）

PM 表不是持续刷新的可读缓存：**每次读前必须发 `0x03`（TransferTableToDram）并等约 20–30 ms**，否则读到的是上一次或全 0。

实测（`0x00621202`）：全程只传输一次时，空闲/全核负载/恢复三次读数**完全一致**（128 个 float 无一变化），说明缓存不会自行更新；改成每次读前传输后，同一批字段随负载变化，值可信。

代价：一次读取 = 一条 SMU 命令 + 约 30 ms 等待 + PCI 互斥。按 500 ms 采样等于每拍多这条开销，只能降到 2 秒级。

### 8.2 字段候选（本机 9955HX，`0x00621202`，未与厂商工具交叉验证）

| float 索引 | 空闲 | 全核负载（墙 100 W） | 全核负载（墙 40 W） | 推断 |
|---|---|---|---|---|
| 0 / 2 / 4 | 56 / 100 / 68 | 不变 | 不变 | STAPM / FAST / SLOW limit（已用于产品） |
| 3、26 | 32–39 | 81–82 | 36–39 | **推断作废**（原猜"CPU 温度"，见 §8.3 实测反驳） |
| 9 | 5–7 | 58 | 17 | 功率（与墙强相关，但非封装总量） |
| 20 | 4–5 | 53 | 11 | 功率（另一域，随墙同步） |
| 18 / 19 | 1.33 / 0.87 | 0.65 / 0.65 | 同 | 电压 |
| 16 | 1700–2400 | 270–350 | 同 | 未知（与负载反相关，不是频率） |

表长 `0x994` 与 ZenStates-Core `PowerTable` 中 `0x621202` 定义一致；该定义里 `offsetCoresPower` 为 -1（未映射），说明连它的维护者也没给这版表定核心功耗字段。

**索引 9 与 20 都是"功率"但都不是封装总量**：40 W 墙下两者合计约 28 W，剩余被 SoC/IO 吃掉。没有寄存器文档，无法判断哪个是 cores、哪个是 SoC，也就不能给用户一个叫"功耗"的数字。

### 8.3 明确的遥测来源：SMN + MSR（2026-11-27 实测通道通）

PM 表不适合做遥测，但有两条更直接、口径可核实的通道，均已在本机（9955HX，管理员会话）跑通。

**模块来源**：`LibreHardwareMonitor`（本地参考项目 `LibreHardwareMonitor-master`）已从 WinRing0 迁到 PawnIO，模块在 `LibreHardwareMonitorLib/Resources/PawnIo/AMDFamily17.bin`（10 656 B），导出 `ioctl_read_smn` 与 `ioctl_read_msr`。该模块属 `PawnIO.Modules`，**LGPL-2.1-or-later**，与本仓库已分发的 `RyzenSMU.bin` 同许可，`licenses/LGPL-2.1.txt` 已覆盖，无需新增许可面。加载方式与现有 `PawnIoSession` 相同（每个句柄加载一个模块，多模块 = 多开一个会话）。

**温度**：`ioctl_read_smn(0x00059800)`（`THM_TCON_CUR_TMP`，Linux `k10temp` 同款）

```
t = (raw >> 21) * 0.125                      // 0.125 °C/LSB
if ((raw & 0x80000) != 0 || (raw & 0x30000) == 0x30000) t -= 49
```

本机实测（`raw = 0x757B0000` → bit19 置位 → 68.4 °C）：空闲 70 °C，全核标量负载 45 s 从 70 → 75.5 °C 持续爬升，与封装功耗同步。无需绑核。

**封装功耗**：`ioctl_read_msr(0xC0010299)`（`MSR_PWR_UNIT`）取 `ESU[12:8]`，本机 = 16 → 每计数 `2^-ESU` **焦耳**（15.3 μJ）。`ioctl_read_msr(0xC001029B)`（`MSR_PKG_ENERGY_STAT`）累加器两次差分 ÷ 秒 = 瓦。

| 状态 | 封装功耗 | 温度 |
|---|---|---|
| 空闲 | 37 W | 70 °C |
| 全核标量负载 | 65–67 W | 70 → 75.5 °C（45 s 内持续上升） |

单位是**焦耳**不是微焦：按微焦算过一次，读数放大 100 万倍（331 kW 级），是本轮唯一踩到的坑。

**每核功耗 / 频率必须绑核**：`0xC001029A`（`CORE_ENERGY_STAT`）、`0xC00000E7/E8`（MPERF/APERF）都是核内 MSR。不绑核时每拍落在不同核，实测读数在 5 位数到 42 亿之间乱跳，差值毫无意义。LHM 的做法是 `ThreadAffinity.Set(core.Affinity)` 读一次再还原，16 核约 16 次绑核。

**PM 表温度推断作废**：§8.2 的 float 3 / 26 空闲 32–39 °C，而 SMN 同刻 68–70 °C，差 30 度，那两个字段不是当前温度。这也说明"PM 表里有像温度的字段"不足以证明它可信——没有交叉验证就不给用户看数字，这条判断在本轮得到印证。

### 8.4 决策（2026-11-27 更新）

- 监控扩**温度 + 封装功耗**：通道已验证、口径可核实（SMN 温度 / AMD RAPL 等价能量计数器），不需要再引第三方工具交叉验证。
- **不做**频率与每核功耗的第一版：需绑核 16 次且有跨拍状态（APERF/MPERF 差分），留待温度/功耗稳定后加。→ 逐核频率已于 2026-11-27 补上（§8.6），每核功耗仍未做。
- **仍然不用 PM 表做遥测**：传输开销（每次约 30 ms）与字段口径不明两项都没解决。
- 模块分发与安装器合并处理：`AMDFamily17.bin` 与 `RyzenSMU.bin` 一起打包，安装阶段一次落位。
- 已实测的传输语义写入本文，以后做这件事不再重测。

### 8.5 实现落点（2026-11-27 完成）

| 项 | 位置 |
|---|---|
| 换算 | `src/Zentwik.Platform/Windows/AmdSensorDecoder.cs`：温度 `(raw>>21)*0.125`，bit19 或 `[17:16]==0b11` 时 −49；能量差按无符号回绕；瓦 = 差值 × `2^-ESU` ÷ 秒 |
| 通道 | `src/Zentwik.Platform/Windows/AmdSensorSource.cs`：独立 PawnIO 句柄加载 `AMDFamily17.bin`，SMN 读持 `Global\Access_PCI` |
| 能力 | `monitor.telemetry.cpu.sensors`，归入 `monitoring` 模块（AnyOf：传感器缺席不影响系统遥测） |
| 数据显示 | `TelemetryDto.TemperatureCelsius` / `PackagePowerWatts` / `SensorDetail`；CLI `zentwik monitor` |
| 间隔 | 推送与传感器间隔各自可配（0 = 关闭传感器），边界见 `architecture/polling-budget.md` §9 |

真机读数（本机 9955HX，管理员会话，2026-11-27）：空闲 71.8 °C / 39.4 W；16 线程忙循环下 80.3 °C / 67.9 W，与 §8.3 的独立实测一致。

已知取舍：

- 报的是 **Tctl**（k10temp 口径）。1600X/1700X/1800X/2700X/Threadripper 19xx-29xx 的 Tdie 修正（−20/−27/−10）未实现——这些型号不在本项目目标机范围，真要支持需按 CPU 型号名查表。
- 封装功耗是**两次读取之间的平均值**，不是瞬时值；读取间隔越长，看到的越是平滑后的功耗。间隔拉到 30 秒时这个数字只能当趋势。
- 每核功耗未做（`0xC001029A`，需绑核 + 跨拍差分，见 §8.6）。

### 8.6 逐核频率（2026-11-27 完成）

**通道**：`0xC00000E7`（MPERF）/ `0xC00000E8`（APERF），核内 MSR，读前必须绑核。频率 = `aperfDelta ÷ 经过时间`，即**窗口内平均频率**（核心睡眠的时间计入）。整窗睡眠的核两个计数器都不动，不给值。

**先否掉了一条看起来更"好听"的路**：LHM 的 `Clock` 传感器 = `min(1, APERF/MPERF) × 当前 P-state 频率`（`0xC0010293` 的 CurCpuFid，Zen5 取 `fid[11:0] × 5 MHz`）。本机实测不成立：

- MPERF 的实际速率只有 52–345 MHz（空闲），不是 P0 频率——比值 `APERF/MPERF` 常在 **1.07–1.27**，被 `min(1,·)` 截断后结果恒等于 P-state 读数，等于没测。
- P-state 读数本身被自己污染：采样要唤醒该核才能读，读到的常是唤醒抬起来的最高档，而窗口内大部分时间它跑在低频。截断后 16 个核一起显示 4450 MHz，与低负载事实不符。

所以报**直接测得的平均频率**，不做 P-state 换算。代价是空闲核会显示个位数到几百 MHz（那不是坏掉，是它真的在睡）。

**成本（本机 9955HX，16 物理核 × 2 次 MSR 读 + 16 次绑核）**

| 场景 | 单帧耗时 |
|---|---|
| 空闲（与温度、封装功耗同帧） | 平均 0.23 ms，p50 0.18 ms |
| 全核满载、未提优先级 | 50–160 ms（几乎全是等时间片，不是 CPU 占用） |
| 全核满载、临时 `SetThreadPriority(Highest)` | 0.4–1.2 ms |

绑到忙核上必须等一个调度时间片，16 个核叠加就是上百毫秒；临时提优先级只占每核几微秒，读完立即还原，是本轮唯一必要的调度侧动作。

**标识**：按 `AmdCoreMap` 去重到物理核（SMT 兄弟只读第一个线程），标签 `(Ccd, Core)` 与 CO 的方案核编号同源，不做操作系统序号直译。

**真机读数**（正式服务，2026-11-27）：空闲 `clk 0.6–0.9 G`（单行只显示最高一核的窗口平均，其余核 3–40 MHz）；16 线程忙循环 `clk 3.7–4.0 G`、封装 68 W、温度 82–84 °C，逐核离散 1.4–3.8 G（负载只压了 16 线程，没压到的核仍然在睡）。

**顺带修掉的真 bug**：传感器刷新原先按"距上次读取是否已满间隔"判断，`PeriodicTimer` 每拍实际间隔抖到 994 ms 时会被判成未到点，于是每隔一拍才读一次——传感器间隔 1 秒实际是 2 秒。改为按拍数：`(tick - 1) % round(sensor/interval) == 0`，并加两条回归测试。

