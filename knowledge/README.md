# Knowledge & Standards 知识体系

本目录是经过核验、可长期复用的**结论性知识**与**项目规范**。它与 `research/`（研究过程与证据）、`decisions/`（正式决策）、`project/`（产品目标与约束）职责分离。

## 核心原则

1. **事实与约定分离**：每份规范区分 A. 外部技术事实（厂商、操作系统、标准、维护者定义）与 B. 项目内部约定（Zentwik 自己选择）。不得把项目约定伪装成官方定义。
2. **数值语义优先**：涉及数值先读 [data/numeric-semantics.md](data/numeric-semantics.md)，再读具体规范。
3. **核验状态显式**：Verified / Inferred / Proposed / Unverified / Deprecated，可分项标注，资料不足时明确标记，不编造单位、范围或 API 行为。
4. **单一维护点**：同一技术定义只在一处维护，其他文档引用，不重复抄写。
5. **来源可访问**：重要结论附官方来源，尽量精确到章节/函数/访问日期。

## 目录职责

| 目录 | 内容 |
|---|---|
| [terminology/](terminology/) | 通用术语与命名规范 |
| [hardware/cpu/](hardware/cpu/) | AMD / Intel CPU 技术（SMU、CO、OC Mailbox） |
| [hardware/gpu/](hardware/gpu/) | NVIDIA / AMD GPU（待研究入口） |
| [hardware/memory/](hardware/memory/) | 内存（待研究入口） |
| [hardware/sensors/](hardware/sensors/) | 传感器与硬件监控 |
| [windows/](windows/) | 系统 API、服务、安全、调度、电源 |
| [architecture/](architecture/) | 软件架构、模块、能力、Provider、插件、生命周期 |
| [domain/](domain/) | Zentwik 领域模型、三类方案、场景、仲裁、状态所有权 |
| [data/](data/) | 数值语义、单位、数据类型、序列化、持久化、兼容性 |

## 检索入口

- [INDEX.md](INDEX.md)：按主题导航
- [GLOSSARY.md](GLOSSARY.md)：全局术语索引
- [AI_GUIDE.md](../AI_GUIDE.md)：AI 使用整个仓库的规则与检索流程
- [_TEMPLATE.md](_TEMPLATE.md)：新规范模板

## 与其他目录的关系

- `research/` 保留原始研究、实验与交接资料；有长期价值的结论提炼到本目录，原始论证不删除。
- `decisions/` 保留正式采用的 ADR；本目录记录支撑决策的事实与约定。
- 旧资料移动时须更新所有引用（如 ADR 中的 `research/02` 引用）。

## 当前成熟度

已建立规范：AMD Curve Optimizer、Intel OC Mailbox、数值语义总则、单位换算、三类方案、场景仲裁、状态所有权、模块与能力、生命周期、IPC、Windows 电源/调度/安全、命名规范、核心词汇。
待研究入口：GPU、内存、部分传感器通道、休眠唤醒、RAPL 单位细节。
