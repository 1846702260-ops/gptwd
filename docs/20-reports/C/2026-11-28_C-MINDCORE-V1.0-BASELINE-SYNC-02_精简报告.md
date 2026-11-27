# 02｜精简报告 · C-MINDCORE-V1.0-BASELINE-SYNC-02

**结论：** 灵核最终候选已与 Z 生核候选对齐，`ACCEPTED_WITH_OBSERVATIONS`。Z 的三处修复全部合入并独立复验，两侧成果完整保留，补丁影响范围 107 项全过，真实 10 页零回归。

## 坐标

| 项 | 值 |
| --- | --- |
| 灵核最终候选 | `cand/mindcore-v1.0-baseline-sync-02` @ `6a5efbd` / Tree `1d1b43fa`（1723 文件） |
| 集成基线（灵核联合） | `39bd712` / Tree `c3f114bc` — 未改动 |
| Z 生核候选 | `b50a2057` / Tree `772b830e` — 未改动 |
| 生产 | `release/z5.2.1` @ `5ac4dd02` — 未改动 |

相对灵核联合候选：5 文件 +341/−19（1 个产品文件 + 4 个测试文件）。相对 Z 生核候选：24 文件 +1989/−31（灵核全部代码与测试）。

## 一、Z MICROFIX-04 核验

### 交付形态（需 G 知悉）

Z 未投放带 `READY.json` / `SHA256SUMS` 的正式包裹，交付以远端分支形式到达：`cand/evocore-v1.0-microfix-04`。共享目录（outbox / inbox / feedback / reports）内无 MICROFIX-04 相关文件。

因此「补丁 SHA256 核验」由 C 自行完成并冻结参考值（可从 commit 确定性复算）：

```
562dff49ae093d139925a6a8e0780e5e4a783b82660ddafaed5472515cd30a62  Z-EVOCORE-V1.0-MICROFIX-04.diff
a67ca4f4c63f370526481e795980aa816abcf79f3d847c293248eba242183edf  z-microfix-04.bundle
```

**核验结果**：父提交 = `e1dea27`（C 收口候选）；Tree `772b830e` 与裁决声明逐字一致；补丁在 `e1dea27` 上重放，`write-tree` = `772b830e` 精确匹配；2 文件 +151/−15。

### 三处修复（C 独立复验，非采信自述）

| 修复 | 声称 | C 复验 |
| --- | --- | --- |
| ① CSS 清理限定上下文 | `stripCssResidue` 只清 CSS 规则体，JSON/代码保留 | `looksLikeCssRuleBody` 先排除引号键、标签、箭头函数、对象字面量，再要求全部段符合声明式且含典型 CSS 属性名。JSON `token_type`、无引号对象 `{color: red, size: 12}` 均保留 ✓ |
| ② 分段裸密钥 | 补 `sk-live-` / `sk_live_` / `sk_test_` | 三种形态全命中；CSS 变量 `--token:12px`、`--md-token:rgba(...)` 仍不报 ✓ |
| ③ 正文损失上报 | 未闭合块吞正文时 `bodyLost`，不得当完整证据 | `extractArticleEx` 返回 `{text, bodyLost, lostChars}`；`webRead` 置 `ok=false`、`degraded=true`、`degradedReason=body_truncated_unrecoverable`，并在 `renderNote` 写明截断 ✓ |

**① 这条是本轮最有价值的修复**：C 上一轮把 `{...}` 一律当 CSS 删除，会误删网页里的 JSON 示例、代码块与配置片段——那是技术文档的证据正文，不是样式泄漏。Z 把它从"按形状删"改成"先判是不是 CSS 声明"，方向正确。

## 二、集成结果

零冲突合并（`eaf8623` / Tree `c03afed8`）。`research.js` 与 Z 版本**逐字一致**（`git diff b50a2057 HEAD -- research.js` 为空），说明没有合入时的二次改写。

成果保留逐项确认：

- 生核：`detectDirective` / `injectionOfPage` / `extractArticleEx` / `bodyLost` / `looksLikeCssRuleBody` / `findSecretLeak` 全在；`inspect` 从 memv2 回填仍在
- 灵核：`src/cognition/` 22 个模块完整；`versionStrategy`/`onSelected` 5 处；两个开关环境变量均在
- 前端：`APP_VERSION = 'Z5.2.1'`（生产并轨未回退）

## 三、C 侧额外修的两处测试缺陷（重要）

这两个缺陷不影响产品，但会**让"通过"变得不可信**，属于验收链自身的污染：

**① `c-m2-closedloop` 硬编码了另一棵工作树的绝对路径。** 后果很隐蔽：在任何别的工作树里跑这套真实闭环测试，导入的都是 `wt-final` 的代码——"通过"与当前树无关。本轮第一次跑影响范围时就踩了：它在未合入 Z 补丁的树上报绿，实际测的是旧代码。已改为相对自身定位（`import.meta.dirname`）。

**② `research-extract` 的 E4 断言源码字面形态。** Z 把 `extractArticle(raw)` 改成 `extractArticleEx(raw)`（要拿 `bodyLost`），字面不匹配但**意图仍满足**。改为断言契约：走段落式提取、阈值 300、兜底用清洗后文本、`extractArticle` 兼容入口仍在。

## 四、影响范围回归（未重复全量）

| 范围 | 结果 |
| --- | --- |
| `research.js` 七个直接消费测试 + Z 新增 + C 新增 | **107/107 通过** |
| 灵核 `cognition` 单测 | **302/302 通过** |
| 真实公开页面 10 个 | 抓取 10/10、判可疑 0、正文含 CSS 0、**误标降级 0** |
| 洁净克隆 | 1723 文件、坐标一致、工作区干净 |
| 补丁重放 | `write-tree` = `1d1b43fa` 精确匹配 |

未跑生核 442 项全量：合并改动面经 `git diff` 确认为 `research.js` 与 Z 的测试文件两处，`index.js` / `autonomy` / `cognition` 一行未动，重跑全量不产生新信息。

## 五、B 类观察（不阻断，建议后续处理）

**降级页的错误码丢了语义。** `tools.js` 把 `!r.ok` 一律归成 `empty_body`，于是学习目标的 `lastError` 也是 `empty_body`——"页是空的"与"正文被截断了"分不开，截断原因只活在错误**文案**里。行为本身是保守的（不会学错，只是失败重试），但下游无法据此决定"换源"还是"放弃"。建议加 `body_truncated` 码，一行改动。

**`ok:false` 带来的行为变化需 Z 知悉。** 以前截断页会返回部分正文（可能被当证据），现在直接失败。这是 Z 有意的选择（"不用残缺正文冒充成功"），C 认同；但它意味着这类页面的学习成功率会下降而非上升——这是正确性换来的，不是退化。

## 六、边界

未部署、未改生产、未恢复自动部署、未恢复历史记忆。灵核影子与限域开关默认关闭（`=== '1'` 才启用），本轮未改其装配条件。真实模型 0 调用，费用 0。
