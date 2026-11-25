> 说明：本文件为脱敏公开版，供 ChatGPT 阅读辅助。完整原件（含资料库稳定链接、原始日志、bundle）保留在 WorkBuddy「知遥AGI-R侧」资料库；不含任何凭证、私人记忆、真实生产配置或内部可定位链接。

# R-MEMCORE-OSS-REUSE-BENCHMARK-01｜忆核优秀开源系统源码复用、实测对标与融合选型·完整研究报告

日期：2026-11-25（北京时间）｜执行：WorkBuddy R（技术研究与隔离实验方）｜裁决：ChatGPT；实施由 K/E 按另行任务书执行
依据：《AI 工作流统一规则》V1.4；主人 2026-11-25 确认的先进技术融合式自主研发原则
性质：隔离研究、源码审查、可复现对照实验。**不修改正式研发基线、不部署、不启动 S3 正式开发、不获取秘密、不跨侧写入、不自派任务。**

---

## 0. 证据分层（读全文前必立规矩）

| 层 | 含义 | 本报告的对应内容 |
|---|---|---|
| **A 类·源码事实** | 从授权候选真实快照读出 | 知遥 S1 `86dd0b9`、S2 `08a81f1`、候选 `37dbbb30`/`d9f8f645` 的真实代码与 diff |
| **B 类·一手官方** | 官方仓库/README/LICENSE/release，核查日期 2026-11-25 | 五套开源项目的版本、许可、依赖、源码路径、机制 |
| **C 类·真实隔离运行** | 本任务在隔离目录真实执行、可复现 | EverOS v1.7.1 安装成功、`demo --plain`、CLI、`init` 真实通过；日志在 `evidence/runtime/` |
| **D 类·确定性 MOCK/机制复刻** | 规则模拟，**不是原版程序、不是真实模型** | 知遥 extractive 复刻（源码事实）；外部原版端到端同条件不可运行，故不写分数、不排名 |
| **BLOCKED** | 真实模型 `hy4flash`/`DeepSeek-V4.1-Flash` 无凭据与额度 | 完整 add→flush→search、真实语义摘要质量、G5 双模型；mock 不冒充 |

核心诚实声明：**没有任何一条结论声称"我们跑通了外部原版的真实模型记忆链路"**。EverOS 真正跑起来的是安装、CLI、demo 静态预览与初始化；需要 LLM/embedding 的完整链路因无授权凭据被阻断，已保留真实失败日志。

---

## 1. 执行摘要（结论先行）

1. **M0 基线核对无误（A）**：任务书给出的两个候选坐标在本地受控只读 repo 中真实存在且吻合。`37dbbb30`（Tree `9fc8ac7a`，完整 `9fc8ac7aa8d77a406af68400c881400395f8198c`）是"备份可靠性修复合入统一研发候选"的 merge 提交，仅改 `ops/` 四文件（+824/−27）；`d9f8f645`（Tree `47b599f8a8885061bd3a1a1b6d104abe9eed6c98`）以 `37dbbb30` 为父提交，仅新增 3 个测试工具文件（+739），与"C 报告称仅新增测试工具"一致。**不擅自认定新正式基线**。
2. **五库一手核查完成（B）**，当日状态见 §3.1。**关键事实**：`EverMind-AI/EverMemOS` 已永久重定向（repositories/1085086903）至 `EverMind-AI/EverOS`；五库均 Apache/MIT 宽松许可、均非归档、均活跃（Letta 最近推送 2026-10-18，略慢）。
3. **最有融合价值的两个对象已定位到源码级（B+C）**：
   - **EverOS v1.7.1**：本地优先、**Markdown 为唯一源真值 + SQLite/LanceDB 可重建索引**，与知遥"memv2 唯一真源 + 派生索引"架构哲学高度同构；双路召回（BM25 over `fact_tokens` + cosine ANN）；`extract/boundary/detection.py`（429 行）做主体边界与隔离；`reflection/orchestrator.py`（873 行）做离线巩固。**真实安装与 CLI/demo/init 已通过（C）**。
   - **Hindsight v1.3.0**：MIT，事实抽取 `fact_extraction.py`（**3901 行**）为 LLM 驱动 + 双时态 + 四路召回 + RRF + 决策重排；能力最强但**依赖 PostgreSQL + pgvector + 可选 torch**，重，且需 PostgreSQL 服务，隔离运行门槛高。
