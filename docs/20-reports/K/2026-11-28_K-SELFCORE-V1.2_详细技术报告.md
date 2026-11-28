# K-SELFCORE-V1.2-REAL-LOOP-01 · 详细技术报告

- 日期：2026-11-28｜执行：K
- 基线：`cand/z5.2.3-joint-integration-02` = `0e508191` / Tree `ba2869706f` / 1729 文件
- 候选：`cand/k-selfcore-v12-01` = `2b2ab06e` / Tree `3364c569` / Parent `0e508191` / 1742 文件

---

## 一、问题：自核此前能"描述自己"，不能"校正自己"

V1.0 解决了"不许模型自述当事实"：能力状态只能由执行回执授予。
V1.1 解决了"置信度不许拍脑袋"：无样本返回 null，有样本用经验率。

但两者都缺一个东西：**闭环**。
V1.1 的 `observe()` 需要外部手动调用，且 `predict()` 与 `observe()` 之间没有关联。
结果是：能算校准，但**没人喂数据**；能存状态，但**重启即失**。

V1.2 的定位就是补上这一段：让"预测→执行→观察→校准→调整"自动转起来，并且这份状态能跨进程活着。

---

## 二、架构

```
                    ┌─────────────────────────────────────┐
   ToolGateway ────►│ receiptBridge                       │
   (真实回执)        │  normalize → 时间戳标注 → 去重 → 配对 │
                    └──────────────┬──────────────────────┘
                                   │ paired / unpaired
                    ┌──────────────▼──────────────────────┐
   独立验证证据 ────►│ tick() / ingestVerification()        │
   (能力测试)        └──────┬───────────────┬──────────────┘
                            │               │
                    ┌───────▼──────┐  ┌─────▼──────────┐
   predict() ──────►│ calibration  │  │ persistence    │
   (事前预测)        │ ledger       │◄─┤ JSONL 账本     │
                    │ Brier/ECE    │  │ (派生缓存)     │
                    └───────┬──────┘  └─────┬──────────┘
                            │               │ rebuild()
                    ┌───────▼───────────────▼──────────┐
                    │ profile() 六态 + 状态迁移 trajectory│
                    └───────┬──────────────────────────┘
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
      renderDynamic   toEvoSuggestions  selfKnowledge
      (对话注入)       (生核建议)        (元认知偏差)
```

---

## 三、模块设计

### 3.1 `persistence.js` — 可追溯、可恢复，但不是第二套记忆

**存储格式**：JSONL，一行一条，追加写。字段白名单**只有 6 个**：

```
{capability, predictedP, actualOk, at, traceId, source}
```

**为什么这些字段不算"长期记忆"**：

| 判据 | 说明 |
|---|---|
| 不含自然语言 | 不存对话、不存摘要、不存语义向量 |
| 不含事实 | 不存"用户是谁""发生过什么"——那是 memv2 |
| 可再生 | `rebuild()` 可从执行回执重放重建全部内容 |
| 有上限 | 默认 20000 行滚动截断，不追求永久 |
| 可删除 | 删掉不损失任何不可再生信息 |

**关键实现点**：

1. **不做 recursive mkdir**。实测 `mkdirSync(recursive:true)` 对 `/proc` 深层路径会阻塞进程 >2 分钟。
   改为：父目录必须存在、只创建最后一级、失败立即降级内存。**这条是硬性安全修复。**
2. **坏行容忍**：解析失败计入 `corrupt` 计数但不中断加载，如实报告而非静默吞掉。
3. **原子重写**：`compact()` 用 tmp + rename，避免写一半崩溃留下半截文件。
4. **mode 三态**：`file` / `memory` / `degraded`，任何非 file 状态都带 `degradedReason`。

### 3.2 `receiptBridge.js` — 真实回执接入

**时间戳的诚实处理**（对应 V1.0 风险 B2）：

| 情况 | 处理 | 标记 |
|---|---|---|
| 回执自带 `at` | 直接使用 | `atSource='native'`, `atKnown=true` |
| 回执无 `at` | 用采集时刻填充 | `atSource='collected'`, `atKnown=false` |

