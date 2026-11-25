> 说明：本文件为脱敏公开版，供 ChatGPT 阅读辅助。完整原件（含资料库稳定链接、原始日志、复现包、哈希）保留在 WorkBuddy「知遥AGI-R侧」资料库；不含任何凭证、私人记忆、真实生产配置或内部可定位链接。

# R-MEMCORE-OSS-LIVE-COMPARISON-02 完整研究报告
## 开源原版记忆链路实测与公开基准最小对照

- 日期：2026-11-25（北京时间）
- 执行：R 侧（隔离研究）｜裁决：ChatGPT
- 承接：R-MEMCORE-OSS-REUSE-BENCHMARK-01（上轮 EverOS 仅安装/CLI/demo/init 成功，server 未监听；Hindsight 下载受阻）
- 边界：不改正式基线、不入生产、不获取秘密、不自行派发 Z 或启动下一任务

---

## 0. 结论摘要（先给判据，再给证据）

1. **上轮「server 起来但端口永不监听」的根因已定位并修复**：不是配置错误，也不是端口冲突，而是 EverOS 启动时 `warm_token_counter()` 会**无超时**地联网下载 tiktoken `o200k_base` 编码；本环境出口实测约 39 KB/s，3.5 MB 文件需 92 秒，超过观测窗口后被误判为「卡死」。预置 `TIKTOKEN_CACHE_DIR` 后，启动在 0.6 秒内完成，`/health` 立即返回 200。**这是环境/网络问题，不是 EverOS 缺陷**，但暴露了一个通用的「首次启动静默长阻塞」风险。
2. **EverOS v1.7.1 完整原版链路已在本机真实跑通**：真实 uvicorn 监听 127.0.0.1:18000、真实 SQLite + LanceDB 落盘、真实 Markdown 源真值写入、真实 BM25 检索、真实删除。链路为 `add → flush(抽取) → Markdown 落盘 → keyword 检索 → delete → 重新写入`。
3. **真实模型环节 BLOCKED**：R 侧环境无 `hy4flash` / `DeepSeek-V4.1-Flash` 的端点与凭据（环境变量层面不存在）。按任务书要求，不以 mock 冒充成绩，改为交付**无秘密测试包 + 报文结构 + 最小预算**，由 ChatGPT 协调 Z 执行授权环节。
4. **检索层做出了真正的同条件对照**（这是本轮唯一可宣称「真实成绩」的部分）：同一 6 条合成记忆、同一 6 个查询、同一 top_k=5，比较 **EverOS 原版 BM25（LanceDB/tantivy + jieba）** 与 **知遥 `recall.js` 原版 BM25（自实现 bigram）**。结果：**top-1 命中率两者均 5/5**；**recall@5 知遥 0.90、EverOS 1.00**（差异全部来自一道双要素长距题）；无答案查询两者都返回低分噪音，说明**双方都缺「无答案阈值」**。
5. **不建议整体采用 EverOS 或 Hindsight**：EverOS 强依赖 LLM + 派生索引服务化，运维面显著变大，而本轮未观测到相对现有实现的检索优势；Hindsight 更重（131 依赖、含嵌入式 PostgreSQL 与 onnxruntime）。**建议只做机制复用**，具体见第 6 节裁决矩阵。

---

## 1. M0 资产复用、依赖与许可复核

### 1.1 复用情况（不重做上轮成果）

| 资产 | 位置 | 复用方式 |
|---|---|---|
| EverOS 隔离 venv | `oss_reuse/venvs/everos312`（Python 3.12.3，557 MB） | 直接复用，未重建 |
| EverOS 源码树 / 关键文件 | `oss_reuse/src`、`oss_reuse/keyfiles` | 直接读，未重新下载 |
| 11 类 + 6 类合成场景 | 上轮 `poc_ossbench` | 作为本轮对照语料设计参考 |
| 安装/运行日志 | `oss_reuse/evidence/install_logs`、`runtime` | 作为对照基线 |

