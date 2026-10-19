# EdgeAdapt Benchmark 选型报告

Oct 18, 2026 · @Chang Yin

## 结论

推荐用 **τ²-bench + LifelongAgentBench** 拼成主任务流，**BFCL + HammerBench** 当 retention 探针，**FreeInference Trace** 提供任务到达节奏。没有任何单个数据集能同时测出 Skill、LoRA、Defer 三条路径的价值，所以必须组合。

| 优先级 | 数据集 | 一句话理由 |
| --- | --- | --- |
| 首选 | τ²-bench（telecom + retail） | 原子子任务组合成 2,285 个任务，长政策文档让 Skill / LoRA / Defer 的取舍最真实 |
| 首选 | LifelongAgentBench（DB + OS） | 自带技能标注、按顺序执行、小模型能拿分，最适合测 Skill 图 |
| 次选 | ToolSandbox | 手机设置 / 短信 / 联系人的状态依赖，端侧场景最像 |
| 辅助 | BFCL、HammerBench | 监控通用工具调用能力是否退化 |
| 辅助 | FreeInference Trace | 真实的会话节奏和空闲窗口，让 Defer 有意义 |
| 辅助 | ALFWorld | 便宜的消融环境 |
| 备选 / 加分 | AppWorld、AndroidWorld | 结构很好但对小模型太难，等主结果稳定再试 |
| 不推荐 | WebArena、Mobile-Eval-E、XSkill / MCE / MobiZO / P&M 的数据集 | 太重、靠人工评审，或不是 agent 任务 |

**开工前必须先做一件事**：用 Gemma 4 E2B / E4B 在上述环境上测零样本基线。τ²-bench 的论文没有任何开源小模型结果，如果接近 0 分，LoRA 侧就要换成 ToolSandbox + HammerBench。

## EdgeAdapt 需要 benchmark 证明什么

EdgeAdapt 的核心主张是：在相同资源预算下，联合决定何时适应、如何适应（Skill / LoRA / Defer），比单独做 Skill learning 或 PEFT 的长期学习效率更高。所以 benchmark 要能同时测出三条路径各自的价值，以及 Controller 选得对不对。下面 8 条是本文给每个候选打分的维度。

| # | 维度 | 为什么 EdgeAdapt 需要它 | 缺了会怎样 |
| --- | --- | --- | --- |
| R1 | 可执行、自动验证（按环境状态判对错） | Skill 是可执行的；定向 regression test 要能自动跑；端侧扛不住 LLM 判分 | 成功率有噪声，regression test 无法落地 |
| R2 | 重复与组合结构 | Skill 图的复用、依赖、组合要有意义 | Skill 复用率低，Tier 1 看起来没用 |
| R3 | 能构造非平稳任务流和旧任务回放 | 测分布漂移下的适应和 capability retention | 只能测静态精度，测不出遗忘 |
| R4 | 同时包含"能写成 Skill"和"难写成 Skill"的行为 | Controller 要真的有选择，Tier 2 才有存在意义 | Controller 永远选同一条路 |
| R5 | 小模型（Gemma 4 E2B/E4B 级别）能拿到非零分 | 学习曲线要有起伏才能比较 | 全是 0 分，看不出任何方法的差别 |
| R6 | 环境可重置、确定性、不依赖实时网络 | 反事实评估：同一个决策点把三个选项都跑一遍，算 Controller 的 regret | 无法证明 Controller 做了好决策 |
| R7 | 有 train / test 划分或足够多同分布实例 | Tier 2 需要足够多已验证的成功轨迹才能训 LoRA | LoRA 没数据可训 |
| R8 | 端侧真实感（手机 App、个人助理类任务、上下文长度适中） | 支撑"端侧 agent"这个场景设定 | 审稿人质疑场景不真实 |

其中 R1、R5、R6 是硬门槛：不满足的数据集只能当辅助，不能当主 benchmark。

## 候选数据集逐个分析

共评估了 15 个候选，外加前面四篇论文的数据集，分成 6 类。结论是：没有任何一个能单独覆盖 R1–R8，最接近的是 τ²-bench 和 LifelongAgentBench，而且两者刚好互补。

### A. 持续学习 / 流式协议类

