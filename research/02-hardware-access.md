# 02 · Windows 硬件访问链路

> 基线：`D:\Projects\Zentwik Project\Zentwik` @ `bc3c59b`（2026-11-25）。
> 标注约定：**【码】**= 旧项目源码事实，**【文档】**= 官方/上游资料，**【推断】**= 有依据但未实测，**【建议】**= 对新项目的主张，**【未验证】**= 必须上机确认。
> 外部资料访问日期：2026-11-25。本文只讨论"能不能访问、成本多大、边界在哪"，不决定界面与产品形态。

---

## 1. 结论摘要

1. 旧项目的全部硬件写操作只有三条链路：**PawnIO（SMU/MSR）→ 电源方案 API → CPU Sets/亲和性**。三者互不依赖，任一失效其余仍可用，这是旧版最值得继承的结构。
2. AMD 链路是**驱动内完成邮箱握手**的通路：应用只搬 32 字节函数名 + long 数组，超时与状态映射都在驱动里。写入后逐核回读、FMax 带容差回读，是整个项目里代码链路与校验逻辑最完整的一段（注：仓库内仍缺原始真机实测日志，详见 §11 及 `knowledge/AUDIT.md` 第三轮）。
3. Intel 链路**只做了"电压三平面 + 八档倍频"**；功耗限制（MSR 0x610）曾实现，已在 2.6.0-can7 连同 AMD 功耗限制一起移除，提交正文为空（后续已通过工作日志查明删除决策经过，见 [LEGACY_POWER_CONTROL_INVESTIGATION.md](LEGACY_POWER_CONTROL_INVESTIGATION.md)），不能当作"全平台技术不可行"的证据。
4. Intel 链路的真实边界不在代码，而在**平台策略**：Plundervolt 微码锁、Undervolt Protection、VBS/HVCI、非 K SKU 与 OEM BIOS，任一条都会让"写入成功且回读一致"变成假阳性。
5. 电源方案链路是**纯用户态 API**，但"写存储值→必须重新激活"这一步不可省，且旧版**没有任何回滚快照**。
6. CPU Sets 是**软约束**，官方明确"亲和性优先于 CPU Sets"；>64 逻辑处理器时只有 CPU Sets 能用，但跨组仍受线程亲和性所在组的限制。
7. **【建议】** 新项目按"用户态只读 / 用户态特权写 / 内核代理"三层划分，把驱动句柄收归单一特权进程；任何一层缺失都降级为"该能力不可用"，而不是整体失败。

---

## 2. AMD SMU 链路

### 2.1 调用栈（【码】）

```
SmuReader(门面/状态机)  →  SmuSession(命令语义)  →  SmuChannel(ioctl_send_smu_command)
                                                  →  PawnIoDevice(CreateFile + DeviceIoControl)
```

- 设备路径 `\\?\GLOBALROOT\Device\PawnIO`，`CreateFileW` 访问位 `GENERIC_READ|GENERIC_WRITE`，共享 `FILE_SHARE_READ|WRITE`，`OPEN_EXISTING`（`Zentwik.Core/Smu/PawnIoDevice.cs:29-36`）。
- 两个 IOCTL（`PawnIoDevice.cs:12-14`）：`CTL_CODE(0xA1B2, 0x821, …)` 加载模块、`CTL_CODE(0xA1B2, 0x841, …)` 执行函数。请求体 = 32 字节 ASCII 函数名 + N×8 字节参数；返回体按 8 字节对齐截断后回填 `long[]`（`:69-100`）。
- 模块文件随包分发到 `AppContext.BaseDirectory\PawnIO\RyzenSMU.bin`（`SmuReader.cs:88`，`Zentwik.Core.csproj:32-40`）。**模块不内嵌进单文件**，是外置文件。
- 使用到的模块函数：`ioctl_send_smu_command`（in 7 / out 6）、`ioctl_get_smu_version`（`SmuChannel.cs:8-9`）。

**【文档】** 上游 `RyzenSMU.p` 的 `main()` 只接受 x64 + AMD 厂商 + family `0x15/0x17/0x19/0x1A`（0x15 仅 Carrizo 系），并按 CodeName 选择 Mailbox 寄存器组；非 Carrizo 走 PCI 配置空间 `0xC4/0xC8`（bus 0/dev 0/fn 0）。**通用 `ioctl_send_smu_command` 不按消息号做白名单**，只校验命令落在 8 位范围；寄存器直读直写另有地址区间校验。

### 2.2 支持范围与命令号（【码】`SmuMessages.cs`）

| 家族判定 | 型号 | SetMargin | GetMargin | GetBoostLimit | SetBoostLimit |
|---|---|---|---|---|---|
| 0x17 | Zen2 | 0x0A | 0x7C | 0x6E | 0x00（旧版未配置写命令） |
| 0x19 | Zen3（除 0x61/0x74/0x78） | 0x0A | 0x7C | 0x6E | 0x00（旧版未配置写命令） |
| 0x19 | Zen4（0x61 Raphael / 0x74·0x78 Phoenix） | 0x06 | 0xD5 | 0x6E | 0x70 |
| 0x1A | Zen5 | 0x06 | 0xD5 | 0x6E | 0x70 |

- 核心掩码 bit[31:28]=CCD、bit[27:24]=CCX、bit[23:20]=核内序号；family>0x17 用 `core%8`，否则 `core%4` 且 CCX 取 `ccx%2`（`CoreMask.cs:9-15`）。写入时低 16 位放偏移量（负数补码，`CoreMask.cs:18`）。
- **【推断】** 这些命令号来自 SMU 逆向资料（RyzenAdj / SMU 工具链一致口径），AMD 公开 PPR 不提供 MP1 消息表；旧代码注释"数值为硬件协议常量，与固件版本无关"过于绝对。
- **【推断】** 家族判定是硬编码型号白名单：新 SKU（如 Hawk Point 0x19 0x75、部分 Dragon Range）会落到 Zen3 分支，从而发出**错误世代的 CO 命令**。这是"静默不生效"的高风险点，而不是崩溃点。
- **【未验证】** Zen5 是否真的接受 0x06/0xD5 与 0x70；旧版仅在少数机型上跑过（CHANGELOG 只记录功能上线，没有机型矩阵）。

