# CREBench 文献与 Metric 总结

## 文献内容总结

| 项目 | 内容 |
|---|---|
| 工作名称 | **CREBench: Evaluating Large Language Models in Cryptographic Binary Reverse Engineering**，COLM 2026。链接：https://arxiv.org/abs/2604.03750 |
| 研究动机 | 密码程序逆向对漏洞发现、恶意软件分析等很重要，但高度依赖人工专业知识；现有 LLM 逆向评测缺少对“自主完成密码二进制逆向”的系统、端到端评估。 |
| 评测对象 | 432 个 CTF 风格密码二进制逆向挑战：48 种标准加密算法 × 3 种不安全密钥使用方式 × 3 个二进制复杂度等级。每个样本包括可执行二进制及 Ghidra 反编译伪代码。 |
| 被测能力 | 识别密码算法；提取密钥/IV；静态与动态二进制分析；理解密钥生成、加密包装层和参数处理；以 Python 重建行为；解密并恢复正确输入。还隐含考察长程推理、工具调用与多步规划。 |
| 输入与工具 | Agent 可见挑战二进制 `challenge`、反编译代码 `decompile`，在 Docker 沙箱中工作。可使用命令行、GDB、angr、Ghidra、radare2、signsrch、Python、Web 搜索等；环境为 Ubuntu 22.04。每题最多 30 个 agent-环境交互回合（提交工具调用不计入），累计 token 上限 600K。 |
| 目标任务 | 四项独立、但具有递进关系的任务：T1 算法识别；T2 密钥（及适用时 IV）提取；T3 重实现完整 wrapper 层加密行为；T4 恢复能使 checker 接受的明文 flag。 |
| 输出结果 | `submit_algorithm` 提交算法族/名称；`submit_key` 或 `submit_key_iv` 提交十六进制密钥/IV；`submit_code` 提交 Python 文件；`submit_flag` 提交 flag。 |
| 正确标准 | T1：规范算法名或别名精确匹配为完全正确，家族级但不完整的答案部分正确。T2：要求的 key 或 key+IV 字段正确。T3：Python 程序对隐藏输入复现二进制完整 wrapper 加密输出。T4：提交的明文 flag 正确，即经程序加密后与目标密文精确相等。 |
| 检查方式 | T1、T2 将提交值按参考答案规则归一化并比对；T3 在 5 个隐藏测试向量上执行提交程序，按通过数评分；T4 环境直接返回 flag 是否正确。每题独立运行 3 次，取三次中最高总分（pass@3）。 |
| 失败原因 | 论文报告三类：算法识别的“原型偏见”（把陌生密码算法过度归为 AES、DES 等熟悉原型）；反复使用 GDB 导致调试循环、无法推进到提交；少量安全拒答（GPT-5.4 的 1,041 次尝试中 9 次）。 |
| 评测局限 | 未覆盖 Tigress、OLLVM 等专业混淆框架；因此不充分评测强混淆鲁棒性。样本聚焦标准密码算法、三类不安全密钥设计和 CTF serial-checker 形式，不能代表所有真实二进制、协议、恶意软件或非密码逆向任务。后一句为根据基准构造范围推理得到。专有模型 API 演化、快照、限流和 token 计费也会影响可复现性。 |

## Metric 记录

### Metric 1：T1 算法识别得分

| 项目 | 内容 |
|---|---|
| 指标名称 | Algorithm Identification Score（T1） |
| 衡量内容 | Agent 对二进制实际密码算法的识别正确性。 |
| 计算公式 | 精确规范名/预定义别名：25；同一算法家族但不完整：15；其他：0。 |
| 统计单位 | 单个 challenge 的一次尝试；pass@3 时取三次尝试中可达到的最高总分。 |
| 分数含义 | 高分表示能细分并正确识别算法；15 分仅表示粗粒度或不完整识别。 |
| 实际问题 | 解决从 stripped/优化/常量混淆二进制中辨别具体密码算法的困难。 |
| 实际场景 | 密码库审计、软件破解分析、恶意样本中密码实现识别；适用于论文列出的标准算法集合。 |
| 不能说明什么 | 不能证明模型理解完整 wrapper、能恢复 key/IV、能重实现程序或能恢复 flag。 |
| 主要影响因素 | 算法集合与别名规则、O0/O3/Const-XOR 难度、签名常量可见性、评分归一化规则。 |