**[LifelongAgentBench](https://arxiv.org/pdf/2505.11942)**：数据库、操作系统、知识图谱 3 个环境，共 1,396 个任务，按严格顺序执行。

- 优点：每个任务都标注了用到的技能（数据库 22 种 SQL 技能，操作系统 29 个 Bash 命令），可以直接当 Skill 图的 ground truth；可执行、可重置；论文测过 Llama-3.1-8B，数据库环境能到 0.6 以上。
- 缺点：几乎所有行为都能写成显式 Skill，很难触发 LoRA；场景是服务器运维，不像端侧助理。
- 定位：**Skill 侧主任务流**。

**[StreamBench](https://arxiv.org/html/2406.08747v1)**（NeurIPS 2024）：把 Spider、CoSQL、BIRD、DS-1000、ToolBench、DDXPlus、HotpotQA 转成固定顺序的流，反馈只给对/错。

- 优点：二值反馈很像端侧场景（用户点赞点踩）；发布了固定顺序，容易和别人比。
- 缺点：大多是单步任务，不是多轮 agent；顺序是随机打乱的，没有真实的分布漂移。
- 定位：**借用协议 + 对比基线**。

**[Evo-Memory](https://arxiv.org/html/2511.20857v1)**（2025 年 11 月）：把 MMLU-Pro、GPQA、AIME、ToolBench 和 AgentBoard 里的 AlfWorld、BabyAI、ScienceWorld、PDDL 重构成流式任务。

- 优点：专门测"记忆类方法"，和 Tier 1 直接可比；有"序列鲁棒性"指标（换任务顺序后是否稳定）。
- 缺点：论文只说会开源，没找到确认的仓库；不涉及参数更新。
- 定位：**Tier 1 的对比基线来源**。

### B. 有状态的 API / 工具调用 agent

**[τ²-bench](https://arxiv.org/pdf/2506.07982)**（Sierra）：客服 agent 按业务政策调用 API 帮模拟用户办事。retail 115 题、airline 50 题；telecom 是"双控制"设置，用户也要在模拟手机上操作。[仓库](https://github.com/sierra-research/tau2-bench)现在还有 banking\_knowledge 领域。

- 优点一：**telecom 的任务是由 15 组原子子任务组合生成的，共 2,285 个可验证任务**。原子子任务天然就是 Skill，组合关系天然就是 Skill 图，而且数据量足够训 LoRA。
- 优点二：每题都要带一份很长的政策文档，在手机上 prefill 很贵。Skill（只注入相关片段）、LoRA（把政策学进权重）、Defer（继续带整份文档）三者的取舍非常真实，**这是所有候选里最能体现 Controller 价值的场景**。
- 优点三：按最终世界状态判分；pass^k 指标测的是稳定性，可以直接用来测 regression。
- 缺点：论文最小只测了 gpt-4.1-mini，**没有任何开源小模型的结果，Gemma 4 E2B 很可能接近 0 分**；用户模拟器用的是 gpt-4.1，有随机性和额外成本。
- 定位：**Skill vs LoRA 取舍的主任务流**（前提是 E4B 基线不为 0）。

**[AppWorld](https://arxiv.org/pdf/2407.18901)**：9 个模拟日常 App、457 个 API，750 个任务来自 250 个场景（每个场景 3 个任务）。

- 优点：用状态单元测试判分；本地单进程 + 内存 SQLite 就能跑；同一场景 3 个任务天然适合测 Skill 复用；有 train / dev / test 划分。
- 缺点：**太难**。GPT-4o 在 Test-Normal 上只有 48.8，最好的开源模型（LLaMA3）是 24.4，挑战集只有 7.0。train 只有 105 题。
- 定位：**备选**，只有 E4B 在简单难度子集上能拿分时才用。

**[ToolSandbox](https://arxiv.org/html/2408.04682v1)**（Apple）：1,032 个测试用例、34 个工具，领域是手机设置、联系人、短信等。

- 优点：**最像手机助理**；工具之间有隐式状态依赖（发短信前要先开蜂窝网络，开蜂窝前又要先关低电量模式），这正是 Skill 图里"依赖"边要学的东西；用里程碑和"雷区"判分，踩雷直接 0 分，适合测安全性退化；最好的开源 7B 模型能到 31.4。
- 缺点：用户模拟器是 GPT-4o；少数工具（如天气）要调实时网络服务，影响复现；规模不大，重复结构不如 τ² telecom。
- 定位：**端侧场景的主任务流候选 / 补充**。

**[BFCL](https://gorilla.cs.berkeley.edu/blogs/16_bfcl_v4_memory.html)**（v4）：单轮、多轮、网页搜索、记忆、格式敏感度几类，用 AST 或状态转移判分。

- 优点：小模型有大量公开分数可比；格式敏感度一类能测出 LoRA 是否让 JSON 变得不稳。
- 缺点：大多是互相独立的单次调用，没有"前面学的能用在后面"的结构。
- 定位：**通用工具调用能力的 retention 探针**。

**[HammerBench](https://arxiv.org/html/2412.16516v2)**（OPPO + 上交）：来自手机 App 功能的 1,063 个 API，单轮 6,531 条，多轮覆盖意图变化、参数变化、代词指代。

- 优点：场景就是手机助理；测了 7B–8B 小模型；指标细到参数幻觉率和缺失率，正好对应小模型的弱点。
- 缺点：对话是预先生成好的，只能静态比对，不能真的执行。
- 定位：**手机场景的 retention 探针**，和 BFCL 互补。

**[ShortcutsBench](https://arxiv.org/html/2407.00132v3)**：7,627 个真实 Apple 快捷指令、88 个 App、1,414 个 API，评估集 5,220 条。

- 优点：**动作序列是人工标注的真实快捷指令，本质上就是人写的 Skill**，可以用来评估"从轨迹里归纳出的 Skill 和人写的有多接近"。
- 缺点：只和标准序列比对，不执行；不是流式。
- 定位：**Skill 归纳质量的离线分析**（可选）。

### C. 移动端 GUI agent

**[AndroidWorld](https://arxiv.org/html/2405.14573v3)**（Google DeepMind）：20 个真实 Android App 上的 116 个任务，另有 92 个 MobileMiniWoB++ 任务。

- 优点：任务是参数化的，可以生成"数百万个不同目标"，天然有"重复但不完全相同"的结构；用 adb 检查系统状态判分，每个任务有 setup 和 teardown，可重置；GUI 定位这类难以写成 Skill 的行为正是 Tier 2 的用武之地；场景最真实。
- 缺点：**太难**，GPT-4 Turbo 最好也只有 30.6%（人类 80%），论文没有小模型结果；需要多模态或无障碍树输入；要跑模拟器（约 2GB 内存、8GB 磁盘），工程量大。
- 定位：**加分项实验**，不建议作为第一版的主 benchmark。

**[Mobile-Agent-E / Mobile-Eval-E](https://arxiv.org/html/2501.11733v1)**（NeurIPS 2025）：用 Tips（经验）和 Shortcuts（可复用操作序列，带前置条件）实现自我进化的手机助理，在真机（Galaxy A15）上跑。

- 优点：Shortcuts 的设计（带前置条件的可复用序列）和你的 Skill 图很像，是最直接的相关工作。
- 缺点：只有 25 个任务，得分靠人工评审，不能自动跑 regression；依赖实时 App 内容。
- 定位：**相关工作 / 基线设计参考**，不当 benchmark。

### D. 文本或 Web 环境

**ALFWorld / ScienceWorld**（通过 AgentBoard 接入，Evo-Memory 也用了它们）：文字版家务和科学实验环境。

- 优点：ALFWorld 只有 6 类任务模板反复出现，确定性强、跑得快、训练实例多；小模型能拿到一些分。适合大量消融实验。
- 缺点：场景和端侧助理无关；已经被用烂了，审稿人不会觉得有新意。
- 定位：**便宜的消融实验环境**。

**WebArena**（[SkillWeaver](https://arxiv.org/pdf/2504.07079) 用的环境）：5 个自建网站、812 个任务。SkillWeaver 把 Skill 写成 Playwright 函数，GPT-4o 合成的 Skill 给 GPT-4o-mini 用，相对提升最高 54.3%。

- 优点：证明了"强模型合成 Skill、弱模型使用"这条路可行，是 Tier 1 的重要先例。
- 缺点：要跑多个 Docker 网站和浏览器，很重；小模型基线低；不是端侧场景。
- 定位：**不推荐作为 benchmark**，只在相关工作里引用。

### E. 负载 trace

**[FreeInference Agentic Trace](https://huggingface.co/datasets/harvardMadsys/freeinference_agentic_trace)**（哈佛 MadSys）：2026 年 5–9 月 16 周的真实 agent 使用记录，12,002 个会话、约 119 万次 LLM 请求、约 121 万次工具调用。

- 优点：有真实的时间戳、请求间隔、token 构成、工具类别和缓存命中。可以用来驱动任务到达节奏和空闲窗口，让 Controller 学会"现在 Defer，空闲充电时再训 LoRA"；也能校准成本模型、写 motivation。
- 缺点：**只有元数据，没有对话内容，也没有会话级成功标签**；是云端 coding agent 负载，不是手机。
- 定位：**任务流的时间结构 + motivation 分析**，不能测成功率。

### F. 前面四篇论文用过的数据集

| 来源 | 数据集 | 能否当主 benchmark | 可以怎么用 |
| --- | --- | --- | --- |
| XSkill | VisualToolBench、TIR-Bench、MMSearch-Plus、MMBrowseComp、AgentVista | 否：依赖实时网页搜索，需要前沿模型 | 多模态扩展时只取纯代码处理图片的子集 |
| MCE | FiNER、USPTO-50k、Symptom2Disease、LawBench、Aegis2.0 | 否：单步分类 | 测 Controller 在"模式匹配、上下文太长"时是否选 LoRA |
| MobiZO | GLUE、SuperGLUE、WinoGrande、ARC | 否：学术分类题 | 通用能力 retention 探针；参考它的真机测量方法 |
| P&M | ImageNet-R、CIFAR100 等 | 否：视觉分类 | 借用 Acc / AAA / forgetting 的计算方式 |

## 总对比

在能当主任务流的多轮 agent 环境里，硬门槛 R1、R5、R6 全部至少达到"中"的只有 LifelongAgentBench、ToolSandbox、ALFWorld；τ²-bench 在 R5 上有风险，但它是唯一在 R2、R4、R7 上都拿"强"的。评级是我根据各论文公开信息做的判断，R5 必须用 Gemma 4 实测确认。

| 数据集 | R1 可验证 | R2 重复/组合 | R3 非平稳 | R4 Skill↔LoRA 分化 | R5 小模型可用 | R6 可重置 | R7 数据量 | R8 端侧真实感 | 定位 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| τ²-bench | 强 | 强 | 中 | 强 | 弱（待测） | 中 | 强 | 中 | 主任务流（LoRA 侧） |
| LifelongAgentBench | 强 | 强 | 强 | 弱 | 中 | 强 | 中 | 弱 | 主任务流（Skill 侧） |
| ToolSandbox | 强 | 中 | 弱 | 中 | 中 | 中 | 中 | 强 | 端侧场景补充 |
| AppWorld | 强 | 强 | 中 | 中 | 弱 | 强 | 弱 | 强 | 备选 |
| AndroidWorld | 强 | 强 | 中 | 强 | 弱 | 中 | 强 | 强 | 加分项 |
| ALFWorld / ScienceWorld | 强 | 强 | 中 | 中 | 中 | 强 | 强 | 弱 | 消融环境 |
| BFCL | 中 | 弱 | 弱 | 弱 | 强 | 强 | 强 | 中 | retention 探针 |
| HammerBench | 中 | 弱 | 弱 | 中 | 强 | 强 | 强 | 强 | retention 探针 |
| ShortcutsBench | 弱 | 中 | 弱 | 弱 | 中 | 强 | 强 | 强 | Skill 质量离线分析 |
| StreamBench | 强 | 中 | 中 | 中 | 中 | 强 | 强 | 弱 | 协议 + 基线 |
| Evo-Memory | 强 | 中 | 中 | 弱 | 中 | 强 | 强 | 弱 | Tier 1 基线 |
| Mobile-Eval-E | 弱 | 强 | 弱 | 中 | 弱 | 弱 | 弱 | 强 | 相关工作 |
| WebArena | 强 | 强 | 弱 | 中 | 弱 | 中 | 中 | 弱 | 不推荐 |
| FreeInference Trace | — | 中 | 强 | — | — | — | 强 | 中 | 负载节奏 |
| 前四篇论文的数据集 | 中 | 弱 | 弱 | 弱 | 中 | 强 | 中 | 弱 | 辅助探针 |

"—"表示不适用：FreeInference 没有任务内容和成功标签，只提供时间结构。

## 推荐方案：怎么组合

用两个互补的主环境拼成一个非平稳任务流，再加两类固定探针和一个真实的到达节奏。τ²-bench 负责"规则密集、值得内化进权重"的行为，LifelongAgentBench 负责"可执行、可组合"的技能，两者放在一起 Controller 才真的有东西可选。

### 各组件的角色

| 角色 | 数据集 | 用法 |
| --- | --- | --- |
| 主任务流（LoRA 侧） | τ²-bench telecom + retail | telecom 的 15 组原子子任务当 Skill 图的参考答案；长政策文档制造 prefill 成本压力 |
| 主任务流（Skill 侧） | LifelongAgentBench DB + OS | 技能标注当 Skill 图 ground truth；测 Skill 复用和定向 regression test |
| 端侧场景补充 | ToolSandbox | 手机设置、短信、联系人类任务；测状态依赖和"雷区"动作 |
| 通用能力 retention 探针 | BFCL 固定子集 + HammerBench 固定子集 | 每学完 N 个任务跑一次，监控 LoRA 是否破坏 JSON 格式和参数填写 |
| 任务级 retention | 主任务流里的旧任务回放 | 用 pass^k 测稳定性下降 |
| 到达节奏 | FreeInference 会话时间结构 | 决定任务何时到达、哪里有空闲窗口 |
| 消融环境 | ALFWorld | 便宜地跑超参数和组件消融 |
| 对比基线 | StreamBench 协议、Evo-Memory 方法、SkillWeaver / Mobile-Agent-E 思路 | 只做 Skill、只做 LoRA、总是 Defer 三个单路径基线 |

### 任务流怎么排

1. **阶段 A：LifelongAgentBench DB**。建立初始 Skill 图，预期 Controller 大多选 Skill。
2. **阶段 B：τ²-bench retail**。分布漂移到规则密集场景，预期在资源够的档位开始选 LoRA。
3. **阶段 C：混入 A 的旧任务（换新参数）**。测 retention 和 Skill 复用。
4. **阶段 D：τ²-bench telecom + ToolSandbox**。组合型任务，测 Skill 图的组合和依赖。
5. **阶段 E：再回到 A、B**。测 LoRA 是否破坏了已有 Skill 能力。

全程要故意混入一些只出现一次的长尾任务。否则 Defer 永远不会被证明有用。任务到达时间按 FreeInference 的会话节奏采样，并报告至少 3 种任务顺序下的结果（借用 Evo-Memory 的序列鲁棒性）。

### 设备档位当作实验变量

| 档位 | 设备 | 模型 | Tier 2 可用性 | 预期 Controller 行为 |
| --- | --- | --- | --- | --- |
| 低 | 中端 Android（6–8GB） | Gemma 4 E2B，Q4 | 不可用 | 只在 Skill 和 Defer 之间选 |
| 中 | 旗舰手机（12GB+，有 NPU） | Gemma 4 E4B，Q4 | 交给同一用户的其他设备，或零阶训练 | 偏 Skill，空闲窗口才选 LoRA |
| 高 | Jetson Orin / M 系列 Mac | Gemma 4 E4B | 可用反向传播训 LoRA | LoRA 比例明显更高 |

同一个任务流在三档下学出不同策略，是"资源感知"这个主张最直接的证据。

### 评估 Controller 本身：反事实 regret

只报最终成功率，没法证明 Controller 做了好决策。在抽样的决策点上，把 Skill / LoRA / Defer 三个选项都实际跑一遍，记录各自的后续收益、成本和退化，得到上帝视角的最优选择，再算 Controller 的 regret。这要求环境可重置：LifelongAgentBench、AppWorld、ALFWorld 满足；τ²-bench 和 ToolSandbox 要固定用户模拟器的模型和采样参数。

### 指标对应

| EdgeAdapt 指标 | 怎么算 |
| --- | --- |
| 累计任务成功率 | 任务流上的成功率曲线下面积；另报 P&M 风格的 AAA |
| 端到端学习与推理成本 | 每次请求的 prefill / decode token 和耗时、LoRA 训练时间、Skill 生成时间 |
| 能效 | 单位能耗的成功任务数（用 Perfetto 功耗轨或外接功率计） |
| capability retention | 旧任务 pass^k 变化 + BFCL / HammerBench 探针分数变化 |
| Controller 质量 | 相对反事实最优的 regret |

## 风险和执行顺序

最大的风险是小模型基线太低：τ²-bench、AppWorld、AndroidWorld 的论文都没有开源小模型的结果，必须先用 Gemma 4 实测，再决定主任务流。

### 主要风险

| 风险 | 影响 | 应对 |
| --- | --- | --- |
| Gemma 4 E2B / E4B 在 τ²-bench 上接近 0 分 | LoRA 侧主任务流不成立 | 先只用 retail 或 telecom 里只含 1–2 个原子子任务的简单任务；不行就用 ToolSandbox 或 HammerBench 代替 |
| 用户模拟器引入随机性 | 成功率有噪声，反事实评估不准 | 固定模拟器模型和 temperature；每个配置至少 3 次；模拟器放云端，成本不计入端侧 |
| τ²-bench 版本变化 | 结果不可比 | 1.0.1 版改过判分规则，锁定版本号并写进论文 |
| 任务重复次数不够训 LoRA | Tier 2 永远没数据 | 优先用 telecom 的组合生成器（2,285 个任务）；把"数据是否足够"写进 Controller 的成本模型 |
| 只报平均成功率 | 无法证明 Controller 的价值 | 反事实 regret + 三个单路径基线 |
| 发热降频 | 后半段变慢，混进"学习效果" | 记录 thermal 状态，分段冷却，固定起始温度 |
| 任务流节奏太假 | Defer 和"空闲时训练"测不出来 | 用 FreeInference 节奏，但在论文里说明它来自云端 coding agent |
| Evo-Memory 代码未确认发布 | 基线要自己复现 | 先查仓库；没有就用 StreamBench 里的记忆类基线 |

### 建议的执行顺序

1. **第 1 周：基线摸底**。在 Mac 上用 llama.cpp 跑 Gemma 4 E2B 和 E4B，测 LifelongAgentBench DB/OS、τ²-bench retail/telecom、ToolSandbox、BFCL 子集的零样本成绩。成功率低于 10% 的环境不进入主任务流。
2. **第 2 周：根据基线定主任务流**。如果 τ² 太难，LoRA 侧换成 ToolSandbox + HammerBench 的训练分布。同时用 FreeInference 做工具重复率统计，当 motivation 图。
3. **第 3 周：搭任务流和探针**。写任务流生成器（阶段、回放、长尾任务、到达时间），固定 BFCL 和 HammerBench 探针子集。
4. **第 4 周：只用 Tier 1 跑通全流程**。得到"只做 Skill"基线和第一批反事实数据。
5. **第 5 周以后：加 Tier 2 和 Controller**。LoRA 先在 Mac / Jetson 上训，推到手机上推理；再加入设备档位对比。
6. **加分项**：主结果稳定后，可以用 AndroidWorld 做一个真实 GUI 场景的小规模实验。

### 待确认的问题

- LifelongAgentBench 和 Evo-Memory 的代码与数据是否已发布、是否方便接自定义 agent。
- τ²-bench 的 Gym 接口 train / test 划分具体怎么切，是否能按原子子任务取子集。
- llama-server 的工具调用格式能否通过 LiteLLM 对接 τ²-bench。

## Sources

- [LifelongAgentBench: Evaluating LLM Agents as Lifelong Learners](https://arxiv.org/pdf/2505.11942)
- [StreamBench: Towards Benchmarking Continuous Improvement of Language Agents](https://arxiv.org/html/2406.08747v1)
- [Evo-Memory: Benchmarking LLM Agent Test-time Learning with Self-Evolving Memory](https://arxiv.org/html/2511.20857v1)
- [τ²-Bench: Evaluating Conversational Agents in a Dual-Control Environment](https://arxiv.org/pdf/2506.07982)
- [τ²-bench GitHub](https://github.com/sierra-research/tau2-bench)
- [AppWorld: A Controllable World of Apps and People for Benchmarking Interactive Coding Agents](https://arxiv.org/pdf/2407.18901)
- [ToolSandbox: A Stateful, Conversational, Interactive Evaluation Benchmark for LLM Tool Use Capabilities](https://arxiv.org/html/2408.04682v1)
- [BFCL V4 Part 2: Memory](https://gorilla.cs.berkeley.edu/blogs/16_bfcl_v4_memory.html)
- [HammerBench: Fine-Grained Function-Calling Evaluation in Real Mobile Device Scenarios](https://arxiv.org/html/2412.16516v2)
- [ShortcutsBench: A Large-Scale Real-world Benchmark for API-based Agents](https://arxiv.org/html/2407.00132v3)
- [AndroidWorld: A Dynamic Benchmarking Environment for Autonomous Agents](https://arxiv.org/html/2405.14573v3)
- [Mobile-Agent-E: Self-Evolving Mobile Assistant for Complex Tasks](https://arxiv.org/html/2501.11733v1)
- [SkillWeaver: Web Agents can Self-Improve by Discovering and Honing Skills](https://arxiv.org/pdf/2504.07079)
- [FreeInference Agentic Trace (Hugging Face)](https://huggingface.co/datasets/harvardMadsys/freeinference_agentic_trace)
