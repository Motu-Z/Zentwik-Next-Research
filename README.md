# Zentwik Next Research

Zentwik Next 的产品定义、知识体系、技术预研、架构决策与原型验证工作区。本仓库是长期事实来源，AI 与人类均通过索引检索。

## 项目关系

- **Zentwik（旧项目）**：现有功能、算法、硬件行为与历史实现的事实来源：`https://github.com/Motu-Z/Zentwik`（main）。
- **Zentwik Next**：重新定义产品和架构后的新实现，产品源码在 `Motu-Z/Zentwik` 的 `next` 分支；旧 main 不改。
- **本仓库**：研究、知识规范、决策与交接；只维护 main。

旧项目是迁移参考，不是新项目的设计边界。

## 目录结构

| 目录 | 职责 |
|---|---|
| `project/` | 产品目标、架构约束、路线图 |
| `knowledge/` | **经核验的长期知识与规范**（术语、硬件、Windows、架构、领域、数据） |
| `research/` | 研究报告、实验、技术调查与交接 |
| `decisions/` | ADR 与正式架构决策 |
| `prototypes/` | 隔离的实验原型（不代表产品实现） |
| `templates/` | 统一文档模板 |
| `prompts/` | AI 工作提示词 |
| `AI_GUIDE.md` | AI 使用整个仓库的入口规则与检索流程 |

## 快速入口

- **AI 使用者**：先读 [AI_GUIDE.md](AI_GUIDE.md)，再按 [knowledge/INDEX.md](knowledge/INDEX.md) 定位领域。
- **查技术术语/单位**：[knowledge/GLOSSARY.md](knowledge/GLOSSARY.md) 与 [knowledge/data/numeric-semantics.md](knowledge/data/numeric-semantics.md)。
- **查产品目标与约束**：[project/PROJECT_BRIEF.md](project/PROJECT_BRIEF.md)、[project/ARCHITECTURE_CONSTRAINTS.md](project/ARCHITECTURE_CONSTRAINTS.md)。
- **查原始研究证据**：[research/README.md](research/README.md)。
- **查正式决策**：[decisions/](decisions/)。

## 当前阶段

阶段 0 已完成（产品定义、旧版盘点、硬件链路、UI/IPC、技术栈评估与 ADR）。M2 完成应用层拆分、CLI 正式入口、服务化骨架、AMD CO 通道与场景接入账本；M3 完成 IPC 拒绝显式报错、服务启停与崩溃恢复动作、场景退出/重启回退、界面可访问性与列表虚拟化实测。硬件实测补充（CO 步进语义纠正、SYSTEM 可开 PawnIO、bit23 部分核被拒）见 `research/02-hardware-access.md` §11。

## 核心原则

1. 以真实需求为中心，证据优先，区分事实、推断、假设和建议。
2. 允许重新设计，不机械继承旧技术栈、抽象和缺陷。
3. 适度扩展：收益明确、成本合理且紧密相关的能力可纳入。
4. 不为模块化而模块化；抽象必须带来隔离、复用、替换或治理收益。
5. 优先验证硬件访问、驱动权限、实时数据、自动化恢复和插件隔离等高风险点。
6. 同一技术定义只在一处维护；旧资料移动时更新所有引用，不删除有价值的历史报告。
7. 文档以简体中文为主，技术名词和代码标识保留原文。

## 核心业务模型

CPU Tuning Plan、Power Plan、Scheduling Plan 必须保持三个独立领域对象，各有完整生命周期。场景不是第四种方案，而是跨领域协调机制，也可执行独立动作。自动化必须处理多规则命中、优先级、状态所有权、用户手动接管、退出恢复和执行历史；禁止无条件快照回写覆盖用户后续修改。

模块、能力、业务领域、硬件平台、产品功能和 UI 页面是不同概念；区分安装、启用、加载、运行和健康状态。插件系统为长期方向：内置模块、官方扩展、第三方插件分层，第三方默认独立进程，第一版不要求插件市场但预留版本化协议、权限、隔离与兼容。

国际化从第一版纳入，首批 zh-CN、zh-TW、en-US，运行时切换与缺失回退；用户自建名称不随语言切换改变。Windows 是首要目标，跨平台按能力逐项定义。

## 知识体系的核验约定

所有规范区分 **A. 外部技术事实**（厂商/OS/标准/维护者定义）与 **B. 项目内部约定**（Zentwik 自己选择），并标注 Verified / Inferred / Proposed / Unverified / Deprecated。例如：AMD Curve Optimizer offset 是**无单位步进档位，不是 mV**；Intel OC Mailbox `0x150` 是 11 位有符号电压定点编码（1/1024 V），二者不可混为一谈。详见 [knowledge/hardware/cpu/](knowledge/hardware/cpu/)。