### 2.3 写入、验证与失败处理（【码】`SmuReader.cs`）

- 就绪探针：先发 `0x01` 测试消息，参数 `[1,0,…]`，要求固件回 `2`（`SmuSession.cs:8,39-46`）。不通即 `ProbeFailed`，不进入任何写入。
- CO 写入：逐核 `SetMargin` → 立即 `GetMargin` 回读 → 值不等即记为 `VerifyFailed`（`SmuReader.cs:376-422`）。输入侧再钳一次 `[-50,+50]`（`:318-319`）。
- FMax 写入：**先读基线**再写，回读容差 ≤10 MHz（`:449-472`）。旧版 `SmuSession.GetFMax()` 在命令号为 0 **或** `TrySend` 失败时都会返回 0；`TuningStore.ResetToFactory()` 明确把基线 0 解释为“基线未知”，并跳过 FMax 写回。因此 0 不能证明平台不支持 FMax，也不能区分命令不支持与通信失败。旧版 `SmuMessages.For()` 对 Zen2/Zen3 将 `SetBoostLimit=0`，只说明该旧实现没有配置对应写命令；不能扩展成“部分移动平台硬件不实现 FMax”。
- 并发：进程内 `static lock` 串行化"遥测读 / 写入"，避免半写状态被读走（`SmuReader.cs:27,241,347`）。**跨进程不成立**：UI 与 Worker 同时持句柄时无人仲裁。
- 超时：由驱动内部完成（轮询响应寄存器，上限 8096 次，失败映射 `STATUS_DEVICE_BUSY` / `STATUS_IO_TIMEOUT`）。**【推断】** 旧版 `Execute` 只取 `Marshal.GetHRForLastWin32Error()`，拿到的是 Win32 错误码而非原始 NTSTATUS，因此 `SmuChannel.Send` 注释所称"区分拒绝与命令不存在"在实现上很可能**做不到**。
- 断电/休眠：`SleepResumeWatcher` 明确假设 S3/休眠恢复后 CO、FMax 被清回出厂（`SleepResumeWatcher.cs:12-18`）。**【未验证】** 该假设在 S3 上并非普遍成立（S3 通常不断电），把它当事实会导致"多写一次"或"漏写一次"。

### 2.4 权限、签名与安装

- **【码】** `Zentwik.Worker/app.manifest` 为 `requireAdministrator`；计划任务用 `InteractiveToken + HighestAvailable`，登录后 15 秒拉起，不弹 UAC（`AutoStartService.cs:29-37,390-405`）。
- **【实测·2026-11-25】** 本机 Ryzen 9 9955HX（16C/32T）、PawnIO 驱动运行中，**非提权**进程执行 `CreateFile("\\?\GLOBALROOT\Device\PawnIO", GENERIC_READ|WRITE)` 返回"访问被拒绝"。→ 旧版"需要管理员"的断言成立；SYSTEM 与其他会话的 ACL 行为仍未验证。
- **【码】** 安装器 `installer/zentwik.iss` **不安装 PawnIO**，只铺文件；README 要求用户自行安装。这是明确的用户摩擦点，也是旧版"没有驱动时其余功能仍可用"的设计前提。
- **【文档】** PawnIO 官方版由作者数字签名（实测 `pnputil /enum-drivers` 显示 `Microsoft Windows Hardware Compatibility Publisher`，即微软硬件兼容签名），安装需管理员；另有"无限制版"不校验模块签名、需测试模式。
- **【文档】** **签名版只加载由 PawnIO 维护者签名的模块**。这条决定架构：新项目**不能**自己写一个 `.bin` 塞进签名版；可选路径只有（a）使用官方已签模块，（b）申请维护者签名，（c）改用无签名版/测试模式（不可发布），（d）商业授权。
- **【文档】** 许可：驱动 GPL-2.0（含模块例外），官方模块 LGPL-2.1。旧版"外置 .bin + 不静态链接"的做法符合例外条款；**商业内置需另行授权**。
- **【文档】** 反作弊：FACEIT 曾整体封禁 PawnIO 所用签名证书（2.1.0 之前），官方明确不会为单个哈希开白。**这是"装了驱动也可能被游戏拦截"的实证**，必须在产品中作为已知不可用场景处理。

---

## 3. Intel 链路

### 3.1 访问方式（【码】+【文档】）

- 走 **PawnIO 的 `IntelMSR.bin`**，函数 `ioctl_read_msr` / `ioctl_write_msr`（`IntelMsrDevice.cs:8-9`），进程内 `lock` 串行。没有 WinRing0 依赖。
- **【文档】** `IntelMSR.p` 实现的是**白名单**，不是任意 MSR：
  - 可写：`0x150`(OC Mailbox)、`0x1AD`(TURBO_RATIO_LIMIT)、`0x650`(SECONDARY_TURBO_RATIO_LIMIT)、`0x620`(UNCORE_RATIO_LIMIT)、`0x610`(PKG_POWER_LIMIT)、`0x601`、`0x607/0x608`、`0x1A2`、`0x1A4`。
  - 只读：`0x606`(RAPL unit)、`0x614`(PKG_POWER_INFO)、`0x611/0x613/0x639/0x641` 能耗、`0x64F/0x6B1` 限频原因、`0x198/0x19C/0x1B1`、`0xE7/0xE8`、`0xCE`。
  - **不可写**：`0x1FC`(BD PROCHOT)、`0x199`(电压)、`0x1AA`(MISC_ENABLE/CFG 相关)。
  - 模块入口再校验 x64 + Intel 厂商。
- **【建议】** 这张白名单就是新项目 Intel 能力的**硬边界**，应直接引用而不是"理论上 MSR 都能写"。

### 3.2 OC Mailbox（0x150）编码（【码】`IntelTuning.cs`）

