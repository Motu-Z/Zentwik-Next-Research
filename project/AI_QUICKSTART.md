# 接手须知（写给下一个 AI）

你在 `D:\Projects\Zentwik Next`（产品仓库，分支 `next`）。研究仓库在 `D:\Projects\Zentwik-Next-Research`（分支 `main`）。
本文件只写**你一定会用到、且最容易搞错**的东西。完整规则在 `AI_GUIDE.md` 与 `project/AI_WORK_RULES.md`，先读那两份再看本文件之外的细节。

---

## 0. 一句话背景

Zentwik Next 是 Windows 上的 CPU/电源/进程调度工具：Agent 以 LocalSystem 服务跑命名管道服务端，CLI 与 WPF 桌面端是客户端。三个独立领域对象（CPU 调校方案 / 电源方案 / 调度方案）+ 场景协调 + 所有权账本。

## 1. 开工前五分钟

```bash
cd "D:/Projects/Zentwik Next"
git fetch origin next && git status --short && git log --oneline -3
dotnet build Zentwik.slnx -c Release        # 必须 0 警告
dotnet test  Zentwik.slnx -c Release        # 当前 160 例
sc.exe query ZentwikAgent                    # 服务应在 Running
```

服务二进制在 `C:\Program Files\Zentwik\Agent`。改了 Agent/Core/Platform 之后必须重新发布并替换，否则你验证的是旧代码：

```bash
dotnet publish src/Zentwik.Agent/Zentwik.Agent.csproj -c Release -o .tmp/publish-agent
# 管理员 PowerShell：
sc.exe stop ZentwikAgent; Copy-Item ".tmp\publish-agent\*" "C:\Program Files\Zentwik\Agent" -Recurse -Force; sc.exe start ZentwikAgent
```

CLI 用 `dotnet src/Zentwik.Cli/bin/Release/net10.0/zentwik.dll <命令>`；`--help` 列出全部命令。

## 2. 事实源分工（别搞混）

| 问题 | 只看这个 |
|---|---|
| 当前代码行为 | 产品仓库源码，不看文档描述 |
| 领域不变量、验证分级、性能门槛 | `project/AI_WORK_RULES.md` |
| 硬件编码、命令号、单位 | `knowledge/hardware/cpu/`（`amd-smu`、`amd-curve-optimizer`、`intel-oc-mailbox`、`implementation-references`） |
| 进程调度 / 电源 / 服务安全的 Windows 语义 | `knowledge/windows/` |
| 旧版做过什么、踩过什么坑 | `research/01-legacy-inventory.md`、`research/02-hardware-access.md`、`D:\Projects\Zentwik Project\.workbuddy\memory\` |
| 明确不修的东西 | `project/KNOWN_LIMITS.md` |
| 上一轮改了什么、验证到哪一步 | `research/handoffs/` 最新那份 |

**没有证据就不要下结论。** 尤其不要凭"看起来像"判断旧版没做某功能——旧版功能比你想的完整。

## 3. 当前真实状态（截至 2026-11-26 夜）

- 真机平台：Ryzen 9 9955HX（Zen5，family `1Ah` model `44h`），16 核 32 线程，单处理器组。
- 服务：已重装到 `C:\Program Files\Zentwik\Agent`，LocalSystem，Running。
- 已真机验证：A1 逐核 CO 读取（16/16）、A2 CO 写入+回读+回退、A3 FMax 基线 4450→写 4750→回读→回退、B1 中/高完整性 CLI 连 Session 0 服务、B2 亲和性/优先级/CPU Sets 三路径、C1 挂起恢复通知注册。
- 未验证：Intel 全通道、真实休眠唤醒回调、重启后保持、多处理器组。见 `KNOWN_LIMITS.md`。
- 测试：160 例（Domain 38 / Platform 30 / Core 52 / Contracts 40 / Desktop 2）。

## 4. 七条会让你当场出错的事

1. **不要逐进程 `OpenProcess`。** 枚举进程只允许 `NativeProcessList`（`CreateToolhelp32Snapshot` 一次快照）。访问每个进程的 `ProcessName`/`StartTime` 会退化成每进程一次句柄操作，300 进程下 10 条规则实测 2.55% 单核。创建时间只对账本里的 PID 按需单查（`StartTicksOf`）。
2. **说"旧版没做"之前先 grep 旧项目。** 旧项目在 `D:\Projects\Zentwik Project\Zentwik`，关键实现都在 `Zentwik.Core/Services/`。我因为没查，误判"核心停泊两端都没做"，实际旧版有 `ProcessWatcher.ParkingSuffix` + `PowerPolicyService` 读写 `CPMINCORES`。
3. **验证分级不许混用。** 单测通过 ≠ fake 通过 ≠ 管理员会话通过 ≠ Session 0 通过 ≠ 真机闭环。写结论时必须写明是哪一级。`AI_WORK_RULES.md` §8。
4. **回读不比对等于假成功。** 任何写入路径：写前读基线、写后读回读、按容差或等价规则比对、不一致记 `Failed`、读不到记 `Unknown`。CPU Sets 的字符串比较要先 `CpuSetSpec.TryNormalize`。
5. **改用户看得见的文案/语义/排期，先问。** 删字段、改提示语、改命令输出格式都算。
6. **不要用"概念更完整"当理由加抽象。** 我加过 `SceneAction.PlanRevision`（场景钉方案版本），用户直接否掉：改方案场景就该跟着变。用户不是模型，不要替他做偏好假设。
7. **CI 看 API 不看网页。** `https://api.github.com/repos/Motu-Z/Zentwik/actions/runs?branch=next&per_page=3` 拿 `head_sha`/`conclusion`；网页版经常只显示登录页。