### Metric 2：T2 密钥（IV）提取得分

| 项目 | 内容 |
|---|---|
| 指标名称 | Key (IV) Extraction Score（T2） |
| 衡量内容 | 是否恢复程序实际使用的加密密钥，及要求时的 IV。 |
| 计算公式 | `25 × 正确恢复字段数 / 要求字段数`；仅 key 样本为 0/25，key+IV 样本为 0/12/25。 |
| 统计单位 | 单个 challenge 的一次尝试。 |
| 分数含义 | 高分表示所要求的密码参数均准确；12 分表示 key 或 IV 仅一项正确。 |
| 实际问题 | 解决硬编码、分片重组、弱 PRNG 派生或运行时恢复的密钥材料提取问题。 |
| 实际场景 | 固件、客户端和恶意程序中嵌入式密钥审计；仅覆盖三种预设的不安全密钥模式。 |
| 不能说明什么 | 不能证明 key 字节序、key/IV 的角色理解或提交格式与实际解密链路完全一致；论文指出这种刚性判分会降低 T2 与 T4 的相关性。 |
| 主要影响因素 | 密钥使用模式、优化和数据流可读性、内存检查质量、字节序、key/IV 混淆、参考值和格式判定。 |

### Metric 3：T3 Wrapper 级代码重实现得分

| 项目 | 内容 |
|---|---|
| 指标名称 | Wrapper-level Code Reimplementation Score（T3） |
| 衡量内容 | 提交的 Python 程序是否复现二进制暴露出的完整加密行为，而非仅复现 cipher core。 |
| 计算公式 | `5 × 通过的隐藏测试向量数`，取值为 0、5、10、15、20、25。 |
| 统计单位 | 单个 challenge 的一次提交代码，使用 5 个隐藏测试向量。 |
| 分数含义 | 高分表示对输入处理、密钥/IV 处理及 wrapper 行为的泛化复现更完整。 |
| 实际问题 | 解决“识别出算法但未还原真实程序行为”的逆向痛点。 |
| 实际场景 | 需要兼容原二进制输入输出的迁移、行为建模、补丁验证和密码例程替换。 |
| 不能说明什么 | 仅由 5 个隐藏向量验证，不能证明对所有输入、异常路径、所有平台行为或源代码语义完全等价。 |
| 主要影响因素 | 隐藏向量覆盖度、wrapper 的复杂度、算法/模式、填充和编码、key/IV 处理、提交程序运行环境。 |

### Metric 4：T4 Flag Recovery Score

| 项目 | 内容 |
|---|---|
| 指标名称 | Flag Recovery Score（T4） |
| 衡量内容 | 是否端到端恢复使 checker 接受的正确明文输入。 |
| 计算公式 | 正确 flag 为 25；错误 flag 为 0。 |
| 统计单位 | 单个 challenge 的一次尝试。 |
| 分数含义 | 25 表示该实例的端到端目标完成；0 表示未完成。 |
| 实际问题 | 解决单独识别算法或提取参数却无法完成实际逆向目标的问题。 |
| 实际场景 | CTF serial checker、受保护程序输入恢复；适用边界是“目标密文精确匹配”的任务。 |
| 不能说明什么 | 不保证模型提交了正确的算法、key/IV 或通用 wrapper 重实现；模型可能通过其他足以解出该实例的路径成功。 |
| 主要影响因素 | flag 随机实例、目标密文、所有前序推理质量、运行与工具预算、提交机会。 |

### Metric 5：单题总分