- 高 32 位 = 命令：`0x80000011` 写 / `0x80000010` 读，OR 上平面位（核显 `1<<8`、缓存 `1<<9`、核心 0）；低 32 位 bit[31:21] 为 11 位有符号偏移，单位 1/1024 V（代码用 `mV × 1.024` 换算，`:42-61`）。
- 读路径也是"先写命令、再读同一 MSR 的低位"（`:273-285`），不是纯读。
- 回读容差 2 mV（`:126`），失败逐平面记账，不中断其余平面（`:164-200`）。
- **【推断】** 命令字与平面位来自 XTU/ThrottleStop 社区口径，**不在公开 Intel SDM 中**；不同世代/固件的域位与符号扩展存在差异，必须以真实平台回读交叉验证。
- **【未验证】** "回读等于写入"不等于"电压真的降了"：PCU 可能接受请求但被下限/保护策略钳住。必须辅以负载下的频率/功耗/温度观测。

### 3.3 倍频（0x1AD）与功耗限制（0x610）

- `0x1AD` 按 8×8 bit 解八档倍频（`IntelTuning.cs:63-78`），写入后逐档比对回读（`:202-241`）。
- **【码·历史】** 功耗限制曾实现：`0x610` PL1[14:0] / PL2[46:32] / 时间窗[23:17]，单位取自 `0x614` bits[3:0]（2^-unit，缺省 1/8 W），写前读改写、写后回读（提交 `84bb998`）。在 `348712a`（2.6.0-can7）与 AMD 功耗限制一并删除，**提交正文为空**。
  - **【审计注记 2026-11-25】** 上述"时间窗单位取自 0x614 bits[3:0]、1/8 W"是把**功率单位**错套到**时间窗**上：功率/能量/时间三类单位都在 `MSR_RAPL_POWER_UNIT 0x606`（PU bits[3:0]、ESU bits[12:8]、TU bits[19:16]），而 `0x614` 是 `MSR_PKG_POWER_INFO`（TDP/功率范围），不含时间窗单位。正确的时间窗编码见 Intel SDM Vol.3B §14.9.3：`Time = 2^Y × (1 + Z/4) × Time_Unit`（Y=bits[21:17]，Z=bits[23:22]，Time_Unit 来自 0x606 TU，缺省 976 µs）。此注记仅更正技术事实，**保留原实现描述作为历史记录**；权威定义见 `knowledge/data/units.md`。
- **【推断】** 0x1AD / 0x610 都是"请求型"寄存器：锁定位（0x610 bit63）、BIOS 策略、非 K SKU 都可能让请求被忽略，而**回读仍返回你写入的值**。这是比"写入失败"更危险的假阳性。
- **【文档】** Intel 官方 Undervolt Protection（UVP，12 代起）：UVP 开启时，低于 BIOS/启动电压的负偏移"不被识别或应用"，处理器保持原电压；官方明确运行时软件降压在 UVP + 虚拟化（VBS/HVCI）组合下不可用。此前的 Plundervolt 修复（2019-12 微码）还可直接锁死 `0x150`，**写入被静默忽略**。
- **【推断】** BIOS 侧常见前置：`Overclocking Lock`（FIVR/OC 锁）必须关闭；社区普遍要求同时关 `CFG Lock`（MSR 0xE2 写保护），但二者与 0x150 的因果关系**缺少官方依据**，按"经验性前提"对待。
- **【文档】** `BD PROCHOT` 需要写 `0x1FC`，当前模块白名单不允许 → 旧版**从未具备**该能力，新项目若无自签模块也不必规划。

### 3.4 已验证 / 仅理论可行

| 能力 | 旧版状态 | 依据 |
|---|---|---|
| 读电压三平面 | 已实现，代码完整 | 【码】；真机覆盖面未知【未验证】 |
| 写电压偏移 + 回读 | 已实现 | 【码】；UVP/锁定时会失败，属预期 |
| 读/写八档倍频 | 已实现 | 【码】；效果未验证 |
| 读功耗墙/能耗 | 已实现 | 【码】 |
| 写 PL1/PL2/tau | **曾实现，已移除** | 【码】`348712a`；理由无记录 |
| BD PROCHOT / 电流墙 / 温度墙 | 未实现 | 【文档】模块白名单不允许 |
| 每核独立电压 | 未实现（仅平面级） | 【码】 |

---

## 4. 电源方案链路

### 4.1 API 与语义（【码】`PowerPolicyApi.cs`，【文档】MS Learn）

| 用途 | API | 备注 |
|---|---|---|
| 取当前方案 | `PowerGetActiveScheme` | 成功时缓冲区由系统 `LocalAlloc`，必须 `LocalFree` |
| 激活 | `PowerSetActiveScheme` | 改完值不调用它 → 不生效 |
| 读写 AC/DC 值 | `PowerRead/WriteACValueIndex`、`…DCValueIndex` | `RootPowerKey` 必须 NULL |
| 枚举 | `PowerEnumerate` | 16=方案 / 17=子组 / 18=设置项；耗尽返回 259 |
| 新建 | `PowerDuplicateScheme` | 无 `PowerCreatePlan`，只能派生 |
| 改名/说明 | `PowerWriteFriendlyName` / `PowerWriteDescription` | UTF-16、长度按字节含结尾符 |
| 删除 | `PowerDeleteScheme` | 活动方案需先切走 |
| 供电事实 | `GetSystemPowerStatus` | 与 powrprof 无关，放在同一接口方便取快照 |

- 使用的设置项 GUID 全部硬编码并于注册表逐项核对过（`PowerPolicyService.cs:25-62`）：EPP、BoostMode、Max/Min Processor State、CoolingPolicy、CPMINCORES、IdleDisable、ASPM。
- **【文档】** `PowerWriteACValueIndex` 只改存储值，"对活动方案的修改在调用 `PowerSetActiveScheme` 前不生效"。旧版正是这样做的（`RefreshActiveScheme`）。

### 4.2 权限与失败检测

