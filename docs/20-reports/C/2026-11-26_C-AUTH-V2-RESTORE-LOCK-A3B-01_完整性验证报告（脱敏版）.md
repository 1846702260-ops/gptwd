# 03 · 完整性验证报告（C-AUTH-V2-RESTORE-LOCK-A3B-01）

## 1. 唯一候选坐标

| 项 | 值 |
|---|---|
| 分支 | `cand/auth-v2-restore-lock-a3b-01` |
| HEAD | `adfc9ede8fada8ac171977bd1802ac4b70860031` |
| Tree | `7523e2038ed5e55acee606c78829c9c2bf875705` |
| Parent | `d311e1520fdccfd32503818f32ffb1a18a101583`（上轮候选 C-AUTH-V2-BOOT-RESTORE-CFIX-01） |
| 共同基线 | `960e6f2` |
| 托管 | Gitee `qq1846702260/zyagi` |
| release/z1.4.7 | `c9e1b9fc7f64719806187af291996bfd8450c1bb`（**未动**） |

远端回读：`git ls-remote` 返回 `adfc9ede8fada8ac171977bd1802ac4b70860031`，与本地 HEAD 一致。

干净复现：从 `02-完整历史bundle.bundle` 全新克隆后 `git checkout adfc9ed` → `git write-tree` = `7523e2038ed5e55acee606c78829c9c2bf875705`，与候选 Tree **精确吻合**。

## 2. 缺陷复现（修改前）与修复确认（修改后）

证据脚本：`test/auth-v2/c-a3b-defect-evidence.mjs`（真实启动路径，整目录回滚后同一枚旧 Cookie 逐入口实测）。

### 修改前（A 段）—— 越权确实存在

| 入口 | 修改前结果 |
|---|---|
| `GET /api/web/auth/status` | `state=full`（full=true，locked=true）← **同一时刻既是 full 又已锁定** |
| `GET /api/web/session-info` | 200，kind=owner |
| `GET /api/web/sessions`（工作区会话列表） | 200 |
| `GET /api/web/devices` | 200 |
| `POST /api/ws/projects`（工具/工作区写入） | 200 |
| `GET /api/web/persona/status`（人格） | 200 |

### 修改后（B 段）—— 全入口拒绝

| 入口 | 修改后结果 |
|---|---|
| `GET /api/web/auth/status` | `state=anonymous`、full=false、locked=true |
| `GET /api/web/session-info` | 401 |
| `GET /api/web/sessions` | 401 |
| `GET /api/web/devices` | 401 |
| `POST /api/ws/projects` | 401 |
| `GET /api/web/persona/status` | 401 |

## 3. 根因

上一轮（`d311e152`）的 `restoreLocked` 共同门禁只挂在 `AuthService.#requireOwner`，覆盖密码登录、TOTP、恢复码、Passkey。但网页侧身份判定走的是另一套 —— `WebSessions.resolveEx()`。**两套身份层没有打通**，逐路由补 `if` 必然遗漏，`session-info` / `devices` / 工作区 / 工具 / 人格五个入口全部漏掉。

本轮把闸门下沉到 `resolveEx` 这一**唯一**的"令牌 → 身份"入口，一次覆盖所有主人专属入口。

## 4. 修复要点

### 4.1 统一身份授权层（`src/websession.js`）

- 新增 `ownerLockProvider`（注入函数）与 `ownerLockState()`。
- `resolveEx()` 在拿到 `kind === OWNER` 之后、返回身份之前插入统一闸门：锁定期间主人令牌**解析不出身份**（返回 `null`），下游一律按"无身份"处理（401 / anonymous）。
- `ownerLockState()` 对 provider 抛错按**锁定**处理（fail-closed）：判不清就拒绝，绝不放行。
- 锁定拒绝写审计 `identity.owner_lock_denied`，按令牌哈希 10 秒节流，避免锁定期刷爆审计日志。
- 影响面：网页 HTTP、认证 V2 路由、会话列表、设备、工作区、工具/WS、人格等全部主人专属入口，**不再逐路由遗漏**。
- 锁的是**主人**，不是整个网站：普通用户与服务身份不受影响（不阻断他人）。

### 4.2 锁定判定实时化（`src/auth/service.mjs`）

- 装配时注入 provider：`webSessions.setOwnerLockProvider(() => authService.restoreLocked)`。
- `restoreLocked` 每次实时读取 `store.read().generation` 与锚点比对（不缓存），代际回退或锚点缺失即锁定。
- 锁定期间认证 `status` 拿不到主人身份 → 只能返回 `anonymous`，不可能同时 `full`；同时附 `restoreLocked` 字段供前端提示（四态契约不变）。

### 4.3 解锁前清账（`src/auth/service.mjs` `acceptRestore`）

受控恢复解锁（`zyagi-admin auth-verify-restore --accept`）按顺序执行：

1. **撤销全部主人旧会话** —— 走 `revokeWhere`，同时写入独立墓碑文件 `revocation-tombstone.jsonl`（只增不改、0600、附 seal 防篡改）。即使 `identities.json` 之后被旧快照整体回写，墓碑仍在，旧 Cookie 依旧判无效 → **解锁后不会复活**。
2. **撤销全部未撤销通行密钥**（Passkey）。
3. **推进 TOTP 计数器**到当前时间步，旧验证码不可重放。
   - ②③ 在 `store.transact` 同一事务内，失败整体不落盘。
