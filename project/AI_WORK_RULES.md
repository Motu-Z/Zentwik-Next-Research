# AI 跨对话工作规范（Zentwik）

- 状态：生效（2026-11-26 建立）
- 适用：所有接手 Zentwik / Zentwik Next 的 AI 对话，包括产品代码、研究文档、真机验证
- 权威顺序：**用户当次明确指示 > 本文 > `AI_GUIDE.md` > 单次对话上下文**
- 维护：新教训按"事实 + 出处 + 违反后果"追加到 §10，不新开文件

---

## 1. 事实源分工（先查再动）

| 要查什么 | 唯一正本 | 不是 |
|---|---|---|
| 技术事实、参数、单位、寄存器 | 研究仓库 `knowledge/`（先读 `INDEX.md`） | 不是对话记忆、不是搜索摘要 |
| 为什么这样决策 | `decisions/ADR-*.md` | 不是交接文档的口头结论 |
| 老产品怎么做、踩过什么坑 | `D:\Projects\Zentwik Project\.workbuddy\memory\`（`MEMORY.md` + 两份细则 + 逐日日志） | 不是"老项目没这功能"这类未经检索的判断 |
| 老产品某个功能的实现细节 | `D:\Projects\Zentwik Project\Zentwik\Zentwik.Core\Services\` 等源码 | 不是凭印象重写 |
| 当前产品状态 | `D:\Projects\Zentwik Next` 代码与 `next` 分支 HEAD | 不是交接文档里的旧 HEAD |
| 参考项目 | `D:\Projects\Reference Projects\`（RyzenAdj / SMUDebugTool / ZenStates-Core-AOT） | 不是 README 摘要 |

**硬规则**

1. 涉及硬件参数、单位、范围，先读 `knowledge/data/` 三件（`numeric-semantics` / `units` / `parameter-record-schema`），并说明属于 L1 固件 / L2 软件钳制 / L3 实际稳定 / L4 典型 / L5 项目约定 哪一层。
2. 涉及"本机实测/默认/出厂"，先读 `measurement-evidence-schema.md`，标注来源七类。
3. 涉及系统 API、Windows 行为、驱动、AOT、WPF，先读 `knowledge/ENGINEERING_NOTES.md`——它已逐条一手核验过，别重新猜。
4. **涉及进程调度、监控采样、常驻开销，先读老项目 `Zentwik-Docs/Zentwik-进程调度优化分析.md` 与 `Zentwik.Core/Services/NativeProcessList.cs` / `ProcessWatcher.cs` / `ProcessInstanceLedger.cs`**。老项目在这些点上做过真机实测，结论可直接引用（见 §4）。

## 2. 评估老项目的基本态度

- 老项目（`D:\Projects\Zentwik`，v2.6.0-can16、360 单测、UiSmoke、xamlcheck、itest 全绿、真机使用一个月）是**基准，不是待推翻的草稿**。
- 只陈述可核实事实：某功能有/无、某实现用 A 还是 B、某处有已知缺陷。**不做"老项目不行/很乱/该重写"这类价值判断**；确实存在缺陷时，写明条件、现象、证据路径。
- 重构默认目标是"同等能力下更清楚、更省资源"，不是"换一套写法"。凡老项目已解决的问题（WMI 卡顿、读回权限、PID 复用、停泊核交集、掩码单组），**先复用结论，再谈改进**。
- 老项目没做的功能（如常驻调度纠偏），可以新做，但不得据此暗示老项目"残缺"。

## 3. 动手前的固定流程

1. 读当前仓库 HEAD 与 `git status`，确认工作区是否混有别的对话改动。
2. 按 §1 找到对应正本，**打开文件读**，不用记忆代替。
3. 需要参考实现时，打开参考项目/老项目的**具体函数**，不看 README 就下结论。
4. 形成一句话技术判断（改什么、依据什么、影响什么），再动手。
5. 复杂或高风险（硬件写入、服务重装、协议变更、持久化格式）先给简明评估等确认；低风险修复直接做。

## 4. 性能与常驻开销门槛

**已实测基准（2026-11-26，Zen5 9955HX / 32 线程）**

| 场景 | 老项目 | Zentwik Next（当前） |
|---|---|---|
| 空闲常驻 CPU | ~0.1% 单核（1 秒一次 Toolhelp32 快照） | 0.052% 单核（2 秒一轮） |
| 进程调度 10 条规则 | 未直接对应（老版只做差集绑定） | 0.83% 单核 |
| 内存 | ~88 MB（UI+Worker） | 42 MB（仅 Agent） |

**红线**

1. **禁止逐进程 `OpenProcess` 做全表枚举。** 名字用 `CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS)` 一次快照；创建时间只对账本里的 PID 单查（`OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION)` + `GetProcessTimes`）。
2. **禁止用 WMI 做高频监听。** 老项目实证：`__InstanceCreationEvent WITHIN 2` 让 `WmiPrvSE` 累计 6418s CPU、8.5% 单核，并拖慢任务管理器。
3. 快照复用：一轮内多条规则共用一份快照（TTL 1s），不为每条规则重枚举。
4. 高频成功动作日志走 Debug 只落盘；失败才 WARN/ERROR。禁止每轮写盘。
5. 声称"性能优化"必须给**改造前后实测数字**（同一场景、同一测量方法），不接受"理论更快"。
6. 轮询周期取舍：老项目 1 秒扫描（只做差集）、参考项目 cpu-affinity-tool 3 秒纠偏；本项目 2 秒（含读回）。改周期要说明对"进程启动到绑定"延迟的影响。

## 5. 调度领域红线（老项目实证，直接沿用）

- 读回/解绑必须另开 `PROCESS_QUERY_LIMITED_INFORMATION` 句柄；只写句柄读 Affinity 或默认 CPU Sets 都返回错误 5。
- 已退出进程仍可打开句柄并读到退出前的值，`SetProcessDefaultCpuSets` 还会"假成功"；动之前先 `GetExitCodeProcess == 259`。
- PID 复用按创建时间判定，全量轮清理；账本键/条目必须带实例标识。
- `SetProcessDefaultCpuSets` 不校验核心停泊，返回 TRUE 但实际范围 = 默认集 ∩ 未停泊核（"绑核成功却跑飞"的真因）。
- 亲和性掩码只覆盖单处理器组；>64 位由上层拒绝并记日志，不静默截断。
- CPU Sets 是软约束、亲和性是硬约束，二者叠加取交集；切换类型先清另一侧。
- 失败重试：2 秒起翻倍、5 次为限、上限 60 秒；写后回读不一致记 `-2`，读不到不判失败。
- 不得用 `System.Management` / `ServiceController`（AOT `IL3052`）。

## 6. 领域不变量（改代码不得违背）

1. CPU 调校方案、电源方案、调度方案是三个独立领域对象，各有生命周期；不存在跨领域"当前方案"。
2. 场景是跨领域协调机制，不是第四种方案；保存方案后引用它的活动场景应立即按最新内容重应用（`PlanRevision` 概念已删除，2026-11-26 决策）。
3. 方案四层态势：定义 / 期望 / 观测 / 实际；`Actual ≠ Desired` 即未生效，不得写"已应用"。
4. 账本记录 Target、prev、desired、actual、source、owner、priority、restorable、failure；**所有权释放后再次应用，prev 取当前实际值**，不得沿用陈旧快照。
5. 用户接管（`Owner=User`, `Restorable=false`）不得被场景/自动恢复覆盖。
6. 部分成功、跳过、失败、结果未知必须分开表达；`LedgerStatus` 含 `Unknown`。
7. 回退失败保留 `Applied` + `Restorable=true` 供重试。
8. 方案仍被账本持有写入时，禁止删除（应先回退）。
9. 休眠/唤醒后先读实际值再决定是否重写，禁止无条件重放。

## 7. 工程与平台红线

- 管道：接受循环里客户端断开必须 `continue`（写成 `break` 会让进程活着但不再建管道）；客户端不用 `PipeOptions.CurrentUserOnly`（`dotnet/runtime#123903` 未修复），用显式 DACL + 服务端校验 SID/完整性/Session。
- 服务恢复：`SERVICE_FAILURE_ACTIONS_FLAG.fFailureActionsOnNonCrashFailures` 必须显式设 TRUE，且**下次系统启动才生效**。
- PawnIO：`0x841`/`0x821` 是 function code，不是完整 IOCTL；SDDL 在 INF（仅 SY/BA）；PCI 访问需共享 `\BaseNamedObjects\Access_PCI`。
- Intel 0x150：OC Mailbox 协议，必须写命令字（`0x80000011`/`0x80000010`）、轮询 bit63 忙位、持 `Global\Access_Intel_OC_Mailbox`、写后经读命令回读。无 Intel 真机时明确标"仅位域单测"。
- 持久化：写临时文件 → `Flush(flushToDisk: true)` → 原子替换；读取失败先备份 `state.json.corrupt` 再按空状态继续。
- 状态文件缺集合字段按空集合处理（STJ 对 record 缺字段返回 null，不是初值）。
- NativeAOT：非泛型 `JsonStringEnumConverter` 不支持，用泛型或 blanket policy。
- 模块状态：跨厂商能力（AMD/Intel）按"任一适用"聚合，另一厂商缺席不是 degraded。

