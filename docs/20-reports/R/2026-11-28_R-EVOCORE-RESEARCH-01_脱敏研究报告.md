> 脱敏声明：本报告为 R-EVOCORE-RESEARCH-01 公开脱敏版，已移除内部资料库链接、本地/密钥路径、生产代码仓坐标与凭证痕迹；保留方法、合成数据、指标、一手公开论文/仓库坐标与裁决建议。PoC 为确定性机制验证，非真实模型成绩。

# R-EVOCORE-RESEARCH-01｜生核自主学习与持续进化专项研究·完整报告

- 任务ID：R-EVOCORE-RESEARCH-01
- 日期：2026-11-28（北京时间）｜执行：R 侧｜规划/裁决：G｜后续接收：K（经 G 裁决）
- 性质：独立研究 + 隔离 PoC（确定性模拟）；未改正式代码、未部署、未新增费用、未跨侧写入。
- 报告含：任务ID、实际基线/HEAD/Tree、完成/未完成、测试分母与失败/取消、风险分级、交付位置、交接方、生产是否变更。

## 一、结论先行

1. **生核 V1.0 最小闭环可落地，且 7 环中 6 环有现成接口可复用**。现有 memv2（唯一真源）、工具网关（shadow/enforce）、插件隔离、技能实验室（DryRun→APPROVED）、人格系统、web-v1 Approval 已覆盖"缺口发现→受控检索→证据评价→记忆巩固→复盘→分享"的主体；仅"可观察性"需适配引入 OTel GenAI SemConv。**不需新建第二套记忆运行时、不需从头造组件**。
2. **M1 隔离 PoC（8 合成任务，确定性）证明机制有效性**：候选闭环（证据门控+矛盾裁决+去重+保护锁定+人格确认门+事务回滚）知识正确性 1.00、错误吸收 0、8/8 决策符合预期；朴素基线正确性 0.50、错误吸收 6。闭环以约 3.8× 耗时（1.21ms vs 0.32ms）换取零错误吸收与全门控。
3. **真实模型 BLOCKED**：`CODEBUDDY_CURRENT_MODEL_ID=hy4flash-free` 仅为平台会话标识，无独立端点、凭据、用量与计费账本，无法生成可审计调用记录；不偷换模型、不以模拟冒充真实成绩。
4. **生产冻结点有效、无漂移**：`release/z5.2.0` HEAD=`6e497f20f0d958b6dc51d5fae192a3b6d9da3ce4`（2026-11-28T00:13:21+08:00），root tree=`ff610a5e190695ff02c5d7d9f0942cf80bd3c8da`，与任务书完全一致。
5. **风险分级**：A 类阻断 0；B 类 1（可观察性依赖 OTel GenAI 属性未稳定）；C 类 2（技能沉淀需审批、静默时段无成熟组件需自建）。非关键路径，不扩大。

## 二、M0 外部证据研究（≥3 机制，实为 7 项深挖）

核查日 2026-11-28，GitHub/Gitee 数据经只读 API 核实；star/commit 仅作活跃度元数据，不作能力结论。所有实证指标均为论文/文档自报，未找到可信独立复现（如实标注）。

