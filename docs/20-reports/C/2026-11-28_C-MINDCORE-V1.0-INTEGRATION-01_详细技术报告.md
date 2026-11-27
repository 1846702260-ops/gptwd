# 01｜详细技术报告

**任务**：C-MINDCORE-V1.0-INTEGRATION-01｜灵核 V1.0 独立集成与能力验收
**执行**：C　**裁决**：G　**时间**：2026-11-28 06:37–07:00 (+08:00)

---

## 一、结论

| 项 | 值 |
| --- | --- |
| 灵核联合候选 | `cand/mindcore-v1.0-joint-01` @ `39bd712ff3ae0773c70fe2d14220bb82aaeec6a6` |
| Tree | `c3f114bcc6ba86fbfa72a71c9cb47aeee4cd95bc` |
| 文件数 | 1721 |
| 集成基线 | `e1dea27` / `819f4ef4`（生核收口候选，**全程未改动，Tree 未变**） |
| 生产 | `release/z5.2.1` @ `5ac4dd02` / `f05cdddf`（未动） |

判定：**集成完整、可测项全过、能力边界需如实限定**。

## 二、M0 交付核验

| 批次 | 件数 | SHA256 | READY 坐标 | 实际 |
| --- | --- | --- | --- | --- |
| M0-RESEARCH-01 | 10 | 全通过 | 研究件，无候选坐标 | — |
| SHADOW-01 | 8 | 全通过 | `d2ec05b0` / `dd3db012` | 一致 |
| LIMITED-02 | 9 | 全通过 | `246a6b55` / `3e14e436` | 一致 |

血缘实测：

```
246a6b55 (limited)  → 父 d2ec05b0 (shadow) → 父 cb4c30a3 (生核 DEEPEN-02)
```

即灵核候选**不继承** C 的 A 类修复与生产 Z5.2.1 对齐，必须由本轮集成补上。

## 三、M1 基线集成

以 `e1dea27` 为基 merge `246a6b55`，零冲突 → `2e83196` / Tree `b27d3b50`（1719 文件）。

**唯一重叠文件 `evocore.js`**，两边改动位置错开：

| 侧 | 位置 | 内容 |
| --- | --- | --- |
| 灵核 | 第 2 行 import、`guidance()` | 引入 `versionStrategy`；`guidance` 增加 `onSelected` 回调，把命中的策略与版本抛给上层 |
| 生核 | 第 4 行 import、`inspect()`、`tick()` 取证 | `detectDirective/extractText`；memv2 回填；三层注入判定 |

合并后逐项确认共存：`detectDirective`×2、inspect 回填注释、`findSecretLeak`、`stripCssResidue`、`versionStrategy`、`guidance(onSelected)`、`cognition/` 四个新模块、`APP_VERSION='Z5.2.1'`。

**不重复执行的四条（G 重点关注）**：

| 关注点 | 证据 |
| --- | --- |
| 重复学习信号 | 影子只 `record`，`shadowModelCalls/shadowToolCalls` 恒 0（K1-4） |
| 重复检索 | 限域命中直接由绑定证据渲染，`toolCallCount: 0`（turn.js 分支；S-3） |
| 重复模型调用 | 限域命中不调 `brain.reply`；影子只做本地分类与快照 |
| 重复任务执行 | 计划差异只 `previewPlanDelta`，注释明确"caller cannot pass this copy to TaskStore"（K4-7） |

## 四、M2 五项认知契约

### 4.1 请求决策

- 通道白名单 `chat`/`task`，其它抛 `invalid_channel`（K1-1）
- 影子 `turn()` 首行 `if (userId !== ownerId) return`，非主人零记录（K1-2）
- 作用域 `['private','work']`，group 等直接丢弃（K1-3）
- 影子只观察：记录真实用量 `actualInputTokens`，但 `shadowModelCalls` 恒 0（K1-4）

### 4.2 证据

`alignEvidence` 的过滤链：`slot`+`scope`+`subject` 全等 → `id/sourceId` 合法 → `observedAt ≤ now` → `validUntil` 未过期 → `maxAgeMs` 未超。

| 场景 | 结果 | 测试 |
| --- | --- | --- |
| 错槽 | `unknown / missing_evidence` | K2-1 |
| 过期（validUntil / maxAge） | `unknown`；产品侧 30 天口径 | K2-2 |
| 冲突（同槽异值） | `review / conflicting_evidence` | K2-3 |
| 跨档（work 证据服务 private） | `unknown` | K2-4 |
| 高置信但与主张不符 | `review`（不判 known） | K2-5 |
| 仅 INFERRED | `inferred / weak_evidence` | K2-6 |

**产品接线只认一个槽**（K2-7）：`recognizesRecordedRequirement` 的正则写死 `Node\.js`，Python / 数据库 / 松散问法全部不认。这是本轮**最需要守住的口径**——G 明确"不得把当前特定 Node.js 事实槽实现描述为通用事实认知"。

### 4.3 预算

两个层次：

**决策层**（`decisionContracts`）：四维 `modelCalls/inputTokens/outputTokens/costMicros` + `deadlineAt`。任一维度非整数即 `unknown_<dim>` 拒（K3-1）；超额 `limit_<dim>`（K3-2）；到点 `deadline`（K3-3）。

**计量层**（`usageMeter`）：按 stage 分桶（chat/plan/replan/runner），实报 token 计 `reported`，估算输入计 `estimated`，缺失计 `unknown`（K3-4）。