4. **真实隔离运行结果（C）**：EverOS 在 Python 3.12 隔离 venv + 腾讯镜像下 `pip install everos==1.7.1` **成功（exit=0）**；`everos demo --plain` **真实运行成功**（EXIT=0，3.98s，峰值 RSS 203,056 KB），产出 Markdown episode 源文件并完成召回；`everos init` 成功。**但**：`everos server start` 默认因 `[llm]` 未配置报 `LLMNotConfiguredError`；改用本地合成 mock 端点后，进程启动并完成 12 个 LLM task 解析、但 HTTP 端口最终未监听（日志停在 tracing ready），判为 **CONDITIONAL/BLOCKED**，不伪装成功。完整记忆链路的真实模型验证 **BLOCKED**。
5. **M3 对照坚持无伪排名（D）**：复用前序 11 类合成场景 + 新增 6 类缺口场景；知遥 extractive-v1 定量保真为硬要素 **10/12**、软语义 **9/10**、archive_only 0/11。**外部四家只做"证据支撑的定性能力覆盖矩阵"，不打分不排名**——因为同条件原版不可运行（EverOS 需 LLM/embedding、Hindsight 需 PostgreSQL/torch、Graphiti 需 Neo4j/LLM），用临时正则模拟只会测出"模拟实现质量"而非机制能力，违反 G3。
6. **基准研究（B）**：LongMemEval（ICLR 2025，MIT，500 题）、LoCoMo（ACL 2024，**CC BY-NC 4.0 非商用**）、PersonaMem v2（MIT）。**LoCoMo 的非商用许可是对知遥项目最关键的合规约束**，若未来引入须由主人/ChatGPT 裁决。选 LongMemEval 为后续小样本对标入口（MIT、官方 coreset 500 题、判据明确）。
7. **融合选型结论（M4）**：**不整套引入、不迁移权威存储**。优先"机制借鉴 + 薄适配"：① 交 E 侧的独立 PoC 首选 **EverOS 的 Markdown 源真值 + 可重建索引 + 双路召回机制**（与 memv2 同构，许可 Apache-2.0，无成本依赖可控）；② 次选 **Graphiti/Hindsight 的双时态"失效不删 + 来源链"机制**，并入 S3 P0 治理。替换底层存储或整套引入需另提迁移/回退/连续性方案交裁决，本侧不自行实施。

---

## 2. M0：基线、边界与可复用资产

### 2.1 候选坐标核对（A 类，全部本地只读 repo 实证）

| 项 | 任务书给定 | 本地核对结果 | 结论 |
|---|---|---|---|
| 统一条件研发候选 | `37dbbb30…` / Tree 短 `9fc8ac7a` / `cand/s2-e1-backup-integration-01` | commit 存在；Tree 完整 `9fc8ac7aa8d77a…`；merge 提交，父 `8131340`+`ec1e2a9`；仅改 `ops/backup-daily.sh` 等 4 文件 | ✓ 吻合 |
| 测试纠偏后继 | `d9f8f645…` / Tree `47b599f8a8885061bd3a1a1b6d104abe9eed6c98` / `cand/s2-g5-fixture-fix-01` | commit 存在；父提交即 `37dbbb30`；仅新增 `g5-judges.mjs`/`s2-live-models-fix01.mjs`/`s2-g5-fixture-fix.test.mjs` 3 文件 | ✓ 吻合，且确为测试工具 |
| 祖先关系 | 后继应建立在候选之上 | `git merge-base --is-ancestor 37dbbb30 d9f8f645` = true | ✓ 线性后继 |
| S1/S2 前序基线 | `86dd0b9` / `08a81f1` | 均存在 | ✓ 可复用 |

**边界执行**：全部操作在 `[本地隔离路径已省略]` 独立目录；只读取得候选；未改生产、桥接、默认模型、部署、备份或他侧工作区；未执行未审查的高风险代码（EverOS 只装官方 PyPI 包并跑官方 CLI/demo）。

### 2.2 可复用资产（不重复劳动）

- **前序 R 研究**：S3 治理报告（双时态/失效不删/来源追溯/可演化元数据/有界语义恢复）、智能压缩报告（`rollupTurns`/`buildRollup` 抽取式 `extractive-v1`、11 类合成场景、递归衰减结论）、E0/DSH 生态研究。
- **复用方式**：11 类合成场景 `dataset_v11.mjs` 直接复用（SHA256 `64947c12…`）；三策略实现 `comp_api.mjs` 复用（`49f8d580…`）；**未重跑千余项无关回归**。
- 门禁 G0 ✓。

