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

## 5. 回滚与持久化（项目约定 Proposed）

- 旧版电源侧无回滚；新项目必须记录前值/来源/所有权/恢复资格（见 domain/state-ownership.md）。
- 区分"计划 GUID""参数存储值""当前生效状态""用户保存状态"。

## 6. 常见误解

- "写完值就生效" → 必须重新激活活动方案。
- "普通用户也能改 HKLM 电源键" → 通常失败或被静默忽略。
- "删方案会恢复硬件" → 删除不回滚硬件，删除活动方案须先切走。

## 7. 核验状态

API 语义与枚举值 = Verified（MS Learn + 本仓库注册表核对）；权限失败的返回码细节 = Inferred（官方未逐项写"必须管理员"）。