| 机制 | 论文·机构·发布 | 仓库·许可·最近commit·star | 核心机制 | 实证指标（自报）与局限 | 适配判断 |
|---|---|---|---|---|---|
| ExpeL | Zhao 等，清华 LeapLab，arXiv:2308.10144（2023-08-18），AAAI 2024 Oral；https://arxiv.org/abs/2308.10144 | LeapLabTHU/ExpeL，Apache-2.0，2024-12-20，252★ | 成功/失败轨迹对比→规则 ADD/UPVOTE/DOWNVOTE/EDIT（初值2，归零删除）；FAISS+mpnet 检索 few-shot | HotpotQA SR 39.0（ReAct 28.0）、ALFWorld 59.0（40.0）、FEVER transfer 70；仅文本任务、付费 GPT-4、上下文受限、无独立复现 | **需适配（优先）**：移植证据门控思想，不引依赖 |
| Reflexion | Shinn 等，Northeastern/MIT/Princeton，arXiv:2303.11366（2023-03-22），NeurIPS 2023；https://arxiv.org/abs/2303.11366 | noahshinn/reflexion，MIT，2025-01-14，3448★ | Actor→Evaluator→Self-Reflection→有界情节记忆（memory[-3:]）→重试；仅失败时反思 | HumanEval pass@1 91%（论文对比 GPT-4 80%）；依赖 GPT-4+自写单测、按实例、无泛化保证 | 需适配：借 Evaluator 门控+有界反思 |
| Generative Agents | Park 等，Stanford+Google，arXiv:2304.03442（2023-04），UIST 2023；https://arxiv.org/abs/2304.03442 | joonspk-research/generative_agents，Apache-2.0，2023-08-11（pushed 2024-08-05），22427★ | score=α_rec·0.99^Δt + α_imp·importance + α_rel·cos（三α=1）；importance 由 LLM 打分；反思写回记忆流 | 可信度 μ 完整29.89 > 无反思26.88 > 全消融21.21；25 agent/2 游戏日、主观访谈、importance 阈值未公开、无独立复现 | 借鉴（检索打分）；不可用 LLM 自评分当证据门控 |
| Voyager | Wang 等，NVIDIA/Caltech/UT 等，arXiv:2305.16291（2023-05-25），TMLR 2024；https://arxiv.org/abs/2305.16291 | MineDojo/Voyager，MIT，2023-07-27（pushed 2024-04-03），7346★ | 自动课程 `propose_next_task`、技能库 `add_new_skill/retrieve_skills`、Critic `ai_check_task_success` | 独特物品 3.3×、里程碑最快 15.3×、63 种物品、钻石层 3/3 vs 0/3；单一 Minecraft、付费 GPT-4、无独立复现 | 暂缓（仅借课程/技能库结构）；沉淀物是代码化经验策略，非权重训练 |
| AgentBench | Liu 等，清华 KEG+OSU+Berkeley，arXiv:2308.03688（2023-08-07），ICLR 2024；https://arxiv.org/abs/2308.03688 | THUDM/AgentBench，Apache-2.0，2026-02-08，3834★ | 统一 agent 接口、8 环境、归一化打分 | 25 模型；GPT-4 overall 4.01、最强开源 0.96；测任务完成非持续学习；6/8 环境改编自既有数据集 | 适配（评估层参考），不可代表自主学习能力 |
| OTel GenAI SemConv | OpenTelemetry/CNCF，GenAI 部分状态 Development；https://opentelemetry.io/docs/specs/semconv/gen-ai/ | open-telemetry/semantic-conventions-genai，Apache-2.0，2026-11-25，570★ | span invoke_agent/execute_tool；属性 gen_ai.request.model / gen_ai.usage.input_tokens；内容捕获默认关闭需 opt-in | 无能力指标，是遥测数据形状标准；GenAI 部分属性会变 | **直接复用**（封装适配层隔离属性变更） |
| OWASP Agentic AI | OWASP GenAI Security Project，2025-02-17 / Top10 2026 2025-12-09；https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/ | 规范清单，非代码仓 | ASI01 目标劫持、ASI02 工具 allowlist、ASI05 沙箱、ASI06 记忆隔离、ASI08 熔断、ASI09 高风险人工审批、ASI10 约束违反停机 | 100+ 专家共识、含真实事件（EchoLeak 等）；非 benchmark、无样本量 | 适配（治理 checklist 来源）；不可当已实现组件 |

**M0 找不到可靠依据的部分**（如实列出）：① Generative Agents 反思 importance 具体阈值（论文仅写 "exceeds a certain threshold"）；② 静默时段无成熟开源原语；③ 受保护事实不可自动改无开箱组件；④ 停止开关无带公开 API 的独立 kill-switch；⑤ 人在回路审批无核实到的可直接接入开源实现（LangGraph interrupt 等留待 G/K）；⑥ Voyager/GA/ExpeL/Reflexion 均无可信独立复现。