---

## 3. M1：优先研究对象源码定位与许可核查

### 3.1 五库当日快照（B 类，核查日期 2026-11-25，通过 GitHub API 令牌只读）

| 项目 | 规范仓库 | 固定 tag/commit | 许可证 | Stars | 最近推送 | 归档 | 默认分支 |
|---|---|---|---|---|---|---|---|
| Hindsight | `vectorize-io/hindsight` | v1.3.0 / `937886199e` | **MIT** | 63,272 | 2026-11-24 | 否 | main |
| EverOS | `EverMind-AI/EverOS` | v1.7.1 / `f1fa76266e` | **Apache-2.0** | 13,998 | 2026-11-25 | 否 | main |
| EverMemOS | → 永久重定向至 EverOS | — | — | — | — | — | — |
| Mem0 | `mem0ai/mem0` | main（latest release `ts-v3.3.1`） | **Apache-2.0** | 69,446 | 2026-11-24 | 否 | main |
| Graphiti | `getzep/graphiti` | v0.32.1 / `b22ae9eb76` | **Apache-2.0** | 33,075 | 2026-11-25 | 否 | main |
| Letta | `letta-ai/letta` | 0.16.8 / `1131535716` | **Apache-2.0** | 25,538 | 2026-10-18 | 否 | main |

> 说明：Hindsight README 标注 License: MIT，GitHub API license.spdx_id 亦为 MIT；Hindsight 另有商业 Hindsight Cloud（闭源托管），**平台宣传的 benchmark 成绩不等于开源版本成绩**。Mem0 有 Apache-2.0 开源版与闭源托管平台，冲突裁决在付费层（README）。EverOS README 提供 LoCoMo/PersonaMem 成绩，均为厂商自报，不冒称独立复现。

### 3.2 运行依赖、资源与数据控制

| 项目 | 运行依赖 | Python/语言 | 模型/embedding | 最低实测资源 | 本地数据控制与导出 |
|---|---|---|---|---|---|
| **EverOS** | lancedb 0.34、pyarrow 26、aiosqlite、sqlalchemy 2.1、openai 2.54、fastapi 0.165、tiktoken | **≥3.12** | LLM 必需（boundary/episode/summary/atomic_fact/profile/reflection 等 12 task）；embedding 可选（缺则退化 keyword） | 实测 demo 峰值 RSS **203 MB**、3.98s | **Markdown 文件为源真值**，SQLite+LanceDB 为可重建索引；`cascade rebuild`；`init --root` 指定记忆根 |
| **Hindsight** | PostgreSQL + pgvector、SQLAlchemy<2.1、openai、numpy；可选 sentence-transformers/torch/llama-cpp/onnxruntime | ≥3.11（后端 Rust CLI + Python API） | 决策/重排/embedding 模型（可 HF 开放权重或托管） | 需 PG 服务，未实测；下载 tarball 217MB 超低速出口，BLOCKED | PostgreSQL 事实/观察 + alembic 迁移；bank 隔离 |
| **Graphiti** | Neo4j（图库）、LLM、embedding | Python | LLM + embedding | 需 Neo4j，未实测 | episodes 原始数据 + 图；`group_id` 隔离 |
| **Mem0** | 向量库（Qdrant 等）、LLM、embedding | Python/TS | 默认 `gpt-5-mini` + `text-embedding-3-small` | 未实测 | `user_id/agent_id/run_id` 过滤；平台迁移指南 |
| **Letta** | 数据库、LLM | Python | LLM | 未实测 | 全状态持久化，evict 消息仍可检索 |

### 3.3 源码级清单（G1：文件路径 → 类/函数 → 调用链 → 依赖闭包 → 可复用范围 → 拟替换模块 → 适配点 → 许可义务）