### 1.2 依赖实况（实测，非声明）

| 组件 | 实测版本 | EverOS 声明约束 | 说明 |
|---|---|---|---|
| Python | 3.12.3 | `>=3.12` | 上轮已确认 3.11 不可用 |
| lancedb | 0.34.0 | `>=0.34.0,<0.35.0` | 与声明一致；`AsyncTable.query().nearest_to_text()` 可用 |
| pyarrow | 随 lancedb 解析 | — | 磁盘 166 MB，是体积最大项 |
| jieba | 随包解析 | — | 中文分词，磁盘 38 MB |
| tiktoken | 随包解析 | — | 需联网首次下载编码（根因） |

依赖体积：venv 合计 **557 MB**（lancedb 141 MB + pyarrow 166 MB + jieba 38 MB）。对比知遥 baseline 源码 26 MB。

### 1.3 许可复核（一手）

| 项目 | 代码许可 | 数据许可 | 结论 |
|---|---|---|---|
| EverOS | Apache-2.0 | 无（框架，不含数据） | 可商用，需保留声明 |
| Hindsight | MIT | 无 | 可商用 |
| Mem0 / Graphiti / Letta | Apache-2.0 | 无 | 可商用 |
| LongMemEval | MIT | **MIT**（HF 卡片 `license: mit`） | **可商用**，需署名 |
| PersonaMem-v2 | — | **CC BY 4.0** | 可商用，需署名 |
| LoCoMo | — | **CC BY-NC 4.0** | **非商用**，仅研究，不引数据入库 |

> 关键纠正：上轮仅记录「LongMemEval 代码 MIT」。本轮核对到**数据集本身也是 MIT**（`xiaowu0162/longmemeval-cleaned`，卡片 front-matter `license: mit`，无附加限制条款，语言 en，三 split：`oracle` / `s_cleaned` / `m_cleaned`）。这与 LoCoMo 的 CC BY-NC 4.0 形成实质差别：**LongMemEval 可用于商用评估，LoCoMo 不可**。

---

## 2. M1 EverOS 完整原版真实运行

### 2.1 根因：为什么上轮「进程在、端口不在」

日志停在 `tracing_score_sink_ready`，之后无 `Application startup complete`。读码定位到启动链：

```
Application.start()
  └─ _start_components()
       ├─ init_score_sink()            → tracing_score_sink_ready   ← 上轮日志最后一行的位置
       ├─ warm_token_counter()         ← 阻塞点
       ├─ root.ensure() / create_schema()
       ├─ start_index() / cascade / OME / memorize_queue
       └─ application_started          → 之后 uvicorn 才开始监听
```

`warm_token_counter()` 的实现（`everos/service/_warmup.py`）明确写着：tiktoken 首次使用会联网下载 `o200k_base`，缓存到 `TIKTOKEN_CACHE_DIR`；异常会被包装成 `TokenizerAssetError`，**但下载本身没有超时**。慢网络下它既不返回也不报错，uvicorn 因此永远不会进入监听状态。

实测证据：

| 项 | 数值 |
|---|---|
| 编码文件 | `o200k_base.tiktoken`，3.5 MB |
| 下载耗时 | **92.0 s**（推算出口带宽 ≈ 39 KB/s） |
| 版本探测 | HTTP 206 正常，非墙、非 403 |
| 预置缓存后启动 | **0.53–0.62 s** 到 `application_started` |

修复方式（不改源码）：把编码文件按 tiktoken 缓存键（`sha1(url).hexdigest()`）落到 `TIKTOKEN_CACHE_DIR`，启动即命中本地缓存。该文件 sha256 已在复现包中登记。

> 通用教训：任何「首次启动联网拉资产且无超时」的组件（tiktoken、spaCy 模型、HF 权重、嵌入式 PG 二进制）在受限出口上都会表现为「进程活着但服务不响应」。诊断时应先怀疑启动期外部下载，而不是端口/防火墙。

