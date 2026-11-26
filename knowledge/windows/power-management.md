# Windows 电源方案 API

- 规范名称：Windows 电源方案与策略（Power Scheme / Power Policy）
- 英文名称：Windows Power Management Functions（PowrProf）
- 缩写：AC/DC、EPP、GUID
- 领域：`windows`
- 核验状态：Verified（Microsoft 官方文档 + 本仓库逐项注册表核对）
- 权威来源：MS Learn PowrProf 函数族；本仓库 `research/02-hardware-access.md` §4
- 相关文档：[process-scheduling.md](process-scheduling.md)、[../data/numeric-semantics.md](../data/numeric-semantics.md)

## 1. 定义

Windows 用 GUID 标识电源计划（Power Scheme）、子组（Subgroup）与设置项（Setting）。策略值分 AC（插电）与 DC（电池）两套，分别读写。

## 2. API 语义（Verified）

| 用途 | API | 关键语义 |
|---|---|---|
| 取当前方案 | `PowerGetActiveScheme` | 缓冲区由系统 `LocalAlloc`，须 `LocalFree` |
| 激活 | `PowerSetActiveScheme` | **改完值不调用它就不生效** |
| 读写 AC/DC 值 | `PowerRead/WriteACValueIndex`、`…DCValueIndex` | `RootPowerKey` 必须为 NULL |
| 枚举 | `PowerEnumerate` | 16=方案 / 17=子组 / 18=设置项；耗尽返回 259（ERROR_NO_MORE_ITEMS） |
| 新建 | `PowerDuplicateScheme` | 无 `PowerCreatePlan`，只能派生 |
| 改名/说明 | `PowerWriteFriendlyName` / `PowerWriteDescription` | UTF-16、长度按字节含结尾符 |
| 删除 | `PowerDeleteScheme` | 活动方案须先切走 |
| 供电事实 | `GetSystemPowerStatus` | 与 powrprof 无关，仅放同接口取快照 |

**存储值 vs 生效值**：`PowerWriteACValueIndex` 只改存储值；对活动方案的修改须调用 `PowerSetActiveScheme` 才生效（本仓库逐项核对过注册表，Verified）。

## 3. 权限与失败检测

- 方案键在 HKLM，受 ACL 保护；方案说明文字只能走 `PowerWriteDescription`（该键 ACL 只给 Administrators 读、全权在 SYSTEM）。
- 部分失败检测逐项做：每项返回 `(Supported, Value, Error)`，`result != 0` 即不支持并带出 Win32 码。这比"整体成功/失败"有用，须保留。
- 写入后应回读（旧版缺口：不回读，只靠快照间接验证）。

## 4. 数值语义

- EPP、BoostMode、Max/Min Processor State、CoolingPolicy、CPMINCORES、IdleDisable、ASPM 都是设置项索引，属于枚举或百分比，须按项声明类型与范围。
- AC/DC 是同一设置的两份实例，字段名与存储键必须区分，不可混用。

## 4.1 方案可见性与派生继承

`PowerEnumerate(ACCESS_SCHEME)` 返回的列表**不等于**注册表里的方案键集合：模板键可以存在而方案不出现在枚举里。2026-11-26 在 9955HX 笔记本（Windows 11，中文）逐项实测：

| 观察 | 结果 |
|---|---|
| `powercfg /L` / `ACCESS_SCHEME` 枚举 | 3 个（平衡 `381b4222`、高性能 `f54850ea`、节能 `f60bedaa`） |
| 注册表 `HKLM\SYSTEM\CurrentControlSet\Control\Power\User\PowerSchemes` 下的键 | 9 个，含标准高性能 `8c5e7fda`（别名 `SCHEME_MIN`）与标准节能 `a1841308`（别名 `SCHEME_MAX`） |
| 对未枚举的 `8c5e7fda` 执行 `PowerDuplicateScheme` | 成功，**派生品出现在枚举里**（可见） |
| 派生品的说明文字 | 继承模板（注册表里仍是 `@powrprof.dll,-12,…` 间接引用） |
| 对未枚举的 `8c5e7fda` 执行 `PowerSetActiveScheme` | 成功（返回 0），它**立刻出现在 `powercfg /L` 并带 `*`** |
| 随后 `PowerSetActiveScheme` 切回原方案 | 它**立刻从 `powercfg /L` 消失**，枚举恢复为 3 个 |

9 个注册表键的构成（同一台机）：

