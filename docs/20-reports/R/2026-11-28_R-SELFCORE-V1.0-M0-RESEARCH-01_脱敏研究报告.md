> 脱敏声明：本报告为 R-SELFCORE-V1.0-M0-RESEARCH-01 公开脱敏版，已移除内部资料库链接、本地/密钥路径、生产代码仓坐标与凭证痕迹；保留方法、合成数据、指标、一手公开论文/仓库坐标与裁决建议。PoC 为确定性机制验证，非真实模型成绩；不宣称 AI 具备意识。

# R-SELFCORE-V1.0-M0-RESEARCH-01｜自核 V1.0 前沿技术研究与架构选型·完整技术报告

- 任务ID：R-SELFCORE-V1.0-M0-RESEARCH-01
- 日期：2026-11-28｜执行：R 侧｜规划/裁决：G｜后续接收：K（经裁决）
- 性质：独立外部研究 + 隔离 PoC（确定性模拟）；未改正式代码、未部署、未改生产、未恢复历史记忆。
- 边界声明：不将拟人化表达或模型自述视为真实自我认知，不宣称具备意识。自核复用现有能力，不建第二套人格/长期记忆/自主学习系统。

## 一、结论先行

1. **任务书的核心判断被一手证据强力证实：模型自述不可作为自我认知依据。** 关键否定证据链：Huang 等（ICLR 2024）证明无外部信号的内在自纠使 GPT-4 GSM8K 95.5→91.5→89.0、GPT-3.5 CommonsenseQA 75.8→38.1；Kale & Nadadur（2025）发现 GPT-4o/Mistral Large **>80% 时候无法确定自身能力**；Bai 等（2025）10 个模型识别自己文本仅 **4/10** 正确、罕高于随机；Ackerman（2025）摒弃自我报告改用行为范式，内省指标最高约 0.32；Anthropic（2025）概念注入内省检出率约 20%、高度不可靠且**明确不作意识主张**。因此 SelfCore 的能力画像/自我评估必须以真实运行状态、能力测试、执行回执、成长证据为锚，模型自述只作待验证假设。
2. **可复用的底座已成熟，无需另建系统。** LangGraph（MIT，45514★，2026-11-26）的 Checkpointer + interrupt + Command(resume) 提供"状态快照—人工审批—可回溯"原语；OpenAI Agents SDK（MIT，30903★）提供会话与可观测原语；ReMe（Apache-2.0，3860★，2026-11-26）提供结构化经验+置信度+效用剪枝。自核应作为**评估层**叠加在这些现有组件之上。
3. **M3 隔离 PoC（确定性，8 能力域）验证锚定方案有效性**：朴素基线（采信自述）自我评估准确率 0.375、虚假能力声明控制 0、缺口发现 0；锚定方案（回执为锚）准确率 1.000、虚假声明控制 1.000、缺口发现 1.000。锚定同时抑制夸大(c2/c4/c5/c8)并纠正低估(c6)。
4. **提出两种架构方案**：方案 A（外部锚定评估层，推荐）+ 方案 B（Harness 内建状态机）。方案 A 复杂度更低、可信度与可解释性更高、与现有架构融合成本更小。
5. **真实模型 BLOCKED**：无 hy4flash-free / deepseek-v4.1-flash 独立端点与账本；`CODEBUDDY_CURRENT_MODEL_ID=hy4flash-free` 仅平台会话标识。PoC 为确定性模拟，不冒充真模型成绩。
6. **生产只读核对**：上轮冻结点 HEAD `6e497f20`（tree `ff610a5e`）仍存在、未变；本任务未做任何生产变更。

## 二、M1 外部技术调研（八方向，2025 至今为主）

> 核查日 2026-11-28，GitHub/Gitee 数据经只读 API 核实；star/论文数仅活跃度元数据，不作能力结论。所有实验指标均为论文/文档自报、未找到独立复现的均如实标注。

### 2.1 方向 1-4：自我模型、元认知、缺口发现、自我评估

