> GitHub 公开归档版（脱敏）：内部资料库节点 ID 已略去，完整原件见 WorkBuddy C 侧资料库批次。

# C-MEMCORE-OSS-COMPONENT-ACCEPTANCE-01　C 侧独立验收报告（R2·实物复现版）

- 任务编号：C-MEMCORE-OSS-COMPONENT-ACCEPTANCE-01（修订 R2）
- 执行：C 侧独立窗口｜时间：2026-11-25（北京时间）
- 依据：《AI 工作流统一规则》v1.4；任务书 M0–M4 与 G0–G4
- 验收对象：E-MEMCORE-OSS-COMPONENT-POC-01（隔离 PoC）
- 前序状态：R1 报告判定 BLOCKED（E 正式批次对本账号不可达）。本轮改为**按节点 ID 直达**后批次已取得，故 R1 的阻断结论**作废**，本文件取代 R1。

---

## 一、结论

| 项 | 结论 |
| --- | --- |
| 总体 | **CONDITIONAL PASS（有条件通过）** |
| G0 交付与真实原版调用可复现 | **PASS**（包哈希 9/9、12 查询逐条复现、EverOS 组件本侧独立跑通；分支坐标 1 处不可核） |
| G1 派生索引来源 / 删除 / 隔离 | **PASS** |
| G2 对照口径可审计、无伪成绩 | **CONDITIONAL**（数字真实可复现；两处口径需统一，见 D-2/D-3） |
| G3 模块级融合建议与维护成本 | **PASS** |
| G4 零生产改动与隔离合规 | **PASS** |

一句话：**E 的核心数字全部可独立复现到逐条一致**；不整体引入 EverOS、保留知遥 BM25 主干、只借鉴"可重建索引 + 来源映射 + 失效软删"的建议成立。需修正的是三处**表述与坐标**问题，不是结论问题。

---

## 二、M0　批次接收与完整性（实测）

### 2.1 批次坐标（节点级直达）

| 对象 | 坐标 |
| --- | --- |
| E 侧资料库 | `（内部节点，略）` |
| 本批次文件夹 | `（内部节点，略）`（2026-11-25_E-MEMCORE-OSS-COMPONENT-POC-01-开源记忆组件复用PoC） |
| 包内文件 | 完整报告 `（内部节点，略）`、精简报告 `（内部节点，略）`、复用清单 `（内部节点，略）`、校验清单 `（内部节点，略）`、完整性报告 `（内部节点，略）`、接收指南 `（内部节点，略）` |
| 源码与日志包 | `（内部节点，略）` |

### 2.2 包完整性与逐文件哈希

| 检查 | 结果 |
| --- | --- |
| 源码包下载 | 15,462 字节，`2026-11-25_E-MEMCORE-OSS-POC_源码与日志包.tar.gz` |
| 包 SHA-256 | `5229e43c5ec1256bb804963c5dc66c3709ffe4975a4bd613e7324f795a3b2d8d` —— **与校验清单声明值逐字一致** |
| 解包 | 成功，26 个文件（`thin/ harness/ everos/ data/ results/ logs/` + `compare.py`） |
| 逐文件哈希 | 校验清单列出的 9 个关键文件 **9/9 一致**（`thin_adapter.py`、`run_thin.py`、`bench_thin.py`、`del_rebuild.py`、`node-recall.mjs`、`bench-node.mjs`、`everos_sparse_recall.py`、`data/synth.json`、`compare.py`） |

### 2.3 基线复核

- 声明基线 `37dbbb3071e4a61d71c08e3159d47a4034d86e13` / Tree `9fc8ac7aa8d77a406af68400c881400395f8198c` / `cand/s2-e1-backup-integration-01`：**在 Gitee 上可取得**，分支 HEAD 即该提交。
- 独立复现所用 `packages/companion/src/recall.js` 直接取自该提交（16,084 字节），**未使用生产机工作区**（生产为 `ded6769`，与声明基线不同，见 §七 说明）。

### 2.4 坐标缺陷 D-1（P3，不影响结论）

