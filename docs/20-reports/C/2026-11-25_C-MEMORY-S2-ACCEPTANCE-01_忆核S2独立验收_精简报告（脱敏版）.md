# C-MEMORY-S2-ACCEPTANCE-01 · 忆核 S2 独立验收精简报告

执行 C｜2026-11-25（北京时间，24 小时制）｜上游 K-MEMORY-S2-CONTEXT-LIFECYCLE-01｜适用规则《AI 工作流统一规则》V1.4

## 一、结论
候选 `08a81f1a7011e2191d64868a839d70e59e42a1c3`（Tree `2d185369`，Parent `86dd0b9`）经 C 侧独立复现与定向验收，**条件通过并受控入库**：G0、G1、G2、G3、G4、G6 六项门禁通过；G5 双真实模型验证因本环境无凭证、无授权模型桥而 BLOCKED（能力阻断，非代码缺陷，按任务书如实记录，未用别名或 mock 冒充）。已把候选以独立 cand 引用推送 Gitee 并回读三坐标，未推 release、未强推、未连生产。有两项须上报 ChatGPT 裁决、不得静默关闭：并发归档 CAS 竞态（A 类边界）、G5 补测安排。

## 二、M0 交付与基线核验
从 K 侧正式资料库批次取得四件套、00–06 号报告、bench/payload 证据、5 份测试日志与 SHA256SUMS，共 11 个文件全部下载成功。逐文件 SHA256 校验：自包含 bundle、增量 bundle、patch、bench.json、payload-evidence.json 与全部 TAP 日志一致（7 个文档以在线 doc 存储，无法逐字节还原，属预期）。自包含 bundle verify 通过、历史完整。两种方式复现三坐标一致：自包含 bundle 克隆（保留 `-b`）得 HEAD `08a81f1`、Tree `2d185369`、Parent `86dd0b9`，工作树干净；基线 `86dd0b9` 上 `git am --3way` 重放补丁后 Tree 同为 `2d185369`（commit sha 因提交者/时间不同属正常）。先读实际 diff 再定相邻回归：候选 13 文件、+2626/−42，比 K 报告「修改6+新增6」多一处 `contextAssembler.js`（两行均为摘要 rollup 要过相关性闸，与 K 报告 M2 表述一致，属计数遗漏非隐藏改动），据此把记忆层全量、assembleContext 相关性与工作档、brain 历史装载与守卫列为相邻回归。备份候选 `ec1e2a9`（分支 `cand/p0-backup-reliability-cfix-01`）全程未触碰，入库后指针未变。

## 三、M1 预算与运行链（G1 通过）
沿真实入口核对 `coreTurn → planContextBudget → beginTurn → recoverArchived → beforeReply/assembleContext（记忆子预算）→ brain.reply（历史子预算）→ #guardMessages 整包复核 → #call`。唯一决策者 `contextLedger.planContextBudget()` 成立：窗口→输出预留（夹 [512,25%窗口]）→安全余量（6%，≥256）→固定区（system+工具 schema+本轮请求+协议开销）→flex→记忆/摘要/历史三份额，K 发现的两本账合起来顶穿缺陷已收口。记忆子预算不放大旧配置；未知窗口用 32000 保守值并标 `windowSource='unknown-conservative'`，不硬编精确上限；估算口径声明为结构化估算、不等于计费口径，真实 usage 到位后由 tokenCalibration 校准并记误差。不可裁超限返回 `context_overflow` 且不发请求（E06 stub fetch 计数 0），不截身份/权限/本轮请求。水位 70/85/60、冷却 3 轮、k 在 6–40 有界伸缩与文档一致。B01–B05、E01–E07 独立复跑全过。

## 四、M2 压缩归档恢复（G2/G3 通过，并发 CAS 待裁决）
先写段（tmp+rename）→写 manifest→才移热窗口，归档失败 `turns` 一条不减、`overflow=true`；摘要失败仅 `archive_only`。崩溃/损坏/幂等/重启均有失败语义与实测（A01–A06）：manifest 丢失可重建、坏行跳过、重复压缩按 id 去重。恢复有界且 scope fail-closed，跨用户/scope/会话/工作档泄漏实测 0（I01、E05）；纠错旧值标 stale 不复活（E09）、删除同步抹归档原文（I03）、旧格式兼容、legacy 回退可读。R-07 审计：`pruneArchives`（默认 90 天硬删段）全仓无调用点，非自动删除唯一经历，不阻断；但默认硬删唯一段无冷归档，列 B 类后续治理。**并发 CAS**：manifest 用「读→+1→rename」无真正比较交换与文件锁，手工构造两进程竞态窗口可复现后写覆盖先写、先写段成孤儿、原文从恢复链丢失（对应 K 报告 R-08）；真实两进程调用本轮未触发，但竞态窗口明确，属核心存储改造，未擅自修复，上报裁决。

## 五、基准、全量回归与 M3
120 轮基准独立复跑：压缩 14 次、归档 210 轮、事实 4/4、纠错泄漏 0、恢复 20/20、泄漏 0/0/0、500 条库 P50 18.91ms，与 K 交付结构性指标一致；时延因本机负载有量级差异属环境抖动，不以压缩率代语义正确。全量同机对比：候选 2906 项/15 失败，基线 2878 项/16 失败，失败集合无 S2 新增项（T11 日志轮转为已知波动，基线另有 2 项候选跑通过亦属波动），未改断言/跳过/放宽标准。M3：指定 `hy4flash`、`DeepSeek-V4.1-Flash`，model-meta 容量表存在但本环境无 BASE/KEY、无模型桥，未换别名/未 mock/未改桥/未读账号池；报文层 `--dry-run` 每模型 12 场景共 158 请求、模型名原样发出、只记长度哈希不落正文，凭证到位即可补测。

## 六、入库、交付与授权
Gitee `cand/memcore-s2-context-lifecycle-01` 回读 HEAD `08a81f1`/Tree `2d185369`/Parent `86dd0b9`，线性快进、K 历史保留；release/z1.4.7 仍 `c9e1b9f` 未动，无生产部署触发。本轮 C 无代码修复，未产生新线性候选。WorkBuddy C 侧单批次交付完整报告、精简报告、风险与 S3 接口台账、复现记录、双进度与知遥记录；GitHub docs/10-tasks/C、docs/20-reports/C 脱敏归档。全程未部署、未重启、未改生产、未动凭证、未越侧。结论交 ChatGPT 裁决 G5 接受、并发 CAS 修复启动与 S3 排序。