- **【码】** 旧版 Worker 以管理员运行，写 HKLM 电源方案键；方案说明文字只能走 `PowerWriteDescription`，因为该键 ACL 只给 Administrators 读、全权在 SYSTEM（`PowerPolicyApi.cs:341-345`）。
- **【推断】** 官方文档未写"必须管理员"，但实现落在受 ACL 保护的 HKLM 子树，**非提权写入会失败（或被静默忽略）**；不要依赖"普通用户也能改"。
- **【码】** 部分失败检测是逐项做的：每项返回 `(Supported, Value, Error)`，`result != 0` 即 `Supported=false` 并把 Win32 码原样带出（`PowerPolicyService.cs:249-259,694-695`）。这比"整体成功/失败"更有用，值得保留。
- **【码】** 缺口：写入后**不回读**，只靠后续快照间接验证；`PowerPolicySnapshot` 把"读不到"和"值为 0"分开，这点正确。

### 4.3 回滚

- **【码】** 电源侧**没有回滚**：没有"改前快照 → 改后恢复"的路径；`DeleteScheme` 注释明确"删方案不回滚硬件"（`PlanService.cs:514-525`）。
- 调校侧的回滚不完整：FMax 有持久化基线 `FMaxBaselineMHz`（`TuningStore.cs:21-24`），CO 无持久化基线；Intel 基线只存在会话内存里（`IntelTuningChannel._baseline`），进程重启即丢。
- **【码】** 持久化与重放不对称：`TuningApplyService` 只依赖 `IAmdTuningChannel`，`PlanService` 的 Tuning 分支只写 CO/FMax。因此 **Intel 调校是会话级能力**：不保存方案、不进开机自启、不被休眠恢复重放，重启即回固件默认。若新项目承诺 Intel 持久化，需要新建"保存期望值 + 重启/唤醒后重放"的链路，不能照抄 AMD 的实现。
- **【码·风险】** 落盘先于写入：`PlanService.Apply` 先 `_tuning.Save(...)`（`TuningStore.Save` 内部即写 `AppliedAt`，`TuningStore.cs:61-63`）再调用 `ApplyFromStore`（`PlanService.cs:586-597`）。写入失败时磁盘上留下"已应用"时间戳，与硬件实际状态不一致；而契约注释又声称"Success 才回写 AppliedAt"（`WorkerPipeContract.cs:136`）——字段语义与调用顺序矛盾。新项目必须把"已提交 / 已确认 / 已回滚"拆成独立状态，写入成功后再推进。
- **【建议】** 新项目必须把"期望状态 / 观测状态 / 出厂或改前基线"三者分开存，基线记录**来源 + 采集时间 + 硬件身份**（对应架构约束 §4、§10）；禁止把"上次写入值"当出厂值。

---

## 5. 进程调度链路

### 5.1 CPU Sets 与 Affinity 的差异（【文档】+【码】）

| 维度 | CPU Sets | Affinity Mask |
|---|---|---|
| 约束强度 | **软约束**：调度器优先在集合内放置，但可越界（集合被独占/保留时尤其明显） | **硬约束**：线程只能在该掩码内运行 |
| 优先级 | 低于亲和性；"进程有更严格的亲和性时，亲和性优先于冲突的 CPU Set 赋值" | 高于 CPU Sets |
| 宽度 | ID 数组，天然无 64 位限制 | 64 位/组，只覆盖单个处理器组 |
| 多组 | "在多组系统上，若 CPU 赋值所在组与线程亲和性掩码的组不匹配，该赋值被忽略" | 需 `SetThreadGroupAffinity` 逐组处理 |
| 权限 | `PROCESS_SET_LIMITED_INFORMATION`（1709 前回退 `PROCESS_SET_INFORMATION`） | `PROCESS_SET_INFORMATION` |
| 清除 | 传空列表 + count 0 | 写回系统掩码 |
| 最低系统 | Win10 / Server 2016 | 更早 |

- **【文档】** `SetProcessDefaultCpuSets` 只影响"未显式设置线程集合"的线程，新建线程自动继承；文档称"传入合法参数时不会失败"，但保护进程/权限不足仍会失败，不能据此省掉错误处理。
- **【码】** 旧版两组 API 都实现了，并处理了最容易踩的坑：两者**叠加取交集**，切换方式前先清另一侧（`CpuSetScheduler.cs:586-588,694-704`；`ProcessWatcher.cs:760-799`）。
- **【码】** 拓扑来自 `GetSystemCpuSetInformation`，按 `LastLevelCacheIndex` 分组（AMD 每 CCD 一块 L3、Intel 单 ring 一组）；解析时 `Type` 必须是 2 字节、推进步长用回填 `Size`（`CpuSetScheduler.cs:86-136`）。旧版曾把 `Type` 当 4 字节读，导致拓扑恒空——这类结构体对齐错误是可复现的典型故障。
- **【码】** >64 逻辑处理器：亲和性直接拒绝（`IsAffinityMaskSupported = GetActiveProcessorGroupCount() <= 1`），不"先写下再报错"。

### 5.2 生命周期与 watcher（【码】`ProcessWatcher.cs`）

- 机制：Toolhelp32 快照差集，1 秒一轮（`NativeProcessList`），**不用 WMI**（旧版实测 WMI 常驻吃 8% 单核）。
- 生效范围：CPU Set 是**进程属性**，进程存活期间有效；进程重启即丢失 → 必须有 watcher 补绑。旧版还做启动补扫（`:485-514`）、每分钟全量重扫、PID 复用检测（创建时间，`:403-410`）、失败重试账本、漂移探测（每 5 轮 ≈5 秒，只记录不重写，`:445-470`）。
- 单实例：`Global\Zentwik.Singleton` 互斥体，避免两个实例互相把线程搬走（`:62-63,176-208`）。
- 解绑：只清"本工具成功绑过"的 PID，按当初方式各清各的（`:733-754`），不碰用户自己设的亲和性。
- **【码】** 软约束的副作用被显式提示：核心停泊 <100% 时 CPU Set 实际范围可能更大（`:929-937`）。

---

## 6. 采样率与抖动预算（供实时监控）

**【码】** 旧版**没有周期性遥测**：SMU 快照只在打开调校页/查询时按需读取（`CommandQuerySmuSnapshot`），`SmuReader.cs:345` 提到的"2 秒遥测定时器"是遗留注释，Worker 里不存在该定时器。因此下表是**机制推算（【推断】），不是实测**，必须由原型上机确认。

