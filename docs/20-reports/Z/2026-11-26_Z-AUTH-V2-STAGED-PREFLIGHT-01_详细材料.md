# Z-AUTH-V2-STAGED-PREFLIGHT-01 · 阶段发布前隔离验收与生产准备（详细材料）

- **任务编号**：Z-AUTH-V2-STAGED-PREFLIGHT-01
- **执行侧**：Z（腾讯云生产运维与 Linux 真机验证方，规则 §2.5）
- **时间**：2026-11-26 00:32—（北京时间）｜采集时点见各节
- **性质**：**准备与隔离验证，不是生产上线授权**；全程只读 + 隔离，未做生产变更
- **依据**：C-AUTH-V2-INTEGRATION-ACCEPTANCE-01 更正版；待接收 C-AUTH-V2-FINAL-RELEASE-CANDIDATE-01 唯一最终候选
- **基线**：生产 `release/z1.4.7`，HEAD `ded6769fc10e0188557d7f5ee60999d371587297`，Tree `5099fa8cf52db4373b58b34b7e3c4e70f7d18e17`（**不把 `fe06629b` 当最终发布坐标**）
- **脱敏声明**：不含真实主人 ID、邮箱、口令、令牌、访问码、Cookie、私钥；仅记录长度/哈希前缀/来源/权限/端口。

> 适用规则：《AI 工作流统一规则》V1.4（2026-11-25 生效，ChatGPT 总协调 + K/C/Z/R/E 五侧）。本任务为只读+隔离验证，非生产授权。

---

## 0. 一句话结论

**M1 隔离演练 BLOCKED、M2 核心（KEK/Passkey/恢复码）BLOCKED**——`C-AUTH-V2-FINAL-RELEASE-CANDIDATE-01` 尚未在 Z/C 两侧交付，不测试过期候选；M0 生产只读盘点与 M3 发布方案已按证据完成。同时如实上报：**P0-BACKUP-ENABLE-01 的 11-25 03:30 定时备份失败（remote-missing-after-upload），04:50 受控启用成功；11-26 03:30 尚未到点，P0 未正式关闭**。**不建议进入生产发布**，继续保持冻结。

---

## 1. 开工检查与前置状态（规则 §3.3）

### 1.1 C 最终候选交付状态：未到（决定 M1）

| 检索范围 | 结果 |
|---|---|
| Z 侧目录（58 子项，全量遍历） | 无 `C-AUTH-V2-FINAL-RELEASE-CANDIDATE-01` |
| C 侧目录（76 子项，全量遍历） | 无该候选；AUTH 相关最新仅到 `2026-11-22_C-V2.0-P0-PHASE1-RELEASE-AUTH-HARDENING-01` |
| 工作区根（12 子项） | 无 |

结论：C 最终候选**未交付**。按任务书"若 C 最终候选未到，交只读准备证据并标注 BLOCKED，不反复测试过期候选"，**M1 标 BLOCKED**，不把 `fe06629b` 当最终坐标、不测过期候选。

### 1.2 P0-BACKUP-ENABLE-01 M3 现状（如实上报）

| 时点 | 事件 | 结果 |
|---|---|---|
| 2026-11-25 03:30（定时 cron `30 3 * * *`） | 每日备份 | 上传成功但 **`ok=false verified=false 远端0份`**，WARN `remote-missing-after-upload`，**失败** |
| 2026-11-25 04:50—04:53（受控启用） | 分片上传+回读+重组复核 | `verified=true outcome=VERIFIED rpo=0`，**成功**，耗时 154s，sha256 `47d0c7c8…` |
| 2026-11-26 03:30 | 每日备份自然续期观察 | **尚未到点**（本报告 00:32 起做得，不能预演/伪造） |

结论：03:30 定时链路已暴露 `remote-missing-after-upload` 问题，需在 11-26 03:30 后继续观察补记；**P0 未正式关闭**。本任务**不重复发起备份、不改时间、不制造生产故障**。

### 1.3 自动部署暂停状态（规则要求"显式暂停且可恢复"）

- `/etc/crontab`/crontab 中自动部署行**已被注释**：`# */5 * * * * TMPDIR=/var/tmp /bin/bash /root/zyagi/ops/deploy-linux.sh`（标记 `[P0-AUTODEPLOY-PAUSE-01 2026-11-21]`）。
- **恢复步骤**：删除该行行首 `# ` 即可；**切换前必须再次确认此行处于注释态**（本任务未改动 cron）。
- `systemctl list-timers` 无部署相关 timer，仅有 cron 一条被注释。

