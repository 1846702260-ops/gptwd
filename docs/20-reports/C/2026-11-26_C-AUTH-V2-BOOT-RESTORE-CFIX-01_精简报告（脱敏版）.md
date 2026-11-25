C-AUTH-V2-BOOT-RESTORE-CFIX-01 精简报告（C→G裁决，Z接收）
2026-11-26 北京时间

【结论】
真实启动 A-1 与生产启用所需的恢复安全条件已在范围内闭合，形成唯一可独立复现候选 d311e152（新 cand 分支 cand/auth-v2-boot-restore-cfix-01）。A 类门禁（真实装配有效、无旁路、旧凭证不复活、独立密钥、主人身份继承与安全恢复可验证）逐项实测通过；无新增风险，建议交 Z 定向复验后进入发布审批。非生产授权，未连生产，未动 release。

【三坐标】
新候选：HEAD d311e1520fdccfd32503818f32ffb1a18a101583，Tree d7a0db1f2be90adb01625472612a51631b214cb7，Parent 2ad957f93d874c334a35cc9b92ca31e1a6dcbdec（上轮最终候选）。共同基线 960e6f2。Gitee 推送后远端回读 HEAD 与本地一致；release/z1.4.7 始终 c9e1b9f 未动，未强推。干净 bundle 克隆 checkout 后 write-tree 与候选 Tree 精确吻合；增量 patch 两份（相对前候选 1113 行、相对共同基线 10416 行）与 SHA256 清单（不自指）齐备。

【修改前证据（A-1 复现）】
隔离环境用真实 config.json（web.auth.enabled=true）调真实 loadConfig：cfg.web.auth=undefined、policy.enabled=false。根因是 loadConfig 的 web 段白名单重建不含 auth 段，配置写了 enabled=true 真实启动也装配不上（status 401、login 404），而嵌入式入口不经 loadConfig，故"手工测试绿、真实启动哑"。

【A 类门禁实测】
A1 真实装配：修复后同配置经 loadConfig 得到白名单归一化的 web.auth（未知字段丢弃、安全默认值、错误配置如 enabled:"true" 抛 E_AUTH_CONFIG_INVALID 拒绝启动）。真实启动集成测试（真实 config→loadConfig→startCompanion→真实 HTTP）26 项全过：关闭态与旧基线一致；开启未初始化 409；初始化后邮箱密码登录 full；绑 TOTP 后旧会话撤销、MFA 完成前不签发会话、challengeToken 冒充 Cookie 按匿名处理；重启后会话与状态持久。启用失败（认证目录被破坏）→ 日志 E_AUTH_V2_ASSEMBLY"拒绝以旧登录链路降级运行"，Web 整体安全拒绝、/health 正常，不回退旧登录旁路、不产生无保护会话。
A2 独立密钥：kek.requireExternal 安全默认 true，KEK 缺失或非法 → TOTP 涉密操作 503 kek_unavailable 并明确告警，绝不回退随密文备份的本地密钥文件（实测不生成 keys/totp.key）；显式 requireExternal=false 保留单机兼容。给 Z 的密钥说明：KEK 经独立环境注入（如 systemd EnvironmentFile，属主 root/600，与数据目录分离、不进备份）；轮换步骤为生成新 32 字节 base64 → 更新环境 → 重启 → 重新绑定 TOTP 后旧密钥方可下线；KEK 与离线恢复凭证不是同一个秘密，密钥内容不入日志、不入报告、不入备份。
A3 旧备份不复活：restoreLocked 实时判定代际回退与锚点缺失，共同门禁覆盖密码、TOTP、恢复码、Passkey、旧 Cookie 与已撤销凭证（423 restore_locked）；合法离线恢复不受阻、ownerId 不变，auth-verify-restore --accept 重建一致锚点后解锁。锚点默认 /var/lib/zyagi/auth-generation，独立于会被旧包覆盖的数据。恢复门禁矩阵见 07 文档；旧快照覆盖后旧密码/会话/Passkey/TOTP/恢复码均无法绕过已用合成数据实证，未声称验证生产数据。
A4 迁移清账与字面量 owner：复核确认残留 userId:'owner' 令牌已可靠拒绝主人权限（无 kind 被 resolveEx 拒绝；直写 kind 的无 amr 证据判 incomplete_auth），按任务书作为明确留债，不批量删除；migrateLegacyOwner 目标集、拒绝语义与幂等复核通过，不误伤普通/服务身份、不折叠真主人。

【测试】
本轮实测：auth-v2 56/56（含新增 6 项）、真实启动 E2E 26/26、security 24 套、identity 2 套、web-v1 39 套接口回归全部 0 fail。继承 2ad957f 已验证证据（前端 348、K 48、identity+security 291、SHA256 24 项）。scrypt N=2^16 与 otplib 13.6.0 未动，25 路由与四态契约未改（status 仅附加 restoreLocked 字段，前端回归 0 fail）。

【B 类裁决点】
① kek.requireExternal 默认 true（生产必须环境注入），若认为过严可裁决放宽；② 锚点缺失也进入锁定（fail-closed），/var/lib 不可写的环境须配 ZY_AUTH_GENERATION_ANCHOR，须写入部署说明；③ status 新增附加字段 restoreLocked，E 前端如需展示另行适配。

【未验证项】
生产数据、真实域名 HTTPS、iPhone PWA/Passkey 真机、03:30 自然备份观察（按原 Z 任务单独执行）；ec1e2a9 备份修复与生产热修经逐文件核对未被本轮触碰（diff 文件清单为证），未重复全量历史测试。

【接收方】
Z 使用本轮唯一新坐标续办 Z-AUTH-V2-STAGED-PREFLIGHT-01：只复验真实启动装配、独立 KEK、旧备份恢复门禁与相关差异，继承不受影响证据；定向复验清单为 c-boot-restore.test.mjs（6 项）与 c-boot-e2e.mjs（26 项真实启动集成），接收方式（bundle 克隆 / patch 应用 / Gitee 直取）与同树判据见 05 文档；03:30 自然备份观察单独按原任务执行，不能拿手动回读代替。确认生产热修、自动部署显式冻结、资源与回滚保护；不得自行部署；真实域名、iPhone PWA/Passkey 实测及生产迁移按主人另行授权执行。交付物（bundle/patch/SHA256/03-07 文档/本报告）已交资料库日期批次并同步脱敏归档 GitHub（失败不阻断）。无新增风险才建议进入发布审批；最终生产授权仍由主人给予。