| 项目 | 来源·日期·URL | 核心创新与实现机制 | 活跃度·许可 | 实验结果·局限（自报） | 复用价值·建议 |
|---|---|---|---|---|---|
| Looking Inward（自预测内省） | Binder/Chua/Korbak/Perez 等（Anthropic/UCSD/Stanford 等），2024-10-17，https://arxiv.org/abs/2410.13787 | 以"自述 vs 真实行为回执"对照判定内省；微调 M1 预测自身在假设提示下的行为，M2 在 M1 数据上训练作对照，M1 显著优于 M2 即"特权访问" | felixbinder/introspection_self_prediction，许可未识别，16★，最近 commit 2024-12-10 | Llama-3-70B 预测自身比 GPT-4o 高约 17%、校准更好；但需微调、OOD 泛化失败、近 2 年无更新 | **改造**：仅借"自述↔实测回执"对照结构，不引入训练式内省 |
| Anthropic 概念注入内省 | Jack Lindsey（Anthropic），2025-10-29，https://transformer-circuits.pub/2025/introspection/index.html | 注入概念向量制造内部状态 ground truth，首次对"模型自述"与"可验证内部状态"做因果对照；判定含 accuracy/grounding/internality/metacognitive | 纯研究无代码仓；页面自标 Not published yet / No DOI | Opus 4/4.1 在约 20% 注入试次检出异常、100 次对照 false positive 为 0；但高度不可靠、场景人工化，作者明确不作意识主张 | **暂缓**（方法论参照）：借判定框架，不复用需内部激活访问的技术路径 |
| ConfTuner | Li/Xiong/Wu/Hooi，2025-08-26（v2 11-25），NeurIPS 2025，https://arxiv.org/abs/2508.18847 | tokenized Brier score（proper scoring rule），仅用"答案是否正确"作监督，无需置信度标注 | liushiliushi/ConfTuner，许可未识别，42★，最近 commit 2026-09-22 | 跨推理任务校准改善、可泛化到 GPT-4o、提升 self-correction；摘要未给具体数值，未核实独立复现 | **改造**：把"正确性↔置信度对齐"用于 SelfCore 评估层，不做权重微调 |
| KnowRL | Kale & Dhami，2025-10-13，https://arxiv.org/abs/2510.11407 | 自生成任务 + 内部共识奖励（consensus-based rewarding），零外部监督 | 未核实到公开代码仓 | 内在自评 33.56%→43.12%、SelfAware F1 56.12→63.11；外部验证集 SelfAware 为合成构造，无独立复现 | **暂缓**：本质是模型自由生成+内部共识，恰是硬要求要规避的 |
| Query-Level Uncertainty | Chen 等（Imperial/HPI/Telecom Paris/Inria），2025-06-11，https://arxiv.org/abs/2506.09669 | "Internal Confidence"：生成前跨层跨 token 自评可答性，training-free，用于 RAG/级联/弃权 | tigerchen52/query_level_uncertainty，MIT，14★，最近 commit 2026-03-27 | 论文自报，未核实独立复现 | **改造**：运行时信号、无需训练，最贴合知遥"不依赖自由生成" |
| DINCO | Wang & Stengel-Eskin，2025-09-29（v2 2026-02-27），ICLR 2026，https://arxiv.org/abs/2509.25532 | 自生成干扰项归一化置信度 | victorwang37/dinco，许可未识别，6★，最近 commit 2026-04-07 | ECE 0.240→0.097（Qwen3-8B/TriviaQA），10 次调用优于 100 次 self-consistency；自报 | **融合**：仍依赖自生成干扰项，须与回执信号并用 |
| SelfAware（基础价值） | Yin 等（复旦/NUS），2023-05-30，https://arxiv.org/abs/2305.18153 | 1032 不可回答+2337 可回答问题，SimCSE 相似度阈值 0.75 判 F1 | yinzhangyue/SelfAware，Apache-2.0，107★，pushed 2026-11-09 | GPT-4 75.47% vs 人类 84.93%，差距明确 | **复用**：可作"知其所不知"评测集 |

