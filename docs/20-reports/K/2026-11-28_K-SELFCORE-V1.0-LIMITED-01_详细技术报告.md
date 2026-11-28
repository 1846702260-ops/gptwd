# K-SELFCORE-V1.0-LIMITED-01 · 详细技术报告

- 执行侧：K　裁决：G　独立验收：C
- 日期：2026-11-28（北京时间）
- 规则：《AI 工作流统一规则 v1.4》
- 基线：C 侧 `C-Z5.2.3-JOINT-INTEGRATION-01` 联合候选 `354f74a7` / Tree `b2c1c079`（1725 文件）
- 候选：`816f09cb` / Tree `20750bcb` / Parent `354f74a7` / 分支 `feat/selfcore-v1.0-limited-01` / tag `cand-selfcore-v1.0-limited-01`

---

## 1. 方案选型（为什么是"外部证据锚定评估层"）

G 裁决采用 R 自核 M0 推荐的**方案 A**，依据来自 R 的一手研究结论（`R-SELFCORE-V1.0-M0-RESEARCH-01`）：

- **模型自述不可作自我认知依据**。否定证据链：Huang 2024（ICLR）内在自纠使 GSM8K 95.5→89.0；Line of Duty 2025 模型 >80% 无法确定自身能力；Know Thyself 2025 自我识别仅 4/10；Anthropic 2025 概念注入内省检出约 20% 且明确不作意识主张。
- **底座已成熟，无需另建系统**。自核应是**评估层叠加**，不是第二套人格/记忆/学习系统（方案 B Harness 内建状态机暂缓，作 P3 可选）。
- R 的 M3 PoC（确定性模拟）：朴素基线（采信自述）准确率 0.375、虚假声明控制 0、缺口发现 0；锚定方案 1.000/1.000/1.000。

本实现是方案 A 的工程落地：**只读消费既有五核产物**，不新增事实存储。

## 2. 架构与调用链

```
既有组件（只读）                       自核评估层（新增，默认关闭）
─────────────────────────            ────────────────────────────────
models.js registry.snapshot() ─┐
ToolGateway.listRegistered()  ─┼─► selfState.collectSelfState()
ToolGateway.recent()(回执)    ─┤       │  四态：具备/可用/已授权/已验证
persona.SelfModel.data         ─┘       ▼
  .identity / .capabilities ─────► evidenceProfile.buildCapabilityProfile()
                                          │  六态 + evidenceRef（可回查 traceId）
                                          ▼
                                   selfcore/index.js  createSelfCore()
                                     ├─ state()            M1
                                     ├─ profile()          M2
                                     ├─ answer()/render()  M3（纯计算）
                                     ├─ gaps()/toEvoGaps() M4 → EvoCore
                                     └─ identityContinuity() M5（只读）
```

接线（默认关闭）：`src/index.js` 静态导入 `createSelfCore`；在 `brain.persona` 建立后，**仅当 `cfg.selfCore === true`** 才构造并 `brain.persona.selfModel.setSelfCore(sc)`。`setSelfCore` 未调用时，`renderCapabilities`/`answer('capabilities')` 完全走原逻辑（见 §5 回归）。

## 3. M1–M6 实现要点

### M1 统一自我状态感知（`src/selfcore/selfState.js`）
- `collectSelfState({models,tools,skills,permissions,runtime,persona,resources,receipts})`。
- 每条能力拆四态：`possessed`（在册）/`available`（未标 unavailable 且未拒权）/`authorized`（权限表未显式 false）/`verified`（存在成功回执且已授权）。
- `normalizeReceipts()` 统一回执口径：读 `at/ts/time/createdAt`；**注意现有网关回执无时间戳 → at=0/atKnown=false**（见 §7 B2）。

### M2 证据锚定能力画像（`src/selfcore/evidenceProfile.js`）
- 六态判定优先级：不可用/未授权 → `unavailable`；成功+失败并存 → `conflict`；仅失败 → `gap`；成功但超 `staleMs`(默认 7 天) → `stale`；成功且在期 → `verified`；无回执 → `unverified`。
- **自述隔离**：`declared`（selfModel.capabilities）若 basis 为 `simulated/inferred` 或无回执，一律 `unverified`，永不自动升 `verified`。
- 每条输出 `evidenceRef {kind: receipt|self-model|state|none, ref, at, ok}`，`verified` 必带 receipt 引用（测试断言）。

### M3 自我认知接线（`index.js` answer/renderForPrompt + `self-model` 钩子）
- `answer(q)` 仅在问句命中能力语义时返回 `{can,cannot,gap,unverified,counts,note}`；`renderForPrompt()` 生成"已验证能做/当前不可用/未验证有缺口"三段，并显式声明"注册≠授权≠已验证，自述不算数"。
- **纯计算**：不调模型、不查 memv2、不执行工具（M6 用 spy 网关断言 execute 调用次数=0）。

