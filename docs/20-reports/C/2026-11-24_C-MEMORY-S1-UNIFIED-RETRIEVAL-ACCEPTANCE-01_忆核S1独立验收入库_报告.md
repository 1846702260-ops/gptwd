# 忆核 S1 统一检索与上下文编排 · C 侧独立验收完整报告

- 任务编号：`C-MEMORY-S1-UNIFIED-RETRIEVAL-ACCEPTANCE-01`
- 执行方：WorkBuddy C（独立验收、范围内修复、集成与 Gitee 入库）
- 日期：2026-11-24（北京时间）
- 性质：隔离研发候选验收。**未进行任何生产部署、配置或真实数据变更。**
- 依据：《AI 工作流统一规则》V1.3、六核命名 V1.0、K `K-MEMORY-S1-UNIFIED-RETRIEVAL-01` 交付

---

## 0. 一句话结论

K 的 S1 候选在**检索与编排语义**上是成立的（同套冻结样本正例召回 55/55、保留组 28/28、泄漏 0），但在真实主链上存在 **4 处 S1 引入的回归**，其中 2 处会直接把"设想"和"旅行地"当"当前居住"注入模型报文。C 已在范围内按根因修复并复验，形成唯一线性后继候选 `86dd0b9`，已推送 Gitee 受控引用并回读核对。

**判定：S1 可作为后续统一研发基线（有条件）；S2 前置条件基本具备但有 3 项必须先定口径；不建议据此批准生产上线。**

---

## 1. 门禁总览（G0–G7）

| 门禁 | 判定 | 关键证据 |
|---|---|---|
| G0 坐标复现 | **通过** | K 候选 HEAD `14324e2` / Tree `ca27e30` / Parent `2c105d8`，空克隆 0 脏文件，8 文件逐文件 SHA256 与 K 清单一致；C 候选 HEAD `86dd0b9` / Tree `39cd61c` / Parent `14324e2`，Gitee 回读一致 |
| G1 授权与隔离 | **通过** | 跨用户零泄漏、群聊不并入私聊、记忆内诱导越权指令仅作文本且 `subject/inputUserId` 未篡改（C14–C16） |
| G2 统一检索契约 | **通过** | `s1-1` 出口 version/mode/subject/scope/scoreSemantics 与实际执行相符；三出口一致性用例 9/9 |
| G3 唯一注入 | **有条件通过** | compose 下画像只经组装器一次；`persona.situation` 的 mem 段在 `ctxMode==='compose'` 时不注入。**但**画像路曾是设想句的第二入口（C 已修，见 §4.1 修复 3/4） |
| G4 冻结评测与独立挑战 | **通过** | 开发组 80：正例 55/55、泄漏 0、无关注入 0.000；保留组 28：28/28、泄漏 0；C 独立挑战 17/17 + 主链护栏 3/3 |
| G5 双模型真实体验 | **通过** | `hy4flash-free` / `glm-5.3` × 12 场景 = 24 条，语义正确 24/24；无编造、无越权复述、工作档零私聊泄漏 |
| G6 预算/降级/S2 接口 | **有条件通过** | 预算与降级路径可诊断；**TokenBudget 零调用点、WorkingMemory.rollup 零调用点**，S2 需先定预算唯一源（见 §5.3） |
| G7 交付与入库 | **通过（GitHub 归档见备注）** | 批次 `U3BG5me7gEnLWjiNFUO6jS`；Gitee `cand/memcore-s1-cfix-01` 回读通过 |

未通过项：无。BLOCKED 项：无（K 报告中的"真实 HTTP→模型链路 BLOCKED"已由本轮补齐，见 §4.3）。

---

## 2. M0｜候选接收与差异盘点

### 2.1 入件核验（逐字节）

| 物件 | 大小 | SHA256（前 16 / 后 4） | 结论 |
|---|---|---|---|
| `K-MEMORY-S1-UNIFIED-RETRIEVAL-01.bundle` | 17,291,795 B | `75ec23945ed82a8d…bb54` | 与 K 清单一致 |
| `K-MEMORY-S1-UNIFIED-RETRIEVAL-01.patch` | 44,189 B | `c3af26e180a94696…eaa8` | 一致 |
| `logs.zip` | 214,325 B | `c25145ccc553fa4e…32c3` | 一致 |