| 通道 | 机制 | 单次开销 | 有效新值速率 | 建议数据面 | 抖动主因 |
|---|---|---|---|---|---|
| Intel 倍频/有效频率（0x198、0xE7/0xE8） | IOCTL + `rdmsr` | 5–20 µs | 随 P-state 变化，≥30Hz 有意义 | 10–30Hz | 调度时钟 ~15.6ms、P-state 跳变 |
| Intel RAPL 功耗（0x611/0x639/0x641） | 累加器差分 | 5–20 µs | 短窗口量化噪声大 | 2–4Hz（窗口 ≥250ms） | 单位量化、累加器回绕 |
| Intel 温度（`0x19C` 核心级 / `0x1B1` 封装级） | DTS | 5–20 µs | ~1Hz 级 | 1Hz | 传感器内部刷新；两者层级不同，见 sensors/README §1.1 |
| AMD 逐核 CO（0x7C/0xD5） | 每核一次 mailbox 事务 | 0.1–2 ms/核 | 不适用（配置读取） | 仅按需 | SMU 忙、PCI 轮询上限 8096 次 |
| AMD PM table 温度/功耗 | transfer + 内存拷贝 | transfer 毫秒级、read 几十 µs | 由 SMU 采样节拍决定，约 1–10Hz | 2–4Hz（勿高频 transfer） | SMU 侧刷新周期 |
| EC / 笔记本 EC | ACPI EC 字节事务 | 1–5 ms/字节 | ≤1Hz | 0.5–1Hz | ACPI 全局锁、SMI 抢占 |
| SuperIO 风扇转速 | LPC IO 寄存器 | 10–100 µs | 每转一次（1000 RPM ≈ 60ms）+ 芯片平均 | 1Hz | 机械惯性、芯片滤波 |
| PDH 利用率/频率 | 性能计数器 | **实测** 单次 ~0.24ms（约 4200 次/秒） | 内核约 15.6ms 更新，短窗口噪声主导 | 4–10Hz（窗口 ≥100ms） | 定时器粒度、采样窗口过短 |
| GPU 帧时间 | ETW / PresentMon | 事件驱动 | 每帧 | 60Hz+ | 与传感器无关 |

- **【实测·2026-11-25】** Ryzen 9 9955HX / Win11，`% Processor Performance`（_Total，标称 2.5GHz）用忙等实现真实间隔采样：1ms 窗口 1187 个样本，均值 155%、标准差 21pp、区间 73–238%；4ms 窗口 301 个样本，均值 147%、标准差 35pp、区间 61–309%。`% Processor Time` 的值变化间隔均值 0.57ms（p95 1.5ms）。
  → **"能调用 4000 次/秒"与"有 4000 个独立信息/秒"是两件事**：高频采样拿到的是短窗口量化噪声，必须按 ≥100ms 窗口聚合。
- **【实测】** 不设定时器精度时，`Start-Sleep 1ms` 实际落在 ~15.6ms（64Hz 量级）——用 `DispatcherTimer` 做"60Hz 采样"既不均匀也不准时；渲染 60Hz 应基于 QPC 自校正的呈现时钟，而不是靠采样节拍。

- 推论：**只有"频率类"和"帧数据"能撑住 30Hz 以上**；温度/功耗/EC/风扇的真实新值速率都在 1–10Hz，60Hz 只是插值与动画。
- **【建议】** 验收标准不要写"60Hz 采样"，写"数据面 ≥4Hz 且同通道 p99 间隔 ≤500ms；渲染面 60Hz 允许重复帧与插值"。采样必须在单一采集器里做（同通道多读者会争 SMU 邮箱），时间戳用 QPC，过期数据丢弃而非补帧。
- **【建议】** 原型可先做微基准（不依赖 UI）：循环 1 万次 `ioctl_read_msr` 统计 QPC 间隔；对 AMD 逐核 `GetMargin` 计时整轮；对 PDH 以 1ms 轮询观察"值真正变化的间隔"。三组数据即可把上表从【推断】升级为实测。

---

## 7. 替代 / 补充访问方式对比

| 方案 | 能做什么 | 签名/安装成本 | 被拦截风险 | 维护成本 | 结论 |
|---|---|---|---|---|---|
| **PawnIO + 官方模块** | SMU 邮箱、白名单 MSR、部分 EC/SMBus | 驱动微软签名；模块须维护者签名；用户需单独安装 | 中：反作弊可能整证书封禁；未进 Defender 驱动黑名单（现状） | 低：模块随上游更新 | **【建议】首选**，但必须接受"模块能力 = 上游白名单" |
| **WinRing0 / 老式端口驱动** | 任意 MSR/IO/物理内存 | 驱动可自签但需测试模式或交叉签名 | **高**：CVE-2020-14979，Defender 报 `VulnerableDriver:WinNT/Winring0`，Win11 2022 Update 起默认启用易受攻击驱动黑名单 | 高：随时被拦 | **【建议】不采用** |
| **自研内核驱动** | 任意，含 0x1FC、私有 IOCTL、批量遥测 | 需 EV/WHQL 或 attestation 签名；每次更新重签 | 中高：HVCI/内存完整性、反作弊 | **高**：签名、兼容、崩溃责任 | 仅在 PawnIO 白名单硬性不足时评估 |
| **`NtSystemDebugControl`** | 理论可读写 MSR | 无签名需求 | — | — | **不可用**：未文档化，Win8.1 后仅调试场景放开，无稳定契约【推断】 |
| **厂商 SDK（AMD Ryzen Master / Intel XTU / PresentMon / PDH）** | 只读遥测较好；写入受厂商限制且不可再分发 | 无 | 低 | 中 | 只做**只读**补充，不做写入 |
| **纯用户态只读**（`GetLogicalProcessorInformationEx`、`GetSystemCpuSetInformation`、`PowerRead*`、PDH `% Processor Performance`） | 拓扑、电源方案、频率/利用率 | 无 | 无 | 低 | **【建议】默认层** |

