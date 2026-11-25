> GitHub 公开归档版（脱敏）；完整原件见 WorkBuddy 共享资料库 C 侧批次。

# C-MEMCORE-OSS-COMPONENT-ACCEPTANCE-01 · 精简报告

**任务**：验收 E 侧开源记忆组件隔离 PoC（E-MEMCORE-OSS-COMPONENT-POC-01），结合 R 侧实测（R-MEMCORE-OSS-LIVE-COMPARISON-02），裁决哪些原版组件可复用、哪些现有实现保留、哪些改进进 S3。非生产发布，不强制合并，不重实现已有算法。

**结论：BLOCKED（G0 硬阻断），附证据级条件性结论。** 唯一阻断点是 E 的正式 WorkBuddy 批次对本执行账号不可达。共享工作区 `共享工作区（内部坐标略）` 返回 `role="invalid"`（12607 permission denied），无法列目录或读任何 E 批次节点；本机文件系统无 PoC 代码；GitHub `gptwd` 只有 E 的脱敏精简报告，无源码、tar、SHA256。任务书 M0 明示"公开 E 报告脱敏、禁止猜测坐标"，故 M0 要求的"接收批次、递归定位四件套、核对真实 HEAD/Tree/Parent、逐文件哈希、独立目录复现"无法执行，我没有用猜测补位。

**可达性已穷尽**：个人空间全量 7 节点无此批次；共享空间根与工作区根文件夹两次列目录均被拒；资料库语义检索三次无命中；本机 `find` 检索 thin_adapter/deriveIndex/everos/AtomicFactRecaller 无结果；GitHub 递归树 109 blob 仅两个脱敏文档；GitHub 亦无写入凭据（gh 未登录），归档标"待补"。全程未提权、未绕过。

**公开证据独立审理**：E 与 R 是两份独立来源（E 做薄适配 PoC，R 跑完整服务实测），核心结论高度收敛，互证方向可信——均主张"不整体引入 EverOS/Hindsight，保留知遥 BM25 主干，只做机制复用"。E 称真实执行 EverOS 1.7.1（Apache-2.0）`AtomicFactRecaller`：sparse_recall 将 f1 排第一、dense_recall 空向量正确降级、demo 2.92s/RSS199MB；thin_adapter.py 实现 deriveIndex/recall/rebuild，删 f3 后"豆豆"→[]、no_residue=true、失效 f5 不入索引；15 facts/12 queries 下三引擎命中 9/10、rank@1 9/10、无答案 0/1，知遥与薄适配对 userB 工作域"服务器密码"正确隔离为空、裸成熟库泄漏。R 用 6 条/6 查询实测：top-1 双方 5/5，recall@5 知遥 0.90 vs EverOS 1.00（差异源于文档粒度），无答案 q6 双方均噪音（都缺阈值），延迟 0.07–1.47ms vs 7.44–26.84ms，RSS 47.6MB vs 419MB；R 还定位并修复了上轮"端口不监听"根因（tiktoken 无超时下载，属环境非缺陷）。两报告一致：真实模型（无 hy4flash/DeepSeek 凭据）与 embedding（未授权）均 BLOCKED，未以 mock 冒充。

**四项不可核对项**：①数据集口径不可合并（E 用 15/12、R 用 6/6，E 未说明"冻结数据"来源，不能把 9/10 与 5/5 拼成同一基准）；②E 的"9/10"缺逐题明细与失败题性质；③E 坐标全脱敏，无法核对是否真等于 `37dbbb30/9fc8ac7a`；④E"源码 tar+SHA256"无具体哈希值，公开侧无法比对；另"原版复用深度"（真实算法 vs 套壳端口）须看源码才能判。

**门禁**：G0 BLOCKED（批次不可达）；G1/G2/G3/G4 均 CONDITIONAL——公开层面无一显示失败或伪造，但证据不足以独立确认。总体不是 FAIL，而是"证据不足"。

**解除条件（任一）**：给出 E 批次 `/space/d/<节点ID>` 链接或授予共享空间读权限；或将四件套（源码 tar、SHA256、包装验证、接收指南、results/logs）复制到我可读位置。到位后按 M0→M1→M2→M4 顺序完成实物复现，预计可达 CONDITIONAL PASS（真实模型/embedding 的 BLOCKED 属合理上游限制，不否定机制价值，但"语义端到端未验证"须如实保留，不得升级为已验证）。

**边界**：本侧零代码改动、零生产变更、零真实凭证接触、零跨侧写入。双进度记录与《知遥记录》待追加；GitHub 归档待补。