## 8. 验证分级（不得混用）

| 级别 | 证据形态 | 能写成 |
|---|---|---|
| 单测通过 | `dotnet test` 全绿 | "单测覆盖" |
| fake 链路 | Fake Provider 走通 | "模拟验证" |
| 构建/发布 | 0 警告 0 错误、AOT 单文件 | "构建通过" |
| 管理员会话实测 | 高完整性 CLI + 运行中服务 | "管理员会话验证" |
| Session 0 实测 | LocalSystem 服务 + 用户会话通信 | "服务实测" |
| 真机硬件写入 | 写入 + 回读 + 回退，系统侧独立复核 | "真机闭环" |

- 构建前不得声称测试通过；`dotnet build Zentwik.slnx -c Release` 与 `dotnet test` 是两条独立结论。
- 硬件写入只算"闭环"当且仅当：写前读基线、写入、回读一致、可回退且已回退。
- 未验证项必须单独列出，写在交付结尾，不混入完成项。

## 9. 交付、文案与仓库纪律

- 改用户可见文案、交互语义、排期前**先问**；技术实现细节自决。
- 界面不加副标题/说明小字，语义进 ToolTip；不给操作回执文字。
- 注释只留一句事实或约束，禁"从为什么开始"的叙述。
- 提交：`fix: 条件 + 现象` / `feat: 能力` / `perf: 对象 + 手段` / `docs: 动作`；只 add 本次文件，**禁 `git add -A`**（可能混入别的对话改动）；本机用 `GIT_AUTHOR_*` 临时导出，不写 config。
- 临时文件只进项目内 `.tmp/`，任务收尾删除；不写盘符根、不写 `%TEMP%` 散落文件（探测脚本除外，需说明路径）。
- 真机验证不得改变用户已调好的硬件参数；验证后必须复位到验证前的实测值并核对。
- 不轮询 CI/构建/下载进度；发起即交付。

