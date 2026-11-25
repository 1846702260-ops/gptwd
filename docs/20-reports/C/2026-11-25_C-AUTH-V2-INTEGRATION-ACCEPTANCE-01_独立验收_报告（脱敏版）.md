# C-AUTH-V2-INTEGRATION-ACCEPTANCE-01｜账户认证 V2.0 独立验收与统一集成报告

任务编号：C-AUTH-V2-INTEGRATION-ACCEPTANCE-01（V1.0）｜裁决 G｜执行 C
验收时间：2026-11-25 23:52 – 2026-11-26 00:08（北京时间）
执行侧：C（独立验收与唯一集成入口）
规则依据：《AI 工作流统一规则》v1.4（2026-11-25，五侧 K/Z/C/R/E）

## 一、总体结论（阶段报告 / BLOCKED）

后端（K 侧）独立验收**通过**，K 候选已作为唯一可集成后端候选推入 Gitee 新 cand 分支并完成远端回读；但本任务的另一验收对象 **E-AUTH-V2-LIVE-WIRING-01（真实接线候选）在全部正式交付渠道中均不存在**，现有 E 侧唯一交付 `E-AUTH-V2-SECURITY-UI-01` 为 MOCK 合成契约实现并自标 BLOCKED。按任务书 G4/G5「未取得 E 或关键证据则阶段报告 BLOCKED，不宣布整体验收完成」，本轮为**阶段通过 + 整体 BLOCKED**：后端可进入待裁决，前端真实接线待 E 交付后补验；不宣布账户认证 V2.0 整体验收完成，不部署生产、不改 release、不迁移真实身份。

A 类阻断（真实提权 / 凭证与恢复绕过 / 旧凭证复活 / 数据丢失 / 不可控迁移 / 开关旁路）：**实测归零**。B 类条件风险 5 项，逐项列明限制、责任人与是否阻止启用。C 类技术债 3 项，不扩张。

## 二、G0 唯一候选与基线

| 项 | 任务书给定 | 实测 | 结论 |
| --- | --- | --- | --- |
| K 候选 HEAD | 3ea0aa70e17bce468cc5d88e1b225425a8ae76fa | 一致 | ✅ |
| K 候选 Tree | 0d8809d384dc7385cd5e74a690da07ebc6d70a09 | 一致 | ✅ |
| K 父提交 | 01a3651eef0910a9685d84728dfd06d09cebaa77 | 一致 | ✅ |
| 共同基线 | 960e6f2… Tree 33ad7eb3 | 为 HEAD 祖先（git merge-base 确认） | ✅ |
| bundle 完整性 | 完整历史 | `git bundle verify` 通过，无缺口 | ✅ |
| 干净恢复 | clone/patch | 干净 clone HEAD/Tree 匹配；patch 可干净应用到基线 | ✅ |

K 从正式 WorkBuddy 批次 `K-AUTH-V2-OWNER-CONTINUITY-01_2026-11-25_final_3ea0aa7.zip`（节点 （节点ID脱敏），整包 SHA256 `577b930a…1335b`）取得，含 bundle/patch/7 测试/设计接口迁移回滚文档/COORDINATES/SHA256SUMS/verify.sh。独立运行 verify.sh：SHA256 全一致、树哈希一致、bundle 完整、patch 可应用、48 项全绿。未把本地分支当远端：Gitee 远端该提交树哈希经 API 回读一致。

R 侧 `R-AUTH-V2-SELECTION-01` 选型报告与 PoC 结论从 GitHub 辅助归档 `docs/20-reports/R/…脱敏报告.md` 取得并核对（组件路径见 G1）。E 侧交付核实见 G4。

## 三、G1 组件选型与身份安全

**选型**：R 报告否决整体引入 Better Auth（双认证真源、59 包/43MB、Passkey 默认 userVerification=preferred 且 TOTP 无重放闸），推荐组件化——沿用现有 scrypt 底座、TOTP 用 otplib 13.6.0、Passkey 用 @simplewebauthn/server 14.1.0 + browser 14.0.0。K 实际采用：Passkey 与 R 一致（14.1.0，MIT，已核 GHSA-6hxq-p678-4hr2 修复于 13.3.2/14.0.0，生产 Node v22.22.2 满足 ≥22）；TOTP 为自研 RFC6238（未用 otplib）；scrypt 为 N=2^15（未升到 R 建议与任务书默认的 2^16）。这两处为 B 类偏差，不擅改，提交 G 裁决（见第六节 B1/B2）。

