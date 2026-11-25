# C-AUTH-V2-INTEGRATION-ACCEPTANCE-01 独立验收报告（更正版）

任务号：C-AUTH-V2-INTEGRATION-ACCEPTANCE-01 ｜ 执行侧：C ｜ 时间：2026-11-26 00:20（北京时间）｜ 规则：v1.4（五侧）｜ 裁决：G

> 本文件更正同批次 2026-11-25 版「阶段通过 + 整体 BLOCKED」结论。更正原因：上一轮检索 E/R 交付时误用个人空间与 `list-nodes --space-id` 入口，未按规则文档第 4.1 节「工作区访问方法 · 节点 ID 直取」访问共享空间，导致把已交付的 E 真实接线候选误判为"不存在"。按规则指定入口复查后，E 真实接线候选与 R 选型报告正式交付件均在共享空间内，故本轮结论更正为**整体验收通过（候选阶段）**。

## 一、总体结论（更正）

**整体验收通过（候选阶段）+ 待授权事项列明**。K 后端（3ea0aa7）与 E 前端真实接线（fe06629b）已形成 K+E 统一集成候选，推入 Gitee 新 cand 分支并远端回读一致。上一轮的 G4 BLOCKED 依据不成立，予以撤销。

边界不变：不部署生产、不改 release、不迁移真实身份、不公开凭证。Passkey 真机、iPhone Safari/主屏 PWA、备份端到端恢复演练属待授权事项，须由 Z 协同主人在生产启用前完成，不以"候选通过"冒充"生产就绪"。

A 类阻断（真实提权 / 凭证与恢复绕过 / 旧凭证复活 / 数据丢失 / 不可控迁移 / 开关旁路）：实测归零。B 类条件风险逐项列明（见第六节）。

## 二、更正依据：规则指定的共享空间节点直取

规则文档 v1.4 第 4.1 节明确：本工作区所在共享空间（space ID `（ID脱敏）`）不在账号 `list-user-spaces` 列表中，按空间维度列目录会返回 `code=12607 permission denied (role=invalid)`；**正确姿势是用目标节点 ID 或 `/space/d/{节点ID}` 调用 `space.workspace.node-info`，从返回 `nodes[]` 取子节点逐个直取**；在个人空间找同名 link 节点是陷阱。

据此直取结果：
- E 侧根 `（ID脱敏）` → 子节点含 `（ID脱敏）` = `2026-11-26_E-AUTH-V2-LIVE-WIRING-01-账户V2真实接口接线`（12 子节点）。
- R 侧根 `（ID脱敏）` → 子节点含 `（ID脱敏）` = `2026-11-25_R-AUTH-V2-SELECTION-01`（9 子节点）。
- 上一轮"space not found / 个人空间语义检索无命中"是入口错误所致，不是交付缺失。

## 三、G0′ 唯一候选与基线（更正后完整）

E-AUTH-V2-LIVE-WIRING-01 正式批次（共享空间节点 `（ID脱敏）`，12 子节点）含：`00-移交包.tar.gz`、`01-E增量-相对K候选.patch`、`02-完整历史bundle.bundle`、`04-SHA256校验清单.txt`、`03/05/06/07` 四份文档、4 张移动端截图、`source/` 源码清单。

三坐标核对（全部一致）：
| 项 | 值 |
| --- | --- |
| 共同基线 | `960e6f2` / Tree `33ad7eb3` |
| K 后端候选 | `3ea0aa7` / Tree `0d8809d3` / Parent `01a3651` |
| E 前端 V1.1（并入） | `8ab4123` → cherry-pick `d1550727` / Tree `52829eea` |
| E 本次候选 | `fe06629b` / Tree `0f72a403` / Parent `d1550727` |

独立还原实测：
- bundle 克隆 `fe06629b` / `0f72a403` / `d1550727` 逐项一致；
- patch 在 K 候选 `3ea0aa7` 干净 worktree 上 `git apply --check` 与 `git apply` 均通过，`git write-tree` = `0f72a40363c9f850186d2e8c17ced01d23076b47`（TREE_MATCH_OK）；
- `sha256sum -c` 对该批 24 项逐文件校验全部 OK（patch/bundle/4 文档/4 截图/13 源码文件，清单不自指）；
- K 候选 `3ea0aa7`、基线 `960e6f2` 经 `git merge-base --is-ancestor` 确认为 E 候选祖先；E 树含 13 个 K 后端 auth 模块 + E 前端 liveAdapter/LoginLive/AuthGate。

R 侧选型报告（节点 `（ID脱敏）`）含 01 选型裁决建议、02 源码路径版本许可依赖风险表、03 接口契约与实施决策-给K、04 接收与测试指南、05 证据与日志、06 精简报告、PoC tar 与两份哈希清单——上一轮从 GitHub 脱敏归档取得其结论（否决 Better Auth、推荐 otplib 13.6.0 + SimpleWebAuthn 14.1.0、scrypt 2^16），本轮确认资料库共享空间有正式原件。

## 四、G1–G3 后端独立验收（沿用上一轮实测，结论不变）