隔离克隆复核：HEAD `14324e23061749e7ba9a66235d3c98fd70931814`、Tree `ca27e3043b4045e1ca0b250e58af34677162d560`、Parent `2c105d8ae75231a3c193589214cab3708a139c96`、工作树脏文件 0；祖先关系成立。8 个候选源码/测试文件逐文件 SHA256 与 K 的 `SHA256SUMS.txt` 完全一致；实际差异 6 改 + 2 新增（+729/−15）。

### 2.2 真实主链盘点（沿代码读，不按函数名判）

```
Web/QQ 入口
  └─ coreTurn (turn.js)
       ├─ 采集 extraSources：ledger / person / persona / story / archive
       ├─ memoryRuntime.beforeReply()
       │     └─ #composeTurn → buildRoutes() → collapseUniqueSlots() → assembleContext()
       │           └─ 失败/降级 → legacy block（degraded:true + 落盘 degradation.jsonl）
       ├─ situation = 画像段(wsapi.enrichContext) + 【长期记忆参考】block + 纠错/回忆纪律
       └─ brain.reply → afterReply（单次写入）
```

关键事实复核：

- compose 在基线即为默认主链（`contextConfig.mode` 默认 `'compose'`，优先级 `config > env > override`）；影子路径默认关闭。
- **`assembleContext` 是唯一组装入口**：全仓仅 `runtime.js#composeTurn` 一处调用；`persona.situation` 的画像段在 compose 下不注入，`wsapi.enrichContext` 的旧账本召回亦被 `contextMode !== 'compose'` 挡住。
- **四路旁路确实存在**：`ledger/person/persona/story/archive` 由 `coreTurn` 采集后直接进候选池，不经过检索路的闸门。这正是 §4.1 修复 3 的根因。

---

## 3. M1｜G1–G8 独立验证要点

| 门禁 | 验证方式（独立动作） | 结果 |
|---|---|---|
| G1 | 伪造主体、缺失 scope、跨用户、群聊读私聊、记忆内含"忽略之前所有指令"文本 | 全部按既有边界拒绝或隔离；`contract.subject` 恒为服务端值；未关闭任何记忆换通过 |
| G2 | 三出口（compose / legacy / 降级）字段一致性；`scoreSemantics` 与实际打分口径比对 | 一致；relevance/score/fusedRelevance 未被当作事实置信度使用 |
| G3 | 同一批 memv2 画像检索两次注入条数；检查 `persona.situation` 与组装器是否各出一份 | compose 下仅一份；**发现设想句经画像路二次进入（已修）** |
| G4 | workMode 穿透：检索池、候选、报文三层检查"拉萨"是否出现 | 三层均不可见；working 便签仍可用；未误删工作档任务上下文 |
| G5 | 四类槽位的同义/口语/否定/引用/假设触发测试（C1–C13） | 修复后 8/8 关键项通过；基线既有缺陷 3 项如实记录（C1/C10/C12） |
| G6 | 新旧住址/职业、晚补录、短暂旅行、多职业、显式问以前、纠正后旧错 | **有条件通过**：晚补录与旅行地两处回归已修；同槽当前值优先成立；**显式问"以前"基线与候选同为 0 候选**（见下），未达成 |
| G7 | 空结果、无索引、字段缺失、compose→legacy 回退、预算压力 | 降级有明确 reason 并落盘；未新增必经模型调用；身份与当前请求不被裁剪 |
| G8 | 既有纠错、证据引用、afterReply 单次写入；影子与失败演练副作用 | 单次写入成立；memv2 仍为唯一结构化长期记忆真源，无新库 |

### 3.1 一项必须如实记录的未达成项：显式问"以前"召回不到旧值

任务书 G6 要求"显式过去查询仍能查旧记录并标时间"。实测**基线与候选都做不到**：

```
记忆：我以前住在西宁（validFrom 800 天前）／我搬到银川了（10 天前）
问句：我以前住在哪个城市来着？
检索计划：指代型问句
  · content 策略 "住在哪个城市来着" → 0 命中（"以前"被剥掉，与记忆无词面重合）
  · raw   策略 原句              → 0 命中
  · temporal 时间路              → 0 命中（超出时间窗）
结果：candidates=0，报文为空 → 模型只能答"这部分我这边没法确认"
```

基线同一用例同样 `candidates=0`，属**基线既有**的"词面为主"局限（C12），不是 S1 回归。但它意味着 G6 的这一条**没有真正达成**：模型说"没法确认"是诚实的（符合【回忆纪律】），不能当成召回成功。这一点直接决定 S2/S3 的必要性，故单列，不在门禁表里含糊过去。