`atKnown=false` 会被 M2 用于**降级置信**，而不是假装这条回执刚刚发生。

**配对规则**（从严，防张冠李戴）：

1. 优先 `traceId` 精确匹配；
2. 否则同 capability + 预测早于回执 + 30 分钟窗口内 + 一对一未使用；
3. 配不上的进 `unpaired`，**不进入校准**。

**为什么第 3 条这么重要**：错配一条回执，会用"别人的结果"去校验"我的预测"，直接污染整个校准结论。
宁可少一条样本，不可错一条。

### 3.3 闭环主循环 `tick()`

```js
tick() {
  读 gateway.recent()
  → normalize（时间戳标注）
  → dedupe（按指纹，防重复计入）
  → matchToPending（与事前预测配对）
  → ledger.record(paired) + store.append
  → 返回 { receiptsSeen, fresh, duplicate, paired, written, unpaired, pendingOpen }
}
```

**幂等性**：同一回执重复 tick 不会重复计数（`consumed` 集合 + 指纹去重），测试 V12-14 覆盖。

### 3.4 `toEvoSuggestions()` — 给生核的建议

只对 `gap / conflict / stale` 三类产出建议，每条带：
- `evidenceLevel`：`measured`（有校准样本）或 `receipt_only`（只有回执）
- `observedSuccessRate`：**仅当校准样本充足时给数字，否则 null**
- `priority`：gap/conflict 为 high，stale 为 medium

**设计理由**：让生核去补一个"其实只是没测过"的能力，是浪费算力。
所以建议必须区分"确实不会"与"只是没验证"。

---

## 四、关键设计决策与权衡

### 决策 1：校准与状态分离（最重要）

| 方案 | 做法 | 问题 |
|---|---|---|
| 合并 | 用回执同时更新状态和校准 | 无事前预测时只能用 0.5 假先验，校准失真且**看起来有数据** |
| **分离（采纳）** | 状态用回执，校准只用预测-结果对 | 校准样本变少，但每条都真实 |

代价：冷启动期校准长期 `sufficient:false`。
接受理由：**宁可说"我不知道我准不准"，也不给一个假数字。** 这与 V1.1 取消硬编码 confidence 是同一条原则的延续。

### 决策 2：持久化默认关闭

`storeDir` 未配置时纯内存。理由：
- 自核是旁路评估层，不能因为写盘失败影响对话；
- 是否落盘、落在哪，属配置决策，应由 C/主人定，不由自核自决。

### 决策 3：不引入外部框架

LangGraph 提供 Checkpointer，但引入它=新增运行时+状态机改造，属重大架构调整。
ReMe 提供结构化经验，但需独立复现其自报增益。
本轮全部用 Node 标准库实现，零依赖。理由见 V1.1 技术选型报告第五章。

---

## 五、验证结果

### 5.1 三层验证

| 层 | 状态 | 说明 |
|---|---|---|
| **L1 确定性测试** | ✅ 完成 | 46 项，全绿，可复现 |
| **L2 真实执行** | ◐ 就绪待数据 | 闭环已实现，需生产挂载积累样本；本轮**不宣称**效果 |
| **L3 跨任务迁移** | ⬜ 未开始 | 方案已给判定条件（等长对照 + 固定分母 + pass^k） |

### 5.2 L1 明细

| 套件 | 项数 | 覆盖 |
|---|---|---|
| selfcore（V1.0） | 7 | 七场景固定分母 |
| dynamic（V1.1） | 19 | 校准/可靠性/轨迹/元认知/纯计算 |
| v12（V1.2） | 20 | 持久化恢复/降级/白名单/坏行/去重/配对/闭环/重建/建议 |
| persona 回归 | 127 | 零新增失败 |

### 5.3 两个"先红后绿"（证明测试有效）

1. **样本不足仍返回数值**：V1.1 首次实现时 `calibrate(3 样本, minN:8)` 返回 `ece=0.01`，
   断言要求 `null` → 失败 → 改为全 null。