| 项目 | 内容 |
|---|---|
| 指标名称 | Per-instance Total Score |
| 衡量内容 | 一个 challenge 上四阶段逆向能力的综合完成程度。 |
| 计算公式 | `S = T1 + T2 + T3 + T4`，每项最多 25，总分 0–100。 |
| 统计单位 | 单个 challenge。 |
| 分数含义 | 分数越高，表示从局部识别到端到端 flag 恢复完成得越多。 |
| 实际问题 | 避免只用 flag 成败而丢失中间逆向进展。 |
| 实际场景 | 比较模型在标准化密码二进制任务上的总体表现。 |
| 不能说明什么 | 不等同于真实世界逆向生产率、安全影响或强混淆下的能力。四项等权是否符合实际风险，论文未报告论证。 |
| 主要影响因素 | 四项任务的评分规则、25 分等权设置、挑战构成及 pass@3 选择规则。 |

### Metric 6：Average Pass@3 Score

| 项目 | 内容 |
|---|---|
| 指标名称 | Average Pass@3 Score |
| 衡量内容 | 模型在全基准上、每题至多三次独立尝试后的平均最佳总分。 |
| 计算公式 | 根据“每题取三次中的最高分，再报告平均分”推理：`mean_c(max(S_c,1, S_c,2, S_c,3))`，范围 0–100。 |
| 统计单位 | challenge；先在每题的 3 次尝试内取最大值，再跨 challenge 平均。 |
| 分数含义 | 高分表示模型在有限重试下能完成更多中间阶段和端到端阶段。 |
| 实际问题 | 降低单次采样随机性，衡量可重复尝试下的综合逆向表现。 |
| 实际场景 | 同一预算、同一基准设置下比较不同 LLM 或 agent 框架。 |
| 不能说明什么 | 不能表示单次成功率、无限预算表现、真实操作成功率，亦不能单独证明完整端到端完成。 |
| 主要影响因素 | 三次尝试数、采样设置、30 回合与 600K token 上限、挑战难度分布、四项等权及模型版本。 |

### Metric 7：Flag Recovery Rate

| 项目 | 内容 |
|---|---|
| 指标名称 | Flag Recovery Rate |
| 衡量内容 | 成功恢复 flag 的 challenge 比例。 |
| 计算公式 | 根据 T4 为二值、论文报告 pass@3 flag recovery rate 推理：`成功恢复 flag 的挑战数 / 挑战总数 × 100%`；主实验采用每题三次尝试中的成功结果。 |
| 统计单位 | challenge。 |
| 分数含义 | 越高表示越常完成最终 CTF 式端到端目标。 |
| 实际问题 | 给出比部分任务总分更直接的“最终目标是否完成”指标。 |
| 实际场景 | 端到端 CTF/serial-checker 密码逆向；论文在框架比较中也以此比较不能参加四阶段评分的 D-CIPHER。 |
| 不能说明什么 | 不反映算法识别、参数提交或通用重实现质量，也不能衡量未覆盖的真实应用场景。 |
| 主要影响因素 | pass@k 设置、flag 判定器、题目难度和随机 flag、资源预算、模型/agent 框架。 |

### Metric 8：Pass@3 Perfect Rate

| 项目 | 内容 |
|---|---|
| 指标名称 | Pass@3 Perfect Rate |
| 衡量内容 | 三次尝试内获得 100/100、即完成全部四项任务的 challenge 比例。 |
| 计算公式 | `获得满分 100 的挑战数 / 挑战总数 × 100%`，每题允许三次独立尝试。 |
| 统计单位 | challenge。 |
| 分数含义 | 高分表示模型能可靠连接算法识别、参数恢复、代码重建与 flag 恢复全链路。 |
| 实际问题 | 防止平均总分被仅完成前序子任务的部分成功抬高。 |
| 实际场景 | 需要完整、可审计的密码二进制逆向链路时的模型比较。 |
| 不能说明什么 | 不能反映部分进展；也不能证明对专业混淆或论文外算法、程序类型具有泛化能力。 |
| 主要影响因素 | 四项均必须成功的严格门槛、T1 别名规则、T2 格式刚性、T3 的 5 个测试向量、三次尝试和资源预算。 |

### Metric 9：Phi Correlation Coefficient

