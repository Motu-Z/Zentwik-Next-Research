# 传感器与硬件监控

- 领域：`hardware/sensors`
- 核验状态：采样率数字 = 报告称（原始日志缺失、来源 Unknown）；机制与设计原则 = Verified/Inferred；其余待研究
- 权威来源：本仓库 `research/02-hardware-access.md` §6、`research/03-ui-ipc-deployment.md` §2.1；Linux 内核 `arch/x86/include/asm/msr-index.h`（MSR 地址定义，2026-11-26 逐行核对）
- 证据规范：[measurement-evidence-schema.md](../../data/measurement-evidence-schema.md)

> **地址勘误（2026-11-26）**：原表把 `0x1B1` 笼统写作"DTS"，易被误读为逐核温度。正确区分为：`0x19C = IA32_THERM_STATUS`（**核心级/每逻辑核** DTS 数字读数，bits[22:16] 为该核距 TjMax 的差值）；`0x1B1 = IA32_PACKAGE_THERM_STATUS`（**封装级**温度状态，bits[22:16] 为封装距 TjMax 的差值）。二者不是同一传感器，读 `0x1B1` 不能得到逐核温度。MSR 地址由 Linux 内核 `msr-index.h` 逐行确认（Verified 源码事实）：`#define MSR_IA32_THERM_STATUS 0x0000019c`、`#define MSR_IA32_PACKAGE_THERM_STATUS 0x000001b1`。详见 §1.1。

> **第三轮审计说明**：下表 PDH 数字（约 4000/s、std 21–35pp、15.6ms）在 `research/02` §6 有采集方法与样本数叙述，但仓库内无原始日志可复核，按证据分类属 **报告称 / Unknown**，不得标"Verified 实测"。分档结论（≥100ms 窗口、双时钟）作为设计原则独立成立，不依赖这些具体数字为真。

## 1. 采样率与数据面分档（报告称 / Inferred 机制）

| 通道 | 有效新值速率 | 建议数据面 | 依据 |
|---|---|---|---|
| PDH 利用率/频率 | 可调用约 4000/s，但 1–4ms窗口 std 21–35pp | 4–10Hz（≥100ms 窗口） | 报告称（原始日志缺失） |
| Intel 倍频 0x198 | ≥30Hz 有意义 | 10Hz | 机制推断 |
| RAPL 功耗 0x611 | 短窗量化噪声大 | 2–4Hz（窗 ≥250ms） | 机制推断 |
| 温度（Intel DTS 0x19C 核心级 / 0x1B1 封装级） | ~1Hz | 1Hz | 机制推断；地址 Verified |
| AMD PM table | SMU 采样节拍约 1–10Hz | 2–4Hz | 机制推断 |
| EC / 风扇 | ≤1Hz / 每转一次 | 0.5–1Hz | 机制推断 |

### 1.1 Intel 温度 MSR 的层级区分（Verified 地址 / 机制推断采样率）

| MSR | 地址 | 官方名称 | 层级 | 数据位 |
|---|---|---|---|---|
| `IA32_THERM_STATUS` | `0x19C` | 数字热传感器状态 | **核心级**（每个逻辑核一份） | bits[22:16] = Digital Readout（距 TjMax 的摄氏度差值） |
| `IA32_PACKAGE_THERM_STATUS` | `0x1B1` | 封装热状态 | **封装级**（整个 CPU 封装一份） | bits[22:16] = Digital Readout（封装距 TjMax 的差值） |

- **不得**把 `0x1B1` 简称为"DTS"或当作逐核温度源；逐核温度须读 `0x19C`（每核各自 MSR）。两者可同时采集，分别用于核心热点与封装整体温度。
- 地址来源：Linux 内核 `arch/x86/include/asm/msr-index.h`（Verified 源码）。Digital Readout 位宽/语义为文档化机制（Inferred 表述精度，未逐字引 Intel SDM Vol.4 原文；Intel 官方 SDM 索引页不提供在线寄存器正文）。
- 相关：本仓库 `research/02` §3.1 把 `0x19C`/`0x1B1` 列为只读白名单；PawnIO `IntelMSR.bin` 允许读取这两个 MSR。

**核心结论（设计原则 Verified；具体数字报告称）**："能调用 4000 次/秒"与"有 4000 个独立信息/秒"是两回事；高频采样拿到的是短窗口量化噪声，必须按 ≥100ms 窗口聚合。未调定时器精度时 `Start-Sleep 1ms` 实际约 15.6ms（Windows 定时器粒度是文档化事实 Verified；15.6ms 这一具体观测值属报告称）。

## 2. 采样时钟（Verified）

- 采样用 QPC + 高分辨率可等待定时器；渲染用呈现时钟（`CompositionTarget.Rendering`）。
- 两个时钟分开；数据面 ≥4Hz（p99 ≤500ms），渲染面 30/60Hz 允许重复帧与插值。
- 过期数据丢弃而非补帧；不做外推。

## 3. 状态标注

温度/EC = Verified 机制；具体数值速率 = Inferred（待三组微基准升级为实测）。

## 4. 待研究

GPU 遥测（NVIDIA/AMD）、内存带宽、I/O、网络、SuperIO 风扇、ETW/PresentMon 帧时间。