## 10. 已发生的错误（禁止重犯）

| # | 错误 | 事实 | 出处 |
|---|---|---|---|
| 1 | 把 `Process.GetProcessesByName` 换成 `Process.GetProcesses()` + 逐进程读 `ProcessName`/`StartTime`，10 条规则 2.55% 单核 | 老项目 `NativeProcessList` 注释已写明"一次 Toolhelp32 快照、不 OpenProcess、300 进程约 1ms"，并记录了 WMI 8.5% 单核的事故 | `2026-11-11.md` §2.2.2；`NativeProcessList.cs` |
| 2 | 说"老项目没有调度功能，所以它 0% 是因为没有这功能" | 事实正确但表述为贬低，且未先读老项目文档就下结论 | 用户 2026-11-26 明确指正 |
| 3 | 用 `PlanRevision` 钉住场景引用的方案版本 | 用户改方案就是要场景跟着变；概念不成立，已删字段与错误码 | `AgentSession`/`Scene` 2026-11-26 |
| 4 | 平台不匹配与"缺 CCD 拓扑"两种安全降级判成不同状态（一个 `Failed` 且"失败 0 项"） | 同属跳过，应统一 `Skipped → NotApplied` | `PlanCoordinator.ApplyCpu` |
| 5 | 客户端用管道安全描述符所有者判服务身份 | SYSTEM 令牌默认所有者是 `S-1-5-32-544`，普通用户读不到服务进程 SID（错误 5）；正确顺序是"进程令牌优先、所有者兜底" | `PipeIdentity` 2026-11-26 |
| 6 | `CpuTuningProbe` 把 `Available` 映射成 `Degraded`，模块聚合又把另一厂商缺席算降级 | AMD 机器上 `cpu-tuning` 长期显示 degraded | `WindowsCapabilityProbes` 2026-11-26 |
| 7 | 所有权释放后复用旧 `PreviousValue` | revert 会把目标写回陈旧快照，覆盖用户后续改动 | `StateLedger.Acquire` |
| 8 | 删除仍被账本持有的方案 | 条目变孤儿，永久无法回退 | `AgentSession.DeletePlanAsync` |
| 9 | 状态文件写入未 `FlushToDisk`，损坏后无备份 | 断电可留半截；损坏后第一次保存就覆盖唯一副本 | `StateStore` |

## 11. 自检（每次交付前）

- [ ] 我读过 §1 对应的正本文件，而不是凭记忆？
- [ ] 涉及性能时给了改造前后实测数字，并对比老项目基线？
- [ ] 我有没有用价值判断描述老项目？
- [ ] 硬件写入是否写前读基线、写后回读、可回退、已复位？
- [ ] 未验证项是否单独列出，没有混进完成项？
- [ ] 提交是否只含本次文件？临时文件是否已清理？
