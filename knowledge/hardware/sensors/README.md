# 传感器与硬件监控

- 领域：`hardware/sensors`
- 核验状态：部分 Verified（采样率来自本仓库实测），其余待研究
- 权威来源：本仓库 `research/02-hardware-access.md` §6、`research/03-ui-ipc-deployment.md` §2.1

## 1. 采样率与数据面分档（Verified 实测 / Inferred 机制）

| 通道 | 有效新值速率 | 建议数据面 | 依据 |
|---|---|---|---|
| PDH 利用率/频率 | 可调用约 4000/s，但 1–4ms窗口 std 21–35pp | 4–10Hz（≥100ms 窗口） | 实测 |
| Intel 倍频 0x198 | ≥30Hz 有意义 | 10Hz | 机制推断 |
| RAPL 功耗 0x611 | 短窗量化噪声大 | 2–4Hz（窗 ≥250ms） | 机制推断 |
| 温度（DTS 0x1B1） | ~1Hz | 1Hz | 机制推断 |
| AMD PM table | SMU 采样节拍约 1–10Hz | 2–4Hz | 机制推断 |
| EC / 风扇 | ≤1Hz / 每转一次 | 0.5–1Hz | 机制推断 |

**核心结论（Verified 实测）**："能调用 4000 次/秒"与"有 4000 个独立信息/秒"是两回事；高频采样拿到的是短窗口量化噪声，必须按 ≥100ms 窗口聚合。未调定时器精度时 `Start-Sleep 1ms` 实际约 15.6ms。

## 2. 采样时钟（Verified）

- 采样用 QPC + 高分辨率可等待定时器；渲染用呈现时钟（`CompositionTarget.Rendering`）。
- 两个时钟分开；数据面 ≥4Hz（p99 ≤500ms），渲染面 30/60Hz 允许重复帧与插值。
- 过期数据丢弃而非补帧；不做外推。

## 3. 状态标注

温度/EC = Verified 机制；具体数值速率 = Inferred（待三组微基准升级为实测）。

## 4. 待研究

GPU 遥测（NVIDIA/AMD）、内存带宽、I/O、网络、SuperIO 风扇、ETW/PresentMon 帧时间。