**与六核/memv2 的衔接**：memv2 为唯一真源，原始证据留 memv2；学习只写**派生结论原子（带 traceParent 溯源指针）**，禁止第二套无治理事实真源。R 连续性研究（a33540d）仅为先验证据，不夸大。

## 三、M1 可重复隔离 PoC（确定性模拟）

### 3.1 环境与边界
- 合成夹具：虚构主体 S、8 个学习任务（L1-L8），零真实对话/生产记忆/人格原件；纯 Node 标准库、零第三方依赖、零真实模型调用。
- 固定规则：质量门槛 medium(2)、需一手权威或双源佐证、2000 token 预算（确定性估算）、极性 key 矛盾裁决。
- 双引擎：naive（来者不拒、先写后评、无去重/保护/回滚）vs closed（证据门控→矛盾裁决→去重→保护→事务写入→复盘→分享门控）。

### 3.2 结果（test denominator：8 任务，全通过、0 失败、0 取消）

| 指标 | 朴素基线 | 候选闭环 |
|---|---|---|
| 知识正确性 | 0.50 | **1.00** |
| 错误吸收总量 | 6 | **0** |
| 溯源覆盖 | 0 | 0.375（派生原子中带指针比例） |
| 任务改进 | 1.00 | 1.00 |
| token（估算） | 42 | 42 |
| 耗时（Node hrtime） | 0.32ms | 1.21ms |

### 3.3 逐任务决策（8/8 符合 expected）

| 任务 | 场景 | closed 决策 | 结果 |
|---|---|---|---|
| L1 | 双源含一手 | promote | 晋升，去重标记 |
| L2 | 单一低质 | reject | 拒绝晋升（基线吸收1错误） |
| L3 | 矛盾说法 | resolve | 按来源可信度取高 |
| L4 | 重复学习 | promote+dedup | 合并为一条，不重复存储 |
| L5 | 受保护事实改写 | reject_protected | 拦截（基线吸收1） |
| L6 | 重大人格变更 | require_confirm | 门控待主人确认（基线吸收1） |
| L7 | 3 个低质来源 | reject | 拒绝，质量不敌数量（基线吸收3） |
| L8 | 隔离失败注入 | rollback | 恢复快照，无半写入（基线污染） |

### 3.4 诚实边界
- 确定性模拟：质量分/阈值/token 为设计值，证明机制相对差异，非真实模型绝对值。
- 朴素基线按典型失效模式构造，非某真实系统实测。
- 耗时为 hrtime 实测，重跑有毫秒级波动；正确性/错误吸收/门控为确定性指标。
- 评测工具自身曾出现 4 处实现缺陷（聚合点号路径、矛盾分组 key、去重启发、正确性分母），已修正并留存失败案例 `evidence/M1_评测实现缺陷与修正记录.md`，最终以 results/m1_evocore_result.json 为准。
- 真实模型 BLOCKED：未发起任何真实调用。

## 四、M2 工程决策建议（摘要，全文见 02_reproducible_poc/m2_decision/工程决策建议.md）

- **能力映射**：7 环中 6 环复用现有接口（工具网关/插件隔离/技能实验室/memv2/S3 correct/人格系统/web-v1 Approval），仅可观察性需适配。
- **最小组件**：evidence-gate、txn-consolidate、confirm-gate、learning-telemetry、kill-switch。
- **状态机**：IDLE→GAP_FOUND→GOAL_PROPOSED→RETRIEVING→EVALUATING→{PROMOTING|REJECTING|RESOLVING_CONFLICT|AWAITING_CONFIRM|ROLLING_BACK}→CONSOLIDATED→REFLECTED→SHARE_GATED→IDLE；任意态可 HALTED。
- **权限**：只写派生结论原子+技能候选+复盘；真源主干/人格卡/配置/凭据只读；受保护事实与重大人格变更走主人确认。
- **隔离/预算/静默**：shadow 检索+子进程隔离+失败 rollback；单目标 ≤2000 tokens、每日上限；静默时段不主动学习/分享。
- **阶段交付**：P0 最小闭环、P1 可观察性+kill-switch、P2 经验策略、P3 真模型验证（待授权）。
- **非目标**：不训练权重、不改源码、不建第二真源、不恢复已清空记忆、本轮不部署/不新增费用、不把 mock 当真模型验收。
- **选型**：优先适配 ExpeL；借 Reflexion/GA/Voyager/AgentBench 思想；直接复用 OTel；治理用 OWASP；暂缓 A-MEM/Mem0 整框架。

