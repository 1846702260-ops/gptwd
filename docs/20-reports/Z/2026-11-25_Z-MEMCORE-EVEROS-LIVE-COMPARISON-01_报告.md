> 本文件为公开仓库脱敏版：已移除内部资料库节点/批次链接、生产地址与配置细节、认证字段、私人数据；完整原件与原始证据保留在 WorkBuddy 正式交付。

# Z-MEMCORE-EVEROS-LIVE-COMPARISON-01 独立实验报告

**EverOS 原版真实模型记忆链路与最小对照验证**

- 任务编号：Z-MEMCORE-EVEROS-LIVE-COMPARISON-01
- 执行：Z（独立实验窗口）｜日期：2026-11-25（北京时间）
- 承接：R-MEMCORE-OSS-LIVE-COMPARISON-02
- 边界：不部署、不迁移、不改生产默认模型/桥/代理/账号/配置/自动部署、不新增费用授权

---

## 一、结论摘要

1. **真实模型链路已打通**：EverOS 1.7.1 在隔离 venv 内，用 `hy4flash-free` 完成 6 条合成记忆的真实 `add → flush → 抽取 → Markdown 落盘 → BM25 检索`，6/6 成功，全程无替身。
2. **同条件对照：质量持平，成本差 9 倍**。同一语料、同一查询、同一模型下，EverOS 与知遥 37dbbb30 的 **top-1 均为 1.00、recall@5 均为 0.90、最终答案均为 6/6**；RSS 为 **435.9 MB vs 47.6 MB**。
3. **R 轮「EverOS 检索更优（1.00 vs 0.90）」在真实抽取下消失**。原因是真实模型把 m6 的「tokio」改写为「Rust 异步运行时」，关键词检索失配。R 的优势来自 FakeLLM 把原文原样当 episode 的夹具效应。
4. **删除语义分层，容易被误用**：`delete(user_id + session_id)` 按设计只清该会话的 episode 与 atomic_fact（检索 0 命中、正文「成都」消失），**但保留 `user.md` 画像**，画像里仍是可读的「从杭州搬到成都」；只有 `user_id` 单独删除才会连画像一起移除。这不是 bug，是作用域分层；风险在于调用方把「删这段对话」当成「删这个人的数据」。
5. **裁决建议**：**不建议整体采用 EverOS**。无质量优势，运维面（554 MB venv、81 包、436 MB RSS、12 个 LLM 角色）却显著变大。可复用其机制（`parent_id` 物理溯源、删除作用域分层），但集成时必须把删除语义在 UI 与流程上区分清楚。

---

## 二、G0 版本、依赖、授权与隔离

| 检查 | 结果 |
| --- | --- |
| EverOS 固定版本 | 1.7.1（PyPI）；lancedb 0.34.0 / pyarrow 26.0.1 / jieba 0.42.1 / tiktoken 0.14.0 / pydantic 2.14.4 / httpx 0.28.1 |
| Python | 3.12.3（满足 ≥3.12 硬要求） |
| tiktoken 缓存 | 预置 3,613,922 B，sha256 `446a9538…1a2d` |
| 授权模型 | `hy4flash-free`、`deepseek-v4.1-flash`，经既有桥；**未读取任何上游 token** |
| 隔离 | 实验在本地沙箱；生产仅经回环 SSH 转发暴露桥端口；生产零安装、零占用 |
| 语料 | R `corpus.json` 原样复用（6 记忆 / 6 查询） |
| 知遥基线 | 37dbbb30 / Tree **9fc8ac7a**（独立 clone，与任务书一致） |
| 许可证 | EverOS wheel 内 `METADATA: License: Apache-2.0`，含 `LICENSE` 与 `NOTICE`（一手核验，非二手引用） |

**判定：PASS**

---

## 三、G1 原版真实抽取与存储闭环

| 项 | 结果 |
| --- | --- |
| dry-run（m1） | 19.54 s，11 次真实调用，episode 1,267 B |
| 全量 6 条 | 8.93–21.31 s，**6/6 ok** |
| 落盘 | episode 4,735 B（6 entry）、atomic_fact 9,432 B、user.md 881 B |
| 溯源 | 每 episode 带 `parent_id: mc_*`，每 fact 带 `parent_id: ep_*` |
| 抽取质量 | 正确识别「杭州→成都」为事实更新；「忆核记忆模块 / Rust 偏好」为持续事实 |
| 失败 | 4xx/5xx **0**；`invalid_reply_retries=0` 下 5 类必需 schema 一次通过 |

