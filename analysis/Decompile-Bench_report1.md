# Decompile-Bench 文献内容总结

## 一、文献内容总结

| 项目 | 内容 |
|---|---|
| 工作名称 | **Decompile-Bench: Million-Scale Binary-Source Function Pairs for Real-World Binary Decompilation**，2025（arXiv v2，19 Oct 2025）。链接：[arXiv:2505.12668](https://arxiv.org/abs/2505.12668)，[代码/数据](https://github.com/albertan017/LLM4Decompile)。 |
| 研究动机 | 现有公开二进制-源码基准要么规模小、竞赛/合成数据与真实世界分布不符，要么只提供变量/类型或行级碎片映射；DWARF 缺失、优化和函数内联又使准确映射困难。因此需要大规模、公开、真实发布级二进制-源码函数对和防数据泄漏的评测集。 |
| 评测对象 | Decompile-Bench-Eval 中的函数级二进制/汇编样本：HumanEval、MBPP 手工翻译并在 O0–O3 编译的 C/C++ 函数，2025 年后 GitHub 仓库函数（GitHub2025，约 60K），以及 ProRec；另有二进制-源码检索任务。 |
| 被测能力 | 从低级二进制/汇编恢复可读源码、功能行为、控制流、变量名、类型和语义相似性；在检索任务中从二进制查询找回对应源码函数。 |
| 输入与工具 | 输入为二进制（论文说明将 binary 与 assembly 视为等价）及提示中的反汇编文本；模型/工具包括 IDA、Ghidra、LLM4Decompile、Idioms、GPT-4.1-mini、Claude-Sonnet-4-reasoning 等。实验用 Linux x64、Clang-19、C++17；LLM 生成用 vLLM、512 新 token、greedy decoding。具体提示模板、反汇编预处理格式对所有基线的完整细节：未报告。 |
| 目标任务 | 对每个二进制函数生成源码，使其行为与原始源码一致并尽量可读、接近参考源码；检索实验则以二进制函数为 query，从源码库返回真实对应函数。 |
| 输出结果 | 解编器输出的 C/C++ 函数源码；检索输出排序后的候选源码函数列表。GPT-Judge 还输出变量命名、控制流、类型恢复三个 1–100 分数及理由（JSON）。 |
| 正确标准 | 理想功能标准是对所有输入 `x` 满足 `s(x)=d(x)`；实际用给定 unit tests `T` 上全部输出一致判定 Re-Executable。可读性用 R2I，文本接近度用 Edit Similarity；检索正确是真实源码匹配进入 top-k（报告 Recall@1）。 |
| 检查方式 | HumanEval/MBPP：将 Python 解答及测试手工翻译为 C/C++，二进制按 O0–O3 编译，解编后运行原测试。R2I 从 AST 提取预定义特征并按权重归一化；Edit Similarity 基于 Levenshtein；Embedding 用 CodeSage 向量余弦；CodeBLEU 综合 BLEU、加权 n-gram、AST 子树和数据流；GPT-4 评审 GitHub2025 输出；检索按余弦相似度排序并统计 top-k 命中。GitHub2025 的真实项目单元测试端到端评测：论文明确称仍很困难，未作为主 Re-Executability 集合报告。 |
| 失败原因 | 论文未按模型逐例报告失败原因。论文仅观察到高优化级别会使部分模型性能下降（如 Idioms 在 O2/O3），并指出真实项目重执行受数据泄漏和人工构建成本阻碍；编译优化/内联造成源码范围不匹配（约 25% 抽样对），属于真实编译伪影而非算法错误。 |
| 评测局限 | 测试输入是有限 unit tests，不能证明对所有输入等价；真实 GitHub 项目缺少可扩展、无泄漏的端到端重执行评测。R2I 只适用于 C；编辑/嵌入/CodeBLEU 可能偏好表面或表示相似，不能单独证明功能正确；GPT-Judge 有成本、主观性和模型偏差。数据主要来自 permissive-license、公开 GitHub，排除了大量商业/混淆软件；复杂度、语言、平台和工具覆盖有限。 |

## 二、Metric 记录

### Metric 1：Re-Executability（重执行率）

| 项目 | 内容 |
|---|---|
| 指标名称 | Re-Executability，也称 I/O accuracy 或 pass rate |
| 衡量内容 | 解编函数与原函数在输入输出行为上的一致性。 |
| 计算公式 | 理想定义：`ReExes(d) ⇔ ∀x∈X, s(x)=d(x)`；实践定义：当且仅当 `∀x∈T, s(x)=d(x)`。数据集上的 rate 为通过测试的函数数/被评函数总数（分母公式未在文中显式写出，根据 rate 表格推理）。 |
| 统计单位 | 函数/样本，按优化级别和数据集汇总。 |
| 分数含义 | 越高表示功能行为恢复越多；100% 仅表示所有给定测试通过，不能外推全输入空间。 |
| 实际问题 | 解决“源码看起来像但功能是否正确”的核心逆向痛点。 |
| 实际场景 | 需要可运行、行为保持的漏洞分析、迁移、验证和自动修复；适用于有可靠测试输入的函数，真实大型项目适用边界受测试获取困难限制。 |
| 不能说明什么 | 不能证明未覆盖输入上的等价、可读性、变量名/类型恢复或源码结构忠实。 |
| 主要影响因素 | 测试集覆盖与质量、编译优化级别、输出编译成功率、模型生成预算/解码、参考测试翻译正确性、数据泄漏。 |

### Metric 2：R2I（Relative Readability Index）

| 项目 | 内容 |
|---|---|
| 指标名称 | Relative Readability Index (R2I) |
| 衡量内容 | 相对于参考/比较集合的 C 解编代码可读性，基于 AST 特征加权得到归一化分数。 |
| 计算公式 | 论文未给出具体特征、权重和归一化公式；明确说明先构造 AST、提取预定义特征，再按 feature weights 计算 0–1 的 normalized score（表中以约 0–100 展示）。 |
| 统计单位 | 函数输出，按数据集、优化级别和模型平均。 |
| 分数含义 | 越高通常表示更易读；不等同于功能正确。 |
| 实际问题 | 衡量传统反编译器输出难读、结构不清的问题。 |
| 实际场景 | 人工审计、漏洞研究和理解代码时比较不同解编器的可读性；仅明确适用于 C。 |
| 不能说明什么 | 不能证明语义正确、能编译、通过测试或真实变量名恢复；跨语言适用性未报告。 |
| 主要影响因素 | AST 特征设计、权重和归一化、C 语法/编译器风格、参考集合、优化级别；具体权重未报告。 |

### Metric 3：Edit Similarity

| 项目 | 内容 |
|---|---|
| 指标名称 | Edit Similarity |
| 衡量内容 | 生成源码与参考源码的文本/编辑接近程度。 |
| 计算公式 | 基于 Levenshtein distance，衡量把生成代码变为参考代码所需的最少插入、删除、替换；论文未明确给出由 distance 转为表中 similarity 百分数的归一化公式。 |
| 统计单位 | 函数，按测试集和优化级别汇总。 |
| 分数含义 | 分数越高表示文本越接近参考实现；通常越低编辑距离越好。 |
| 实际问题 | 解决输出与已知原源码差异过大的可比性问题。 |
| 实际场景 | 有可靠参考源码、需要比较生成结果接近程度的离线评测；不适合无参考源码的真实黑盒逆向。 |
| 不能说明什么 | 文本相似不保证功能等价；变量重命名、格式差异或等价重构会影响分数。 |
| 主要影响因素 | 参考代码风格、token/字符化方式、归一化公式（未报告）、格式化、命名差异和优化级别。 |

### Metric 4：Embedding Similarity（CodeSage）

| 项目 | 内容 |
|---|---|
| 指标名称 | CodeSage Embedding Similarity |
| 衡量内容 | 生成函数与参考函数在 CodeSage 向量空间中的语义表示接近程度。 |
| 计算公式 | 分别编码为多维向量，计算 cosine similarity；具体模型版本、池化和预处理未报告。 |
| 统计单位 | 函数，按数据集/优化级别平均。 |
| 分数含义 | 越高表示表示空间中的语义/文本接近度越高。 |
| 实际问题 | 缓解纯字符串编辑指标无法识别语义等价改写的问题。 |
| 实际场景 | 大规模自动排序、模型间语义相似度比较；依赖嵌入模型域适配。 |
| 不能说明什么 | 不能证明可执行等价、编译成功或安全语义正确；嵌入模型偏差可能掩盖关键控制流错误。 |
| 主要影响因素 | CodeSage 训练分布和版本、代码预处理、向量归一化/余弦实现、参考代码风格和优化级别。 |

### Metric 5：CodeBLEU

| 项目 | 内容 |
|---|---|
| 指标名称 | CodeBLEU |
| 衡量内容 | 代码文本、加权 n-gram、语法 AST 子树和语义数据流四个方面的综合匹配。 |
| 计算公式 | 论文未给出权重公式；明确列出四个组成部分：BLEU、weighted n-gram match、AST subtree overlap、data-flow graph comparison。 |
| 统计单位 | 函数，按数据集和优化级别平均。 |
| 分数含义 | 越高表示综合代码匹配更强；论文也提醒该指标存在争议。 |
| 实际问题 | 在词面相似之外纳入语法与数据流结构，评价代码生成质量。 |
| 实际场景 | 有参考源码的批量代码/解编器比较；需要谨慎解释，不能替代执行测试。 |
| 不能说明什么 | 不能单独证明功能全等、可读性或安全性；对等价但不同写法仍可能低分。 |
| 主要影响因素 | n-gram 分词、AST/数据流解析、四部分权重、语言支持、参考实现风格和工具版本。 |

### Metric 6：GPT-Judge 三维评分

| 项目 | 内容 |
|---|---|
| 指标名称 | GPT-Judge：variable naming、control flow、type recovery |
| 衡量内容 | GPT-4 作为专家评审，对变量名恢复、复杂控制流重建、类型推断分别打 1（很差）到 100（优秀）分，并给理由。 |
| 计算公式 | 每个维度由评审模型直接给整数分；表格对样本/优化级别求平均。人工评审规则之外的聚合细节未报告。 |
| 统计单位 | 解编函数，按维度、数据集和优化级别平均。 |
| 分数含义 | 越高表示评审模型认为相应语义/可读性维度越好。 |
| 实际问题 | 覆盖自动字符串指标难以表达的命名语义、控制流清晰度和类型恢复质量。 |
| 实际场景 | 大规模近似人工评估、快速比较解编器；适用边界受 GPT 评审稳定性限制。 |
| 不能说明什么 | 不是客观真值，不能证明功能执行正确；存在评审模型偏差、提示敏感性和不可重复性。 |
| 主要影响因素 | GPT 版本、提示词、参考源码和生成源码呈现方式、评分校准、随机性及平均方法。 |

### Metric 7：Recall@1（二进制-源码检索）

| 项目 | 内容 |
|---|---|
| 指标名称 | Recall@1 |
| 衡量内容 | 二进制 query 的真实源码匹配是否位于检索结果前 1 名。 |
| 计算公式 | `Recall@k = 命中真实匹配的 query 数 / query 总数`；论文明确这样定义，报告 k=1。 |
| 统计单位 | Query（二进制函数），按 O0–O3 汇总。 |
| 分数含义 | 越高表示二进制到源码定位能力越强。 |
| 实际问题 | 支持第三方库识别、软件成分分析和二进制-源码匹配。 |
| 实际场景 | 从已知源码数据库查找发布二进制函数；不适用于数据库没有真实对应项的开放世界检索。 |
| 不能说明什么 | 只反映 top-1 命中，不说明候选排序的其他位置、代码功能正确或跨项目泛化。 |
| 主要影响因素 | 参考库规模（论文实验 17K 源码函数）、query 数（66K）、嵌入模型、余弦检索、优化级别和真实匹配标注。 |

### Metric 8：Cyclomatic Complexity（圈复杂度）

| 项目 | 内容 |
|---|---|
| 指标名称 | Cyclomatic Complexity |
| 衡量内容 | 程序中线性独立路径数量，用于量化控制流复杂度。 |
| 计算公式 | 论文未给出具体图论公式；仅定义为 “number of linearly independent paths through a program”。 |
| 统计单位 | 程序/函数，论文以数据子集平均值和分布比较。 |
| 分数含义 | 越高表示控制流路径更复杂；不是解编正确率。 |
| 实际问题 | 检查训练/评测数据是否具有真实世界代码复杂度，而非过于简单。 |
| 实际场景 | 数据集质量与分布审计；不用于直接评判某个解编器输出。 |
| 不能说明什么 | 不能说明功能恢复、可读性或漏洞风险；路径数量不等于语义难度。 |
| 主要影响因素 | 代码控制流结构、条件/循环数量、解析工具和统计粒度；具体实现未报告。 |

### Metric 9：Halstead Difficulty

| 项目 | 内容 |
|---|---|
| 指标名称 | Halstead Difficulty |
| 衡量内容 | 基于操作符/操作数计数估计代码开发与维护难度；论文采用 Difficulty 子指标。 |
| 计算公式 | 论文给出定义为“unique operators / total operators”；原文如此表述，完整 Halstead 变体及是否含操作数的实现细节未报告。 |
| 统计单位 | 程序/函数，报告数据集平均值与分布。 |
| 分数含义 | 越高通常表示操作符层面的实现难度更高；图中为便于可视化按 10 倍归一化展示。 |
| 实际问题 | 识别合成/可执行子集是否比一般 GitHub 代码简单，从而评估训练数据真实感。 |
| 实际场景 | 数据集复杂度对比与筛选；不是解编器性能指标。 |
| 不能说明什么 | 不能证明源码可读性、功能正确性或逆向难度的全部方面。 |
| 主要影响因素 | 操作符词表、解析/计数规则、语言和预处理、统计粒度；具体实现未报告。 |

## 三、文献内容总结索引

| 模板项 | 页码/位置 | 关键原文或说明 |
|---|---|---|
| 工作名称 | p.1，标题、摘要 | “Decompile-Bench: Million-Scale Binary-Source Function Pairs for Real-World Binary Decompilation”；“arXiv:2505.12668v2…19 Oct 2025”。 |
| 研究动机 | pp.1–3，引言 | “there still lacks a comprehensive benchmark…”；DWARF 缺失、优化和 inlining “obscure the correspondence”。 |
| 评测对象 | p.6 §4.1 Dataset | “HumanEval and MBPP…compile…at four optimization levels (-O0—O3)”；“121 GitHub repositories…about 60K…functions”；“include ProRec”。 |
| 被测能力 | p.1 引言；p.6 §4.2 | “convert low-level binaries into human-readable source code”；三类指标分别对应 functionality/readability/text similarity。 |
| 输入与工具 | p.7 §5.1–5.2 | “Ubuntu 20.04…Clang-19…C++17”；“vllm…max…512…greedy decoding”；基线列表见 §5.2。完整预处理：未报告。 |
| 目标任务 | p.6 §4.1；p.7 §5.1 | “decompile it back to source, and run the original tests”；“generation (decompilation) process”。 |
| 输出结果 | p.6 §4.1；p.20–21 Appendix C | 解编源码；GPT prompt 要求 JSON 三字段分数和 rationale。 |
| 正确标准 | p.6 §4.2 | “∀x ∈ X, s(x) = d(x)”；实践中 “Given the unit tests T”。 |
| 检查方式 | p.6 §4.1–4.2；pp.20–21 | 原测试执行；AST 特征；Levenshtein；CodeSage cosine；CodeBLEU 四部分；GPT prompt。 |
| 失败原因 | p.8 §5.3.1；p.22 Appendix G | “performance degrades sharply…O2/O3”；真实项目重执行障碍；约 25% “Compiler-Induced Scope Mismatches”。逐样本失败：未报告。 |
| 评测局限 | pp.22–23 Appendix G；p.21 Appendix C | “Re-executability…significant challenge”；数据泄漏、人工构建成本；其他指标只评估孤立元素等。 |

## 四、Metric 记录索引

| Metric | 页码/位置 | 关键原文或说明 |
|---|---|---|
| Re-Executability | p.6 §4.2 | “functionality matches…”；“∀x ∈ X, s(x)=d(x)”；“In practice…finite test set”。rate 的显式分子/分母未报告，根据表 1 的 rate 汇总推理。 |
| R2I | p.6 §4.2；p.8 Table 2 | “normalized score between 0 and 1”；“construct an AST…extract pre-defined features…feature weights”。具体公式/权重未报告。 |
| Edit Similarity | p.6 §4.2；p.8 Table 3 | “Based on Levenshtein Distance”；“minimum number of insertions, deletions, or substitutions”。表中 similarity 的归一化未报告。 |
| Embedding Similarity | p.20 Appendix C | “embed…using…CodeSage…and compute cosine similarity”。模型细节未报告。 |
| CodeBLEU | p.20 Appendix C | “combines four evaluation aspects, i.e., BLEU score, weighted n-gram match, syntactic match via…AST…semantic match via data-flow graph”。权重未报告。 |
| GPT-Judge | p.20–21 Appendix C，Figure 7/Table 9 | “scale of 1 (poor) to 100 (excellent)”；三项标准和 JSON 输出格式在 Figure 7。聚合/一致性流程未报告。 |
| Recall@1 | p.21–22 Appendix D，Table 10 | “fraction of queries whose true match appears within the top k retrieved results”；17K reference、66K query；报告 “27.0% recall@1”。 |
| Cyclomatic Complexity | p.19 Appendix A.2–B，Figure 5–6 | “number of linearly independent paths through a program”；用于比较 ExeBench、GitHub、raw/clean 数据复杂度。 |
| Halstead Difficulty | p.19 Appendix A.2–B，Figure 5–6 | “ratio of the number of unique operators to the total number of operators”；图注称 “normalized by factor of 10 for better visualization”。 |

> 注：论文明确提到 symbolic execution、CodeAlign、D-Helix、human judgments、exact match/variable similarity 等其他指标未纳入本研究（p.21 Appendix C），故不虚构其计算流程。