4. **重新锚定**到当前代际（`store.commitAnchor`，写不进去抛 `anchor_write_failed` 500 → 仍处锁定，fail-closed，绝不当成功）。

保留能力：`ownerId` 不变（不重建主人），密码与 TOTP 密钥保留，**合法离线恢复路径不受阻**。

### 4.4 CLI（`bin/zyagi-admin.js`）

`auth-verify-restore` 改走 `svc.acceptRestore()`，输出撤销条数、Passkey 撤销数、TOTP 计数器推进情况与 ownerId。未加 `--accept` 时区分 `ROLLBACK_SUSPECTED`（代际回退）与 `ANCHOR_MISSING`（锚点缺失），并说明锁定语义（网页/工作区/工具全部拒绝主人身份）。

## 5. 测试证据

### 5.1 本次缺陷相关测试（新增 3 个文件）

| 文件 | 数量 | 结果 |
|---|---|---|
| `test/auth-v2/c-a3b-owner-lock.test.mjs` | 6 | 6/6 通过 |
| `test/auth-v2/c-a3b-owner-lock-e2e.mjs` | 27 | 27/27 通过（真实启动路径） |
| `test/auth-v2/c-a3b-defect-evidence.mjs` | 前后对照 | 缺陷复现 ✅ / 修复确认 ✅ |

`c-a3b-owner-lock.test.mjs` 覆盖：T1 锁定前主人令牌可解析、普通用户/服务身份不受影响；T2 锁定后统一层解析为 null；T3 锁定期间 status 不得返回 full；T4 `acceptRestore` 先撤销旧会话/凭证并推进 TOTP 计数再重锚；T5 解锁后正负对照（新登录可用、旧 Cookie 不可用）；T6 幂等（未锁定时 `acceptRestore` 不改动任何数据）。

`c-a3b-owner-lock-e2e.mjs` 走真实路径 `真实 config → loadConfig → startCompanion → 真实 HTTP`，场景含：
- 锁定前：密码登录 full、status=full、session-info=owner、工作区会话列表 200
- 备份后发生真实状态写入（totp/setup 200）→ 代际前进
- 退出后旧 Cookie 立即失效（负向基线）→ 重新登录拿到新 Cookie
- **真实整目录回滚**（锚点在 dataDir 之外，未被覆盖）→ 代际 1 < 锚点 2，锁定成立
- 锁定期间：status 非 full、session-info / 工作区会话列表 / 工作区写入 / 设备 / V2 工作区 / 人格入口全拒、密码登录 423 `restore_locked`
- 锁主人不锁整站：匿名引导仍可拿到普通身份（非主人）
- 解锁前仍锁定 → 解锁已撤销旧会话（含备份带回的 Cookie）→ ownerId 不变 → 锚点已对齐
- 解锁后：旧 Cookie 仍不可用、打工作区仍 401、新登录正常 full、新会话可访问工作区（正控）

### 5.2 必要回归（仅本次受影响范围）

| 套件 | 结果 |
|---|---|
| `test/auth-v2/`（12 个文件） | 63/63 通过，0 fail |
| `test/web-v1/`（39 个文件） | 全部 0 fail |
| `test/security/`（23 个文件） | 全部 0 fail |
| `test/identity/`（2 个文件） | 全部 0 fail |
| `test/workspace/`（3 个文件） | 全部 0 fail |
| `test/persona/`（3 个文件） | 全部 0 fail |
| `test/ops/`（1 个文件） | 全部 0 fail |

**已确认的既有基线问题（非本轮引入）**：`test/persona-system/chenzhiyao-persona.test.mjs` 5 项 + `test/persona-system/injection.test.mjs` 3 项失败。用 `git stash` 移除本轮全部改动后重跑，失败集合**完全一致**（8 项），故属既有基线问题，不在本轮范围，未擅自扩大研发范围。

### 5.3 发布保护确认

- `scrypt` N=2^16 与 `otplib` 13.6.0 **未动**。
- 25 路由与四态契约**未改**（`status` 仅附加 `restoreLocked` 字段，不改四态语义）。
- 本轮 diff 未触碰备份/bridge/switch/learning/前端。

## 6. 交付物清单

| 文件 | 说明 |
|---|---|
| `01-C增量-相对前候选.patch` | 相对 `d311e152`（上轮候选）的增量，881 行 |
| `01b-全量增量-相对共同基线.patch` | 相对共同基线 `960e6f2` 的全量增量，11200 行 |
| `02-完整历史bundle.bundle` | 完整历史（`git bundle create --all`） |
| `03-完整性验证报告.md` | 本文件 |
| `04-逐文件SHA256清单.txt` | 逐文件 SHA256（不自指） |
| `05-Z接收与定向复验指南.md` | Z 接收与复验步骤 |
| `06-测试证据与变更说明.md` | 测试证据明细 |
| `07-生产范围清单与裁决点.md` | 生产范围、留债与裁决点 |
| `08-原交付文档更正说明.md` | 对上轮文档"旧 Cookie 全部受保护"不实声明的更正 |
| `2026-11-26_C-AUTH-V2-RESTORE-LOCK-A3B-01-精简报告.txt` | 精简报告 |

## 7. 边界声明

- **未连生产**，未部署生产，自动部署未触碰。
- **未修改 release/z1.4.7**（始终 `c9e1b9f`），未强推。
- 未擅自清令牌 / 轮换凭证。
- 本轮仅做本次缺陷相关测试，已通过证据按规则继承，未重复验证。