- **【文档】** PawnIO 的"签名版只跑已签模块"既是安全设计，也是**能力上限**：需要新能力时，路径是向上游提模块，而不是自己加代码。
- **【推断】** 企业环境/开启内存完整性的机器上，任何内核驱动都可能被策略拒绝；产品必须有"无驱动"完整路径。

---

## 8. 新项目的访问分层建议

### 8.1 三层职责

| 层 | 归属进程 | 能力 | 权限 | 失败后的行为 |
|---|---|---|---|---|
| **L0 用户态只读** | UI 或普通服务 | 拓扑、CPU Set 枚举、电源方案读取、供电状态、PDH 频率/利用率、ETW | 无 | 显示"未知"，不阻塞 |
| **L1 用户态特权写** | 单一特权服务 | 电源方案读写+激活、CPU Sets/亲和性、优先级、计划任务、watcher | 管理员（计划任务免 UAC） | 逐项报告，可重试，不回滚未拥有的状态 |
| **L2 内核代理** | 同一特权服务内（独占句柄） | SMU CO/FMax、白名单 MSR | 管理员 + 已装 PawnIO | 通道标记 `DriverUnavailable/ProbeFailed/NotSupported`，L0/L1 照常 |

- **【建议】** SMU/MSR 句柄**只能由一个进程持有**。旧版把锁做成进程内 `static`，UI 与 Worker 同时打开时无人仲裁邮箱；新项目应把"硬件写"收归单一特权进程，UI 只走 IPC。
- **【建议】** 驱动安装**不静默执行**：检测 → 说明 → 引导用户运行官方安装包（或 `winget install namazso.PawnIO`）→ 重启后复检。旧版由用户自装、安装器不管，摩擦大但责任边界清晰。
- **【建议】** 启动护栏（先落 `Pending` 标记、活着出来才清除，连续 3 次阻断自动写入）是**必须保留**的机制（`BootGuard.cs:38-120`），它解决的是"进程没机会记录失败"的场景。

### 8.2 降级矩阵（缺什么 → 少什么）

| 环境 | AMD 调校 | Intel 调校 | 电源方案 | 进程调度 | 监控 |
|---|---|---|---|---|---|
| 未装 PawnIO | 不可用（读写都无） | 不可用 | 可用 | 可用 | 仅 L0 |
| 已装、非管理员 | 不可用 | 不可用 | 不可用（写） | 仅自身进程 | 仅 L0 |
| AMD 未知家族 | 只读快照不可信 → 标 `NotSupported` | — | 可用 | 可用（L3 分组） | 部分 |
| Intel + UVP/VBS 锁 | — | 读可用、写失败（回读仍可能"一致"） | 可用 | 可用 | 可用 |
| 反作弊拦截驱动 | 不可用 | 不可用 | 可用 | 可用 | 仅 L0 |

### 8.3 成立前提（不满足就不要承诺该能力）

1. **签名链可持续**：能接受"只能使用上游已签模块"；否则需要自研驱动预算与长期签名维护。
2. **验证必须到"效果"层**：寄存器回读一致只是必要条件。至少要有"写入 → 短时空载/负载观测（频率、功耗、温度、限频原因 MSR 0x64F）→ 判定"的路径，否则 UVP/锁定位下的假阳性无法识别。
3. **基线可信**：记录来源（SMU 读出 / 用户输入 / 推断）、时间、CPU 身份（family/model/stepping/微码）；"读不到就回 5400"这类兜底必须显式标注为推测值（旧版已改为"未知则跳过"，见 `TuningStore.cs:78-97`）。
4. **环境探测先于写入**：至少能判定 PawnIO 是否存在、模块是否能加载、CPU 厂商与家族、是否 VBS/HVCI 开启（`Win32_DeviceGuard`/注册表）、BIOS 锁定状态（写一次探测值并回读）。**不能因为 API 存在就宣称支持。**
5. **不可用是一等状态**：`NotSupported`（平台不符）、`DriverUnavailable`（环境缺失）、`ProbeFailed`（硬件不应答）三者必须在协议与界面上分开——旧版这个三态设计是正确的，应原样保留。

---

## 9. 待上机验证清单（按优先级）

1. AMD：Zen3/Zen4/Zen5 各一台，验证 `0x0A/0x7C` 与 `0x06/0xD5` 的 CO 读写、`0x6E/0x70` 的 FMax 读写；确认家族白名单对 Hawk Point / Dragon Range 的误分类。
2. AMD：S3 与 S4 恢复后 CO/FMax 是否真的丢失（旧版假设需证实或推翻）。
3. Intel：12/13/14 代 K + Z 板（UVP 可关）与 B 板非 K 各一台，验证 `0x150` 三平面读写、`0x1AD` 八档；对照 UVP 开关与 VBS 开关四种组合。
4. Intel：确认锁定状态下 `0x150` 回读是"旧值/0"还是"写入值"——决定假阳性风险等级。
5. Intel：`0x610` PL 写入是否被锁定位/BIOS 忽略（旧版移除原因不明，需先弄清再决定是否恢复）。
6. 电源：非提权调用 `PowerWriteACValueIndex` 的真实返回码；以及"写值后不激活"在多长时间窗内可见。
7. 调度：>64 逻辑处理器机器上，`SetProcessDefaultCpuSetMasks`（跨组掩码）能否替代 ID 数组路径；CPU Set 越界行为在停泊开启时的实际表现。
8. 驱动：PawnIO 设备句柄是否允许多进程并发、是否有 ACL 限制（管理员 vs SYSTEM）；升级/卸载后旧句柄行为。

---

## 10. 与旧项目的事实分歧（供决策记录引用）

- 旧 README 称"调校只支持 AMD Ryzen"，但代码已有完整 Intel 通道并被 UI 接入（CHANGELOG 2.6.0-can1）；文档与实现不一致。
- 旧代码称 SMU 命令号"与固件版本无关"，但 CO 命令按家族分叉。旧版 `GetFMax()` 将“读命令号未配置”和“邮箱发送失败”都折叠为返回 0；`TuningStore` 将基线 0 解释为未知并跳过恢复写回。旧版 Zen2/Zen3 命令表的 `SetBoostLimit=0` 只代表该实现未配置对应写命令，不构成“部分移动平台硬件不实现 FMax”的证据。**命令号与固件无直接绑定，但与平台/世代强绑定**。
- 旧代码称能通过 HRESULT 区分"SMU 拒绝"与"命令不存在"，实现只保留 Win32 错误码，能力被削弱。
- 功耗限制"曾实现后移除"在提交正文未留理由（工作日志调查见 [LEGACY_POWER_CONTROL_INVESTIGATION.md](LEGACY_POWER_CONTROL_INVESTIGATION.md)），不能据此推断"全平台技术上不可行"。

