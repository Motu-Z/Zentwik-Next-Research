# Knowledge 主题索引

按主题定位规范。状态缩写：Verified(V) / Inferred(I) / Proposed(P) / Unverified(U)。

## 数值与数据（跨领域前置）

- [参数分层记录规范](data/parameter-record-schema.md) — **写任何硬件参数范围前必读**：L1 固件/L2 软件钳制/L3 实际稳定/L4 典型调校/L5 项目约定 · P
- [实测数据证据记录规范](data/measurement-evidence-schema.md) — **写任何"本机实测/出厂/默认/实测数字"前必读**：来源七分类、设备状态分类、必填字段、回读语义、结论—证据映射 · P
- [数值语义总则](data/numeric-semantics.md) — 有单位/无单位、原始值/工程值、有符号、百分比、物理量、位域、四层表示 · 混合
- [单位与换算](data/units.md) — 电压/频率/功率/能量/时间/温度换算；RAPL 单位 1/2^n（非 2^n）、FMax 上限范围；CO 与 Intel 单位对照 · V/I
- [序列化与持久化](data/serialization-persistence.md) — SchemaVersion、原子替换、多态、兼容性 · V
- [专项技术事实审计记录](AUDIT.md) — 数值/单位/层级错误的审计发现与修正、单一维护点清单 · V
- [工程约束与上游勘误补充](ENGINEERING_NOTES.md) — **跨领域避坑清单，非参数权威定义**：命名管道 MIC/`#123903`、PawnIO SDDL/IOCTL/RSMU 路径/PCI 互斥、服务恢复 flag、TPMI、WPF 渲染与 DataGrid 虚拟化、NativeAOT JSON；逐条标 Verified/Inferred/Unverified，附一手链接 · V/I/U

## 术语与命名

- [核心词汇](terminology/core-vocabulary.md) — 模块/能力/领域/Provider/方案/场景/账本的裁决点 · P
- [命名规范](terminology/naming-conventions.md) — 标识符、命令字、单位后缀、状态名 · P

## 硬件 · CPU（重点核验）

- [AMD Curve Optimizer](hardware/cpu/amd-curve-optimizer.md) — **步进档位非 mV**；范围**不对称**（负向常见 −50、正向常见 +30），显示/工具/固件三层须区分；换算、误解、实测 · 混合
- [Intel OC Mailbox](hardware/cpu/intel-oc-mailbox.md) — **11 位有符号 1/1024 V 定点、×1.024**；UVP、邮箱协议 · 混合
- [AMD SMU 链路](hardware/cpu/amd-smu.md) — 邮箱命令号、核心掩码、世代绑定、权限、三态；**FMax Override -1000…+200 MHz / 25 MHz（上限非偏移）** · 混合
- [AMD 硬件功耗墙](hardware/cpu/amd-power-limit.md) — **RSMU `0x56/0xCB/0x57/0x58` 与 MP1 `0x4F/0x3E/0x5F` 不可混用**；PM 表须先 `0x03` 传输；写入按 `applied/diverged/accepted/rejected` 四态回读；本机 0.8 折算 · 混合
- [成熟实现参考（按领域索引）](hardware/cpu/implementation-references.md) — AMD/Intel 调校等项目的已研究源码证据与实现细节 · V/I
- [跨领域参考项目目录](implementation-reference-catalog.md) — 进程、调度、电源计划、场景、监控、服务/IPC、插件、UI、CLI、安装维护等未来候选项目；区分已研究/待研究/功能对照，不代表必须引入

## 硬件 · 传感器与待研究

- [传感器与采样率](hardware/sensors/README.md) — 数据面分档、双时钟、采样≠信息量 · V/I
- [GPU 技术](hardware/gpu/README.md) — 待研究入口 · U
- [内存技术](hardware/memory/README.md) — 待研究入口；DDR5 MT/s≠MHz、时序按周期数 = Verified，其余 · U

## Windows 系统

- [电源方案 API](windows/power-management.md) — GUID、AC/DC、写值须激活 · V
- [进程调度](windows/process-scheduling.md) — CPU Sets 软约束 vs 亲和性硬约束、多组 · V
- [服务与 IPC 安全](windows/security-service.md) — LocalSystem、DACL、调用方校验、完整性 · V/U

## 架构与领域

- [模块与能力](architecture/modules-capabilities.md) — 模块/能力/Provider/插件区分、五态 · P
- [生命周期状态模型](architecture/lifecycle.md) — 模块/方案/通道/服务状态维度 · P/V
- [IPC 契约与错误模型](architecture/ipc.md) — 帧格式、错误码、背压、退出码 · V
- [安装器](architecture/installer.md) — Inno 6 选型理由与商业许可、服务逻辑单一实现、四条路径实测、BOM/pdb/等待语义等坑 · V
- [轮询预算与后台开销](architecture/polling-budget.md) — **加周期性任务前必读**：旧版三次"定时器空转"教训、当前轮询拓扑表、实测 CPU 增量（空转 −75%、带负载 −36%）· V
- [三种独立方案](domain/three-plans.md) — 三类领域对象、场景非第四方案 · P
- [场景与仲裁](domain/scenes-arbitration.md) — 触发/优先级/确定性仲裁/退出恢复 · P
- [状态账本与所有权](domain/state-ownership.md) — 前值/目标/实际/所有权/恢复资格 · P

## 按任务检索

| 任务 | 先读 |
|---|---|
| 改电压/频率偏移、设计数值字段 | 数值语义总则 → 单位换算 → 对应 CPU 规范 |
| 理解某术语是否等于某概念 | 核心词汇 → 对应领域规范 |
| 接触外部 API/寄存器 | 对应规范 → 核对官方来源章节 → 成熟实现参考看工具怎么做 |
| 做逐核调校/CORE 掩码/APOB 映射 | 成熟实现参考 → amd-smu §3.1 → amd-curve-optimizer |
| 设计 IPC/持久化 | IPC 契约 → 序列化持久化 → 领域规范 |
| 寻找进程/电源/监控/安装等第三方参考 | 跨领域参考项目目录 → 对应领域规范 → 按当前任务定点研究 |
| 遇到规范冲突 | 报告冲突，不要自行补造约定（见 [AI_GUIDE.md](../AI_GUIDE.md)） |