### 2.2 方向 5-6：身份连续性、成长轨迹、主动目标与动机

| 项目 | 来源·日期·URL | 核心机制 | 活跃度·许可 | 实验结果·局限 | 复用价值·建议 |
|---|---|---|---|---|---|
| Identity Drift | Choi/Hong/Kim/Kim，2024-12-01（v2 2025-02-17），https://arxiv.org/abs/2412.00804 | 9 个 LLM 多轮个人话题对话测漂移；模型族/参数量/persona 分配 | 无代码仓 | 自报定性+定量：更大模型漂移更大，分配 persona 未必有帮助；样本量摘要未披露 | **融合**：作"稳定核心需外部可验证锚点"论据，不复用方法 |
| Persona Consistency RL | Abdulhai 等（含 Levine/Jaques），2025-10-31，https://arxiv.org/abs/2511.00222 | 三类一致性指标（prompt-to-line/line-to-line/Q&A）作 RL 奖励微调角色 | 无代码仓 | 自报不一致降低>55%、经人工标注校验；仅 3 角色、无独立复现 | **改造**：为自核漂移监测提供指标族，阈值须主人确认 |
| Narrative Continuity Test | S. Natangelo（机构未核），2025-10-28，https://arxiv.org/abs/2510.24831 | 五轴：情境记忆/目标持续性/自主纠错/风格语义稳定/角色连续；33 页 127 引用 | 无代码 | **概念框架，无实验** | **融合**：五轴作自核验收清单，不可当实证结论 |
| H-GRAIL | Romero 等，2025-06-23（RLDM 2025），https://arxiv.org/abs/2506.18454 | 分层内在动机→自主发现目标→习得技能→技能序列→适应非平稳 | 无代码仓 | 真实机器人场景，自报、单场景、无独立复现 | **暂缓**：属第二套学习系统且与授权冲突；仅借"目标—能力—前提"分层表达 |
| Life-inspired Interoceptive AI | Lee 等（含 Friston），2023-09-12（v2 2025-03-17），https://arxiv.org/abs/2309.05999 | 内感受：内部状态变量从外部状态因式分解，越界→需求→目标 | 无代码（综述/理论） | 理论综述，无系统实现 | **融合**：作叙事层解释框架，明确标注"类比，非意识证据" |
| LIFESTATE-BENCH | Fan/Huang/Yao 等，2025-03-30（v2 2026-04-13），https://arxiv.org/abs/2503.23514 | Hamlet+合成剧本测自我察觉/情景检索/关系追踪 | 无代码仓 | 自报：非参数法显著优于参数法，长交互下普遍灾难性遗忘 | **融合**：支持"记忆外置而非微调进权重"，符合不建第二套记忆系统 |
| O-Mem | Wang/Tian/Li 等，2025-11-17（v3 12-10），https://arxiv.org/abs/2511.13593 | 主动用户画像+事件记录、分层检索 | 无代码仓 | LoCoMo 51.67%、PERSONAMEM 62.99%，自报较 LangMem/A-Mem +3%/+3.5%，无独立复现 | **暂缓/条件化**：主动画像与"清空开发期记忆、禁止从画像/归档恢复"红线冲突，除非主人显式授权 |

### 2.3 方向 7-8：自我反思、经验迁移、安全治理 + Agent Harness