| 类别 | GUID | 特征 |
|---|---|---|
| 可见方案 | `381b4222` 平衡、`f54850ea` 高性能、`f60bedaa` 节能 | 有完整子键树，出现在枚举里 |
| overlay 占位键 | `3af9b8d9` 高性能、`961cc777` 最佳能效、`ded574b5` 最佳性能 | **无子键**，只有 `FriendlyName`/`Description`；不是方案，`ACCESS_SCHEME` 不返回 |
| 标准模板残留 | `8c5e7fda`（`SCHEME_MIN`）、`a1841308`（`SCHEME_MAX`）、`e9a42b02`（卓越性能） | 有少量子键，不在枚举里，但可读、可派生、可激活 |

结论（本机 Verified，跨机器 Inferred）：

- "方案被隐藏"在本机表现为**模板键存在但不在枚举结果里**，可用 `powercfg /query <GUID>` 证实其仍可读写，也可作为派生模板使用。
- **隐藏不是注册表里可写的标志**：这 9 个键都只有 `FriendlyName`/`Description` 两个值，没有 `Attributes` 之类的可见性值；被隐藏键的 ACL 与可见方案逐项一致（`Users: ReadKey`、`SYSTEM: FullControl`、`CREATOR OWNER`）。所以无法通过改注册表或 ACL 让方案重新可见。
- **激活是唯一能让这类方案进入枚举的动作，但不持久**：激活即出现，切走即消失。因此"未启用即隐藏"在具备该特征的机器上确实成立，产品侧只能靠"创建后激活一次"解决（即 `power create --activate` 的语义依据）；也不能靠"激活一次再切回"把它永久注册进去。
- overlay 键与方案是两套东西：Win11 的"电源模式"（最佳能效/平衡/最佳性能）由 `PowerSetActiveOverlayScheme` 表达，取值来自 `PowerGetEffectiveOverlayScheme`，与活动方案 GUID 无关。第三方工具用它表达"性能模式"而不必新建方案（G-Helper 即如此）。

### 读写、改名、删除一个不可见的方案

可见性只影响**枚举**，不影响**寻址**。2026-11-26 在 9955HX 实测（目标 `8c5e7fda`，全程它都不在枚举里）：

| 操作 | 结果 |
|---|---|
| 对隐藏方案枚举子组 | 6 个（对比「平衡」19 个；模板只覆盖部分子组） |
| `PowerReadValueIndex` 读「最小处理器状态」 | `AC=100`、`DC=100` |
| `PowerWriteACValueIndex` 写 50 | 返回 0；注册表 `…\PowerSchemes\8c5e7fda…\54533251…\893dee8e…\ACSettingIndex` 读回 50，`DCSettingIndex` 仍 100（AC/DC 独立） |
| 写后回读 | 一致；随后写回 100，注册表复原 |
| `PowerWriteFriendlyName` 改名 `a1841308`（标准节能） | 成功（返回 0）；名字变为新值，**方案仍是不可见**，`powercfg /L` 依然只有 3 个 |
| 写回原字符串 `@C:\WINDOWS\system32\powrprof.dll,-11,Power saver` | 成功；`PowerReadFriendlyName` 又解析为「节能」，与改名前一致 |
| 删除 | 未在真机执行（本机 3 个隐藏键全是系统模板，产品层按内置 GUID 拒删）。逻辑与改名同源，只受「内置拒删 + 活动方案先切走」约束 |

结论（本机 Verified，跨机器 Inferred）：

- **改设置项与可见性无关**：所有值接口按方案 GUID 直接寻址，方案在不在枚举里都能读写。所以"用户要改一个看不见的方案"在 API 层面不存在障碍。
- **改名同样与可见性无关，且不会让方案变可见**：可见性只跟「是否被激活」绑定，改名字、改说明、改 ACL 都不影响。所以"把隐藏方案改成看得见"这条路不存在。
- **不要用 `PowerEnumerate` 判断方案是否存在或是不是方案**。两个实测坑：
  - `ACCESS_SCHEME` 只返回可见方案，拿它做存在性校验会对隐藏方案给出假否定（表现为"方案不存在"，而 `powercfg /query` 明明能读）。
  - `ACCESS_SUBGROUP` 对**不存在**的 GUID（如 Win11 电源模式的 overlay 占位键 `3af9b8d9`/`961cc777`/`ded574b5`）可能返回成功并给出非空内容，会被误判成"这是个有子组的方案"。
  - 可靠判据是注册表 `…\User\PowerSchemes\<GUID>` 下的**子键数**：真方案有子组子键，overlay 占位键为 0。全部方案 GUID 也只能从注册表键拿到（`powrprof` 没有"列出全部方案"的接口）。