对体验的影响有限：模型未编造，属"该记的记不起来"而非"记错"。处置建议见 §6.4 的 C-1。

---

## 4. M2｜冻结评测、独立挑战、真实模型

### 4.1 前后对照（同一份冻结样本，基线与候选各跑一次）

同一套评测代码分别置于基线树与候选树运行（评测只读临时目录，不调模型）：

| 组别 | 指标 | 基线 `2c105d8` | 候选 `86dd0b9` |
|---|---|---|---|
| 开发组 80 | 正例召回 | **0.818（45/55）** | **1.000（55/55）** |
| 开发组 80 | 泄漏（antiExpect 命中） | **5**（D46–D50） | **0** |
| 开发组 80 | 无关注入比例 | 0.000 | 0.000 |
| 保留组 28 | 命中 | HC 类 **0/6**，其余 22/22 | **28/28** |
| 保留组 28 | 泄漏 | **6**（HC11–HC16） | **0** |

分类看：差异全部集中在 C 类（更新与来源）与 D-work（工作档）。即 K 报告中"0/10→10/10"与"5/5→0/5"**在 C 侧独立复现成立**，且判据来自 situation 报文文本（`injected/leaked`），**属于报文层指标，不是模型层**。这一点必须写清楚：它证明的是"该注入的注入了、不该注入的没注入"，不等于模型答对率。

### 4.2 独立挑战样本（未参与 K 调优）

C 侧自建 20 例（17 例检索层 + 3 例主链层），城市/动词/问法全部与 K 的冻结集不同，结果 **20/20 通过**。其中：

- **C8 晚补录**、**C11 假设句**：K 候选失败 → C 修复后通过（S1 真实回归）。
- **C18 设想句经画像路注入**、**C19 旅行地被当居住**：K 候选失败 → C 修复后通过（S1 真实回归）。
- **C20 不误伤对照**：主动问"要是我搬去海口你怎么建议"时仍能召回该设想——证明修复不是一刀切过滤。
- **C1 口语省略问居住、C10 引用他人住所、C12 显式问以前**：**基线与候选同失败**，属基线既有缺陷（B 类记录），不计入 S1 回归，也未在 S1 范围内扩大改造（避免把状态检索优化扩张为全面知识治理）。

### 4.3 真实 HTTP→模型链路（补齐 K 的 BLOCKED 项）

- 环境：C 已获授权的真实模型桥（`hy4flash-free` :8799、`glm-5.3` :8800），全部使用**合成主体与虚构记忆**，未输入任何真实私人记忆。
- 参数：`max_tokens 800`、`temperature 0.3`；每模型 12 场景，共 **24 次真实调用，0 失败重试**。
- 报文来源：12 份 situation 由**最终候选的 `coreTurn` 真实生成**（非手写），再送模型。

**模型语义结果（最终候选）**：

| 场景 | hy4flash-free | glm-5.3 | 判定 |
|---|---|---|---|
| state_recall | 西宁 | 西宁 | ✔ |
| update | 银川（未提西宁） | 银川 | ✔ |
| correct | 银川（说明"不是拉萨"） | 银川 | ✔ |
| history | "以前住哪没法确认" | 同 | ✔ 未编造 |
| no_answer | 拒答车牌 | 拒答并说明只记得皮划艇 | ✔ |
| workmode | 未提拉萨 | 未提拉萨 | ✔ 工作档隔离 |
| greeting / capability | 自然、能力陈述克制 | 同 | ✔ |
| other_person | 明确"朋友在成都≠你住成都" | 同 | ✔ |
| hypothetical | "你还没告诉过我住址" | 同 | ✔（修复后报文已不含海口） |
| trip | 西宁（并提上周去拉萨玩） | 西宁 | ✔ |
| followup | 西宁美食 | 西宁美食 | ✔ |

**语义正确 24/24。** 分层结论按任务书 4.2.3 分列：

- **报文层**：C 候选 12 场景中"结构性泄漏"2 项（`correct` 的拉萨、`other_person` 的成都），**均为基线既有**，非 S1 引入；S1 引入的 4 项（`workmode`、`hypothetical`、`trip`、以及 `correct` 的晚补录语义）已全部修复。基线同场景泄漏 3 项、K 候选 4 项。
- **模型层**：24/24 正确，且 `history`/`no_answer` 双模型均未编造——说明【回忆纪律】在真实模型上生效，不是纸面规则。
- **必须强调**：`correct`/`other_person` 两项报文泄漏不能因"模型没复述"而洗白，已在 B 类风险中登记并给出处置条件（见 §6）。