### 2.2 真实 server 运行证据

```
$ everos server start --root <isolated_root> --port 18000
2026-11-25T09:49:46 [info] cascade_lifespan_ready
2026-11-25T09:49:46 [info] ome_engine_started
2026-11-25T09:49:46 [info] memorize_queue_started accepting=True max_concurrency=4
2026-11-25T09:49:46 [info] application_started
2026-11-25T09:49:46 [info] Application startup complete.
2026-11-25T09:49:46 [info] Uvicorn running on http://127.0.0.1:18000
2026-11-25T09:49:47 [info] "GET /health HTTP/1.1" 200
```

`/health` 真实返回（节选）：

```json
{"status":"ok","version":"1.7.1","pid":15262,
 "capabilities":{"embed":false,"rerank":false,"multimodal_llm":false,"parser":true},
 "disabled_features":["vector_search","hybrid_search","reflection","skill_extraction",
                      "agentic_search","knowledge","multimodal_upload"],
 "cascade":{"healthy":true,"reasons":[],"pending":0,...}}
```

**关键事实**：无 embedding/rerank 时，服务**降级为 keyword-only** 并显式列出被禁能力——不是静默假装全功能。这一点对选型很重要：它把「能力边界」暴露在健康检查里。

### 2.3 真实 API 契约与失败行为（负例保留）

| 调用 | 结果 | 说明 |
|---|---|---|
| `POST /api/v2/memory/add`（mock LLM） | **503** `EXTERNAL_SERVICE_UNAVAILABLE`：「no usable reply in 2 attempts ... reply failed validation against Reply」 | 抽取回复必须严格匹配 Pydantic schema，**不静默降级、不吞错** |
| `POST /api/v2/memory/search`（带 `session_id`） | **422** `INVALID_INPUT`：「Extra inputs are not permitted: session_id」 | 请求体严格 `extra=forbid`，`session_id` 不属于检索入参 |
| `POST /api/v2/memory/search`（`method=hybrid`，无 embedding） | **500** `ProviderNotConfiguredError`：「Provider 'embedding' (required by user_hybrid) is not configured」 | **不静默降级为 keyword**，必须显式选 `keyword` |
| `POST /api/v2/memory/add`（`defer_extraction=true`） | **200** `{"message_count":2,"status":"accumulated"}` | 只入 durable buffer，未抽取；此时 Markdown 尚未产生 |

> 这三条负例比「跑通」更有价值：它们说明 EverOS 在契约层是**严格失败**风格。好处是问题早暴露；代价是集成方必须精确对齐 schema 与 provider 配置，没有「凑合能跑」的中间态。

### 2.4 完整链路（真实引擎 + 官方 FakeLLMClient）

为在不引入真实模型的前提下验证**存储、落盘、检索、删除**四条机制，使用 EverOS 自带官方测试替身 `everalgo.testing.FakeLLMClient` 进程内注入（`build_application(settings, providers=Providers(llms=...))`），替身内容由确定性规则从输入文本抽取。

> 分层声明：**引擎、SQLite/LanceDB、Markdown 写入、BM25 排序、删除语义均为真实代码**；**只有 LLM 回复是替身**。本报告不据此声称任何语义抽取质量或 benchmark 成绩。

链路实测：

| 步骤 | 结果 | 耗时 |
|---|---|---|
| `app.start()` | 成功，index_backend=lancedb | 0.53 s |
| `add`（2 条消息） | `message_count=2 status=accumulated` | 0.02 s |
| `flush` | `status=extracted` | 0.36–0.37 s |
| cascade 写入索引 | `episode upserted=1`、`atomic_fact upserted=1` | < 1 s |
| Markdown 落盘 | 1 个 episode 文件，976 bytes | — |
| `delete` | `DeleteResult(filters=['user_id'], count=2)` | 0.05 s |
| 删除后检索 | 0 命中 | 2.7 ms |
| 重新写入 + 检索 | 新 id `..._00000002`，命中 score 0.2095 | 0.36 s |

