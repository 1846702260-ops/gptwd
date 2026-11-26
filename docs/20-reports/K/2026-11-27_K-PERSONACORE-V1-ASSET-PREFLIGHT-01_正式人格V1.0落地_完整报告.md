# K-PERSONACORE-V1-ASSET-PREFLIGHT-01 完整研发报告

- 任务：`K-PERSONACORE-V1-ASSET-PREFLIGHT-01`（任务书 V1.1，含正式人格卡）
- 执行侧：K｜接收验收：C｜签发：G
- 日期：2026-11-27（北京时间）
- 基线：`release/z5.2.0` HEAD `136aba5b5e062ab4ef171caa13c3f988214a9941`，Tree `d02b9dde6bdb134129d2d403c76789622ffce34a`，父 `ee73e08f`
- 候选：分支 `feat/personacore-v1-baseline-01`，HEAD `afbf09db6bfbe22c03aeb64c8d1322b74e9da427`，Tree `dfffcff3f574835cb2593ae104f657e49bda5611`，父 `136aba5b`
- 结论：**CONDITIONAL PASS**（资产/接线/确定性验证通过；真实模型 12 组场景 BLOCKED）

---

## 一、M0 资产接收

### 1.1 收到什么

| 项 | 实际 | 说明 |
|---|---|---|
| 任务书 V1.1 | 22152 字节，SHA256 `d74eda7e…6cfe3` | 含附录 A 全文 |
| 附录 A 正文 | 7535 字节，SHA256 `d14298b4…055a` | 与任务书给定值**逐字节一致** |
| 结构 | 5 章（01–05）、60 个非空段、2723 字符 | 首段"### 陈知遥｜完整人格卡"，末段"说明：以上为…" |
| DOCX 二进制 | **未收到** | 任务书宣称 41951 字节／`a056c42f…8b5e`，本轮无法核验，如实标注 |

### 1.2 资产分层

- **源设计（治理与验收用）**：`assets/陈知遥_完整人格卡_V1.0_附录A正文.md`，只读，不进运行时路径。
- **运行资产（实际注入）**：`packages/companion/role.chenzhiyao.md`，改写为 V1.0 受控表达，frontmatter 指回源版本与源哈希。
- **不进提示词**：文档性质说明、"暂不启动开发"、任务书正文、哈希、实施注释。`role.js` 注入前剥离整段 frontmatter，测试 G 断言哈希值不出现在 system prompt。

### 1.3 版本标识

| 版本 | 用途 | 状态 |
|---|---|---|
| `chenzhiyao-v1.0` | 正式 V1.0 运行卡 | 当前生效 |
| `chenzhiyao-v1` | P1A 过渡卡 | 历史，仍可从 git 取回，不再作为当前人格 |
| `role.md`（无版本） | 旧设定（鲸鱼娘·大肥鱼） | 历史，校验必判不合规 |

三者仅靠文件与 frontmatter 区分，**不靠"都叫陈知遥"判断**。

---

## 二、M1 现有心核盘点（只读）

### 2.1 加载与注入真实链路

| 环节 | 位置 | 行为 |
|---|---|---|
| 配置默认值 | `src/index.js:822` | `roleCard: rel(cfg.roleCard, 'role.chenzhiyao.md')` |
| 配置校验 | `src/index.js:610-615` | `roleCard` 非字符串/空串 → 配置拒绝段（不给垃圾值穿透） |
| 角色卡实例 | `src/index.js:1423` | `new Role({ cardPath, maxChars })` |
| 启动体检 | `src/index.js:1426` | `role.lint()`，不通过只告警不阻断 |
| 人格系统 | `src/index.js:2104-2118` | `createPersonaSystem({ dir, cardPath, memv2, identity })` |
| 卡校验 | `src/index.js:2123` | `core.validate()`，失败告警不阻断 |
| 提示词装配 | `src/brain.js:1098` | `role.build({ situation, memory, pending, recent, past, selfNote, platform, model })` |
| 当下状态 | `src/brain.js:1093` → `persona.promptBlock()` | 自我模型 + 情绪，只放"当下"，不重复人格卡 |
| 身份锁 | `src/role.js:38-47,135` | 末尾硬约束，姓名/称呼取自卡 frontmatter |
| 工作档隔离 | `src/workspace/persona.js:175` | `mode==='work'` 直接返回，关系/画像/world 全不装 |
| 外部渠道 | `src/workspace/persona.js:140` | `modeLineOf()`，QQ 恒陪聊档，只取模式行 |