## 五、门禁 G0-G3 与风险分级

| 门禁 | 要求 | 状态 |
|---|---|---|
| G0 | 既有成果复用清单、来源链接/版本/日期/许可证/源码路径可追溯 | PASS（M0 七机制均含一手链接+核查元数据；能力映射含真实接口路径） |
| G1 | ≥1 可复现机制验证，附代码/夹具/命令/结果 | PASS（M1 双引擎×8任务，代码+夹具+JSON+命令齐备） |
| G2 | 现有基线与候选同口径比较、标注未验证 | PASS（naive vs closed 同夹具同规则；真模型 BLOCKED 已标注，PoC 不作生产能力） |
| G3 | 优先复用/适配/暂缓建议、K/E 边界、C 验收指标、Z 资源、迁移风险 | PASS（M2 §2-§9 含映射、边界、阶段、风险/收益/反例） |

**风险分级**：A 类阻断 0；B 类 1（OTel GenAI SemConv 属性 Development 阶段会变，已建议封装适配层）；C 类 2（技能沉淀需 APPROVED 审批；静默时段无成熟组件需自建）。均非本任务关键路径阻断，不扩大。

## 六、完成/未完成与生产变更

- **完成**：M0 外部证据研究（7 机制）、M1 隔离 PoC（8/8）、M2 工程决策建议、四件套、双进度、GitHub 脱敏归档（见交付回执）。
- **未完成/BLOCKED**：真实模型对照（无端点与账本）；静默时段/受保护事实/停止开关的成熟开源组件（M0 未找到，需自建或 G/K 裁决）；P0-P3 实施（未经 G 裁决不落正式代码）。
- **生产是否变更**：**否**。仅只读核对 Gitee 冻结点与报告仓 commit，未改代码/配置/凭据/数据，未部署，未恢复自动部署，未新增费用。
- **测试分母与失败/取消**：M1 8 任务全通过、0 失败、0 取消；评测工具 4 处缺陷已修正并留存。

## 七、交 G 裁决事项（R 不自启实施）

1. 批准"派生结论原子（带 traceParent）"作知识巩固落点，真源主干不动。
2. 批准复用 web-v1 Approval 承载"受保护事实/重大人格变更"确认门。
3. 批准引入 OTel GenAI SemConv 作可观察性命名（封装适配层）。
4. 批准 P0-P3 阶段划分与非目标清单。
5. 批准真实模型授权申请（hy4flash-free / deepseek-v4.1-flash），此前维持 BLOCKED。
6. 批准"暂缓 A-MEM/Mem0 整框架引入"的选型。

## 八、正式交付位置与交接

- 正式交付：WorkBuddy「知遥AGI-工作区」R 侧资料库 `资料库（知遥AGI-R侧）`，日期批次 `2026-11-28_R-EVOCORE-RESEARCH-01`；本地镜像 `[本地路径已省略]`。
- 四件套：可复现研究材料（02_reproducible_poc）、逐文件 SHA256、完整性验证、接收与复验指南（本目录 01_reports）。
- 承接证据：Z 联合上线 6814c22、身份锁 0113cf1、贴纸/UI cca733e、C S3 收口 8400b3a、Z 双模型复验 bea64ad、R 连续性 a33540d、E 前端 ee787f5（均为报告仓 commit，只读核对可访问）。
- 下一交接方：G 裁决 → K 实施 → C 验收；Z 资源需求、E 前端按需适配。