**Markdown 源真值结构**（真实落盘，节选）：

```markdown
---
id: episode_log_u1_2026-11-25
type: episode_daily
file_type: episode_daily
schema_version: 1
user_id: u1
track: user
date: '2026-11-25'
entry_count: 2
last_appended_at: '2026-11-25T10:21:21.817969+00:00'
---
<!-- entry:ep_20261125_00000002 -->
## ep_20261125_00000002

**owner_id**: u1
**session_id**: r-official-1
**timestamp**: 2026-11-25T10:21:01.043000+00:00
**parent_type**: memcell
**parent_id**: mc_213276e8f1a5
**sender_ids**: [u1]
**language**: Chinese

### Subject
...
### Summary
...
### Content
...
```

要点：
- **`parent_type` + `parent_id` 构成物理级溯源**：每条 episode 指回 memcell，可反查原始消息批次。这是「来源可追溯」的落地形态，不是文档里的一句承诺。
- **`deprecated_by` 字段存在**于索引表（`episode` 表列名可见），为事实级「失效不删」预留了位置；本次小样本未触发（未构造同键冲突）。
- **删除是硬删除**：delete 后 Markdown 中旧 entry 块被移除，LanceDB 对应行同步消失，检索立即 0 命中。这与 Graphiti 的「失效不删、保留历史」是**相反策略**——需要区分场景：用户级删除（GDPR 类）本就该硬删；事实级更新才需要 deprecated 机制。EverOS 把两者分开了（delete API vs deprecated_by 字段），设计是清楚的。
- **观察项（不判定为缺陷）**：删除后 `entry_count: 2`，但文件内实际只剩 1 个 entry 块。计数未随删除重算。可能语义为「累计写入次数」而非「当前条目数」，需上游确认；此处仅如实记录。

### 2.5 Hindsight：评估后不部署（记录原因）

| 项 | 实测 |
|---|---|
| PyPI 发行物 | `hindsight-api-slim==1.3.0` 存在（与源码 tag 一致） |
| 依赖解析（uv dry-run） | **131 个包** |
| 是否含 torch/CUDA | **否**（torch 属 `local-ml` 可选 extra，base 不含） |
| 数据库 | `pg0-embedded==0.17.0`（嵌入式 PostgreSQL）+ asyncpg + pgvector 0.5.1 |
| 向量/推理 | onnxruntime 1.32.0（`local-onnx`） |
| 模型客户端 | anthropic / cohere / openai 等 |

判断：Hindsight 技术上可装（比预判轻，无 torch），但——
1. 仍**强依赖外部 LLM**，无凭据时同样无法产出真实成绩，装了也拿不到本轮要的东西；
2. `pg0-embedded` 首次启动需要下载 PostgreSQL 二进制，与 tiktoken 属**同一类无超时下载陷阱**，在本出口带宽下高风险；
3. 任务书明确「不盲目部署两套重服务」。

因此本轮只记录依赖事实与阻断原因，不执行安装。若后续需要，建议在**有真实模型凭据 + 充足带宽**的环境中一次性完成，预计 venv 数 GB 级。

---

## 3. M2 模型角色与端点契约

### 3.1 EverOS 的 LLM 角色（源码实测）

`LLMClients` 共 **12 个 task**，每个可独立配置模型与端点：

`boundary`（会话切分）、`language`（语种判定）、`episode`（事件抽取）、`summary`（摘要）、`atomic_fact`（原子事实）、`profile`（画像）、`reflection`（反思）、`agent_case`（案例）、`agent_skill`（技能）、`knowledge`（知识树）、`search`（检索辅助）、`decider`（多轮决策）

