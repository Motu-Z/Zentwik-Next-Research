# AI_GUIDE：AI 使用本仓库的规则

你是使用 Zentwik Next Research 仓库的 AI。本文件是你检索与产出的**入口规则**，优先于任何单次对话的临时指示。

## 1. 仓库职责地图

| 目录 | 用途 | 你怎么用 |
|---|---|---|
| `project/` | 产品目标、架构约束、路线图 | 理解产品不变量与边界 |
| `knowledge/` | 经核验的长期知识与规范 | **技术问题的首选事实源** |
| `research/` | 研究报告、实验、交接 | 查证据、原始数据、历史论证 |
| `decisions/` | ADR 正式决策 | 查已采纳决策与推翻条件 |
| `prototypes/` | 实验原型 | 查可运行验证与非目标 |
| `templates/` | 统一模板 | 写新文档时套用 |
| `prompts/` | AI 工作提示词 | 复杂研究任务时参考 |

## 2. 检索流程（必须遵守）

1. **先读总索引**：`knowledge/INDEX.md`，确定相关领域。
2. **定位领域**：进入 `knowledge/` 对应子目录，读具体规范；不确定术语时查 `knowledge/GLOSSARY.md`。
3. **涉及单位与数值**：先读 `knowledge/data/numeric-semantics.md` 与 `knowledge/data/units.md`，再读具体硬件规范。**不得凭记忆判断单位。**
4. **涉及硬件参数范围**：先读 `knowledge/data/parameter-record-schema.md`。任何范围必须回答"属于 L1 固件实际 / L2 软件钳制 / L3 实际稳定 / L4 典型调校 / L5 项目约定 哪一层"，并给出单位、步进、适用代际与来源。**不得把某一层当成另一层，尤其不得把软件钳制或典型调校值冒充固件/硬件限制。**
5. **涉及外部 API/寄存器/硬件行为**：核对规范中列出的官方来源，并（在资料过期或冲突时）复核官方文档章节。Verified 必须是**实际打开原始资料并定位到具体章节/字段/代码行**；搜索摘要、二手转述、其他 AI 生成内容不得标 Verified。
6. **区分事实、推断与项目约定**：每份规范标注了 Verified / Inferred / Proposed / Unverified / Deprecated，并区分外部技术事实（A）与项目内部约定（B）。产出时保持这种区分。
   - **涉及本机实测/出厂/默认/任何实测数字**：先读 `knowledge/data/measurement-evidence-schema.md`。每条数据必须标注来源七类之一（User-provided / Direct observation / Reproduced / Derived / Inferred / Example / Unknown）与读取时的设备状态。**不得默认用户设备处于出厂状态，不得把用户自设参数当固件默认，不得把 Derived/Inferred/Example/Unknown 改写为实测事实，不得把 User-provided 升格为 Direct observation。** 没有原始日志就写"来源 Unknown/原始日志缺失"，不得编造实验过程。
7. **查证据与历史**：需要原始实验数据、论证过程或交接背景时读 `research/`；需要"为什么这么决定"时读 `decisions/`。

## 3. 冲突处理

- 遇到规范之间、规范与官方来源、规范与代码之间的冲突：**明确报告冲突**，列出双方出处、版本与日期，不要把结论平均化，也不要自行选边。
- 发现旧结论错误、证据过期或前提变化：指出差异并建议更新原规范，不静默覆盖。
- 无可靠资料的领域（如 GPU、内存的细节）：保持 Unverified，只写待研究条目，**不得编造定义、单位、范围或 API 行为**。

## 4. 禁止行为

- 不得自行补造尚未定义的项目约定；需要新约定时标注为 Proposed 并说明需决策。
- 不得把项目约定伪装成厂商/操作系统的官方定义。
- 不得把 AMD CO offset 标成 mV，不得把 Intel 0x150 的电压编码与 AMD CO 混为一谈。
- 不得孤立记录数值范围：必须分层（L1 固件 / L2 软件钳制 / L3 实际稳定 / L4 典型调校 / L5 项目约定），无单位整数（枚举、位域、档位）同样适用。
- 不得把 FMax 绝对频率、Boost Clock Override 偏移范围与实际运行频率三者画等号。
- 不得在多个文件重复维护同一段技术定义；引用而不是复制。
- 不得删除或覆盖 `research/` 中的历史报告与交接文件；移动资料时更新所有引用。
- 不得把"未验证"写成"不可行"，也不把"理论可行"写成"已实现"。
- 不得把"读到非零值"归因于出厂预设，不得把"读到零值"当作恢复默认，不得把回读一致当作实际生效（见 [measurement-evidence-schema.md](knowledge/data/measurement-evidence-schema.md)）。
- 不得在没有原始日志时声称"本仓库实测"；不得为凑齐字段自行补造设备型号、工具版本、测试时间或实验步骤，缺失的写 Unknown。

## 5. 产出要求

- 写新规范用 `knowledge/_TEMPLATE.md`。
- 写研究报告用 `templates/RESEARCH_REPORT.md`；新报告放 `research/`，命名 `主题-YYYYMMDD.md`。
- 写决策用 `decisions/TEMPLATE.md`。
- 所有技术结论附来源（尽量精确到章节/函数）与访问日期；标注核验状态。
- 文档以简体中文为主，技术名词与代码标识保留原文。
- 输出先给结论与证据，长数据写入文件、对话中只给摘要与路径。

## 6. Git 边界

- 研究仓库：`Motu-Z/Zentwik-Next-Research`（main）。
- 产品源码：`Motu-Z/Zentwik` 的 `next` 分支；本仓库的整理**不得触碰产品仓库**。
- 不 force push；提交前检查 diff，确认没有误删研究成果。

## 7. 快速自检（回答前）

- [ ] 我先读了 INDEX 吗？
- [ ] 涉及数值时我查了 data/（含 parameter-record-schema.md 的分层规则）吗？
- [ ] 我说明了每个范围属于哪一层，没有把软件钳制/典型值当固件限制吗？
- [ ] 涉及外部事实时我**打开**了官方原文并定位到具体章节/行吗（不是只看搜索摘要）？
- [ ] 涉及实测数据时我标注了来源七类与设备状态吗？没有原始日志的我没有写成"实测事实"吗？
- [ ] 我区分了事实、推断和项目约定吗？
- [ ] 遇到冲突我报告了吗，而不是自行补造？