**依赖与许可证**：`@simplewebauthn/server@14.1.0` 精确锁定，MIT，无第二认证真源；生产 Node ≥22 已满足。未引入 Better Auth。

**ownerId 单一稳定与 fail-closed**（C 独立测试实测 9 项全过）：
- 已落盘 ownerId 与 `config.web.ownerId` 冲突 → 409 `owner_identity_mismatch`，不静默二选一；
- 配置 ownerId 在身份库不存在 → 拒绝创建"新主人"；
- 身份库存在其他 ownerKind 且与配置冲突 → 歧义拒绝；
- 配置为空且无状态 → `owner_missing`；
- 服务身份 `svc:` 前缀被拒；字面量 `owner` 在首次迁移的身份库命中检查下被拒（真实库无该 id）；
- 邮箱文本、请求体 ownerId、客户端字段一律不采信（源码审查确认）。

**开关与旁路**（任务书重点）：
- `web.auth.enabled` 默认 false，关闭时与基线行为一致；
- **启用但初始化失败回退旧路径** 风险已排除并实测：`#bootCheck` 捕获损坏状态仅记 warning 不抛错，`authService` 仍装配（非 null → `AUTH_HTTP` 生效），`status()`/`requireState()` 对损坏状态抛 `auth_state_corrupt`，HTTP 转 500，不静默放行；仅当 `web.auth.enabled=false` 才走旧路径；
- 访问口令层（`web.access`）职责不变仍在，未擅自移除；V2 登录在口令门禁之后可达。

## 四、G2 真实链路认证、会话与恢复

C 独立负例 + K 专项 + 相邻回归实测（详见第八节测试清单）：
- **MFA 前无完整会话**：密码正确但未过第二步只发 challenge token、不发 Cookie；challenge 内存态、5 分钟过期、最多 5 次失败（实测过期与单次语义通过）；
- **TOTP 重放**：`verifyTotp` 以持久化 `minCounter` 严格大于判定，同秒同码二次提交被拒（C 独立实测通过）；±1 窄窗、6 位/30 秒、常量时间比较；
- **未知账号同等计算**：邮箱不存在与密码错误统一 `bad_credentials`，两者均跑一次 scrypt（源码审查 + 实测）；IP+账号双维限速；
- **全入口权限**：主人会话缺第二因素证据时，webapi 兜底闸对所有主人资源返回 403 `incomplete_auth`（放行 session-info 供前端提示）；API/WS/SSE/聊天/工具/记忆统一经该闸；普通用户不可进主人安全中心（`#requireOwner` 判 `kind==='owner'`）；
- **会话**：签发 `zy_web` Cookie（HttpOnly/SameSite=Lax/HTTPS 加 Secure），登录换发撤销旧令牌，设备列表/单撤/全撤，改密改邮箱与禁用 MFA 要求近期重认证（reauth 窗口 300s）；
- **CSRF/Origin**：state-changing 请求需 Origin 白名单精确命中 + `x-zy-csrf:1` + JSON content-type；伪造 Origin 被拒（C 实测）；`normalizeOrigin` 拒绝 userinfo/file/null；
- **Passkey**：RP ID/expectedOrigin 由服务端 `policy.origins` 白名单预设，`#expectedOrigins` 强制请求 origin 必须命中白名单（不从任意 Host/转发头推导）；注册/登录均强制 UV required；可发现凭证 + userHandle 不透明随机 32B；计数器判定兼容同步凭证 0→0、拒绝回退（C 实测 judgeCounter）；撤销台账哈希链；默认关闭，缺 rpID/origins 或库不可用时 503，回滚检测时 409，均不影响密码+TOTP+离线恢复。软件虚拟认证器验证的是协议数据，不冒称苹果真机；真机 Safari/PWA 留 Z 授权验收；
- **离线恢复**：仅本机 CLI `zyagi-admin auth-recover`，256bit 一次性凭证只显示一次、只存 scrypt 散列，重置后撤销全部会话/通行密钥、关闭 TOTP 需重绑、ownerId 不变；无网页忘记密码、无 SMTP 邮箱验证。