- 协议：OpenAI 兼容 `chat/completions`；`response_format_mode` 可选 `prompt_only` / 结构化模式
- 校验：每个 callsite 绑定固定 Pydantic Reply 模型，校验失败重试 `invalid_reply_retries` 次后抛错
- **embedding**：独立 Provider（OpenAI 兼容 `/embeddings`），**必需**才能启用 `vector_search` / `hybrid_search` / `agentic_search`
- **rerank**：`DeepInfra` 专用实现，需独立 key；**不支持本地 rerank**（源码仅见 DeepInfra 分支）

### 3.2 本环境授权状态（实测）

| 检查项 | 结果 |
|---|---|
| `hy4flash` 端点/凭据 | 不存在（环境变量无任何 hy4 相关项） |
| `DeepSeek-V4.1-Flash` 端点/凭据 | 不存在 |
| 任何通用模型网关 key | 不存在（仅有平台内部代理变量，非模型凭据） |
| embedding 本地方案 | 未配置；EverOS 无内置本地 embedding provider |

**结论：生成层 BLOCKED。** 按任务书，不以 mock 或规则模拟填补真实成绩，改为交付第 3.3 节测试包。

> 注意一个易混淆点：`onnxruntime` 出现在 Hindsight 依赖里，不等于「EverOS 可用本地 embedding」。EverOS 的 embedding 走 OpenAI 兼容 HTTP，本轮未验证任何本地 embedding 服务，故不能假设 embedding 可用。

### 3.3 交付：无秘密测试包（供 Z 执行）

**目标**：用已授权模型完成「同一输入的生成层对照」，产出与第 4 节检索对照可拼接的完整结果。

**所需授权（最小集）**：

| 项 | 值 | 说明 |
|---|---|---|
| 生成模型 | `hy4flash` 或 `DeepSeek-V4.1-Flash` | 二选一即可起步；两者都跑可做模型敏感性 |
| 端点形态 | OpenAI 兼容 `POST {base_url}/chat/completions` | EverOS 唯一支持的协议 |
| 需要的角色 | 先只配 `boundary` `language` `episode` `summary` `atomic_fact` | 其余 7 个可先复用同一模型 |
| embedding | **可选**，若要做 hybrid 对照则必需 | 无则只能 keyword，与第 4 节同条件 |
| rerank | 可选，DeepInfra 专用 | 不做 rerank 对照可省 |

**最小预算估算**（6 条记忆 / 6 查询 / 单轮）：

| 环节 | 调用次数（估） | 输入 token（估） | 输出 token（估） |
|---|---|---|---|
| boundary | 6 | ~3 K | ~0.3 K |
| episode + summary | 6–12 | ~6 K | ~2 K |
| atomic_fact | 6 | ~6 K | ~2 K |
| profile（可选） | 1 | ~2 K | ~1 K |
| 合计 | **约 20–30 次** | **约 17 K** | **约 5 K** |

按主流定价，单轮总成本在**分级钱级别**，适合先 dry-run 3 条再放开。

**报文结构**（供 Z 直接套用）：

```http
POST {base_url}/chat/completions
Authorization: Bearer ${MODEL_KEY}
Content-Type: application/json

{
  "model": "hy4flash",
  "messages": [{"role":"system","content":"<EverOS 自带 prompt，勿改>"},
               {"role":"user","content":"<待抽取会话文本>"}],
  "temperature": 0,
  "max_tokens": 2048,
  "response_format": {"type":"json_object"}
}
```

**执行约束（必须遵守）**：
1. 只在隔离 venv 内注入环境变量 `EVEROS_LLM__*`，不写配置文件、不落密钥到磁盘；
2. 先 `defer_extraction=false` + 单条记忆 dry-run，确认 200 与 Markdown 落盘，再放开 6 条；
3. 记录每次请求的 usage（prompt/completion tokens）与耗时，输出到独立 JSONL；
4. 失败即停，保留原始错误，不用重试掩盖 schema 不匹配；
5. 结果与第 4 节检索结果合并时，**必须分列「检索」「生成」两栏**，不得合成单一分数。