---

## 2. M0 生产只读盘点（已完成）

### 2.1 Git 坐标与前端构建对应

| 项 | 值 |
|---|---|
| 分支 | `release/z1.4.7` |
| HEAD | `ded6769fc10e0188557d7f5ee60999d371587297` |
| Tree | `5099fa8cf52db4373b58b34b7e3c4e70f7d18e17` |
| HEAD 提交 | `fix(webapi): B-1 姿态端点误用…「免密曾配置」留痕丢失` |
| 前端 build-info | `appVersion=Z5.0.8`、`gitCommit=ded6769…`、`gitTree=5099fa8…`、`branch=release/z1.4.7`、`dirty=true`、`builtAt=2026-11-24T11:42:26Z`（北京 19:42）、`node=v22.22.2` |
| dist 路径 | `/root/zyagi/packages/web-v1/dist`（index.html mtime 11-24 19:42） |

**判定**：前端构建与 HEAD **同提交同树**，一致；但 `dirty=true` 表明构建时工作区含未提交改动，前端产物来源并非"纯 HEAD 树"，需在发布重建时以候选干净树重构建。

### 2.2 进程与单实例

| 进程 | PID | 说明 |
|---|---|---|
| `src/index.js`（companion） | 632449 | 唯一实例，监听 127.0.0.1:3111，已运行 ~1 天 4 小时 |
| `dsh web` | 1830518 | 127.0.0.1:3080（本地工具服务） |
| `zy-tools-mcp/server.mjs` | 1830634 | MCP 工具子进程 |
| QQ（Electron node service） | 1407514 | 应用内 Electron 子进程，非 companion 实例 |

- 单实例确认：`ps` 精确匹配 `src/index.js` 仅 **1 个**（632449）；`ss` 显示 `:3111` 仅 632449 监听。**单实例成立**。
- 监听端口：`18080`(nginx SSL)、`80`(nginx)、`3111`(companion)、`3080`(dsh)。

### 2.3 资源

| 项 | 值 |
|---|---|
| CPU | 4 核 |
| 内存 | 3654 MB（used 2102 / available 1552） |
| 磁盘 | `/dev/vda1` 40G，已用 34G，**可用 6.5G（84%）** ✅风险关注 |
| Node | v22.22.2 |

> 磁盘 84% 为既有状态，非本任务引入；发布前需预留重建/备份空间，建议先清理历史 `dist.old-*`（各约 13M×8）。

### 2.4 脏文件 29→6 差异解释 + 待保留清单（**未清理**）

`git status --porcelain` 实测 **29 项** = **6 已跟踪修改 + 23 未跟踪**。此前报告口径差异即：**29 为全量项数，6 为已跟踪修改项数**，二者不矛盾。

**6 项已跟踪修改（生产热修/在途，勿丢）**：
`ops/backup-daily.sh`、`ops/bridge/ensure.sh`、`ops/bridge/server.py`、`ops/lib/nas-backend.sh`、`ops/switch/ensure.sh`、`packages/companion/src/autonomy/learning.js`

**23 项未跟踪（备份/快照，勿清理）**：`.learn-fix-backup-*`、`.model-sync-backup-*`、`.sync-fix-backup-*`、`ops/.脱敏改造备份-*`、`ops/bridge/ensure.sh.bak.20261125T0*`（2）、`ops/lib/nas-shard.py`、`ops/lib/shard-failtest.py`、`packages/companion/src/{autonomy,index.js,tools.js,turn.js,webapi.js,workspace}*.pre*-bak/*.pre-p0sec-*`、`packages/web-v1/dist.previous-20261124-194225/`、`packages/web-v1/src/version.ts.predeploy-bak`

**stash**：1 条（`local fix: loadConfig autonomy + createDshExecutor import (superseded by 94c304a)`）。

**恢复方法（记录，不执行）**：
- 已跟踪修改：`git diff` 保存补丁；如需回退某文件 `git checkout -- <file>`（**需授权**）。
- 未跟踪备份：均为文件级副本，原样保留即可；`dist.previous-*` 可作前端回滚目录。
- stash：`git stash show -p stash@{0}` 取内容，`git stash pop` 恢复（需授权）。