| 项目 | 内容 |
|---|---|
| 指标名称 | Phi Correlation Coefficient（子任务相关性） |
| 衡量内容 | 两个二值化子任务成功/失败之间的总体统计关联。 |
| 计算公式 | `φ=(N11N00−N10N01)/sqrt((N11+N10)(N01+N00)(N11+N01)(N10+N00))`。 |
| 统计单位 | 成对任务上的样本/尝试结果；论文未明确说明该附录分析是否先按 pass@3 聚合。 |
| 分数含义 | 越高表示两任务的成功和失败越共同出现；接近 0 表示线性二元关联弱。 |
| 实际问题 | 判断哪一逆向阶段与另一阶段共同构成能力瓶颈。 |
| 实际场景 | 设计分阶段 RE benchmark、诊断代码重实现与 flag 恢复等任务依赖。 |
| 不能说明什么 | 不能证明因果或前置关系，也不能排除评分规则造成的低相关。 |
| 主要影响因素 | 任务二值化阈值、样本分布、评分刚性、任务定义、模型和尝试聚合方式。 |

### Metric 10：Conditional Success Rate

| 项目 | 内容 |
|---|---|
| 指标名称 | Conditional Success Rate，`P̂(Tj=1 | Ti=1)` |
| 衡量内容 | 在源任务 `Ti` 成功的样本中，目标任务 `Tj` 也成功的比例。 |
| 计算公式 | `N11 / (N11 + N10)`；其中分母表示 `Ti=1` 的样本数。公式根据论文定义和 `Nab` 记号推理得到。 |
| 统计单位 | 成对任务上的样本/尝试结果；聚合层级未报告。 |
| 分数含义 | 越高表示解出 `Ti` 时通常也能解出 `Tj`。 |
| 实际问题 | 识别逆向流水线中阶段成功能否伴随下游成功。 |
| 实际场景 | 分析是否应优先改进算法识别、参数恢复或 wrapper 重实现。 |
| 不能说明什么 | 不证明 `Ti` 是 `Tj` 的因果前提，也不衡量 `Ti` 失败时 `Tj` 的表现。 |
| 主要影响因素 | 各任务成功率、二值化标准、样本难度分布、提交格式与评分规则。 |

### Metric 11：Conditional Success Rate Difference

| 项目 | 内容 |
|---|---|
| 指标名称 | Conditional Success Rate Difference，`ΔP̂(Tj | Ti)` |
| 衡量内容 | `Ti` 成功相对失败时，`Tj` 成功率的提升幅度。 |
| 计算公式 | `P̂(Tj=1 | Ti=1) − P̂(Tj=1 | Ti=0)`。 |
| 统计单位 | 成对任务上的样本/尝试结果；聚合层级未报告。 |
| 分数含义 | 值越大，论文将其解释为 `Ti` 对 `Tj` 越强的前置条件。 |
| 实际问题 | 在相关性之外区分“完成某阶段后，下游任务成功率提升多少”。 |
| 实际场景 | 评测分层设计与能力依赖诊断。 |
| 不能说明什么 | 不能证明真正的因果前置关系；混杂的题目难度和评分规则也可产生大差值。 |
| 主要影响因素 | 任务成功率、难度混杂、样本量、二值化与评分规则。 |

## 文献内容总结索引