---

## 4. M3 LongMemEval 口径 + 同条件检索对照（本轮唯一真实成绩）

### 4.1 基准口径

| 项 | 结论 |
|---|---|
| 数据集 | `xiaowu0162/longmemeval-cleaned`（2025-09 清洗版，替代原版） |
| 许可 | 代码 MIT + **数据 MIT**（可商用，需署名） |
| split | `oracle` / `s_cleaned`（≈115 K token/题）/ `m_cleaned`（≈1.5 M token/题） |
| 官方评估 | 生成式问答 + 正确性判定；5 类能力 + 弃答 |
| **红线** | **`oracle` split 是「已知证据定位」用途，绝不能当完整 benchmark 成绩对外表述** |
| 本轮用法 | 未直接跑 LongMemEval（需真实生成模型，BLOCKED）；改用自建 6 条合成语料做**检索层**对照，LongMemEval 留作有模型后的入口 |

### 4.2 对照设计（可复现）

- 语料：6 条合成记忆（`corpus.json`），覆盖任务书点名的六类能力
- 查询：6 条，**5 条有答案 + 1 条无答案（负例）**
- 参数：`top_k = 5`；同一语料文本、同一查询、同一时间基准
- 两侧实现：
  - **EverOS**：真实引擎 + LanceDB FTS（tantivy BM25）+ jieba 分词，经 `POST /memory/search` 同款内部路径
  - **知遥**：`recall.js` 原版 BM25（自实现 bigram + 单字 + 英文词，含覆盖度/时间衰减/where/who 加权），`rebuild()` + `search()` 原样调用，仅注入合成 `allTurns()`

### 4.3 结果（`result_compare.json`）

| qid | 能力维度 | gold | 知遥 top1 | 知遥 r@5 | EverOS top1 | EverOS r@5 |
|---|---|---|---|---|---|---|
| q1 | 事实更新（杭州→成都） | m1 | m1 ✓ | 1.0 | m1 ✓ | 1.0 |
| q2 | 长距召回（双要素） | m1,m6 | m1 ✓ | **0.5** | m1 ✓ | **1.0** |
| q3 | 时间/关系 | m2 | m2 ✓ | 1.0 | m2 ✓ | 1.0 |
| q4 | 否定 | m3 | m3 ✓ | 1.0 | m3 ✓ | 1.0 |
| q5 | 假设 | m4 | m4 ✓ | 1.0 | m4 ✓ | 1.0 |
| q6 | **无答案** | — | m1（噪音） | — | m1（噪音） | — |

汇总：

| 指标 | 知遥 recall.js | EverOS 1.7.1 |
|---|---|---|
| top-1 命中率（5 题有答案） | **1.00（5/5）** | **1.00（5/5）** |
| recall@5 | **0.90** | **1.00** |
| 单查询延迟 | **0.07–1.47 ms** | **7.44–26.84 ms** |
| 进程 RSS | **47.6 MB** | **419 MB** |
| 索引规模 | 7 blocks（内存） | 6 episodes（LanceDB） |
| 落盘体积 | 0（纯内存/JSONL） | 2.0–3.9 MB / 6 条 |

**结果解读（严格限定）**：