接收指南与校验清单均写明"分支 `feat/eco-memcore-oss-poc-01`"。实查 Gitee 该仓库共 19 个分支，**不存在此分支**（`Branch does not exist`），含 `eco`/`memcore` 的分支只有 `cand/eco-e1-cfix-01`、`cand/memcore-s1-cfix-01` 等 5 个，均非该分支。

结论：该分支为 E 侧**本地分支，未推送**，坐标不可核。源码包自包含，M0 复现不依赖该分支；但按规则 5.1「报告声称产物存在不等于实际可获取」，建议后续交付**不要把未推送分支写成可获取坐标**，或明确标注"仅本地、不可获取"。

---

## 三、M0/M1　原版组件与成熟库的真实性（独立核实）

### 3.1 EverOS 原版包

| 项 | 实测 |
| --- | --- |
| PyPI 可得性 | `everos==1.7.1` 存在，wheel 922,311 字节 |
| 许可证 | `License: Apache-2.0`（含 LICENSE/NOTICE）——与 E 声明一致 |
| 运行约束 | `Requires-Python: >=3.12`（Python 3.11 下 pip 直接判"无匹配版本"，可解释 E 为何单开 3.12 venv） |
| 组件坐标 | `everos/memory/search/recall/atomic_fact.py` 第 61 行 `class AtomicFactRecaller`；第 76 行 `sparse_recall`、第 86 行 `dense_recall`、第 104 行 `facts_for_episodes` —— 与 E 声称的导入路径、类名、方法名**逐项吻合** |
| 依赖规模 | METADATA 声明 38 个直接依赖（含 `lancedb`、`pyarrow`、`textual`），与"547MB/82 包"同量级 |

### 3.2 本侧独立运行（已跑通）

在 Python 3.12.3 隔离 venv（519 MB，与 E 声明的 547 MB 同量级）安装 `everos==1.7.1` 后，直接执行 E 的 `everos/everos_sparse_recall.py`，输出与 E 的 `logs/everos-component.txt` **逐行一致**：

```
[everos] recaller kind: atomic_fact | memory_type: episodic
[sparse_recall] candidates: [('u_owner_f1', 5.0), ('u_owner_f2', 2.0), ('u_owner_f3', 2.0), ('u_owner_f5', 1.0), ('u_owner_f6', 1.0)]
[facts_for_episodes] buckets: {'ep1': ['u_owner_f1', 'u_owner_f2'], 'ep2': ['u_owner_f3']}
[dense_recall empty-vector] candidates: []
EVEROS_COMPONENT_CALL=OK
```

结论：**E 关于"真实安装了 EverOS 1.7.1 并调用了原版 AtomicFactRecaller"的陈述成立**——包、版本、许可证、模块路径、类与方法全部对得上，且本侧独立复现得到 `EVEROS_COMPONENT_CALL=OK`，候选顺序、分数、buckets 与 E 日志一致。这是**组件级可执行性**的证据，不是检索质量证据（见下条）。

### 3.3 必须讲清的"复用深度"（D-3，P3）

这一点 E 报告写得清楚，但结论段的"EverOS 9/10"容易被误读，本侧明确区分：

1. **代码复用 = 0 行**。`thin_adapter.py` 头注释自述"不复制 EverOS 源码，用成熟现成库组合复现其机制"——实际是 `jieba` 分词 + `rank_bm25.BM25Okapi` + 自写 `DerivedIndex`。EverOS 在薄适配里**没有被 import**。
2. **EverOS 的"真实执行"仅在 `everos_sparse_recall.py`**：它 import 原版 `AtomicFactRecaller`，但把索引换成本文件内的 `InMemoryAtomicFactIndex`——`sparse_search` 的打分是**词项重合计数**（`sum(1 for t in toks if t in ts)`），不是 BM25，也不是 EverOS 的原版索引。因此该脚本证明的是**接口兼容与可执行性**，不证明 EverOS 的检索质量。
3. **`9/10` 是"薄适配（rank_bm25）"的成绩**，三引擎同为 9/10 也正说明：在这套 15 facts 数据上，检索效果由 BM25 决定，而非由 EverOS 决定。真正测过 EverOS 原版检索质量的是 R 侧（6 查询，recall@5 1.00），两套数据集不可合并——R1 报告已指出，本轮复现后该判断不变。

