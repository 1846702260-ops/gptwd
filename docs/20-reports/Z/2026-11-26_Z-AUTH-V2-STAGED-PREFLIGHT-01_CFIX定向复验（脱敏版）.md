# 2026-11-26 · Z-AUTH-V2-STAGED-PREFLIGHT-01（续办）· 候选 d311e152 定向复验（脱敏版）

- 执行：Z；裁决：G；日期：2026-11-26（北京时间）
- 候选：`C-AUTH-V2-BOOT-RESTORE-CFIX-01` · HEAD `d311e152` / Tree `d7a0db1f` / Parent `2ad957f`
- 分支：`cand/auth-v2-boot-restore-cfix-01`；`release/z1.4.7` 未动
- 范围：仅隔离验收与发布准备；未部署、未做真实账户迁移、未启用 Passkey

## 结论

**接收通过；A1/A2/A4 通过；A3 不通过（1 项 A 类阻断）→ 不建议发布。**

## 三坐标与可复现性

- Gitee 远端回读：`d311e152`，上一候选仍 `2ad957f`，release `c9e1b9f` 未变
- 干净克隆后 `HEAD^{tree}` = `d7a0db1f`，与候选 Tree 精确一致；`HEAD^` = `2ad957f`
- 变更 10 文件（+805/−19），全部在认证子系统与测试；未改 `webapi.js` / `auth/http.mjs`（25 路由与四态不变）
- 构建：`Z5.2.0`，`commit=d311e15`、`tree=d7a0db1f`、`dirty=false`

## 门禁判定

| 门禁 | 结果 | 要点 |
| --- | --- | --- |
| A1 真实装配 | 通过 | 关/开/未初始化/装配失败四形态可解释；`enabled="true"` 拒绝启动；装配失败 Web 与旧入口 404 不降级 |
| A1 无旁路 | 通过 | 关闭态与基线一致；装配失败态无旧链路 |
| A2 独立 KEK | 通过 | 缺 KEK → 503 且不生成 `keys/totp.key`；错 KEK → 500 `totp_secret_unavailable` |
| A3 旧备份锁定 | **不通过** | 认证入口 423 已生效，但 webapi 会话层可复活主人权限 |
| A4 迁移继承 | 通过 | ownerId 连续、二次迁移被拒、提升 owner=0、非目标身份未误伤 |
| 回归 | 通过 | auth-v2 57/57、相邻 775/775、锁护栏 2/2；C e2e 26/26、恢复 6/6 |

## A 类阻断 A3-b

锁定态（`restoreLocked=true`）下，旧备份中的主人会话（`amr` 完整）仍：

- `GET /api/web/session` → `isOwner=true`
- `GET /api/web/persona/status`、`/api/web/devices`、`/api/ws/home`、`/api/ws/projects` → 200
- `GET /api/web/auth/status` → 同时返回 `state=full` 与 `restoreLocked=true`

根因：`AuthService` 的 `#assertNotRestoreLocked` 只覆盖认证 HTTP 层；`webapi.js` 的身份解析（1162 行）与 persona/workspace 主人判定（1775 行）只看 `kind/userId`；`status()` 的 `full` 未叠加锁定。C 的 e2e 只验"旧口令路由不存在"，未覆盖"整目录回滚 + 旧 Cookie"。

修复建议：webapi 层对锁定态 owner 一律 423；`status` 锁定态不返回 full；补 1 条 e2e；解锁后提示撤销旧会话/设备。

## 备份与令牌续期

- 11-26 03:30 定时备份自然到点成功：9 分片、sha256 `e93a08bc…`，上传与独立回读双通过，`verified=true`、RPO=0
- Z 只读重组 9 片，sha256 与回执一致（证明可恢复，而非日志自报）
- NAS 令牌 01:11 自然续期（有效至 11-27 01:11），03:30 备份使用续期后令牌，无 401

## 生产保护

- HEAD `ded6769`，服务健康；自动部署 cron 仍带暂停注释
- 工作区 8 项（6 改 + 2 未跟踪），6 文件 patch 存档在案
- 逐文件比对：备份三文件（`backup-daily.sh`/`nas-backend.sh`/`nas-shard.py`）与候选一致；`bridge/ensure.sh`、`bridge/server.py`、`switch/ensure.sh`、`learning.js` 4 处产线热修与候选不等价 → 发布须白名单合并或先入库
- 磁盘 85%

## B 类

1. C 声称资料库批次已交，实际 C 侧无 CFIX 批次（bundle/patch/SHA256/文档全缺），交付只剩 Gitee 单通道；
2. 错 KEK 返回 500，建议统一 503；
3. 4 处产线热修未入库；
4. `auth-verify-restore --accept` 不清理旧会话/设备；
5. 锚点默认路径需可写，部署须显式配置环境变量。

## 未验证（不写通过）

生产数据恢复、真实域名 HTTPS/反代 origin、iPhone Safari 与主屏 PWA、真实 Passkey 注册/登录、生产迁移本身。

## 建议

C 修 A3-b 出新候选 → Z 只复验"锁定态旧 Cookie 全端点 423 + status 不报 full + 既有回归" → G 复核 → 主人授权。最终生产授权由主人给出，本报告不构成上线许可。