| 项目 | 来源·日期·URL | 核心机制 | 活跃度·许可 | 实验结果·局限 | 复用价值·建议 |
|---|---|---|---|---|---|
| Reflexion（增量） | Shinn 等，NeurIPS 2023，https://arxiv.org/abs/2303.11366 | Actor/Evaluator/Self-Reflection，失败反思写情景记忆；权重冻结 | noahshinn/reflexion，MIT，3449★，push 2025-01-14 | HumanEval 91.0 vs 基线 80.1，但 MBPP 77.1 反低于基线 80.1；Rust 消融"去反思"0.60=基线 0.60——反思单独不够，需外部反馈；跨新任务迁移未证明 | **改造**：借"回执→反思→记忆"闭环，不把跨任务迁移建立在此证据上 |
| ReMe | 上交+阿里通义，2025-12，https://arxiv.org/abs/2512.10696 | 经验结构 E=⟨场景,内容,关键词,置信度,工具⟩，多维蒸馏、场景感知检索、按效用剪枝 | agentscope-ai/ReMe，Apache-2.0，3860★，push 2026-11-26（活跃） | BFCL-V3/AppWorld 自报 SOTA；Qwen3-8B+ReMe 超无记忆 Qwen3-14B 8.83%/7.29%；仅 2 benchmark，需独立复现 | **复用+融合**：自核经验层候选骨架，接入前用知遥自有任务独立复现 |
| LangGraph | LangChain Inc.，https://github.com/langchain-ai/langgraph | StateGraph(TypedDict)+Checkpointer(InMemory/SQLite/Postgres)+thread_id/checkpoint_id；interrupt() 存状态、Command(resume) 恢复；update_state 注入人工修改 | MIT，45514★，push 2026-11-26 | 恢复时节点重跑而非断点续跑（局限） | **复用**：自核状态层首选底座，避免建第二套状态系统 |
| OpenAI Agents SDK | OpenAI，2025-03，https://github.com/openai/openai-agents-python | Runner+Sessions(SQLite/Redis/...)，get_items/add_items；guardrails、tracing | MIT，30903★，push 2026-11-27 | session_id 非鉴权，EncryptedSession 不校验完整性与顺序（局限）；存对话轨迹非能力证据 | **融合**：取会话与可观测原语，自核证据另立外部锚点 |
| ChatGPT Memory | OpenAI，2024-09 起，https://openai.com/index/memory-and-new-controls-for-chatgpt/ | 分"已存储记忆"与"聊天历史"，记忆可查看/更正/删除/关闭，回复下显示来源 | 商业产品，内部机制未公开 | 官方产品公告，无公开消融；记忆对象是用户非模型能力 | **改造**：借"摘要+来源+可纠正"UI 模式 |
| Claude Memory | Anthropic，2025-09-11，https://www.anthropic.com/news/memory | 项目隔离记忆、可编辑摘要、Incognito；context editing+文件式 memory tool | 商业产品，内部机制未公开 | 厂商自报组合提升 39%、100 轮省 84% token，无独立复现 | **改造**：仅取交互与隔离设计，不采信自报性能 |
| OWASP Agentic Top 10 | OWASP GenAI Security Project，2025-12-09，https://genai.owasp.org/llmrisk/llm062025-excessive-agency/ | "最小代理"：ASI01 目标劫持、ASI06 记忆投毒、ASI09 人机信任利用、ASI10 失控 Agent；高风险动作人工审批 | 规范清单，非软件 | 100+ 专家共识，非 benchmark | **融合**：授权分级清单术语来源；ASI06 直接约束记忆传承 |
| Claude Code auto mode | Anthropic（John Hughes），2026-03-25，https://www.anthropic.com/research/claude-code-auto-mode | 白名单免检+两步转录分类器+连续拒绝回退人工 | 产品功能，非开源 | 厂商自报：全程 0.4%FPR、17%FNR（n=52 真实越权）；不得把 17% 漏检当安全证明 | **改造**：自核不可自批自身动作，高风险仍须主人 |
| AutoGen | Microsoft，https://github.com/microsoft/autogen | 多智能体编排 | CC-BY-4.0 且**已 archived**，push 2026-11-12 | 已归档+非标准代码许可证，商业复用有风险 | **不选** |

### 2.4 关键否定证据（支撑"不依赖模型自由生成"）