1. **top-1 双方全对**，说明两份 BM25 实现都足以处理这类「单跳可命中」的中文短记忆。这一点对选型意义重大：**检索质量不是引入外部框架的理由**。
2. **唯一的 recall 差异在 q2（双要素长距）**：EverOS 把 `m6`（tokio）也召回进 top-5，知遥只召回 `m1`。原因是可解释的——EverOS 的 episode 文档由「多轮会话 + 模板」构成，单文档覆盖词更广；知遥的 block 以 4 轮为界切分，`m6` 独立成块且与查询词共现弱。这不是「算法优劣」，是**文档粒度策略差异**。
3. **延迟差 1–2 个数量级**：知遥是纯内存 JS 计算（7 个块），EverOS 走异步 IO + Arrow 反序列化 + Python 层。**但语料仅 6 条，此延迟绝不代表规模化表现**，只能说明「小规模下 EverOS 有固定开销」。
4. **RSS 差 8.8 倍**：EverOS 419 MB 主要来自 pyarrow / lancedb / pydantic 栈常驻；知遥 47.6 MB 含 Node 运行时基线。若要边缘部署，这是硬约束。
5. **双方都缺「无答案」判定**：q6 两边都返回低分噪音（0.2632 / 0.2714），而真命中分数在 1.0–11.6 区间。**这说明两者都有可用的分数区分度，但都没有阈值或弃答策略**。这是共同短板，也是知遥可以直接补的改进点（加阈值/弃答），无需引入外部框架。

> 诚实边界（必须随结果一起引用）：
> - 语料 6 条、查询 6 条，**统计意义极弱**，只能证明「机制可用、行为可解释」，不能外推为 benchmark 成绩；
> - 两侧分数**不可跨系统比较**（BM25 变体与归一化不同）；
> - EverOS 侧 episode 含 `<conversation>` 模板文本，对分数有影响（但对所有 episode 一致）；
> - 两侧语料粒度不同（原始块 vs 抽取后文档），属真实架构差异，非人为对齐。

---

## 5. M4 资源与维护成本

| 维度 | 知遥 recall.js | EverOS 1.7.1 | 备注 |
|---|---|---|---|
| 代码体积 | 372 行（单文件） | 5000+ 文件（monorepo 多子包） | — |
| 运行依赖 | 0（Node 内建） | 557 MB venv（lancedb+pyarrow+jieba） | 不含模型 |
| 外部服务 | 无 | 无（embedded sqlite+lancedb），但需 LLM | 有模型才能抽取 |
| 启动时间 | 即时 | 0.53–0.62 s（缓存就绪后） | 首启需下载 tiktoken |
| 常驻内存 | 47.6 MB | 419 MB | 小语料 |
| 写入延迟 | 即时 | 0.38–0.87 s/会话（含抽取） | 假替身，真实模型更慢 |
| 失败模式 | 无外部依赖，几乎不失败 | 严格 schema 校验、provider 缺失即报错 | 边界清晰但集成成本高 |
| 迁移成本 | 已有 | 需 LLM 凭据 + 语料重灌 + 索引重建 | — |
| 许可风险 | 自有 | Apache-2.0（低） | — |

---

## 6. M5 决策矩阵（六路比较）

| 路径 | 检索层可行性 | 生成层可行性 | 结论 | 理由 |
|---|---|---|---|---|
| **整体直接采用** | 可行 | 需 LLM | **否决** | 无实证质量优势（top-1 打平）；运维面 +557 MB、+419 MB RSS；需 12 个 LLM 角色对齐 |
| **源码抽取（摘取算法）** | 可行 | 不涉及 | **可选（低优先）** | 单列 BM25 逻辑本身不复杂，摘取收益低于自研成本 |
| **SDK 调用** | 可行 | 需 LLM | 暂缓 | 需常驻服务，与「本地优先」冲突 |
| **侧车（sidecar 服务）** | 可行 | 需 LLM | 暂缓 | 同 SDK，且增加进程与端口面 |
| **薄适配（协议层对齐）** | 可行 | 需 LLM | 暂缓 | 仅在确需 external memory API 时考虑 |
| **现有实现 + 机制补齐** | 已可 | 现有 | **推荐** | 见下 |

### 6.1 推荐方案：现有实现 + 三项机制补齐

本轮证据支持的不是「换引擎」，而是「补三处机制」：