---

## 11. 实测补充（2026-11-25，Ryzen 9 9955HX / 服务以 SYSTEM 运行）

- **SYSTEM 令牌可以打开 PawnIO 设备**：`ZentwikAgent` 服务加载 `RyzenSMU.bin` 成功，SMU 测试消息 `0x01` 回 `2`、`ioctl_get_smu_version` 正常返回。此前"管理员 vs SYSTEM 的 ACL 行为未验证"一项，对 SYSTEM 成立。
- **Fire Range 被模块当作 Granite Ridge**：`RyzenSMU.p` 的 `get_code_name` 把 `(1Ah<<8)|44h` 映射为 `CPU_GraniteRidge`，因此邮箱寄存器组是 `cmd 0x3B10524 / rsp 0x3B10570 / args 0x3B10A40`（与桌面 Zen4/5 同组）。这说明读 CO 的 `0xD5` 走的是 RSMU 邮箱，与工具链口径一致。
- **`0xD5` 被 SMU 拒绝**：`tuning` 以掩码 `0x00000000`（核 0，序号编码）读 CO，模块返回 `STATUS_UNSUCCESSFUL` → Win32 31。按 `smu_status_to_nt` 反推，SMU 应答是 `0xFF`（`SMU_Failed`），不是 `0xFE`（未知命令，会得到 87）。即邮箱通、命令被拒。
- **待判定（已收敛）**：拒绝原因是"核地址编码不对（0x00100000 独热位才是核 0）"还是"移动平台固件锁定 CO"。已改为探测阶段自动协商两种编码；两种都失败才判 `Rejected`，并把每次尝试的掩码与错误码写进明细。
- **探测通过、整轮读取失败**：换上掩码协商版本后，探测用独热位 `0x00100000` 读核 0 被 SMU 接受，能力状态因此是 `degraded`（内部 `Available`）；而 `TryReadOffsets` 逐核扫 16 个物理核时某个核返回 `0xFF`，整轮读取失败，`tuning` 把"探测可用但读取失败"映射成 `rejected`。两条命令并不矛盾：前者只做核 0 探测，后者要求全核读完。
- 结论修正：**不是移动平台整体锁死 CO**，核 0 的 `0xD5` 可读；失败点落在部分核。原先首错会覆盖逐核信息，已改为继续读其余核并汇总每个失败核的 `核号(掩码)`，下次运行即可判定是 CCD 位编码（核 ≥8 的 `(ccd<<28)|(1<<(20+local))`）不被接受，还是停泊核不应答。
- **逐核结果（掩码协商版实测）**【历史记录：其中"bit23 在该平台不被接受"的**平台归因已于第四轮撤回**，见下方注记；下列掩码为错误独热位编码产物】：16 个物理核里只有核 3 与核 11 被拒，掩码分别是 `0x00800000` 与 `0x10800000`——两者是同一个位：`bit23`（CCD 内第 4 个核）。其余 14 个核（`bit20`、`bit21`、`bit22`、`bit24`…`bit27`，两个 CCD 都是）全部接受。位 23 专属被拒说明这不是"整平台锁定"，也不是"CCD 位放错"（若 CCD 位错，核 8–15 整段都会失败），而是 bit23 在该平台上不被邮箱接受；真实布局仍需一次掩码扫描（单 bit 20–27 与组合）才能定论，暂不改编码。
  - **【第四轮审计注记 2026-11-25：编码错误，非平台限制】** 上述"掩码协商版"使用的核地址编码 `(ccd<<28)|(1<<(20+local))` 是**独热位**（把 core index 当作位序号），与外部权威实现不符：
    1. `irusanov/ZenStates-Core`（dev 分支 `Cpu.cs`）`MakeCoreMask`：桌面 Zen2+ 为 `(ccd << 28) | ((core % 8) << 20)`，源码注释明确 `// [23-20] core index`——即 `[23:20]` 是**4 位数值字段**（0–7），不是独热位。
    2. `irusanov/SMUDebugTool`（`SettingsForm.cs:2562-2584`）`EncodeCoreMarginBitmask` 的兜底路径为 `int mask = (ccd<<8) | localCoreIndex; return (uint)(mask << 20);`，等价于 `(ccd<<28)|(local<<20)`，同为数值编码；其首选路径 `MakeCoreMaskForLogicalCore` 还用 APOB 核心映射跳过被禁用的 CCD/核。
    3. 本仓库 `amd-smu.md` §3 文档描述的 `bit[23:20]=核内序号` 也与该数值编码一致——因此是**实现与文档不一致**，不是文档描述错误。
    - 按独热位编码，核 3 被编码为 `1<<23 = 0x00800000`，而正确值应为 `3<<20 = 0x00300000`；核 11 应为 `0x10300000` 而非 `0x10800000`。**用错误编码得到的 SMU 拒绝，不能归因为"该平台不接受 bit23"。**
    - 用户报告（**User-provided**，未提供工具名/版本/原始输出）：其他工具可修改全部核心，与本判定一致。
    - **处理**：撤回"bit23 在该平台上不被邮箱接受"的平台归因，降级为"早期实现使用错误编码时的观测，原始日志缺失、来源 Unknown；编码与外部权威实现不符已由源码逐字核对确认"。"其余 14 个核全部接受"同样不能作为平台事实——独热位编码下它们可能读写到错误的核心（Inferred，需实机验证）。
    - **验证方法**：改用 `(ccd<<28)|(local<<20)` 并优先用 APOB 映射，在有硬件的环境逐核读/写/回读、保存原始日志，确认 16 核全部可读并观察效果层（频率/功耗/温度）。本仓库无硬件，不能自行验证，保持 Unverified。
