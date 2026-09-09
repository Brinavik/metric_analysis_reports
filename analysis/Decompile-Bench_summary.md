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