---

## 四、M1　派生索引 / 来源映射 / 删除重建（实测）

代码审读（`thin/thin_adapter.py`，9/9 哈希一致）确认：

- `derive(facts)`：`valid=False` 不入索引；返回 `{records, version, sourceIds}`，`version` 为全量 facts 的 sha256 前 16 位，**任一 fact 变更即换版**，可用于缓存失效判定。
- `recall(query, {scope, subject, topK, method})`：**先按 scope 过滤再打分**（`_scope_ok`：owner 全见 / user 只见自己 uid 或 `where:uid`），结果携带 `sourceId` + `episode` + `version`——事实可挂回来源，符合 EverOS `dual parent_id` 的机制要点。
- `rebuild(facts)`：全量重建，删除/撤回后无残留。

**删除重建复跑（`del_rebuild.py`）结果与随包结果逐字段一致**：

```json
{"positive_before_delete_豆豆": ["f3"], "negative_after_rebuild_豆豆": [],
 "no_residue": true, "invalid_fact_f5_in_index": false,
 "version_before": "daa206a41ebfdbdb", "version_after": "603b52693c9a7259", "records_after": 13}
```

`version_after` 与 E 交付值**完全相同**（`603b52693c9a7259`），说明重建是确定性的，不是"跑一次碰巧"。

### 缺陷 D-2（P2，口径问题）

同一批数据里还有第二处删除用例（`run_thin.py` 内联的 q4）：删 f3 后查"我养了什么猫"，返回 `['f1','f4']`，被判 `ok=False`。**随包 `py-thin.json` 里本身就是 `ok=False`**，E 没有改它，这点是诚实的；但 E 报告与摘要只引用 `del_rebuild` 的"豆豆→[]"，会给人一种"删除重建全绿"的印象。

问题在判据设计：q4 的 `expect` 为空，被判据解读为"结果必须为空"，而删 f3 后仍会命中 f1/f4（与"猫"无关但共享词项）。**"无残留"应判"被删事实是否仍出现"，而不是"结果集是否为空"**。建议后续统一：删除用例断言 `deleted_id ∉ hits`，另设独立的"无答案阈值"用例。

---

## 五、M2/M3　同条件对比与资源（实测复跑）

### 5.1 复跑方式

- 知遥侧：从基线提交取整棵 `packages/companion/src`（219 个 JS 文件），跑 `harness/node-recall.mjs`。
- 薄适配/裸用：`jieba 0.42.1` + `rank_bm25 0.2.2` + `numpy 2.3.5`（E 用 2.4.6，本侧环境版本略低，不影响 BM25 结果）。
- 数据：`data/synth.json`（15 facts / 12 queries / topK=5），哈希与清单一致。

### 5.2 结果：**12/12 查询逐条一致，`compare.md` 逐字节一致**

| 指标 | 知遥 Recall | 薄适配 | 裸用成熟库 |
| --- | --- | --- | --- |
| 可答命中 | 9/10 | 9/10 | 9/10 |
| rank@1 | 9/10 | 9/10 | 9/10 |
| 无答案正确 | 0/1 | 0/1 | 0/1 |
| 跨域隔离正确 | 1/1 | 1/1 | **0/1（泄漏）** |
| 失效值泄漏 | 0 | 0 | 0 |

复跑后 `results/compare.md` 与随包文件 `diff` 无差异；`py-thin.json`、`py-naive.json`、`node-recall.json` 三份逐查询比对**零差异**（含 `hit`、`expectRank`、`hits` 顺序）。

### 5.3 资源复跑（600 次采样）

| 引擎 | 本侧 p50 / p95 | E 声明 p50 / p95 | 本侧 RSS | E 声明 RSS |
| --- | --- | --- | --- | --- |
| 知遥 Recall | 0.0235 / 0.0541 ms | 0.040 / 0.109 ms | 60.2 MB | 56.4 MB |
| 薄适配 | 0.047 / 0.064 ms | 0.066 / 0.104 ms | 105.5 MB | 97.9 MB |

