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

结论（本机 Verified，跨机器 Inferred）：

- "方案被隐藏"在本机表现为**模板键存在但不在枚举结果里**，可用 `powercfg /query <GUID>` 证实其仍可读写，也可作为派生模板使用。
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