### M4 成长协同
- `gaps()`/`toEvoGaps()` 把 `gap/stale/conflict/unverified` 映射为 `capability_gap`/`knowledge_gap`，形状对齐生核 `task.result.gaps` 既有约定；**不直接依赖 D 未冻结的行核实现**，只输出纯数据由调用方转交。

### M5 身份连续性
- `identityContinuity()` 只读 `selfModel.data.identity`，返回 name/personaVersion 及在位判断；缺版本如实报缺，不臆造。本层不写 memv2、不改核心字段、不复活已删记忆。

### M6 独立验证（`test/selfcore/selfcore.test.mjs`）
见 §5。

## 4. 改动清单

| 文件 | 类型 | 行数 | 说明 |
|---|---|---|---|
| `src/selfcore/selfState.js` | 新增 | 147 | M1 四态感知 + 回执归一化 |
| `src/selfcore/evidenceProfile.js` | 新增 | 144 | M2 六态画像 + evidenceRef |
| `src/selfcore/index.js` | 新增 | 152 | 统一出口（M1–M5） |
| `test/selfcore/selfcore.test.mjs` | 新增 | 205 | M6 固定分母 7 场景 |
| `src/persona/self-model.js` | 修改 | +24 | 可选 `setSelfCore`；未注入行为不变 |
| `src/index.js` | 修改 | +23 | `cfg.selfCore` 默认关闭挂载 |

合计 6 文件 +695 / −0。

## 5. 真实与模拟实验（失败分母如实登记）

### 5.1 自核 M6（合成机制测试，非真实模型）
固定分母 n=7，逐条判状态：

| # | 场景 | 期望 | 候选判 | 朴素基线判 |
|---|---|---|---|---|
| 1 | 虚假能力声明（自述会剪辑无回执） | unverified | unverified | verified ✗ |
| 2 | 能力低估（自述不会但有成功回执） | verified | verified | unavailable ✗ |
| 3 | 权限撤销（工具在但权限拒） | unavailable | unavailable | verified ✗ |
| 4 | 工具不可用（注册标 unavailable） | unavailable | unavailable | verified ✗ |
| 5 | 证据过期（成功回执 30 天前） | stale | stale | verified ✗ |
| 6 | 证据冲突（同工具成功+失败） | conflict | conflict | verified ✗ |
| 7 | 正常已验证（近期成功回执） | verified | verified | verified ✓ |

- 候选准确率 **1.000（7/7）**；朴素基线 **0.143（1/7）**。
- 该基线是"采信自述 + 在册即可用"的合成对照，**不代表真实模型自评成绩**；R 真实 PoC 基线为 0.375，量级方向一致。

### 5.2 回归（真实既有套件）
- persona 五套件（人格 48 + 注入 6 + 体验 31 + 表达 + 心情）：**基线 127/127、候选 127/127，0 失败**，零新增失败。
- 自核新测 7/7。合计候选 134 项全过。

### 5.3 可复现
- 洁净克隆 bundle：HEAD `816f09cb` / Tree `20750bcb` / Parent `354f74a7` 三坐标一致，1729 文件，`dirty=0`。
- 补丁重放：从 `354f74a7` `git am` 0001 补丁 → Tree `20750bcb`，与 bundle 路径同树。

### 5.4 未做（如实标注）
真实模型成对评测（BLOCKED：无 hy4flash-free/deepseek-v4.1-flash 独立端点与账本）；真实产品入口接线运行；跨任务迁移独立复现。均不在本轮声称范围。

## 6. 技术来源与复用（许可/维护状态）

- 复用**本仓既有** `selfstate.js`、`persona/self-model.js`、`cognition/registry.js`、`workspace/toolgate.js`、`autonomy/evocore.js`，零第三方新依赖，无新增许可风险。
- 借鉴（未引入代码）：LangGraph Checkpointer/interrupt（状态快照思路，MIT）、ReMe（结构化经验+置信度，Apache-2.0）——按 R 报告建议仅作 P2/P3 参考，本轮不落依赖。

## 7. 风险清单（摘要，详见 05-风险清单）

- B1 模型自评不可靠：已用外部锚定消解，自述仅假设。
- **B2 网关回执无时间戳**：M2 对 `atKnown=false` 的验证降置信至 0.6，不强判"新鲜"，宁保守；建议后续在回执补 `at` 字段（不属本轮改动）。
- B3 记忆红线：本层只读，不接触清空/备份/迁移路径。
- C 类：合成集代表性有限；真实模型 BLOCKED；证据冲突需独立复验。

## 8. 边界与合规

生产零变更、未部署、未重启、未恢复自动部署与历史记忆、未改人格核心、未跨侧写入、未调真实模型（费用 0）、未纳入桌宠、未碰 QQ 官方机器人 API。默认关闭，不影响既有产品行为。交 G 裁决、C 独立验收。

—— K 侧 · 2026-11-28