**费用**：`snapshot().totalCost` 在零调用时是 `known_no_calls/0`，有调用时恒为 `unknown/null`（K3-5）——即"未经核验的费用永远不编造"。

**fail closed**：`enforce` 且配置了有限 `limitCostMicros` 但无报价时，`begin()` 直接返回 `allowed:false`（K3-6）；`max_tokens` 缺失同样拒绝（K3-7）。

### 4.4 计划差异

`planSnapshot` 用 `digest([taskId, steps])` 生成 revision，每步的 `definition` 与 `receipt` 各自摘要。

`validatePlanDelta` 的拒绝路径：

| 条件 | 码 |
| --- | --- |
| version/taskId/baseRevision 不符 | `stale_revision` |
| 任一步副作用未知 | `unknown_side_effect` |
| 失败步骤已非 failed | `failure_not_current` |
| 追加空/超 8 条 | `bad_delta` |
| 步骤非法或工具不在白名单 | `invalid_step` |
| 缺 actionKey | `missing_action_key` |
| actionKey 重复 | `duplicate_effect` |
| 依赖未完成 | `invalid_dependency` |

`previewPlanDelta` 只返回 `nextRevision` 与预览步骤，不回写 TaskStore（K4-7）。

**覆盖范围必须说清**：影子里的 proposal 只有只读工具才给 `actionKey`（`'readonly:' + revision + ':' + index`），写操作一律空串 → 被 `missing_action_key` 拒。所以当前只支撑"只读资料清单重规划"，**不是通用自主决策**。

### 4.5 策略效果

- 版本：`versionStrategy` = `sha256([strategy,title,steps]).slice(0,16)`，内容决定，改内容即变（K5-1）
- 无独立对照 → `unmeasured`（K5-2 / K5-6）
- 候选正确数低于基线 → `negative`，`eligible=false`（K5-3）
- 样本总量不同 → `unmeasured`（K5-4），不给"提升"结论
- 命中判定：只有整行出现在 `system`/`developer` 消息里才算（K5-5）；用户消息里出现同样文字不算

## 五、M3 真实产品接线与安全

四种开关组合各起一个真实实例（真实 config、真实登录、隔离目录与端口）：

| 项 | 默认 | 仅影子 | 仅限域 | 都开 |
| --- | --- | --- | --- | --- |
| 主人登录 | 200 | 200 | 200 | 200 |
| 生核状态 | 200 | 200 | 200 | 200 |
| 生核详情带 2 条证据 | ✓ | ✓ | ✓ | ✓ |
| 忆核待确认 | 200 | 200 | 200 | 200 |
| 伪造会话 | 401 | 401 | 401 | 401 |
| 未登录 | 401 | 401 | 401 | 401 |

开关装配：`process.env.ZY_MINDCORE_LIMITED === '1'` 与 `ZY_MINDCORE_SHADOW === '1'` 两个独立判断，默认不设即 `null`（S-2 静态断言 + 运行断言）。

**进程恢复**：同目录关掉再起，重新登录正常、生核目标数不变（1→1）、开关回默认关闭。

**跨主体隔离**：单账号产品无法本地构造第二个登录主体；已在单元层证明影子与限域都按 `ownerId` 丢弃他人事件（K1-2 / K5-6 / S-1），服务层用"伪造会话 401"证明非签发身份进不来。

**工具权限**：限域命中时不调模型、不调工具（`toolCallCount: 0`）；计划差异的白名单仍受 `allowedTools` 约束，写操作无 `actionKey` 一律拒。新增认知决策没有扩大任何工具权限。

## 六、M4 能力完整性

| 能力 | 状态 |
| --- | --- |
| 决策信封 / 证据对齐 / 预算预留 / 计划差异 / 策略回执（纯函数层） | **已实现 + 已实测** |
| 影子观察（turn/replan/task/strategyExposed） | **已实现 + 已接线 + 已实测**（零模型调用） |
| 限域计量（Brain 发送点 token 分阶段） | **已实现 + 已接线 + 已实测** |
| 限域证据回答（Node.js 运行要求） | **已实现 + 已接线 + 已实测**；覆盖面 = 1 槽 1 问句 |
| 计划差异用于只读资料清单 | **已实现 + 已接线**；写操作未支持 |
| 策略正向效果 | **未证明**（只有确定性对照，D 自己标"仅为确定性试验"） |
| 真实模型下的学习与延迟 | **BLOCKED**（无授权凭据） |
| 通用事实认知 / 通用自主决策 / 全局推理预算 / 广泛经验迁移 / 完整世界模型 | **不在本轮范围**，也未声称 |

## 七、回归

| 套件 | 结果 |
| --- | --- |
| 生核 + workspace | 442 项：436 过 / 5 失败 / 1 跳过（与 e1dea27 **逐项相同**） |
| 灵核原有 | 266 项全过 |
| 本轮新增契约 | 36 项全过 |
| 真实服务场景 | 27 项全过 |
| 补丁重放 | 相对收口增量 → `write-tree` = `c3f114bc`，逐字一致 |

## 八、边界

未部署、未改生产、未恢复历史记忆、未纳入桌宠与 QQ 官方 API 迁移；真实模型 0 调用，费用 0；生核冻结候选 `e1dea27` / `819f4ef4` 全程未被修改（供 Z 复验）。