#### (1) EverOS v1.7.1（重点，Apache-2.0）
| 文件路径（一手读取） | 关键实现/职责 | 调用链位置 | 可复用范围 | 拟替换/补强知遥模块 | 适配点 | 许可义务 |
|---|---|---|---|---|---|---|
| `src/everos/memory/search/recall/atomic_fact.py`（185 行，已读） | `AtomicFactRecaller`：BM25 over `fact_tokens` + cosine ANN；`facts_for_episodes` 把原子事实挂回父 episode；**dual parent_id 策略**（新 `episode_entry_id`、旧 `memcell_id`，无需回填） | 检索 → 召回 → `FactCandidate` → HYBRID 组装 | **机制复用（高）**：双 parent_id 兼容新旧数据、稀疏+稠密双路 | 补强忆核"派生索引重建 + 事实挂接来源" | 需把 SQLite/LanceDB 索引映射到 memv2 派生视图；不改真源 | Apache-2.0：保留版权/许可声明，衍生作品需同许可声明 |
| `src/everos/memory/extract/boundary/detection.py`（429 行，已读） | 对话边界检测 + `quarantine` 隔离 | ingest → boundary → extract | **机制复用（高）**：主体/会话边界判定 | 补强 `detectSubject` 的最小正则（SELF/OTHER/UNKNOWN） | 需本地化中文规则；不可引入 LLM 依赖时可裁剪 | 同上 |
| `src/everos/memory/reflection/orchestrator.py`（873 行，已读） | 离线反思/巩固编排 | flush → cascade → reflection | **机制复用（中）**：持续巩固、记忆演化 | 补强 S3"持续巩固、反思不引入未证实事实" | 反思需模型；本地零模型场景需降级 | 同上 |
| `src/everos/infra/cascade/_projections/user_profile.py`（34 行，已读） | Markdown → 用户画像投影 | cascade 投影链 | 机制复用（中） | 画像/关系结构化槽 | 与心核边界隔离，不改人格卡 | 同上 |
| `src/everos/infra/persistence/markdown/_base/frontmatter.py`（441 行，已读） | Markdown frontmatter 解析/写入/路径安全 | 持久化基础 | **直接借鉴（高）**：源真值文件格式 | 派生索引可重建的落盘格式参考 | 需与 memv2 schema 对齐 | 同上 |
| `src/everos/entrypoints/api/routes/{memorize,search,delete,purge,knowledge,health}.py`（已列） | `/api/v{1,2}` 的 add/flush/search/delete/health | FastAPI 路由 | 接口参考（中） | E 侧 PoC 接口形状 | 默认 loopback、无内置 auth（安全注意） | 同上 |

**依赖闭包**：`everos==1.7.1` → lancedb→pyarrow；aiosqlite/sqlalchemy/sqlmodel；openai SDK；fastapi/uvicorn；tiktoken。全部经腾讯镜像可解析（实测）。

#### (2) Hindsight v1.3.0（重点，MIT）
| 文件路径（一手读取/树定位） | 关键实现/职责 | 可复用范围 | 拟替换/补强 | 适配点 | 许可义务 |
|---|---|---|---|---|---|
| `hindsight-api-slim/hindsight_api/engine/retain/fact_extraction.py`（**3901 行**，已读头部） | LLM 抽取语义事实/实体/时间；`LLMConfig`、Pydantic schema、reask/sanitize/zero-shot 标签 | **机制复用（中高）**：分类型事实抽取、时间信息 | 补强知遥"软语义槽/关系槽" | LLM 驱动，与"模型无关/不增成本"冲突 → 只借结构不引代码 | MIT：保留版权/许可声明 |
| `.../engine/retain/fact_storage.py`（575 行） | 事实入库/更新 | 机制复用（中） | 冲突消解/失效不删 | 需 PG | 同上 |
| `.../engine/memories/pg/retain.py`（551 行）、`.../memories/pg/recall.py` | PostgreSQL 事实/观察存储与召回 | 参考（低，绑定 PG） | — | — | 同上 |
| `.../engine/reflect/{agent,delta_ops}.py` | 可追溯反思/增量操作 | **机制复用（高）**：溯源反思 | 补强 S3 语义恢复的可追溯性 | 需模型 | 同上 |
| `.../engine/search/{reranking,decision_reranker}.py`（599/438 行）、`decision_rerank_interface.py` | RRF + 决策重排 | 机制复用（中） | 检索排序 | 需重排模型 | 同上 |
| alembic `versions/*bi-temporal*` | valid_at/invalid_at、实体维护队列 | 机制复用（高） | 双时态治理 | 概念借鉴，不引迁移脚本 | 同上 |