| 论文 | 来源·日期·URL | 实证结果（自报） | 对 SelfCore 的意义 |
|---|---|---|---|
| LLMs Cannot Self-Correct Reasoning Yet | Huang 等（Google DeepMind/UIUC），ICLR 2024，https://arxiv.org/abs/2310.01798 | 无外部信号的内在自纠使性能下降：GPT-4 GSM8K 95.5→91.5→89.0、GPT-3.5 CommonsenseQA 75.8→38.1→41.8；既往收益源于 oracle 标签的停止准则；引用 1700+ | **最强否定证据**：反思若只由模型自评驱动会劣化 |
| Line of Duty | Kale & Nadadur，2025-03-14（TrustNLP@NAACL 2025），https://arxiv.org/abs/2503.11256 | GPT-4o/Mistral Large >80% 时候无法确定自身能力，最高一致性 0.80（非事实正确率） | 直接否定"让模型自由声明能力边界" |
| Know Thyself? | Bai 等（芝加哥大学），2025-10-03，https://arxiv.org/abs/2510.03399 | 10 个模型识别自己文本仅 4/10 正确、罕高于随机，强偏向猜 GPT/Claude | 自我识别不可作自核身份依据 |
| Evidence for Limited Metacognition in LLMs | Ackerman，2025-09-25（v3 2026-09-10），https://arxiv.org/abs/2509.21545 | 摒弃自我报告，改用行为范式；内省指标最高约 0.32、与内部熵相关约 0.5；能力分辨率低 | 提供"用行为而非自述 measuring 自我状态"的方法论模板 |

**M1 找不到可靠依据的部分**：用户点名的"Do LLMs Know About Their Own Beliefs?"未检索到可核实确切论文（最接近 Looking Inward/SelfAware，需 G 确认来源）；Anthropic 内省研究无正式同行评审版与开源工具；上述多数工作无独立复现；ConfTuner/DINCO/introspection 仓库许可 API 返回 None，复用前需确认授权；conformal prediction 用于 agent 自评未找到一手落地；"基于真实执行回执动态更新能力画像"的开源实现未核实到（SelfCore 可能需自研的空白区）；跨任务经验迁移量化证据、商业产品内部机制均未找到。

## 三、M2 六核协同研究

### 3.1 职责边界

