# 阶段 C：面向三类逆向情境的最小 Metric Stack

## 1. 真正要解决的问题

表面问题是：`metric-tasks.md` 提出的五层框架是否应该原样保留。

真正的问题更严格：

> 如何从真实逆向工作的决策需求出发，审查“结果、语义、证据、工具过程、鲁棒性”这五个先验层各自是否是不可替代的测量对象；在不把三篇文献强行套进同一套分数的前提下，构造一个足够小、跨场景可复用、又能揭示 agent 是否真正理解二进制并形成可审计证据链的 metric stack，并给出能够检验该 stack 有效性的标准。

这不是寻找五个漂亮指标，也不是把三篇论文已有分数加权平均。需要同时解决四个约束：

1. **通用性**：同一套概念应能描述源码恢复、密码行为重建和恶意行为审计。
2. **情境适配**：算法名、源码相似度、恶意行为证据和最终 flag 不应被误当成同一种能力。
3. **最小性**：删除一个组件后，不能仍由其他组件等价替代；否则它不是最小 stack。
4. **可检验性**：每个组件都必须对应结构化输出、真值/属性 oracle、失败模式和可证伪实验。

因此，本阶段的产物不是最终 benchmark 规范，而是一个待 pilot 的测量假设：哪些层是跨场景核心，哪些层应降为条件模块或压力测试维度。

## 2. 重新审查先验五层框架

`metric-tasks.md` 的五层是合理的起点，但它把“测量对象”和“测量条件”混在了一起：

- Task Outcome、Semantic Model、Evidence Groundedness 是 agent 产出或内部理解的不同侧面；
- Process/Tool Use 是行为过程；
- Robustness/Reliability 更像跨任务的外部效度与压力条件，而不是与结果并列的单一能力；
- Efficiency/Budget Awareness 在原五层中容易被遗漏，但对 agent 实际工作不可忽视。

三篇文献提供了具体反例：

| 文献 | 已有原语 | 暴露的测量问题 |
|---|---|---|
| Decompile-Bench | Re-Executability、R2I、Edit/Embedding/CodeBLEU、GPT-Judge、Recall@1 | 行为测试比文本接近更有意义，但有限测试不能证明全输入等价；表示指标不提供地址级证据；工具过程没有被测。 |
| CREBench | T1/T2、T3 hidden-vector execution、T4 flag、Pass@3、任务条件成功率 | T3 是较强的行为 oracle，T1/T2/T4 仍可能是参考答案或实例捷径；最终成功不等于理解和证据链；pass@3 还掩盖成本与稳定性。 |
| MalEval | ΔS、AE-F1、SAS、EAS、B-F1、RQ、FPCR/TPMR/F1c | 是三者中最接近证据链的 stack，但受静态 CIR、taxonomy、LLM judge 和调用图边界限制，不能证明动态真实性或自主工具规划。 |

初步判断：五层不应原样作为五个等权一级分数。应将其改造成“**三个核心测量面 + 两个横切验证轴**”。

## 3. 建议的最小 Stack：3 个核心面 + 2 个横切轴

### 核心面 A：验证任务结果（Verified Task Outcome, O）

回答：agent 是否完成了该情境下真正要支持的决策？

输出必须是任务特定且结构化的结果，而不是一段自由文本。例子：

- 源码恢复：多族输入/属性测试上的行为通过率，关键函数/调用边恢复；
- 加密逆向：算法与 key/IV、wrapper 行为、可接受输入的分阶段验证；
- 恶意审计：行为集合、verdict、类别、关键行为触发条件的验证。

建议原语：Verified Success、行为/路径 F1、关键定位 Top-k、隐藏测试或动态属性测试、部分分和关键错误惩罚。

约束：O 不能只采用单一最终答案。CREBench 的 T4 flag 可作为一个终局 outcome，但不能替代 T1–T3；Decompile-Bench 的有限 Re-Executability 需扩展测试族；MalEval 的 FPCR/TPMR/F1c 需与行为证据结合。

### 核心面 B：证据支撑的语义模型（Evidence-Grounded Semantic Model, S+E）

这是建议的最重要合并。原来的 Semantic Model 和 Evidence Groundedness 在逆向工作中不能完全分离：一个没有地址、路径、trace 或可执行属性支撑的“语义模型”，无法判断是否是真理解；一个只检查引用存在的“证据”，也不能证明语义正确。

每个关键 claim 要求：

```json
{
  "claim": "...",
  "semantic_object": "function|path|field|algorithm|behavior|verdict",
  "locations": ["binary address/range or function id"],
  "evidence": ["disassembly|xref|trace|test|config|diff"],
  "conditions": ["input/state/environment condition"],
  "confidence": 0.0
}
```

