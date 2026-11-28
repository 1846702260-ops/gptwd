# K-SELFCORE-V1.0-LIMITED-01 · 补篇：V1.1 动态自我模型变更说明（C 验收用）

- 日期：2026-11-28｜执行：K｜依据：G《补充技术选型指令》
- 性质：在 V1.0 候选基础上的**增量升级**，不推翻 V1.0，不改变基线
- 基线不变：`354f74a7` / Tree `b2c1c079`
- 生产影响：**零**（未部署、未改生产、未改人格核心）

---

## 一、为什么要升级（一句话）

V1.0 的 `profile()` 只能回答"此刻我会什么"，是**状态面板**；
G 指令要求"面向动态自我模型与持续自我认知发展"，因此必须能回答
**"我准不准、我稳不稳、我在长还是在退"**。

---

## 二、API 增量（向后兼容，V1.0 全部保留）

### 2.1 新增（M7–M9）

```js
const core = createSelfCore({ gateway, selfModel, minN: 8, k: 4 });

// M7 校准闭环
core.predict('code.run');
// → { capability, p, basis, sufficient, n, bias, note }
//   冷启动: { p: null, basis: 'cold_start', sufficient: false, n: 0 }

core.observe({ capability: 'code.run', predictedP: 0.8, actualOk: true, at, context });
// → true / false（缺 capability 或 actualOk 不明确则拒绝）

core.calibration();
// → { overall: {n, sufficient, brier, ece, bias, empiricalRate, verdict}, byCapability, total, minN }

// M8 可靠性
core.reliability(4);
// → { k, overall: {mean, passAtK, passHatK, reliabilityGap}, perCapability, undefinedCount }

// M9 轨迹与元认知
core.recordSnapshot();
core.trajectoryOf();
// → { spans, steps, total: {gained, degraded, appeared, disappeared, changed}, netVerified }

core.selfKnowledge();
// → { overconfident, underconfident, illusory, honest, total, deviationRate }

// M3 动态版
core.renderDynamicForPrompt();
```

### 2.2 行为变更（唯一一处破坏性语义）

`profile().rows[*].confidence`：

| | V1.0 | V1.1 |
|---|---|---|
| 无经验样本 | 硬编码 0.9/0.85/0.6/0.5/0.3/0.2 | **null** |
| 有经验样本（n ≥ minN） | 同上 | 经验成功率（0–1） |
| 溯源字段 | 无 | `confidenceBasis: 'empirical' \| 'insufficient_samples'`，原常量保留在 `priorConfidence` |

> 若下游有代码依赖 `confidence` 为数字，需改为判 null。本轮已确认仓内除测试外无此依赖。

---

## 三、新增文件

| 文件 | 行数量级 | 依赖 |
|---|---|---|
| `src/selfcore/calibration.js` | ~260 | 零依赖 |
| `src/selfcore/reliability.js` | ~110 | 零依赖 |
| `src/selfcore/trajectory.js` | ~150 | 零依赖 |
| `test/selfcore/dynamic.test.mjs` | ~290 | node:test |

修改：`src/selfcore/index.js`（扩展出口 + profile 单次取数修正）。

---

## 四、验收方法（C 可独立复跑）

```bash
# 1. 克隆候选
git clone -b feat/selfcore-v1.0-limited-01 <bundle> repo && cd repo

# 2. 坐标核对（应为 HEAD 见 05 号文件，Tree 见 05 号文件）
git rev-parse HEAD HEAD^{tree} HEAD^

# 3. V1.1 新测试（期望 19/19）
node --test packages/companion/test/selfcore/dynamic.test.mjs

# 4. V1.0 回归（期望 7/7）
node --test packages/companion/test/selfcore/selfcore.test.mjs

# 5. persona 回归（期望 127/127）
node --test packages/companion/test/persona-system/chenzhiyao-persona.test.mjs \
  packages/companion/test/persona-system/injection.test.mjs \
  packages/companion/test/persona-system/experience-fix.test.mjs \
  packages/companion/test/persona/p147-expression.test.mjs \
  packages/companion/test/mood.test.mjs
```

**验收要点（建议重点看这三条）**

1. **诚实性**：`calibrate(3 条样本, {minN: 8})` 是否返回 `brier/ece/bias = null`。若返回数字，即判定不合格。
2. **不作外推**：`metrics([1,0,1], 4)` 是否 `passHatK === null`。
3. **纯计算**：`predict/observe/calibration` 是否不新增模型调用（可用 mock 计数）。

---

## 五、与 V1.0 的关系（避免误解）

- **V1.0 未被推翻**：六态分类、M1 四态、M4 缺口、M5 身份连续性、M6 七场景测试全部保留并通过。
- **V1.1 是叠加**：在六态之上加时间/校准/可靠性/轨迹。
- **基线未动**：仍是 `354f74a7`，本层只增不改基线文件（除 `selfcore/` 与 `self-model.js` 钩子、`index.js` 挂载）。

---

## 六、待裁决项（详见 10 号报告第七章）

| # | 事项 | 现状 | K 建议 |
|---|---|---|---|
| 1 | 校准账本是否落盘 | 纯内存 | 允许落盘但标注"可丢弃缓存" |
| 2 | 是否允许回写 self-model | 只读 | **维持只读** |
| 3 | ToolGateway 回执补 `at` 时间戳 | 无时间戳 | 建议批（跨核契约，须 G 批） |
| 4 | 是否引入 ReMe 结构化经验 | 未引入 | 暂缓至 P2，先独立复现 |

---

## 七、已知局限（不掩盖）

1. **冷启动期自核会说"我不知道我准不准"**：因为生产回执既无时间戳、也无先验预测记录。这是设计意图，不是故障。
2. `reliability()` 受 `recent()` 500 条窗口限制，长周期会失真。
3. 轨迹默认仅内存 50 快照，跨会话不成立（待裁决 1）。
