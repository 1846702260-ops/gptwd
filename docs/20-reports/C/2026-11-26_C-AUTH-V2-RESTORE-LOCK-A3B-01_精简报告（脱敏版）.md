C-AUTH-V2-RESTORE-LOCK-A3B-01 · 精简报告（2026-11-26）

【结论】
Z 对 d311e152 的定向复验判定 A1、A2、A4 已通过，仅 A3-b 未过。本轮只收口 A3-b：受控恢复锁定接入统一服务端身份授权层，消除锁定期间旧 Cookie 仍能拿到主人全权的越权。缺陷已复现、已修复、已实测；唯一新候选已推 Gitee 并远端回读一致，release/z1.4.7 未动，未连生产、未部署。

【缺陷复现（修改前实测）】
真实启动后整目录回滚，锁定成立（状态代际 1 < 锚点 2，锚点在 dataDir 之外未被覆盖），同一枚旧 Cookie 逐入口实测：status 返回 state=full（且同时 locked=true）、session-info 200 owner、工作区会话列表 200、devices 200、ws/projects 200、persona/status 200。根因不是某个 if 写错，而是闸门挂错层：上轮 restoreLocked 门禁只挂在 AuthService.#requireOwner（覆盖密码/TOTP/恢复码/Passkey），网页身份判定走的是另一套 WebSessions.resolveEx，两层未打通，逐路由补必然漏，实测至少漏五个入口。

【修复（统一层单点生效）】
src/websession.js 增 ownerLockProvider 与 ownerLockState，在 resolveEx 拿到 kind=owner 之后、返回身份之前插入统一闸门：锁定期间主人令牌解析不出身份，下游统一按无身份处理（401/anonymous）。一处生效即覆盖网页 HTTP、认证 status、会话列表、设备、工作区、工具/WS、人格等全部主人专属入口，不再逐路由遗漏。provider 抛错按锁定处理（fail-closed），拒绝写审计并按哈希 10 秒节流。AuthService 装配时注入 provider，锁定判定实时读代际与锚点，不缓存。锁的是主人不是整站：普通用户与服务身份不受影响。

【旧 Cookie 不复活 / status 不返回 full】
锁定期间 status 拿不到主人身份，四态只能落 anonymous，不可能同时 full，只附加 restoreLocked 字段不改四态契约；旧 Cookie 访问任一主人入口一律 401，密码登录 423 restore_locked。实测修改后六入口全 401、status full=false locked=true。

【解锁前清账（避免解锁后复活）】
zyagi-admin auth-verify-restore --accept 改走 AuthService.acceptRestore：先撤销全部主人旧会话（走独立墓碑 revocation-tombstone.jsonl，只增不改、0600、附 seal 防篡改，即使 identities.json 之后被旧快照整体回写旧 Cookie 仍判无效），再撤销全部未撤销通行密钥，并推进 TOTP 计数器使旧验证码不可重放（后二者同一事务，失败整体不落盘），最后重新锚定；锚点写不进去抛 anchor_write_failed 500 且仍处锁定，绝不当成功。ownerId 保留不变（不重建主人），密码与 TOTP 密钥保留，合法离线恢复路径不受阻。未锁定时执行 acceptRestore 幂等，不改动任何数据。

【三坐标（已远端回读）】
分支 cand/auth-v2-restore-lock-a3b-01，HEAD adfc9ede8fada8ac171977bd1802ac4b70860031，Tree 7523e2038ed5e55acee606c78829c9c2bf875705，Parent d311e1520fdccfd32503818f32ffb1a18a101583（上轮候选），共同基线 960e6f2。Gitee ls-remote 回读与本地 HEAD 一致；release/z1.4.7 始终 c9e1b9f 未动、未强推；从 bundle 干净克隆 checkout 后 write-tree 与候选 Tree 精确吻合。源码改动仅 4 个文件（websession.js、auth/service.mjs、auth/store.mjs、bin/zyagi-admin.js），250 增 16 删。

【测试】
新增 3 个文件：c-a3b-owner-lock.test.mjs 6/6（统一层解析、status 不返回 full、acceptRestore 清账与重锚、解锁前后正负对照、幂等）；c-a3b-owner-lock-e2e.mjs 27/27（真实 config→loadConfig→startCompanion→真实 HTTP，覆盖真实整目录回滚、旧 Cookie、解锁前后正负对照）；c-a3b-defect-evidence.mjs（修改前越权复现 ✅ / 修改后全拒 ✅）。回归：auth-v2 63/63，web-v1 39 套、security 23 套、identity 2 套、workspace 3 套、persona 3 套、ops 1 套全 0 fail。persona-system 8 项失败经 git stash 对照确认为既有基线问题、非本轮引入，未擅自扩大研发范围。保留 scrypt N=2^16 与 otplib 13.6.0；不改 25 路由与四态契约；本轮 diff 未触碰备份/bridge/switch/learning/前端。

【更正（任务书第 5 项）】
上轮交付文档"共同门禁覆盖旧 Cookie 与已撤销凭证""旧 Cookie 的主人操作全部拒绝"表述不准确，撤回该声明：上轮门禁仅覆盖认证操作，网页侧身份判定未打通，实测 session-info/devices/工作区/工具/人格五入口在锁定期间仍返回 200 主人数据，且 status 同时返回 full 与 locked=true。已由本轮统一层修复与实测取代，更正说明见 08 文档；上轮批次原文档保持历史快照不改写。

【交付与交接】
交付目录九件：双 patch、完整历史 bundle、03 完整性验证报告、04 SHA256 清单、05 Z 接收与定向复验指南、06 测试证据与变更说明、07 生产范围清单与裁决点、08 原交付文档更正说明、本精简报告。资料库新建日期批次交付；总进度、C 侧工作进度、知遥记录三处追加；GitHub 脱敏归档辅助进行（失败不阻断）。Z 续办 Z-AUTH-V2-STAGED-PREFLIGHT-01 后续批次：只复验 A3-b（统一层覆盖、status 不返回 full、解锁前清账与不复活、真实启动正负对照），A1/A2/A4 继承已通过证据不重复验证；原交付批次已确认完整无需重复补交；极空间自动备份与自然续期观察已通过，不重复开启备份专项。裁决点 A–E（前端是否展示锁定态、persona-system 既有失败归属、锁定期整站策略、acceptRestore 摩擦、墓碑备份策略）见 07 文档留待决定。最终生产授权仍由主人给予。