建议原语：

- 结构/语义：函数边界、调用边、数据流、字段偏移、算法/状态机属性、行为谓词准确率；
- 证据：Claim Support Rate、Evidence Precision/Recall、Evidence Locality、Unsupported Critical Claim Rate、Hallucinated Address Rate；
- 可验证性：引用是否真实存在、是否语义支持、是否由动态/属性 oracle 复核。

这样，Decompile-Bench 的 R2I、Edit Similarity、CodeBLEU 可作为表示诊断，不再被当作 S+E 的充分证据；CREBench 的 T3 可作为语义行为验证的一部分，但必须补 claim–evidence；MalEval 的 AE-F1/SAS/EAS/RQ 可作为 S+E 的现成原语组合。

### 核心面 C：预算内的有效分析过程（Effective Process under Budget, P）

回答：agent 是否通过有效工具行为得到 O 和 S+E，而不是靠偶然猜中或无限搜索？

过程分数不奖励调用次数，而奖励**产出被验证且被最终结论使用的证据**。记录：

- 有效工具调用率和失败恢复；
- Evidence-producing Call Rate；
- 重复/无效调用率；
- 从静态到动态、从局部到跨函数的合理策略切换；
- 首次正确关键对象的时间/调用数；
- 在不同预算下的成功—成本曲线。

建议核心汇总量为 Budget-aware Outcome AUC 或 success-at-budget，而非 Tool Coverage 的简单计数。CREBench 的 GDB 停滞现象说明过程诊断有价值；但其当前 pass@3 只取最好结果，不能替代 P。Decompile-Bench 和 MalEval 若固定输入前端或不记录 agent 工具日志，则 P 为不可观测项，不能虚构分数。

### 横切轴 R：鲁棒性与可靠性压力（Robustness/Reliability, R）

R 不作为与 O、S+E、P 等权的“第四种能力”，而作为对三类核心面施加的测试条件：

- 符号剥离、优化等级、函数内联/拆分；
- 常量变换、混淆、加壳、反射、动态加载、native bridge；
- 无关字符串、误导性 API、未知算法或 taxonomy 外行为；
- 重复运行稳定性、置信度校准、合理 abstention。

报告 `performance drop`、`calibration error`、`abstention quality` 和关键幻觉/安全错误率，而不是把所有退化合成一个不可解释总分。三篇文献当前均只能覆盖 R 的一小部分：Decompile-Bench 有 O0–O3，CREBench 有有限难度变体但无专业混淆，MalEval 有 CIR 删除扰动但不覆盖动态/native。

### 横切轴 V：测量有效性与可复现性（Validity/Reproducibility, V）

V 不是 agent 能力，而是审查 metric 是否值得信任的元指标。它检查：真值是否来自二进制可观察事实而非任意源码名；多解是否允许；评分器是否能复算；人工/LLM judge 是否校准；标注成本和跨版本稳定性如何。

没有 V，O、S+E、P 的高分都可能只是评分器偏好。Decompile-Bench 未报告若干相似度归一化和 judge 聚合细节，MalEval 的 EAS/RQ 也未完全公开数值聚合；这些不是 agent 低分，而是 metric 可解释性风险。

## 4. 最小性与“是否需要五层”的判据

建议把正式最小 stack 写成：

```text
Minimum Stack = { O: Verified Outcome,
                   S+E: Evidence-Grounded Semantic Model,
                   P: Effective Process under Budget }
Test Axes = { R: Robustness/Reliability,
              V: Validity/Reproducibility }
```

三项核心面各自不可替代：

- 删除 O，只剩“说得对且有证据”但不知是否完成真实任务；
- 删除 S+E，只剩可能猜中的 outcome 和高效过程，无法区分理解与捷径；
- 删除 P，只测最终产物，无法测工具辅助 agent 的有效性、成本和可复现工作流。

R 和 V 不应强行作为每篇论文都具备的同类任务分数。它们是验证条件：每个 benchmark 至少报告适用的 R 子集，并对 V 的未知项明确标注。

## 5. 三篇文献的情境化实例化