上一轮 C 独立编写的 21 项负例与 K 专项 48 项在 E 候选树内复跑一致：
- ownerId fail-closed、已落盘与配置冲突即拒绝、歧义拒绝、服务身份/字面量 owner 挡住、邮箱文本与请求体字段不采信；
- "启用却回退旧路径"风险排除：状态损坏时 authService 仍装配、读接口抛 `auth_state_corrupt` 转 500，不静默放行；
- TOTP 同码重放拒绝、MFA 前不发会话、challenge 过期失效、CSRF/Origin 伪造拒绝、Passkey RP ID/Origin 白名单 + 强制 UV + 零计数器兼容 + 回退拒绝；
- 撤销台账 append-only 哈希链截断判篡改、数据目录外代际锚点发现回滚、坏 auth.json fail-closed、KEK 优先 env 且 AAD 绑定账户（跨账户解不开）。

测试矩阵（本轮在 E 候选树内复现）：
| 套件 | 结果 |
| --- | --- |
| K 后端 auth-v2 专项 | 48 / 48 |
| C 独立负例脚本 c_audit.mjs | 21 / 21（上一轮，结论继承） |
| E 真实端到端 m3-e2e.mjs | 27 / 27 |
| 前端 vitest（36 文件） | 348 / 348 |
| 相邻 identity/security/web-v1 | 791 / 791 |
| 相邻 memory/ops/concurrency | 1186 / 1186 |
| MCP 跨进程补测 | 16 / 16（已补装 SDK，不计遗漏） |

## 五、G4′ E 前端真实接线验收（更正：通过）

上一轮判 BLOCKED 的对象本轮独立验收通过。核查项：

1. **真实适配器非 mock**（源码审查）：`security/liveAdapter.ts` 全部方法经 `authFetch` 调 K 的 25 条 `/api/web/auth/*` 路由；challengeId 仅模块内存暂存、用完即清、不写任何持久存储；成功只认服务端 2xx + ok:true；错误码原样透传。`http.ts` 对所有 POST 统一加 `x-zy-csrf: 1` + `Content-Type: application/json`，`credentials: same-origin` 只认 HttpOnly Cookie，请求体从不放 userId/ownerId 做授权。
2. **开关与登录切换**：`AuthGate` 探测 `GET /api/web/auth/status` 的 `authInitialized`；为真用真实登录页 `LoginLive`，否则保留旧 `AccountLogin`；V2 启用时不回退旧认证。
3. **四态与错误码**：anonymous/incomplete/full/reauth_required 全覆盖；`AUTH_ERROR_TEXT` 统一文案不透露账号存在性；敏感操作 403 弹重新验证。
4. **真实端到端**：m3-e2e.mjs 用真 AuthService + 真 WebSessions + 真密码学虚拟认证器 + 真 createAuthHttp 路由，27/27 通过，覆盖 CSRF/Origin/Content-Type、统一错误码、TOTP 10 恢复码、备用码登录、Passkey 注册/列表/登录/撤销、改密改邮箱、设备撤销、logout-all、离线恢复说明无秘密。
5. **截图真实性**：核看 `01-live-login.png`、`03-live-mfa.png`，均为 390×844 移动端视口真实渲染界面，含"邮箱密码·两步验证·通行密钥"、真实 MFA 挑战页，版本 Z5.2.0；截图为合成账号，不含 Cookie/令牌/密钥明文。
6. **敏感不持久化**：前端源码无 secret/恢复码/令牌持久化；TOTP setup 的 secret 仅在展示后交由后端保管，恢复码一次性展示。

## 六、G5′ 集成候选、裁决与交付

**集成候选**：Gitee 新分支 `cand/auth-v2-k-e-integration-01`（HEAD `fe06629b` / Tree `0f72a403` / Parent `d1550727`），推送成功并远端 API 回读一致，未合并 release。上一轮 K 后端分支 `cand/auth-v2-k-backend-acceptance-01`（3ea0aa7）保留，两者并存。

A/B/C 风险分级：
- **A 类（必须归零）**：真实提权、凭证/恢复绕过、旧凭证复活、数据丢失、不可控迁移、开关旁路——实测归零。
- **B 类（条件可控）**：① scrypt N=2^15（任务书默认 2^16，实测约 93ms）；② TOTP 自研 RFC6238（未用 otplib，跨进程重放闸实测正确）；③ challenge/限速内存态（生产须单进程）；④ KEK 兜底随备份 + 备份端到端恢复未演练（**生产启用前必须由 Z 在隔离副本闭合**）；⑤ 旧 171 条令牌未清账（迁移后清）。待验类：iPhone Safari/主屏 PWA 真机、Passkey 真机认证器（生产保持关闭）——由 Z 协同主人执行。
- **C 类（不扩张）**：邮箱大小写本地部分合并、Passkey 撤销后设备选择器仍显示旧凭证、前端单 chunk >500 kB、旧 AccountLogin 保留（V2 关闭时）。

**接收方与下一步**：G 裁决组件选型与口令参数；Z 在授权后做真机/域名验收、隔离副本恢复演练、改用 env KEK、保持单实例部署；生产迁移由主人最终授权。E 不自派下一任务。

**未验证事项如实保留**：iPhone Safari/主屏 PWA 真机、生产域名证书、备份端到端恢复、真实模型与真实 QQ 通道，均不在本轮或未被授权，不以候选通过冒充生产就绪。

## 七、交付物

- 更正版独立验收报告、精简报告、完整性验证报告、逐文件 SHA256 清单、接收与测试指南；沿用 K 后端 C 独立验收脚本 c_audit.mjs。
- 资料库 C 侧空间批次 `C-AUTH-V2-INTEGRATION-ACCEPTANCE-01`；进度记录追加更正章节。
- GitHub 辅助归档：更正版任务结论与报告脱敏版，上传 `docs/20-reports/C`。
