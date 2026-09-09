# MalEval 文献内容总结

## 一、文献内容总结

| 模板项 | 内容 |
|---|---|
| 工作名称 | **Is “Knowing It’s Malicious” Enough? Evaluating LLMs for Fine-Grained Malware Behavior Auditing**（2026）。作者：Xinran Zheng 等。正式发表信息为 *Proc. ACM Softw. Eng.* 3(ISSTA), Article ISSTA096, October 2026，DOI：[10.1145/3832187](https://doi.org/10.1145/3832187)；预印本：[arXiv:2509.14335](https://arxiv.org/abs/2509.14335)。 |
| 研究动机 | 传统分类器只给出“恶意/ benign”信号；签名方法缺少语义，程序分析覆盖有限，学习式 XAI 难以追溯到具体代码。LLM 审计还面临三项评测障碍：缺少细粒度人工行为真值、大型代码超出上下文窗口、无法可靠验证行为声明是否有代码证据支持。因此需要能定位失败阶段、以证据为基础、可复现比较的评测框架。 |
| 评测对象 | 以 **Android APK 应用**为样本（230 个恶意、25 个良性），在样本内部评测可达函数、函数调用上下文、原子证据、行为标签和最终样本判定；不是单个孤立函数或协议。基准含 654,188 个可达函数、62 份专家报告、56 个家族、6 类恶意软件。 |
| 被测能力 | 四阶段逆向/审计能力：T1 函数优先级（找出安全关键函数）；T2 证据归因（从代码提取 `〈action, asset, target〉` 并链接支持函数）；T3 行为综合（把分散证据组成攻击链和行为结论）；T4 样本甄别（纠正良性误报、保留真实恶意、归属恶意类别）。 |
| 输入与工具 | 输入来自 APK 的 Manifest、生命周期/框架回调、反编译 Dalvik 代码、CHA 风格调用图及 BFS 可达函数；每个函数扩展一跳 caller/callee，生成 CIR（Context-driven Intermediate Representation），按敏感度排序，最多输入 300 个 CIR（受模型上下文窗口限制）。模型先总结 caller/callee，再生成 CIR 和结构化报告。工具/实现包括 Androguard、FlowDroid 风格入口点、vLLM（开源模型）、商业模型 API；温度 0。论文未报告允许被测 Agent 使用动态调试器、网络检索或额外逆向工具，评测实例主要是静态 CIR。 |
| 目标任务 | 在给定被上游检测器标记的 APK 上，完成函数筛选、证据提取与支持、行为级叙述、恶意 verdict 与类别；对良性 APK 要推翻误报，对恶意 APK 要保持正确判定。输出必须经过约束结构化 CoT 协议。 |
| 输出结果 | 原子证据列表（动作/资产/目标及支持函数）、规范化行为标签、总体行为摘要、`malware/benign` verdict、恶意类别（及家族信息在真值中）；模型自由文本先被转换为该结构。 |
| 正确标准 | 专家厂商报告经统一协议转换为代码可落地真值：证据三元组、MalRadar 行为标签、摘要、verdict/类别。四名博士研究者检查引用证据存在性、动作/资产/目标标签、证据到行为逻辑支持、行为与摘要一致性，并修正 11 个 target、8 个 asset、9 个行为标签错误。预测证据按受控同义等价类匹配，行为按 MalRadar taxonomy，样本按良性/恶意及类别比较。 |
| 检查方式 | 证据：对预测/真值三元组做加权受控语义匹配、一次一配，阈值 `S≥0.8` 后计算 AE-F1；SAS 检查引用函数是否在输入，EAS 由三个异构 LLM judge 判断函数是否语义支持证据。行为：B-F1 比较标签集合，RQ 由 LLM judge 按目标识别、叙事连贯、逻辑可推导三维评分。样本：计算 FPCR、TPMR、类别宏 F1。另用移除 top 10% 敏感 CIR 的扰动计算 ΔS；50 个随机样本×7 模型（350 报告）人工审查失败模式，judge 与人工 RQ 仅 8/350 不一致。 |
| 失败原因 | 论文报告五类：FM-1 决定性证据缺失（34.48%）；FM-2 证据误解；FM-3 攻击链组合失败（23.56%）；FM-4 威胁归因失败（35.63%）；FM-5 无支持外推。FM-1、FM-3、FM-4 合计超过 93% 的错误；摘要另指出决定性证据缺失与攻击链组合失败合计超过 58% 的审计错误。 |
| 评测局限 | 当前实例仅覆盖 Android Dalvik 字节码与 Manifest 的静态前端；不覆盖 native code、WebView JavaScript、动态加载 payload、运行时专有行为。CHA 调用图/BFS 不能完整建模反射、动态加载、native callback、框架分发，换调用图或动态轨迹可能改变函数池和 CIR。专家报告数量少于样本且部分按家族级；EAS、RQ 依赖 LLM-as-a-judge；良性集合为 25 个模拟误报，不能代表全部真实 SOC 分布。评测只说明受控输入/输出协议下的能力，不能证明真实环境端到端审计、动态行为发现或工具自主规划能力。 |

## 二、Metric 记录

### 1. Sensitivity Index（ΔS）

| 项目 | 记录 |
|---|---|
| 指标名称 | Sensitivity Index（敏感度指数，ΔS） |
| 衡量内容 | 模型对排名最高、最关键 CIR/代码证据的依赖程度，用于 T1 函数优先级。 |
| 计算公式 | `ΔS = AE-F1_full − AE-F1_retracted`。后者移除敏感度最高的 10% CIR，并用原先未入选的低排名 CIR 替换，保持输入大小不变。 |
| 统计单位 | 论文未用一句话明确规定聚合层级；由定义可知以样本-模型运行（再汇总模型平均）为单位。 |
| 分数含义 | 高 ΔS 表示移除关键函数后证据 F1 明显下降，模型确实依赖高价值证据；低值表示不敏感或依赖先验/表面线索。 |
| 实际问题 | 检验模型是否优先查看真正驱动恶意行为的函数，而非只凭恶意先验或表面词。 |
| 实际场景 | 适用于大 APK 中函数筛选、上下文裁剪和检验检索/排序质量；边界是只能评估被 CIR 和静态排名覆盖的证据，不能证明遗漏的动态/native 函数不重要。 |
| 不能说明什么 | 不能单独说明证据语义正确、行为链完整或最终类别正确；高值也可能来自真值/排序共同偏差。 |
| 主要影响因素 | top-10% 比例、替换策略、敏感度排序、AE-F1 阈值/等价类、CIR 上限（300）、输入上下文窗口和测试样本。 |

### 2. Atom Evidence F1（AE-F1）

| 项目 | 记录 |
|---|---|
| 指标名称 | Atom Evidence F1 score（AE-F1） |
| 衡量内容 | 原子证据三元组 `〈action, asset, target〉` 的识别与代码归因准确性。 |
| 计算公式 | 先算 `S(e_p,e_g)=0.6 s_action+0.3 s_asset+0.1 s_target`；组件精确相等或落入预定义等价类得 1，否则 0。一次一配，`S≥τ=0.8` 为 TP，未配预测为 FP、未配真值为 FN；再按常规 `P=TP/(TP+FP)`、`R=TP/(TP+FN)`、`F1=2PR/(P+R)`。 |
| 统计单位 | 原子证据项在样本内匹配，结果通常按样本/模型汇总；论文未报告更细的宏/微平均公式。 |
| 分数含义 | 高表示既找到了更多真值证据，又少报/少漏；低表示证据恢复或归因失败。 |
| 实际问题 | 解决“模型说有恶意行为但拿不出具体、可核验代码证据”的痛点。 |
| 实际场景 | 适合静态代码证据抽取、支持函数链接和细粒度行为审计；不适合直接评价动态执行轨迹或自由文本解释。 |
| 不能说明什么 | 不保证引用函数语义真的支持证据（需 EAS），不保证证据能组成完整攻击链或正确类别。 |
| 主要影响因素 | 受控词表和等价类、权重（动作 0.6/资产 0.3/目标 0.1）、阈值 0.8、真值报告质量、模型可见 CIR、一次匹配策略。 |

### 3. Evidence Authenticity Score（EAS）

| 项目 | 记录 |
|---|---|
| 指标名称 | Evidence Authenticity Score（证据真实性分数，EAS） |
| 衡量内容 | 被引用函数在语义上是否支持模型声称的原子证据，检测证据幻觉。 |
| 计算公式 | 论文未报告显式数值公式或阈值；仅说明由 LLM-as-a-judge 对代码-证据一致性进行语义判断。 |
| 统计单位 | 论文未明确；语义上以每条引用证据/样本报告评分后汇总。 |
| 分数含义 | 高表示引用函数与声称证据语义一致；低表示引用存在但不能支撑结论。 |
| 实际问题 | 区分“引用了真实存在的函数”与“函数真的证明了恶意操作”。 |
| 实际场景 | 适合审计报告的代码支撑核验；边界是依赖 judge，不能替代人工或形式化程序分析。 |
| 不能说明什么 | 不能说明证据列表召回完整、行为标签正确或样本分类正确；也不能消除 judge 偏差。 |
| 主要影响因素 | judge 模型/提示词、代码上下文、证据表述、真值结构化质量、语义等价标准。 |

### 4. Syntax Authenticity Score（SAS）

| 项目 | 记录 |
|---|---|
| 指标名称 | Syntax Authenticity Score（语法真实性分数，SAS） |
| 衡量内容 | 报告中引用的函数是否实际出现在模型输入 CIR 中。 |
| 计算公式 | 论文未报告显式公式；定义为对引用函数存在性的检查。 |
| 统计单位 | 论文未明确；根据定义可按引用函数/报告统计。 |
| 分数含义 | 高表示少有虚构函数名或输入外引用；低表示引用了模型不可见/不存在的函数。 |
| 实际问题 | 检测“代码证据引用幻觉”，保证引用至少来自可见输入。 |
| 实际场景 | 适合约束静态 CIR 审计输出；不能验证函数语义，也不能覆盖模型从外部工具获得的真实代码。 |
| 不能说明什么 | 函数存在不等于支持恶意证据；不能说明行为、攻击链或 verdict 正确。 |
| 主要影响因素 | 输入 CIR 覆盖率、函数去重/命名、上下文上限、输出解析及引用格式。 |

### 5. Behavior F1（B-F1）

| 项目 | 记录 |
|---|---|
| 指标名称 | Behavior F1（行为 F1，B-F1） |
| 衡量内容 | 预测的规范化恶意行为标签集合与 MalRadar 真值行为集合的一致性。 |
| 计算公式 | 论文未另写展开式；按标签集合的标准 F1（行为 TP/FP/FN 的 precision、recall 调和平均）计算。 |
| 统计单位 | 样本级行为标签集合，结果跨样本/模型汇总。 |
| 分数含义 | 高表示模型能把证据归纳为正确行为；低表示行为遗漏、误报或标签归因错误。 |
| 实际问题 | 衡量从多个原子证据到可读、可比较行为语义的综合能力。 |
| 实际场景 | 适合行为 taxonomy 级审计与模型比较；受限于 MalRadar 标签，不能表达 taxonomy 外的新行为。 |
| 不能说明什么 | 不检查每个行为是否有充分代码支持（EAS）、叙事因果是否连贯（RQ）或类别是否正确（F1c）。 |
| 主要影响因素 | taxonomy 覆盖、标签真值、集合匹配/平均方式、证据召回和行为映射提示词。 |

### 6. Report Quality（RQ）

| 项目 | 记录 |
|---|---|
| 指标名称 | Report Quality（报告质量，RQ） |
| 衡量内容 | 结构化审计叙述的总体理解质量，包含 Objective Identification、Narrative Coherence、Logical Derivability 三维。 |
| 计算公式 | 论文未报告显式加权公式、量表范围或聚合规则；采用 LLM-as-a-judge 对照结构化真值评分。 |
| 统计单位 | 每份样本报告，经多个 judge 评分后汇总；具体宏/微平均未报告。 |
| 分数含义 | 高表示目标识别准确、叙述连贯且结论能由机制推出；低表示叙述缺目标、跳跃或不可推导。 |
| 实际问题 | 弥补 BERTScore 只看词汇相似、不能判断安全因果正确性的缺陷。 |
| 实际场景 | 适合评价面向分析师的行为报告可读性和因果质量；边界是主观 judge 评分。 |
| 不能说明什么 | 高 RQ 不保证所有证据/行为都覆盖，也不保证恶意类别正确；论文发现 RQ 与类别正确性可不一致。 |
| 主要影响因素 | judge 模型、评分提示词、结构化真值、报告语言表达、三维评分标定和语义熵。 |

### 7. False Positive Correction Rate（FPCR）

| 项目 | 记录 |
|---|---|
| 指标名称 | False Positive Correction Rate（误报纠正率，FPCR） |
| 衡量内容 | 对被上游检测器误报的良性 APK，模型正确判为 benign 的能力。 |
| 计算公式 | 论文未给出显式公式；根据定义可推理为 `正确纠正的良性误报数 / 良性误报总数 × 100%`。 |
| 统计单位 | 良性样本（APK）级，再跨良性集合汇总。 |
| 分数含义 | 高表示能过滤误报；低表示过度把可疑样本判成恶意。 |
| 实际问题 | 解决 SOC 中“上游检测器把良性应用送来后无法纠偏”的痛点。 |
| 实际场景 | 适合衡量下游审计层的误报过滤；25 个良性样本且为模拟误报，外部代表性有限。 |
| 不能说明什么 | 不能说明恶意召回、行为证据质量或类别归因；单独优化 FPCR 可能导致把恶意都判 benign。 |
| 主要影响因素 | 良性样本选择/比例、verdict 阈值和提示词、输入上下文、样本分布漂移。 |

### 8. True Positive Maintenance Rate（TPMR）

| 项目 | 记录 |
|---|---|
| 指标名称 | True Positive Maintenance Rate（真阳性保持率，TPMR） |
| 衡量内容 | 对真实恶意 APK 保持正确 malware 判定、避免新增假阴性的能力。 |
| 计算公式 | 论文未给出显式公式；根据定义可推理为 `仍正确判恶意的恶意样本数 / 恶意样本总数 × 100%`。 |
| 统计单位 | 恶意样本（APK）级，再跨恶意集合汇总。 |
| 分数含义 | 高表示审计不会因纠正误报而漏掉真实威胁；低表示假阴性多。 |
| 实际问题 | 与 FPCR 配对，防止模型为降低误报而过度否定恶意样本。 |
| 实际场景 | 适合 SOC 二次审计的保真度监控；不能替代家族/行为级诊断。 |
| 不能说明什么 | 不说明恶意类别、行为完整性、代码证据真实性；恶意 verdict 正确也可能理由错误。 |
| 主要影响因素 | 恶意样本类别/家族分布、verdict 规则、输入截断、上游标签和真值报告质量。 |

### 9. Category F1（F1c）

| 项目 | 记录 |
|---|---|
| 指标名称 | Category F1-score（类别 F1，`F1c`） |
| 衡量内容 | 六类恶意软件（adware、spyware、trojan、banker、rootkit、ransomware）的细粒度威胁类别归因。 |
| 计算公式 | 宏平均类别 F1：各类别按 TP/FP/FN 计算 F1 后取平均；论文未写出展开式，但明确为 macro F1。 |
| 统计单位 | 恶意 APK 样本的类别标签，按类别宏平均。 |
| 分数含义 | 高表示能判断哪类威胁最具诊断性；低表示虽可能识别一般恶意行为，却无法正确归因类别。 |
| 实际问题 | 解决“报告看起来合理但把 rootkit/banker 等归成泛化 Trojan”的威胁归因痛点。 |
| 实际场景 | 适合威胁情报分流、家族/类别优先级判断；受当前六类 taxonomy 和样本分布限制。 |
| 不能说明什么 | 不表示行为证据或叙述质量；不能推断未覆盖类别、家族级或真实运行时能力。 |
| 主要影响因素 | 类别不平衡、taxonomy 定义、真值类别标签、类别决策提示词、样本数量和输入证据。 |

## 三、文献内容总结索引

| 模板项 | 论文页码与大致位置 | 关键原文/证据 |
|---|---|---|
| 工作名称 | p.1 顶部、ACM Reference Format | “Is ‘Knowing It’s Malicious’ Enough?...”; “2026... Article ISSTA096”; DOI 链接。 |
| 研究动机 | pp.1–3，摘要与 §1 | “three hurdles: ... lack of detailed... ground truth... context limits... verify...” |
| 评测对象 | pp.5–7，§3.2、Table 1 | “255 Android applications... 230 malware samples and 25 benign”; “654,188 reachable functions”; “62 expert reports”。 |
| 被测能力 | pp.3、9–12，§1/§3.6.2 | “T1: Function Prioritization... T2: Evidence Attribution... T3: Behavioral Synthesis... T4: Sample Discrimination”。 |
| 输入与工具 | pp.5–10，§3.2–§3.5；p.12 §4.1.2 | “decompiled code and reachable function call graph”; “FlowDroid-style”; “Androguard”; “cap of 300 CIRs”; “temperature set to 0”。未报告动态调试器/外部检索权限。 |
| 目标任务 | pp.3、9–12 | “four stage-wise evaluation tasks”; “reject benign false alarms while preserving true threats”。 |
| 输出结果 | pp.9–10，§3.4–§3.5 | “atomic evidence triplets... normalized behavior labels... final verdict/category”; 图 2 的结构化报告示例。 |
| 正确标准 | pp.8–11，§3.4、§3.6.1；p.13 §4.1.3 | “controlled vocabularies”; “pairs with S ≥ τ (τ=0.8)”; “manually audited by four PhD-level security researchers”。 |
| 检查方式 | pp.10–13，§3.6、§4.1.2–4.1.3 | “AE-F1... one-to-one matching”; “SAS... EAS... LLM-as-a-judge”; 三个 judge、50 样本人工审计、8 mismatches。 |
| 失败原因 | pp.3、17–19，Key Findings 与 §4.5/Table 8 | “FM-4... 35.63%”; “FM-1... 34.48%”; “FM-3... 23.56%”; 五类失败模式定义。 |
| 评测局限 | pp.19–20，§4.6 | “Native code, WebView JavaScript, dynamically loaded payloads... outside this front end”; CHA “cannot fully model... reflection, dynamic loading...” |

## 四、Metric 记录索引

| Metric | 页码与大致位置 | 关键原文/公式 |
|---|---|---|
| ΔS | p.11 §3.6.1；p.12 §3.6.2；p.13 §4.1.2 | “ΔS = AE-F1_full − AE-F1_retracted”; “removing the top 10% highest-sensitivity CIRs... keeping input size unchanged”。 |
| AE-F1 | pp.10–11 §3.6.1 | “S(e_p,e_g)=ω1·s_action+ω2·s_asset+ω3·s_target”; “ω1=0.6, ω2=0.3, ω3=0.1”; “pairs with S≥τ... τ=0.8”; TP/FP/FN。 |
| EAS | p.11 §3.6.1；p.13 §4.1.2 | “Evidence Authenticity Score (EAS)... whether those functions semantically support the claimed evidence”; “LLM-as-a-judge”。显式公式未报告。 |
| SAS | p.11 §3.6.1；图3 p.10 | “Syntax Authenticity Score (SAS), which verifies that all cited functions exist in the input”。显式公式未报告。 |
| B-F1 | p.11 §3.6.1；p.12 §3.6.2 | “measures whether the set of inferred behavior labels aligns with the ground-truth behaviors defined under the MalRadar taxonomy”。标准 F1 展开式未直接写出。 |
| RQ | p.11 §3.6.1；p.13 §4.1.2 | “Objective Identification... Narrative Coherence... Logical Derivability”; “LLM-as-a-judge”。显式公式/量表未报告。 |
| FPCR | p.11 §3.6.1；p.12 §3.6.2 | “capacity to dismiss benign false alarms”; 图3 “Measure how many benign false alarms are correctly dismissed”。分母公式未直接报告，比例形式为根据定义推理。 |
| TPMR | p.11 §3.6.1；p.12 §3.6.2 | “ability to preserve correct malware judgments without introducing new false negatives”。分母公式未直接报告，比例形式为根据定义推理。 |
| F1c | p.11 §3.6.1；p.12 §3.6.2 | “Category F1-score (F1c), the macro F1 across malware categories”。 |

> 注：页码按 PDF 印刷页码（论文第 1–22 页）记录；“未报告”表示正文没有给出该模板所要求的更细公式、阈值、聚合或流程细节，而非对其实现的否定。对 FPCR/TPMR 的比例分母、B-F1 标准展开式等，已明确标注为“根据定义推理”。
