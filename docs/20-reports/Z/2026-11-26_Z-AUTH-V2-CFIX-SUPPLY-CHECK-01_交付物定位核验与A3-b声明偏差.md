# Z-AUTH-V2-CFIX-SUPPLY-CHECK-01 · C 交付物定位核验与 A3-b 声明偏差（脱敏版）

- 执行侧：Z；时间：2026-11-26 05:40（北京时间）
- 关联候选：HEAD `d311e152` / Tree `d7a0db1f` / Parent `2ad957f`
- 性质：只读核验，未连生产、未部署、未修改 C 侧任何交付物

## 一、核验缘由

G 侧反馈：C 声称资料库批次已交，但在「知遥AGI-C」下查不到 CFIX 批次，bundle / patch / SHA256 / 文档全缺，目前只有 Gitee 单通道。Z 按统一规则 §4.1「节点 ID 直取」方法复核。

## 二、方法

以节点 ID 直取 C 侧目录（不按空间列目录，规避 `12607`），逐个读取 78 个子节点；三个 drive 文件实际下载后重算 sha256；bundle 干净克隆后核对树哈希与 diff 规模；源码级核对锁定门禁覆盖范围。

## 三、结果

### 3.1 批次存在且完整

批次 `2026-11-26_C-AUTH-V2-BOOT-RESTORE-CFIX-01` 建立于 11-26 02:19，下挂 9 项：两份 patch、完整历史 bundle、完整性验证报告、逐文件 SHA256 清单、Z 接收与定向复验指南、测试证据、生产范围清单、精简报告。

### 3.2 哈希实测一致

| 文件 | 清单 | 实测 | 结果 |
| --- | --- | --- | --- |
| 01-C增量 patch | `a89dc5b044c8…` | 同 | 一致 |
| 01b-全量增量 patch | `f5178d2f2e38…` | 同 | 一致 |
| 02-完整历史 bundle | `7428893f7bb6…` | 同 | 一致 |

### 3.3 同树复现

bundle 克隆 → checkout `d311e152` → `git write-tree` = `d7a0db1f`，Parent `2ad957f`，diff 10 files / +805 −19。四件套齐全，可独立复现。

### 3.4 为何「查不到」

1. 按空间维度列目录返回 `12607 permission denied`，属权限模型正常现象，不代表文件不存在；
2. C 侧根下另有同名嵌套旧目录 `知遥AGI-C`（11-08 建立，仅含 01-开工必读、02-部署与服务器等 6 项旧分类），不含任何 2026-11-26 批次；真实批次平铺在上一级根目录。

## 四、A3-b 声明偏差

C 文档称「旧 Cookie 主人操作全部 423」。源码核对：`webapi.js` 中 `restoreLocked` 出现 0 次；`/api/web/session-info` 用 `curEx.kind`、`/api/web/persona/*` 用 `sessions.ownerId`、`ensureIdentity` 用 `isOwnerPrincipal` 判主人；`#assertNotRestoreLocked` 仅存在于 `auth/service.mjs` 的 6 处内部方法。门禁只覆盖 auth 子路由，未覆盖 webapi 会话、人格、设备与 `/api/ws/*`。

风险分级：A 类（主人身份可被旧备份 Cookie 绕过）。

## 五、结论与建议

1. 资料库批次存在、齐全、可读、可复现，C 无需补交文件；「只有 Gitee 单通道」不成立。
2. 需 C 补：批次内 `08-更正说明` 收窄声明范围；修复 A3-b（webapi 身份解析统一接入 restoreLocked，锁定态返回 423，`status` 不叠加 `full`）。
3. 发布建议维持**不建议发布**；自动部署与生产热修继续冻结。

## 六、边界

未重跑 C 测试套件（继承 auth-v2 57/57、相邻 775/775、锁护栏 2/2、E2E 26/26）；未连生产、未部署、未改配置、未动 release；未修改 C 侧节点。