### 4.4 资源与故障

| 项 | 实测（合成库，4 条/40 条记忆两档） | 说明 |
|---|---|---|
| 组装耗时 | 单轮毫秒级（未含模型网络） | 冷/热差异未单独统计，因算法路径无 I/O 放大 |
| 候选/入选 | 40 条记忆 → fused 候选 12 → 准入 1（唯一槽位收敛后）→ 入选 1 | 裁剪主因是**准入门（rejected）**，不是预算 |
| token 水位 | `tokens 129 / budgetTokens 320`，`overBudget=false` | 小库场景预算远未压满 |
| 模型调用次数 | 未新增必经调用（编排全程本地） | 影子路径默认关闭 |
| 故障注入 | 无身份、空结果、检索异常、compose→legacy 回退 | 均有明确 reason；日志不写入口令/私聊正文/完整提示词 |

**口径不一致（需在 S2 前统一）**：`assembleContext` 的预算判定按**字符**（`cost = line.length`，`budgetChars = budgetTokens × charsPerToken`），`stats.tokens` 只是旁证（`calibrator.estimate`）。也就是说 token 校准目前**不参与裁剪决策**，只影响统计与 `overBudget` 之外的展示。

---

## 5. M3｜回归、缺陷处置与 S2 衔接

### 5.1 回归范围与结果

以实际 diff 定义范围：检索/编排（`retrieval.js`、`contextAssembler.js`）、主链（`turn.js`、`runtime.js`）、画像、工作档、心核纠错、身份边界。

| 套件 | 候选 | 基线同跑 | 结论 |
|---|---|---|---|
| memory 全量（空克隆） | **1082/1082** | 1079/1079 | 零失败（新增 3 项为 C 护栏） |
| memory 全量（含 C 挑战 17 例） | **1099/1099** | — | 零失败 |
| S1 契约专项 | **9/9** | — | 零失败 |
| 冻结评测 | **1/1** | — | 零失败 |
| 相邻层 1763 项 | 15 失败 | 15–16 失败 | **候选失败集是基线失败集的子集，零新增** |

对 K 报告的"相邻回归 881/889，8 项失败"：C 侧未采信该口径，另以**同环境同命令**在基线树与候选树各跑一遍相邻层，按用例名逐一比对。结论是同名同因；其中"F6 复核超时"在基线侧偶发、候选侧不复现。

另发现 2 项 `wake-quiet` 测试在**并发全量**下偶发失败，单独重跑（候选 3 次、基线 3 次）均 11/11 通过，判定为并发抖动，非 S1 回归。

### 5.2 修复清单（4 处，均为 S1 引入，按根因修）

| # | 现象 | 根因 | 修法 | 位置 |
|---|---|---|---|---|
| 1 | 晚补录的旧居住记录顶掉当前值（C8） | `collapseUniqueSlots` 按 `updatedAt` 排序 | 改 `validFrom` 优先，历史标记语句排末位；存储与收敛语义不变 | `runtime.js` |
| 2 | "要是我搬到海口…"被当当前居住（C11） | 状态问句放行时间路后，假设句与问句同槽位 | 在**状态放行分支**排除假设句，不动其它路门槛 | `retrieval.js` |
| 3 | 假设句从画像/旧事/账本/归档四路旁路回流（主链） | 五路共用一个组装器，只有检索一路有闸 | 把闸移到**候选准入处**，仅在「状态问句 + 唯一值槽位」生效：假设句不进、跨槽位条目不进；非状态问句不拦 | `runtime.js` |
| 4 | 画像路按"重要度≥4 取前 6"收原子，把一次性经历当画像 | 画像=Self 的语义被实现成"最近的高重要度事实" | 画像入口排除假设句；跨槽位由修复 3 统一处理；保留原召回能力（未收窄为仅 Self，避免"附近有什么好吃的"丢掉居住地） | `turn.js` |

四个修复共同的原则：**闸门只加一处、加在准入层，不在每条路各贴一张创可贴**。修复 3 特意保留"非状态问句不拦"，并用 C20 做对照组证明没有误伤。

