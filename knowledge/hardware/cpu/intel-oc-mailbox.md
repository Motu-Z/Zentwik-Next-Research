# Intel OC Mailbox（MSR 0x150）电压偏移

- 规范名称：OC Mailbox Voltage Offset / Intel 电压偏移邮箱
- 英文与官方名称：OC Mailbox；寄存器常称 `MSR_IA32_OC_MAILBOX`
- 缩写：OC Mailbox、UV（undervolt）
- 领域：`hardware/cpu`
- 核验状态：混合（见第 12 节分项标注）
- 权威来源（访问 2026-11-25）：
  - Linux 内核源码：`arch/x86/include/asm/msr-index.h` 定义 `MSR_IA32_OC_MAILBOX = 0x150`；`intel_turbo_max_3.c` 使用 `cmd<<32` + bit63 busy 位。[LKML 补丁讨论](https://lkml.iu.edu/hypermail/linux/kernel/2009.0/07178.html)
  - Linux 内核维护者 Borislav Petkov 在 LKML 明确表示：**OC Mailbox 命令没有公开架构文档**，超频非架构接口（Verified 该"事实缺失"本身）。
  - coreboot 上游 OC Mailbox 驱动（Haswell–Cometlake，逐平面偏移）。[Gerrit CL 7726869](https://chromium-review.googlesource.com/c/chromiumos/third_party/coreboot/+/7726869/2)
  - `intel-undervolt`（kitsunyan）源码 `undervolt.c`：11 位有符号定点，换算 `mV × 1.024`。[DeepWiki 分析](https://deepwiki.com/kitsunyan/intel-undervolt/5.2-msr-hardware-control)
  - `remittor/pyhwinfo`：`VoltageOffset` 解码为 S11.0.10（11 位、10 位小数、有符号），单位伏特。[文档](https://zread.ai/remittor/pyhwinfo/11-msr-mailbox-for-oc-and-voltage)
  - Intel 官方支持文档：Undervolt Protection（UVP）。[中文页](https://www.intel.com/content/www/cn/zh/support/articles/000094219.html)、[非 K 处理器降压](https://www.intel.cn/content/www/us/en/support/articles/000094707/processors.html)
  - 主板厂商文档：ASRock UEFI 手册明确指出"核心/环形电压偏移以 mV 为单位，使用 Mailbox MSR 0x150，cmd 0x10/0x11"。[ASRock H610M 手册](https://download.asrock.com/Manual/BIOS/H610M-HDVM.2%20R2.0_UEFI_CN.pdf)
  - 本仓库一手资料：`research/02-hardware-access.md` §3、§11
- 相关文档：[amd-curve-optimizer.md](amd-curve-optimizer.md)、[numeric-semantics.md](../../data/numeric-semantics.md)、[units.md](../../data/units.md)、[ENGINEERING_NOTES.md](../../ENGINEERING_NOTES.md) §3（互斥锁生态仅 pyhwinfo 一手确认）/§5（Meteor Lake+ TPMI 与 MSR 并存、单位布局不同，2026-11-26 核验）

## 1. 准确定义（外部技术事实）

Intel OC Mailbox 是 Intel 部分 CPU 上通过 MSR `0x150` 访问的一组**非架构性（non-architectural）**电压/频率控制命令接口。它采用"写命令字 + 轮询 busy 位 + 读响应"的邮箱协议，用于读取和设置各电压平面（IA Core / GT / Cache 等）的偏移与 VF 点。它是**请求型接口**，处理器可能因保护策略忽略请求。

与 AMD CO 的根本区别（重点）：**这里是真正的电压量，有明确的电压单位（1/1024 V 定点编码），而不是 AMD 那种无量纲步进档位。**

## 2. 适用范围

- 平台：Haswell/Broadwell 到 Cometlake（4–10 代）的资料最完整（coreboot 驱动）；本仓库代码覆盖的域位来自社区口径，需按世代/固件交叉验证（Inferred）。
- 第 12 代及以后：受 Undervolt Protection（UVP）约束，运行时降压在 UVP + VBS/HVCI 下不可用（Verified，Intel 官方文档）。
- 非 K 锁定处理器：Intel 不提供降压控制工具；BIOS 是否开放取决于 OEM（Verified，Intel 支持文档）。

### 2.1 电压域（Plane / Domain ID）映射按世代分表（**不得合并为通用表**）

> **跨代勘误（2026-11-26）**：Plane 3/4 的语义在 12 代前后发生漂移。旧代写作 uncore/analogio 的通用表若沿用到 Alder Lake+ 会写入保留域或错误域。以下按两代分列，未覆盖代际标 Unknown。

**Haswell – 10th Gen（`georgewhewell/undervolt` `PLANES` 字典，Verified 源码事实）**：

```python
PLANES = {'core': 0, 'gpu': 1, 'cache': 2, 'uncore': 3, 'analogio': 4,
          # 'digitalio': 5, # not working?}
```

| Plane | 0 | 1 | 2 | 3 | 4 | 5 |
|---|---|---|---|---|---|---|
| 语义 | Core | GT（GPU） | Cache（Ring） | **Uncore / System Agent** | **Analog I/O** | Digital I/O（源码注释 not working，未启用） |

**12th Gen Alder Lake 及以后（`remittor/pyhwinfo` `msrbox.py` `MAILBOX_OC_DOMAIN_ID_*`，Verified 源码事实）**：

```python
MAILBOX_OC_DOMAIN_ID_DDR                = 0x00
MAILBOX_OC_DOMAIN_ID_IA_CORE            = 0x00
MAILBOX_OC_DOMAIN_ID_GT                 = 0x01
MAILBOX_OC_DOMAIN_ID_RING               = 0x02
MAILBOX_OC_DOMAIN_ID_RESERVED           = 0x03  # UNCORE
MAILBOX_OC_DOMAIN_ID_SYSTEM_AGENT       = 0x04
MAILBOX_OC_DOMAIN_ID_L2_ATOM            = 0x05
MAILBOX_OC_DOMAIN_ID_MEMORY_CONTROLLER  = 0x06
```

| Plane | 0 | 1 | 2 | 3 | 4 | 5 | 6 |
|---|---|---|---|---|---|---|---|
| 语义 | IA Core（DDR 共用 0x00） | GT | Ring/Cache | **Reserved（源码注释仍写 UNCORE）** | **System Agent** | **L2 Atom（E-core L2）** | **Memory Controller** |

**证据等级与限制（必须显式说明）**：

- 两组映射均为**工具/逆向源码事实（Verified 源码存在）**，不是 Intel 官方公开规范。OC Mailbox 本身无公开架构文档（见 §1/§12）；pyhwinfo 源码注释明确其 Domain ID 来自"泄露的 BIOS 源码 `CpuMailboxLib.h`"（leaked BIOS sources），因此作为硬件协议仍是 **Inferred**，不得声称 Intel 官方定义。
- pyhwinfo 源码中 `check_mailbox_mutex` 的 VF 读取用 `domain_list = [IA_CORE, RING, SYSTEM_AGENT, RESERVED]`，其中标签 `UNCORE` 对应的 DomainId 是 `0x03`（即 RESERVED）——这印证旧称"uncore"在 12 代+ 已改为保留域、System Agent 迁到 0x04；但源码把 `0x03` 仍标注 UNCORE，说明**命名与语义在工具内部也不完全一致**，按编号而非按名称使用。
- **11th Gen（Rocket/Tiger Lake）未见一手映射**：不要推断它属于旧表还是新表，标 **Unknown / 待验证**。
- **对 Zentwik 的影响**：当前 `voltageOffsetmV` 只覆盖 Core/GT/Cache 三平面（0/1/2），两代表在这三项一致，故现有参数与电压域描述**不受该跨代差异影响**；但若未来扩展 SA/Uncore 或内存控制器偏移，**必须按 CPU 世代选择对应 Plane 表**，12 代+ 不得沿用旧代 Plane 3（=Reserved），且 L2 Atom（0x05）/Memory Controller（0x06）仅在 12 代+ 出现。在任何代际实际写入前须真机回读验证（Inferred→实测）。

## 3. 技术行为（邮箱协议）

1. 高 32 位为命令字：写 `0x80000011`、读 `0x80000010`，OR 上平面位（核显 `1<<8`、缓存 `1<<9`、核心 0）（Verified：本仓库代码 + coreboot cmd 0x10/0x11 口径）。
2. bit63 为 busy/run busy 位，写入命令时置 1，等待硬件清零后读取响应（Verified：Linux 内核 `intel_turbo_max_3.c`）。
3. 读路径是"先写读命令、再读同一 MSR 的低位"，不是纯读（Verified：本仓库实现）。
4. 共享互斥：核心与线程使用本地/全局互斥锁串行化邮箱事务（如 `Local\Access_Intel_OC_Mailbox`、`Global\Access_Intel_OC_Mailbox`），防止半写状态（Verified：pyhwinfo 维护者文档）。
5. 访问走 PawnIO 的 `IntelMSR.bin` 白名单，仅开放特定 MSR，非任意 MSR 可写（Verified：`research/02` §3.1）。

## 4. 数据类型与编码（重点）

- 寄存器：64 位 MSR。命令在**高 32 位**，数据在**低 32 位**。
- 电压偏移字段：低 32 位的 bit[31:21]，共 **11 位有符号定点数**。
- 编码格式：**S11.0.10** —— 11 位有符号、10 位小数、单位伏特（Inferred：pyhwinfo / intel-undervolt 等社区逆向口径；**Intel 无公开架构文档**）。
- 换算：物理电压值(V) = 定点整数 × 2^-10 = 整数 / 1024；以 mV 表达时 = 整数 × 1000/1024 ≈ 整数 × 0.9766，反向量化 = mV × 1.024（Inferred：intel-undervolt 逆向源码，本仓库实现沿用）。

> **第三轮审计**：`mihic/linux-intel-undervolt` 原文明确声明"**没有官方 MSR 文档、全部靠逆向 ThrottleStop 系统调用与试错、只在一台 i7-6700HQ 上测过**"。因此本表所有编码细节只能标 **Inferred（社区逆向口径，多源一致但非官方规范）**，不得标 Verified。"多源一致"不构成官方证据。

| 属性 | 取值 | 核验状态 |
|---|---|---|
| 字段位置 | 低 32 位 bit[31:21] | Inferred（社区逆向口径；本仓库实现沿用） |
| 宽度 / 符号 | 11 位 / 有符号（二进制补码） | Inferred（社区逆向，多源一致） |
| 定点格式 | S11.0.10（10 位小数） | Inferred（pyhwinfo 文档） |
| 单位 | 伏特（LSB = 1/1024 V ≈ 0.9766 mV） | Inferred（社区逆向，多源交叉） |
| 换算常数 | `code = mV × 1.024`；`mV = code / 1.024` | Inferred（intel-undervolt 逆向口径） |
| 符号语义 | 负值为降压 | Inferred（社区逆向） |

**必须显式说明**：`1.024` 这个系数是**电压单位换算**（mV↔1/1024 V 定点），与 AMD CO 的"步进档位"毫无关系。不得因为两者都涉及"降压偏移"就混为一谈。

## 5. 转换规则（Zentwik 约定 / Proposed）

1. 编码函数 `encodeVoltageOffsetmV(mv) -> u11`：先算 `mv × 1.024`，就近舍入到整数，再按 11 位补码符号扩展，放入 bit[31:21]。
2. 解码函数 `decodeVoltageOffset(raw) -> mV`：取 bit[31:21]，做 11 位符号扩展，再 `code / 1.024`。
3. 舍入规则必须固定（就近舍入），并有对称逆函数的单元测试。
4. 回读容差：本仓库用 2 mV（Verified：本仓库实现），比较在工程值层进行。
5. 钳制范围来自能力探测/BIOS，不硬编码；负偏移超过 BIOS/启动电压时会被忽略（UVP）。

## 6. 关联术语

MSR、IA32_OC_MAILBOX (0x150)、电压平面（IA Core / GT / Cache）、Undervolt Protection (UVP)、VBS/HVCI、Plundervolt、CFG Lock、Turbo Ratio Limit (0x1AD)、PL1/PL2 (0x610)、PawnIO `IntelMSR.bin`、Curve Optimizer（对照项）

## 7. 常见误解及错误用法

| 误解 | 纠正 | 依据 |
|---|---|---|
| "Intel 偏移和 AMD CO 是同一种降压值" | 完全不同：这里是电压量（1/1024 V），AMD 是无量纲档位 | Verified |
| "0x150 是普通可读寄存器" | 是邮箱，读也需先写命令字；且有 busy 位与互斥要求 | Verified |
| "写入成功且回读一致 = 电压真的降了" | 否。请求型接口，锁定/UVP 下回读仍可能是写入值；需负载下观测频率/功耗/温度 | Inferred，`research/02` §3.2/§8.3 |
| "公开 SDM 有完整命令表" | 没有。内核维护者明确指出非架构接口、无公开文档 | Verified（LKML） |
| "任意 MSR 都能写" | PawnIO `IntelMSR.bin` 是白名单；`0x1FC`/`0x199` 不可写 | Verified，`research/02` §3.1 |
| "所有 Intel CPU 都能降压" | 非 K 锁定、UVP+VBS、Plundervolt 微码锁均可使其不可用 | Verified（Intel 官方） |

## 8. Zentwik 内部约定（Proposed）

1. 电压偏移字段命名 `voltageOffsetmV` 或强类型 `VoltageOffsetMv`，类型与 AMD 的 `CurveOffsetStep` **类型系统隔离**。
2. `PlanApplyRecord.PersistsAcrossReboot` 对 Intel MSR 通道必须为 `false`（ADR-005）；Intel 调校是会话级能力，重启即回固件默认，除非新建"期望值 + 重放"链路。
3. 密码学意义上的安全约束：`0x150` 写入可能被微码静默忽略；产品必须把"通道可用性"和"效果验证"分开表达，不得伪装成功。
4. 每平面独立记账，某平面失败不中断其余平面（沿用本仓库实现）。

## 9. 权威资料来源与访问状态

| 来源 | 类型 | 访问状态 | 用途 |
|---|---|---|---|
| Linux `msr-index.h` / LKML 讨论 | 官方源码 | 已访问 | 寄存器号、邮箱协议、无公开文档这一事实 |
| coreboot OC Mailbox 驱动 | 维护者源码 | 已访问 | 逐平面偏移、支持世代 |
| georgewhewell/undervolt `PLANES` | 维护者源码 | 已访问 2026-11-26 | 旧代 Plane 0–4：core/gpu/cache/uncore/analogio |
| remittor/pyhwinfo `msrbox.py` `MAILBOX_OC_DOMAIN_ID_*` | 维护者源码 | 已访问 2026-11-26 | 12 代+ Plane 3=RESERVED、4=SYSTEM_AGENT、5=L2_ATOM、6=MEMORY_CONTROLLER；来源注释为泄露 BIOS `CpuMailboxLib.h` |
| intel-undervolt / pyhwinfo | 维护者源码 | 已访问 | 11 位定点、S11.0.10、1.024 换算 |
| Intel UVP 支持文档 | 官方 | 已访问 2026-11-25 | UVP/VBS 组合下运行时降压不可用 |
| ASRock UEFI 手册 | 厂商文档 | 已访问 2026-11-25 | cmd 0x10/0x11、mV 单位（BIOS 侧表述） |
| `research/02-hardware-access.md` §3/§11 | 本仓库一手资料 | 已核验 | 命令字、平面位、代码换算 |

## 10. 待核验问题

- 锁定状态下 `0x150` 回读到底是旧值/0 还是写入值（决定假阳性风险等级）。
- 12/13/14 代 K + Z 板，UVP/VBS 四种组合下的真实行为（未上机，本仓库当前为 AMD 平台）。
- `0x610` PL1/PL2/tau 曾被实现后移除，原因无记录，需先弄清再决定是否恢复。

## 11. 与 AMD CO 对照表（检索要点）

| 维度 | AMD CO | Intel OC Mailbox |
|---|---|---|
| 单位 | 档（无单位步进） | 电压，1/1024 V 定点 |
| 编码 | 补码整数放低 16 位 | 11 位有符号 S11.0.10 放 bit[31:21] |
| 换算常数 | 无可靠 mV 换算 | ×1.024（mV↔定点） |
| 通道 | SMU 邮箱（PawnIO RyzenSMU.bin） | MSR 邮箱（PawnIO IntelMSR.bin） |
| 官方命令文档 | PPR 不含 MP1 消息表 | 无公开架构文档 |
| 生效验证 | 逐核回读 + 效果观测 | 逐平面回读 + 效果观测 |
| 持久化 | 通常固件持久化 | 会话级（当前实现） |

## 12. 核验状态分项

| 结论 | 状态 |
|---|---|
| 寄存器号 0x150、邮箱协议、busy 位、命令字高 32 位 | Verified |
| 11 位有符号 S11.0.10、单位 1/1024 V、×1.024 换算 | **Inferred（社区逆向口径，多源一致但非官方规范）**——与 §4 第三轮审计结论一致；此前本表误标 Verified，此处更正 |
| Intel 无公开 OC Mailbox 命令文档 | Verified（官方维护者声明） |
| UVP + VBS/HVCI 下运行时降压不可用 | Verified（Intel 官方文档） |
| 本仓库代码位域编解码 | Verified（单测）；真机行为 Unverified |
| RAPL MSR 地址 0x606/0x610/0x611 | Verified（Linux kernel `msr-index.h` 本轮逐字核对）；单位公式见 [units.md](../../data/units.md) |
| OC Mailbox 0x150 不在 kernel msr-index.h | Verified（本轮核对，佐证非架构接口） |
| 命令字/域位跨世代通用性 | Inferred，须真实平台回读交叉验证 |
| Plane 0/1/2 = Core/GT/Cache（旧代与 12 代+ 一致） | Verified 源码事实（undervolt `PLANES` + pyhwinfo `MAILBOX_OC_DOMAIN_ID_*` 两源一致）；作为硬件协议 Inferred |
| Plane 3/4 跨代漂移（旧代 uncore/analogio；12 代+ Reserved/System Agent） | Verified 源码事实（上述两源分别逐行核对，2026-11-26）；作为硬件协议 Inferred（pyhwinfo 来源为泄露 BIOS 头） |
| 11th Gen 的 Plane 3/4 归属 | Unknown / 待验证（未找到一手映射，不推断归属） |
| 锁定状态回读行为 | Unverified |
