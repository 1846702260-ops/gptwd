# 01｜详细技术报告 · C-MINDCORE-V1.0-BASELINE-SYNC-02

生成时间：2026-11-28
执行方：C（集成与验收）
基线：灵核联合候选 `39bd712`；合入 Z `b50a2057`（MICROFIX-04）

---

## 1. 任务与判定

G 已批准生核 V1.0 进入受控上线授权流程。本轮只做「最终生核基线与灵核候选对齐」，不重新开展研发，不改动正在受复验的候选。

判定：**ACCEPTED_WITH_OBSERVATIONS**。三处修复均成立，两侧成果保留，影响范围无回归；两项 B 类观察（错误码语义、`ok:false` 的行为面扩大）已登记，均不阻断。

---

## 2. 基线核验

### 2.1 两个基线的实际坐标

| 角色 | 分支 | HEAD | Tree | 文件 |
| --- | --- | --- | --- | --- |
| 灵核联合候选 | `cand/mindcore-v1.0-joint-01` | `39bd712` | `c3f114bc` | 1721 |
| Z 生核候选 | `cand/evocore-v1.0-microfix-04` | `b50a2057` | `772b830e` | 1710 |
| 生产 | `release/z5.2.1` | `5ac4dd02` | — | — |

Z 候选的父提交是 `e1dea27`（C 收口候选），说明它接在生核收口之后、灵核集成之前——两条线从同一点分叉，合并面最小。

### 2.2 交付形态的偏差（如实记录）

Z 未投放 `READY.json` + `SHA256SUMS` 的正式包裹。共享目录 `outbox/`（截至 05:37）、`inbox/`、`feedback/`、`reports/` 内均无 MICROFIX-04 相关文件；交付仅以远端分支 `cand/evocore-v1.0-microfix-04` 形式存在。

C 的处理：不当作阻断，改为**可复算核验**——从分支生成补丁与 bundle，计算并冻结 SHA256 作为下游参照：

```
562dff49ae093d139925a6a8e0780e5e4a783b82660ddafaed5472515cd30a62  Z-EVOCORE-V1.0-MICROFIX-04.diff  (14348 B)
a67ca4f4c63f370526481e795980aa816abcf79f3d847c293248eba242183edf  z-microfix-04.bundle            (18345517 B)
```

核验链闭合：父提交正确 → Tree 与声明逐字一致 → 补丁在 `e1dea27` 重放 `write-tree` 精确等于 `772b830e`。三条独立路径互证，比单一 manifest 更强——manifest 只能证明"文件没被改过"，重放能证明"内容确实生成这棵树"。

**建议**：后续 Z 侧交付统一带 manifest，否则每次都要由接收方重建参照值，且无法证明交付瞬间的完整性。

---

## 3. Z MICROFIX-04 技术解析

补丁 2 文件 +151/−15：`research.js`（+75/−15）、`z-microfix-04.test.mjs`（+91）。

### 3.1 修复①：CSS 清理的判定对象

**问题**：C 上一轮的 `stripCssResidue` 用 `/\{[^{}]{0,600}\}/g` 把任何花括号块当样式删掉。这个判据在真实网页上会误删技术证据：

- 文档页里的 JSON 示例 `{"token_type":"Bearer","expires_in":3600}`
- 代码块里的对象字面量 `{ color: red, size: 12 }`
- API 说明里的配置片段

这些都进得了正文，且**正是需要保留的证据**——它们是页面在讲的内容，不是样式泄漏。

**Z 的修法**：新增 `looksLikeCssRuleBody(body)`，先排除再确认：

```
排除：带引号的键（JSON）→ 含标签 → 箭头函数/function/const/return/++/===
     → 逗号后紧跟「键名:」（对象字面量，CSS 声明用分号）
确认：按 ; 切段后每段都符合 `名称: 值` 形态
     → 且至少一段以典型 CSS 属性名开头（--, color, margin, display, flex, ...）
```

只有全部通过才当 CSS 删除。这是把判据从「形状像」改成「内容是」——与上一轮密钥判据的修法同一思路，方向正确。

**残留风险**：`CONTENT` / `src` 等属性名同时出现在 CSS 与配置里；若某页正文正好是一段"每行都是 `key: value`"的配置清单，仍可能被误删。概率低（要求每段都是声明式且无逗号键），且删除不产生错误结论、只损失证据，属可接受。已由 S-5 覆盖正向用例。

### 3.2 修复②：分段裸密钥

原 `BARE_SECRET` 的 `(sk|pk|rk)[-_][A-Za-z0-9]{16,}` 要求分隔符后直接是长串。真实密钥常见分段形态 `sk-live-…` / `sk_live_…` / `sk_test_…`，分隔符后是 `live`/`test` 再接下划线——原正则不匹配，**漏报真实泄漏**。