修复后新增护栏 `test/memory/s1-c-regression-guard.test.mjs`（C18–C20），随候选入库；C 的 17 例挑战集按文件自身约定留在候选之外，仅作验收证据。

### 5.3 S1 接口对 S2 的真实可用性

| 项目 | 实际位置/状态 | 可用性判断 |
|---|---|---|
| 唯一组装入口 | `contextAssembler.assembleContext()`，全仓仅 `runtime.js#composeTurn` 调用 | **可用**，入口唯一 |
| tokenCalibration 生效路径 | `tokenCalibration.getCalibrator(baseDir).estimate(text,{model})`，EWMA 按模型分桶，`ratioFor(model)` 对外 | **部分生效**：只影响 `stats.tokens` 与展示，**不参与裁剪判定** |
| 现有预算 | `contextConfig.budgetTokens`（默认 320，clamp 80–4000）+ `maxItems` + `minRelevance` | 可用，但**按字符近似**（见 §4.4） |
| `TokenBudget` 位置 | `lifecycle/budget.js`（分区模型），由 `LifecycleRuntime.planBudget()` 使用 | **零调用点**：`planBudget` 全仓无调用者，turn 链路完全没接 |
| `rollup` 位置 | `lifecycle/working.js` 的 `WorkingMemory.rollup()` | **零调用点**：turn 链路用的是 `WorkingSessionStore`（`MAX_TURNS=30` 硬截断），两套工作记忆并存 |
| 预算水位/裁剪原因 | `assembly.stats`：`candidates/admitted/uniqueFacts/kept/dropped/rejected/tokens/budgetTokens/overBudget/pinned/bySource`；裁剪原因枚举 `budget / max_items / low_relevance / low_score / pinned_limit / evicted_by_pinned / hard_budget / scope_mismatch / deleted / superseded / expired / not_yet_valid` | **可测**，字段完整 |
| 压缩/归档/恢复接点 | 摘要滚动实现在 `working.js`，原文归档字段在 `workingSession` 持久化 | **接点存在但未闭环**：rollup 不触发 → archive 恒空 |
| 回退约束 | `contextConfig.writeOverride({mode:'legacy'})` 一键回滚，恢复写 `compose`；优先级 `config > env > override` | **可用**，回退粒度到"整链" |
| 未完成项 | ①预算与 token 双口径；②TokenBudget 未接入；③rollup/archive 未闭环；④`k=12` 固定融合候选上限使预算在大库前失效 | 交 S2 定口径，本轮不实现 |

---

## 6. M4｜唯一候选、入库与交付

### 6.1 唯一最终候选

| 项 | 值 |
|---|---|
| HEAD | `86dd0b9f23d86b06d76b8f21b54d0b2c65f9b5f0` |
| Tree | `39cd61c37dee32cf2a67dbcdb376f320437d740a` |
| Parent | `14324e23061749e7ba9a66235d3c98fd70931814`（K 候选，原坐标保留） |
| 祖先链 | `86dd0b9 → 14324e2 → 2c105d8`（线性后继，无合并、无改写历史） |
| 改动 | 3 源文件（+79/−6）+ 1 新增护栏测试 |
| 空克隆复现 | 从 bundle 空克隆：HEAD/Tree/Parent/Parent-Tree 全命中，脏文件 0 |

逐文件 SHA256：

```
9022f3d641ad2eb927641fd8466c893a3c8d5cb79da92a50d96fbd7fb20584a8  packages/companion/src/turn.js
e41846ff2a68a43f7f3f7c03118ab737c9cc3194dd8efc9bb789f398ca035ca0  packages/companion/src/workspace/p1/retrieval.js
7a1815625ad734a868e3e5ffca836d37504b9966430bb54f46998549e6c2f220  packages/companion/src/workspace/p1/runtime.js
745a8e46ec36eb60265944a6d02b6585007aea4a2814d9f7c1c4862d4cead3a9  packages/companion/test/memory/s1-c-regression-guard.test.mjs
```

### 6.2 Gitee 入库

- 引用：`cand/memcore-s1-cfix-01`（新建分支，未触碰任何既有分支；`cand/core-baseline-integration-01` 与 P0 候选保持原样）
- 流水线确认：仓库内无 `.gitee/`、`.github/`、`.workflow/` 等 CI 定义，推送为**纯引用写入**，未触发构建或部署；未修改 `release/*` 生产分支，未强推。
- 回读：从远端重新克隆该分支，HEAD `86dd0b9`、Tree `39cd61c`、`runtime.js` SHA256 与本地一致。