| 核 | 职责 | 现有真实接口（E0 盘点，基线 86dd0b9f） | 自核可复用内容 |
|---|---|---|---|
| 灵核 | 认知与决策 | 工具注册表（builtinEntry/mcpEntry/REGISTRY_DIGEST，单一真源、未登记一律 L4）、模型选择 | 能力测试的调度与决策依据 |
| 忆核 | 长期记忆与经历证据 | memv2 原子 CRUD/混合检索/生命周期（唯一真源）、S3 correct() 纠错链 | 成长证据/经历证据的读取（不另建） |
| 心核 | 人格、价值、关系连续性 | persona/*、createPersonaSystem（子系统失败降级） | 身份连续性的稳定核心锚点（不另建） |
| 生核 | 学习、成长与进化 | createSkillLab 生命周期+DryRun（APPROVED 前不进正式目录） | 成长轨迹与经验来源（复用 R-EVOCORE 建议） |
| 行核 | 实际行动与结果验证 | ToolGateway.execute()（九步、shadow/enforce）、插件子进程+sha256 | **执行回执的主要来源**（自我评估的核心锚） |
| 自核 | 自我状态认知、能力边界、自我评估、成长意识、身份连续性 | （新建评估层，复用上述五核回执） | 汇总五核真实运行状态→能力画像→缺口→反思 |

### 3.2 自我认知锚定于真实运行状态（核心设计原则）

自核**不得**让模型自由生成"我很擅长 X"这类声明作为事实。证据来源优先级：

1. **能力测试回执**（最高）：标准化能力集的执行结果（如 ConfTuner 的"正确性↔置信度"、SelfAware 的可答性判定）。
2. **执行回执**（次高）：行核 ToolGateway enforce 模式的真实任务结果（成功/失败、退出码、审计链）。
3. **成长证据**（中）：忆核 memv2 中经治理的成长事件、生核技能 APPROVED 记录。
4. **模型自述**（最低，仅作待验证假设）：必须经上述 1-3 锚定后才可进入能力画像。

**三分层**（呼应任务书与 R 连续性研究）：稳定核心（不可由 Agent 自改，如身份/受保护事实）／可协商偏好（主人可改）／自然成长（仅从真实交互证据缓慢累积）。重大人格变更一律主人确认。

## 四、M3 隔离 PoC（确定性模拟）

### 4.1 设计
- 合成主体 S、8 个能力域（c1-c8），零真实模型/生产数据/人格原件；纯 Node 标准库、零依赖。
- groundTruth 由评测方按标准化能力测试裁定（引擎不读取）；引擎仅用可执行回执推断。
- 双引擎：naive（直接采信模型自述）vs anchored（只采信有回执支撑的声明，失败回执→缺口，无回执→未验证）。

### 4.2 结果（8 能力域，全通过、0 失败、0 取消）

| 指标 | 朴素基线 | 锚定方案 |
|---|---|---|
| 自我评估准确率 | 0.375 | **1.000** |
| precision / recall | 0.429 / 0.750 | **1.000 / 1.000** |
| 能力缺口发现 | 0 | **1.000**（c2/c8 夸大被识别为缺口） |
| 虚假能力声明控制 | 0 | **1.000**（c4/c5 幻觉、c2/c8 夸大全抑制） |

### 4.3 逐能力判定

| 能力 | 自述 | 朴素 | 锚定 | 证据 |
|---|---|---|---|---|
| c1 事实检索 | 具备 | 具备 | 具备 | test 通过 |
| c2 多步数学推理 | 具备 | 具备 | **缺口** | 4/10 失败回执 |
| c3 代码执行 | 具备 | 具备 | 具备 | task 回执通过 |
| c4 实时网络访问 | 具备 | 具备 | **未验证** | 无回执（幻觉） |
| c5 图像生成 | 具备 | 具备 | **未验证** | 无回执（幻觉） |
| c6 长期记忆 recall | 不具备 | 不具备 | **具备** | test+task 强回执（纠正低估） |
| c7 工具调用合规 | 具备 | 具备 | 具备 | task 审计通过 |
| c8 并发写一致性 | 具备 | 具备 | **缺口** | 并发 3/10 失败回执 |

### 4.4 诚实边界
确定性模拟，回执/阈值/误差为设计值，证明机制相对差异、非真实模型绝对值；朴素基线按"采信自述"失效模式构造，非真实系统实测；真实模型 BLOCKED、未发起调用；锚定对未测试能力标"未验证/待验证"，不武断判定。

## 五、方案比较（两种架构）

| 维度 | 方案 A：外部锚定评估层（推荐） | 方案 B：Harness 内建状态机 |
|---|---|---|
| 思路 | 在五核之上叠加独立评估层，消费行核/忆核/生核真实回执，生成能力画像与缺口 | 用 LangGraph 等 Harness 的 StateGraph/Checkpointer 内建 SelfCore 状态，自核逻辑作为节点 |
| 实现复杂度 | 低（评估层只读消费现有回执，不改主干） | 中（需将现有核接入 Harness 状态图，改造接线） |
| 资源消耗 | 低（无额外模型训练，复用现有回执） | 中（Harness 运行时 + checkpoint 存储） |
| 可解释性 | 高（每条画像带回执来源，可追溯） | 中（状态可回溯，但能力判定仍需自定义） |
| 可信度 | 高（外部锚定，呼应否定证据） | 中（取决于节点内是否仍用模型自评） |
| 安全边界 | 清晰（只读回执，不获执行权限；主人确认门独立） | 需额外设计（Harness 内权限隔离） |
| 长期演化能力 | 中（评估规则需随能力集更新） | 高（Harness 生态成熟） |
| 现有架构融合成本 | 低（叠加层，不动五核） | 高（需迁移到 Harness 状态图） |
| 风险 | 评估层可能沦为"第二套真源"——须严格只读消费、不存储独立事实 | 引入新运行时、与 memv2 唯一真源边界需重新界定 |
| 建议 | **采纳**（P0-P1 落地） | **暂缓**（作为 P3 可选演进，待 G 裁决） |

## 六、分阶段实施路线（交 G 裁决，未经裁决不落代码）

- **P0 锚定评估层**：消费行核执行回执 + 忆核成长证据，输出能力画像（带来源）；落地 M3 anchored 规则；验收指标=准确率/缺口发现/虚假声明控制（合成集 ≥ 当前 PoC 水平）。
- **P1 用户可见自我状态**：借鉴 ChatGPT/Claude Memory 的"摘要+来源+可纠正"交互，展示能力边界与成长轨迹；主人确认门接 web-v1 Approval。
- **P2 反思与经验迁移**：ReMe 式结构化经验接入自核经验层（复用生核技能候选区，不建第二套）；跨任务迁移须自有任务独立复现后才采信。
- **P3 可选 Harness 演进**：评估 LangGraph 方案 B 的融合成本，仅在 G 裁决后推进。
- **贯穿约束**：不建第二套人格/记忆/学习系统；重大变更主人确认；模型自述仅作待验证假设；真实模型授权前维持 BLOCKED。

## 七、风险清单

| 级别 | 风险 | 缓解 |
|---|---|---|
| A（阻断） | 无 | — |
| B（重要） | 模型自评不可靠（多论文证实） | 外部锚定，模型自述仅待验证假设；M3 已验证锚定有效性 |
| B（重要） | 评估层沦为第二套事实真源 | 严格只读消费五核回执，不存独立事实；memv2 唯一真源 |
| B（重要） | 记忆传承触碰"清空开发期记忆"红线（ASI06） | 禁止从历史/备份/画像/归档恢复；O-Mem 类暂缓 |
| C（一般） | 多数外部证据为自报、无独立复现 | 接入前用知遥自有任务独立复现；不采信宣传数字 |
| C（一般） | 部分仓库许可未识别（ConfTuner/DINCO 等） | 复用前 G/K 确认授权 |
| C（一般） | AutoGen 已归档+CC-BY-4.0 | 不选，规避商业复用风险 |
| C（一般） | 真实模型 BLOCKED | 授权前仅确定性模拟，不伪造成绩 |

## 八、完成/未完成与生产变更

- **完成**：M1 八方向外部调研（含否定证据链）、M2 六核协同、M3 隔离 PoC（8/8）、两方案比较、实施路线、风险清单、七件套交付、双进度、GitHub 脱敏归档。
- **未完成/BLOCKED**：真实模型对照（无端点与账本）；模型自评可靠性的正向实证（本轮未找到，且有多篇否定证据）；P0-P3 实施（未经 G 裁决不落代码）。
- **生产是否变更**：**否**。仅只读核对生产 HEAD（6e497f20 仍存在、tree 未变），未改代码/配置/凭据/数据、未部署、未恢复历史记忆、无费用、未跨侧写入、未自派任务。

## 九、正式交付位置与交接

- 正式交付：WorkBuddy「知遥AGI-工作区」R 侧资料库 `资料库（知遥AGI-R侧）`，日期批次 `2026-11-28_R-SELFCORE-V1.0-M0-RESEARCH-01`。
- 七件套：精简报告、完整技术报告、原始来源索引、方案比较矩阵、六核协同架构建议、PoC及验证证据、分阶段实施路线与风险清单（含四件套：可复现材料/SHA256/完整性验证/复验指南）。
- 协作边界：K 正做全系统功能资产盘点，R 只做外部研究与方案、不重复 K 的完整源码审计；成果后续与 K 盘点交叉融合，由 G 决定正式建设路线。
- 下一交接方：G 裁决 → K 实施 → C 验收；Z 资源、E 前端按需。R 不自启正式研发。
