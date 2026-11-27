# MindCore V1.0 限域接入详细技术报告

## 基线及技术选型

冻结父提交 d2ec05b01af2bed13c8b6cb0a422c2cc9ca70bf8，Tree dd3db012c6fdf73562254a5bc5c43da9948c756b。候选 cand/mindcore-v1.0-limited-02，HEAD 246a6b552d8cdfe325390496bdbceaf25e8af624，Tree 3e14e4363062e08529377c62d47fc24bc558f992。14 个文件涉及限域决策、传输计量、只读任务和产品验证。生核、灵核冻结目录及 C 联合工作区未改。

沿用 G 已批准的方案 A。跨异步阶段的计量用 [Node.js AsyncLocalStorage](https://nodejs.org/api/async_context.html)，避免改写生产调用签名或从高层推测请求次数。版本化计划与执行分离参考 [Plan-and-Act](https://arxiv.org/abs/2503.09572)，策略迁移验证参考 [ExpeL](https://ojs.aaai.org/index.php/AAAI/article/view/29936)。M0 研究表仍是架构的外部对照；本轮没有复制第三方代码或引入依赖。有限规则优先于通用模型 NLI，传统任务原路回退。

仅 ZY_MINDCORE_LIMITED=1 创建限域对象，影子开关独立。所有授权使用原服务会话身份。只对真实 Web 主人入口接新能力，不扩展 QQ。六核继续使用同一个身份和原权限，memv2 是唯一结构化长期记忆真源。

## M1 证据关联

逻辑链为 Web 登录 → index.js 的 coreTurn 包装 → turn.js:517 明确意图识别 → workspace/p1/runtime.js:548 的原 compose 召回 → :700 取同一批候选 → cognition/limitedDecision.js:11 槽绑定 → turn.js:669 用模板作出有边界的回答。“已记录的 Node.js 运行要求是什么？”可匹配。“Node.js 最新版本是什么？”继续原路径。工作档、群聊、含图及非配置主人不进入新分支。

绑定要求来源为 retrieval、同 scope 及 recalledFrom、Semantic 原子、已识别证据等级、可核对 ID。仅解析完整单句的 Node.js required 语法。validFrom、updatedAt、validUntil 和 30 日上限决定时效。重用 alignEvidence 核对主体、槽、值和引用，区分 known、inferred、unknown、expired、review。跨档、错槽和高置信但无依据的文本得不到事实地位。强证据矛盾或 DISPUTED 要复核。绑定在同一批候选进入最终输出之前运行，不再次召回，不写事实副本。

回答仅陈述本轮可用记忆。引用 ID 保留在本轮内部结构，诊断输出状态和数量。legacy 回滚、损坏、上下文不可用和缺证时弃权。上游已滤掉的过期项自然表现为 unknown。该方案无法识别所有自由表述，也不能保证召回已涵盖所有冲突；跨语言事实、一般语义蕴含和产出级引用接口仍待评估。

## M2 真实计量及范围

index.js Web 包装 / Runner drive → cognition/limitedRuntime.js:4 → usageMeter.js:9 → Brain.#call:859 和 quick:952。Runner 为 Planner、replanner、步骤执行挂阶段标签。寻址同一任务计量对象，阶段切换不归零。发送尝试含超时和网络失败；报告累计耗时、实报输入和输出、以输入 UTF-8 字节为代理的估算、未知输出、未知费用请求数。计量不依赖旧上报把缺失 token 当 0 的逻辑。后台服务、跨进程 DSH 与任务重启另算，未宣称覆盖全部外部调用。

没有可信价格或账单接口。存在发送的作用域 totalCost.kind=unknown、micros=null。调用次数为 0 的本地作用域才报告 known_no_calls。估算输入不是实报 token。保留费用的未知状态，不从模型名推断免费。当前不会改变普通聊天的预算或调用次数。

共享预算组件支持阶段内预留调用数、输入/输出估值、deadline 和金额。有限费用但缺可信报价、缺输出上限或额度耗尽时拒绝发送。代码在新本地资料清单作用域主动使用 0 次调用、0 token、0 费用的模型约束，并按任务总时限暂停。普通模型任务只新增观察，保留原有预算和路由；没有通过候选开关替代生产的全模型预算。启用非零费用硬门限仍需要可信报价和持久用量对账。

## M3 只读动态计划与失败恢复

POST /api/ws/autonomy/tasks 的可选 mindcoreInventory.group 只允许 all、rules、facts、material。autonomy/index.js:1188 校验 owner 身份、开关、网关 enforce、原权限档和并发。TaskStore 持久化只读配置和固定 file_list 步骤；Runner 原并发队列领取。autonomy/limitedInventory.js:36 不走模型，使用已验证组和 allowlist。首次读 limit=50。TIMEOUT、TOOL_BUSY 或 TEMPORARY_UNAVAILABLE 可追加一条 limit=20 的缩小读取。没有其他工具路径。拒绝权限、意外返回及普通失败不触发第二次读取。若网关要求审批，保留现有审批状态，窄域分支不会自行批准或续用。

动作标识 inventory:组:限额，先留原意图，实际结果先写原工具回执，再完成步骤。TaskStore:423 以当前状态和 PlanSnapshot 同步校验追加，重读修订、取消、原失败步骤、许可工具、动作键及依赖。重复或过期提交被拒，已完成回执不被覆盖。使用原跨进程写者锁。成功回执后中断的任务能补完成状态，无需重读。接收回执前中断只能在剩余窗口重复只读。此行为不声称可使写入恰好一次。总时限从任务创建起算，默认 12 秒，限制为 1–20 秒。取消、关开关或 enforce 模式变化阻断后续动作。已发的只读请求仍由网关自行完成或超时。写、发送、删除、支付和通用重规划的权限保持原样。

## M4 策略版本及效果

autonomy/evocore.js:326 对原 guidance 的选中条目增加观察回调，原文本和单次检索保持。版本是 strategy/title/steps 规范 JSON 的 SHA256 前 16 个十六进制字符。meter 在真实发送前检查完整策略行是否存在于 system/developer 消息，忽略被截断和用户原话里的同文。回执关联版本、阶段与任务结束状态。摘要不存提示正文或原任务 ID，终态实际效果仍标 unmeasured。请求尝试不能证明供应商收到或模型采用，更不证明策略造成结果。中断前后的长期效果账本没有迁移。

独立留出共三条，两条已有写回执、一条未执行。相同输入和执行判据下，保留回执策略 3/3，盲重试 1/3，报告 negative 和 eligible=false。试验只是确定性回放。实际任务成功不提升权重、事实置信度或权限。SkillLab 沿用既有批准机制，无自动启用。

## M5 测试过程、失败与边界

Node.js 22.22.2 定向 110 项，110 通过、0 失败、0 跳过、0 取消，约 10.37 秒。新用例与旧影子、统一召回、自主任务、生核联动一起执行。两组实际 Companion 进程使用合成账号的真实 Cookie、真实 Web 和 TaskStore/Runner。上游为 loopback 固定响应，无真实模型。普通聊天开关请求次数相同；单次冷态 1358.13 和 1479.75 毫秒，未构成 p95 或延迟增幅证据。真实证据问答读取同一范围的 22，实际 enforce 网关返回隔离库计数 0，Runner 作用域模型发送为 0。另经普通任务验证真实 Planner 发送点的计量，费用为 unknown。

独立组件用例覆盖错槽、误置信度、跨档、过期、证据冲突、有限额度、版本竞态、重启、取消及回执恢复。暂时工具故障使用夹具，真实产品用例未注入故障。初次真实进程启动因编辑脚本少返回值，Runner 文件被清空导致缺导出。恢复冻结文件再重放局部变更后启动通过。另一处初始断言把整个进程的后台模型活动计到资料清单，改为验证 Runner 计量，同时保留普通聊天的真实请求数对照。这两种失败均未通过弱化产品安全条件处理。

不开通已知费用模型或未经授权 fallback。真实模型测试 BLOCKED：尚无可核对的 hy4-flash-free 响应/零费用证据或可执行的 DeepSeek 授权账本。Z 复验另见交接方案。无真实模型调用和费用、无生产部署或远程写入。本轮等待独立验收。