同量级、同结论（知遥在延迟与内存上均优于薄适配）。重建：知遥 0.823 ms；薄适配首建 419 ms（含 jieba 冷启动）。

### 5.4 结论

- **"不整体引入 EverOS/Hindsight、保留知遥 BM25 主干、只借鉴机制"的建议成立**，且理由（依赖 547MB/82 包、第二真源风险、无 embedding 时 ANN 无收益）经本侧独立复核。
- **q3（"我住在哪个城市"）三引擎全 miss、q10（"我喜欢什么颜色"）三引擎全误召回**，均属纯词法共同上限，与 R 侧结论一致，**不是本次回归**。
- **隔离差异是本轮唯一真实分水岭**：裸用成熟库把 userB 的"服务器密码"查给了别人（q11 泄漏），知遥与薄适配都为空。这条支撑"必须加 scope 隔离层，不能裸用成熟库"。

---

## 六、M4　S2 测试工具 B-5 收尾（本轮补做）

### 6.1 缺陷复核：**未归档，且可现场复现**

在测试工具线提交 `d9f8f645ab10fd7e383ac773820df6d9d6d58fe1`（`cand/s2-g5-fixture-fix-01`）中：

- `scripts/lib/g5-judges.mjs`：`export function judgeL02(text)` —— 形参是**字符串**。
- `scripts/s2-live-models-fix01.mjs` 第 340 行：`const j = sc.judge({ text: askRes.text });` —— 传的是**对象**。
- 第 95 行场景注册：`judge: judgeL02,` —— 裸函数直接注册，于是 `String({text})` 得到 `"[object Object]"`。

本侧实测（不发请求、不调模型）：

```
judgeL02('你现在住在杭州——上个月从成都搬过去的。')  → {"ok":true,"why":"new_current_old_historical"}
judgeL02({ text: '同上' })                          → {"ok":false,"why":"no_new_value"}
```

**任何正确回答都会被判 `no_new_value`**，与 Z 侧描述完全一致。根因不是判定规则，而是**场景表签名不统一**：其余场景都是 `(r) => judgeX(r.text, …)` 包装，只有 L02 注册了裸函数。

### 6.2 归档（本轮已完成）

按 M4"单独在测试工具线归档一行等价修正 + 脚本级正负例回归"执行：

| 项 | 内容 |
| --- | --- |
| 分支 | `cand/s2-g5-b5-callsite-fix-01`（基于 `d9f8f645` 新建，**未改动任何已有分支**） |
| 修正提交 | `d178808bf5ca`：第 95 行改为 `judge: (r) => judgeL02(r.text),` —— **仅改调用点，判定规则与断言语义一字未动** |
| 回归提交 | `9b6fcf8a5292`：新增 `scripts/lib/g5-judges-callsite.test.mjs` |
| 测试结果 | `node --test` **5/5 通过** |

回归用例覆盖：

1. 正例：字符串入参 → `new_current_old_historical`；
2. 负例：对象入参 → `no_new_value`（把 B-5 现象固化成用例，防止再踩）；
3. 负例：把旧值当现在 → `old_as_current`；
4. 脚本级：真实脚本中 L02 必须是 `(r) => judgeL02(r.text)` 形式，且不得回退为裸注册；
5. 脚本级：**任何场景都不得裸注册 `judge` 函数**（防同类缺陷复发）。

边界遵守：**未重开 S2 门禁、未调用任何模型、未改产品基线**；Z 侧原始通过/失败证据保持原样（本侧只读取、未改写）。

---

## 七、零触碰与生产影响

| 检查 | 结果 |
| --- | --- |
| 包内文件归属 | 26 个文件全部位于 `oss-poc/` 结构内，**不含任何基线已有文件**，与"仅新增"自述一致 |
| 基线源码 | 仅**只读**取得 `packages/companion/src`（来自 Gitee 基线提交），未写回、未修改 |
| 生产 | **零变更**。本轮对生产机只做只读查询，未部署、未重启、未改配置 |
| memv2 | 未写、未建第二记忆库（PoC 侧自述，本侧未发现反证） |
| Gitee | 仅**新增一个分支** `cand/s2-g5-b5-callsite-fix-01`（2 个提交，1 改 1 增），未碰 `release/*`、未强推、未合并 |

