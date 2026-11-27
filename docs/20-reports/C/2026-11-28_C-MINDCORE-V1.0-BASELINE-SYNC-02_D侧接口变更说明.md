# 07｜D 侧行核接口变更说明

**用途**：D 开展行核研究时若消费 `research.js`（正文提取 / 网页读取 / 密钥检测），需按本说明适配。
**不要求 D 重新开展研究**，也不要求 D 重写已有结论；只影响「如何读一个网页、如何判断读到的是否完整」。

**变更来源**：`b50a2057`（Z MICROFIX-04）合入灵核最终候选 `6a5efbd`。

---

## 一、变化清单

| 接口 | 变更 | 兼容性 |
| --- | --- | --- |
| `extractArticle(html)` | 保留，仍返回 `string` | **完全兼容**，老调用方无需改 |
| `extractArticleEx(html)` | **新增**，返回 `{ text, bodyLost, lostChars }` | 新增能力 |
| `webRead(url, opts)` 返回值 | **新增** `degraded`、`degradedReason`；`ok` 语义收紧 | 见 §3 |
| `findSecretLeak(text)` | 增加分段形态（`sk-live-` / `sk_live_` / `sk_test_`） | 只增检测面，行为更严 |
| CSS 残留清理 | 只清"确认是 CSS 声明"的花括号块 | 技术证据（JSON/代码）不再被删 |
| `injectionOfPage` / `detectDirective` | **未变**（签名、返回、判据层次全同） | 完全兼容 |
| SSRF 校验、`maxChars`、`untrusted` 标记 | **未变** | 完全兼容 |

---

## 二、最需要注意的一条：`ok` 不再只由长度决定

```js
// 修前
ok: text.length >= 120
// 修后
ok: text.length >= 120 && !bodyLost
```

`bodyLost` 的含义：清理未闭合的 `<style>` / `<script>` 时，被删掉的区间里出现了块级内容标签（`p/div/h1-6/li/article/section/table/pre/code/blockquote`），说明吃掉的**不只是样式，还有正文**。此时正文是残缺的。

**D 侧处理建议**：

```js
const r = await research.webRead(url);
if (!r.ok) {
  // 不要退回去用 r.text —— 那就是"用残缺正文冒充证据"
  // 区分两种失败：
  //   r.degraded === true  → 页面正文确实残缺（截断/未闭合块），换一个源，不要重试同一 URL
  //   r.degraded === false → 正文过短/被拒/反爬，看 r.renderNote 与 r.status
  return null;
}
if (r.degraded) { /* 理论上不会同时出现 ok:true + degraded:true，防御性断言 */ }
```

---

## 三、`webRead` 返回字段（新增部分）

| 字段 | 值 | 含义 |
| --- | --- | --- |
| `ok` | `boolean` | 是否读到**可用且完整**的正文 |
| `degraded` | `boolean` | 正文是否因未闭合块被部分吞掉 |
| `degradedReason` | `'body_truncated_unrecoverable'` \| `''` | 降级原因；空串表示未降级 |
| `renderNote` | `string` | 人可读说明。降级时为「正文在未闭合样式/脚本处被截断，提取不完整（不可当作完整证据）」 |

**注意**：`degraded:true` 时 `webRead` 会**先尝试一次 Chrome 渲染**再下结论（原来只在正文 <300 字时渲染）。所以 `requestImpl` 被注入的测试环境下不会真开浏览器，降级会直接返回——写夹具时请知悉。

---

## 四、如果你的研究需要判断"这页为什么读不到"

经 `web_read` 工具调用时，`tools.js` 把 `!ok` 统一抛成 `empty_body`（B 类观察，可能改为 `body_truncated`）。目前机器可读的 code 分不出"页是空的"与"正文被截断"，截断原因只在错误文案里。

**建议 D 的判读**：不要依赖 code 区分，改为直接调 `research.webRead()` 拿 `degraded` / `degradedReason`；或匹配错误文案里的「截断」二字。

---

## 五、夹具注意事项（会直接影响研究结论可信度）

1. **两页正文不要写成几乎一样**。系统有近似转载判据，两页文本雷同时会被（正确地）判为转载，而不是"两个独立来源"。要让两份正文实质不同、只有被引结论相同。
2. **正文长度 ≥ 120 字**，否则触发"正文过短"保护，`ok` 为 false，会把夹具缺陷误读成产品缺陷。
3. **构造截断样本时注意位置**：未闭合 `<style>`/`<script>` 在正文**之前**才会吞正文并触发 `degraded`；在正文**之后**是正常情况（正文保留、无降级）。
4. **不要用 `{...}` 形状的 JSON 当"CSS 泄漏"判据**。现在 JSON/对象字面量会保留，这是有意的。判泄漏请看是否出现规则块（含 `属性: 值` 且带典型 CSS 属性名）。

---

## 六、C 侧已覆盖、D 无需重复验证

| 项 | 证据 |
| --- | --- |
| 三处修复成立（CSS 上下文 / 分段密钥 / 正文损失上报） | `test/autonomy/z-microfix-04.test.mjs` 11 项 |
| 合并后接口契约（含降级不进学习链路） | `test/autonomy/c-baseline-sync02.test.mjs` 10 项 |
| 真实 10 个公开页面零误判零泄漏零误降级 | `logs/real-pages.log` |
| 注入四条与正常页不误判双向不回退 | 同上 |

---

## 七、坐标

```
灵核最终候选  cand/mindcore-v1.0-baseline-sync-02 @ 6a5efbd / Tree 1d1b43fa
生核侧文件    packages/companion/src/workspace/research.js（与 Z 的 b50a2057 逐字一致）
```

依赖方式不变：`import { Research, extractArticle, extractArticleEx } from '.../src/workspace/research.js'`。

如行核研究需要新的提取能力（例如保留表格结构、按段落给偏移量），请单独提需求——本轮不扩范围。
