# E-MEMCORE-OSS-COMPONENT-POC-01 · 精简报告

编号 E-MEMCORE-OSS-COMPONENT-POC-01｜2026-11-25 北京时间｜执行 WorkBuddy E｜裁决 ChatGPT｜独立验收 C。受控基线 <基线-脱敏>/Tree <基线Tree-脱敏>/cand/s2-e1-backup-integration-01，独立分支 feat/eco-memcore-oss-poc-01，隔离 PoC，未改正式基线、未写 memv2、未引入第二记忆库、未碰生产与凭证。

**结论。** 首选 EverOS 的"可重建派生索引 + BM25/ANN 双路 + 事实到 episode 来源映射"机制，已用原版组件真实执行 + 成熟库薄适配落地。最终建议：保留知遥现有 BM25 主干，借鉴 EverOS 的可重建索引与来源映射机制，ANN 待 embedding 授权再接，不整套引入。

**M0 基线与前置。** 基线 HEAD/Tree 精确命中、工作树干净、令牌未落盘；已读 R 的 R-MEMCORE-OSS-REUSE-BENCHMARK-01 公开任务书与脱敏报告，并从上游 v1.7.1 真实读取源文件 atomic_fact.py（185 行 AtomicFactRecaller）。复用清单齐全，EverOS 依赖闭包经镜像实测可解析。

**M1 原版与成熟库真实调用。** EverOS 1.7.1（Apache-2.0）在 Python 3.12 隔离 venv 安装成功（腾讯镜像），CLI 正常；真实施行原版 AtomicFactRecaller：sparse_recall("我的生日是哪天") 将 f1 排第一，facts_for_episodes 将 ep1 事实挂回、dense_recall 空向量正确降级（EVEROS_COMPONENT_CALL=OK）；everos demo --plain 端到端召回成功（2.92 秒、峰值 RSS 199MB、Source episode-2026-06-20.md）。成熟库 jieba 0.42.1 + rank_bm25 0.2.2 + numpy 2.4.6 完成真实检索。内存 index 仅作端口实现驱动原版，不改原版源码；demo 属静态场景，不代表需 LLM 的完整语义链已通。安装初因 pypi.org 直连超时曾缓慢，改用腾讯镜像后十秒完成，属网络环境问题非组件缺陷。

**M2 薄适配。** thin_adapter.py 实现 deriveIndex(facts)/recall(query,{scope,subject,topK,method})/rebuild(facts)：派生索引非真源、失效事实不入索引、结果带 sourceId 与 episode、无 embedding 时 ann/hybrid 明确降级为 bm25(degraded)，不编造向量性能；不写 memv2，K 写入以只读契约模拟并报告差异。BM25 与中文分词全部委托 rank_bm25 与 jieba 成熟库，本侧只写"来源映射 + scope 隔离 + 可重建 + 失效淘汰"的胶水层，不自写 BM25/ANN/数据库算法，符合"不是照论文重写"的要求。delete f3 后重建，"豆豆"查询由 [f3] 变为空、f3 不在 sourceIds、版本号随之变更，证明派生索引可完全由真源重建且删除无残留；失效事实 f5（valid=false）始终不入索引，不会被当作当前事实返回。

**M3 同条件对比。** 同一冻结数据 15 facts/12 queries、topK=5、纯词法：三引擎可答命中均 9/10、rank@1 均 9/10、无答案均 0/1；关键差异在隔离——知遥与薄适配对 userB 查工作域"服务器密码"正确隔离为空，裸用成熟库因无 scope 隔离而泄漏。失效更新"我住在哪个城市"三引擎均召不回"搬到上海"，暴露词法检索共同上限，需语义或 embedding。资源：知遥 Recall 检索 p50 0.040ms、重建 1.07ms、RSS 56.4MB、零新增依赖；薄适配 p50 0.066ms、RSS 97.9MB、98MB/7 包；EverOS 全栈 547MB/82 包、demo RSS 199MB。真实模型 hy4flash/DeepSeek-V4.1-Flash 无凭据额度，真实语义评分 BLOCKED；embedding 未授权，ANN 降级，不伪排名。评测口径：三引擎只比较命中、排名、隔离与资源，不比较绝对分数（不同 BM25 变体分数量纲不可横比）；无答案与失败关闭场景单列，避免以词法噪音冒充能力；两次运行取中位以抵消抖动。

**证据链与可复现。** 每条结论均可复核：环境版本（Node v22.13.1、Python 3.11.1/3.12.3、everos 1.7.1、numpy 2.4.6）落于日志；EverOS 安装、CLI、组件调用、demo、薄适配、删除重建、三引擎对比、资源测量均有原始输出（logs/）与结果（results/），全部可重跑。源码以 tar 包交付并逐文件 SHA-256 校验；三套检索器吃同一份冻结合成数据，指标口径一致（topK=5、同一 12 query 与 15 facts）。数据全为虚构，零真实记忆。C 可按接收指南独立重跑并比对，无需信任本侧结论。

**M4 推荐与回退。** 保留 memv2 真源与知遥 BM25 主干（性能与零依赖占优）；借鉴 EverOS 可重建索引 + 来源映射（薄适配已实现，删 f3 后 rebuild "豆豆"→[]，no_residue=true，失效 f5 不入索引）；ANN 条件引入；整套框架（EverOS/Hindsight/Graphiti/Mem0/Letta）不引入。全部为新增文件，未改基线已有文件；回退即丢弃分支与隔离 venv，生产零变更。

**门禁。** G0 基线与许可、G1 原版/成熟库调用、G2 隔离/来源/删除重建正负对照、G3 同条件指标与资源、G4 薄接口与回退，均通过；真实模型与 embedding 未验证项如实标记 BLOCKED，未冒称。

**已知限制。** 合成数据规模小，仅验证机制与工程能力，不代表生产吞吐；真实语义链路与 embedding 未验证（无凭据/授权）；q3、q10 属词法检索固有上限而非本次回归；EverOS 内存 index 为端口实现驱动原版，demo 为静态场景，不等于需 LLM 的完整记忆链已通。上述均如实标注，不冒称。

**交接与风险。** 交 C 独立验收（排队先 S2），必要时经真机与协调方裁决、主人授权后集成。全程零生产变更、零真实凭证接触、零跨侧写入。下一决策项：是否批准 embedding 与 hy4flash/DeepSeek 额度以把定性对比升级为真实语义端到端对照，以及是否在 embedding 到位后接入 EverOS 式双路 ANN。