**判定：PASS**

---

## 四、G2 同输入同模型对照

| 指标 | EverOS 1.7.1 | 知遥 37dbbb30 |
| --- | --- | --- |
| 检索 top-1（5 题） | **1.00** | **1.00** |
| recall@5 | **0.90** | **0.90** |
| 最终答案（6 题含无答案） | **6/6** | **6/6** |
| 检索延迟 | 7–27 ms | 0.4–2.4 ms |
| RSS | 435.9 MB | 47.6 MB |

逐题：q1 成都 ✓、q2 Rust ✓（双方 m6 均漏召回）、q3 2026-03 + 王 ✓、q4 不喜咖啡 ✓、q5 地铁 ✓、q6 **双方均拒答「不知道」**。

deepseek-v4.1-flash 子集（m1 + q1/q6）同向：q1「成都」、q6「不知道」。

**判定：PASS（附条件）**——条件：样本 6 条，不外推；embedding/hybrid/rerank BLOCKED（见第七节）。

---

## 五、G3 资源、用量与可维护性

| 项 | 数值 |
| --- | --- |
| venv / 包数 | 554 MB / 81 包 |
| 6 条存储 | 3.6 MB |
| 启动 | 0.72 s（索引已建） |
| 真实调用 | 80 次（代理）+ 5 次探针 = 85，**在 100 次预算内** |
| tokens | prompt 73,029 / completion 56,005（含 reasoning） |
| 延迟 | p50 2.4 s / max 26.9 s |
| finish=length | 12（异步任务截断，主链路无损） |
| 客户端取消 | 3（22–25 s 长任务） |

**判定：PASS**

---

## 六、G4 生产零变更与 P0 观察

（见 08 号证据，21:02 自然续期观察后定稿）

---

## 七、删除语义、持久化与误用风险

- 会话级删除（`user_id + session_id`）：`count=6`，检索 0 命中，episode/atomic_fact 清空、「成都」消失 → **PASS**
- 画像层：`user.md` 881 B 逐字节未变，仍含「从杭州搬到成都」→ **按设计保留，非 bug**
- 用户级删除（`user_id` alone）：`user.md` **整体消失** → 分层完整
- 代码依据：`_remove_session` 只删 episode/atomic_fact；`_owner` 才 `erase_owner(["user_profile"])`
- 风险定性：**B 类「易误用」**——不是缺失能力，而是语义陷阱
- 跨进程：二次启动索引重建后仍可召回 → **持久化 PASS**
- 夹具坑：`add/flush` 后约 0.3 s 内检索可能为空（cascade 未完成），需轮询就绪

---

## 八、未验证（BLOCKED / 未做）

embedding / hybrid / ANN（无 provider）、rerank（无 DeepInfra key）、LongMemEval 正式成绩（需预算，oracle 不可冒充）、规模化性能（6 条不可外推）、多用户并发、生产迁移。

---

## 九、给 ChatGPT 的裁决证据

| 路径 | 证据 | 建议 |
| --- | --- | --- |
| 整体直接采用 | 质量无优势；+554 MB、+81 包、+388 MB RSS、12 角色 | **不建议** |
| 源码抽取（BM25 逻辑） | 双方 top-1 已持平，摘取收益低 | 低优先 |
| SDK / 侧车 / 薄适配 | 需常驻服务与 embedding | 暂缓 |
| **现有实现 + 机制补齐** | `parent_id` 溯源、删除分层（需修画像残留）、无答案阈值 | **推荐** |

---

## 十、风险汇总

**B 类**：R1 删除不覆盖画像；R2 真实抽取改写削弱关键词召回；R3 推理模型预算不足返回空 content。
**C 类**：R4 长任务客户端取消；R5 写入到可检索的最终一致窗口；R6 资源成本。

> 本报告只代表本轮独立实验结论，不构成采用、迁移或上线授权；生产切换与新增费用须主人另行明确授权。
