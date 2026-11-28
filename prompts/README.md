# 提示词索引

优先使用 MASTER_COORDINATOR.md。它面向具备多智能体能力的 AI，由总协调者读取仓库资料、拆分研究、自行派发子任务、汇总和交叉审查。

总协调者可根据复杂度决定是否派发，不为形式而派发。子智能体可以继续拆分，但建议最多两层递归。研究默认不修改旧项目；需要制作原型时放在本仓库 prototypes/。

## 当前任务（2026-11-28）

- [AUDIT-LEGACY-VS-NEXT-20261128.md](AUDIT-LEGACY-VS-NEXT-20261128.md)：旧项目 ↔ 新项目 ↔ 研究文档逐项对照审计。10 个子 Agent（A–J），只读静态对照，交付到 `research/handoffs/` 与 `research/audit/20261128/`。