### 2.5 自动部署 / 备份 cron（见 §1.3，未改）

- 自动部署：已注释暂停，可恢复。
- 备份：`30 3 * * * … backup-daily.sh >> /var/log/zyagi-backup.log`（03:30 每日）。
- 证书续期：`2 2,8,14,20 * * * acme.sh --cron`（每日 4 次检查）。
- 健康巡检：`*/2 * * * * health-watch.sh`；bridge/dsh ensure 守护在位。

### 2.6 域名 / HTTPS / 证书 / 端口 / RP ID / origin

| 项 | 值 |
|---|---|
| 域名 | `zhiyao.duckdns.org` |
| HTTPS 监听 | **`18080`（非默认端口）** `ssl`；`80` 明文（跳转） |
| 反代 | `location /` → `http://127.0.0.1:3111`；`/health` → `:3111/health` |
| 证书 | `/etc/nginx/ssl/zhiyao/fullchain.pem`，CN=`zhiyao.duckdns.org`，Let's Encrypt，`2026-11-11 ~ 2027-02-09`（有效）；SAN 仅该域名 |
| **RP ID / origin** | **生产未接线 WebAuthn/Passkey**：`config.web` 键为 `enabled,displayName,maxSessionsPerUser,maxMessagesPerSession,sessionTtlDays,maxBodyBytes,defaultModel,ownerId,account,access,unifyQqIdentity`，**无 rpId/origin/passkey/webauthn**；源码 `grep expectedOrigin/rpId` 为空 |
| 计划值（候选引入后） | `rpId=zhiyao.duckdns.org`；`origin=https://zhiyao.duckdns.org:18080`（**含非默认端口 18080**）——仅为计划，**未验、未配置** |

> **重要**：Passkey/WebAuthn 属候选新能力，生产当前**无实体配置**。故"M3 真实 Passkey 注册/登录"须候选到位且主人授权后与主人协作，本任务**保持 Passkey 生产关闭、不伪造 Face ID**。

### 2.7 Gitee 与生产差距

- origin：`https://gitee.com/qq1846702260/zyagi.git`（fetch/push）。
- `git rev-list --left-right --count HEAD...@{u}` = **55 / 0**：本地 HEAD 领先已记录的 upstream 55 提交（**未 fetch，upstream ref 可能过期**，真实差距需 fetch 后核，本任务不做 fetch 以保持仓库只读）。
- 结论：Gitee 与生产存在待同步差距，发布时应以 C 认定并推送的正式候选坐标为准，不以本地 55 计数推定。

---

## 3. M1 隔离演练：**BLOCKED**

- 触发条件：`C-AUTH-V2-FINAL-RELEASE-CANDIDATE-01` 唯一最终候选交付。
- 现状：候选未交付（§1.1）。**不做**：不下载/合并/重建替代候选、不测 `fe06629b` 等过期坐标。
- 待候选到达后再执行（按任务书 M1）：节点 ID 直取 → 校验哈希/祖先/干净 bundle 同树 → 隔离目录+独立端口+合成身份 → 禁连真实记忆/写生产配置/发真实消息 → 验构建启动/单实例/CPU·RSS 上限/Web·PWA 入口 → 仅覆盖变更与环境依赖。

---

## 4. M2 身份迁移与安全恢复：部分可做，核心 BLOCKED

### 4.1 可做（只读/隔离）：极空间 NAS M3 证据

- 见 §1.2：11-25 03:30 失败（remote-missing）、04:50 成功（分片回读+重组复核 verified）；NAS 后端 `zspace`、`shard_enabled=1`、`ZY_ALLOW_PRODUCTION=1`、`ZY_NAS_DIR` 已设。
- 本地可信回滚点：`/root/backups/zyagi-20261125T045034.tar.gz`（33.9M，sha256 `47d0c7c8…`）+ `ROLLBACK-Z-BACKUP-ENABLE-01-20261125T045012/`。
- 未删除任何 NAS 历史包/旧证据；本轮**未新增远程上传**。

### 4.2 迁移工具现状

- 生产存在：`packages/companion/scripts/merge-identity.mjs`（11-14，数据合并）、`zy-migrate.mjs`（11-12）。
- 但 AUTH-V2 的"旧口令+本机受控迁移 / KEK / Passkey 撤销不复活"实现**不在生产代码**（`grep KEK|keywrap|passkey|webauthn|rpId` 生产 src 为空）。