注入顺序固定：处境 → 记忆 → 待办 → 最近对话 → 往事 → 当下状态 → 基本规矩 → 人格卡 → 平台说明 → 身份锁。人格卡在基本规矩之后、身份锁之前——"定义"在中段，"最后一句话"留给身份。

### 2.2 六项能力状态

| 能力 | 实现 | 状态 | 证据 |
|---|---|---|---|
| 人格版本/快照/回滚 | `persona/core-persona.js` | 已实现已接线 | `snapshot/rollback/listVersions`，回滚前自动快照 |
| 核心层保护 | `persona/self-model.js:116-122` | 已实现已接线 | `CORE_FIELDS` 变更需 `ownerConfirm`，否则抛 `owner_confirm_required` |
| 情绪（11 维、有据） | `persona/emotion.js` | 已实现已接线 | 无 `evidence` 直接 `no_evidence` 拒绝；半衰期回落 |
| 自我模型（七问/兴趣/能力三态） | `persona/self-model.js` | 已实现已接线 | `answer(who/capabilities/…/improvements)`，能力 `ready/limited/absent` |
| 反思与成长记录 | `persona/reflection.js` | 已实现已接线 | 失败→弱点/改进项，写 `reflections.jsonl`，另写 memv2 经历 |
| 关系与经历 | `persona/relations.js` | 已实现已接线 | `recall/correct/forget/rememberExperience`，全部走 memv2 |
| 主动策略 | `persona/proactive-policy.js` | 已实现，**通道未接** | 四道闸 + 静默时段；发送通道仍 `mock` |
| 社交 | `persona/social/*` | 已实现，**外发未接** | 白名单/审批/审计齐，`MockChannel` 为默认通道 |
| 成长日记 | `src/diary.js:196` | 已实现已接线 | 读 `cfg.roleCard` 生成，是记忆的可读视图，不另建真源 |

### 2.3 发现的问题（本轮修）

| 编号 | 问题 | 证据 | 处理 |
|---|---|---|---|
| P-1 | 身份版本硬编码 `chenzhiyao-v1` | `index.js` 原 2101 行 | 改为从卡 frontmatter 读 |
| P-2 | 否定句被体检误报 | 实测 `lint() → ok:false`，命中行"不自称 AI 小助手…" | 按句+否定词判定 |
| P-3 | 旧卡/缺卡只报"校验未通过"，未明示"当前不是正式人格" | 启动日志实测 | 增加显式降级说明（缺卡/旧卡分文案） |
| P-4 | 源-运行无哈希关联，无法反查正式版 | frontmatter 无源哈希 | 增加 `persona_source_*` 并校验格式 |

---

## 三、M2 最小实现

### 3.1 改动清单（8 文件，+171/−144）

| 文件 | 改动 | 原因 |
|---|---|---|
| `role.chenzhiyao.md` | 换为 V1.0 运行卡（2968 字） | 正式人格落地 |
| `src/index.js` | 身份三元组从卡读取；启动日志报版本+卡哈希+源哈希；旧卡/缺卡明示 | 消除双源、可追溯、不静默冒充 |
| `src/persona/core-persona.js` | `current()` 暴露 source 三元组；`validate()` 校源哈希 | 治理可核验 |
| `src/persona/self-model.js` | 无种子时版本留空 | 不再写死旧版本 |
| `src/role.js` | 否定句感知的客服腔检测 | 修误报，保留有效校验 |
| `test/persona-system/personacore-v1.test.mjs` | 新增 11 项 | 定向验收 |
| `test/persona-system/{chenzhiyao-persona,injection}.test.mjs` | 4 处断言更新到 V1.0 口径 | 基线资产变更 |
| `test/auth-v2/c-persona-load-e2e.mjs` | 版本/源哈希断言；新增启动日志溯源；P2.3 加"非正式人格"明示 | 真实加载链 |

### 3.2 运行卡如何做到"不失真"