#### (3) 补充三库（定位，不要求完整部署）
- **Graphiti v0.32.1**（Apache-2.0）：`graphiti_core/`（183 文件）、`mcp_server/`（60）、`tests/evals/data/longmemeval_data/`（内置 LongMemEval 数据）。README 确认**双时态**（`valid_at`/`invalid_at`，事实失效不删、保留完整历史）、episode 溯源、混合检索（semantic+keyword+graph）。依赖 Neo4j + LLM + embedding。
- **Mem0**（Apache-2.0）：README 确认 v3 改 **single-pass ADD-only**（无 UPDATE/DELETE），默认 `gpt-5-mini` + `text-embedding-3-small`；`user_id/agent_id/run_id` 过滤；有 graph 能力与闭源 Platform。与 memv2 高度重叠 → 暂缓。
- **Letta 0.16.8**（Apache-2.0）：`letta/`（564）、`tests/`（312）、`alembic/`（170）。全状态持久化、core/recall/archival 三层、compaction 后旧消息可检索（前序研究）。整框架 ≠ memv2 → 暂缓。

门禁 G1 ✓（有源码路径、调用链、依赖、可复用清单，非仅观点）。

---

## 4. M2：实际运行与适配可行性（G2）

### 4.1 真实隔离运行记录（C 类，命令/版本/退出码/RSS/磁盘/延迟）

| # | 命令（隔离目录执行） | 结果 | 版本 | 关键指标 |
|---|---|---|---|---|
| 1 | `uv venv --python /usr/bin/python3.12 venvs/everos312` | **exit=0** | Python 3.12.3 | 隔离环境，无全局安装 |
| 2 | `uv pip install everos==1.7.1`（腾讯镜像） | **exit=0** | everos 1.7.1 | 装 40+ 依赖；lancedb 0.34.0、pyarrow 26.0.1、openai 2.54.0 |
| 3 | `everos --help` / `demo --help` / `init --help` / `server start --help` | **exit=0** | CLI 正常 | 5 命令组，健康检查/路由可列出 |
| 4 | `/usr/bin/time everos demo --plain`（timeout 60） | **EXIT=0** | — | **3.98s，峰值 RSS 203,056 KB**；产出 `episode-2026-06-20.md` 并完成召回 |
| 5 | `everos init --root <隔离目录>` | **exit=0** | — | 生成 `everos.toml` + `ome.toml` |
| 6 | `everos server start --root …`（默认配置） | **BLOCKED** | — | `LLMNotConfiguredError`：task boundary 的 api_key/base_url 缺失；Application startup failed |
| 7 | 同上 + 本地合成 mock OpenAI 端点（12 task 环境变量指向 mock） | **CONDITIONAL** | — | 12 个 LLM task 解析成功；multimodal/embedding/rerank 能力告警；进程存活但 **HTTP 18000 端口未监听**，日志停在 `tracing_score_sink_ready` |

日志原始文件：`evidence/install_logs/everos_py312_uv_install.txt`、`evidence/runtime/everos_demo_plain.txt`、`evidence/runtime/everos_init.txt`、`evidence/runtime/everos_server{2,3}.log`、`evidence/runtime/mock_llm.log`。

**磁盘/依赖度量**：隔离 venv 为本次新增磁盘占用；未写入共享工作树；未全局安装。Hindsight 完整 tarball 217,092,469 bytes 在 180s 内仅下载约 6.35MB（低速出口）→ tarball 获取 BLOCKED，改用 API 单文件读取（已获核心源码）。

### 4.2 五种接入方式比较（任务书要求）

| 方式 | 适用对象 | 可行性 | 成本/风险 | 判断 |
|---|---|---|---|---|
| 本地库/SDK 直接引入 | EverOS | 技术上可行（已装成功） | **违反 memv2 唯一真源**：会成为第二套长期记忆系统；需 LLM/embedding | **否决整引**，仅留机制 |
| 薄适配 | 两者 | 可行 | 接口层包装，不改真源 | **推荐**（E 侧 PoC 首选接口层） |
| 独立进程/服务 | EverOS | 部分可行（server 启动需 LLM；默认 loopback 无 auth） | 新增常驻服务/数据库/费用，违反"集成前不新增永久服务" | **暂缓** |
| 源码抽取 | EverOS/Hindsight 关键模块 | **可行**（已读源码） | Apache/MIT 允许，需保留许可；适配成本高 | **推荐机制抽取**（非整文件拷贝） |
| 局部替换 | 忆核摘要冲突消解/来源链 | 可行 | 改核心需 K/C 走正式流程 | **条件推荐**：仅机制，交 K/E 按新任务书 |

### 4.3 真实模型实验状态