Z 改为 `(sk|pk|rk)[-_](?:(?:live|test)[-_])?[A-Za-z0-9]{16,}`。C 复验三种形态全命中，同时 CSS 变量仍不报（`findSecretLeak` 的值判定未被削弱）。

**注意方向**：这是**收紧**（多抓），不是放宽。上一轮 C 把密钥判据从"命中即报"改成"按值判定"时，风险是漏报；这条补丁正补上了其中一类。

### 3.3 修复③：正文损失的上报

**问题**：C 的未闭合块清理（`②` 层）从开标签吃到下一个闭合标签或文末。若截断点落在正文中间，这一吃会把**正文本身**吃掉，然后剩下的一点点文本仍然"看起来像正文"，被当作成功返回。

**Z 的修法**：被删区间里若出现块级内容标签（`p/div/h1-6/li/article/section/table/pre/code/blockquote`），说明吃的不是样式而是正文：

```
bodyLost = true；lostChars 累计
extractArticleEx → { text, bodyLost, lostChars }
webRead：
  - bodyLost 也触发 Chrome 渲染重试（原来只在 text.length < 300 时渲染）
  - 最终 ok = text.length >= 120 && !bodyLost
  - degraded = bodyLost；degradedReason = 'body_truncated_unrecoverable'
  - renderNote 写明「正文在未闭合样式/脚本处被截断，提取不完整」
```

**兼容性**：`extractArticle(html)` 保留为字符串入口（内部转调 `extractArticleEx().text`），老调用方不用改。C 用 S-8 固化这条。

**C 的评价**：这是本轮最关键的一处。它把"我看不清"和"我看清了"区分开了。上一轮 C 修的是"别把样式当正文"，Z 这一轮补的是"别把吃掉一半的正文当完整正文"——后者更难发现，因为它不报错、只是少了内容。

---

## 4. 集成实现

### 4.1 合并

```
39bd712 (灵核联合) ──┐
                     ├─ merge ─→ eaf8623 / c03afed8 (1722 文件)
b50a2057 (Z 微修)  ──┘
```

`ort` 策略，零冲突，改动面正是 Z 的两个文件。`git diff b50a2057 HEAD -- research.js` 为空，说明合并没有引入二次改写——这比"合并成功"更强的结论：**最终 research.js 就是 Z 审过的那一份**。

### 4.2 成果保留核验（逐项）

| 归属 | 检查点 | 结果 |
| --- | --- | --- |
| 生核 A-1 | `detectDirective` 4 处、`injectionOfPage` 3 处 | 在 |
| 生核 A-2 | 语义等价判据（词项/数字/极性） | 在（`evoEvidence.js` 未被触碰） |
| 生核收口 | `findSecretLeak` 按值判定、`stripCssResidue` 三层清理 | 在 |
| 生核接口 | `inspect` 对 promoted 目标从 memv2 回填 | 在 |
| 灵核 | `src/cognition/` 22 模块、`versionStrategy`/`onSelected` 5 处 | 在 |
| 灵核开关 | `ZY_MINDCORE_SHADOW` / `ZY_MINDCORE_LIMITED`，均 `=== '1'` | 在，默认关 |
| 前端 | `APP_VERSION = 'Z5.2.1'` | 在 |

---

## 5. C 侧新增与修正

### 5.1 新增接口验收（`c-baseline-sync02.test.mjs`，10 项）

只验**补丁引入的对外契约变化**，不重复既有行为：

| 用例 | 断言要点 |
| --- | --- |
| S-1 | 正常页 `ok=true`、`degraded=false`、`degradedReason=''` |
| S-2 | 未闭合 style 吞正文 → `ok=false`、`degraded=true`、reason 精确值、`renderNote` 含"截断" |
| S-3 | 未闭合 script 吞正文 → 同上 |
| S-4 | 未闭合块在正文**之后** → 不判降级、正文完好、样式不泄漏 |
| S-5 | JSON 与无引号对象经 `webRead` 仍保留（技术证据不被当 CSS 删） |
| S-6 | CSS 变量不报密钥；分段裸密钥四类照报 |
| S-7 | 注入（英/中/隐藏）仍拦；正常页不因 script 标签误判 |
| S-8 | `extractArticle` 返回字符串（兼容入口） |
| S-9 | 灵核两个开关默认关闭（源码契约 + 进程环境双重断言） |
| S-10 | **降级页不得进入学习链路**：web_read 失败 → 目标不得走向待审批/晋升，memv2 不新增原子 |

S-4 与 S-2/S-3 是一对：只验"会报降级"不够，还要验"不该报时不报"，否则相当于用放宽换通过。

S-10 是本轮唯一跨层断言（抓取 → 工具 → EvoCore → memv2），它证明 Z 的 `ok=false` 真的传导到了学习决策，而不只是返回值里多了个字段。