## 五、G3 备份恢复与资源安全

C 独立实测 5 项全过：
- 备份清单含 `auth.json`、`auth-revocations.jsonl/.seal`、旧 `account.json`、`keys/totp.key`（源码核对）；
- 撤销台账 append-only + 哈希链，截断一行即判 `tampered=true`（实测通过）；
- 回滚锚点：数据目录外 generation 高于磁盘状态即 `rollback.detected`（实测通过）；检测到回退 Passkey 自动不可用；
- 坏 auth.json 抛 `auth_state_corrupt` 不重置为空（fail-closed，实测）；
- KEK 优先环境变量 `ZY_AUTH_KEK`、兜底文件 600、AES-256-GCM + AAD 绑定账户（跨账户密文解不开，实测）。

**防旧凭证复活**：撤销靠台账（独立于 auth.json 字段）+ 代际锚点（独立于可回滚快照）。旧密码/旧会话/旧 TOTP 复活路径：回滚后代际检测使 Passkey 失效，离线凭证一次性且散列，旧 `account.json` 保留但迁移后不再作真源。软/硬回滚手册明确"禁止回滚恢复旧凭证"，V2 改过的口令不回写 account.json 需回滚前 CLI 重置——此点属中风险需 Z 按手册执行。

**资源实测**：scrypt N=2^15/keylen=64 单次哈希受控沙箱约 93ms（记录值）。challenge 与限速为内存态，生产必须单进程（当前 systemd 单进程），多 worker 会破坏 challenge 亲和性与限速共享——B 类，见 B3。

**G3 未闭合**：备份端到端恢复演练（备份→恢复→登录/记忆核对）未做；代际锚点默认 `/var/lib/zyagi/auth-generation` 不可写时会静默跳过，此时回滚检测只剩包内 generation，无法独立证明未回滚；KEK 兜底模式下密钥与密文同包。三者均为 B 类，必须在启用 V2 前由 Z 在隔离副本补验/改用 env KEK。按任务书"若恢复安全不能保证则阻断启用 V2，不以 Passkey 关闭掩盖其他认证问题"——本轮不阻断后端候选入库，但**阻断 V2 生产启用**，直至 Z 完成恢复演练与 KEK 隔离。

## 六、G4 E 真实接线（BLOCKED）

**穷尽检索结论**：E-AUTH-V2-LIVE-WIRING-01 在以下渠道均不存在——
1. WorkBuddy 资料库全空间语义检索（仅命中 K 报告提及的"对 E 接口"）；
2. GitHub 辅助归档 `docs/10-tasks/E`、`docs/20-reports/E`：仅有 `E-AUTH-V2-SECURITY-UI-01`（任务/精简报告）；
3. Gitee `（账号脱敏）/zyagi` 全部分支：无任何 auth/e/wiring 候选分支，`cand/s3-governance-foundation-01` 仍为基线 960e6f2。

现有 `E-AUTH-V2-SECURITY-UI-01` V1.1 报告明确：状态"研发候选完成，交 C 独立验收（**BLOCKED：K 接口未定，非正式交付**）"，按 MOCK 合成契约实现，9 文件 +1303/−2，含安全中心/登录/MFA/Passkey 前端组件与 9 项合成契约测试，未连生产、未调真实模型、未推受控仓库。它**不是**本任务要求的真实接线候选。按任务书，E 真实适配器、接口四态、登录切换、密码自动填充、移动端键盘焦点、页面路径与必要前端回归均无法验收，**G4 判 BLOCKED**，不计为通过。待 E 交付真实接线候选后，仅增加接口/前端及必要相邻回归，已有后端结论继承。

## 七、G5 集成候选、裁决与交付

**集成候选**：Gitee `（账号脱敏）/zyagi` 新分支 `cand/auth-v2-k-backend-acceptance-01` → HEAD `3ea0aa7` / Tree `0d8809d3` / 父 `01a3651`，远端 API 回读一致，未合并 `release/z1.4.7`。这是唯一可集成后端候选（E 未交付，无法形成 K+E 完整集成候选）。逐文件 SHA256 见 `03-逐文件SHA256清单.txt`；迁移与回滚步骤见 K 包 `06-迁移与回滚.md`（随 bundle 原样保留）；依赖/资源与启用矩阵见第八节。

