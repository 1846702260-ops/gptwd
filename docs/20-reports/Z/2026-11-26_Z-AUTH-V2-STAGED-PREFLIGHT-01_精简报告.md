Z-AUTH-V2-STAGED-PREFLIGHT-01 精简报告
执行侧：Z｜2026-11-26 00:32—（北京时间）｜准备与隔离验证，非上线授权
基线：release/z1.4.7，HEAD ded6769fc10e0188557d7f5ee60999d371587297，Tree 5099fa8c
结论：不建议进入生产发布，继续保持冻结。M1 隔离演练 BLOCKED、M2 核心 BLOCKED、A 类阻断无法判定。

一、P0 原任务（P0-BACKUP-ENABLE-01 M3）
11-25 03:30 定时备份失败：上传成功但 ok=false/verified=false/远端 0 份，WARN remote-missing-after-upload。04:50 受控启用成功：分片上传+回读+重组复核通过，verified=true、RPO=0。11-26 03:30 观察尚未到点，不预演不伪造。结论：03:30 定时链路存在远端缺失，P0 未关闭，待观察补记。本任务未重复发起备份、未改时间。

二、只读环境盘点（M0，完成）
HEAD ded6769 与前端构建 gitCommit/Tree 一致，但 dirty=true，产物非纯头树，发布须以候选干净树重建。单实例成立：仅 PID 632449 监听 3111。资源 4 核、内存 3.6G（可用 1.5G）、磁盘 40G 用 84%（可用 6.5G，需关注）、Node v22.22.2。脏文件 29=6 已跟踪改+23 未跟踪备份，"29→6"即总数与已跟踪改口径差，全部可归属、未清理。自动部署 cron 已注释暂停，恢复仅删行首"# "，切换前须再确认。域名 zhiyao.duckdns.org，HTTPS 端口 18080（非默认），反代 127.0.0.1:3111，证书 Let's Encrypt 有效至 2027-02-09。生产未接线 WebAuthn/Passkey（config 与源码均无 rpId/origin）；计划 rpId=zhiyao.duckdns.org、origin=https://zhiyao.duckdns.org:18080，未验未配。Gitee 本地 HEAD 领先记录 upstream 55 提交（未 fetch，refs 或过期）。

三、隔离实测（M1）
BLOCKED。C-AUTH-V2-FINAL-RELEASE-CANDIDATE-01 在 Z 侧 58 项、C 侧 76 项、工作区根均未交付。按任务书不测过期候选、不以 fe06629b 为最终坐标，不下载/合并/重建替代候选。

四、待真机/待授权（M3）
发布方案与回滚已编制：停写窗口、冻结自动部署、可信本地恢复点、认证状态/代际锚点/独立 KEK 一致性、候选对齐与前端重建、身份迁移、回滚不复活旧凭证、24h 观察。虚拟认证器≠苹果真机；真实 Passkey 注册/登录、真实域名与生产认证开关须主人授权协作，Passkey 生产继续关闭，不伪造 Face ID。同域隔离验证须先交隔离/路由/凭证方案，不擅改生产。

五、正式迁移条件
M2 核心 BLOCKED：旧口令本机受控迁移幂等、ownerId 不变、身份不提升、旧令牌撤销与旧密码旧会话不可绕过、独立 KEK fail-closed、撤销后 Passkey/会话/恢复码不经旧备份复活——相关实现不在产线（grep KEK/passkey/webauthn 为空），须候选到位后验证，不以旧 merge-identity.mjs 冒称通过。极空间 M3 证据已只读核对，未新增远程上传、未删 NAS 历史包。

六、门禁
候选未到，无法验证是否引入 A 类风险，A 类阻断无法判定（非零）。按规则 A 类零方可建议发布；未验证不写通过，无真实变更授权不上线。建议：C 交付唯一最终候选→Z 做 M1 与 M2 核心→G 裁决→主人授权→按方案受控发布。

七、未执行与声明
未部署、未拉取/合并/切换、未改文件/配置/权限/令牌、未重启、未启停自动部署、未改 cron、未新增远程上传、未删 NAS 历史包、未执行真实迁移、未做真实模型调用或真实 Passkey。仅只读 SSH 执行 git/ps/ss/nginx/openssl/stat/tail/grep/node。本报告不构成部署授权。