| 机制 | 现状 | 建议 | 依据 |
|---|---|---|---|
| **来源追溯** | 知遥已有 block 级 who/where | 补 `parent_id` 式物理指针（指回原始 turn 批次） | EverOS 的 `parent_type/parent_id` 是可直接照抄的最小设计 |
| **无答案判定** | **缺失**（q6 返回噪音） | 加分数阈值 + 弃答分支 | 实测两侧分数区分度明显（噪音 0.26 vs 命中 1.0+），阈值可用 |
| **事实级失效不删** | 需确认 | 引入 `deprecated_by` 式软删除指针，与「用户级硬删」分离 | EverOS 已把两者分离，是正确范式 |

> 明确不做：不引入 LLM 抽取依赖、不引入 LanceDB/pyarrow、不切换权威存储。

### 6.2 若将来仍要整体采用（预案，不实施）

| 议题 | 方案要点 |
|---|---|
| 数据迁移 | Markdown 为源真值，可双向；先影子写入，比对两份检索结果，再切读 |
| 回退 | 保留原 `turns.jsonl` + recall.js 为一级回退；EverOS 侧仅作派生索引，删库即回退 |
| 唯一真源切换 | 切换前必须确认 Markdown 与 SQLite 一致性（`cascade` 健康、`index_rows` 与文件条目对账） |
| 人格/身份连续性 | EverOS 的 `profile` / `user_profile` 是独立抽取结果，与知遥人格资产**不同构**；切换会造成画像漂移，需先做画像映射与人工比对，不可直接替换 |
| 前置条件 | 真实模型凭据、embedding provider、PostgreSQL 或 LanceDB 运维能力 |

### 6.3 门禁判定

| 门禁 | 判定 | 依据 |
|---|---|---|
| **G0** 固定版本、许可、可复现 | **PASS** | EverOS 1.7.1 / lancedb 0.34.0 / Python 3.12.3 全固定；许可一手核实；复现包含哈希 |
| **G1** 原版真实运行证据 | **PASS** | 真实 server 监听 + `/health` 200 + 真实落盘 + 真实检索 + 真实删除；失败日志齐备（503/422/500） |
| **G2** 同条件对照、无伪排名 | **PASS（条件）** | 检索层真实同条件、含负例；**生成层 BLOCKED**，未用 mock 冒充；未做单一综合排名 |
| **G3** 资源成本 / 复用清单 / 决策矩阵 | **PASS** | 第 5、6 节；成本为实测值 |
| **G4** 无秘密、无生产改动 | **PASS** | 全程隔离 root、loopback、无生产访问；未落任何密钥；交付物已扫描无凭证 |

---

## 7. 遗留与下一步（需裁决）

1. **生成层对照**：需 ChatGPT 协调 Z 用授权模型执行第 3.3 节测试包；R 不自行扩权。
2. **LongMemEval 小样本**：有模型后建议先取 `oracle` 12–20 题做**链路验证**（明确标注「非 benchmark 成绩」），再评估是否上 `s_cleaned`。
3. **`entry_count` 语义**：删除后计数未重算，建议向上游确认是缺陷还是「累计写入次数」语义。
4. **无答案阈值**：建议知遥侧先做，成本最低、收益明确（q6 已暴露）。
5. **Hindsight**：本轮未部署；如需对比，建议单独排期并在充足带宽+凭据环境执行。

---

## 8. 附：本轮未做与不能做（防止误读）

| 项 | 状态 | 原因 |
|---|---|---|
| 真实模型语义抽取质量 | **BLOCKED** | R 无 hy4flash/DeepSeek 凭据 |
| embedding / hybrid / rerank 对照 | **BLOCKED** | 无 embedding provider、无 DeepInfra key |
| LongMemEval 正式成绩 | **未做** | 需真实生成模型；且 oracle 不可冒充完整 benchmark |
| LoCoMo 数据实验 | **不做** | CC BY-NC 4.0，非商用，仅研究参考 |
| 规模化性能 | **未做** | 语料仅 6 条，延迟/RSS 不可外推 |
| 生产改动 | **零** | R 侧边界 |