- **改系统模板有连带影响**：`8c5e7fda`（`SCHEME_MIN`）、`a1841308`（`SCHEME_MAX`）、`e9a42b02`（卓越性能）是派生母版。改动会通过"从该模板派生新方案"以及系统的"恢复默认设置"传播出去。派生是拷贝当时的取值，不改已存在的派生品。所以"能改"不等于"该改"：对模板的写入必须可回退，且应在账本里明确记成模板改动。
- 注册表里模板的 `FriendlyName` 是本地化间接引用（`@C:\WINDOWS\system32\powrprof.dll,-13,High performance`），不是纯文本。用 `PowerWriteFriendlyName` 写纯文本会破坏这条引用；写回原字符串可复原，但跨语言环境不再跟随系统语言。
- 另一处路径差异（未深究原因，仅记录）：同一进程、同一权限下，`RegOpenKeyExW` 打开 `SYSTEM\CurrentControlSet\Control\Power` 及其子键返回 `ERROR_FILE_NOT_FOUND`，而 `Microsoft.Win32.Registry` 能正常读取同一路径。实现时改用 Registry API。

### 快速启动（混合启动）与方案可见性：已排除

曾怀疑"快速启动开启时新建方案不显示"（现象：旧版只能复制方案，关闭快速启动后才正常）。2026-11-26 在 9955HX 上做对照实验排除：

| 条件 | 判定依据 | 结果 |
|---|---|---|
| 休眠关闭（每次 `Kernel-Boot` 事件 `0x0` 冷启动） | 之前的会话 | `PowerDuplicateScheme` 新建方案**可见** |
| 休眠开启后关机再开机（事件 `0x1` 混合启动） | `hibernate /a` 显示"休眠/快速启动"可用；`hiberfil.sys` 已被消费删除 | 新建方案**可见**，从隐藏模板 `8c5e7fda` 派生的方案**也可见** |

结论：**方案可见性与启动类型无关**，`PowerDuplicateScheme` 创建方案会把它登记进系统的方案列表（冷启动与混合启动会话均如此）。若现场仍出现"新建方案不显示"，先核实观察渠道——Win11「设置 → 电源和电池 → 电源模式」只呈现三个 overlay 档位，不列自定义方案；自定义方案只在控制面板「电源选项」与 `powercfg /L` 中出现。此点待后续在真机上对截图核实。

测试方法（可复现）：`Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Microsoft-Windows-Kernel-Boot'; Id=27}` 读引导类型（`0x0` 冷启动 / `0x1` 混合启动 / `0x2` 休眠恢复）。混合启动后 `C:\hiberfil.sys` 会被删除，`Test-Path` 为 false 属正常，不代表休眠未启用。
- **未复现**"从隐藏模板派生的方案继承隐藏标记"：本机从 `8c5e7fda` 派生的方案可见。旧版 Zentwik 的注释声称在部分笔记本上会继承，属真实环境观察但本机无法复现，保留为跨机器风险而非既成事实。
- 旧版为此采用的规避策略（一律从「平衡」派生，再把模板的设置项 AC/DC 全部搬过去，并搬说明文字）在结构上仍然更稳：它保证派生基线一定是可见方案。新工程沿用该策略，并把"派生品不可见"作为运行时检测条件——只有在真的检测到不可见时才回退，正常机器上仍是按模板直接派生。
- 派生品会继承模板的说明，而 `PowerReadDescription` 返回的是**已本地化文本**（间接引用被解析），所以"搬说明"写入的是解析后的字符串，不是原始 `@dll,-id,文本`。跨语言环境不保证与模板字面一致。
- 搬设置时逐项读写，单项失败跳过：部分设置项在目标方案上不接受写入（受硬件或驱动限制），为一项失败放弃整次新建会让用户连方案都建不出来。
- 「卓越性能」`e9a42b02` 的模板键只显式写了少数几项，单独搬它等于复制平衡；须先铺高性能的底再让模板自己的取值覆盖。

## 5. 回滚与持久化（项目约定 Proposed）

- 旧版电源侧无回滚；新项目必须记录前值/来源/所有权/恢复资格（见 domain/state-ownership.md）。
- 区分"计划 GUID""参数存储值""当前生效状态""用户保存状态"。

## 6. 常见误解

- "写完值就生效" → 必须重新激活活动方案。
- "普通用户也能改 HKLM 电源键" → 通常失败或被静默忽略。
- "删方案会恢复硬件" → 删除不回滚硬件，删除活动方案须先切走。

## 7. 核验状态

API 语义与枚举值 = Verified（MS Learn + 本仓库注册表核对）；权限失败的返回码细节 = Inferred（官方未逐项写"必须管理员"）。
§4.1 的方案可见性实测（枚举 vs 注册表键、派生品可见性、说明继承、设置项枚举数量 19 子组 / 174 项）= 本机 Verified（2026-11-26，9955HX）；"派生继承隐藏标记"= 旧版注释的真机观察，本机未复现，按跨机器风险处理。