- 核实 `hy4flash`、`DeepSeek-V4.1-Flash`：本次无凭据与额度，**先 dry-run 的前置即不满足** → **BLOCKED**。未替换未授权模型、未读生产账号池、未额外付费。
- 本地合成 mock 仅验证 EverOS 服务链路与 API 结构，**明确标注 MOCK，不计入真实语义成绩**。
- 规则模拟（D 类）、mock、真实运行（C 类）三类结果在全部交付物中分开统计。

门禁 G2 ✓（有真实运行与完整失败日志；未实证项如实 BLOCKED）。

---

## 5. M3：同条件对照与质量指标（G3）

### 5.1 冻结样本与判据

- 复用前序 **11 类合成场景**（共同经历、称呼关系、人物主体、时间顺序、否定、数值、未完成计划、冲突更新、引用假设、情绪语境、工作私聊隔离）。
- 新增 **6 类缺口场景**（`synth/gap6.mjs`）：A1 共同经历/关系/情绪 + 主体/假设/否定/引用区分；A2 事实时间/更新/冲突消解（普通纠错 vs 受保护确认 vs 跨会话长期）；A3 原文归档/长距召回/同义恢复/来源追溯/无证据不编造；A4 两轮递归压缩（硬/软保真、旧值复活、幻觉、错误归因）；A5 未完成计划/持续巩固/反思；A6 作用域隔离/跨会话长期事实/删除撤回残留/导出重建。
- 全部虚构主体、虚构事实、零私人数据、零主人真实记忆。

### 5.2 知遥 extractive-v1 定量保真（A 类源码事实复刻，可复现）

| 指标 | 本轮结果 | 说明 |
|---|---|---|
| 硬要素保留 | **10/12** | 完整性校验保护对象；本轮 token 预算/匹配口径与前序报告（12/12）略有差异，以本次 `P_ossbench_result.json` 为准 |
| 软语义保留 | **9/10** | 关系/时序/计划状态/情绪，超预算时无硬保护 |
| archive_only | **0/11** | 本轮样本未触发完整性失败降级 |
| 递归 2 轮 | 引用前序证据 **硬 4/12、软 1/10** | 本轮不重复跑，复用前序 PoC 结论 |

### 5.3 反伪排名声明与外部定性覆盖矩阵

**为什么不打分排名**：同条件原版不可运行（EverOS 需 LLM/embedding；Hindsight 需 PostgreSQL+pgvector+torch；Graphiti 需 Neo4j+LLM；Mem0/Letta 需 LLM+向量库）。若用临时正则模拟原版，测出的是"模拟实现质量"而非机制能力。首版曾产出 `everos 13 / graphiti 5` 之类的分数，经审查发现由正则强弱决定，**已废弃该结果，改为证据支撑的定性矩阵**（详见 `03_对照矩阵.md` 与 `P_ossbench_result.json` 的 `antiPseudoRanking` 字段）。这是执行 G3"不给伪排名、模拟与真实分离"的必要修正。

### 5.4 基准数据集研究（B 类，选小样本入口）

| 基准 | 来源/许可 | 规模/判据 | 对知遥的可用性 |
|---|---|---|---|
| **LongMemEval** | `xiaowu0162/LongMemEval`，ICLR 2025，**MIT**；数据 HF `xiaowu0162/longmemeval-cleaned` | 500 题，5 类能力 + 弃答（`_abs`）；S≈115K token、M≈1.5M token；LLM-as-judge（默认 gpt-4o-mini）；2025-09 coreset | **选为后续小样本入口**：MIT 友好、判据明确、oracle 版只含证据会话便于隔离 |
| **LoCoMo** | `snap-research/locomo`，ACL 2024，**CC BY-NC 4.0** | 10 对话/7512 QA；QA F1、事件摘要 ROUGE/FactScore、多模态 BLEU/MMRelevance | **许可红线**：非商用，知遥若商业化引入须主人/ChatGPT 裁决；本轮只研究不引入数据 |
| **PersonaMem v2** | `bowen-upenn/PersonaMem-v2`，MIT，arXiv:2512.06688 | 隐式偏好/人格一致性 | 与心核/人格连续性相关，作第二阶段候选 |

> 本轮**不要求全量跑榜**；小样本结果不冒称完整公开榜单成绩。基准成绩（Hindsight/EverOS 宣称的 LongMemEval/LoCoMo SOTA）属厂商自报/第三方复现，非本侧实测。

门禁 G3 ✓（评测可复现、同条件限制明确、模拟与真实分离、无伪排名）。

---

## 6. M4：逐模块裁决矩阵与交接（G4）