## 5. 代码地图（只列容易踩的）

| 文件 | 职责 | 陷阱 |
|---|---|---|
| `src/Zentwik.Core/Plans/PlanCoordinator.cs`（约 1900 行） | 唯一写入口。三条 Apply 路径：`ApplyCpu`/`ApplyPower`/`ApplyScheduling`，加 `EnforceSchedulingAsync` 持续执行 | 直接应用与持续执行是**两套并行实现**，改一处必须同步另一处（CPU Sets 回读比对曾只修了一套）。账本键 `sched.{pid}.{field}` 不带创建时间，PID 复用在 `DropStaleSchedulingEntries` 与 `ProcessStillAlive` 里处理 |
| `src/Zentwik.Domain/Ownership/StateLedger.cs` | 所有权、前值、状态机 | `IsHeld` = Owner 为 Scene/Plan 且 Status 为 Applied/Unknown。任何"是否仍持有"的判定都用它，不要自己写 `Owner == Scene && Status == Applied`（会漏 Plan 与 Unknown） |
| `src/Zentwik.Core/Scenes/SceneEngine.cs` | 激活/停用/仲裁/重应用 | 锁顺序固定 Scene → Plan，不要反向 |
| `src/Zentwik.Core/State/StateStore.cs` | 原子写、损坏备份 | record 缺字段反序列化得到 `null` 而不是初值，集合字段必须兜空 |
| `src/Zentwik.Platform/Windows/AffinityScheduler.cs` | 亲和性/优先级/CPU Sets | 读回必须另开 `PROCESS_QUERY_LIMITED_INFORMATION` 句柄；写 CPU Sets 前查硬亲和性与存活；多处理器组直接拒绝 |
| `src/Zentwik.Platform/Windows/AmdSmuTuningProvider.cs` | CO / FMax | 每次 SMU 发送持 `Global\Access_PCI`；MP1 型号显式报不支持；FMax 基线 0 不等于不支持 |
| `src/Zentwik.Platform/Windows/IntelMsrTuningProvider.cs` | 0x150 / 0x1AD | 读也是"先写读命令"；必须持 `Global\Access_Intel_OC_Mailbox` |
| `src/Zentwik.Agent/AgentServer.cs` | 管道接受循环 | 客户端断开必须 `continue`；拒绝连接先回 Unauthorized 帧再关 |
| `src/Zentwik.Contracts/PipeIdentity.cs` | 客户端识别服务端 | 令牌优先、安全描述符所有者兜底；SYSTEM 令牌所有者常是 `S-1-5-32-544` |

## 6. 改完自检

- `dotnet build Zentwik.slnx -c Release`：0 警告 0 错误（Debug 也要过）。
- `dotnet test Zentwik.slnx -c Release`：全绿，新增行为有测试。
- 涉及硬件/调度的改动：重新发布服务 → 真机跑一次 → 用系统侧独立工具复核（`Get-Process`、`GetProcessDefaultCpuSets`）。
- 涉及持久化的改动：手写一份旧格式 `state.json` 验证能加载。
- 提交前：`git diff` 自查有没有把 `.tmp/` 或临时方案带进去。

## 7. 交付怎么说

- 先说结论与实测数字，再说改了什么；不贴大段代码。
- 明确区分"已验证（哪一级）"与"未验证"。
- 修了 5 项就说 5 项，不要把"有依据但没修"混进已完成。
- 研究仓库同步更新：`research/handoffs/` 记本轮结论，`project/KNOWN_LIMITS.md` 记不修项。
