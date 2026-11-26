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

**0.8 折算的原因 = Inferred，不是事实。** 最可能的解释是 RSMU 参数表示封装总功耗，而 PM 表的 limit 表示 CPU 部分份额，移动 SoC 有 iGPU/SoC 分项；但没有固件文档或寄存器证据闭合。不同机型/BIOS 的系数可能不同，不得把 0.8 写成通用换算。

**这推翻了旧结论"本机功耗写入被固件拒绝 `0xFD`"的适用范围**：旧版第二代用 MP1 命令号（`0x4F/0x3E/0x5F`）走 RSMU 通道，失败更可能来自通道错配；改用 RSMU 命令号后写入被接受。旧记录的 `0xFD` 本身仍成立（当时确实被拒），但归因需要修正。此判断为 Inferred（未复现旧版调用序列逐条对比）。

## 6. 边界

- 仅 Zen4/Zen5 的 RSMU DPTC 可用；Zen3/2 与走 MP1 的移动型号（如 `0x19` 的 `0x40/0x44/0x50/0x74/0x75/0x78`）不提供。
- STAPM 在 MP1，本通道不可达，只读展示。
- TDC/EDC 无回读，只能记 `accepted`。
- PM 表读数不代表 EC 在外层没有更低的限制；"看到的 limit"不等于"负载下一定跑到"。
- 写入会改变硬件状态且重启前持续；产品侧应作为可选组件、默认关闭，并要求写前基线可读。

## 7. 核验状态分项

RSMU `0x56/0xCB/0x57/0x58` 与 MP1 `0x4F/0x3E/0x5F/0x3C/0x3D` = Verified 上游源码逐行；`ioctl_send_smu_command` 仅 RSMU = Verified 上游源码；PM 表两个 IOCTL 与 `0x03` 传输 = Verified 实测 + RyzenAdj 源码同向；前六项偏移 = Verified（RyzenAdj 与旧版一致）；本机读写与 0.8 折算 = Verified 实测（已复原）；折算原因 = Inferred；旧版 `0xFD` 归因 = Inferred；TDC/EDC 写入与是否存在读命令 = Unverified。