| 情境 | O：主结果 | S+E：最小语义/证据 | P：过程 | R/V 优先项 |
|---|---|---|---|---|
| Decompile-Bench | 多族测试/属性行为通过；关键边/字段定位 | binary 地址→控制流/数据流/参数类型 claim；R2I/CodeBLEU 仅诊断 | 工具调用、编译/测试迭代、预算内首次正确结果 | 内联/优化/函数范围错配；公开测试泄漏；相似度公式可复算 |
| CREBench | T1–T3 分阶段验证 + T4 checker；扩展输入族 | 算法、key/IV、wrapper 每项绑定地址/trace/来源；T3 隐藏测试 | GDB/静态/动态切换、无效循环、success-at-budget | 原型偏见、常量变换、未知算法；pass@3 与成本分离 |
| MalEval | 行为/触发条件/verdict/类别及误报—真阳性权衡 | AE-F1 + SAS/EAS + 地址/路径/动态支持；RQ 作为辅助 | CIR 选择、调用图查询、动态验证、证据产出效率 | native/反射/动态加载；LLM judge 校准；taxonomy 外行为 |

不同情境的权重不应相同。例如源码恢复可提高 O 中行为等价权重，恶意审计可提高 S+E 与关键漏报惩罚，密码逆向可提高 wrapper 行为和过程预算权重。权重应由 pilot 与专家判断校准，而不是先验等权。

## 6. “可检验的 metric 评估指标”：先评 metric，再评 agent

对每个候选 metric，至少报告以下七项元评估指标：

| 元指标 | 可检验问题 | 最小检验 |
|---|---|---|
| **Construct validity** | 是否真的测目标能力，而非参考文本/标签捷径？ | 语义等价重写、地址证据审计、与专家排序相关性。 |
| **Decision alignment** | 是否能区分真实决策中的好坏结果？ | 盲评“可接受/不可接受”结论，比较 metric 排序与专家排序。 |
| **Discriminative power** | 是否区分不同 agent/人类基线/预算？ | effect size、置信区间、跨任务排序稳定性。 |
| **Failure sensitivity** | 是否捕捉已知严重失败，而不是被平均分掩盖？ | 注入漏证据、错误路径、幻觉地址、错误类别，检查分数下降和 critical penalty。 |
| **Robustness** | 编译、优化、混淆、输入扰动改变时是否仍解释得通？ | 成对变体的分数下降曲线和错误类型变化。 |
| **Reproducibility** | 他人能否按公开 oracle/版本复算？ | 重跑一致性、judge 一致性、跨模型/seed 方差、缺失参数清单。 |
| **Cost and gaming resistance** | 是否可在合理成本下使用，且不奖励刷调用/猜答案？ | 记录时间、token、工具调用；比较无证据猜测与有证据分析；测试简单 shortcut agent。 |

可进一步构造一个不用于替代分数、只用于筛选指标的质量向量：

```text
MetricQuality(m) = (validity, alignment, discrimination,
                    failure-sensitivity, robustness,
                    reproducibility, cost/gaming)
```

只有在一个 metric 在关键维度达到预设最低门槛后，才进入正式 stack；不能因为它易于自动计算就纳入。

## 7. 阶段 C 的实际交付边界与待专家验证

本阶段可以交付：

1. 三核心面、两横切轴的最小 stack 假设；
2. 三篇文献的情境化实例化；
3. 候选 metric 的七项元评估标准；
4. 每个情境一个最小 pilot 和反例注入方案。

本阶段不能宣称已经确定：

- O、S+E、P 的最终权重；
- 哪些证据对每类专家是“充分证据”；
- 动态 trace、源码等价、行为 taxonomy 的统一 oracle；
- LLM judge 是否足以替代专家；
- 工具成本、分析时效和误报/漏报代价的行业通用阈值。

以上均为 **L3，待专家验证**。推荐的下一步是对每类情境选 3–5 个样本，邀请至少两名专家独立标注关键决策、可接受输出、充分证据和严重错误；然后用上表七项元指标筛掉不能预测专家判断或无法复算的候选 metric。

## 8. 结论

原五层框架不应被否定，而应被重新分工：Outcome、Semantic Model、Evidence Groundedness 是逆向结果有效性的核心，但后两者在实际测量上应合并为“证据支撑的语义模型”；Process/Tool Use 保留为工具辅助 agent 的第三个核心面；Robustness/Reliability 改为跨任务压力轴；并新增 Validity/Reproducibility 作为 metric 本身的审计轴。

因此，当前最小、兼顾通用性与三种情境适用性的假设是：

> **用 Verified Outcome 测是否完成真实任务，用 Evidence-Grounded Semantic Model 测是否真正理解并能举证，用 Effective Process under Budget 测是否以可行工具工作流完成；再用 Robustness/Reliability 和 Validity/Reproducibility 作为必须报告的横切验证轴。**

这比五个等权分数更小，也更接近三篇文献共同暴露的核心问题：最终结果可能被猜中，参考表示可能被模仿，静态证据可能被误读；只有结果、证据支撑的语义和有效过程三者同时成立，才有资格被称为工具辅助下的真实逆向能力。