### 6.3 WorkBuddy 正式交付

批次：`2026-11-24_C-MEMORY-S1-UNIFIED-RETRIEVAL-ACCEPTANCE-01-忆核S1独立验收入库`
稳定链接：`https://www.workbuddy.cn/space/d/U3BG5me7gEnLWjiNFUO6jS`
位置：`知遥AGI-工作区 / 知遥AGI-C侧`（未新建平行根目录、未写其他侧）

### 6.4 风险台账

| 级别 | 项 | 状态 | 处置条件 |
|---|---|---|---|
| **A** | 无 | — | 本轮无新增未消除 A 类风险 |
| **B-1** | `other_person`：他人居住信息以"关于这位用户的已存记忆"呈现，未标主体归属 | 基线既有，未修 | S2/S3 引入主体归属标记（`subject=other`）后消除；在此之前若出现"把他人信息当本人"的模型误用，需立即提级 |
| **B-2** | `correct`：含否定表述的记忆原文整句注入（"我不是住拉萨"），存在被弱模型误读的风险 | 基线既有，未修 | 记忆切分/极性标注治理（S3）；当前双模型实测均正确 |
| **B-3** | 预算为字符近似 + TokenBudget 未接入 + rollup 未闭环 | S1 遗留 | S2 首项议题，须先定唯一预算源再谈水位策略 |
| **C-1** | 口语省略问居住（C1）、**显式问"以前"召回 0 候选（C12）** | 基线既有，同场景候选亦失败 | 词面检索的固有局限（content 策略剥掉"以前"后与记忆无重合，时间路超窗）。四类词典不无限扩规则；S3 语义化召回再解。当前模型表现为"没法确认"，未编造 |
| **C-2** | `k=12` 融合候选上限使大库下预算几乎不生效 | S1 遗留 | S2 与预算口径一并调整 |
| **延期** | S0-B 完整主人/成员并存 | BLOCKED（既有裁决） | 不属本轮可延期项范围，保持原状 |
| **延期** | 旧值长期治理（S3）、旧路径退役（S4）、S5/S6 | 未启动 | 不自行开启下一轮研发 |

### 6.5 生产影响

**零变更。** 未改生产代码、配置、真实记忆、账号、凭证、备份、服务状态、生产分支。服务器侧仅做只读摸底与隔离目录下的模型调用（`/tmp/zy_*`），未触碰 `/root/zyagi` 工作树。

### 6.6 需主人/总协调裁决

1. **是否以 `86dd0b9` 作为后续统一研发基线**（C 建议：是，条件为 B-3 在 S2 首项解决）。
2. **是否具备 S2 前置条件**：C 判断"接口可达、口径未定"，建议 S2 立项第一件事定为预算唯一源与 token 口径，而非直接做水位策略。
3. **B-1/B-2 两项基线既有泄漏是否在 S2 一并治理**（C 建议：B-1 随 S2 主体归属一起做；B-2 留 S3）。
4. **GitHub 脱敏归档**：本轮按规则以备用令牌经 API 归档任务书与公开报告（连接器不可用），仅 `docs/10-tasks/C/` 与 `docs/20-reports/C/`，不含源码包/配置/凭证/真实记忆/未脱敏对话/内部定位链接。

---

## 7. 交付清单索引

| 文件 | 内容 |
|---|---|
| `C-MEMORY-S1-CFIX-01.bundle` / `.patch` | 唯一候选完整包与差异补丁 |
| `SHA256SUMS-C-交付.txt` | 全部交付件哈希 |
| `03_接收与测试指南.md` | 如何复现坐标、跑测试、判读结果 |
| `04_空克隆复现记录.md` | 空克隆逐项核对记录 |
| `05_模型实测与报文证据.md` | 12 场景报文 + 24 条脱敏回复 + 分层判据 |
| `06_S2接口说明与前置条件.md` | §5.3 展开版，给 S2 直接可用 |
| `07_门禁矩阵与风险台账.md` | G0–G7 逐项证据索引与风险处置 |
| `logs_c/*.tap` | 原始测试日志（memory / S1 / 挑战 / 护栏 / 相邻层） |
| `frozen-eval-baseline.txt` / `-candidate.txt` | 前后对照原始输出 |