> 附带说明（与本任务无关，供参考）：只读核对时发现生产机 `release/z1.4.7` 本地 HEAD `ded6769` 领先 Gitee 远端 55 个提交，且工作区有 6 个未提交改动。已在《知遥记录》第 38 节单列，交 Z/ChatGPT 处置，本任务未介入。

---

## 八、缺陷与建议清单

| 编号 | 级别 | 内容 | 处置建议 |
| --- | --- | --- | --- |
| D-1 | P3 | 接收指南/校验清单写的 `feat/eco-memcore-oss-poc-01` 在 Gitee 不存在（本地分支未推送），坐标不可核 | 后续交付对未推送分支注明"仅本地"，或干脆不列为坐标 |
| D-2 | P2 | 删除用例判据不统一：`run_thin` 内联 q4 判 `ok=False`（要求结果为空），而 `del_rebuild` 的"豆豆"例判 `no_residue=true` | 统一为"被删 fact id 不出现在 hits"；无答案另设阈值用例，不混用 |
| D-3 | P3 | 结论段"EverOS 9/10"易被误读为 EverOS 原版检索成绩；实为薄适配（rank_bm25）成绩，EverOS 只做了组件级 import+执行 | 表述改为"薄适配 9/10；EverOS 原版检索质量见 R 侧 6 查询实测，两数据集不可合并" |
| — | 已知限制 | 真实模型/embedding 无授权 → 语义端到端未验证；q3/q10 词法上限；合成数据 15 facts 不代表生产规模 | 保留 BLOCKED 标注，不升级为"已验证" |

---

## 九、模块级裁决（M3 建议，本侧认可并补充）

| 模块 | 裁决 | 本侧意见 |
| --- | --- | --- |
| 结构化真源 | 保留 memv2 | 同意，不换源 |
| 检索主干 | 保留知遥 Recall 的 bigram BM25 | 同意：延迟/内存/零依赖均优于薄适配；本侧复跑数据同向 |
| 可重建派生索引 + 来源映射（sourceId/episode） | **采纳**，按 EverOS 机制但用自有实现 | 同意：机制收益真实（version 可判定缓存失效、删后重建确定），且不引入 547MB |
| 双路 ANN | 待 embedding 授权后再评估 | 同意：本轮 `ann` 明确降级，无向量收益证据，不得据此否决或采纳 |
| 整套框架（EverOS/Hindsight/Graphiti/Mem0/Letta） | 不引入 | 同意：第二真源与重依赖风险大于收益 |
| Hindsight 语义抽取 | 暂缓 | 同意（需模型授权） |

**给 K 的接口对齐要点**（不代为开发）：派生索引的 `version` 应对齐 memv2 的事实版本（而非 PoC 内的 facts 快照 sha），否则真源更新后派生层无法判定失效；`sourceId` 需能回溯到 memv2 事实与 episode 双向映射。

---

## 十、交付与坐标

| 交付物 | 位置 |
| --- | --- |
| 本报告（R2） | C 侧资料库批次文件夹（见进度记录链接） |
| 精简报告（≤1300 字） | 同上 |
| SHA256 清单 | 同上 |
| M4 修正与回归 | Gitee 分支 `cand/s2-g5-b5-callsite-fix-01`（`d178808bf5ca` / `9b6fcf8a5292`） |
| GitHub 辅助归档 | `docs/20-reports/C/`（脱敏版） |

回退：E 侧 PoC 丢弃 tar 与隔离 venv 即可（零生产影响）；M4 修正丢弃该分支即可（未合并、未部署）。

---

**验收人**：C 侧｜**结论**：CONDITIONAL PASS｜**不宣称上线、不宣称 S2/S3 关闭**｜**生产影响：零**