### 5.2 修正的两处测试缺陷

**(a) `c-m2-closedloop` 硬编码工作树路径**

```js
// 修前
const ROOT = 'D:/WorkBuddy数据/.../work/wt-final/packages/companion';
// 修后
const ROOT = path.resolve(import.meta.dirname, '../..');
```

危害不是"跑不起来"，而是**静默地测错对象**：本轮第一次跑影响范围时，它在尚未合入 Z 补丁的树上报 8/8 通过——实际跑的是 `wt-final` 的代码。若不修，本轮"影响范围无回归"的结论就是假的。这类缺陷比产品 bug 更危险，因为它污染的是证据链本身。

**(b) `research-extract` E4 断言源码字面**

原断言 `/let text = extractArticle\(raw\)/`。Z 改成两行（`const ex0 = extractArticleEx(raw); let text = ex0.text;`），意图未变但字面不匹配。改为断言契约：走段落式提取、阈值 300、兜底 `text: extractText(clean)`、`extractArticle` 兼容入口仍在。

原则：**测试应钉住行为契约，不钉住实现写法**。否则每次合理重构都要改测试，久而久之测试就变成"防止别人改代码"而不是"防止行为退化"。

---

## 6. 回归与证据

| 项 | 命令范围 | 结果 |
| --- | --- | --- |
| 补丁影响范围 | `research.js` 7 个直接消费测试 + Z 新增 11 项 + C 新增 10 项 = 9 文件 | 107/107 |
| 灵核单测 | `test/cognition/*.test.mjs` | 302/302 |
| 真实网页 | 10 个公开页（nodejs / MDN / python / TS / PostgreSQL / git-scm / endoflife / rust / tailwind / bootstrap） | 抓取 10/10；判可疑 0；正文含 CSS 0；误标降级 0 |
| 洁净克隆 | bundle 克隆后核对 | 1723 文件、Tree 一致、工作区干净 |
| 补丁重放 | `git diff 39bd712 HEAD` 在 `39bd712` 上 apply | `write-tree` = `1d1b43fa` 精确匹配 |

**为何不跑全量**：合并改动面 = 2 文件（`research.js` + Z 的测试），`index.js`/`autonomy`/`cognition` 一行未动。全量的其余部分与上一轮 `39bd712` 完全相同，重跑只增加耗时，不增加信息。这是 G 第 4 条的本意。

---

## 7. 能力与边界（四象限）

| 能力 | 已实现 | 已接线 | 已实测 | 未证明 |
| --- | --- | --- | --- | --- |
| 正文提取抗截断 | ✓ | ✓ | ✓（合成 6 形态 + 真实 10 页） | 真实"半截页"样本（Z 原 4 个 URL 仍未提供） |
| 密钥检测（含分段形态） | ✓ | ✓ | ✓（6 形态 + 4 误报反例） | 其他云厂商密钥前缀 |
| 降级上报 | ✓ | ✓ | ✓（含跨层到 memv2） | 渲染重试后恢复的成功率 |
| 灵核五项契约 | ✓ | ✓ | ✓（36 项） | 真实模型下的效果（BLOCKED） |
| 灵核限域单槽 | ✓ | ✓ | ✓ | 多槽/通用事实认知（**不在本轮范围**） |

---

## 8. 观察与建议

### B-1 降级页错误码丢语义（建议改一行）

`tools.js` 的 `if (!r.ok)` 分支把一切失败归为 `empty_body`，使目标的 `lastError` 无法区分"页是空的"与"正文被截断"。截断原因只活在错误文案里，机器读不到。

建议增加 `body_truncated`：

```js
if (!r.ok) {
  let code = r.degradedReason === 'body_truncated_unrecoverable' ? 'body_truncated' : 'empty_body';
  ...
}
```

不阻断：当前行为保守（失败重试，不学错），只是无法据此决策"换源"还是"放弃"。

### B-2 `ok:false` 扩大了失败面（需 Z 确认口径）

以前截断页返回部分正文，可能被当证据用（这正是缺陷）；现在直接失败。学习成功率会因此下降——这是正确性换来的，不是退化。请 Z 在真实环境确认：失败率上升是否在可接受区间，以及失败后是否有重试/换源策略兜底。

### B-3 真实"半截页"样本仍缺

C 只能用合成形态证明修复成立，无法证明 Z 原始 0/4 的具体页面已解决。需要原始 URL 与 HTML（或至少失败时的 `renderNote`）。

---

## 9. 边界声明

未部署、未改生产（`5ac4dd02` 未动）、未恢复自动部署、未恢复历史记忆、未纳入桌宠或 QQ 官方 API 迁移。灵核影子与限域开关默认关闭，装配条件未改。真实模型 0 调用，费用 0。