### 6.1 裁决矩阵（现有知遥 / 外部候选 / 处置）

| 模块 | 知遥现状(A) | 外部候选(B) | 裁决 | 实测收益/证据 | 接入成本 | 长期维护 | 风险 | 真源/迁移约束 | 证据强度 |
|---|---|---|---|---|---|---|---|---|---|
| 结构化长期记忆真源 | memv2 唯一真源，纠错链+tombstone 实证 | EverOS Markdown 源真值；Mem0 ADD-only | **保留现状** | S3 已实证；无替换必要 | 0 | 低 | 换源破坏连续性 |  authoritative 不可并行 | A+C |
| 派生索引可重建 | 派生索引存在 | EverOS cascade rebuild + dual parent_id | **源码融合（机制）** | 新旧 ID 兼容设计 | 中 | 中 | 索引漂移 | 不改真源，可重建 | B |
| 冲突消解/失效不删 | 摘要层无 supersede；memv2 有纠错 | Graphiti/Hindsight 双时态 + 失效不删 | **适配改造（交 E PoC）** | 机制清晰、许可宽松 | 中 | 中 | 需与 memv2 语义对齐 | 派生元数据，不动事实真源 | A+B |
| 来源追溯 | 区间/版本/主体级 provenance | EverOS episode md；Hindsight reflect；Graphiti episode | **薄适配** | 句子级溯源是缺口 | 低-中 | 中 | 格式兼容 | 派生视图增强 | A+B |
| 主体/作用域隔离 | detectSubject 最小正则 + scope | EverOS boundary/quarantine(429行) | **源码融合（机制）** | 中文需本地化 | 中 | 中 | 人格卡边界 | 不得接管心核 | A+B |
| 语义/关系摘要 | 抽取式，无语义槽 | Hindsight fact_extraction(3901行) LLM 驱动 | **暂缓（需模型授权）** | 真实语义质量 BLOCKED | 高 | 高 | 需 LLM，成本/模型无关冲突 | 不改真源 | B+BLOCKED |
| 双路召回 | memv2 结构化检索 | EverOS BM25+ANN | **适配改造（交 E PoC）** | hybrid 无 LLM 0.27s（厂商自报） | 中 | 中 | embedding 可选 | 派生索引 | B |
| 递归防失真 | 无防护（2轮硬 4/12） | MMPO 不确定性门禁（前序研究） | **必要自研（本地零模型）** | 前序 PoC 实证 | 低 | 低 | 无 | 摘要层增强 | A+C |
| 整套框架引入 | — | Letta/Hindsight/Graphiti/Mem0 | **否决** | 均为第二真源/需重依赖 | 极高 | 高 | 破坏唯一真源/模型无关/零成本 | 需另提迁移方案 | B |
| 真实模型语义验证 | 未做 | — | **BLOCKED** | 无 hy4flash/DeepSeek 凭据 | — | — | 不得替换未授权模型 | — | BLOCKED |

### 6.2 交 E 侧独立适配 PoC 的 1–2 个组件（精确接口）

**组件 1（首选）：EverOS 式"Markdown 源真值 + 可重建索引 + 双路召回"薄适配**
- 接口草案（隔离 PoC，不改 memv2）：
  - `deriveIndex(memv2Facts) → {records, version, sourceIds}`：从 memv2 事实生成**派生索引**（不是新真源），带版本与来源区间。
  - `recall(query, {scope, subject, topK, method: 'bm25'|'ann'|'hybrid'}) → [{id, score, sourceId}]`：双路召回，返回来源 ID 指回 memv2。
  - `rebuild(fromFacts) → {ok, count}`：索引可从真源全量重建；删除/撤回后重建不得残留。
- 验收：删除某事实后 rebuild，索引与摘要中无残留；合法长期事实跨会话可读；工作/私聊 scope 隔离。
- 许可：Apache-2.0，只抽取机制与接口形状，不整体拷贝代码；保留许可声明。

**组件 2（次选）：双时态"失效不删 + 来源链"元数据适配**
- 接口草案：
  - `supersede(factId, newFact, {reason, source, at})`：旧值标 `invalid_at` 不删，不作为当前事实。
  - `currentFacts(scope, subject, atTime) → [...]`：按时间点取当前值；历史可查。
  - `provenance(factId) → {episodeId, turnIds}`：句子级来源链。
- 验收：普通纠错、受保护确认更新、跨会话长期事实三类分别通过；旧值不复活；反思不引入未证实事实。

