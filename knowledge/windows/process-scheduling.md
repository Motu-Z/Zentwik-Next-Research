# Windows CPU Sets 与亲和性调度

- 规范名称：进程调度约束（CPU Sets / Affinity Mask / 处理器组）
- 英文名称：CPU Sets, Thread Affinity Mask, Processor Groups
- 缩写：CPU Set、Affinity
- 领域：`windows`
- 核验状态：Verified（Microsoft 官方文档）+ 本仓库源码事实
- 权威来源：[MS Learn — CPU Sets](https://learn.microsoft.com/en-us/windows/win32/procthread/cpu-sets)（访问 2026-11-25）；本仓库 `research/02-hardware-access.md` §5
- 相关文档：[../hardware/cpu/amd-smu.md](../hardware/cpu/amd-smu.md)、[power-management.md](power-management.md)

## 1. 定义（Microsoft 原文 / Verified）

CPU Sets 提供以"软（soft）"方式声明应用亲和性的 API，与操作系统电源管理兼容。辅助机制 Process Default 可将进程内后台线程重关联到处理器子集。

 affinity mask（亲和性掩码）是硬约束：线程只能在掩码内运行。

## 2. 二者差异（Verified）

| 维度 | CPU Sets | Affinity Mask |
|---|---|---|
| 约束强度 | 软约束，调度器优先在集合内放置，可越界 | 硬约束 |
| 优先级 | 低于亲和性：`affinity mask is respected above any conflicting CPU Set assignment` | 高 |
| 宽度 | ID 数组，天然无 64 位限制 | 64 位/组，单组 |
| 多组 | `On multi-group systems, CPU assignments are ignored if they are in groups that do not match the group in the thread's affinity mask` | 需逐组设置 |
| 权限 | `PROCESS_SET_LIMITED_INFORMATION` | `PROCESS_SET_INFORMATION` |
| 最低系统 | Win10 / Server 2016 | 更早 |

## 3. 官方行为要点（Microsoft 原文 / Verified）

- Process Default CPU Sets 只影响"未显式设置线程集合"的线程；新建线程自动继承。
- 独占分配给其他进程的 CPU Set，其请求会被忽略，线程被调度到别处。
- 文档称"传入合法参数时不会失败"，但保护进程/权限不足仍可能失败，不能省错误处理（本仓库推断 Inferred）。

## 4. 本仓库实现事实（Verified）

- 两组 API 都实现，切换前先清另一侧，因二者叠加取交集。
- 拓扑来自 `GetSystemCpuSetInformation`，按 `LastLevelCacheIndex` 分组；`Type` 字段是 2 字节，推进步长用回填的 `Size`（曾因按 4 字节读导致拓扑恒空）。
- >64 逻辑处理器时亲和性直接拒绝，不用 128 位掩码硬塞 64 位。
- CPU Set 是进程属性，进程重启丢失，需 watcher 补绑；Toolhelp32 快照差集 1 秒一轮，不用 WMI（WMI 常驻吃 8% 单核）。
- 解绑只清"本工具成功绑过"的 PID，不碰用户自己设的亲和性。
- PID 复用检测用创建时间；漂移每约 5 秒只记录不重写。

## 5. 单位与取值

- 优先级 6 档（含 EcoQoS，不含 RealTime）——项目约定。
- 优先级数值越低越高（Windows 惯例）；`SetPriorityClass` 用枚举常量，不直接用数字。

## 6. 常见误解

- "CPU Sets 是硬绑定" → 软约束，亲和性优先。
- "写一次掩码永久生效" → 进程重启丢失，需 watcher。
- ">64 核用 128 位掩码就能表达" → 亲和性执行层只接受单组 64 位。

## 7. 核验状态

Microsoft 文档事实 = Verified；本仓库实现细节 = Verified（源码）；错误处理的权限推断 = Inferred。
