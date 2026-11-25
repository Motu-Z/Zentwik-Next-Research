# Knowledge 主题索引

按主题定位规范。状态缩写：Verified(V) / Inferred(I) / Proposed(P) / Unverified(U)。

## 数值与数据（跨领域前置）

- [数值语义总则](data/numeric-semantics.md) — 有单位/无单位、原始值/工程值、有符号、百分比、物理量、位域、四层表示 · 混合
- [单位与换算](data/units.md) — 电压/频率/功率/能量/时间/温度换算；CO 与 Intel 单位对照 · V/I
- [序列化与持久化](data/serialization-persistence.md) — SchemaVersion、原子替换、多态、兼容性 · V

## 术语与命名

- [核心词汇](terminology/core-vocabulary.md) — 模块/能力/领域/Provider/方案/场景/账本的裁决点 · P
- [命名规范](terminology/naming-conventions.md) — 标识符、命令字、单位后缀、状态名 · P

## 硬件 · CPU（重点核验）

- [AMD Curve Optimizer](hardware/cpu/amd-curve-optimizer.md) — **步进档位非 mV**；范围、换算、误解、实测 · 混合
- [Intel OC Mailbox](hardware/cpu/intel-oc-mailbox.md) — **11 位有符号 1/1024 V 定点、×1.024**；UVP、邮箱协议 · 混合
- [AMD SMU 链路](hardware/cpu/amd-smu.md) — 邮箱命令号、核心掩码、世代绑定、权限、三态 · 混合

## 硬件 · 传感器与待研究

- [传感器与采样率](hardware/sensors/README.md) — 数据面分档、双时钟、采样≠信息量 · V/I
- [GPU 技术](hardware/gpu/README.md) — 待研究入口 · U
- [内存技术](hardware/memory/README.md) — 待研究入口 · U

## Windows 系统

- [电源方案 API](windows/power-management.md) — GUID、AC/DC、写值须激活 · V
- [进程调度](windows/process-scheduling.md) — CPU Sets 软约束 vs 亲和性硬约束、多组 · V
- [服务与 IPC 安全](windows/security-service.md) — LocalSystem、DACL、调用方校验、完整性 · V/U

## 架构与领域

- [模块与能力](architecture/modules-capabilities.md) — 模块/能力/Provider/插件区分、五态 · P
- [生命周期状态模型](architecture/lifecycle.md) — 模块/方案/通道/服务状态维度 · P/V
- [IPC 契约与错误模型](architecture/ipc.md) — 帧格式、错误码、背压、退出码 · V
- [三种独立方案](domain/three-plans.md) — 三类领域对象、场景非第四方案 · P
- [场景与仲裁](domain/scenes-arbitration.md) — 触发/优先级/确定性仲裁/退出恢复 · P
- [状态账本与所有权](domain/state-ownership.md) — 前值/目标/实际/所有权/恢复资格 · P

## 按任务检索

| 任务 | 先读 |
|---|---|
| 改电压/频率偏移、设计数值字段 | 数值语义总则 → 单位换算 → 对应 CPU 规范 |
| 理解某术语是否等于某概念 | 核心词汇 → 对应领域规范 |
| 接触外部 API/寄存器 | 对应规范 → 核对官方来源章节 |
| 设计 IPC/持久化 | IPC 契约 → 序列化持久化 → 领域规范 |
| 遇到规范冲突 | 报告冲突，不要自行补造约定（见 [AI_GUIDE.md](../AI_GUIDE.md)） |
