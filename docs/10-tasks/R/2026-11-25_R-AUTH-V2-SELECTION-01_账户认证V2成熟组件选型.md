# R-AUTH-V2-SELECTION-01｜账户认证 V2 成熟组件选型（限时小专项） · V1.1
日期：2026-11-25（北京时间）｜执行 R｜裁决 G｜协作 K/E/C

**修订 V1.1（2026-11-25）：** 本版完整替代 V1.0 执行依据，任务编号不变；已完成且未受影响的工作及测试有效，不重新开工、不删除旧交付，仅增量补齐 Passkey。原 WorkBuddy V1.4 流程不变。Passkey 为本轮正式范围，不是苹果账号 OAuth 登录，不需 Apple Developer 付费会员；使用 WebAuthn/FIDO2，优先成熟稳定组件。

## 目标与范围
用户已批准邮箱+密码、可选 Authenticator TOTP（默认关闭）、Passkey 免密登录、账户安全中心、现有主人 ID 永久继承；自托管、无新增付费服务、无需 SMTP 即可使用。不扩张到 OAuth 社交登录、多租户、自研生物识别、短信、独立认证服务器。现有三主页面总览/聊天/状态不变。研发基线候选 `960e6f292dce1a859809449ae18fb43b57927192`，Tree 短 `33ad7eb3`，Gitee `cand/s3-governance-foundation-01`；只读核对实际取得坐标，若后续 G 指定更新基线则记录差异。生产严禁改动。

## 工作内容
1. 核对实际 Node 版本、框架、账户持久化、ownerId/ownerJudge、会话签发与撤销、现有 Web/PWA/QQ/访问口令入口、反向代理 HTTPS、备份恢复契约。只读源码，不读真实密钥或私人记忆。候选获取失败先按正式交付节点及全树检索，勿反复盲查。
2. 优先实际验证 Better Auth 的稳定版本及许可、安全公告、数据库/会话适配、关闭注册、可选 TOTP、恢复码和已有账户迁移能力。若整体框架造成双认证真源、额外常驻服务或明显重构，选成熟小组件（如 otplib 的稳定受支持版本、Node 内建 scrypt 或可靠 Argon2id 库、现有会话层），复用而非重写密码哈希/TOTP/随机数算法。固定版本、锁依赖，列出近期漏洞与修复证据。至少真实安装并运行最小隔离测试，无法安装如实阻断；不凭功能表定选型。
3. 给 K 输出明确的最小实现决策：唯一认证入口与身份映射、密码参数、TOTP 密钥加密与密钥恢复、备用码散列和一次性事务、离线恢复（无 SMTP）、会话撤销和旧入口迁移、原系统低成本兼容。邮件作为登录名不代表已验证邮箱所有权。默认不发邮件；不得把邮箱当授权依据。区分 TOTP 恢复码（第二因素替代，仍需密码）和离线忘密恢复（独立高熵凭证+受控本机恢复流程，不可用公开邮箱找回接口绕过认证）。无需新外部账户、费用或模型调用。
4. 最小 PoC：正确/错误密码、关闭/开启 TOTP、过期/重放/错误验证码、备用码只用一次、关闭公开注册、原 ID 映射不变、撤销会话。合成账户，隔离目录。若框架不能安全适配，给出替代方案及真实依据；避免另建第二套认证权威。

## 已核查的公开研究入口
- https://better-auth.com/docs/authentication/email-password
- https://better-auth.com/docs/plugins/2fa
- https://github.com/better-auth/better-auth/releases
- https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
补查 RFC 6238、OWASP Password Storage/Forgot Password/Authentication；以当前稳定版本实际验证为准。

## Passkey 选型与验证（正式增量）
- 核对现有稳定 HTTPS 登录域名、RP ID、origin、可信代理、iPhone Safari 与主屏幕 PWA；域名/RP ID 不擅自修改，实际设备兼容性待 Z 验证。优先 Better Auth 官方 Passkey 插件；若不适配，评估成熟 WebAuthn 库（如 SimpleWebAuthn），不自写密码学或签名验证。实际固定稳定版本并核验所有传递依赖、许可证和安全公告。
- 官方 2026-11-03 安全公告指出 Passkey origin 与用户验证缺陷（GHSA-gv75-j3v8-3crm），稳定 1.7 线修复于 1.7.16；不得使用受影响版本，选当前已修复稳定版本，所有 Better Auth 相关包配套升级，检查后续公告与配置要求。复验撤销会话相关公告。禁止仅凭版本号断言安全。
- 隔离 PoC 实测注册、登录、撤销、错误 origin/RP ID、过期/重放 challenge、UV 缺失、他人凭证、跨账户绑定、并发消耗及认证恢复。明确 Passkey 必须 userVerification=required 且服务端验证 UV，允许作为独立完整登录方式；邮箱密码路径开启 TOTP 后仍必须完成 TOTP，不得借 Passkey 接口绕过未授权注册。敏感操作要求近期完整重新认证。明确同步通行密钥的计数器兼容、域名变化后重新绑定及旧设备丢失恢复策略。
- 输出给 K/E 的最小接口契约和建议，报告区分浏览器自动化/虚拟认证器与真实 iPhone；未拿到真机不冒称通过。无需额外服务或费用。
- 官方来源：https://better-auth.com/docs/plugins/passkey 、https://better-auth.com/blog/security-release-2026-11-03 、https://www.w3.org/TR/webauthn-3/ 、https://developer.apple.com/passkeys/ 。核查日期 2026-11-25；R 须在实施时复核最新版本与公告。

## 交付与协同
输出一页选型裁决建议、源码路径/版本/许可/依赖/风险表、可运行最小 PoC、日志和 SHA256、接收指南。正式交付原件到 WorkBuddy「知遥AGI-工作区」资料库（知遥AGI-R侧），日期批次、稳定节点链接，总/侧进度追加北京时间；GitHub `1846702260-ops/gptwd` docs/10-tasks/R、docs/20-reports/R 仅公开脱敏辅助归档，不取代 WorkBuddy/Gitee。不得跨侧写入、不得自派下轮。K 可先做身份盘点与接口设计，选型定案后实施；E 可先做独立界面。无需为本专项扩大研发周期或重复已有安全评测。报告约1300字，任务证据不限。G 审阅后确认选择；重大身份迁移及生产操作另行批准。