### 4.3 核心 BLOCKED 项（须候选到位）

| 项 | 状态 | 原因 |
|---|---|---|
| 旧口令+本机受控迁移幂等 / 主人 ID 不变 / 普通·服务身份不提升 | **BLOCKED** | 迁移实现随候选交付，现无 |
| 迁移后旧令牌按契约撤销、旧密码/会话不可绕过 | **BLOCKED** | 同上 |
| 独立 KEK 注入、禁止随密文包上传、fail-closed（缺失/错密钥/代际锚点不一致） | **BLOCKED** | 生产无 KEK 代码 |
| 撤销后的 Passkey/会话/恢复码不得经旧备份复活 | **BLOCKED** | 生产无 Passkey，无法验证 |
| 离线恢复（受控本机+一次性高熵凭证，重置后撤会话/通行密钥、重绑 TOTP、ownerId 不变） | **BLOCKED** | 同上 |

> 说明：以上**不得用合成数据在旧工具上"假装通过"**——旧 `merge-identity.mjs` 与候选新迁移语义不同，测它不能代表候选；按任务书"未验证不能写通过"，一律标 BLOCKED。

---

## 5. M3 真机验证准备与发布方案

发布方案与真机验证准备清单见独立文档《2026-11-26_AUTH-V2阶段发布前隔离验收_发布回滚方案.md》。要点：
- 虚拟认证器 ≠ 苹果真机；真实 Passkey 注册/登录须主人授权协作，**Passkey 生产继续关闭**，不伪造 Face ID。
- 计划 RP ID/origin：`zhiyao.duckdns.org` / `https://zhiyao.duckdns.org:18080`（未验）。
- 明确可启用 vs 仍关闭能力；SMTP 不新增、不迁 QQ 官方、不涉桌宠。

---

## 6. 风险分级与发布门禁（规则 §7）

| 维度 | 结论 |
|---|---|
| A 类阻断 | **无法判定**（候选未到，无法验证身份迁移/KEK/Passkey 是否引入 A 类风险） |
| M1 隔离演练 | BLOCKED（候选未到） |
| M2 核心 | BLOCKED（KEK/Passkey/迁移实现不在产线） |
| M0 只读盘点 | 通过（证据完整） |
| M3 方案 | 已备，含未验项标注 |
| P0-BACKUP-ENABLE-01 | 未关闭（03:30 定时链路 remote-missing，待 11-26 03:30 观察） |

**发布门禁**：A 类阻断未清零 + 候选未到 → **不建议进入生产发布审批**，继续保持冻结。A 类阻断零方可建议发布。

---

## 7. 未执行与边界声明

未部署、未拉取/合并/切换、未改任何生产文件/配置/权限/令牌、未重启服务、未启停自动部署、未改 cron、未新增远程备份上传、未删除 NAS 历史包、未执行真实身份迁移、未做真实模型调用或真实 Passkey。仅只读 SSH（paramiko，凭据经加密库 `tok` 解密、不进 argv/日志）执行 `git/ps/ss/nginx -T/openssl/stat/ls/tail/grep/node 只读`。**本材料不构成部署授权**；是否上线由主人另行决定。

## 8. 证据采集方式

| 证据组 | 命令/文件 | 时点 |
|---|---|---|
| Git 坐标 | `git rev-parse HEAD/HEAD^{tree}`, `git status --porcelain`, `git rev-list --left-right` | 00:35 |
| 前端构建 | `packages/web-v1/dist/build-info.json` | 00:38 |
| 进程/单实例 | `ps -eo …`, `ss -lntp`, `systemctl show MainPID` | 00:36 |
| 资源 | `nproc`, `free -m`, `df -h`, `node -v` | 00:36 |
| cron | `crontab -l`, `systemctl list-timers` | 00:36 |
| nginx/证书 | `nginx -T`, `openssl x509 -in fullchain.pem` | 00:39 |
| RP/origin | `config.web` 键名（脱敏）、`grep expectedOrigin/rpId` | 00:39 |
| 备份 M3 | `/var/log/zyagi-backup.log`, `runtime/nas-status.json`, `/root/backups` | 00:42 |
| 迁移工具 | `ls scripts/`, `grep KEK/passkey/webauthn` | 00:42 |
