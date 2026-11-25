# AMD SMU 访问链路

- 规范名称：AMD SMU 邮箱访问链路（PawnIO / RyzenSMU）
- 英文名称：System Management Unit mailbox access
- 缩写：SMU、MP1、RSMU
- 领域：`hardware/cpu`
- 核验状态：混合（见分项）
- 权威来源：RyzenSMU.p 上游模块源码；PawnIO 项目文档（签名模块、许可、反作弊）；本仓库 `research/02-hardware-access.md` §2/§11
- 相关文档：[amd-curve-optimizer.md](amd-curve-optimizer.md)、[security-service.md](../../windows/security-service.md)

## 1. 定义与范围

SMU 是 AMD 处理器内负责电源管理与调校的固件微控制器。应用经 PawnIO 驱动加载 `RyzenSMU.bin` 模块，通过 PCI 配置空间寄存器组（cmd/rsp/args）发送邮箱命令。本仓库聚焦 CO（`0x06`/`0xD5`）与 FMax（`0x6E`/`0x70`）。

## 2. 调用栈（Verified，本仓库源码事实）

```
SmuReader（门面/状态机）→ SmuSession（命令语义）→ SmuChannel（ioctl_send_smu_command）
                                                 → PawnIoDevice（CreateFile + DeviceIoControl）
```

- 设备路径 `\\?\GLOBALROOT\Device\PawnIO`；两个 IOCTL：加载模块、执行函数。
- 请求体 = 32 字节 ASCII 函数名 + N×8 字节参数；返回体按 8 字节对齐回填。
- 模块文件外置分发（`PawnIO/RyzenSMU.bin`），不内嵌单文件。

## 3. 命令号与世代绑定（Verified 实测 / Inferred 来源）

| 家族 | 型号 | SetMargin(CO写) | GetMargin(CO读) | GetBoostLimit | SetBoostLimit |
|---|---|---|---|---|---|
| 0x17 | Zen2 | 0x0A | 0x7C | 0x6E | 0x00（不支持） |
| 0x19 | Zen3 | 0x0A | 0x7C | 0x6E | 0x00 |
| 0x19 | Zen4 | 0x06 | 0xD5 | 0x6E | 0x70 |
| 0x1A | Zen5 | 0x06 | 0xD5 | 0x6E | 0x70 |

- 命令号来自逆向工具链口径，AMD 公开 PPR 不含 MP1 消息表（Inferred）。
- 家族判定是硬编码白名单，新 SKU（Hawk Point 0x75、部分 Dragon Range）可能误落到 Zen3 分支，发出错误世代命令（Inferred，高风险静默失败点）。
- 核心掩码：bit[31:28]=CCD、bit[27:24]=CCX、bit[23:20]=核内序号；写入低 16 位放偏移量补码。
- 实测：`0xD5` 在本机（Ryzen 9 9955HX / Fire Range 被识别为 Granite Ridge）走 RSMU 邮箱，box 寄存器组 cmd 0x3B10524 / rsp 0x3B10570 / args 0x3B10A40；bit23 对应的核被拒，其余 14 核可读。

## 4. 写入、验证与失败处理（Verified 本仓库）

- 就绪探针：发 `0x01`，参数 `[1,0,…]`，要求回 `2`；不通即 `ProbeFailed`，不进入写入。
- CO 写入：逐核 SetMargin → GetMargin 回读 → 不等即 `VerifyFailed`。
- FMax 写入：先读基线再写，回读容差 ≤10 MHz；基线为 0 视为该平台不实现该消息。
- 并发：旧版进程内 `static lock`，**跨进程不成立**；新项目句柄收归单一特权服务（ADR-003）。
- 超时：驱动内部轮询上限 8096 次；旧版只取 Win32 错误码而非 NTSTATUS，"区分拒绝与命令不存在"在实现上做不到（Inferred）。

## 5. 权限、签名与安装（Verified 文档 + 实测）

- 非提权进程打开 PawnIO 设备返回访问被拒绝；SYSTEM 服务可打开（实测）。
- 签名版驱动**只加载维护者签名的模块**；新项目不能自造 `.bin`。
- 许可：驱动 GPL-2.0（含模块例外），官方模块 LGPL-2.1；商业内置需授权。
- 反作弊：FACEIT 曾整体封禁 PawnIO 所用签名证书，属已知不可用场景。
- 安装器不安装 PawnIO，由用户自装（`winget install namazso.PawnIO`）。

## 6. 通道三态（Verified 设计原则）

`NotSupported`（平台不符）/ `DriverUnavailable`（环境缺失）/ `ProbeFailed`（硬件不应答）必须分开，不可压成 bool；单项不可用不得让产品整体失效。

## 7. 常见误解

- "SMU 命令号与固件版本无关" → 与固件无直接绑定，但与平台/世代强绑定。
- "能在两个进程同时打开设备" → 无跨进程仲裁，须单进程独占。
- "功耗限制移除=技术不可行" → 移除提交无理由，不能据此推断。

## 8. 待核验

Zen3/4/5 各一台的 CO/FMax 读写矩阵；S3/S4 恢复后是否丢 CO；PawnIO 多进程并发 ACL。

## 9. 核验状态分项

命令号与掩码编码、写入回读、权限实测 = Verified（本仓库）；协议来源、家族误分类 = Inferred；休眠清回出厂 = Unverified。