- 实现影响：读取改为"部分成功也算可读"，失败核通过 `error` 报出、不阻塞其余核；方案里引用了读不到的核会记为 `Skipped` 且不写入（没有前值就不写）。
- **写入通道打通（2026-11-25）**：核 0 读 `-22` → 写 `-10` → 回读 `-10`，`tuning` 复核其余 14 个核数值不变（排除掩码别名）；`ledger` 记 `prev=-22 desired=-10 actual=-10`。写入参数是 `(mask & 0xFFF00000) | 值`，与旧项目口径一致。核 3/11 仍不可读，这两个核的写入未验证。
  - **【第三轮审计注记 2026-11-25，来源 Unknown】** 上述写入/回读叙述当年未保留原始日志（本仓库无任何 `ledger`、`result-*.txt` 或原始输出文件），读取时间、工具版本、接口参数与测试前设备状态均未记录。按 [measurement-evidence-schema.md](../knowledge/data/measurement-evidence-schema.md) §3/§4，归入 **Unknown（原始记录缺失）**，不得标为"本仓库实测事实"。回读一致只证明写入值被读回，不证明效果层生效（§6）。
- **语义纠正**：CO 值是有符号整数步进（负值降压），不是毫伏。早期实现把字段命名为 `millivolts`、CLI 打印 `offset(mV)` 属于臆断；已改为无单位的 `offset`，CLI 打印 `core  co`。真正的 mV 只存在于 Intel `0x150`（单位 1/1024 V）。
- **【第三轮审计·撤回原"出厂值"归因】** 原表述（历史记录，保留）："出厂值：本机 14 个可读核出厂即为非零（-11…-26），说明 OEM 预设了每核偏移；把某核写成 0 不是'恢复默认'，只是取消该核的出厂降压。"
  - **判定：无来源、状态混淆，撤回。** 依据：
    1. **无可查原始来源**：全仓库仅 Markdown，无任何原始日志/账本/实验记录；`-11…-26` 这组数值没有可追溯的原始输出，读取时间、工具、版本、接口、前置状态均未记录。来源类别 = **Unknown**。
    2. **设备状态不满足"出厂"条件**：同一 §11 明确记载该会话对核 0 做过 `-22 → -10` 写入，**写入后读取不能称为出厂值**；既无证据排除此前用户改动，也无"未改前首次读取"的记录。按 [measurement-evidence-schema.md](../knowledge/data/measurement-evidence-schema.md) §3，状态 = **未知（9）**。
    3. **归因不可追溯**："OEM 预设出厂偏移"的提出者与依据在仓库与提交历史中均找不到（`aa1c8de` 原文即如此，无附证）。
  - **仍可保留的有限结论**：单次观测读到过非零 CO 值（具体数值无原始日志佐证）；但"读到非零 = 出厂预设""读到 0 = 默认状态"两个推断都**不成立**——理由是"单次读取值不足以证明出厂/默认状态"，而**不是**"出厂即为非零"。
  - **补证方法**：在明确为 BIOS Load Optimized Defaults 后、任何 CO 写入前，用记录版本的工具读取全部可读核，保存原始输出并注明 CPU 身份/BIOS 版本/接口/时间，方可讨论"默认/出厂偏移"。
  - **【第六轮注记 2026-11-25：写 0 的语义】** 上条"把某核写成 0 不是恢复默认，只是取消出厂降压"中的"取消出厂降压"同样依赖已被撤回的"出厂非零"前提，一并不作为结论。就写入行为而言，写 0 = 设 0 档 = 无偏移（上游源码 Verified，见 [implementation-references.md](../knowledge/hardware/cpu/implementation-references.md) §3.2），在未调校的中性状态即为默认值；工具的"Zero all"即置 0，"ResetToBaseline"恢复的是读回基线而非某个非零出厂值。"厂商出厂普遍预置负 CO"无证据，不作常态假设。这与"读到非零/零都不能证明出厂状态"的读取语义是两个独立命题，见 [measurement-evidence-schema.md](../knowledge/data/measurement-evidence-schema.md) §1。
- **`0x1AD` 的第二组**：`IntelMSR.p` 白名单包含 `0x150`、`0x1AD`、`0x650`、`0x1A4`，写入只对这些地址开放；在 AMD 机器上模块 `main()` 直接返回 `STATUS_NOT_SUPPORTED`（Win32 50），无法实测。
- **Intel 通道已实现、未上机**：`IntelMsrTuningProvider` 读 `0x150`（11 位有符号、1/1024 V）与 `0x1AD`（1C/8C 倍频），电压偏移写入后回读比对；位域编解码有单测，真机行为待 Intel 机器验证。

## 12. A1 注记（2026-11-26）：编码修正与核映射来源

- 产品代码已删除独热位编码（`1<<(20+local)`）与"探测协商两种编码"的逻辑；掩码固定为 §3.1 的数值编码，物理位置一律由核映射给出：优先 APOB 原始表（`ZENTWIK_APOB_TABLE`），否则 CPUID `0x8000001E` 核号 + 系统 L3 缓存域，校验不过则整条通道报 `rejected`。
- 本机（9955HX / Fam1Ah Model 44）只读实测：CPUID 核号 0–15、NodeId 恒 0；L3 域掩码 `0x0000FFFF` / `0xFFFF0000` → 2 CCD × 8 核 × 2 线程。即"核号是 die 内连续编号"，CCD 必须由缓存域切分。原始数据与遗留项见 [handoffs/A1-2026-11-26.md](handoffs/A1-2026-11-26.md)。
- §11 中"bit23 不被邮箱接受""其余 14 核全部接受"均不得再作为平台事实引用；新证据反而与数值编码一致：当时被拒的两个掩码 `0x00800000` / `0x10800000` 按数值编码正好是越界核号（CCD0 核8、CCD1 核8）。掩码 `0x00000000` 被拒一事仍无解释，列为遗留项。
- CO 读写在真机仍未验证（当前会话非管理员，PawnIO `privilegeMissing`）。不得把本节的拓扑结论当作 CO 可用的证明。