| 模板项 | 页码、位置 | 关键原文或依据 |
|---|---|---|
| 工作名称 | PDF/论文第 1 页，标题与摘要 | “CREBench: Evaluating Large Language Models in Cryptographic Binary Reverse Engineering”；“Published as a conference paper at COLM 2026”。 |
| 研究动机 | 第 1–2 页，Introduction | “RE remains labor-intensive and requires substantial expertise”；“fully autonomous capabilities in cryptographic RE remains largely absent”。 |
| 评测对象 | 第 1 页摘要；第 2、5 页 | “432 challenges built from 48 standard cryptographic algorithms, 3 insecure crypto key usage scenarios, and 3 difficulty levels”；“executable binary and its decompiled pseudocode”。 |
| 被测能力 | 第 2、6、21 页 | “algorithm identification… key (IV) extraction… wrapper-level code reimplementation… flag recovery”；附录称其需要 “long-horizon reasoning, strong coding ability, reliable tool use”。 |
| 输入与工具 | 第 2、6–7、51–54 页 | “provided with both the executable binary and its decompiled pseudocode”；“sandboxed execution environment”；第 51–54 页列出 GDB、angr、Ghidra、radare2、signsrch、Python、Web 搜索等。 |
| 目标任务 | 第 6 页，§3.3 | “each challenge is evaluated through four sub-tasks”；随后逐一定义 T1–T4。 |
| 输出结果 | 第 50–54 页，Appendix F | “submit_algorithm”，“submit_key / submit_key_iv”，“submit_code”，“submit_flag”。 |
| 正确标准 | 第 4、6、21、50–54 页 | 第 4 页：“If the output and the target ciphertext match exactly, the input will be accepted”；第 21 页列出 T1–T4 的明确评分条件。 |
| 检查方式 | 第 6、21、53–54 页 | “each LLM is given three independent attempts… highest score”；“T3 is evaluated on 5 hidden test vectors”；第 53–54 页描述各 `submit_*` evaluator。 |
| 失败原因 | 第 9 页，§4.4.3 | “Prototype bias in algorithm identification”；“Heavy GDB use as a marker of stalled trajectories”；“Safety refusal on benchmark instances”。 |
| 评测局限 | 第 9–10 页，Limitations；第 10 页 Reproducibility | “we do not cover professional obfuscation frameworks such as Tigress and O-LLVM”；专有模型不可完全复现。真实程序、协议和非密码任务的覆盖不足是根据第 5 页的构造范围推理得到。 |

## Metric 记录索引

| Metric | 页码、位置 | 关键原文或依据 |
|---|---|---|
| T1 算法识别得分 | 第 21 页，Appendix C.2 | “three-level scheme: 25, 15, or 0 points. Exact match… 25… Family-level but incomplete… 15”。 |
| T2 密钥（IV）提取得分 | 第 6、21 页 | “scored by the proportion of required fields”；“key-only… 0 or 25… key+IV… 0, 12, or 25”。 |
| T3 代码重实现得分 | 第 6、21、50–51 页 | “evaluated on 5 hidden test vectors”；“full wrapper-level behavior”；“0, 5, 10, 15, 20, or 25”。 |
| T4 Flag Recovery Score | 第 6、21、53 页 | “Task 4 is binary: a correct flag receives 25 points”；“submit_flag… Returns whether the flag is correct”。 |
| 单题总分 | 第 6、21 页 | “each sub-task worth 25 points and a maximum of 100 points per challenge”；“Each benchmark instance is scored out of 100 points”。 |
| Average Pass@3 Score | 第 6 页，§4.1 | “three independent attempts per challenge, and the highest score across the three attempts is taken as the final score”。跨题平均的公式为根据该定义及图表 “Average Pass@3 Score” 推理得到。 |
| Flag Recovery Rate | 第 1 页摘要；第 7–8 页 Table 2；第 22 页 | “recovers the flag in 59% of challenges under pass@3”；“Flag Recovery Rate”。其计数公式是根据二值 T4 和该比例表述推理得到。 |
| Pass@3 Perfect Rate | 第 7、22 页；第 23 页 Figure 5 | “fraction of challenges solved with a full score of 100/100”；“proportion… obtains the full score of 100/100… within three attempts”。 |
| Phi Correlation | 第 8 页 §4.4.2；第 27 页 Appendix D.7 | “using three metrics”；“measures the overall statistical association”；第 27 页给出完整公式。 |
| Conditional Success Rate | 第 27 页，Appendix D.7 | “measures the proportion of samples in which Tj succeeds among those where Ti also succeeds”。显式分式由该定义与 `Nab` 定义推理得到。 |
| Conditional Success Rate Difference | 第 27 页，Appendix D.7 | “measures how much the success rate of Tj changes depending on whether Ti is solved”；并给出 `ΔP̂(Tj│Ti)=...`。 |
