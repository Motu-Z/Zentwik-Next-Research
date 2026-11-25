# Research 索引

本目录保存**研究报告、实验数据、技术调查与阶段交接**——原始过程与证据。经过核验、可长期复用的结论已提炼到 [`knowledge/`](../knowledge/)，正式决策在 [`decisions/`](../decisions/)。

## 报告

- [01-legacy-inventory.md](01-legacy-inventory.md) — 旧版能力与实现盘点（基线 bc3c59b / 2.6.0-can16）
- [02-hardware-access.md](02-hardware-access.md) — AMD SMU / Intel MSR / 电源 / 调度链路（含 §11 硬件实测补充）
- [03-ui-ipc-deployment.md](03-ui-ipc-deployment.md) — UI 框架、进程模型、IPC、部署
- [04-tech-stack-evaluation.md](04-tech-stack-evaluation.md) — 全栈技术评估（含实测数据）
- [RESEARCH_EFFICIENCY.md](RESEARCH_EFFICIENCY.md) — 研究效率与任务消耗控制

## 交接

- `handoffs/M1-2026-11-25.md` — 骨架与纵向切片
- `handoffs/M2-2026-11-25.md` — CLI、服务化、硬件通道
- `handoffs/M3-2026-11-25.md` — IPC 拒绝、服务恢复动作、场景退出回退、界面可访问性实测

## 约定

- 报告使用 `../templates/RESEARCH_REPORT.md`。
- 报告命名建议：`主题-YYYYMMDD.md`。
- 标注约定：`【码】` 源码事实、`【文档】` 官方/上游资料、`【推断】` 有依据未实测、`【建议】` 决策主张、`【未验证】` 需上机确认。
- 同一技术定义只在 `knowledge/` 维护一份，本报告引用而不复制；报告保留论证过程与原始数据，不删除历史成果。