- 五章逐条对应（身份与核心／关系与日常／自主社交责任／记忆成长治理／核心宣言），每章保留源文的判断句，不新增性格设定。
- 五项核心锚点、核心宣言**原句照录**（"成长不意味着忘记来处"等），因为它们承担身份稳定性。
- 删除的是**文档状态**（"暂不启动开发""不构成开发任务书"）与页眉页脚式元信息——这些若进提示词，模型会把"暂不开发"当成对自己的指令。
- 未搬运 P1A 的"场景示例"与口头禅清单：任务书 4.3 明确禁止固定答句、按场景硬编码人格；语气节奏由 `role.js` 的 BASE_RULES 承担（短句是节奏不是上限、不总结升华、不刷问句）。

### 3.3 三层治理落点

| 层 | 机制 | 是否新建 |
|---|---|---|
| 核心层 | 卡文件 + git 历史 + `snapshot/rollback`；`setIdentity` 核心字段需 `ownerConfirm` | 复用 |
| 协商层 | 卡变更走代码评审 + 交付验收；无面向主人的审批 UI | 复用，缺口记录 |
| 成长层 | 兴趣权重、反思、能力记录、成长日记（均为 memv2 派生视图） | 复用 |

---

## 四、M3 验证

| 套件 | 结果 | 说明 |
|---|---|---|
| 人格全量 `persona-system + persona + identity` | 179/179 | 基线 168 + 新增 11 |
| 加载链 e2e（真实 `startCompanion`） | 10/10 | 含 P1.4 启动日志溯源、P2.3 非正式明示 |
| 相邻 `web-v1/closure + reply-budget` | 31/31 | 角色卡闭合与回复预算未受影响 |
| 真实模型 T01–T12 | **BLOCKED** | 无 `hy4flash`／`DeepSeek-V4.1-Flash` 凭据与预算授权 |

基线失败口径：初始克隆未装依赖时 `qq-unify` 因缺 `otplib` 失败，`npm ci` 后全绿——属环境缺失，不计入代码问题，日志保留。

---

## 五、门禁

| 门禁 | 结论 | 依据 |
|---|---|---|
| G0 资产 | 通过 | 源哈希逐字节匹配、五章完整、源/运行分层 |
| G1 基线 | 通过 | 父提交 = C 确认的 `136aba5b`；未动 release |
| G2 接线 | 通过 | 默认加载、旧卡不冒充、工作档隔离、核心层不自动改写 |
| G3 行为 | **条件通过** | 确定性路径全绿；真实模型矩阵 BLOCKED |
| G4 交付 | 通过 | 三坐标、patch/bundle、SHA256、完整性验证、接收指南 |

---

## 六、风险

| 级别 | 项 | 说明 | 建议 |
|---|---|---|---|
| B | 真实模型体验未验 | 运行卡比 P1A 短 597 字，自然度是否下降只能真机对照 | 授权模型后跑 T01–T12 A/B |
| B | 协商层无审批入口 | 卡变更依赖 git 与人工评审 | 后续单独立项，不在本轮造壳 |
| B | 纠正队列未闭环（C 已认定 S3-M2） | T07 只验"旧错不当现事实" | 随 S3 R1/R2 收口 |
| C | 主动通道 mock | 自称能力需与部署一致 | 部署侧明确"有策略无通道" |
| C | DOCX 原件未核 | 只有附录 A | 如收到 DOCX 再核 41951/`a056c42f` |

---

## 七、生产与回退

- 生产零变更：未连生产机、未部署、未重启、未改配置、未改模型桥。
- 部署建议（供 Z）：仅替换 `role.chenzhiyao.md` 一个文件，`roleCard` 配置无需改动；上线后确认启动日志出现 `chenzhiyao-v1.0 … 源 V1.0/d14298b4…`。
- 回退：`git show 136aba5b:packages/companion/role.chenzhiyao.md > role.chenzhiyao.md` 即回到 P1A 卡；回退**不回滚**主人记忆、认证状态与人格快照。

## 八、交付物

资料库批次：`2026-11-27_K-PERSONACORE-V1-ASSET-PREFLIGHT-01`（本侧）。含本报告、精简报告、资产记录与映射、实现与回退、测试证据与 12 场景方案、打包完整性、tech 三件、7 份日志、SHA256SUMS。GitHub 仅脱敏版（移除附录 A、运行卡原文）。