2. **`profile()` 重复取数**：一次调用 `gateway.recent()` 两次（state 与 profile 各一次），
   计数断言失败 → 改为单次取数复用，同时消除两次读之间的状态漂移风险。

V1.2 另有一例**真实缺陷**：`mkdirSync(recursive)` 阻塞进程，测试挂起 2 分 55 秒才发现。

### 5.4 增量性核验（V1.5 §8）

```
compare 0e508191...2b2ab06e → 15 files
  modified +27 -0  packages/companion/src/index.js
  modified +24 -0  packages/companion/src/persona/self-model.js
  added   +n  -0   (13 个新文件)
ahead_by: 1, behind_by: 0
```

两个修改文件均为**纯新增、零删除**；无全树替换；文件数 1729 → 1742（+13）；`truncated: false`。

---

## 六、六核协同

| 核 | 交互 | 边界 |
|---|---|---|
| 灵核 | 只读工具注册表 | 不改注册表 |
| 忆核 | **不交互**（不写 memv2） | 校准账本与 memv2 物理隔离 |
| 心核 | 只读身份（name/personaVersion） | 不改人格核心、不复活已删记忆 |
| 生核 | `toEvoSuggestions()` 输出缺口 | 不建第二训练系统，只给建议 |
| 行核 | 消费 `recent()` 回执；提出契约 | 不改 toolgate（契约待 C 会签） |
| 自核 | 本层 | 只读旁路，默认关闭 |

---

## 七、风险与局限

| 级 | 风险 | 缓解 |
|---|---|---|
| B | 回执无时间戳 → 时效判定降级 | 已标 `atKnown=false`；契约提议待裁决 |
| B | 冷启动期校准不可用 | 设计意图；n<minN 时明确拒绝给数字 |
| B | 持久化文件被误当事实真源 | 代码 status() 与文档双处标注 `derived_cache`；可 rebuild 证伪 |
| C | `recent()` 500 条窗口截断 | 长周期可靠性会失真，已在报告标注 |
| C | 配对靠时间窗可能误配 | 优先 traceId；窗口 30 分钟；误配风险已文档化 |
| C | 真实模型对照 BLOCKED | 不伪造成绩 |

---

## 八、复现指南

```bash
# 1. 取候选
git clone -b cand/k-selfcore-v12-01 https://github.com/1846702260-ops/zhiyao-agi.git
cd zhiyao-agi
git rev-parse HEAD HEAD^{tree} HEAD^
# 期望 2b2ab06e... / 3364c569... / 0e508191...

# 2. 自核三套（期望 7 / 19 / 20）
node --test packages/companion/test/selfcore/selfcore.test.mjs
node --test packages/companion/test/selfcore/dynamic.test.mjs
node --test packages/companion/test/selfcore/v12.test.mjs

# 3. persona 回归（期望 127）
node --test packages/companion/test/persona-system/chenzhiyao-persona.test.mjs \
  packages/companion/test/persona-system/injection.test.mjs \
  packages/companion/test/persona-system/experience-fix.test.mjs \
  packages/companion/test/persona/p147-expression.test.mjs \
  packages/companion/test/mood.test.mjs
```

**验收建议重点看三条**：
1. `calibrate([3条], {minN:8})` 是否返回全 null（诚实性）；
2. `tick()` 对无事前预测的回执是否 `paired=0, written=0`（不造假）；
3. 持久化文件是否只有 6 个字段（非第二套记忆）。

---

## 九、结论

| 判断 | 结论 |
|---|---|
| 是否重建自核 | **否**，V1.0/V1.1 全部保留并通过 |
| 是否引入外部框架 | **否**，零依赖 |
| 闭环是否完整 | **机制完整**（predict→tick→calibrate→adjust→persist→rebuild） |
| 是否可恢复 | **是**，重启恢复 + 回执重放双路径 |
| 是否第二套长期记忆 | **否**，6 字段派生缓存、可再生、可删除 |
| 是否宣称真实效果 | **否**，L2 待数据，L3 未做 |

交 C 独立验收。