**A/B/C 风险分级**：
- **A 类（必须归零）**：真实提权、凭证/恢复绕过、旧凭证复活、数据丢失、不可控迁移、开关旁路——实测全部归零。
- **B 类（条件可控，逐项）**：见下表，含限制、责任人、是否阻止启用。
- **C 类（不扩张）**：邮箱大小写本地部分合并（L7）、Passkey 撤销后设备选择器仍显示旧凭证（L8）、单账号模型不塞多成员（L11）——记入后续，不影响本阶段。

| 编号 | B 类风险 | 限制/条件 | 责任人 | 是否阻止启用 V2 |
| --- | --- | --- | --- | --- |
| B1 | scrypt N=2^15 与任务书默认 2^16 偏差 | 实测 93ms/次，2^16 约 128ms、并发有内存峰值；版本化记录支持登录无感升级，不降级已有强哈希 | G 裁决参数；K 后续改默认 | 否（需 G 确认） |
| B2 | TOTP 自研未用 R 推荐 otplib 13.6.0 | 自研 RFC6238 + 跨进程重放闸已实测正确；R 报告亦强调库只算码、重放须自重 | G 裁决是否换库 | 否 |
| B3 | challenge/限速内存态、单进程假设 | 多 worker 破坏亲和与限速共享 | Z 部署保持单实例或 sticky | 否（部署约束） |
| B4 | KEK 兜底随备份 + 代际锚点不可写失效 + 备份恢复未端到端 | 备份泄露=种子可解；锚点不可写时回滚检测仅剩包内 generation | Z 隔离副本演练、改 env KEK、确保锚点可写 | **是**（启用前必须闭合） |
| B5 | 旧令牌未清账、回滚口令分叉 | 171 条旧 owner 令牌仍在文件（被 resolveEx 拒）；V2 口令不回写 account.json | C/Z 迁移验收后 `migrate-legacy-owner --apply`；回滚前 CLI 重置 | 否（迁移流程约束） |

**接收方**：G（裁决 WebAuthn 选型、scrypt 参数、OTP 库、生产迁移窗口与 ownerId）；Z（授权后真机/域名验、恢复演练、env KEK、单实例部署）；E（真实接线交付后交 C 补验）；主人（生产迁移最终授权）。

**未验证事项**：E 真实接线（BLOCKED）；iPhone Safari/主屏 PWA 真机与生产域名证书；备份端到端恢复；7 项跨进程 MCP 用例首次因缺 SDK cancelled、补装 `@modelcontextprotocol/sdk` 后 16/16 通过（已补测，不计遗漏）；真实模型与真实 QQ（本任务范围外，未调用）。

## 八、测试清单与启用矩阵

| 测试组 | 命令/来源 | 结果 |
| --- | --- | --- |
| K 包装验 | verify.sh（SHA256/树/bundle/patch/48 测试） | 全通过 |
| auth-v2 专项 | node --test test/auth-v2/*.test.mjs | 48/48 |
| C 独立负例 | c_audit.mjs（G1 9 / G2 7 / G3 5） | 21/21 |
| 相邻回归一 | identity/security/web-v1（补装 MCP SDK 后） | 791/791 |
| 相邻回归二 | memory/ops/concurrency | 1186/1186 |
| MCP 门禁跨进程 | mcp-gate.test.mjs（补测） | 16/16 |

**启用矩阵**：`web.auth.enabled=false`（默认）= 旧链路不变，本候选可安全入库；`=true` + 迁移完成 + 单实例 + env KEK + 隔离恢复演练通过 = 可申请生产启用（需 Z 真机 + 主人授权）；Passkey 独立于主开关，真机前保持 `passkey.enabled=false`。S3 仍为附条件通过，本专项不宣称 S3 完整上线、不扩张桌宠/S3 未接通功能。

## 九、边界声明

未部署生产、未重启服务、未迁真实身份、未改生产配置、未访问真实记忆、未变更 QQ 桥、未强推或发布分支写入、未合并 release、未调用真实模型。GitHub 仅脱敏辅助归档，不替代 WorkBuddy 正式交付与 Gitee 入库。归档失败不阻塞正式交付。