### 6.3 给 K 侧的 S3 调整建议（不越界开发）

| 优先级 | 建议 | 依据 | 立即/后续 |
|---|---|---|---|
| P0-1 | 在 S3 可演化事实元数据中纳入 `invalid_at`/`superseded_by`/`source` 字段，实现"失效不删" | 双时态机制（Graphiti/Hindsight）+ memv2 已有纠错链 | **立即缺口** |
| P0-2 | 摘要层加冲突消解（识别"不对/改成/记错/撤回"，旧值标 superseded） | 本轮缺口 A2；前序报告 G1 | **立即缺口** |
| P1-1 | 句子级 provenance 落到派生索引（episodeId + turnIds） | EverOS/Hindsight 机制 | 后续阶段 |
| P1-2 | 软语义槽（关系/未完成状态/情绪/假设标记）结构化 | 本轮缺口 A1/A5 | 后续阶段 |
| P1-3 | 递归再摘要前强制回原文取证（coveredIds/fromAt/toAt） | 前序递归失真 4/12 | 后续阶段 |
| P2 | 作用域/私有域显式隔离增强（detectSubject → 规则集） | 本轮缺口 A6 + EverOS boundary | 后续阶段 |
| 红线 | 语义模型摘要（G2）、整套引入、第二真源、LoCoMo 数据商用 | 授权/许可/架构 | 需主人/ChatGPT 裁决 |

**若建议整套引入或权威存储迁移**：本侧结论是**不建议**，故无需迁移方案；如 ChatGPT 另行裁决迁移，须由 K/C 提交完整迁移、回退与连续性验证方案，本侧不自行实施。

门禁 G4 ✓（有可执行融合/替换建议、成本、连续性与回退设计）。

---

## 7. G0–G5 门禁汇总

| 门禁 | 判定 | 依据/局限 |
|---|---|---|
| G0 基线/版本/许可/来源可核实 | **PASS** | 候选坐标本地实证；五库 API + README/LICENSE 核查 |
| G1 源码路径/调用链/依赖/可复用清单 | **PASS** | §3.3 逐文件清单；源码真实读取 |
| G2 真实运行或独立组件实证、失败日志齐备 | **PASS（条件）** | EverOS 安装/CLI/demo/init 真实通过；server 完整链路 CONDITIONAL/BLOCKED，日志齐备 |
| G3 对照可复现、同条件限制明确、模拟与真实分离 | **PASS** | 11+6 场景、无伪排名、分层声明；真实模型 BLOCKED |
| G4 可执行融合/替换建议、资源预算、连续性/回退 | **PASS** | §6 裁决矩阵 + 2 个 E 侧 PoC 接口 + K 侧建议 |
| G5 隔离与交付合规、生产零变更 | **PASS** | 独立目录、合成数据、无秘密/无费用/无跨侧写入/无生产变更 |

研究任务按实际证据条件交付；真实模型项如实 BLOCKED，不为凑全绿扩大范围。

---

## 8. 交付物清单与复现指南

- 01 完整研究报告（本文）
- 02 源码定位与许可证矩阵
- 03 对照矩阵（6 系统 × 8 维度证据）
- 04 PoC 与评测复现材料（四件套 + SHA256）
- 05 精简报告（约 1300 字）
- 06 接收指南（E/K 侧）
- 07 R 侧进度记录

复现：`cd <隔离目录>/poc_ossbench && node poc.mjs`；校验 `sha256sum -c evidence/SHA256.txt`。EverOS 真实运行步骤见 `04` 与 `evidence/runtime/` 原始日志。

---

## 9. 待 ChatGPT 裁决（R 不自启下一轮）

1. 是否批准以组件 1（EverOS 式派生索引/双路召回）为 E 侧独立适配 PoC 的方向？
2. 是否确认"整套引入 + 权威存储迁移"为否决/需另提方案？
3. LoCoMo 为 CC BY-NC 4.0，是否维持"只研究不引入数据"？LongMemEval 是否定为后续小样本入口？
4. 是否批准模型凭据/额度，以便把 M3 定性矩阵升级为真实模型端到端对照？
5. 是否确认 G1/G3 双时态与冲突消解并入 S3 现有 P0/P1，而非另起炉灶？

> 本侧不改基线、不自启下一轮、不自派任务；裁决后交 K/E 实施、C 核验。全程生产零变更、无凭证暴露、无费用产生、未跨侧写入资料库。
