# 01｜精简报告（交 G / C）

**任务**：E-CROSS-DEVICE-CHAT-SYNC-01（知遥多端聊天实时同步）
**结论**：根因是实时事件通道只覆盖「消息落盘/已读」两处，会话生命周期与生成状态未走事件总线；已复用现有 SSE 基础设施修复，双浏览器实测跨端同步通过。

## 坐标

| 项 | 值 |
|---|---|
| 冻结基线（C） | `cand/evocore-v1.0-release-closeout-02` @ `e1dea27` / Tree `819f4ef4` |
| 候选分支 | `cand/cross-device-chat-sync-01` @ `882cfd98` / Tree `28577372` / Parent `e1dea27` |
| 改动 | 10 文件，patch 1484 行 |
| 生产 | `release/z5.2.1` @ `5ac4dd02` 未动 |

## 根因（五条）

1. `WebSessions` 的 `create/rename/pin/archive/remove/restore/purge` 全部无通知回调 —— A 建/删会话，B 无感知。
2. 流式 `onDelta/onTool` 只写发起端 HTTP 流，不进事件总线 —— B 看不到「正在回答」。
3. 前端 `useEventStream` 只监听 `message.created`/`session.read` —— 即便后端补事件也不处理。
4. 重连不带 `Last-Event-ID`，无 `visibilitychange` 重连 —— 后台恢复停在旧内容。
5. 消息全量覆盖、无去重排序 —— 并发下冲掉乐观消息、可能重复。

## 修复

- **后端**：`WebSessions.onSessionChanged` → `session.changed`（7 动作）；流式 `emitTurn` → `turn.state`（start/delta/done/error，120ms 节流，带 clientMsgId）。全部走现有 UserEventBus，按 uid 分道。
- **前端**：监听面扩到四类事件 + Last-Event-ID 带回 + 后台恢复重连；`mergeMessages` 去重/按 ts 排序/保留乐观消息；会话变更重拉权威列表；远端流式气泡（无停止键）。
- **约束**：草稿仍 sessionStorage 设备独立；三主页面结构未改；后端无 `events` 时行为回退基线。

## 验收

| 项目 | 结果 |
|---|---|
| HTTP 契约 | 10/10 |
| 双独立浏览器（手机 390×844 + 电脑 1280×900） | 11/11 |
| 前端定向 / 全量 | 13/13 / 398/398 |
| 后端 web 测试组 | 505/505 |
| 类型检查 / 构建 | 通过 |

双浏览器覆盖：双向实时同步、流式回答、会话新建/删除/列表、并发发送、断线重连、草稿独立、三主页面底栏。patch 在基线 apply 后 `write-tree`=`28577372` 与候选一致。

## 边界与风险

未部署、未改生产、未调用真实模型（费用 0）、未新建记忆系统、未改六核契约。A 类无；B 类为单进程事件总线（多机需跨进程分发）、有界缓冲靠全量重拉兜底、无 iPhone 真机、回收站物理清理不逐条广播；C 类为事件分支可后续拆 hook。浏览器为 Chromium 模拟，**非 iPhone 真机**。

**交接**：交 G 裁决、C 独立验收。
